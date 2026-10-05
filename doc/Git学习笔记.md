## 前言
参考资料中的命令按用途重新分类：记录修改、查看历史、管理分支、同步仓库，以及撤销和恢复，即**修改现在保存在哪里，这条命令会改变哪里，操作错了还能怎样恢复。**

环境为 Windows、`git version 2.55.0.windows.5`

在学习过程中，我发现[有用的小插件视频](https://www.bilibili.com/video/BV1Q6w3zrEvL/?buvid=XU85422896C0C74BE555FA229714074D8DB18&from_spmid=search.search-result.0.0&is_story_h5=false&mid=yTXG%2BleUJmFN6%2FeKR8CVsg%3D%3D&plat_id=116&share_from=ugc&share_medium=harmony&share_plat=harmony&share_session_id=86eecea2-4062-4131-ad92-caaa92e6221d&share_source=weixin&share_tag=s_i&spmid=united.player-video-detail.0.0&timestamp=1789824204&unique_k=iajhcyb&up_id=480804525)非常好，但廖雪峰和很多的人的教程过于不生动形象具体。于是我要求chatgpt辅助我学习，真实地生成在开发场景中会遇到的情景。

## 一、Git 解决什么问题
### 1. Git 是什么，用什么语言开发
Git 是**分布式版本控制系统**，核心主要使用 **C 语言**实现，项目中也包含 Shell、Perl 等代码。因此，“Git 用 C 开发”可以作为简答，但“Git 完全由 C 编写”不够准确。[T24]

版本控制主要帮助回答三个问题：一个文件为什么变成现在这样、某次修改由谁提交，以及需要时怎样回到之前的状态。[T26]

Git 与 GitHub 不是同一个东西：Git 是版本控制工具；GitHub 是提供仓库托管、代码审查和协作功能的平台。没有 GitHub，也能在本地使用 Git。[T1][T23]

### 2. 集中式与分布式
| 对比项 | 集中式版本控制，如 CVS、SVN | 分布式版本控制，如 Git |
| --- | --- | --- |
| 历史主要存在哪里 | 中央仓库 | 常规完整克隆在本地保存仓库历史 |
| 能否离线编辑 | 可以 | 可以 |
| 能否离线提交历史 | 通常需要连接中央仓库 | 可以提交到本地仓库 |
| 与别人交换修改 | 通过中央仓库 | 可以通过一个或多个远程仓库 |


所以，集中式版本控制并不是联网才能工作。离线编辑仍然可以进行，某些本地比较也可以；受限的是向中央仓库提交、取得未缓存历史等操作。[T26]

分布式项目通常也会设置一个共同的远程仓库。它在团队流程中是主要协作入口，但并不是 Git 在技术上唯一允许存在的仓库。本地能够离线提交，不代表本地提交已经被别人看到。[T26][T27]

### 3. 文本和二进制文件都能管理，但体验不同
Git 可以记录和恢复文本、图片、PDF、Word 等文件的版本。普通文本适合逐行比较和合并；二进制文件通常不能直接得到有意义的逐行差异。[T26][T22]

旧式 `.doc` 是二进制文档；`.docx` 是包含 XML 等内容的 ZIP 容器，也不是直接供 Git 按正文逐行比较的普通文本。借助格式转换或专用 diff 驱动，可以改善部分格式的比较体验。[T22]

**能保存版本，不等于能理解文件内容。** 代码和学习笔记采用纯文本或 Markdown，通常更容易看清每次改动。

## 二、检查环境，建立第一个仓库
### 1. 检查版本与提交身份
```bash
git --version
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --show-origin --get user.name
git config --show-origin --get user.email
```

`user.name` 和 `user.email` 写入提交记录，**不是登录 GitHub 的账号密码**。公开仓库中的提交邮箱可能被公开；填写前应确认是否使用平台提供的隐私邮箱。`--global` 设置当前操作系统用户的默认值；只想修改当前仓库时，去掉它。`--show-origin` 可以检查设置来自哪个配置文件。[T5][T28]

Windows 与 WSL 中的 Git 是不同环境，版本和配置未必相同。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791200809540-c308fe85-9ac6-47fc-9bb4-4e3502441cd9.png)

### 2. 新建仓库：`git init`
在尚不存在的目录中执行：

```bash
mkdir git-learning
cd git-learning
git init -b main
git status
```

普通仓库初始化后会出现 `.git`，用于保存对象、引用和配置等数据。`main` 是这里选择的分支名称，不是 Git 强制规定的名称；项目也可能使用 `master`。[T1][T16]

### 3. 获取已有仓库：`git clone`
```bash
git clone <仓库地址> git-learning
cd git-learning
git remote -v
git branch -a
```

`clone` 不是只下载一份当前源码。常规克隆还会获取历史、建立远程配置，并检出默认分支。它通常只建立对应的本地初始分支，其他分支通过远程跟踪引用表示；`--depth`、`--single-branch`、部分克隆等选项会改变获取范围。[T11]

