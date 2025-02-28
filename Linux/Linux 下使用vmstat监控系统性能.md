### 简介

`Linux` 中的 `vmstat`（虚拟内存统计）命令用于监控系统性能，包括CPU使用情况、内存使用情况、交换活动、磁盘I/O和系统进程。它提供实时性能指标，有助于诊断系统瓶颈。

### 基础语法

```shell
vmstat [options] [delay] [count]
```

* `delay`：更新之间的间隔（以秒为单位）

* `count`：命令在停止之前运行的次数

### 示例用法

#### 不带参数运行 vmstat

这将显示一份包含自上次重启以来的系统统计信息的报告

```shell
vmstat
```

**输出示例**

每 2 秒更新一次，共5 次

```shell
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b    swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0       0  50000  12000 800000    0    0     3     5  120  300  5  2 90  3  0
 0  0       0  51000  11500 805000    0    0     2     4  115  290  4  1 94  1  0
```

**字段解释**

* `Process`：procs
    * `r`：正在运行的进程数
    * `b`：处于不可中断睡眠状态的进程数

* `Memory`：memory
    * `swpd`：使用的交换内存（KB）
    * `free`：可用内存 (KB)
    * `buff`：缓冲内存 (KB)
    * `cache`：缓存内存 (KB)

* `Swap`：swap
    * `si`：换入内存（KB/秒）
    * `so`：换出内存（KB/秒）

* `I/O`：io
    * `bi`：从块设备接收的块（KB/s）
    * `bo`：发送到块设备的块数（KB/s）

* `System`：system
    * `in`：每秒中断的次数
    * `cs`：每秒上下文切换的次数

* `CPU`：cpu
    * `us`：用户 CPU 使用率百分比
    * `sy`：系统（内核）CPU 使用率百分比
    * `id`：空闲 CPU 百分比
    * `wa`：等待 I/O 的 CPU 百分比
    * `st`：虚拟机管理程序窃取的 CPU 百分比（仅与虚拟化环境相关）

#### 实时监控系统性能

每 1 秒更新一次，无限期

```shell
vmstat 1
```

#### 限制报告数量

每2秒更新一次，运行5次

```shell
vmstat 2 5
```

#### 以兆字节而不是千字节显示

使用 `-S M` 以兆字节为单位显示值

```shell
vmstat -S M 1 5
```

#### 监视磁盘活动

显示磁盘 I/O 统计信息

```shell
vmstat -d
```

#### 显示详细的 CPU 统计信息

显示各种系统统计信息的摘要

```shell
vmstat -s
```

#### 监控 NUMA（非统一内存访问）节点

显示活动和非活动内存

```shell
vmstat -a
```

### 与其他工具的比较

|  命令   |  特性   |
| --- | --- |
|  top   |  每个进程的实时 CPU 和内存使用情况   |
|  htop   |  top的交互式版本   |
|  iostat  |  详细的磁盘 I/O 统计信息   |
|  free  |  内存使用情况详细信息   |
|  sar  |  高级系统性能监控   |

