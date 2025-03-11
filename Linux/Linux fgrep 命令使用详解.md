### 简介

`fgrep`（`fix GREP`）命令搜索固定字符串，而不将特殊字符解释为正则表达式。它相当于 `grep -F`。

### 基础语法

```shell
fgrep [OPTIONS] "STRING" FILE

或

grep -F [OPTIONS] "STRING" FILE
```

### 示例用法

#### 在文件中查找包含“error”的所有行

```shell
fgrep "error" logfile.txt

或

grep -F "error" logfile.txt
```

#### 搜索多个字符串

`keywords.txt` 文件包含以下内容: 

```shell
error
warning
failed
```

```shell
fgrep -f keywords.txt logfile.txt

或
grep -F -f keywords.txt logfile.txt

# 这将匹配 logfile.txt 中包含以上keywords.txt中的任何单词
```

#### 搜索包含特殊字符的字符串

与 `grep` 不同，`fgrep` 不会将 `. * []` 视为特殊正则表达式字符。 例如，在文件中搜索 `1.2.3`

```shell
fgrep "1.2.3" file.txt

或

grep -F "1.2.3" file.txt

# 使用 grep 时，需要转义（grep "1\.2\.3"），但 fgrep 将其视为文字
```

#### 在多个文件中搜索

```shell
fgrep "error" file1.txt file2.txt

或

grep -F "error" file1.txt file2.txt
```

#### 大小写不敏感搜索

```shell
fgrep -i "error" logfile.txt

或

grep -F -i "error" logfile.txt
```

#### 统计匹配行数 (-c)

```shell
fgrep -c "error" logfile.txt

或

grep -F -c "error" logfile.txt
```

#### 显示行号

```shell
fgrep -n "error" logfile.txt

或

grep -F -n "error" logfile.txt

# 这将打印匹配的行及其行号
```

#### 反转匹配（-v 表示排除）

```shell
fgrep -v "error" logfile.txt

或

grep -F -v "error" logfile.txt

# 排除包含“error”的行
```

#### 合并选项使用

* 不区分大小写地查找“error”，显示行号，并计算出现次数

```shell
fgrep -i -n -c "error" logfile.txt

或

grep -F -i -n -c "error" logfile.txt
```