# ROS 2 四个考核任务：核心知识与复习笔记

> 工作空间：`~/turtlebot3_ws`  
> 功能包：`tb3_navigation_project`

---

# 0. 先把四个任务串成一条完整链路

四个任务并不是四个孤立的小实验，而是在逐步搭建一套完整的移动机器人自主导航系统。

```text
任务1：搭建 Gazebo 自定义仿真环境
        ↓
产生机器人、障碍物、激光雷达、里程计、TF 等数据
        ↓
任务2：SLAM Toolbox 建图
        ↓
得到并保存 assessment_map.yaml + assessment_map.pgm
        ↓
任务3：加载静态地图 + AMCL 定位 + Nav2 自主导航
        ↓
人工在 RViz2 中发送单个导航目标
        ↓
任务4：Python Action Client
        ↓
程序自动、依次发送多个导航目标
        ↓
TurtleBot3 连续完成多目标点自主导航
```

真正需要掌握的总链路是：

```text
Gazebo 环境
→ 传感器数据
→ ROS 2 Topic / TF
→ SLAM 建图
→ 保存地图
→ Map Server 加载地图
→ AMCL 定位
→ Nav2 Planner 规划路径
→ Controller 输出 /cmd_vel
→ TurtleBot3 运动
→ 传感器继续反馈
→ Python Action Client 自动发送下一个目标
```

如果老师问：

**“这四个任务整体是在做什么？”**

可以回答：

> 我先在 Gazebo 中搭建了一个封闭的 TurtleBot3 仿真环境，然后利用激光雷达、里程计和 TF 通过 SLAM Toolbox 建立二维地图。地图保存后，我使用 Map Server 和 AMCL 在已知地图中定位，再使用 Nav2 完成路径规划和避障。最后我编写了一个 Python ROS 2 节点，通过 `NavigateToPose` Action Client 自动依次发送三个目标点，实现多目标点连续自主导航。

---

# 1. 任务1：Gazebo 自定义仿真环境搭建

## 1.1 我实际完成了什么

任务1中完成了：

- 创建 ROS 2 功能包 `tb3_navigation_project`；
- 创建自定义 Gazebo 世界 `assessment_world.world`；
- 世界尺寸大约为：
  - X：`-5 m ~ 5 m`
  - Y：`-4 m ~ 4 m`
- 使用外围墙体形成封闭环境；
- 使用中间隔墙形成左右两个相互连通的区域；
- 中间保留约 `1.4 m` 宽的通道；
- 放置三个不同尺寸的障碍物：
  - 大型长方体；
  - 细长长方体；
  - 圆柱体；
- 创建 `assessment_world.launch.py`；
- 通过 Launch 启动 Gazebo、自定义 World、机器人状态发布以及 TurtleBot3；
- TurtleBot3 默认生成位置约为：
  - `x = -3.0`
  - `y = -1.5`
- 验证机器人能够移动；
- 验证 `/scan`、`/odom`、`/imu` 等数据正常发布；
- 使用 `ros2 topic hz /scan` 测得激光雷达发布频率约 `4.997 Hz`。

---

## 1.2 Gazebo 到底是什么

Gazebo 是一个**机器人仿真器**。

它主要模拟：

- 机器人模型；
- 墙体和障碍物；
- 重力；
- 碰撞；
- 摩擦；
- 轮子运动；
- 激光雷达；
- 机器人在环境中的真实仿真位置。

一句话：

> Gazebo 模拟的是“物理世界”。

Gazebo 本身不负责：

- SLAM；
- AMCL 定位；
- 路径规划；
- Nav2 自主导航。

这些功能由 ROS 2 中其他节点完成。

---

## 1.3 Gazebo 和 RViz2 的区别

这是非常高概率的提问。

### Gazebo

```text
Gazebo = 模拟真实世界
```

能看到：

- 真实仿真墙体；
- 真实仿真障碍物；
- TurtleBot3；
- 机器人运动；
- 物理碰撞。

### RViz2

```text
RViz2 = 可视化 ROS 2 数据
```

能显示：

- LaserScan；
- Occupancy Grid；
- TF；
- RobotModel；
- Path；
- Costmap；
- AMCL 定位结果。

可以用一句话回答：

> Gazebo 是仿真器，负责产生物理世界和传感器数据；RViz2 是可视化工具，用于显示机器人系统接收到和计算出的 ROS 数据。

---

## 1.4 `.world` 文件是什么

`.world` 文件描述 Gazebo 中的整个仿真场景。

例如：

```text
assessment_world.world
```

里面可以定义：

- 地面；
- 墙；
- 障碍物；
- 光源；
- 圆柱体；
- 静态模型；
- 物理参数。

可以理解为：

```text
World
├── Ground
├── Walls
├── Obstacles
└── Environment settings
```

注意：

> TurtleBot3 不一定要永久写死在 World 文件中，也可以在 Launch 时通过 Spawn 动态生成。

---

## 1.5 `visual` 和 `collision` 的区别

Gazebo 模型中非常重要的两个概念：

### `visual`

决定：

> 这个物体“看起来是什么样”。

例如：

- 形状；
- 颜色；
- 材质。

### `collision`

决定：

> 这个物体“物理上能不能被撞到”。

所以：

```text
有 visual，没有 collision
→ 看得见，但机器人可能直接穿过去

有 collision，没有 visual
→ 看不见，但机器人会撞上
```

老师如果问：

**“为什么墙不仅要有 visual，还要有 collision？”**

回答：

> `visual` 只决定显示效果，不参与物理碰撞。机器人能否真正撞到墙是由 `collision` 决定的，所以用于导航和避障的障碍物必须具有碰撞模型。

---

## 1.6 `static=true` 是什么意思

对于墙、地面和固定障碍物，通常设置为静态模型。

含义：

```text
static = true
```

表示该模型不会被物理碰撞推走。

否则机器人撞墙时，墙可能被当成普通动态刚体处理。

---

## 1.7 ROS 2 功能包结构为什么重要

本项目功能包大致为：

```text
tb3_navigation_project/
├── config/
├── launch/
│   └── assessment_world.launch.py
├── maps/
├── resource/
├── tb3_navigation_project/
│   └── __init__.py
├── worlds/
│   └── assessment_world.world
├── package.xml
├── setup.cfg
└── setup.py
```

需要知道：

### `package.xml`

保存：

- 功能包名称；
- 依赖；
- 版本；
- ROS 2 包元数据。

### `setup.py`

Python ROS 2 包的安装配置。

后续 Python 节点如果希望通过：

```bash
ros2 run 包名 节点名
```

启动，通常需要在 `setup.py` 中配置入口。

### `launch/`

存放启动文件。

### `worlds/`

存放 Gazebo World。

### `maps/`

存放 SLAM 保存的地图。

---

## 1.8 Launch 文件的作用

本项目中的：

```text
assessment_world.launch.py
```

主要负责：

1. 启动 Gazebo `gzserver`；
2. 启动 Gazebo `gzclient`；
3. 加载 `assessment_world.world`；
4. 启动 `robot_state_publisher`；
5. Spawn TurtleBot3。

Launch 的本质：

> 用一个入口统一启动多个 ROS 2 节点、进程和参数。

老师可能问：

**“为什么不一个终端一个终端启动？”**

回答：

