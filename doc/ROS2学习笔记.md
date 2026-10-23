# ROS2 学习与实践记录：入门篇第 1 至第 5 章
## 前言
前两次任务接触了 Linux、Git 和 CMake。这次继续使用 Windows 中已经配置好的 WSL 2 与 Ubuntu 22.04，学习《动手学 ROS2》Humble 版的整个入门篇。

正文对应教程前五章，同时补充任务书明确要求的 TF 树和 RViz 调试。C++ 是主要实践语言，Python 部分用于对照理解同一套机制。基础、入门和进阶主题都纳入整理。

本次在现有 WSL 环境中运行了一个小型通信与可视化项目。

### 阅读导航
| 章节 | 我重点回答的问题 | 实践落点 |
| --- | --- | --- |
| 第 1 章：ROS2 介绍与安装 | ROS2 解决什么问题，安装与加载环境有什么区别 | 检查现有 Ubuntu、Humble 与工具包 |
| 第 2 章：第一个节点 | 工作空间、功能包、编译器和节点怎样联系 | C++ 类节点、CMake、colcon 编译 |
| 第 3 章：话题与服务 | 连续数据与一次请求为什么使用不同机制 | C++ 发布订阅、服务客户端、QoS 对照 |
| 第 4 章：参数与动作 | 配置怎么改变行为，长任务如何反馈和取消 | 运行时改参、C++ 动作及取消验证 |
| 第 5 章：常用工具 | 多节点如何启动，数据和坐标问题怎样定位 | Launch、CLI、TF、RViz、RQT、rosbag |


## 第 1 章 ROS2 介绍与安装
### 1.1 ROS2 是什么
ROS2 的全称是 Robot Operating System 2。虽然名称中有“操作系统”，在这里它更适合理解为构建机器人应用的软件库、通信机制和工具集合。Ubuntu 负责进程、内存、文件和设备等操作系统职责；ROS2 帮助上层程序组织节点、交换消息、加载配置和调试系统。[R1][T1]

以视觉小车为例，可以把任务分成相机采集、图像处理、决策和底盘控制。它们不必写在一个巨大的 `main()` 中，也不必由同一个人使用同一种语言完成。各模块只要约定好接口，就能分别开发和测试。

我把 ROS2 的作用理解为两层：一层是**把程序连起来**，另一层是**让连起来的系统可以被观察和管理**。前者包括话题、服务和动作，后者包括参数、Launch、RViz、RQT 和 rosbag。一个例子，知道 `publish()`，怎样确认对方收到数据。

### 1.2 ROS 与 ROS2：为什么会有第二代
ROS1 积累了大量机器人软件，但多机器人、不同网络环境、实时性和产品部署等需求推动了 ROS2 的发展。不能把 ROS2 理解成“给 ROS1 的所有命令加一个 2”。接口、构建方式和底层通信都存在变化。[R1]

| 对比项 | ROS1 常见方式 | ROS2 本次接触的方式 |
| --- | --- | --- |
| 节点发现 | 依赖 ROS Master 进行名称与连接信息协调 | 常见 DDS 实现通过分布式发现建立图关系 |
| 客户端库 | `roscpp`、`rospy` | `rclcpp`、`rclpy` |
| 常见构建组合 | catkin | ament 与 colcon |
| 参数 | 通常谈集中式参数服务器 | 参数归属于具体节点 |
| 通信质量 | 可配置能力与 DDS QoS 不同 | 显式提供可靠性、持久性等 QoS 策略 |
| 节点管理 | 普通节点的启动与退出 | 除普通节点外，还支持受管理的生命周期节点 |


这里“没有 ROS Master”不等于“绝不能出现中心服务”。应用可以主动设计调度器，DDS 也存在发现服务器等配置。既可以框架默认的发现方式，也可以应用自己采用的架构。

### 1.3 Linux、Ubuntu、虚拟机和 WSL 的关系
Linux 是内核；Ubuntu 是包含内核、用户空间工具和软件管理体系的发行版。虚拟机提供虚拟硬件运行完整系统；WSL 2 则让 Linux 环境与 Windows 集成，仍然要区分两边的路径、进程和软件安装位置。

本次选择现有 WSL 环境，没有重新安装系统和 ROS2。先前的 Linux 笔记记录了发行版导入、普通用户、systemd 和网络配置，这次只核对与 ROS2 有关的部分。

```powershell
# Windows PowerShell
wsl --list --verbose
wsl -d ubuntu2204
```

进入 Ubuntu 后再运行：

```bash
cat /etc/os-release
uname -r
ls /opt/ros
source /opt/ros/humble/setup.bash
printenv ROS_DISTRO
command -v ros2
command -v colcon
```

这组命令分别确认发行版、内核、已安装的 ROS 发行版、当前终端环境及命令位置。`ls /opt/ros` 看的是安装目录，`printenv ROS_DISTRO` 看的是当前进程环境，两者回答的问题不同。

本次环境如下：

| 项目 | 实际结果 |
| --- | --- |
| Ubuntu | `22.04.5 LTS`，代号 `jammy` |
| 内核 | `6.18.33.2-microsoft-standard-WSL2` |
| ROS2 | `humble` |
| ROS 安装位置 | `/opt/ros/humble` |
| 图形工具 | RViz2、RQT、turtlesim 已安装 |
| 本次实验域 | `ROS_DOMAIN_ID=73` |
| 本次通信范围 | `ROS_LOCALHOST_ONLY=1`，只做本机验证 |


### 1.4 与 ROS2 实验直接相关的 Linux 命令
| 命令 | 用途 | 备注 |
| --- | --- | --- |
| `pwd`、`ls`、`cd` | 确认和切换工作目录 | `src` 中创建包，工作空间根目录中构建 |
| `mkdir -p ros2_ws/src` | 创建多层目录 | 目录名不是 ROS2 的强制要求，结构约定更重要 |
| `source setup.bash` | 在当前 Shell 中执行环境脚本 | 因为 `source`会在**当前 Shell 中执行脚本，修改环境变量**，使 ROS 2 能找到对应的软件包、可执行文件和库。 |
| `printenv ROS_DISTRO` | 查看变量 | 另一个终端未必拥有相同变量 |
| `apt-cache policy 包名` | 检查软件包候选版本和已安装版本 | 下载文件不代表已经安装成功 |
| `dpkg-query -W 包名` | 查询 Debian 包的安装记录 | 与当前是否已经 source 是两件事 |
| `ps -ef` | 查看进程 | 节点是 ROS 图中的逻辑参与者，不等于操作系统进程的一对一别名 |
| `Ctrl+C` | 向前台进程发送中断信号 | 关闭一个终端不会自动关闭其他终端的节点 |


编辑代码可使用 VS Code 的 WSL 工作区，也可以使用终端编辑器。**编译器、Python 解释器和 ROS 库必须属于同一个目标环境。**Windows 上安装了 CMake，不代表 WSL 里就存在同一套 CMake；同理，Windows Python 不能直接替代 Ubuntu 的系统 Python 来导入 Humble 的二进制扩展。

### 1.5 安装、依赖与环境加载
Humble 的 Ubuntu deb 安装目标包含 Ubuntu 22.04。官方安装流程还包括 locale、软件源和开发工具配置。[T2]

这次已有安装，所以不重复执行安装脚本。需要补充工具时，先检查：

```bash
dpkg-query -W ros-humble-desktop ros-humble-rviz2 ros-humble-tf2-tools
ros2 pkg prefix rviz2
ros2 pkg prefix tf2_tools
ros2 pkg executables turtlesim
```

在已正确配置 ROS 软件源的 Ubuntu 22.04 上，安装常用依赖：

```bash
sudo apt update
sudo apt install ros-humble-desktop python3-colcon-common-extensions \
  ros-humble-tf2-tools ros-humble-tf2-ros ros-humble-rqt-graph
```

`desktop` 包含图形工具与示例；`ros-base` 侧重基础通信和命令行。[R1][T2]

加载环境时，我重点区分两层：

```bash
source /opt/ros/humble/setup.bash
source install/setup.bash
```

第一行加载系统 ROS 环境，称为 underlay；第二行加载自己构建的工作空间，称为 overlay。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792666623110-a2d70beb-724d-4ba4-b806-f9c9977fdfcb.png)自己的包没有被当前终端发现时，先查工作空间是否编译成功、是否 source 了正确路径。[T3]

### 1.6 第一次运行示例：怎样才算“安装可用”
```bash
# 终端 A
source /opt/ros/humble/setup.bash
ros2 run demo_nodes_cpp talker
```

```bash
# 终端 B
source /opt/ros/humble/setup.bash
ros2 run demo_nodes_py listener
```

预期是 A 持续发布，B 打印收到的消息。命令能够启动，只证明可执行程序被找到；两端能交换消息，才进一步证明类型支持和通信链路可用。这个示例还说明 C++ 与 Python 节点可以通过共同的消息接口协作。[T2]

我又一次的debug正是来源于此：

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792675986922-57ee4c40-7bb0-4c4d-adfd-cfd22fa0ed99.png)

### 1.7 ROS2 系统架构与 DDS
从应用向下看，可以按下面的层次理解。不表示所有调用都会严格经过相同数量的函数：

```latex
应用节点：相机、检测、控制
        |
客户端库：rclcpp / rclpy
        |
公共客户端层：rcl
        |
中间件抽象：rmw
        |
DDS 实现及类型支持，例如 Fast DDS
        |
操作系统、网络与可选的本机传输机制
```

DDS 是 Data Distribution Service，即数据分发服务。它把发现、发布订阅和通信策略等问题交给中间件处理。ROS2 在上面提供符合机器人开发习惯的节点、接口、名称和工具。`rmw` 的意义在于隔离中间件实现细节，应用通常不直接调用 Fast DDS 的 `DataWriter`。[R1][T4]

理解几个常见 DDS 对象就足够建立联系：Participant 参与通信域，Topic 对应有名称和类型的数据流，DataWriter 写入数据，DataReader 接收数据。ROS2 的 Publisher/Subscription 与这些概念有关，但底层对象数量和节点数量并不相等。

**ROS2 帮助我约定“数据代表什么、谁来处理”，DDS 等中间件负责“端点怎样发现和交换数据”。**

## 第 2 章 ROS2 第一个节点
### 2.1 先区分四个名字
| 名称 | 含义 | 本次例子 |
| --- | --- | --- |
| 工作空间 | 一组 ROS 包及构建产物的组织目录 | `ros2_ws` |
| 功能包 | 依赖、代码、配置和安装规则的组织单元 | `ros2_learning` |
| 可执行程序 | 操作系统能够启动的程序文件 | `status_publisher` |
| 节点 | ROS 图中的计算与通信参与者 | `/status_publisher` |


