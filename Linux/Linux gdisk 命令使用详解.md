### 简介

`gdisk` 命令是 `Linux` 上管理 `GPT`（`GUID` 分区表）分区的强大工具。它可替代仅支持 `MBR`（主引导记录）分区的 `fdisk`。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt install gdisk
```

* `RHEL/CentOS`

```shell
sudo yum install gdisk
```

### 检查分区类型

```shell
sudo gdisk -l /dev/sdX
```

* `GPT 磁盘`：显示带有 `GUID` 代码的分区

* `MBR 磁盘`：`gdisk` 发出警告并询问是否要转换为 `GPT`

### 运行 `gdisk`

```shell
sudo gdisk /dev/sdX
```

### 常用交互式命令

* `p`：打印分区表

* `n`：创建新分区

* `d`：删除一个分区

* `t`：更改分区类型

* `w`：写入更改并退出

* `q`：退出而不保存

* `x`：高级特性

* `i`：显示分区的详细信息

### 示例用法

**打开 `gdisk`，进入交互式模式**

```shell
sudo gdisk /dev/sdX
```

#### 创建新分区

* 输入 `n`，然后回车

* 选择分区号

* 设置第一个扇区

* 设置最后一个扇区，或者指定分区大小，例如 +10G 表示 10GB

* 选择分区类型（默认为 `Linux` 文件系统或 8300）

* 输入 `w`，按回车确认保存

#### 删除分区

* 输入 `d`，然后选择分区编号

* 输入 `w`，按回车确认保存

#### 更改分区类型

* 输入 `t`，然后回车

* 选择分区类型：
    * `8300`：表示 `Linux` 文件系统
    * `8200`：表示 `Linux` 交换
    * `EF00`：表示 `EFI` 系统分区

* 输入 `w`，按回车确认保存

#### 将 MBR 转换为 GPT

> 将会警告提示可能会丢失数据

```shell
sudo gdisk /dev/sdX
```

* `gdisk` 自动检测 `MBR` 并提示转换

* 输入 `w`，按回车确认保存

#### 分区后创建文件系统

> 创建新分区后，使用以下命令对其进行格式化：

**示例**

```shell
sudo mkfs.ext4 /dev/sdX1
```

#### 挂载分区

```shell
sudo mount /dev/sdX1 /mnt

# 如果要在系统启动时自动挂载，添加以上命令到 /etc/fstab中
```

#### 验证 `GPT` 分区

```shell
lsblk
sudo gdisk -l /dev/sdX
```

