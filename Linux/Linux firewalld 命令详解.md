### 简介

`firewalld` 是一个在 `Linux` 中的防火墙管理工具，提供动态接口管理网络流量，它使用区域来定义网络连接的信任级别，并支持 IPv4 和 IPv6。

### 常用示例

#### 启动防火墙

```shell
sudo systemctl start firewalld
```

#### 停止防火墙

```shell
sudo systemctl stop firewalld
```

#### 设置防火墙开机自启动

```shell
sudo systemctl enable firewalld
```

#### 禁止防火墙开机自启动

```shell
sudo systemctl disable firewalld
```

#### 检查防火墙的状态

```shell
sudo systemctl status firewalld
```

#### 重新加载防火墙的配置

```shell
sudo firewall-cmd --reload
```

#### 查看活动防火墙状态

```shell
sudo firewall-cmd --state
```

#### 列出所有活动区域

```shell
sudo firewall-cmd --get-active-zones
```

#### 查看指定区域的规则

```shell
sudo firewall-cmd --list-all --zone=public
```

#### 列出所有区域

```shell
sudo firewall-cmd --get-zones
```

#### 设置默认的区域

```shell
sudo firewall-cmd --set-default-zone=trusted
```

#### 添加一个接口到区域

```shell
sudo firewall-cmd --zone=public --add-interface=eth0
```

#### 从区域中移除一个接口

```shell
sudo firewall-cmd --zone=public --remove-interface=eth0
```

#### 查看指定区域的接口

```shell
sudo firewall-cmd --get-zone-of-interface=eth0
```

#### 列出所有支持的服务

```shell
sudo firewall-cmd --get-services
```

#### 添加一个服务到区域

```shell
sudo firewall-cmd --zone=public --add-service=http
```

#### 从区域中移除一个服务

```shell
sudo firewall-cmd --zone=public --remove-service=http
```

#### 检查服务是否是启动的

```shell
sudo firewall-cmd --zone=public --query-service=http
```

#### 使服务变更永久生效

```shell
sudo firewall-cmd --zone=public --add-service=http --permanent
```

#### 临时打开一个端口

```shell
sudo firewall-cmd --zone=public --add-port=8080/tcp
```

#### 临时关闭一个端口

```shell
sudo firewall-cmd --zone=public --remove-port=8080/tcp
```

#### 使变更端口永久生效

```shell
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
```

#### 列出在区域中打开的端口

```shell
sudo firewall-cmd --zone=public --list-ports
```

#### 允许来自指定 IP 的流量

```shell
sudo firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.100" accept'
```

#### 阻止来自指定 IP 的流量

```shell
sudo firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.100" drop'
```

#### 记录并丢弃来自指定 IP 的流量

```shell
sudo firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.100" log prefix="Blocked: " level="info" drop'
```

#### 使规则永久生效

```shell
sudo firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.100" accept' --permanent
```

#### 启用伪装

```shell
sudo firewall-cmd --zone=public --add-masquerade
```

#### 禁用伪装

```shell
sudo firewall-cmd --zone=public --remove-masquerade
```

#### 在区域之间转发流量

```shell
sudo firewall-cmd --zone=trusted --add-forward-port=port=80:proto=tcp:toport=8080:toaddr=192.168.1.100
```

#### 在脚本或恢复模式中使用 `firewall-offline-cmd`

```shell
firewall-offline-cmd --add-service=http
```

#### 永久保存所有变更到磁盘

```shell
sudo firewall-cmd --runtime-to-permanent
```


