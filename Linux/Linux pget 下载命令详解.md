### 简介

`pget` 命令是一个实用程序，它允许通过将文件分成多个部分并同时下载每个部分来并行下载文件。这使得文件下载速度更快，特别是对于大文件。

### 安装

* `Debian/Ubuntu`

```shell
sudo apt install pget
```

* `Red Hat/CentOS`

```shell
sudo yum install pget
```

* 从源码构建

```shell
make
sudo make install
```

### 示例用法

#### 基础用法

```shell
pget http://example.com/file.zip
```

#### 设置并发连接数

```shell
pget -n 4 http://example.com/file.zip
```

#### 恢复中断的下载

```shell
pget -r http://example.com/file.zip
```

#### 指定输出文件

```shell
pget -o custom_name.zip http://example.com/file.zip
```

#### 设置连接超时时间

```shell
# 单位：秒
pget -t 10 http://example.com/file.zip
```

#### 限制下载速度

```shell
pget -l 500k http://example.com/file.zip
```

#### 启用详细日志的详细模式

```shell
pget -v http://example.com/file.zip
```

#### 与其他命令集成

```shell
pget -n 4 http://example.com/file.zip | tar -xz
```

### 使用场景

* `pget` 对于下载大文件特别有用，将文件分成几部分可以​​显著减少下载时间。

* 由于 `pget` 具有恢复下载的功能，因此对于容易断线的网络来说，它是理想的选择。

### 与其他工具的比较

**`pget` vs `wget`**

* `pget` 原生支持并行下载

* `wget` 可以使用 `--continue` 和 `--limit-rate` 模拟类似的行为

**`pget` vs `aria2`**

* `aria2` 功能更加丰富，支持多源并行下载

* `pget` 是轻量级的，更易于用于简单的并行下载。