`init`** 与 **`clone`** 是两种起点，不需要在已经克隆的仓库中再初始化一遍。**

## 三、掌握工作区、暂存区与本地仓库
### 1. 三大区域
| 区域 | 含义 | 我怎样理解 |
| --- | --- | --- |
| 工作区，working tree | 当前实际编辑的文件 | 正在修改的内容 |
| 暂存区，index / staging area | 下一次提交准备采用的文件状态 | 本次提交的候选快照 |
| 本地仓库，repository | 对象数据库、提交历史和引用等 | 已记录的版本及其关系 |


暂存区并不是与 `.git` 完全分离的第三个普通文件夹；上表是在区分逻辑职责。[T27]

```latex
工作区 --git add--> 暂存区 --git commit--> 本地仓库
本地仓库 --git push--> 远程仓库
```

`add`** 不等于提交，**`commit`** 不等于上传。** 远程仓库也不是本地“三大区域”里的暂存区。[T2][T5][T14]

### 2. 为什么 `add` 之后再修改，还要重新 `add`
本次练习先提交 `version 1`，再把文件改为 `version 2` 并执行 `add`，最后把工作区改为 `version 3`，但不再暂存。

```bash
git status --short
git diff -- notes.txt
git diff --staged -- notes.txt
git commit -m "Record staged version"
git show HEAD:notes.txt
```

实际结果是：

| 观察位置 | 内容 |
| --- | --- |
| 提交前的 HEAD | `version 1` |
| 提交前的暂存区 | `version 2` |
| 工作区 | `version 3` |
| 执行 commit 后的新提交 | `version 2` |


<!-- 这是一张图片，ocr 内容为： -->
![](images/01-staging.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791200790956-e2affeb2-b549-4307-94ba-e593a4a7cf65.png)

`git status --short` 中的 `MM notes.txt` 并不是重复报错。在普通、非冲突状态下，第一列表示暂存区相对 HEAD 的差异，第二列表示工作区相对暂存区的差异。[T3]

****`git add`** 记录的是执行当时的内容，并不会持续跟随编辑器里的修改。** 这也解释了为什么提交前要检查 `git diff --staged`。[T2][T5]

## 四、日常操作：状态、暂存与提交
### 1. 最常用的一组操作
编辑并保存文件后：

```bash
git status
git diff
git add README.md
git diff --staged
git diff --staged --check
git commit -m "docs: explain the Git learning workflow"
git status
```

`status` 查看状态，`diff` 查看尚未暂存的差异，`add` 选择本次要提交的内容。`--staged` 只看已暂存差异，`--check` 检查引入的空白错误和冲突标记等问题，但不能代替程序测试。[T2][T3][T4][T5]

我把提交前的检查分成两部分：**diff 检查“到底提交了什么”，测试检查“这些修改是否还能正常工作”。**

### 2. 几个选项的区别
| 命令 | 含义与边界 |
| --- | --- |
| `git add README.md` | 暂存指定路径的变化 |
| `git add .` | 暂存当前目录及其子目录中未被忽略的变化 |
| `git add -A` | 不指定路径时，暂存整个工作树的新增、修改和删除 |
| `git add -p` | 按差异块选择要暂存的内容 |
| `git commit -am "说明"` | 自动暂存已跟踪文件的修改和删除再提交，不会自动包含尚未跟踪的新文件 |
| `git commit --amend` | 重做最近一次提交；通常会生成新的提交 ID |


可以明确写出文件名，避免把配置、密钥或无关文件一起提交。`--amend` 适合整理尚未共享的提交；对别人已经基于其工作的提交，不应擅自改写。[T2][T5]<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791200903471-b849e93f-b6ef-47bf-9884-f101ad37843c.png)

### 3. 忽略、删除与重命名
`.gitignore` 可以写：

```plain
.env
*.log
node_modules/
dist/
```

它主要影响未跟踪文件，不会让已经跟踪的文件自动退出版本控制。[T21]

```bash
git rm obsolete.txt
git mv old-name.txt new-name.txt
git rm --cached .env
```

前两条分别安排删除和重命名；`git rm --cached .env` 保留本地文件，但把停止跟踪该路径的变化放入暂存区，需要再提交。[R3]

**停止跟踪不会清除历史中的密钥。** 如果敏感凭据曾被推送，应先使凭据失效并更换，再按项目流程处理历史和泄露影响。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791200971240-43bf219b-af86-4137-978e-4c87cdb7498a.png)

## 五、查看历史：`log`、`diff` 与 `HEAD`
### 1. 看提交和看差异不是同一件事
```bash
git log --oneline --graph --decorate --all
git log -p -- README.md
git log --follow -- README.md
git show <提交ID>
```

