### 简介

管理用户与组别是 `Linux` 系统中的基本部分，也是使用 `Linux` 必须要掌握的技能。

### `useradd` 常用选项

> 用于创建新用户账户

#### 创建一个新用户

```shell
sudo useradd alice

# 创建一个用户名为 alice 的用户。
```

#### 创建新用户时且指定家目录位置

```shell
sudo useradd -m alice

# 创建用户名为 alice，家目录位置在 /home/alice
```

#### 创建新用户时指定默认的 `shell`

```shell
sudo useradd -s /bin/bash alice

# 创建用户名为 alice，且分配 /bin/bash 作为默认shell
```

#### 创建新用户时指定 `User ID(UID)`

```shell
sudo useradd -u 1500 alice

# 创建用户名为 alice，且分配UID为1500
```

#### 创建新用户时分配主要的组

```shell
sudo useradd -g developers alice
```

#### 创建新用户时分配额外的组

```shell
sudo useradd -G sudo,staff alice

# 多个组用逗号隔开
```

### `passwd` 常用选项

> 用于设置或变更用户密码

#### 设置用户的密码

```shell
sudo passwd alice
```

#### 强制用户在下次登录时更改密码

```shell
sudo passwd -e alice
```

#### 锁定用户的账户

```shell
sudo passwd -l alice

# 阻止用户登录
```

#### 解锁用户的账户

```shell
sudo passwd -u alice
```

### `usermod` 常用选项

> 用于修改现有的用户帐户

#### 修改用户的家目录位置

```shell
sudo usermod -d /new/home alice
```

#### 添加用户到组

```shell
sudo usermod -aG sudo alice
```

#### 更变用户的默认 `shell`

```shell
sudo usermod -s /bin/zsh alice
```

#### 锁定用户账户

```shell
sudo usermod -L alice
```

#### 用户重命名

```shell
sudo usermod -l newname alice
```

### `userdel` 常用选项

> 用于删除用户账户

#### 删除指定的用户但保留家目录

```shell
sudo userdel alice
```

#### 删除指定的用户且删除家目录

```shell
sudo userdel -r alice
```

### `groupadd` 常用选项

> 用于创建新的组

#### 创建一个组

```shell
sudo groupadd developers
```

#### 创建一个组时指定组ID(GID)

```shell
sudo groupadd -g 1500 developers
```

### `groupmod` 常用选项

> 用于修改存在的组

#### 重命名组

```shell
sudo groupmod -n newgroupname developers
```

#### 变更组的GID

```shell
sudo groupmod -g 2000 developers
```

### `groupdel` 常用选项

> 用于删除组

#### 删除一个组

```shell
sudo groupdel developers
```

### `id` 常用选项

> 用于显示用户和组ID

#### 显示当前用户的信息

```shell
id
```

#### 显示指定用户的信息

```shell
id alice
```

### `who`、`whoami` 常用选项

> 用于显示用户登录的信息

#### 显示所有已登录的用户

```shell
who
```

#### 显示当前登录的用户

```shell
whoami
```

### `groups` 常用选项

> 显示用户所属的组

#### 显示当前用户所属的组

```shell
groups
```

#### 显示指定用户所属的组

```shell
groups alice
```

### `chage` 常用选项

> 用于管理密码老化策略

#### 设置密码过期天数

```shell
sudo chage -M 90 alice
```

#### 查看密码到期详情

```shell
sudo chage -l alice
```





