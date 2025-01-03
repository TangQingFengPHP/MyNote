## 一、命令介绍

lsof是一个功能强大的实用程序，可用于基于Linux和unix的系统，字面意思是“打开文件列表“。

其主要功能是检索由不同运行进程打开的各种类型文件的详细信息。这些文件可以是普通文件、目录、块文件、网络套接字、命名管道等。

## 二、常用选项

```shell
-a：表示其他选项之间为”与“的关系；
-c<进程名>：列出指定进程所打开的文件；
-g：列出指定GID号进程详情；
-d<文件号>：列出占用该文件号的进程；
+d<目录>：列出目录下被打开的文件；
+D<目录>：递归列出目录下被打开的文件；
-n<目录>：列出使用NFS的文件；
-i<条件>：列出符合条件的进程（协议、:端口、 @ip ）
-p<进程号>：列出指定进程号所打开的文件；
-u：列出指定UID号或用户名的进程详情；
-h或-?：显示帮助信息；
-v：显示版本信息
-t：只显示进程id
-r<time-interval>：重复执行，直到它接收到来自用户的中断/终止信号
+r<time-interval>：重复模式将在其输出没有打开文件时立即结束
```

## 三、安装方法

* 在CentOS / RHEL / Fedora中

```shell
yum -y install lsof
```

* 在CentOS / RHEL8中

```shell
dnf install lsof
```

* 在Ubuntu / Debian中

```shell
apt install lsof
```
## 四、使用实例

* 列出所有打开的文件
  
```shell
lsof | less

此处使用 | less 传给less来分页输出
```
  
输出内容示例如下：  

```shell
COMMAND  PID  TID  USER   FD  TYPE  DEVICE  SIZE/OFF    NODE NAME
systemd    1       root  cwd   DIR   253,0       224      64 /
systemd    1       root  rtd   DIR   253,0       224      64 /
systemd    1       root  txt   REG   253,0   1632776  308905 /usr/lib/systemd/systemd
systemd    1       root  mem   REG   253,0     20064   16063 /usr/lib64/libuuid.so.1.3.0
systemd    1       root  mem   REG   253,0    265576  186547 /usr/lib64/libblkid.so.1.1.0
systemd    1       root  mem   REG   253,0     90248   16051 /usr/lib64/libz.so.1.2.7
systemd    1       root  mem   REG   253,0    157424   16059 /usr/lib64/liblzma.so.5.2.2
systemd    1       root  mem   REG   253,0     23968   59696 /usr/lib64/libcap-ng.so.0.0.0
systemd    1       root  mem   REG   253,0     19896   59686 /usr/lib64/libattr.so.1.1.0
systemd    1       root  mem   REG   253,0     19248   15679 /usr/lib64/libdl-2.17.so
systemd    1       root  mem   REG   253,0    402384   16039 /usr/lib64/libpcre.so.1.2.0
systemd    1       root  mem   REG   253,0   2156272   15673 /usr/lib64/libc-2.17.so
systemd    1       root  mem   REG   253,0    142144   15699 /usr/lib64/libpthread-2.17.so
systemd    1       root  mem   REG   253,0     88720      84 /usr/lib64/libgcc_s-4.8.5-20150702.so.1
systemd    1       root  mem   REG   253,0     43712   15703 /usr/lib64/librt-2.17.so
systemd    1       root  mem   REG   253,0    277808  229793 /usr/lib64/libmount.so.1.1.0
systemd    1       root  mem   REG   253,0     91800   76005 /usr/lib64/libkmod.so.2.2.10
systemd    1       root  mem   REG   253,0    127184   59698 /usr/lib64/libaudit.so.1.0.0
systemd    1       root  mem   REG   253,0     61680  229827 /usr/lib64/libpam.so.0.83.1
systemd    1       root  mem   REG   253,0     20048   59690 /usr/lib64/libcap.so.2.22
systemd    1       root  mem   REG   253,0    155744   16048 /usr/lib64/libselinux.so.1
```

> lsof输出各列信息的解释如下：

```
COMMAND：进程名称
PID：进程id
PPID：进程父id
USER：进程所有者
PGID：进程所属组
FD：文件描述符，应用程序通过它识别该文件
```

> 文件描述符类型列表：

```shell
cwd：当前目录
txt：程序文本，如应用程序二进制文件或共享库
Lnn：库引用 library references (AIX)
err：文件描述符信息错误
jld：jail目录（FreeBSD）
ltx：共享库文本（代码和数据）
mxx：十六进制内存映射类型编号xx
m86：DOS合并映射文件
mem：内存映射文件
mmap：内存映射设备
pd：父目录
rtd：根目录
tr：内核跟踪文件（OpenBSD）
v86：VP/ix 映射文件
0：标准输入
1：标准输出
2：标准错误
```

> 一般在标准输出、标准错误、标准输入后还跟着文件状态模式：

```
u：表示该文件被打开并处于读取/写入模式
r：表示该文件被打开并处于只读模式
w：表示该文件被打开并处于写入模式
空格：表示该文件的状态模式为 unknow，且没有锁定
-：表示该文件的状态模式为 unknow，且被锁定
```

> 文件类型：

```
DIR：目录
CHR：字符类型
BLK：块设备类型
UNIX：UNIX 域套接字
FIFO：先进先出 (FIFO) 队列
IPv4：IP套接字
DEVICE：指定磁盘的名称
SIZE：文件的大小
NODE：索引节点（文件在磁盘上的标识）
NAME：打开文件的确切名称
REG：常规文件
```

