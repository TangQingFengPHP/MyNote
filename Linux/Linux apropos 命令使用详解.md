### 简介

`apropos` 是一个模糊搜索工具，可以在所有 `man` 页面里搜输入的关键词。相比 `whatis` 只能搜命令名，`apropos` 描述内容也能搜。

#### 常用选项

* `-e, --exact`：返回与关键字完全匹配的名称和描述

* `-d`：打印调试消息

* `-w, --wildcard`：使用通配符搜索关键字

* `-a, --and`：功能类似于逻辑与。当所有关键字匹配时返回输出

* `-l, --long`：输出不截断

* `-C`：使用用户配置文件而不是 `$MANPATH`

* `-s`：仅在特定的手册页部分中搜索。

* `-M`：将搜索路径设置为 `PATH` 而不是默认的 `$MANPATH`

* `-L`：设置搜索的区域设置

* `-r, --regex`：将每个关键字解释为正则表达式

* `-d`：打印 `debug` 信息

* `-v`：打印详细的警告信息

### 示例用法

#### 基本用法

```shell
apropos "list"
```

**示例输出

```shell
ls (1)               - list directory contents
dir (1)              - list directory contents
nmcli (1)            - command-line tool for controlling NetworkManager
```

#### 搜索多个词（默认是或关系）

```shell
apropos "list copy"
```

#### 搜索多个词，逻辑与匹配

```shell
apropos -a list directory
```

#### 精准匹配

```shell
apropos -e set
```

#### 搜索指定章节的

在第 1 节和第 8 节中搜索

```shell
apropos -s 1,8 list
```

#### 使用正则表达式搜索

查找以单词 `list` 开头的所有手册页

```shell
apropos '^list'
```

#### 正则表达式实现或的关系

```shell
apropos "zip(note|cloak|info)"
```

#### 多种选项结合

```shell
apropos -a -s 3,8 "^list" "(implementation|devices|users)"
```

#### 避免截断

默认会修剪输出中的描述，输出以省略号结尾

```shell
apropos -l list
```

### whatis vs apropos

|  功能   |  whatis   |  apropos   |
| --- | --- | --- |
|  搜命令名   |  ✅   |  ✅   |
|  搜描述内容  |  ❌   |  ✅   |
|  匹配模糊内容   |  ❌   |  ✅   |
|  精准   |  ✅   |  ❌   |