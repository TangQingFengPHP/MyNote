### 简介

`Linux` 中的 `parted` 命令是一个用于创建、修改和管理磁盘分区的多功能工具。它支持传统的 `MBR`（Master Boot Record：主引导记录）和现代的 `GPT`（GUID Partition Table：GUID 分区表）分区方案。

### 磁盘分区的主要原因

* 最佳性能：正确管理分区可以提高系统的速度和响应性。例如，将操作系统文件从用户数据中分离出来，或者将频繁访问的数据放在磁盘上更快的部分上，都可以提高性能。

* 数据组织：分区允许用户根据类型、目的或重要性来隔离数据。例如，可以将系统文件、个人数据和备份文件放在单独的分区中，以确保更好地组织和更快地检索数据。

* 数据安全：通过将敏感或关键数据隔离在其自己的分区中，可以降低受系统崩溃、恶意软件或其他分区上的损坏软件影响的风险。

* 备份和恢复：分区使备份数据更加直接。而不是备份整个驱动器，可以专注于特定的分区。这使得恢复过程更快，在数据丢失时更有针对性。

* 双启动和系统升级：对于那些想要运行多个操作系统或测试新软件版本的人来说，单独的分区是至关重要的。这允许用户拥有多个操作系统版本或设置，而不会干扰他们的主系统。

### 主要特点

* 创建、删除、调整大小以及移动分区。

* 使用 `GPT` 支持大磁盘大小（>2 TB）。

* 可以处理各种文件系统，如 `ext4、NTFS、FAT32` 等。

* 适用于交互和非交互模式。

### 语法

```shell
parted [options] [device] [command [arguments]]
```

* `device`：目标磁盘，例如：`/dev/sda`, `/dev/nvme0n1`

* `command`：具体操作，例如创建或调整分区大小。

### 常用选项及其子命令

* `-l, --list`：列出所有块设备上的分区布局

* `-a <alignment-type>, --align <alignment-type>`：为新创建的分区设置对齐

* `align-check <type> <partition>`：对齐检查，`type` 类型为：`minimal` 或 `optimal`

* `mklabel <label-type>`：创建新的分区表

* `mkpart [part-type name fs-type] [start] [end]`：创建新分区

* `print [print-type]`：显示分区表

* `rescue [start] [end]`：拯救恢复丢失的分区

* `resizepart [partition] [end]`：重新分配分区大小

* `rm [partition]`：删除分区

* `select [device]`：选择设备，选择设备作为要编辑的当前设备。设备通常应为 `Linux` 硬盘设备，但必要时它可以为分区、软件 raid 设备或 LVM 逻辑卷。

* `set [partition] [flag] [state]`：设置分区的标志和状态

### 示例用法

#### 以交互模式启动 `Parted`

```shell
sudo parted /dev/sdX
```

#### 以非交互式模式启动 `Parted`

```shell
sudo parted /dev/sdX mklabel gpt
```

#### 查看分区表

```shell
sudo parted /dev/sdX print
```

#### 创建分区表

> 选择 `GPT`（推荐用于现代系统）或 `MBR`

```shell
sudo parted /dev/sdX mklabel gpt
```

```shell
sudo parted /dev/sdX mklabel msdos
```

#### 创建分区

> 创建一个主分区

```shell
sudo parted /dev/sdX mkpart primary ext4 0% 50%
```

* `primary`：分区类型

* `ext4`：文件系统类型

* `0%`：起始位置（磁盘的开头）

* `50%`：结束位置（磁盘空间的 50%）

#### 格式化分区

```shell
sudo mkfs.ext4 /dev/sdX1
```

#### 重新分配分区大小

> 将分区大小调整为磁盘的 80％：

```shell
sudo parted /dev/sdX resizepart 1 80%

# 1代表分区的编号
```

#### 删除一个分区

```shell
sudo parted /dev/sdX rm 1

# 1代表分区的编号
```

#### 交互式模式示例

```shell
sudo parted /dev/sdX

(parted) mklabel gpt
(parted) mkpart primary ext4 1MiB 100%
(parted) print
(parted) quit
```

#### 对齐分区

```shell
sudo parted /dev/sdX mkpart primary ext4 1MiB 100% --align optimal
```

#### 检查分区

```shell
sudo parted /dev/sdX check 1
```

#### 设置可启动的标志

> 将分区标记为可启动

```shell
sudo parted /dev/sdX set 1 boot on
```

#### 变更分区的名称

```shell
sudo parted /dev/sdX name 1 MyPartition
```

#### 创建全磁盘分区

```shell
sudo parted /dev/sdX mklabel gpt
sudo parted /dev/sdX mkpart primary ext4 0% 100%
```

#### 脚本分区创建

> 为了实现自动化，无需交互即可运行命令

```shell
sudo parted /dev/sdX --script mklabel gpt
sudo parted /dev/sdX --script mkpart primary ext4 1MiB 50%
```

### MBR与GPT的主要异同

| 特点 |  MBR  |  GPT  |
| --- | --- | --- |
|  最大磁盘大小   |  2 TB   |  >9 ZB   |
|  最大分区数   |  4个主分区   |  无限制(默认是128个)   |
|  兼容性   |  老系统   |  现代系统   |
|  冗余   |  无备份表   |  有备份表   |