* 通过指定文件名来列出所有的进程

```shell
lsof /var/log/messages
```

* 通过用户名来列出打开的文件

```shell
lsof -u nginx
```

* 通过`^`符号来排除，其他选项也可以使用

```shell
lsof -u ^nginx
```

* 通过`kill`来快速杀死指定用户的进程

```shell
kill -9 `lsof -t -u nginx`
或kill -9 'lsof -t -u nginx'
或kill -9 $(lsof -t -u nginx)

说明：先用lsof -t -u nginx 来列出nginx用户所有打开的进程，通过-t只输出进程id，再用kill -9来杀死
```

* 只输出进程id

```shell
lsof -t
```

* 多个选项后面接参数，则为”或“的逻辑

```shell
lsof -u nginx -c bash

说明：以上输出用户为nginx或进程名为bash打开的文件
```

* 使用`-a`把选项变成”且“的关系

```shell
lsof -u nginx -c bash -a
```

* 通过进程名列出打开的文件
  
```shell
lsof -c ssh
```

* 通过进程id列出打开的文件

```shell
lsof -p 663
```

* 列出打开的文件包含目录（递归）

```shell
lsof +D /var/log
```

* 列出打开的文件包含目录（不递归）

```shell
lsof +d /var/log
```

* 重复执行模式

```shell
lsof -c bash -r3

说明：每三秒执行一次

输出示例如下：
COMMAND  PID    USER  FD   TYPE DEVICE  SIZE/OFF     NODE NAME
bash    1425 ftpuser mem    REG  253,0 106172832 50548523 /usr/lib/locale/locale-archive
=======
COMMAND  PID    USER  FD   TYPE DEVICE  SIZE/OFF     NODE NAME
bash    1425 ftpuser mem    REG  253,0 106172832 50548523 /usr/lib/locale/locale-archive
=======
COMMAND  PID    USER  FD   TYPE DEVICE  SIZE/OFF     NODE NAME
bash    1425 ftpuser mem    REG  253,0 106172832 50548523 /usr/lib/locale/locale-archive
=======
```

* 列出使用网络协议打开的文件

```shell
lsof -i

示例：
COMMAND  PID         USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
chronyd  639       chrony    5u  IPv4  14333      0t0  UDP localhost:323
chronyd  639       chrony    6u  IPv6  14334      0t0  UDP localhost:323
sshd     997         root    3u  IPv4  17330      0t0  TCP *:ssh (LISTEN)
sshd     997         root    4u  IPv6  17339      0t0  TCP *:ssh (LISTEN)
master  1229         root   13u  IPv4  18129      0t0  TCP localhost:smtp (LISTEN)
master  1229         root   14u  IPv6  18130      0t0  TCP localhost:smtp (LISTEN)
sshd    1235         root    3u  IPv4  18318      0t0  TCP centos7vm:ssh->192.168.1.61:23566 (ESTABLISHED)
sshd    1239 abhisheknair    3u  IPv4  18318      0t0  TCP centos7vm:ssh->192.168.1.61:2356 (ESTABLISHED)
```

* 通过指定进程id来列出所有打开的网络连接

```shell
lsof -i -a -p 997
```

* 通过指定进程名称来列出所有打开的网络连接

```shell
lsof -i -a -c ssh
```

* 通过指定网络协议类型来过滤输出

```shell
lsof -i tcp

lsof -i udp
```

* 通过指定端口来过滤输出

```shell
lsof -i :22
```

* 通过ipv4/ipv6来过滤输出

```shell
lsof -i4

lsof -i6
```

* 查找已经被删除，但是未释放进程锁的文件

```shell
lsof / | grep deleted

说明：从根目录下查找，把结果传递给`grep`来搜索已经被删除的文件，处于已删除，未释放进程锁的文件则会带有`deleted`的标识
```

* `-c`选项支持正则表达式

```shell
lsof -c /ab[cd]/
```

* 指定当前的进程id且组合文件描述符

```shell
lsof -a -p $$ -d0,1,2

说明：`-p $$` 表示指定当前的进程id，-d0,1,2用逗号隔开指定多个文件描述符
```

* `-i`选项语法：[46][protocol][@hostname|hostaddr][:service|port]

```shell
变形使用如下：
lsof -i 4（指定ipv4）
lsof -i 6（指定ipv6）
lsof -i tcp（指定tcp）
lsof -i udp（指定udp）
lsof -i tcp:22（指定tcp且22端口）
lsof -i @127.0.0.1:22（指定IP地址及端口）
lsof -i tcp:1-1024（指定tcp及端口范围）
lsof -i :mdns（指定服务名称）
```

* 抑制输出内核内容输出

```shell
lsof -b | less
```

* 打印终端文件

```shell
lsof /dev/tty*
```

* 查找正在等待连接的端口

```shell
lsof -i -sTCP:LISTEN
或者：lsof -i | grep -i LISTEN
```

* 查找已经建立连接的连接

```shell
lsof -i -sTCP:ESTABLISHED
或者：lsof -i | grep -i ESTABLISHED 
```

## 五、lsof源码

![lsof源码](/images/lsof-image.png)

## 六、官方文档

![lsof官方文档](/images/lsof-image-1.png)

## 七、man pages

![lsof man pages](/images/lsof-image-2.png)