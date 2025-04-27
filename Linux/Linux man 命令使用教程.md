### 简介

`man` = `manual(手册)` 命令用来查看 `Linux` 系统命令、函数、配置文件、系统调用等的官方文档。几乎所有标准程序和工具都有对应的 `man` 手册。

### 基本语法

```shell
man [options] [section number] [command name]
```

* `options`：选项

* `section number`：章节编号

* `command name`：要搜索的命令名称

### 基本用法

* `man <命令名>`：查看某个命令的手册

* `man <数字> <命令名>`：指定章节查看，比如系统调用、库函数等

* `man -k <关键词>`：模糊搜索相关手册（相当于搜索标题）

* `man -f <命令名>`：快速查看该命令属于哪一类

### 常用选项

* `-k`：在手册页描述中搜索关键字。

* `-f`：显示指定命令的部分编号和简要说明。

* `-a`：显示指定命令的所有可用手册页。

* `-w`：显示手册页文件的路径但不显示其内容。

* `--help`：显示帮助消息，列出 man 命令的可用选项。

* `-P [pager]`：指定用于显示手册页的分页程序。

* `-s [num]`：仅在指定的手册章节内搜索。

### man 命令输出部分

当查看特定的手册页时，其内容会被组织成几个输出部分，以便以逻辑格式呈现详细信息。这些部分包括：

* `NAME`：命令或函数的名称及其简要说明。提供手册页所涵盖主题的简明概述。

* `SYNOPSIS`：使用命令或函数的语法，包括选项和参数。

* `DESCRIPTION`：详细说明命令或函数的功能。提供上下文、示例和用例。

* `OPTIONS`：与命令一起使用的选项列表及其含义。

* `EXIT STATUS`：命令返回的退出代码用于指示成功或错误。有助于调试和脚本编写。

* `EXAMPLES`：演示命令用法的实际示例。

* `FILES`：相关文件的列表，例如配置或日志文件。

* `ENVIRONMENT`：环境变量。

* `SEE ALSO`：相关命令或主题的参考。

* `HISTORY`：系统命令或功能出现的版本，或在其操作中发生重大变化的版本的简要摘要。

* `BUGS`：命令或程序的已知问题或限制。

* `COPYRIGHT`：许可和版权详情。

* `AUTHORS`：感谢开发者或贡献者。

### 示例用法

#### 查看 ls 的官方文档

```shell
man ls
```

#### 查看系统调用（比如 open）

```shell
man 2 open
```

> 2 是系统调用章节，常用于开发者查函数接口。

#### 模糊搜索手册，找出所有包含 copy 的命令

```shell
man -k copy
```

**示例输出**

```shell
cp (1)             - copy files and directories
strncpy (3)        - copy a fixed-size string
```

#### 快速查看命令类型

```shell
man -f ls
```

相当于 `whatis ls`，输出类似：

```shell
ls (1) - list directory contents
```

### man 手册章节解释

* `1`：用户命令（如 `ls, cp`）

* `2`：系统调用（如 `open, read`）

* `3`：库函数（如 `printf, malloc`）

* `4`：设备文件和驱动

* `5`：配置文件格式（如 `/etc/passwd`）

* `6`：游戏和娱乐

* `7`：杂项（如协议、文件格式描述）

* `8`：系统管理命令（如 `mount, ifconfig`）

* `9`：内核例程，面向 `Linux` 内核开发者。包含内核级例程和函数的文档。

### 在 man 页面内常用快捷键

> 内置 `vim` 模式

* 空格键：向下翻页

* `b`：向上翻页

* `/关键字`：搜索关键字

* `n`：跳到下一个搜索结果

* `q`：退出 man 页面

### 高阶用法

#### 查看不同章节的冲突命令

比如 `printf` 既有 `Shell` 内置的，也有 `C` 语言库函数：

```shell
man 1 printf   # 查看 Shell 命令
man 3 printf   # 查看 C 标准库函数
```

#### 输出为纯文本（便于复制）

```shell
man ls | col -b > ls_help.txt
```

* `col -b` 去除控制字符。

* 把 `ls` 的 `man` 文档保存为纯文本文件。

#### 修改 man 页面语言

比如改成英文（避免中文翻译不准）：

```shell
LANG=C man ls
```

> 临时设置环境变量 LANG 为 C（英文原版）

### 自定义编写简单 man 手册

`man` 页基本格式：`Groff/troff` 格式

#### 简单示例

给 `mycmd` 的脚本写 `man` 页面

```roff
.TH MYCMD 1 "April 30, 2025" "1.0" "MyCMD Manual"
.SH NAME
mycmd \- a simple example command
.SH SYNOPSIS
.B mycmd
[\fIoptions\fR] [\fIarguments\fR]
.SH DESCRIPTION
This is a simple command for demonstration purposes.
.SH OPTIONS
.TP
.B \-h, --help
Show help message and exit.
.TP
.B \-v, --version
Display version information.
.SH AUTHOR
Written by ChatGPT.
```

* `.TH`：标题（命令名、章节号、日期、版本、标题）

* `.SH`：节标题（Section Header）

* `.B`：加粗（Bold）

* `.I`：斜体（Italic）

* `.TP`：开始一个选项段落（indented paragraph）

#### 保存成文件

```shell
nano mycmd.1
```

#### 查看效果

本地直接用 `man` 打开：

```shell
man ./mycmd.1
```

#### 安装到系统（让系统自动识别）

* 拷贝到 `man` 目录

```shell
sudo cp mycmd.1 /usr/share/man/man1/
```

* 更新 `man` 数据库

```shell
sudo mandb
```

* 全局使用

```shell
man mycmd
```

### 官方风格 man page 编写模板

```roff
.TH COMMAND_NAME 1 "日期" "版本" "标题"
.SH NAME
command_name \- 简短的一句话描述命令
.SH SYNOPSIS
.B command_name
[\fIOPTIONS\fR] [\fIFILES\fR] ...
.SH DESCRIPTION
详细描述这个命令的用途、特点、使用场景。

可以分成多段落，分别说明功能、注意事项等。
.PP
支持 Markdown 风格段落，空行后换段落。

.SH OPTIONS
.TP
.BR -h ", " --help
显示帮助信息并退出。
.TP
.BR -v ", " --version
显示版本信息并退出。
.TP
.BR -o " " OUTPUT
指定输出文件路径。
.TP
.BR -f ", " --force
强制执行，不进行确认提示。

.SH EXAMPLES
.PP
普通用法：
.PP
.B
command_name input.txt
.PP
带选项用法：
.PP
.B
command_name -o output.txt input.txt

.SH FILES
.PP
/etc/command_name.conf
.br
默认的配置文件路径。

.SH ENVIRONMENT
.PP
.B COMMAND_NAME_CONFIG
环境变量，指定配置文件路径。

.SH SEE ALSO
.BR other_command (1),
.BR another_tool (1)

.SH AUTHOR
Written by Your Name <your.email@example.com>

.SH COPYRIGHT
Copyright (C) 2025 Your Name.
License GPLv3+: GNU GPL version 3 or later.
```

#### 指令说明

* `.TH`：文档标题（名字、章节、日期、版本、描述）

* `.SH`：标题（一级小节，比如 NAME、SYNOPSIS）

* `.PP`：新段落（Paragraph）

* `.TP`：新条目（比如一个选项）

* `.B`：加粗（命令本身或重要内容）

* `.BR`：加粗多个内容，中间逗号分隔

* `.I`：斜体（一般参数、文件名）

* `.IR`：斜体多个内容，中间逗号分隔

* `.br`：强制换行

