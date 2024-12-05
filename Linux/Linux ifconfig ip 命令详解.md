### 简介

`ifconfig` 和 `ip` 命令用于配置和显示 `Linux` 上的网络接口。虽然 `ifconfig` 是传统工具，但现在已被弃用并被提供更多功能的 `ip` 命令取代。

### `ifconfig`

#### 安装 

```shell
sudo apt install net-tools

sudo yum install net-tools
```

#### 查看所有活动的网络接口

```shell
ifconfig
```

#### 启动/激活网络接口

```shell
sudo ifconfig <interface_name> up

# 例如：
sudo ifconfig eth0 up

# eth0：代表系统上的第一个以太网网络接口
# 其他的网络接口名字示例：
# wlan0：第一个无线网络接口
# lo：环回接口，用于同一台机器内的本地通信
# enp0s3：有线以太网接口的可预测名称
```

#### 关闭/停用网络接口

```shell
sudo ifconfig <interface_name> down

# 例如：
sudo ifconfig eth0 down
```

#### 分配 `IP` 地址

```shell
sudo ifconfig <interface_name> <IP_address> netmask <subnet_mask>

# 例如：
sudo ifconfig eth0 192.168.1.100 netmask 255.255.255.0
```

#### 分配广播地址

```shell
sudo ifconfig <interface_name> broadcast <broadcast_address>

# 例如：
sudo ifconfig eth0 broadcast 192.168.1.255
```

#### 启用混杂模式

```shell
sudo ifconfig <interface_name> promisc

# 例如：
sudo ifconfig eth0 promisc
```

#### 禁用混杂模式

```shell
sudo ifconfig <interface_name> -promisc

# 例如：
sudo ifconfig eth0 -promisc
```

### `ip` 命令

#### 查看所有网络接口

```shell
ip link show
```

#### 查看所有接口的IP地址

```shell
ip addr show
```

#### 查看具体接口详情

```shell
ip addr show <interface_name>

# 例如：
ip addr show eth0
```

#### 启用网络接口

```shell
sudo ip link set <interface_name> up

# 例如：
sudo ip link set eth0 up
```

#### 禁用网络接口

```shell
sudo ip link set <interface_name> down

# 例如：
sudo ip link set eth0 down
```

#### 分配IP地址

```shell
sudo ip addr add <IP_address>/<CIDR> dev <interface_name>

# 例如：
sudo ip addr add 192.168.1.100/24 dev eth0
```

#### 移除IP地址

```shell
sudo ip addr del <IP_address>/<CIDR> dev <interface_name>

# 例如：
sudo ip addr del 192.168.1.100/24 dev eth0
```

#### 显示路由表

```shell
ip route show
```

#### 添加默认网关

```shell
sudo ip route add default via <gateway_ip>

# 例如：
sudo ip route add default via 192.168.1.1
```

#### 删除路由

```shell
sudo ip route del <network>/<CIDR>

# 例如：
sudo ip route del 192.168.1.0/24
```



