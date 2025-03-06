### 简介

`tcpdump` 是一个命令行数据包分析器，可实时捕获和检查网络流量。它通常用于网络故障排除、性能分析和安全监控。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt update && sudo apt install tcpdump -y
```

* `CentOS/RHEL`

```shell
sudo yum install tcpdump -y
```

* `macOS`

```shell
brew install tcpdump
```

### 基础语法

```shell
tcpdump [options] [filter]
```

### 示例用法

#### 在默认接口上捕获数据包

实时捕获并显示网络数据包。

```shell
sudo tcpdump
```

#### 列出可用网络接口

```shell
sudo tcpdump -D
```

**输出示例**

```shell
1. eth0
2. wlan0
3. lo
```

**可以使用此列表中的接口名称来捕获特定接口上的数据包。**

#### 在特定接口上捕获数据包

```shell
sudo tcpdump -i eth0
```

#### 限制捕获的数据包数量

仅捕获 10 个数据包然后停止

```shell
sudo tcpdump -c 10 -i eth0
```

#### 将捕获的数据包保存到文件

```shell
sudo tcpdump -i eth0 -w capture.pcap
```

#### 从文件读取数据包

```shell
sudo tcpdump -r capture.pcap
```

#### 仅捕获特定协议

* 仅 TCP 数据包

```shell
sudo tcpdump -i eth0 tcp
```

* 仅 UDP 数据包

```shell
sudo tcpdump -i eth0 udp
```

* 仅 ICMP（ping）数据包

```shell
sudo tcpdump -i eth0 icmp
```

#### 捕获特定主机的数据包

捕获来自/到 `192.168.1.1` 的流量

```shell
sudo tcpdump -i eth0 host 192.168.1.1
```

#### 捕获特定端口上的数据包

* 捕获 HTTP 流量（端口 80）

```shell
sudo tcpdump -i eth0 port 80
```

* 捕获 SSH 流量（端口 22）

```shell
sudo tcpdump -i eth0 port 22
```

#### 从特定源或目标捕获数据包

* 仅捕获来自源 `192.168.1.100` 的数据包

```shell
sudo tcpdump -i eth0 src 192.168.1.100
```

* 仅捕获目的地址为 `192.168.1.100` 的数据包

```shell
sudo tcpdump -i eth0 dst 192.168.1.100
```

#### 组合多个过滤器

在端口 443 (HTTPS) 上捕获往返于 192.168.1.100 的 TCP 流量

```shell
sudo tcpdump -i eth0 tcp and host 192.168.1.100 and port 443
```

#### 以十六进制和 ASCII 格式显示数据包

```shell
sudo tcpdump -X -i eth0
```

#### 无需解析主机名即可捕获数据包

`-n` 选项可防止 DNS 查找，从而提高性能

```shell
sudo tcpdump -n -i eth0
```

#### 仅捕获数据包头（无有效负载） 

`-s 0` 标志捕获完整的数据包而不是截断

```shell
sudo tcpdump -s 0 -i eth0
```

#### 仅捕获 HTTP 流量并显示内容

`-A` 选项以 ASCII 格式打印数据包内容

```shell
sudo tcpdump -A -i eth0 port 80
```

