## 一、命令介绍

`ps` 命令是 `Process Status` 的缩写，是一个命令行实用程序，用于显示或查看与Linux系统中运行的进程相关的信息。

命令原理：`ps` 是通过读取虚拟文件：`/proc` 拿到进程数据的，不需要给 `ps` 设置任何的权限就可以运行。

更深层次的后面再分析。

> ps可以接受几种不同的选项：

* `UNIX` 选项，带有前导 `-` 风格的，也叫 `Standard-Style` (标准风格)。

* `BSD` 选项，不带前导 `-` 风格的，也叫 `BSD-Style` 。

* `GNU long` 选项，前面带两个 `-`，即 `--`

不同类型的选项可以自由混合，但也会出现冲突，有一些同义的选项，它们功能是相同的，由于许多标准进行了兼容实现。

## 二、命令选项

### 简单筛选

* `a` ：选择所有进程（`BSD-Style`）。

* `-A` ：选择所有进程，与 `-e` 等同（`标准格式`）。

* `-a` ：选择除 `session` 领导者和没有与终端关联的进程之外的所有进程。

> 在Linux中，每个进程都有多个ID来关联它，包括：

```
1. Process ID (PID)，即进程ID

这是标识进程的任意数字。每个进程都有一个唯一的ID，但是在进程退出并且父进程检索了退出状态之后，进程ID将被释放，供新进程重用。

2. Parent Process ID (PPID)，即父进程ID

如果父进程在子进程退出之前退出，则子进程的PPID将更改为另一个进程。

3. Process Group ID (PGID)

这是进程组leader的PID。如果PID == PGID，则此进程是进程组领导。

4. Session ID (SID)

这是会话领导者的PID。如果PID == SID，则此进程为会话领导进程。

会话和进程组只是将一些相关的进程作为一个单元来对待的方法。一个进程组的所有成员总是属于同一个会话，但是一个会话可以有多个进程组。

通常，shell将是会话领导者，该shell执行的每个管道将是一个进程组。这是为了在shell退出时很容易杀死它的子进程。
```

* `-d` ：选择除了 `session leader` 的所有进程。

* `--deselect` ：选择除满足指定条件之外的所有进程，即反选，与 `-N` 等同。

* `-e` ：选择所有进程，与 `-A` 等同。

* `-N` ：与 `--deselect` 等同。

* `T` ：选择与该终端关联的所有进程，和不带参数的 `t` 选项等同。

* `r` ：仅选择运行中的进程。

* `x` ：与 `a` 一起配合使用。

### 通过列表筛选进程

> [某某list] 可接受两种格式，空格分隔和逗号分隔，
>
> 例如：`ps -p "1 2"` 或 `ps -p 1,2`

* `-C [cmdlist]` ： 通过命令的名称筛选，以前的 `procps` 和内核版本会截断这个命令名为15个字符，在新版本这个限制已剔除，如果想通过模糊匹配的方式，则此法不通。

* `-G [grplist]` ：通过真实的组ID或组名来筛选，即创建此进程的用户所属的组。

* `-g [grplist]` ：通过session或有效的组名来筛选，仅仅当组名也指定时，组ID才能生效。

* `--Group [grplist]` ：与 `-G` 等同。

* `--group [grplist]` ：与 `-g` 等同。

* `p [pidlist]` ：通过进程id筛选。

* `-p [pidlist]` ：与 `p` 等同。

* `--pid [pidlist]` ：与 `p` 等同。

* `--ppid [pidlist]` ：通过父进程id筛选子进程。

* `q [pidlist]` ：通过进程id筛选（快速模式）。

* `-q [pidlist]` ：与 `q` 等同。

* `--quick-pid [pidlist]` ：与 `q` 等同。

* `-s [sesslist]` ：通过sessionID筛选。

* `--sid [sesslist]` ：与 `-s` 等同。

* `t [ttylist]` ：通过tty（终端）筛选，`ttylist` 可以为空，与 `-t` 和 `--tty` 几乎相同。

* `-t [ttylist]` ：通过tty（终端）筛选，`tty` 可以使用多种形式，如：`/dev/ttyS1`、`ttyS1`、`S1`，`[ttylist]`如果是 `-` 则筛选没有关联到任何终端的进程。 

* `--tty [ttylist]` ：与 `-t` 和 `t` 等同。

* `U [userlist]` ：通过有效的用户ID或用户名来筛选。

* `-U [userlist]` ：通过真实的用户ID或用户名来筛选，创建此进程的用户即为真实的用户。

* `-u [userlist]` ：与 `U` 等同。

* `--User [userlist]` ：与 `-U` 等同。

* `--user [userlist]` ：与 `-u` 和 `U` 等同。

#### 输出格式控制

