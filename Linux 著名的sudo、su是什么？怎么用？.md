## 一、su

### 什么是su？

su命令（简称是：`substitute` 或者 `switch user` ）用于切换到另一个用户，没有指定用户名，则默认情况下将以root用户登录。

为了向后兼容，su默认不改变当前目录，只设置环境变量 `HOME` 和 `SHELL` (如果目标用户不是根用户，则加上 `USER` 和    `LOGNAME`)。

### 常用选项

* `-c, --command=[command]` ：指定执行的命令，然后恢复到原来的用户。

* `-, -l, --login` ：将shell作为登录shell启动，其环境类似于实际登录。
    ```
    1. 清除所有的环境变量，除了 `TERM` 和通过 --whitelist-environment选项指定的变量。
    2. 初始化环境变量：HOME, SHELL, USER, LOGNAM, PATH。
    3. 变更目标用户的家目录。
    4. 设置shell的第一个参数，即argv[0]为 - ，使shell成为登录shell。
    ```

* `-m, -p, --preserve-environment` ：保留整个环境变量，例如，不会更新 `HOME`, `SHELL`, `USER`, `LOGNAME`，此选项与 `--login` 是互斥的，不要同时使用。

* `-s, --shell=[shell]` ：切换用户后指定 `shell` 而不是使用默认 `shell`，`shell` 使用的顺序如下：
    ```
    1. 通过 `--shell` 指定的优先级最高。
    2. 如果 `--preserve-environment` 选项指定了，且设置了 `SHELL` 环境变量，则使用此shell。
    3. 目标用户的passwd条目中列出的shell
    4. 以上都没有，则默认使用 `/bin/sh`
    ```

* `-w, --whitelist-environment=[list]` ：环境变量白名单，即如果指定了 `--login` 时，不会重置白名单中指定的环境变量，但忽略 `HOME`、`SHELL`、`USER`、`LOGNAME` 和  `PATH`，`list` 格式通过逗号分隔，

* `-h, --help` ：打印帮助信息。

* `-V, --version` ：打印版本信息。

### 使用实例

* 切换到普通用户

```shell
su - phoenix
或
su phoneix
```

* 切换到root用户

```shell
su -
或
su
```

* 切换用户时指定命令

```shell
su -c ls

su -c 'ls -l /home/username' phoenix
切换到普通用户并指定命令，命令指定了选项和参数则用引号引上。
```

* 切换的时候指定shell

```shell
su -s /usr/bin/zsh
```

* 保留环境

```shell
su -p phoenix
```

* 对于像 `Ubuntu` 没有root密码，可使用如下方式切到root

```shell
sudo su -
```
 
### su源码

![alt text](/images/su-image.png)

### man pages

![alt text](/images/su-image-1.png)


## 二、sudo

### 什么是sudo？

sudo简称Super User Do，它允许非root用户运行通常需要超级用户权限的其他Linux命令。

### 获得root权限的方式

* 直接使用 `ssh` root登录到主机

```shell
ssh root@[server_domain_or_ip]
```

* 使用 `su` 切换到root用户

```shell
su -
```

* 使用 `sudo` 临时获取root权限来执行需要root权限的命令，此时不会产出一个新的shell。

```shell
sudo [command]
```

### 什么是sudoers？

`sudo` 的配置文件即为：`sudoers`，位置在：`/etc/sudoers`

`sudoers` 文件指示系统如何处理 `sudo` 命令(每个 `sudo` 用户可以做什么)。

### 什么是/etc/sudoers.d？

`/etc/sudoers.d` 是 `/etc/sudoers` 同级配置文件目录，一般情况不建议直接修改 `/etc/sudoers` 而是在 `/etc/sudoers.d` 目录下面新建自定义配置文件，配置规则与 `/etc/sudoers` 相同，此中任何没有以 `~` 结尾的，且不包含 `.` 的文件会视作正确的配置文件，`sudo` 会读取所有配置文件追加到 `sudo` 配置中。

### 什么是Visudo？

由于 `/etc/sudoers` 任何语法错误将可能会引起系统崩溃的风险，而使用 `visudo` 会对配置文件作语法检查，防止配置错误阻塞 `sudo` 操作。

`visudo` 默认会使用 `vi` 作为文本编辑器，也可以配置 `visudo` 使用的编辑器。

> 在Ubuntu上配置

```shell
sudo update-alternatives --config editor
```

```shell
Output
There are 4 choices for the alternative editor (providing /usr/bin/editor).

  Selection    Path                Priority   Status
------------------------------------------------------------
* 0            /bin/nano            40        auto mode
  1            /bin/ed             -100       manual mode
  2            /bin/nano            40        manual mode
  3            /usr/bin/vim.basic   30        manual mode
  4            /usr/bin/vim.tiny    10        manual mode

Press <enter> to keep the current choice[*], or type selection number:

通过编号来选择合适的编辑器
```

> 在CentOS上配置

```shell
export EDITOR=`which [编辑器名称]`
```

```shell
. ~/.zshrc
或 source ~/.zshrc

加载生效
```

### 怎么修改sudoers文件？

使用 `sudo visudo` 会打开 `/etc/sudoers` 文件

`sudoers` 权限行解释：

```shell
root ALL=(ALL:ALL) ALL

%admin ALL=(ALL) ALL

#includedir /etc/sudoers.d
```

* root表示此规则是给root用户使用的。

* 第一个 `ALL` 表示此规则可应用所有的主机。

* 第二个 `ALL` 表示root用户可以以所有用户身份执行命令。

* 第三个 `ALL` 表示root用户可以以所有用户组身份执行命令。

* 第四个 `ALL` 表示root用户可以执行所有命令。

* `%admin`，以 `%` 开头是组名，表示只要用户属于admin组，则可以有以上指定的所有权限。

