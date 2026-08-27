# ROS2 RGB-D Vision Notes

> 附加任务3核心知识笔记  
> 目标：掌握 TurtleBot3 + RGB-D Camera + ROS 2 + OpenCV 的视觉感知主线，能够独立解释并复现“红色目标检测 + 深度测距”。

---

# 1. 本任务完成的核心链路

```text
Gazebo RGB-D Camera
        ↓
/camera/image_raw
/camera/depth/image_raw
        ↓
ROS 2 Python Node
        ↓
cv_bridge
        ↓
OpenCV
        ↓
HSV 红色分割
        ↓
形态学处理
        ↓
轮廓检测
        ↓
Bounding Box + Center
        ↓
目标中心附近深度
        ↓
Distance
        ↓
/red_detection/image
        ↓
rqt_image_view
```

最终显示：

```text
RED OBJECT
Distance: 2.65 m
Center: (342, 181)
```

必须理解的本质：

> RGB 图像负责“找到目标”，Depth 图像负责“测量目标距离”。

---

# 2. RGB-D Camera 是什么

RGB-D Camera 同时输出：

```text
RGB 彩色图像
+
Depth 深度图像
```

普通 RGB 相机主要回答：

```text
目标在哪里？
```

RGB-D 相机还能回答：

```text
目标距离相机多远？
```

所以本任务不是单纯图像识别，而是：

```text
二维目标检测
+
深度测距
```

---

# 3. 本任务必须掌握的 ROS 2 Topic

启动 RGB-D 版本后：

```bash
ros2 topic list | grep -Ei "camera|image|depth|rgb"
```

实际得到：

```text
/camera/camera_info
/camera/depth/camera_info
/camera/depth/image_raw
/camera/image_raw
/camera/points
```

## `/camera/image_raw`

消息类型：

```text
sensor_msgs/msg/Image
```

作用：

```text
RGB 彩色图像
```

本任务中用于红色目标检测。

## `/camera/depth/image_raw`

消息类型：

```text
sensor_msgs/msg/Image
```

作用：

```text
深度图像
```

本任务中用于目标距离估计。

## `/camera/camera_info`

保存相机内参，例如：

```text
fx, fy
cx, cy
畸变参数
```

当前任务没有直接使用，但以后做：

```text
像素坐标 → 三维坐标
```

时需要。

## `/camera/points`

通常是：

```text
sensor_msgs/msg/PointCloud2
```

表示三维点云。本任务没有直接使用。

---

# 4. `cv_bridge` 是什么

ROS 2 相机发布的是：

```text
sensor_msgs/msg/Image
```

OpenCV 处理的是：

```text
NumPy ndarray
```

因此需要：

```text
cv_bridge
```

完成：

```text
ROS Image
   ↓
cv_bridge
   ↓
OpenCV Image
```

核心代码：

```python
from cv_bridge import CvBridge

self.bridge = CvBridge()
```

ROS Image 转 OpenCV：

```python
frame = self.bridge.imgmsg_to_cv2(
    msg,
    desired_encoding='bgr8'
)
```

OpenCV 转回 ROS Image：

```python
result_msg = self.bridge.cv2_to_imgmsg(
    frame,
    encoding='bgr8'
)
```

一句话记忆：

> `cv_bridge` 是 ROS 图像消息和 OpenCV 图像之间的桥梁。

---

# 5. 为什么是 BGR

OpenCV 默认颜色通道顺序不是 RGB，而是：

```text
BGR
```

因此：

```python
desired_encoding='bgr8'
```

表示：

```text
Blue + Green + Red
每通道 8 bit
```

---

# 6. 为什么颜色检测使用 HSV

直接用 RGB/BGR 判断颜色容易受到亮度变化影响。

HSV：

```text
H = Hue        色相
S = Saturation 饱和度
V = Value      亮度
```

其中：

```text
H → 是什么颜色
S → 颜色有多纯
V → 有多亮
```

所以颜色检测常用：

```text
BGR
 ↓
HSV
 ↓
阈值分割
```

核心代码：

```python
hsv = cv2.cvtColor(
    frame,
    cv2.COLOR_BGR2HSV
)
```

---

