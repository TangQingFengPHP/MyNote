### 简介

`mosquito` 是一个流行的 `Eclipse` 开源的 `MQTT`（消息队列遥测传输）代理。它轻巧，易于安装，适用于小型或大型 `IoT`（物联网）应用程序。

### 安装

```shell
# For Debian/Ubuntu-based
sudo apt install mosquitto mosquitto-clients

# Required for Mosquitto on RHEL/CentOS
sudo yum install epel-release  
sudo yum install mosquitto mosquitto-clients

# 使用dnf安装
sudo dnf install mosquitto mosquitto-clients
```

### 启动 `Mosquitto`

```shell
sudo systemctl enable mosquitto
sudo systemctl start mosquitto
```

### 常用选项

* `-h`：指定主机IP地址或域名

* `-t`：指定主题的名称

* `-m`：发送的消息

* `-p`：指定端口

* `-u`：授权账户的用户名

* `-P`：授权账户的密码

### 常用操作

#### 客户端连接使用IP地址的方式

```shell
mosquitto_pub -h localhost -t "home/temperature" -m "22.5"

mosquitto_sub -h localhost -t "home/temperature"
```

#### 客户端连接使用域名的方式

域名只需要解析到IP就行

```shell
mosquitto_pub -h your-domain.com -p 1883 -t "test/topic" -m "Hello MQTT"

mosquitto_sub -h your-domain.com -p 1883 -t "test/topic"
```

#### 客户端连接使用SSL的方式

* 安装 `certbot`

```shell
# Ubuntu/Debian
sudo apt update
sudo apt install certbot

# CentOS/RHEL
sudo yum install epel-release
sudo yum install certbot
```

* 使用 `Certbot` 生成免费证书

使用 `Standalone` 模式，没有 `Web` 服务时

```shell
sudo certbot certonly --standalone -d mqtt.example.com
```

使用 `Webroot` 模式，有 `Web` 服务时

```shell
sudo certbot certonly --webroot -w /var/www/html -d mqtt.example.com
```

* 配置 `Mosquitto` 使用证书

在 `mosquitto.conf` 配置文件中添加： 

```shell
# 设置 TLS/SSL 监听端口（默认 8883）
listener 8883

# 根证书链文件
cafile /etc/letsencrypt/live/mqtt.example.com/fullchain.pem

# 域名证书文件
certfile /etc/letsencrypt/live/mqtt.example.com/cert.pem

# 私钥文件
keyfile /etc/letsencrypt/live/mqtt.example.com/privkey.pem
```

* 重启 `Mosquitto` 服务

```shell
sudo systemctl restart mosquitto
```

* 连接配置了 `SSL` 的 `Mosquitto` 

```shell
# --capath 指定包含可信任 CA 根证书的目录路径
mosquitto_pub -h mqtt.example.com -p 8883 --capath /etc/ssl/certs -t "test" -m "Hello, MQTT over TLS!"

# --cafile 用于自定义的单个 CA 文件，例如测试环境或自签名证书

mosquitto_pub -h mqtt.example.com -p 8883 --cafile /path/to/custom-ca.pem -t "test" -m "Hello, MQTT over TLS!"
```

* 设置自动续期证书

```shell
sudo certbot renew --dry-run
```

* 使用计划任务自动续期

```shell
0 3 * * * certbot renew --quiet && systemctl restart mosquitto
```

#### 设置用户名密码

* 创建或更新 `Mosquitto` 密码文件

```shell
sudo mosquitto_passwd -c /etc/mosquitto/password_file [username]

# username 为自定义设置的用户名
# 输入完以上命令后会提示输入密码
```

* 修改 `Mosquitto` 配置文件

在 `mosquitto.conf` 中启用密码认证

```ini
# 表示禁止匿名用户登录，需要用户名密码
allow_anonymous false

# 指定密码文件的位置
password_file /etc/mosquitto/password_file
```

其 `password_file` 可以自定义名称，里面内容与 `/etc/shadow` 里面存储密码格式类型。

* 重启 `Mosquitto` 服务

```shell
sudo systemctl restart mosquitto
```

* 客户端连接时提供用户名密码

```shell
mosquitto_pub -h your-domain.com -p 1883 -u "username" -P "password" -t "test/topic" -m "Hello MQTT"

mosquitto_sub -h your-domain.com -p 1883 -u "username" -P "password" -t "test/topic"
```

#### 配置 `ACL` 规则

`ACL`（Access Control List，访问控制列表） 是用来定义客户端对主题的访问权限的规则配置文件。

通过 `ACL`，可以精确控制哪些客户端可以订阅或发布哪些主题

* 创建 `ACL` 文件

```shell
sudo vim /etc/mosquitto/acl
```

* 配置 `ACL` 规则

`ACL` 规则语法

```shell
# 允许匿名客户端的默认权限（如果启用匿名访问）
topic read #

# 为某个用户指定权限
user <username>
topic <access> <topic>
```

示例配置：

```shell
# 允许匿名用户只读访问所有主题
topic read #

# 限制特定用户访问
user client1
topic write sensors/temperature
topic read sensors/temperature

# client2 只能读取 home/light
user client2
topic read home/light

# client3 可以读写所有主题
user client3
topic read #
topic write #

# read 允许订阅
# write 允许发布
# #号匹配当前及其子级所有主题
# +号匹配当前级别的单个主题 
```


