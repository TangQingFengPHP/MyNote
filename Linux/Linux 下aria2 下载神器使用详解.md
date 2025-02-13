### 简介

`aria2` 是一个轻量级的多协议命令行下载实用工具。它支持各种协议，如HTTP， HTTPS， FTP， SFTP， BitTorrent和Metalink。它以使用多个连接同时从多个来源下载文件的能力而闻名，从而提高了下载速度。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt update
sudo apt install aria2
```

* `CentOS/RHEL`

```shell
sudo yum install aria2
```

* `Fedora`

```shell
sudo dnf install aria2
```

### 示例用法

#### 下载单个文件

```shell
aria2c http://example.com/file.zip
```

#### 下载多个文件

```shell
aria2c http://example.com/file1.zip http://example.com/file2.zip
```

#### 使用恢复支持下载文件

> `-c`：可以恢复之前中断的下载

```shell
aria2c -c http://example.com/largefile.zip
```

#### 使用多个连接下载文件

默认情况下，`aria2` 仅使用一个连接来下载文件。要使用多个连接（也称为多线程下载），使用 `-x` 选项，后跟连接数。

```shell
aria2c -x 4 http://example.com/largefile.zip
```

#### 从 Metalink 文件下载

> `Metalink` 是一种提供文件镜像和校验和列表的格式

```shell
aria2c metalink://example.com/file.metalink
```

#### 从 Torrent 文件下载

> 使用 aria2 通过 BitTorrent 下载文件

```shell
aria2c example.torrent
```

#### 使用磁力链接下载

```shell
aria2c "magnet:?xt=urn:btih:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

#### 设置下载速度限制

```shell
aria2c --max-download-limit=500K http://example.com/file.zip
```

#### 下载到特定目录

```shell
aria2c -d /path/to/directory http://example.com/file.zip
```

#### 后台下载（守护进程模式）

```shell
aria2c --daemon http://example.com/file.zip
```

#### aria2 配置文件

> 可以使用配置文件来设置 aria2 的默认参数。配置文件通常位于 ~/.aria2/aria2.conf

```shell
# aria2.conf
continue=true
max-connection-per-server=4
dir=/path/to/downloads
max-download-limit=1M
```

#### 使用配置文件运行 aria2

```shell
aria2c --conf-path=/path/to/aria2.conf http://example.com/file.zip
```

#### 将 aria2 与 JSON-RPC 结合使用

> aria2 支持通过 JSON-RPC 进行远程控制，可以通过编程方式或远程方式与程序进行交互

启动 `aria2c RPC` 服务器

```shell
aria2c --enable-rpc --rpc-listen-all=true --rpc-allow-origin-all --rpc-listen-port=6800
```

* `--enable-rpc`：启用 RPC 模式

* `--rpc-listen-all=true`：允许来自任何 IP 的连接

* `--rpc-allow-origin-all`：允许跨源请求

* `--rpc-listen-port=6800`：监听端口 6800

#### 显示下载进度

```shell
aria2c -q --show-console-readout=false http://example.com/file.zip
```

* `-q`：安静模式（抑制控制台输出）

* `--show-console-readout=false`：禁用控制台读数以获得更清晰的显示

#### 检查下载速度

> 可以通过指定 -j（最大并发下载数量）来获取当前下载速度

```shell
aria2c -j 4 http://example.com/file.zip
```

#### 检查文件完整性（哈希检查）

> 通过检查校验和

```shell
aria2c --check-integrity http://example.com/file.zip
```

#### 使用 Cron 自动进行每日下载

```shell
0 3 * * * /usr/bin/aria2c -d /path/to/directory http://example.com/dailyfile.zip
```