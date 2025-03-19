### 简介

`syslog` 是 `Linux` 和类 `Unix` 系统中用于记录系统消息的标准协议。它允许应用程序、守护进程和内核将日志消息发送到集中式日志管理系统。

### Syslog 组件

* `Syslog` 守护进程：`syslogd` 或 `rsyslogd`，收集并管理日志消息

* 日志设施：日志类别（例如，auth、daemon、mail、local0-local7）

* 日志级别(严重级别)：定义消息的优先级（例如信息、警告、错误）

* 日志文件位置：默认储存位置：`/var/log/syslog`，`/var/log/messages`，`/var/log/auth.log`

### Syslog消息格式

典型的系统日志消息具有以下结构：

```shell
<priority> timestamp hostname application_name [PID]: message
```

**示例**

```shell
Mar 18 12:34:56 myserver sshd[12345]: Failed password for user root from 192.168.1.1 port 22 ssh2
```

### 常见的 Syslog 工具

* auth / authpriv：身份验证日志

* cron：Cron 作业日志

* daemon：系统守护进程日志

* kern：内核日志

* mail：邮件服务器日志

* syslog：内部系统日志消息

* user：用户应用程序日志

* local0 - local7：应用程序的自定义日志

### Syslog 严重性级别

|  级别   |  名称   |  描述   |
| --- | --- | --- |
|  0   |  emerg   |  系统无法使用   |
|  1   |  alert   |  需要立即采取行动   |
|  2   |  crit   |  危急情况   |
|  3   |  err   |  错误情况   |
|  4   |  warning   |  警告消息   |
|  5   |  notice   |  正常但重要的事件   |
|  6   |  info   |  信息性消息   |
|  7   |  debug   |  调试消息   |

### 示例用法

#### 查看Syslog消息

```shell
cat /var/log/syslog

或

tail -f /var/log/syslog
```

#### 发送自定义系统日志消息

```shell
logger -p local0.info "This is a test log message"

# 将日志存储在 /var/log/syslog 中（或按照 /etc/rsyslog.conf 中的配置）
```

#### 过滤特定日志

* 查看认证日志

```shell
cat /var/log/auth.log
```

* 查看内核日志

```shell
dmesg | tail -20
```

* 查看启动日志

```shell
journalctl -b
```

#### 配置 Syslog（rsyslog）

修改 `Syslog` 规则

编辑 `/etc/rsyslog.conf` 或 `/etc/rsyslog.d/*.conf`

```shell
authpriv.*    /var/log/auth.log
*.info;mail.none;authpriv.none;cron.none /var/log/syslog
```

重启日志服务

```shell
sudo systemctl restart rsyslog
```

#### 监控 SSH 登录尝试

* 查看失败的登录尝试

```shell
grep "Failed password" /var/log/auth.log
```

* 查看登录成功

```shell
grep "Accepted password" /var/log/auth.log
```

* 检查系统错误

```shell
grep "error" /var/log/syslog
```

#### 集中日志记录

* 将日志发送到远程系统日志服务器（`192.168.1.100`）

编辑 `/etc/rsyslog.conf` 并添加：

```shell
*.* @192.168.1.100:514
```

重启 `rsyslog`

```shell
sudo systemctl restart rsyslog
```

