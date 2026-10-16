## 前言
这次分两部分：一是直接用 Python 调用 DeepSeek API（`ai/1.py`），亲眼看到请求和响应的完整结构；二是完成 `traning_repo/skeleton`，写一个生产级的石头剪刀布决策网关，处理 AI 会返回的各种坏输出。

任务书里说"使用 Deepseek 官方 API"——本质上就是发一个 HTTP POST，拿回一个 JSON，再从里面取出 AI 的回答。难的不是发请求，而是 AI 的回答永远不可信，需要设计多层容错。

完成之后有 93 个测试全绿，容错覆盖了 12 种 AI 坏回复的场景。记录里保留了完整的实现过程、每个函数为什么这么写，以及从 `main` 分支（空实现）到 `my-solution` 分支（完整实现）的每一处改动。

## 一、DeepSeek API 是什么
### 1. 接口格式
DeepSeek 兼容 OpenAI 的 Chat Completions 接口格式。发的请求体：

```json
{
  "model": "deepseek-chat",
  "messages": [
    {"role": "system", "content": "你是石头剪刀布的决策者，用 JSON 格式回答……"},
    {"role": "user",   "content": "{\"time\": \"2026-10-16T12:00:00+00:00\", \"history\": [\"rock\", \"rock\"]}"}
  ],
  "response_format": {"type": "json_object"},
  "temperature": 0.2,
  "max_tokens": 128
}
```

`messages` 是一个列表，`system` 给模型定角色，`user` 发本轮问题。`response_format: json_object` 要求模型返回合法 JSON（但这不是 100% 的保证，后面会看到为什么）。`temperature` 越低输出越稳定，石头剪刀布这类任务设 0.2 就够了。

API 的鉴权通过 `Authorization: Bearer sk-xxx` 请求头完成。key 不要写死在代码里，从环境变量读：

```python
import os
api_key = os.environ.get("DEEPSEEK_API_KEY")
```

### 2. 响应结构
成功时响应体：

```json
{
  "id": "chatcmpl-xxx",
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "{\"time\": \"2026-10-16T12:00:00+00:00\", \"option\": \"paper\"}",
      "reasoning_content": null
    },
    "finish_reason": "stop",
    "index": 0
  }],
  "usage": {
    "prompt_tokens": 87,
    "completion_tokens": 28,
    "total_tokens": 115
  }
}
```

实际答案在 `choices[0].message.content`。`finish_reason` 是 `"stop"` 表示正常结束，`"length"` 表示被 `max_tokens` 截断——截断时 content 可能是半截 JSON，容错层需要处理这种情况。

思考模型（比如 deepseek-reasoner）还会额外返回 `reasoning_content`，这是思维链，不是答案，**必须忽略**。如果只有 `reasoning_content` 而 `content` 为空，网关要把这次调用当做失败来重试。

## 二、用 openai SDK 先跑一个实验（ai/1.py）
### 1. 为什么要先跑 ai/1.py
在写网关之前，先直接调一次 API 有三个好处：

1. 验证 key 是否有效、余额是否够
2. 眼见为实地看到原始响应结构，而不是从文档里想象
3. 确认 `response_format: json_object` 确实让模型返回了 JSON

### 2. 运行 ai/1.py
`D:\Downloads\lc\ai\1.py` 用 openai SDK 调用 DeepSeek（DeepSeek 兼容 OpenAI SDK，只需要换 `base_url`）：

```python
from openai import OpenAI
import os, json

client = OpenAI(
    api_key=os.environ.get("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
)

request_data = {
    "time": "2026-10-16T12:00:00+00:00",
    "history": ["rock", "rock", "scissor"]
}

response = client.chat.completions.create(
    model="deepseek-chat",
    messages=[
        {"role": "system", "content": "你是石头剪刀布决策者，只返回 JSON，字段为 time 和 option（rock/scissor/paper）。"},
        {"role": "user",   "content": json.dumps(request_data, ensure_ascii=False)},
    ],
    response_format={"type": "json_object"},
    temperature=0.2,
    max_tokens=128,
)

content = response.choices[0].message.content
print(content)
print("tokens:", response.usage.total_tokens)
```

PowerShell 里运行：

```powershell
$env:DEEPSEEK_API_KEY = "sk-你的key"
cd D:\Downloads\lc\ai
python ./1.py
```

典型输出：

```plain
{"time": "2026-10-16T12:00:00+00:00", "option": "paper"}
tokens: 115
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792161602383-e74ee8c9-4f98-435c-8440-3758d9fe0ed9.png)

### 3. 观察 finish_reason 和 usage
把输出改成打印完整对象可以看到更多信息：

```python
choice = response.choices[0]
print("finish_reason:", choice.finish_reason)
print("content:", choice.message.content)
print("reasoning_content:", choice.message.reasoning_content)
print("prompt_tokens:", response.usage.prompt_tokens)
print("completion_tokens:", response.usage.completion_tokens)
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792161984197-f894637b-eeae-481d-9820-92fe27beef6e.png)`finish_reason` 正常是 `stop`。如果 `max_tokens` 设得很小（比如 10），会看到 `length`，content 会是截断的半截 JSON，这就是后面容错层要处理的场景之一。

