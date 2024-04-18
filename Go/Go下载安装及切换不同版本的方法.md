## 一、下载安装

[Go下载地址](https://go.dev/dl/)
![alt text](/images/go-switch-version-image.png)

Go提供了Windows、MacOS(ARM64) 和 MacOS(x86-64)、Linux版本，也可以下载源码自己编译安装。

> Linux && MacOS

* 下载压缩包

* 解压到指定目录，如：/usr/local

```shell
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.22.2.linux-amd64.tar.gz
```

* 把go的bin目录添加到环境变量

```shell
在 ~/.zshrc或~/.bashrc中添加如下行

export PATH=$PATH:/usr/local/go/bin
```

* 使用环境变量立即生效

```shell
source ~/.zshrc 或 source ~/.bashrc
```

* 测试go是否安装成功

```shell
go version
```

> Windows

* 直接双击 .msi文件进行引导安装

* 安装完成后打开 cmd 或 power shell 输入 go version 验证安装是否成功

## 二、如何切换Go版本？

### 方法一：直接下载不同版本的压缩包，使用软连接指向不同的版本

* 下载完压缩包，解压到其他目录，如：

```shell
tar -C ~/Downloads -xzf go1.21.9.linux-amd64.tar.gz
```

* 把go目录重命名为 go1.21.9

```shell
mv go go1.21.9
```

* 新建一个go全局目录，如：go_version

```shell
mkdir ~/go_version
```

* 做一个软链接指向go1.21.9版本

```shell
ln -s ~/Downloads/go1.21.9 ~/go_version/go
```

* 把~/go_version/go/bin目录加到到环境变量

```shell
export PATH=~/go_version/go/bin:$PATH
```

* 验证是否安装成功

```shell
go1.21.9 version
```

扩展：此法为切换软件的通法，其他软件也适用。

### 方法二：使用go install 命令安装其他版本

已经安装go的情况下(例如当前版本为：1.22.2)，可以通过go install 来安装其他版本

```shell
go install golang.org/dl/go1.22.1@latest

go1.22.1 download
```

go install 命令会把go1.22.1版本作为1.22.2的可执行安装包，存放在 ~/go/bin下面

![alt text](/images/go-switch-version-image-1.png)

再去用go1.22.1 download 则会下载1.22.1的源码，放到~/sdk下面

![alt text](/images/go-switch-version-image-2.png)

此时就可以用go1.22.1 version来验证是否安装成功了

为什么可以直接敲go1.22.1呢？实际上执行的是~/go/bin/go1.22.1这个二进制文件，而~/go/bin又加入了PATH变量，所以能执行。

![alt text](/images/go-switch-version-image-3.png)

sdk是不能删除的，go1.22.1会读取sdk里面的源码，删除后会提示sdk没有下载。

### 方法三：使用gvm来切换

gvm全称：Go Version Manager (GVM)是一个用于管理Go环境的开源工具。它支持安装多个Go版本，并使用GVM "pkgsets" 管理每个项目的模块。GVM(与Ruby中的RVM一样)最初是由Josh Bussdieker开发的，它允许为每个项目或项目组创建开发环境，分离不同的Go版本和包依赖关系，以提供更大的灵活性并防止版本问题。

> 安装gvm

```shell
bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)

命令解析：

-s -S 连用表示禁用进度条但可以显示错误信息

-L 表示重定向跟随

<(...) 此处为<(curl ...) ，此语法表示curl的执行结果被视作文件

bash < 表示拿到curl的执行结果作为标准输入传给bash
```

> 通过gvm安装go

```shell
gvm install go1.22.2
```

> 通过gvm切换go版本

```shell
gvm use go1.22.2
```

> 列出所有通过gvm安装的go版本

```shell
gvm list
```

> 列出所有可用的线上go版本

```shel
gvm listall
```

> 卸载go版本

```shell
gvm uninstall go1.22.2
```

> 完全移除gvm及其所有安装的go版本和依赖包

```shell
gvm implode

如果卸载失败，直接 rm -rf ~/.gvm
```

> 管理go的依赖包

pkgset允许独立管理不同的Go包集及其版本，从而更容易在不同的项目依赖关系之间切换。

```shell
// 创建包集合
gvm pkgset create [name]

// 选择包集合
gvm pkgset use [name]

// 列出创建的包集合
gvm pkgset list

// 删除包集合
gvm pkgset delete [name]
```

切换到指定的包集后，后续使用go build、go run命令时会把下载的包安装到包集目录

![alt text](/images/go-switch-version-image-4.png)


> 其他gvm命令

* 打印gvm版本

```shell
gvm version
```

* 获取gvm最新版本

```shell
gvm get
```

* 打印帮助信息

```shell
gvm help
```

> gvm 原理

其内部核心也是使用软连接，通过指向不同的版本使用不同的环境变量

后面再进行源码分析



