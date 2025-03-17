### 简介

`GNU Binutils`（二进制实用程序）是用于在 `Unix/Linux` 系统中操作二进制文件的工具集合。这些工具有助于汇编、链接、反汇编和检查二进制可执行文件、目标文件、库和汇编代码。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt update && sudo apt install binutils
```

* `RHEL/CentOS`

```shell
sudo yum install binutils
```

* `macOS`

```shell
brew install binutils
```

### 常用 Binutils 命令

* `as`：GNU 汇编器（将汇编代码转换为机器代码）

* `ld`：GNU 链接器（将目标文件链接到可执行文件中）

* `objdump`：显示有关二进制文件的信息

* `nm`：列出目标文件中的符号

* `ar`：创建和管理静态库

* `ranlib`：生成静态库的索引

* `strip`：从二进制文件中删除调试符号	

* `size`：显示二进制文件的节大小

* `strings`：从二进制文件中提取可读文本

* `readelf`：显示有关 ELF（可执行和可链接格式）文件的信息

* `addr2line`：将地址转换为源文件和行号

* `c++filt`：解密 C++ 符号名称

### 示例用法

#### `objdump` 检查二进制文件

```shell
objdump -d /bin/ls  # 反汇编 ls 命令
objdump -x /bin/ls  # 显示完整的二进制信息
```

#### `nm` 显示目标文件中的符号

```shell
nm /bin/ls  # 列出 ls 二进制文件中的符号
```

#### `strings` 提取可读文本

```shell
strings /bin/ls  # 从二进制文件中提取可读文本
```

#### `readelf` 检查 ELF 文件

```shell
readelf -h /bin/ls  # 显示 ELF 标头
```

#### `strip` 删除调试符号

```shell
strip myprogram  # 通过删除符号来减少二进制大小
```

#### 创建并链接汇编程序

* 编写一个汇编程序（hello.s）

```shell
.global _start
.section .text
_start:
    mov $1, %rax
    mov $1, %rdi
    mov $msg, %rsi
    mov $len, %rdx
    syscall

    mov $60, %rax
    xor %rdi, %rdi
    syscall

.section .data
msg: .ascii "Hello, Binutils!\n"
len = . - msg
```

* 用 as 组装

```shell
as hello.s -o hello.o
```

* 使用 ld 来链接

```shell
ld hello.o -o hello
```

* 运行程序

```shell
./hello
```

**输出**

```shell
Hello, Binutils!
```