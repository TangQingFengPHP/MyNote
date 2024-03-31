## 一、df

### 1.简介

df 是 disk free的缩写，从UNIX和类UNIX操作系统的早期开始，它就是UNIX和类UNIX操作系统的一部分。它被设计为一种工具，用于监视系统上已使用和可用的磁盘空间数量。

df 命令主要用于需要检查文件系统上已使用和可用的磁盘空间的数量。这在管理服务器系统时尤其重要，因为服务器系统中磁盘空间耗尽可能导致严重的问题。

如果没有指定文件名，则显示在当前所有挂载的文件系统上可用的空间。空间默认情况下以1K块显示，除非环境变量POSIXLY_CORRECT被设置，在这种情况下使用512字节的块。

显示的用量默认是字节为单位的。

> 原理

命令从 /proc/mounts 或 /etc/mtab 中检索磁盘信息。

### 2.常用选项

* `-a, --all` ：包括伪的（具有0块的伪文件系统(没有直接绑定到物理设备)）、重复的、不可访问的文件系统。

```shell
df -a
```

* `-h, --human-readable` ：以人类可读的方式打印，如：KB、MB、GB，打印大小以1024为单位。

```shell
df -h

Filesystem    Size  Used Avail Use% Mounted on
devtmpfs       863M     0  863M   0% /dev
tmpfs          893M  168K  893M   1% /dev/shm
tmpfs          893M  9.5M  883M   2% /run
tmpfs          893M     0  893M   0% /sys/fs/cgroup
/dev/map[...]   17G  6.9G   11G  41% /
/dev/sda1     1014M  255M  760M  26% /boot
tmpfs          179M  120K  179M   1% /run/user/1000
```

* `-H, --si` ：与-h相似，打印大小以1000为单位。 

```shell
df -H
```

* `-k` ：以1024字节的块显示所有挂载的文件系统信息和使用情况，以千字节(kb)表示大小。

```shell
df -k
```

* `-m` ：以兆字节显示大小

```shell
df -m
```

* `-i, --inodes` ：列出索引节点信息而不是块使用情况。

> inode是存储文件和目录信息的数据结构，例如所有权、权限和时间戳。

```shell
df -i

Filesystem    Inodes IUsed IFree IUse% Mounted on
devtmpfs        216K   393  216K    1% /dev
tmpfs           224K     3  224K    1% /dev/shm
tmpfs           224K   857  223K    1% /run
tmpfs           224K    17  224K    1% /sys/fs/cgroup
/dev/map[...]   8.5M  168K  8.4M    2% /
/dev/sda1       512K   310  512K    1% /boot
tmpfs           224K    74  224K    1% /run/user/1000
```

节点信息字段解释：

```
Filesystem：文件系统名称

Inodes：文件系统上的 inode 总数

IUsed：已使用的索引节点数

IFree：未使用的索引节点数

IUse%：已使用索引节点的百分比

Mounted on：文件系统挂载的目录
```

* `-l, --local` ：将输出限制为本地文件系统。

```shell
df -l
```

* `--output[=FIELD_LIST]` ：自定义输出字段。

```shell
df -h --output=source,avail,pcent,target

Filesystem      Avail Use%  Mounted on
devtmpfs         863M   0%  /dev
tmpfs            893M   1%  /dev/shm
tmpfs            883M   2%  /run
tmpfs            893M   0%  /sys/fs/cgroup
/dev/map[...]     11G  41%  /
/dev/sda1        760M  26%  /boot
tmpfs            179M   1%  /run/user/1000
```

* `-P, --portability` ：使用POSIX输出格式

```shell
df -P
```

* `--total` ：删除所有对可用空间不重要的条目，对总量求和统计。

```shell
df -h --total

Filesystem     Size  Used Avail Use% Mounted on
devtmpfs       863M     0  863M   0% /dev
tmpfs          893M  168K  893M   1% /dev/shm
tmpfs          893M  9.5M  883M   2% /run
tmpfs          893M     0  893M   0% /sys/fs/cgroup
/dev/map[...]   17G  6.9G   11G  41% /
/dev/sda1     1014M  255M  760M  26% /boot
tmpfs          179M  120K  179M   1% /run/user/1000
total           22G  7.2G   15G  33% -
```

* `-t, --type=[TYPE]` ：只列出指定的文件系统类型的相关信息。

```shell
df -t ext4

Filesystem     1K-blocks      Used Available Use% Mounted on
/dev/nvme0n1p3 222284728 183666112  27257432  88% /
/dev/sda1      480588496 172832632 283320260  38% /data
```

* `-T, --print-type` ：打印文件系统类型

```shell
df -T

Filesystem     Type     1K-blocks     Used Available Use% Mounted on
/dev/sda1      ext4     102384432 45735432  51335636  47% /
tmpfs          tmpfs      4145120        4   4145116   1% /dev/shm
```

* `-x, --exclude-type=[TYPE]` ：排除指定的文件系统类型

```shell
df -x tmpfs
```

* `--help` ：打印帮助信息

* `--version` ：打印版本信息

### 3.命令示例

* 普通用法

