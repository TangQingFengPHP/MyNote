### 简介

`setfacl` 、`lsattr`、`chattr` 这些命令用于管理 `Linux` 中的文件权限和属性。虽然传统的 `Linux` 权限（`chmod`、`chown`）提供基本控制，但这些命令提供了更高级的选项。

### `setfacl`(Set File ACL)

用于设置或修改访问控制列表 (`ACL`)，允许比传统的所有者-组-其他模型更细粒度的权限控制。

#### 常用选项

* `-m`：修改或设置 `ACL` 条目

* `-x`：删除 `ACL` 条目

* `-b`：删除所有 `ACL` 条目（重置）

* `-k`：删除默认的 `ACL`

* `-R`：将更改递归应用于目录及其内容

#### 授予用户读/写权限

```shell
setfacl -m u:<username>:rw <file>
```

#### 授予组执行权限

```shell
setfacl -m g:<groupname>:x <file>
```

#### 删除特定用户的 `ACL`

```shell
setfacl -x u:<username> <file>
```

#### 查看文件的 `ACL`

```shell
getfacl <file>
```

### `lsattr`(List File Attributes)

列出文件的扩展属性，控制特定行为，如不变性、仅追加模式等。

#### 常用选项

* `-a`：包括隐藏文件

* `-d`：显示目录属性而不是内容的

* `-R`：递归列出目录及其内容的属性

#### 查看文件的属性

```shell
lsattr <file>

# 输出示例：
----i--------e-- file
# i表示不可变属性
# e表示默认属性
```

#### 列出目录中所有文件的属性

```shell
lsattr </path/to/dir>
```

### `chattr`(更改文件属性)

更改文件或目录的扩展属性，允许控制诸如防止修改或删除之类的行为。

#### 常用选项

* `+`：添加一个属性

* `-`：移除一个属性

* `=`：准确设置属性

#### 常用属性

* `i`：不可变（防止修改、重命名或删除）

* `a`：仅可追加内容（只能添加数据，不能修改现有内容）

* `c`：自动压缩文件

* `u`：允许取消删除

#### 使文件不可变

```shell
sudo chattr +i <file>
```

#### 移除文件不可变属性

```shell
sudo chattr -i file
```

#### 将日志文件设置为仅可追加

```shell
sudo chattr +a /var/log/syslog
```

