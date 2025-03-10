### 简介

`egrep`（扩展 `GREP`）命令是 `grep` 的一个变体，支持扩展正则表达式 。它在功能上等同于 `grep -E`。

### 基础语法

```shell
egrep [OPTIONS] PATTERN [FILE...]

或

grep -E [OPTIONS] PATTERN [FILE...]
```

### 示例用法

#### 在文件中查找包含“error”的所有行

```shell
egrep "error" logfile.txt
```

#### 大小写不敏感搜索

```shell
egrep -i "error" logfile.txt
```

#### 使用多种模式 (|)

查找包含"error"或"warning"的行

```shell
egrep "error|warning" logfile.txt
```

#### 使用 ?（匹配零次或一次出现）

查找“colou*r”（匹配“color”或“colour”）

```shell
egrep "colou?r" file.txt
```

#### 使用 +（匹配出现一次或多次）

查找“ab”、“abb”、“abbb”等

```shell
egrep "ab+" file.txt
```

#### 使用 *（匹配零次或多次出现）

查找“ab”、“abb”、“abbb”甚至“a”

```shell
egrep "ab*" file.txt
```

#### 使用 {}（精确或范围重复）

查找后面跟着 2 到 4 个“b”的“a”

```shell
egrep "ab{2,4}" file.txt
```

#### 匹配行的开头 (^) 和结尾 ($)

* 查找以“Error”开头的行

```shell
egrep "^Error" logfile.txt
```

* 查找以“done”结尾的行

```shell
egrep "done$" logfile.txt
```

#### 匹配特定字符集

* 查找带有“gray”或“grey”的行

```shell
egrep "gr[ae]y" file.txt
```

* 查找包含任意数字的行

```shell
egrep "[0-9]" file.txt
```

* 查找没有数字的行（`[]` 内的 `^` 表示否定）

```shell
egrep "[^0-9]" file.txt
```

#### 使用括号进行分组

查找“foo1”或“foo2”但不查找“foo3”

```shell
egrep "foo(1|2)" file.txt
```

#### 在多个文件中搜索

在所有 `.log` 文件中查找“error”

```shell
egrep "error" *.log
```

#### 统计匹配到的次数

```shell
egrep -c "error" logfile.txt
```

#### 显示行号（-n）

```shell
egrep -n "error" logfile.txt
```
