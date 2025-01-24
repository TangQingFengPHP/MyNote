### 简介

`Linux` 中的 `lsblk` 命令，全称叫做：`list block devices` 用于列出块设备的信息，如硬盘、ssd盘及其分区。它提供了系统上所有块设备的树形结构，显示了它们的安装方式、大小和类型。

`lsblk` 命令读取 `sysfs` 文件系统和 `udev db` 收集信息。如果 `udev db` 不可用或在没有 `udev` 支持的情况下编译 `lsblk`，然后它尝试读取来自块设备的标签、`uuid` 和文件系统类型

### 基础语法

```shell
lsblk [options]
```

### 输出的字段

* `NAME`：块设备的名称（例如，`sda`, `nvme0n1`）。

* `MAJ:MIN:`：主设备号和次设备号

* `RM`：该设备是否可移动（1 表示可移动，0 表示不可移动）

* `SIZE`：块设备的大小

* `RO`：设备是否为只读（1 为只读，0 为读写）

* `TYPE`：设备的类型，如：`disk, part, rom`

* `MOUNTPOINT`：挂载点：设备在文件系统中的安装位置

### 常用选项

* `-a`：在输出中包含空设备

* `-f`：显示文件系统信息（类型、标签、UUID）

* `-l`：以列表格式显示输出

* `-J`：以 `JSON` 格式显示输出

* `-m`：显示设备所有者、组和模式

* `-n`：抑制输出中的标题行

* `-p`：显示完整的设备路径（例如，`/dev/sda`，而不仅仅是 `sda`）

* `-e <dev>`：从输出中排除特定设备

* `-I <dev>`：在输出中仅包含特定设备

* `-o <columns>`：指定要显示的列

* `x`：按指定字段对输出进行排序

### 示例用法

#### 列出所有块设备

> 这将以树结构显示所有块设备

```shell
lsblk
```

**示例输出**

```shell
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0   500G  0 disk
├─sda1        8:1    0   200G  0 part /
├─sda2        8:2    0   100G  0 part /home
└─sda3        8:3    0   200G  0 part [SWAP]
sdb           8:16   1   256G  0 disk
└─sdb1        8:17   1   256G  0 part /mnt/usb
```

#### 仅显示设备名称

```shell
lsblk -n
```

#### 以 `JSON` 或列表格式显示输出

```shell
lsblk -J  # JSON format
lsblk -l  # List format
```

**示例输出**

```json
{
   "blockdevices": [
      {"name": "sda", "maj:min": "8:0", "rm": "0", "size": "238.5G", "ro": "0", "type": "disk", "mountpoint": null,
         "children": [
            {"name": "sda1", "maj:min": "8:1", "rm": "0", "size": "512M", "ro": "0", "type": "part", "mountpoint": "/boot/efi"},
            {"name": "sda2", "maj:min": "8:2", "rm": "0", "size": "238G", "ro": "0", "type": "part", "mountpoint": "/"}
         ]
      }
   ]
}
```

#### 显示带有文件系统信息的设备

> 包含有关文件系统类型、标签和 `UUID` 的详细信息

```shell
lsblk -f
```

**示例输出**

```shell
NAME        FSTYPE LABEL    UUID                                 MOUNTPOINT
sda
├─sda1      ext4   rootfs   1234-5678-ABCD-EFGH                 /
├─sda2      ext4   home     8765-4321-HGFE-DCBA                 /home
└─sda3      swap   swap     1122-3344-5566-7788                 [SWAP]
sdb
└─sdb1      vfat   USB_DISK ABCD-1234                           /mnt/usb
```

#### 显示具有权限的设备

```shell
lsblk -m
```

#### 显示所有设备，包括空设备

> 默认情况下，`lsblk` 不会显示没有文件系统或挂载点的设备

```shell
lsblk -a
```

#### 显示内核信息

> 显示有关设备的内核信息（例如主设备号和次设备号）

```shell
lsblk -o KNAME,MAJ:MIN
```

#### 自定义字段展示

```shell
lsblk -o NAME,SIZE,FSTYPE,UUID,MOUNTPOINT
```

#### 仅列出已挂载的文件系统

```shell
lsblk -f | grep "/"
```

#### 按 `UUID` 列出设备

```shell
lsblk -o NAME,UUID | grep sda1
```

#### 排除可移动设备

> 排除 `USB` 驱动器和其他可移动设备

```shell
lsblk -e 7

设备类型 7 通常对应于循环设备
```

#### 显示特定设备的详细信息

```shell
lsblk /dev/sda
```

#### 识别未使用的分区

> 列出所有未挂载的分区

```shell
lsblk -f | grep -v "MOUNTPOINT" | grep -v "[SWAP]"
```

#### 在脚本中使用 `lsblk`

```shell
for dev in $(lsblk -ln -o NAME); do
    echo "Device: $dev"
done
```

