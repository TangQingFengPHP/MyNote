### 一、通过发行版的特定文件查看

发行版的说明文件一般位于 `/etc` 目录下面文件名包含：`release` 字样的文件

#### 查看 `CentOS`, `Red Hat` 的版本

```shell
cat /etc/centos-release
# Example: CentOS Linux release 7.9.2009 (Core)

cat /etc/redhat-release
# Example: Red Hat Enterprise Linux Server release 8.5 (Ootpa)
```

#### 查看 `Debian` 和 `Ubuntu` 的版本

```shell
cat /etc/debian_version

cat /etc/os-release
# Example:
# PRETTY_NAME="Ubuntu 22.04.3 LTS"
# VERSION="22.04.3 LTS (Jammy Jellyfish)"

cat /etc/issue
# Example: Ubuntu 22.04.3 LTS \n \l
```

#### 查看 `Fedora` 的版本

```shell
cat /etc/fedora-release
# Example: Fedora release 38 (Thirty Eight)
```

### 二、使用 `lsb_release` 命令查看

`lsb_release` 命令以标准的方式提供详细的发行版信息

如果没有安装先安装

```shell
sudo apt install lsb-release  # Debian/Ubuntu
sudo yum install redhat-lsb   # CentOS/RHEL
```

用法：

```shell
lsb_release -a

# Output:
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 22.04.3 LTS
Release:        22.04
Codename:       jammy
```

### 三、使用 `hostnamectl` 命令查看

`hostnamectl` 命令是包含在 `systemd` 工具套件的里面的

用法：

```shell
hostnamectl

# Output:
   Static hostname: myserver
         Icon name: computer-vm
           Chassis: vm
        Machine ID: xxxxxxxxxxxxx
           Boot ID: xxxxxxxxxxxxx
  Operating System: Ubuntu 22.04.3 LTS
            Kernel: Linux 5.15.0-79-generic
      Architecture: x86_64
```

### 四、使用 `uname` 命令查看 

`uname` 一般用来查看 `Linux` 内核版本

用法：

```shell
uname -a

# Output:
Linux myserver 5.15.0-79-generic #86-Ubuntu SMP Wed Sep 27 15:51:31 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
```

### 五、使用 `neofetch` 或 `screenfetch` 命令查看

安装：

```shell
sudo apt install neofetch   # Debian/Ubuntu
sudo yum install neofetch   # CentOS/RHEL

sudo apt install screenfetch   # Debian/Ubuntu
sudo yum install screenfetch   # CentOS/RHEL
```

用法：

```shell
neofetch

screenfetch
```

### 六、使用 `/etc/os-release` 文件查看

`/etc/os-release` 文件在现代化 `Linux` 系统中基本都存在，所以是一种标准的获取系统信息的方式
用法：

```shell
cat /etc/os-release
```

### 七、使用 `sw_vers` 命令可查看 `Mac` 版本信息

```shell
sw_vers
```


