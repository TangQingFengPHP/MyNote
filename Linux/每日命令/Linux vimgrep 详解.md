### 简介

* `:vimgrep` 是 `Vim` 提供的「直接在指定文件集里用正则查找」的命令

* 与外部 `grep` 不同，`vimgrep` 在查到结果后会将匹配行写入 快速修复列表（`quickfix list`），并可通过 `:copen、:cnext、:cfirst` 等命令逐条跳转

* 支持 `Vim` 的正则引擎，允许灵活使用 `Vim` 正则、分组、魔法模式等

### 基本语法

```shell
:vimgrep[!] /{pattern}/[g][j] {file_pattern}[ ...]
```

* `!`：表示清空当前 `quickfix` 列表；不带 `!` 时会追加到现有列表

* `/…/`：`Vim` 正则模式，若包含 `/`，可改用其它分隔符，如 `#…#`

* `g`：对匹配到多处的行，依然只在列表中记录一次；不加 `g` 时同一行多次匹配会重复记录

* `j`：匹配后不跳转到第一个结果，仅更新列表不移动光标

* `{file_pattern}`：文件通配符，支持 `*`, `**`（递归）、`?` 等

例 1：搜索当前目录下所有 `.c` 文件中包含 `“TODO”` 的行

```shell
:vimgrep /TODO/ *.c
```

* 执行后，打开 `quickfix` 列表：

```shell
:copen
```

* 跳到下一个匹配：

```shell
:cnext
```

* 跳到上一个匹配：

```shell
:cprevious
```

### 文件搜索范围示例

|  场景   |  命令示例   |  说明   |
| --- | --- | --- |
|  当前文件   |  `:vimgrep /error/ %`   |  `%` 表示当前缓冲区   |
|  所有文本文件   |  `:vimgrep /TODO/ *.txt`   |  当前目录下 `.txt` 文件  |
|  递归搜索子目录   |  `:vimgrep /function_name/ **/*.py`   |  所有 Python 文件   |
|  多个文件   |  `:vimgrep /pattern/ file1.txt file2.txt`   |  多个文件   |
|  多类型文件组合搜索	   |  `:vimgrep /pattern/ **/*.cpp **/*.h`   |   C++ 头文件和源文件  |
|  所有缓冲区	   |  `:vimgrep /pattern/ ##`   |   搜索所有打开的缓冲区  |

### 核心用法示例

#### 基础搜索（当前目录指定文件）

```shell
:vimgrep "error" *.log  " 在当前目录所有.log文件中搜索"error"
```

#### 递归搜索（包含子目录）

使用 `**` 匹配所有子目录：

```shell
:vimgrep "func"**/*.py  " 在所有子目录的.py文件中搜索"func"
```

#### 正则表达式搜索

支持 `Vim` 风格正则（如 `\d` 匹配数字，`\w` 匹配单词字符）：

```shell
:vimgrep "age=\d+" **/*.html  " 搜索所有html中"age=数字"的模式
```

#### 区分大小写 / 忽略大小写

* 默认区分大小写

* 加 `i` 参数忽略大小写：

```shell
:vimgrep /Error/ig **/*.js " i = 忽略大小写，g = 全局匹配（每个文件找所有匹配）
```

### 结合快速修复列表（Quickfix）

* 打开列表

```shell
:copen    " 在窗口下方打开 quickfix 窗口
:cclose   " 关闭 quickfix 窗口
```

* 在列表中跳转

```shell
:cfirst   " 跳到第一个匹配
:clast    " 跳到最后一个匹配
:cnext    " 下一个
:cprevious" 上一个
:cc [nr]  " 跳到第 nr 条（不带 nr 跳到下一条）  
```

* 在列表中执行批量操作

比如对所有匹配行做修改，可配合 `:cfdo`

```shell
:cfdo s/old/new/ge | update
```

这会对 `quickfix` 中的每个文件执行替换并保存。

* 快捷键映射（加入 `~/.vimrc` 提升效率）：

```shell
nnoremap <F8> :copen<CR>     " F8 打开 Quickfix
nnoremap <F9> :cclose<CR>    " F9 关闭
nnoremap <F10> :cnext<CR>    " F10 下一项
nnoremap <F11> :cprev<CR>    " F11 上一项
```

### 常用选项和技巧

* 只列出文件名

```shell
:vimgrep /pattern/ **/*.py | cw
```

然后在 `quickfix` 窗口里可以只关注文件名列。

* 使用非常规分隔符

当搜索模式里包含 `/` 时，用其它符号：

```shell
:vimgrep #{foo/bar}# **/*.js
```

* 只更新列表、不跳转

```shell
:vimgrep j /TODO/ **/*.java
```

搜索后光标不动，便于接着做别的。

* 区分大小写

`Vim` 默认受 `ignorecase` 和 `smartcase` 影响，你可临时加 `\C` 强制大小写敏感，或 `\c` 强制忽略大小写：

```shell
:vimgrep /\CError/ **/*.log
```

* 与 `:args／argdo` 配合

先加载参数列表：

