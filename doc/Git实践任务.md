## 前言
这次我在 Windows 上使用 Git Bash、PowerShell 和 VS Code，以 `git_training` 中的 C++ 计算器为练习对象，学习 Git 配置、分支管理、暂存、合并与变基，最后将代码推送到 Gitee。

过程中，我遇到了命令粘贴异常、配置作用域错误、把分支名当成远程名称、变基冲突和 API 认证失败等问题。最让我困惑的是：**master 里是加减，功能分支里是乘除，为什么 rebase 还能直接成功？** 后来通过回退和对照实验，我逐渐理解了 Git 如何根据提交历史处理修改。

本文结合 2026 年 10 月 1 日的截图和后续补充的终端记录整理，按实践过程解释操作，保留必要的错误输入、问题分析和个人理解。

## 一、生成 SSH 密钥并测试连接
### 1. 生成密钥
我在 Git Bash 中执行：

```bash
ssh-keygen -t rsa -C "<用于标识的邮箱>"
cat ~/.ssh/id_rsa.pub
```

`-t rsa` 指定密钥类型，`-C` 设置注释。这里的邮箱用于识别密钥，不等于 Git 提交时使用的邮箱，也不会因此自动登录对应账户。[T1]

本次输出显示生成了 RSA 3072 位密钥，默认保存位置为：

```latex
~/.ssh/id_rsa       私钥
~/.ssh/id_rsa.pub   公钥
```

**添加到代码托管平台的是公钥，不能上传私钥。** 如果默认位置已有密钥，重新生成前需要确认是否会覆盖现有文件。[T1]

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791198453880-48d50ab6-0f00-4b5f-b9f2-42ca7e56fdac.png)

### 2. 排查第一次连接命令的报错
第一次粘贴连接命令后，终端提示：

```latex
bash: $'\302\203': command not found
```

截图中，`ssh` 前面存在异常字符。这是因为使用Ctrl+xxx的快捷键在git bash无对应，便直接作为输入。

清除异常字符后，我重新输入：

```bash
ssh -T git@gitee.com
```

这次进入了主机身份确认流程。首次连接应先核对平台公布的主机密钥指纹，再决定是否接受。[T2]

后续输出显示认证成功，同时说明平台不提供 Shell 访问。指的是

+ **可以**：在本地通过 SSH 拉取、推送代码，例如 `git pull`、`git push`，前提是你有对应仓库权限。
+ **不可以**：通过这个 SSH 入口进入 GitHub 服务器，执行 `ls`、安装软件等。这种远程命令行操作就是这里说的 **Shell 访问**。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791198552019-bab5dc17-a219-4d15-9354-9b4546b92a69.png)

## 二、配置提交身份并初始化仓库
### 1. 全局配置与仓库配置
我先设置全局用户名和 Gitee 邮箱，后来又把全局邮箱改成 GitHub 邮箱。为了让当前练习仓库继续使用 Gitee 身份，又尝试设置本地配置：

```bash
git config --global user.name "Jun Zhu"
git config --global user.email "<Gitee 提供的 noreply 邮箱>"
git config --global user.email "<GitHub 提供的 noreply 邮箱>"
git config --local user.name "Jun Zhu"
git config --local user.email "<Gitee 提供的 noreply 邮箱>"
```

正文中的邮箱使用占位符，不是可直接使用的实际地址。这时终端报错：

```latex
fatal: --local can only be used inside a git repository
```

原因是当前目录还没有初始化为 Git 仓库。`--local` 写入当前仓库的配置，不能直接用于普通目录。[T3]

随后我执行：

```bash
git init
git config --local user.name "Jun Zhu"
git config --local user.email "<Gitee 提供的 noreply 邮箱>"
git config --show-origin --get user.email
```

最后一条命令显示邮箱来自 `.git/config`，说明当前仓库的本地配置已生效。

| 配置 | 作用 | 我的理解 |
| --- | --- | --- |
| SSH 密钥 | 远程连接认证 | 决定平台是否认可这次连接 |
| `user.name`、`user.email` | 记录提交作者身份 | 不会代替 SSH 认证 |
| `--global` | 当前系统用户的默认 Git 配置 | 多个仓库可以继承 |
| `--local` | 当前仓库的 Git 配置 | 同名设置通常覆盖全局设置 |


