# ROS 2 Python 发布者与订阅者笔记

> 本笔记整理 ROS 2 Python 发布者与订阅者学习中的三个重点：发布者的一般结构、订阅者的一般结构、关键函数与 `String` 消息类型。

---

## 1. 发布者的一般结构

发布者节点负责通过某个话题向 ROS 2 系统发送消息。

### 1.1 通用代码结构

```python
import rclpy
from rclpy.node import Node
from 消息包.msg import 消息类型                     #导入ROS2库


class 节点类名(Node):

    def __init__(self):
        super().__init__('节点名称')

        self.publisher_ = self.create_publisher(    #创建发布者
            消息类型,
            '话题名称',
            10
        )

        self.timer = self.create_timer(             #创建定时器
            时间间隔,
            self.timer_callback
        )

    def timer_callback(self):                       #回调函数
        msg = 消息类型()        # 创建消息
        msg.数据字段 = 数据     # 写入消息内容

        self.publisher_.publish(msg)    # 发布消息


def main(args=None):
    rclpy.init(args=args)

    node = 节点类名()
    rclpy.spin(node)

    node.destroy_node()   #销毁节点并释放资源
    rclpy.shutdown()      #关闭 ROS 2 Python 客户端库


if __name__ == '__main__':
    main()
```

### 1.2 发布流程

```text
创建节点
→ 创建发布者
→ 创建定时器
→ 定时器触发回调函数
→ 创建消息对象
→ 给消息字段赋值
→ publish() 发布消息
```
---

## 2. 订阅者的一般结构

订阅者节点负责监听某个话题。当该话题出现新消息时，ROS 2 会自动调用订阅回调函数。

### 2.1 通用代码结构

```python
import rclpy
from rclpy.node import Node
from 消息包.msg import 消息类型         #导入ROS2库


class 节点类名(Node):

    def __init__(self):
        super().__init__('节点名称')

        self.subscription = self.create_subscription(       #创建订阅者
            消息类型,
            '话题名称',
            self.listener_callback,
            10
        )

    def listener_callback(self, msg):               #订阅回调函数
        data = msg.数据字段


def main(args=None):
    rclpy.init(args=args)

    node = 节点类名()
    rclpy.spin(node)    #让节点持续运行并处理回调函数

    node.destroy_node() #销毁节点并释放资源
    rclpy.shutdown()    #关闭 ROS 2 Python 客户端库


if __name__ == '__main__':
    main()
```

### 2.2 订阅流程

```text
创建节点
→ 创建订阅者
→ spin() 等待消息
→ 收到新消息
→ 自动调用订阅回调函数
→ 从消息对象中读取数据
```

---


## 3. 其他知识点

### 3.1 `rclpy.spin()` 的作用

对于发布者：

```text
spin()
→ 等待定时器到期
→ 执行定时器回调函数
→ 发布消息
→ 继续等待
```

对于订阅者：

```text
spin()
→ 等待新消息
→ 执行订阅回调函数
→ 处理消息
→ 继续等待
```

因此，`spin()` 可以理解为：

> 让节点进入持续运行状态，并不断处理定时器、订阅等事件产生的回调函数。

---

### 3.2 `String` 消息类型

`String` 是 ROS 2 标准消息包 `std_msgs` 中定义的字符串消息类型。

#### 3.2.1 导入消息类型

```python
from std_msgs.msg import String
```

注意：这里的 `String` 是 ROS 2 消息类型，不是普通的 Python 字符串类型 `str`。

#### 3.2.2 创建消息对象

```python
msg = String()
```

#### 3.2.3 给消息赋值

```python
msg.data = 'Hello ROS 2'
```

`String` 消息中的实际字符串内容存放在：

```python
msg.data
```

#### 3.2.4 发布 `String` 消息

```python
msg = String()
msg.data = 'Hello ROS 2'
self.publisher_.publish(msg)
```

不能直接发布普通 Python 字符串：

```python
self.publisher_.publish('Hello ROS 2')  # 错误
```

原因是 `publish()` 需要接收 ROS 2 消息对象，而不是普通的 Python 数据。

#### 3.2.5 读取 `String` 消息

```python
def listener_callback(self, msg):
    self.get_logger().info(msg.data)
```

这里的 `msg.data` 就是发布者发送的字符串内容。

---

