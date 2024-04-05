## 一、tar

Linux中的tar命令是文件管理中最重要的命令之一。它是Tape Archive的缩写，用于创建和解压缩归档文件。存档文件是一种压缩文件，其中包含一个或多个捆绑在一起的文件，以便更易于访问存储和可移植性。

tar只负责打包，打包是指将一大堆文件或目录捆绑成一个文件；压缩则是将一个大的文件通过一些压缩算法变成一个小文件，需要用到zip、
gzip、bzip2、xz等。

### 常用选项

* `-c` ：创建打包文件，会递归目录中的每个文件，如果想改变此行为，可以指定 --no-recursion

* `-x` ：提取打包文件

* `-f` ：指定打包后的文件名

* `-v` ：在打包时打印详细debug信息

* `-t` ：列出在打包文件中的所有文件

* `-u` ：打包文件并添加到存在的打包文件中

* `-r` ：更新打包文件中的文件或目录

* `-z` ：打包文件时使用 gzip 进行压缩

* `-j` ：打包文件时使用 bzip2 进行压缩

* `-J` ：打包文件时使用 xz 进行压缩

* `W` ：验证打包文件是否被损坏

* `-A` ：追加打包文件到另一个打包文件

* `-d` ：比较打包文件与源文件，如果没有提供源文件参数，则默认使用当前目录

* `--delete` ：删除打包文件中的成员

* `--wildcards` ：搜索打包文件中的文件，通过通配符来匹配

* `-?, --help` ：打印帮助信息

* `--usage` ：打印可用的选项

* `--version` ：打印版本信息

### 命令示例

* 只打包不压缩

```shell
tar -cvf abc.tar abc
```

* 创建一个打包并压缩的文件

```shell
tar -czvf abc.tar.gz a.txt b.txt c.txt

解释：c：打包、z：使用gzip压缩、v：打印debug信息、f：指定打包的文件名
```

* 把文件夹进行打包并压缩

```shell
tar -czvf dir.tar.gz dir
```

* 列出压缩文件的内容

```shell
tar -tf abc.tar.gz
```

* 提取打包的文件

```shell
tar -xvf abc.tar
```

* 提取打包的文件到指定的目录

```shell
tar -xvf abc.tar -C /tmp/files
```

* 提取打包文件中指定的文件

```shell
tar -xvf abc.tar file1.txt file2.txt
```

* 添加/追加文件到打包文件

```shell
tar -rvf abc.tar file3.txt
```

* 删除打包文件中指定的成员文件

```shell
tar --delete -f abc.tar file3.txt
```

* 指定 gzip 算法进行压缩

```shell
tar -zcvf abc.tar.gz abc
```

* 解压缩 gzip 算法的压缩包

```shell
tar -zxvf abc.tar.gz
```

* 指定 bzip2 算法进行压缩

```shell
tar -jcvf abc.tar.bz2 abc
```

* 解压缩 bzip2 算法的压缩包

```shell
tar -jxvf abc.tar.bz2
```

* 指定 xz 算法进行压缩

```shell
tar -Jcvf abc.tar.xz abc
```

* 解压缩 xz 算法的压缩包

```shell
tar -Jxvf abc.tar.xz
```

* 通过通配符提取匹配到的文件

```shell
tar -xvf abc.tar --wildcards '*.txt'
```

* 验证压缩包是否完整或被损坏

```shell
tar -W abc.tar
```

* 创建压缩包时，指定排除的文件

```shell
tar -zcvf abc.tar.gz abc --exclude='file1.txt'
```

* 创建压缩包，指定排除的目录

```shell
tar -zcvf abc.tar.gz abc --exclude='/etc/*'
```

* 通过 grep 查找压缩包中匹配的文件

```shell
tar -tf abc.tar.gz | grep file1.txt
```

* 通过wildcards查找压缩包中的多个文件

```shell
tar -tf abc.tar.gz --wildcards '*.png'
```

* 合并打包文件

```shell
tar -Af abc.tar def.tar

def.tar 会合并到 abc.tar中，def.tar继续存在，不会消失。
```

* 比较打包文件和源文件

```shell
tar -df abc.tar abc
```

* 提取文件并保留源文件权限

