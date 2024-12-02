### 简介

`sort` 命令用于按特定顺序（例如升序或降序）排列文件中的行或输入数据。它可以按字母顺序、数字顺序和基于特定字段进行排序，如果没有指定文件，则从标准输入中读取。

### 常用选项

* `-r`：反转排序顺序（降序）

* `-n`：按数字排序

* `-k`：指定字段或列排序

* `-t`：定义字段的分隔符，默认是空格

* `-u`：在排序之后移除重复的行

* `-f`：忽略大小写

* `-o`：排序之后指定输出的文件

* `--help`：显示帮助信息

### 示例用法

#### 通过字母排序（默认排序）

```shell
sort file.txt
```

#### 通过字母降序排序

```shell
sort -r file.txt
```

#### 通过数字排序

```shell
sort -n numbers.txt
```

#### 通过指定字段排序

```shell
sort -k 2 file.txt

# 按每行第二个字段排序

#例如源文件内容是：
apple 2
banana 1
cherry 3

# 排序后：
banana 1
apple 2
cherry 3
```

#### 指定字段分隔符

```shell
sort -t: -k 2 file.txt

# 此处指定分割符为冒号

# 例如源文件内容是：
user1:1001
user3:1003
user2:1002

# 排序后：
user1:1001
user2:1002
user3:1003
```

#### 排序后移除重复的行

```shell
sort -u file.txt

# 例如源文件内容是；
apple
banana
apple
cherry

# 排序后：
apple
banana
cherry
```

#### 将排序后的输出保存到文件中

```shell
sort file.txt -o sorted_file.txt
```

#### 排序不区分大小写

```shell
sort -f file.txt
```

#### 检查文件是否已排序

```shell
sort -c file.txt
```

#### 使用分隔符按特定列进行数字排序

```shell
sort -t, -k 2n file.csv

# 按第二列的数字顺序对 CSV 文件 (file.csv) 进行排序，并使用逗号作为分隔符。
```

#### 对 `IP` 地址进行排序

```shell
sort -n -t . -k 1,1 -k 2,2 -k 3,3 -k 4,4 ips.txt
```

#### 排序多个文件

```shell
sort default1.txt default2.txt
```

#### 按时间戳对日志进行排序

```shell
sort -t ' ' -k 3,4 logs.txt

# 按第三和第四个字段对日志进行排序
```
