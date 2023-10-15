### 简介

查看文件内容并打印到标准输出中（终端）

全称：Concatenate FILE(s) to standard output

#### 一般用法

```shell
cat abc.txt
```

#### 显示行号

> `-n`或`--number`

```shell
cat -n abc.txt
```

#### 查看文件并写入其他文件中

```shell
cat a.txt b.txt > c.txt
```