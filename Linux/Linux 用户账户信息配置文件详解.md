### 一、`/etc/passwd` 文件存储用户的账户信息

在 `Linux` 系统中，用户账户信息存储在 `/etc/passwd` 文件中

这个文件里面每个用户一行数据，每行数据的字段使用 `:` 分割，

例如：`username:x:uid:gid:comment:home_directory:shell`。 

#### 1、`/etc/passwd` 文件字段解释 

* `username`：表示登录的用户名，如：`john`

* `x`：表示加密密码的占位符，实际上的密码存储在 `/etc/shadow` 文件里面

* `uid`：用户的 `uid`，每个用户都有唯一的标识符，如：`1000`

* `gid`：用户的组id(`group ID`)，定义在 `/etc/group` 文件中的用户组，如：`1000`

* `comment`：这里 `comment` 只是示例，此字段用来表示用户额外的信息，如备注、全名、联系方式等

* `home_directory`：表示用户的家目录，如：`/home/john`

* `shell`：表示用户使用的登录 `shell`，指定默认的命令行解释器，如：`/bin/bash`

示例：

`john:x:1000:1000:John Doe:/home/john:/bin/bash`

#### 2、特殊的例子

* `root` 用户

```shell
root:x:0:0:root:/root:/bin/bash

# root用户的uid和gid都为0，家目录在/root
```

* 系统账户

```shell
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin

# 系统用户的shell一般是：/usr/sbin/nologin或/bin/false，用来阻止登录的。
```

### 二、`/etc/shadow` 文件存储用户的加密密码

`/etc/shadow` 存储用户账户的加密密码和其他有用的信息，每个用户一行数据，每行数据的字段用 `:` 分割。

例如：

`username:password:last_change:min_days:max_days:warn_days:inactive_days:expire_date:reserved`


#### 1、`/etc/shadow` 文件字段解释

* `username`：表示用户名

* `password`：表示加密过的密码，如：`$6$abcd1234...`

* `last_change`：自1970年1月1日上次更改密码以来的天数

* `min_days`：更改密码所需的最少天数，0表示能够在任何时候更改密码

* `max_days`：必须更改密码的最长天数，99999表示密码永久有效

* `warn_days`：密码过期前多少天警告用户。

* `inactive_days`：表示密码过期后多少天账户会被锁定。

* `expire_date`：帐户被禁用的日期（自1970年1月1日以来的天数）。

* `reserved`：预留字段，将来可能需要使用，一般是空。

示例：

```shell
john:$6$abcd1234$abcd5678/abcdef...:19345:7:90:7:30:20000:
```

#### 2、密码Hash类别

* `$1$`：表示使用 `MD5` 哈希算法

* `$2y$`：`Blowfish` (河豚)算法，`bcrypt` 的变体

* `$5$`：`SHA-256` 哈希算法

* `$6$`：`SHA-512` 哈希算法

#### 3、密码字段中特殊值的含义

* `$+hash`：表示哈希密码

* `!`：表示密码已被锁定，用户不能登录系统，即使密码输入正确的情况下也是如此。

* `*`：表示账户已被禁用，与 `!` 相似，通常用于系统账户，如：`nobody`, `bin`, `daemon`

* `(empty)`：留空表示登录不需要密码。


