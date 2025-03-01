### 简介

`mtr（My Traceroute）` 命令是一个结合了 `ping` 和 `traceroute` 功能的网络诊断工具。它提供网络路径的实时分析并帮助诊断连接问题

### 安装

* `Debian/Ubuntu`

```shell
sudo apt install mtr -y
```

* `RHEL/CentOS`

```shell
sudo yum install mtr -y
```

* `Fedora`

```shell
sudo dnf install mtr -y
```

### 示例用法

#### 基础用法

```shell
mtr <hostname/IP>
```

**示例**

这会持续追踪数据包到 `google.com` 的路由，并实时更新结果

```shell
mtr google.com
```

**示例输出**

```shell
  Host                Loss%   Snt   Last   Avg  Best  Wrst StDev
  1. router.lan       0.0%    10   1.1    1.0   0.9   1.3  0.2
  2. 192.168.1.1      0.0%    10   2.2    2.1   1.9   2.4  0.2
  3. isp-gateway      0.0%    10  10.2   11.1   9.8  12.2  0.8
  4. core-router      0.0%    10  20.1   21.3  19.8  23.2  1.1
  5. google.com       0.0%    10  30.5   32.0  29.9  34.1  1.3
```

**字段解释**

* `Host`：数据包经过的路由器/跳跃

* `Loss%`：该跳的数据包丢失百分比

* `Snt`：已发送的数据包数量

* `Last`：最后一个数据包的响应时间

* `Avg`：平均响应时间

* `Best/Wrst`：最佳和最差响应时间

* `StDev`：标准差（网络稳定性）

#### 针对固定数量的数据包运行 mtr

`mtr` 默认连续运行，使用 `-c <count>` 发送固定数量的数据包后停止

```shell
mtr -c 10 google.com
```

#### 显示数字 IP 地址

默认情况下，`mtr` 解析主机名，使用 `-n` 选项显示 IP 地址

```shell
mtr -n google.com
```

#### 显示为报告模式

一次性报告而不是实时更新

```shell
mtr -r google.com
```

#### 限制跳数

为了防止检查超出一定跳数

```shell
mtr -m 10 google.com
```

#### 显示已发送和已接收的数据包

```shell
mtr -b google.com
```

#### 显示每跳数据包数

控制发送到每一跳的数据包数量

```shell
mtr -c 5 --report google.com
```

### mtr 与 ping、traceroute比较

|  命令   |  功能   |
| --- | --- |
|  `ping`   |  测试与主机的连接，显示数据包丢失和延迟   |
|  `traceroute`   |  显示数据包到达目的地所采用的路由   |
|  `mtr`   |  将 `ping` 和 `traceroute` 与实时统计数据相结合   |