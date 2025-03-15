### 简介

`Linux` 和 `Unix` 都提供各种开箱即用的 `shell` 。可以找到 `bash （Bourne Again shell）`、ksh `（Korn shell）`、`csh (C shell)/tcsh （TC shell）`、`sh （Bourne shell）`等默认安装的 `shell`。但是，如何检查我使用的是哪个 `shell` ？

### 方法

#### 使用 $0（最佳方法）

```shell
echo $0

# $0 包含当前正在运行的 shell 或脚本的名称
# 如果在交互式 shell 中运行，它会显示 shell 名称（bash、zsh 等）
# 如果运行脚本，它会显示脚本的文件名
```

* 显示当前正在运行的 `shell` 的名称

* 示例输出：`/bin/bash、zsh、fish`

#### 使用 $SHELL（默认登录 Shell）

```shell
echo $SHELL
```

* 显示用户设置的默认 `shell`（不一定是当前 `shell` ）

#### 使用 ps 命令

```shell
ps -p $$

# $$ 保存当前 shell 会话的进程 ID (PID)
# 如果在脚本中使用，它会提供脚本 shell 的 PID
```

* 显示当前 `shell` 的进程

* 示例输出

```shell
 PID TTY          TIME CMD
```

#### 使用 ps 命令直接输出shell名称

```shell
ps -o comm= -p $$
```

#### 使用带有基本名称的 echo $0

```shell
basename "$0"

# 显示不带完整路径的 shell 名称
```

#### 使用 readlink 获取

```shell
readlink /proc/$$/exe
```

##### 查看系统上安装的所有shell

```shell
cat /etc/shells
```

示例输出

```shell
/bin/bash
/bin/csh
/bin/dash
/bin/ksh
/bin/sh
/bin/tcsh
/bin/zsh
```

#### 使用 grep 查看

```shell
grep "^$USER" /etc/passwd
```

#### 使用 lsof 查看

```shell
lsof -p $$
```