# 7. OpenCV 中 HSV 的范围

OpenCV 中：

```text
H: 0 ~ 179
S: 0 ~ 255
V: 0 ~ 255
```

注意：

> OpenCV 中 H 不是 0~360，而是 0~179。

---

# 8. 为什么红色需要两个 HSV 区间

红色位于 Hue 色相环首尾两端。

本任务使用：

```python
lower_red1 = np.array([0, 100, 40])
upper_red1 = np.array([10, 255, 255])

lower_red2 = np.array([170, 100, 40])
upper_red2 = np.array([179, 255, 255])
```

也就是：

```text
H: 0~10
或者
H: 170~179
```

因为 Gazebo 中的目标是偏深红色，所以：

```text
V 下限 = 40
```

不能设得太高。

---

# 9. `cv2.inRange()` 的作用

```python
mask1 = cv2.inRange(
    hsv,
    lower_red1,
    upper_red1
)
```

它会生成二值 Mask：

```text
满足阈值 → 255（白）
不满足   → 0（黑）
```

所以：

```text
白色 = 候选红色目标
黑色 = 背景
```

两个红色区间合并：

```python
mask = cv2.bitwise_or(
    mask1,
    mask2
)
```

---

# 10. 为什么要做形态学处理

颜色阈值分割后可能有：

```text
零散白色噪声
目标内部小黑洞
边缘断裂
```

本任务使用：

```text
Opening
+
Closing
```

结构元素：

```python
kernel = np.ones(
    (5, 5),
    np.uint8
)
```

---

# 11. Opening 开运算

```python
mask = cv2.morphologyEx(
    mask,
    cv2.MORPH_OPEN,
    kernel
)
```

作用：

```text
去除小型噪声
```

简单记：

> OPEN = 去掉外面的“小白点”。

---

# 12. Closing 闭运算

```python
mask = cv2.morphologyEx(
    mask,
    cv2.MORPH_CLOSE,
    kernel
)
```

作用：

```text
填补目标内部小孔洞
连接小范围断裂
```

简单记：

> CLOSE = 填补里面的“小黑洞”。

---

# 13. 什么是轮廓 Contour

目标经过颜色分割后形成白色区域。

轮廓就是：

```text
白色区域的边界
```

代码：

```python
contours, _ = cv2.findContours(
    mask,
    cv2.RETR_EXTERNAL,
    cv2.CHAIN_APPROX_SIMPLE
)
```

其中：

```text
RETR_EXTERNAL
```

表示只提取最外层轮廓。

---

# 14. 为什么选择最大轮廓

场景中可能存在：

```text
小红点
颜色噪声
主红色障碍物
```

因此选择：

```python
largest_contour = max(
    contours,
    key=cv2.contourArea
)
```

即：

```text
面积最大的红色区域
```

再增加面积阈值：

```python
if area > 1500:
```

排除小噪声。

所以实际判断逻辑是：

```text
颜色满足
+
面积足够大
=
认为检测到目标
```

---

# 15. Bounding Box 是什么

Bounding Box：

```text
目标外接矩形
```

代码：

```python
x, y, w, h = cv2.boundingRect(
    largest_contour
)
```

其中：

```text
x, y = 左上角
w    = 宽度
h    = 高度
```

右下角：

```text
(x + w, y + h)
```

绘制：

```python
cv2.rectangle(
    frame,
    (x, y),
    (x + w, y + h),
    (0, 255, 0),
    3
)
```

---

# 16. 目标中心怎么计算

```python
cx = x + w // 2
cy = y + h // 2
```

本次最终示例：

```text
Center: (342, 181)
```

目标中心不仅用于显示，更重要的是：

```text
RGB 图中获得 (cx, cy)
        ↓
去 Depth 图查对应位置
        ↓
获得目标距离
```

所以：

> `(cx, cy)` 是 RGB 检测和 Depth 测距之间的连接点。

---

# 17. OpenCV 图像坐标系

```text
(0,0) ─────────→ x
  │
  │
  ↓
  y
```

因此：

```text
x 向右增加
y 向下增加
```

和普通数学坐标系不同。

---

# 18. `32FC1` 是什么意思

