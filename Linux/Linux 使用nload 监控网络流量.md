### 简介

`Linux` 中的 `nload` 命令是一个用于实时监控网络流量的工具。它提供了传入和传出流量的可视化表示，帮助用户一目了然地了解网络活动。对于需要监控网络接口上的流量的系统管理员和网络工程师来说，它尤其有用。

### 安装

* `Ubuntu/Debian`

```shell
sudo apt update
sudo apt install nload
```

* `Red Hat/CentOS`

```shell
sudo yum install nload
```

* `Fedora`

```shell
sudo dnf install nload
```

### 示例用法

#### 基础用法

将显示系统上默认网络接口的网络统计信息。输出将实时显示传入（下载）和传出（上传）流量。

```shell
nload
```

**输出概述**

* `Incoming traffic`：进入系统的数据

* `Outgoing traffic`：系统传出的数据

* `Total traffic`：传入和传出流量的组合

* `Traffic speed`：显示实时传输的数据量

**示例输出**

```shell
Device: eth0
             Incoming         Outgoing         Total
   Rate:     15.5 kB/s        10.2 kB/s        25.7 kB/s
   Average:  14.5 kB/s        9.6 kB/s         24.1 kB/s
   Min:       3.5 kB/s        2.0 kB/s         5.5 kB/s
   Max:      35.0 kB/s        15.0 kB/s        50.0 kB/s
```

#### 指定网络接口

默认情况下，`nload` 将监视默认网络接口（通常是 eth0 或 wlan0）

```shell
nload eth1
```

#### 限制显示特定流量类型

* 仅显示传入流量

```shell
nload -t in
```

* 仅显示传出流量

```shell
nload -t out
```

#### 指定刷新率

`-d` 选项允许设置刷新显示的延迟（以秒为单位）。默认刷新率为 1 秒

```shell
nload -d 2
```

#### 设置流量速率的显示单位

`-u` 选项允许设置数据速率的单位，例如位、字节、千字节或兆字节

* 将单位设置为KB（千字节）

```shell
nload -u K
```

* 将单位设置为MB（兆字节）

```shell
nload -u M
```

#### 监控多个接口

```shell
nload eth0 wlan0
```

#### 监控特定网络接口（eth1）

```shell
nload eth1
```

#### 仅监控传出流量并以 MB 为单位显示速率

```shell
nload -t out -u M
```

#### 以 2 秒的刷新率监控传入流量

```shell
nload -t in -d 2
```

