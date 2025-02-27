### 简介

`tracepath` 命令是 `Linux` 中的一个网络诊断工具，类似于 `traceroute` ，但专门用于跟踪到目标主机的网络路径，同时自动处理路径MTU发现。这是一种简单的方法，可以找出机器和远程目的地之间的跃点，同时还可以识别沿途的任何问题。

### 基本语法

```shell
tracepath [options] <destination_host>
```

* `<destination_host>`：要跟踪路径的目标目的地的 IP 地址或主机名

### 常用选项

* `-n`：以数字形式显示跳转地址（无需 DNS 解析）

* `-l <length>`：设置数据包的长度（默认为 1500）

* `-p <port>`：设置用于测试的端口（默认为 33434）

* `-m <max_hops>`：设置最大跳数

* `-q <number>`：每跳发送的探测数（默认为 1）

* `-f <first_hop>`：从指定的跳跃开始跟踪

* `-T`：关闭路径MTU（路径最大传输单元）发现的检测

### 示例用法

#### 跟踪主机的路径

这将逐跳显示到 `example.com` 的网络路径，并提供有关沿路径的最大传输单元 (MTU) 的信息。

```shell
tracepath example.com
```

#### 使用数字输出追踪路径

为了避免 DNS 查找并显示数字 IP 地址而不是主机名

```shell
tracepath -n example.com
```

#### 设置最大跳数

仅跟踪最多 10 个跳数

```shell
tracepath -m 10 example.com
```

#### 更改数据包长度

要跟踪数据包大小为 1200 字节

```shell
tracepath -l 1200 example.com
```

#### 指定自定义端口

```shell
tracepath -p 8080 example.com
```

#### 显示禁用 MTU 发现的路径

默认情况下，`tracepath` 会尝试发现路径 MTU，但可以使用 -T 选项禁用此行为

```shell
tracepath -T example.com
```

#### 指定每跳探测次数

```shell
tracepath -q 3 example.com
```

#### 从特定跳开始跟踪路径

从第 5 跳开始跟踪

```shell
tracepath -f 5 example.com
```

### 示例输出

```shell
 1?: [LOCALHOST]                      pmtu 1500
 1:  <your local router>               0.123ms 
 2:  <ISP Gateway>                    12.345ms 
 3:  <ISP Network>                    15.678ms 
 4:  <some intermediate router>       16.123ms 
 5:  <example.com>                    20.456ms reached
```

**输出解释**

经过 5 跳后到达目的地 (`example.com`)

* `pmtu 1500`：路径上的最大传输单元 (MTU) 大小

* `1到5`：本地机器和目的地（`example.com`）之间的路由器或设备

* `ms时间`：每次跳跃的往返时间

### 与 traceroute 的比较

* MTU 发现：tracepath 具有内置的 MTU 发现功能，而 traceroute 默认没有

* 默认行为：tracepath 尝试确定沿路径的 MTU，而 traceroute 仅显示跳数而没有此功能