检查深度图：

```bash
ros2 topic echo /camera/depth/image_raw --once --field encoding
```

实际结果：

```text
32FC1
```

拆开理解：

```text
32F = 32-bit Floating Point
C1  = 1 Channel
```

即：

> 每个深度像素是一个单通道 32 位浮点数。

当前 Gazebo RGB-D 相机中，可以按米处理。

例如：

```text
2.65
```

表示约：

```text
2.65 m
```

---

# 19. 为什么不直接读取一个深度像素

最简单：

```python
distance = depth_image[cy, cx]
```

但单个深度像素可能出现：

```text
NaN
Inf
0
局部异常值
```

所以不够稳定。

---

# 20. 本任务如何做稳定深度估计

不读取单点，而是读取目标中心附近：

```text
7 × 7
```

区域。

```python
radius = 3

x1 = max(cx - radius, 0)
x2 = min(cx + radius + 1, width)

y1 = max(cy - radius, 0)
y2 = min(cy + radius + 1, height)
```

因为：

```text
3 + 中心 + 3 = 7
```

所以是 `7×7`。

---

# 21. 无效深度过滤

```python
valid_depths = depth_region[
    np.isfinite(depth_region) &
    (depth_region > 0.0)
]
```

其中：

```python
np.isfinite()
```

排除：

```text
NaN
Inf
```

同时：

```text
depth > 0
```

排除：

```text
0 或负值
```

---

# 22. 为什么使用 Median 中位数

```python
distance = float(
    np.median(valid_depths)
)
```

假设：

```text
2.60
2.61
2.62
2.63
10.00
```

平均数会受到 `10.00` 影响。

中位数对异常值更鲁棒。

所以：

> 目标中心附近有效深度的中位数，比单点或简单平均更稳定。

---

# 23. 视觉检测核心代码

```python
# ROS Image → OpenCV
frame = self.bridge.imgmsg_to_cv2(
    msg,
    desired_encoding='bgr8'
)

# BGR → HSV
hsv = cv2.cvtColor(
    frame,
    cv2.COLOR_BGR2HSV
)

# 红色两个区间
lower_red1 = np.array([0, 100, 40])
upper_red1 = np.array([10, 255, 255])

lower_red2 = np.array([170, 100, 40])
upper_red2 = np.array([179, 255, 255])

# 二值分割
mask1 = cv2.inRange(
    hsv,
    lower_red1,
    upper_red1
)

mask2 = cv2.inRange(
    hsv,
    lower_red2,
    upper_red2
)

mask = cv2.bitwise_or(
    mask1,
    mask2
)

# 形态学去噪
kernel = np.ones((5, 5), np.uint8)

mask = cv2.morphologyEx(
    mask,
    cv2.MORPH_OPEN,
    kernel
)

mask = cv2.morphologyEx(
    mask,
    cv2.MORPH_CLOSE,
    kernel
)

# 找轮廓
contours, _ = cv2.findContours(
    mask,
    cv2.RETR_EXTERNAL,
    cv2.CHAIN_APPROX_SIMPLE
)

if contours:

    largest_contour = max(
        contours,
        key=cv2.contourArea
    )

    area = cv2.contourArea(
        largest_contour
    )

    if area > 1500:

        x, y, w, h = cv2.boundingRect(
            largest_contour
        )

        cx = x + w // 2
        cy = y + h // 2
```

必须理解这条主线：

```text
Image
↓
HSV
↓
Mask
↓
Morphology
↓
Contour
↓
Bounding Box
↓
Center
```

---

# 24. 深度测距核心代码

```python
def get_distance(self, cx, cy):

    if self.depth_image is None:
        return None

    height, width = self.depth_image.shape

    if cx < 0 or cx >= width:
        return None

    if cy < 0 or cy >= height:
        return None

    radius = 3

    x1 = max(cx - radius, 0)
    x2 = min(cx + radius + 1, width)

    y1 = max(cy - radius, 0)
    y2 = min(cy + radius + 1, height)

    depth_region = self.depth_image[
        y1:y2,
        x1:x2
    ]

    valid_depths = depth_region[
        np.isfinite(depth_region) &
        (depth_region > 0.0)
    ]

    if valid_depths.size == 0:
        return None

    distance = float(
        np.median(valid_depths)
    )

    return distance
```

