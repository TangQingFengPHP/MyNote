### 简介

平常在执行linux命令时，想查看程序的二进制文件、源码、手册具体在哪个目录，就需要下面的命令来辅助。

### 使用 `which` 命令

`which` 命令主要是定位二进制可执行文件的位置，它在 `PATH` 环境变量中搜索。

#### 用法：

```shell
which <command>
```

#### 示例：

```shell
which ssh
# Output: /usr/bin/ssh
```
会打印可执行文件的完整路径

### 使用 `whereis` 命令

`whereis` 命令可以定位二进制文件、源码、命令手册

#### 用法

```shell
whereis <command>
```

#### 示例

```shell
whereis ssh
# Output: ssh: /usr/bin/ssh /usr/share/man/man1/ssh.1.gz
```

与 `which` 相比提供更广泛的搜索，包括源码和手册文件的搜索

### 使用 `locate` 命令

`locate` 是利用预构建的数据库文件进行搜索，所以速度很快

#### 用法

```shell
locate <filename>
```

#### 示例

```shell
locate ssh
# Output: /usr/bin/ssh, /usr/share/doc/ssh
```

### 使用 `find` 命令搜索

实时搜索命令所在的位置

#### 用法

```shell
find <directory> -name <filename>
```

#### 示例

```shell
find /usr -name ssh
# Output: /usr/bin/ssh
```

`find` 相比较其他命令速度比较慢

### 使用 `type` 命令

`type` 命令决定命令在shell中的解释方式（例如，它是别名、函数还是二进制）。

#### 用法

```shell
type <command>
```

#### 示例

```shell
type ssh
# Output: ssh is /usr/bin/ssh
```

### 使用 `command -v` 命令

`command -v` 返回 shell 中命令的路径或其别名。  

#### 用法

```shell
command -v <command>
```

#### 示例

```shell
command -v ssh
# Output: /usr/bin/ssh
```

`command -v` 与 `which` 类似，不同的是它是shell内建的命令

### 使用 `readlink` 命令

`readlink` 命令将符号链接解析为其目标路径。

#### 用法

```shell
readlink -f $(which <command>)
```

#### 示例

```shell
readlink -f $(which ssh)
# Output: /usr/bin/ssh
```

`readlink` 的特点是确保能获取到命令的真实路径，即使提供的是符号链接