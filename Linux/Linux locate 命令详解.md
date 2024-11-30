### 简介

`locate` 命令用于通过查询预构建的数据库来快速搜索文件和目录，该数据库包含来自文件系统的索引文件路径。它比 `find` 之类的命令要快得多，因为它不会实时搜索整个文件系统。

### 关键概念

* `locate` 命令依赖于数据库，通常位于 `/var/lib/mlocate/mlocate.db`，数据库文件会定期通过后台进程任务更新。

* 因为数据库更新不是实时的，最新创建的文件可能不会被查询到，直到下次数据库更新。

### 安装

```shell
# Debian/Ubuntu
sudo apt update
sudo apt install mlocate

# CentOS/RHEL/Fedora
sudo yum install mlocate
sudo updatedb  # 初始化数据库
```

### 示例用法

#### 搜索指定的文件

```shell
locate [filename]

locate example.txt
```

#### 搜索部分匹配项

```shell
locate [example]

# 示例输出如下：
# /home/user/documents/example.txt
# /home/user/example_folder
```

#### 搜索时不区分大小写

```shell
locate -i [Example]
```

#### 显示输出的结果条数

```shell
locate -n 5 filename
```

#### 配合 `grep` 使用

```shell
locate [filename] | grep '/home/user'
```

#### 搜索实际的文件系统，不通过数据库

```shell
locate -e filename
```

#### 更新本地数据库

```shell
sudo updatedb
```

#### 从数据库中排除目录

修改 `/etc/updatedb.conf` 文件，添加如下示例行：

```shell
PRUNEPATHS="/tmp /var/tmp /var/cache"
```

#### 统计匹配到的结果数

```shell
locate [filename] | wc -l

locate -c [filename]
```

#### 查找以指定后缀结尾的文件

```shell
locate *.log
```

#### 查找最近修改的文件

```shell
sudo updatedb

locate [example] | grep "$(date +%Y-%m-%d)"
```

#### 结合正则表达式使用

```shell
locate --regex -i "(\.mp4|\.avi)"
```


