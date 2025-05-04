### 简介

`watch` 命令会以固定间隔（默认每 2 秒）重复运行给定命令，并在终端上显示其输出。它非常适合监控不断变化的输出，例如磁盘使用情况、内存使用情况、文件更改、服务状态等。

### 基础语法

```shell
watch [options] command
```

### 常用选项

* `-n, --interval`：允许指定输出更新之间的间隔，单位：秒

* `-d, --differences`：突出显示输出更新之间的差异

* `-g, --chgexit`：当用户定义命令的输出发生变化时退出监视命令

* `-t, --no-title`：删除显示间隔、命令和当前时间和日期的标题

* `-b, --beep`：如果命令因错误退出，则播放声音警报（蜂鸣声）

* `-p, --precise`：尝试在 `--interval` 选项定义的精确秒数后运行命令

* `-e, --errexit`：出现错误时停止输出更新并在按下按键后退出命令

* `-c, --color`：解释 `ANSI` 颜色和样式序列

* `-x, --exec`：将用户定义的命令传递给 `exec`，减少额外引用的需要

* `-w, --no-linewrap`：关闭换行并截断长行

* `-h, --help`：显示帮助文本并退出

* `-v, --version`：显示版本信息并退出

### 示例用法

#### 每 5 秒显示一次系统时间和日期

```shell
watch -n 5 date
```

#### 以默认的 2 秒间隔显示系统日期和时间，并突出显示更改

```shell
watch -d date
```

#### 变更时退出

```shell
watch -g free
```

#### 隐藏监视命令标头

```shell
watch -t date
```

#### 用于用户自定义的复杂命令参数

* 使用 `\` 来换行

```shell
watch -n 5 \
echo "watch command example output"
```

* 使用引号括起来

```shell
watch -n 5 'echo "watch command example output"'
```

#### 监控内存使用情况

```shell
watch -n 1 free -h
```

#### 检查进程是否正在运行

```shell
watch pgrep nginx
```

#### 观察 CPU 消耗最高的 5 个进程

```shell
watch -n 1 "ps -eo pid,comm,%cpu --sort=-%cpu | head -n 6"
```

#### 监控文件夹文件数

```shell
watch "ls | wc -l"
```

#### 突出显示更改

```shell
watch -d ifconfig
```

#### 与 grep 结合以获得过滤输出

```shell
watch "ps aux | grep nginx"
```

#### 使用颜色使其更具可读性

```shell
watch -c "ls --color=always"
```

#### 监控日志

```shell
watch tail -n 20 /var/log/syslog
```

对于动态日志，`tail -f` 比 `watch` 更合适

#### 观察CPU动态频率

```shell
 watch -n1 'grep "^cpu MHz" /proc/cpuinfo | sort -nrk4'
```