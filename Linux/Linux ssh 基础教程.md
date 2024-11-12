### 简介：

SSH是一种安全协议，用作远程连接Linux服务器的主要手段。它通过生成远程shell提供基于文本的界面。连接之后，在本地终端中键入的所有命令都将发送到远程服务器并在那里执行，一般说ssh即openssh。

### 什么是OpenSSH？

OpenSSH（Open Secure Shell），是一套安全的网络工具，它在不安全的网络上提供加密通信。它主要用于安全远程登录、文件传输和其他安全网络服务。OpenSSH是由OpenBSD团队开发的一个开源项目，它实现了SSH （Secure Shell）协议，该协议通过加密数据和身份验证来帮助保护网络服务。

### OpenSSH 的主要功能和组件：

* 远程登录：允许用户安全的登录到远程系统，用于代替不加密、不安全的协议，如：Telnet、rlogin。

* 文件传输：SCP和SFTP。

* 端口隧道转发：一般用来做网络代理和内网穿透。

* 安全认证：支持密码、公私钥认证登录。

* 包含服务端和客户端：sshd是服务端守护进程，ssh是客户端程序，ssh-keygen是管理ssh秘钥的程序。

* ssh-agent：存储密钥密码的工具。

### SSH是如何使秘钥验证用户身份的？

要使用SSH密钥进行身份验证，用户必须在其本地计算机上拥有SSH密钥对。在远程服务器上，必须将公钥复制到用户家目录的~/.ssh/authorized_keys文件中。该文件包含一个公钥列表，每行一个，这些公钥被授权登录到这个帐户。

当客户端连接到主机，希望使用SSH密钥身份验证时，它将通知服务器这个意图，并告诉服务器使用哪个公钥。然后，服务器检查其authorized_keys文件中的公钥，生成一个随机字符串，并使用公钥对其进行加密。此加密消息只能使用关联的私钥解密。服务器将此加密消息发送给客户端，以测试它们是否确实拥有关联的私钥。

在收到此消息后，客户端将使用私钥对其进行解密，并将显示的随机字符串与先前协商的会话ID结合起来。然后生成该值的MD5散列并将其传回服务器。服务器已经拥有原始消息和会话ID，因此它可以比较由这些值生成的MD5散列，并确定客户端必须拥有私钥。

### 如何生成秘钥？

#### 生成2048位的秘钥对

秘钥加密算法包括：RSA、DSA、ECDSA，默认是RSA，使用如下命令生成密钥对

```shell
ssh-keygen
```

默认会存放在家目录下的.ssh目录，如：`~/.ssh/id_rsa`(表示私钥)、`~/.ssh/id_rsa.pub`(表示公钥)

可以输入秘钥的密码，也可以不输，直接回车到底。

#### 生成4096位的秘钥对

```shell
ssh-keygen -b 4096

b (bit)就表示指定位
```

#### 修改原秘钥的密码或置空

```shell
ssh-keygen -p

p (password)就表示要修改密码
```

### 如何把公钥复制到服务器上？

#### 通过ssh-copy-id命令

```shell
ssh-copy-id username@remote_host

此命令内部会执行ssh登录，提示输入服务器的密码，执行成功后，服务器对应的 ~/.ssh/authorized_keys 将会追加当前用户的公钥：~/.ssh/id_rsa.pub内容 到文件末尾
```

#### 通过管道的方式追加

```shell
cat ~/.ssh/id_rsa.pub | ssh username@remote_host "cat >> ~/.ssh/authorized_keys"

先用cat命令显示在终端，然后输出传给ssh，服务器的cat拿到输出追加到文件中。
```

#### 手动复制到服务器上

先登录到服务器，再把公钥粘贴到文件中

### 常用连接选项

#### 一般连接方式

```shell
ssh username@remote_host

用户名 + 主机名的方式
```

#### 连接的时候指定在服务器上执行的命令

```shell
ssh username@remote_host [command_to_run]

这种方式执行完命令后立马退出，不会登录到服务器
```

#### 连接的时候指定端口

```shell
ssh -p port_num username@remote_host

如：ssh -p 1234 root@127.0.0.1
```

#### 如何配置服务器别名的方式登录？

编辑 `~/.ssh/config` 文件，配置格式如下：

```shell
Host remote_alias
Hostname remote_host
Port port_num
User user_name
remote_alias表示主机别名指向remote_host

示例：
Host demo
Hostname demo.com
Port 1234
User root

配置完之后只需要输入，例如：ssh demo，即可登录到服务器
```

#### 如何避免每次输入秘钥密码？

如果在生成秘钥对的时候输入了密码，则默认每次登录服务器都需要输入密码，可以使用 `ssh-agent` 来处理。

```shell
首先启动ssh-agent，执行：eval $(ssh-agent)

接着使用ssh-add，输入秘钥密码
```

#### 在服务器上使用本地秘钥连接另一个服务器

使用场景一般是服务器上不想退出，直接在服务器上连接另一个服务器。

```shell
ssh -A username@remote_host

在服务器上连接另一个服务器时，会使用本地的秘钥来连接。
```

### 常用服务端配置选项

编辑 `/etc/ssh/sshd_config` 文件，修改如下位置为：

#### 禁用密码登录服务器

```shell
PasswordAuthentication no
```