**能连接 Gitee，不代表提交邮箱一定是 Gitee 邮箱；修改提交邮箱，也不等于切换了 SSH 登录身份。** 这两套机制需要分别检查。[T3]

### 2. 创建第一次提交
刚初始化仓库时，我运行过 `git checkout`，终端提示：

```latex
You do not have the initial commit yet
fatal: You are on a branch yet to be born
```

初始化后，`HEAD` 可以指向预定的分支名称，但首次提交尚未产生，没有已有提交可供这次检出操作使用。完成第一次提交后，分支才有了实际指向的提交对象。[T4]

```powershell
git add .
git commit -m "Initial commit"
git branch
```

本次初始提交是 `b21792d`，包含 `.gitignore`、`README.md` 和 `calculator.cpp` 三个文件。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791198757610-2d64b00f-f685-4839-9e37-ec2aface8d88.png)

### 3. 区分换行符警告与提交失败
执行 `git add .` 时，我看到了：

```latex
LF will be replaced by CRLF the next time Git touches it
```

这与换行符转换配置有关，不是合并冲突。本次提交随后成功。[T5]

## 三、创建分支与暂时保存修改
### 1. 分支并不是另一份项目目录
首次提交后，我先创建过 `features` 分支：

```powershell
git branch features
git checkout features
```

后续又重命名并创建了两个功能分支：

```latex
feature/add-subtract
feature/multiply-divide
```

乘除分支的创建过程：

```powershell
git checkout -b feature/multiply-divide
```

`git branch features` 只创建分支，`git checkout features` 才切换；`git checkout -b` 把两步合在一起。分支本质上是指向提交的引用，并不是在磁盘上另建一份项目目录。[T4][T6]

早期截图中，两个功能分支和 `master` 都指向 `b21792d`。虽然名称不同，当时它们的已提交历史仍停留在同一位置。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791198863817-3dfd5ea7-cf60-4da9-b33a-a467378b0223.png)

### 2. 使用 stash
我在乘除分支上暂时保存了未提交修改，再切到加减分支恢复：

```powershell
git stash
git checkout feature/add-subtract
git stash apply
```

恢复后，`calculator.cpp` 显示为未暂存修改。这里要区分三种操作：[T7]

| 命令 | 恢复修改 | 是否删除 stash 记录 |
| --- | --- | --- |
| `git stash apply` | 是 | 不删除 |
| `git stash pop` | 是 | 成功应用后删除；发生冲突时保留 |
| `git stash drop` | 否 | 删除指定记录，省略时默认处理最新一条 |


后面我又在 `master` 上执行了 `git stash pop`。

**stash 的来源分支会出现在描述里，但它不是只能回到来源分支使用。** 跨分支应用仍可能发生冲突，需要检查结果。默认 `git stash` 不包含未跟踪文件；需要一起保存时可以使用 `git stash push -u`，但被忽略的文件仍不在 `-u` 的范围内。[T7]

## 四、整理基础版本、快进合并与标签
### 1. 把本地分支误当成远程仓库
恢复修改后，我在 `master` 上创建基础提交：

```powershell
git add .
git commit -m "basic"
git checkout feature/add-subtract
git fetch master
```

提交为 `2d13ae4`，`calculator.cpp` 新增 1 行、删除 33 行。但最后一条命令失败：

```latex
fatal: 'master' does not appear to be a git repository
```

当时我把分支名和远程仓库名混淆了。`git fetch` 的这个参数位置接收仓库来源，因此 Git 尝试把 `master` 当成远程名称或仓库地址，而不是“从本地 master 同步”。[T8]

我的目标是合并本地分支，所以改为：

```powershell
git merge master
```

输出：

```latex
Updating b21792d..2d13ae4
Fast-forward
```

<!-- 这是一张图片，ocr 内容为： -->
![](assets/05-fetch-error-and-fast-forward.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791198918365-e3203393-5a5d-4b71-809a-819f428f49cd.png)

当前分支没有独有的新提交，`b21792d` 是 `2d13ae4` 的祖先，所以 Git 直接把分支指针向前移动，没有创建新的合并提交，这就是快进合并。[T9]

如果需要从远程获取内容，应先通过 `git remote -v` 确认远程名称。只有配置了相应远程，`git fetch origin` 这类命令才有对应目标。

### 2. 用标签标记基础版本
合并后，我创建了附注标签：

