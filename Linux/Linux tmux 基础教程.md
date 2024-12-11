### 简介

`tmux`（Terminal Multiplexer：终端复用器）是一款功能强大的工具，可以在单个终端窗口中管理多个终端会话。它使用户能够分离和重新连接会话、将窗口拆分为窗格以及运行持久终端会话，非常强大。

### 安装

```shell
sudo apt install tmux    # On Debian/Ubuntu
sudo yum install tmux    # On CentOS/RHEL
sudo dnf install tmux    # On Fedora
```

### 常用命令

#### 启动 `tmux` 创建一个新的 `session` 会话

```shell
tmux
```

#### 创建一个新的 `session` 并指定名称

```shell
tmux new -s <session_name>
```

#### 重命名 `session` 名称

```shell
tmux rename-session -t <old_session_name> <new_session_name>
```

#### 列出所有激活的 `session`

```shell
tmux ls
```

#### 重新附加到 `session`

```shell
tmux attach -t <session_name>

tmux a -t <session_name>

tmux a 不接会话名则进入最近的回话
```

#### 杀死一个 `session`

```shell
tmux kill-session -t <session_name>
```

#### 水平分割窗口

```shell
先按 Ctrl-b，再按 %
```

#### 垂直分割窗口

```shell
先按 Ctrl-b，再按 "
```

#### 在窗格之间切换

```shell
先按 Ctrl-b，再按上下箭头切换
```

#### 调整窗格大小

```shell
先 Ctrl-b，再按 :resize-pane -L是调整左边，-R是调整右边，-U是调整上边，-D是调整下边

还有简单方法是：
1. 先Ctrl-b
2. 再按住shift键不松
3. 然后按h是缩小左窗格，按l是缩小右窗格，按j是缩小下窗格，按k是缩小上窗格
其对应的是Vim的上下左右键
```

#### 关闭窗格/窗口

```shell
Ctrl-d
exit
```

#### 创建新的窗口

```shell
先按 Ctrl-b，再按 c
```

#### 切换到下一个窗口

```shell
先按 Ctrl-b，再按 n
```

#### 切换到上一个窗口

```shell
先按 Ctrl-b，再按 p
```

#### 使用窗口编号切换到指定窗口

```shell
先按 Ctrl-b，再按数字，例如：0，1，2
```

#### 重命名当前的窗口

```shell
先按 Ctrl-b，再按逗号(,)，然后输入新名称
```

#### 进入复制模式

可以上下翻页，查看命令历史

```shell
先按 Ctrl-b，再按 [
```

#### 列出所有窗口

```shell
先按 Ctrl-b，再按 w
```

#### 离开 `session`

```shell
先按 Ctrl-b，再按 d
```

#### 已经在 `tmux` 里面时查看所有会话

```shell
先按 Ctrl-b，再按 s
```

#### 查看所有快捷键

```shell
先按 Ctrl-b，再按 ?
```

#### 关闭当前的窗格

```shell
先按 Ctrl-b，再按 x
```

#### 打开命令行模式

```shell
先按 Ctrl-b，再按 :
```

#### 修改配置文件

```shell
# Set prefix to Ctrl-a instead of Ctrl-b
# 设置前缀为Ctrl-a而不是Ctrl-b
set -g prefix C-a
unbind C-b
bind C-a send-prefix

# Enable mouse support
# 启用鼠标支持
set -g mouse on

# Use 256 colors
set -g default-terminal "screen-256color"

# 配置水平分割键绑定为 -
bind - split-window -h

# 配置垂直分割键绑定为 _
bind _ split-window -v

# 解除默认分割窗口快捷键
unbind %
unbind '"'
```

修改完使用 `tmux source-file ~/.tmux.conf` 来生效