`--oneline` 压缩每次提交的显示，`--graph` 显示拓扑关系，`--decorate` 显示分支、标签等引用，`--all` 从所有引用展示可达历史。`--follow` 用于跟踪单个文件跨重命名的历史。[T29]<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201001823-de6fe79a-e0b9-493c-b20d-67bf06379f9f.png)

| 命令 | 主要比较内容 |
| --- | --- |
| `git diff` | 工作区与暂存区 |
| `git diff --staged` | 暂存区与 HEAD |
| `git diff HEAD` | 工作区中已跟踪内容与 HEAD |
| `git diff A B` | 两个提交的文件快照 |
| `git diff origin/main...HEAD` | 共同祖先到当前 HEAD 的变化，常用于自查功能分支 |
| `git log origin/main..HEAD` | HEAD 可达、但 origin/main 不可达的提交 |


`diff` 默认不会把尚未跟踪文件的正文显示出来。因此，“`git diff` 没输出”不等于“目录里没有新文件”，还要看 `status`。[T3][T4][T10]<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791200844046-1b6f413b-5185-4adf-92ff-407bf46d1f18.png)

### 2. `HEAD` 和提交 ID
正常在分支上工作时，`HEAD` 通常指向当前分支，分支再指向某个提交；分离头指针状态下，`HEAD` 直接指向提交。它不是整个项目所有分支中“时间最新”的提交。[T10][T17]

| 写法 | 含义 |
| --- | --- |
| `HEAD` | 当前检出的提交 |
| `HEAD^` 或 `HEAD~1` | 当前提交的第一个父提交 |
| `HEAD~2` | 沿第一父提交链向前两代 |
| `HEAD~100` | 沿第一父提交链向前一百代，前提是存在 |
| `HEAD^2` | 合并提交的第二个父提交，不是“前两个版本” |
| `"HEAD@{1}"` | 本地 HEAD reflog 中的上一个位置，不一定是父提交 |


提交 ID 是提交对象内容的哈希标识，不是随机生成的流水号。提交包含文件树、父提交、作者、提交者、时间和说明等信息；即便文件内容相同，父提交或元数据不同，也可能有不同 ID。[T5][T10][T27]

短 ID 必须能在当前仓库中唯一识别对象。应使用 Git 显示的缩写，遇到歧义就增加长度。[T10]<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201023327-0b547932-e318-4a0a-8c4c-c3d3dc70de83.png)

## 六、分支：创建、切换与合并
### 1. 创建和切换
```bash
git branch
git branch -a
git branch -vv
git branch feature/notes
git checkout feature/notes
```

`git branch feature/notes` 只创建分支，不会自动切换。创建并切换可以用：

```bash
git checkout -b feature/notes
```

上面两种创建方式二选一，不要给已存在的分支重复创建。较新 Git 也可以用职责更明确的 `git switch feature/notes` 和 `git switch -c feature/notes`。[T16][T17][R3]

**分支不是重新复制一整套项目，而是指向某个提交的可移动引用。** 切换分支会让工作区匹配相应版本，但未提交改动并不天然“属于”某个分支；不冲突时可能跟随切换，可能被覆盖时 Git 通常会阻止。[T16][T17]<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201070560-79440650-f882-4d1b-afc6-8b003e2cf41f.png)

### 2. 把功能合并回主分支
功能分支完成提交和测试后：

```bash
git checkout main
git diff main...feature/notes
git merge feature/notes
git log --oneline --graph --decorate -8
```

**合并方向由当前分支决定。** 上面的含义是“把 `feature/notes` 合入当前的 `main`”，不是反过来。[T18]

如果当前分支是目标分支的祖先，可能只需快进，移动分支引用，不产生新的合并提交；两个分支已经分叉时，常规合并会生成合并提交。[T18]

确认功能已经保留在目标历史中、也不再需要该分支后：

```bash
git branch -d feature/notes
```

`-d` 带有合并状态检查；`-D` 强制删除会绕过相应检查。备份分支和标签可以保留已有提交，但不能保存尚未提交的工作区内容。[T16]<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201088418-22ec9787-c86c-49f0-81d9-660731ef3ecb.png)

## 七、合并冲突：先理解内容，再决定保留什么
### 1. 复现同一行冲突
本次练习的共同起点是 `config.txt` 中的一行：

```latex
color=blue
```

在 `feature/color` 上把它改为 `color=green` 并提交；回到 `main`，把同一行改为 `color=red` 并提交。随后在 `main` 上执行：

```bash
git merge feature/color
git status
git diff
```

实际产生的冲突内容为：

```latex
<<<<<<< HEAD
color=red
=======
color=green
>>>>>>> feature/color
```

在这次普通 merge 中，上半部分来自当前分支，下半部分来自待合入分支。冲突表示 Git 不能自动决定最终内容，不表示仓库损坏。[T18]<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201230299-3b2868b7-593f-4386-8194-6822debe8a3e.png)

