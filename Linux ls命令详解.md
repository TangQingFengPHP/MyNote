### 简介：

ls是的作用是列出目录和文件，提供有用的信息，例如：文件名、属性权限、所属者、文件大小、修改时间等。

> 下面列出常用的的选项进行详解：

#### 列出详细信息

> `-l`

```shell
ls -l

列出长格式的详细内容，有：属性权限、所属组名、所属用户名、文件大小、修改时间

示例：
drwxr-xr-x    4 staff  staff        128  4 26 11:25 电子书
```

#### 列出所有文件，包括.和..

> `-a`或`--all`

```shell
ls -a
```

#### 列出所有文件，不包括.和..

> `-A`或`--almost-all`

```shell
ls -A
```

#### 通过文件最新修改时间排序

> `-t`

```shell
ls -lt
```

#### 按照文件大小从大到小排序

> `-S`

```shell
ls -lS
```

#### 反转字母表顺序排序

> `-r`或`--reverse`

```shell
ls -r
```

#### 递归列出目录文件，包括子目录

> `-R`或`--recursive`

```shell
ls -R abc
```

#### 以人类可读的方式显示文件大小

> `-h`或`--human-readable`

```shell
ls -lh
```

#### 列出目录本身的详细信息，不包括内容

> `-d`或`--directory`

```shell
ls -ld abc
```

#### 显示文件或目录的索引编号

> `-i`或`--inode`

```shell
ls -li
```

#### 在文件名后面追加指示符，标识此文件是：目录、可执行文件、符号链接、socket文件、FIFO文件。

> `-F`

```shell
ls -F

/：是目录
*：表示可执行文件
@：表示符号链接
=：表示socket文件
|：表示FIFO文件
没有符号表示是普通正常文件
```

#### 显示用户ID和群组ID

> `-n`或`--numeric-uid-gid`

```shell
ls -ln
```

#### 以"?"显示非字母字符或者叫隐藏控制字符

> `-q`或`--hide-control-chars`

```shell
ls -lq
```

#### 以逗号分割的方式显示目录文件

> `-m`

```shell
ls -m
```

#### 显示文件的大小

> `-s`或`--size`

```shell
ls -ls
```

#### 无序显示文件

> `-f`

```shell
ls -f
```

#### 打印帮助信息

> `--help`

```shell

ls --help
```

#### 打印版本信息

> `--version`

```shell
ls --version
```