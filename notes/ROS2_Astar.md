# 附加任务2：A* 路径规划与 Nav2 集成核心知识笔记

## 1. 任务主线

附加任务2的完整链路：

```text
RViz2 Publish Point
        ↓
/clicked_point
        ↓
自定义 astar_planner.py
        ↓
读取 /map + TF 获取机器人当前位置
        ↓
OccupancyGrid 转二维栅格
        ↓
障碍物膨胀 + 软代价
        ↓
自定义 A* 搜索
        ↓
生成 nav_msgs/Path
      ↙            ↘
/astar_path      FollowPath Action
    ↓                ↓
 RViz2        Nav2 Controller Server
                       ↓
                 DWB Local Planner
                       ↓
                    /cmd_vel
                       ↓
                  TurtleBot3
```

必须能口头解释：

> 我自己实现的是全局路径规划部分。A* 根据地图、起点和目标点生成完整路径；Nav2 的 DWB Controller 负责让 TurtleBot3 实际沿这条路径运动。

---

## 2. A* 最核心公式

```text
f(n) = g(n) + h(n)
```

- `g(n)`：从起点到当前节点已经产生的真实累计代价。
- `h(n)`：当前节点到目标点的估计代价。
- `f(n)`：A* 用来决定优先搜索哪个节点的总代价。

本项目的启发式函数使用欧氏距离：

```text
h(n) = sqrt((x_goal-x)^2 + (y_goal-y)^2)
```

与 Dijkstra 的区别：

```text
Dijkstra：主要看 g(n)
A*：同时看 g(n) 和 h(n)
```

所以 A* 一般会更有方向性地朝目标搜索。

---

## 3. 为什么使用 8 邻域

```python
self.motions = [
    (1, 0, 1.0),
    (-1, 0, 1.0),
    (0, 1, 1.0),
    (0, -1, 1.0),

    (1, 1, math.sqrt(2)),
    (1, -1, math.sqrt(2)),
    (-1, 1, math.sqrt(2)),
    (-1, -1, math.sqrt(2))
]
```

其中每项表示：

```text
(dx, dy, move_cost)
```

横向、纵向移动距离为 `1`，斜向移动距离为 `sqrt(2)`。

8 邻域相比 4 邻域能够生成更自然、更接近真实二维距离的路径。

---

# 4. A* 核心代码与逐段注释

## 4.1 启发式函数

```python
def heuristic(self, current, goal):

    # 当前节点与目标节点在 x 方向上的距离差
    dx = goal[0] - current[0]

    # 当前节点与目标节点在 y 方向上的距离差
    dy = goal[1] - current[1]

    # 欧氏距离，作为 A* 的 h(n)
    return math.sqrt(dx * dx + dy * dy)
```

必须理解：

> `h(n)` 并不表示已经走过的距离，而是“从当前位置到目标还需要多少代价”的估计。

---

## 4.2 判断节点能不能走

```python
def is_valid(self, node, grid):

    x, y = node

    height = len(grid)
    width = len(grid[0])

    # 越过左右边界
    if x < 0 or x >= width:
        return False

    # 越过上下边界
    if y < 0 or y >= height:
        return False

    # 本项目二维地图中：
    # 1 = 障碍物
    if grid[y][x] == 1:
        return False

    # 不越界而且不是障碍物
    return True
```

本项目约定：

```text
0 = 可通行
1 = 障碍物
```

---

## 4.3 防止斜穿障碍物墙角

```python
def is_diagonal_valid(self, current, neighbor, grid):

    dx = neighbor[0] - current[0]
    dy = neighbor[1] - current[1]

    # 如果不是斜向移动，不需要额外检查
    if abs(dx) != 1 or abs(dy) != 1:
        return True

    # 斜向移动时的两个侧边格
    side_1 = (
        current[0] + dx,
        current[1]
    )

    side_2 = (
        current[0],
        current[1] + dy
    )

    # 任何一个侧边格不可通行
    # 都禁止此次斜向移动
    if not self.is_valid(side_1, grid):
        return False

    if not self.is_valid(side_2, grid):
        return False

    return True
```

为什么需要它？

如果只有 8 邻域而不检查墙角，路径可能出现：

```text
■ □
□ ■
```

并直接从两个障碍物之间的角点斜穿。

真实 TurtleBot3 有实体尺寸，因此这种路径不合理。

---

## 4.4 A* 主循环——最重要

