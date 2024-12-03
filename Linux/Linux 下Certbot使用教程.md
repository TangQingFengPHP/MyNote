### 简介

`Certbot` 是一个命令行工具，用于自动执行从 `Let’s Encrypt` 获取和更新`SSL/TLS` 证书的过程。它通过管理证书、配置 `Web` 服务器并自动更新证书来简化网站安全保护。

### 安装

* 在 `Ubuntu/Debian` 上

```shell
sudo apt update
sudo apt install certbot python3-certbot-nginx  # For Nginx
sudo apt install certbot python3-certbot-apache  # For Apache
```

* 在 `CentOS/RHEL` 上

```shell
sudo yum install epel-release
sudo yum install certbot python3-certbot-nginx  # For Nginx
sudo yum install certbot python3-certbot-apache  # For Apache
```

### 常用选项

* `--nginx`：获取并配置 Nginx 的证书。

* `--apache`：获取并配置 Apache 的证书

* `certonly`：仅获取证书，不执行安装

* `renew`：更新所有已安装的证书

* `delete`：删除证书

* `revoke --cert-path <path>`：吊销证书

### 证书文件说明

证书文件位置在：`/etc/letsencrypt/live/<domain>/`

* `fullchain.pem`：完整的证书链

* `privkey.pem`：私钥文件

* `cert.pem`：域名证书

* `chain.pem`：证书颁发机构链

### 示例用法

#### 使用 `Nginx` 自动获取并安装证书

```shell
sudo certbot --nginx

# 根据提示输入以下信息
# 输入邮箱用于过期提醒
# 同意服务条款
# 选择nginx配置的域名
```

#### 使用 `Apache` 自动获取并安装证书

```shell
sudo certbot --apache

# 与nginx类似
```

#### 使用独立模式生成证书

一般用于没有 `Web` 服务在运行

```shell
# 先停止 `nginx`
sudo systemctl stop nginx

# 运行certbot
sudo certbot certonly --standalone -d example.com -d www.example.com

# 启动nginx
sudo systemctl start nginx
```

#### 通过写入已运行的 `Web` 服务器的 `Webroot` 目录来获取证书。

```shell
certbot certonly --webroot -w /var/www/example -d www.example.com
```

#### 测试证书更新过程

```shell
sudo certbot renew --dry-run
```

#### 手动更新证书

```shell
sudo certbot renew
```

#### 使用通配符证书

```shell
sudo certbot certonly --manual --preferred-challenges=dns -d "*.example.com" -d example.com

# Certbot 将提示需要添加 DNS TXT 记录以进行域验证
```

#### `Nginx` 配置 `https` 证书

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
}
```

#### `Apache` 配置 `https` 证书

```xml
<VirtualHost *:443>
    ServerName example.com

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem
</VirtualHost>
```

#### 强制更新证书

```shell
sudo certbot renew --force-renewal
```

#### 重新颁发证书

```shell
sudo certbot certonly --nginx -d example.com -d www.example.com
```

#### 删除一个证书

```shell
sudo certbot delete

# 会提示选择已配置证书的域名
Which certificate would you like to delete?
  1: example.com
  2: test.com
```

#### 吊销证书

```shell
sudo certbot revoke --cert-path /etc/letsencrypt/live/example.com/fullchain.pem
```

#### 默认的证书更新脚本

通常 `Certbot` 会设置 `/etc/cron.d/certbot` 计划任务，默认是每天两次自动续订距离到期日期还有三十天的证书，并且 `systemd` 会配置一个 `certbot.timer` 的服务来执行。

```shell
● certbot.timer - Run certbot twice daily
     Loaded: loaded (/lib/systemd/system/certbot.timer; enabled; vendor preset:>
     Active: active (waiting) since Mon 2022-04-11 20:52:46 UTC; 4min 3s ago
    Trigger: Tue 2022-04-12 00:56:55 UTC; 4h 0min left
   Triggers: ● certbot.service

Apr 11 20:52:46 jammy-encrypt systemd[1]: Started Run certbot twice daily.
```

#### 手动添加计划任务更新证书

```shell
0 3 * * * certbot renew --quiet --post-hook "systemctl reload nginx"

0 3 * * * certbot renew --quiet && systemctl reload nginx

```



