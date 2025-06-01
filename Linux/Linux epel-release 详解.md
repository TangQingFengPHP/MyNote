### 简介

`epel-release` 是一个 `RPM` 软件包，用于在基于 `Red Hat` 的 `Linux` 发行版（如 `RHEL、CentOS、Rocky Linux、AlmaLinux 和 Oracle Linux`）上启用 `EPEL（Extra Packages for Enterprise Linux`） 软件仓库。`EPEL` 是一个由 `Fedora` 项目维护的社区驱动的额外软件包仓库，提供不在标准 `RHEL` 或其衍生发行版基础仓库中的高质量开源软件包。

### 什么是 epel-release？

* 全称：`Extra Packages for Enterprise Linux Release`

* 作用：`epel-release` 是一个配置包，安装后会在系统中添加 `EPEL` 仓库的配置文件（通常位于 `/etc/yum.repos.d/`）和 `GPG` 密钥，用于验证软件包的完整性。它允许使用 `yum` 或 `dnf` 包管理器从 `EPEL` 仓库安装额外软件包。

* 目标系统：主要用于 `RHEL` 及其衍生发行版（如 `CentOS、Rocky Linux、AlmaLinux、Oracle Linux`），支持版本通常包括 `RHEL 6、7、8、9` 等。

* 内容：
    * 提供 `EPEL` 仓库的元数据和访问地址。
    * 包含 `GPG` 密钥，确保软件包来源可信。
    * 默认启用稳定版 `EPEL` 仓库，另有可选的 `epel-testing` 仓库（包含尚未稳定的实验性软件包）。

* 特点：

    * `EPEL` 软件包基于 `Fedora` 软件包，遵循 `Fedora` 打包准则，确保与 `RHEL` 基础仓库兼容，不会替换或冲突核心软件包。
    * 仅包含自由和开源软件，不包含专利限制的软件（如多媒体编解码器）或专有软件。
    * 提供工具（如 `htop、etckeeper`）、编程语言模块（如 `Python、Perl、Ruby`）、浏览器（如 `Chromium`）等。

### 安装 epel-release

#### CentOS / Rocky Linux / AlmaLinux

这些发行版通常在默认的 `extras` 仓库中包含 `epel-release` 软件包，可直接安装：

```shell
sudo yum install epel-release    # CentOS 7 或更早版本
sudo dnf install epel-release    # CentOS 8/9、Rocky Linux、AlmaLinux
```

`extras` 仓库默认启用，包含 `epel-release` 包。安装后，`EPEL` 仓库会自动配置并启用

#### RHEL

`RHEL` 系统中，`epel-release` 不在默认仓库中，需手动下载 `RPM` 包或启用额外仓库（如 `CodeReady Builder`）

* 启用 `CodeReady Builder` 仓库（提供构建工具，可能为 `EPEL` 依赖所需）

```shell
ARCH=$(arch)
sudo subscription-manager repos --enable "codeready-builder-for-rhel-9-${ARCH}-rpms"
```

* 安装 epel-release

```shell
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```

#### Oracle Linux

`Oracle Linux` 使用特定的 `EPEL` 包（`oracle-epel-release-elX`）

```shell
sudo dnf install oracle-epel-release-el9  # Oracle Linux 9
sudo dnf install oracle-epel-release-el8  # Oracle Linux 8
```

#### 验证安装

安装完成后，检查 `EPEL` 仓库是否启用

```shell
sudo yum repolist | grep epel
sudo dnf repolist | grep epel
```

输出示例

```shell
repo id      repo name                                    status
epel         Extra Packages for Enterprise Linux 9 - x86_64  13,746
```

确认 `/etc/yum.repos.d/epel.repo` 和 `/etc/yum.repos.d/epel-testing.repo` 文件已创建

### 配置文件详解

安装 `epel-release` 后，会在 `/etc/yum.repos.d/` 目录下生成以下文件

* `epel.repo`：稳定版 `EPEL` 仓库配置，默认启用。

* `epel-testing.repo`：测试版 `EPEL` 仓库，默认禁用（需手动启用）。

* `GPG` 密钥：通常位于 `/etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-X`（`X` 为版本号，如 9）。

`epel.repo` 示例内容

```ini
[epel]
name=Extra Packages for Enterprise Linux $releasever - $basearch
baseurl=https://download.fedoraproject.org/pub/epel/$releasever/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-$releasever
```

* 字段说明
    * `name`：仓库名称。
    * `baseurl`：软件包下载地址。
    * `enabled=1`：仓库启用（0 表示禁用）。
    * `gpgcheck=1`：启用 `GPG` 签名验证。
    * `gpgkey：GPG` 密钥路径。

**启用 `epel-testing`**

编辑 `/etc/yum.repos.d/epel-testing.repo`，将 `enabled=0` 改为 `enabled=1`

```shell
sudo sed -i 's/enabled=0/enabled=1/' /etc/yum.repos.d/epel-testing.repo
```

### EPEL 提供的软件包

`EPEL` 提供数千种额外软件包，涵盖以下类别

* 系统工具：`htop、inxi、etckeeper` 等。

* 开发工具：`Python、Perl、Ruby` 模块等。

* 多媒体：`ImageMagick、GraphicsMagick` 等（不含专利受限的编解码器）。

* 浏览器：`chromium` 等。

* 其他：如 `nginx`（早期版本）、`zabbix` 等。

查看可用软件包：

```shell
sudo dnf list --available | grep ^epel
```

或访问 `Fedora` 包网页查看完整列表：`https://src.fedoraproject.org/projects/epel[]`

### 禁用 EPEL 仓库

#### 临时禁用：

```shell
sudo dnf --disablerepo=epel install `<package>`
```

#### 永久禁用：

```shell
sudo sed -i 's/enabled=1/enabled=0/' /etc/yum.repos.d/epel.repo
```

#### 移除 EPEL：

```shell
sudo dnf remove epel-release
```

### 相关命令

#### 查看系统版本（确保 EPEL 版本匹配）

```shell
cat /etc/os-release | grep VERSION_ID | cut -d '"' -f 2 | cut -d '.' -f 1
```

#### 安装依赖工具

```shell
sudo dnf install wget
```

#### 检查 GPG 密钥

```shell
rpm -qa gpg-pubkey* | xargs rpm -qi | grep -i epel
```