* 正常情况以 `#` 号开头的被视作注释，但此处 `#includedir` 被解析为引入文件的指令。

### 怎么授予普通用户sudo权限？

最简单的方式是把用户加入超级权限组

* 例如在 `Ubuntu` 上，使用 `sudo` 组作为超级权限组，则可以把普通用户加入 `sudo` 组。

```shell
sudo usermod -aG sudo [username]

或使用 `gpasswd` 命令

sudo gpasswd -a [username] sudo
```

* 在 `CentOS` 上，通常是使用 `wheel` 组作为超级权限组。

```shell
%wheel ALL=(ALL) ALL
```

### 怎么自定义sudoers的规则？

除了使用单个用户或用户组指定一行规则，还可以使用一种称之为别名的方式来分组指定。

* 用户别名

```shell
User_Alias FULLTIMERS = albert, ronald, ann

此处指定了一个用户别名 FULLTIMERS，里面包括三个用户名，分别用逗号隔开，表示里面每一个用户都应用此规则。应用示例如下：

FULLTIMERS ALL=(ALL) ALL
```

* 所有者身份别名

```shell
Runas_Alias OP = root, operator

此处指定了一个所有者身份别名，OP，里面包括三个用户身份，分别用逗号隔开，表示运行命令后能用OP里面任一身份。应用示例如下：

[username/group] ALL=(OP) ALL
```

* 主机别名

```shell
Host_Alias PRODSERVERS = master, mail, www, ns

此处指定了一个主机别名 PRODSERVERS，里面包含四个主机名，分别用逗号隔开，表示运行命令能应用与任一主机。应用示例如下：

[username/group] PRODSERVERS=(ALL) ALL
```

* 执行的命令别名

```shell
Cmnd_Alias POWER = /sbin/shutdown, /sbin/halt

此处指定的命令组别名，多个命令用逗号隔开，表示能应用命令组中的任一命令。应用示例如下：

[username/group] ALL=(ALL) POWER
```

* 希望允许用户以root权限执行命令而无需输入密码

```shell
[username/group] ALL = NOPASSWD: [command] 

如：GROUPONE ALL = NOPASSWD: /usr/bin/updatedb
```

* 同时指定无需密码的命令和需要密码的命令

```shell
[username/group] ALL = NOPASSWD: [command1], PASSWD: [command2]

如：GROUPTWO ALL = NOPASSWD: /usr/bin/updatedb, PASSWD: /bin/kill
```

* 通过 `NOEXEC` 限制用户不能执行指定的命令

```shell
[username/group] ALL = NOEXEC: /usr/bin/less
```

* 取反操作，即除某某之外的意思

> 示例一

```shell
jane ALL = /usr/bin/passwd [A-z]*, !/usr/bin/passwd root

以上表示jane可以修改除root之外的任何人的密码
```

> 示例二

```shell
jen	ALL, !PRODSERVERS = VIEWSHADOW

以上表示jen可以在除PRODSERVERS之外的所有机器上运行VIEWSHADOW命令
```

### sudo常用的选项

* 指定用户的身份执行命令，需要在配置文件设定好的

```shell
sudo -u [username] [command]

sudo -g [groupname] [command]
```

* 修改 `sudo` 密码有效期

> `sudo` 密码有效期默认是5分钟，通过以下配置可设置有效期

```shell
Timeout_Spec = [time]

时间格式：超时可以以天、小时、分钟和秒的组合形式指定，并以不区分大小写的单字母后缀表示时间单位。例如，7天8小时30分10秒的超时将写入'7d8h30m10s'。如果指定的数字没有单位，则假定为秒。天、分、小时或秒中的任何一个都可以省略。顺序必须从最大单位到最小单位，一个单位不能指定多次。
```

* 延长（验证、刷新有效期） `sudo` 密码有效期

```shell
sudo -v
```

* 立即让 `sudo` 密码过期，终止当前用户的特权

```shell
sudo -k
```

* 列出当前用户 `sudo` 配置的权限

```shell
sudo -l
```

* 重复上一条命令

```shell
应用场景：当执行需要 `sudo` 的命令时，忘记输入了 `sudo` 前缀，此处只需 sudo !! 即可
```

* 指定重复之前的第几条命令

```shell
sudo !6

6是第几条命令
```

* 一个有趣的配置

```shell
在配置文件中添加如下行
Defaults insults
```

当密码输错之后，会输出如下信息：

```shell
Output
[sudo] password for demo:    # enter an incorrect password here to see the results
Your mind just hasn't been the same since the electro-shock, has it?
[sudo] password for demo:
My mind is going. I can feel it.

sudo 会侮辱用户(假笑)：电击之后你的思维就不一样了，是吗?
```

* 打印版本号

```shell
sudo -V
```

* 打印帮助信息

```shell
sudo -h 或 -help
```

* 在后台运行命令

```shell
sudo -b [command]
```

* 非交互式运行 `sudo`，不询问密码

```shell
sudo -n [command]
```

* 指定运行的shell

```shell
sudo -s [command]

如果设置了shell环境变量，-s选项将运行shell指定的shell，或者运行文件passwd中指定的shell。
```

* 设置家目录

```shell
sudo -H [command]

-H选项将HOME环境变量设置为目标用户的主目录(默认为root)，如passwd中指定的。默认情况下，sudo不修改HOME。
```

* 停止解析命令行参数

```shell
sudo -- [command]
```

* 在一行运行多个命令

```shell
sudo ls; whoami; hostname

多个命令用分号隔开
```

### sudo官网

![alt text](/images/sudo-image.png)


看 Sudo Manual、Sudoers Manual、Visudo Manual即可
![alt text](/images/sudo-image-1.png)
