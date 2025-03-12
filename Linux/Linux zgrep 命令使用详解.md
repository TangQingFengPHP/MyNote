### 简介

`zgrep` 用于在压缩 (`.gz`) 文件中搜索模式，就像 `grep` 在常规文本文件中所做的那样。它的工作原理是将文件临时解压到内存中，搜索模式并显示匹配的行。

### 基础语法

```shell
zgrep [OPTIONS] PATTERN FILE.gz

或

gzip -dc FILE.gz | grep [OPTIONS] PATTERN
```

### 示例用法

#### 在 .gz 文件中搜索字符串

```shell
zgrep "error" logfile.gz

或

gzip -dc logfile.gz | grep "error"
```

#### 大小写不敏感搜索

```shell
zgrep -i "error" logfile.gz
```

#### 在多个压缩文件中搜索

```shell
zgrep "error" *.gz

或

gzip -dc *.gz | grep "error"
```

#### 显示行号

```shell
zgrep -n "error" logfile.gz

# 显示匹配的行以及行号
```

#### 统计匹配到的行数

```shell
zgrep -c "error" logfile.gz
```

#### 仅显示匹配的文件名

```shell
zgrep -l "error" *.gz

# 仅列出包含“error”的 .gz 文件的文件名
```

#### 反向匹配

```shell
zgrep -v "error" logfile.gz

# 显示除包含“error”的行之外的所有行
```

#### 在目录中递归搜索

```shell
zgrep -r "error" /var/log/

# 在 /var/log/ 下的 .gz 文件中递归搜索“error”
```

#### 使用正则表达式（-E 表示扩展正则表达式）

```shell
zgrep -E "error|warning|failed" logfile.gz

# 查找包含“error”、“warning”或“failed”的行
```
