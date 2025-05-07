### 简介

`tldr` 代表 `Too Long; Didn't Read`。它是一个由社区维护的类 `unix` 命令的简化和实用命令行示例集合。 
 
它为常用命令提供了简洁的、由示例驱动的帮助，而不像详细而冗长的手册页。

### 安装

依赖 `node.js`, 需要先安装 `node.js`

* 使用 `npm` 安装

```shell
npm install -g tldr
```

* `Ubuntu/Debian`

```shell
sudo apt install tldr
```

* `macOS`

```shell
brew install tldr
```

### 基础用法

```shell
tldr <command>
```

示例：

```shell
tldr tar
```

示例输出：

```shell
# tar

> Archiving utility.
> Commonly used for compressing and extracting files.

- Create a .tar archive from files:
  tar cf target.tar file1 file2 file3

- Extract a .tar archive:
  tar xf archive.tar
```

### 常用选项

* `-u, --update`：更新本地 `tldr` 页面缓存

* `-l, --list`：列出所有可用命令

* `--version`：显示 `tldr` 客户端版本

* `--help`：显示可用的标志和选项

* `--platform`：显示其他平台的结果（例如 `Linux、OSX、Windows`）

### 用法示例

#### 列出所有页面（本地缓存）

```shell
tldr -l
```

#### 关键词搜索

```shell
tldr -s "list of all files, sorted by modification date"
```

#### 更新本地缓存

```shell
tldr -u
```

#### 查询其他平台

```shell
tldr --platform osx find
```

#### 清理本地数据库缓存

```shell
tldr -c
```

#### 与 fzf 结合进行模糊搜索

```shell
tldr -l | fzf --preview "tldr {}"
```

#### 在浏览器中查看 tldr 内容

在线使用 `tldr` 的功能

```url
https://tldr.inbrowser.app
```

### fzf 集成 tldr

将下列函数添加到 `~/.bashrc 或 ~/.zshrc`

#### 基本集成

```shell
tldr-fzf() {
  local cmd
  cmd=$(tldr -l | fzf --height 40% --reverse --preview "tldr {}" --preview-window=wrap)
  [[ -n "$cmd" ]] && tldr "$cmd"
}
```

重载配置文件

```shell
source ~/.bashrc    # or ~/.zshrc
```

运行

```shell
tldr-fzf
```

将看到所有可用 `tldr` 页面的可搜索列表，右侧有实时预览 - 只需选择一个并按 `Enter` 即可打开它。

#### 使用 less 作为分页器

```shell
tldr-fzf() {
  local cmd
  cmd=$(tldr -l | fzf --height 40% --reverse --preview "tldr {}" --preview-window=wrap)
  [[ -n "$cmd" ]] && tldr "$cmd" | less
}
```

#### 高级版

```shell

tldr-fzf() {
  local selection
  selection=$(find ~/.tldr/pages*/ -type f -name "*.md" \
    | sed 's|.*/||; s|\.md$||' \
    | sort -u \
    | fzf --height 40% --reverse --prompt="TLDR > " \
          --preview 'tldr {}' \
          --preview-window=wrap)

  [[ -n "$selection" ]] && tldr "$selection"
}
```

* 查找所有平台（通用、`Linux、OSX` 等）的所有 `.md tldr` 文件

* 剥离路径和扩展

* 让 `fzf` 实时预览每个选定命令的完整 `tldr` 页面

#### 更高级：使用缓存搜索描述

```shell
tldr-fzf() {
  local cmd
  cmd=$(tldr -l | while read -r c; do
    echo -e "$c\t$(tldr $c | sed -n '2p')"
  done | fzf --height 50% --reverse --with-nth=1,2 --delimiter='\t' \
        --preview='tldr {1}' \
        --preview-window=wrap \
        --bind='enter:execute(tldr {1} | less)' \
        | cut -f1)

  [[ -n "$cmd" ]] && tldr "$cmd"
}
```

* 显示命令名称及其单行描述

* 使用 `fzf` 预览完整的命令帮助

* 按 `Enter` 键打开 `less` 命令