## 一、简介

rsync (Remote Sync)是Linux/Unix系统中最常用的命令，用于远程和本地复制和同步文件和目录。

在rsync命令的帮助下，可以跨目录、磁盘、网络远程和本地复制和同步数据，执行数据备份，并在两台Linux机器之间进行镜像。

rsync与传统的复制命令不同，rsync使用增量传输算法只传输源文件和目标文件之间的差异，这种方法极大地减少了带宽使用并加快了传输速度。   

rsync可以用于镜像数据、增量备份、系统间文件复制，也可以替代scp、sftp和cp命令。

## 二、基本语法

* 本地到本地：rsync [OPTION] [SRC] [DEST]

* 本地到远程：rsync [OPTION] [SRC] [USER@][HOST]:[DEST]

* 远程到本地：rsync [OPTION] [USER@][HOST]:[SRC] [DEST]

OPTION表示命令的选项

SRC表示源地址

DEST表示目标地址

USER表示远程主机用户名

HOST表示远程主机

## 三、常用选项

* `-v或-verbose`：在传输过程中提供更详细的输出。

* `-a或-archive`：归档模式，传输过程中包括递归复制和保存文件权限、时间戳、符号链接和设备文件。

* `-r或-recursive`：递归复制目录中的文件。

* `-delete`：文件或目录在源地址中不存在，但在目标中已存在，则删除。

* `–exclude=[PATTERN]`：排除与指定模式匹配的文件或目录。

* `–include=[PATTERN]`：包含与指定模式匹配的文件或目录。

* `-z或-compress`：在传输过程中压缩文件数据以减少带宽使用。

* `–dry-run`：执行试运行而不进行任何实际更改。

* `–temp-dir`：指定存储临时文件的目录。

* `-u或–update`：跳过目标目录中比源文件新的文件，以便仅更新较旧的文件。

* `-h或–human-readable`：以人类可读的格式输出数字。

* `-i或–itemize-changes`：输出传输过程中所做更改的列表。

* `–progress`：在传输过程中显示进度条。

* `–stats`：完成后提供文件传输统计信息。

* `-e或–rsh=[COMMAND]`：指定要使用的远程 shell。

* `–bwlimit=[RATE]`：限制带宽以提高网络效率。

* `-P或–partial –progress`：保留部分传输的文件并显示进度。

* `-q或--quiet`：抑制信息输出。

* `--max-size`：指定传输的大小限制。

* `--remove-source-files`：传输完成后删除源地址文件或目录。

* `--backup`：备份文件操作。

* `--backup-dir`：指定备份存储的目录

### 四、用法示例

* 在本地复制或同步文件

```shell
rsync -zvh abc.tar.gz /tmp/backups/
```

* 在本地复制或同步目录

```shell
rsync -avzh /root/abc /tmp/backups/
```

* 从本地复制文件到远程主机

```shell
rsync -avzh /root/abc root@abc:/root/
```

* 从远程主机复制文件到本地

```shell
rsync -avzh root@abc:/root/abc /tmp/abc
```

* 使用SSH将文件从远程主机复制到本地

```shell
rsync -avzhe ssh root@abc:/root/abc /tmp
```

* 使用SSH将文件从本地复制到远程主机

```shell
rsync -avzhe ssh /tmp root@abc:/root/abc
```

* 使用SSH传输时指定端口

```shell
rsync -a -e "ssh -p 2322" /opt/media/ root@abc:/opt/media/
```

* 传输过程中显示进度条

```shell
rsync -avzhe ssh --progress /root/abc root@abc:/root/abc
```

* 只复制指定的文件

```shell
rsync -avz --include='*.txt' /abc root@abc:/root/abc/
```

* 排除指定的文件

```shell
rsync -avz --exclude='*.ext' /abc root@abc:/root/abc/
```

* 包含和排除同时使用

```shell
rsync -avze ssh --include '.txt' --exclude '.obj' root@abc:/var/lib/ /root/abc
```

* 传输过程中删除目标地址存在，但源地址中不存在的文件

```shell
rsync -avz --delete root@abc:/var/lib/abc/ /root/abc/
```

* 设置传输文件大小限制

单位：K、M、G

```shell
rsync -avzhe ssh --max-size='200K' /var/lib/abc/ root@abc:/root/abc
```

* 传输完成后自动删除源地址中的文件或目录

```shell
rsync --remove-source-files -zvh backup.tar.gz root@abc:/tmp/backups/
```

* 模拟传输过程，不会对文件进行更改。

```shell
rsync --dry-run --remove-source-files -zvh backup.tar.gz root@abc:/tmp/backups/
```

* 设置传输的带宽限制

```shell
rsync --bwlimit=100 -avzhe ssh  /var/lib/abc/  root@abc:/root/tmp/
```

* 查看rsync的版本

```shell
rsync --version
```

* 递归复制目录中的文件

```shell
rsync -r abc/ duplicate/
```

* 创建一个备份

备份时会生成增量文件列表，并在原始文件名后附加波形符 (~)

```shell
rsync -av --backup --backup-dir=/path/to/backup/ original/ duplicate/
```