一个包可以提供多个可执行程序，一个进程也可以承载多个节点。我们为便于观察，不妨让六个 C++ 程序分别承担不同职责。`ros2 run ros2_learning status_publisher` 中，第二个参数是包名，第三个参数是可执行程序名；节点名则由代码和启动参数决定。[T1][T5]

### 2.2 创建工作空间与功能包
从零创建空包的学习命令为：

```bash
mkdir -p ros2_ws/src
cd ros2_ws/src
ros2 pkg create ros2_learning --build-type ament_cmake \
  --dependencies rclcpp std_msgs example_interfaces
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792670999581-2bd380ce-6614-4107-862a-e1c7026077cd.png)

正常构建后，工作空间还会出现 `build/`、`install/` 和 `log/`。`build` 保存中间产物，`install` 保存可供环境发现和运行的文件，`log` 保存构建记录。[T3]

### 2.3 从 g++、Make 到 CMake，再到 colcon
教程先介绍底层构建工具，是为了看清自动化工具替我们做了什么。对一个不依赖 ROS 的简单 C++ 文件：

```bash
g++ hello.cpp -o hello
./hello
```

`g++` 是适合 C++ 的编译驱动；`gcc` 也能根据扩展名编译 C++，但直接链接时往往还需要正确链接 C++ 标准库。ROS 节点还依赖头文件、多个库和类型支持，因此手动拼出一长串 `-I`、`-L`、`-l` 容易遗漏。

| 工具 | 主要解决什么问题 | 不负责什么 |
| --- | --- | --- |
| 编译器，如 g++ | 把源码编译并参与链接 | 不会自动理解一个 ROS 工作空间的包依赖 |
| Make | 根据目标、依赖和时间戳执行构建规则 | 不直接把 C++ 语法翻译为机器码 |
| CMake | 根据项目描述生成构建系统 | 本身通常不是最终编译器 |
| ament_cmake | 给 CMake 提供 ROS 包相关约定与扩展 | 不代替编译器 |
| colcon | 发现包、组织依赖顺序并调用构建工具 | 不取代每个包的 CMakeLists.txt |


<!-- 这是一张图片，ocr 内容为： -->
![](images/ros2/02-build-flow.png)

图中的“安装”是把本项目生成物放到工作空间安装区，不是向系统重新安装 ROS。

### 2.4 package.xml 与 CMakeLists.txt 分别做什么
`package.xml` 描述包身份、维护者、许可证和依赖。`CMakeLists.txt` 描述怎样找到依赖、生成目标以及安装目标。这两份文件相互配合，不能只改一份。[T5]

例如，在包清单中声明：

```xml
<buildtool_depend>ament_cmake</buildtool_depend>
<depend>rclcpp</depend>
<depend>std_msgs</depend>
<exec_depend>launch_ros</exec_depend>

```

在 CMake 中配置一个发布者目标：

```cmake
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)

add_executable(status_publisher src/status_publisher.cpp)
ament_target_dependencies(status_publisher rclcpp std_msgs)

install(TARGETS status_publisher
  DESTINATION lib/${PROJECT_NAME})
install(DIRECTORY launch config
  DESTINATION share/${PROJECT_NAME})
ament_package()
```

这里是解释结构用的精简片段。实际文件还包含参数校验、服务和可视化节点所需的依赖。`install(TARGETS ...)` 使 `ros2 run` 能找到已安装目标；`install(DIRECTORY ...)` 使 Launch、YAML 和 RViz 配置能够随包一起定位。只生成了可执行文件而没有安装规则，可能出现“编译成功但 ros2 run 找不到程序”。

### 2.5 CMake 与 Python 的依赖查找
`#include` 能找到头文件，与 `find_package()` 能找到包配置，是不同层次的问题。`find_package(rclcpp REQUIRED)` 通常查找 CMake 包配置；ROS 环境脚本把相应前缀加入查找环境，包配置再提供目标和依赖信息。修改环境后，旧的 CMake 缓存也可能影响结果。

排查时可以检查：

```bash
printenv CMAKE_PREFIX_PATH
printenv AMENT_PREFIX_PATH
ros2 pkg prefix rclcpp
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792671132661-931a458a-9628-4fb1-b241-5cc57d105fbb.png)Python 的 `import` 则由解释器的模块搜索路径决定：

```bash
python3 -c 'import sys; print(sys.executable); print(*sys.path, sep="\n")'
python3 -c 'import rclpy; print(rclpy.__file__)'
```

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792671207490-4bff04ab-7016-4f7d-b389-0504856e7c3f.png)本机 Humble 的 `rclpy` 位于 `/opt/ros/humble/local/lib/python3.10/dist-packages/`。如果换成 Windows Python 或另一个 Python 主版本的虚拟环境，即使模块名字相同，也可能无法加载对应二进制扩展。遇到 `ModuleNotFoundError`，应先看解释器和 source 状态，再决定是否安装缺失依赖。

### 2.6 C++ 节点的最小运行过程
本项目继承 `rclcpp::Node`，把发布器、定时器、参数及计数器放进同一个对象中。它比散落的全局变量更容易表达“这些资源属于这个节点”。

`rclcpp`是 ROS Client Library for C++，即 ROS 2 的 C++ 客户端库。

这里的“客户端”不是指向服务器发送请求的客户端，而是指应用程序通过这个库访问 ROS 2 的功能。<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792674087992-b14890b3-4c17-4652-8dcc-1a9b74d3aa67.png)

```cpp
class StatusPublisher : public rclcpp::Node
{
public:
  StatusPublisher() : Node("status_publisher")
  {
    // 创建发布器、声明参数、注册定时器。
  }
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<StatusPublisher>());
  rclcpp::shutdown();
  return 0;
}
```

`init` 初始化客户端上下文并处理 ROS 参数；构造对象时创建通信实体；`spin` 让执行器等待和调度回调；退出后释放资源并调用 `shutdown`。定时器不是写到哪一行便自动重复执行哪一段代码，它要依赖执行器处理到期事件。[T6]

`SharedPtr` 是共享所有权的智能指针。发布器和定时器保留为类成员，能让资源随节点对象保持存活。回调中的 `[this]` 捕获当前对象，使回调访问成员；它并不复制整个节点。多线程执行器下还要考虑对象访问同步，本例使用普通单线程 `spin`。

### 2.7 编译、加载、运行和查询
| 工具 | 职责 |
| --- | --- |
| `CMake` | 生成、组织**具体项目**的构建规则 |
| `colcon` | 发现并按依赖关系构建**工作空间**中的软件包 |
| `ros2 run` / `ros2 launch` | 运行已经构建安装的软件 |


常规复现流程是：

```bash
# 构建终端：进入仓库中的 ros2_ws
source /opt/ros/humble/setup.bash
colcon build --packages-select ros2_learning --executor sequential
```

+ `colcon`：把你写的 ROS 2 源代码构建为可执行程序。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792671851629-212946fb-f128-4caa-96a3-94e2fc49aa85.png)

```bash
# 新运行终端，同样进入 ros2_ws
source /opt/ros/humble/setup.bash
source install/setup.bash
export ROS_DOMAIN_ID=73
export ROS_LOCALHOST_ONLY=1
ros2 launch ros2_learning learning.launch.py
```

`export` 让这个变量还能够被该 Shell 后续启动的子进程读取。

ROS 2 就利用类似的变量控制通信行为。

#### `export ROS_DOMAIN_ID=73`<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/svg/64602776/1792674589131-19dfa8d8-ba1c-4254-b4c9-3960e156609c.svg)
你可以把 DDS Domain 理解成一种逻辑通信隔离机制。

Domain 73 的节点可以发现同一 Domain 内的其他节点，但一般不会发现 Domain 0 的节点。

`73` 不是 IP 地址、端口号，也不是某个节点的编号，只是这次实验选择的 Domain ID。ROS 2 的默认值通常是 `0`。

#### `export ROS_LOCALHOST_ONLY=1`
这个设置是限制通信范围：

+ `1`：只允许通过本机回环接口进行 ROS 2 通信。
+ `0`：不启用这个本机限制，节点可以尝试发现其他机器的节点。

它并不是关闭网络，而是限制 ROS 2 的通信范围。ROS 2 Documentation: Humble documentation

这里有两道独立的隔离条件：

`ROS_DOMAIN_ID` 决定通信域，`ROS_LOCALHOST_ONLY` 决定通信是否限于本机。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792672154790-51030282-41b4-49cb-bd4e-a303715f56b9.png)

我们在colcon时候使用了选项？是的——

先看默认情况：

```plain
colcon build --packages-select ros2_learning
```

执行后，`colcon` 默认在当前工作空间产生：

| 路径 | 存储什么 | 通俗理解 |
| --- | --- | --- |
| `build/` | CMake 缓存、中间构建文件等 | 加工车间 |
| `install/` | 安装后的程序、库、环境配置 | 成品仓库 |
| `log/` | 构建日志、错误输出 | 工作记录 |


本次为避免把构建产物混入交付目录，实际构建使用了 colcon 的 `--build-base`、`--install-base` 和 `--log-base`。注意 `--log-base` 是 `colcon` 的全局参数，因此上面的命令将其写在 `build` 之前。

五个基础 C++ 目标完成构建后，又补充并编译了动作服务端，最终共有六个可执行程序。

另一个终端必须加载同一环境并设置相同实验变量，再查询：

```bash
ros2 node list --no-daemon --spin-time 3
ros2 node info /status_publisher --no-daemon --spin-time 3
```

+ 两个终端：一个运行节点，另一个观察节点或发送请求。
+ `export`：设置环境变量，让 ROS 2 程序知道自己应该加入哪个通信环境。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792672346843-c9cd04db-b71b-41ee-9ba9-d779dd03acb9.png)

实测节点列表包含：

```latex
/add_server
/scene_publisher
/status_publisher
/status_subscriber
```

客户端只有执行请求时短暂存在，因此它不一定出现在这张长驻节点列表中。这不代表客户端没有运行。

### 2.8 Python 节点与 Setup 工具
Python 与 C++ 使用相同的通信概念，API 写法不同。下面是对照用的完整最小 Python 节点，可独立保存运行；它不是本项目 C++ 节点的替换实现：

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class Demo(Node):
    def __init__(self):
        super().__init__('python_demo')
        self.publisher = self.create_publisher(String, '/python/status', 10)
        self.timer = self.create_timer(0.5, self.publish_status)

    def publish_status(self):
        message = String()
        message.data = 'python node is running'
        self.publisher.publish(message)


def main():
    rclpy.init()
    node = Demo()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        if rclpy.ok():
            rclpy.shutdown()


if __name__ == '__main__':
    main()
```