```powershell
git tag -a basics -m "basic tag , ready to develop"
git tag
```

分支用于继续推进开发，标签用于标记某个版本。后续提交会推进所在分支，但不会自动把标签一起向前移动。[T10]

此时创建的是本地标签，后续远程推送单独记录在第十二节。

## 五、提交加减功能并同步分支
### 1. 提交前检查暂存区
实现加减功能后，我查看了准备提交的差异：

```powershell
git add .
git diff --cached
```

记录中可以确认新增了：

```cpp
double add(double a, double b) { return a + b; }
double subtract(double a, double b) { return a - b; }
```

同时，主循环补充了输入处理、运算符分派和结果输出。

`git diff --cached` 比较暂存区与当前提交，适合确认“这次到底会记录哪些修改”。普通 `git diff` 则主要查看工作区相对暂存区的修改。[T11]

### 2. 未提交的修改不会自动按分支隔离
随后我切换过两次分支：

```powershell
git checkout master
git checkout feature/add-subtract
```

两次输出都带有：

```latex
M       calculator.cpp
```

这不表示修改已经分别提交到了两个分支，而是修改仍未提交，并被保留在工作区或暂存区中；**如果**切换会覆盖本地修改，Git 通常会拒绝。[T4] 这里切换丝滑是因为没有覆盖。

**分支名称不会自动隔离尚未提交的工作。** 需要分清工作区、暂存区和提交历史。

### 3. 为什么最初的 rebase 没有处理新内容
第一次运行时，我误用了不存在的分支名：

```powershell
git rebase main
```

报错为 `invalid upstream 'main'`。本次仓库的主分支叫 `master`，不能照搬其他教程中的 `main`。

改为 `git rebase master` 后，Git 又提示暂存区存在未提交修改，需要先提交或暂存。于是我执行：

```powershell
git commit -m "complete"
git rebase master
git rebase feature/multiply-divide
```

提交生成了 `be83af5`，两次 rebase 都提示当前分支已经是最新状态。

`master` 当时位于 `2d13ae4`，乘除分支还位于更早的 `b21792d`，二者都是加减分支的祖先。当前分支已建立在这些历史上，没有需要移到新基底上的分叉历史。[T12]

stash 中的修改也不会自动参与 rebase。

### 4. 把三个分支同步到同一提交
随后，我依次执行：

```powershell
git checkout master
git merge feature/add-subtract
git switch feature/multiply-divide
git merge feature/add-subtract
```

两次合并都是快进，三个分支此时都指向 `be83af5`：

```latex
b21792d  Initial commit
    |
2d13ae4  basic                  [basics]
    |
be83af5  complete               [master]
                                [feature/add-subtract]
                                [feature/multiply-divide]
```

我还执行过 `git stash drop` 和不带目标的 `git reset --hard`。前者删除 stash 记录，后者按当前 `HEAD` 重置暂存区和工作区。尤其是 `--hard`，可能丢弃未提交内容，不适合当作日常清理命令随手执行。[T7][T13]

## 六、cherry-pick 旧提交产生冲突
我在乘除分支上执行：

```powershell
git cherry-pick 2d13ae44bd243da0dffa5cfb58944048c2853252
```

结果是：

```latex
CONFLICT (content): Merge conflict in calculator.cpp
error: could not apply 2d13ae4... basic
```

`2d13ae4` 已经在当前历史中，但我又要求 Git 应用一次它引入的改动。**cherry-pick 不是把文件直接恢复成某次提交的完整快照，而是把那次提交相对其父提交的改动，应用到当前状态上。**[T14]

旧提交曾大量删除计算器代码，而当前版本又添加了加减逻辑。重新应用旧改动时，Git 无法自动处理部分内容，于是停下来要求解决冲突。

之后命令是：

```powershell
git add .
git cherry-pick --continue
```

新提交为 `92d5a9f`，提交信息通过终端编辑为：

```latex
basic

add for mul and div instead of add and sub
```



## 七、为什么加减和乘除不同，却能直接 rebase
### 1. 第一次变基的结果
我切到 `master`，创建了 `546b9af`，然后回到乘除分支：

```powershell
git checkout master
git add .
git commit -m "add new"
git switch feature/multiply-divide
git rebase master
```

输出：

```latex
Successfully rebased and updated refs/heads/feature/multiply-divide.
```

