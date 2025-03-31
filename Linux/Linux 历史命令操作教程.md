### 简介

`Linux` 使用 `history` 命令记录命令历史记录并将其存储在文件 (`~/.bash_history` 或 `~/.zsh_history`) 中。可以使用不同的方法配置和操作历史记录。

### 常用操作

#### 查看所有命令

```shell
history

# 这将显示所有先前执行的命令以及行号
```

#### 显示最新10条命令

```shell
history 10
```

#### 搜索命令历史

```shell
history | grep "keyword"
```

示例：

```shell
history | grep ls
```

使用反向搜索（`CTRL + R`）

* 按 `CTRL + R` 并开始输入部分命令

* 继续按 `CTRL + R` 循环搜索命令

* 按 `Enter` 键执行选定的命令

#### 执行历史记录中的命令

```shell
!<command_number>
```

示例：执行历史记录中第100号命令

```shell
!100
```

#### 重新运行最后一个命令

```shell
!!
```

#### 运行以特定单词开头的最后一个命令

```shell
!ls
```

#### 清除当前会话历史记录

```shell
history -c
```

#### 删除指定命令

```shell
history -d <command_number>
```

示例：

```shell
history -d 50

# 删除编号50的命令
```

#### 永久清除历史记录

```shell
> ~/.bash_history

或

cat /dev/null > ~/.bash_history
```

#### 变更历史文件位置

> 修改 `HISTFILE` 变量

```shell
export HISTFILE=~/.my_custom_history
```

#### 设置存储命令的数量

```shell
export HISTSIZE=1000   # 内存中存储的命令数
export HISTFILESIZE=2000  # 历史文件中存储的命令数
```

#### 忽略特定命令

```shell
export HISTIGNORE="ls:pwd:exit"

# 列出的命令将不会保存在历史记录中
```

#### 忽略重复项

```shell
export HISTCONTROL=ignoredups
```

#### 忽略重复的命令和前导空格

```shell
export HISTCONTROL=ignoreboth
```

#### 实时将所有命令记录到文件中

```shell
export PROMPT_COMMAND='history -a'

# 这会将每个命令立即附加到历史记录中
```

#### 保存时间戳在历史记录中

```shell
export HISTTIMEFORMAT="%F %T "
```

现在历史记录将显示：

```shell
  1  2024-03-31 10:15:30  ls
  2  2024-03-31 10:15:35  cd /home
```

#### 防止其他用户查看你的历史记录

```shell
chmod 600 ~/.bash_history
```

#### 在不同的 Shell 中查看历史记录

* `Bash`：`history`, `~/.bash_history`

* `Zsh`：`history`, `~/.zsh_history`

* `Fish`：`history`, `~/.local/share/fish/fish_history`





