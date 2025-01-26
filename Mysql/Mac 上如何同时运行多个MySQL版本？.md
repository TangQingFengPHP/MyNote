### 前言

由于我准备下载最新的 `Redmine 6.0` 版本到本地运行，所需的 `MySql` 版本要求为：`8.0-8.1`，且我打算保留本地已经在运行的 `5.7` 版本，不做干涉。

### 使用 `Homebrew` 下载 `MySql8.0` 版本

```shell
# 先搜索mysql版本
brew search mysql

# 搜索到就安装
brew install mysql@8.0
```

### 配置及初始化数据目录

正常安装好 `MySql` 后，会自动创建 `MySql` 的配置文件 `my.cnf` 及 数据目录，由于已经存在 `5.7` 版本，`Homebrew` 不会自动创建了。

`5.7` 版本的 `my.cnf` 配置文件如下：

```shell
# Default Homebrew MySQL server config
[mysqld]
# Only allow connections from localhost
bind-address = 127.0.0.1
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

这时直接启动 `brew services start mysql@8.0` 已经报失败了。如果看不到错误原因，可以使用 `/opt/homebrew/opt/mysql@8.0/bin/mysqld --debug` 来启动，这时已经有详细的错误原因了：

```shell
2025-01-23T10:00:42.390289Z 0 [ERROR] [MY-000077] [Server] /opt/homebrew/opt/mysql@8.0/bin/mysqld: Error while setting value 'STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION' to 'sql_mode'.
2025-01-23T10:00:42.390320Z 0 [ERROR] [MY-010119] [Server] Aborting
2025-01-23T10:00:42.390550Z 0 [Note] [MY-010120] [Server] Binlog end
```

错误表明：于 `5.7` 配置文件中的 `sql_mode` 设置不兼容或某些选项在当前 `8.0` 版本中不支持导致的。

为什么会报这个错呢？因为 `8.0` 也是使用默认的 `5.7 my.cnf` 配置文件。

那好，接下来手动新建一个 `8.0` 配置文件，复制 `/opt/homebrew/etc/my.cnf` 为 `/opt/homebrew/etc/my80.cnf`

`my80.cnf` 配置如下：

```shell
# Homebrew MySQL8.0 server config
[mysqld]

port=3307

# 数据目录
datadir=/opt/homebrew/var/mysql80new
socket=/opt/homebrew/var/mysql80new/mysql.sock

# 错误日志
log_error=/opt/homebrew/var/mysql80new/error.log

# 默认字符集
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci

# Only allow connections from localhost
bind-address = 127.0.0.1

