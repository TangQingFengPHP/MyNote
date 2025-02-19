### 简介

`SFTP`（安全文件传输协议）是一种通过网络在计算机之间传输文件的安全方法。它是 `SSH` 协议的一部分，这意味着它在文件传输过程中提供加密通信。`SFTP` 通常用于安全文件管理、远程文件访问和远程文件编辑。

### 常用命令

* `sftp user@host`：通过 SFTP 连接到远程服务器

* `ls`：列出当前远程目录中的文件

* `cd`：更改远程目录

* `pwd`：显示当前远程目录

* `lcd`：更改本地目录

* `lpwd`：显示当前本地目录

* `put`：将文件从本地上传到远程

* `get`：将文件从远程下载到本地

* `rm`：从远程服务器删除文件

* `rename`：重命名远程服务器上的文件

* `exit`：退出 SFTP 会话

### 示例用法

#### 启动 `SFTP` 会话

```shell
sftp user@hostname
```

* `user`：远程主机的用户名

* `hostname`：远程服务器的地址（可以是IP地址或域名）

**示例**

```shell
sftp user@192.168.1.100
```

#### 在 SFTP 中导航

* 列出当前目录中的文件

```shell
ls
```

* 更改远程目录

```shell
cd /path/to/remote/directory
```

* 更改本地目录

```shell
lcd /path/to/local/directory
```

* 打印当前远程目录

```shell
pwd
```

* 打印当前本地目录

```shell
lpwd
```

#### 传输文件

* 上传文件（本地到远程）

```shell
put localfile
```

**示例**

```shell
put myfile.txt
```

* 上传文件到特定的远程目录

```shell
put localfile /remote/directory/remote_file
```

* 下载文件（远程到本地）

```shell
get remotefile
```

**示例**

```shell
get remote_file.txt
```

* 下载文件到特定的本地目录

```shell
get remotefile /local/directory/local_file
```

#### 传输多个文件

* 上传多个文件

```shell
put *.txt
```

* 下载多个文件

```shell
get *.log
```

#### 删除文件

* 删除远程服务器上的文件

```shell
rm remotefile
```

#### 重命名文件

* 重命名远程服务器上的文件

```shell
rename oldfile newfile
```

#### 退出 SFTP 会话

```shell
exit
```

#### 批量 SFTP 命令

新建一个文本文件放置 `sftp` 命令

```shell
put file1.txt
get file2.txt
```

使用 `-b` 选项执行文本文件

```shell
sftp -b sftp_batch.txt user@hostname
```

### SFTP 会话操作示例

```shell
$ sftp user@192.168.1.100
user@192.168.1.100's password: ********
sftp> ls
file1.txt  file2.txt  directory/
sftp> cd directory
sftp> get file3.txt
Fetching /directory/file3.txt to file3.txt
sftp> put newfile.txt
Uploading newfile.txt to /directory/newfile.txt
sftp> exit
```