逻辑：

```text
目标中心
   ↓
取周围 7×7
   ↓
过滤无效值
   ↓
取中位数
   ↓
得到距离
```

---

# 25. RGB 和 Depth 两个回调如何配合

节点有两个订阅：

```text
RGB Subscriber
Depth Subscriber
```

Depth 回调：

```python
def depth_callback(self, msg):
    self.depth_image = ...
```

作用：

```text
不断保存最新深度图
```

RGB 回调：

```python
def image_callback(self, msg):
```

作用：

```text
检测目标
↓
得到 (cx, cy)
↓
调用 get_distance(cx, cy)
↓
读取最新深度图
```

当前实现属于：

```text
最新 RGB
+
最近一帧 Depth
```

并没有严格做时间同步。

如果以后机器人或目标运动很快，可进一步使用：

```text
message_filters
ApproximateTimeSynchronizer
```

这是进阶内容。

---

# 26. Sensor QoS

相机属于高频传感器数据。

代码使用：

```python
from rclpy.qos import qos_profile_sensor_data
```

订阅：

```python
self.create_subscription(
    Image,
    '/camera/image_raw',
    self.image_callback,
    qos_profile_sensor_data
)
```

原因：

```text
相机数据频率高
数据量大
实时性优先
```

---

# 27. 为什么检测结果要重新发布 ROS Topic

不是只使用：

```text
cv2.imshow()
```

而是发布：

```text
/red_detection/image
```

这样检测结果仍在 ROS 2 数据流中。

其他节点可以继续：

```text
订阅
显示
记录
处理
```

这更符合 ROS 系统设计。

---

# 28. `rqt_image_view` 的作用

启动：

```bash
ros2 run rqt_image_view rqt_image_view
```

查看原始图像：

```text
/camera/image_raw
```

查看检测结果：

```text
/red_detection/image
```

所以可以直观看到：

```text
原始图像
vs
检测后图像
```

---

# 29. 必须掌握的检查命令

查看相机相关 Topic：

```bash
ros2 topic list | grep -Ei "camera|image|depth|rgb"
```

查看深度编码：

```bash
ros2 topic echo /camera/depth/image_raw --once --field encoding
```

结果：

```text
32FC1
```

查看相机图像：

```bash
ros2 run rqt_image_view rqt_image_view
```

当前检测节点测试运行：

```bash
cd ~/turtlebot3_ws
source /opt/ros/humble/setup.bash

python3 src/tb3_navigation_project/tb3_navigation_project/red_object_detector.py
```

---

# 33. 当前算法的优点

HSV + Contour：

```text
简单
速度快
计算量低
不需要训练
不需要模型权重
容易解释
适合仿真验证
```

所以非常适合当前：

```text
颜色明显的 Gazebo 红色障碍物
```

---

# 34. 当前算法的局限

HSV 颜色检测不是通用语义目标检测。

它依赖：

```text
颜色特征
```

所以在以下情况容易失败：

```text
光照变化
背景存在相同颜色
目标颜色变化
需要识别多类别
真实复杂环境
```

---

# 35. HSV 和 YOLO 的区别

HSV：

```text
规则型方法
根据颜色阈值识别区域
```

它知道：

```text
“这是红色区域”
```

但并不知道它一定是：

```text
箱子
汽车
人
```

YOLO：

```text
学习型目标检测
```

可以根据训练得到的视觉特征识别语义类别。

一句话：

> HSV 识别颜色特征，YOLO 识别学习到的目标类别。

---

# 36. 为什么本任务用 HSV 就足够

题目要求：

```text
部署简单的目标检测算法，展示结果
```

当前场景：

```text
红色目标
+
灰色墙面
+
灰色地面
```

颜色区分明显。

所以：

```text
HSV
+
Morphology
+
Contour
```

已经满足任务要求，而且稳定、易解释。

---

# 37. 这次任务最关键的 10 个知识点

