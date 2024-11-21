### 简介

`dnf` 是基于 `Red Hat Linux` 发行版的下一代包管理工具，它代替 `yum` 提供更好的性能、更好的依赖处理和更好的模块化架构。

### 基础语法

```shell
dnf [options] [command] [package]
```

### 常用命令用法

#### 更新元数据缓存

```shell
sudo dnf check-update

# 检查已安装的包是否有可用的更新
```

#### 更新所有安装的包到最新的版本

```shell
sudo dnf update
```

#### 更新所有安装的包到最新的版本且移除过时的包

```shell
sudo dnf upgrade
```

#### 安装指定的包

```shell
sudo dnf install <package_name>
```

#### 安装多个包

```shell
sudo dnf install <package1> <package2>
```

#### 安装指定的包版本

```shell
sudo dnf install <package_name-version>
```

#### 移除指定的包

```shell
sudo dnf remove <package_name>
```

#### 移除包和它未使用的依赖包

```shell
sudo dnf autoremove
```

#### 通过关键词搜索指定的包

```shell
dnf search <keyword>
```

#### 显示包的详细信息

```shell
dnf info <package_name>
```

#### 列出所有可用的包组

```shell
dnf group list
```

#### 安装一组包

```shell
sudo dnf group install "<group_name>"
```

#### 移除一组包

```shell
sudo dnf group remove "<group_name>"
```

#### 列出所有仓库源

```shell
dnf repolist
```

#### 启用指定的仓库

```shell
sudo dnf config-manager --set-enabled <repo_name>
```

#### 禁用指定的仓库

```shell
sudo dnf config-manager --set-disabled <repo_name>
```

#### 清除所有缓存的数据

```shell
sudo dnf clean all
```

#### 仅清除过期的缓存数据

```shell
sudo dnf clean expire-cache
```

#### 列出所有已安装的包

```shell
dnf list installed
```

#### 列出所有可用的包

```shell
dnf list available
```

#### 列出指定的已安装的包

```shell
dnf list <package_name>
```

#### 包降级到上一个版本

```shell
sudo dnf downgrade <package_name>
```

#### 查看包操作的历史记录

```shell
dnf history
```

#### 撤销指定的操作

```shell
sudo dnf history undo <transaction_id>
```

#### 重做指定的操作

```shell
sudo dnf history redo <transaction_id>
```

### 配置文件

`dnf` 主配置文件在 `/etc/dnf/dnf.conf`

示例配置如下：

```shell
[main]
gpgcheck=1 # 确保软件包使用 GPG 密钥签名
installonly_limit=3 # 确保软件包使用 GPG 密钥签名
clean_requirements_on_remove=True # 当删除包时，删除未使用的依赖项。
```

### `DNF` 模块

模块提供多个软件包的版本

#### 列出可用的模块

```shell
dnf module list
```

#### 安装指定的模块

```shell
sudo dnf module install <module_name>
```

#### 启用指定的模块

```shell
sudo dnf module enable <module_name>
```

#### 禁用指定的模块

```shell
sudo dnf module disable <module_name>
```

### `DNF` 插件

`DNF` 支持插件扩展额外的功能，如：

* `dnf-plugins-core`：提供如 `config-manager` 的工具的插件

* `dnf-plugin-subscription-manager`：管理 `Red Hat` 订阅

> 安装插件

```shell
sudo dnf install dnf-plugins-core
```

### 高级用法

#### 并行下载包

在配置文件 `/etc/dnf/dnf.conf` 中添加如下配置：

```ini
max_parallel_downloads=5
```

#### 锁定包版本阻止更新

```shell
sudo dnf versionlock add <package_name>
```

#### 解除锁定包版本

```shell
sudo dnf versionlock delete <package_name>
```
