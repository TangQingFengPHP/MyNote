### 简介

`Linux` 中的 `diff` 命令用于逐行比较文件。它以各种格式报告差异，广泛应用于脚本编写、开发和补丁生成。

### 基础语法

```shell
diff [OPTION]... FILES
```

### 常用选项

* `-i`：忽略大小写

* `-u`：打印输出时不包含任何多余的上下文行

* `-c`：输出不同行周围的几行上下文

* `-a / --text`：将文件视为文本并逐行进行比较

* `-b / --ignore-space-change`：比较文件时忽略空格

* `--binary`：以二进制模式比较和写入数据

* `-e / --ed`：使输出成为有效的 `ed` 脚本

* `-E / --ignore-tab-expansion`：比较文件时忽略标签扩展名

* `-N / --new-file`：将丢失的文件视为存在但为空

* `-q / --brief`：输出文件是否不同，无需指定详细信息

* `-s / --report-identical-files`：当文件相同时输出

* `-w / --ignore-all-space`：比较文件时忽略空格

* `--version`：打印版本信息

* `--help`：打印帮助信息

### 示例用法

#### 比较两个文本文件

```shell
diff file1.txt file2.txt
```

#### 递归比较目录

```shell
diff -r dir1/ dir2/
```

#### 简要比较

只说明文件是否不同或相同

```shell
diff --brief file1 file2
```

#### 报告相同文件

如果文件相同则明确打印

```shell
diff -s file1 file2
```

#### 补丁文件生成

* 创建补丁

```shell
diff -u original.txt updated.txt > changes.patch
```

* 应用补丁

```shell
patch original.txt < changes.patch
```

#### 添加颜色高亮差异

* 安装 `colordiff`

```shell
sudo apt install colordiff
```

* 使用

```shell
colordiff file1 file2
```

### 输出格式

#### 正常模式(默认)

```shell
diff file1 file2
```

* 以 `<` 开头的行指的是第一个文件中的内容

* 以 `>` 开头的行指的是第二个文件中的内容

* 行号：与第一个文件相对应

* 使用 `a、c、d` 表示添加/更改/删除，指示需要如何编辑第一个文件才能与第二个文件匹配

* 人类可读性较差，但更适合脚本编写

#### 上下文格式

```shell
diff -c file1 file2
```

* 每个块以 `***` 和 `---` 开头

* 以 `***` 开头的行，提供有关第一个文件的时间戳和信息

* 以 `___` 开头的行，提供时间戳和第二个文件的相关信息

* `****************` 表示分隔符

* 符号说明：
    * `-`：表示要从第一个文件中删除的内容
    * `+`：表示要添加到第一个文件的内容
    * `!`：表示从第二个文件改到相应行的内容

* 显示有关变化的 3 行上下文

* 适用于较旧的补丁工具

#### 统一格式

```shell
diff -u file1 file2
```

* 更紧凑、现代，由 `Git` 使用

* 省略上下文行

* 行范围指示，`@@ line-ranges @@` 用于描述行范围

* 显示带有以下前缀的行
    * `-` 表示从文件1中删除
    * `+` 表示在文件2中添加
    * 无前缀表示未变更

**输出示例**

```diff
@@ -1,3 +1,3 @@
-line 1
+line one
 line 2
```

#### 并排格式

```shell
diff -y file1 file2
```

* 并排显示两个文件

* 差异以 `|、< 或 >` 标记

```shell
line 1                             | line one
line 2                             line 2
```

添加 `--suppress-common-lines` 以仅显示差异

### 各模式比较详细示例

#### 创建两个待比较的文本文件

* `example1.txt`

```txt
Apple
Orange
Banana
Watermelon
Chery
```

* `example2.txt`

```txt
Orange
Peach
Apple
Banana
Melon
Cherry
```

#### 使用正常模式比较

```shell
diff example1.txt example2.txt
```

![alt text](/images/Linux/diff-image-1.png)

**输出解释：**

* `1d0`：第一个文件的第一行 (1) 应该被删除 (d)。如果没有删除，它将出现在第二个文件的第 0 行

* `< Apple`：需要删除的内容（如 `1d0` 所示）

* `2a2,3`：在第一个文件的第 2 行中，添加 (a) 第二个文件的第 2 行和第 3 行 (2,3)。

* `> Peach, > Apple`：需要添加的内容（如 `2a2,3` 所述）

* `4c5`：第一个文件中的第四行（4）应更改（c）为第二个文件中的第五行（5）

* `< Watermelon`：需要更改的内容

* `> Melon`：需要将其更改为什么

#### 使用上下文模式比较

```shell
diff -c example1.txt example2.txt
```

![alt text](/images/Linux/diff-image-2.png)

**输出解释**

* 前两行：显示两个文件的名称和时间戳

* `****************`：用作分隔符

* 两条信息线：显示有关第一个和第二个文件的信息，以 `***` 和 `---` 开头

* `*** 1,6 **** and --- 1,7 ----`：指示文件的行范围

* 文件内容：每行开头指示如何修改 `example1.txt` 以使其与 `example2.txt` 相同
    * `-`：需要从第一个文件中删除它
    * `+`：需要被添加到第一个文件中
    * `!`：需要将其更改为第二个文件中的相应行

因此，在上面的例子中，从第一行删除 Apple，在第四行用 Melon 替换 Watermelon，并在第二行和第三行添加 Peach 和 Apple

#### 使用统一（更紧凑）的模式比较

```shell
diff -u example1.txt example2.txt
```

![alt text](/images/Linux/diff-image-3.png)

**输出解释**

* 显示文件信息的行：第一个文件信息以 `---` 开头，而表示第二个文件的行以 `+++` 开头

* 前两行：显示两个文件的名称和时间戳

* `@@ -1,5 +1,6 @@`：显示两个文件的行范围

* 文件的内容