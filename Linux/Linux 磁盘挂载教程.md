### 简介

在 `Linux` 中挂载磁盘包括将一个文件系统从存储设备附加到文件系统层次结构中的目录。这允许与磁盘及其内容进行交互。

### 检查可用磁盘

在挂载磁盘之前，需要确定要挂载的磁盘

* 使用 `lsblk`

> lsblk 列出有关所有可用块设备（磁盘和分区）的信息

```shell
lsblk
```

**示例输出**

```shell
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0   100G  0 disk 
├─sda1   8:1    0    50G  0 part /mnt/data
├─sda2   8:2    0    50G  0 part 
```

* 使用 `fdisk -l`

> fdisk 提供有关磁盘分区的更多详细信息

### 挂载磁盘

#### 基本命令

```shell
sudo mount /dev/sdXn /mount/point
```

* `/dev/sdXn`：想要挂载的分区或磁盘（例如，/dev/sda1）

* `/mount/point`：想要挂载磁盘的目录（例如，/mnt/data）

**例如，将 /dev/sda1 挂载到 /mnt/data**

```shell
sudo mount /dev/sda1 /mnt/data
```

#### 使用特定文件系统类型进行挂载

```shell
sudo mount -t ext4 /dev/sda1 /mnt/data
```

#### 以只读方式挂载

> 要将磁盘挂载为只读，可以使用 -o ro 选项

```shell
sudo mount -o ro /dev/sda1 /mnt/data
```

#### 挂载 NTFS 文件系统

> 如果磁盘使用 NTFS 文件系统，可能需要安装 ntfs-3g 包

**安装ntfs-3g**

```shell
sudo apt install ntfs-3g
```

**挂载**

```shell
sudo mount -t ntfs-3g /dev/sda1 /mnt/data
```

### 查看已挂载的磁盘

* 使用 `mount`

> 这将列出所有已挂载的设备及其挂载点

```shell
mount
```

* 使用 `df`

> 以人类可读的格式（例如 MB、GB）显示所有已安装文件系统的磁盘空间使用情况。

```shell
df -h
```

### 启动时自动挂载磁盘

要在启动时自动挂载磁盘，需要在 `/etc/fstab` 文件中添加一个条目

**编辑 /etc/fstab 文件**

```shell
/dev/sda1  /mnt/data  ext4  defaults  0  2
```

* `/dev/sda1`：储存设备名称

* `/mnt/data`：挂载点

* `ext4`：文件系统类型

* `defaults`：挂载选项

* `0`：转储选项（通常为 0）

* `2`：启动期间文件系统检查顺序（1 表示根文件系统，2 表示其他文件系统）

**使变更生效**

```shell
sudo mount -a

# 这将挂载 /etc/fstab 中列出的所有文件系统
```

### 卸载磁盘

> 使用 umount

* 通过指定挂载点卸载

```shell
sudo umount /mnt/data
```

* 通过指定设备名称卸载

```shell
sudo umount /dev/sda1
```

* 强制卸载

> 如果磁盘正在使用，可能需要强制卸载

```shell
sudo umount -f /mnt/data
```

* 延迟卸载

> 如果无法立即卸载设备（例如，有任务正在处理），则可以使用延迟卸载

```shell
sudo umount --lazy /mnt/data

# 这将推迟卸载操作，直到设备不再繁忙
```

### 文件系统检查与修复

> 如果遇到错误或者想要检查并修复文件系统，可以使用 fsck

* 检查已挂载的文件系统

> 如果文件系统已被卸载，可以运行fsck来检查并修复

```shell
sudo fsck /dev/sda1
```

> 如果文件系统已挂载，需要先将其卸载

```shell
sudo umount /dev/sda1
sudo fsck /dev/sda1
```

### 挂载 ISO 文件

还可以挂载 ISO 文件（磁盘映像文件）。例如，要在 /mnt/iso 挂载 ISO 映像

* 创建挂载点

```shell
sudo mkdir /mnt/iso
```

* 挂载 ISO 文件

```shell
sudo mount -o loop /path/to/file.iso /mnt/iso

# -o loop 选项告诉 mount 将文件视为设备
```

### 挂载网络文件系统

如果需要挂载远程网络共享，例如 NFS 或 SMB（Samba）共享，则可以使用以下命令

* 挂载 NFS 共享

```shell
sudo mount -t nfs 192.168.1.100:/remote/share /mnt/nfs

# 192.168.1.100:/remote/share 是 NFS 服务器和共享位置。
```

* 挂载 Samba (SMB) 共享

**安装 cifs-utils 包**

```shell
sudo apt install cifs-utils  # On Ubuntu/Debian
```

**挂载**

```shell
sudo mount -t cifs //server/share /mnt/samba -o username=user,password=pass
```

* `//server/share`：SMB 共享路径

* `-o username=user,password=pass`：SMB 凭证