```shell
tar xf abc.tar.gz --preserve-permissions
```

* 提取文件并把标准输出写入到外部程序

```shell
tar xf abc.tar --to-command='mkdir $TAR_FILENAME'

提取abc.tar，--to-command内部通过通道把标准输出传给指定的命令

此处mkdir会创建名字为abc的文件夹，文件夹里面有提取的每个文件。
```

* 检查打包或压缩包文件的大小

```shell
tar -czvf abc.tar.gz | wc -c
```

## 二、rar

rar文件是一种用于压缩和归档文件的流行格式，rar是Roshal Archive的缩写。尽管是专有格式，但选择默认rar实用程序文件类型而不是其他文件类型有几个原因:

* 采用无损数据压缩技术

* 支持将大档案拆分为更小的卷，简化存储和共享

* 便于存档注释和可定制的压缩设置

虽然与zip相比，rar表现出较慢的性能，但它具有更高的压缩率和优越的数据冗余。因此，rar文件平均节省更多的存储空间。

### 安装方法

* 基于Debian系的发行版

```shell
sudo apt install rar unrar
```

* 基于REHL系的发行版

```shell
sudo yum install rar unrar 或

sudo dnf install rar unrar
```

* 手动安装

```shell
wget https://www.rarlab.com/rar/rarlinux-[xxx].tar.gz

tar -zxvf rarlinux-x64-xxx.tar.gz

cd rar

sudo cp -v rar unrar /usr/local/bin/
```

下载地址：
![](/images/compress-1.png)

### 常用选项

* `a` ：创建压缩包，添加文件到压缩包

* `x` ：提取完整文件的完整路径，即保留原始目录结构

* `e` ：提取文件不保留原始路径，即所有文件平铺在一个目录

* `p` ：打印文件到标准输出

* `l` ：列出压缩包内容

* `t` ：测试压缩包完整性，有没有损坏

* `v` ：列出压缩包时打印debug信息

* `p` ：设置压缩包解压密码

* `-?` ：打印帮助信息

* `-r` ：创建压缩包时，递归添加文件夹中的文件

### 命令示例

* 创建压缩包 

```shell
rar a abc.rar file1.txt file2.txt
```

* 创建压缩包，递归添加文件夹的文件

```shell
rar a -r abc.rar file1.txt file2.txt ~/dir
```

* 分割大尺寸的压缩包为多个指定大小的压缩包

```shell
rar a -v50M abc.rar file1.txt fil2.txt file3.txt

-v50M指定每个分割的压缩包大小为50兆

此时会分割成如此压缩包：abc.part1.rar、abc.part2.rar、abc.part3.rar等
```

* rar压缩包设置密码

```shell
rar a -p abc.rar file1.txt file2.txt

命令输入完毕会提示输入密码
```

* 加密rar压缩包

```shell
rar a -hp abc.rar file1.txt file2.txt
```

* 压缩包解压到当前目录，所有文件平铺在一个目录

```shell
unrar e abc.rar
```

* 压缩包解压到当前目录，并保留原始目录结构

```shell
unrar x abc.rar
```

* 压缩包解压到指定目录

```shell
unrar e abc.rar -o ~/tmp
```

* 解压缩设置密码的压缩包

```shell
unrar e abc.rar -p [password]

-p 后面接密码
```

* 测试压缩包的完整性、有没有损坏

```shell
unrar t abc.rar
```

* 列出压缩包内容

```shell
unrar l abc.rar

会显示压缩包中文件的属性、大小、日期、时间、权限信息。
```

* 删除压缩包中指定的文件

```shell
rar d abc.rar file1.txt file2.txt

此操作会直接修改原始压缩包
```

* 修复压缩包文件

```shell
rar r abc.rar
```

* 添加文件/更新压缩包的文件

```shell
rar u abc.rar file3.txt
```

* 压缩包加锁

```shell
rar k abc.rar

加锁后压缩包不能被修改
```

## 三、7z

7zip是一个免费的开源文件压缩器，类似于Windows上的WinZip或WinRAR。它是由Igor Pavlov开发的，可用于Windows, Linux和macOS。7zip的一个主要优点是能够将文件压缩到很高的程度，这可以节省大量的磁盘空间。它还支持多种文件格式，包括它自己的7z格式，以及ZIP、TAR和其他格式。