#### 修改登录端口

```shell
#Port 22
Port 1234
```

#### 设置登录白名单

```shell
AllowUsers user1 user2
只有在允许的用户列表内才能登录

AllowUsers alice@192.168.1.10 bob@10.0.0.20
用户+主机限定

AllowGroups sudoers
可以指定用户组
```

#### 禁用root登录

```shell
PermitRootLogin no
```

### 客户端配置选项

编辑 `~/.ssh/config` 文件

#### 配置心跳保持连接避免超时

```shell
Host *
ServerAliveInterval 120
```

#### 禁用主机检查

当每次连接新服务器时，会提示是否确认连接

```shell
Host *
StrictHostKeyChecking no
UserKnownHostsFile /dev/null

首先禁用检查，并把known_hosts文件路径指向垃圾桶

也可以单独指定主机需要检查

Host testhost
HostName your_domain
StrictHostKeyChecking ask
UserKnownHostsFile ~/.ssh/known_hosts
```

#### 通过单个TCP连接多路复用SSH

SSH多路复用为多个SSH会话重用同一个TCP连接，减少TCP连接的浪费，加快速度

```shell
Host *

// 表示启用多路复用并自动重用现有连接。
ControlMaster auto

// 指定用于连接的套接字文件路径，下面两种命名都比较合适
// %r@%h:%p是用户名、主机名、端口的占位符，如：username@host:22
ControlPath ~/.ssh/multiplex/%r@%h:%p
ControlPath ~/.ssh/sockets/%r@%h:%p

// 在上一个会话结束后保持连接有效 10 分钟。在此期间，新会话可以重用该连接而无需重新进行身份验证。
ControlPersist 10m
```

配置完之后，在 `~/.ssh` 目录下创建 `multiplex` 或 `sockets` 目录。

当建立ssh连接后，ssh将自动在以上目录创建socket文件，同一主机的后续连接将通过socket文件重用现有的TCP连接。

在 `ControlPersist` 超时之后，连接将会关闭

如果想临时绕过连接重用机制，可使用下面的选项：

```shell
ssh -S none username@remote_host

-S none 表示指定socket为none
```

### SSH隧道的用法

#### 本地端口转发

通过ssh重定向本地端口到远程服务器端口，访问远程服务，就像它在本地运行一样。

通常用于通过绕过防火墙来建立隧道到限制较少的网络环境

另一个常见的用途是从远程位置访问“仅限本地主机”的web界面。

```shell
ssh -L [local_port]:[remote_host]:[remote_port] user@ssh_server
```

* `local_port`：本地端口

* `remote_host`：远程主机，一般为服务器本机：localhost

* `remote_port`：远程端口

* `user@ssh_server`：远程主机用户和ip地址

示例：

navicat等数据库连接工具是典型的利用ssh本地端口转发

```shell
ssh -L 3307:localhost:3306 user@remote_server
```

#### 远程端口转发

通过ssh重定向远程端口到本地端口，主要用于内网穿透

通常用于访问本地开发的web项目等

```shell
ssh -R [remote_port]:[local_host]:[local_port] user@ssh_server
```

示例：

转发远程服务器的8080端口到本地的80端口，即访问服务器的8080端口即为本地的80端口。

```shell
ssh -R 8080:localhost:80 user@remote_server
```

#### 动态端口转发

创建一个 SOCKS 代理，它将流量通过 SSH 服务器动态路由到任何目的地。

通常用于绕过网络限制或通过SSH服务器安全网页浏览，即网络代理。

```shell
ssh -D [local_port] user@ssh_server
```

示例：

```shell
ssh -D 1080 user@remote_server
```

#### 配置端口转发为后台进程

使用 `-f(Fork to Background)` 和 `-N(No Command)`

* `-f`：使SSH进程在身份验证后在后台运行，适用于运行SSH隧道或端口转发而无需保持终端会话打开。

* `-N`：阻止SSH执行远程命令或打开shell。

使用场景：

* 为长时间运行的任务或安全访问设置SSH隧道，而无需交互式SSH会话。

* 自动化或脚本化SSH隧道设置。

示例：

```shell
ssh -f -N -L 3307:localhost:3306 user@remote_server
```

### SSH转义码的使用

通常在ssh会话卡主、僵住的时候使用。

SSH转义码是一种特殊的序列，允许在不退出活动SSH连接的情况下控制该连接。

它们用于各种目的，例如挂起连接、打开新会话或终止会话。通过按一个特殊的转义字符（通常是~），后跟一个特定的命令来调用这些代码。

首先按回车键，然后再按转移码，有效的转义码如下：

* `~.`：断开ssh会话

* `~Ctrl-Z`：挂起ssh会话并返回到shell

* `~#`：列出激活的转发连接

* `~&`：将ssh会话放到后台执行

* `~?`：显示所有可用的转义命令

* `~?`：打开一个命令行接口或修改转发的端口

* `~R`：请求重新设置连接密钥（对于长时间会话有用）。

还可以 `-e` 选项自定义转义前缀指令，`ssh -e @ user@host`，此时前缀从 `~` 变为 `@`

### 资源

* [ssh官网](https://www.ssh.com/academy/ssh)

* [openssh源码](https://github.com/openssh/openssh-portable)