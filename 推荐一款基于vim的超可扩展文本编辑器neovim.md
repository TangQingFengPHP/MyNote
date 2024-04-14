## 一、简介
Vim是一个基于流行的Vi编辑器的文本编辑器，最初是在20世纪70年代发布的。Vim代表“改进的Vi”，它拥有广泛的用户基础和广泛的可用插件和扩展。 
 
Neovim是Vim的一个分支，创建于2014年，旨在解决Vim的一些缺点，并提供额外的特性和功能。Neovim向后兼容Vim，并支持Vim的大部分功能。

## 二、neovim特性

* 相比 Vim 改进了性能和稳定性

* 支持异步插件和脚本

* 改进了对现代用户界面框架和 Unicode 字符的支持

* 更好的终端集成和 UI 支持

* 支持对代码的本机调试和分析支持

* 改进了对 Lua 脚本的支持

## 三、安装neovim

> 由于neovim跨平台，可在Windows、Linux、MacOS系统上安装。

### Windows上安装

仅支持Win8+

* 使用Winget安装

```shell
winget install Neovim.Neovim
```

* 使用Chocolatey安装

```shell
choco install neovim
```

### MacOS上安装

* 使用Homebrew安装

```shell
brew install neovim
```

* 使用MacPorts安装

```shell
sudo port selfupdate
sudo port install neovim
```

* 直接下载压缩包安装

> x86_64版本的

```shell
curl -LO https://github.com/neovim/neovim/releases/download/nightly/nvim-macos-x86_64.tar.gz

tar -xzvf nvim-macos-x86_64.tar.gz

sudo mv nvim-macos-x86_64 /usr/local/

cd /usr/local

sudo mv nvim-macos-x86_64 nvim

echo 'export PATH="/usr/local/nvim/bin:$PATH"' >> ~/.zshrc
```

> arm64版本的

```shell
curl -LO https://github.com/neovim/neovim/releases/download/nightly/nvim-macos-arm64.tar.gz

tar -xzvf nvim-macos-arm64.tar.gz

sudo mv nvim-macos-arm64 /usr/local/

cd /usr/local

sudo mv nvim-macos-arm64 nvim

echo 'export PATH="/usr/local/nvim/bin:$PATH"' >> ~/.zshrc
```

### Linux上安装

> 直接下载压缩包安装

```shell
curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim-linux64.tar.gz

tar -xzvf nvim-linux64.tar.gz

sudo mv nvim-linux64 /usr/local/

cd /usr/local

sudo mv nvim-linux64 nvim

echo 'export PATH="/usr/local/nvim/bin:$PATH"' >> ~/.zshrc
```

> 基于CentOS系的使用yum安装

```shell
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

yum install -y neovim python3-neovim
```

> 基于Debian系的使用apt-get安装

```shell
sudo apt-get install neovim
```

> 基于Fedora系的使用dnf安装

```shell
sudo dnf install -y neovim python3-neovim
```

## 四、配置neovim

Vim采取硬编码的路径来存储插件和配置文件。通常是~/.vim目录。尽管更改这个硬编码路径并非不可能，但与NeoVim如何构建其配置目录相比，仍然需要做很多工作。 
 
NeoVim遵循XDG基本目录规范。遵循此规范的程序将其配置文件存储在由XDG_CONFIG_HOME环境变量指定的目录中。按照惯例，它通常指向~/.config目录。 
 
因此，NeoVim将所有插件和配置文件存储在~/.config/nvim目录，使其符合XBD规范。

### 可以直接复用原vim配置

直接做一个软链接链到neovim的配置

```shell
先在~/.config下创建nvim目录

cd ~/.config

mkdir nvim

ln -s ~/.vimrc ~/.config/nvim/init.vim

做一个判断设置不同的命令
if has('nvim')
    " NeoVim specific commands
else
    " Standard Vim specific commands
endif
```

### 流行的插件

