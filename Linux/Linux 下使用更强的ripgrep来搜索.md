### 简介

`ripgrep`（通常缩写为 `rg` ）是一个快速高效的命令行搜索工具，它可以递归地在当前目录中搜索正则表达式模式。它类似于 `grep` ，但设计得更快，特别是对于大型代码库。它可以使用优化的算法和多线程，以闪电般的速度搜索文件、目录甚至压缩文件。它支持高级搜索功能，如正则表达式、文件类型过滤等。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt update
sudo apt install ripgrep
```

* `CentOS/RHEL`

```shell
sudo yum install ripgrep
```

* `Fedora`

```shell
sudo dnf install ripgrep
```

* `macOS`

```shell
brew install ripgrep
```

### 基础语法

```shell
rg [options] PATTERN [PATH]
```

* `PATTERN`：要搜索的正则表达式或字符

* `PATH`：要搜索的目录（或文件）。如果未指定，则默认为当前目录

### 示例用法

#### 基本用法

> 递归搜索当前目录及其子目录中的所有文件中的单词 “error”

```shell
rg "error"
```

#### 在特定目录中搜​​索

```shell
rg "error" /var/log
```

#### 不区分大小写搜索

> 默认情况下，ripgrep 区分大小写。使用 -i 使搜索不区分大小写

```shell
rg -i "error"
```

#### 显示行号

```shell
rg -n "error"
```

#### 仅列出包含匹配项的文件的名称

> 不显示实际匹配项

```shell
rg -l "error"
```

#### 显示匹配数

> 显示每个文件的匹配数

```shell
rg -c "error"
```

#### 在特定类型的文件中搜索

> 要在特定类型的文件中搜索（例如，仅 .txt 文件）

```shell
rg -t txt "error"
```

#### 指定文件类型且合并其他选项

```shell
rg -t txt -i "error"
```

#### 列出可用的文件类型

```shell
rg --type-list
```

#### 仅搜索整个单词

> 要搜索整个单词的模式（而不是单词的一部分）

```shell
rg -w "error"
```

#### 排除文件或目录（--glob 或 -g）

> 这会将 .git 目录下的文件排除在搜索之外

```shell
rg -g "!.git/*" "error"
```

#### 使用多种模式搜索 (-e)

> 可以通过为每个模式提供 -e 选项来搜索多个模式

```shell
rg -e "error" -e "warning"
```

#### 搜索压缩文件

> 默认情况下，ripgrep 会跳过压缩文件，但可以使用 -z 标志让它搜索压缩文件（例如 .gz、.tar.gz）

```shell
rg -z "error"
```

#### 限制搜索深度（--max-depth）

> 这会将搜索限制在前两级子目录中

```shell
rg --max-depth 2 "error"
```

#### 搜索二进制文件（-a 或 --binary）

> 默认情况下，ripgrep 会跳过二进制文件

```shell
rg -a "error"
```

#### 显示所有匹配的行（不仅仅是第一行）

> 默认情况下，ripgrep 仅显示每个文件中的第一个匹配项

```shell
rg -H "error"
```

#### 搜索单词边界

> 要搜索单词边界（例如，error 但不是errors）

```shell
rg "\berror\b"
```

#### 搜索多个单词

> 可以使用 | 作为 OR 条件来搜索多个模式

```shell
rg "error|warning"
```

#### 搜索行首

```shell
rg "^error"
```

#### 搜索行尾

```shell
rg "error$"
```

#### 在所有 .js 文件中搜索

```shell
rg -t js "console"
```