* `-c` ：在指定了 `-l` 格式时，显示不同的调度程序的信息。

* `--context` ：对于 `SELinux` 显示安全的上下文格式。

* `-f` ：列出完整的格式，一些合并使用的选项，如接 `-L`，NLWP(线程数)，LWP(线程id) 两个字段会显示出来。

* `--format [format]` ：用户可以指定格式来显示，与 `-o` 和 `o` 等同。

* `j` ：`BSD` 任务控制格式。

* `-j` ：任务格式。

* `l` ：`BSD` 长格式。

* `-l` ：长格式，长格式即显示更多的列。

* `-M` ：添加一个安全数据的列，与 `Z` 等同。

* `O [format]` ：自定义显示的列，但会显示一些公共的预定义列，预 `-O` 等同，此为 `BSD` 风格。

* `-O [format]` ：自定义显示的列，但会显示一些公共的预定义列，输入 `-O` 等于 `-o pid,format,state,tname,time,command`，或者 `-o pid,format,tname,time,cmd`，与 `O [format]` 等同。

* `o [format]` ：指定用户定义的格式，与 `-o` 和 `--format` 等同。

* `-o [format]` ：

指定用户定义的格式，`[format]` 的参数格式可以用空格分隔或逗号分隔的列表，可用的列关键词在 `标准格式说明符` 中会说明。

预定义的列可用 `=` 重命名，例如：`ps -o pid,ruser=RealUser -o comm=Command`。

如果 `=` 后面为空，则列头名称则不输出，例如：`ps -o pid= -o comm=`。

可以显示指定列宽，通过使用 `wchan:[num]` ，例如：`ps opid,wchan:42,cmd`。

* `s` ：显示信号格式。

* `u` ：显示以用户为主的格式。

* `v` ：显示虚拟内存格式。

### 输出修饰符

* `c` ：显示真正的命令名称，是源自于可执行文件的名称，而不是命令行参数值。该选项有效地把 `args` 格式关键字转换为  `comm` 格式的关键字。

* `--cumulative` ：包含一些已经死亡的子进程数据。

* `f` ：用ASCII字符显示树形结构，与 `--forest`、`-H` 相似。

* `--headers` ：在输出的每一页重复显示列头名称。

* `k [spec]` ：指定列排序，`+[key]` 或 `-[key]` ，`+` 是升序或字母表顺序排序，`-` 是倒序或叫降序，与 `--sort` 等同。

例如：`ps k -ppid,+pid`

* `--no-headers` ：不显示列头名称，与 `--no-heading` 等同。

* `--sort [spec]` ：与 `k` 等同。

### 显示线程信息

* `H` ：将线程显示为进程。

* `-L` ：显示线程信息，可能显示 `LWP` 和 `NLWP` 字段。

* `m` ：在进程后面显示线程信息。

* `-m` ：与 `m` 等同。

* `-T` ：显示线程，可能带有 `SPID` 列。

### 其他帮助信息

* `--help [section]` ：打印帮助信息，`[section]` 可为 `simple|list|output|threads|misc|all` 之一。也可简写为：`s|l|o|t|m|a`，即可打印简单信息、传多个参数的列表格式、输出控制定制、显示线程信息、显示混杂的信息和所有帮助信息。

用法：`ps --help simle` 或 `ps --help s` ，其他同理。

* `--info` ：打印 `debug` 信息。

* `L` ：列出所有格式说明符。

* `V` ：打印 `procps-ng` 版本

* `-V` ：与 `V` 等同。

* `--version` ：与 `V` 等同。

### 注意：

标记为 `defunct` 的进程是死进程(所谓的 
“僵尸进程”)，因为它们的父进程没有适当的销毁它们。

那么如果杀死僵尸进程呢？解决方法是：找到父进程，杀死/退出父进程，僵尸进程将会被销毁。

如果用户名的长度大于显示列时，用户名将被截断。

不建议使用 `ps -aux` 等命令选项，因为它是一个两种不同标准的混淆。根据 `POSIX` 和按照 `UNIX` 标准，此命令会显示用户名为：`x` 用户的所有进程，如果用户 `x` 不存在，`ps` 才会假定你实际上是想使用 `ps aux` ，所以此处有歧义。

### 进程状态码解析

`s` 、`stat` 、`stat` 都是状态码的名称。

状态码如下：

* `D` ：不间断的休眠（通常是 `IO`）

* `I` ：闲置的内核线程。

* `R` ：正在运行的或可运行的进程。

* `S` ：可中断的休眠（通常是等待事件完成）

* `T` ：被作业控制信号停止。

* `t` ：在跟踪期间由调试器停止。

* `X` ：已经死亡的进程，正常情况下是看不到的。

* `Z` ：僵尸进程，已经终止，但未被父进程销毁的进程。

