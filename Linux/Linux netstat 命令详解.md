### 简介

`netstat` 全称是：`network statistics`，是一个用于监控、排除网络连接故障、路由表的命令行工具，它提供关于网络统计和 `socket` 连接的详细信息。

### 安装

```shell
sudo apt install net-tools  # For Debian/Ubuntu
sudo yum install net-tools  # For CentOS/RHEL
```

### 常用选项示例

#### 查看所有连接

```shell
netstat -a

# 显示所有活动的连接和监听的端口
```

#### 仅显示监听的端口

```shell
netstat -l
```

#### 仅显示 `TCP` 连接

```shell
netstat -t
```

#### 仅显示 `UDP` 连接

```shell
netstat -u
```

#### 显示带有数字地址的连接

```shell
netstat -an

# 跳过主机名解析以实现更快的输出。
```

#### 显示连接时包括进程名和PID

```shell
netstat -p
```

#### 显示路由表

```shell
netstat -r

# 输出内核路由表，与route 命令相似
```

#### 查看网络接口统计信息

```shell
netstat -i

# 提供有关发送/接收的数据包和接口错误的详细信息
```

#### 持续监控连接

```shell
netstat -c

# 每秒刷新一次输出
```

#### 合并多个选项

```shell
netstat -tunlp

# -t：TCP
# -u：UDP
# -n：数字地址
# -l：监听的端口
# -p：PID和进程名称
```

### 关键输出字段解释

* `Proto`：协议类型：`TCP` 或 `UDP`

* `Recv-Q`：接收队列大小（等待读取的数据）

* `Send-Q`：发送队列大小（等待发送的数据）

* `Local Address`：连接本地的地址和端口。

* `Foreign Address`：连接远程的地址和端口

* `State`：连接的状态，`LISTEN`、`ESTABLISHED` 等

* `PID/Program name`：进程ID和进程名称
