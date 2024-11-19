### 简介

`sed` 的全称是：`Stream Editor` 流编辑器，在 `Linux` 中是一个强大的文本处理工具，可以处理文件或标准输入流。

### 基本语法

```shell
sed [options] 'command' file
```

通过管道传输入流：

```shell
echo "text" | sed 'command'
```

### 常用子命令

#### 文本替换(`s`)

```shell
sed 's/old/new/' file

# s代表文本替换

# old表示被替换的旧文本

# new表示替换的新文本
```

* 替换第一次匹配到的文本

```shell
sed 's/Linux/Unix/' file
```

* 替换所有匹配到的文本(`g`)

```shell
sed 's/Linux/Unix/g' file
```

* 替换时不区分大小写(`i`)

```shell
sed 's/linux/unix/gi' file
```

#### 删除文本行(`d`)

```shell
sed '<n>d' file

# n代表第几行
```

* 删除指定行的文本

```shell
sed '2d' file
```

* 删除指定范围行的文本

```shell
sed '5,10d' file
```

* 删除模式匹配到的行

```shell
sed '/pattern/d' file
```

#### 追加文本(`a`)

```shell
sed '/pattern/a\new text' file
```

* 在模式匹配到的文本下面添加一行文本

```shell
sed '/Linux/a\This is added text' file
```

#### 插入文本(`i`)

```shell
sed '/pattern/i\new text' file
```

* 在模式匹配到的文本上面添加一行文本

```shell
sed '/Linux/i\This is inserted text' file
```

#### 替换指定行的内容(`c`)

```shell
sed '<n>c\new content' file

# n表示第几行
```

```shell
sed '3c\new content' file
```

#### 打印与模式匹配的行文本(`p`)

```shell
sed -n '/pattern/p' file

# -n 表示抑制默认的打印输出，不加此选项会打印两遍
# p 表示打印文本
```

```shell
sed -n '/Linux/p' file
```

#### 修改后覆盖源文件(`-i`)

```shell
sed -i 's/old/new/g' file
```

```shell
sed -i 's/Linux/Unix/g' file
```

#### 同时执行多个 `sed` 子命令(`-e`)

使用 `-e` 选项或把命令放在文件中

```shell
sed -e 's/old/new/' -e '/pattern/d' file
```

新建一个文件，`sed_commands.sed`

```shell
s/old/new/
5,10d
```

通过 `-f` 指定 `sed` 命令文件

```shell
sed -f sed_commands.sed file
```

#### 替换指定行模式匹配到的文本

```shell
sed '3s/old/new/' file
```

#### 删除空白行

```shell
sed '/^$/d' file

# ^表示行首，$表示行尾
# ^$合在一起，就表示行首和行尾之间没有任何内容，即空白行
```

#### 添加行号

```shell
sed = <file> | sed 'N;s/\n/\t/'

# 第一步：sed = <file> 会生成行号，且打印如下的文本格式：
# 1
# Line 1 content
# 2
# Line 2 content

# 第二步：通过 | 管道传到下一个sed命令

# 第三步：N子命令合并当前行和下一行的内容，如下：
# 1\nLine 1 content

# 第四步：替换 \n 为 \t，如下：
# 1\tLine 1 content

# 最终输入结果：
# 1   Line 1 content
# 2   Line 2 content
```

#### 提取指定行范围的文本

```shell
sed -n '5,10p' file

# 打印第5到第10行的文本
```

#### 替换指定行范围的文本

```shell
sed '5,10s/old/new/g' file

# 替换仅在第5到第10行模式匹配的文本
```

#### 移除行尾的空格

```shell
sed 's/[ \t]*$//' file

# [ \t] 表示匹配空格或者tab(\t)
# * 表示匹配0个或多个
# $ 表示行尾
# // 表示替换成空
```

#### 同时操作多个文件

```shell
sed 's/old_string/new_string/g' filename1.txt filename2.txt
```

#### 结合 `find` 命令操作

```shell
find /file -type f -exec sed -i 's/old_string/new_string/g' {} \;
```

#### 操作后追加到新文件

```shell
sed -n 's/pattern/p' logfile.log > extracted_data.txt
```

### 多种转义字符的使用

* 使用反斜杠 `\`

```shell
sed 's/\/old\/path/\/new\/path/' file
```

* 使用竖线或者叫管道符号 `|`

```shell
sed 's|/old/path|/new/path|' file
```

