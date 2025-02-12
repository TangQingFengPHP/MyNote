### 简介

索引节点（`Index Node`）是 `Linux/类unix` 系统文件系统上的一种数据结构，用于存储有关文件或目录的元数据。它包含文件的所有信息，除了文件名和数据。`inode` 在文件系统如何存储和检索数据方面起着至关重要的作用。

当在 `Linux` 系统上创建一个文件时，系统会为它分配一个 `inode` 。这个`inode` 包含有关文件的详细信息，如权限、所有者、大小以及文件数据块在磁盘上的位置。

### Inode 包含什么？

* `File type`：常规文件、目录、符号链接等

* `File permissions`：文件的读、写和执行权限

* `Owner`：文件所有者的用户 ID (UID)

* `Group`：文件组的组 ID (GID)

* `File size`：以字节为单位

* `Timestamps`：
    * `ctime`：最后一次改变 `inode` 的时间
    * `mtime`：文件数据的最后修改时间
    * `atime`：上次访问文件的时间

* `Link count`：该文件的硬链接数

* `Pointers to data blocks`：包含文件实际数据的块的地址（这些通常是间接指针）

### 查看 Inode 信息

* 使用 `ls -i`

> ls 命令可以与 -i 选项一起使用来显示文件的 inode 编号

```shell
ls -i filename
```

**示例输出**

```shell
ls -i myfile.txt
1234567 myfile.txt
```

* 使用 `stat`

> stat 命令提供有关文件的更多详细信息

```shell
stat filename
```

**示例输出**

```shell
$ stat myfile.txt
  File: myfile.txt
  Size: 1024       Blocks: 8          IO Block: 4096   regular file
Device: 803h/2051d  Inode: 1234567     Links: 1
Access: 2025-02-06 10:15:47.000000000 +0000
Modify: 2025-02-05 09:24:38.000000000 +0000
Change: 2025-02-05 09:24:38.000000000 +0000
 Birth: - 
```

### Inode 在文件操作中的作用

#### 文件创建

当创建文件时，文件系统会为其分配一个 `inode` ，用于存储文件的元数据。文件名存储在链接到 `inode` 的目录条目中

#### 硬链接

硬链接是指向索引页的指针。多个文件名可以指向相同的索引节点，并且彼此之间无法区分。它们共享相同的 `inode` ，但可以位于不同的目录中。直到所有硬链接被删除，文件的数据才会被删除。

**创建一个硬链接**

```shell
ln original_file hard_link_name
```

#### 符号（软）链接

符号链接 (`symlink`) 是一种特殊类型的文件，它通过存储路径指向另一个文件或目录。符号链接有自己的 `inode`，但使用路径指向目标文件

**创建一个软链接**

```shell
ln -s target_file symlink_name
```

#### 文件删除

当一个文件被删除时，它的索引节点中的链接数会减少。一旦链接计数达到零（即，没有更多的文件名指向该 `inode` ），`inode` 及其数据块将被释放以供重用。但是，`inode` 本身直到不再使用时才会被删除。

### 索引节点表

索引节点表是存储文件系统所有索引节点的内部结构。该表是在格式化文件系统时创建的，它包含固定数量的 `inode` 。如果文件系统耗尽了 `inode` ，则无法创建新文件，即使磁盘上有可用的空闲空间。这种情况在具有许多小文件的文件系统上尤其常见。

**检查文件系统上可用的 inode**

```shell
df -i
```

**示例输出**

```shell
Filesystem     Inodes  IUsed  IFree IUse% Mounted on
/dev/sda1      524288  10000  514288    2% /

# 文件系统有 524,288 个 inode，其中 10,000 个已被使用，514,288 个是空闲的
```

### 索引节点限制

* 有限的 `Inode` 数量：每个文件系统都有固定数量的 `Inode`，这些 `Inode` 是在创建文件系统时创建的。如果 `Inode` 用尽，即使有可用磁盘空间，也无法创建新文件

* 无文件名存储：`inode` 不存储文件名，因此文件名与 `inode` 结构无关。如果文件被重命名，`inode` 保持不变

* 硬链接限制：某些文件（例如使用某些选项挂载的目录和文件系统）不能有硬链接

### 通过 Inode 查找文件

```shell
find /path/to/search -inum inode_number
```

**示例用法**

```shell
find / -inum 1234567
```