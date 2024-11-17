### 一、简介

`dpkg` 是基于 `Debian` 发行版 `Linux` 系统的低级包管理工具，可以手动安装、配置、移除 `.deb` 包，与 `apt` 命令不同的是，`dpkg` 不会自动处理包之间的依赖关系。

### 二、常用选项

#### 安装包

```shell
sudo dpkg -i <package_name>.deb
```

#### 手动处理包依赖

```shell
sudo apt --fix-broken install

# 这个命令会处理并安装丢失的依赖包
```

#### 移除包但保留配置文件

```shell
sudo dpkg -r <package_name>
```

#### 移除包且删除配置文件

```shell
sudo dpkg --purge <package_name>
```

#### 列出已经安装的包

```shell
dpkg -l
```

#### 搜索已安装的包

```shell
dpkg -l | grep <package_name>
```

#### 查找已安装的包的详细信息

详细信息包括：包名、版本、架构等

```shell
dpkg -s <package_name>
```

#### 查找已安装的包产生的文件

```shell
dpkg -L <package_name>
```

#### 查找指定的文件属于哪个包

```shell
dpkg -S </path/to/file>
```

#### 解压缩包但不安装

```shell
dpkg --unpack <package_name>.deb
```

#### 配置已经解压缩的包

```shell
sudo dpkg --configure <package_name>
```

#### 清理安装失败的包文件

```shell
sudo dpkg --remove --force-remove-reinstreq <package_name>
```

#### 查看包的文件内容

```shell
dpkg-deb -c <package_name>.deb
```

#### 提取包文件内容且不安装

```shell
dpkg-deb -x <package_name>.deb </path/to/extract>
```