# 第1次进度汇报

**汇报人：** 袁崇皓  
**时间范围：** 2026年7月25日—2026年7月27日  
**当前阶段：** 机器人软件与算法基础学习  

---

## 1. 本阶段目标

本阶段主要目标是完成机器人方向的基础环境搭建，开始学习 ROS 2 基础内容，并建立后续科研学习、实验记录和代码管理方式。

---

## 2. 已完成内容

### 2.1 Ubuntu 与 ROS 2 环境配置

- 在 VMware 中完成 Ubuntu 系统安装；
- 完成 ROS 2 Humble 安装与环境配置；
- 完成 `turtlesim` 和 `rqt` 安装；
- 能够正常启动 ROS 2 节点和小乌龟仿真程序；
- 完成 VS Code 及相关开发环境的基本配置。

### 2.2 ROS 2 基础学习

已完成 ROS 2 初学者教程中的部分基础内容，包括：

- 节点（Node）的基本概念和常用命令；
- 话题（Topic）的发布与订阅机制；
- 服务（Service）的请求与响应机制；
- 参数和接口的基本查看方法；
- 使用 `rqt` 查看 ROS 2 系统中的节点和通信关系。

目前能够使用以下常用命令查看 ROS 2 系统状态：

```bash
ros2 node list
ros2 node info <node_name>

ros2 topic list
ros2 topic echo <topic_name>
ros2 topic info <topic_name>
ros2 topic hz <topic_name>

ros2 service list
ros2 service type <service_name>
ros2 service call <service_name> <service_type> "<request_data>"

ros2 interface show <interface_type>
```

### 2.3 ROS 2 客户端库基础

已开始学习 ROS 2 Python 客户端库，完成了以下内容：

- 创建 ROS 2 工作空间；
- 创建 Python 功能包；
- 了解工作空间、功能包和节点之间的关系；
- 初步了解 ROS 2 Python 程序的基本组织方式。

### 2.4 科研记录和代码管理

- 建立本地科研学习文件夹；
- 使用 Markdown 记录实验过程和每周进度；
- 安装并配置 Git；
- 创建本地 Git 仓库并完成第一次提交；
- 创建 GitHub 远程仓库；
- 初步了解 `add`、`commit` 和 `push` 的作用。

当前仓库结构：

```text
robotics_learning/
├── README.md
├── experiment_logs/
├── weekly_reports/
├── code/
└── images/
```

---

## 3. 当前成果

1. ROS 2 Humble 环境可以正常运行；
2. 能够使用命令行查看节点、话题、服务和接口；
3. 能够完成基础的 ROS 2 话题和服务实验；
4. 已建立科研实验记录和代码版本管理框架；
5. 已创建 GitHub 仓库，用于保存后续代码和实验记录。

**GitHub 仓库：** https://github.com/jimmmmmyy1124-wq/robotics_learning

---

## 4. 遇到的问题及解决情况

### 4.1 GitHub 连接失败

首次上传 GitHub 时，当前网络无法连接 GitHub 的 443 端口，终端提示：

```text
Recv failure: Connection was reset
```

通过检查网络连接，确认原 Wi-Fi 对 GitHub 连接存在限制。切换手机热点后，443 端口连接恢复正常。

### 4.2 ROS 2 概念较多，容易混淆

在学习过程中，节点、话题、服务、客户端和服务端之间的关系容易混淆。

目前的理解是：

- 节点是 ROS 2 程序运行的基本单位；
- 话题用于连续、异步的数据传输；
- 服务用于一次请求和一次响应；
- Client 和 Server 通常是节点内部承担通信功能的部分。

---

## 5. 当前不足

- 目前主要完成了 ROS 2 命令行操作，对底层通信过程理解还不够深入；
- 尚不能完全独立编写 ROS 2 Python 节点；
- 还没有开始规划、感知和 SLAM 算法的系统学习。

---

## 6. 下一阶段计划

1. 继续学习 ROS 2 Python 客户端库；
2. 独立完成 Publisher 和 Subscriber 程序；
3. 独立完成 Service Client 和 Server 程序；
4. 学习 Action 和 Launch 文件的基本使用；
5. 将代码、实验记录和运行截图持续上传至 GitHub；
6. 完成 ROS 2 基础后，逐步进入机器人规划和感知算法学习。

