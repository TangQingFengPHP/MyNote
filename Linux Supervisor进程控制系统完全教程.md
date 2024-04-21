## 一、简介

Supervisor是一个进程控制系统，它使用户能够监视和控制类unix操作系统进程。它通过提供基于配置或事件启动、停止和重新启动进程的机制，帮助管理应该在系统中连续运行的进程。对于需要控制和监视Linux或其他类unix操作系统上多个进程的状态的开发人员和系统管理员来说，Supervisor特别有用。

监督程序通常作为后台守护进程运行，并充当负责管理多个进程的集中实体。它可用于管理各种类型的进程，例如web服务器、数据库服务器、任务队列和自定义应用程序。

在UNIX上，通常很难获得进程的准确的上/下状态。Pidfiles经常说谎。Supervisord将进程作为子进程启动，因此它总是知道其子进程的真正up/down状态，并且可以方便地查询这些数据。

> Supervisor特性：

* 简单：Supervisor是通过一个简单的ini风格的配置文件来配置的，它很容易学习。它提供了许多每个进程的选项，如重新启动失败的进程和自动日志轮换。

* 集中管理：Supervisor提供了一个启动、停止和监视进程的位置。进程可以单独控制，也可以分组控制。也可以配置Supervisor提供本地或远程命令行和web界面。

* 高效：Supervisor通过fork/exec启动它的子进程，子进程不被daemon化。当进程终止时，操作系统会立即向Supervisor发送信号，不像某些解决方案依赖麻烦的PID文件和定期轮询来重新启动失败的进程。

* 可扩展性：Supervisor有一个简单的事件通知协议，用任何语言编写的程序都可以用它来监视它，还有一个用于控制的XML-RPC接口。它还使用扩展点构建，Python开发人员可以利用这些扩展点。

* 兼容性：除了Windows, Supervisor几乎可以在所有设备上工作。它在Linux、Mac OS X、Solaris和FreeBSD上进行了测试和支持。它完全用Python编写，因此安装不需要C编译器。

> Supervisor的主要功能包括：

* 进程控制：Supervisor可以启动、停止和重新启动进程。它确保指定的进程正在运行，如果它们意外崩溃或终止，它可以自动重新启动它们。

* 监视：Supervisor不断地监视被管理流程的状态。它可以跟踪进程是否正在运行、是否已退出或是否遇到任何错误。这种监视功能允许Supervisor根据进程状态采取适当的操作。

* 日志记录和输出捕获：Supervisor捕获子进程的标准输出和标注错误流。这样就可以存储和分析日志，帮助进行故障排除和调试。

* 配置管理：Supervisor通常提供配置文件或配置管理系统来定义要管理的进程、启动参数和其他相关设置。这允许从一个中心位置轻松地管理和配置多个进程。

* Web界面和命令行：Supervisor提供基于web的用户界面和命令行界面，用于管理和监控进程。

* 分组：它允许对可以一起控制的进程进行分组，这在管理包含多个部分的微服务或应用程序时非常有用。

* 事件通知：当被管理进程的状态发生变化时，Supervisor可以发出事件，这些事件可用于与其他监视或警报系统集成。

> Supervisor组件

* supervisord：Supervisor的服务器部分被命名为supervisord。它负责在自己的调用时启动子程序，响应来自客户端的命令，重新启动崩溃或退出的子进程，记录其子进程的标准输出和标准错误输出，以及生成和处理与子进程生命周期中的的“事件”。

服务器进程使用配置文件。它通常位于/etc/supervisord.conf文件中。这个配置文件是一个“Windows-INI”风格的配置文件。通过适当的文件系统权限来保证这个文件的安全是很重要的，因为它可能包含未加密的用户名和密码。

* supervisorctl：Supervisor的命令行客户端部分命名为supervisorctl。它为supervisord提供的特性提供了一个类似shell的接口。通过supervisorctl，用户可以连接到不同的supervisord进程(一次一个)，获取被控制的子进程的状态，停止和启动子进程，以及获取正在运行的supervisord进程的列表。