### 安装

* 基于Debian系的发行版

```shell
sudo apt install -y p7zip-full
```

* 基于RHEL系的发行版

```shell
sudo yum install p7zip p7zip-plugins -y 或

sudo dnf install p7zip p7zip-plugins -y 或

sudo apt install p7zip-rar -y 

p7zip-rar 可以处理rar格式
```

下载地址：
![](/images/compress-2.png)

### 常用选项

* `a` ：添加文件到压缩包

* `d` ：删除压缩包中指定的文件

* `e` ：解压缩，不保留原始目录结构，提取的文件会平铺在同一个目录

* `x` ：解压缩，保留原始目录结构

* `l` ：列出压缩包内容

* `t` ：测试压缩包完整性、是否损坏

* `u` ：更新压缩包文件

* `-o` ：指定解压缩后的目录

* `-p` ：压缩包设置密码

* `-t[type]` ：设置压缩包的格式类型，例如：zip、gzip、bzip2、xz，默认是自己的格式7z

* `-x` ：压缩排除文件和解压缩提取排除文件

### 命令示例

* 创建压缩包

```shell
7z a abc.7z file1.txt file2.txt
```

* 压缩包解压，不保留原来的目录结构，平铺在同一个目录

```shell
7z e abc.7z
```

* 压缩包解压，保留原始目录结构

```shell
7z x abc.7z
```

* 压缩包解压，保留原始目录结构，并指定解压到的目录

```shell
7z x abc.7z -o /tmp/abc
```

* 指定压缩级别

```shell
7z a -m0=lzma2 abc.7z file1.txt file2.txt

7z a -m9=lzma2 abc.7z file1.txt file2.txt

级别从0到9，数字越小，速度越快，压缩率越低，数字越大，两者就调换过来
```

* 压缩一个目录为压缩包

```shell
7z a abc.7z ~/abc
```

* 压缩添加密码并使用算法加密

```shell
7z a -p[password] -mhe=on abc.7z file1.txt file2.txt

-p 后面填自定义密码

-mh2=on 表示开启加密

注意：密码一旦忘记，文件就解压不开，不可恢复
```

* 分割压缩包

```shell
7z a -v1m abc.7z file1.txt file2.txt

-v指定每个压缩包的大小，1m为1兆

压缩后，压缩包如：abc.7z.001，abc.7z.002

后面解压缩的时候，只需要解压abc.7z.001，7z会自动检测其他压缩包部分并解压
```

* 添加文件到存在的压缩包

```shell
7z u abc.7z file3.txt
```

* 创建其他格式的压缩包

```shell
7z a -ttar abc.tar.7z file1.txt file2.txt

通过 -t指定格式
```

* 从压缩包中提取指定的文件

```shell
7z x abc.7z file2.txt
```

* 压缩包添加密码

```shell
7z a -p[password] abc.7z file1.txt file2.txt
```

* 列出压缩包内容

```shell
7z l abc.7z

输出文件的名称、大小、压缩比率等
```

* 解压缩时显示进度条

```shell
7z x -bsp1 abc.7z
```

* 压缩时排除指定的文件

```shell
7z a abc.7z -x!*.log -x!temp/

排除了以log后缀的文件和temp目录
```

* 解压缩时排除指定的文件

```shell
7z x abc.7z -x!*.log -x!temp/ 
```

* 创建自解压的压缩包

自解压即不需要在目标机器安装7zip，打包后的压缩包内部包含了7zip程序

```shell
7z a -sfx abc.exe file1.txt file2.txt

exe后缀也可以解压
```

* 解压自解压的压缩包

```shell
./abc.exe

直接运行此压缩包即可
```

* 测试压缩包的完整性、是否损坏

```shell
7z t abc.7z
```

* 删除压缩包中的指定的文件

```shell
7z d abc.7z file2.txt
```

* 如果源文件变更了，想重新压缩的便捷方式

```shell
7z u abc.7z

系统会检测文件的变更，然后更新压缩包
```

完结撒花

搞懂以上即通关了！







