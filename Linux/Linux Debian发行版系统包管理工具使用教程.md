### 一、简介

基于 `Debian` 发行版的 `Linux` 的包管理命令使用 `apt` 或 `apt-get`，可以用来安装包、更新包、更新包源、移除包等操作。

### 二、`apt` 与 `apt-get` 比较  

* `apt` 是新版包管理工具，提供了用户友好的命令行接口来管理包，其内部对 `apt-get` 相关的工具进行了更高级别的包装。

* `apt-get` 是老版本的工具，通常比 `apt` 更稳定，一般用于脚本和需要高级用法。

### 三、基础命令

#### 更新包源列表

```shell
sudo apt update
```

#### 更新所有安装的包到最新的版本

```shell
sudo apt upgrade
```

#### 执行完整的包升级

在完整包升级过程中还做额外的操作，如：移除过时的包

```shell
sudo apt full-upgrade
```

#### 安装指定的包

```shell
sudo apt install <package_name>
```

#### 移除指定的包并保留包的配置文件

```shell
sudo apt remove <package_name>
```

#### 移除指定的包并删除包的配置文件

```shell
sudo apt purge package_name
```

#### 删除下载的包文件

```shell
sudo apt autoclean
```

#### 移除未使用的包和依赖包

```shell
sudo apt autoremove

sudo apt-get autoremove
```

#### 通过包名搜索指定的包

```shell
sudo apt search <package_name>
```

#### 查看包的详细信息

```shell
sudo apt show <package_name>
```

#### 列出已安装的包

```shell
sudo apt list --installed
```

#### 使用 `apt-get` 下载包文件而不安装

```shell
sudo apt-get download <package_name>
```

#### 使用 `apt-get` 修复损坏的包依赖关系

```shell
sudo apt-get install -f
```

#### 使用 `apt-get` 添加自定义包仓库

```shell
sudo add-apt-repository <repository_name>
sudo apt update
```

#### 使用 `apt-get` 升级整个系统到下一个发行版

```shell
sudo apt-get dist-upgrade
```

#### 同时安装多个包

```shell
sudo apt install <package1> <package2> <package3>
```

#### 检查包版本

```shell
apt-cache policy <package_name>
```

#### 模拟包安装

模拟包安装会检查安装的过程，实际上不会安装包

```shell
sudo apt-get install --simulate <package_name>
```

#### 将软件包保留为其当前版本防止软件包被升级

```shell
sudo apt-mark hold <package_name>
```

#### 取消保留软件包允许软件包再次升级

```shell
sudo apt-mark unhold <package_name>
```

### 四、常见问题

#### 不能获取锁异常

`Could not get lock`

一般发生在已经有 `apt` 进程在运行中了，需要手动杀死之前的进程

```shell
sudo killall apt apt-get
```

#### 清理损坏的包并清理缓存

```shell
sudo apt clean
```

