### 简介

`Zsh` (Z Shell) 是一种基于 `Bash` 的增强型命令行 `shell` 和脚本语言，可提供更多功能、自定义和易用性。它因其灵活性、强大的插件和主题（例如通过 `Oh-My-Zsh`）而广受欢迎。

### 特性

* 命令自动完成：文件、命令甚至选项的高级自动完成。

* 定制：支持主题和广泛的配置。 

* 插件支持：使用插件轻松扩展功能。

* 改进的历史记录管理：在会话之间共享命令历史记录。

* 拼写纠正：自动纠正小的打字错误。

* Glob Patterns：比 Bash 更强大的通配符匹配。

### 安装

* `Ubuntu/Debian`

```shell
sudo apt update
sudo apt install zsh
```

* `CentOS/Red Hat`

```shell
sudo yum install zsh
```

* `MacOS`

```shell
brew install zsh
```

设置 `zsh` 为默认的 `shell`

```shell
chsh -s $(which zsh)
```

### 基础用法

#### 启动 `zsh`

```shell
zsh
```

#### 查看 `zsh` 版本

```shell
zsh --version
```

#### 配置 `zsh`

* 配置别名

```shell
alias ll='ls -la'
alias gs='git status'
```

* 设置环境变量

```shell
export PATH=$PATH:/custom/path
```

* 启用命令自动纠正

```shell
setopt CORRECT
```

* 启用大小写不敏感自动补全

```shell
setopt NO_CASE_GLOB
```

#### 编写 `zsh` 脚本

```shell
#!/bin/zsh
echo "Hello from Zsh!"
```

#### 调试 `zsh` 脚本

```shell
zsh -x <script>.zsh

# 会在执行之前打印每个命令
```

### 通过 `Oh-My-Zsh` 增强 `zsh`

`Oh-My-Zsh` 是一个流行的用于管理主题和插件的 `zsh` 框架。

#### 安装 `Oh-My-Zsh`

```shell
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

#### 设置 `zsh` 主题

编辑 `~/.zshrc` 配置文件

```shell
ZSH_THEME="agnoster"

# 流行的主题有：robbyrussell, agnoster, powerlevel10k 等
```

#### 启用插件

编辑 `~/.zshrc` 配置文件

```shell
plugins=(git zsh-autosuggestions zsh-syntax-highlighting)
```

#### 安装插件

* `Zsh-Autosuggestions` 插件

```shell
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

* `Zsh-Syntax-Highlighting` 插件

```shell
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

#### 使 `zsh` 配置生效

```shell
source ~/.zshrc
```