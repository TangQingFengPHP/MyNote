### 简介

`stat` 命令打印文件和文件系统的详细信息。该工具提供有关所有者是谁、修改日期、访问权限、大小、类型等信息。

该实用程序对于故障排除、在更改文件之前获取有关文件的信息以及例行文件和系统管理任务至关重要。

### 基本语法

```shell
stat [arguments] [filename]
```

### 常用选项

* `-L, --dereference`：跟随符号链接

* `-f, --file-system`：显示文件系统状态而不是文件状态

* `-c  --format=<FORMAT>`：使用指定的 `<FORMAT>` 而不是默认的

* `--printf=<FORMAT>`：类似于 `--format`，但解释反斜杠转义，并且不输出强制尾随换行符

* `-t, --terse`：以简洁的形式打印信息

### 示例用法

#### 查看文件的信息

```shell
stat file.txt
```

**示例输出**

```shell
  File: file.txt
  Size: 4030      	Blocks: 8          IO Block: 4096   regular file
Device: 801h/2049d	Inode: 13633379    Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/   test)   Gid: ( 1000/   test)
Access: 2019-11-06 09:52:17.991979701 +0100
Modify: 2019-11-06 09:52:17.971979713 +0100
Change: 2019-11-06 09:52:17.971979713 +0100
Birth: -
```

**输出的字段解释**

* `File`：文件的名称

* `Size`：文件的大小（以字节为单位）

* `Blocks`：文件占用的分配块的数量

* `IO Block`：每个块的大小（以字节为单位）

* `File type`：文件类型：例如常规文件、目录、符号链接

* `Device`：十六进制和十进制的设备编号

* `Inode`：Inode 编号

* `Links`：硬链接的数量

* `Access`：以数字和符号方法表示的文件权限

* `Uid`：用户 ID 和所有者名称

* `Gid`：群组 ID 和所有者的名称

* `Context`：`SELinux` 安全上下文

* `Access`：上次访问文件的时间

* `Modify`：上次修改文件内容的时间

* `Change`：上次更改文件属性或内容的时间

* `Birth`：文件创建时间

#### 显示文件系统的信息

```shell
stat -f file.txt
```

**示例输出**

```shell
  File: "package.json"
    ID: 8eb53097b4494d20 Namelen: 255     Type: ext2/ext3
Block size: 4096       Fundamental block size: 4096
Blocks: Total: 61271111   Free: 25395668   Available: 22265851
Inodes: Total: 15630336   Free: 13979610
```

**输出的字段解释**

* `File`：文件名

* `ID`：十六进制的文件系统 ID

* `Namelen`：文件名的最大长度

* `Fundamental block size `：文件系统上每个块的大小

* `Blocks`：
    * `Total`：文件系统中的块总数
    * `Free`：文件系统中的可用块的数量
    * `Available`：非 `root` 用户可用的空闲块数

* `Inodes`：
    * `Total`：文件系统中的 `inode` 总数
    * `Free`：文件系统中可用 `inode` 的数量

#### 跟随符号链接

> 默认情况下，`stat` 不跟踪符号链接

```shell
stat /etc/resolv.conf
```

**示例输出**

```shell
  File: /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
  Size: 39        	Blocks: 0          IO Block: 4096   symbolic link
Device: 801h/2049d	Inode: 8126659     Links: 1
Access: (0777/lrwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2019-11-06 21:12:26.875956073 +0100
Modify: 2018-07-24 11:11:48.128794519 +0200
Change: 2018-07-24 11:11:48.128794519 +0200
 Birth: -
```

> 使用 `-L` 选项跟随符号链接

```shell
stat -L /etc/resolv.conf
```

**示例输出**

```shell
  File: /etc/resolv.conf
  Size: 715       	Blocks: 8          IO Block: 4096   regular file
Device: 17h/23d	Inode: 989         Links: 1
Access: (0644/-rw-r--r--)  Uid: (  101/systemd-resolve)   Gid: (  103/systemd-resolve)
Access: 2019-11-06 20:35:25.603689619 +0100
Modify: 2019-11-06 20:35:25.555689733 +0100
Change: 2019-11-06 20:35:25.555689733 +0100
Birth: -
```

#### 自定义输出

`stat` 命令有两个选项，允许根据需要定制输出：`-c，（--format=<format>）`和 `--printf=<format>`。

这两个选项的区别在于，当使用两个或多个文件作为操作数时，`format` 会在每个操作数的输出后自动添加一个换行符，`--printf` 解释反斜杠转义。

* 仅查看文件的类型

```shell
stat --format="%F" /dev/null
```

**示例输出**

```shell
character special file
```

* 组合任意数量的格式指令

```shell
stat --format="%n,%F" /dev/null
```

**示例输出**

```shell
/dev/null,character special file
```

* 解释换行符或制表符等特殊字符

```shell
stat --printf='Name: %n\nPermissions: %a\n' /etc
```

**示例输出**

```shell
Name: /etc
Permissions: 755
```

#### 显示简洁的信息

```shell
stat -t /etc
```

**示例输出**

```shell
/etc 12288 24 41ed 0 0 801 8126465 147 0 0 1573068933 1573068927 1573068927 0 4096
```
