### 简介

`nftables` 是 `iptables、ip6tables、arptables和ebtables` 的继承者，用于管理 `Linux` 中的包过滤和网络地址转换。它提供了一种更现代、更灵活和更有效的方式来配置防火墙，取代了旧的工具。

`nftables` 在 `Linux` 内核 `3.13` 及以上版本中可用，它是 `nft` 包的一部分。用于配置 `nftables` 的主要命令行工具是 `nft`。

### 基本概念

* `Tables`：用于组织规则的容器。每个表可以有一个或多个链，并且链可以处理不同类型的数据包（输入、输出、转发）

* `Chains`：链包含处理数据包的规则。链的示例包括输入、输出、转发等

* `Rules`：规则定义如何处理匹配的数据包（例如，接受、丢弃或记录）

* `Sets`：集合是 `IP` 地址或端口等值的集合，使规则定义更加高效

* `Hooks`：这些指定在数据包流（输入、输出、转发）的哪个点应用链

### 命令结构

```shell
sudo nft [add|delete|list] [table family] [object] [options]
```

* `add`：添加规则、表或链

* `delete`：删除规则、表或链

* `list`：列出当前规则集

### 示例用法

#### 检查当前 nftables 配置

> 列出当前的完整规则集，包括表、链和规则

```shell
sudo nft list ruleset
```

#### 创建新表

> 将在 inet 系列（支持 IPv4 和 IPv6）中创建一个名为 filter 的表

```shell
sudo nft add table inet filter
```

* `inet`：包含了 ipv4 和 ipv6

* `ip`：对于 IPv4

* `ip6`：对于 IPv6

* `arp`：对于 ARP 数据包

* `bridge`：对于以太网帧

#### 向表添加链

```shell
sudo nft add chain inet filter input { type filter hook input priority 0 \; }
```

* `input`：链的名称

* `type filter`：链类型（可以是filter，nat，mangle等等）

* `hook input`：链与数据包流（例如，输入、输出、转发）相连接的点

* `priority 0`：链的优先级（当一个钩子上有多个链可用时使用）

#### 向链中添加规则

> 此规则允许端口 22 (SSH) 上的传入 TCP 流量

```shell
sudo nft add rule inet filter input tcp dport 22 accept
```

* `tcp dport 22`：匹配目标端口为 22 的 TCP 数据包

* `accept`：对匹配的数据包采取的操作

#### 删除规则

> 要从链中删除特定规则

```shell
sudo nft delete rule inet filter input handle 5
```

#### 删除链

```shell
sudo nft delete chain inet filter input
```

#### 删除表

```shell
sudo nft delete table inet filter
```

#### 刷新表中的所有规则

> 要删除特定表中的所有规则

```shell
sudo nft flush table inet filter
```

#### 保存当前 nftables 配置

```shell
sudo nft list ruleset > /etc/nftables.conf
```

**要在启动时应用保存的配置**

```shell
sudo systemctl enable nftables
sudo systemctl start nftables
```

#### 立即应用更改

```shell
sudo nft -f /etc/nftables.conf
```

#### 使用集合

集合是一组可用于规则的值（例如 IP 地址、端口）。通过将多个值组合在一起，可以提高性能和可读性

* 创建一组 IP 地址

```shell
sudo nft add set inet filter my_ips { type ipv4_addr \; }
```

* 要将 IP 地址添加到此集合

```shell
sudo nft add element inet filter my_ips { 192.168.1.1, 192.168.1.2 }
```

* 在规则中引用此集合

```shell
sudo nft add rule inet filter input ip saddr @my_ips accept
```

#### NAT（网络地址转换）

* 创建 NAT 表

```shell
sudo nft add table ip nat
```

* 为 NAT 添加后路由链

```shell
sudo nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }
```

* 添加规则在访问互联网时伪装（隐藏）内部 IP

```shell
sudo nft add rule ip nat postrouting oifname "eth0" masquerade
```

#### 使用日志操作记录特定的数据包

> 这会记录输入链中发往 192.168.1.100 的所有数据包

```shell
sudo nft add rule inet filter input ip daddr 192.168.1.100 log
```

#### 阻止传入的 SSH 连接

```shell
sudo nft add rule inet filter input tcp dport 22 drop
```

#### 允许 HTTP 和 HTTPS 流量

```shell
sudo nft add rule inet filter input tcp dport { 80, 443 } accept
```

#### 允许特定 IP 地址

```shell
sudo nft add rule inet filter input ip saddr 192.168.1.100 accept
```

#### 丢弃所有传入流量（默认拒绝）

```shell
sudo nft add rule inet filter input drop
```