```python
def plan(self, grid, start, goal, cost_grid=None):

    # open_list：
    # 等待被搜索的候选节点。
    # heapq 让 f(n) 更小的节点优先被取出。
    open_list = []

    # 起点 g=0，因此起点 f=h
    start_f = self.heuristic(start, goal)

    # 起点加入候选队列
    heapq.heappush(
        open_list,
        (start_f, start)
    )

    # came_from：
    # 记录每个节点最优路径上的父节点。
    # 最终用它恢复完整路径。
    came_from = {}

    # g_score：
    # 保存从起点到每个节点的当前最小真实代价。
    g_score = {
        start: 0.0
    }

    # 已经完成搜索的节点
    closed_set = set()

    # 只要还有候选节点就继续搜索
    while open_list:

        # 取出当前 f(n) 最小的节点
        _, current = heapq.heappop(open_list)

        # 已经处理过就跳过
        if current in closed_set:
            continue

        # 搜索到了目标点
        if current == goal:
            return self.reconstruct_path(
                came_from,
                current
            )

        # 当前节点标记为已搜索
        closed_set.add(current)

        # 遍历当前节点周围的 8 个方向
        for dx, dy, move_cost in self.motions:

            neighbor = (
                current[0] + dx,
                current[1] + dy
            )

            # 越界或障碍物不能走
            if not self.is_valid(neighbor, grid):
                continue

            # 禁止斜穿墙角
            if not self.is_diagonal_valid(
                current,
                neighbor,
                grid
            ):
                continue

            # 已完成搜索的节点不重复处理
            if neighbor in closed_set:
                continue

            # 默认没有额外安全代价
            extra_cost = 0.0

            # 如果有障碍物软代价图，
            # 越靠近障碍物，extra_cost 越高。
            if cost_grid is not None:
                extra_cost = cost_grid[
                    neighbor[1]
                ][
                    neighbor[0]
                ]

            # 假设从 current 走到 neighbor 后，
            # 起点到 neighbor 的新总代价
            tentative_g = (
                g_score[current]
                + move_cost
                + extra_cost
            )

            # 如果以前没走过 neighbor，
            # 默认旧代价为无穷大。
            old_g = g_score.get(
                neighbor,
                float('inf')
            )

            # 新路线更优
            if tentative_g < old_g:

                # 记录父节点
                came_from[neighbor] = current

                # 更新 g(n)
                g_score[neighbor] = tentative_g

                # A* 核心：f = g + h
                f_score = (
                    tentative_g
                    + self.heuristic(
                        neighbor,
                        goal
                    )
                )

                # 加入优先队列
                heapq.heappush(
                    open_list,
                    (f_score, neighbor)
                )

    # 所有候选节点都搜完仍没到终点
    return None
```

这段必须掌握四个核心变量：

| 变量 | 含义 |
|---|---|
| `open_list` | 还没搜索完、等待选择的候选节点 |
| `closed_set` | 已经正式搜索过的节点 |
| `g_score` | 起点到当前节点的最优真实累计代价 |
| `came_from` | 当前节点的最优父节点，用于恢复路径 |

一句话记忆：

```text
open_list：接下来搜谁
closed_set：谁已经搜过
g_score：走到这里花了多少
came_from：我是从哪里来的
```

---

## 4.5 路径恢复

```python
def reconstruct_path(self, came_from, current):

    # current 此时就是 goal
    path = [current]

    # 不断沿父节点往起点回溯
    while current in came_from:
        current = came_from[current]
        path.append(current)

    # 当前顺序：
    # goal → ... → start
    # 反转后变成正确顺序
    path.reverse()

    # start → ... → goal
    return path
```

A* 搜索过程中并不是每一步都直接保存一整条路线，而是保存：

```text
每个节点的父节点
```

最终到达目标后再回溯得到完整路径。

---

# 5. 本项目的 A* 不是最基础版本：加入了软安全代价

基础 A*：

```text
g_new = g_current + move_cost
```

本项目：

```text
g_new
=
g_current
+
move_cost
+
extra_cost
```

`extra_cost` 来自障碍物软代价图。

所以两条路线长度差不多时：

> A* 会更偏向距离墙体和障碍物更远的路线。

这并没有改变 A* 的核心结构，只是重新定义了真实路径代价 `g(n)`。

---

# 6. 硬膨胀和软代价的区别

最终自定义规划代码：

```python
inflated_grid = self.inflate_grid(
    grid,
    radius=3
)
```

地图分辨率：

```text
0.05 m/cell
```

因此：

```text
3 × 0.05 = 0.15 m
```

## 硬膨胀

```text
障碍物周围约 0.15 m
直接判为不可通行
```

作用：给机器人预留最低安全距离。

## 软代价

更外围区域：

```text
可以走
但越靠近障碍物代价越大
```