`reasoning_content` 在普通的 `deepseek-chat` 模型上是 `None`。如果用 `deepseek-reasoner`，这里会有思维链文本，长度可能是 content 的几十倍，而且思维链里的"想法"不保证和最终 content 一致，所以必须忽略。

## 三、为什么需要容错（json_guard.py 的三级保险）
### 1. mock_llm_server.py 的 12 种坏回复
开了 `response_format: json_object` 还是会出问题。`mock_llm_server.py` 预设了 12 种真实 AI 返回过的坏回复：

```powershell
cd D:\Downloads\lc\traning_repo\skeleton
python mock_llm_server.py --list
```

```plain
 1. ```json\n{"time": "{time}", "option": "rock",}\n```   ← 围栏 + 尾随逗号
 2. 好的，我出剪刀：{"time": "{time}", "option": "scissor"}  ← 前后夹带自然语言
 3. {"time": "{time}", "option": "SPOCK"}                   ← 非法取值
 4. 抱歉，我无法完成这个请求。                              ← 完全不是 JSON
 5. {"option": "paper"}                                     ← 缺 time 字段
 6. {"time": "{time}", "option": "ROCK", "extra": 123}      ← 大写 + 多余字段
 7. {'time': '{time}', 'option': 'paper'}                   ← 单引号
 8. {"time": "{time}", "option": "石头"}                    ← 中文
 9. {"time": "1999-01-01T00:00:00+00:00", "option": "scissor"}  ← time 不一致
10. （空回复）
11. {"time": "{time}", "option": "paper"                    ← 缺右括号
12. {"time": "{time}", "option": "scissor"}                 ← 正常
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792162019638-8f488231-7cc5-49f5-91d6-a9bacdd6e2c4.png)场景 1-2 是 JSON 格式问题（需要文本层处理），3-9 是字段语义问题（JSON 解析成功但字段非法，需要规则层处理），10-11 是边界极端情况，12 是正常。

### 2. 三级处理顺序
**第一级（**`strip_code_fence`**）**：AI 喜欢把代码包在 markdown 围栏里，遇到 ` ```json {...} ``` ` 就把围栏内容提取出来。

**第二级（**`find_first_json_object`**）**：文本前后夹带自然语言时（"好的，我出：{...}"），从中找出第一个括号配平的 `{...}`。难点是字符串感知——不能把字符串内部的 `}` 算作结束。

**第三级（**`repair_json_text`**）**：对有点小毛病但能救回来的文本做修复：去注释、Python 字面量转换、单引号变双引号、尾随逗号、缺失右括号。

三级都没救回来，走本地兜底出拳（`fallback_decision`）。

### 3. 为什么不用 eval
AI 的回答是不可信输入，`eval` 或 `exec` 会把它当代码执行。一个精心构造的回答可以通过 `eval` 执行任意 Python 代码。容错层只做可解释的文本级操作，不做语义推测，也不执行任何 AI 给的代码。

## 四、git diff 看两个分支的改动
`main` 分支（或初始提交）里，所有 10 个 TODO 函数都是：

```python
raise NotImplementedError("TODO X：实现 xxx（见函数文档里的提示）")
```

`my-solution` 分支里是完整实现，93 个测试通过。用 git diff 对比：

```powershell
cd D:\Downloads\lc\traning_repo
git diff 8355c4e 5f8b229 --stat
```

```plain
skeleton/doc/LLM实践任务.md  | 184 ++++++++++++++++++
skeleton/gateway/app.py      | 109 变动
skeleton/gateway/client.py   | 205 变动
skeleton/gateway/json_guard.py | 148 变动
skeleton/gateway/rules.py    | 154 变动
5 files changed, 451 insertions(+), 349 deletions(-)
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792162075025-a3f8adb3-f3d9-4ce2-81a4-b9a8c907d40b.png)变动最大的文件是 `client.py` 和 `rules.py`，因为它们包含了最复杂的逻辑。`json_guard.py` 变动量看起来不多，但那是因为大量的改动是把 `raise NotImplementedError` 换成了实际实现，以及删掉了大量 TODO 提示注释。

## 五、10 个函数详解（json_guard.py 部分）
### 函数 1：strip_code_fence
**所在文件**：`gateway/json_guard.py`  
**作用**：如果 AI 在答案外面套了 markdown 代码块，把围栏剥掉，只返回内部内容。

**实现**：

```python
def strip_code_fence(text: str) -> str:
    if not isinstance(text, str) or "```" not in text:
        return text
    parts = text.split("```")
    bodies: List[str] = []
    for index in range(1, len(parts), 2):
        body = parts[index]
        newline = body.find("\n")
        if newline != -1:
            head = body[:newline].strip()
            if len(head) <= 20 and "{" not in head:
                body = body[newline + 1:]
        body = body.strip()
        if body:
            bodies.append(body)
    if not bodies:
        return text
    for body in bodies:
        if body.lstrip().startswith("{"):
            return body
    return bodies[0]
