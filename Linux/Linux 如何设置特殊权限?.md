### 简介

通过使用 `setuid`、`setgid` 、`sticky`，它们是 `Linux` 中的特殊权限，可以对文件和目录的访问和执行方式提供额外的控制。


| 命令 | 八进制数字 | 功能 |
| :-: | :-: | :-- |
| `setuid` | 4 | 当执行文件时，它以文件所有者的权限运行，而不是执行它的用户的权限运行。 |
| `setgid` | 2 | 当执行文件时，它将以文件组的权限运行。对于目录，它将确保文件继承目录的组。 |
| `sticky` | 1 | 对于目录，它确保只有文件所有者可以删除或重命名文件，即使其他人具有写权限。 |

### `setuid`（Set User ID）

通常用于需要提升权限的可执行二进制文件

#### 将 `setuid` 添加到文件中

```shell
chmod u+s <filename>
```

#### 验证 `setuid`

```shell
ls -l filename

# 示例输出如下：
-rwsr-xr-x 1 root root 12345 Nov 29 12:00 filename

所有者的执行位置 (rws) 中的s表示 setuid
```

#### 移除 `setuid`

```shell
chmod u-s <filename>
```

### `setgid`（Set Group ID）

使用在文件上时，确保文件以文件的组权限运行，而不是用户的主要组权限运行

使用在目录上时，确保目录内创建的所有文件都继承目录的组所有权，而不是用户的主要组

#### 将 `setgid` 添加到文件或目录中

```shell
chmod g+s <filename>/<directory>
```

#### 验证 `setgid`

```shell
ls -ld <filename>/<directory>

# 示例输出如下：
drwxr-sr-x 2 user group 4096 Nov 29 12:00 directory_name

组执行位置(r-s)中的s表示 setgid
```

#### 移除 `setgid`

```shell
chmod g-s <filename>/<directory>
```

### `sticky`

通常用于目录以防止用户删除或重命名不属于他们自己的文件，即使该目录对他们具有写权限，适用于 `/tmp` 等共享目录

#### 添加 `sticky` 位

```shell
chmod +t <directory_name>
```

#### 验证 `sticky` 位

```shell
ls -ld <directory_name>

# 示例输出如下：
drwxrwxrwt 2 user group 4096 Nov 29 12:00 directory_name
其他人的执行位置（rwt）中的t表示 sticky(粘滞位)
```

#### 移除 `sticky` 位

```shell
chmod -t <directory_name>
```

#### 使用八进制数字的形式设置

```shell
chmod 6755 <filename>

# 第一个6 = setuid + setgid (4 + 2)
# 第二个7 = 所有者的权限 rwx
# 后面两个5 = 组和其他人的权限 r-x
```