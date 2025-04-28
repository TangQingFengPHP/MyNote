### 简介

`whatis` 命令用来快速查看某个命令的简要描述。它其实就是快速查 `man` 页的 `NAME` 部分。

非常适合：

* 想知道某个命令干什么

* 不想翻长长的 `man` 页面

* 快速回忆工具功能

### 语法

```shell
whatis [选项] 关键词
```

* 关键词：要查询的命令、程序或文件名。

* 支持多个关键词一起查询。

### 常用选项

* `-i`：忽略大小写

* `-w`：使用通配符

* `-r`：使用正则表达式匹配

* `-l`：列出所有匹配项

* `-v`：打印详细的警告信息

* `-d`：打印调试信息

* `-h`：打印帮助信息

* `-s`：将只搜索给定的手册部分

* `-M`

### 示例用法

#### 查看一个命令的作用

```shell
whatis ls
```

**示例输出**

```shell
ls (1)               - list directory contents
```

* `ls` 是第 `1` 节（普通命令）

* 简要描述是 "list directory contents"（列出目录内容）

#### 一次查多个命令

```shell
whatis ls cp mv
```

#### 查询不区分大小写

默认区分大小写，加 `-i` 可忽略大小写：

```shell
whatis -i LS
```

#### 模糊查询（支持正则）

用 `-w`（wildcard 模式，支持通配符 *）

```shell
whatis -w "ls*"
```

比如会列出 `ls, lsof, lsblk` 等相关命令。

#### 打印详细信息

```shell
whatis -v ls
```

#### 正则匹配

```shell
whatis -r ^ls
```

#### 指定要检索的章节

```shell
whatis -s 3 cat
```

### 注意事项

`whatis` 依赖系统上的 `man` 数据库，如果提示：

```shell
whatis: no manual entry for xxx
```

说明：

* 可能没有对应命令的 `man` 页

* 或者 `man` 数据库没更新

可以用下面命令强制重建数据库：

```shell
sudo mandb
```

`mandb` 会扫描所有 `man` 页，更新索引）