后续确认，第一次变基后生成的提交是 `bfe9f1f`。

### 2. 文件不同，不等于修改冲突
我当时疑惑：master 是加减，功能分支是乘除，为什么 Git 不问我保留哪一边？

关键在于：**Git 不是要求两个分支内容相同，而是尝试合并双方相对于基准的变化。** 如果只有一侧修改某段内容，另一侧仍保留基准内容，通常可以自动采用这一侧的修改。[T9]

针对讨论中的函数区域，可以这样理解：

| 位置 | 内容 | 相对基准的变化 |
| --- | --- | --- |
| 基准 `be83af5` | 加减 | 基准内容 |
| `master` | 仍为加减 | 该区域没有变化 |
| 乘除分支 | 改为乘除 | 修改了该区域 |


第一次重放“加减改为乘除”的修改时，没有遇到主分支对同一区域的不兼容修改，所以可以直接应用。

rebase 会把分支上的提交重放到新的基底，必要时生成新的提交对象。[T12] 如果提交表达的是“删除加减，换成乘除”，没有冲突也可能只留下乘除。**Git 完成了文本层面的整合，不代表业务上已经同时保留四则运算。**

### 3. 核查是否误提交冲突标记
我检查：

```powershell
git show --check 92d5a9f
```

输出只报告：

```latex
calculator.cpp:42: trailing whitespace.
+        }
```

上面的整理文本未保留不可见的行尾空格。

`--check` 可以检查差异中引入的冲突标记和空白错误。[T11]

## 八、撤销已经完成的 rebase
### 1. 不带目标的 reset 为什么没有回退
我先尝试：

```powershell
git reset --soft
git reset --hard
git reset --soft ^HEAD
```

前两条省略目标时默认使用当前 `HEAD`。`--soft` 不改变暂存区和工作区；`--hard` 将它们重置为目标提交，因此这里并没有把分支退回变基前的位置。[T13]

第三条报错，是因为 `^HEAD` 不是此处表示父提交的正确写法。父提交可写成 `HEAD~1` 或 `HEAD^`，但退到父提交也不等于恢复到变基前的原提交。

### 2. 先保留备份，再恢复旧提交
随后我执行：

```powershell
git stash push -u -m "before undo rebase"
git branch backup-after-rebase
git reset --hard 92d5a9f
```

stash 提示 `No local changes to save`，这一步没有创建新的 stash 记录。备份分支保留了变基后的 `bfe9f1f`，当前功能分支则回到原来的 `92d5a9f`。

**** `reset --hard` 会丢弃目标覆盖范围内的未提交内容；备份分支只保护已提交历史，不能保护未提交修改。`stash -u` 也不包含被忽略文件。[T7][T13]

| 所处阶段 | 处理思路 |
| --- | --- |
| rebase 尚未结束 | 可以用 `git rebase --abort` 放弃本次操作 |
| rebase 已经结束 | 保存修改和备份引用，再根据旧提交或 reflog 恢复 |


不知道旧提交编号时，可以先用 `git reflog` 查找记录。[T12][T13]

## 九、修改主分支，再做一次冲突实验
### 1. 对照实验触发冲突
回退后，我切到 `master`，故意修改计算器代码并提交：

```powershell
git switch master
git add .
git commit -m "deliberately destory"
git switch feature/multiply-divide
git rebase master
```

这里保留原提交信息中的拼写。主分支新增提交为 `97d255d`，这次出现：

```latex
CONFLICT (content): Merge conflict in calculator.cpp
error: could not apply 92d5a9f... basic
```

主分支新增修改与 `92d5a9f` 的重放发生了文本冲突。

| 实验 | 主分支基底 | 结果 |
| --- | --- | --- |
| 第一次 rebase | `546b9af` | 自动完成，得到 `bfe9f1f` |
| 第二次 rebase | 新增 `97d255d` | 重放 `92d5a9f` 时产生冲突 |


**冲突来自修改之间的不兼容。**

### 2. `continue` 少了两个短横线
标记文件后，我误输入：

```powershell
git add .
git rebase continue
```

Git 提示已存在 `rebase-merge` 目录。此时确实正在进行 rebase，问题是把控制选项 `--continue` 写成了普通参数 `continue`。

改为：

```powershell
git rebase --continue
```

操作生成 `478d2ca`，随后提示变基成功。[T12]

