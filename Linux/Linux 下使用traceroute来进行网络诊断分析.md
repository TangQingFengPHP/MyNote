### 简介

`traceroute` 命令是一种网络诊断工具，用于跟踪数据包从系统到目标服务器的路径。它有助于识别网络延迟和路由问题。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt install traceroute -y
```

* `RHEL/CentOS`

```shell
sudo yum install traceroute -y
```

* `Fedora`

```shell
sudo dnf install traceroute -y
```

### 示例用法

#### 基础用法

发送具有增加的 `TTL`（生存时间）值的数据包以发现数据包所采用的路径

```shell
traceroute google.com
```

**示例**

```shell
traceroute 8.8.8.8
```

**示例输出**

```shell
traceroute to google.com (142.250.190.78), 30 hops max, 60 byte packets
 1  router.lan (192.168.1.1)  1.013 ms  0.986 ms  1.010 ms
 2  192.168.0.1 (192.168.0.1)  2.105 ms  2.098 ms  2.100 ms
 3  isp-gateway (203.0.113.1)  10.258 ms  10.302 ms  10.310 ms
 4  core-router (203.0.113.2)  20.551 ms  20.564 ms  20.590 ms
 5  google.com (142.250.190.78)  30.759 ms  30.802 ms  30.820 ms
```

**字段解析**

* `Hop Number`：数据包经过的路由器序列

* `Host`：路由器的主机名或 IP 地址

* `Round-Trip Times (ms)`：路由器的响应时间为三次

**常用符号**

* `* * *`：没有响应（可能是数据包被阻止或丢失）

* `!H`：主机无法访问

* `!N`：网络不可达

* `!X`：防火墙阻止

#### 仅显示 IP 地址

为了避免主机名解析并仅显示 IP

```shell
traceroute -n google.com
```

#### 指定最大跳数

默认情况下，`traceroute` 最多允许 30 个跳数

```shell
traceroute -m 20 google.com
```

#### 更改每跳探测次数

默认情况下，`traceroute` 每跳发送 3 个数据包

```shell
traceroute -q 1 google.com
```

#### 使用 ICMP 代替 UDP

默认情况下，`traceroute` 使用 `UDP` 数据包，如果某些网络阻止 `UDP`，可以改用 `ICMP`

```shell
traceroute -I google.com
```

#### 使用 TCP SYN 数据包

当 `ICMP` 和 `UDP` 被阻止时有用

```shell
traceroute -T google.com
```

#### 设置数据包大小

指定数据包大小（默认值：60 字节）

```shell
traceroute google.com 100
```

### traceroute 与 ping 和 mtr 对比

|  命令   |  功能   |
| --- | --- |
|  `ping`   |  检查主机是否可访问并测量延迟   |
|  `traceroute`   |  显示数据包到达目的地所采用的路线   |
|  `mtr`   |  `ping` 和 `traceroute` 的实时组合   |