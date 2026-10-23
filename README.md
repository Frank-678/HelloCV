# HelloCV

哈尔滨工业大学（威海）联创迎新科创训练营视觉（算法）方向的学习笔记与实践记录。

## 仓库结构

| 文件 / 目录 | 内容 |
| --- | --- |
| `README.md` | 仓库说明、环境配置与使用方法 |
| [notes-links.md][notes] | 全部语雀笔记链接及对应本地文稿 |
| [repos-links.md][repos] | 实践仓库链接 |
| [doc/][docs] | 各阶段学习笔记与实践记录的 Markdown 存档 |
| `LICENSE` | 许可证 |

## 各阶段内容

### 第一次任务：Linux + Git

- **Linux**：在 Windows 中配置 WSL 2，安装 Ubuntu 22.04，配置网络与软件源。
- **Git**：配置提交身份，分支管理，暂存与合并，变基，远程推送到 Gitee。

实践仓库：[frank-678/git_training (Gitee)][git-training]

### 第二次任务：CMake + LLM

- **CMake**：学习 CMakeLists.txt 的写法，模块化构建（`add_subdirectory` + `add_library`），`target_include_directories` 的 PUBLIC 传播，源码外构建流程。
- **LLM**：通过 DeepSeek API 实现石头剪刀布决策网关，学习 HTTP 调用 API、JSON 容错处理和重试逻辑。

实践仓库：[frank-678/UnitConverter][unitconv]、[frank-678/learn-cmake][learn-cmake]、[frank-678/ai-response][ai-response]、[frank-678/llm_traning_repo][llm-training]

### 第三次任务：ROS2

- **ROS2**：在 WSL 2 + Ubuntu 22.04 的 ROS 2 Humble 中学习《动手学 ROS2》入门篇，以 C++ 实践节点、话题、服务、参数与动作，并结合 Launch、CLI、TF、RViz、RQT、rosbag 完成调试与可视化。

实践仓库：[frank-678/ros2_ws (Gitee)][ros2-ws]

## 环境配置

复现各实践任务时，按对应笔记配置环境：

1. **Linux**：WSL 2 + Ubuntu 22.04，完整过程见 [Linux 学习与实践记录][linux-local]。
2. **Git**：安装 Git，配置提交身份，SSH 公钥配置，见 [Git 学习笔记][git-local] 与 [Git 实践任务][git-practice-local]。
3. **CMake**：Windows 上需要 CMake ≥ 3.16 和 MinGW-w64（含 `mingw32-make`），见 [CMake 学习笔记][cmake-local] 与 [CMake 实践任务][cmake-practice-local]。
4. **LLM**：Python 3.x，`pip install requests`，DeepSeek API Key（`DEEPSEEK_API_KEY` 环境变量），见 [LLM 实践任务][llm-local]。
5. **ROS2**：WSL 2 + Ubuntu 22.04 中的 ROS 2 Humble（`source /opt/ros/humble/setup.bash`）与 colcon，见 [ROS2 学习与实践记录][ros2-local]。

## 使用说明

从[学习笔记索引][notes]选择主题，阅读语雀原文或本地文稿。从[实践仓库索引][repos]查看实践代码。

笔记包含排障过程和错误输入，复现实验时结合上下文，不要直接批量执行历史命令。

[notes]: ./notes-links.md
[repos]: ./repos-links.md
[docs]: ./doc/
[git-training]: https://gitee.com/frank-678/git_training
[unitconv]: https://gitee.com/frank-678/UnitConverter
[learn-cmake]: https://gitee.com/frank-678/learn-cmake
[ai-response]: https://gitee.com/frank-678/ai-response
[llm-training]: https://gitee.com/frank-678/llm_traning_repo
[ros2-ws]: https://gitee.com/frank-678/ros2_ws
[linux-local]: <./doc/安装、配置和学习 Linux 的过程，以及遇到的问题和解决方法.md>
[git-local]: ./doc/Git学习笔记.md
[git-practice-local]: ./doc/Git实践任务.md
[cmake-local]: ./doc/CMake学习笔记.md
[cmake-practice-local]: ./doc/CMake实践任务.md
[llm-local]: ./doc/LLM实践任务.md
[ros2-local]: ./doc/ROS2学习笔记.md