`git add` 标记的是“把当前内容作为解决结果”，不会检查 C++ 业务逻辑。

## 十、合并功能分支与备份
### 1. 变基后合并到 master
第二次变基后，我执行：

```powershell
git switch master
git merge feature/multiply-divide
```

结果：

```latex
Updating 97d255d..478d2ca
Fast-forward
```

功能分支刚建立在新的 `master` 上，因此主分支可以直接快进到 `478d2ca`，无需另建合并提交。[T9]

### 2. 合并备份分支再次冲突
随后，我回到功能分支，合并第一次变基后的备份：

```powershell
git switch feature/multiply-divide
git merge backup-after-rebase
```

这次再次发生 `calculator.cpp` 冲突。备份分支指向 `bfe9f1f`，不是“取消当前修改”的特殊按钮；合并它是在整合另一条历史。[T9]

我还执行了：

```powershell
git branch -d backup-after-rebase
```

这一段终端记录到这里结束。删除分支引用本身不等于解决文件冲突，也不等于结束正在进行的 merge。[T6][T9]

### 3. 后续状态
<!-- 这是一张图片，ocr 内容为： -->
![](assets/06-rebase-and-merge-history.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1791199592423-8cd3b3eb-99ee-4798-b92e-010a2e0c5105.png)

| 提交 | 截图中的信息 |
| --- | --- |
| `6243af48` | 合并 `backup-after-rebase` 的提交，功能分支指向这里 |
| `9daa8f9a` | 主分支上的 `.gitignore` 更新，提交标题提到 `.exe` 文件 |


## 十一、通过 Gitee API 创建远程仓库
### 1. 令牌认证失败与重新请求
本地练习之后，我在 Windows PowerShell 中输入个人令牌：

```powershell
$secret = Read-Host "输入 Gitee 令牌" -AsSecureString
$env:GITEE_TOKEN = [pscredential]::new("token", $secret).GetNetworkCredential().Password
Remove-Variable secret
```

`Read-Host -AsSecureString` 避免直接回显输入内容。随后通过 `GetNetworkCredential().Password` 取得字符串用于 API 请求；此时环境变量中的值已经不是 `SecureString`。[T16][T20]

第一次请求把 `private` 设为 `"false"`，返回：

```latex
401 Unauthorized: Access token does not exist
```

报错说明服务端没有识别本次传入的令牌。仅凭记录，无法进一步确定是空输入、复制错误、令牌失效还是其他原因。

我重新输入令牌，并重新构造请求体，第二次请求为：

```powershell
$body = @{
    access_token = $env:GITEE_TOKEN
    name         = "git_training"
    path         = "git_training"
    private      = "true"
    auto_init    = "false"
}

$repo = Invoke-RestMethod -Method Post `
    -Uri "https://gitee.com/api/v5/user/repos" `
    -Body $body -ErrorAction Stop

$repo | Select-Object html_url, ssh_url
```

这一次成功返回：

```latex
https://gitee.com/frank-678/git_training.git
git@gitee.com:frank-678/git_training.git
```

**重新输入令牌后，还要更新请求体里的令牌值。** `$body` 保存的是构造时的值，不会因为后来修改环境变量而自动更新。我的第二次操作重新创建了 `$body`。

## 十二、推送主分支、功能分支和标签
### 1. 添加远程并推送 master
创建仓库后，我使用 API 返回的 SSH 地址：

```powershell
git remote add gitee $repo.ssh_url
git push gitee HEAD
```

关键输出：

```latex
To gitee.com:frank-678/git_training.git
 * [new branch]      HEAD -> master
```

这条输出确认远程新建了 `master` 分支，首次推送成功。

`gitee` 是我设置的远程名称，并不是固定关键字。它与前面误写的 `git fetch master` 不同，这里确实配置了一个名为 `gitee` 的远程。[T18]

接着我执行：

```powershell
git push gitee --all
git push gitee --tags
```

第一条的关键输出：

```latex
 * [new branch]      feature/add-subtract -> feature/add-subtract
 * [new branch]      feature/multiply-divide -> feature/multiply-divide
```

第二条的关键输出：

```latex
 * [new tag]         basics -> basics
```

由此可以确认：

| 远程引用 | 成功记录 |
| --- | --- |
| `master` | `git push gitee HEAD` |
| `feature/add-subtract` | `git push gitee --all` |
| `feature/multiply-divide` | `git push gitee --all` |
| `basics` | `git push gitee --tags` |