### 2. 解决与检查
本次练习将最终内容确定为 `color=purple`。修改文件、删除全部冲突标记后：

```bash
git add config.txt
git diff --staged
git diff --staged --check
git commit -m "Resolve color conflict"
git status
```

<!-- 这是一张图片，ocr 内容为： -->
![](images/03-conflict.png)

解决冲突要同时检查两边修改的目的，必要时重写为第三种结果；`git add` 只是标记结果已准备好，并不证明业务逻辑正确。

如果决定暂时取消这次合并：

```bash
git merge --abort
```

`**git merge --abort**`** 是“取消本次合并”，不是“保证所有文件精确恢复原样”的备份机制。** 合并前已有未提交修改，尤其合并开始后又继续修改时，Git 可能无法完整重建合并前的状态。

因此，合并前先：

+ **提交修改**：用 `git commit` 保存版本。
+ **暂存未完成工作**：这里指 `git stash`，不是仅执行 `git add`。[T18]<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201167128-93e80b03-cdf8-4de7-b705-10c27fe80bb0.png)

## 八、远程仓库：`remote`、`fetch`、`pull` 与 `push`
### 1. 分清三个名称
| 名称 | 含义 |
| --- | --- |
| `main` | 本地分支 |
| `origin` | 远程仓库配置的常用别名 |
| `origin/main` | 本地保存的远程跟踪引用，不是实时读取服务器 |


`origin` 没有特殊权限，只是常规克隆默认采用的名字；`origin/main` 反映本地最近获知的远程状态，并不保证此刻仍然最新。[T11][T12][T15]

```bash
git remote -v
git remote show origin
git remote add origin <仓库地址>
git remote set-url origin <新的仓库地址>
```

已有 `origin` 时不必再次 `add`。`git remote remove origin` 只删除本地远程配置和相关远程跟踪引用，不会删除服务器仓库。[T15]<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201350600-7244bbee-dddb-4d98-86cf-4df182bcd60f.png)

### 2. `fetch` 与 `pull` 的区别
```bash
git fetch origin
git log --oneline --graph --decorate --all -10
git diff HEAD origin/main
```

在常规远程配置下，`fetch` 获取对象并更新远程跟踪引用，不把这些提交自动合入当前分支，也不直接更新工作区。[T12]

`pull` 则先获取，再按指定方式整合进**当前分支**：

| 命令 | 整合方式 |
| --- | --- |
| `git pull --ff-only origin main` | 只允许快进；分叉时停止 |
| `git pull --no-rebase origin main` | 采用 merge 方式整合 |
| `git pull --rebase origin main` | 采用 rebase 方式整合 |


这三条是不同选择，不是顺序执行的步骤。裸写 `git pull` 的具体行为受版本和配置影响，明确写出策略更容易判断结果。[T13]

**特别注意：在功能分支上执行 **`git pull origin main`**，不会替我切换到本地 **`main`**，而是把远程 **`main`** 整合到当前功能分支。**

<!-- 这是一张图片，ocr 内容为： -->
![](images/05-fetch.png)

本次用两个本地克隆模拟协作者。另一份克隆推送后，第一份执行 `fetch`，出现 `behind 1`，HEAD 和工作区未变化；再执行 `pull --ff-only`，才取得新文件。由此可见“已经拿到历史”和“当前工作区已经采用历史”的区别。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201528015-72ab88ee-ca65-4307-97d9-c9f8a8b40bd7.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201561449-3e581b35-c0b3-40d2-8d4d-95145f943d1e.png)

### 3. 推送：`push`
```bash
git push -u origin feature/notes
```

这里把本地 `feature/notes` 推送到同名远程分支，并为成功推送的分支设置上游关系。之后在匹配的配置下，可以简写为 `git push`。推送传播的是提交及相关对象，不会上传工作区中尚未提交的编辑。[T14]

遇到 `non-fast-forward` 拒绝时，先查看远程新增了什么，再按团队规则合并或变基。不要因为“推不上去”就直接使用 `--force`。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201588731-5d35b4f3-7e0c-4a42-bee0-ecb945817c10.png)

### 4. 删除服务器分支与删除本地记录不同
```bash
git branch -dr origin/feature/notes
```

这只删除本地远程跟踪引用；服务器分支若仍存在，后续 fetch 可能重新建立它。[T16]

真正删除服务器上的分支，使用：

```bash
git push origin --delete feature/notes
```

只有确认分支无需保留、且拥有相应权限时才执行。之后可以用 `git fetch --prune origin` 清理已经过期的本地远程跟踪引用。[T12][T14]<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201632544-c5f7aca0-62e7-46f5-a6a2-ae762cf5a4a9.png)

