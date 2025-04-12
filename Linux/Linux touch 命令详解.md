### 简介

在 `Linux` 中 `touch` 命令用于创建空文件、更新文件的访问或修改时间戳。

### 常用选项

* `-c`：不创建文件

* `-t`：设置指定的时间

* `-r FILE`：使用另一个文件的时间戳

* `-a`：仅更改访问时间

* `-m`：仅更改修改时间

### 用法示例

#### 创建空文件

```shell
touch file.txt
```

如果 `file.txt` 不存在，则创建它。如果已存在，则更新其修改时间

#### 创建多个文件

```shell
touch file1.txt file2.txt file3.txt
```

#### 更新时间戳

* 将访问和修改时间更新为现在

```shell
touch existing.txt
```

* 设置具体日期和时间

```shell
touch -t 202404120830 file.txt
```

将 `file.txt` 设置为 `2024-04-12 08:30`

#### 与目录一起使用

```shell
touch /path/to/somefile.txt
```

如果目录不存在，则不会创建。仅当路径有效时才有效

#### 引用另一个文件的时间戳

```shell
touch -r source.txt target.txt
```

设置 `target.txt` 的时间戳以匹配 `source.txt`

#### 防止文件创建（仅当文件存在时才更新时间）

```shell
touch -c file.txt
```

#### 与 find 结合

更新所有 `.log` 文件

```shell
find . -name "*.log" -exec touch {} +
```

