# Linux 学习与实践记录
## 前言
上学期注册一些账号时，我使用过 VMware 虚拟机。假期里，我又卸载了 Windows，体验了两个月的 Arch Linux。在创新创业课上，老师演示过 VS Code 与 WSL 协同开发，让我很感兴趣，但一直没有付诸实践。这次联创作业，我选择使用 WSL，并把安装、配置、使用和排查问题的过程整理下来。

整理时，我结合了终端历史、软件安装日志、配置文件和之前保留的聊天记录。正文按用途解释操作，附录保留原始输入。

## 一、Windows 端准备
我先在 PowerShell 中检查软件包管理工具，并尝试安装终端、Git、VS Code 和 PowerShell：

```powershell
winget -v
winget search Microsoft.VisualStudioCode
winget install --id Microsoft.WindowsTerminal --exact
winget install --id Git.Git --exact
winget install --id Microsoft.VisualStudioCode --exact
winget install --id Microsoft.PowerShell --exact
winget upgrade --all
```

`winget -v` 查看版本，`search` 查询软件包，`install --id ... --exact` 按精确标识安装，最后再批量升级命令。

## 二、安装 WSL2 和 Ubuntu
### 1. 启用组件与检查状态
我参考了文末的 WSL 安装教程，并在管理员 PowerShell 中启用相关组件：

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
wsl --status
wsl --version
wsl --install --no-distribution
wsl --update
wsl --set-default-version 2
wsl --list --online
```

两条 DISM 命令启用 Windows 功能，`/norestart` 将重启留给后续处理。`--status` 查看总体状态，`--version` 查看组件版本，`--no-distribution` 安装 WSL 组件，`--set-default-version 2` 设置新发行版的默认 WSL 版本，`--list --online` 查询可安装发行版。[T1][T2][T4]

最初的教程并不明确，我采取安装尝试：

```powershell
wsl --install -d Ubuntu --location D:\wsl
```

但始终无法安装。（后来我才知道需要先下载 Ubuntu 22.04 的根文件系统）

### 2. 导入到 D 盘
考虑到 C 盘空间，我从 Ubuntu 官方镜像目录下载了 Ubuntu 22.04 的根文件系统，放到 `D:\temp\`，再创建安装目录并导入：[H1][H3][T3]

```powershell
mkdir D:\WSL\Ubuntu2204
wsl --import ubuntu2204 D:\WSL\Ubuntu2204 D:\temp\ubuntu-jammy-wsl-amd64-ubuntu22.04lts.rootfs.tar.gz --version 2
wsl -d ubuntu2204
```

`ubuntu2204` 是发行版名称，`D:\WSL\Ubuntu2204` 是安装位置，压缩包提供初始文件系统。目录中的 `ext4.vhdx` 保存 Linux 文件系统。

### 3. 创建用户
进入 Ubuntu 后，我创建了普通用户 `frank`，将其加入 `sudo` 组：

```bash
adduser frank
usermod -aG sudo frank
sudo nano /etc/wsl.conf
```

`adduser` 创建账户；`usermod -aG` 追加补充组，保留原有组成员关系。`sudo` 组的授权由系统 sudo 配置决定。[T8]

### 4. 启用 systemd、设置默认用户
`/etc/wsl.conf` 中的配置为：

```properties
[boot]
systemd=true

[user]
default=frank
```

这份文件控制当前发行版。`systemd=true` 在发行版启动时启用 systemd（很多服务需要用到），`default=frank` 设置默认登录用户。[T5][T7]

保存后，我通过 `exit` 返回 PowerShell，停止并重新启动 WSL，再设置默认发行版：

```powershell
wsl --shutdown
wsl -d ubuntu2204
wsl --set-default ubuntu2204
wsl
wsl -l -v
```

`wsl --shutdown` 会停止所有运行中的 WSL 发行版；重启前需要保存工作（**因为这个不同于关掉标签，是彻彻底底地关闭！直接释放掉内存**）。设置默认发行版后，输入 `wsl` 即可进入该环境。[T4]

我还多次编辑 `~/.bashrc`（这个东西是开始wsl时候自动会完成的一些命令），在其中加入 `cd ~`，使交互式 Bash 启动后进入家目录。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1790829927499-57c0492d-9faf-436a-a915-0d05ef971b11.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1790830225297-2c84c1ff-4997-41ca-a866-e0a94d39a7cb.png)

（为什么这里会出现tilix？因为tilix是sudo apt install nodejs npm的依赖项之一）

## 三、目录操作与基础检查
开始使用后，我尝试运行这些目录命令：

```bash
cd ~
cd ..
ls
ls d2l/
ls d2l/book/
cd ~/d2l
ls eda/
ls d2l
pwd
exit
```

`~` 表示当前用户家目录，`..` 表示上一级目录；`ls` 查看目录内容，`pwd` 查看当前工作目录，`exit` 结束当前 shell。`ls d2l/` 采用相对路径，实际查找位置取决于执行时的工作目录。

我还检查过发行版信息和硬件工具：

```bash
cat /etc/os-release
lsb_release -sc
nvidia-smi
intel-smi
```

前两条用于确认发行版和代号，当前环境可核对到 Ubuntu 22.04、代号 `jammy`。后两条属于当时的硬件查询尝试：`nvidia-smi` 用于 NVIDIA 设备信息，`intel-smi`用于 INTEL 设备信息。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1790830420603-78324d49-7509-4bbd-ae29-1cef5f50c4e8.png)

## 四、镜像网络与终端代理
### 1. Windows 侧 WSL 配置
我在 PowerShell 中尝试用 Vim 和记事本编辑配置：

```powershell
vim /mnt/c/Users/ZhuJun/.wslconfig
notepad C:\Users\ZhuJun\.wslconfig
wsl --shutdown
wsl
```

实际保存的配置如下：

```properties
[wsl2]
networkingMode=mirrored
dnsTunneling=true
firewall=true
autoProxy=true
memory=8GB
processors=2
swap=0
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1790830597299-8c7964fd-4320-4e90-87d7-b8c84081001d.png)