```shell
:args **/*.md
```

再 `vimgrep` 并跳转：

```shell
:vimgrep /TODO/ % | copen
```

### 高级技巧与性能优化

#### 结合通配符精确筛选文件

```shell
:vimgrep "config" **/*.{json,yml}  " 搜索所有子目录的.json和.yml文件
```

#### 将结果保存到文件

```shell
:vimgrep "todo"**/*.php | :cwrite todo_list.txt  " 搜索结果保存到todo_list.txt
```

#### 批量替换搜索结果

结合 `cdo` 命令（对快速修复列表中的每个文件执行命令）：

```shell
:vimgrep "old_str" **/*.py  " 先搜索目标字符串
:cdo %s/old_str/new_str/g | w  " 对所有匹配文件执行替换并保存
```

#### 自定义快捷键

在 `.vimrc` 中配置快捷键，简化操作：

```shell
nnoremap <leader>vg :vimgrep // **/*.<C-R>=expand('%:e')<CR><Left><Left>
```

按 `<leader>vg` 后，自动填充当前文件类型的搜索模板，直接输入关键词即可

#### 限制搜索范围

```shell
" 只在修改过的文件中搜索
:vimgrep /warning/ `bufname()`  " 当前缓冲区
:vimgrep /bug/ `expand('%:p:h')/*.c`  " 当前目录
```

#### 性能优化

```shell
" 忽略大目录（如 node_modules）
:set wildignore+=*/node_modules/*
:vimgrep /pattern/ **/*.js      " 自动跳过忽略目录
```

#### 正则表达式转义

`Vim` 的正则语法与 `Perl/PCRE` 不同，需注意 `\v`（非常魔术模式）简化语法。

```shell
:vimgrep /\v<word>/ *.txt
```

#### 多文件编辑：

结合 `:argdo` 或 `:bufdo` 进行批量操作

```shell
:argadd *.c
:argdo vimgrepadd /TODO/ % | cdo s/TODO/DONE/g | update
```

#### 添加全局忽略配置

修改 `.vimrc` 文件

```shell
set wildignore+=*.o,*.obj,.git
```

#### `lvimgrep` 替代方案

使用 `:lvimgrep` 将结果存入 `Location List`（窗口局部列表），适用于多窗口并行搜索：

```shell
:lvimgrep /warning/ **/*.log  
:lopen   " 打开 Location List
:lnext 
```

#### 加速 vimgrep 性能

* 问题：`vimgrep` 需模拟打开每个文件，大项目较慢

* 优化：

    * 使用 `:noautocmd` 跳过插件/事件干扰：

    ```shell
    :noautocmd vimgrep /pattern/ **/*.js
    ```

    * 换用外部 `grep`（需配置）：

    ```shell
    :set grepprg=grep\ -nri\ --include=*.{c,h}   定义外部 grep 参数
    :grep "function_name" .                     调用外部 grep    
    ```


### 区别：:vimgrep vs :grep

|              | `:vimgrep`                         | `:grep`                                        |
| ------------ | ---------------------------------- | ---------------------------------------------- |
| 搜索引擎     | Vim 自带正则                       | 调用外部程序（如 grep、ack、rg 等）            |
| 性能         | 较慢（对小项目足够）               | 快（尤其用 ripgrep 等现代搜索工具）            |
| 快速修复列表 | 默认输出到 quickfix                | 需 `:grepprg` 设置或 `:make` 设置才可输出      |
| 正则语法     | Vim 正则（魔法模式、分组语法不同） | 外部工具的正则（POSIX / PCRE / Rust regex 等） |

如果更喜欢 `rg、ag` 的速度并想结合 `quickfix`，只需在 `.vimrc` 中：

```shell
set grepprg=rg\ --vimgrep\ --no-heading\ --smart-case
set grepformat=%f:%l:%c:%m
```

这样用 `:grep` 就等同于 `:vimgrep` 加速版。

### 进阶：批量替换与脚本化

#### 全项目批量替换

```shell
:vimgrep /foo()/ **/*.cpp
:cfdo %s/foo()/bar()/g | update
```

#### 在 Vim 脚本中使用

```shell
function! ReplaceInProject(pat, rep)
  execute 'vimgrep /'.a:pat.'/j **/*.js'
  copen
  execute 'cfdo %s/'.a:pat.'/'.a:rep.'/g | update'
  cclose
endfunction

" 用法：
:call ReplaceInProject('oldFunc', 'newFunc')
```

### 实战示例

#### 案例 1：项目级重构

```shell
" 查找所有使用旧 API 的地方
:vimgrep /\<old_api_call\>/ **/*.c
:copen   " 查看结果并手动验证
:cfdo %s/old_api_call/new_api_call/g | update
```

#### 案例 2：日志分析

```shell
" 在所有日志中提取 ERROR 行
:vimgrep /^$$ERROR$$/ /var/log/*.log
:copen
" 在 quickfix 窗口按 : 输入：
:cfdo g/^$$ERROR$$/yank A | put! A 将所有错误复制到新缓冲区
```