使用 `ament_python` 建包时，`setup.py` 的 `console_scripts` 将可执行入口映射到模块函数，例如 `python_demo = learning_py.demo:main`；`setup.cfg` 指定脚本安装位置；`package.xml` 继续负责 ROS 包元数据与依赖。Python 不需要编译这段源码，并不表示完全不需要安装和入口配置。

### 2.9 colcon 进阶与多机发现
| 命令或选项 | 用途 |
| --- | --- |
| `colcon list` | 先确认工具发现了哪些包。输出格式是：<br/>```plain 软件包名称    软件包路径    软件包类型 ```  |
| `--packages-select ros2_learning` | 只选择指定包，不自动替你补齐未构建的源码依赖 |
| `--packages-up-to 某包` | 包含该包在工作空间内需要先构建的依赖 |
| `--symlink-install` | 对适合链接的安装项减少复制；修改 C++ 仍需要重新编译 |
| `--event-handlers console_direct+` | 直接显示构建输出 |
| `--cmake-clean-cache` | 需要重新配置时清除 CMake 缓存，而非盲目删除整个工作空间 |


多机通信的前提不只是“连了同一个 Wi-Fi”。还要检查域 ID、可达地址、DDS 发现、网络与防火墙、类型支持，以及是否启用了只允许本机的设置。`ROS_DOMAIN_ID` 是通信域选择，不是节点编号；默认值通常为 0，不同域用于隔离互不干扰的 ROS 图。[T7]

本次只做单机 WSL 实验，所以所有终端统一设置 `73` 和 `ROS_LOCALHOST_ONLY=1`。以后做跨机通信时，需要移除或调整只允许本机的设置，并在两端重新启动节点。

---------------10.22终于学完第二章节！---------------

## 第 3 章 ROS2 节点通信之话题与服务
### 3.1 从底层通信理解“约定接口”
两个程序不能因为都在处理“坐标”就自动理解彼此。至少需要约定消息格式、字段含义、单位、坐标系、时间和通信方式。`x=100` 可能是图像像素，也可能是毫米；只有一个没有上下文的数值，还不能支撑可靠的模块协作。

同进程可以调用函数或共享对象；跨进程需要管道、共享内存、套接字等机制；跨机器还要处理网络。TCP 提供有序字节流，但应用仍要定义消息边界；UDP 提供数据报，不自动保证每个报文都到达。ROS2 把常用通信模型抽象成 API，让开发者更多关注数据与业务。

教程中的 ZeroMQ 是理解中间件的过渡：它在套接字之上提供发布订阅、请求应答等模式，但仍需要处理业务协议。DDS 又提供数据类型、发现与 QoS 等能力。** **ZeroMQ 和 DDS 是不同的通信方案，并**不是**先运行 ZeroMQ，再在它上面运行 DDS。  [R1][T4]

更详细地说：

**套接字（socket）是程序收发数据的接口。**

A 中有一个变量 `temperature = 25.0`，B 无法直接读取另一个程序里的变量。它们可以通过操作系统提供的 socket 发送和接收数据。

