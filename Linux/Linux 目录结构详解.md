### 简介

`Linux` 遵循文件系统层次结构标准 (`FHS`)，该标准以结构化方式组织文件和目录。了解此结构有助于系统管理、故障排除和开发。

### Linux 根目录 /

`Linux` 中的所有文件和目录都源自根目录 (`/`)，它是层次结构中的顶级目录。

```shell
/
├── bin/
├── boot/
├── dev/
├── etc/
├── home/
├── lib/
├—— /lost+found
├── media/
├── mnt/
├── opt/
├── proc/
├── root/
├── run/
├── sbin/
├── srv/
├── sys/
├── tmp/
├── usr/
└── var/
```

#### /bin 目录：基本系统二进制文件

包含 `ls、cp、mv、cat` 等基本命令

#### /boot 目录：引导加载程序文件

存储 `Linux` 内核（ `vmlinuz` ）、`grub` 引导加载程序文件

#### /dev 目录：设备文件

代表硬件的虚拟文件（例如，磁盘的 `/dev/sda、/dev/null`）

#### /etc 目录：配置文件

系统范围的配置文件，如 `/etc/passwd、/etc/ssh/sshd_config`

#### /home 目录：用户家目录

每个用户都有一个个人目录 (`/home/user`)

#### /lib,/lib64 目录：共享库

存储程序所需的 `.so`（共享对象）文件

#### /lost+found 目录：恢复的文件

如果文件系统崩溃，将在下次启动时执行文件系统检查。发现的任何损坏的文件都将放在 `lost+found` 目录中，因此可以尝试恢复尽可能多的数据

#### /media 目录：自动挂载可移动媒体

`USB` 驱动器、`CD` 都安装在这里

#### /mnt 目录：临时挂载点

用于手动挂载分区

#### /opt 目录：可选软件

第三方应用程序（例如 `Google Chrome、VirtualBox`）安装在这里

#### /proc 目录：进程信息

用于进程数据的虚拟文件系统（`/proc/cpuinfo、/proc/meminfo`）

#### /root 目录：根用户主目录

超级用户的主目录（`/home` 是普通用户的主目录）

#### /run 目录：运行时流程数据

临时文件，如进程 `ID（/run/systemd/）`

#### /sbin 目录：系统二进制文件

基本管理命令（`fdisk、shutdown、iptables`）

#### /srv 目录：服务器数据

`Web` 或 `FTP` 服务器文件（`/srv/www/、/srv/ftp/`）

#### /sys 目录：系统信息

与内核相关的文件（网络设备为 `/sys/class/net/`）

#### /tmp 目录：临时文件

重启时清除，存储临时文件（`/tmp/session.log`）

#### /usr 目录：用户实用程序和应用程序

包含用户程序的 `bin、lib、share 例如：（/usr/bin/vim）`

#### /var 目录：变量数据

日志文件、邮件和数据库等（`/var/log/、/var/www/`）

### 常用使用场景

#### 检查系统日志

```shell
cat /var/log/syslog
```

#### 查看安装的程序

```shell
ls /usr/bin
```

#### 检查CPU信息

```shell
cat /proc/cpuinfo
```

#### 列出已挂载的磁盘

```shell
lsblk
```

#### 创建临时文件

```shell
touch /tmp/myfile.txt
```

#### 查找配置文件

```shell
ls /etc
```

#### 挂载一个USB驱动

```shell
mount /dev/sdb1 /mnt
```