### 简介

`Linux` 中的 `mkfs` 命令用于在存储设备（例如分区、逻辑卷或整个磁盘）上创建文件系统。它代表 `make file system`（创建文件系统），是磁盘格式化的基本命令。

### 语法

```shell
mkfs [options] <device>
```

* `<device>`：目标设备，例如：`/dev/sda1, /dev/sdb, /dev/loop0`

* `[options]`：定制文件系统的选项

### 支持的文件系统

* `ext2/3/4`：第二、第三和第四个扩展文件系统（`mkfs.ext2、mkfs.ext3、mkfs.ext4`）

* `xfs`：高性能日志文件系统（`mkfs.xfs`）

* `vfat`：FAT32 文件系统（`mkfs.vfat`）

* `ntfs`：Windows NT 文件系统（`mkfs.ntfs`）

* `btrfs`：B-Tree 文件系统（`mkfs.btrfs`）

* `f2fs`：Flash 友好文件系统（`mkfs.f2fs`）

### 示例用法

#### 检测可用的储存设备

```shell
lsblk
或
fdisk -l
```

#### 创建一个 `ext4` 文件系统

```shell
mkfs.ext4 /dev/sdX
```

#### 创建一个 `FAT32` 文件系统

> 对于需要 `FAT32` 兼容性的 `USB` 驱动器或设备

```shell
mkfs.vfat /dev/sdX
```

#### 创建一个 `NTFS` 文件系统

> 为了与 `Windows` 兼容

```shell
mkfs.ntfs /dev/sdX
```

#### 创建一个 `XFS` 文件系统

> 对于高性能文件系统

```shell
mkfs.xfs /dev/sdX
```

#### 创建块大小为 4 KB 的 ext4 文件系统

```shell
mkfs.ext4 -b 4096 /dev/sdX
```

#### 创建 ext4 文件系统并为其分配标签

```shell
mkfs.ext4 -L MyData /dev/sdX
```

#### 格式化 USB 驱动器

```shell
mkfs.vfat -F 32 -n MyUSB /dev/sdb1
```

#### 检查设备上的坏块

```shell
mkfs -c /dev/sdb1
```

#### 查看文件系统的类型

```shell
file -sL /dev/sdb1
```

### 常用选项

**通用选项**

* `-t <type>`：指定文件系统类型

```shell
mkfs -t ext4 /dev/sdX

等同于：
mkfs.ext4 /dev/sdX
```

* `-V`：显示版本信息

* `-n`：执行试运行而不做任何更改

* `-L <label>`：分配一个标签到文件系统

```shell
mkfs.ext4 -L MyDisk /dev/sdX
```

**ext4 特定的选项**

* `-m <percentage>`：为root用户保留一定比例的磁盘空间

```shell
mkfs.ext4 -m 1 /dev/sdX
```

* `-b <block-size>`：指定块大小（例如，1024、2048、4096）

```shell
mkfs.ext4 -b 4096 /dev/sdX
```

**xfs 特定的选项**

* `-s <size>`：指定扇区大小

```shell
mkfs.xfs -s size=512 /dev/sdX
```

**vfat 特定的选项**

* `-F`：指定 `FAT` 大小（12、16 或 32）

```shell
mkfs.vfat -F 32 /dev/sdX
```

* `-n <name>`：分配卷名

```shell
mkfs.vfat -n MyUSB /dev/sdX
```

### 使用镜像文件的方式来体验文件系统

#### 使用 `dd` 命令来创建一个容量为 250M 的 镜像文件

```shell
dd if=/dev/zero of=~/test_filesystem.img bs=1M count=250

if指定输入源
of指定输出镜像的名称
bs指定块大小
count指定分配多少个快
```

#### 创建文件系统

```shell
mkfs.ext2 ~/test_filesystem.img
```

#### 创建挂载点

```shell
mkdir /mnt/testfilesystem
```

#### 挂载镜像文件

```shell
mount ~/test_filesystem.img /mnt/testfilesystem

# 挂载完之后就能正常使用文件的一些操作了
```

#### 取消挂载

```shell
umount /mnt/testfilesystem

# 取消挂载之后就看不到任何文件了
```