`.wslconfig` 管理 WSL2 的全局设置。我的配置启用了镜像网络、DNS 隧道、防火墙集成和自动代理，并设置了内存、处理器数量与交换空间。[T5][T6]（当然是[T5][T6]教程的推荐，我也看不懂，但这里埋了一个深坑！后面会debug到）

镜像模式支持从 WSL 通过 IPv4 回环地址访问 Windows 上的服务。`autoProxy` 与 Windows 系统代理配置有关；Clash 的运行状态、系统代理开关和 TUN 模式需要分别检查。[T6][T21]

我还运行过：

```powershell
netstat -aon | findstr :7890
```

这条命令筛选包含 `7890` 的连接或监听记录，可以进一步结合本地地址、状态和 PID 判断对应进程。

### 2. 设置代理变量
在 Ubuntu 里，我手动设置过：

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
export all_proxy=http://127.0.0.1:7890
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"
export ALL_PROXY="$all_proxy"
export no_proxy="localhost,127.0.0.1,192.168.0.0/16,172.16.0.0/12"
export NO_PROXY="$no_proxy"
```

这些变量用于向支持相应约定的程序提供代理信息。不同客户端支持的变量和匹配规则有差异，`no_proxy` 中 CIDR 网段的处理也需要结合客户端及版本核对。`https_proxy` 的值使用 `http://`，描述的是连接代理的协议。[T17]

随后，我把设置和清除代理的操作整理进 `~/.bashrc`，通过 `set_proxy`、`unset_proxy` 调用。它们是自己定义的 shell 函数。当前保存的函数定义如下：

```bash
set_proxy() {
    export http_proxy=http://127.0.0.1:7890
    export https_proxy=http://127.0.0.1:7890
    export all_proxy=http://127.0.0.1:7890
    export HTTP_PROXY="$http_proxy"
    export HTTPS_PROXY="$https_proxy"
    export ALL_PROXY="$all_proxy"
    export no_proxy="localhost,127.0.0.1,192.168.0.0/16,172.16.0.0/12"
    export NO_PROXY="$no_proxy"
    echo "Proxy enabled."
}

unset_proxy() {
    unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY no_proxy NO_PROXY
    echo "Proxy disabled."
}
```

我用 `env | grep proxy`、`env | grep -i proxy` 查看变量，用 `type set_proxy` 检查函数定义。`grep -i` 忽略大小写。

我实际运行过的检查包括：

```bash
ping -c 4 google.com
ping -c 4 8.8.8.8
ip addr
ip addr show eth0
ip -4 route
cat /etc/resolv.conf
curl -I https://github.com
curl -I https://www.google.com
curl -I http://archive.ubuntu.com
curl -I -m 10 https://github.com
```

我通过域名和 IP 地址分别测试连通性，再查看地址、路由和 DNS 配置。`ping` 使用 ICMP；HTTP 代理环境下，网页请求和 ICMP 测试可能走不同路径。`curl -I` 请求响应头，`-m 10` 设置总超时。[T17]

还有 `ping -c 4 172.xx.xx.xx`。附录中的两条长命令将代理变量、DNS、地址及站点访问结果串联输出，用途是把同一轮排查结果集中显示。

至此，wsl可以连接上windows端的代理。

## 五、APT 更新、换源与修复
### 1. 更新索引和软件
我反复执行过：

```bash
sudo apt update && sudo apt upgrade -y
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl git build-essential
sudo apt install -y build-essential
```

`update` 更新软件包索引，`upgrade` 升级已安装的软件，`install` 安装指定软件。`&&` 使后一条命令在前一条成功后继续，`-y` 自动确认常规提示。[T9][T10]

APT 日志确认了升级和 `build-essential` 安装完成。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1790830903770-bf8821c2-d89a-417f-8e1c-3b0f209e0864.png)

### 2. 备份并更换清华镜像
我先查询发行版代号并备份软件源：

