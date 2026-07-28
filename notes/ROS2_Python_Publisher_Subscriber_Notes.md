# ROS 2 Python 发布者与订阅者笔记

> 本笔记整理 ROS 2 Python 发布者与订阅者学习中的三个重点：发布者的一般结构、订阅者的一般结构、关键函数与 `String` 消息类型。

---

## 1. 发布者的一般结构

发布者节点负责通过某个话题向 ROS 2 系统发送消息。

### 1.1 通用代码结构

```python
import rclpy
from rclpy.node import Node
from 消息包.msg import 消息类型


class 节点类名(Node):

    def __init__(self):
        super().__init__('节点名称')

        self.publisher_ = self.create_publisher(
            消息类型,
            '话题名称',
            10
        )

        self.timer = self.create_timer(
            时间间隔,
            self.timer_callback
        )

    def timer_callback(self):
        msg = 消息类型()
        msg.数据字段 = 数据

        self.publisher_.publish(msg)


def main(args=None):
    rclpy.init(args=args)

    node = 节点类名()
    rclpy.spin(node)

    node.destroy_node()
    rclpy.shutdown()


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

### 1.3 创建发布者的一般形式

```python
self.publisher_ = self.create_publisher(
    消息类型,
    '话题名称',
    队列深度
)
```

三个参数分别表示：

| 参数 | 作用 |
|---|---|
| `消息类型` | 发布的数据类型 |
| `'话题名称'` | 消息发送到哪个话题 |
| `队列深度` | 暂时来不及处理时可保留的消息数量 |

### 1.4 创建定时器的一般形式

```python
self.timer = self.create_timer(
    时间间隔,
    回调函数
)
```

例如：

```python
self.timer = self.create_timer(
    0.5,
    self.timer_callback
)
```

含义：每隔 `0.5` 秒执行一次 `timer_callback()`。

注意，传入的是函数本身：

```python
self.timer_callback
```

而不是立即调用函数：

```python
self.timer_callback()  # 错误写法
```

### 1.5 发布消息的一般形式

```python
msg = 消息类型()
msg.数据字段 = 数据
self.publisher_.publish(msg)
```

核心过程：

```text
创建消息对象 → 填写消息内容 → 发布消息
```

---

## 2. 订阅者的一般结构

订阅者节点负责监听某个话题。当该话题出现新消息时，ROS 2 会自动调用订阅回调函数。

### 2.1 通用代码结构

```python
import rclpy
from rclpy.node import Node
from 消息包.msg import 消息类型


class 节点类名(Node):

    def __init__(self):
        super().__init__('节点名称')

        self.subscription = self.create_subscription(
            消息类型,
            '话题名称',
            self.listener_callback,
            10
        )

    def listener_callback(self, msg):
        data = msg.数据字段


def main(args=None):
    rclpy.init(args=args)

    node = 节点类名()
    rclpy.spin(node)

    node.destroy_node()
    rclpy.shutdown()


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

### 2.3 创建订阅者的一般形式

```python
self.subscription = self.create_subscription(
    消息类型,
    '话题名称',
    回调函数,
    队列深度
)
```

四个参数分别表示：

| 参数 | 作用 |
|---|---|
| `消息类型` | 订阅的数据类型 |
| `'话题名称'` | 监听哪个话题 |
| `回调函数` | 收到消息后自动执行的函数 |
| `队列深度` | 暂时来不及处理时可保留的消息数量 |

### 2.4 订阅回调函数

```python
def listener_callback(self, msg):
    data = msg.数据字段
```

其中：

- `msg` 是收到的 ROS 2 消息对象；
- `msg.数据字段` 是消息中的实际数据；
- 回调函数不需要手动调用，收到消息后 ROS 2 会自动执行。

---

## 3. 关键函数

| 函数或类 | 作用 |
|---|---|
| `rclpy.init()` | 初始化 ROS 2 Python 客户端库 |
| `Node` | ROS 2 节点基类 |
| `super().__init__('节点名称')` | 初始化节点并设置节点名称 |
| `create_publisher()` | 创建发布者 |
| `publish()` | 发布消息 |
| `create_subscription()` | 创建订阅者 |
| `create_timer()` | 创建定时器，周期性执行回调函数 |
| `rclpy.spin()` | 让节点持续运行并处理回调函数 |
| `get_logger().info()` | 输出 ROS 2 日志信息 |
| `destroy_node()` | 销毁节点并释放资源 |
| `rclpy.shutdown()` | 关闭 ROS 2 Python 客户端库 |

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

## 4. `String` 消息类型

`String` 是 ROS 2 标准消息包 `std_msgs` 中定义的字符串消息类型。

### 4.1 导入消息类型

```python
from std_msgs.msg import String
```

注意：这里的 `String` 是 ROS 2 消息类型，不是普通的 Python 字符串类型 `str`。

### 4.2 创建消息对象

```python
msg = String()
```

### 4.3 给消息赋值

```python
msg.data = 'Hello ROS 2'
```

`String` 消息中的实际字符串内容存放在：

```python
msg.data
```

### 4.4 发布 `String` 消息

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

### 4.5 读取 `String` 消息

```python
def listener_callback(self, msg):
    self.get_logger().info(msg.data)
```

这里的 `msg.data` 就是发布者发送的字符串内容。

---

## 5. 发布者与订阅者通信条件

发布者和订阅者要正常通信，需要保证：

```text
话题名称相同
消息类型相同
QoS 设置兼容
```

例如，发布者：

```python
self.publisher_ = self.create_publisher(
    String,
    'topic',
    10
)
```

订阅者：

```python
self.subscription = self.create_subscription(
    String,
    'topic',
    self.listener_callback,
    10
)
```

对应关系：

```text
String ↔ String
topic  ↔ topic
```

如果话题名称不同，订阅者将收不到消息；如果消息类型不同，也无法按照该发布—订阅关系正常通信。

---

## 6. 本节必须掌握

1. 发布者通过 `create_publisher()` 创建，通过 `publish()` 发布消息。
2. 订阅者通过 `create_subscription()` 创建。
3. 收到消息后，ROS 2 会自动调用订阅回调函数。
4. `create_timer()` 可以周期性执行回调函数。
5. `rclpy.spin()` 让节点持续运行并处理回调。
6. `String` 消息需要先创建消息对象，再通过 `msg.data` 保存字符串。
7. 发布者与订阅者的话题名称和消息类型必须对应。

---

## 7. 易错点

### 7.1 回调函数后误加括号

正确：

```python
self.create_timer(0.5, self.timer_callback)
```

错误：

```python
self.create_timer(0.5, self.timer_callback())
```

### 7.2 直接发布普通字符串

错误：

```python
self.publisher_.publish('Hello')
```

正确：

```python
msg = String()
msg.data = 'Hello'
self.publisher_.publish(msg)
```

### 7.3 发布者和订阅者话题名称不同

```python
# 发布者
self.create_publisher(String, 'topic', 10)

# 订阅者
self.create_subscription(String, 'another_topic', callback, 10)
```

结果：订阅者无法接收到发布者的消息。

### 7.4 忘记使用 `spin()`

没有 `rclpy.spin(node)`，节点通常无法持续处理定时器或订阅消息。

---

## 8. 一句话总结

```text
发布者创建消息并发送到话题，订阅者监听同一话题，并在收到消息后自动执行回调函数。
```
