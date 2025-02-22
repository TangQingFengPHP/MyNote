### 简介

`LFTP` 是一个命令行 `FTP` 客户端，支持多种文件传输协议，包括 `FTP、FTPS、HTTP、HTTPS和SFTP` 。它以其通过镜像、后台操作和脚本支持等特性有效管理复杂传输的能力而闻名。

### 安装

* `Ubuntu/Debian`

```shell
sudo apt update
sudo apt install lftp
```

* `CentOS/RHEL/Fedora`

```shell
sudo yum install lftp
```

### 常用选项

* `-u`：指定用户名和密码

* `-e`：打开连接后执行命令

* `-f`：使用脚本文件执行命令

* `-c`：启动 LFTP 并直接运行命令（无需进入交互模式）

* `--parallel`：启用多个并行连接以提高下载/上传速度

* `-p`：为 FTP 或 SFTP 服务器设置自定义端口

### 常用子命令

* `open`：打开与服务器的连接

* `ls`：列出远程服务器上的文件和目录

* `cd`：更改远程服务器上的目录

* `get`：从远程服务器下载文件

* `put`：将文件上传到远程服务器

* `mget`：下载多个文件

* `mput`：上传多个文件

* `mirror`：镜像（同步）目录

* `exit`：退出 LFTP 会话

* `set`：设置各种 LFTP 选项（例如速度限制）

* `-u username,password`：指定用户名和密码

* `-e "command"`：连接后执行单个命令

### 示例用法

#### 启动 LFTP

只需在终端中输入 `lftp` 即可启动 `LFTP` 交互模式

```shell
lftp
```

#### 连接到服务器

使用 `open` 命令连接到服务器。适用于任何受支持的协议（FTP、FTPS、SFTP 等）

```shell
lftp open ftp://username:password@hostname
```

**示例**

```shell
lftp open ftp://user:password@ftp.example.com
```

**使用SFTP**

```shell
lftp sftp://username@hostname
```

**具有显式 SSL/TLS 加密的 FTP（FTPS）**

```shell
lftp -u username,password -e "set ftp:ssl-allow yes; open ftp://hostname"
```

#### 列出远程服务器上的文件

```shell
ls
```

#### 更改目录

```shell
cd remote_directory
```

#### 上传文件

```shell
put local_file
```

#### 上传多个文件

```shell
mput *.txt
```

#### 下载文件

```shell
get remote_file
```

#### 下载多个文件

```shell
mget *.txt
```

#### 镜像目录

* 将远程镜像到本地

```shell
mirror remote_directory local_directory
```

* 本地镜像到远程

```shell
mirror -R local_directory remote_directory
```

* 使用附加选项进行镜像

使用 `--delete` 删除源上不再存在的文件

```shell
mirror --delete remote_directory local_directory
```

#### 退出 LFTP

```shell
exit
```

#### 后台传输

```shell
lftp -e "get remote_file &"
```

#### 后台传输多个命令

多个命令用分号隔开

```shell
lftp -e "open ftp://username:password@hostname; get remote_file; exit"
```

#### 在脚本中使用 LFTP

```shell
#!/bin/bash
lftp -e "open ftp://username:password@hostname; put local_file; get remote_file; exit"
```

#### 设置传输速率

```shell
lftp -e "set net:limit-rate 100000; open ftp://username:password@hostname; get remote_file; exit"
```

#### 递归文件下载

```shell
lftp -e "mirror --reverse --verbose /remote_path /local_path; exit"
```

#### 并行连接

```shell
lftp -u username,password -e "set mirror:parallel-transfer-count 5; mirror remote_directory local_directory; exit"
```