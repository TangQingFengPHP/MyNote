### 简介

`Linux` 中的 `dd` 命令是一个功能强大的数据复制和转换实用程序。它以较低级别运行，通常用于创建可启动的 `USB` 驱动器、克隆磁盘和生成随机数据等任务。

`dd` 全称可以为：`data duplicator`、`disk destroyer` 和 `Data Definition`

### 功能和能力

* `磁盘映像`：创建整个磁盘或分区的精确、逐位副本

* `数据擦除`：使用零或随机数据安全地覆盖驱动器

* `文件转换`：ASCII 和 EBCDIC 之间的转换、字节顺序交换和文件填充

* `数据恢复`：通过忽略读取错误从故障驱动器读取数据

* `可启动媒体创建`：将磁盘映像写入 USB 驱动器或 SD 卡

* `存储性能测试`：对驱动器写入速度进行粗略的基准测试

### 语法

```shell
dd if=<input_file> of=<output_file> [options]
```

* `if`：输入文件（源文件或设备，例如 `/dev/sda`、`/dev/zero`）

* `of`：输出文件（目标文件或设备，例如，`/dev/sdb`，`myfile.img`）

* `Options`：自定义的行为选项

### 常用选项

* `bs=[BYTES]`：将输入和输出块大小都设置为 `BYTES`

**块大小表示 `dd` 命令每次输入或输出一次性读取或写入的数据大小**

* `count=[N]`：仅复制 `N` 个输入块

* `skip=[N]`：开始复制之前跳过输入文件中的 `N` 个块

* `seek=[N]`：开始写入之前跳过输出文件中的 `N` 个块

* `conv=[TYPE]`：指定转换类型（例如，`sync、noerror、notrunc`）

* `status=[LEVEL]`：控制输出详细程度（例如，`none、 noxfer、 progress`）

* `iflag=[FLAGS]`：输入特定标志（`direct、sync`）

* `oflag=[FLAGS]`：输出特定标志（`append、sync`）

* `ibs`：设置输入块大小

* `obs`：设置输出块大小

* `noerror`：读取错误后继续

* `notrunc`：不要截断输出文件

* `sync`：使用 `NULL` 填充每个输入块至 `ibs` 大小

### 示例用法

#### 基础用法

```shell
dd if=source.txt of=destination.txt

# 如果目标文件不存在，则自动创建，否则会覆盖目标文件
```

#### 创建可启动的 `USB` 驱动器

> 将 `ISO` 文件写入 `USB` 驱动器

```shell
sudo dd if=ubuntu.iso of=/dev/sdb bs=4M status=progress
```

* `if=ubuntu.iso`：输入的 `ISO` 文件

* `of=/dev/sdb`：输出的 `USB` 设备

* `bs=4M`：使用 4 MB 的块大小来加快复制速度

* `status=progress`：操作过程中显示进度

#### 备份磁盘

> 创建磁盘镜像

```shell
sudo dd if=/dev/sda of=backup.img bs=64K conv=sync,noerror
```

* `if=/dev/sda`：输入的原磁盘设备

* `of=backup.img`：输出的磁盘镜像

* `bs=64K`：块大小为 64 KB

* `conv=sync,noerror`：当发生错误时继续读取，并用控制填充

#### 从镜像中恢复磁盘

```shell
sudo dd if=backup.img of=/dev/sda bs=64K
```

#### 创建包含随机数据的文件

```shell
dd if=/dev/urandom of=random_data.bin bs=1M count=10
```

* `if=/dev/urandom`：随机输入源

* `of=random_data.bin`：输出的文件

* `bs=1M`：区块大小为 1 MB

* `count=10`：创建一个 10 MB 的文件

#### 安全擦除磁盘

> 使用随机数据覆盖磁盘

```shell
sudo dd if=/dev/urandom of=/dev/sda bs=1M status=progress
```

#### 测试磁盘写入速度

> 将零写入磁盘以测试写入速度

```shell
sudo dd if=/dev/zero of=testfile bs=1G count=1 oflag=direct
```

#### 将文件拆分成块

> 将文件分割成更小的块

```shell
dd if=largefile of=smallfile bs=1M count=100
```

#### 防止覆盖目标文件

```shell
dd if=source.txt of=destination.txt conv=notrunc
```

#### 将数据追加到文件

```shell
dd if=users.txt of=newusers.txt conv=append
```

#### 压缩 `dd` 读取的数据

```shell
sudo dd if=/dev/sda bs=1M | gzip -c -9 > sda.dd.gz
```

#### 操作过程中显示进度条

```shell
dd if=source_file of=destination_file status=progress
```

#### 将文件的数据格式从 EBCDIC 转换为 ASCII

```shell
sudo dd if=textfile.ebcdic of=textfile.ascii conv=ascii
```

#### 将文件的数据格式从 ASCII 转换为 EBCDIC

```shell
sudo dd if=textfile.ascii of=textfile.ebcdic conv=ebcdic
```

### 关键转换标志

* `sync`：用空字节填充每个块以达到指定的大小

* `noerror`：尽管读取有错误，仍继续操作

* `notrunc`：不要截断输出文件

* `ucase`：将文本转换为大写

* `lcase`：将文本转换为小写
