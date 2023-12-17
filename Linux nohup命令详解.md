### 一.命令介绍

1. nohup 英文全称 no hang up（不挂起），用于在系统后台不挂断地运行命令，退出终端不会影响程序的运行

2. nohup 命令，在默认情况下（非重定向时），会输出一个名叫 nohup.out 的文件到当前目录下，如果当前目录的 nohup.out 文件不可写，输出重定向到 $HOME/nohup.out 文件中。

### 二、语法格式

> nohup Command [ Arg … ] [　& ]

解释：

1. Command：要执行的命令。
2. Arg：一些参数，可以指定输出文件。
3. &：让命令在后台执行，终端退出后命令仍旧执行。

### 三.命令参数

1. --help：显示帮助信息
2. --version：显示版本信息
3. -p [`pid`]：指定进程id来运行进程

### 四.命令用法及示例

1. 后台运行命令，并把输出的信息重定向到指定的文件
```
nohup [mycommamd] > nohup_output.log 2>&1 & 
```
2. 后台运行命令，忽略输出的信息，重定向到垃圾桶
```
nohup [mycommamd] > /dev/null 2>&1 & 
```
3. 执行多个命令 
```
nohup bash -c 'cal && ls'
```

### 五.查看进程id及终止进程

使用nohup 执行命令后会返回任务id和进程id，如：[1] 80132，其中1为任务id，80132为进程id

1. 查找进程

```shell
ps -aux | grep [mycommand.sh]

pgrep -a [mycommand.sh]

jobs -l
```

2. 终止进程

```shell
kill -9 [PID]
```

### 六.退出码

1. 125 表示nohup命令自身执行失败
2. 126 表示命令可以查找到但不能被调用
3. 127 表示命令找不到

### 七.扩展及相关命令

1. 使用screen工具可达到同样的目的
2. 或者使用tmux这个款现代化工具更棒
3. 或者使用supervisor更平滑的管理进程

