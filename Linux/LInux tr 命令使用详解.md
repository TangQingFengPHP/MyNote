### 简介

`tr （translate）`命令用于在 `Linux` 中翻译或删除输入流（通常是 `stdin` ）中的字符。它主要用于文本操作，并且可以作为转换或删除文本文件或流中的特定字符的方便工具。

### 基本语法

```shell
tr [OPTION] [SET1] [SET2]
```

* `SET1`：要替换或删除的字符集

* `SET2`：将替换 `SET1` 中的字符的字符集

### 常用选项

* `-d`：删除 `SET1` 中的字符

* `-s`：挤压 `SET1` 中的连续相同字符

* `-c`：对 `SET1` 中的字符进行补充（反匹配）

### 示例用法

#### 将小写字母转换为大写字母

```shell
echo "hello world" | tr 'a-z' 'A-Z'
```

**输出**

```shell
HELLO WORLD
```

#### 将大写字母转换为小写字母

```shell
echo "HELLO WORLD" | tr 'A-Z' 'a-z'
```

**输出**

```shell
hello world
```

#### 删除指定字符

> 将从输入中删除所有数字

```shell
echo "hello 123 world" | tr -d '0-9'
```

**输出**

```shell
hello  world
```

#### 挤压(删除)重复字符

> 使用 `-s` 选项将多个连续出现的字符替换为单个字符

```shell
echo "aaabbbccc" | tr -s 'a-c'
```

**输出**

```shell
abc
```

#### 用另一个字符替换一个字符

```shell
echo "hello world" | tr ' ' '_'
```

**输出**

```shell
hello_world
```

#### 删除换行符

> 即将多行输入转换为单行

```shell
echo -e "hello\nworld\n" | tr -d '\n'
```

**输出**

```shell
helloworld
```

#### 转换特殊字符

> 将空格转换为制表符

```shell
echo "hello world" | tr ' ' '\t'
```

**输出**

```shell
hello    world
```

#### 转换文件中的文本

> 读取 `input.txt` 文件，将所有小写字母转换为大写，并将结果写入 `output.txt`

```shell
tr 'a-z' 'A-Z' < input.txt > output.txt
```

#### 从文件中删除特定字符

> 将从 `input.txt` 文件中删除所有元音 `(a、e、i、o、u)`，并将结果写入 `output.txt`

```shell
tr -d 'aeiou' < input.txt > output.txt
```

#### 使用 `-c` 选项对 `SET1` 中的字符进行补充

> 删除除数字之外的所有字符

```shell
echo "Your PIN is: 1234" | tr -cd [:digit:]
```

**输出**

```shell
1234
```

#### 删除所有非字母字符

```shell
echo "Hello, World! 123" | tr -cd 'a-zA-Z'
```

**输出**

```shell
HelloWorld
```

#### 将句子转换为除第一个字母外的其他字母为小写

```shell
echo "HELLO WORLD" | tr 'a-z' 'A-Z' | tr 'A-Z' 'a-z' | sed 's/^\(.\)/\U\1/'
```

**输出**

```shell
Hello world
```