> 分开启动也可以，但系统模块很多，Launch 能统一管理启动项和参数，减少人工操作错误，同时更适合工程复现。

---

## 1.9 `robot_state_publisher` 是干什么的

它根据机器人的 URDF/关节状态发布机器人各连杆之间的 TF 关系。

例如：

```text
base_link
   ↓
base_scan
```

RViz2 能够正确显示 RobotModel，很大程度依赖这些 TF。

---

## 1.10 `/cmd_vel`

`/cmd_vel` 是移动机器人最重要的控制话题之一。

消息类型通常为：

```text
geometry_msgs/msg/Twist
```

核心字段：

```text
linear.x
angular.z
```

含义：

```text
linear.x > 0   向前
linear.x < 0   向后

angular.z > 0  左转
angular.z < 0  右转
```

键盘控制和任务2中的 Python 自动探索脚本，本质上最终都要让机器人获得速度指令。

---

## 1.11 `/scan`

`/scan` 是二维激光雷达数据。

消息类型：

```text
sensor_msgs/msg/LaserScan
```

其中最核心的是：

```text
ranges[]
```

表示不同角度方向上测得的障碍物距离。

本项目测得：

```text
/scan ≈ 4.997 Hz
```

任务1为什么要验证 `/scan`？

因为后面：

```text
SLAM
AMCL
Costmap
避障
```

都需要环境感知数据。

所以可以回答：

> `/scan` 是整个项目后续建图、定位和避障的重要传感器输入。

---

## 1.12 `/odom`

`/odom` 是里程计。

它用于估算：

- 机器人移动距离；
- 机器人旋转角度；
- 当前局部位姿。

但里程计会累积误差：

```text
短期：连续、平滑
长期：会漂移
```

因此：

> `/odom` 不是绝对准确的全局位置。

后面 SLAM 和 AMCL 都会利用其他信息对它进行修正。

---

## 1.13 任务1常用命令

编译：

```bash
cd ~/turtlebot3_ws
colcon build --symlink-install --packages-select tb3_navigation_project
source install/setup.bash
```

启动：

```bash
ros2 launch tb3_navigation_project assessment_world.launch.py
```

键盘控制：

```bash
ros2 run turtlebot3_teleop teleop_keyboard
```

检查关键话题：

```bash
ros2 topic list | grep -E '^/(scan|odom|cmd_vel|imu|joint_states)$'
```

检查激光雷达频率：

```bash
ros2 topic hz /scan
```

---

## 1.14 任务1老师可能问什么

### Q1：Gazebo 和 RViz2 有什么区别？

> Gazebo 负责物理仿真，RViz2 负责显示 ROS 2 数据。

### Q2：World 文件干什么？

> 描述 Gazebo 世界中的地面、墙体、障碍物和其他场景元素。

### Q3：`visual` 和 `collision` 有什么区别？

> `visual` 决定外观，`collision` 决定物理碰撞。

### Q4：为什么需要 `/scan`？

> 激光雷达数据是后续 SLAM、AMCL、Costmap 和避障的重要输入。

### Q5：`/cmd_vel` 是什么？

> 机器人速度控制话题，通常传输 `Twist`，主要使用 `linear.x` 和 `angular.z`。

### Q6：Launch 文件为什么重要？

> 它能统一启动多个节点和进程，并统一传递参数，便于重复实验和工程管理。

### Q7：为什么有时 `/spawn_entity` 不可用？

> 可能是 Gazebo 相关服务没有正确启动，或者旧的 Gazebo 进程残留导致系统状态异常。

---

# 2. 任务2：SLAM Toolbox 二维建图

## 2.1 我实际完成了什么

任务2在任务1环境基础上完成：

- 启动 Gazebo + TurtleBot3；
- 启动 SLAM Toolbox；
- 在 RViz2 中查看：
  - Map；
  - LaserScan；
  - RobotModel；
  - TF；
- 使用 Python 控制脚本让机器人自动遍历主体区域；
- 自动脚本多次修改，解决：
  - 靠墙；
  - 卡障碍物；
  - 原地循环；
  - 覆盖不足；
- 对门口、通道、墙角等遗漏区域使用 teleop 手动补图；
- 检查地图：
  - 房间轮廓；
  - 墙体；
  - 通道；
  - 障碍物；
  - 未知区域；
  - 漂移；
  - 鬼影墙；
- 最终保存：
  - `assessment_map.pgm`
  - `assessment_map.yaml`

---

## 2.2 SLAM 是什么

SLAM：

```text
Simultaneous Localization and Mapping
```

中文：

```text
同时定位与建图
```

核心问题是：

```text
地图未知
+
机器人也不知道自己精确在哪里
```

所以 SLAM 同时回答：

```text
我在哪里？
+
环境长什么样？
```

老师如果问：

**“为什么叫同时定位与建图？”**

回答：

> 因为机器人要根据传感器数据一边估计自身位姿，一边利用这个位姿把环境信息拼成地图；而地图又反过来帮助修正机器人位姿，两者是相互依赖的。

---

## 2.3 本项目中 SLAM 的输入

最重要的三个：

```text
/scan
/odom
TF
```

再加上正确的时间同步。

### `/scan`

提供环境障碍物距离。

### `/odom`

提供机器人短时间内连续的运动估计。

### TF

告诉算法：

> 激光雷达、机器人本体、里程计等坐标系之间是什么关系。

信息流：

```text
/scan
   +
/odom
   +
TF
   ↓
SLAM Toolbox
   ↓
机器人位姿估计
   +
Occupancy Grid
```

---

## 2.4 TF 是什么

TF 的作用：

> 描述不同坐标系之间的位置和姿态关系。

本项目必须记住：

```text
map
 ↓
odom
 ↓
base_link
 ↓
laser / base_scan
```

---

## 2.5 `map`、`odom`、`base_link`

### `map`

全局地图坐标系。

特点：

```text
全局稳定
可以被定位算法修正
```

### `odom`

里程计坐标系。

特点：

```text
运动连续
但长期会漂移
```

### `base_link`

机器人本体坐标系。

可以理解为机器人车体的参考坐标系。

---

## 2.6 为什么同时需要 `map` 和 `odom`

这是高概率提问。

`odom`：

- 优点：连续；
- 缺点：长期漂移。

`map`：

- 优点：全局更准确；
- 定位修正时可能发生全局调整。

因此使用：

```text
map → odom → base_link
```

目的：

> 同时获得全局准确性和局部运动连续性。

---

## 2.7 TF 出问题会怎样

可能出现：

- RViz2 无法显示 LaserScan；
- 机器人和地图错位；
- SLAM 地图变形；
- Nav2 transform timeout；
- AMCL 定位异常；
- Planner 无法规划。

所以：

> TF 是 ROS 2 机器人系统最核心的基础设施之一。

---

## 2.8 `use_sim_time:=true`

Gazebo 有自己的仿真时间：

```text
/clock
```

如果 Gazebo 使用仿真时间，而 SLAM 节点使用电脑真实时间，就可能出现：

- TF 时间戳不一致；
- 消息被丢弃；
- SLAM 不更新；
- Nav2 工作异常。

所以任务中使用：

```text
use_sim_time:=true
```

核心意义：

> 让 ROS 2 节点和 Gazebo 使用同一套时间。

---

## 2.9 Occupancy Grid 是什么

SLAM 最终生成的是二维栅格地图。

