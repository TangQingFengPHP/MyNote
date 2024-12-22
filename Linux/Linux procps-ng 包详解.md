### 简介

`procps-ng` 包是用于监视和管理 `Linux` 上的进程和系统性能的实用程序集合。它与 `/proc` 文件系统交互以检索实时系统信息。`procps-ng` 中的实用程序包括 `ps`、`top`、`free`、`uptime` 等命令。

### 安装 `procps-ng`

#### 使用包管理工具安装

* `Debian/Ubuntu`

```shell
sudo apt update
sudo apt install procps
```

* `RHEL/CentOS`

```shell
sudo yum install procps-ng
```

* `Fedora`

```shell
sudo dnf install procps-ng
```

#### 拉取源码从源码构建

```shell
git clone https://gitlab.com/procps-ng/procps.git
cd procps
./autogen.sh
./configure
make
sudo make install

# 验证是否安装成功，输入套件包含的命令，如：
ps --version
```

### `procps-ng` 包含的命令

* `free`：报告系统中空闲和使用的内存容量（包括物理和交换内存）

* `pgrep`：根据名称和其它属性查找进程

* `pidof`：报告指定程序的 `PID`

* `pkill`：根据名称和其它属性给进程发送信号

* `pmap`：报告指定进程的内存映射情况

* `ps`：列出正在运行的进程

* `pwdx`：报告进程的当前工作目录

* `slabtop`：实时显示内核 `slab` 缓存信息

* `sysctl`：运行时修改内核参数

* `tload`：打印当前系统平均负荷曲线图

* `top`：显示最 `CPU` 密集型进程列表；它可以实时地连续查看处理器活动

* `uptime`：报告系统运行时长、登录用户数目以及系统平均负荷

* `vmstat`：报告虚拟内存统计信息、给出关于进程、内存、分页、块输入/输出(`IO`)、陷阱以及 `CPU` 活动的信息

* `w`：显示当前登录的用户、以及登录地点和时间

* `watch`：重复运行指定命令，显示输出的第一个整屏；这允许用户查看随着时间的输出变化

* `libprocps`：包含该软件包大部分程序使用的函数

### 常用的命令示例

#### `ps` -  Process Status

显示有关正在运行的进程的信息

* `ps aux`：显示所有进程的详细信息

* `ps -e`：显示所有进程

* `ps -ef`：显示具有父/子关系的完整格式列表

* `ps aux | grep "nginx"`：使用 `grep` 过滤进程

#### `top` - 实时进程监控

实时显示系统任务，包括 `CPU` 和内存使用情况。

* `k`：通过输入其 `PID` 来终止进程

* `q`：退出 `top` 界面

* `h`：显示可用的命令信息

* `1`：显示每个核心的 `CPU` 使用率

* `top -p 1234`：指定 `PID`

#### `free` - 显示内存使用情况

显示有关内存使用情况的信息

* `-h`：显示成人类可读的格式

* `-m`：以兆字节显示内存

* `-g`：以千兆字节显示内存

#### `uptime` - 显示系统正常运行时间

显示系统运行的时间以及平均负载

示例输出：

```shell
12:00:01 up 3 days, 4:53, 3 users, load average: 0.11, 0.25, 0.30
```

#### `kill` - 终止进程

向进程发送信号以终止它。

* `-9`：强制终止一个进程

* `-15`：正常(平滑)终止一个进程

示例：

```shell
kill -9 1234
```

#### `pkill` - 按进程名称终止

按名称终止进程

示例：

```shell
pkill nginx
```

#### `pgrep` - 按名称查找进程

列出与名称匹配的进程的 `PID`

示例：

```shell
pgrep sshd
```

#### `vmstat` - 虚拟内存统计

显示系统性能指标

```shell
vmstat 5 10

# 每 5 秒更新一次指标，共 10 次
```

#### `pidof` - 查找进程 `ID`

通过名称返回正在运行的进程的 `PID`

示例：

```shell
pidof sshd
```

#### `watch` - 间隔运行命令

按照指定的间隔重复运行命令并显示输出

* `-n <seconds>`：指定间隔（默认为 2 秒）

* `-d`：突出显示更新之间的变化

示例：

```shell
watch -n 5 df -h

# -n 5 是选项
# df -h 是运行的命令
```