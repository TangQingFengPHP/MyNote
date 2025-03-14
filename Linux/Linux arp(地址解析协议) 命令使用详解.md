### 简介

`arp`（地址解析协议）命令用于查看、添加和修改系统的 `ARP` 缓存。`ARP` 缓存存储了 `IP` 地址和 `MAC` 地址之间的映射，有助于网络中的计算机高效通信。

许多现代 `Linux` 发行版使用 `ip neigh`（来自 `iproute2`）而不是 `arp`。但是，`arp` 对于管理 `ARP` 表仍然有用。

### 示例用法

#### 显示 ARP 表

```shell
arp -a

或

ip neigh show

# 这将显示当前 ARP 缓存，显示 IP 地址、MAC 地址和网络接口
```

**示例输出**

```shell
192.168.1.1    ether   00:1A:2B:3C:4D:5E   C eth0
192.168.1.10   ether   00:1B:3C:4D:5E:6F   C eth0
```

* `ether`：以太网连接

* `C`：条目已完成

* `eth0`：网络接口名称

#### 显示特定 IP 的 ARP 条目

```shell
arp -a 192.168.1.1

或

ip neigh show 192.168.1.1
```

#### 添加静态 ARP 条目

```shell
sudo arp -s 192.168.1.100 00:11:22:33:44:55

或

sudo ip neigh add 192.168.1.100 lladdr 00:11:22:33:44:55 dev eth0

# 这会将 IP 192.168.1.100 手动映射到 MAC 地址 00:11:22:33:44:55
# 静态 ARP 条目将保留到重新启动，除非添加到启动脚本
```

#### 删除 ARP 条目

```shell
sudo arp -d 192.168.1.100

或

sudo ip neigh del 192.168.1.100 dev eth0
```

#### 清除整个 ARP 缓存

```shell
sudo ip -s -s neigh flush all

# 这将删除所有动态学习的 ARP 条目
```

#### 在接口上启用或禁用 ARP

* 禁用 `ARP`

```shell
sudo ip link set dev eth0 arp off
```

* 启用 `ARP`

```shell
sudo ip link set dev eth0 arp on
```

#### 使用 tcpdump 监控 ARP 流量

```shell
sudo tcpdump -nn -e -i eth0 arp

# 这将捕获 eth0 上的实时 ARP 流量，显示请求和响应
```

#### 查找 ARP 欺骗攻击

检查重复的 `MAC` 地址（可能的 `ARP` 欺骗）

```shell
arp -a | sort

# 如果多个 IP 映射到同一个 MAC 地址，则可能有人试图拦截流量
```


