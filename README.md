# HelloCV

哈尔滨工业大学（威海）联创迎新科创训练营视觉（算法）方向的学习笔记与实践记录。

## 仓库结构

| 文件 / 目录 | 内容 |
| --- | --- |
| `README.md` | 仓库说明、环境配置与使用方法 |
| [notes-links.md][notes] | 全部语雀笔记链接及对应本地文稿 |
| [repos-links.md][repos] | 实践仓库链接，包括 [git_training][git-training] |
| [doc/][docs] | Linux、Git 学习笔记与实践记录的 Markdown 存档 |
| `LICENSE` | 许可证 |

## 环境配置

仅阅读文档无需安装依赖。复现实践时，按对应笔记配置环境：

1. **Linux**：在 Windows 中准备终端等工具，安装 WSL 2 与 Ubuntu 22.04，创建用户并配置网络、软件源。完整过程见 [Linux 学习与实践记录][linux-local]。
2. **Git**：安装 Git，检查版本并设置提交姓名与邮箱，步骤见 [Git 学习笔记][git-local]；向 Gitee 推送前的 SSH 公钥配置与连接测试见 [Git 实践任务][git-practice-local]。

## 使用说明

1. 从[学习笔记索引][notes]选择主题，阅读语雀原文或本地文稿。
2. 从[实践仓库索引][repos]进入 `git_training`，查看练习代码、分支和提交记录。

笔记包含排障过程和错误输入，复现实验时需结合上下文，不要直接批量执行历史命令。

[notes]: ./notes-links.md
[repos]: ./repos-links.md
[docs]: ./doc/
[git-training]: https://gitee.com/frank-678/git_training
[linux-local]: <./doc/安装、配置和学习 Linux 的过程，以及遇到的问题和解决方法.md>
[git-local]: ./doc/Git学习笔记.md
[git-practice-local]: ./doc/Git实践任务.md
