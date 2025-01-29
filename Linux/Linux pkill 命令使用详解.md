### 简介

`pkill` 命令用于根据进程名称、用户、组或其他属性终止进程。它是 `procps-ng` 包的一部分，通常比 `kill` 更受欢迎，因为它无需查找进程 `ID (PID)`。

### 常用选项

* `-<signal>, --signal <signal>`：定义要发送给每个匹配进程的信号，可以使用数字或符号信号名。

* `-c, --count`：抑制正常输出，而是打印匹配进程的数量

* `-e, --echo`：显示被终止的进程的名称和 `PID`

* `-f, --full`：使用完整的命令行匹配

* `-g <group>`：匹配列出的进程组 `ID` 中的进程

* `-i, --ignore-case`：匹配进程不区分大小写

* `-l, --list-name`：列出进程名称以及进程 `ID`

* `-n, --newest`：仅选择最新的（最近启动的）匹配进程

* `-o, --oldest`：仅选择最旧的（最近最少启动的）匹配进程

* `-P, --parent <ppid>`：仅匹配列出了父进程 `ID` 的进程

* `-v, --inverse`：反转匹配

* `-x, --exact`：精准匹配

### 示例用法

#### 通过名称终止进程

> 终止所有 `Firefox` 进程

```shell
pkill firefox
```

#### 不区分大小写的匹配

```shell
pkill -i FiReFoX
```

#### 终止以特定用户身份运行的进程

```shell
pkill -u username processname
```

#### 终止某个用户的所有进程

```shell
pkill -u username
```

#### 通过完整命令行终止进程

> 匹配完整的命令行而不是仅仅匹配进程名称

```shell
pkill -f "python my_script.py"
```

#### 终止除特定进程之外的进程

> 终止除精确匹配之外的所有 `bash` 进程

```shell
pkill -v -x bash
```

#### 平滑终止进程

> 不强制终止，而是发送 `SIGTERM`（默认）信号来终止

```shell
pkill -15 processname
或
pkill -SIGTERM processname
```

#### 强制终止SIGKILL

```shell
pkill -9 processname
或
pkill -SIGKILL processname
```

#### 重载配置

```shell
pkill -HUP processname
```

#### 暂停进程

```shell
pkill -STOP processname
```

#### 恢复进程

```shell
pkill -CONT processname
```

#### 终止进程前确认

> 将列出 PID，但不会终止它们

```shell
pgrep processname
```

#### 根据进程年龄进行杀戮

> 终止运行时间超过1小时的进程

```shell
# 终止最老的进程实例
pkill -o processname

# 终止最新的进程实例
pkill -n processname
```

#### 使用正则表达式匹配

> 终止所有以 `fire` 开头的进程

```shell
pkill '^fire'
```

#### 以交互方式终止进程

```shell
ps aux | grep processname
pkill processname
```

#### `kill`、`pkill`、`killall` 三者区别

* `kill`：需要 `PID`

* `pkill`：使用进程名称、部分匹配并支持正则表达式

* `killall`：根据确切名称终止某个进程的所有实例

#### 重启一个进程

> 重新加载 `nginx` 配置

```shell
pkill -HUP nginx
```

#### 终止所有 `Python` 脚本

```shell
pkill -f python
```

#### 按组终止进程

```shell
pkill -G groupname
```

#### 以交互方式终止进程（执行前确认）

```shell
pgrep -a processname
pkill processname
```