## 九、撤销操作：先判断修改所处的位置
### 1. 先选目标
| 当前需求 | 常见操作 | 主要影响 |
| --- | --- | --- |
| 已暂存，但本次不想提交 | `git restore --staged -- file.txt` | 取消暂存，保留工作区内容 |
| 丢弃某文件未暂存的修改 | `git restore -- file.txt` | 工作区恢复为暂存区版本，有丢失风险 |
| 撤掉本地最近提交，保留为已暂存 | `git reset --soft HEAD~1` | 移动当前分支，不改暂存区和工作区 |
| 撤掉本地最近提交，重新选择内容 | `git reset --mixed HEAD~1` | 移动分支并重置暂存区，保留工作区 |
| 抵消已经共享的错误提交 | `git revert <提交ID>` | 增加一个反向提交 |
| 暂时放下未完成的修改 | `git stash push -u -m "说明"` | 临时保存工作区和暂存区状态 |


这些命令不可互换。“撤销”可能是撤销暂存、丢弃文件编辑、移动分支，或者增加反向提交；不先说清楚目标，就很容易使用过度破坏性的操作。[T6][T7][T8][T19]

### 2. `restore` 与按路径的 `reset`
**取消暂存，不等于丢弃修改。** `--staged` 操作暂存区；不带它时，`restore` 默认操作工作区。

假设 `README.md` 经历了这些操作：

```latex
提交了内容 A                → HEAD 中是 A
改成 B，执行 git add        → 暂存区是 B
又改成 C，尚未执行 git add  → 工作区是 C
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201698385-c1a09b72-aca8-43f9-8b58-50ea8e413357.png)下面三种操作**分别从这个状态开始，不是连续执行**。

**1. 取消暂存，保留文件内容**

```bash
git restore --staged -- README.md
```

结果：`HEAD=A，暂存区=A，工作区=C`。

文件仍然是 C，只是不再有待提交的暂存修改。`git reset HEAD -- README.md` 效果相同，**不会移动 HEAD，也不会撤销提交**。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201713121-2d1dd665-a95d-402a-81cf-490dca190a7e.png)

**2. 丢弃未暂存修改**

```bash
git restore -- README.md
```

结果：`HEAD=A，暂存区=B，工作区=B`。

文件从 C 恢复成 B，**不是 A**，因为默认从暂存区取内容。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201724079-6e65a996-f3c5-4682-894e-9a86505f528c.png)

**3. 丢弃该文件全部未提交修改**

```bash
git restore --source=HEAD --staged --worktree -- README.md
```

结果：`HEAD=A，暂存区=A，工作区=A`。

用已提交的 A 同时覆盖暂存区和工作区，B、C 的修改都不再保留在这两个位置。**确认不要这些修改时才能执行。**<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201737938-0836e380-2635-4b90-a783-2a6c256f4965.png)

至于 `git reset --mixed HEAD~1`，它会让当前分支退到上一个提交，并重置整个暂存区，但保留工作区内容。[T6][T7]

### 3. `reset --soft`、`--mixed`、`--hard`
对 `git reset <模式> <目标提交>`，在正常检出分支的情况下：

| 模式 | 当前分支与 HEAD | 暂存区 | 工作区 |
| --- | --- | --- | --- |
| `--soft` | 移到目标提交 | 保持不变 | 保持不变 |
| `--mixed`，默认模式 | 移到目标提交 | 重置为目标提交 | 保持不变 |
| `--hard` | 移到目标提交 | 重置为目标提交 | 已跟踪内容重置为目标提交 |


本次在三个独立仓库里，从干净的第二次提交回退到第一次提交，得到：[T6]

<!-- 这是一张图片，ocr 内容为： -->
![](images/02-reset.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201778593-1d3d7398-0c82-4746-aa23-1851b0b5d1e8.png)

**soft 只移动引用，mixed 再重置暂存区，hard 进一步覆盖工作区。** 

大改或试验前可以先保留提交位置：

```bash
git status
git branch backup/before-change
```

但备份分支只保留已有提交。未提交内容还需要独立备份、提交到临时分支，或使用 stash。

`git reset --hard`** 会丢弃已跟踪文件的相关未提交修改，也可能覆盖或删除阻碍目标版本写入的未跟踪路径。它既不是安全的默认撤销方式，也不等于清理所有未跟踪文件。**[T6][T9]

### 4. `revert`：保留历史，增加修正
```bash
git show <错误提交ID>
git revert <错误提交ID>
```

`revert` 根据该提交引入的变化，创建一个反向提交。原来的提交仍保留在历史中，因此更适合已共享分支上的常规纠错。[T8]<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201815820-8f7f1f41-792d-4cce-8df4-7d00c809a176.png)

它不是简单地把整个项目复制成某个旧版本，也可能与后续修改冲突。解决冲突后，暂存结果，再执行 `git revert --continue`；取消本次操作可用 `git revert --abort`。[T8]

## 十、临时保存：`stash` 不是长期备份
工作未完成，但需要暂时处理别的任务时：

```bash
git status
git stash push -u -m "WIP: Git notes"
git stash list
git checkout main
```

完成其他任务后，先回到原功能分支，确认工作区状态，再恢复：

```bash
git checkout feature/notes
git stash show -p "stash@{0}"
git stash apply "stash@{0}"
git status
```

核对文件和测试结果后，再决定是否删除这份记录：

```bash
git stash drop "stash@{0}"
```

默认 stash 不包含尚未跟踪的新文件；`-u` 会包含未跟踪文件，但不包含被忽略文件；`-a` 还会包含忽略文件，因此可能连构建产物或敏感文件一起收进去。[T19]

`apply` 恢复但保留记录，`pop` 在成功恢复后删除相应记录；`pop` 发生冲突时通常保留原 stash。需要同时尝试恢复原暂存状态时，可以了解 `apply --index`，但它也可能失败。[T19]

**先 **`apply`**、检查，再 **`drop`**。** 普通分支推送不会自动备份 stash，因此它不应长期充当唯一的工作记录。PowerShell 中把 `"stash@{0}"` 加引号，也能避免花括号被 shell 误解析。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201831792-d8decf8a-6b2d-4b39-9882-8566a8e8e18c.png)

## 十一、Rebase：整理提交的基点，不是普通复制
### 1. 与 merge 的区别
假设历史是：

```latex
A---B---C  main
     \
      D---E  feature