`BSD` 格式的还有如下几种：

* `<` ：高优先级。

* `N` ：低优先级。

* `L` ：页面是否被锁定在内存中(用于实时和自定义输入输出)。

* `s` ：session领导者

* `l` ：表示有多个线程。

* `+` ：在前台进程组中。

### AIX格式描述符

* CODE NORMAL HEADER 

* %C   pcpu   %CPU（CPU使用率）

* %G   group  GROUP（组id）

* %P   ppid   PPID 父级进程id）

* %U   user   USER（用户id）

* %a   args   COMMAND（命令）

* %c   comm   COMMAND（命令）

* %g   rgroup RGROUP（进程真实组名）

* %n   nice   NI（优先级值）

* %p   pid    PID（进程id）

* %r   pgid   PGID（进程组id）

* %t   etime  ELAPSED（进程消耗的时间）

* %u   ruser  RUSER（进程真实用户id）

* %x   time   TIME（CPU执行累计时间）

* %y   tty    TTY（终端）

* %z   vsz    VSZ（进程的虚拟内存大小）

### 标准格式说明符

```
分别代表：CODE HEADER DESCRIPTION

%cpu %CPU CPU利用率

%mem %MEM 进程内存占比

ag_id AGID 与CFS调度程序一起运行以提高交互式桌面性能的进程关联的自动组标识符。

ag_nice AGNI 自动组nice值影响该组中所有进程的调度

args COMMAND 命令和参数

blocked BLOCKED 被阻挡信号的掩码

bsdstart START 命令开始执行的时间

bsdtime bsdtime CPU执行累计的时间，包括用户的操作+系统的调度。

c C 处理器利用率

caught CAUGHT 捕获信号的掩码

cgname CGNAME 显示进程所属的控制组的名称

cgroup CGROUP 显示进程所属的控制组

cgroupns CGROUPNS 描述进程所属的名称空间的唯一inode号

class CLS 进程的调度类

cls CLS 进程的调度类

cmd CMD 与 args, command等同

comm COMMAND 命令名称

command COMMAND 与 args, command 等同

cp CP 每单位(百分之十)的CPU使用率

cputime TIME CPU累计的执行时间，"[DD-]hh:mm:ss" 格式，与time等同

cputimes TIME CPU累计的执行时间的秒数，与times等同。

cuc %CUC 进程的CPU利用率，包括死亡的子进程

cuu %CUU 进程的CPU利用率，不包括死亡的子进程

drs DRS 数据驻留集大小，进程保留的私有内存量

egid EGID 进程的有效组ID号，是一个十进制整数

egroup EGROUP 进程的有效组ID

etime ELAPSED 自进程启动以来经过的时间，[[DD-]hh:]mm:ss格式

etimes ELAPSED 自进程启动以来经过的秒数

euid EUID 有效的用户ID

euser EUSER 有效的用户名

exe EXE 可执行文件的路径

fgid FGID 文件系统访问组ID

fgroup FGROUP 文件系统访问组ID

fname COMMAND 进程可执行文件基本名称的前8个字节

fuid FUID 文件系统访问用户ID

fuser FUSER 文件系统访问用户ID

gid GID 与egid等同。

group GROUP 与egroup等同。

ignored IGNORED 被忽略信号的掩码

ipcns IPCNS 描述进程所属的名称空间的唯一inode号

label LABEL 安全标签，最常用于SELinux上下文数据

lstart STARTED 命令启动的时间

lsession SESSION 如果包含systemd支持，则显示进程的登录会话标识符

luid LUID 显示与进程关联的登录ID

lwp LWP 可调度实体的轻量级进程(线程)ID，LWP即(light weight process) 的缩写

lxc LXC 正在其中运行任务的lxc容器的名称，如果进程不在容器内运行，将会显示 "-"

machine MACHINE 显示分配给VM或容器的进程的机器名称

maj_flt MAJFLT 此进程中发生的主要页面错误的数量

min_flt MINFLT 此进程中发生的小页面错误的数量

mntns MNTNS 描述进程所属的名称空间的唯一inode号

netns NETNS 描述进程所属的名称空间的唯一inode号

ni NI 优先级值

nlwp NLWP 进程中的LWPS(线程)数

numa NUMA 与最近使用的处理器相关联的节点

nwchan WCHAN 进程正在休眠的内核函数的地址

oom OOM 内存不足评分，取值范围为0 ~ +1000，用于在内存耗尽时选择要终止的任务。

oomadj OOMADJ 内存不足调整因子，该值被添加到当前内存不足评分中，然后用于确定当内存耗尽时要杀死哪个任务。

ouid OWNER 如果包含systemd支持，则显示进程会话所有者的Unix用户标识符。

pcpu %CPU 与%cpu等同。

pending PENDING 挂起信号的掩码

pgid PGID 进程组id

pgrp PGRP 进程组id

pid PID 进程id

pidns PIDNS 描述进程所属的名称空间的唯一inode号

pmem %MEM 与%mem等同。

policy POL 与class、cls等同。

ppid PPID 父进程id

pri PRI 进程优先级，数字越大表示优先级越高

psr PSR 进程最后执行的处理器。

pss PSS 比例共享大小，即未交换的物理内存，共享内存按比例占所有映射它的任务

rbytes RBYTES 此进程确实导致从存储层提取的字节数

rchars RCHARS 此任务导致从存储器中读取的字节数。

rgid RGID 真正的组ID

rgroup RGROUP 真正的组名

rops ROPS 系统调用，I/O操作的次数

rss RSS 常驻集大小，任务已使用的未交换的物理内存(单位为千字节)

rssize RSS 与rss等同

rsz RSZ 与rss等同。

rtprio RTPRIO 实时优先级

ruid RUID 真正的用户id

ruser RUSER 真正的用户id

s S 最小状态显示（一个字符）

sched SCH 进程的调度策略

sess SESS sessionID

sgi_p P 进程当前正在执行的处理器

sgid SGID 保存的组名

sid SID 与sess等同

sig PENDING 与pending等同

sigcatch CAUGHT 与caught等同

sigignore IGNORED 与ignored等同

sigmask BLOCKED 与blocked等同

size SIZE 交换空间的大致数量

spid SPID 与lwp等同

stackp STACKP 进程栈的底部(开始)地址

start STARTED 命令开始的时间

start_time START 命令开始时间或日期

stat STAT 多字符的进程状态

state S 单字符进程状态，与s等同

stime STIME 与start_time等同

suid SUID 保存的用户id

supgid SUPGID 组补充组id

supgrp SUPGRP 组补充组名

suser SUSER 保存的用户名

svgid SVGID 与sgid等同

svuid SVUID 与suid等同

tgid TGID 表示任务所属线程组的编号

thcount THCNT 与nlwp等同

tid TID 与spid等同

time TIME 与cputime等同

timens TIMENS 描述进程所属的名称空间的唯一inode号

times TIME 与cputimes等同

tname TTY 与tty等同

tpgid TPGID 进程所连接的tty(终端)上前台进程组的ID

trs TRS 文本驻留集大小，用于可执行代码的物理内存量

tt TT 与tty等同

tty TT 控制tty(终端)

ucmd CMD 与comm等同

ucomm COMMAND 与comm等同

uid UID 与euid等同

uname USER 与euser等同

unit UNIT 显示进程所属的单元

user USER 与euser等同

userns USERNS 描述进程所属的名称空间的唯一inode号

uss USS 唯一的集合大小，未交换的物理内存

uunit UUNIT 如果包含systemd支持，则显示进程所属的用户单元。

vsize VSZ 与vsz等同

vsz VSZ 进程占用的虚拟内存大小

wbytes WBYTES 此进程导致发送到存储层的字节数。

wcbytes WCBYTES 取消的写字节数。

wchan WCHAN 进程在其中休眠的内核函数的名称。

wchars WCHARS 该任务已写入或将写入磁盘的字节数。

wops WOPS 系统调用的I/O操作数
```