```

**逐步理解**：

`text.split("```")` 把文本按三个反引号切开。切完之后，下标 0、2、4…… 是围栏外的文本，下标 1、3、5…… 是围栏内的文本，所以 `range(1, len(parts), 2)` 正好取到所有围栏内容。

围栏的第一行通常是语言标记（`json`、`js`、`JSON`）——这种行的特征是：很短（`≤ 20` 个字符），且不包含 `{`。满足这两个条件就把第一行丢掉，用剩下的内容。长度限制防止把 JSON 的第一行误判为语言标记。

有多个围栏时优先返回第一个以 `{` 开头的，因为这最像 JSON；都不是就返回第一个非空片段。

### 函数 2：find_first_json_object
**所在文件**：`gateway/json_guard.py`  
**作用**：在自由文本里找出第一个括号配平的 `{...}` 子串。处理场景 2（AI 回复自然语言后夹着 JSON）。

**实现**：

```python
def find_first_json_object(text: str) -> Optional[str]:
    if not isinstance(text, str):
        return None
    length = len(text)
    start = 0
    while start < length:
        if text[start] != "{":
            start += 1
            continue
        depth = 0
        in_string = False
        escaped = False
        index = start
        while index < length:
            char = text[index]
            if in_string:
                if escaped:
                    escaped = False
                elif char == "\\":
                    escaped = True
                elif char == '"':
                    in_string = False
            else:
                if char == '"':
                    in_string = True
                elif char == "{":
                    depth += 1
                elif char == "}":
                    depth -= 1
                    if depth == 0:
                        return text[start:index + 1]
            index += 1
        start += 1
    return None
```

**逐步理解**：

外层循环找每一个 `{` 作为候选起点。内层循环从这个 `{` 开始，用三个状态变量追踪：

+ `depth`：花括号嵌套深度，从 1 开始（找到第一个 `{` 后进入），`}` 让它减 1，减到 0 说明找到了配平的结束位置
+ `in_string`：当前是否在字符串内部——字符串内部的 `{` 和 `}` 不参与计数
+ `escaped`：上一个字符是否是 `\`——`\"` 不算字符串结束

如果内层循环跑完了 `depth` 没回到 0（即这个 `{` 没有配平的 `}`），外层把 `start` 加 1，尝试下一个 `{`。

关键边界：`escaped` 只在字符串内部处于 `in_string=True` 时才有意义。如果代码只检查 `char == "\\"` 而不先判断 `in_string`，文本 `\{}` 这种字符串外的反斜杠会错误地触发转义状态。

### 函数 3：repair_json_text
**所在文件**：`gateway/json_guard.py`  
**作用**：对能识别但有小毛病的 JSON 文本做修复。

**实现**：

```python
def repair_json_text(text: str) -> str:
    if not isinstance(text, str):
        return text
    repaired = _strip_comments(text).strip()
    repaired = _PY_LITERAL_RE.sub(lambda match: _PY_LITERALS[match.group(1)], repaired)
    repaired = re.sub(r",(\s*[}\]])", r"\1", repaired)
    repaired = re.sub(r"([}\]])[\s,]*$", r"\1", repaired)
    repaired = _normalize_strings(repaired)
    return _close_unfinished(repaired)
```

**六步流水线**：

1. `_strip_comments(text).strip()`：去掉 `//` 和 `/* */` 注释。注意字符串内部的 `//` 要保留（比如 URL），所以用状态机而不是简单的正则替换。
2. `_PY_LITERAL_RE.sub(...)`：用正则把 `None/True/False` 替换成 `null/true/false`。`_PY_LITERAL_RE = re.compile(r"(?<![\w\"'])(None|True|False)(?![\w\"'])")` 用前后各自的负向断言，防止把 `TrueType` 里的 `True` 替换掉。
3. `re.sub(r",(\s*[}\]])", r"\1", repaired)`：去掉尾随逗号，即 `{...,}` 或 `[...,]` 里最后一个元素后面的逗号。正则匹配"逗号后面（允许空白）紧跟 `}` 或 `]`"，把逗号删掉。
4. `re.sub(r"([}\]])[\s,]*$", r"\1", repaired)`：去掉结尾多余的逗号和空白。有些 AI 回复会在最后一个 `}` 后面还有逗号或空行。
5. `_normalize_strings(repaired)`：把各种引号统一成标准 `"`，并把字符串内裸换行转义成 `\n`。单引号包裹的字符串、中文弯引号包裹的字符串，都被转换成双引号，方便后续 `json.loads`。
6. `_close_unfinished(repaired)`：补齐缺失的右引号和右括号。如果 AI 回复被 max_tokens 截断，JSON 可能到一半就结束了。这个函数用栈追踪未闭合的 `{` 和 `"`，在末尾补全。

**为什么这六步顺序不能乱**：

+ 必须先去注释再做其他处理，否则注释里的 `True/None` 也会被替换
+ 必须先去尾随逗号再补右括号，否则 `{"a": 1,` 补括号后变成 `{"a": 1,}`，仍然非法
+ 必须先规范化引号再补括号，否则补了括号后字符串的引号状态追踪会出错

## 六、10 个函数详解（rules.py 部分）
### 函数 4：normalize_option
**所在文件**：`gateway/rules.py`  
**作用**：把 AI 可能写出来的各种出拳表达统一成标准值（`rock`/`scissor`/`paper`）。

**实现**：

```python
def normalize_option(value: Any) -> Optional[str]:
    if not isinstance(value, str):
        return None
    return ALIASES.get(value.strip().lower())
```

**两行做了什么**：

第一行先排除非字符串——数字 `42`、列表 `["rock"]`、`None`、布尔值都不是有效出拳，直接返回 `None`。

第二行 `value.strip().lower()` 去掉首尾空格并转小写，然后查 `ALIASES` 字典：

```python
ALIASES = {
    "rock": "rock", "rocks": "rock", "石头": "rock", "拳头": "rock", "锤子": "rock",
    "scissor": "scissor", "scissors": "scissor", "剪刀": "scissor",
    "paper": "paper", "papers": "paper", "布": "paper",
}
```

`ALIASES` 覆盖了英文复数、中文别名、大小写变体（因为先 `.lower()`）。查不到说明这个值无法识别，返回 `None`，上层会走兜底。

注意这里的大写处理不在 `ALIASES` 里显式列出 `"ROCK": "rock"`——因为 `.lower()` 在查表之前已经把大写转成了小写，所以 `ALIASES` 里只需要小写形式。

### 函数 5：validate_decision
**所在文件**：`gateway/rules.py`  
**作用**：接收 AI 输出（`candidate` dict），结合原始请求（`request`），输出一个合法的决策 dict 和修正说明列表。永不抛异常。

**实现**：

```python
def validate_decision(candidate: Any, request: Dict[str, Any]) -> Tuple[Dict[str, Any], List[str]]:
    notes: List[str] = []
    if not isinstance(candidate, dict):
        fallback, extra = fallback_decision(request)
        return fallback, notes + ["非对象->兜底"] + extra

    option = normalize_option(candidate.get("option"))
    if option is None:
        fallback, extra = fallback_decision(request)
        return fallback, notes + ["option 非法->兜底"] + extra

    request_time = get_time(request)
    if request_time is not None:
        if normalize_time(candidate.get("time")) not in (None, request_time):
            notes.append("丢弃模型的 time")
        time_value = request_time
    else:
        model_time = normalize_time(candidate.get("time"))
        if model_time is not None:
            notes.append("请求无 time->用模型的 time")
            time_value = model_time
        else:
            notes.append("请求无 time->用当前时间")
            time_value = now_iso()
    return build_decision(time_value, option), notes
```

**三层判断**：

**第一层**：`candidate` 不是 dict（AI 返回了数组、字符串或其他类型）→ 直接兜底。

**第二层**：`option` 字段非法（不在 ALIASES 里）→ 直接兜底。这里会覆盖 mock 的场景 3（`"SPOCK"`）、场景 6（`"ROCK"` 大写——不对，`normalize_option` 会处理大写，这个能救回来）、场景 8（`"石头"`——查 ALIASES 能找到）。

**第三层**：`time` 字段的处理。这是整个网关最重要的设计决策：`time`** 以请求为准，不信任模型**。

+ 请求有 `time` → 用请求的，如果模型给了不同的 time，记录"丢弃模型的 time"
+ 请求没 `time` → 尝试用模型的（同时记录），都没有就用当前时间

最后 `build_decision(time_value, option)` 确保输出只有 `time` 和 `option` 两个键，多余字段一律不带出（覆盖场景 6 的 `"extra": 123`）。

### 函数 6：fallback_decision
**所在文件**：`gateway/rules.py`  
**作用**：不依赖 AI，纯本地计算出一个合法决策。任何输入都能给出合法结果。

**实现**：

```python
def fallback_decision(request: Dict[str, Any]) -> Tuple[Dict[str, Any], List[str]]:
    history = history_of(request)
    if history:
        option = counter_to(history[-1])
        notes = ["兜底:counter(%s)" % history[-1]]
    else:
        option = "rock"
        notes = ["兜底:rock"]
    time_value = get_time(request) or now_iso()
    return build_decision(time_value, option), notes
```

**逻辑**：

有历史出拳记录 → 反制对手上一手：`counter_to(history[-1])` 查 `BEATS` 字典，找出能赢最后一手的拳。比如对手上一手出 `"rock"`，`BEATS` 的键是"我的拳"、值是"被我赢的拳"，找到 `BEATS["paper"] == "rock"` 所以出 `"paper"`。

没有历史 → 出 `"rock"`（默认策略，无历史时没有更好的信息可利用）。

时间取请求里的，没有就用当前时间 `now_iso()`。注意这里 `get_time(request) or now_iso()` 而不是 `get_time(request) if get_time(request) else now_iso()`——因为 `get_time` 返回 `None` 或非空字符串，`or` 的行为正确，且只调用一次。

## 七、10 个函数详解（client.py 部分）
### 函数 7：build_body
**所在文件**：`gateway/client.py`  
**作用**：把请求 dict 组装成符合 OpenAI Chat Completions 格式的请求体。

**实现**：

```python
def build_body(self, request: Dict[str, Any]) -> Dict[str, Any]:
    user_content = USER_PREFIX + json.dumps(request, ensure_ascii=False, separators=(",", ":"))
    body: Dict[str, Any] = {
        "model": self.config.model,
        "messages": [
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": user_content},
        ],
        "stream": False,
        "temperature": self.config.temperature,
        "max_tokens": self.config.max_tokens,
    }
    if self.config.enable_json_format:
        body["response_format"] = {"type": "json_object"}
    body["thinking"] = {"type": "enabled" if self.config.enable_thinking else "disabled"}
    return body
```

**细节说明**：

`json.dumps(request, ensure_ascii=False, separators=(",", ":"))` 把请求 dict 序列化为 JSON 字符串放进 `user` 消息的 content。`ensure_ascii=False` 让中文原样保留（不转成 `\uXXXX`），`separators=(",", ":")` 去掉多余空格，减少 token 数量。

`SYSTEM_PROMPT` 是固定的系统提示，里面必须包含"json"这个词（大小写均可）——DeepSeek 的 JSON 模式要求系统提示里提到 JSON，否则即使开了 `response_format: json_object` 也可能报错。

`"stream": False` 显式关闭流式模式——流式模式下响应是一段段返回的，当前实现用 `response.json()` 一次性解析，流式会出错。

`thinking` 字段默认设为 `disabled`，关闭思维链，因为思维链会消耗大量 token 且 `reasoning_content` 不是我们需要的答案。

### 函数 8：decide
**所在文件**：`gateway/client.py`  
**作用**：带重试的 HTTP 发送主流程，返回 `(content 文本, 元信息 dict)`。

**实现（关键骨架）**：

```python
def decide(self, request: Dict[str, Any]) -> Tuple[str, Dict[str, Any]]:
    if not self.api_key:
        raise LlmAuthError("没有可用的 API Key")
    
    body = self.build_body(request)
    payload = json.dumps(body, ensure_ascii=False).encode("utf-8")
    headers = self.build_headers()
    attempts = max(1, self.config.max_retries + 1)

    last_error = None
    for attempt in range(1, attempts + 1):
        started = time.monotonic()
        try:
            response = self._session.post(self.endpoint, data=payload,
                                          headers=headers, timeout=timeout)
            latency = time.monotonic() - started
            status = response.status_code

            if status == 200:
                content, has_reasoning, finish_reason = self.extract_content(response)
                return content, {...}
            if status in (401, 403):
                raise LlmAuthError(...)      # 不重试
            if status == 400:
                raise LlmFatalError(...)     # 不重试
            if status in RETRY_STATUS and attempt < attempts:
                delay = self.retry_delay(attempt, response.headers.get("Retry-After"))
                self._sleep(delay)
                continue
            raise LlmError("HTTP %d" % status)

        except (LlmAuthError, LlmFatalError):
            raise                            # 直接透传，不重试
        except LlmError as exc:
            last_error = exc
            if attempt >= attempts: break
            self._sleep(self.retry_delay(attempt, None))
        except requests.exceptions.RequestException as exc:
            last_error = exc
            if attempt >= attempts: break
            self._sleep(self.retry_delay(attempt, None))

    raise LlmError("上游不可用（已尝试 %d 次）：%s" % (attempts, last_error))
```

**状态码分支设计**：

| 状态码 | 处理 | 原因 |
| --- | --- | --- |
| 200 | 正常返回 |  |
| 401 / 403 | 立刻抛 `LlmAuthError` | Key 错了，重试只是浪费钱 |
| 400 | 立刻抛 `LlmFatalError` | 请求参数非法，重试不会变好 |
| `RETRY_STATUS` (408/429/5xx) | 退避重试 | 临时故障，等一下可能恢复 |
| 其他 | 抛 `LlmError` |  |


`LlmError`（比如空回复、只有思维链）也是可重试的，放在 `except LlmError` 里处理。

`except (LlmAuthError, LlmFatalError): raise` 必须放在 `except LlmError` 之前——因为 `LlmAuthError` 和 `LlmFatalError` 都继承自 `LlmError`，如果顺序反了，它们会被 `except LlmError` 捕获并进入重试流程。

**指数退避**：第 N 次重试等 `base × 2^(N-1) + random(0, base)` 秒。第 1 次等约 1 秒，第 2 次约 2 秒，第 3 次约 4 秒。加随机抖动是为了防止多个客户端同时重试把服务器再打一遍（雷群效应）。如果服务器的 `Retry-After` 响应头给了等待时间，就用那个时间（但不超过 `backoff_max`）。

### 函数 9：extract_content
**所在文件**：`gateway/client.py`  
**作用**：从 HTTP 200 响应里提取 content 字段，区分各种空响应情况。

**实现**：

```python
@staticmethod
def extract_content(response: Any) -> Tuple[str, bool, str]:
    try:
        data = response.json()
    except ValueError as exc:
        raise LlmError("上游返回的不是 JSON：%s" % exc) from exc
    
    choices = data.get("choices") or []
    if not choices:
        raise LlmError("上游响应里没有 choices：%s" % (data.get("error") or data))
    
    choice = choices[0] or {}
    finish_reason = str(choice.get("finish_reason") or "")
    message = choice.get("message") or {}
    content = message.get("content")
    reasoning = message.get("reasoning_content")
    has_reasoning = isinstance(reasoning, str) and bool(reasoning.strip())
    
    if not isinstance(content, str) or not content.strip():
        if has_reasoning:
            raise LlmError("上游只返回了 reasoning_content（思维链），content 为空")
        if finish_reason == "length":
            raise LlmError("上游 content 为空且 finish_reason=length：max_tokens 不够用")
        raise LlmError("上游 content 为空（finish_reason=%s）" % finish_reason)
    
    return content, has_reasoning, finish_reason
```

**三种失败原因区分**：

+ 有 `reasoning_content` 但 `content` 为空：模型用了思维链模式，但主回答没生成出来。提示里说明原因，让上层（或运维）知道是 `max_tokens` 不够还是模型配置问题。
+ `finish_reason == "length"`：`max_tokens` 不够，生成被截断，content 可能是半截 JSON。
+ 其他：`content` 为 `null`、`""`，原因不明。

区分这三种原因很重要：`max_tokens` 不够的情况可以通过增大配置解决，思维链问题可以通过配置 `enable_thinking=False` 解决，而不是靠重试。错误信息帮助排查根本原因，而不仅仅告诉你"出错了"。

## 八、10 个函数详解（app.py 部分）
### 函数 10：handle_request
**所在文件**：`gateway/app.py`  
**作用**：串联所有零件——调上游、容错解析、合法性校验、兜底——保证永远输出一行合法决策。

**实现**：

```python
def handle_request(self, request: Dict[str, Any]) -> Tuple[Dict[str, Any], List[str]]:
    notes: List[str] = []
    decision: Optional[Dict[str, Any]] = None

    if self.client is not None:
        try:
            content, meta = self.client.decide(request)
            candidate = loads_object(content)
            decision, notes = validate_decision(candidate, request)
            self.log("上游返回 %.2fs（attempt=%s, finish=%s, 含思维链=%s）" % (...))
        except JsonGuardError as exc:
            notes.append("模型输出无法整理成 JSON: %s" % exc)
        except Exception as exc:
            notes.append("上游异常: %s" % exc)

    if decision is None:
        decision, extra = fallback_decision(request)
        notes = notes + ["本地兜底"] + extra

    if not is_valid_decision(decision):
        decision = build_decision(get_time(request) or now_iso(), "rock")
        notes.append("出口自检失败->rock")

    return decision, notes
```

**三道防线**：

**第一道**：在线模式才调上游（`self.client is not None`）。离线模式直接走本地兜底，省掉整个网络请求。调用链：`client.decide`（HTTP + 重试）→ `loads_object`（三级 JSON 容错）→ `validate_decision`（字段校验 + time 替换）。任何异常（`JsonGuardError`、网络超时、HTTP 错误）都被 catch，不往外抛。

**第二道**：`decision is None` 时走 `fallback_decision`。这一步在以下情况触发：

+ 离线模式
+ 在线模式但上游全部重试失败
+ `loads_object` 解析失败（JSON 完全无法救回来）
+ `validate_decision` 返回了兜底（option 非法等）

**第三道**：出口自检 `is_valid_decision(decision)`。理论上前两道已经保证了决策合法，这一道是最后的安全网，防止 `validate_decision` 或 `fallback_decision` 的实现里有 bug。检查失败就强制出 rock，同时记录到 notes 里方便排查。

`except Exception as exc`（捕获所有异常）：

这里 catch 了 `Exception`（不是 `BaseException`）。`BaseException` 包含 `SystemExit` 和 `KeyboardInterrupt`，这两个不应该被 catch。`Exception` 包含了所有运行时错误。这是网关的契约：对调用方来说，调用 `handle_request` 永不失败，永远返回 `(决策, 说明)` 元组。

## 九、运行验证
### 1. 运行测试套件
```powershell
cd D:\Downloads\lc\traning_repo\skeleton

# 自检（先确认所有 TODO 都实现了）
python selfcheck.py

# 完整测试
python -m pytest tests/ -v
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792162200200-a6fdabd5-9ba0-42be-9f96-7525200ed2c2.png)93 个测试的覆盖范围：

+ `test_json_guard.py`：`strip_code_fence`（5 个）、`find_first_json_object`（6 个）、`repair_json_text`（9 个）、`loads_object`（集成）
+ `test_rules.py`：`normalize_option`（别名覆盖）、`validate_decision`（各种字段异常）、`fallback_decision`（有无历史）
+ `test_client.py`：`build_body`（字段组装）、`decide`（重试逻辑、各 HTTP 状态码）、`extract_content`（空回复、截断、思维链）
+ `test_app.py`：`handle_request`（端到端各种异常）、`run`（逐行处理统计）

### 2. 离线模式验证
```powershell
cd D:\Downloads\lc\traning_repo\skeleton

# 手动构造输入
$input = '{"time":"2026-10-16T12:00:00+00:00"}' + "`n" +
         '{"time":"2026-10-16T12:01:00+00:00","history":["rock"]}'
$input | python run_gateway.py --offline
```

输出：

```plain
{"time":"2026-10-16T12:00:00+00:00","option":"rock"}
{"time":"2026-10-16T12:01:00+00:00","option":"paper"}
```

第一行无历史，兜底出 rock。第二行历史最后一手是 rock，反制出 paper。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792162395013-a499370d-047d-49d7-8ef9-43e73dd265c7.png)

### 3. 对着 mock 服务器验证容错
```powershell
# 终端 1：启动 mock 服务器（模拟 12 种坏回复）
python mock_llm_server.py --port 8801

# 终端 2：跑网关对着 mock 服务器
Get-Content samples\requests.jsonl |
    python run_gateway.py --api-base-url http://127.0.0.1:8801 --api-key test
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792162538534-8a84505b-7493-4374-b27b-fe6cf3fc47e8.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792162519621-a1006fad-01e4-488b-b6da-4405f0c5b9ef.png)PowerShell 里后台启动 mock 服务器并在同一个终端继续：

```powershell
$job = Start-Job { python D:\Downloads\lc\traning_repo\skeleton\mock_llm_server.py --port 8801 }
Start-Sleep 1   # 等服务器启动

Get-Content D:\Downloads\lc\traning_repo\skeleton\samples\requests.jsonl |
    python D:\Downloads\lc\traning_repo\skeleton\run_gateway.py `
        --api-base-url http://127.0.0.1:8801 --api-key test

Stop-Job $job
Remove-Job $job
```

stderr 会显示每行的处理过程，包括哪种容错被触发、是否走了兜底。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792162578291-5e6cdecc-f7b8-4f46-8cf6-79620bcfe5d5.png)

### 4. 真实 API 验证
```powershell
$env:DEEPSEEK_API_KEY = "sk-你的key"

Get-Content D:\Downloads\lc\traning_repo\skeleton\samples\requests.jsonl |
    python D:\Downloads\lc\traning_repo\skeleton\run_gateway.py
```

默认使用 `config.py` 里的 `deepseek-flash` 模型和 `https://api.deepseek.com` 端点。

## 十、time 字段为什么不用模型返回的
### 1. 问题来源
网关收到请求里有 `"time"` 字段，模型回的 JSON 里也有 `"time"` 字段。哪个算数？

直觉可能是"用模型给的，因为它更聪明"——但这是错的。

### 2. 可信边界原则
模型是外部输入，不是可信系统。`mock_llm_server.py` 的场景 9 故意返回 `"time": "1999-01-01T00:00:00+00:00"`，如果网关直接用模型给的 time，下游会收到时间轴完全错位的决策。

现实中 AI 模型在时间相关字段上的可靠性很低，因为：

+ 它没有当前时间的感知能力（训练数据有截止日期）
+ 即使提示里传了时间，它也可能随机"编"一个

**网关是可信边界**：它站在业务系统和外部 AI 中间，负责确保输出的数据格式和字段值符合协议。`time` 字段的值由网关决定，AI 给的只是参考（实际上直接忽略）。

`validate_decision` 里的实现体现了这个原则：请求里的 `time` 永远优先，模型给的 `time` 在有请求时间的情况下被丢弃，并记录到 notes 里。

## 十一、重试策略详解
### 1. 哪些情况重试，哪些不重试
这是 `decide` 函数里最需要细想的地方。

**应该重试**：

+ 网络超时（服务器暂时忙或网络抖动）
+ HTTP 429（限流，等一下再试）
+ HTTP 5xx（服务器端临时故障）
+ HTTP 408（请求超时）
+ `LlmError`（空回复、截断——可能是偶发问题）

**不应该重试**：

+ HTTP 401/403（API Key 错误，重试不会让 Key 变对，只是浪费钱和时间）
+ HTTP 400（请求参数非法，重试发同样的请求还是 400）

`RETRY_STATUS = frozenset({408, 409, 425, 429, 500, 502, 503, 504, 522, 524})` 这些都是服务端临时不可用的信号。

### 2. 指数退避公式
```plain
delay(N) = base × 2^(N-1) + random(0, base)
```

`N` 是第几次重试（从 1 开始），`base` 默认是 1 秒：

+ 第 1 次重试：等约 1-2 秒
+ 第 2 次重试：等约 2-3 秒
+ 第 3 次重试：等约 4-5 秒

加随机量（jitter）的原因：如果多个客户端同时在第 1 秒遇到了 503，它们会在第 1 秒后同时重试，同时把服务器再打一遍。加了随机量之后，它们的重试时间分散开，服务器负载更平均，更容易恢复。

## 十二、网关的架构意义
### 1. 为什么要有"网关"这一层
游戏程序（`play_game.py`）不直接调用 AI，而是通过管道发一行 JSON 给网关（`run_gateway.py`），网关保证返回一行干净的 JSON。

这个设计解耦了两个问题：

+ `play_game.py` 只需要知道"怎么玩游戏"，不需要知道"怎么和 AI 通信"
+ 网关只需要知道"怎么从 AI 拿到一个合法的决策"，不需要知道"游戏规则"

好处：换一个 AI 服务（比如从 DeepSeek 换成 Claude），只需要改网关，游戏程序完全不动。添加新的容错逻辑，也只在网关里改，游戏程序不受影响。

### 2. 现实对应
现实里所有接入 AI 的应用后台都在做类似的事：接收业务请求 → 组装 prompt → 调 AI API → 容错解析 → 返回结构化结果。区别只是通常被打包进了 SDK、框架或平台的 LLM proxy 里，代码不可见。

这次任务手动实现了这个流程的每一步，等以后用 LangChain、Instructor 或类似框架时，会知道框架在做什么，遇到框架处理不了的场景也有能力自己处理。

## 十三、loads_object 的三级调用链
`loads_object` 是 `json_guard.py` 里的集成入口，不是 TODO 函数，但它把前三个函数串起来，理解它能看清三级容错的完整调用顺序。

```python
def loads_object(text: str) -> Dict[str, Any]:
    if not isinstance(text, str):
        raise JsonGuardError("输入不是字符串：%r" % type(text))

    # 第一级：剥围栏
    cleaned = strip_code_fence(text)

    # 尝试直接解析
    try:
        obj = json.loads(cleaned)
        if isinstance(obj, dict):
            return obj
    except (ValueError, TypeError):
        pass

    # 第二级：括号配对
    extracted = find_first_json_object(cleaned)
    if extracted is not None:
        try:
            obj = json.loads(extracted)
            if isinstance(obj, dict):
                return obj
        except (ValueError, TypeError):
            pass

    # 第三级：文本修复
    repaired = repair_json_text(extracted or cleaned)
    try:
        obj = json.loads(repaired)
        if isinstance(obj, dict):
            return obj
    except (ValueError, TypeError) as exc:
        raise JsonGuardError("无法从文本中提取 JSON 对象") from exc

    raise JsonGuardError("解析结果不是 JSON 对象（可能是数组或原始值）")
```

**三级失败降级**：每一级先尝试直接 `json.loads`，失败了才进入下一级，不会在已经能解析成功的情况下多做操作。这样覆盖了 mock 的所有 12 种场景：

+ 场景 1（围栏 + 尾随逗号）：围栏剥完后直接 `json.loads` 仍会失败（尾随逗号），进入第三级修复
+ 场景 2（前后夹带自然语言）：第一级不管自然语言，第二级括号配对提取出 JSON 部分
+ 场景 7（单引号）：第三级 `_normalize_strings` 处理
+ 场景 11（缺右括号）：第三级 `_close_unfinished` 补全

场景 3-6、8-9（语义问题，JSON 格式合法）：`loads_object` 能成功解析，返回 dict，然后 `validate_decision` 处理字段级的问题。

## 十四、config.py 和 prompts.py 的设计
### 1. GatewayConfig 的关键字段
`config.py` 里的 `GatewayConfig` 是所有可调参数的集中地：

```python
@dataclass
class GatewayConfig:
    model: str = "deepseek-flash"
    api_base_url: str = "https://api.deepseek.com"
    api_key: str = ""
    max_retries: int = 2
    temperature: float = 0.2
    max_tokens: int = 256
    connect_timeout: float = 5.0
    request_timeout: float = 30.0
    backoff_base: float = 1.0
    backoff_max: float = 60.0
    offline: bool = False
    enable_json_format: bool = True
    enable_thinking: bool = False
```

`resolved_api_key()` 方法：`api_key` 字段本身可以为空，`resolved_api_key()` 优先读环境变量 `DEEPSEEK_API_KEY`，fallback 到字段值。这样测试时可以从环境变量注入，代码里不会出现硬编码的 key。

`offline = True` 时 `app.py` 不创建 `LlmClient`，整个上游调用链路被绕过，完全靠本地规则出拳。这是开发和测试时的标准模式，不消耗 API 额度。

### 2. SYSTEM_PROMPT 的约束
`prompts.py` 里的 `SYSTEM_PROMPT` 有一个必须满足的约束：**必须包含"json"这个词**（大小写不限）。

这是 DeepSeek（以及 OpenAI）JSON 模式的要求：开了 `response_format: json_object` 之后，如果系统提示里没有提到 JSON，API 会返回 400 错误，提示"messages must contain the word 'json'"。

测试里有一个专门验证这个约束的用例：`test_prompt_contains_json_word_for_json_mode`，确保有人改了 `SYSTEM_PROMPT` 之后不会忘记这条规则。

## 参考资料
+ [DeepSeek API 文档](https://platform.deepseek.com/api-docs/)
+ [DeepSeek JSON Output 文档](https://platform.deepseek.com/api-docs/zh-cn/api/json_output)
+ [OpenAI Chat Completions 格式（DeepSeek 兼容）](https://platform.openai.com/docs/api-reference/chat)
+ skeleton 仓库 `README.md` 和各文件 docstring
+ `mock_llm_server.py --list` 查看所有坏回复场景

