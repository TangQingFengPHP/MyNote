### 简介

`ncdu （NCurses Disk Usage）` 是一个用于 `Linux` 和类 `unix` 系统的磁盘实用程序，它提供了一种比 `du` 等传统命令更具交互性和用户友好性的方式来查看和分析磁盘空间使用情况。它提供了一个导航界面，允许您轻松识别哪些目录和文件占用了最多的磁盘空间。

### 安装

* `Ubuntu/Debian`

```shell
sudo apt update
sudo apt install ncdu
```

* `RedHat/CentOS/Fedora`

```shell
sudo yum install ncdu
# or
sudo dnf install ncdu
```

### 示例用法

#### 扫描当前目录

它将列出当前目录中的所有目录和文件，并按磁盘使用量排序。最大的文件和目录将首先列出。

```shell
ncdu
```

![alt text](/images/ncdu-image-1.png)

**支持的键**

```shell
 ↑,↓ or k,j to Move
 →,l to enter
 ←,h to return
 g toggle graph
 c toggle counts
 a toggle average size in directory
 m toggle modified time
 u toggle human-readable format
 n,s,C,A,M sort by name,size,count,asize,mtime
 d delete file/directory
 v select file/directory
 V enter visual select mode
 D delete selected files/directories
 y copy current path to clipboard
 Y display current path
 ^L refresh screen (fix screen corruption)
 r recalculate file sizes
 ? to toggle help on and off
 q/ESC/^c to quit
```

#### 扫描特定目录

```shell
ncdu /path/to/directory
```

**示例**

```shell
ncdu ~/Documents
```

#### ncdu 中的常用导航

描完成后，将看到一个屏幕，其中显示指定目录中的目录和文件及其磁盘使用情况。

* 箭头键（上/下）：在目录和文件列表中上

* 回车/右箭头：进入选定的目录

* 左箭头：返回父目录

* q：退出 `ncdu` 程序

* d：删除选择的文件或目录

* ?：显示帮助信息

* n：按名称（而不是磁盘使用情况）对输出进行排序

* s：按大小对输出进行排序（默认排序方法）

#### 排除特定目录

如果不想在扫描中包含某些目录，可以使用 `-x` 选项排除单独挂载的文件系统（对于排除外部驱动器或网络挂载很有用）

```shell
ncdu -x /path/to/directory
```

#### 显示隐藏的文件

默认情况下，`ncdu` 可能不会显示隐藏文件（以点开头的文件，如 `.bashrc`）

```shell
ncdu -a /path/to/directory
```

#### 扫描多个目录

```shell
ncdu /path/to/dir1 /path/to/dir2
```

#### 排除特定文件或目录

`-x` 或 `--exclude`

```shell
ncdu --exclude /path/to/exclude /path/to/directory
```

#### 保存输出

可以使用 `-o`（输出）标志将扫描结果保存到文件中。允许生成报告以供以后分析

```shell
ncdu -o output_file /path/to/directory
```

#### 加载保存的输出文件

```shell
ncdu -f output_file
```

#### 扫描家目录

```shell
ncdu ~
```