`--all` 推送所有本地分支，不等于推送所有类型的 Git 引用；这里另用 `--tags` 推送标签。[T19]

### 3. 区分两种认证和两类交付
| 操作 | 使用的认证 |
| --- | --- |
| 调用 Gitee API 创建仓库 | 请求体中的访问令牌 |
| 向 `git@gitee.com:...` 推送代码 | SSH 认证 |


API 建仓与 SSH 推送是两步操作，不能因为建仓使用了令牌，就认为后面的 SSH 推送也依赖同一个环境变量。

## 参考资料
### 学习参考
+ **[H1]** [CSDN 学习参考](https://blog.csdn.net/maxle/article/details/124867297)。本次提供的参考链接；前次整理时未能读取正文，因此未引用该页的具体结论。

### 命令与配置说明
+ **[T1]** [OpenSSH：ssh-keygen](https://man.openbsd.org/ssh-keygen)。密钥类型、注释与文件。
+ **[T2]** [OpenSSH：ssh](https://man.openbsd.org/ssh)。连接认证与主机身份核对。
+ **[T3]** [Git：git-config](https://git-scm.com/docs/git-config)。配置作用域、来源及提交身份。
+ **[T4]** [Git：git-checkout](https://git-scm.com/docs/git-checkout)。检出、分支切换与本地修改。
+ **[T5]** [Git：gitattributes](https://git-scm.com/docs/gitattributes)。文本文件与换行符规范化。
+ **[T6]** [Git：git-branch](https://git-scm.com/docs/git-branch)。分支引用、创建与删除。
+ **[T7]** [Git：git-stash](https://git-scm.com/docs/git-stash)。未提交内容的保存、恢复和删除。
+ **[T8]** [Git：git-fetch](https://git-scm.com/docs/git-fetch)。远程来源与获取操作。
+ **[T9]** [Git：git-merge](https://git-scm.com/docs/git-merge)。快进、三方合并与冲突。
+ **[T10]** [Git：git-tag](https://git-scm.com/docs/git-tag)。附注标签。
+ **[T11]** [Git：git-diff](https://git-scm.com/docs/git-diff)。差异比较与 `--check`。
+ **[T12]** [Git：git-rebase](https://git-scm.com/docs/git-rebase)。基底、提交重放及继续、放弃操作。
+ **[T13]** [Git：git-reset](https://git-scm.com/docs/git-reset)。重置目标及各模式。
+ **[T14]** [Git：git-cherry-pick](https://git-scm.com/docs/git-cherry-pick)。应用指定提交与冲突处理。
+ **[T15]** [Git：git-merge-base](https://git-scm.com/docs/git-merge-base)。祖先关系检查。
+ **[T16]** [Microsoft：Read-Host](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/read-host?view=powershell-5.1)。安全输入。
+ **[T17]** [Microsoft：about_Environment_Variables](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_environment_variables?view=powershell-5.1)。环境变量作用域、持久化与移除。
+ **[T18]** [Git：git-remote](https://git-scm.com/docs/git-remote)。远程名称与地址。
+ **[T19]** [Git：git-push](https://git-scm.com/docs/git-push)。分支、标签推送与上游跟踪。
+ **[T20]** [Microsoft：NetworkCredential.Password](https://learn.microsoft.com/dotnet/api/system.net.networkcredential.password)。字符串密码属性。
+ [Gitee API 文档入口](https://gitee.com/api/v5/swagger)。本文创建请求和返回结果以实际终端记录为依据。

## 附录：本次提交编号对照
| 编号 | 含义 |
| --- | --- |
| `b21792d` | 初始提交 |
| `2d13ae4` | 基础版本，标签 `basics` |
| `be83af5` | 加减功能完成，三个分支曾在此汇合 |
| `92d5a9f` | cherry-pick 处理后生成的原始功能提交 |
| `546b9af` | 第一次变基使用的主分支基底 |
| `bfe9f1f` | 第一次变基后生成的提交 |
| `97d255d` | 主分支上为对照实验新增的修改 |
| `478d2ca` | 第二次变基完成后的提交 |
| `6243af48` | 后续合并备份历史的提交，由截图补充确认 |
| `9daa8f9a` | 截图中的主分支 `.gitignore` 更新 |