```

在 `feature` 上执行 `git rebase main`，通常会得到：

```latex
A---B---C---D'---E'  feature
        |
       main
```

Rebase 把需要保留的提交变更重放到新的基点上。被重写提交的父提交等内容改变，因此 `D'`、`E'` 不再是原来的提交 ID；如果本来无需重放，则不应理解为每次 rebase 必然改写全部提交。[T20]

merge 注重保留已有分叉关系，rebase 常用于整理个人功能分支的线性历史。二者没有脱离团队工作流的绝对优劣。

### 2. 一个明确的工作流
先提交或妥善保存工作区修改，再在自己的功能分支上执行：

```bash
git checkout feature/notes
git fetch origin
git branch backup/before-rebase
git rebase origin/main
```

发生冲突时，处理文件并检查结果，然后：

```bash
git add <已解决的文件>
git rebase --continue
```

取消整次变基可用 `git rebase --abort`。`--skip` 会跳过当前待重放提交，不是“自动解决冲突”，不能为了继续运行随手使用。[T20]<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201862657-b69454d5-21b3-4e0f-a07a-c2c5c5a65e58.png)

### 3. 已推送分支的边界
不要擅自 rebase 大家共同使用的主分支。如果个人功能分支已经推送，且团队明确允许重写它，更新远程可能需要：

```bash
git push --force-with-lease origin feature/notes
```

`--force-with-lease` 会检查远程引用是否符合预期，比无条件强推多一道保护，但不是“不可能覆盖别人工作”的保证。后台 fetch 可能影响基于远程跟踪引用的预期值；多人使用同一分支时仍应先沟通并遵守保护规则。[T14]<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201902351-bd741e6f-48aa-4cf4-8f13-1b6f62480726.png)

交互式变基 `git rebase -i HEAD~3` 可用于重排或合并最近的本地提交。先掌握普通 rebase 的继续、取消和恢复，再扩展到 `pick`、`reword`、`squash` 等操作。[T20]

## 十二、Pull Request：把修改提交给别人审查
### 1. PR 不等于 `git pull`
**Pull Request 是托管平台上的合并与审查请求；**`git pull`** 是本地 Git 命令。** 把代码推送到分支，也不等于已经创建 PR。[T13][T23]

有仓库写权限时，一般在项目中建立功能分支；没有写权限时，通常先 fork，再把自己的分支作为来源。Fork 是托管平台上的仓库副本，不是本地 clone。[T23]

### 2. 从分支到 PR
在自己的协作权限范围内：

```bash
git checkout main
git pull --ff-only origin main
git checkout -b feature/notes
```

修改完成后先跑项目规定的测试，再检查和提交：

```bash
git status
git diff
git add README.md
git diff --staged
git diff --staged --check
git commit -m "docs: add Git learning notes"
git fetch origin
git log --oneline origin/main..HEAD
git diff origin/main...HEAD
git push -u origin feature/notes
```

到 GitHub 创建 PR 时，重点核对：[T23]

1. **base** 是希望合入的目标仓库与目标分支，例如项目的 `main`。
2. **compare / head** 是自己的功能分支，例如 `feature/notes`。
3. 标题说清修改主题，说明中写出修改原因、主要变化、测试结果和风险。
4. 自己先看一遍 Files changed，排除无关文件、密钥和误改。
5. 等待自动检查与审查；后续修改提交并推送到同一个来源分支，通常会更新现有 PR。
6. 满足权限和项目要求后，由有权限的人合并；之后同步本地目标分支。

Fork 模式中通常把个人仓库命名为 `origin`，原项目命名为 `upstream`。这时应获取 `upstream/main`、以它为目标检查差异，再推送到自己的 `origin`，不能机械照抄上面同仓库协作的所有远程名。[T15][T23]

**PR 的价值不只是“申请合并”，还在于让别人能判断为什么改、改了什么、验证到什么程度。** 

## 十三、使用 reflog 恢复“丢失”的提交
### 1. `log` 看历史，`reflog` 看本地引用如何移动
当分支被 reset 到旧位置时，原提交可能不再出现在当前分支的普通 `git log` 中，但对象仍可能存在；本地 reflog 可能记录了之前的位置。[T9]

本次在练习仓库中创建两次提交，再执行一次故意的回退。随后：

```bash
git reflog --oneline
git show <找到的提交ID>
git branch rescue/notes <找到的提交ID>
git checkout rescue/notes
```

<!-- 这是一张图片，ocr 内容为： -->
![](images/04-recovery.png)

这样先建立恢复分支，能查看并保留找回的提交，不必立即移动原分支。确定内容正确后，再决定合并、拣选提交，还是调整原分支。

### 2. 恢复的限制
reflog 主要是当前本地仓库的记录，不会随着常规 clone、fetch、push 自动作为共享历史传播。它会过期，不可达对象也可能被回收；常见默认配置中，普通记录与不可达记录的过期阈值分别是 90 天和 30 天，但这不是恢复保证期。[T9]

**从未提交过的文件内容，不能指望靠 reflog 恢复。** 提交曾存在、能够找到它、对象仍保留，这些条件不能省略。发现误删后，应尽早检查，避免主动清理对象，并先建立引用保住找到的提交。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201942649-a02e8870-45a2-4c7c-95b6-b485487964b3.png)

## 十四、问题分析与参考资料纠错
### 1. 实际遇到的问题：没改正文，为什么 pull 说会覆盖修改
**现象：** 远程同步练习第一次执行 `pull --ff-only` 时被拒绝，提示本地 `README.md` 的修改会被覆盖。文件正文看起来仍然是同一个单词。

**检查：**

```bash
git config --show-origin --get-all core.autocrlf
git ls-files --eol
git diff -- README.md
```

当时系统配置的 `core.autocrlf=true`。克隆检出时，文件写成了 CRLF；练习随后把仓库配置改成 `false`。检查显示：

```latex
i/lf    w/crlf
```

即索引内容使用 LF，工作区文件使用 CRLF。配置发生变化后，Git 对这份工作区文件的规范化处理不同，于是产生了“正文一样、字节不同”的差异。[T22][T28]

<!-- 这是一张图片，ocr 内容为： -->
![](images/07-line-endings.png)<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791201995050-47dcd558-ab48-4ef3-adce-adbec75b42ff.png)

**本次处理：** 没有覆盖真实项目文件，也没有全局关闭换行转换；在全新的练习克隆中，检出前就明确使用同一配置：

```bash
git clone --config core.autocrlf=false <本地练习仓库地址> <新的练习目录>
```

重新验证后，同步流程通过。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791202019171-41db7817-8039-4e9b-8484-8871fb410073.png)

**我的总结：** 遇到“文件没改却出现 diff”，要检查编码、换行符、文件权限和配置来源。真实项目应遵循已有 `.gitattributes`；需要统一规则时，先保存修改、审查差异，再把规范化作为单独变更处理。[T22][T28]

### 2. 其他常见问题
| 现象 | 先判断什么 | 处理思路 |
| --- | --- | --- |
| `not a git repository` | 当前目录是否正确 | 进入正确的仓库，不要直接在未知目录执行 init |
| `Author identity unknown` | 当前环境是否设置提交身份 | 核对 user.name、user.email 及其来源 |
| `remote origin already exists` | 是否已经克隆或配置过远程 | 先看 remote -v，需要时 set-url |
| 分支没有上游 | 推送目标与分支名称 | 确认后用 push -u 建立关系 |
| `non-fast-forward` | 是否存在远程新增提交或历史重写 | fetch 并检查图，再选择 merge 或 rebase |
| `Your local changes ... would be overwritten` | 哪些修改尚未保存，是否为换行差异 | 先检查并保留内容，不要直接 hard reset |
| 切到旧提交后处于 detached HEAD | 是否准备在这里继续开发 | 需要保留新工作时创建分支 |


这些是排查入口，不是保证有效的万能修复。处理后还要重新运行触发问题的操作，确认原因与结果相符。[T1][T5][T11][T14][T15][T17][T28]

### 3. 对原资料中说法的修正
| 原说法或写法 | 更准确的理解 |
| --- | --- |
| 记事本保存 UTF-8 带 BOM，不能用 | 这是过时的说法。微软已于 2018 年 12 月 10 日公布无 BOM UTF-8 支持及新文件默认值调整。[T25] |


UTF-8 BOM 的字节是 `EF BB BF`。是否允许 BOM，应根据文件格式和工具要求决定。选择 VS Code 便于看清编码、换行符和差异，但也应检查实际保存设置。[T25]

## 参考资料
### 原始学习资料
+ **[R1]** [Dainy Jose：Git Complete Commands Cheat Sheet for Developers](https://dev.to/dainyjose/git-complete-commands-cheat-sheet-for-developers-96j)。日常命令分类；其中 pull 与 PR 的表述已纠正。
+ **[R2]** [freeCodeCamp：Git Cheat Sheet – Helpful Git Commands with Examples](https://www.freecodecamp.org/news/git-cheat-sheet-helpful-git-commands-with-examples/)。用于扩展主题，不把所有高级命令纳入本次必学范围。
+ **[R3]** [Git 官方 Cheat Sheet](https://git-scm.com/cheat-sheet)。核心命令与操作入口。
+ **[R4]** [廖雪峰：Git 教程](https://liaoxuefeng.com/books/git/introduction/index.html)。中文学习顺序与概念引导。

### 整理时核对的官方资料
+ **[T1]** [git 总览](https://git-scm.com/docs/git)。
+ **[T2]** [git-add](https://git-scm.com/docs/git-add)。暂存范围与按块选择。
+ **[T3]** [git-status](https://git-scm.com/docs/git-status)。状态与短格式列含义。
+ **[T4]** [git-diff](https://git-scm.com/docs/git-diff)。比较对象与检查选项。
+ **[T5]** [git-commit](https://git-scm.com/docs/git-commit)。提交、身份、`-a` 与 amend。
+ **[T6]** [git-reset](https://git-scm.com/docs/git-reset)。模式、路径形式及风险。
+ **[T7]** [git-restore](https://git-scm.com/docs/git-restore)。恢复来源和目标。
+ **[T8]** [git-revert](https://git-scm.com/docs/git-revert)。反向提交及冲突处理。
+ **[T9]** [git-reflog](https://git-scm.com/docs/git-reflog)。引用记录及过期机制。
+ **[T10]** [gitrevisions](https://git-scm.com/docs/gitrevisions)。HEAD、父提交、缩写与范围。
+ **[T11]** [git-clone](https://git-scm.com/docs/git-clone)。克隆范围及配置。
+ **[T12]** [git-fetch](https://git-scm.com/docs/git-fetch)。获取对象与更新引用。
+ **[T13]** [git-pull](https://git-scm.com/docs/git-pull)。获取后的整合策略。
+ **[T14]** [git-push](https://git-scm.com/docs/git-push)。推送、删除远程分支和强推保护。
+ **[T15]** [git-remote](https://git-scm.com/docs/git-remote)。远程仓库配置。
+ **[T16]** [git-branch](https://git-scm.com/docs/git-branch)。创建与删除本地和远程跟踪分支。
+ **[T17]** [git-checkout](https://git-scm.com/docs/git-checkout)。分支切换、路径恢复及分离 HEAD。
+ **[T18]** [git-merge](https://git-scm.com/docs/git-merge)。合并、冲突和取消操作。
+ **[T19]** [git-stash](https://git-scm.com/docs/git-stash)。临时保存范围与恢复行为。
+ **[T20]** [git-rebase](https://git-scm.com/docs/git-rebase)。重放、继续、跳过与取消。
+ **[T21]** [gitignore](https://git-scm.com/docs/gitignore)。忽略规则及已跟踪文件的边界。
+ **[T22]** [gitattributes](https://git-scm.com/docs/gitattributes)。换行规范化、二进制差异与转换驱动。
+ **[T23]** [GitHub：Creating a pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)。PR 创建与审查流程。
+ **[T24]** [Git 官方源码镜像](https://github.com/git/git)。实现语言与项目说明。
+ **[T25]** [Microsoft：Windows 10 Insider Preview Build 18298](https://blogs.windows.com/windows-insider/2018/12/10/announcing-windows-10-insider-preview-build-18298/)。2018 年公布的记事本 UTF-8 无 BOM 支持。
+ **[T26]** [Pro Git：About Version Control](https://git-scm.com/book/en/v2/Getting-Started-About-Version-Control)。版本控制与集中式／分布式。
+ **[T27]** [Pro Git：What is Git?](https://git-scm.com/book/en/v2/Getting-Started-What-is-Git%3F)。快照、本地操作与三大区域。
+ **[T28]** [git-config](https://git-scm.com/docs/git-config)。配置作用域、来源与换行设置。
+ **[T29]** [git-log](https://git-scm.com/docs/git-log)。历史查看与文件追踪。

