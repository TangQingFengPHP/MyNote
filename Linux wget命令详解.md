#### 命令介绍

GNU Wget是一个免费的非交互式文件下载工具。它支持HTTP、HTTPS和FTP协议，以及通过HTTP代理进行检索。

Wget是非交互式的，这意味着它可以在后台下载，当用户未登录时。这允许你启动检索并断开与系统的连接，让Wget完成工作。相比之下，大多数Web浏览器都需要持续的用户介入，当传输大量数据，这可能是一个很大的障碍。

Wget命令用来从指定的URL下载文件。Wget非常稳定，它在带宽很窄的情况下和不稳定网络中有很强的适应性，如果是由于网络的原因下载失败，Wget会不断的尝试，直到整个文件下载完毕。如果是服务器打断下载过程，它会再次联到服务器上从停止的地方继续下载。这对从那些限定了链接时间的服务器上下载大文件非常有用。

#### 基本用法

```
wget https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```

#### 指定保存的文件名

> `-O`或`--output-document`

```
wget -O node.pkg https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```

#### 指定保存的目录

> `-P`或`--directory-prefix`

```
wget -P documents/archives/ https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```

#### 后台下载

> `-b`或`--background`

```
wget -b https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg

默认下载完成后，把输出结果会写入wget默认的日志文件（生成在当前目录）：wget-log，可使用-o 自定义日志文件
```

#### 指定后台下载输出的日志文件名

> `-o`或`--output-file`

```
wget -o diylog -b https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```

#### 指定后台下载输出的日志文件名（追加写入）

> `a`或`--append-output`

```
wget -a diylog -b https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```

#### 断点续传

> `-c`或`--continue`

```
wget -c https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```

#### 限速下载

> `--limit-rate`

```
wget --limit-rate=300k https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg

可用的单位有：k、m，可使用小数，例如300.5k
```

#### 测试下载链接

> `--spider`

```
wget --spider https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg

当链接有效，则返回200ok，否则返回404等特定的错误信息
```

#### 同时下载多个文件（指定文件的方式）

> `-i`或`--input-file`

```
wget -i abc.txt

abc.txt文件中包含多个url，格式要求为每个url为一行
```

#### 同时下载多个文件（直接在命令中指定多个）

```
wget https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg https://nodejs.org/dist/v18.18.0/node-v18.18.0.pkg
```

#### 同时下载多个文件（链接是带有数字模式的）

```
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.1.{1..15}.tar.gz
```

#### 伪装代理名称下载

> `-U`或`--user-agent`

```
wget -U "Mozilla/5.0 (Windows; U; Windows NT 6.1; en-US) AppleWebKit/534.16 (KHTML, like Gecko) Chrome/10.0.648.204 Safari/534.16" https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```

#### 设置重试次数

> `-t`或`--tries`

```
wget -t 20 https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```

#### 下载ftp链接文件

> `--ftp-user`和`--ftp-password`

```
wget --ftp-user=YOUR_USERNAME --ftp-password=YOUR_PASSWORD ftp://example.com/something.tar
```

#### 下载需要授权的http链接文件

> `--http-user`和`--http-password`

```
wget --http-user=narad --http-password=password http://http.example.com/filename.tar.gz
```

#### 下载整个站点

```
wget --mirror --convert-links --page-requisites --no-parent -P documents/websites/ https://some-website.com

--mirror：递归下载
--convert-links：下载后，转换成本地的链接。
--page-requisites：下载所有必须的文件：CSS、JS、图片
–-no-parent：它确保不会检索层次结构之上的目录
```

#### 忽略SSL证书检查

> `--no-check-certificate`

```
wget --no-check-certificate https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.1.1.tar.gz
```

#### 进度条样式设置

> `--progress`

```
wget --progress=dot https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg

以上会显示点状进度条

可用的样式有：dot、bar，默认是进度条
```

#### 设置使用代理服务器下载

```
1. 在环境变量中配置
http_proxy=http://username:password@proxy.server.address:port/
https_proxy=http://username:password@proxy.server.address:port/

2. 在.wgetrc配置文件中设置
use_proxy = on
http_proxy = http://username:password@proxy.server.address:port/
https_proxy = http://username:password@proxy.server.address:port/

3. 在命令行中使用-e或--execute
wget -e use_proxy=yes -e http_proxy=http://proxy.server.address:port/ https://example.com
```

#### 不使用代理

> `--no-proxy`

```
wget --no-proxy https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```

#### 限制总下载文件大小

> `-Q`或`--quota`

```
wget -Q10k https://example.com/ls-lR.gz

可用的单位有：b、k、m

设置额度为：0或inf（infinite）则表示不限制

注意对单个文件无效
```

#### 显示版本信息

> `-V`或`--version`

```
wget -V
```

#### 显示帮助信息

> `-h`或`--help`

```
wget -h
```

#### 执行格式为.wgetrc配置文件的命令

> `-e`或`execute`

```
wget -e use_proxy=yes -e http_proxy=http://proxy.server.address:port/ https://example.com 
```

#### 开启debug模式

> `-d`或`--debug`

```
wget -d https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```

#### 静默模式

> `-q`或`--quiet`

```
wget -q https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg

不输出任何信息
```

#### 打印详细的信息

> `-v`或`--verbose`

```
wget -v https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg

默认是带有-v选项
```

#### 强制把读取的文件视作html

> `-F`或`--force-html`

```
wget -F a.txt
```

#### 不覆盖存在的文件

> `-nc`或`–-no-clobber`

```
wget -nc https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```

#### 不重新下载文件，除非服务器上的比本地的新

> `-N`或`--timestamping`

```
wget -N https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```

#### 下载文件，并打印服务器响应

> `-S`或`--server-response`

```
wget -S https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```

#### 设置响应超时的秒数

> `-T`或`--timeout`

```
wget -T 60 https://nodejs.org/dist/v20.8.0/node-v20.8.0.pkg
```