* LazyVim：简化Neovim的配置。

* CoC.nvim：是Neovim的语言服务器协议客户端，它为各种编程语言提供代码补全、语法高亮显示和错误检查。

* Vim-Plug：是一个流行的 Vim 插件管理器，但它也适用于 Neovim。它可以轻松安装和管理 Neovim 插件，并支持延迟加载和自动更新等功能。

* nvim-tree.lua：是一个用于Neovim的文件系统资源管理器，它提供了项目目录结构的树状视图。它支持基本的文件管理功能，如创建、删除和重命名，并可以自定义各种图标和主题。

* nvim-telescope：这是一个高度可扩展的列表模糊查找器。

* nvim-treesitter：提供了一种简单的方法来使用Neovim中的tree-siter，还提供了高亮显示等功能。

### 如何安装配置插件

使用vim-plug来便利安装

#### 安装vim-plug

> Linux、MacOS

```shell
sh -c 'curl -fLo "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/autoload/plug.vim --create-dirs \
       https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim'
```

> Windows

```shell
iwr -useb https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim |`
    ni "$(@($env:XDG_DATA_HOME, $env:LOCALAPPDATA)[$null -eq $env:XDG_DATA_HOME])/nvim-data/site/autoload/plug.vim" -Force
```

#### 使用示例

在call plug#begin()和call plug#end()中间列出所要安装的插件即可，如下：

```shell
call plug#begin()

" 默认的插件存储目录:
"   - Vim (Linux/macOS): '~/.vim/plugged'
"   - Vim (Windows): '~/vimfiles/plugged'
"   - Neovim (Linux/macOS/Windows): stdpath('data') . '/plugged'

" 可以在begin里面指定插件的目录

"   - e.g. `call plug#begin('~/.vim/plugged')`
"   - 避免使用vim标准目录名：plugin
"   - 必须使用单引号包裹

" Shorthand notation for GitHub; translates to https://github.com/junegunn/vim-easy-align
Plug 'junegunn/vim-easy-align'

" Any valid git URL is allowed
Plug 'https://github.com/junegunn/seoul256.vim.git'

" Using a tagged release; wildcard allowed (requires git 1.9.2 or above)
Plug 'fatih/vim-go', { 'tag': '*' }

" Using a non-default branch
Plug 'neoclide/coc.nvim', { 'branch': 'release' }

" Use 'dir' option to install plugin in a non-default directory
Plug 'junegunn/fzf', { 'dir': '~/.fzf' }

" Post-update hook: run a shell command after installing or updating the plugin
Plug 'junegunn/fzf', { 'dir': '~/.fzf', 'do': './install --all' }

" Post-update hook can be a lambda expression
Plug 'junegunn/fzf', { 'do': { -> fzf#install() } }

" If the vim plugin is in a subdirectory, use 'rtp' option to specify its path
Plug 'nsf/gocode', { 'rtp': 'vim' }

" On-demand loading: loaded when the specified command is executed
Plug 'preservim/nerdtree', { 'on': 'NERDTreeToggle' }

" On-demand loading: loaded when a file with a specific file type is opened
Plug 'tpope/vim-fireplace', { 'for': 'clojure' }

" Unmanaged plugin (manually installed and updated)
Plug '~/my-prototype-plugin'

" Initialize plugin system
" - Automatically executes `filetype plugin indent on` and `syntax enable`.
call plug#end()
" You can revert the settings after the call like so:
"   filetype indent off   " Disable file-type-specific indentation
"   syntax off            " Disable syntax highlighting
```

#### 常用的命令

* PlugInstall：安装插件

* PlugUpdate：安装或更新插件

* PlugClean：移除未列出的插件

* PlugUpgrade：更新vim-plug插件自身

* PlugStatus：检查插件状态

vim-plug地址：

![alt text](/images/nvim-image-1.png)

### 注意

敲命令的时候使用nvim

![alt text](/images/nvim-image.png)