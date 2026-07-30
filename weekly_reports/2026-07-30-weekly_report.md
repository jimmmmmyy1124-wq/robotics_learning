# 第1次进度汇报

**汇报人：** 袁崇皓  
**时间范围：** 2026年7月25日—2026年7月30日  
**当前阶段：** 机器人软件与算法基础学习  

---

## 1. 本阶段目标

本阶段主要目标是完成 Ubuntu 与 ROS 2 基础开发环境搭建，学习 ROS 2 的基础概念和常用命令，并进一步使用 Python 编写发布者、订阅者、服务端和客户端程序。同时建立科研学习记录、代码管理和每周进度汇报方式，为后续机器人规划、感知和控制方向的学习打好基础。

---

## 2. 已完成内容

### 2.1 Ubuntu 与 ROS 2 环境配置

本阶段首先完成了 Ubuntu 与 ROS 2 Humble 的基础开发环境配置，主要包括：

- 完成 Ubuntu 系统安装与基本设置；
- 完成 ROS 2 Humble 安装与环境变量配置；
- 完成 `turtlesim` 和 `rqt` 安装；
- 能够正常启动 ROS 2 节点和小乌龟仿真程序；
- 完成 VS Code 及相关开发环境的基础配置；
- 完成 ROS 2 工作空间创建；
- 掌握使用 `colcon build` 编译工作空间；
- 掌握使用 `source install/setup.bash` 加载工作空间环境。



### 2.2 turtlesim 与 rqt 基础操作

使用 turtlesim 完成了 ROS 2 基础通信实验，主要包括：

- 启动 turtlesim 仿真节点；
- 启动键盘控制节点；
- 使用键盘控制小乌龟移动；
- 使用 `rqt_graph` 查看节点和话题之间的通信关系；
- 使用 `rqt_plot` 查看话题数据变化；
- 理解不同节点通过话题等通信机制完成信息传输。

通过这一部分学习，目前对 ROS 2 的基本认识是：

> ROS 2 中不同功能通常由不同节点实现，节点之间通过话题、服务等方式进行通信。

### 2.3 ROS 2 节点、话题、服务和接口

学习了 ROS 2 中节点、话题、服务和接口的基本概念及常用命令。
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


### 2.4 ROS 2 Python 客户端库基础

开始学习 ROS 2 Python 客户端库，并完成以下内容：

- 创建 ROS 2 工作空间；
- 创建 Python 功能包；
- 理解工作空间、功能包和节点之间的关系；
- 了解 Python ROS 2 程序的基本组织方式；
- 理解 `package.xml` 和 `setup.py` 的基本作用；
- 掌握功能包的编译、环境加载和节点运行流程。


### 2.5 Python 发布者与订阅者

完成了 ROS 2 Python 发布者和订阅者程序的编写、编译和运行。

发布者的基本运行流程为：

```text
创建节点
→ 创建发布者
→ 创建定时器
→ 定时器触发回调函数
→ 创建并填写消息对象
→ 发布消息
```


订阅者的基本运行流程为：

```text
创建节点
→ 创建订阅者
→ spin() 等待消息
→ 收到消息后执行回调函数
→ 读取并处理消息
```


通过这一部分学习，理解了：

```text
发布者 → Topic → 订阅者
```

发布者负责发送数据，订阅者负责接收数据，发布者通常不需要等待订阅者返回结果。

### 2.6 Python 服务端与客户端

完成了 ROS 2 Python 服务端和客户端程序的编写、编译和运行。


最终完成的服务通信过程为：

```text
客户端发送两个数字
→ 服务端接收请求
→ 服务端计算两数之和
→ 服务端返回结果
→ 客户端显示结果
```

### 2.7 功能包配置文件

进一步理解了 `package.xml` 和 `setup.py` 的作用。

`package.xml` 主要用于：

- 记录功能包名称和基本信息；
- 声明编译和运行依赖；
- 供 ROS 2 构建工具识别功能包。



`setup.py` 主要用于：

- 配置 Python 功能包；
- 指定 Python 模块；
- 注册可以通过 `ros2 run` 执行的程序入口。


### 2.8 科研记录和代码管理

本阶段建立了科研学习记录和代码版本管理方式，主要包括：

- 建立本地科研学习文件夹；
- 使用 Markdown 记录学习笔记、实验过程和每周进度；
- 安装并配置 Git；
- 创建本地 Git 仓库并完成提交；
- 创建 GitHub 远程仓库；
- 掌握 `git add`、`git commit` 和 `git push` 的基本作用；
- 将 ROS 2 学习笔记和相关记录上传至 GitHub。

当前仓库结构为：

```text
robotics_learning/
├── README.md
├── experiment_logs/
├── weekly_reports/
├── code/
├── note/
└── images/
```

---

## 3. 当前成果

1. ROS 2 Humble 环境可以正常运行；
2. 能够使用 turtlesim 和 rqt 完成基础仿真实验；
3. 能够使用命令行查看节点、话题、服务和接口；
4. 能够创建 ROS 2 工作空间和 Python 功能包；
5. 能够使用 `colcon build` 编译功能包；
6. 能够编写并运行 Python 发布者和订阅者；
7. 能够编写并运行 Python 服务端和客户端；
8. 理解 Topic 和 Service 两种通信方式的主要区别；
9. 能够解释定时器、回调函数、`spin()`、`request`、`response` 和 `future` 的基本作用；
10. 已建立科研实验记录和 GitHub 代码管理框架。

**GitHub 仓库：** https://github.com/jimmmmmyy1124-wq/robotics_learning

---

## 4. 遇到的问题及解决情况

### 4.1 GitHub 连接失败

首次上传 GitHub 时，当前网络无法连接 GitHub 的 443 端口，终端提示：

```text
Recv failure: Connection was reset
```

通过检查网络连接，确认原 Wi-Fi 对 GitHub 连接存在限制。切换手机热点后，443 端口连接恢复正常，并完成代码上传。

### 4.2 ROS 2 概念较多，容易混淆

在学习过程中，节点、话题、服务、客户端和服务端之间的关系容易混淆。

目前的理解是：

- 节点是 ROS 2 程序运行的基本单位；
- 话题用于连续、异步的数据传输；
- 服务用于一次请求和一次响应；
- Client 和 Server 通常是节点内部承担服务通信功能的部分；
- Publisher 和 Subscriber 通常是节点内部承担话题通信功能的部分。

### 4.3 Ubuntu 终端无法使用 `code` 命令

在终端中尝试使用 `code` 命令打开 Python 文件时，系统提示无法找到该命令。

为避免影响学习进度，暂时改用 Ubuntu 自带的 `nano` 编辑器创建和修改文件：

```bash
nano <file_path>
```





### 4.4 对 `package.xml` 和 `setup.py` 的作用容易混淆

目前已明确：

- `package.xml` 主要记录功能包信息和依赖；
- `setup.py` 主要配置 Python 功能包和可执行程序入口；
- 使用 `--dependencies` 创建功能包时，依赖可自动写入 `package.xml`；
- 新增 Python 可执行节点后，仍需要在 `setup.py` 中注册入口。


---

## 5. 当前不足

- 目前仍主要参考教程代码完成程序，独立编写完整节点的能力需要继续练习；
- 尚未学习 Action 和 Launch 文件；
- Ubuntu 中代码编辑工具和开发环境仍需进一步完善；
- 尚未开始规划、感知和 SLAM 算法的系统学习。

---
