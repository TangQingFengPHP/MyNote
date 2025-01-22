### 简介

`fdisk` 命令是 `Linux` 中用于管理磁盘分区的强大文本实用程序。它可以创建、删除、调整大小和修改硬盘上的分区。

### 基本语法

```shell
fdisk [options] <device>
```

* `<device>`：要管理的磁盘，例如 `/dev/sda、/dev/nvme0n1 或 /dev/vda`

### 示例用法

#### 列出所有分区

> 将显示所有可用的磁盘及其分区，包括它们的大小和文件系统

```shell
fdisk -l
```

**示例输出**

```shell
Disk /dev/sda: 500 GB
Sector size (logical/physical): 512B/512B
Device     Boot   Start       End   Sectors  Size Id Type
/dev/sda1  *       2048   1050623  1048576  512M 83 Linux
/dev/sda2       1050624 976773167 975722544 465G 83 Linux
```

#### 查看指定磁盘的区分

```shell
fdisk -l /dev/sda
```

#### 管理指定的磁盘

> 这将打开一个交互式会话来管理磁盘 `/dev/sda`

```shell
fdisk /dev/sda
```

#### 进入交互式模式

```shell
fdisk /dev/sda
```

**示例输出**

```shell
WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
         switch off the mode (command 'c') and change display units to
         sectors (command 'u').

Command (m for help): m
Command action
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition
   l   list known partition types
   m   print this menu
   n   add a new partition
   o   create a new empty DOS partition table
   p   print the partition table
   q   quit without saving changes
   s   create a new empty Sun disklabel
   t   change a partition's system id
   u   change display/entry units
   v   verify the partition table
   w   write table to disk and exit
   x   extra functionality (experts only)

Command (m for help):
```

**示例**

```shell
Command (m for help):
```

**常用的交互式命令有**：

* `m`：显示帮助（显示所有可用的命令）

* `p`：打印所选磁盘的分区表

* `n`：添加一个新分区

* `d`：删除一个存在的分区

* `t`：变更分区类型，如：`Linux、swap`

* `a`：切换分区的可启动标志

* `w`：将更改写入磁盘并退出

* `q`：退出而不保存更改

#### 创建一个新的分区

1. 指定目标磁盘

```shell
fdisk /dev/sda
```

2. 输入 `n` 来创建一个新分区

* 选择主分区（`p`）或 扩展分区（`e`）

* 指定分区号、起始扇区和结束扇区（或大小）

3. 输入 `w` 来保存变更然后退出

#### 删除一个存在的分区

1. 指定目标磁盘

```shell
fdisk /dev/sda
```

2. 输入 `d` 接分区编号来删除一个分区

3. 输入 `w` 来保存变更然后退出

#### 变更分区类型

1. 指定目标磁盘

```shell
fdisk /dev/sda
```

2. 输入 `t` 来变更分区类型

* 输入分区编号

* 输入类型代码，例如：`82` 表示 `Linux swap`，`83` 表示 `Linux`，`7` 表示 `NTFS`

3. 输入 `w` 来保存变更然后退出

#### 将分区标记为可引导

1. 指定目标磁盘

```shell
fdisk /dev/sda
```

2. 输入 `a` 来切换可引导标志

3. 输入 `w` 来保存变更然后退出

#### 检查分区大小

```shell
fdisk -s /dev/sda2
```

#### 设置磁盘的扇区大小

```shell
fdisk -b 2048 /dev/sda
```

#### 列出分区表时，给出扇区大小，而不是柱面大小

```shell
fdisk -u /dev/sda
```

#### 设置磁盘的磁头数

```shell
fdisk -H 16 /dev/sda
```

#### 设置磁盘的柱面数

```shell
fdisk -C 100 /dev/sda
```

#### 设置磁盘每个磁道的扇区数

```shell
fdisk -S 63 /dev/sda
```

#### 检查分区变化

```shell
partprobe
```

### 使用场景

* 管理基于 `MBR` 的分区（针对 ≤ 2 TB 的磁盘）

* 对于更大的磁盘或 `GPT` 分区，需要使用 `gdisk` 或 `parted`