地图中的每一个栅格表示一小块空间。

通常分为：

```text
Free Space       可通行
Occupied Space   障碍物
Unknown Space    未探索
```

在常见 PGM 显示中大致表现：

```text
白色：可通行区域
黑色：障碍物
灰色：未知区域
```

---

## 2.10 `.pgm` 和 `.yaml` 分别干什么

任务2保存：

```text
assessment_map.pgm
assessment_map.yaml
```

### `.pgm`

保存：

> 地图图像本身。

### `.yaml`

保存：

> ROS 应该如何解释这张地图。

常见内容包括：

```yaml
image: assessment_map.pgm
resolution: ...
origin: ...
negate: ...
occupied_thresh: ...
free_thresh: ...
```

### `resolution`

表示：

> 一个像素对应现实世界多少米。

例如：

```text
0.05 m/pixel
```

表示：

```text
1 pixel = 5 cm
```

### `origin`

用于建立：

```text
地图图片坐标
↔
ROS 世界坐标
```

---

## 2.11 为什么机器人走过了，地图还可能有缺口

这是你实际遇到的问题。

核心原因：

```text
机器人经过某个位置
≠
激光雷达完整观察了附近所有区域
```

可能原因：

- 激光角度不合适；
- 机器人离门口太远；
- 通道遮挡；
- 速度过快；
- 转向过快；
- 轨迹覆盖不充分。

所以门口、墙角出现少量未知区域是可以解释的。

---

## 2.12 什么是“鬼影墙”

表现：

```text
同一堵墙出现两条甚至多条边界
```

通常说明：

> 位姿估计发生漂移，机器人在不同时间对同一物体的估计没有正确重合。

老师可能问：

**“地图质量怎么判断？”**

答：

重点看：

- 墙体是否清晰；
- 房间轮廓是否闭合；
- 通道是否正确；
- 障碍物是否能识别；
- 是否存在大面积未知区域；
- 是否有明显漂移；
- 是否有重复墙/鬼影墙。

---

## 2.13 Python 自动遍历脚本本质是什么

你任务2中的自动脚本本质上是一个：

```text
Reactive Controller
```

基本逻辑：

```text
读取 LaserScan
↓
判断前方障碍物距离
↓
安全 → 向前
危险 → 转向
↓
发布 Twist 到 /cmd_vel
```

简化成：

```python
if front_is_clear:
    move_forward()
else:
    turn()
```

必须知道：

> 这个 Python 脚本不是 SLAM 算法。

它只负责：

```text
让机器人移动
```

真正进行建图的是：

```text
SLAM Toolbox
```

---

## 2.14 为什么简单自动脚本会卡住

因为这种控制器只利用：

```text
当前局部的 LaserScan
```

它通常没有：

- 全局地图；
- 已访问区域记忆；
- 全局路径规划；
- 明确目标；
- Frontier 探索策略。

所以可能：

- 在墙角反复转向；
- 原地旋转；
- 陷入局部循环；
- 漏扫区域。

这也是为什么你最后采用：

```text
自动遍历主体区域
+
人工 teleop 补图
```

---

## 2.15 为什么“自动 + 手动”是合理方案

自动：

- 效率高；
- 可重复；
- 能覆盖主体区域。

手动：

- 可以针对门口；
- 墙角；
- 狭窄区域；
- 地图缺口。

所以任务阶段使用两者结合，比强行让一个简单反应式脚本做到完全自主探索更合理。

---

## 2.16 任务2主要启动命令

启动 Gazebo：

```bash
source /opt/ros/humble/setup.bash
source ~/turtlebot3_ws/install/setup.bash
export TURTLEBOT3_MODEL=burger

ros2 launch tb3_navigation_project assessment_world.launch.py use_sim_time:=true
```

启动 SLAM Toolbox：

```bash
source /opt/ros/humble/setup.bash
source ~/turtlebot3_ws/install/setup.bash

ros2 launch slam_toolbox online_async_launch.py use_sim_time:=true
```

启动 RViz2：

```bash
rviz2 --ros-args -p use_sim_time:=true
```

键盘补图：

```bash
ros2 run turtlebot3_teleop teleop_keyboard
```

---

## 2.17 任务2老师可能问什么

### Q1：SLAM 的输入是什么？

> 主要是激光雷达、里程计和 TF，同时仿真环境中还必须保证时间同步。

### Q2：`/scan` 和 `/odom` 分别干什么？

> `/scan` 提供环境观测，`/odom` 提供机器人短时间连续的运动估计。

### Q3：为什么里程计不能直接当真实位置？

> 因为轮子打滑、模型误差等会使里程计误差不断累积。

### Q4：为什么需要 TF？

> 因为不同传感器数据位于不同坐标系，算法必须知道它们之间的变换关系才能融合。

### Q5：`.pgm` 和 `.yaml` 的区别？

> `.pgm` 是地图图像，`.yaml` 保存分辨率、原点和阈值等地图解释参数。

### Q6：什么是鬼影墙？

> 同一物体在地图中出现重复边界，通常说明位姿估计发生漂移。

### Q7：为什么还要人工补图？

> 简单自动控制只保证机器人运动，不能保证传感器从合适角度覆盖每一个局部区域。

### Q8：自动控制脚本是不是 SLAM？

> 不是。脚本负责控制运动，SLAM Toolbox 才负责建图和位姿估计。

---

# 3. 任务3：基于静态地图的 Nav2 自主导航

## 3.1 我实际完成了什么

任务3完成：

- 加载任务2保存的 `assessment_map.yaml`；
- 启动 Map Server；
- 启动 AMCL；
- 启动 Nav2；
- RViz2 中显示：
  - Map；
  - RobotModel；
  - LaserScan；
  - Costmap；
  - Path；
- 使用 `2D Pose Estimate` 设置初始位姿；
- 使用 `2D Goal Pose` 设置导航目标；
- 在多个不同位置进行导航测试；
- 验证机器人能够穿过房间之间的通道；
- 验证全局路径不会穿墙；
- 在 Gazebo 中增加静态箱体障碍物；
- 验证 Local Costmap 更新和局部避障；
- 排查并解决 Gazebo Pause 导致机器人不动的问题；
- 处理重复打开系统后出现的地图/节点状态异常问题。

---

## 3.2 SLAM 和 Nav2 的区别

必须能秒答。

### SLAM

```text
地图未知
↓
机器人运动
↓
边定位边建图
```

### Nav2

```text
地图已经存在
↓
在地图中定位
↓
规划路径
↓
避障
↓
到达目标
```

一句话：

> SLAM 解决“地图从哪里来”，Nav2 解决“有了地图以后怎么去目标点”。

---

## 3.3 Nav2 不是一个单独算法

Nav2 是一套移动机器人导航软件栈。

核心模块包括：

```text
Map Server
AMCL
Global Costmap
Local Costmap
Planner
Controller
Behavior Tree
Recovery / Behavior
Lifecycle Manager
```

所以老师问：

**“Nav2 是一个路径规划算法吗？”**

答：

> 不是。Nav2 是一套导航框架，其中包含定位、全局规划、局部控制、代价地图、行为树和恢复行为等多个模块。

---

## 3.4 Map Server

作用：

> 把任务2保存的静态地图加载到 ROS 2 中并发布。

输入：

