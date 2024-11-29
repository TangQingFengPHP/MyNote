### 简介

`Linux` 中的 `xattr` 命令用于管理文件的扩展属性。扩展属性存储标准属性中未包含的附加元数据（如权限、所有者和时间戳）。它们通常由特定的应用程序或文件系统（例如，`acl`、`SELinux` 标签）使用。

### 关键概念

* 扩展属性：与文件相关的元数据，以键值对的形式存储

* 属性的通用命名空间：

    * `user`：通用，普通用户可访问

    * `security`：用于安全框架，如 `SELinux`

    * `system`：用于系统级别的元数据

    * `trusted`：需要 `root` 访问权限的元数据

### 用法示例

#### 列出文件的所有扩展属性

```shell
xattr example.txt

# 输出如：user.comment
```

#### 查看扩展属性的值

```shell
xattr -p [attribute_name] [file]

xattr -p user.comment example.txt

# 输出如：This is a sample comment.
```

#### 设置或更新扩展属性

```shell
xattr -w [attribute_name] [value] [file]

xattr -w user.comment "This is a test comment" example.txt
```

#### 移除指定的扩展属性

```shell
xattr -d [attribute_name] [file]

xattr -d user.comment example.txt
```

#### 列出文件的所有扩展属性的键和值

```shell
xattr -l [file]

xattr -l example.txt
```

#### 复制一个文件的扩展属性到另一个文件

```shell
xattr --copy-source=[source_file] [destination_file]

xattr --copy-source=example.txt copy.txt
```

#### 递归列出目录所有文件的扩展属性

```shell
xattr -r [directory]
```

#### 递归删除目录所有文件的扩展属性

```shell
xattr -cr [directory]
```

### 常见问题

* `ext4`、`XFS`、`Btrfs` 文件系统支持扩展属性，`FAT32` 文件系统不支持。

* 如果扩展属性不工作，使用下列命令启用：

```shell
sudo mount -o remount,user_xattr /mount/point
```
