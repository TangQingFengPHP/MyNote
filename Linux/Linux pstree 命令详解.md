### 简介

`pstree` 命令用于以分层（树状）格式显示正在运行的进程。它显示进程之间的关系，例如哪些进程是由其他进程生成的。

### 安装

```shell
# 基于 Debian/Ubuntu 的系统
sudo apt install psmisc

# 基于 CentOS/RHEL/Fedora 的系统
sudo yum install psmisc

# 使用dnf安装
sudo dnf install psmisc
```

### 基本语法

```shell
pstree [options] [pid | user]

# pid：显示以指定进程 ID 为根的树。
# user：仅显示指定用户拥有的进程。
```

### 示例用法

#### 以树状格式显示所有进程

```shell
pstree
```

#### 显示特定用户的进程

```shell
pstree <username>
```

#### 显示特定进程 ID 的树状结构

```shell
pstree <pid>
```

#### 显示进程 ID

```shell
pstree -p

# 这会在每个进程的名称旁边添加其 PID
```

#### 显示用户/组 ID

```shell
pstree -n
```

#### 显示命令行参数

```shell
pstree -a

# 显示包括用于启动每个进程的命令行参数
```

#### 高亮显示特定进程及其后代

```shell
pstree -h <pid>
```

#### 查看不截断的进程树

```shell
pstree -l

# 可以避免截断长行并将输出扩展为多行以提高可读性。
```

#### 查看指定进程ID的进程及其子进程

```shell
pstree -p <pid>
```

#### 按 PID 对具有相同祖先的进程进行排序

```shell
pstree -n
```

#### 不要压缩相同的子树 

```shell
pstree -c
```

#### 显示源自当前进程的进程树

```shell
pstree -p $$
```
