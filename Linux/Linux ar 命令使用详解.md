### 简介

`Linux` 中的 `ar（Archive）` 命令用于创建、修改和从存档中提取文件。它通常用于在 `C/C++` 开发中创建静态库（`.a` 文件）。

### 基础语法

```shell
ar [options] archive-file file(s)
```

* `archive-file`：要创建/修改的档案的名称

* `file(s)`：要添加到档案的文件

* `[options]`：控制操作

### 常用选项

* `c`：创建一个新的档案（如果不存在）

* `r`：替换或添加文件到档案中

* `d`：从档案中删除文件

* `t`：列出档案的内容

* `x`：从档案中提取文件

* `v`：详细模式（显示详细信息）

### 示例用法

#### 创建存档文件

```shell
ar rcs libexample.a file1.o file2.o
```

* `r`：添加/替换文件

* `c`：如果档案不存在则创建该档案

* `s`：添加索引以便更快地查找符号

示例：

```shell
gcc -c file1.c file2.c
ar rcs libexample.a file1.o file2.o

# 从 file1.o 和 file2.o 创建静态库 libexample.a
```

#### 列出存档内容

```shell
ar t libexample.a
```

示例输出：

```shell
file1.o
file2.o
```

详细列出：

```shell
ar tv libexample.a
```

#### 提取文件

```shell
ar x libexample.a file1.o
```

提取所有文件：

```shell
ar x libexample.a
```

#### 从存档中删除文件

```shell
ar d libexample.a file1.o

# 从 libexample.a 中删除 file1.o
```

#### 更新存档文件

```shell
ar r libexample.a file1.o
```

#### 在编译中使用静态库（.a）

```shell
gcc main.c -L. -lexample -o myprogram
```

* `-L.`：在当前目录中查找库

* `-lexample`：与 `libexample.a` 链接