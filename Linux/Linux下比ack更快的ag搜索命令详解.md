### 简介

`ag` 命令（`The Silver Searcher`）是一款用 `C` 编写的快速且对开发人员友好的文本搜索工具，针对源代码搜索进行了优化。它与 `ack` 类似，但速度更快，因此深受开发人员喜爱，可用于搜索代码库。

它最初是 `ack` 的克隆版，但此后其功能集略有不同。在典型使用中，`ag` 比 `ack` 快 5-10 倍，使用 `Pthreads` 来利用多个 `CPU` 核心并行搜索文件。

默认情况下，`ag` 将忽略文件名匹配 `.gitignore`、`.hgignore` 或 `.ignore`，这些文件可以在正在搜索的目录中的任何位置。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt update
sudo apt install silversearcher-ag
```

* `CentOS/Fedora`

```shell
sudo yum install epel-release
sudo yum install the_silver_searcher
sudo dnf install the_silver_searcher
```
* `MacOS`

```shell
brew install the_silver_searcher
```

### 常用选项

* `-a, --all-types`：搜索所有文件，不包括隐藏文件，也不考虑任何忽略文件

* `-c, --count`：仅打印每个文件中的匹配数，而不是匹配行数，如果想要匹配行数，将输出通过管道传输到 `wc -l`

* `--[no]color`：打印时启用颜色或禁用颜色。

* `-D, --debug`：调试模式，输出额外调试信息

* `--depth <NUM>`：指定搜索深度，-1 表示无限制，默认值为 25。

* `--[no]filename`：打印文件名，默认启用

* `-g <PATTERN>`：指定正则表达式

* `-G, --file-search-regex <PATTERN>`：仅搜索名称与 `PATTERN` 匹配的文件

* `--hidden`：搜索隐藏的文件，但应用忽略文件指定的文件

* `--ignore <PATTERN>`：忽略名称与此模式匹配的文件/目录

* `--ignore-dir <NAME>`：`--ignore` 的别名，与 `ack` 兼容

* `-i, --ignore-case`：忽略大小写

* `-l --files-with-matches`：仅打印包含匹配项的文件的名称，而不是匹配的行。

* `-L, --files-without-matches`：仅打印不包含匹配项的文件的名称

* `--list-file-types`：列出能识别的文件类型

* `-n, --norecurse`：不递归

* `--[no]numbers`：打印/不打印行号，默认是忽略行号

* `-o, --only-matching`：仅打印行的匹配部分

* `-p, --path-to-ignore <STRING>`：指定 `.ignore` 文件的路径

* `--pager <COMMAND>`：指定分页器，如 `less`

* `--parallel`：并行搜索，智能拆分搜索的词，如：

```shell
echo "foo\nbar\nbaz" | parallel "ag {} ."

# 将会启用三个实例并行搜索foo、bar、baz
```

* `-Q, --literal`：不将 `PATTERN` 解析为正则表达式，尝试按普通字符串进行匹配

* `-r, --recurse`：递归搜索目录，默认启用

* `-s, --case-sensitive`：启用大小写敏感匹配

* `-S, --smart-case`：如果 `PATTERN` 中有大写字母，则匹配区分大小写，否则不区分大小写，默认启用

* `--search-binary`：搜索二进制文件以查找匹配项

* `--silent`：抑制所有日志消息，包括错误

* `-t, --all-text`：搜索所有文本文件，不包括隐藏文件

* `-u, --unrestricted`：宽松模式，搜索所有文件，包括二进制、隐藏文件，忽略掉 `ignore` 文件指定的文件。

* `-v, --invert-match`：反向匹配

* `-V, --version`：打印版本信息

* `--vimgrep`：输出结果与 `Vim` 的 `:vimgrep /pattern/g` 格式相同

在 `~/.vimrc` 中配置：

```shell
set grepprg=ag\ --vimgrep\ $* set grepformat=%f:%l:%c:%m
```

* `-w, --word-regexp`：仅匹配整个单词

* `--workers <NUM>`：指定 `<NUM>` 个工作线程，默认值为 `CPU` 核心数，最多为 8 个

* `-z, --search-zip`：搜索压缩文件的内容，目前支持 `gz` 和 `xz`

* `-0, --null, --print0`：使用 `\0` 而不是 `\n` 分隔文件名，允许 `xargs -0 <command>` 正确处理包含空格或换行符的文件名。

* `--`：`--` 用于表示其余参数不应被视为选项，如：

```shell
ag -- --foo
```

### 示例用法

#### 在当前目录下搜索

```shell
ag "<pattern>"

# 如：
ag "function"
```

#### 指定文件的类型搜索

```shell
ag "pattern" --python

# 仅在 Python 文件中搜索“class”
```

#### 列出 `ag` 能识别的文件类型

```shell
ag --list-file-types
```

#### 忽略大小写

```shell
ag -i "pattern"
```

#### 在指定文件或目录中搜索

```shell
ag "pattern" path/to/file_or_dir
```

#### 仅显示匹配到的文件名

```shell
ag -l "pattern"
```

#### 统计匹配到的次数

```shell
ag -c "pattern"
```

#### 显示行号

```shell
ag -n "pattern"
```

#### 搜索时排除指定的文件或目录

```shell
ag "pattern" --ignore-dir=<dir_name>

# 如：
ag "TODO" --ignore-dir=node_modules
```

#### 使用正则表达式搜索

```shell
ag "^class\s\w+"

# 匹配以 class 开头、后跟空格和单词的行
```

#### 反向匹配（显示不匹配的行）

```shell
ag -v "pattern"
```

#### 限制搜索的深度

```shell
ag "pattern" --depth=2
```

#### 仅在 JavaScript 文件中搜索

```shell
ag "debugger" --js
```

#### 同时在多个文件或目录中搜索

```shell
ag UNIX foo bar foobar

# UNIX 是要搜索的字符串
# 后面都是文件名
```