```bash
lsb_release -sc
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

随后，通过 `sudo tee /etc/apt/sources.list > /dev/null <<'EOF'` 写入配置。原始多行输入保留在附录中，内容包含清华镜像的 `jammy`、`jammy-updates`、`jammy-backports`、`jammy-security`，同时启用了二进制包源和源码包源。

`tee` 在管理员权限下写入文件，带引号的 `EOF` 分隔符让正文按字面内容传入。镜像站帮助页提供了相应配置说明。[T15]

### 3. 比较镜像与调整连接
我检查过解析结果，并比较两个镜像的访问情况：

```bash
getent ahostsv4 mirrors.tuna.tsinghua.edu.cn
curl -4 --connect-timeout 6 --max-time 15 -L -sS -o /dev/null -w 'TUNA: HTTP %{http_code}; %{speed_download} B/s\n' https://mirrors.tuna.tsinghua.edu.cn/ubuntu/dists/jammy/InRelease
curl -4 --connect-timeout 6 --max-time 15 -L -sS -o /dev/null -w 'ALIYUN: HTTP %{http_code}; %{speed_download} B/s\n' https://mirrors.aliyun.com/ubuntu/dists/jammy/InRelease
```

这里用 `-4` 选择 IPv4，限制连接及总时长，丢弃下载正文，再输出 HTTP 状态码与平均下载速度。原始测速结果没有保存，因此保留测试方法和后续配置变化。[T17]

随后，我备份清华配置，替换为阿里云镜像，并为 APT 设置 IPv4：

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.tuna.bak
sudo sed -i 's|https://mirrors.tuna.tsinghua.edu.cn/ubuntu/|https://mirrors.aliyun.com/ubuntu/|g' /etc/apt/sources.list
printf 'Acquire::ForceIPv4 "true";\n' | sudo tee /etc/apt/apt.conf.d/99force-ipv4 > /dev/null
```

`sed -i` 直接修改文件，备份保留了可恢复的原配置。`Acquire::ForceIPv4` 的范围是 APT。[T11]

### 4. 清理缓存与恢复包状态
我为处理更新异常执行过以下命令：

```bash
sudo rm -rf /var/lib/apt/lists/*
sudo mkdir -p /var/lib/apt/lists/partial
sudo apt clean
sudo apt --fix-broken install
sudo dpkg --configure -a
```

这些命令分别涉及删除索引缓存、恢复索引临时目录、清理下载缓存、尝试修复依赖，以及配置已解包但尚未配置的软件包。[T9][T13]

### 5. 检查终端代理与 APT 配置
我用以下命令检查 APT 配置文件：

```bash
grep -R proxy /etc/apt/apt.conf.d/
sudo grep -RniE 'Acquire::(http|https )::Proxy|Proxy' /etc/apt/apt.conf /etc/apt/apt.conf.d 2>/dev/null || echo '未发现 APT 代理配置'
sudo apt-config dump | grep -iE 'Acquire::(http|https ).*Proxy|ForceIPv4|Pipeline' || true
```

`grep -R` 递归检索，`-n` 显示行号，`-i` 忽略大小写；`apt-config dump` 展示加载后的 APT 配置。历史正则中 `https ` 含多余空格，可能漏掉对应项；`2>/dev/null` 隐藏标准错误，`|| true` 又会改变整体退出状态。排查时需要结合原始输出判断。

我还比较过启用代理前后的环境变量和 `sudo -E` 的保留情况：

```bash
type set_proxy 2>&1
set_proxy
env | grep -iE '^(http|https|all|no )_proxy=' || true
sudo -E env | grep -iE '^(http|https|all|no )_proxy=' || true
```

**后来我才了解**，这处正则中的 `no ` 同样多了空格，会漏掉常见的 `no_proxy` 变量。`sudo -E` 是保留环境的请求。

我尝试通过一条带多个 `-o` 参数的 `apt update` 临时覆盖代理、IPv4 和 HTTP 管线设置，完整命令保留在附录。原输入将代理值写为 `"false"`；APT 文档用于明确直连的特殊值是 `DIRECT`。[T12]

之后，我给 APT 写入独立代理配置：

```bash
printf 'Acquire::http::Proxy "http://127.0.0.1:7890";\nAcquire::https::Proxy "http://127.0.0.1:7890";\n' | sudo tee /etc/apt/apt.conf.d/99proxy > /dev/null
```

之后还有将源地址从 HTTPS 改为 HTTP，再恢复 HTTPS 的尝试：

```bash
sudo sed -i 's|https://|http://|g' /etc/apt/sources.list
sudo sed -i 's|http://|https://|g' /etc/apt/sources.list
sudo apt update
```

这些修改帮助我比较不同传输条件。**但是**协议替换会影响整个文件中匹配的地址，**以至于最后确实没有办法自动恢复。**

意思是：命令会处理 `/etc/apt/sources.list` 的所有行，不是只修改某一个软件源。

例如，文件原来有：

```plain
https://a.example/ubuntu
https://b.example/ubuntu
http://c.example/ubuntu
```

执行第一条命令后，所有 `https://` 都变成 `http://`：

```plain
http://a.example/ubuntu
http://b.example/ubuntu
http://c.example/ubuntu
```

再执行第二条命令，三个地址都会变成 `https://`。

## 六、Node.js、pnpm 与命令行工具
### 1. 安装 nvm 和 Node.js
运行 nvm 安装脚本：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh | bash
```

这条命令下载远程脚本并直接交给 Bash 执行。[T16]

重新加载 nvm 后，我安装并选择 LTS 版本：

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install --lts
nvm use --lts
nvm alias default 'lts/*'
npm install -g pnpm@latest
node -v
npm -v
pnpm -v
git --version
```

`[ -s ... ]` 检查文件存在且非空，`.` 在当前 shell 中加载脚本。`nvm install` 安装版本，`use` 切换当前版本，`alias default` 指定默认选择。`--lts` 和 `latest` 会随时间变化，历史输入本身无法固定最终版本。[T16]

npm 日志确认 pnpm 安装成功。各条版本查询如下：

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1790831754347-df15100b-5279-46fc-ae1f-fa291f387125.png)

### 2. 安装好用的ai工具
我执行过：

```bash
npm install -g @anthropic-ai/claude-code
npm config set allow-scripts=@anthropic-ai/claude-code --location=user
npm install -g @openai/codex-cli
```

