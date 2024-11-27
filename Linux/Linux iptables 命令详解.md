### 简介

`iptables` 是一个在 `Linux` 中的管理防火墙规则的命令行工具，它作为 `Linux` 内核的 `netfilter` 框架的一部分运行，以控制传入和传出的网络流量。

### 与 `firewalld` 相比

* `iptables` 是基于规则的，每个规则必须独立定义，`firewalld` 是基于区域的，规则适用于预定义或自定义区域。

* `iptables` 适合高度精细和手动的配置，`firewalld` 动态规则更简单且更加用户友好。

* `iptables` 需要刷新或重新启动才能应用更改，`firewalld` 支持不间断的即时更改。

* `iptables` 对于静态、简单的配置来说非常高效，`firewalld` 由于抽象层，速度稍微慢一些，但大多数情况下可以忽略不计。
### 基础概念

#### `Chains`

> 处理数据包的一组规则

* `INPUT`: 控制传入数据包。

* `FORWARD`: 控制通过系统转发的数据包。

* `OUTPUT`: 控制传出的数据包。

#### `Tables`

> 规则处理类别

* `filter`: 基本数据包过滤的默认表。

* `nat`: 处理网络地址转换（NAT）。

* `mangle`: 改变数据包。

* `raw`: 在连接跟踪之前配置数据包。

#### `Rules`

> 对数据包应用的操作，例如：`ACCEPT`、`DROP`

### 常用操作

#### 列出所有链的规则

```shel
sudo iptables -L
```

#### 列出带行号的规则

```shell
sudo iptables -L --line-numbers
```

#### 列出特定表中的规则

```shell
sudo iptables -t nat -L
```

#### 允许指定端口传入流量

```shell
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# -A INPUT: 追加到 INPUT链

# -p tcp: 指定协议为 TCP

# --dport 80: 指定目标端口为80

# -j ACCEPT: 接受数据包
```

#### 阻止来自指定IP的流量

```shell
sudo iptables -A INPUT -s 192.168.1.100 -j DROP

# -s 192.168.1.100: 指定源IP地址
```

#### 允许来自子网的流量

```shell
sudo iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT
```

#### 允许端口 22 (SSH) 上的传出流量

```shell
sudo iptables -A OUTPUT -p tcp --dport 22 -j ACCEPT
```

#### 在网络之间转发流量

```shell
sudo iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
```

#### 按行号删除规则

```shell
sudo iptables -D INPUT 2

# -D INPUT 2: 删除 INPUT 链的第二个规则
```

#### 将传入流量的默认策略设置为 DROP

```shell
sudo iptables -P INPUT DROP
```

#### 将传出流量的默认策略设置为 ACCEPT

```shell
sudo iptables -P OUTPUT ACCEPT
```

#### 保存当前规则到指定文件

```shell
sudo iptables-save > /etc/iptables.rules
```

#### 从文件中恢复规则

```shell
sudo iptables-restore < /etc/iptables.rules
```

#### 将端口 8080 转发至 80

```shell
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j REDIRECT --to-port 80
```

#### 伪装流量（NAT）

```shell
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

#### 转发流量到其他IP

```shell
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.200:80
```

#### 记录丢弃的数据包

```shell
sudo iptables -A INPUT -j LOG --log-prefix "Dropped Packet: " --log-level 4
```

#### 记录已接受的数据包

```shell
sudo iptables -A INPUT -j LOG --log-prefix "Accepted Packet: " --log-level 4
```

#### 查看数据包和字节数

```shell
sudo iptables -L -v
```

#### 重置计数器

```shell
sudo iptables -Z
```

#### 刷新所有规则

```shell
sudo iptables -F
```

#### 在指定的 `Table` 上刷新规则

```shell
sudo iptables -t nat -F
```

#### 删除所有用户定义的 `Chains`

```shell
sudo iptables -X
```

### 常用选项

* `-A`：追加到规则到链中

* `-D`：从链中删除一个规则

* `-P`：为链设置默认策略

* `-F`：刷新链中的所有规则

* `-L`：列出链中的所有规则

* `-t [table]`：指定 `table`

* `-i`：指定输入接口，例如：`eth0`

* `-o`：指定输出接口

* `-s`：指定源IP地址

* `-d`：指定目标IP地址

* `-p`：指定协议类型，例如：`tcp`、`udp`、`icmp`

