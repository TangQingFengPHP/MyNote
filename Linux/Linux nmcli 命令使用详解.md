### 简介

`nmcli` 是与 `NetworkManager` 交互的命令行工具，用于管理 `Linux` 系统上的网络连接。它提供了一种配置、监控和排除网络连接故障的全面方法。

**nmcli特性**

* `Network management`：轻松配置网络接口（Wi-Fi、以太网、VPN等）

* `Automation`：通过脚本自动执行网络设置或状态检查

* `Monitoring`：检查网络状态和统计数据

* `Troubleshooting`：从终端快速诊断网络问题

### 安装

* `Debian/Ubuntu`：

```shell
sudo apt update
sudo apt install network-manager
```

* `CentOS/RHEL`：

```shell
sudo yum install NetworkManager
```

* `Fedora`：

```shell
sudo dnf install NetworkManager
```

### 常用子命令

* `nmcli device status`：检查网络接口的状态

* `nmcli device wifi list`：列出可用的 Wi-Fi 网络

* `nmcli connection up <connection_name>`：激活网络连接

* `nmcli connection down <connection_name>`：停用网络连接

* `nmcli connection add`：添加新的网络连接

* `nmcli device disconnect <interface>`：断开网络接口

* `nmcli connection modify`：修改网络连接

### 示例用法

#### 检查网络状态

```shell
nmcli
```

#### 显示可用的网络设备

> 列出所有可用的网络接口

```shell
nmcli device
```

#### 显示所有网络连接的详细信息

```shell
nmcli connection show
```

#### 显示连接详细信息

```shell
nmcli connection show <connection_name>
```

#### 连接到 Wi-Fi 网络

```shell
nmcli device wifi connect <SSID> password <password>

# 示例
nmcli device wifi connect "MyWiFi" password "mypassword123"
```

#### 断开网络

```shell
nmcli device disconnect <interface>

# 示例
nmcli device disconnect eth0
```

#### 启用/禁用网络接口

```shell
nmcli device set eth0 managed no
nmcli device set eth0 managed yes
```

#### 为连接设置静态 IP

> 为以太网连接设置静态 IP 地址

```shell
nmcli connection modify <connection_name> ipv4.addresses <IP>/24 ipv4.gateway <gateway> ipv4.dns "<DNS>"
nmcli connection up <connection_name>

# 示例
nmcli connection modify "Wired connection 1" ipv4.addresses 192.168.1.100/24 ipv4.gateway 192.168.1.1 ipv4.dns "8.8.8.8"
nmcli connection up "Wired connection 1"
```

#### 创建新的 Wi-Fi 热点

```shell
nmcli device wifi hotspot ifname wlan0 con-name "MyHotspot" ssid "MyHotspotSSID" password "mypassword"
```

#### 添加新的 VPN 连接

```shell
nmcli connection add type vpn vpn-type <vpn_type> con-name "VPN Connection" --vpn-service-type <vpn_service> --vpn-username <username> --vpn-password-flags 0
```

#### 查看 VPN 连接详细信息

```shell
nmcli connection show <vpn_connection_name>
```

#### 重新启动 NetworkManager 服务

```shell
sudo systemctl restart NetworkManager
```

#### 检查日志中的错误

```shell
journalctl -u NetworkManager
```