```text
assessment_map.yaml
+
assessment_map.pgm
```

后续 AMCL 和 Planner 才能使用这个地图。

---

## 3.5 AMCL 是什么

AMCL：

```text
Adaptive Monte Carlo Localization
```

中文常称：

```text
自适应蒙特卡洛定位
```

作用：

> 在一张已经知道的地图中估计机器人当前位姿。

主要利用：

```text
静态地图
+
LaserScan
+
Odometry
```

进行定位修正。

现阶段不需要推导粒子滤波数学公式，但要理解：

> AMCL 的任务是“已知地图中的定位”，而不是建图。

---

## 3.6 `2D Pose Estimate`

它告诉 AMCL：

```text
机器人现在大概在哪里
+
机器人现在朝哪个方向
```

也就是：

```text
初始位置 + 初始朝向
```

---

## 3.7 `2D Goal Pose`

它告诉 Nav2：

```text
机器人应该去哪里
+
到达后朝哪个方向
```

所以：

```text
2D Pose Estimate
= 我现在在哪里

2D Goal Pose
= 我要去哪里
```

这是非常高频的口头提问。

---

## 3.8 Nav2 的完整数据链

```text
Goal
 ↓
Behavior Tree
 ↓
Planner
 ↓
Global Path
 ↓
Controller
 ↓
Local Costmap
 ↓
/cmd_vel
 ↓
TurtleBot3
 ↓
/scan + /odom
 ↓
继续反馈和控制
```

导航是：

> 一个持续闭环，而不是只规划一次路线后就不再更新。

---

## 3.9 Planner 是什么

全局规划器负责：

> 根据地图和目标点找到一条从当前位置到目标位置的可行路径。

输入大致包括：

- 当前位姿；
- 目标位姿；
- Global Costmap。

输出：

```text
Global Path
```

一句话：

> Planner 决定“大体走哪条路”。

---

## 3.10 Controller 是什么

Controller 负责：

> 让机器人真正沿着规划路径运动。

它不断生成：

```text
/cmd_vel
```

并结合 Local Costmap 对运动进行实时调整。

一句话：

```text
Planner：决定路线
Controller：决定当前这一刻怎么走
```

---

## 3.11 Costmap 是什么

Costmap：

> 将地图和实时障碍物转换成不同通行代价的二维区域。

可以理解为：

```text
障碍物          不允许走
障碍物附近      代价高
安全区域        代价低
```

Planner 和 Controller 不只是看黑白地图，而是利用 Costmap 判断安全程度。

---

## 3.12 Global Costmap 和 Local Costmap

### Global Costmap

主要用于：

```text
全局路径规划
```

范围通常较大。

### Local Costmap

主要用于：

```text
机器人周围实时避障
```

范围较小并不断滚动更新。

---

## 3.13 Inflation Layer

作用：

> 在障碍物周围形成一圈逐渐降低的高代价区域。

这样机器人不会：

```text
贴着墙走
```

而是倾向于与障碍物保持一定安全距离。

---

## 3.14 为什么地图里没有的新障碍物，机器人也能避开

你在 Gazebo 后加入的箱子并不存在于任务2保存的地图中。

但是：

```text
LaserScan 实时检测新障碍物
↓
Local Costmap 更新
↓
Controller 调整
↓
必要时重新规划
```

所以：

> Nav2 不只依赖历史静态地图，也利用实时传感器数据。

这正是你做静态箱体避障测试的意义。

---

## 3.15 Behavior Tree 是什么

现阶段不需要掌握 Nav2 行为树源码。

知道：

> Behavior Tree 用于组织整个导航任务中的高层执行逻辑。

例如可能管理：

- 规划；
- 跟踪路径；
- 重新规划；
- 清除 Costmap；
- 恢复行为。

---

## 3.16 Lifecycle Node

Nav2 很多节点不是：

```text
启动 = 立即正常工作
```

而是经历：

```text
unconfigured
↓
inactive
↓
active
```

只有真正进入：

```text
active
```

导航功能才正常工作。

所以：

```text
ros2 node list 能看到节点
```

并不代表：

```text
Nav2 一定已经可以导航
```

---

## 3.17 为什么启动顺序重要

系统依赖关系大致为：

```text
Gazebo
↓
机器人和 TF
↓
Map Server
↓
AMCL
↓
Nav2
↓
初始定位
↓
Goal
```

如果上游还没准备好，下游可能出现：

- timeout；
- TF error；
- 地图不显示；
- 不能规划；
- 节点存在但 inactive。

所以后面你做统一启动工具，不只是为了少输入命令，也是为了：

> 管理复杂 ROS 2 系统的启动顺序和残留进程。

---

## 3.18 机器人设置目标后不动，怎么排查

你实际遇到过一次：

> Gazebo 被 Pause。

建议顺序：

### 第一步：Gazebo 是否暂停

如果暂停：

```text
物理仿真停止
```

即使 Nav2 仍在运行，机器人也不会移动。

### 第二步：检查 `/cmd_vel`

```bash
ros2 topic echo /cmd_vel
```

如果有非零速度：

> Nav2 已经在输出控制命令，问题更可能位于底盘/仿真执行侧。

如果没有速度：

> 应继续检查 Nav2 上层状态。

### 第三步：AMCL 是否已经定位

看：

- RobotModel；
- LaserScan；
- 地图墙体是否大致重合。

### 第四步：目标点是否有效

目标不能在：

- 墙内；
- 障碍物内；
- 未知区域；
- Costmap 不可通行区。

### 第五步：检查 TF

重点：

```text
map → odom → base_link
```

### 第六步：Nav2 Lifecycle 是否 Active

---

## 3.19 任务3启动命令

Gazebo：

```bash
source /opt/ros/humble/setup.bash
source ~/turtlebot3_ws/install/setup.bash
export TURTLEBOT3_MODEL=burger

ros2 launch tb3_navigation_project assessment_world.launch.py use_sim_time:=true
```

加载地图并启动 Nav2：

```bash
source /opt/ros/humble/setup.bash
source ~/turtlebot3_ws/install/setup.bash
export TURTLEBOT3_MODEL=burger

ros2 launch turtlebot3_navigation2 navigation2.launch.py \
  use_sim_time:=True \
  map:=/home/jimmy/turtlebot3_ws/src/tb3_navigation_project/maps/assessment_map.yaml
```

---

## 3.20 任务3老师可能问什么

### Q1：SLAM 和 Nav2 区别？

> SLAM 在未知地图中定位并建图；Nav2 在已有地图中定位、规划、避障并到达目标。

### Q2：AMCL 是干什么的？

> 在已知地图中融合地图、激光和里程计估计机器人位姿。

### Q3：2D Pose Estimate 和 2D Goal Pose？

> 前者告诉系统“我在哪里”，后者告诉系统“我要去哪里”。

### Q4：Planner 和 Controller？

> Planner 规划全局路线，Controller 产生实时速度指令让机器人沿路线运动。

### Q5：Global Costmap 和 Local Costmap？

> Global Costmap 主要服务全局规划；Local Costmap 关注机器人附近实时障碍，用于局部控制和避障。

### Q6：为什么机器人不会贴墙走？

> Inflation Layer 会在墙附近设置较高代价。

### Q7：为什么新加入的障碍物不在地图里也能避开？

> 激光雷达实时检测后，Local Costmap 会更新，Controller 根据新障碍调整运动。

