### 简介

`pidof` 命令用于查找 `Linux` 中正在运行的程序的进程 `ID (PID)`。它有助于管理和控制进程。

### 基本语法

```shell
pidof [options] program_name
```

### 常用选项

* `-s`：单次 - 指示程序仅返回一个 pid

* `-q`：安静模式，抑制任何输出并仅相应地设置退出状态

* `-w`：还显示没有可见命令行的进程 （例如内核工作线程）

* `-x`：这会导致程序也返回运行指定脚本的 `shell` 的进程 ID

* `-o <omitpid>`：告诉 `pidof` 忽略具有该进程 ID 的进程

* `-t`：显示所有线程 id 而不是 pid

* `-S <separator>`：使用指定的分隔符作为 pid 之间的分隔符。仅当为程序打印多个 pid 时使用

#### 示例用法

##### 获取正在运行的程序的 PID

```shell
pidof bash

# 示例输出：1234
```

#### 获取多个实例的 PID

```shell
pidof firefox

# 如果有多个实例正在运行，它将返回多个 PID：4567 8901
```

#### 获取系统守护进程的 PID

```shell
pidof systemd
```

#### 仅显示一个 PID

```shell
pidof -s python
```

#### 排除特定 PID

```shell
pidof -o 4567 firefox
```

#### 包含 Shell 脚本

```shell
pidof -x myscript.sh

# 查找脚本和程序的 PID
```

#### 将 ps 与 grep 结合使用

```shell
ps aux | grep nginx | grep -v grep
```

#### 使用 pgrep

```shell
pgrep nginx
```

#### 将 ps 与 awk 结合使用

```shell
ps -e | awk '/nginx/ {print $1}'
```

#### 使用 pidof 终止进程

```shell
kill $(pidof firefox)
```

#### 重新启动进程

```shell
kill -HUP $(pidof nginx)
```