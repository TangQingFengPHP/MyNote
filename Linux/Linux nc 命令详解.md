### 简介

`nc` 全称 `netcat`，是一个在 `Linux` 中多功能的网络工具，通常用于通过 `TCP` 或 `UDP` 读取和写入网络连接，也能作为客户端或服务端用来 `debug`，测试，网络问题分析。

### 常用示例

#### 检查端口是否是打开的

```shell
nc -zv <hostname> <port>

nc -zv example.com 80

# -z：扫描但不发送数据
# -v：详细输出模式
```

#### 启动一个简单的 `TCP` 服务

```shell
nc -l <port>

nc -l 1234

# 启动一个监听在 1234 端口的服务，任何数据发送在这个端口上将会显示在终端
```

#### 连接指定端口的 `TCP` 服务

```shell
nc <hostname> <port>

nc localhost 1234

# 连接 localhost 的 1234 端口
```

#### 发送文件到指定服务端口

```shell
nc <receiver_ip> <port> < file_to_send

nc 192.168.1.10 1234 < file.txt
```

#### 从指定服务端口接受文件

```shell
nc -l <port> > received_file

nc -l 1234 > file.txt
```

#### 端口扫描

```shell
nc -zv <hostname> <start_port>-<end_port>

nc -zv 192.168.1.1 20-25

# 扫描从20到25的端口
```

#### 建立简单的聊天会话

```shell
nc -l 1234

nc <server_ip> 1234
```

#### 发送 `HTTP` 请求到目标服务 

```shell
echo -e "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n" | nc example.com 80
```

#### 连接到 `UDP` 服务

```shell
nc -u <hostname> <port>

nc -u localhost 1234
```

#### 发送数据到指定的主机端口

```shell
echo "Hello, World!" | nc <hostname> <port>
```

### 高级用法

#### 创建一个 `Web` 服务

```shell
while true; do echo -e "HTTP/1.1 200 OK\r\n\r\nHello, World!" | nc -l 8080; done

# 创建一个监听在 8080 端口的服务
```

#### 监控网络流量

```shell
nc -l 1234 | tee output.log

# 记录发送到 1234 端口的数据并写到日志文件
```