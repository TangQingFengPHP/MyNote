### 简介

`axel` 命令是一个轻量级、快速且用户友好的 `Linux` 命令行下载加速器。它通过将文件分割成片段并同时下载来加快下载速度，这对于大文件和网络不好时尤其有用。

`axel` 支持 `HTTP`、`HTTPS`、`FTP` 和 `FTPS` 协议。 

### 安装

* `Debian/Ubuntu`

```shell
sudo apt update
sudo apt install axel
```

* `CentOS/RHEL`

```shell
sudo yum install axel
```

* `Fedora`

```shell
sudo dnf install axel
```

* `MacOS`

```shell
brew install axel
```

### 关键特性

* 并行连接：将文件分成几部分并同时下载

* 断点续传：恢复中断的下载

* 简单而最小的输出：为提高效率而设计

### 常用选项

* `--max-speed, -s`：指定最大下载速度

* `--num-connections, -n`：指定连接数

* `--output, -o`：指定输出的文件名称

* `--no-proxy, -N`：不使用代理服务器

* `--verbose, -v`：显示更多状态信息

* `--quiet, -q`：安静模式，最小化输出

* `--alternate, -a`：显示一个可选的进度条

* `--header, -H`：添加额外的 `HTTP` 标头

* `--help, -h`：打印帮助信息

* `--version, -V`：打印版本信息

### 示例用法

#### 使用多个连接下载（默认值：4）

```shell
axel -n 8 http://example.com/file.zip

# 使用8个连接
```

#### 指定输出的文件名

```shell
axel -o custom_name.zip http://example.com/file.zip
```

#### 断点续传

```shell
axel -c http://example.com/file.zip
```

#### 限制下载速度

```shell
axel -s 500k http://example.com/file.zip

# 将下载速度限制为 500 KB/s
```

#### 设置重试次数

```shell
axel -r 3 http://example.com/file.zip
```

#### 安静模式

```shell
axel -q http://example.com/file.zip

# 最小化输出信息
```

#### 调试模式

```shell
axel -v http://example.com/file.zip

# 输出更多调试信息
```

#### 使用代理服务器

```shell
axel -x http://proxy_server:port http://example.com/file.zip
```

#### 设置用户代理

```shell
axel -U "Mozilla/5.0" http://example.com/file.zip
```

#### 指定可选的镜像

```shell
axel -a http://mirror1.com/file.zip http://mirror2.com/file.zip
```

### 配置文件示例

配置文件在 `/etc/axelrc` 或 `~/.axelrc`

```shell
# 重连延迟
reconnect_delay = 20

# 最大下载速度
max_speed = 500000

# 同时下载的连接数
num_connections = 4

# 连接超时时间
connection_timeout = 30

# 一次从所有当前连接读取的最大数量
buffer_size = 10240

# 输出更多信息
verbose = 1

# 默认下载目录
default_directory = /downloads

# 代理服务器
http_proxy=127.0.0.1
```


