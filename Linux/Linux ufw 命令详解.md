### 简介

`UFW(Uncomplicated Firewall)` 简单防火墙是一款基于 `iptables` 构建的、用于管理防火墙规则的用户友好型工具。它简化了在 `Linux` 系统上配置防火墙的过程。

### 安装

* 在 `Ubuntu/Debian` 上安装

```shell
sudo apt update
sudo apt install ufw
```

* 在 `CentOS/Red Hat` 上安装

```shell
sudo yum install epel-release
sudo yum install ufw
```

### 示例用法

#### 查看当前 `ufw` 的状态

```shell
sudo ufw status
```

#### 查看 `ufw` 的详细状态信息

```shell
sudo ufw status verbose
```

#### 启用 `ufw`

```shell
sudo ufw enable
```

#### 禁用 `ufw`

```shell
sudo ufw disable
```

#### 允许指定的流量

```shell
sudo ufw allow <port/service>
```
 
#### 允许 `http` 80端口

```shell
sudo ufw allow 80
```

#### 允许 `https` 443端口

```shell
sudo ufw allow 443
```

#### 允许 `ssh` 服务

```shell
sudo ufw allow ssh
```

#### 拒绝流量

```shell
sudo ufw deny <port/service>
```

#### 拒绝 `FTP` 21端口

```shell
sudo ufw deny 21
```

#### 允许来自特定 `IP` 的流量

```shell
sudo ufw allow from <IP>

# 例如：
sudo ufw allow from 192.168.1.100
```

#### 允许从特定 `IP` 到特定端口的流量

```shell
sudo ufw allow from <IP> to any port <port>

# 例如：
sudo ufw allow from 192.168.1.100 to any port 22
```

#### 允许子网的流量

```shell
sudo ufw allow from <subnet>

# 例如：
sudo ufw allow from 192.168.1.0/24
```

#### 移除指定的规则

```shell
sudo ufw delete allow <port/service>

# 例如：
sudo ufw delete allow 80
```

#### 按编号删除规则

先用 `sudo ufw status numbered` 查看编号

再执行：

```shell
sudo ufw delete <rule_number>

# 例如：
sudo ufw delete 3
```

#### 启用 `ufw` 日志

日志记录在 `/var/log/ufw.log` 文件

```shell
sudo ufw logging on
```

#### 设置日志级别

可用的日志级别有：`low`, `medium`, `high`, `full`

```shell
sudo ufw logging medium
```

#### 禁用 `ufw` 日志

```shell
sudo ufw logging off
```

#### 限制 `ssh` 连接尝试次数

```shell
sudo ufw limit ssh
```

#### 向发送者发送拒绝响应

```shell
sudo ufw reject <port/service>

# 例如：
sudo ufw reject 8081
```

#### 启用 `IPv6` 支持

编辑 `ufw` 配置文件：`/etc/ufw/ufw.conf`，添加如下行：

```ini
IPV6=yes
```

#### 重置 `ufw` 为默认设置

将禁用 `ufw`，删除所有规则并重置为默认配置。

```shell
sudo ufw reset
```

#### 列出可用的应用程序配置文件

```shell
sudo ufw app list
```

#### 允许一个应用的流量

```shell
sudo ufw allow <app_name>

# 例如：
sudo ufw allow nginx
```

#### 显示应用流量配置详情

```shell
sudo ufw app info <app_name>

# 例如：
sudo ufw app info nginx
```

#### 允许 `MySql` 3306端口

```shell
sudo ufw allow from 192.168.1.0/24 to any port 3306
```

#### 允许范围端口

```shell
sudo ufw allow 6000:6007/tcp
sudo ufw allow 6000:6007/udp
```





