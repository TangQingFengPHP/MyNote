### 简介

`mpstat` 命令（ `sysstat` 包的一部分）用于报告 `Linux` 下的 `CPU` 使用统计信息。它提供有关 `CPU` 性能的详细统计信息，如果存在多核系统，则包括有关每个单独 `CPU`（或核心）的信息。该命令可用于性能监视和识别 `CPU` 瓶颈。

### 安装

> 在使用 `mpstat` 之前，确保系统上安装了 `sysstat` 包

* `Debian/Ubuntu`：

```shell
sudo apt update
sudo apt install sysstat
```

* `CentOS/RHEL`：

```shell
sudo yum install sysstat
```

* `Fedora`：

```shell
sudo dnf install sysstat
```

### 基本语法

```shell
mpstat [options] [interval] [count]
```

* `interval`：每个报告之间的时间间隔（以秒为单位）。如果没有指定间隔，`mpstat` 将提供单个快照。

* `count`：要生成的报告数量。默认情况下，它会一直运行，直到用户停止它（例如，使用 `Ctrl + C`）

### 常用选项

* `-P`：显示特定 CPU 的统计信息
    * 例如：`mpstat -P 0` 显示 CPU 0 统计信息，`mpstat -P ALL` 显示所有 CPU 的统计信息

* `-u`：仅显示用户级 CPU 利用率（默认显示所有级别）

* `-V`：显示 `mpstat` 的版本

* `-I`：显示 `I/O` 统计数据

* `-A`：显示所有 CPU 的统计信息以及其他选项（如中断和上下文切换）

### 示例用法

#### 显示所有 CPU 的使用率统计信息

> 此命令将显示每个可用 `CPU`（或核心）的使用率。输出包括 `CPU` 在各种任务（如用户进程、系统进程、空闲时间等）上花费的时间百分比

```shell
mpstat
```

**输出示例**

```shell
Linux 5.4.0-52-generic (hostname)    05/02/2023    _x86_64_    (2 CPU)

06:40:01 PM  CPU    %usr   %nice    %sys  %iowait   %irq  %soft   %steal   %guest  %gnice   %idle
06:40:01 PM  all    3.00    0.00     2.00     0.00    0.00    0.00     0.00    0.00    0.00   95.00
06:40:01 PM    0    4.00    0.00     2.00     0.00    0.00    0.00     0.00    0.00    0.00   94.00
06:40:01 PM    1    2.00    0.00     2.00     0.00    0.00    0.00     0.00    0.00    0.00   96.00
```

**字段解释**

* `%usr`：用户级进程使用的 CPU 百分比

* `%nice`：具有正 `nice` 值（低优先级）的进程使用的 CPU 百分比

* `%sys`：系统级进程（内核）使用的 CPU 百分比

* `%iowait`：CPU 等待 I/O 操作完成的时间百分比

* `%irq`：硬件中断使用的 CPU 百分比

* `%soft`：软件中断使用的 CPU 百分比

* `%steal`：虚拟机管理程序从虚拟机“窃取”的 CPU 时间百分比

* `%guest`：虚拟机中客户操作系统使用的 CPU 百分比

* `%gnice`：具有正 `nice` 值的客户机使用的 CPU 百分比

* `%idle`：CPU 空闲的时间百分比

#### 显示特定 CPU 的使用率

> 仅显示 `CPU 0` 的 CPU 统计信息。-P 选项允许指定特定的 CPU/核心。还可以使用 -P ALL 显示所有 CPU 的统计信息

```shell
mpstat -P 0
```

#### 定期监控 CPU 使用率

> 报告每秒的 CPU 使用率，总共 5 次迭代

```shell
mpstat 1 5
```

**输出示例**

```shell
Linux 5.4.0-52-generic (hostname)    05/02/2023    _x86_64_    (2 CPU)

06:40:01 PM  CPU    %usr   %nice    %sys  %iowait   %irq  %soft   %steal   %guest  %gnice   %idle
06:40:02 PM  all    3.00    0.00     2.00     0.00    0.00    0.00     0.00    0.00    0.00   95.00
06:40:03 PM  all    3.50    0.00     2.50     0.00    0.00    0.00     0.00    0.00    0.00   94.00
06:40:04 PM  all    3.00    0.00     2.00     0.00    0.00    0.00     0.00    0.00    0.00   95.00
06:40:05 PM  all    3.20    0.00     2.10     0.00    0.00    0.00     0.00    0.00    0.00   94.70
06:40:06 PM  all    3.10    0.00     2.00     0.00    0.00    0.00     0.00    0.00    0.00   94.90
```

#### 以 5 秒为间隔显示 CPU 使用率

```shell
mpstat 5
```

#### 显示扩展 CPU 统计信息

> -x 标志提供扩展统计数据，包括有关 CPU 使用率的更多详细信息，如“上下文切换”和“中断”

```shell
mpstat -x 1 5
```

#### 每 2 秒显示一次每个 CPU 的统计信息，共 10 次迭代

```shell
mpstat -P ALL 2 10
```