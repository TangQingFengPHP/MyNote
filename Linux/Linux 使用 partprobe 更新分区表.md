### 简介

`partprobe` 是一个命令行实用程序，它可以在不重启的情况下更新内核有关分区表更改的信息。它强制内核重新读取指定磁盘的分区表。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt update
sudo apt install parted
```

* `RHEL/CentOS/Fedora`

```shell
sudo yum install parted  # CentOS/RHEL 7
sudo dnf install parted  # Fedora, RHEL 8+
```

### 示例用法

#### 通知内核分区表更改

这将扫描所有块设备并将任何更改通知内核

```shell
sudo partprobe
```

#### 指定磁盘

```shell
sudo partprobe /dev/sdX
```

#### 检查内核是否识别分区

```shell
lsblk
fdisk -l
cat /proc/partitions
```

### 何时使用 partprobe

#### 创建或修改分区后

使用 `fdisk`、`gdisk` 或 `parted` 创建或修改分区时

```shell
sudo partprobe /dev/sdX
```

#### 在 parted 中使用 mklabel 之后

```shell
sudo parted /dev/sdX mklabel gpt
sudo partprobe /dev/sdX
```

#### 当 fdisk -l 显示旧分区时

如果 `partprobe` 不起作用，可使用

```shell
sudo partx -u /dev/sdX
```

#### 如果分区正在使用中，partprobe 可能会失败

**运行 `partprobe` 之前卸载分区**

```shell
sudo umount /dev/sdX1
sudo partprobe /dev/sdX
```