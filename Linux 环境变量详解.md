## 一、什么是环境变量

环境变量，通常称为 `env` ，是对 `Linux` 操作系统中程序和进程的行为具有重要影响的动态值。这些变量作为一种手段，向软件传达基本信息，并塑造它们如何与环境交互。每个 `Linux` 进程都与一组环境变量相关联，这些环境变量指导 `Linux` 进程的行为和与其他进程的交互。

 `Linux` 环境变量是定义系统 `shell` 会话和程序行为的键值对。它们使系统管理员可以轻松地在应用程序和用户之间共享和区分配置。 
 
管理员可以使用 `Linux` 命令修改它们，以根据托管应用程序的需要调整服务器设置。根据任务的不同，还可以设置临时或永久环境变量。

## 二、变量的分类

> 变量分为环境变量和shell变量

环境变量：

* 环境变量在系统范围内可用且有效
* 脚本和应用程序可以使用环境变量
* 这些变量由所有生成的子进程和 `shell` 继承
* 按照惯例，环境变量的名称都是大写的

`shell` 变量：

* `shell` 变量仅仅在当前 `shell` 会话中可用
* 一般用于临时储存值的场景
* 每个 `shell` (如 `zsh` 和 `bash` )都有自己的一组内部 `shell` 变量

## 三、常见的环境变量

* `USER` 当前登录的用户
* `HOME` 当前用户的家目录
* `SHELL` 当前用户的shell路径
* `LANG` 当前语言设置
* `MAIL` 当前用户的邮件储蓄位置
* `EDITOR` 默认使用的编辑器
* `PATH` 执行命令时要搜索的目录列表
* `TERM` 当前的终端模拟器
* `PWD` 当前的工作目录
* `OLDPWD` 上一次的工作目录，保存在变量中，用于使用 `cd -` 来切换回上个目录

## 四、如何列出环境变量

* 通过 `env` 来列出对于当前会话的所有环境变量

```shell
env
```

* 通过 `env` 来指定运行时变量参数

```shell
env VAR="value" command_to_run command_options
```

* 通过 `printenv` 来列出所有环境变量，不分会话

```shell
printenv
```

* 通过 `printenv [VARIABLE]` 来打印指定变量的值

```shell
printenv HOME
```

* 通过 `printenv [VARIABLE1] [VARIABLE2]` 来打印多个变量的值

```shell
printenv HOME PWD
```

> 在大多数情况下，`env` 输出的环境变量应该与 `printenv` 输出的环境变量相同，除了 `_=` 变量，因为 `_=` 变量是一个特殊的 `bash` 参数，被用于调用 `shell` 脚本，使用 `env`，`_=` 会打印 `env` 的二进制运行目录：（`/usr/bin/env`），而使用 `printenv`，`_=` 会打印：（`/usr/bin/printenv`）

* 通过 `set` 来列出所有变量的值，包括：环境变量、`shell` 变量、`shell` 函数

```shell
set
```

* 如果使用 `set` 不想打印 `shell` 函数，可使用如下命令

```shell
(set -o posix; set)
```

* 通过 `declare` 来打印所有环境变量和 `shell` 变量

```shell
declare
```

* 如果仅仅显示环境变量或 `shell` 变量的名称，可使用 `compgen -v`

```shell
compgen -v
```

* 使用 `echo $[VARIABLE]` 来打印环境变量或 `shell` 变量

```shell
echo $PATH
```

* 以上列出变量列表的命令皆可使用通道传递到 `less` 命令同屏显示

```shell
env | less
printenv | less
set | less
```

* 引申而言，通道后面可以接任何其他命令做处理，如 `grep` 等

```shell
env | grep PWD
printenv | grep PWD
set | grep PWD
```

## 五、如何设置环境变量

* 使用 `export` 来设置单个变量值

```shell
export MY_VARIABLE=value

如果变量值有空格或特殊字符，需要用引号括起来，单引号、双引号皆可

export MY_VARIABLE="hello world!"
```

* 使用 `export` 来设置多个变量值

> 使用 `:` 来分割多个值

```shell
export MY_VARIABLE="value1:value2"
```

* 追加变量值到已存在的变量中

```shell
export PATH=$PATH:/abc
```

## 六、如何设置shell变量

* 没有 `export` 命令，直接使用键值对的方式

```shell
MY_VARIABLE="value"
```

## 七、如何把shell变量转换成环境变量

* 使用 `export`

```shell
export MY_VARIABLE
```

## 八、如何把环境变量降级为shell变量

* 使用 `export -n`

```shell
export -n MY_VARIABLE
```

## 九、如何删除环境变量

* 使用 `unset`

```shell
unset MY_VARIABLE
```

* 变量赋值为空字符串

```shell
MY_VARIABLE=""
```

## 十、登录与非登录shell会话的区别

* 登录 `shell` 对用户进行身份验证开始，如果登录到终端会话或通过 `SSH` 和身份验证，那么 `shell` 会话将被设置为登录 `shell`

* 如果在经过身份验证的会话中启动一个新的 `shell` 会话，就像从终端调用bash命令所做的那样，则会启动一个非登录的 `shell` 会话。在启动子 `shell` 时，不会要求提供身份验证详细信息

## 十一、交互式与非交互式shell会话的区别

