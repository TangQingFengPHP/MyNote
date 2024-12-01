### 简介

`history` 命令显示当前 `shell` 会话中以前执行过的命令列表。这对于无需重新输入命令即可重新调用或重新执行命令特别有用。

### 示例用法

#### 显示命令历史列表

```shell
history

# 示例输出如下：

1  ls -l
2  cd /var/log
3  cat syslog
```

#### 执行历史记录中的命令

```shell
!<number>

!2

# number 表示执行第几条命令
```

#### 限制命令历史显示的条数

```shell
history <number>

history 10
```

#### 清空当前 `shell` 会话的历史命令

```shell
history -c
```

#### 把命令历史写入 `~/.bash_history` 文件中

```shell
history -w
```

#### 从 `~/.bash_history` 文件中读取命令

```shell
history -r
```

#### 删除命令历史中的指定命令

```shell
history -d <number>

history -d 5
```

#### `Ctrl + r` 搜索历史命令

```shell
(reverse-i-search)`cat': cat syslog
```

#### 重新执行上一条命令

```shell
!!
```

#### 重新执行以指定字符串开头的最新历史命令

```shell
!<string>

!cat
```

#### 结合 `grep` 使用

```shell
history | grep "ls"
```

#### 搜索不以指定字符串开头的命令

```shell
!?ls
```

#### 使用负数执行倒数最新的命令

```shell
!-2
```

#### 追加命令历史到 `~/.bashrc_history` 文件

```shell
history -a
```

#### 设置多个 `shell` 会话的命令都追加写入到 `~/.bash_history` 文件

```shell
# 修改 ~/.bashrc 文件，添加以下行

shopt -s histappend
```

### 环境变量设置

#### 设置会话期间存储在内存中的命令数

```shell
export HISTSIZE=1000
```

#### 设置保存在 `~/.bash_history` 文件中的最大命令行数

```shell
export HISTFILESIZE=2000
```

#### 定义重复或某些命令如何存储在历史记录中

* `ignoredups`：忽略重复的命令

* `ignorespace`：忽略以空格开头的命令

* `ignoreboth`：合并以上两者

```shell
export HISTCONTROL=ignoreboth
```

#### 从历史记录中排除指定的命令

```shell
export HISTIGNORE="ls:pwd:exit"
```

#### 启用命令历史中的时间戳

```shell
export HISTTIMEFORMAT="%F %T "

# 示例输出如下：

1  2024-11-29 15:30:01 ls -l
2  2024-11-29 15:32:15 cd /var/log
```