1. `sensor_msgs/Image` 是 ROS 相机图像消息；
2. `cv_bridge` 用于 ROS Image 和 OpenCV 图像互转；
3. OpenCV 默认颜色顺序是 BGR；
4. HSV 更适合颜色阈值检测；
5. OpenCV 中 H 范围是 `0~179`；
6. 红色通常需要两个 Hue 区间；
7. Opening 去小噪声，Closing 填小孔洞；
8. Contour 提取目标边界，Bounding Box 框出目标；
9. RGB 中的 `(cx, cy)` 可用于查询对应 Depth；
10. `32FC1` 深度图可以通过中心附近有效深度中位数得到稳定距离。

---

# 38. 必须能自己画出的流程图

```text
/camera/image_raw
        ↓
cv_bridge
        ↓
BGR
        ↓
HSV
        ↓
Red Threshold
        ↓
Binary Mask
        ↓
Opening + Closing
        ↓
Contours
        ↓
Largest Contour
        ↓
Bounding Box
        ↓
Center (cx, cy)
        │
        ↓
/camera/depth/image_raw
        ↓
7×7 Depth Region
        ↓
Filter Invalid Values
        ↓
Median
        ↓
Distance
        ↓
/red_detection/image
```

如果这条流程可以独立解释，附加任务3的主线就掌握了。

---

# 39. 老师可能问的问题

## Q1：为什么用 HSV 而不是 RGB？

答：

HSV 将色相、饱和度和亮度部分分离，在颜色阈值检测中比直接使用 RGB/BGR 更方便、更稳定，因此适合提取颜色明显的红色目标。

## Q2：为什么红色需要两个 HSV 范围？

答：

OpenCV Hue 范围为 0~179，红色位于色相环首尾两侧，所以需要同时检测 0~10 和 170~179 两个区间。

## Q3：Opening 和 Closing 分别做什么？

答：

Opening 主要去除零散小噪声；Closing 主要填补目标区域中的小孔洞并连接小范围断裂。

## Q4：为什么选择最大轮廓？

答：

颜色分割后可能存在小噪声，因此选面积最大的红色轮廓作为主要目标，并通过面积阈值进一步排除小噪声。

## Q5：Bounding Box 怎么得到？

答：

使用 `cv2.boundingRect()` 对目标轮廓求外接矩形，得到 `(x, y, w, h)`。

## Q6：目标中心怎么计算？

答：

```text
cx = x + w/2
cy = y + h/2
```

程序使用整数除法。

## Q7：`32FC1` 是什么意思？

答：

表示单通道 32 位浮点图像，本任务中每个像素对应一个深度值。

## Q8：为什么不用中心单个像素的深度？

答：

单点可能出现 NaN、Inf 或局部异常值，因此取中心附近 `7×7` 区域，过滤无效值后取中位数，更稳定。

## Q9：RGB 和 Depth 怎么关联？

答：

先在 RGB 图中检测目标并得到中心 `(cx, cy)`，然后去深度图中读取该位置附近的有效深度，从而获得目标距离。

## Q10：当前算法有什么局限？

答：

它依赖颜色特征，对复杂光照、相同颜色背景以及多类别语义识别的适应能力有限，更复杂场景应考虑 YOLO 等学习型方法。

---

# 40. 最后一页速记

```text
RGB-D
=
RGB + Depth

RGB Topic
=
/camera/image_raw

Depth Topic
=
/camera/depth/image_raw

ROS Image ↔ OpenCV
=
cv_bridge

颜色检测
=
BGR → HSV → inRange

红色 HSV
=
0~10
+
170~179

去噪
=
Opening + Closing

找目标
=
findContours

最大目标
=
max(contours, key=cv2.contourArea)

目标框
=
boundingRect

目标中心
=
(cx, cy)

Depth Encoding
=
32FC1

稳定测距
=
中心附近 7×7
→ 去 NaN / Inf / <=0
→ Median

输出 Topic
=
/red_detection/image
```

---

# 41. 一句话总结附加任务3

> 使用 RGB 图像通过 HSV 颜色分割和轮廓检测定位红色目标，再利用目标中心像素在 RGB-D 深度图中读取稳定深度，实现目标检测、二维定位和距离估计，并将检测结果作为 ROS 2 图像话题实时发布。