```shell
df

Filesystem     1K-blocks      Used Available Use% Mounted on
dev              8172848         0   8172848   0% /dev
run              8218640      1696   8216944   1% /run
/dev/nvme0n1p3 222284728 183057872  27865672  87% /
tmpfs            8218640    150256   8068384   2% /dev/shm
tmpfs            8218640         0   8218640   0% /sys/fs/cgroup
tmpfs            8218640        24   8218616   1% /tmp
/dev/nvme0n1p1    523248    107912    415336  21% /boot
/dev/sda1      480588496 172832632 283320260  38% /data
tmpfs            1643728        40   1643688   1% /run/user/1000
```

输出的字段解释：

```
Filesystem：文件系统名称

1K-blocks：文件系统的大小（以 1K 块为单位）

Used：以1K块为单位的已使用空间

Available：以1K块为单位的可使用空间

Use%：已使用空间的百分比

Mounted on：文件系统挂载的目录
```

* df联合grep一起只打印出空间量总量

```shell
df -h --total|grep ^total

total  22G  7.2G  15G  33% -
```

* 打印指定挂载点的空间用量

```shell
df -h /

Filesystem                Size  Used Avail Use% Mounted on
/dev/mapper/centos-stream  17G  6.9G   11G  41% /

df -h /boot

Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1      1014M  255M  760M  26% /boot
```

* 指定文件名，查看文件名所在挂载点的信息

```shell
df -h abc.txt
```

* df联合sort通过用量大小排序

```shell
df --output=size,target | sort -n -r

Size  Mounted on
98G   /
4.0G  /dev/shm
```

### 4.输出格式字段

* source：文件系统源

* fstype：文件系统类型

* itotal：索引节点总数

* iused：已使用的索引节点数

* iavail：可用的索引节点数

* ipcent：已使用的索引节点数百分比

* size：总磁盘空间

* used：已使用的磁盘空间

* avail：可使用的磁盘空间

* pcent：已使用的磁盘空间百分比

* file：在命令行指定的文件名

* target：文件系统挂载的目录

### 5.man pages

![alt text](/images/df-image.png)


## 二、du

### 1.简介

du是disk usage的缩写，从早期开始就是UNIX和类UNIX系统的一部分。它的设计目的是提供目录树(包括其子目录)的磁盘使用情况摘要。

du命令主要用于需要了解系统上的目录或文件所使用的磁盘空间量。当试图识别占用大部分磁盘空间的大文件或目录时，它特别方便。

### 2.常用选项

* `-0, --null` ：以NUL结束每个输出行，而不是换行。

```shell
du -0
```

* `-a, --all` ：显示每个单独文件的磁盘使用情况，而不仅仅是目录。

```shell
du -a
```

* `-B, --block-size=[SIZE]` ：指定尺寸格式打印

```shell
du --block-size=1M
```

* `--apparent-size` ：打印表面的文件大小，而不是磁盘使用量，虽然表面文件大小可能比较小，但可能因文件尺寸增大而文件中内部出现一些碎片，实际上占用磁盘要大。

```shell
du --apparent-size
```

* `-c, --total` ：提供磁盘使用情况的总计。

```shell
du -c

/home/abc/article_submissions/
12K    /home/abc/article_submissions/my_articles
36K    /home/abc/article_submissions/community_content
48K    /home/abc/article_submissions/
48K    total
```

* `-d, --max-depth=N` ：指定递归的深度

```shell
du --max-depth=1
```

* `-h, --human-readable` ：以人类可读的单位打印

```shell
du -h

64K  ./test_dir
128K .
```

* `--inodes` ：列出索引节点使用信息，而不是块使用情况

```shell
du --inodes
```

* `-k` ：以KB为单位输出

```shell
du -k

等同于：du --block-size=1K
```

* `-m` ：以MB(兆字节)为单位输出

```shell
du -m

等同于：du --block-size=1M
```

* `-S, --separate-dirs` ：不包含子目录大小

```shell
du -S
```

* `--si` ：类似于 `-h`，使用1000的幂，而不是1024

```shell
du --si
```

* `-s, --summarize` ：仅显示每个参数的总数

```shell
du -s
```

* `--time` ：显示目录或该目录子目录下所有文件的最后修改时间

```shell
du --time
```

* `--time=[WORD]` ：显示指定的时间格式，而不是默认的修改时间，例如：atime，access，use，ctime，status

```shell
du --time=atime
```

* `-X, --exclude-from=[FILE]` ：排除与[FILE]中任何模式匹配的文件

* `--exclude=[PATTERN]` ：排除匹配到的文件

```shell
du -ah --exclude="*.dll" 
```

PATTERN是一个shell模式(不是正则表达式)。 模式 `?` 匹配任何一个字符，而 `*` 匹配任何字符串 (由零个、一个或多个字符组成)。例如:*.o 
将匹配任何以 .o 结尾的文件。因此, 命令：`du --exclude='*.o'` 将跳过所有以 .o 结尾的文件和子目录(包括 `*.o` 文件本身)。

* `-x, --one-file-system` ：跳过不同文件系统上的目录

```shell
du -x
```

* `--help` ：打印帮助信息

```shell
du --help
```

* `--version` ：打印版本信息

```shell
du --version
```

### 3.命令示例

* `-h` 接指定目录

```shell
du -h /home/user/documents
```

* `--exclude` 接指定目录

```shell
du -h --exclude='*.txt' /home/user/documents
```

* 配合sort命令一起使用，按照文件使用量排序

```shell
du -h --max-depth=1 | sort -hr

128K    .
64K     ./test_dir
```

* 打印当前目录所有文件的用量总和

```shell
du -sh .
```

### 4.man pages

![alt text](/images/du-image.png)