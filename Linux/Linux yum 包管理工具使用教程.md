### 一、简介

`yum` 是基于 `RPM` Linux 发行版的包管理工具，例如：`CentOS`，`RHEL`，`Fedora`，它简化包管理操作，例如：安装、更新、移除、搜索包。

### 二、基础命令

#### 更新包仓库

```shell
sudo yum update
```

#### 安装指定的包

```shell
sudo yum install <package_name>
```

#### 同时安装多个包

```shell
sudo yum install <package1> <package2> <package3>
```

#### 移除指定的包但保留配置文件

```shell
sudo yum remove <package_name>
```

#### 移除指定的包和它的配置文件

```shell
sudo yum erase <package_name>
```

#### 更新所有包到最新的版本

```shell
sudo yum upgrade
```

#### 更新指定的包到最新的版本

```shell
sudo yum upgrade <package_name>
```

#### 清理缓存的包文件

```shell
sudo yum clean all
```

#### 通过关键词搜索指定的包

```shell
sudo yum search <keyword>
```

#### 显示包的详细信息

```shell
sudo yum info <package_name>
```

#### 列出所有安装的包

```shell
sudo yum list installed
```

#### 列出在仓库中所有可用的包

```shell
sudo yum list available
```

### 三、仓库管理

#### 添加一个仓库源

在 `/etc/yum.repos.d/` 文件夹下创建自定义的仓库文件，如：`custom.repo`

添加以下内容

```shell
[custom-repo] # 仓库ID标识符
name=Custom Repository # 自定义仓库名
baseurl=http://example.com/repo/ # 仓库元数据地址
enabled=1 # 表示启用仓库
gpgcheck=1 # 表示启用GPG签名验证，通过验证下载包的 GPG 签名来确保其真实性和完整性。
gpgkey=http://example.com/repo/RPM-GPG-KEY # GPG key的文件位置，可以是本地文件或远程地址
```

然后执行 `sudo yum update`

#### 启用/禁用仓库

* 启用仓库

```shell
sudo yum --enablerepo=<repo_name> install <package_name>
```

* 禁用仓库

```shell
sudo yum --disablerepo=<repo_name> install <package_name>
```

#### 查看所有配置的仓库

```shell
sudo yum repolist
```

### 四、高级命令

#### 仅下载包不安装

```shell
sudo yum install --downloadonly --downloaddir=/path/to/dir <package_name>
```

#### 检查可用的包更新

```shell
sudo yum check-update
```

#### 移除不再依赖的包

```shell
sudo yum autoremove
```

#### 查看 `yum` 操作历史

```shell
sudo yum history
```

#### 指定操作id撤销操作

```shell
sudo yum history undo <transaction_id>
```

#### 查看包的依赖包

```shell
sudo yum deplist <package_name>
```

#### 锁定包版本防止更新

需要提前安装个 `yum-plugin-versionlock` 包

```shell
sudo yum versionlock <package_name>
```

#### 强制重新安装包

```shell
sudo yum reinstall <package_name>
```

#### 仅清理包的元数据

```shell
sudo yum clean metadata
```

#### 从URL中安装包

```shell
sudo yum install http://example.com/packages/package.rpm
```

#### 跳过不能下载的依赖包

```shell
sudo yum install -y <package_name> --skip-broken
```

#### 重新构建 `RPM` 数据库

```shell
sudo rpm --rebuilddb
```


