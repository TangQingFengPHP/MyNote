### 简介

`ss` 全称 `Socket Statistics`，是一个用于探究 `Linux` 上的套接字和网络连接的强大实用程序，它被用来替代老版的 `netstat` ，提供更快、更详细的信息输出。

### 常用选项

#### 查看所有 `sockets`

```shell
ss -a

# 显示所有监听和未监听的sockets
```

#### 显示监听的 `sockets`

```shell
ss -l

# 输出主动等待连接的服务
```

#### 仅显示 `TCP sockets` 

```shell
ss -t
```

#### 仅显示 `UDP sockets`

```shell
ss -u
```

#### 显示数字地址

```shell
ss -n

# 跳过 DNS 解析以显示 IP 地址和端口号
```

#### 显示包含进程的信息

```shell
ss -p

# 显示进程ID和进程名称
```

#### 仅显示 `IPv4`

```shell
ss -4
```

#### 仅显示 `IPv6`

```shell
ss -6
```

#### 显示已建立的连接

```shell
ss -t -a state established

# 显示所有已建立的 TCP 连接
```

#### 持续监控

```shell
ss -c

# 实时更新socket信息。
```

#### 显示摘要统计信息

```shell
ss -s

# 提供套接字使用情况的摘要，包括打开和已建立的连接数
```

#### 显示监听的 TCP 端口

```shell
ss -lt
```

#### 连接到指定地址

```shell
ss dst 192.168.1.100
```

#### 连接到指定端口

```shell
ss dport = 22
```

#### 显示路由表

```shell
ss -r

# 显示内核路由表
```

### 关键字段解释

* `Netid`：网络类型或协议，如：`tcp`、`udp`、`unix`

* `State`：连接的状态，如：`LISTEN`、`LISTEN`

* `Recv-Q`：接收队列中的字节数

* `Send-Q`：发送队列中的字节数

* `Local Address:Port`：连接本地端的地址和端口

* `Peer Address:Port`：连接远端的地址和端口

* `Process`：关联的进程ID和名称

### 高级用法

#### 显示 `UNIX` 域套接字

```shell
ss -x

# 显示 UNIX 套接字连接（例如，进程间通信）
```

#### 显示已建立的 `TCP` 连接

```shell
ss state established
```

#### 显示正在监听的 `UDP` 连接

```shell
ss -u state listening
```

#### 显示详细的接口统计信息

```shell
ss -i
```