* 交互式 `shell` 会话是附加到终端的 `shell` 会话
* 非交互式 `shell` 会话是不附加到终端会话的会话

> 以 `SSH` 开始的正常会话通常是一个交互式登录 `shell` ，从命令行运行的脚本通常在非交互式、非登录的 `shell` 中运行。终端会话可以是这两个属性的任意组合

## 十二、系统读取环境变量配置文件的顺序

* 一个登录 `shell` 会话首先读取 `/etc/profile` 配置文件，然后在当前登录的用户家目录依次查找读取 `~/.bash_profile`、`~/.bash_login`、`~/.profile`

* 一个非登录 `shell` 会话首先读取 `/etc/bash.bashrc` 配置文件，然后在当前登录的用户家目录查找读取 `~/.bashrc`
  
## 十三、系统级环境变量各配置文件的差异之处

* `/etc/environment` ：
  1. 此中设置的环境变量在所有进程和所有用户中都可用，不区分 `shell`
  2. 变量设置的格式使用简单的键值对：`KEY="value"`
  3. 此文件不是脚本，仅仅是配置文件

* `/etc/profile` ：
  1. 是登录 `shell` 和 交互式 `shell` 读取的配置文件，就是说在此文件添加或修改的内容需要再下一次登录时读取生效，或重启 `shell` 生效，且不影响非交互式的 `shell` 会话。
  2. 与 `/etc/environment` 不同，它是一个 `shell` 脚本文件，仅在用户登录时运行一次，职责是设置用户的环境和执行命令。
  3. 虽然它是一个 `bash shell` 脚本，但是 `zsh` 等其他 `shell` 也能够运行。
  4. `~/.bash_profile` 、`~/.bash_login`、`~/.profile` 用户级配置文件都来源自 `/etc/profile`
  5. `/etc/profile` 文件中加载了 `/etc/profile.d` 目录，所以在 `/etc/profile.d` 添加的配置都会被引入到 `/etc/profile` 中

  ```shell
  if [ -d /etc/profile.d ]; then
    for profile_file in /etc/profile.d/*.sh; do
        [ -r "$profile_file" ] && . "$profile_file"
    done
    unset profile_file
  fi
  ```
* `/etc/bash.bashrc`
  1. 是系统级别的非登录 `bash shell` 初始化脚本文件
  2. 在每一次交互式 `bash shell` 时被执行
  3. 通常用于设置 `bash` 指定的配置和别名
  4. 只用于 `bash` `shell`
  5. 用户级的 `~/.bashrc` 来源于此

## 十四、用户级环境变量各配置文件的差异之处

* `~/.bash_profile` ：
  1. 被登录 `shell` 执行
  2. 如果此文件存在，则忽略 `~/.bash_login` 和 `~/.profile` 文件
  3. 用于设置环境变量和执行任务，在登录过程中仅仅执行一次

* `~/.bash_login` ：
  1. 与 `~/.bash_profile` 相似
  2. 如果 `~/.bash_profile` 不存在，则执行此文件
  3. 通常情况下使用 `~/.bash_profile` 较多

* `~/.profile` ：
  1. 如果 `~/.bash_profile` 和 `~/.bash_login` 都不存在，则执行此文件
  2. 非 `bash` 指定的文件，更通用

* `~/.bashrc` ：
  1. 被用于执行交互式非登录 `shell` 
  2. 一般用于设置环境变量、设置别名、定义函数等
  3. 此文件被 `~/.bash_profile` 或 `~/.bash_login` 加载

* 总结：
  1. `~/.bash_profile` 、`~/.bash_login` 、`~/.profile` 三者用于执行登录 `shell`
  2. `~/.bashrc` 用于执行交互式非登录 `shell`
  3. `~/.bash_profile` 和 `~/.bash_login` 加载 `~/.profile`      
  4. `~/.bash_profile` 或 `~/.bash_login` 加载 `~/.bashrc`
  5. `~/.bash_profile` 、`~/.bash_login` 、`~/.profile` 三者通常用于设置系统范围的配置
  6. `~/.bashrc` 通常用于用户指定的配置

## 十五、如何让环境变量永久生效

* 在 `/etc/environment` 中写入保存：
  1. 写入键值对
  ```shell
  FOO=bar
  ```
  2. 重新登录后生效

* 或在 `/etc/profile` 中写入保存：
  1. 使用 `export` 格式
  ```shell
  export PATH=$PATH:/abc
  ```
  2. 重新登录生效

* 或在 `~/.bashrc` 中写入保存：
  1. 使用 `export` 格式
  ```shell
  export PATH=$PATH:/abc 
  ```
  2. 使用 `source ~/.bashrc` 立即加载生效

* 或使用 `echo` 直接追加内容到文件中：

```shell
echo export PATH=$PATH:/abc >> ~/.bashrc
```

## 十六、如何让环境变量永久删除

直接在配置文件中删除对应的配置项即可

## 十七、为什么配置文件命名为 `**.rc` 、`**.d` ？

* 例如 `.bashrc` 全称是：`Bourne Again SHell run commands` ，即rc代表的是 `run command`

* 例如 `.profile.d` ，`d` 代表的是 `directory` ，即目录的意思，一般设置环境变量在 `.profile.d` 文件夹中添加、修改即可，不用维护 `.profile` 文件