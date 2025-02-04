### 简介

`iostat` 命令用于监控 `Linux` 系统输入/输出设备的加载情况。它提供有关`CPU` 统计信息以及设备和分区的输入/输出统计信息。通过显示 `I/O` 操作如何影响系统性能，它对于诊断性能瓶颈（例如磁盘或网络活动缓慢）特别有用。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt update
sudo apt install sysstat
```

* `CentOS/RHEL`

```shell
sudo yum install sysstat
```

* `Fedora`

```shell
sudo dnf install sysstat
```

### 基本语法

```shell
iostat [options] [interval] [count]
```

* `interval`：每次报告之间的时间间隔（以秒为单位）

* `count`：要显示的报告数量

### 常用选项

* `-c`：仅显示 CPU 统计信息

* `-d`：仅显示设备级统计信息

* `-x`：显示扩展统计数据，包括详细的 I/O 指标，例如每个设备的平均队列大小和平均服务时间

* `-p`：显示设备分区的统计信息。例如，`iostat -p sda` 将显示 `sda` 所有分区的统计信息（例如 `sda1`、`sda2`）

* `-t`：打印每个报告的时间戳

* `-h`：显示带有可用选项的帮助消息

### 示例用法

#### 显示基本 CPU 和 I/O 统计信息

> 默认情况下，iostat 将显示所有块设备（例如硬盘、SSD）的 CPU 统计信息和设备 I/O 统计信息。它不会自动刷新，因此只会看到一份报告

```shell
iostat
```

**示例输出**

```shell
Linux 5.4.0-70-generic (hostname)     04/27/2023      _x86_64_        (8 CPU)

avg-cpu:  %user   %nice    %system   %iowait  %steal   %idle
           3.52    0.03     1.30      0.71     0.00    94.44

Device            tps    kB_read/s   kB_wrtn/s   kB_read   kB_wrtn
sda              10.58        206.23        98.64   1350224    656928
```

**字段解释**

* `CPU stats`：显示 CPU 处于不同状态的时间百分比
    * `%user`：运行用户级进程所花费的时间
    * `%nice`：运行具有正 nice 值的用户进程所花费的时间
    * `%system`：运行内核进程所用的时间
    * `%iowait`：等待 I/O 操作完成所花费的时间
    * `%idle`：CPU 空闲的时间

* `Device stats`：显示每个设备的 I/O 统计信息
    * `tps`：每秒I/O操作次数
    * `kB_read/s`：每秒读取的千字节数
    * `kB_wrtn/s`：每秒写入的千字节数
    * `kB_read`：读取的总千字节数
    * `kB_wrtn`：写入的总千字节数

#### 每秒显示 I/O 统计信息，共 5 个报告

> 每 1 秒刷新一次报告，并在退出前显示总共 5 份报告

```shell
iostat 1 5
```

**示例输出**

```shell
Linux 5.4.0-70-generic (hostname)     04/27/2023      _x86_64_        (8 CPU)

avg-cpu:  %user   %nice    %system   %iowait  %steal   %idle
           3.52    0.03     1.30      0.71     0.00    94.44

Device            tps    kB_read/s   kB_wrtn/s   kB_read   kB_wrtn
sda              10.58        206.23        98.64   1350224    656928
...

avg-cpu:  %user   %nice    %system   %iowait  %steal   %idle
           3.51    0.02     1.29      0.70     0.00    94.47

Device            tps    kB_read/s   kB_wrtn/s   kB_read   kB_wrtn
sda              10.55        205.12        99.56   1350324    657024
```

#### 显示特定设备的 CPU 和 I/O 统计信息

```shell
iostat -d sda
```

#### 仅显示 CPU 统计信息

```shell
iostat -c
```

**示例输出**

```shell
Linux 5.4.0-70-generic (hostname)     04/27/2023      _x86_64_        (8 CPU)

avg-cpu:  %user   %nice    %system   %iowait  %steal   %idle
           3.52    0.03     1.30      0.71     0.00    94.44
```

#### 仅显示 I/O 统计信息

```shell
iostat -d
```

#### 实时监控磁盘 I/O

> 将每秒显示所有设备的扩展统计信息

```shell
iostat -x 1
```

**扩展信息包括的字段示例**

* `%util`：设备繁忙的时间百分比

* `await`：I/O 操作的平均时间（以毫秒为单位）

* `svctm`：I/O 操作的平均服务时间

#### 分析一段时间内的磁盘性能

> 每 5 秒显示一次扩展统计信息，总共显示 3 份报告

```shell
iostat -x 5 3
```

#### 显示特定设备的统计信息

```shell
iostat -d sda1 sdb
```