作用：让路径在有空间时主动远离障碍物。

必须会区分：

```text
硬膨胀 = 不能走
软代价 = 可以走，但不鼓励走
```

---

# 7. 为什么代码里还有 Dijkstra

最终的 `build_clearance_cost_grid()` 使用了多源 Dijkstra。

但它不是最终导航路径规划器。

正确理解：

```text
多源 Dijkstra
↓
计算每个栅格到最近障碍物的大致距离
↓
生成 soft cost

A*
↓
根据 start、goal 和 soft cost
生成最终导航路径
```

所以老师问：

> 你到底用的是 A* 还是 Dijkstra？

回答：

> 最终从起点到目标点的全局路径由 A* 生成；Dijkstra 只用于辅助建立障碍物距离软代价场。

---

# 8. ROS OccupancyGrid 如何变成 A* 地图

ROS 地图类型：

```text
nav_msgs/msg/OccupancyGrid
```

本项目转换规则：

```python
if occupancy >= 0 and occupancy < 50:
    row.append(0)
else:
    row.append(1)
```

即：

```text
0 ≤ occupancy < 50
→ 可通行 0

occupancy ≥ 50
→ 障碍物 1

occupancy < 0
→ Unknown
→ 本项目同样当障碍物 1
```

把 Unknown 当障碍物的原因：

> 未知区域没有被确认安全，因此不让 A* 主动规划进去。

---

# 9. 世界坐标和栅格坐标转换

## 世界坐标 → 栅格

```python
grid_x = int(
    (world_x - origin_x) / resolution
)

grid_y = int(
    (world_y - origin_y) / resolution
)
```

含义：

```text
减去地图原点
↓
得到相对地图原点的距离
↓
除以每格的长度
↓
得到第几个栅格
```

## 栅格 → 世界坐标

```python
world_x = (
    origin_x
    + (grid_x + 0.5) * resolution
)

world_y = (
    origin_y
    + (grid_y + 0.5) * resolution
)
```

为什么 `+0.5`？

> 为了取这个栅格的中心，而不是栅格边界。

---

# 10. A* 起点和终点分别怎么得到

## 起点：TF

查询：

```text
map → base_footprint
```

获得机器人在 `map` 坐标系中的实时位置。

再转换为：

```text
start = (grid_x, grid_y)
```

## 终点：RViz Publish Point

RViz 的：

```text
Publish Point
```

发布：

```text
/clicked_point
```

消息类型：

```text
geometry_msgs/msg/PointStamped
```

节点收到目标世界坐标后再转成：

```text
goal = (grid_x, grid_y)
```

注意：

> 本任务不能直接用 Nav2 Goal，因为目标必须先进入自己的 A* 算法。

---

# 11. A* 路径如何变成 ROS Path

A* 得到的是：

```text
[(x0,y0), (x1,y1), ..., (xn,yn)]
```

Nav2 需要：

```text
nav_msgs/msg/Path
```

而 `Path` 内部是一组：

```text
geometry_msgs/msg/PoseStamped
```

结构：

```text
Path
└── poses[]
    ├── PoseStamped
    ├── PoseStamped
    └── ...
```

程序还根据相邻路径点方向计算：

```python
yaw = math.atan2(
    next_world_y - world_y,
    next_world_x - world_x
)
```

再转换为四元数，使每个路径点的朝向与实际前进方向一致。

---

# 12. /astar_path 是什么

```python
self.path_pub = self.create_publisher(
    Path,
    '/astar_path',
    10
)
```

作用：

> 发布自定义 A* 路径。

RViz 中添加：

```text
Path
Topic = /astar_path
```

就能看到绿色路线。

但是必须知道：

> `/astar_path` 只是显示和传输 Path，本身不会控制机器人运动。

真正让机器人运动的是 `FollowPath`。

---

# 13. FollowPath 必须掌握

```python
self.follow_path_client = ActionClient(
    self,
    FollowPath,
    'follow_path'
)
```

规划完成后：

```python
goal_msg = FollowPath.Goal()

goal_msg.path = path_msg
goal_msg.controller_id = 'FollowPath'
goal_msg.goal_checker_id = 'general_goal_checker'
```

然后：

```python
send_goal_async(...)
```

含义：

```text
自己的 A*
↓
已经算好完整 Path
↓
FollowPath
↓
Nav2 Controller
↓
沿这条 Path 实际控制机器人
```

---

# 14. NavigateToPose 和 FollowPath 的区别

以前多目标导航任务使用：

```text
NavigateToPose
```

意思：

> 只把目标点给 Nav2，路径由 Nav2 自己规划。

本任务使用：

