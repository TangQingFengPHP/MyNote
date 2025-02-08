### 简介

`iftop` 是一个用于 `Linux` 的实时网络监控工具，用于显示接口上的带宽使用情况。它显示当前连接及其带宽使用情况的列表，帮助用户确定哪些进程或连接正在消耗网络资源。

它是跟踪机器上的实时网络使用情况的一个很好的工具，类似于 `top` 对 `CPU` 使用情况的跟踪，但它特别关注网络流量。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt update
sudo apt install iftop
```

* `CentOS/RHEL`

```shell
sudo yum install iftop
```

* `Fedora`

```shell
sudo dnf install iftop
```

### 示例用法

#### 启动 iftop

> 启动 iftop 并监控默认网络接口上的网络流量

```shell
sudo iftop
```

#### 选择特定的网络接口

> 想监控特定接口（例如，eth0，wlan0），使用 -i 选项

```shell
sudo iftop -i eth0
```

#### 显示的字段解释

* `Source`：流量的源 IP 地址

* `Destination`：流量的目标 IP 地址

* `TX (Transmit)`：系统传输的数据量

* `RX (Receive)`：系统接收的数据量

* `Cumulative data`：监控期间累计传输的数据

#### iftop 键绑定

> 启动 iftop后，就可以使用以下键绑定与该程序进行交互

* `q`：退出 `iftop`

* `t`：切换总字节数的显示

* `p`：在显示每个端口的流量和显示每个主机的流量之间切换

* `b`：在两个方向（入站和出站）的流量显示之间切换

* `s`：按来源对连接进行排序

* `d`：按目的地对连接进行排序

* `n`：切换显示/隐藏端口号（显示名称）

* `l`：在列表中显示源端口和目标端口

* `u`：仅显示上行流量

#### 按 IP 地址过滤

> 要按 IP 地址过滤流量（例如 192.168.1.100）

```shell
sudo iftop -f "host 192.168.1.100"
```

#### 按端口过滤

> 要按特定端口过滤流量（例如，HTTP 流量为 80）

```shell
sudo iftop -f "port 80"
```

#### 按来源或目的地过滤

* 要按源 IP 过滤流量（例如 192.168.1.100）

```shell
sudo iftop -f "src 192.168.1.100"
```

* 要按目标 IP 过滤流量（例如 192.168.1.101）

```shell
sudo iftop -f "dst 192.168.1.101"
```

#### 显示总使用量

```shell
sudo iftop -t
```

#### 更改更新频率

> 默认值为 2 秒

```shell
sudo iftop -s 1
```

#### 显示端口号而不是名称

```shell
sudo iftop -n
```

#### 在脚本中使用 iftop

> 将捕获 10 秒的数据并将其保存到名为 iftop_output.txt 的文件中

```shell
sudo iftop -t -s 10 > iftop_output.txt
```

#### 仅显示上行流量

```shell
sudo iftop -u
```

#### 监控端口 443 (HTTPS) 的流量

```shell
sudo iftop -f "port 443"
```