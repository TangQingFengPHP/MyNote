### 简介

`Linux` 中的 `cut` 命令是一个命令行实用程序，用于从文件或标准输入中提取文本行的部分。当希望从文件或数据流中提取特定字段或列时，例如处理以逗号分隔或制表符分隔的文件时，它非常有用。

### 基础语法

`cut` 命令通过指定分隔符（例如空格、制表符或特定字符）并选择想要显示的列或字段来工作

```shell
cut OPTION... [FILE]...
```

### 常用选项

* `-b, --bytes=LIST`：通过指定一个字节、一组字节或一个字节范围进行选择

* `-c, --characters=LIST`：通过指定一个字符、一组字符或一个字符范围进行选择

* `-d, --delimiter=DELIM`：指定将用来代替默认“TAB”分隔符的分隔符

* `-f, --fields=LIST`：仅选择这些字段；还打印任何不包含分隔符的行，除非指定了 -s 选项

* `--complement`：补充选择。使用此选项时，cut 将显示除所选内容之外的所有字节、字符或字段

* `-s, --only-delimited`：不打印不包含分隔符的行

* `--output-delimiter=STRING`：cut 的默认行为是使用输入分隔符作为输出分隔符。此选项允许指定不同的输出分隔符字符串

### 范围选择

* `N`：第 N 个字节、字符或字段，从 1 开始计数

* `N-`：从第 N 个字节、字符或字段到行尾

* `N-M`：从第 N 到第 M (含) 个字节、字符或字段

* `-M`：从第一个到第 M 个（含）字节、字符或字段

### 示例用法

#### `-f`：字段选择

此选项用于指定要提取哪些字段。字段由分隔符分隔（通常是制表符或空格，但可以使用 `-d` 选项指定任何分隔符）。

示例：要从文件中提取第一列和第三列

```shell
cut -f 1,3 filename
```

#### `-d`：分隔符

此选项指定分隔字段的分隔符。默认情况下，`cut` 假定字段由制表符分隔，但可以指定其他分隔符，如逗号、冒号或空格

示例：要从逗号分隔文件 (CSV) 中提取字段

**csv文件**

```csv
Name,Age,Location
Alice,30,New York
Bob,25,Los Angeles
Charlie,35,Boston
```

```shell
cut -d ',' -f 1,3 filename
```

**示例输出**

```csv
Name,Location
Alice,New York
Bob,Los Angeles
Charlie,Boston
```

#### `-c`：字符选择

这个选项允许从每行中提取特定字符。可以指定要提取的字符位置（或字符范围）

示例：提取每行位置 1 至 5 的字符

```shell
cut -c 1-5 filename
```

#### `-b`：字节选择

此选项允许根据字节而不是字符来截断输入。当处理面向字节的数据（例如二进制文件）时，此功能非常有用。

```shell
cut -b 1-5 filename
```

#### `--complement`：反向选择

该选项允许补充选择，这意味着它不是选择指定的字段，而是将其排除

示例：排除第一列（字段）并显示其余部分

```shell
cut -f 1 --complement filename
```

#### `-s`：禁止使用无分隔符的行

此选项会隐藏不包含分隔符的行。如果想要排除缺少分隔符的行，此选项非常有用

示例：从文件中提取字段并忽略没有分隔符的行

```shell
cut -d ',' -f 1 -s filename
```

#### 提取特定字符

有一个字符串并想提取前 3 个字符

```shell
echo "abcdefg" | cut -c 1-3
```

**输出**

```shell
abc
```

#### 提取多个字符范围

要提取多个范围的字符（例如，字符 1-3 和 6-8）

```shell
echo "abcdefg" | cut -c 1-3,6-8
```

**输出**

```shell
abcfg
```

#### 使用 cut 和 ps 列出进程

可以使用 cut 从 ps 命令输出中提取特定信息

例如：提取进程ID和正在运行的进程的命令

```shell
ps aux | cut -d ' ' -f 1,11
```

#### 使用--complement排除字段

要从 `passwd` 文件中排除第一个字段（用户名）

```shell
cut -d ':' -f 1 --complement /etc/passwd
```

#### 从 ls 的输出中提取特定列

此命令列出了文件和目录，但只输出它们的名称（ls -l 输出中的第 9 列）

```shell
ls -l | cut -d ' ' -f 9
```

#### 获取当前目录中文件的磁盘使用情况

这将仅输出每个文件或目录的大小，不包括路径信息

```shell
du -h | cut -f 1
```