```text
FollowPath
```

意思：

> 路径已经由自己的 A* 算好了，只需要 Nav2 Controller 按这条路径执行。

一句话记住：

```text
NavigateToPose：给目标
FollowPath：给完整路径
```

---

# 15. A* 和 DWB 分别负责什么

必须避免说：

> A* 控制机器人运动。

正确关系：

```text
A*
= 全局规划
= 算路线

DWB
= 局部控制
= 根据路径和局部障碍计算速度

/cmd_vel
= 最终发给 TurtleBot3 的速度指令
```

所以：

```text
A*
↓
Path
↓
DWB
↓
/cmd_vel
↓
TurtleBot3
```

---

# 16. 为什么使用 Action

`FollowPath` 是 Action。

Action 适合：

> 持续时间比较长，并且需要 Goal、Feedback、Result 的任务。

```text
Goal
→ 发送完整路径

Feedback
→ 执行过程中持续反馈

Result
→ 最终成功、取消或失败
```

导航正是典型 Action 场景。

---

# 17. 为什么要有 goal_sequence

异步 Action 可能出现：

```text
任务 #1 还没完全返回
↓
已经开始任务 #2
↓
任务 #1 的旧 Result 晚一点回来
```

所以代码用：

```python
self.goal_sequence += 1
self.active_goal_sequence = goal_sequence
```

并在 Feedback / Result 中检查：

```python
if goal_sequence != self.active_goal_sequence:
    return
```

作用：

> 旧导航任务的反馈和结果不会干扰当前任务。

---

# 18. 为什么还要自己计算“是否真正到达”

项目中通过 TF 获取：

```text
robot_x, robot_y
```

并与最终目标计算：

```python
distance = math.hypot(
    goal_x - robot_x,
    goal_y - robot_y
)
```

最终使用：

```text
0.10 m
```

作为位置到达阈值，并连续满足 3 次后确认。

意义：

> 不完全依赖 Action Result，而是增加一个基于真实 TF 位置的独立到达确认。

---

# 19. 自动重新规划逻辑

如果 FollowPath 在真正到达前异常结束：

```text
重新读取机器人当前位置
↓
目标点保持不变
↓
重新 A*
↓
重新发布 Path
↓
重新 FollowPath
```

项目中自动重规划最大：

```text
2 次
```

作用：

> 提高异常情况下的容错能力，同时避免无限重试。

---

# 20. Goal Checker 和 Progress Checker

最终专用 Nav2 参数中：

```yaml
progress_checker:
  plugin: "nav2_controller::SimpleProgressChecker"
  required_movement_radius: 0.15
  movement_time_allowance: 15.0

general_goal_checker:
  stateful: False
  plugin: "nav2_controller::SimpleGoalChecker"
  xy_goal_tolerance: 0.10
  yaw_goal_tolerance: 3.2
```

## Progress Checker

判断：

> 机器人是不是长时间没有取得运动进展。

本项目：

```text
15 秒内需要产生约 0.15 m 的位移进展
```

## Goal Checker

判断：

> 是否到达最终位置。

```text
xy_goal_tolerance = 0.10 m
```

约等于允许 10 cm 位置误差。

```text
yaw_goal_tolerance = 3.2 rad
```

设置很宽松。

原因：

> Publish Point 只给位置，不给严格的最终朝向。

---

# 21. use_sim_time：本任务最重要的调试知识之一

调试过程中出现过：

```text
Transform data too old when converting from map to odom
```

原因是：

```text
A* Path 的时间戳
和
Gazebo / Nav2 / TF 的时间
不一致
```

Gazebo 使用：

```text
/clock
```

因此自定义 A* 节点也要使用：

```text
use_sim_time:=true
```

必须掌握：

> TF 坐标变换不仅要求 frame 对，还要求消息时间戳和 TF 时间处在相同时间体系。

所以：

```text
坐标系正确
但时间戳错误
```

同样可能导致导航异常。

---

# 22. 这个任务中各 ROS 接口

| 名称 | 类型 | 作用 |
|---|---|---|
| `/map` | Topic | OccupancyGrid 静态地图 |
| `/clicked_point` | Topic | RViz Publish Point 目标 |
| `/astar_path` | Topic | 自定义 A* Path |
| `map → base_footprint` | TF | 机器人实时地图位姿 |
| `follow_path` | Action | 把完整路径交给 Nav2 Controller |
| `/cmd_vel` | Topic | 最终底盘速度指令 |

---

# 23. A* 与 Dijkstra 必须会比较

## Dijkstra

主要根据：

```text
g(n)
```

搜索。

特点：

