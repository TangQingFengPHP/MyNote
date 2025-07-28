### 简介

* `SSHFS（SSH File System）` 基于 `FUSE（Filesystem in Userspace）` 和 `SSH` 协议实现。

* 通过 `SSH` 协议将远程目录“挂载”到本地文件系统，读写操作由 `SSH` 隧道加密传输。

* 不需在服务器上安装额外的文件系统服务，只要开启了 `SSH` 即可使用。

### 核心特性

* 安全传输：所有数据通过 `SSH` 加密通道传输

* 无需特殊权限：使用标准 `SSH` 认证（密钥/密码）

* 跨平台支持：`Linux/macOS/Windows`（通过第三方工具）

* 透明访问：像操作本地文件一样操作远程文件

* 权限继承：文件权限基于 `SSH` 用户权限

### 安装与环境准备

### 安装 Fuse 与 sshfs

* `Debian/Ubuntu`

```shell
sudo apt update
sudo apt install sshfs
```

* `CentOS/RHEL`

```shell
sudo yum install epel-release
sudo yum install sshfs
```

* `Fedora`

```shell
sudo dnf install sshfs
```

* `Arch Linux`

```shell
sudo pacman -S sshfs
```

* `macOS`

```shell
brew install macfuse
brew install sshfs
```

* `Windows`

