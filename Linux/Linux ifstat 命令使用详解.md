### 简介

`Linux` 中的 `ifstat` 命令用于显示网络接口统计信息，显示系统中每个网络接口的网络流量信息（如发送和接收的字节数或包数）。它提供了一种实时监视网络接口活动的方法，帮助系统管理员和用户诊断与网络相关的问题。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt update
sudo apt install ifstat
```

* `CentOS/RHEL`

```shell
sudo yum install net-tools
```

* `Fedora`

```shell
sudo dnf install net-tools
```

### 基本语法

```shell
ifstat [options] [interval] [count]
```

* `interval`：每次更新之间的时间间隔（以秒为单位）

* `count`：退出前更新统计信息的次数。如果未指定，它将无限期地继续，直到用户使用 `Ctrl+C` 中断

### 常用选项

* `-i`：指定想要显示统计信息的接口。
    * 例如：`ifstat -i eth0` 将仅显示 `eth0` 接口的统计信息

* `-a`：显示所有接口（包括非活动接口）的统计信息。

* `-n`：以每秒千字节为单位显示统计数据（默认）

* `-t`：打印所有接口的总数

* `-h`：显示帮助或使用信息

* `-b`：以字节而不是千字节为单位显示统计数据

* `-p`：以数据包而不是字节为单位显示统计数

* `-s`：以秒为单位显示统计数据

* `-d`：以十进制格式而不是十六进制格式显示统计数据

### 示例用法

#### 显示网络统计信息

```shell
ifstat
```

**示例输出**

```shell
   eth0      eth1
   KB In   KB Out   KB In   KB Out
1.01      0.22      0.00     0.00
0.99      0.10      0.00     0.00
1.00      0.10      0.00     0.00
```

* `KB In`：已接收千字节数

* `KB Out`：传输的千字节数

#### 显示特定间隔的网络统计信息

> 每 1 秒显示一次网络统计信息

```shell
ifstat 1
```

#### 显示特定接口的网络统计信息

> 仅显示 `eth0` 接口的统计信息

```shell
ifstat eth0
```

#### 显示特定间隔和计数的网络统计信息

> 将每秒显示一次网络统计信息，总共更新 5 次

```shell
ifstat 1 5
```

**示例输出**

```shell
   eth0      eth1
   KB In   KB Out   KB In   KB Out
1.01      0.22      0.00     0.00
1.00      0.23      0.00     0.00
0.99      0.20      0.00     0.00
1.00      0.21      0.00     0.00
1.01      0.22      0.00     0.00
```

#### 显示所有接口的统计信息

> 将显示所有接口的网络统计信息，包括不活动的接口

```shell
ifstat -a
```

#### 以每秒千字节数显示所有接口的统计信息

```shell
ifstat -a -n
```

#### 显示所有接口的统计数据（以每秒字节数为单位） 

```shell
ifstat -a -b
```

#### 显示所有接口的统计信息，运行10次迭代

```shell
ifstat -a 1 10
```
