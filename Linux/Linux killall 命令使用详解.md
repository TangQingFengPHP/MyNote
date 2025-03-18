### 简介

`Linux` 中的 `killall` 命令用于按名称终止所有进程。与需要进程 ID (`PID`) 的 `kill` 不同，`killall` 通过指定进程名称来工作。

`killall` 向运行任何指定命令的所有进程发送信号。如果没有指定信号名称，则发送 `SIGTERM`。

信号可以通过名称（例如 `-HUP` 或 `-SIGHUP`）或数字（例如 `-1`）或选项 `-s` 来指定。

`killall` 进程永远不会终止自身（但可能会终止其他 `killall` 进程）。

### 基础语法

```shell
killall [OPTIONS] process_name

# process_name 表示进程名称
```

### 常用选项

* `-e, --exact`：对于非常长的名称要求精确匹配。

* `-I, --ignore-case`：忽略大小写匹配

* `-g, --process-group`：终止该进程所属的进程组。`killall` 信号每个组仅发送一次，即使发现多个 进程属于同一进程组

* `-i, --interactive`：在杀死之前以交互方式请求确认

* `-l, --list`：列出所有已知的信号名称

* `-o, --older-than`：只匹配较旧（在此之前启动）的进程

```shell
时间被指定为浮点数 
可用的单位有：s m h d w m y
分别表示秒、分、小时、日、周、月、年
```

* `-q, --quiet`：安静模式，如果没有进程被终止不会输出任何东西

* `-r, --regexp`：将进程名称模式解释为 `POSIX` 扩展正则表达式

* `-s, --signal, -SIGNAL`：发送指定信号而不是默认的 `SIGTERM`

* `-u, --user`：仅终止指定用户拥有的进程，命令名称是可选的

* `-v, --verbose`：报告信号是否成功发送

* `-y, --younger-than`：只匹配较年轻（在进程之后启动）的进程，时间单位同上

 
### 示例用法

#### 通过名称终止进程

```shell
killall firefox

# 此命令将终止所有 Firefox 实例
```

#### 强制终止进程

```shell
killall -9 firefox

# -9 选项发送 SIGKILL 信号，强制进程立即终止
```

#### 终止属于特定用户的进程

```shell
killall -u username

# 这将终止属于用户名的所有进程
```

#### 以交互方式终止进程

```shell
killall -i firefox

# 这会在终止每个进程之前要求确认
```

#### 仅终止特定时间之前的正在运行的进程

```shell
killall -o 1h firefox

# 这将终止运行超过 1 小时的 Firefox 进程
```

#### 仅终止特定时间之前正在运行的进程

```shell
killall -y 10m firefox

# 这将终止过去 10 分钟内启动的 Firefox 进程
```

#### 列出可用信号

```shell
killall -l
```

#### 检查是否成功

```shell
killall -v firefox

# -v 选项使 killall 详细显示有关终止进程的信息
```