你在代码里调用发送、接收接口，操作系统负责底下的网络通信。“在套接字之上”就是：利用这些基础能力，再封装出更方便的功能。[Socket API](https://zeromq.org/socket-api/?utm_source=chatgpt.com)

**中间件，是替程序处理公共通信工作的软件。**

假如每写一个程序，都要自己处理连接、消息分发、寻找通信对象，就会反复写很多代码。把这些公共工作做成可以复用的软件，供业务程序调用，就是这里所说的通信中间件。

“中间件”是一类软件的名称；ZeroMQ 是具体的通信库，DDS 则是一套中间件标准，有具体软件来实现它。[ROS 2 Documentation: Kilted documentation](https://docs.ros.org/en/ros2_documentation/kilted/Concepts/Intermediate/About-Different-Middleware-Vendors.html?utm_source=chatgpt.com)

**ZeroMQ 帮你组织“消息怎样来往”。**

它是一个供代码调用的库，提供了现成的通信模式：

| 术语 | 通俗解释 | 温度例子 |
| --- | --- | --- |
| 发布—订阅，Pub/Sub | 一方持续发布，感兴趣的程序订阅接收 | A 每秒发布温度；显示程序、记录程序都可以订阅 |
| 请求—应答，Request/Reply | 一方先提出请求，另一方回复 | B 问“现在多少度？”，A 回答“25 度” |


这些“模式”就是**消息来往的组织方式**。ZeroMQ 已经帮你实现了相应机制。[Socket API](https://zeromq.org/socket-api/?utm_source=chatgpt.com)

**但“消息发到了”，还需要双方约定“消息是什么意思”。这就是业务协议。**

例如，A 发来：

```plain
25
```

B 怎么知道它代表温度，而不是湿度？是摄氏度，还是华氏度？

你可以自行约定，温度消息统一写成：

```plain
{"type": "temperature", "value": 25, "unit": "C"}
```

这里的字段名称、单位、允许的取值，以及收到后如何处理，都是你设计的业务约定。**ZeroMQ 可以传递这条消息，消息内容的含义由你的程序解释。**

**DDS 则把更多通信能力纳入了统一体系。**

DDS 全称是 **Data Distribution Service，数据分发服务**。Fast DDS、Cyclone DDS 是实现 DDS 标准的具体软件。

### 3.2 贯穿项目的模块关系
<!-- 这是一张图片，ocr 内容为： -->
![](images/ros2/01-system-overview.png)

| 程序 | 节点职责 | 主要接口 |
| --- | --- | --- |
| `status_publisher` | 每 0.5 秒发布状态，并读取当前参数 | `/vision/status` |
| `status_subscriber` | 接收并打印状态 | 订阅 `/vision/status` |
| `add_server` | 提供短时计算服务 | `/add_two_ints` |
| `add_client` | 发送一次请求并等待结果 | 调用 `/add_two_ints` |
| `fibonacci_server` | 按目标逐步计算、反馈并处理取消 | 可选启动的 `/fibonacci` 动作 |
| `scene_publisher` | 发布测试图像、Marker 和 TF | `/camera/image_raw`、`/demo/marker`、`/tf`、`/tf_static` |


### 3.3 Topic：一条持续的数据流
话题采用 Publisher-Subscriber 模式。发布者向某个名字、某种类型的话题发布消息；订阅者在收到匹配消息时执行回调。两端不需要知道对方的代码路径，也不需要每发布一帧就写一次“调用订阅者”的函数。[T6]

同一话题可以有多个发布者和多个订阅者。增加一个观察节点，通常无需修改原发布者。它很适合图像、雷达、姿态和状态这类持续数据。

本次使用 `std_msgs/msg/String` 降低初次观察门槛。它适合教学，但真实视觉模块应优先使用有明确字段语义的标准或自定义消息，避免把所有信息拼进字符串，再让接收者解析。

发布者的核心代码是：

```cpp
publisher_ = create_publisher<std_msgs::msg::String>("status", rclcpp::QoS(10));
timer_ = create_wall_timer(500ms, [this]() {
  if (!get_parameter("enabled").as_bool()) {
    return;
  }
  std_msgs::msg::String message;
  message.data = get_parameter("prefix").as_string() +
    " frame=" + std::to_string(count_++) +
    " threshold=" + std::to_string(get_parameter("threshold").as_double());
  publisher_->publish(message);
});
```

`"status"` 是代码中的相对话题名；本次 Launch 将它 remap 为绝对名称 `/vision/status`。`500ms` 是定时回调周期，名义频率为 `1 / 0.5 = 2 Hz`。实际接收频率还会受到调度、系统负载和传输影响。

订阅者的核心代码是：

```cpp
subscription_ = create_subscription<std_msgs::msg::String>(
  "status", rclcpp::QoS(10),
  [this](const std_msgs::msg::String::SharedPtr message) {
    RCLCPP_INFO(get_logger(), "Received: %s", message->data.c_str());
  });
```

这段代码注册一个回调，并没有在构造函数里一直等待消息。执行器调度回调时，才会访问收到的 `message`。如果图像处理耗时很长，把整个算法直接塞进单线程订阅回调，可能阻塞其他回调；增大队列不能从根本上解决处理速度不足。

### 3.4 用命令验证话题，而不是只看日志
启动 Launch 后，在另一个准备好环境的终端执行：

```bash
ros2 topic list -t --no-daemon --spin-time 3
ros2 topic info /vision/status -v --no-daemon --spin-time 3
ros2 topic echo /vision/status std_msgs/msg/String --once --no-daemon
ros2 topic hz /vision/status
```

本次实测依次确认了话题存在、类型为 `std_msgs/msg/String`、有一对发布与订阅端点，以及接收端真正拿到了一条内容：<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792758220504-e991f835-d8bd-433c-8378-25eea65bea79.png)

```latex
Publisher count: 1
Subscription count: 1

data: vision frame=48 threshold=0.500000
---

average rate: 2.006
average rate: 2.002
```

频率是本次采样值，不是以后每次都必须相同的常数。`topic hz` 测的是它自身订阅到的消息频率，不是直接读取发布者定时器的内部配置。

注意：没有--no-daemon不能工作---为什么？？

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792755950995-f2a8cfb7-ab58-4cbf-a508-f6acf6aae1d4.png)

那为什么daemon会坏？？

————————————————————

DEBUG插入：

现在基本可以定位原因了：你的 WSL2 使用 `mirrored`（镜像网络）模式，导致 ROS 2 Humble 的 CLI 在检测 Daemon 时可能发生本地网络连接超时。

这不是 C++ 程序错误，也不是 `colcon build`或 `source`导致的。

更重要的是，我找到了一个与你的环境和现象高度一致的官方 Issue：

[ROS2CLIIssue#934：Daemon检测在WSL2中失败](https://github.com/ros2/ros2cli/issues/934?utm_source=chatgpt.com)

这份报告明确描述：

+ Windows 11 + WSL2 + Ubuntu 22.04 + ROS 2 Humble。
+ WSL2 使用 `networkingMode=mirrored`。
+ 执行 `ros2 topic list`卡住，可能等待约两分钟才超时。

显式执行 `ros2 daemon start`后，普通查询能够正常工作。

与你刚才复现的情况几乎一致。



正常情况下，执行：

```plain
ros2 topic list
```

ROS 2 CLI 会先通过本机 TCP 连接检查 Daemon 是否正在运行。Daemon 使用 XML-RPC 接口提供查询服务。

如果 Daemon 没有运行，理想情况下，连接应当立即失败（`Connection refused`），随后 CLI 就可以启动 Daemon。

但在你的 `mirrored`网络环境下，可能发生了以下过程：<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/64602776/1792755923271-6ef0f0b2-e894-46a6-be55-2f506ae16f6c.png)

执行 `ros2 topic list`

CLI 检测 Daemon 是否存在

WSL2 mirrored 网络异常

本地 TCP 连接未及时成功，也未立即返回拒绝

CLI 阻塞等待 socket.connect() 返回

命令长时间没有输出，最终被 Ctrl+C 中断

Microsoft 的 [WSLIssue#10855](https://github.com/microsoft/WSL/issues/10855?utm_source=chatgpt.com)对这一底层现象进行了复现：在镜像网络模式下，连接一个未监听的本地 TCP 端口，有时会耗费约 132 秒，而不是立即返回失败。





这也解释了你遇到的三个现象：

+ `--no-daemon`正常：绕过了有问题的 Daemon 连接检测。
+ `ros2 daemon stop`卡住：停止操作也可能涉及同一个连接检查问题。
+ `ros2 daemon start`后恢复：显式启动了能够响应请求的 Daemon，此后 CLI 可以正常连接并查询。

需要区分：这是与你日志高度吻合的故障机制，但没有实际的网络抓包或超时堆栈，还不能证明你的每一次卡顿都发生在完全相同的代码位置。



如果再次遇到，先执行：

```plain
ros2 daemon start
ros2 topic list
```

如仍然卡住，临时使用：

```plain
ros2 topic list --no-daemon --spin-time 3
```

如果以后每次新开 WSL 都复发，可以考虑升级 WSL，或评估切换到 `nat`模式。但 `mirrored`对 ROS 2 跨机器网络通信可能有价值，不建议仅为解决这一次问题就立即切换。

最终结论：问题最可能发生在 WSL2 镜像网络模式下的本地 TCP 连接处理，直接表现为 ROS 2 CLI 检查 Daemon 时阻塞；不是 ROS 2 Topic 本身无法通信。



——————————————————





命令行本身也可以成为临时发布者：

```bash
ros2 topic pub --once /vision/status std_msgs/msg/String \
  "{data: 'manual message from CLI'}"
```

### 3.5 Python 对照：同一个 Topic 如何订阅
第二章给出了完整的 Python 发布节点。订阅端使用同样的消息接口，核心写法为：

```python
# 放在一个 rclpy.node.Node 子类的 __init__ 中。
self.subscription = self.create_subscription(
    String, '/vision/status', self.on_status, 10)

# 该类的另一个方法。
def on_status(self, message):
    self.get_logger().info(f'Received: {message.data}')
```

这里 `String` 来自 `from std_msgs.msg import String`。C++ 头文件名使用 `string.hpp`，Python 使用 `String` 类名，CLI 写成 `std_msgs/msg/String`。三种写法都指向相同接口定义，不能把命名差异误认为三种不兼容的业务消息。

### 3.6 Service：一次请求，对应一次应答
服务使用 Client-Server 模型，适合请求一个短时操作，例如查询配置、保存地图、重置计数器。与话题相比，调用者更关心“这一次请求最后得到了什么回应”。本次选择两个整数相加，以便把注意力放在通信流程。[T8]

```bash
ros2 interface show example_interfaces/srv/AddTwoInts
```

接口的核心字段是：

```latex
int64 a
int64 b
---
int64 sum
```

`---` 上面是 Request，下面是 Response。接口中的字段不是节点参数；服务请求里的 `a` 和 `b` 只属于这次调用。

服务端的关键结构是：

```cpp
using Add = example_interfaces::srv::AddTwoInts;
auto service = node->create_service<Add>("add_two_ints",
  [node](const std::shared_ptr<Add::Request> request,
         std::shared_ptr<Add::Response> response) {
    response->sum = request->a + request->b;
    RCLCPP_INFO(node->get_logger(), "Handled one request");
  });
```

这是解释接口的精简片段。实际 `add_server.cpp` 还检查有符号整数溢出；该教学接口没有错误字段，溢出时只能记录错误并返回约定值 0，调用方无法仅凭返回值区分正常的 0 和失败。因此，这个接口适合普通范围整数的通信练习，生产接口应显式表达成功与错误原因。

### 3.7 C++ 客户端：发送异步请求，也要处理超时
本次客户端先等待服务出现，再发送请求，然后在 `main()` 中等待 future 完成：

```cpp
auto client = node->create_client<Add>("add_two_ints");
if (!client->wait_for_service(5s)) {
  // 报错并以非零状态退出。
}
auto future = client->async_send_request(request);
auto state = rclcpp::spin_until_future_complete(node, future, 5s);
if (state == rclcpp::FutureReturnCode::SUCCESS) {
  auto response = future.get();
  // 使用 response->sum。
}
```

`async_send_request` 返回 future，不等于结果已经到达。执行器仍要处理响应，程序也要决定等待多久、超时后怎么办。这里在独立客户端的 `main()` 中等待是为了简化初学流程；不能不加分析地把阻塞等待搬到同一个单线程执行器的服务或订阅回调中，否则可能阻塞响应处理。

实际验证：

```bash
ros2 service list -t --no-daemon --spin-time 3
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 7, b: 5}"
ros2 run ros2_learning add_client
ros2 run ros2_learning add_client --ros-args -p a:=-3 -p b:=8
```

```latex
example_interfaces.srv.AddTwoInts_Response(sum=12)
Result: 7 + 5 = 12
Result: -3 + 8 = 5
```

后面两条命令里的 `-p` 设置的是**客户端节点的启动参数**，客户端再将它们填入请求。这与服务接口自身没有冲突。代码演示了参数与服务能够共同组成一个节点行为。

### 3.8 Python 的服务与客户端
Python 使用相同的 Request/Response 接口。下面是机制对照片段，需置于已初始化的节点中：

```python
from example_interfaces.srv import AddTwoInts

# 服务端
def add(request, response):
    response.sum = request.a + request.b
    return response

service = node.create_service(AddTwoInts, '/add_two_ints', add)

# 客户端
client = node.create_client(AddTwoInts, '/add_two_ints')
if client.wait_for_service(timeout_sec=5.0):
    request = AddTwoInts.Request()
    request.a, request.b = 7, 5
    future = client.call_async(request)
    rclpy.spin_until_future_complete(node, future, timeout_sec=5.0)
    if future.done() and future.exception() is None:
        print(future.result().sum)
```

API 中 `create_service` 和 `create_client` 也分别创建两种实体。语言不同不会改变“先有可用服务、再发请求、处理响应或超时”的基本逻辑。

### 3.9 自定义 msg、srv 与跨语言接口
标准消息能满足需求时优先复用。确实需要新业务字段时，建立独立接口包更方便多个模块共同依赖。下面是**用于学习生成过程的示例**，本次贯穿项目并未构建这个额外接口包。[T9]

```latex
# my_interfaces/msg/Target.msg
std_msgs/Header header
float64 x
float64 y
float64 confidence
```

`header.stamp` 表示数据时间，`header.frame_id` 说明坐标属于哪个参考系。仅把字段写成 `float64`，不能自动保证发布者和订阅者对单位、轴方向及置信度范围理解一致；这些仍然需要文档约定。

服务接口可以表达修改阈值是否被接受：

```latex
# my_interfaces/srv/SetThreshold.srv
float64 value
---
bool success
string message
```

接口包的 CMake 核心是：

```cmake
find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(std_msgs REQUIRED)
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/Target.msg"
  "srv/SetThreshold.srv"
  DEPENDENCIES std_msgs)
ament_export_dependencies(rosidl_default_runtime)
ament_package()
```

同时在 `package.xml` 中声明 `rosidl_default_generators`、`rosidl_default_runtime`、`std_msgs`，并加入 `<member_of_group>rosidl_interface_packages</member_of_group>`。应用包再声明对 `my_interfaces` 的依赖。生成后，C++ 使用 `my_interfaces::msg::Target`，Python 使用 `from my_interfaces.msg import Target`。

`.msg` 是接口源定义，生成工具会创建不同语言的消息类与类型支持。直接修改 `install/` 中生成的头文件，并没有修改真正的接口来源，下次构建还会被覆盖。

### 3.10 原始类型与包装消息
普通 `double` 是 C++ 值，`std_msgs::msg::Float64` 是拥有 `data` 字段的 ROS 消息类型。二者的数值可以相同，但一个普通变量不会因为被赋值就进入 ROS 图。

| 写法 | 意义 |
| --- | --- |
| `double score = 0.8;` | 当前进程内的普通变量 |
| `std_msgs::msg::Float64 message;` | 符合指定 ROS 接口的消息对象 |
| `message.data = score;` | 把业务值写入消息字段 |
| `publisher->publish(message);` | 通过发布器提交消息 |


跨语言共享的是接口的类型与字段契约，而不是直接共享一个 C++ 对象的内存布局。IDL、类型支持及序列化等机制让不同实现能够解释同一份数据；传输时是否复制、采用什么本机优化，需要结合具体中间件和配置判断。

### 3.11 QoS：名字相同，也可能收不到
QoS 不是单独一个“队列大小”。它描述端点对可靠性、历史和持久性等行为的要求。发布者提供的能力要满足订阅者提出的要求，端点才能匹配。[T10]

| 策略 | 关注的问题 | 初学时的理解 |
| --- | --- | --- |
| History / Depth | 保留哪些历史、最多多少样本 | `KeepLast(10)` 不是永久保存最后十秒 |
| Reliability | 是否采用可靠传输机制 | Reliable 不是无限条件下“绝不丢任何一帧”的承诺 |
| Durability | 新订阅者能否取得保留的历史样本 | 静态信息常用 Transient Local；通常还需订阅端匹配 |
| Deadline | 期望多久收到新数据 | 可用于发现更新不及时，不会自动替算法加速 |
| Lifespan | 样本多久失效 | 过期控制信息没有继续使用的价值 |
| Liveliness | 怎样判断发布实体仍活跃 | 与单纯看到进程存在不同 |


可靠性常见组合如下：

| 发布者提供 | 订阅者要求 | 是否兼容 |
| --- | --- | --- |
| Best Effort | Best Effort | 是 |
| Best Effort | Reliable | 否 |
| Reliable | Best Effort | 是 |
| Reliable | Reliable | 是 |


本次额外建立两组测试端点，得到实际结果：

```latex
PASS best_effort -> reliable: 0 received
PASS reliable -> best_effort: received messages
Last incompatible policy: RELIABILITY
```

具体兼容组计数保存在 `evidence/08_checks.txt`。测试在短时间内快速发布，计数只用来确认“确实收到”，不把它作为吞吐量基准。对不兼容组而言，零条消息要结合 QoS 警告和兼容对照组一起看，不能只凭“没输出”就证明问题一定来自 QoS。

本项目状态消息使用默认 `QoS(10)`。图像和雷达更常关注最新数据，可能选择 `SensorDataQoS`；配置和静态坐标信息的策略又可能不同。应由数据用途决定，不应把所有问题都改成 Reliable 来碰运气。

### 3.12 DDS 进阶：Fast DDS 环境与原生发布订阅
教程还介绍 Fast DDS 的原生开发。它与“通过 ROS2 调用 DDS”是两条学习层次。本机已经有 Humble 依赖的中间件，因此本次没有再从源码替换系统 Fast DDS。

原生 DDS 示例的一般过程可以整理为：

1. 在独立目录固定 Fast CDR、Fast DDS 与生成工具的兼容版本，配置构建依赖。
2. 编写 IDL，例如描述包含计数和文本的消息，再生成类型支持。
3. 创建 DomainParticipant，注册类型并创建 Topic。
4. 发布端建立 Publisher 与 DataWriter，订阅端建立 Subscriber 与 DataReader。
5. 等待匹配，写入样本，在读取回调中取样，最后按依赖顺序释放实体。

与 ROS2 对比：ROS2 的 `create_publisher<T>` 和 `create_subscription<T>` 把大量底层细节封装起来，还提供 ROS 名称映射、类型约定与调试生态。原生 DDS 端点不能只因“Topic 字符串看起来一样”，便保证与 ROS2 话题直接互通。

搭建原生环境时，最容易混淆的是系统已经安装的库和自己编译的新库。应固定版本、检查 CMake 前缀与动态库加载路径，不把不兼容版本直接覆盖进 `/opt/ros/humble`。本节属于原理与环境规划，未声称完成原生 DDS 编译或互通实测。[R1][T4]

## 第 4 章 ROS2 通信之参数与动作
### 4.1 开环与闭环：有没有利用反馈调整控制
开环控制按预先给定的输入执行，不根据实际输出纠正本次控制。例如估计电机转两秒能前进一米，于是持续输出两秒后停止；路面阻力改变时，实际距离可能变化。

闭环控制测量输出，与目标比较，根据误差调整输入。例如读取编码器得到速度，再根据目标速度与实测速度的差异调整电机驱动。闭环并不自动等于更准确，传感器误差、延迟和控制器设计仍会影响稳定性。[R2]

<!-- 这是一张图片，ocr 内容为： -->
![](images/ros2/03-control-loops.png)

可以把最基本的控制关系写成：

```latex
误差 e(t) = 目标 r(t) - 测量值 y(t)
控制量 u(t) = 控制器根据误差计算的输入
```

ROS2 可以传输测量、目标和控制命令，但**使用 ROS2 并不会自动产生闭环控制**。本项目按给定轨迹发布 TF，并没有用传感器反馈修正机器人运动，因此不能把 RViz 中的运动方块称为真实闭环控制实验。

### 4.2 参数属于节点，为什么仍有人叫“参数服务器”
ROS2 参数用来配置节点。每个节点保存自己的参数，并默认提供相应服务供其他进程查询和修改。可以沿用任务书中“参数服务器”的说法理解配置能力，但不能由此推断 ROS2 默认存在一个全局集中式参数服务器。[T11]

本次三个业务参数都属于 `/status_publisher`：

| 参数 | 类型与默认值 | 对节点行为的影响 |
| --- | --- | --- |
| `prefix` | 字符串，`vision` | 改变状态消息的文字前缀 |
| `enabled` | 布尔，`true` | 为 false 时，定时回调不发布状态 |
| `threshold` | 浮点数，`0.5` | 进入状态消息，示范运行时读取配置 |


参数类型包括布尔、整数、浮点数、字符串和支持的同类数组等，不是“任何 ROS 消息都能直接作为参数”。图像帧应通过消息接口传输，而不是每帧写进节点参数。[T11]

### 4.3 C++ 声明与读取参数
```cpp
declare_parameter<std::string>("prefix", "vision");
declare_parameter<bool>("enabled", true);
declare_parameter<double>("threshold", 0.5);
```

声明明确参数名、类型和默认值。节点运行时再读取：

```cpp
const auto prefix = get_parameter("prefix").as_string();
const auto enabled = get_parameter("enabled").as_bool();
const auto threshold = get_parameter("threshold").as_double();
```

本例在每次定时回调中读取，因而成功修改参数后，下一次回调便使用新值。如果只在构造函数里读取一次并缓存到成员，之后即使 `param set` 成功，业务逻辑也可能继续使用旧缓存。**参数系统保存了新值，不等于代码已经正确应用了新值。**[T12]

### 4.4 设置参数前先校验
本次将阈值限制在有限浮点数的 `[0, 1]` 范围。运行时校验使用 `add_on_set_parameters_callback`，返回 `SetParametersResult`：

```cpp
rcl_interfaces::msg::SetParametersResult result;
result.successful = true;
// 检查即将写入的参数。
if (invalid_threshold) {
  result.successful = false;
  result.reason = "threshold must be a finite double in [0, 1]";
}
return result;
```

实际代码同时检查类型、`std::isfinite()` 和范围，避免 NaN、无穷大或非法数值进入业务状态。构造函数还单独检查启动时读到的初值，因为启动参数可能在注册运行时校验回调之前就已应用。

校验回调应重点决定“接受还是拒绝”，不要在里面先更新一堆业务状态，再假定参数必定会被接受。还可能存在其他校验回调；若后续拒绝，提前修改的业务状态就会与参数系统不一致。[T11]

### 4.5 实测：改参、拒绝、保留原值、暂停发布
```bash
ros2 param list /status_publisher --no-daemon
ros2 param get /status_publisher threshold --no-daemon
ros2 param set /status_publisher threshold 0.8 --no-daemon
ros2 topic echo /vision/status std_msgs/msg/String --once --no-daemon
```

本次输出由 `0.5` 变成消息中的 `threshold=0.800000`，没有重新编译程序。然后尝试：

```bash
ros2 param set /status_publisher threshold 1.5 --no-daemon
ros2 param get /status_publisher threshold --no-daemon
```

```latex
Setting parameter failed: threshold must be a finite double in [0, 1]
Double value is: 0.8
```

拒绝后原值仍是 `0.8`，比只观察一次报错更能说明校验确实保护了已有状态。这里还有一个实际细节：**本机这版 CLI 对参数拒绝打印 failed，但进程退出码仍然为 0**，所以自动检查必须读取服务结果或输出，不能仅凭退出码判断参数设置成功。

继续修改前缀：

```bash
ros2 param set /status_publisher prefix competition --no-daemon
```

接收到的实际内容包含：

```latex
data: competition frame=180 threshold=0.800000
```

再把 `enabled` 设置为 `false`，观察窗口内没有新的状态消息；恢复 `true` 后重新发布。有限观察时间只能证明该时间窗内停止了输出，结合代码中的布尔判断和恢复行为，才构成完整解释。最后把参数恢复为初始值，避免后续录包混入前一次实验配置。

### 4.6 YAML、启动参数与持久化
本次配置文件为：

```yaml
status_publisher:
  ros__parameters:
    prefix: "vision"
    enabled: true
    threshold: 0.5
```

顶层名称应匹配实际节点名或正确的匹配规则；`ros__parameters` 的拼写与缩进也有意义。运行时 `param set` 改的是节点当前状态，不会自动改写源码中的 YAML。

```bash
# 从参数文件启动单个节点的复习示例。
ros2 run ros2_learning status_publisher --ros-args \
  --params-file src/ros2_learning/config/learning.yaml

# 导出当前参数；输出重定向由 Shell 完成。
ros2 param dump /status_publisher --no-daemon > saved_parameters.yaml
```

如果希望重启后保持新的配置，应检查导出文件并在下次启动时显式加载。保存前确认文件中节点名称与下次启动一致，不应认为“改过一次，所有后续程序就永久变了”。

Python 对应的基本 API 是：

```python
self.declare_parameter('threshold', 0.5)
threshold = self.get_parameter('threshold').value
```

校验可使用 `add_on_set_parameters_callback`，返回 `rcl_interfaces.msg.SetParametersResult`；原则与 C++ 相同，语言变化不会自动解决缓存或校验副作用问题。

### 4.7 Action：有过程的长任务
服务适合快速完成的一次操作。如果要求小车前往目标点，过程可能持续几十秒，只在最后返回一个布尔值不便于上层管理。动作把通信拆成 Goal、Feedback 和 Result，还提供取消请求等机制。[T13]

| 部分 | 回答什么问题 | 导航例子 |
| --- | --- | --- |
| Goal | 希望完成什么 | 前往某个位置 |
| Feedback | 当前进行得怎样 | 剩余距离、当前状态 |
| Result | 最终得到了什么 | 到达、失败或取消结果 |


客户端发出取消请求，并不表示任务已经立刻停止；服务端需要处理取消请求、停止工作并确认结束。Action 也不默认提供通用的“暂停/恢复业务逻辑”，需要应用自行设计。反馈通道只是传输进度，并不等价于反馈控制中的闭环算法。

动作的底层使用服务和话题来组织目标、结果及状态反馈，所以它不是独立于前面通信体系的神秘协议。[T14]

### 4.8 自定义动作接口与 C++ 实现思路
示例接口 `CountFrames.action` 可以定义为：

```latex
int32 target_count
---
int32 completed_count
---
int32 current_count
```

三段依次是目标、结果和反馈。接口生成方法与第三章类似，在 `rosidl_generate_interfaces` 中加入 action 文件，并声明 `action_msgs` 等所需依赖。这里是接口设计学习示例，本次可运行的 C++ 服务端复用已安装的 `action_tutorials_interfaces/action/Fibonacci` 接口。[T15]

C++ 使用 `rclcpp_action`，核心对象是 `Server<Action>`、`Client<Action>` 和 GoalHandle。服务端至少要思考三类回调：

| 回调 | 处理的决定 | 常见 API |
| --- | --- | --- |
| Goal 回调 | 参数是否有效，要不要接收 | `GoalResponse::REJECT` 或 `ACCEPT_AND_EXECUTE` |
| Cancel 回调 | 是否接受取消请求 | `CancelResponse::ACCEPT` |
| Accepted 回调 | 目标已被接收，怎样安排执行 | 启动受管理的工作任务或执行状态机 |


执行过程中，通过 `goal_handle->publish_feedback(feedback)` 发送进度；完成时调用 `succeed(result)`；处理中止调用 `abort(result)`；确认取消后调用 `canceled(result)`。不能接收目标后永远不设置终态，否则客户端会一直等下去。

创建客户端的典型调用关系是：

```cpp
// 类型别名和节点对象省略，这里只展示 API 的连接关系。
auto client = rclcpp_action::create_client<CountFrames>(node, "count_frames");
// 等待 action server 可用后：
// SendGoalOptions 注册 goal_response_callback、feedback_callback、result_callback。
// client->async_send_goal(goal, options);
```

长任务不应直接堵住接收目标的回调。使用工作线程时，还要安排对象生命周期、停止条件与线程回收；简单 `detach()` 后引用即将销毁的节点对象，会带来新的问题。学习示例越简单，越需要写清它省略了哪些部署条件。

### 4.9 Python 动作的对应关系
Python 使用 `rclpy.action.ActionServer` 和 `ActionClient`。接口类型不变，调用概念也对应：

```python
# 结构示意，CountFrames 需要先由接口包生成。
server = ActionServer(
    node, CountFrames, 'count_frames', execute_callback,
    goal_callback=goal_callback, cancel_callback=cancel_callback)

client = ActionClient(node, CountFrames, 'count_frames')
# 等待服务端后发送：
# future = client.send_goal_async(goal, feedback_callback=on_feedback)
# 接收 GoalHandle 后：result_future = goal_handle.get_result_async()
# 需要取消时：goal_handle.cancel_goal_async()
```

`execute_callback` 中可以发布反馈，并在确认状态后返回 Result。若执行回调包含长时间阻塞，还要配置合适的执行器与回调组，使取消请求有机会被处理。仅把函数改名为 `async`，不意味着 ROS 执行器就自动支持任意第三方异步调度方式。

### 4.10 动作实操：C++ 服务端逐步计算 Fibonacci
本次自己编写 `fibonacci_server.cpp`，复用官方教学接口，观察目标接收、过程反馈、结果和取消。以此把动作 API 与一个完整的可运行程序联系起来：

```bash
# 终端 A，选择此命令替代之前的普通 Launch，不要重复启动同名节点。
ros2 launch ros2_learning learning.launch.py use_action:=true
```

```bash
# 终端 B，加载相同 ROS 环境与域设置后
ros2 action send_goal /fibonacci \
  action_tutorials_interfaces/action/Fibonacci "{order: 5}" --feedback --timeout 15
```

接口目标为 `order`，反馈为 `partial_sequence`，结果为 `sequence`。服务端限定 `order` 在 1 至 20 之间，并且一次只接受一个活动目标；每 0.5 秒在定时回调中追加一个数并发布反馈。用这种短回调推进任务，使执行器仍有机会处理取消请求，不需要让工作线程脱离对象管理。

本次最终实测输出包含：

```latex
Goal accepted with ID: ...
Feedback: partial_sequence: [0, 1, 1]
Feedback: partial_sequence: [0, 1, 1, 2]
Result: sequence: [0, 1, 1, 2, 3, 5]
Goal finished with status: SUCCEEDED
```

这里将原来的多行序列压缩显示以便阅读，原始输出完整保存在 `evidence/09_action.txt`。随后在相同环境运行 `python3 scripts/check_action.py`，依次验证正常目标、拒绝 `order=0`、接收 `order=20` 后取消，并取得 `CANCELED` 终态。结果保存在 `evidence/09_action_checks.txt`。

普通动作列表命令为 `ros2 action list -t`，但本机该命令会经过发生等待的守护进程路径，且不支持 `--no-daemon`。需要查询节点接口时可尝试 `ros2 node info /fibonacci_server --no-daemon --spin-time 3`；发送目标本身不以列表查询成功为前提。[T14]

早期单独启动的动作实验遇到发现等待。本次最终在已有基线话题正常收发的同一环境中验证成功，没有据此确定唯一根因。不能把“先额外启动若干节点”写成所有机器必需的动作步骤，也不能将早期未完成的 turtlesim 尝试写成已成功的旋转实验。

### 4.11 普通节点的运行过程，不等于受管理的生命周期
第二章中的 `init → 构造 → spin → shutdown` 描述普通节点的运行过程。ROS2 还有显式的 LifecycleNode 状态机，用于在复杂系统中安排配置、启动、停止与清理。[T16]

<!-- 这是一张图片，ocr 内容为： -->
![](images/ros2/07-lifecycle.png)

| 主要状态 | 含义 | 常见下一步 |
| --- | --- | --- |
| Unconfigured | 对象存在，业务资源尚未完成配置 | configure |
| Inactive | 已配置，尚未激活业务输出 | activate 或 cleanup |
| Active | 处于已激活工作状态 | deactivate |
| Finalized | 已结束的最终状态 | 等待外部销毁 |


configure、activate 等转换还具有过渡状态与失败处理，图中只画主要学习路径。C++ 通过 `rclcpp_lifecycle::LifecycleNode` 及 `on_configure`、`on_activate` 等回调实现；普通 `rclcpp::Node` 不会自动拥有这些管理接口。

```bash
# 仅适用于真正实现生命周期管理的节点，/camera 是示例名称。
ros2 lifecycle get /camera
ros2 lifecycle set /camera configure
ros2 lifecycle set /camera activate
```

本项目没有把普通节点伪装成生命周期节点，这组命令是拓展复习步骤。它的价值在于解释系统启动顺序：驱动先配置成功，感知再激活，控制最后输出，比“所有进程一启动就立即开始工作”更容易管理。

### 4.12 四种机制怎样选择
| 需求 | 优先考虑 | 原因 |
| --- | --- | --- |
| 连续上传图像、姿态和检测结果 | Topic | 关心持续数据流，允许多个订阅者 |
| 请求一次快速操作并取得应答 | Service | 请求与结果之间有明确对应关系 |
| 调整阈值、设备号、开关等配置 | Parameter | 配置归属于节点，通常不是高频数据流 |
| 执行耗时任务并跟踪、取消 | Action | 有目标、进度、结果和任务状态 |


例如，视觉系统可以通过话题发布检测结果，用参数配置置信度阈值，用服务请求保存一张图片，用动作执行一次持续扫描任务。它们可以共存于同一个系统，也可以由同一个节点提供，并不是四选一的框架模式。

## 第 5 章 ROS2 常用工具
### 5.1 Launch：把启动方式变成项目的一部分
如果每次实验都要手动启动四个节点、输入参数和改话题名，很容易出现两个终端配置不一致。Launch 文件描述要启动的程序及其配置，使同一实验可以重复启动。[T17]

本次 `learning.launch.py` 的基本结构为：

```python
def generate_launch_description():
    share = Path(get_package_share_directory('ros2_learning'))
    return LaunchDescription([
        DeclareLaunchArgument('use_rviz', default_value='false'),
        DeclareLaunchArgument('use_action', default_value='false'),
        Node(
            package='ros2_learning', executable='status_publisher',
            name='status_publisher', output='screen',
            parameters=[str(share / 'config' / 'learning.yaml')],
            remappings=[('status', '/vision/status')],
        ),
        # 其余节点见仓库中的完整文件。
    ])
```

| 配置项 | 作用 |
| --- | --- |
| `package` | 指定从哪个已安装包查找程序 |
| `executable` | 指定程序目标，不是随意填写源文件名 |
| `name` | 设置运行时节点名 |
| `parameters` | 为节点提供参数字典或 YAML 文件 |
| `remappings` | 改变名称解析结果，不必修改业务源码 |
| `output='screen'` | 把节点输出显示在启动终端 |
| `IfCondition` | 按启动参数决定是否启动可选程序 |


`get_package_share_directory` 从已安装包定位资源，不依赖终端恰好位于哪个源码目录。相应目录必须通过 CMake 安装，否则源文件存在也不代表运行时能找到它。

本次实际启动入口：

```bash
ros2 launch ros2_learning learning.launch.py
ros2 launch ros2_learning learning.launch.py use_rviz:=true
```

第二条会额外打开 RViz，加载随包提供的显示配置。两条命令选择一条运行，避免同时启动两套同名节点。

### 5.2 参数、remap、namespace 不是同一件事
本次发布和订阅代码都使用 `"status"`，Launch 同时把它们映射到 `/vision/status`。如果只改发布者，订阅者还在监听原来的 `/status`，系统可能两端都正常启动却没有业务数据。

单节点命令的对应写法是：

```bash
ros2 run ros2_learning status_publisher --ros-args \
  -r status:=/vision/status -p prefix:=vision
```

`-r` 改名称映射，`-p` 设置配置值，namespace 为一组相对名称提供前缀。例如节点放进 `/robot1` 命名空间后，相对话题 `status` 可变成 `/robot1/status`；写死的绝对话题 `/vision/status` 不会自动再加该前缀。

Launch 列表的书写顺序，不等于“第一个节点业务初始化已经完成，第二个节点才启动”。如果存在严格就绪条件，应使用服务可用检查、事件或生命周期管理。本次客户端的 `wait_for_service` 正是处理服务端未就绪的一种方式。

### 5.3 ROS2CLI：按调试问题选择命令
下面是围绕本项目整理的命令表。查询与设置命令需要节点正在运行；不支持某个选项时先看对应子命令的 `--help`，不能假设所有命令参数完全统一。

| 目的 | 命令 |
| --- | --- |
| 查看包提供哪些程序 | `ros2 pkg executables ros2_learning` |
| 运行单个节点 | `ros2 run ros2_learning add_client` |
| 一次启动多个节点 | `ros2 launch ros2_learning learning.launch.py` |
| 查看节点 | `ros2 node list --no-daemon` |
| 看一个节点提供什么 | `ros2 node info /status_publisher --no-daemon` |
| 查看话题和类型 | `ros2 topic list -t --no-daemon` |
| 看端点数量及 QoS | `ros2 topic info /vision/status -v --no-daemon` |
| 订阅并打印一条消息 | `ros2 topic echo /vision/status std_msgs/msg/String --once --no-daemon` |
| 临时发送消息 | `ros2 topic pub --once /vision/status std_msgs/msg/String "{data: 'test'}"` |
| 观察接收频率 | `ros2 topic hz /vision/status` |
| 看消息或服务字段 | `ros2 interface show example_interfaces/srv/AddTwoInts` |
| 查看服务 | `ros2 service list -t --no-daemon` |
| 调用服务 | `ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 7, b: 5}"` |
| 查看参数列表 | `ros2 param list /status_publisher --no-daemon` |
| 查看参数值 | `ros2 param get /status_publisher threshold --no-daemon` |
| 修改参数值 | `ros2 param set /status_publisher threshold 0.8 --no-daemon` |
| 发送动作并观察过程 | `ros2 action send_goal /fibonacci action_tutorials_interfaces/action/Fibonacci "{order: 5}" --feedback` |


我的调试顺序是：**包能不能找到 → 程序能不能启动 → 节点能不能发现 → 接口是否匹配 → 实际有没有数据 → 数据是否符合业务含义。** 前一步成立，不自动证明后一步也成立。

### 5.4 TF：数据的坐标系也是接口的一部分
任务书要求生成 TF 树和使用 RViz 观察坐标。虽然坐标变换在教程后续章节还有更深入介绍，这里提前补充能够完成本次调试的基础。[T18]

TF 描述坐标系之间随时间变化的关系。一个点的数值只有结合它属于哪个坐标系才有意义；`header.frame_id` 告诉使用者数据的参考系，而 TF 提供把它转换到另一个参考系的关系。

本次 TF 树为：

```latex
odom
  └─ base_link       动态：模拟底盘位置与朝向，约 10 Hz
      └─ camera_link 静态：相机相对底盘固定安装
```

`odom → base_link` 表示子坐标系相对父坐标系的平移和旋转。本例中底盘绕半径约 0.8 m 的圆运动；`base_link → camera_link` 固定平移 `(0.2, 0, 0.3)` m，旋转为单位四元数。它们只是构造出来的测试关系，不是编码器或真实定位结果。

在本例无旋转的静态关系下，相机坐标系中点 `(1.2, 0, 0)` 转到底盘坐标系可得到 `(1.4, 0, 0.3)`。存在旋转时，需要先按旋转关系变换点再加平移，不能始终把三个坐标直接相加。

### 5.5 C++ 发布动态与静态变换
`scene_publisher.cpp` 使用两种广播器：

```cpp
dynamic_tf_ = std::make_unique<tf2_ros::TransformBroadcaster>(*this);
static_tf_ = std::make_unique<tf2_ros::StaticTransformBroadcaster>(*this);
```

消息类型是 `geometry_msgs::msg::TransformStamped`，需填写时间、父 frame、子 frame、平移和旋转。四元数 `(0, 0, 0, 0)` 不是有效旋转；无旋转应使用 `(0, 0, 0, 1)`。

动态关系约每 100 ms 更新一次；静态相机安装关系只发送一次，静态广播器使用适合后加入订阅者取得数据的策略。不要为了“确保能看到”，在每个节点里重复广播同一个子坐标系的父关系。

TF 应保持无环结构，每个子 frame 在一棵连通树中只有一个父 frame。两个节点同时把 `camera_link` 接到不同父节点，会造成冲突；两棵互不连通的树之间也不能凭空查询变换。

### 5.6 实测 tf2_echo 和 view_frames
```bash
ros2 run tf2_ros tf2_echo odom camera_link
ros2 run tf2_tools view_frames
```

第一个工具持续打印 `camera_link` 相对 `odom` 的变换。刚启动监听器时可能短暂提示等待 frame，应该观察后续是否成功收到，而不是把第一行等待提示当成最终失败。

第二个工具会监听一段时间，生成 TF 拓扑与统计信息。本机版本生成带时间戳的 `.gv` 和 `.pdf` 文件；文件名以实际输出为准，不一定总叫 `frames.pdf`。

<!-- 这是一张图片，ocr 内容为： -->
![](images/ros2/06-tf-frames.png)

图中确实存在 `odom → base_link → camera_link`。动态边记录约 `10.214 Hz`；静态边显示的 `10000.0` 是该工具对静态缓存的统计表现，**不是相机以一万赫兹反复广播**。数值要结合发布方式解释，不能把工具显示的所有“rate”都当作实际频率测量。

### 5.7 RViz：显示收到的数据，而不是自动模拟世界
RViz 是可视化工具。数据源可以是真机、仿真器或测试节点；它本身不会因为显示了一辆车，就自动计算摩擦、碰撞和电机动力学。[T18]

本次配置中的主要显示项：

| Display | 数据或设置 | 看什么 |
| --- | --- | --- |
| Global Options | Fixed Frame 为 `odom` | 整个显示采用哪个参考坐标系 |
| Grid | 0.5 m 单元 | 便于观察相对距离 |
| TF | `/tf`、`/tf_static` | 父子结构、坐标轴和名称 |
| Marker | `/demo/marker` | 一个带坐标参考系的测试立方体 |
| Image | `/camera/image_raw` | 测试图像是否真的被收到与显示 |


启动命令：

```bash
ros2 launch ros2_learning learning.launch.py use_rviz:=true
```

<!-- 这是一张图片，ocr 内容为： -->
![](images/ros2/04-rviz.png)

截图左侧有 Image 显示，右侧能看到 `odom`、`base_link`、`camera_link` 与绿色方块，Global Status 为 Ok。绿色图像矩形与三维方块分别由程序构造，**本例没有相机标定、像素到三维投影或目标检测算法，不能据此声称实现了图像目标定位**。

实际订阅检查确认图像为 `320×240`、`rgb8`，每行 960 字节，共 230400 字节，`frame_id` 为 `camera_link`。这说明消息格式和显示链路能工作；不能直接证明任何真实相机驱动已经工作。

Image 显示适合直接查看像素。若要使用 Camera 显示把图像与三维场景结合，还需要 `CameraInfo`、相机模型和正确的光学坐标约定，本项目没有伪造这些标定数据。

### 5.8 在 RViz 中定位“什么也看不见”
| 现象 | 优先检查 | 为什么 |
| --- | --- | --- |
| Fixed Frame 提示不存在 | TF 是否发布、名字是否准确 | 参考系不存在，其他数据无法正常定位 |
| Marker 提示变换失败 | Marker 的 frame_id、时间及 TF 连通性 | 有消息不等于能转换到显示坐标系 |
| 消息存在但对象看不到 | 相机视角、尺度、透明度和显示开关 | 坐标变换正确，也可能只是视角不合适 |
| 图像没有内容 | 话题名、encoding、step、数据长度与 QoS | 格式错误或端点不匹配都会影响显示 |
| 数据时有时无 | 发布频率、生命周期、TF 时间和过期策略 | 问题不一定是显卡或 RViz 安装损坏 |


使用 `use_sim_time=true` 时，还要确认是否有 `/clock`。一个节点采用系统时间，另一个采用仿真时间，可能出现 TF 时间不一致。看到 extrapolation 提示，应先分清请求的是过去、未来还是缺少缓存，不要只增加等待时间。

### 5.9 RQT：先看结构，再看数值
RQT 是图形调试工具框架，`rqt_graph` 用于观察 ROS 图关系。它与 `view_frames` 展示的不是同一张图：前者关心节点和话题通信，后者关心坐标系之间的变换。[T17]

```bash
ros2 run rqt_graph rqt_graph
```

<!-- 这是一张图片，ocr 内容为： -->
![](images/ros2/05-rqt-graph.png)

本次截图采用 Nodes only 视图，能够看到状态发布者与订阅者之间的 `/vision/status` 连接。图形过滤器可能隐藏无订阅话题、调试话题、图像和其他端点；要查看完整关系，应检查视图模式与过滤选项并刷新。**RQT 里没有画出来，不等于节点实际上没有该接口。**

例如，状态消息的内容可用 `topic echo` 看；数值型姿态可以用 `rqt_plot` 看曲线；图像可以用图像显示插件看。字符串状态消息不适合直接当成数值曲线，不能把所有数据都强行塞进同一种工具。

### 5.10 rosbag：把一次运行的数据保留下来
录包可以把真实发布过的消息保存下来，之后重放给订阅者，便于复现和比较。它记录的是指定话题的数据，不是自动把完整程序、所有参数和所有外部设备状态打包成快照。[T19]

```bash
# 发布者运行时，在另一个终端录制。
ros2 bag record -o status_bag /vision/status
# 观察几秒后 Ctrl+C 停止录制。
ros2 bag info status_bag
```

本次结果：

```latex
Storage id:        sqlite3
Duration:          5.969566953s
Messages:          13
Topic: /vision/status | Type: std_msgs/msg/String | Count: 13
```

录制启动包含发现时间，所以不能简单用“墙上等待了七秒 × 2 Hz”要求恰好十四条。应以实际开始接收后的录制窗口、时间戳和工具统计为准。

回放时先停止原发布者，再启动一个独立订阅者：

```bash
# 终端 A
ros2 run ros2_learning status_subscriber --ros-args -r status:=/vision/status
```

```bash
# 终端 B
ros2 bag play status_bag
```

这样可以区分收到的是录包数据还是现场新数据。原始回放日志见 `evidence/06_bag_replay_listener.txt`，录包文件位于 `evidence/status_bag/`。这个实验强调“停止源节点后仍可收到历史消息”，而不是只看播放器提示已打开数据库。

如果需要完整重放 TF 场景，还要录制 `/tf`、`/tf_static`、图像和相关业务话题，并核对静态数据 QoS 与时间策略。本次只录状态话题，不声称重放了整个三维场景。

### 5.11 Gazebo：与 RViz 不同的职责
教程第五章还介绍 Gazebo。RViz 主要显示数据；Gazebo 则可以依据模型、物理参数、环境和插件计算运动、碰撞及模拟传感器输出。二者经常配合使用，但不能互相替代。

| 工具 | 输入 | 主要输出或作用 |
| --- | --- | --- |
| RViz | 图像、点云、TF、Marker 等消息 | 将已有数据可视化 |
| Gazebo | 机器人模型、场景、物理与传感器配置 | 计算仿真状态，产生模拟数据 |
| ROS 接口或桥接 | 两边的消息与插件配置 | 让控制和传感器数据跨系统传递 |


教程中的传统 Gazebo Classic 需要注意版本时效：官方已明确其于 **2025 年 1 月结束维护**。本次整理日期为 2026 年 10 月，不能照旧把它写成仍处于长期维护期的最新方案。阅读旧教程时应区分 Classic 与后来使用 `ros_gz` 等组件的 Gazebo 体系。[T20]

本机检查 `ros2 pkg prefix gazebo_ros` 返回 `Package not found`，因此本次没有补装或运行 Gazebo，也没有提供仿真成功截图。下面是**已有匹配 Classic 环境时**的学习入口，不是本次实测命令：

```bash
ros2 launch gazebo_ros gazebo.launch.py
```

后续真正做仿真时，需要按目标版本配置 world/model、惯性与碰撞、关节、驱动插件、模拟传感器以及 `/clock`。不要只在 RViz 里显示一个模型，便称为物理仿真；也不要混装不兼容插件后，把加载错误归因于 ROS 话题失效。

## 六、实际问题与排查记录
### 6.1 节点在通信，为什么 CLI 查询却卡住
**现象：** 本次 C++ 发布与订阅日志已经成对出现，`ros2 node list --no-daemon` 可以发现节点，但默认 `ros2 node info` 在记录脚本的 20 秒观察窗内没有返回，部分默认参数查询也有同类现象。

**判断依据：** 数据链路能工作，说明不能仅凭 CLI 等待就判定中间件完全损坏。检查本机安装的 `ros2cli/node/strategy.py` 后，确认相关查询可通过守护进程路径或直接节点路径执行。(AI原话)

**本次处理：** 对支持该选项的查询使用 `--no-daemon --spin-time 3`，然后重新验证节点、端点、消息和参数结果。首轮等待记录保存在 `evidence/10_cli_first_attempt.txt`，后续成功记录保存在对应主题日志。

### 6.2 第一次查询少了一个节点
**现象：** 一次短等待的节点列表没有包含发布者，但启动日志表明进程已经运行，随后订阅端也开始收到消息。

**分析：** DDS 端点发现和图信息传播需要时间；程序被操作系统启动，与查询节点已经得到完整 ROS 图信息，并不发生在同一瞬间。

**处理：** 给启动与发现留出观察时间，使用带 `--spin-time` 的直接查询，必要时重新查询。本次没有通过重装 ROS 或改包名处理这个问题。若等待后仍不完整，再检查域、网络、同名节点与进程是否实际退出。

### 6.3 常见问题速查
| 现象 | 先检查 | 后续处理 |
| --- | --- | --- |
| `ros2: command not found` | 当前是不是 Ubuntu 终端、是否 source | 加载正确系统 ROS 环境 |
| `Package 'ros2_learning' not found` | 是否构建成功、overlay 是否加载 | 检查安装路径与终端环境 |
| 包存在但 executable 找不到 | CMake 的目标和 install 规则 | 重新配置构建并加载结果 |
| `find_package` 失败 | 包是否安装、CMake 前缀与缓存 | 处理缺失依赖或错误环境 |
| `No module named rclpy` | Python 路径、版本和 source | 使用与 Humble 匹配的解释器 |
| 服务客户端一直等待 | 服务名、域、服务器是否运行 | 限时等待并报告失败 |
| 参数文件加载后仍是默认值 | YAML 节点名、缩进和类型 | 核对最终运行时节点名 |
| 改了 YAML 没效果 | 程序是否重新加载文件 | 重启或显式加载，不把磁盘文件当实时配置 |
| RViz 看不到对象 | Fixed Frame、TF、消息、视角 | 逐层查询，避免先入为主重装工具 |
| rosbag 回放收到双份数据 | 原发布者是否还在运行 | 停止原源节点后再比较 |


## 七、这次学习后的理解
### 7.1 我怎样理解节点拆分
拆分节点不是把每个函数都变成独立程序，而是围绕稳定职责与清楚接口组织系统。相机采集与检测算法可以分别替换，控制模块只依赖约定好的目标数据，不需要读取另一个人的源码内部变量。

但拆分也增加了通信、配置和时间问题。一个大程序内部能直接看到的状态，跨节点后需要通过接口表达；因此消息中的单位、坐标、时间和有效性，与“用了哪个 API”同样重要。

### 7.2 “不用重新编译”有明确条件
参数很方便，但不是把任何常量换成参数就完成了设计。配置需要有归属、类型、校验和使用时机。改错阈值可以被拒绝；改传感器设备号可能需要重新打开设备；调整定时周期则需要更新定时器，而不只是改一个数字。

本次通过每次回调读取参数，让前缀和阈值能立即进入新消息。对于更复杂的参数，要再明确变更是否原子、是否允许在 Active 状态修改，以及失败时怎样保持一致。

### 7.3 调试要建立证据链
一张软件窗口截图只说明当时显示了什么；一份日志可能只说明某段代码执行到了打印语句。本次把源码、构建日志、CLI 查询、订阅结果、反例和界面放在一起，是为了回答：程序是否能构建、接口是否存在、数据是否到达、配置是否改变行为，以及显示出的内容究竟代表什么。

遇到未知问题时，我更愿意先写出“已经证实什么、还有什么没证实”，再决定下一步。比如没有拿到消息，可能涉及名字、类型、域、QoS、处理速度或发布开关，不应该第一步就断定“ROS2 没装好”。

## 八、范围与材料索引
### 8.1 对照教程第 1 至第 5 章
| 教程内容组 | 本文位置 | 范围说明 |
| --- | --- | --- |
| Linux/Ubuntu、安装方式、常用命令、编辑工具与软件 | 1.3 至 1.6 | 延续既有 Linux 环境；本次核验安装与使用 |
| ROS 历史、ROS/ROS2 对比、初体验 | 1.1、1.2、1.6 | 概念整理与可复习命令 |
| 系统架构、DDS 中间件 | 1.7、3.12 | 架构理解；不声称重建原生 DDS 环境 |
| gcc、Make、CMake、CMake/Python 依赖查找、Setup | 2.3 至 2.5、2.8 | 解释工具链和包入口 |
| 节点、包、工作空间、colcon、C++/Python 节点 | 2.1 至 2.8 | C++ 项目已编译运行，Python 给出对照实现 |
| 面向对象、colcon 进阶、发现与多机通信 | 2.6、2.9 | 类节点实测；多机条件属于拓展 |
| 底层通信、ZeroMQ、中间件 | 3.1 | 理解通信模式与协议边界 |
| Topic 与 Service 的 C++、Python 实现 | 3.3 至 3.8 | C++ 两种机制实测；Python API 对照 |
| ROS 接口、自定义 msg/srv、原始与包装类型 | 3.9、3.10 | 接口生成链路与跨语言使用 |
| QoS、Fast DDS 环境与原生发布订阅 | 3.11、3.12 | QoS 实测；原生 DDS 为拓展整理 |
| 开环/闭环、参数、C++/Python 参数 API | 4.1 至 4.6 | 参数声明、读取、修改与拒绝实测 |
| 动作定义、自定义接口、C++/Python 实现、机制比较 | 4.7 至 4.10、4.12 | 完整 C++ 服务端与成功、拒绝、取消实测；Python 客户端验证 |
| 参数通信原理、生命周期节点 | 4.2、4.4、4.11 | 普通与受管理生命周期分开说明 |
| Launch、ROS2CLI、RViz、RQT、rosbag、Gazebo | 第 5 章 | 前五种工具有实际记录；Gazebo 说明版本及未运行边界 |
| 任务书额外要求：TF 树与 RViz 中的 TF | 5.4 至 5.8 | 实际发布、查询、生成拓扑并显示 |


## 参考资料
以下保留教程与本次核对的官方来源。部分 ROS 文档网页在读取时有访问限制，因此核对了 ROS2 官方文档仓库的 Humble 分支源文件；链接直接指向实际读取的来源。

+ **[R1]** [《动手学 ROS2》Humble 教程](https://fishros.com/d2lros2/)，章节范围以[官方教程目录](https://github.com/fishros/d2l-ros2/blob/master/docs/_sidebar.md)为准。
+ **[R2]** [教程：开环控制与闭环控制](https://github.com/fishros/d2l-ros2/blob/master/docs/humble/chapt4/basic/1.开环控制与闭环控制.md)。
+ **[T1]** [ROS2：Nodes](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Concepts/Basic/About-Nodes.rst)。节点、进程及通信职责。
+ **[T2]** [Humble Ubuntu deb 安装说明](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Installation/Ubuntu-Install-Debs.rst)。安装目标、环境加载与示例。
+ **[T3]** [Creating a Workspace](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Tutorials/Beginner-Client-Libraries/Creating-A-Workspace/Creating-A-Workspace.rst)。underlay、overlay 与构建目录。
+ **[T4]** [教程：ROS2 系统架构](https://github.com/fishros/d2l-ros2/blob/master/docs/humble/chapt1/advanced/1.ROS2系统架构.md)与[DDS 架构](https://github.com/fishros/d2l-ros2/blob/master/docs/humble/chapt1/advanced/2.中间件DDS架构.md)。
+ **[T5]** [Creating Your First ROS2 Package](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Tutorials/Beginner-Client-Libraries/Creating-Your-First-ROS2-Package.rst)。包元数据、构建方式与程序入口。
+ **[T6]** [C++ Publisher and Subscriber](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Cpp-Publisher-And-Subscriber.rst)。回调、执行器与依赖。
+ **[T7]** [ROS_DOMAIN_ID](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Concepts/Intermediate/About-Domain-ID.rst)。通信域与端口范围。
+ **[T8]** [C++ Service and Client](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Cpp-Service-And-Client.rst)。请求、响应与客户端等待。
+ **[T9]** [Custom ROS2 Interfaces](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Tutorials/Beginner-Client-Libraries/Custom-ROS2-Interfaces.rst)。msg/srv 生成与应用包依赖。
+ **[T10]** [Quality of Service Settings](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Concepts/Intermediate/About-Quality-of-Service-Settings.rst)。QoS 策略与兼容性。
+ **[T11]** [Parameters](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Concepts/Basic/About-Parameters.rst)。参数归属、类型、服务与校验回调。
+ **[T12]** [Using Parameters in a C++ Class](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Tutorials/Beginner-Client-Libraries/Using-Parameters-In-A-Class-CPP.rst)。声明与读取。
+ **[T13]** [Actions](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Concepts/Basic/About-Actions.rst)。长任务、反馈和取消。
+ **[T14]** [Understanding ROS2 Actions](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Actions/Understanding-ROS2-Actions.rst)。turtlesim 动作与命令。
+ **[T15]** [Creating an Action](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Tutorials/Intermediate/Creating-an-Action.rst)。动作接口的三段结构。
+ **[T16]** [Managed Nodes 设计文档](https://raw.githubusercontent.com/ros2/design/gh-pages/articles/node_lifecycle.md)。主要状态与转换。
+ **[T17]** [Creating Launch Files](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Tutorials/Intermediate/Launch/Creating-Launch-Files.rst)。多节点配置与 RQT 图。
+ **[T18]** [Introducing tf2](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Tutorials/Intermediate/Tf2/Introduction-To-Tf2.rst)。tf2_echo、view_frames 和 RViz。
+ **[T19]** [Recording and Playing Back Data](https://raw.githubusercontent.com/ros2/ros2_documentation/humble/source/Tutorials/Beginner-CLI-Tools/Recording-And-Playing-Back-Data/Recording-And-Playing-Back-Data.rst)。录制、统计与回放。
+ **[T20]** [Gazebo Classic 官方页面](https://classic.gazebosim.org/)。Classic 的作用与维护结束信息。

