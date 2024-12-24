### 简介

`xargs` 是一个功能强大的 `Linux` 命令，用于从标准输入构建和执行命令。它接受一个命令的输出，并将其作为参数提供给另一个命令。它在处理大量输入时特别有用，其含义可以解释为：`extended arguments`，使用 `xargs` 允许 `echo`、`rm`、`mkdir` 等命令接受标准输入作为参数。

### 与管道的对比

* 管道仅将一个命令的输出传递到下一个命令的输入。

* `xargs` 将输入（通常来自标准输出）转换为另一个命令的参数，这在有些命令不能使用标准输入作为参数时特别有用。

### 命令语法

```shell
xargs [options] [command]
```

### 常用选项

* `-0, --null`：将输入项视为由空字符分隔，而不是通常的空格或换行符分隔

* `-n, --max-args`：限制每个命令的参数数量

* `-d, --delimiter`：使用自定义字符作为分隔符

* `-I, --replace`：用输入的参数替换占位符

* `-L, --max-lines`：限制每个命令的输入行数

* `-P, --max-procs`：并行运行多个命令

* `-t, --verbose`：执行前打印命令

* `-r, --no-run-if-empty`：如果输入为空则不运行

* `-E`：设置文件结束字符串

* `-a file, --arg-file=file`：从文件而不是标准输入读取输入

* `-o, --open-tty`：将 `stdin` 重新打开为 `/dev/tty`，以供交互式应用程序使用

* `--process-slot-var=<name>`：设置每个子进程独有的环境变量。用于管理并行性

* `-s max-chars, --max-chars=max-chars`：限制每个命令的总字符数，包括参数

* `--show-limits`：显示系统对命令行长度施加的限制

* `--`：停止选项解析，处理以 `-` 开头的命令或参数时很有用

* `--help`：显示帮助信息

* `--version`：显示版本号

### 示例用法

#### 基础用法

```shell
echo "file1 file2 file3" | xargs rm

# 此命令删除 file1、file2 和 file3，echo 的输出作为参数传递给 rm
```

#### 查找并删除 `.log` 文件

```shell
find /path -name "*.log" | xargs rm -f
```

#### 限制参数的数量

```shell
echo "1 2 3 4 5 6" | xargs -n 2 echo

# 使用 -n 选项来限制传递给单个命令执行的参数数量

# 输出如下：一次输出两个
1 2
3 4
5 6
```

#### 并行运行命令

```shell
echo "file1 file2 file3 file4" | xargs -P 2 -n 1 touch

# 使用最多2个并行进程创建文件1、文件2、文件3和文件4
```

#### 使用占位符

```shell
echo "file1 file2" | xargs -I {} mv {} /new/location/

# 将 file1 和 file2 移动到 /new/location/
```

#### 处理带换行符的输入

```shell
echo -e "file1\nfile2\nfile3" | xargs -d '\n' rm

# xargs 默认处理空格分隔的输入，使用 -d 选项指定分隔符
```

#### 处理空字符

```shell
find /path -name "*.log" -print0 | xargs -0 rm

# 处理以空字符结尾的字符串
```

#### 限制每个命令的最大参数数量

```shell
echo "1 2 3 4" | xargs --max-args=2 echo
```

#### 搜索并压缩文件

```shell
find /path -name "*.txt" | xargs tar -czvf archive.tar.gz
```

#### 重命名多个文件为新的后缀名

```shell
ls *.txt | xargs -I {} mv {} {}.bak
```

#### `xargs` 多个命令

```shell
cat file4.txt | xargs -I % sh -c 'echo "%"; mkdir "%"'
```

#### 删除字符串中的空格

```shell
echo "  Line  with  spaces" | xargs

# 由于 xargs 在查找参数时会忽略空格，因此该命令对于从字符串中删除不必要的空格很有用
# 输入如下：
Line with spaces
```

#### `find` 结合 `xargs` 删除文件

> 方法一：

```shell
find ./foo -type f -name "*.txt" -exec rm {} \;
```

> 方法二：

```shell
find ./foo -type f -name "*.txt" | xargs rm
```

测试耗时对比：

```shell
time find . -type f -name "*.txt" -exec rm {} \;
0.35s user 0.11s system 99% cpu 0.467 total

time find ./foo -type f -name "*.txt" | xargs rm
0.00s user 0.01s system 75% cpu 0.016 total
```

显然使用 `xargs` 效率更高，使用 `xargs` 比使用 `exec {}` 效率大概高出六倍。

#### 在执行命令时提示用户运行它

```shell
echo 'one two three' | xargs -p touch

# 提示如下：
touch one two three ?...
```

#### `find` 结合 `xargs` 复制文件

```shell
find ./xargstest -type f -name '*.txt' | xargs -t -I % cp -a % ~/backups

# 执行以下命令：
cp -a ./xargstest/test2.txt /home/user/backups
cp -a ./xargstest/test1.txt /home/user/backups
```