* Web Server：启用internet socket，则可以通过浏览器访问Web用户界面。启用配置文件的[inet_http_server]部分后，访问服务器URL(例如http://localhost:9001/)，通过web界面查看和控制进程状态。

* XML-RPC Interface：提供web UI的同一个HTTP服务器同时提供了一个XML-RPC接口，该接口可用于查询和控制管理程序及其运行的程序。

## 二、安装方法

* 通过pip安装

```shell
pip install supervisor
```

* Linux Debian系

```shell
sudo apt-get install supervisor
```

* Linux CentoOS系

```shell
sudo yum install -y supervisor
```

* MacOS

```shell
brew install supervisor
```

* Windows不支持

Supervisor的发布包的一个特点是，它们通常已经集成到发布的服务管理基础设施中，例如，允许在系统启动时自动启动Supervisor，一般都自动配置到了systemd服务中。

如果没有自动创建配置文件，则使用如下命令创建：

```shell
echo_supervisord_conf > /etc/supervisord.conf

echo_supervisord_conf 会打印supervisord的配置信息，然后写到/etc/supervisord.conf中
```

## 三、运行supervisord

```shell
supervisord
```
supervisord默认会作为守护进程来运行，运行的日志可以在supervisor.log中查看，也可以通过 -n 选项来指定supervisord在前台运行，一般用于查看debug信息。

> supervisord的常用的命令行选项有：

* `-c [FILE], --configuration=[FILE]`：指supervisord使用的配置文件。

* `-n, --nodaemon`：在前台运行。

* `-s, --silent`：静音模式，没有输出定向到标准输出。

* `-h, --help`：打印帮助信息。

* `-d [PATH], --directory=[PATH]`：当supervisord作为守护进程运行时，在守护进程之前切换到该目录。

* `-l FILE, --logfile=FILE`：指定supervisord的日志文件路径。

* `-y BYTES, --logfile_maxbytes=BYTES`：轮换发生前 Supervisord 活动日志文件的最大大小。该值是后缀相乘的，例如“1”是一个字节，“1MB”是 1 MB，“1GB”是 1 GB。

* `-z NUM, --logfile_backups=NUM`：要保留的 Supervisord 活动日志的备份副本数量。每个日志文件的大小为 logfile_maxbytes。

* `-e LEVEL, --loglevel=LEVEL`：日志记录级别，分别有：trace, debug, info, warn, error, critical。

* `-j FILE, --pidfile=FILE`：Supervisord 应将其 pid 文件写入的文件名。

* `-i STRING, --identifier=STRING`：此Supervisor实例的各种客户端 UI 公开的任意字符串标识符。

* `-a NUM, --minfds=NUM`：在成功启动之前，supervisord 进程必须可用的文件描述符的最小数量。

* `-v, --version`：打印版本号

* `--minprocs=NUM`：在成功启动之前，supervisord 进程必须可用的操作系统进程槽的最小数量。

## 四、Supervisorctl

> 常用的命令行选项

* `-c, --configuration`：指定supervisord.conf配置文件路径

* `-h, --help`：打印帮助信息

* `-i, --interactive`：进行交互式shell

* `-s, --serverurl URL`：Supervisord 服务器正在侦听的 URL（默认“http://localhost:9001”）。

* `-u, --username`：用于向服务器进行身份验证的用户名：用于向服务器进行身份验证的用户名。

* `-p, --password`：用于与服务器进行身份验证的密码。

> 常用的操作

* `help`：打印可用的操作列表

* `help [action]`：打印指定操作的用法

* `update`：重新加载所有配置并根据需要添加/删除，并重新启动受影响的程序。

* `update [group]`：重新加载指定组名的进程，并重新启动受影响的程序。

* `clear [name]`：清理进程日志文件。

* `clear [name] [name]`：清理多个进程日志文件，用空格隔开。

* `clear all`：清理所有进程日志文件。

* `pid`：获取supervisord的PID，也可以使用 `ps -ef | grep supervisord` 获取。

* `pid [name]`：获取指定子进程的PID。

* `pid all`：获取所有子进程的PID。

* `reread`：重新加载配置文件，不会重新启动，也就是改动的还不会生效。

* `restart [name]`：指定进程重新启动，注意：重新启动不会重新读取配置文件。

* `restart [gname]:*`：重新启动进程组中的所有进程，不会重新读取配置文件。

* `restart [name] [name]`：一次重新启动多个进程，用空格分隔，不会重新读取配置文件。

* `restart all`：重新启动所有进程，不会重新读取配置文件。

* `start [name]`：启动指定的进程。

* `start [gname]:*`：启动指定组的所有进程。

* `start [name] [name]`：一次启动多个进程，用空格隔开。

* `start all`：启动所有进程。

* `status`：获取所有进程状态信息。

* `status [name]`：获取指定进程的状态信息。

* `status [name] [name]`：一次获取多个进程的状态信息，用空格隔开。

* `stop [name]`：停止指定的进程。

* `stop [gname]:*`：停止指定的进程组的所有进程。

* `stop [name] [name]`：一次停止多个进程，用空格分隔。

* `stop all`：停止所有进程。

* `tail -f [name] [stdout|stderr] (default stdout)`：查看指定进程的日志，在supervisorctl内部调用tail -f。

一般情况下，添加或修改了配置文件，使用两个命令即可：supervisorctl reread、supervisorctl update。

> Supervisord自动启动脚本：

![alt text](/images/supervisor-image.png)

## 五、配置文件说明

Supervisor默认的配置文件名为：supervisord.conf，里面包含supervisord和supervisorctl的配置，可以通过在supervisord或supervisorctl后面接 -c 选项指定配置文件。

> supervisor搜索配置文件的默认顺序如下：
>
> $CWD表示当前目录

* ../etc/supervisord.conf

* ../supervisord.conf

* $CWD/supervisord.conf

* $CWD/etc/supervisord.conf

* /etc/supervisord.conf

* /etc/supervisor/supervisord.conf

### [unix_http_server]部分配置

其中应该插入侦听UNIX域套接字的HTTP服务器的配置参数。如果配置文件中没有[unix_http_server]部分，则不会启动UNIX域套接字HTTP服务器。

配置项有：

* `file`：socket文件路径，supervisor将在该套接字上监听HTTP/XML-RPC请求。supervisorctl使用XML-RPC通过该端口与supervisord通信。一般位于：/tmp/supervisor.sock。

* `chmod`：socket文件权限，默认值是：0700。

* `chown`：socket文件所有者，格式为：uid:gid。

* `username`：对此HTTP服务器进行身份验证所需的用户名，默认为空。

* `password`：对此HTTP服务器进行身份验证所需的密码，默认为空，可以是一个明文密码，也可以指定为SHA-1哈希，如果前缀是字符串{SHA}。例如，{SHA}82ab876d1387bfafe46cc1c8a2ef074eae50cb1d是密码thepassword的SHA存储版本，默认为空。

配置项示例：

```shell
[unix_http_server]
file = /tmp/supervisor.sock
chmod = 0777
chown= nobody:nogroup
username = user
password = 123
```

### [inet_http_server]部分配置

启用此配置，则可用通过Web UI 来访问Supervisor后台，简易的Web页面上可以做常用的supervisorctl的操作。

配置项有：

* `port`：指定访问的IP地址和端口，，格式如下：

```shell
port=127.0.0.1:9001

或

port=*:9001
```

* `username`：UI界面登录的用户名，默认为空

* `password`：UI界面登录的密码，默认为空，同上也可以是哈希密码。

配置项示例：

```shell
[inet_http_server]
port = 127.0.0.1:9001
username = user
password = 123
```

### [supervisord]部分配置

* `logfile`：Supervisord 进程的活动日志的路径，默认值：$CWD/supervisord.log。

* `logfile_maxbytes`：活动日志文件在轮换之前可能消耗的最大字节数，（可以在值中使用“KB”、“MB”和“GB”等后缀乘数）。将此值设置为 0 以指示日志大小不受限制，默认值是：50M。

* `logfile_backups`：由活动日志文件轮换产生的要保留的备份数量。如果设置为 0，则不会保留任何备份，默认值是：10。

* `loglevel`：日志级别，可用的值有：critical, error, warn, info, debug, trace, blather，默认值是：info。

* `pidfile`：Supervisord的PID文件路径，默认值是：$CWD/supervisord.pid。

* `umask`：Supervisord 进程的掩码，默认值是：022。 

* `nodaemon`：如果为 true，Supervisord 将在前台启动而不是守护进程，默认值是：false。

* `silent`：如果为 true 并且未守护进程，日志将不会定向到 stdout，默认值是：false。

* `minfds`：在Supervisord成功启动之前必须可用的文件描述符的最小数量，默认值是：1024。

* `minprocs`：在Supervisord成功启动之前必须可用的进程描述符的最小数量，默认值是：200。

* `user`：指示Supervisord在进行任何有意义的处理之前将用户切换到此UNIX 用户帐户。仅当Supervisord以root用户身份启动时才能切换用户。

* `directory`：当Supervisord守护进程时，切换到该目录。

* `strip_ansi`：从子日志文件中删除所有 ANSI 转义序列，默认值是：false。

* `environment`：键/值对的列表，格式为key="val"，KEY2="val2"，将放置在所有子进程的环境中。这并没有改变Supervisord本身的环境。包含非字母数字字符的值应该加引号(例如KEY="val:123"，KEY2="val,456")。否则，引号是可选的，但建议使用。要转义百分比字符，只需要再加一个百分号。(例如URI="/first%%20name")请注意，子进程将继承用于启动Supervisord的shell的环境变量，默认为空。

* `identifier`：该Supervisor进程的标识符字符串，由 RPC 接口使用，默认值是：supervisor。

配置项示例：

```shell
[supervisord]
logfile = /tmp/supervisord.log
logfile_maxbytes = 50MB
logfile_backups=10
loglevel = info
pidfile = /tmp/supervisord.pid
nodaemon = false
minfds = 1024
minprocs = 200
umask = 022
user = chrism
identifier = supervisor
directory = /tmp
nocleanup = true
childlogdir = /tmp
strip_ansi = false
environment = KEY1="value1",KEY2="value2"
```

### [supervisorctl]部分配置

* `serverurl`：用于访问 Supervisord 服务器的 URL，例如http://localhost:9001。对于 UNIX socket，使用 unix:///absolute/path/to/supervisor.sock。

* `username`：传递到 Supervisord 服务器以用于身份验证的用户名，默认值是：空。

* `password`：传递到 Supervisord 服务器以用于身份验证的密码，默认值是：空。

* `prompt`：用作supervisorctl提示符的字符串，默认值是：supervisor。

配置项示例：

```shell
[supervisorctl]
serverurl = unix:///tmp/supervisor.sock
username = chris
password = 123
prompt = mysupervisor
```

### [program:x]部分配置

配置文件必须包含一个或多个程序段，以便Supervisord知道它应该启动和控制哪些程序。报头值是复合值。它是单词“program”，后面直接跟一个冒号，然后是程序名。例如：头文件的值[program:foo]描述了一个名为“foo”的程序。该名称用于控制因此配置而创建的进程的客户端应用程序。创建没有名称的程序节是错误的。名称不能包含冒号字符或括号字符。名称值的字符串表达式是：%(program_name)s，可用于其他地方当做变量使用。

一个[program:x]段实际上代表了一个“同构进程组”(从3.0开始)。组的成员由配置中的numprocs和process_name参数组合定义。默认情况下，如果numprocs和process_name保持默认值不变，则由[program:x]表示的组将被命名为x，并且其中将包含一个名为x的进程。这提供了与旧版本的少量向后兼容性，旧版本不将程序段视为同构进程组定义。

但是，例如，如果有一个[program:foo]节，其numprocs为3,process_name表达式为%(program_name)s_%(process_num)02d，那么“foo”组将包含三个进程，分别命名为foo_00、foo_01和foo_02。这使得使用单个[program:x]段启动多个非常相似的进程成为可能。所有日志文件名、所有环境字符串和程序命令也可以包含类似的Python字符串表达式，以便向每个进程传递略有不同的参数。简单来说：如果只有一个进程则组名等于程序名，有多个进程，则组名后加上编号作为程序名。

* `command`

启动该程序时将运行的命令。该命令可以是绝对路径(例如/path/to/programname)或相对路径(例如../programname)。如果它是相对的，则将在supervisord的环境变量$PATH中搜索可执行文件。程序可以接受参数，例如/path/to/program foo bar。命令行可以使用双引号将带有空格的参数分组传递给程序，例如/path/to/program/name -p "foo bar"。请注意command的值可能包含Python字符串表达式，例如/path/to/programname——port=80%(process_num)02d可能在运行时展开为/path/to/programname——port=8000。字符串表达式根据包含关键字group_name、host_node_name、program_name、process_num、numprocs的字典求值。

如果该命令看起来像配置文件注释，则会被截断，例如command=bash -c 'foo;Bar '将被截断为command=bash -c 'foo。引号不会阻止这种行为，因为配置文件读取器不会像shell那样解析命令，所以不要加上分号。

* `process_name`：一个Python字符串表达式，用于组成此进程的Supervisor进程名。除非更改numprocs，否则通常不需要担心设置这个，默认值是：%(program_name)s，取的是program的名字。

* `numprocs`：Supervisor 将启动由 numprocs 指定的多个该程序的实例。注意，如果 numprocs > 1，则 process_name 表达式中必须包含 %(process_num)s （或任何其他包含 process_num 的有效 Python 字符串表达式），默认值是：1。

* `numprocs_start`：一个整数偏移量，用于计算 process_num 开始的编号，默认值是：0，例如改为1，则process_name显示如：foo_01、foo_02，那么就是从1开始编号。 

* `directory`：程序在执行之前切换到的目录。

* `priority`：程序在启动和关闭顺序中的相对优先级。较低的优先级表示程序首先启动并在启动时最后关闭，当在各种客户端中使用聚合命令时(例如“start all”/“stop all”)。较高的优先级表示最后启动并首先关闭的程序，默认值是：999。

* `autostart`：如果为true，则该程序将在supervisord启动时自动启动，默认值是：true。

* `startsecs`：启动后程序需要保持运行的总秒数，以认为启动成功(将进程从STARTING状态移动到RUNNING状态)。设置为0表示程序不需要在任何特定的时间内保持运行，默认值是：1。

* `startretries`：尝试启动的次数，超过此次数还是启动失败则把进程置为FATAL状态，默认值是：3。

* `autorestart`：指定程序在RUNNING状态下退出后是否自动重启。

可选的值有：false（表示不会自动重启），unexpected（意外退出的情况，必须是退出码不在exitcodes中列出的退出码才能触发），true（表示无条件重启，不考虑退出码）

默认值是：unexpected。

* `exitcodes`：与autorestart一起使用，预期的退出码列表，默认值是：0，多个退出码用逗号隔开，如：0,2。

* `stopsignal`：杀死进程的信号名称，可选的值有：TERM, HUP, INT, QUIT, KILL, USR1, USR2；默认值是：TERM。

* `stopwaitsecs`：在程序被发送停止信号后等待操作系统返回SIGCHLD到supervisord的秒数。如果在supervisord收到进程的SIGCHLD之前超过了这个秒数，那么supervisord将尝试使用最后的SIGKILL杀死进程，默认值是：10。

* `stopasgroup`：如果为true，则该标志会导致Supervisor向整个进程组发送停止信号，并隐式设置killasgroup为true，默认值是：false。

* `killasgroup`：如果为true，则当向程序发送SIGKILL 来终止它时，将其发送到整个进程组，默认值是：false。

* `user`：指示supervisord 使用此用户帐户作为运行程序的帐户。仅当supervisord以root用户身份运行时才能切换用户。如果supervisord无法切换到指定用户，程序将不会启动。

* `redirect_stderr`：如果为true，进程的标准错误重定向到标准输出，默认值是：false，相当于：2>&1。

* `stdout_logfile`：将进程的标准输出记录在这个文件中(如果redirect_stderr为true，也将标准输出记录在这个文件中)。如果stdout_logfile未设置或设置为AUTO，则管理程序将自动选择文件位置。如果设置为NONE, supervisord将不创建日志文件。AUTO日志文件及其备份将在重启时被删除，默认值是：AUTO。

启用轮换 (​​stdout_logfile_maxbytes) 时，两个进程不能共享单个日志文件 (stdout_logfile)。这将导致文件被损坏，多个进程不能同时写同一个文件。

如果stdout_logfile设置为特殊文件，如/dev/stdout，则必须通过设置stdout_logfile_maxbytes = 0来禁用日志轮转。

* `stdout_logfile_maxbytes`：stdout_logfile 在旋转之前可以消耗的最大字节数（可以在值中使用“KB”、“MB”和“GB”等后缀乘数）。将此值设置为 0 以指示日志大小不受限制，默认值是：50M。

* `stdout_logfile_backups`：标准输出日志文件保留的备份数量。如果设置为0，则不会保留任何备份，默认值是：10。

* `stdout_capture_maxbytes`：当进程处于“标准输出捕获模式” 时，写入捕获 FIFO 的最大字节数。应为整数（值中可以使用“KB”、“MB”和“GB”等后缀乘数）。如果该值为 0，则进程捕获模式将关闭，默认值是：0。

* `stdout_syslog`：如果为true，标准输出将带上进程名称标识发送到syslog，默认值是：false。

* `stderr_logfile`：标准错误写入的日志文件路径，默认值是：AUTO。

启用轮换 (​​stderr_logfile_maxbytes) 时，两个进程不能共享单个日志文件 (stderr_logfile)。这将导致文件被损坏，多个进程不能同时写同一个文件。

如果stderr_logfile设置为不可查找的特殊文件（例如 /dev/stderr），则必须通过设置 stderr_logfile_maxbytes = 0 来禁用日志轮转。

* `stderr_logfile_maxbytes`：stderr_logfile的日志文件轮换之前的最大字节数。接受与 stdout_logfile_maxbytes 相同的值类型，默认值是：50M。

* `stderr_logfile_backups`：标准错误日志文件保留的备份数量。如果设置为0，则不会保留任何备份，默认值是：10。

* `stderr_capture_maxbytes`：当进程处于“标准错误捕获模式” 时，写入捕获 FIFO 的最大字节数。应为整数（值中可以使用“KB”、“MB”和“GB”等后缀乘数）。如果该值为 0，则进程捕获模式将关闭，默认值是：0。

* `stderr_syslog`：如果为true，标准错误将带上进程名称标识发送到syslog，默认值是：false。

* `environment`：环境变量，键/值对的列表，格式为key ="val"，KEY2="val2"，将放置在子进程的环境中。

配置项示例：

```shell
[program:cat]
command=/bin/cat
process_name=%(program_name)s
numprocs=1
directory=/tmp
umask=022
priority=999
autostart=true
autorestart=unexpected
startsecs=10
startretries=3
exitcodes=0
stopsignal=TERM
stopwaitsecs=10
stopasgroup=false
killasgroup=false
user=chrism
redirect_stderr=false
stdout_logfile=/a/path
stdout_logfile_maxbytes=1MB
stdout_logfile_backups=10
stdout_capture_maxbytes=1MB
stdout_events_enabled=false
stderr_logfile=/a/path
stderr_logfile_maxbytes=1MB
stderr_logfile_backups=10
stderr_capture_maxbytes=1MB
stderr_events_enabled=false
environment=A="1",B="2"
serverurl=AUTO
```

### [include]部分配置

即在主配置文件中加载目录下的子配置文件

* `files`：可以为绝对路径，也可以为相对路径，可以指定多个路径，用空格隔开。

配置项示例：

```shell
[include]
files = /opt/homebrew/etc/supervisor.d/*.ini ../etc/supervisor.d/*.ini
```

### [group:x]部分配置

进程组的定义

要将程序放入一个组中，以便可以将它们视为一个单元，在配置文件中定义一个[group:x]节。组头值是一个复合值。它是单词“group”，后面直接跟一个冒号，然后是组名。头值[group:foo]描述了一个名为“foo”的组。该名称用于控制因此配置而创建的进程的客户端应用程序。创建没有名称的组节是错误的。名称不能包含冒号字符或括号字符。

对于 [group:x]，配置文件中的其他位置必须有一个或多个 [program:x] 部分，并且组必须在程序值中按名称引用它们。

* `programs`：以逗号分隔的程序名称列表。列出的程序成为该组的成员，例如：programs=program1,program2。

* `priority`：进程组的优先级，默认值是：999。

配置项示例：

```shell
[group:foo]
programs=bar,baz
priority=999
```

## 六、子进程

Supervisor的主要目的是基于其配置文件中的数据创建和管理进程。它通过创建子流程来实现这一点。由Supervisor生成的每个子进程在其整个生命周期内都由supervisord管理(supervisord是它创建的每个进程的父进程)。当子进程死亡时，通过SIGCHLD信号通知Supervisor，并执行相应的操作。

> 子进程状态：

* STOPPED：进程已经停止或从未启动。

* STARTING：进程正在启动中。

* RUNNING：进程正在运行中。

* BACKOFF：该进程进入STARTING 状态，但随后退出得太快（在 startsecs 中定义的时间之前），无法进入 RUNNING 状态。

* STOPPING：进程正在停止中。

* EXITED：进程从RUNNING状态退出，

* FATAL：进程无法启动成功。

* UNKNOWN：未知的进程状态，一般情况是supervisord自身的程序错误。

当自动重启进程处于BACKOFF状态时，该进程将被supervisord自动重启。它将在STARTING和BACKOFF状态之间切换，直到它明显无法启动，因为启动次数已经超过了最大值，此时它将转换到FATAL状态。

根据后续尝试的次数，重试所花费的时间会越来越长，每次增加一秒钟。

如果自动重启的进程最终处于FATAL状态，它将永远不会自动重启(必须手动从此状态重启)。

## 日志

活动日志由supervisord根据配置文件[supervisord]部分中的logfile_maxbytes和logfile_backups参数组合“旋转”。当活动日志达到logfile_maxbytes字节时，将当前日志文件移动到备份文件中，并创建一个新的活动日志文件。发生这种情况时，如果现有备份文件的数量大于或等于logfile_backups，则删除最旧的备份文件，并相应地重命名备份文件。如果要写入的文件名为“supervisord.log”，则当其超过logfile_maxbytes时，将关闭该文件并将其重命名为supervisord.log.1，如果supervisor.log.1, supervisord.log都存在，则将其重命名为supervisord.log.2, supervisord.log.3 等等。如果logfile_maxbytes为0，则不会轮换日志文件(因此不会进行备份)。如果logfile_backups为0，则不保留任何备份。

>  注意事项：路径不支持~，即家目录要写全称。

## 官方文档

![alt text](/images/supervisor-image-1.png)