## 三、使用实例

* 通过 `-o` 自定义指定列头

```shell
ps -e -o pid,user,comm

# Output:
# PID USER     COMMAND
# 1 root     systemd
# 2 root     kthreadd
# 3 root     rcu_gp
# ...
```

* 通过进程id筛选

```shell
ps -p 1234

# Output:
# PID TTY          TIME CMD
# 1234 pts/1    00:00:00 bash
```

* 通过用户筛选

```shell
ps -u root

# Output:
# PID TTY          TIME CMD
# 1 ?        00:00:02 systemd
# 2 ?        00:00:00 kthreadd
# 3 ?        00:00:00 rcu_gp
# ...
```

* 处理僵尸进程

```shell
ps -A -ostat,ppid,pid,cmd | grep -e '^[Zz]'

# Output:
# Z  1001 1234 [process-name] <defunct>

拿到ppid，杀死父进程，僵尸进程也会销毁
```

* 显示当前shell匹配的进程信息

```shell
ps -p $$

$$：当前shell的pid
```

* 列出所有进程并带有完整格式

```shell
ps -ef | less
```

* 通过进程名筛选

```shell
ps -C systemd
```

* 使用BSD风格显示所有进程

```shell
ps aux | less
```

* 组合 `grep` 来过滤

```shell
ps -ef | grep systemd
```

## 四、ps源码

![alt text](/images/ps-image.png)

## 五、man pages

![alt text](/images/ps-image-1.png)