日志确认前两条命令退出码为 `0`。第二条写入了用户级 npm 配置。

第三条返回 `E404`，请求的 `@openai/codex-cli` 包未能找到。这次错误让我把包名和发布来源列为安装前的检查项。

后来，我使用了官方文档列出的包名，并更新 npm：

```bash
npm install -g @openai/codex
npm install -g npm@12.0.2
```

两条命令均有退出码 `0` 的日志。此次 npm 安装解析到 Codex `0.155.1`。这是日志记录的安装结果；我在其他位置还尝试过 Snap 渠道，下面单独记录它的排障过程。[T19]



## 七、Snap 网络故障与 Clash
### 1. 复现现象
我尝试启动 `codex`，随后通过 Snap 安装：

```bash
codex
sudo snap install codex
```

安装获取 `core24` 依赖时超时，报错包含：

```latex
cannot install snap base "core24"
Client.Timeout exceeded while awaiting headers
```

这是当时尝试的第三方 Snap 包。官方 npm 渠道的安装过程已在6.2记录。

### 2. 对比 DNS、IPv4、IPv6 和 Snap 服务
我运行了以下检查：

```bash
sudo snap debug connectivity
getent ahosts api.snapcraft.io
curl -4 -I --max-time 15 https://api.snapcraft.io
curl -6 -I --max-time 15 https://api.snapcraft.io
```

保留的输出能够确认：

| 检查 | 当时的结果 | 我的判断（其实是ai给的命令） |
| --- | --- | --- |
| Snap 连通性 | `api.snapcraft.io: unreachable` | Snap 服务访问失败 |
| 域名解析 | 返回 IPv4、IPv6 地址 | 此次查询能够解析域名 |
| IPv4 curl | `200 Connection established`，随后 `200 OK` | 代理连接与目标 HTTP 请求成功 |
| IPv6 curl | `Couldn't connect to server` | 本次指定 IPv6 的连接路径失败 |


我继续查看环境变量、snapd 自身设置，并测试直连：

```bash
env | cut -d= -f1 | grep -i proxy
sudo snap get system proxy.http
sudo snap get system proxy.https
curl --noproxy '*' -4 -I --connect-timeout 5 --max-time 10 https://api.snapcraft.io/
```

`cut -d= -f1` 只显示变量名，避免输出代理凭据；`--noproxy '*'` 让本次 curl 请求绕过显式代理。终端代理指向 `127.0.0.1:7890`，经代理访问成功，直连超时，snapd 的服务环境缺少对应配置。[T17][T20]

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1790832764493-0117faa8-516b-4693-a9a6-8e647d9d6892.png)

### 3. 修复与持久化
我的目标是让 WSL 公网流量进入 Clash 的策略控制范围。我首先确认了 Clash Verge 的 TUN 与 WSL 镜像模式，并进行了两项处理：

1. 保留 `/etc/environment` 的备份，将代理地址写入该文件，并继续核对服务访问情况。
2. 将 `eth0` 的 MTU 从 `9000` 调整为 `1500`，比较修改前后的 TLS 请求结果。

同时，我保存了一个 systemd 单元，使 MTU 设置随发行版启动执行：

