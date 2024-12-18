### 简介

`Tinyproxy` 是一款轻量级 `HTTP` 代理服务器，使用最少的资源，非常适合硬件有限的系统。尽管体积小，但它可以处理大量流量，而不会出现明显的性能问题。旨在处理简单的代理任务。它通常用于路由网络流量以保护隐私、缓存或访问受限资源。

它的设计初衷是快速而小巧，是嵌入式部署等用例的理想解决方案。

`Tinyproxy`占用空间小，并且只需要很少的系统资源。使用 `glibc` 时，内存占用大约为2 MB， CPU负载随着同时连接的数量线性增加（取决于连接的速度）。因此，`Tinyproxy` 可以在较旧的机器上运行，也可以在基于 `Linux` 的宽带路由器等网络设备上运行，而不会对性能产生任何明显影响。

### 安装

#### Debian/Ubuntu:

```shell
sudo apt update
sudo apt install tinyproxy
```

#### CentOS/RHEL/Fedora:

```shell
sudo yum install tinyproxy
sudo dnf install tinyproxy
```

#### MacOS

```shell
brew install tinyproxy
```

#### 从 `github` 拉取源码后手动编译 

```shell
./autogen.sh
./configure
make
make install
```

编译选项

* `--enable-debug`：启用完整的调试支持
* `--enable-xtinyproxy`：编译对 `XTinyproxy` 标头的支持
* `--enable-filter`：允许 `Tinyproxy` 过滤掉某些域名和 `URL`
* `--enable-upstream`：启用上游代理支持
* `--enable-transparent`：允许将 `Tinyproxy` 用作透明代理守护程序
* `--enable-reverse`：启用反向代理支持
* `--with-stathost=HOST`：设置统计主机的默认名称

### 启动运行

#### 启动 `tinyproxy`

```shell
systemctl start tinyproxy
```

#### 设置开机自启动

```shell
systemctl enable tinyproxy
```

#### 查看运行状态

```shell
systemctl status tinyproxy
```

### 如何配置 `tinyproxy`？

`tinyproxy` 配置文件位于 `/etc/tinyproxy/tinyproxy.conf`

#### 设置 `tinyproxy` 监听传入连接的端口（默认值：8888）

```shell
Port 8888
```

#### 设置 `tinyproxy` 绑定到的网络接口（例如，`localhost` 仅供本地使用）：

```shell
Listen 127.0.0.1:8888
```

#### 指定 `tinyproxy` 应将传入连接绑定到哪个本地网络接口

```shell
# for IPv4
Bind 192.168.1.100

# for IPv6
Bind 2001:db8::1

# 绑定到本机
Bind 127.0.0.1
```

#### `Port`、`Listen`、`Bind` 比较

* `Port` 仅绑定端口
* `Bind` 仅绑定网络接口或IP地址
* `Listen` 可以同时绑定IP地址和端口，一步到位
* `Listen` 的优先级最高，会覆盖 `Port` 指定的端口

#### 指定哪些 `IP` 地址可以使用代理

```shell
Allow 192.168.1.0/24
```

#### 设置调试的日志级别

可用的级别有：`Critical`、`Error`、`Warning`、`Notice`、`Connect`、`Info`

```shell
LogLevel Info
```

#### 启用基本身份验证来限制访问

客户端在设置代理连接时需要输入用户名和密码

```shell
BasicAuth username password
```

#### 修改代理发送的标头

```shell
AddHeader X-Proxy-Name "Tinyproxy"
```

#### 通过创建过滤器列表来阻止指定的域名或URL

```shell
# 配置过滤器文件的位置
Filter "/etc/tinyproxy/filter"
```

在 `/etc/tinyproxy/filter` 文件配置示例：

可以使用基本正则表达式

```shell
facebook.com
example.com

# 精确过滤 cnn.com
 ^cnn\.com$

# 过滤 cnn.com 的所有子域名，但不过滤 cnn.com 本身
.*\.cnn.com$

# 过滤任何包含 cnn.com 的域名，例如
cnn\.com

# 过滤以 cnn.com 结尾的任何域名
cnn\.com$

# 过滤所有以 adserver 开头的域名
^adserver

```

#### 将 `tinyproxy` 作为透明代理运行

```shell
TransparentProxy On
```

使用 `iptables` 重定向流量

```shell
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 8888
```

#### `tinyproxy` 使用上游代理，将请求转发到另一个代理服务器上

```shell
Upstream http proxy.example.com:8080
Upstream https proxy.example.com:8443
```

#### 把 `tinyproxy` 作为反向代理服务器

```shell
ReversePath "/api" "http://backend-server.local/"
```

#### 配置 `tinyproy` 只作为反向代理服务器

> 关闭正常代理

```shell
ReverseOnly Yes
```

#### 使 `tinyproxy` 使用 `cookie` 来跟踪反向代理映射

```shell
ReverseMagic Yes
```

#### 设置 `tinyproxy` 的 `PID` 文件位置

```shell
PidFile "/var/run/tinyproxy/tinyproxy.pid"
```

#### 指定 `tinyproxy` 在关闭空闲连接之前应等待的时间。

```shell
Timeout 600
```

#### 限制同时连接到 `tinyproxy` 的客户端数量

```shell
MaxClients 100
```

#### 启用 `HTTPS` 连接支持

```shell
ConnectPort 443
```

#### 启用匿名模式来隐藏内部网络信息

```shell
Anonymous "headers"
```

#### 查看 `tinyproxy` 日志

```shell
tail -f /var/log/tinyproxy/tinyproxy.log
```

#### 使用 `debug` 模式启动 `tinyproxy`

```shell
sudo tinyproxy -d -c /etc/tinyproxy/tinyproxy.conf
```

#### 设置 `tinyproxy` 的日志文件位置

```shell
LogFile /var/log/tinyproxy.log
```

#### 添加一个包含客户端 `IP` 地址的标头 `X-Tinyproxy`

```shell
XTinyproxy Yes
```

#### 查看 `tinyproxy` 的版本

```shell
tinyproxy -v
```

### 资源

* [GitHub源码](https://github.com/tinyproxy/tinyproxy)
* [发行版资源](https://github.com/tinyproxy/tinyproxy/releases)
* [官网](https://tinyproxy.github.io)