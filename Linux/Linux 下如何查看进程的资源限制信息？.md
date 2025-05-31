### 简介

`Linux` 上的 `cat /proc/$pid/limits` 命令提供有关特定进程的资源限制的信息，其中 `$pid` 是相关进程的进程 `ID （pid）`。该文件是 `/proc 文件系统的一部分，该文件系统是一个虚拟文件系统，提供有关进程和系统资源的信息。

### 基本用法

```shell
cat /proc/1234/limits

# 其中 1234 是目标进程的 PID。
```

* `$pid`：目标进程的进程 `ID（PID）`。可以通过 `ps` 命令查找，例如 `ps aux | grep <进程名>`。

* 输出：显示指定进程的软限制（`soft limit`）、硬限制（`hard limit`）以及限制单位。

输出示例：

```shell
Limit                     Soft Limit           Hard Limit           Units
Max cpu time               unlimited            unlimited             seconds
Max file size              unlimited            unlimited             bytes
Max data size              unlimited            unlimited             bytes
Max stack size             8388608              8388608               bytes
Max core file size        0                    unlimited             bytes
Max resident set          unlimited            unlimited             bytes
Max processes             6348                 6348                  processes
Max open files            1024                 1024                  files
Max locked memory         65536                65536                 bytes
Max address space         unlimited            unlimited             bytes
Max file locks            unlimited            unlimited             locks
Max pending signals       6348                 6348                  signals
Max msgqueue size         819200               819200                bytes
Max nice priority         20                   20                    priority
Max realtime priority     99                   99                    priority
Max realtime timeout      unlimited            unlimited             us
```

**关键字段解释**

* `Max cpu time`：该进程可以消耗无限量的 `CPU` 时间（没有上限）

* `Max file size`：该进程可以创建任意大小的文件

* `Max data size`：数据段（存储变量和数组）不受限制

* `Max stack size`：堆栈大小限制为 `8MB`（8388608 字节），堆栈存储函数调用数据

* `Max open files`：该进程最多可以同时打开 1024 个文件

* `Max processes`：该进程最多可以产生 6348 个子进程

**修改限制**

* `ulimit`：用于调整当前 `shell` 会话的限制

* `prlimit`：用于对已经运行的进程设置限制

* `/etc/security/limits.conf`：用于设置用户和组的默认资源限制

### 资源限制

这些是应用于进程的各种限制和约束，以控制其可以使用的资源，例如内存、CPU 和文件描述符。

此文件中列出的常见资源包括：

* `Limit`：资源的实际限制

* `Current`：该进程当前对该资源的使用情况

* `Soft Limit`：当前应用于进程的限制，可以由进程进行调整（在硬限制范围内）

* `Hard Limit`：不可超过的最大限制，它由系统管理员设置或采用默认设置

* `Units`：衡量限制的单位（例如字节、KB 等）

### 常见的限制类型

* `Max process limit`：用户可以创建的最大进程数。它限制了用户可以生成的进程数量

* `Max open files`：进程可以拥有的最大文件描述符数量。这会影响进程可以同时打开的文件、套接字等的数量

* `Max locked memory`：可以锁定到 `RAM` 中的最大内存量，防止其被换出

* `Max address space`：进程可以分配的最大虚拟地址空间量，其中包括内存、堆和堆栈

* `Max CPU time`：进程可使用的最大 `CPU` 时间。以秒为单位

* `Max file locks`：进程可以拥有的文件锁的最大数量

* `Max number of threads`：进程可以创建的最大线程数

* `Max user time`：进程在用户空间中花费的最长时间（以秒为单位）（即不包括内核时间）

* `Max virtual memory`：进程可以分配的虚拟内存总量，通常控制进程内存使用的上限

* `Max file size`：进程创建文件的最大大小

* `Max data size`：进程数据段的最大大小（包括堆和数据）

* `Max stack size`：进程堆栈的最大大小

* `Max core file size`：核心转储文件（`core dump`）的最大大小

* `Max resident set`：驻留内存（`RSS`，物理内存）的最大大小

* `Max pending signals`：进程可排队的最大信号数

* `Max msgqueue size`：`POSIX` 消息队列的最大大小

* `Max realtime priority`：实时调度优先级的最大值

* `Max realtime timeout`：实时任务的最大超时时间（微秒）

### 软限制与硬限制

* 软限制：这是当前为进程设置的限制，进程可以更改它，管理员也可以使用 `ulimit`（用于 `shell`）或 `prlimit`（用于正在运行的进程）等命令更改它

* 硬限制：这是除非超级用户 (`root`) 更改，否则无法超过的最大限制，硬限制由内核强制执行，它是软限制的上限

### 常见用法

#### 检查进程资源限制

用于诊断进程是否因资源限制（如文件描述符不足）而失败：

```shell
cat /proc/$(pidof bash)/limits
```

查看当前 `bash` 进程的限制

#### 查找文件描述符限制

检查进程的最大文件描述符数：

```shell
cat /proc/1234/limits | grep "Max open files"
```

输出示例：

```shell
Max open files            1024                 1048576              files
```

#### 结合 ulimit 调整限制

`ulimit` 命令可修改当前 `shell` 的软限制（需要硬限制允许）。例如，增加文件描述符限制

```shell
ulimit -n 2048
cat /proc/$$/limits | grep "Max open files"
```

#### 监控系统限制

检查所有进程的限制模式

```shell
for pid in /proc/[0-9]*; do echo "PID: $(basename $pid)"; cat $pid/limits; done
```

#### 诊断文件描述符不足

假设某个服务（`PID 1234`）报错 `Too many open files`

```shell
cat /proc/1234/limits | grep "Max open files"
lsof -p 1234 | wc -l
```

如果打开的文件数接近软限制，临时增加限制：

```shell
prlimit --pid 1234 --nofile=2048:1048576
```

或修改服务配置文件（如 `systemd` 的 `LimitNOFILE`）

#### 检查核心转储

确保进程可以生成核心转储：

```shell
cat /proc/1234/limits | grep "Max core file size"
```

如果软限制为 0，启用核心转储：

```shell
ulimit -c unlimited
```

### 相关配置文件

`/proc/$pid/limits` 的值通常来自以下来源：

* `/etc/security/limits.conf`：定义用户或组的默认资源限制

```shell
# 格式：<domain> <type> <item> <value>
* soft nofile 1024
* hard nofile 1048576
```

`*` 表示所有用户，`nofile` 对应 `Max open files`

* `/etc/security/limits.d/`：包含额外的限制配置文件

* 系统默认值：由内核参数或系统配置（如 `/proc/sys/`） 决定

* `ulimit` 命令：动态修改当前 `shell` 或进程的软限制

* `systemd` 配置：服务进程的限制可在 `systemd` 单元文件中的`[Service]` 快设置（例如 `LimitNOFILE=2048`）