> 不知道目标在哪个方向，通常会比较均匀地向周围扩展。

## A*

根据：

```text
f(n)=g(n)+h(n)
```

多了一个指向目标的启发信息。

特点：

> 通常更有方向性地朝目标搜索，从而减少无意义扩展。

---

# 24. 为什么用 heapq

A* 每次都要寻找：

```text
f(n) 最小的候选节点
```

所以使用：

```python
heapq.heappush(...)
heapq.heappop(...)
```

构造最小堆。

因此：

```text
open_list
本质是优先队列
```

---

# 25. 必须能完整口述程序流程

```text
1. astar_planner 启动
2. 订阅 /map
3. RViz 完成 AMCL 初始定位
4. Publish Point 点击目标
5. /clicked_point 获得目标坐标
6. TF 获得机器人当前位置
7. 起点和终点转成栅格
8. OccupancyGrid 转 0/1 地图
9. 障碍物硬膨胀
10. 建立软代价图
11. A* 执行 f=g+h 搜索
12. came_from 回溯恢复路径
13. 路径转成 nav_msgs/Path
14. 发布 /astar_path
15. FollowPath 发送给 Nav2
16. DWB 计算 /cmd_vel
17. TurtleBot3 实际运动
18. TF 持续计算距离
19. 达到阈值后确认完成
```

---

# 26. 老师最可能问的问题

### Q1：A* 最核心公式？

```text
f(n)=g(n)+h(n)
```

---

### Q2：你这里 h(n) 用什么？

欧氏距离。

---

### Q3：open_list 是什么？

等待搜索的候选节点优先队列，优先取 `f(n)` 最小的节点。

---

### Q4：closed_set 是什么？

已经完成搜索的节点集合，避免重复处理。

---

### Q5：came_from 有什么用？

记录父节点，到达目标后从终点回溯恢复完整路径。

---

### Q6：为什么用 8 邻域？

允许斜向移动，路径更自然；但需要额外防止斜穿墙角。

---

### Q7：为什么 A* 算完机器人不会自动动？

因为 A* 只得到数学路径，仍要转换为 ROS Path，再通过 FollowPath 交给 Nav2 Controller。

---

### Q8：为什么不用 NavigateToPose？

因为本任务已经自己生成了完整路径，因此使用 FollowPath；NavigateToPose 是只给目标点，让 Nav2 自己规划。

---

### Q9：A* 和 DWB 怎么分工？

A* 负责全局路径；DWB 负责跟踪路径并生成 `/cmd_vel`。

---

### Q10：为什么代码里又有 Dijkstra？

Dijkstra 只用于建立障碍物软代价场，真正起点到终点的全局路径还是 A*。

---

### Q11：硬膨胀和软代价区别？

```text
硬膨胀：不能走
软代价：能走，但越靠障碍越不划算
```

---

### Q12：为什么要用 TF？

获取机器人在地图坐标系中的实时位置，用作 A* 起点和最终到达判定。

---

### Q13：为什么 use_sim_time 很重要？

Gazebo、Nav2、TF、自定义 Path 的时间戳必须统一，否则会产生 TF 时间转换错误。

---

# 27. 最少必须背熟的 10 条

1. A* 核心：`f(n)=g(n)+h(n)`。
2. `g(n)` 是真实累计代价，`h(n)` 是到目标的估计代价。
3. 本项目 `h(n)` 使用欧氏距离。
4. `open_list` 是优先队列，`closed_set` 是已搜索集合。
5. `came_from` 用于从 goal 回溯得到完整路径。
6. 本项目使用 8 邻域，并禁止斜穿墙角。
7. OccupancyGrid 被转换成 0/1 的 A* 二维栅格。
8. A* 只负责生成 Path，DWB 负责生成 `/cmd_vel`。
9. 自定义完整路径通过 Nav2 `FollowPath` 执行，而不是 `NavigateToPose`。
10. Gazebo、Nav2、TF 和 A* Path 必须使用一致的仿真时间。

---

# 28. 30 秒答辩版本

> 本任务中我使用 Python 自主实现了基于二维 OccupancyGrid 的 A* 全局路径规划器。算法采用 8 邻域搜索和欧氏距离启发函数，通过 f=g+h 选择优先扩展节点，同时加入斜向穿角限制、障碍物硬膨胀和软安全代价。机器人当前位置通过 TF 获取，RViz 的 Publish Point 作为目标输入。规划结果被转换为 nav_msgs/Path 并发布到 /astar_path，随后通过 Nav2 的 FollowPath Action 交给 DWB Controller 实际跟踪，实现了自定义 A* 与 Nav2 导航栈的完整集成。
