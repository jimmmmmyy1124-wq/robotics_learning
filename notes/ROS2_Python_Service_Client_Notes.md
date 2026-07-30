# ROS 2 Python 服务端与客户端笔记

> 本笔记整理 ROS 2 Python 服务通信学习中的四个重点：服务端的一般结构、客户端的一般结构、服务接口与 `future`，以及服务通信和话题通信的区别。

---

## 1. 服务端的一般结构

服务端节点负责提供某项功能，等待客户端发送请求，处理请求后返回响应。

### 1.1 通用代码结构

```python
import rclpy
from rclpy.node import Node
from 接口包.srv import 服务接口类型                 # 导入服务接口


class 服务端节点类名(Node):

    def __init__(self):
        super().__init__('服务端节点名称')

        self.service = self.create_service(         # 创建服务端
            服务接口类型,
            '服务名称',
            self.service_callback
        )

    def service_callback(self, request, response):  # 服务回调函数
        input_data = request.请求字段               # 读取请求数据

        response.响应字段 = 处理结果                # 写入响应数据

        return response                             # 返回响应


def main(args=None):
    rclpy.init(args=args)

    node = 服务端节点类名()
    rclpy.spin(node)                                # 持续等待客户端请求

    node.destroy_node()                             # 销毁节点并释放资源
    rclpy.shutdown()                                # 关闭 ROS 2 Python 客户端库


if __name__ == '__main__':
    main()
```

### 1.2 服务端运行流程

```text
创建节点
→ 创建服务端
→ spin() 等待请求
→ 收到客户端请求
→ 自动调用服务回调函数
→ 从 request 中读取数据
→ 把结果写入 response
→ return response 返回响应
```


---

## 2. 客户端的一般结构

客户端节点负责寻找指定服务，创建请求对象，向服务端发送请求，并接收服务端返回的响应。

### 2.1 通用代码结构

```python
import rclpy
from rclpy.node import Node
from 接口包.srv import 服务接口类型                 # 导入服务接口


class 客户端节点类名(Node):

    def __init__(self):
        super().__init__('客户端节点名称')

        self.client = self.create_client(           # 创建客户端
            服务接口类型,
            '服务名称'
        )

        while not self.client.wait_for_service(timeout_sec=1.0):    #检查服务端是否启动
            self.get_logger().info('服务尚未启动，正在等待……')

        self.request = 服务接口类型.Request()       # 创建请求

    def send_request(self, input_data):
        self.request.请求字段 = input_data          # 写入请求数据

        return self.client.call_async(self.request) # 发送请求


def main(args=None):
    rclpy.init(args=args)

    node = 客户端节点类名()

    future = node.send_request(请求数据)

    rclpy.spin_until_future_complete(               # 等待服务端返回响应
        node,
        future
    )

    response = future.result()                      # 获取响应对象
    result = response.响应字段                      # 读取返回结果

    node.destroy_node()                             # 销毁节点并释放资源
    rclpy.shutdown()                                # 关闭 ROS 2 Python 客户端库


if __name__ == '__main__':
    main()
```

### 2.2 客户端运行流程

```text
创建节点
→ 创建客户端
→ 等待对应服务启动
→ 创建请求对象
→ 给请求字段赋值
→ call_async() 发送请求
→ 获得 future 对象
→ 等待请求完成
→ future.result() 获取响应
```



---

## 3. 其他知识点

### 3.1 服务接口

服务通信必须使用服务接口。服务接口规定客户端能够发送哪些数据，以及服务端必须返回哪些数据。

服务接口的一般形式：

```text
请求字段（客户端发送的请求）
---
响应字段（服务端返回的响应）
```


ROS 2 自带的两整数相加接口为：

```text
example_interfaces/srv/AddTwoInts
```

它的结构是：

```text
int64 a
int64 b
---
int64 sum
```

含义为：

```text
客户端发送整数 a 和 b
→ 服务端计算 a + b
→ 服务端返回 sum
```

可以使用下面的命令查看接口：

```bash
ros2 interface show example_interfaces/srv/AddTwoInts
```

一般形式：

```bash
ros2 interface show 接口包名/srv/接口名称
```

---

### 3.2 `request` 和 `response`

服务端回调函数的一般形式：

```python
def service_callback(self, request, response):
    response.响应字段 = 对 request 的处理结果
    return response
```

其中：

- `request` 保存客户端发送的数据；
- `response` 保存服务端准备返回的数据；
- `return response` 把结果返回给客户端。


---

### 3.3 创建请求对象

客户端必须按照服务接口创建请求对象：

```python
request = 服务接口类型.Request()
```



不能直接把普通 Python 数据发送给服务端，必须先把数据写入对应的请求对象。

---


### 3.4 `spin()` 和 `spin_until_future_complete()`

服务端使用：

```python
rclpy.spin(node)
```

服务端需要持续运行并等待不同客户端的请求：

```text
spin()
→ 等待请求
→ 执行服务回调函数
→ 返回响应
→ 继续等待
```

客户端常使用：

```python
rclpy.spin_until_future_complete(node, future)
```

客户端只需要等待本次请求完成：

```text
发送一次请求
→ 等待该请求完成
→ 收到响应后继续执行程序
```

---