```properties
[Unit]
Description=Set WSL eth0 MTU for Clash TUN path stability
After=network.target
ConditionPathExists=/sys/class/net/eth0

[Service]
Type=oneshot
ExecStart=/usr/sbin/ip link set dev eth0 mtu 1500
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

经此更改，Snap 连通性检查通过，绕过显式代理的 IPv4、IPv6 请求返回 HTTP 200，Snap 安装完成，Snap 渠道的命令行版本为 `0.114.0`。随后 npm 渠道安装了前文记录的版本。

DNS 保留了 Windows DNS 隧道。[T6][T21]

## 八、Docker 与虚拟磁盘排查
我还进行了 Docker Desktop 安装和磁盘检查。我在下载目录中先尝试：

```powershell
cd D:\Downloads
ls
"Docker Desktop Installer.exe"  install --installation-dir="D:\Program Files\Docker"
```

随后使用 PowerShell 调用运算符执行安装程序：

```powershell
& ".\Docker Desktop Installer.exe" install --installation-dir="D:\Program Files\Docker"
& ".\Docker Desktop Installer.exe" install --installation-dir="D:\Docker"
```

带引号的路径作为命令执行时需要正确的调用语法。（**这个&是必须的，对于powershell**）`--installation-dir` 指定应用安装目录，Docker 数据位置另有配置和检查需求。[T14][H16]

我为检查虚拟硬盘、目录权限、文件属性和磁盘情况运行了：

```powershell
Get-Item "D:\Program Files\Docker\DockerDesktopWSL\main\ext4.vhdx" | Format-List FullName,Length,Attributes
icacls "D:\Program Files\Docker\DockerDesktopWSL\main\ext4.vhdx"
icacls "D:\Program Files\Docker\DockerDesktopWSL\main"
fsutil fsinfo volumeinfo D:
attrib "D:\Program Files\Docker\DockerDesktopWSL\main\ext4.vhdx"
Get-Volume D: | Format-List DriveLetter,FileSystem,HealthStatus,OperationalStatus,Path
Get-Disk | Format-Table Number,FriendlyName,BusType,PartitionStyle,OperationalStatus,HealthStatus,Size
Get-Partition -DriveLetter D | Get-Disk | Format-List Number,FriendlyName,BusType,OperationalStatus,HealthStatus
Get-DiskImage -ImagePath "D:\Program Files\Docker\DockerDesktopWSL\main\ext4.vhdx" | Format-List Attached,DevicePath,ImagePath,Size,LogicalSectorSize,PhysicalSectorSize
```

这里的 `icacls` 和 `attrib` 没有附带修改参数，用于查看信息；`Get-Volume`、`Get-Disk`、`Get-DiskImage` 分别检查卷、物理磁盘和磁盘映像。

## 参考资料
我从导出的 Edge 浏览历史中筛选了 21 条本次范围内实际访问过的相关公开页面，去除了重复地址、搜索跳转、账户登录页和无关页面。安装、软件仓库、代理客户端及 Docker 相关资料分别保留其用途。正文中的技术解释还结合了 22 条补充核对资料，同一链接只列一次。

### 实际访问过的资料
以下条目是实际浏览的网络上教程。

+ **[H1]** [WSL2 安装配置全流程](https://www.cnblogs.com/Comets9224/p/20233634)。安装过程参考。
+ **[H2]** [Install Ubuntu on WSL 2](https://ubuntu.com/wsl/docs/latest/howto/install-ubuntu-wsl2/)。Ubuntu 官方安装文档。
+ **[H3]** [Ubuntu Jammy WSL 根文件系统目录](https://cloud-images.ubuntu.com/wsl/jammy/current/)。根文件系统下载。
+ **[H4]** [Releases · microsoft/WSL](https://github.com/microsoft/WSL/releases)。WSL 官方发布记录。
+ **[H5]** [保姆级教程：WSL2 的安装和使用](https://zhuanlan.zhihu.com/p/2017602632177427017)。安装与使用。 
+ **[H6]** [Windows10/11 D 盘安装 WSL2](https://zhuanlan.zhihu.com/p/509079404)。指定磁盘安装。
+ **[H7]** [Windows10 11 D 盘安装 WSL2](https://blog.csdn.net/gaochaoran/article/details/124568920)。指定磁盘安装。
+ **[H8]** [WSL2（Ubuntu 22.04 为例）安装到 D 盘完全指南](https://twj0.github.io/blog/2025/12/01/16WSL%E5%AE%89%E8%A3%85%E5%88%B0D%E7%9B%98/)。根文件系统与存储位置。
+ **[H9]** [Windows11 怎么安装 WSL 2，并安装到 D 盘](https://www.langya.org.cn/tech/windows-install-wsl/)。安装步骤。
+ **[H10]** [Windows 终极开发环境：WSL2 从零到一实战指南](https://zhuanlan.zhihu.com/p/1961718875029758545)。开发环境配置。 
+ **[H11]** [Win11 终极开发环境搭建：WSL2 + VS Code + Docker 全攻略](https://blog.csdn.net/qq_45141261/article/details/158346481)。开发工具协同。
+ **[H12]** [极智开发：win11+wsl2+docker+vscode 开发环境构建](https://zhuanlan.zhihu.com/p/618014395)。开发工具协同。 
+ **[H13]** [WSL2 下的环境配置 · Hello CTF](https://hello-ctf.com/hc-pwn/wsl2_environment/)。Linux 工具环境。
+ **[H14]** [Clash Verge Rev v2.5.2](https://github.com/clash-verge-rev/clash-verge-rev/releases/tag/v2.5.2)。代理客户端发布页。
+ **[H15]** [Ubuntu 软件仓库目录](https://archive.ubuntu.com/ubuntu/)。APT 仓库。
+ **[H16]** [Install Docker Desktop on Windows](https://docs.docker.com/desktop/setup/install/windows-install/)。Docker 官方安装说明。
+ **[H17]** [Docker Desktop](https://www.docker.com/products/docker-desktop/)。容器开发工具。
+ **[H18]** [Docker 官方网站](https://www.docker.com/)。Docker 产品与文档入口。
+ **[H19]** [如何在 Windows 上更改 Docker 的默认安装路径？](https://www.zhihu.com/question/359332823/answer/3090155660)。Docker 安装位置。 
+ **[H20]** [在 Windows 上 D 盘上安装 Docker](https://zhuanlan.zhihu.com/p/648138327)。Docker 安装位置。 
+ **[H21]** [Docker Windows 安装（附 D 盘安装）](https://zhuanlan.zhihu.com/p/1894560146467841166)。Docker 安装位置。 



### 整理时补充核对的资料
以下条目用于核对命令和配置的含义，是整理时查阅的项目文档。

+ **[T1]** [Microsoft：Install WSL](https://learn.microsoft.com/en-us/windows/wsl/install)。安装入口。
+ **[T2]** [Microsoft：Manual installation steps for older versions of WSL](https://learn.microsoft.com/en-us/windows/wsl/install-manual)。Windows 可选组件。
+ **[T3]** [Microsoft：Import any Linux distribution to use with WSL](https://learn.microsoft.com/en-us/windows/wsl/use-custom-distro)。根文件系统导入。
+ **[T4]** [Microsoft：Basic commands for WSL](https://learn.microsoft.com/en-us/windows/wsl/basic-commands)。管理命令。
+ **[T5]** [Microsoft：Advanced settings configuration in WSL](https://learn.microsoft.com/en-us/windows/wsl/wsl-config)。两类配置文件。
+ **[T6]** [Microsoft：Accessing network applications with WSL](https://learn.microsoft.com/en-us/windows/wsl/networking)。镜像网络与 DNS 隧道。
+ **[T7]** [Microsoft：Use systemd to manage Linux services with WSL](https://learn.microsoft.com/en-us/windows/wsl/systemd)。服务管理。
+ **[T8]** [Ubuntu：User management](https://ubuntu.com/server/docs/how-to/security/user-management/)。账户与 sudo。
+ **[T9]** [Ubuntu：Install and manage packages](https://ubuntu.com/server/docs/how-to/software/package-management/)。软件包管理。
+ **[T10]** [Debian：apt(8)](https://manpages.debian.org/bookworm/apt/apt.8.en.html)。APT 基本子命令。
+ **[T11]** [Debian：apt.conf(5)](https://manpages.debian.org/bookworm/apt/apt.conf.5.en.html)。APT 配置选项。
+ **[T12]** [Debian：apt-transport-http(1)](https://manpages.debian.org/bookworm/apt/apt-transport-http.1.en.html)。代理与直连设置。
+ **[T13]** [Debian：dpkg(1)](https://manpages.debian.org/bookworm/dpkg/dpkg.1.en.html)。包配置状态。
+ **[T14]** [Docker：Docker Desktop WSL 2 backend on Windows](https://docs.docker.com/desktop/features/wsl/)。WSL 后端。
+ **[T15]** [清华大学开源软件镜像站：Ubuntu 镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)。软件源格式。
+ **[T16]** [nvm-sh/nvm 官方仓库](https://github.com/nvm-sh/nvm)。Node.js 版本管理。
+ **[T17]** [curl：How To Use](https://curl.se/docs/manpage.html)。代理、超时与请求选项。
+ **[T18]** [Astral：Using uv with Jupyter](https://docs.astral.sh/uv/guides/integration/jupyter/)。项目环境中的 Jupyter。
+ **[T19]** [OpenAI：Codex CLI](https://developers.openai.com/codex/cli/)。官方 npm 安装渠道。
+ **[T20]** [Snap：Network requirements](https://snapcraft.io/docs/reference/administration/network-requirements/)。商店网络访问要求。
+ **[T21]** [Mihomo：TUN 配置](https://wiki.metacubex.one/config/inbound/tun/)。TUN 路由及相关边界。
+ **[T22]** [Astral：uv CLI reference](https://docs.astral.sh/uv/reference/cli/#uv-run)。`uv run` 与锁文件选项。



## 附录：原始命令历史
下面保留这次能够恢复并纳入范围的历史原文，包括重复输入、注释、错误命令和多行输入。正文已按用途解释。这里是操作记录。

| 来源 | 读取内容 |
| --- | --- |
| PowerShell 的 `ConsoleHost_history.txt` | Windows 端输入过的命令 |
| WSL 的 `/home/frank/.bash_history`<br/>、`/root/.bash_history` | Linux 命令历史 |
| APT、npm 日志 | 安装时间、包名、退出码和错误 |
| `.bashrc`<br/>、WSL 配置、systemd 服务文件 | 实际保存的配置 |
| 浏览历史 CSV | 访问过的页面和链接 |


### A. Windows 端准备、WSL 与 Docker 相关历史
```powershell
# Confirm winget is available
winget -v
# Search by name or package ID
winget search Microsoft.VisualStudioCode
# Install core Windows-side tools
winget install --id Microsoft.WindowsTerminal --exact
winget install --id Git.Git --exact
winget install --id Microsoft.VisualStudioCode --exact
winget install --id Microsoft.PowerShell --exact
# Later: upgrade packages managed by winget
winget upgrade --all
# 设置默认使用 WSL2 版本
wsl --set-default-version 2
# 查看所有可安装的Linux发行版
wsl --list --online
# 将Ubuntu直接安装到 D:\wsl 目录（自动创建文件夹）
wsl --install -d Ubuntu --location D:\wsl
# 开启WSL子系统功能
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
# 开启虚拟机平台功能（WSL2核心依赖）
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
# 设置默认使用 WSL2 版本
wsl --set-default-version 2
wsl
# 查看所有可安装的Linux发行版
wsl --list --online
# 将Ubuntu直接安装到 D:\wsl 目录（自动创建文件夹）
wsl --install -d Ubuntu --location D:\wsl
wsl --status
wsl --version
mkdir D:\WSL\Ubuntu2204
wsl --import ubuntu2204 D:\WSL\Ubuntu2204 D:\temp\ubuntu-jammy-wsl-amd64-ubuntu22.04lts.rootfs.tar.gz --version 2
# 检查 WSL 状态
wsl --status
# 若尚未启用，安装 WSL 内核组件（注意：这里只装组件，下一步再指定发行版）
wsl --install --no-distribution
# 更新到最新内核
wsl --update
# 把默认版本设为 WSL 2（性能更好，机器学习必需）
wsl --set-default-version 2
wsl --import ubuntu2204 D:\WSL\Ubuntu2204 D:\temp\ubuntu-jammy-wsl-amd64-ubuntu22.04lts.rootfs.tar.gz --version 2
wsl -d ubuntu2204
wsl --shutdown
wsl -d ubuntu2204
wsl --shutdown
wsl -d ubuntu2204
# 以后直接敲 wsl 就能进自己装的 22.04，不用每次 -d
wsl --set-default ubuntu2204
wsl
wsl --shutdown
wsl
wsl --shutdown
wsl
wsl --shutdown
wsl
vim C:\Users\ZhuJun\.wslconfig
notepad C:\Users\ZhuJun\.wslconfig
wsl --shutdown
notepad C:\Users\ZhuJun\.wslconfig
wsl
netstat -aon | findstr :7890
notepad C:\Users\ZhuJun\.wslconfig
wsl --shutdown
Vmmem
vmmenwsl
vmmenWSL
wsl
wsl --shutdown
wsl
grep -R proxy /etc/apt/apt.conf.d/
wsl --shutdown
tree
cd D:\Downloads
ls
"Docker Desktop Installer.exe"  install --installation-dir="D:\Program Files\Docker"
& ".\Docker Desktop Installer.exe" install --installation-dir="D:\Program Files\Docker"
wsl --shutdown
Get-Item "D:\Program Files\Docker\DockerDesktopWSL\main\ext4.vhdx" | Format-List FullName,Length,Attributes
icacls "D:\Program Files\Docker\DockerDesktopWSL\main\ext4.vhdx"
fsutil fsinfo volumeinfo D:
attrib "D:\Program Files\Docker\DockerDesktopWSL\main\ext4.vhdx"
icacls "D:\Program Files\Docker\DockerDesktopWSL\main"
fsutil fsinfo volumeinfo D:
Get-Volume D: | Format-List DriveLetter,FileSystem,HealthStatus,OperationalStatus,Path
Get-Disk | Format-Table Number,FriendlyName,BusType,PartitionStyle,OperationalStatus,HealthStatus,Size
wsl --status
wsl -l -v
Get-Partition -DriveLetter D | Get-Disk | Format-List Number,FriendlyName,BusType,OperationalStatus,HealthStatus
Get-DiskImage -ImagePath "D:\Program Files\Docker\DockerDesktopWSL\main\ext4.vhdx" | Format-List Attached,DevicePath,ImagePath,Size,LogicalSectorSize,PhysicalSectorSize
& ".\Docker Desktop Installer.exe" install --installation-dir="D:\Docker"
cd D:\Downloads
& ".\Docker Desktop Installer.exe" install --installation-dir="D:\Docker"
```

### B. root 用户历史
```bash
adduser frank
usermod -aG sudo frank
sudo nano /etc/esl.conf
exit
sudo nano /etc/wsl.conf
exit
```

### C. frank 用户历史
```bash
exit
nano ~/.bashrc
exit
nano ~/.bashrc
exit
nano ~/.bashrc
exit
cd ~
cd ..
ls
exit
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
export all_proxy=http://127.0.0.1:7890
# 同时设置大写变量，有些程序只认大写
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"
export ALL_PROXY="$all_proxy"
# no_proxy 里只放本地地址，避免本地回环也被塞进代理
export no_proxy="localhost,127.0.0.1,192.168.0.0/16,172.16.0.0/12"
export NO_PROXY="$no_proxy"
nano ~/.bashrc
env | grep -i proxy
curl -I https://github.com      # 配置生效后正常应返回 HTTP/2 200 之类的状态码
curl -I https://www.google.com  # 换个站点再测一次，能正常返回状态码即代理生效
curl -I https://github.com      # 配置生效后正常应返回 HTTP/2 200 之类的状态码
curl -I https://www.google.com  # 换个站点再测一次，能正常返回状态码即代理生效
exit
nvidia-smi
intel-smi
env | grep proxy
ping -c 4 google.com
nano ~/.bashrc
ping -c 4 8.8.8.8
curl -I https://github.com
ip addr
cat /etc/resolv.conf
ping -c 4 172.xx.xx.xx
1
sudo apt update && sudo apt upgrade -y
cat /etc/os-release
sudo apt update && sudo apt upgrade -y
exit
sudo apt update && sudo apt upgrade -y
unset_proxy
sudo apt update && sudo apt upgrade -y
env | grep -i proxy
grep -R proxy /etc/apt/apt.conf.d/
curl -I http://archive.ubuntu.com
cat /etc/resolv.conf
echo "=== proxy ==="; env | grep -i proxy; echo "=== resolv ==="; cat /etc/resolv.conf; echo "=== ip ==="; ip addr show eth0
echo "=== github ==="; curl -I -m 10 https://github.com; echo "=== ubuntu ==="; curl -I -m 10 http://archive.ubuntu.com; echo "=== apt proxy ==="; grep -R proxy /etc/apt/apt.conf.d/ 2>/dev/null
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git build-essential
# 安装 nvm；关闭并重新打开 Ubuntu 终端后继续。
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh | bash
lsb_release -sc
# 备份原软件源
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
# 写入清华镜像源（Ubuntu 22.04 / Jammy）
sudo tee /etc/apt/sources.list > /dev/null <<'EOF'
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse

deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse

deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse

deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
EOF

# 刷新索引并更新已安装软件
sudo apt update && sudo apt upgrade -y
unset_proxy
sudo apt update && sudo apt upgrade -y
wsl --shutdown
quit
exit
getent ahostsv4 mirrors.tuna.tsinghua.edu.cn
sudo rm -rf /var/lib/apt/lists/*
sudo mkdir -p /var/lib/apt/lists/partial
sudo apt clean
sudo apt update
sudo apt --fix-broken install
sudo apt upgrade -y
unset_proxy
sudo rm -rf /var/lib/apt/lists/*
sudo mkdir -p /var/lib/apt/lists/partial
sudo apt clean
sudo apt update
sudo apt --fix-broken install
sudo apt upgrade -y
set_proxy
sudo apt update
curl -4 --connect-timeout 6 --max-time 15 -L -sS -o /dev/null -w 'TUNA: HTTP %{http_code}; %{speed_download} B/s\n' https://mirrors.tuna.tsinghua.edu.cn/ubuntu/dists/jammy/InRelease
curl -4 --connect-timeout 6 --max-time 15 -L -sS -o /dev/null -w 'ALIYUN: HTTP %{http_code}; %{speed_download} B/s\n' https://mirrors.aliyun.com/ubuntu/dists/jammy/InRelease
getent ahostsv4 mirrors.tuna.tsinghua.edu.cn
# 1) 备份当前镜像配置，并改为阿里云镜像
sudo cp /etc/apt/sources.list /etc/apt/sources.list.tuna.bak
sudo sed -i 's|https://mirrors.tuna.tsinghua.edu.cn/ubuntu/|https://mirrors.aliyun.com/ubuntu/|g' /etc/apt/sources.list
# 2 ) 强制 APT 只走 IPv4，避免 WSL/代理环境中的 IPv6 连接问题
printf 'Acquire::ForceIPv4 "true";\n' | sudo tee /etc/apt/apt.conf.d/99force-ipv4 > /dev/null
# 3) 清理此前取消操作留下的索引与缓存
sudo rm -rf /var/lib/apt/lists/*
sudo mkdir -p /var/lib/apt/lists/partial
sudo apt clean
sudo dpkg --configure -a
# 4) 仅更新索引；先不要执行 upgrade
sudo apt update
# 检查 APT 是否保留代理配置；有输出就先不要删，发给我看
sudo grep -RniE 'Acquire::(http|https )::Proxy|Proxy' /etc/apt/apt.conf /etc/apt/apt.conf.d 2>/dev/null || echo '未发现 APT 代理配置'
# 直接绕过 APT 代理、只用 IPv4、关闭易出问题的 HTTP 管线
sudo apt   -o Acquire::http::Proxy="false"   -o Acquire::https::Proxy="false"   -o Acquire::ForceIPv4="true"   -o Acquire::http::Pipeline-Depth="0"   -o Acquire::https::Pipeline-Depth="0"   update
# 重新加载 nvm 环境
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
# 安装 Node.js 的长期支持版 (LTS)
nvm install --lts
# 设为默认版本
nvm use --lts
nvm alias default 'lts/*'
npm install -g pnpm@latest
node -v
npm -v
pnpm -v
git --version
sudo sed -i 's|https://|http://|g' /etc/apt/sources.list
sudo apt update
printf '\n=== set_proxy 定义 ===\n'
type set_proxy 2>&1
printf '\n=== 当前代理变量（先启用）===\n'
set_proxy
env | grep -iE '^(http|https|all|no )_proxy=' || true
printf '\n=== sudo -E 后保留的代理变量 ===\n'
sudo -E env | grep -iE '^(http|https|all|no )_proxy=' || true
printf '\n=== APT 的代理相关设置 ===\n'
sudo apt-config dump | grep -iE 'Acquire::(http|https ).*Proxy|ForceIPv4|Pipeline' || true
printf '\n=== WSL 网络信息 ===\n'
ip -4 route
;29;0;0;48;1_
# 1. 告诉 APT 明确使用 127.0.0.1:7890 代理
printf 'Acquire::http::Proxy "http://127.0.0.1:7890";\nAcquire::https::Proxy "http://127.0.0.1:7890";\n' | sudo tee /etc/apt/apt.conf.d/99proxy > /dev/null
# 2. 再次测试更新（使用阿里云 HTTPS 源 ）
sudo sed -i 's|http://|https://|g' /etc/apt/sources.list
sudo apt update
sudo apt upgrade -y
sudo apt install -y build-essential
npm install -g @anthropic-ai/claude-code
npm config set allow-scripts=@anthropic-ai/claude-code --location=user
npm install -g @openai/codex-cli
ls
ls d2l/
ls d2l/book/
ls
ls d2l/
ls d2l/book/
cd ~/d2l
shutdown
quit
q
exit
uv run --locked jupyter lab
pwd
command -v python; python -c "import sys; printsys.executable"
command -v python3; python3 -c "import sys; printsys.executable"
ls
ls eda/
ls d2l
codex
sudo snap install codex
sudo snap debug connectivity
getent ahosts api.snapcraft.io
curl -4 -I --max-time 15 https://api.snapcraft.io
curl -6 -I --max-time 15 https://api.snapcraft.io
# 只列出已设置的代理变量名，不显示代理地址
env | cut -d= -f1 | grep -i proxy
sudo snap get system proxy.http
sudo snap get system proxy.https
# 绕过代理，直接测试 IPv4 连 Snap Store
curl --noproxy '*' -4 -I --connect-timeout 5 --max-time 10 https://api.snapcraft.io/
npm install -g @openai/codex
npm install -g npm@12.0.2
```



### D. 软件包日志补充
APT 日志还记录了以下操作。

```bash
apt upgrade -y
apt install -y build-essential
apt-get -o Acquire::https::Timeout=30 -o Acquire::Retries=1 install -y --no-install-recommends git help2man perl python3 make autoconf g++ flex bison ccache libgoogle-perftools-dev libjemalloc-dev numactl perl-doc libfl2 libfl-dev zlib1g zlib1g-dev liblz4-1 liblz4-dev gtkwave
```