### Q8：为什么 `ros2 node list` 有 Nav2 节点却不能导航？

> 节点可能还没有进入 Lifecycle 的 `active` 状态。

### Q9：你遇到机器人不动是怎么解决的？

> 我先排查目标点和导航状态，最后发现 Gazebo 处于 Pause 状态；恢复仿真后机器人继续执行导航。

---

# 4. 任务4：Python 多目标点自主导航节点

## 4.1 老师对任务4的要求

任务4核心要求：

- 使用 Python；
- 使用 `rclpy`；
- 编写 ROS 2 节点；
- 与 Nav2 的 `navigate_to_pose` Action 通信；
- 至少设置三个目标点；
- 目标点按顺序依次执行；
- 输出当前目标和到达状态；
- 有基本错误处理；
- 代码结构清晰。

---

## 4.2 我实际完成了什么

编写：

```text
multi_goal_nav.py
```

核心功能：

```text
Python Node
↓
连接 Nav2 NavigateToPose Action Server
↓
发送目标点1
↓
等待结果
↓
成功后发送目标点2
↓
等待结果
↓
成功后发送目标点3
↓
等待结果
↓
全部完成
```

三个实测目标点：

```text
目标点1：(-1.744,  0.072, 0.0)
目标点2：( 2.027, -0.046, 0.0)
目标点3：( 2.789,  2.863, 0.0)
```

最终三个目标均成功到达。

---

## 4.3 为什么任务4要用 Action，而不是普通 Topic

ROS 2 常见通信方式：

```text
Topic
Service
Action
```

### Topic

适合：

> 连续的数据流。

例如：

```text
/scan
/odom
/cmd_vel
```

特点：

- 发布者持续发布；
- 订阅者持续接收；
- 不强调“任务最终完成”。

### Service

适合：

> 请求一次，快速返回一次结果。

例如：

```text
请求 → 返回
```

通常用于执行时间较短的操作。

### Action

适合：

> 执行时间较长，并且过程中需要反馈，最终还需要结果。

导航任务正好是：

```text
发送导航目标 Goal
↓
机器人持续导航
↓
不断产生 Feedback
↓
最后返回 Result
```

所以：

> `NavigateToPose` 使用 Action 最合理。

---

## 4.4 Action 的三个核心概念

必须记住：

```text
Goal
Feedback
Result
```

### Goal

客户端发送：

> 我要机器人去哪里。

### Feedback

执行过程中返回：

> 任务进行到什么程度。

本节点中可以读取：

```text
distance_remaining
```

也就是距离目标还有多远。

### Result

任务结束时返回：

> 成功、失败、取消等最终状态。

---

## 4.5 Action Client 和 Action Server

在任务4中：

### Python 节点

是：

```text
Action Client
```

它发送目标。

### Nav2

提供：

```text
NavigateToPose Action Server
```

它负责执行导航。

关系：

```text
multi_goal_nav.py
       │
       │ Goal
       ▼
Nav2 NavigateToPose Server
       │
       ├── Feedback
       │
       └── Result
```

---

## 4.6 为什么 Action 名叫 `navigate_to_pose`

因为目标不仅有：

```text
位置 x, y
```

还有：

```text
姿态 orientation
```

也就是：

```text
Pose = Position + Orientation
```

机器人不仅需要到达某个坐标，还可以要求最终朝向某个方向。

---

## 4.7 `PoseStamped` / 目标位姿中的关键内容

导航目标最重要的是：

```text
frame_id
position
orientation
timestamp
```

### `frame_id = 'map'`

表示：

> 目标点坐标是在全局地图坐标系下定义的。

如果坐标本来是地图中的点，就应该使用：

```text
map
```

### `position.x` / `position.y`

目标点平面坐标。

### `orientation`

ROS 中姿态通常使用四元数，而不是直接存 yaw。

---

## 4.8 为什么代码里要把 yaw 转成四元数

二维机器人只需要绕 Z 轴旋转。

如果 yaw 为：

```text
θ
```

常用转换：

```text
qz = sin(θ / 2)
qw = cos(θ / 2)
```

并且：

```text
qx = 0
qy = 0
```

所以代码中会看到：

```python
orientation.z = math.sin(yaw / 2.0)
orientation.w = math.cos(yaw / 2.0)
```

老师如果问：

**“为什么不直接把 yaw 填进 orientation.z？”**

回答：

> 因为 ROS 的姿态字段使用四元数表示，`orientation.z` 只是四元数的一个分量，不是直接的偏航角，因此需要进行转换。

---

## 4.9 为什么要设置时间戳

目标 Pose 中包含：

```text
header.stamp
```

用于标记这个目标消息对应的时间。

代码使用：

```python
self.get_clock().now().to_msg()
```

生成当前 ROS 时钟时间。

仿真系统中使用 `use_sim_time` 时，ROS 时钟可与 Gazebo 仿真时间保持一致。

---

# 4.10 `multi_goal_nav.py` 完整代码逐行注释版

> 说明：当前保存的项目资料中没有单独保留原始 `multi_goal_nav.py` 文件。本节按照你此前实际运行通过版本中已经确认的**类名、函数结构、三个目标点、`NavigateToPose` Action Client、反馈距离、GoalStatus 结果判断以及顺序导航逻辑**进行复原。功能逻辑与实际完成的任务一致；若后续把 Ubuntu 中原始 `.py` 文件上传，可以再做逐字符校对。

