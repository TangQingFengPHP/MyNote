### 简介

`awk` 是 `Linux` 中强大的文本处理工具，广泛用于模式匹配扫描，数据提取，文本操作。

使用场景：

* 解析日志文件

* 汇总数据

* 格式化文本输出

* 从文件中提取指定的信息

### 历史 

`awk` 由三个人共同创造的，以三个人的 `last name` 的首字母组成

* Alfred V. Aho

* Peter J. Weinberger

* Brian W. Kernighan

### 基本语法

```shell
awk 'pattern { action }' file

# pattern 是匹配的模式，如正则表达式
# action 是匹配后进行的操作，如：打印，修改等
# file 要操作的文件，如果不指定文件，则从标准输入中读取
```

###  核心概念

#### 记录和字段

* 记录：文件中的每一行作为一个记录

* 字段：字段是记录的一部分，通过指定的分隔符分割，默认的分隔符是空格，可以通过 `-F` 选项自定义分隔符

其中，`$1`，`$2`，`$<n>` 等代表第几个字段

`$NF` 代表最后一个字段，`$0` 代表所有记录，即全部内容

`abc def`，其中 `abc` 是一个字段，`def` 是一个字段

#### 模式

可以是正则表达式、数字比较、条件判断等

#### 要执行的操作

定义在花括号 `{}` 里面

### 常用示例

#### 打印所有行数据

```shell
awk '{ print $0 }' file
```

#### 打印指定的字段

```shell
awk '{ print $1, $3 }' file

# 打印每行的第一个和第三个字段
```

#### 打印模式匹配的行

```shell
awk '/error/ { print $0 }' file

# 打印包含 error 文本的行
```

#### 使用条件表达式

```shell
awk '$3 > 50 { print $1, $2 }' file

# 当第三个字段大于50时打印第一个和第二个字段
```

#### 使用范围比较表达式

```shell
awk 'NR >= 5 && NR <= 10 { print $0 }' file

# NR表示行号
# 以上表示打印第五到第十行的内容
```

#### 自定义字段分隔符

```shell
awk -F ',' '{ print $1, $2 }' file

# 此处指定分隔符为逗号
```

### 内建变量

* `$0`：所有记录/全部内容

* `$1`，`$1`,...：第几个字段

* `NF`：当前行的字段数

* `NR`：行号

* `FS`：字段分隔符

* `OFS`：输出的字段分隔符

* `RS`：记录分隔符/行分隔符，默认 `\n`

* `ORS`：输出的记录分隔符/行分隔符

### 高级用法示例

#### 打印行号

```shell
awk '{ print NR, $0 }' file
```

#### 统计字段

```shell
awk '{ sum += $3 } END { print "Total:", sum }' file

# 统计每行第三个字段之和
```

#### 替换字段

```shell
awk '{$2 = "REPLACED"; print $0 }' file

# 替换每行的第二个字段值为 REPLACED
```

#### 打印模式匹配到的行数

```shell
awk '/pattern/ { count++ } END { print count }' file
```

#### 格式化输出

```shell
awk '{ printf "Line %d: %s\n", NR, $0 }' file
```

#### 通过管道处理标准输入

```shell
cat file | awk '{ print $1, $2 }'
```

#### `awk` 命令写到脚本里复用

新建 `script.awk` 文件，写入以下内容：

```shell
{ print $1, $NF }
```

使用 `-f` 执行脚本文件

```shell
awk -f script.awk file
```





