### 简介

`ack` 命令是一款专为开发人员设计的强大文本搜索工具。它比 `grep` 更快速、更高效地搜索源代码，并具有忽略不相关文件（例如二进制文件、版本控制文件、临时文件）等内置功能，`ack` 命令的目标是通过应用它自己来搜索特定类型文件

`ack` 通过识别相关文件并只搜索这些文件而提高了性能。`ack` 还引入了优化的正则表达式，旨在提高模式匹配的效率。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt update
sudo apt install ack
```

* `CentOS、Fedora`

```shell
sudo yum install ack
sudo dnf install ack
```

* `MacOS`

```shell
brew install ack
```

* 小于等于 `Debian 9` 或 `Ubuntu 19.10 ` 版本

由于历史原因已经有一个与该 `ack` 无关的名为“ack”的软件包

```shell
sudo apt install ack-grep
```

### 常用选项

`ack` 可以智能地搜索文件，它根据文件的扩展名以及某些情况下的文件内容来识别某些文件类型。可以使用 `--type` 选项进行这些选择。

* `-a, --all`：对所有文件进行操作，无论类型如何（但仍会跳过 `blib`、`CVS` 等目录以及备份文件：`~`）

* `-c, --count`：打印每个输入文件的匹配行计数

* `--color`：突出显示匹配的文本，默认开启

* `--nocolor`：抑制颜色

* `--noenv`：禁用所有环境变量，不会读取 `.ackrc` 配置文件的内容，并忽略所有环境变量。默认情况下，`ack` 会考虑 `.ackrc` 和环境变量。

* `-f`：仅打印要搜索的文件，而不实际进行任何搜索。

* `-H, --with-filename`：打印每个匹配项的文件名

* `--help`：打印帮助信息

* `-i, --ignore-case`：匹配时忽略大小写

* `--[no]ignore-dir=<DIRNAME>`：忽略或不忽略指定的目录

* `-l, --files-with-matches`：仅打印匹配文件的文件名，而不是匹配的文本。

* `-L, --files-without-matches`：仅打印不匹配的文件的文件名。相当于指定 `-l` 和 `-v`

* `--match <REGEX>`：明确指定正则表达式，在不想将 `regex` 作为第一个参数，非常有用

* `-m=<NUM> , --max-count=<NUM>`：在达到指定的匹配数量后停止读取文件

* `-n, --no-recurse`：禁止递归

* `-o`：仅显示每行与 `PATTERN` 匹配的部分（关闭文本突出显示）

* `--pager=<program>`：指定分页器，如指定 `--pager=less`

* `--print0`：文件名以空字节分隔输出，而不是通常的换行符，这在处理包含空格的文件名时很有用

> 删除所有 `html` 类型的文件
> `ack -f --html --print0 | xargs -0 rm -f`

* `-r, -R, --recurse`：递归到子目录，默认开启

* `--show-types`：输出 `ack` 与每个文件关联的文件类型

* `--type=<TYPE>, --type=<noTYPE>`：指定要在搜索中包含或排除的文件类型。`TYPE` 是文件类型，如 `perl` 或 `xml`。`--type=perl` 也可以指定为 `--perl` ，而 `--type=noperl` 可以指定为 `--noperl`。

* `--type-add <TYPE> =.EXTENSION[,.EXT2[,...]]`：追加类型的扩展

* `--type-set <TYPE> =.EXTENSION[,.EXT2[,...]]`：替换现有的类型或定义新的类型和相关的扩展

* `-u, --unrestricted`：自由模式，搜索所有文件和目录，不会跳过任何内容

* `-v, --invert-match`：反转匹配，选择不匹配的行

* `--version`：显示版本信息

### 示例用法

#### 在当前目录和子目录中搜索单词 `main`

```shell
ack "main"
```

#### 搜索时指定文件类型

```shell
ack "pattern" --type=type

# 如：在Python文件中搜索function关键字
ack "function" --type=python
```

#### 列出 `ack` 能识别的所有文件类型

```shell
ack --help-types
```

#### 匹配时忽略大小写

```shell
ack -i "pattern"
```

#### 仅显示匹配到的文件名

```shell
ack -l "pattern"
```

#### 排除指定的文件或目录

```shell
ack "pattern" --ignore-dir=dir_name

ack "TODO" --ignore-dir=vendor
```

#### 在指定的文件或目录中搜索

```shell
ack "pattern" file_or_dir

# 如：
ack "import" src/
```

#### 显示匹配到的文件行号

```shell
ack -n "pattern"
```

#### 使用正则表达式搜索

```shell
ack "^class\s\w+"

# 匹配以 class 开头、后跟空格和单词的行
```

#### 统计匹配项

```shell
ack -c "pattern"
```

#### 反转匹配（没有模式的行）

```shell
ack -v "pattern"
```

#### 禁用颜色

```shell
ack "pattern" --nocolor
```

#### 配置 `ack` 使用 `less` 作为分页器

```shell
echo '--pager=less -RFX' >> ~/.ackrc
```

#### 获取文件数量

```shell
ack -f | wc -l
```

#### 使用文件类型作为选项来搜索

```shell
ack --sass blue

# ack 仅查看 SASS 文件（以 .sass 或 .scss 结尾）的文件
```

#### 使用文件类型作为选项排除指定的文件类型

```shell
ack -w --nosass red

# 在类型前加 no 即可
```

#### `Vim` 中集成 `ack`

在 `.vimrc` 配置文件中设置：

```shell
set grepprg=ack\ -a
```

### 如何修改配置文件？

`~/.ackrc` 一般位于家目录下，文件包含命令行选项，这些选项在处理之前会添加到命令行中。多个选项可以位于多行中。以 # 开头的行将被忽略。

带有空格的参数不需要用引号引起来，因为它们不会被 `shell` 解释

```shell
# Always sort the files
--sort-files

# Always color, even if piping to a another program
--color

# Use "less -r" as my pager
--pager=less -r

# 在配置文件中必须使用以下格式定义
# 选项=类型=扩展1,扩展2
# 选项(换行)
# 类型=扩展1,扩展2

# 追加到已存在的类型上
--type-add=perl=.xs 

# 定义新类型或替换
--type-set=eiffel=.e,.eiffel
--type-set
eiffel=.e,.eiffel
```