安装 [WinFsp](https://winfsp.dev/)

安装 [SSHFS-Win](https://github.com/winfsp/sshfs-win)

#### 确认 FUSE 可用

```shell
lsmod | grep fuse
# 如未加载，可手动加载：
sudo modprobe fuse
```

#### 用户加入 fuse 组（可选）

```shell
sudo usermod -aG fuse $USER
# 重新登录后生效
```

### 基本用法

#### 挂载远程目录

```shell
mkdir -p ~/mnt/remote
sshfs user@remote.server:/path/to/dir ~/mnt/remote
```

* `user@remote.server`：SSH 登录信息

* `/path/to/dir`：远程目录

* `~/mnt/remote`：本地挂载点目录

执行后，本地 `~/mnt/remote` 就像本地目录一样，可以 `ls、cat、cp、vim` 等操作。

#### 卸载（Unmount）

```shell
fusermount -u ~/mnt/remote
# 或
sudo umount ~/mnt/remote

# Windows (管理员权限)
net use X: /delete

# 强制卸载
# 如果挂载点异常（如远程连接断开但未自动卸载），可加 -z 强制卸载：
fusermount -uz ~/remote_docs
```

#### 挂载测试：

```shell
sudo mount -av  # 检查并挂载所有 fstab 条目
```

#### 指定端口

```shell
sshfs -p 2222 user@example.com:/data ~/mnt/data
```

#### 使用密钥认证

```shell
sshfs -o IdentityFile=~/.ssh/id_rsa user@server:/backup ~/backups
```

### 常用挂载选项

| 选项                            | 含义                                                                          |
| ------------------------------- | ----------------------------------------------------------------------------- |
| `-o reconnect`                  | 网络断开后自动重连                                                            |
| `-o cache_timeout=60`           | 本地缓存超时（秒），读取目录列表等结果会缓存在本地                            |
| `-o attr_timeout=60`            | 元数据（权限、时间戳等）缓存时间                                              |
| `-o allow_other`                | 允许其他用户访问挂载目录（需要在 `/etc/fuse.conf` 中启用 `user_allow_other`） |
| `-o IdentityFile=~/.ssh/id_rsa` | 指定私钥文件                                                                  |
| `-o Ciphers=…`                  | 指定 SSH 加密算法，如 `aes256-ctr`，可根据性能与安全需求调整                  |
| `-o uid=1000,gid=1000`          | 强制以本地指定的用户/组 ID 访问远程文件，避免权限不匹配                       |
| `-o follow_symlinks`            | 在远程跟随符号链接（否则会以文本文件形式显示链接内容）                        |
| `-p PORT`            | 指定 SSH 端口|
| `-C`            | 启用压缩|
| `-o ServerAliveInterval=15`            | 保持连接活跃|
| `-o compression=no`            | 禁用压缩（高速网络）|
| `-d 或 -o debug`            | 启用 debug 模式 |
| `-o sshfs_debug`            | 详细的 SSHFS 特定调试 |

示例：

```shell
sshfs user@host:/data ~/mnt/data \
  -o reconnect \
  -o cache_timeout=120 \
  -o attr_timeout=120 \
  -o uid=$(id -u),gid=$(id -g) \
  -o allow_other
```

### 配置文件优化 (~/.ssh/config)

```shell
Host myserver
  HostName server.example.com
  User myuser
  Port 2222
  IdentityFile ~/.ssh/myserver_key
  ServerAliveInterval 15
  ServerAliveCountMax 3
  Compression yes
```

使用配置简写：

```shell
sshfs myserver:/path ~/mountpoint
```

### SSHFS 的性能与优化

#### 禁用压缩（高速网络）：

```shell
sshfs -o compression=no user@server:/path ~/mnt
```

#### 调整缓存策略：

```shell
sshfs -o cache=yes -o kernel_cache user@server:/path ~/mnt
```

#### 选择高效密码：

```shell
sshfs -o Ciphers=aes128-ctr user@server:/path ~/mnt
```

#### 增加缓存大小：

```shell
sshfs -o max_readahead=524288 user@server:/path ~/mnt
```

#### 缓冲与缓存

* 目录与属性缓存：`cache_timeout、attr_timeout、entry_timeout`

* 读写缓存：`-o big_writes`（启用大包写入，一般默认开启）

#### 网络层优化

* 控制通道复用：在 `~/.ssh/config` 中开启

```shell
Host remote.server
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h:%p
  ControlPersist 10m
```

* 调整 `TCP` 参数（如窗口大小）可在系统层面配置

#### 并发与可靠性

* 对于大量小文件操作，可先使用 `rsync` 进行一次性同步，然后再长期挂载

* 若对并发性能要求极高，可考虑 `NFS over SSH tunnel` 或基于 `SFTP` 的专用方案

### 典型应用场景

* 远程开发：本地编辑远程项目，兼容各类 `IDE` 与编辑器

* 备份与同步：将远程日志/数据挂载到本地，由本地脚本定时处理

* 跨服务器迁移：在服务器间传输文件

* 受限环境访问：通过跳板机访问隔离网络

* 云资源管理：挂载云存储桶到本地

* 协作共享：无需配置 `NFS/SMB`，即可在多台机器间临时共享目录

* 数据分析：直接在本地化工具（如 `Python/R`）中处理远程数据，省去下载步骤

### 常见问题与排查

| 问题                                                  | 解决思路                                                                                          |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **挂载失败：`fusermount: unknown option: reconnect`** | 可能版本较旧，不支持该选项，升级 `sshfs` 或去掉该选项。                                           |
| **权限不足：`permission denied`**                     | 检查本地挂载目录权限；如使用 `allow_other`，需确认 `/etc/fuse.conf` 中已开启 `user_allow_other`。 |
| **I/O 卡顿**                                          | 增大缓存超时；使用 `big_writes`；或先 `rsync` 同步高频文件到本地再操作。                          |
| **自动重连无效**                                      | 确认网络不可用时 SSHfs 进程是否存活；可结合 `systemd --user` 写服务单元实现自动重启。             |

### 进阶脚本示例

在后台自动挂载并监控状态：

```shell
#!/usr/bin/env bash
MNT=~/mnt/remote
REMOTE=user@host:/path
LOG=/var/log/sshfs-mount.log

mount_sshfs() {
  sshfs "$REMOTE" "$MNT" \
    -o reconnect \
    -o cache_timeout=60 \
    -o attr_timeout=60 \
    -o uid=$(id -u),gid=$(id -g) \
    &>>"$LOG"
}

umount_sshfs() {
  fusermount -u "$MNT" &>>"$LOG"
}

case "$1" in
  start)
    mkdir -p "$MNT"
    mount_sshfs
    ;;
  stop)
    umount_sshfs
    ;;
  status)
    mount | grep "$MNT" && echo "Mounted" || echo "Not mounted"
    ;;
  *)
    echo "Usage: $0 {start|stop|status}"
    exit 1
    ;;
esac
```

### 系统启动自动挂载

* `/etc/fstab` 配置：

```shell
# 格式：
sshfs#[user@]host:[path] /mountpoint fuse.sshfs delay_connect,_netdev,reconnect 0 0

# 示例：
sshfs#user@example.com:/backups /mnt/backups fuse.sshfs delay_connect,_netdev,reconnect,IdentityFile=/home/user/.ssh/id_rsa,allow_other 0 0
```

* `systemd` 自动挂载：

创建 `/etc/systemd/system/mnt-remote.mount`：

```ini
[Unit]
Description=SSHFS Mount for Remote Server
After=network-online.target

[Mount]
What=user@example.com:/remote/path
Where=/mnt/remote
Type=fuse.sshfs
Options=allow_other,reconnect,ServerAliveInterval=15,IdentityFile=/home/user/.ssh/id_rsa

[Install]
WantedBy=multi-user.target
```

* 启用服务：

```shell
sudo systemctl daemon-reload
sudo systemctl enable --now mnt-remote.mount
```

### 高级用法

#### 端口转发挂载

```shell
ssh -L 2222:localhost:22 user@gateway.example.com
sshfs -p 2222 user@localhost:/path ~/mnt
```

#### 多服务器聚合挂载

```shell
# 使用 mergerfs 合并多个 SSHFS 挂载点
mergerfs -o defaults,allow_other,use_ino \
  /mnt/server1:/mnt/server2 /mnt/combined
```

#### 使用代理服务器

```shell
sshfs -o ssh_command="ssh -J proxyuser@proxy.example.com" \
  user@target:/path ~/mnt
```

#### 挂载远程服务器的子目录

如果远程目录有嵌套子目录，可直接挂载子目录，无需挂载整个父目录：

```shell
sshfs user@remote_host:/home/user/projects/blog ~/local_blog
```

#### 限制本地用户对挂载点的权限

通过 `-o uid 和 -o gid` 选项，指定本地用户 / 用户组对挂载点的权限（避免权限混乱）：

```shell
# 让本地 UID 为 1000、GID 为 1000 的用户拥有挂载点权限
sshfs -o uid=1000 -o gid=1000 user@remote_host:/data ~/remote_data
```

#### 挂载远程 Docker 容器目录

```shell
sshfs -o ssh_command="ssh -t user@host docker exec -i containername" :/container/path ~/local_mount
```

#### 使用 ProxyJump 跳板机

```shell
sshfs -o ProxyJump=jump_user@jump_host \
        target_user@target_host:/path \
        ~/mount_point
```

#### 限制挂载目录权限

```shell
mkdir -m 700 ~/secure_mount   # 仅当前用户可访问
sshfs -o umask=0077 user@server:/path ~/secure_mount
```

#### 审计挂载操作

```shell
# 记录所有文件访问
sshfs -o debug \
        -o logfile=/var/log/sshfs_audit.log \
        user@server:/path ~/mount_point
```