```python
import math
# 导入 Python 数学库，后面把二维 yaw 角转换成四元数时需要 sin() 和 cos()。

import rclpy
# 导入 ROS 2 的 Python 客户端库 rclpy。
# 所有 Python ROS 2 节点都需要通过它完成初始化、Spin 和关闭。

from rclpy.node import Node
# 导入 Node 基类。
# 我们自己的 MultiGoalNavigator 会继承 Node，从而成为一个 ROS 2 节点。

from rclpy.action import ActionClient
# 导入 ActionClient。
# 任务4需要主动向 Nav2 发送导航 Goal，因此我们的节点扮演 Action Client。

from action_msgs.msg import GoalStatus
# 导入 Action 的标准状态常量。
# 后面通过 GoalStatus.STATUS_SUCCEEDED 判断导航是否成功。

from nav2_msgs.action import NavigateToPose
# 导入 Nav2 提供的 NavigateToPose Action 接口。
# 这是任务4与 Nav2 进行导航通信的核心接口。


class MultiGoalNavigator(Node):
    # 定义多目标导航节点类。
    # 继承 Node 后，该类就可以使用 ROS 2 的日志、时钟、Action、参数等功能。

    def __init__(self):
        # 构造函数；创建对象时自动执行。

        super().__init__('multi_goal_navigator')
        # 调用 Node 父类构造函数。
        # 'multi_goal_navigator' 是这个 ROS 2 节点在系统中的节点名称。

        self.action_client = ActionClient(
            self,
            NavigateToPose,
            'navigate_to_pose'
        )
        # 创建 Action Client。
        # self：Action Client 属于当前节点。
        # NavigateToPose：Action 的接口类型。
        # 'navigate_to_pose'：Nav2 Action Server 名称。
        # 创建以后，我们就可以通过这个客户端向 Nav2 发送导航目标。

        self.goals = [
            (-1.744, 0.072, 0.0),
            (2.027, -0.046, 0.0),
            (2.789, 2.863, 0.0),
        ]
        # 定义需要依次访问的三个目标点。
        # 每一个元组都是 (x, y, yaw)。
        # x、y 是 map 坐标系中的目标位置。
        # yaw 是机器人最终期望朝向。
        # 本次三个目标的 yaw 都设置为 0.0 rad。


    def create_goal(self, x, y, yaw):
        # 定义一个辅助函数，用 x、y、yaw 构造 Nav2 能够识别的 Goal 消息。

        goal_msg = NavigateToPose.Goal()
        # 创建一个 NavigateToPose 的 Goal 对象。

        goal_msg.pose.header.frame_id = 'map'
        # 指定这个目标位姿属于 map 坐标系。
        # 因为我们的目标坐标是在任务2生成的全局地图上取得的。

        goal_msg.pose.header.stamp = self.get_clock().now().to_msg()
        # 写入当前 ROS 时间戳。
        # get_clock() 获取当前节点时钟；
        # now() 获取当前时间；
        # to_msg() 转换为 ROS 消息格式。

        goal_msg.pose.pose.position.x = x
        # 设置目标点的 x 坐标。

        goal_msg.pose.pose.position.y = y
        # 设置目标点的 y 坐标。

        goal_msg.pose.pose.position.z = 0.0
        # TurtleBot3 在二维平面中导航，因此 z 坐标固定为 0。

        goal_msg.pose.pose.orientation.x = 0.0
        # 二维平面中不考虑绕 X 轴的旋转，因此四元数 x 分量设为 0。

        goal_msg.pose.pose.orientation.y = 0.0
        # 二维平面中不考虑绕 Y 轴的旋转，因此四元数 y 分量设为 0。

        goal_msg.pose.pose.orientation.z = math.sin(yaw / 2.0)
        # 将二维 yaw 角转换为四元数的 z 分量。
        # 对纯 Z 轴旋转：qz = sin(yaw / 2)。

        goal_msg.pose.pose.orientation.w = math.cos(yaw / 2.0)
        # 将二维 yaw 角转换为四元数的 w 分量。
        # 对纯 Z 轴旋转：qw = cos(yaw / 2)。

        return goal_msg
        # 返回已经构造完成的 NavigateToPose Goal。


    def feedback_callback(self, feedback_msg):
        # Action Server 在导航执行过程中会周期性返回 Feedback。
        # 每收到一次反馈，ROS 2 都会调用这个回调函数。

        distance_remaining = feedback_msg.feedback.distance_remaining
        # 从反馈消息中读取 distance_remaining。
        # 表示机器人距离当前目标点还剩多少米。

        self.get_logger().info(
            f'剩余距离：{distance_remaining:.2f} m'
        )
        # 将剩余距离输出到终端。
        # :.2f 表示保留两位小数。


    def navigate_to_goal(self, index, goal):
        # 定义“导航到一个目标点”的完整过程。
        # index：目标点编号，例如 1、2、3。
        # goal：对应的 (x, y, yaw)。

        x, y, yaw = goal
        # 将目标元组拆成 x、y、yaw 三个变量。

        self.get_logger().info(
            f'正在前往目标点 {index}：'
            f'x={x:.3f}, y={y:.3f}, yaw={yaw:.3f}'
        )
        # 在终端输出当前正在执行哪一个目标以及目标坐标。
        # 这也满足老师要求的“记录当前正在前往的目标点”。

        goal_msg = self.create_goal(x, y, yaw)
        # 调用 create_goal()，将普通数字转换成 NavigateToPose Goal。

        self.action_client.wait_for_server()
        # 等待 Nav2 的 navigate_to_pose Action Server 启动。
        # 如果 Nav2 尚未准备好，这里会继续等待，避免直接发送 Goal 导致失败。

        send_goal_future = self.action_client.send_goal_async(
            goal_msg,
            feedback_callback=self.feedback_callback
        )
        # 异步发送导航 Goal。
        # send_goal_async() 不会让底层 ROS 通信线程彻底阻塞。
        # 同时注册 feedback_callback，在导航过程中读取 Feedback。

        rclpy.spin_until_future_complete(self, send_goal_future)
        # 让当前节点持续处理 ROS 回调，
        # 直到“发送 Goal”这一步得到服务器响应。

        goal_handle = send_goal_future.result()
        # 获取 GoalHandle。
        # GoalHandle 表示服务器对这一次 Goal 请求的接收情况。

        if not goal_handle.accepted:
            # 如果 accepted=False，说明 Nav2 拒绝了这个目标。

            self.get_logger().error(
                f'目标点 {index} 请求被拒绝。'
            )
            # 输出错误日志。

            return False
            # 返回 False，告诉上层这个目标没有成功执行。

        result_future = goal_handle.get_result_async()
        # Goal 被接受后，继续异步等待最终导航结果。

        rclpy.spin_until_future_complete(self, result_future)
        # 持续处理 ROS 事件，
        # 直到当前导航任务结束并得到 Result。

        wrapped_result = result_future.result()
        # 读取 Action 的最终结果对象。

        status = wrapped_result.status
        # 读取最终状态码。
        # Action 可能成功、失败、取消等。

        if status == GoalStatus.STATUS_SUCCEEDED:
            # 如果最终状态为 STATUS_SUCCEEDED，
            # 说明 Nav2 判定机器人成功到达目标。

            self.get_logger().info(
                f'✓ 已成功到达目标点 {index}'
            )
            # 输出成功日志。

            return True
            # 返回 True，让 run() 继续发送下一个目标点。

        self.get_logger().error(
            f'目标点 {index} 导航失败，状态码：{status}'
        )
        # 如果不是成功状态，则打印错误信息和状态码。

        return False
        # 返回 False，告诉上层当前目标导航失败。


    def run(self):
        # 定义整个“三目标顺序导航”流程。

        self.get_logger().info(
            '等待 Nav2 navigate_to_pose Action Server...'
        )
        # 启动后先给出提示。

        self.action_client.wait_for_server()
        # 等待 Nav2 Action Server 真正可用。

        self.get_logger().info(
            'Nav2 已连接，开始多目标点自主导航。'
        )
        # Action Server 可用以后输出提示。

        for index, goal in enumerate(self.goals, start=1):
            # 遍历 self.goals 中的三个目标。
            # enumerate(..., start=1) 让编号从 1 开始，而不是 Python 默认的 0。

            success = self.navigate_to_goal(index, goal)
            # 调用 navigate_to_goal()。
            # 只有当前目标执行完成，这个函数才会返回，
            # 因而天然保证目标1、目标2、目标3按顺序执行。

            if not success:
                # 如果当前目标失败：

                self.get_logger().error(
                    f'目标点 {index} 未完成，停止后续导航。'
                )
                # 输出错误提示。

                return
                # 直接退出 run()，
                # 避免在前一个目标失败的情况下继续盲目执行后面的目标。

        self.get_logger().info(
            f'✓ 全部 {len(self.goals)} 个目标点导航完成！'
        )
        # for 循环完整结束，说明所有目标全部成功。
        # len(self.goals) 在这里等于 3。


def main(args=None):
    # ROS 2 Python 程序的主入口函数。

    rclpy.init(args=args)
    # 初始化 ROS 2 Python 客户端库。
    # 没有这一步就不能正常创建 ROS 2 节点。

    node = MultiGoalNavigator()
    # 创建我们自己定义的多目标导航节点对象。

    try:
        # 使用 try/finally，确保即使中间发生异常，
        # 最后仍然可以执行节点销毁和 ROS 关闭操作。

        node.run()
        # 启动三个目标点的连续导航流程。

    finally:
        # 无论 node.run() 正常完成还是出现异常，
        # finally 中代码都会执行。

        node.destroy_node()
        # 销毁 ROS 2 节点，释放节点资源。

        rclpy.shutdown()
        # 关闭 rclpy，结束 ROS 2 Python 客户端。


if __name__ == '__main__':
    # Python 标准入口判断。
    # 只有直接运行这个文件时，下面的 main() 才会执行。

    main()
    # 调用主函数，正式启动程序。
```