# SQL模式
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO
```

再次指定新的配置文件启动 `8.0`

```shell
# 使用 mysqld_safe 启动 MySQL，它会更安全地启动 MySQL，并记录更多调试信息
/opt/homebrew/opt/mysql@8.0/bin/mysqld_safe --defaults-file=/opt/homebrew/etc/my80.cnf
```

`tail -f /opt/homebrew/var/mysql80new/error.log` 查看日志果然又报错了

```shell
2025-01-23T23:36:45.925467Z 0 [System] [MY-010116] [Server] /opt/homebrew/opt/mysql@8.0/bin/mysqld (mysqld 8.0.39) starting as process 74975
2025-01-23T23:36:45.933475Z 0 [Warning] [MY-010159] [Server] Setting lower_case_table_names=2 because file system for /opt/homebrew/var/mysql80/ is case insensitive
2025-01-23T23:36:45.946573Z 1 [ERROR] [MY-011011] [Server] Failed to find valid data directory.
2025-01-23T23:36:45.946727Z 0 [ERROR] [MY-010020] [Server] Data Dictionary initialization failed.
2025-01-23T23:36:45.946740Z 0 [ERROR] [MY-010119] [Server] Aborting
2025-01-23T23:36:45.947616Z 0 [System] [MY-010910] [Server] /opt/homebrew/opt/mysql@8.0/bin/mysqld: Shutdown complete (mysqld 8.0.39)  Homebrew.
2025-01-23T23:36:45.6NZ mysqld_safe mysqld from pid file /opt/homebrew/var/mysql80/panfengdeMacBook-Pro.local.pid ended
```

错误原因是：Failed to find valid data directory，Data Dictionary initialization failed，表明：MySQL 没有找到有效的数据目录，启动时尝试初始化数据字典时失败了。

接下来就在启动的时候执行初始化：

```shell
/opt/homebrew/opt/mysql@8.0/bin/mysqld_safe --initialize --user=$(whoami) --datadir=/opt/homebrew/var/mysql80new --defaults-file=/opt/homebrew/etc/my80.cnf
```

又报错了：

```shell
2025-01-24T01:16:27.584805Z 0 [ERROR] [MY-000077] [Server] /opt/homebrew/opt/mysql@8.0/bin/mysqld: Error while setting value 'STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION' to 'sql_mode'.
2025-01-24T01:16:27.585040Z 0 [ERROR] [MY-013236] [Server] The designated data directory /opt/homebrew/var/mysql is unusable. You can remove all files that the server added to it.
2025-01-24T01:16:27.585281Z 0 [ERROR] [MY-010119] [Server] Aborting
2025-01-24T01:16:27.586993Z 0 [Note] [MY-010120] [Server] Binlog end
```

很奇怪明明指定了配置文件及数据目录，怎么还是跑到 `5.7` 版本的位置去了？

经过查阅 `MySql 8.0` 文档得知：`--defaults-file` 选项必须要放在第一个选项才能生效。

好了，重新调整选项顺序执行：

```shell
/opt/homebrew/opt/mysql@8.0/bin/mysqld_safe --defaults-file=/opt/homebrew/etc/my80.cnf --initialize --user=$(whoami) --datadir=/opt/homebrew/var/mysql80new 
```

执行成功了，但是立马又退出了：

```shell
2025-01-24T03:04:10.172558Z 0 [System] [MY-013169] [Server] /opt/homebrew/opt/mysql@8.0/bin/mysqld (mysqld 8.0.39) initializing of server in progress as process 5416
2025-01-24T03:04:10.179400Z 0 [Warning] [MY-010159] [Server] Setting lower_case_table_names=2 because file system for /opt/homebrew/var/mysql80new/ is case insensitive
2025-01-24T03:04:10.192972Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2025-01-24T03:04:10.326354Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2025-01-24T03:04:10.749031Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: ;jO?5puqH?(3
2025-01-24T03:04:11.082422Z 0 [System] [MY-013172] [Server] Received SHUTDOWN from user <via user signal>. Shutting down mysqld (Version: 8.0.39).
```

以上日志可以看到 `MySql` 已经启动成功了，且生成了临时 `root` 密码，但最后接受到 `SIGTERM` 信号从而中断了 `MySql`。

加个 `&` 符号放在后台执行看看：

```shell
/opt/homebrew/opt/mysql@8.0/bin/mysqld --defaults-file=/opt/homebrew/etc/my80.cnf --datadir=/opt/homebrew/var/mysql80new --user=$(whoami) &
```

果然执行成功不会退出了。

### 尝试登录下 `8.0` 的 `MySql`

```
/opt/homebrew/opt/mysql@8.0/bin/mysql -uroot -p
```

输入临时密码：`;jO?5puqH?(3` ，哦豁，登不上去，难道是要用引号转义？试一下：

```shell
/opt/homebrew/opt/mysql@8.0/bin/mysql -uroot -p';jO?5puqH?(3'
8.0版本已经不能直接把密码输在后面了，已经报错了：
[Warning] Using a password on the command line interface can be insecure.

或者 `/opt/homebrew/opt/mysql@8.0/bin/mysql -uroot -p 然后非明文输入，还是报密码不对。
```

试一下用 `navicat` 连下吧：

![alt text](/images/MySQL/MySQL多个版本image.png)
![alt text](/images/MySQL/MySQL多个版本image-1.png)

`navicat` 连成功了！

**思考一下**

再试一次用 `5.7` 的密码来连，这次指定了端口

```shell
 /opt/homebrew/opt/mysql@8.0/bin/mysql -uroot --port=3307 -p
```

居然连到 `5.7` 的版本上去了

```shell
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 26
Server version: 5.7.40 Homebrew

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql
```

**为什么呢？**

原因：`MySql` 客户端在连接时，会优先选择基于 `socket` 文件（在 `UNIX` 系统上）。当不指定 `socket` 文件时，`MySql` 客户端通常会默认使用系统中配置的标准值来进行连接。

`navicat` 为什么能正确连接呢？猜测 `navicat` 应该是读取并解析 `MySql` 配置文件中的 `socket` 参数。

当设置连接主机为 `localhost` 时，`MySql` 客户端库会直接使用 `socket`  文件来连接，而不是 `TCP/IP`。这是 `MySql` 客户端的默认行为，`Navicat` 通过调用 `MySql` 客户端库继承了这一特性。

**如何解决？**

连接的时候明确指定 `socket`

```shell
/opt/homebrew/opt/mysql@8.0/bin/mysql -uroot -p --port=3307 --socket=/opt/homebrew/var/mysql80new/mysql.sock
```

### 尝试使用 `Homebrew` 来正常启动 `MySql8.0` 版本

`Homebrew` 使用 `LaunchAgents` 来管理服务（如 `MySql`）。可以手动修改其 `.plist` 文件来指定 `--defaults-file` 或其他参数。

找到 `mysql@8.0` 的 `plist` 文件路径

```shell
ls ~/Library/LaunchAgents | grep mysql
```

输出：

```shell
homebrew.mxcl.mysql@5.7.plist
```

发现没有 `8.0` 版本的 `plist`，那么就复制从 `5.7` 复制一份到 `8.0`

```shell
cp ~/Library/LaunchAgents/homebrew.mxcl.mysql@5.7.plist ~/Library/LaunchAgents/homebrew.mxcl.mysql@8.0.plist
```

编辑如下：

```shell
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>KeepAlive</key>
	<true/>
	<key>Label</key>
	<string>homebrew.mxcl.mysql@8.0</string>
	<key>LimitLoadToSessionType</key>
	<array>
		<string>Aqua</string>
		<string>Background</string>
		<string>LoginWindow</string>
		<string>StandardIO</string>
		<string>System</string>
	</array>
	<key>ProgramArguments</key>
	<array>
		<string>/opt/homebrew/opt/mysql@8.0/bin/mysqld</string>
		<string>--datadir=/opt/homebrew/var/mysql80new</string>
	</array>
	<key>RunAtLoad</key>
	<true/>
	<key>WorkingDirectory</key>
	<string>/opt/homebrew/var/mysql80new</string>
</dict>
</plist>
```

使用 `launchctl` 启动下看看

```shell
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.mysql@8.0.plist
```

查看下进程是否启动成功了：

```shell
ps aux | grep mysqld
```

![alt text](/images/MySQL/MySQL多个版本image-3.png)

截图看可以看到能正常启动成功了，那我用 `brew services start mysql@8.0` 行不行？

先停止再启动

```shell
brew services stop mysql@8.0
brew services start mysql@8.0
```

发现问题了，停止后 `8.0 的 plist` 被清空了。

原因：`Homebrew` 使用 `brew services` 来管理服务时，会自动控制相关的 `plist` 文件的生成和加载。如果停止了 `MySql` 服务 (`brew services stop mysql@8.0`)，`Homebrew` 并不会删除 `plist` 文件，当你再次启动服务时（`brew services start mysql@8.0`），它会重新生成并覆盖 `plist` 文件内容，因此，如果手动修改了 `plist` 文件，例如指定了自定义的 `datadir` 或 `cnf` 文件路径，这些修改会在重新启动时被覆盖，还是使用的 `5.7` 的配置。

为了防止意外使用 `brew` 来启动 `8.0`，复制一份 `8.0的plist` 到自定义的 `plist`

```shell
cp ~/Library/LaunchAgents/homebrew.mxcl.mysql@8.0.plist ~/plist/mysql8.0-custom.plist
```

使用 `launchctl` 启动 `8.0`

```shell
launchctl load ~/mysql8.0-custom.plist
```

### 总结

目前尝试的有两种方式启动 `MySql8.0`：

1. 直接使用 `mysqld` 命令启动：

```shell
/opt/homebrew/opt/mysql@8.0/bin/mysql -uroot -p --port=3307 --socket=/opt/homebrew/var/mysql80new/mysql.sock
```

2. 使用自定义的 `plist` 文件，使用 `launchctl` 启动

```shell
launchctl load ~/mysql8.0-custom.plist
```