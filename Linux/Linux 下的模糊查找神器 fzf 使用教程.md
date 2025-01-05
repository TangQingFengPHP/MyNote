### 简介

`fzf` 是一款功能强大且用途广泛的 `Linux` 命令行模糊查找器。它允许用户使用模糊匹配高效地搜索和过滤文本、文件和命令历史记录。

它是一个交互式过滤程序，适用于任何类型的列表；文件、命令历史、进程、主机名、书签、git提交等。它实现了一种“模糊”匹配算法，因此可以快速键入带有省略字符的模式，并且仍然可以得到想要的结果。

![alt text](/images/fzf.png)

### 安装

* `Debian/Ubuntu`

```shell
sudo apt install fzf
```

* `Red Hat/CentOS`

```shell
sudo dnf install fzf
```

* `MacOS`

```shell
brew install fzf
```

* 从源码安装

```shell
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf

~/.fzf/install
```

### 常用选项

> 大多数长选项都有带有 `--no-` 前缀的相反版本。

* `-x, --extended`：扩展搜索模式，默认启用。可以使用 `+x` 或 `--no-extended` 禁用

* `-e, --exact`：精确匹配搜索

* `-i, --ignore-case`：大小写不敏感，默认是(smart-case)，即智能识别大小写匹配

* `+i, --no-ignore-case`：大小写敏感匹配

* `+s, --no-sort`：不排序匹配到的结果

* `--tail=<NUM>`：内存中保存的最大项目数，当使用 `fzf` 浏览无限数据流（例如日志流）同时限制内存使用量时，这很有用。

示例：

```shell
# Interactive filtering of a log stream
tail -f *.log | fzf --tail 100000 --tac --no-sort --exact
```

* `--tac`：反转输入的顺序

* `-m, --multi`：使用 `tab/shift-tab` 启用多选

* `+m, --no-multi`：禁用多选

* `--bind=<KEYBINDS>`：自定义绑定键事件，用逗号分割

* `--wrap`：启用换行

* `--height=<EXPR>`：按照给定的高度在光标下方显示 `fzf` 窗口，而不是使用全屏，如果指定了负值，则高度计算为终端高度减去给定值

示例：

```shell
fzf --height=-1
```

当以 `~` 为前缀时，`fzf` 会根据输入的大小自动确定范围内的高度

```shell
fzf --height=~70%
```

* `--min-height=<HEIGHT>`：当 `--height` 以百分比形式指定时，指定的最小高度（默认值：10）。当未指定 `--height` 时，将被忽略。

* `--layout=<LAYOUT>`：布局选择，可用的值有：

1. `default`：从屏幕底部显示

2. `reverse`：从屏幕顶部显示

3. `reverse-list`：从屏幕顶部显示，在底部提示

* `--reverse`：与 `--layout=reverse` 等同

* `--prompt=<STR>`：输入提示符，默认是 `>`

* `--pointer=<STR>`：指向当前行的指针，默认是 `▌ 或 > `

* `--marker=<STR>`：多选标记，默认是 `▌ 或 > `

* `-q, --query=<STR>`：使用给定的查询启动查找器

* `-0, --exit-0`：如果初始查询（`--query`）没有匹配项，则不启动交互式查找器并立即退出

* `-f, --filter=<STR>`：过滤模式。不启动交互式查找器，与 `--no-sort` 一起使用时，`fzf` 将成为 `grep` 的模糊匹配版本

* `--version`：显示版本信息

* `--help`：显示帮助信息

* `--man`：查看命令手册

### `shell` 集成

* `--bash`

```shell
eval "$(fzf --bash)"
```

* `--zsh`

```shell
source <(fzf --zsh)
```

* `--fish`

```shell
fzf --fish | source
```

### 设置环境变量

* `FZF_DEFAULT_COMMAND`

> 设置默认命令

```shell
export FZF_DEFAULT_COMMAND='find . -type f'
```

* `FZF_DEFAULT_OPTS`

> 设置默认选项

```shell
export FZF_DEFAULT_OPTS="--layout=reverse --border --cycle"
```

* `FZF_DEFAULT_OPTS_FILE`

> 包含默认选项的文件的位置

```shell
export FZF_DEFAULT_OPTS_FILE=~/.fzfrc
```

### 扩展搜索模式

* 精确匹配（引用）

以单引号字符 (`'`) 为前缀的术语被解释为“完全匹配”（或“非模糊”）术语。

* 锚定匹配

以 `^` 为前缀表示匹配以给定字符串开头的行
以 `$` 为后缀表示匹配以给定字符串结尾的行

* 否定匹配

以 `!` 开头的字符串，表示排除给定字符串的行

* `OR` 操作符，匹配多个字符串中的任一个

示例：

```shell
^core go$ | rb$ | py$
```

### 示例用法

#### 基础用法

> 打开交互式界面，搜索当前目录中的所有文件。
>
> 使用箭头键或键入字符来过滤结果
```shell
fzf
```

#### 查找指定的文件

```shell
find . -type f | fzf
```

#### 搜索命令历史

```shell
history | fzf
```

#### 预览文件内容

> 启用文件内容的预览窗口

```shell
fzf --preview="cat {}"
```

#### 通过 `SSH` 使用 `fzf`

> 从 `~/.ssh/config` 或 `known_hosts` 中过滤 `SSH` 主机

```shell
cat ~/.ssh/known_hosts | cut -f 1 -d ' ' | fzf
```

#### 搜索进程

```shell
ps aux | fzf
```

#### 绑定自定义导航键

```shell
fzf --bind='ctrl-k:kill-line,ctrl-u:unix-line-discard'
```

#### 指定输入源

```shell
ls /path/to/files | fzf
```

#### 直接向 `fzf` 传递查询

```shell
fzf -f "query"
```

#### 与 `vim` 结合使用

```shell
vim $(find . -type f | fzf)
```

#### 交互式变更目录

```shell
cd $(find . -type d | fzf)
```

#### 与 `git` 集成

```shell
git branch | fzf
```

#### 启用多选

> 使用 `Tab` 选择多个项目

```
fzf --multi
```

#### 使用 `--preview` 选项获取详细输出

```shell
fzf --preview="bat --style=numbers --color=always --line-range :500 {}"
```

#### 过滤指定的文件

```shell
find . -name "*.txt" | fzf
```