---

## 4.11 这段代码最重要的执行逻辑

不要死背代码，先理解：

```text
main()
↓
rclpy.init()
↓
MultiGoalNavigator()
↓
创建 ActionClient
↓
保存三个目标点
↓
run()
↓
目标1 → navigate_to_goal()
↓
成功
↓
目标2 → navigate_to_goal()
↓
成功
↓
目标3 → navigate_to_goal()
↓
成功
↓
全部完成
```

---

## 4.12 为什么它能保证“按顺序”执行

关键在：

```python
for index, goal in enumerate(self.goals, start=1):
    success = self.navigate_to_goal(index, goal)
```

而 `navigate_to_goal()` 内部：

```python
发送 Goal
↓
等待 Goal 被接受
↓
等待当前导航 Result
↓
函数 return
```

所以：

```text
目标1没有结束
→ for 循环不会进入目标2
```

这就是顺序导航的保证。

---

## 4.13 `send_goal_async()` 为什么叫 async

`async`：

```text
asynchronous
```

即异步。

它的设计目的：

> 发送请求以后，不强制把整个 ROS 2 执行线程完全锁死，可以继续处理 Action 回调和 ROS 通信。

但是本程序为了实现：

```text
目标1完成以后再发目标2
```

又使用：

```python
rclpy.spin_until_future_complete(...)
```

等待对应 Future 完成。

所以它是：

> 底层用异步 Action API，但程序控制逻辑按顺序等待结果。

---

## 4.14 Future 是什么

可以把 Future 理解为：

> “一个现在还没有结果，但未来会得到结果的对象”。

例如：

```python
send_goal_future = self.action_client.send_goal_async(...)
```

刚执行时：

```text
Nav2 可能还没回复
```

等未来服务器回复以后：

```python
send_goal_future.result()
```

才能得到 GoalHandle。

---

## 4.15 GoalHandle 是什么

GoalHandle 表示：

> Action Server 对当前 Goal 的控制句柄。

最重要的是：

```python
goal_handle.accepted
```

判断：

```text
Nav2 有没有接受这个目标
```

如果 Goal 都没被接受，就没有必要继续等待导航结果。

---

## 4.16 `get_result_async()` 是什么

Goal 被接受只表示：

```text
“我接这个任务”
```

不表示：

```text
“我已经到达目标”
```

所以之后还要：

```python
goal_handle.get_result_async()
```

等待导航最终完成。

---

## 4.17 `GoalStatus.STATUS_SUCCEEDED`

最终 Action 有一个状态码。

代码：

```python
if status == GoalStatus.STATUS_SUCCEEDED:
```

用于判断：

> 当前导航任务最终是否成功完成。

注意：

```text
Goal accepted
≠
Goal succeeded
```

这是非常容易被老师追问的点。

---

## 4.18 为什么导航失败后停止后续目标

代码中：

```python
if not success:
    return
```

这是基本的错误处理。

原因：

如果目标2都没有到达，却继续执行目标3：

- 路径起点和预期不一致；
- 无法说明任务按要求完成；
- 出错原因可能被掩盖。

所以当前版本选择：

> 一旦某个中间目标失败，就停止后续任务并输出错误日志。

---

## 4.19 为什么目标点可以硬编码

老师原要求允许：

```text
硬编码
或者
YAML / 配置文件
```

本任务只有三个固定测试点，因此直接放进：

```python
self.goals = [...]
```

简单、直观、方便验证。

如果以后需要：

- 很多目标；
- 经常更改目标；
- 多套路线；

再把目标放进 YAML 会更合适。

---

## 4.20 为什么使用 `map` 坐标系

任务4目标点来自任务2保存的地图。

所以：

```python
goal_msg.pose.header.frame_id = 'map'
```

表示：

> `x`、`y` 是全局地图坐标。

如果误用成 `base_link`：

> 这些数字会被理解为相对于机器人当前位置的坐标，语义完全不同。

---

## 4.21 任务4老师最可能问的问题

### Q1：为什么使用 Action，而不是 Topic？

> 导航持续时间较长，需要知道目标是否接受、执行中的反馈以及最终成功/失败结果，所以比 Topic 更适合用 Action。

### Q2：Action 的三个核心部分是什么？

> Goal、Feedback、Result。

### Q3：你这个节点谁是 Client，谁是 Server？

> `multi_goal_nav.py` 是 Action Client，Nav2 的 `NavigateToPose` 是 Action Server。

### Q4：`wait_for_server()` 干什么？

> 等待 Nav2 的 Action Server 准备完成，避免在服务端还不存在时发送目标。

### Q5：`send_goal_async()` 干什么？

> 异步向 Nav2 发送一个导航 Goal。

### Q6：`goal_handle.accepted` 是不是说明机器人到了？

> 不是。它只说明服务器接受了任务，之后还要等待 `get_result_async()` 得到最终执行结果。

### Q7：如何判断导航成功？

> 读取最终 Action 状态，并与 `GoalStatus.STATUS_SUCCEEDED` 比较。

### Q8：为什么使用 `map` frame？

> 因为目标点坐标是在全局静态地图上定义的。

### Q9：为什么 orientation 要用四元数？

> ROS Pose 的姿态字段使用四元数，而不是直接存 yaw。

### Q10：为什么是 `sin(yaw/2)` 和 `cos(yaw/2)`？

> 因为二维移动机器人只绕 Z 轴旋转，纯 yaw 旋转对应四元数可写成 `qz=sin(yaw/2)`、`qw=cos(yaw/2)`。

### Q11：怎么保证三个目标依次执行？

> `navigate_to_goal()` 会等待当前目标得到最终 Result 才返回，for 循环得到返回值后才进入下一个目标。

### Q12：如果一个目标失败怎么办？

> 当前版本输出失败状态并停止后续导航，属于基本错误处理。

### Q13：Feedback 有什么作用？

> 可以在导航执行过程中得到实时状态，例如 `distance_remaining`，便于观察任务进度和调试。

### Q14：为什么目标点不放 YAML？

> 老师允许硬编码或 YAML。本项目只有三个固定验证点，硬编码更直接；目标多或需要频繁修改时更适合 YAML。

### Q15：你这个节点和 RViz 的 2D Goal Pose 本质区别是什么？

> 两者最终都是给 Nav2 提交导航目标；RViz 是人工点击一次发送一个目标，而任务4是 Python 节点自动按程序顺序发送多个目标。

---

# 5. ROS 2 Topic / Service / Action 总结

这四个任务做完以后，至少要能区分：

## Topic

```text
连续数据流
```

例：

```text
/scan
/odom
/cmd_vel
```

典型模式：

```text
Publisher → Topic → Subscriber
```

---

## Service

```text
一次请求
→
一次响应
```

适合短时间操作。

---

## Action

```text
Goal
↓
长时间执行
↓
Feedback
↓
Result
```

任务4：

```text
NavigateToPose
```

就是 Action。

一句话：

> Topic 适合持续数据，Service 适合快速请求-响应，Action 适合耗时任务并支持反馈和最终结果。

---

# 6. 四个任务中最重要的 ROS 2 数据接口

至少认识：

```text
/scan
/odom
/cmd_vel
/map
/tf
/tf_static
/clock
/amcl_pose
```

核心作用：

```text
/scan
= 环境感知

/odom
= 局部运动估计

/tf
= 坐标系变换

/map
= 全局地图

/cmd_vel
= 机器人速度控制

/clock
= Gazebo 仿真时间

/amcl_pose
= AMCL 定位结果
```

---

# 7. 四个任务中最重要的“谁产生什么”

```text
Gazebo
→ 模拟环境、机器人、传感器和物理运动

Laser Plugin
→ /scan

机器人底盘 / 仿真插件
→ /odom

robot_state_publisher 等
→ TF

SLAM Toolbox
→ /map + 建图过程中的全局位姿关系

Map Server
→ 加载并发布静态地图

AMCL
→ 已知地图中的机器人定位

Planner
→ Global Path

Controller
→ /cmd_vel

RViz2
→ 数据可视化 + 人工发送初始位姿/目标

multi_goal_nav.py
→ 自动向 NavigateToPose Action Server 依次发送目标

Nav2
→ 接收 Goal，完成规划、控制和避障
```

---

# 8. 答辩时最需要避免的几个概念错误

## 错误1：说“Gazebo 做 SLAM”

错误。

正确：

> Gazebo 只提供仿真环境和传感器数据，SLAM Toolbox 才负责 SLAM。

---

## 错误2：说“RViz2 是仿真软件”

错误。

正确：

> RViz2 是 ROS 数据可视化工具。

---

## 错误3：说“自动探索 Python 脚本负责建图”

错误。

正确：

> Python 脚本控制机器人移动，SLAM Toolbox 根据传感器数据完成建图。

---

## 错误4：把 `2D Pose Estimate` 当目标点

错误。

正确：

```text
2D Pose Estimate = 初始定位
2D Goal Pose     = 导航目标
```

---

## 错误5：把 AMCL 和 SLAM 混在一起

错误。

正确：

```text
SLAM
= 未知地图下定位 + 建图

AMCL
= 已知地图下定位
```

---

## 错误6：认为 Planner 直接控制电机

错误。

正确：

```text
Planner → 路径
Controller → /cmd_vel
机器人 → 执行速度
```

---

## 错误7：认为 Goal 被 accepted 就代表成功到达

错误。

正确：

```text
accepted
= Nav2 接受任务

STATUS_SUCCEEDED
= 导航最终成功
```

---

# 9. 老师如果让你从头解释整个项目，可以这样回答

> 我的项目基于 ROS 2 Humble、Gazebo Classic 和 TurtleBot3 Burger。首先我创建了一个自定义封闭 Gazebo World，包括两个相互连通的区域和三个不同尺寸的障碍物，并验证 TurtleBot3、激光雷达和里程计能够正常工作。  
>
> 第二步我使用 SLAM Toolbox，通过 `/scan`、`/odom` 和 TF 在机器人运动过程中建立二维 Occupancy Grid。主体区域先用 Python 自动控制脚本遍历，再用 teleop 对门口和墙角等缺失区域补图，最后保存为 `assessment_map.yaml` 和 `assessment_map.pgm`。  
>
> 第三步我加载保存的静态地图，使用 AMCL 完成机器人在地图中的定位，然后通过 Nav2 的 Planner、Controller 和 Costmap 完成自主路径规划和避障。我还在 Gazebo 中加入新的障碍物验证 Local Costmap 的实时避障能力。  
>
> 最后我用 `rclpy` 编写 `multi_goal_nav.py`，创建 `NavigateToPose` Action Client，把三个预设目标点依次发送给 Nav2。程序等待每一个目标完成并检查 `GoalStatus`，当前目标成功后才发送下一个，最终三个目标全部成功到达。

---

# 10. 一页速记版

```text
Gazebo
= 物理仿真

RViz2
= ROS 数据可视化

World
= 仿真场景

visual
= 看起来什么样

collision
= 物理上能不能撞到

/cmd_vel
= 速度控制

/scan
= 激光雷达

/odom
= 里程计，连续但会漂移

TF
= 坐标系关系

map
= 全局地图坐标系

odom
= 局部连续里程计坐标系

base_link
= 机器人本体坐标系

SLAM
= 未知地图下同时定位与建图

SLAM Toolbox
= 本项目建图模块

Occupancy Grid
= 二维占据栅格地图

.pgm
= 地图图像

.yaml
= 地图解释参数

Map Server
= 加载静态地图

AMCL
= 已知地图中的定位

2D Pose Estimate
= 我在哪里

2D Goal Pose
= 我要去哪里

Nav2
= ROS 2 移动机器人导航框架

Planner
= 规划全局路线

Controller
= 实时控制机器人运动

Global Costmap
= 全局规划

Local Costmap
= 局部实时避障

Inflation Layer
= 障碍物周围提高代价，避免贴墙

Lifecycle
= Nav2 节点状态管理

Topic
= 连续数据

Service
= 请求-响应

Action
= Goal + Feedback + Result

NavigateToPose
= Nav2 导航 Action

Action Client
= multi_goal_nav.py

Action Server
= Nav2

goal_handle.accepted
= Nav2 接受任务，不代表已经到达

GoalStatus.STATUS_SUCCEEDED
= 最终导航成功

map frame
= 目标坐标采用全局地图坐标

yaw → quaternion
= qz=sin(yaw/2), qw=cos(yaw/2)

任务4顺序执行
= 当前 Goal 得到 Result 后才发送下一个
```

---

# 11. 最后真正需要掌握到什么程度

不要求现阶段推导：

- SLAM 位姿图优化全部数学公式；
- AMCL 粒子滤波完整概率推导；
- Nav2 Planner 源码；
- DWB/MPPI 控制器完整目标函数；
- Costmap 每一个插件参数；
- Gazebo 物理引擎内部实现。

但是一定要做到：

```text
知道每个模块是什么
↓
知道它接收什么数据
↓
知道它输出什么数据
↓
知道它在整个系统中处于什么位置
↓
知道出故障时先检查什么
↓
能够解释自己写的任务4 Python 代码
```

最核心的闭环：

```text
仿真环境产生数据
→ ROS 2 传输
→ SLAM 建图
→ 保存地图
→ AMCL 定位
→ Nav2 规划与控制
→ /cmd_vel
→ 机器人运动
→ 传感器反馈
→ Python Action Client 自动发送下一目标
```

如果这一条链能不看资料完整讲出来，再把任务4代码中的：

```text
ActionClient
NavigateToPose
Goal
Feedback
Result
Future
GoalHandle
GoalStatus
map frame
quaternion
```

全部解释清楚，基本就足以应对老师围绕这四个任务的大部分提问。
