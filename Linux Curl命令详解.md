#### 简介：

1. 简称：Client URL<br>

2. curl是一个用于从服务器传输数据或向服务器传输数据的工具。它 
支持这些协议:DICT, FILE, FTP, FTPS, GOPHER, gopers， 
Http、https、imap、imaps、ldap、ldaps、mqtt、pop3、pop3s、rtmp、 
rtmp、rtsp、scp、sftp、smb、smbs、smtp、smtps、telnet、tftp、ws 
WSS。该命令被设计为无需用户即可工作 交互。<br>

3. curl的所有传输相关功能都由libcurl提供支持

4. curl把接收到的数据写入到stdout（标准输出）

5. curl默认不对数据的请求和接受进行加密或解密

6. 所有boolean选项都有--no-option相反的选项

7. 所有可以指定文件的选项，如果接 `-` 则表示从stdin（标准输入）中读取

#### 选项：

* `--abstract-unix-socket <path>`

```
(HTTP)通过抽象Unix域套接字连接， 而不是使用网络
```

* `-a, --append `

```
在使用ftp上传文件时，追加内容到目标文件中，而不是覆盖，如果远程文件不存在则创建

示例：

curl --upload-file local --append ftp://example.com/
```

* `-K, --config <file>`

```
指定一个文件，用于curl读取作为选项参数

示例：curl --config file.txt https://example.com

file.txt内容如下：

# --- Example file ---
# this is a comment
url = "example.com"
output = "curlhere.html"
user-agent = "superagent/1.0"

# and fetch another URL too
url = "example.com/docs/manpage.html"
-O
referer = "http://nowhereatall.example.com/"
# --- End of example file ---
```
* `--connect-timeout <fractional seconds>`

```
指定curl连接超时时间，单位是秒，可以指定小数

示例：

curl --connect-timeout 20 https://example.com
curl --connect-timeout 3.14 https://example.com
```
* `-C, --continue-at <offset>`

```
恢复被终止的传输进程，当下载文件时，遇到意外终止的请求，添加此选项可以继续下载，offset可指定从何处偏移下载，单位是k，如果offset不给值，则可以用-C - 让curl自动推断偏移的位置

示例：

curl -C - https://example.com
curl -C 400 https://example.com
```

* `-c, --cookie-jar <filename>`

```
请求完成后保存内存中的cookie

示例：

curl -c store-here.txt https://example.com
```

* `-b, --cookie <data|filename>`

```
请求时，传递cookie到http服务头

示例：

curl -b cookiefile https://example.com
```

* `--create-dirs`

```
当保存下载的文件时，可以指定保存的目录，如果目录不存在，则自动创建，文件默认的所有者权限是0750

示例：
curl --create-dirs -o local/dir/file https://example.com
```

* `--create-file-mode <mode>`

```
使用ftp上传文件时，带上此选项可以指定所有者权限，默认是0644

示例：

curl --create-file-mode 0777 -T localfile sftp://example.com/new
```

* `--data-binary <data>`

```
传输二进制数据到服务器，以@开头的表示是文件

curl --data-binary @filename https://example.com
```

* `--data-raw <data>`

```
原样传输数据，不解析、不转义

示例：

curl --data-raw "hello" https://example.com
curl --data-raw "@at@at@" https://example.com
```
* `--data-urlencode <data>`

```
请求的数据进行url encode

示例：

curl --data-urlencode name=val https://example.com

curl --data-urlencode name@file https://example.com

curl --data-urlencode @fileonly https://example.com
```
* `-d, --data <data>`

```
发送post请求到服务器，并指定请求的参数

示例：

curl -d "name=curl" https://example.com

curl -d "name=curl" -d "tool=cmdline" https://example.com

此处可以合并为：curl -d "name=curl&tool=cmdline" https://example.com

curl -d @filename https://example.com
```

* `-D, --dump-header <filename>`

```
把请求头信息写入指定的文件中

示例：

curl --dump-header store.txt https://example.com
```

* `-F, --form <name=content>`

```
发送http表单请求数据

示例：

上传文件
curl -F profile=@portrait.jpg https://example.com/upload.cgi

指定两个字段
curl -F name=John -F shoesize=11 https://example.com/

从文件中读取数据
curl -F "story=<hugefile.txt" https://example.com/

指定type类型
curl -F "web=@index.html;type=text/html" example.com

使用filename指定上传的文件名
curl -F "file=@localfile;filename=nameinpost" example.com

如果文件名包含逗号或分号，则必须使用双引号括住，并且用反斜杠转义
curl -F "file=@\"local,file\";filename=\"name;in;post\"" example.com

curl -F 'file=@"local,file";filename="name;in;post"' example.com

headers添加自定义头
curl -F "submit=OK;headers=\"X-submit-type: OK\"" example.com

自定义头从文件中获取
curl -F "submit=OK;headers=@headerfile" example.com
```

* `-G, --get`

```
使用get方法发送http请求，如果命令中使用了-d, --data, --data-binary or --data-urlencode，则请求参数全部放在url后面作为查询字符串

示例：

curl --get https://example.com

curl --get -d "tool=curl" -d "age=old" https://example.com

```

* `-g, --globoff`

```
关闭全局解析器，遇到正常需要解析转义的符号会当做普通的字符串

示例：

curl -g "https://example.com/{[]}}}}"
```

* `-I, --head`

```
仅输出请求头

示例：

curl -I https://example.com
```

* `-H, --header <header/@file>`

```
指定请求头，可以设置请求头为空来覆盖默认的请求头，例如：-H "Host:"

示例：

curl -H "X-First-Name: Joe" https://example.com

curl -H "User-Agent: yes-please/2000" https://example.com

curl -H "Host:" https://example.com

curl -H @headers.txt https://example.com
```

* `-h, --help <category>`

```
打印帮助信息

示例：

curl --help all
```

* `--json <data>`

```
指定http json请求，发送json数据

此选项是以下三种选项的组合：

--data [arg]
--header "Content-Type: application/json"
--header "Accept: application/json"

示例：

curl --json '{ "drink": "coffe" }' https://example.com
curl --json @prepared https://example.com
curl --json @- https://example.com < json.txt
```

* `--keepalive-time <seconds>`

```
发送心跳保持连接的时间间隔，单位：秒

示例：

curl --keepalive-time 20 https://example.com
```

* `--libcurl`

```
指定一个文件名，然后curl请求的过程会以C源码的方式写入文件

示例：

curl --libcurl client.c https://example.com
```

* `--limit-rate <speed>`

```
限制传输的速率，默认的单位是K，有K、M、G、T、P可用

示例：

curl --limit-rate 100K https://example.com

curl --limit-rate 1000 https://example.com

curl --limit-rate 10M https://example.com
```

* `-l, --list-only`

```
列出ftp的目录

示例：

curl --list-only ftp://example.com/dir/
```

* `-L, --location`

```
使curl请求到的页面如果被重定向，会自动跟随请求

示例：

curl -L https://example.com
```

* `--max-filesize <bytes>`

```
最大文件下载大小，可用的单位：K、M、G

示例：

curl --max-filesize 100K https://example.com
```

* `--max-redirs <num>`

```
最大重定向次数，与-L相结合使用

示例：

curl -L --max-redirs 3 https://example.com
```

* `-:, --next`

```
可以指定多个请求，每个请求可以用不同的请求方法，例如先用post请求，然后用get请求，此选项作用是重置当前的选项

示例：

curl www1.example.com --next -d postthis www2.example.com
```

* `-o, --output <file>`

```
下载文件时指定保存的文件名

示例：

一般用法
curl -o aa example.com

#1对应的是{one,two}这个变量，自动取名
curl "http://{one,two}.example.com" -o "file_#1.txt"

#1，#2分别对应{site,host} [1-5]
curl "http://{site,host}.host[1-5].com" -o "#1_#2"

抑制输出，重定向到垃圾桶中
curl example.com -o /dev/null
```

* `-#, --progress-bar`

```
显示以#符号的进度条，而不是详细输出

示例：

curl -# -O https://example.com
```

* `-x, --proxy [protocol://]host[:port]`

```
使用代理服务器请求，默认是使用http协议，当不指定端口时，默认端口是1080

示例：

curl -x 127.0.0.1:9090 https://example.com
```

* `-U, --proxy-user <user:password>`

```
使用代理服务器时，需要指定用户名和密码

示例：

curl --proxy-user name:pwd -x proxy https://example.com
```

* `--rate <max request rate>`

```
限制重试请求的速率，格式为：N/U，N表示请求的次数，U表示单位，可用的单位有：s(second), m(minute), h(hour), d(day)，默认的单位是每小时，此选项内部函数是使用毫秒级的方案，如果设置每秒超过1000次，则选项会变为不限制速率

示例：

curl --rate 2/s https://example.com

curl --rate 3/h https://example.com

curl --rate 14/m https://example.com
```

* `-e, --referer <URL>`

```
指定来源地址，如果指定为：";auto"，则curl会自动推断

示例：

curl --referer "https://fake.example" https://example.com

curl --referer "https://fake.example;auto" -L https://example.com

curl --referer ";auto" -L https://example.com
```

* `-O, --remote-name`

```
下载文件时，指定此选项，会使用远程文件的名称

示例：

curl -O https://example.com/filename
```

* `--retry <num>`

```
指定curl重试的次数

示例：

curl --retry 7 https://example.com
```

* `-s, --silent`

```
静默模式，不输出任何信息

示例：

curl -s https://example.com
```

* `--trace <file>`

```
打印完整的输入输出、错误等信息，可指定输出的文件

示例：

curl --trace log.txt https://example.com
```

* `-T, --upload-file <file>`

```
告诉curl当前是ftp上传文件

示例：

curl -T file https://example.com

curl -T "img[1-1000].png" ftp://ftp.example.com/
```

* `-A, --user-agent <name>`

```
指定http请求头的用户代理

示例：

curl -A "Agent 007" https://example.com
```

* `-v, --verbose`

```
打印详细的debug信息

示例：

curl -v https://example.com
```

* `-V, --version`

```
打印curl版本信息
```

* `-w, --write-out <format>`

```
使用curl预定义的变量，打印出输入输出操作过程的有用信息

示例：

curl -w '%{response_code}\n' https://example.com
```

* `-X, --request <method>`

```
指定http请求的方法，如：GET, POST, PUT, PATCH, DELETE

示例：

curl -X DELETE https://example.com

curl -X PUT localhost:8080/employees/3 -H 'Content-type:application/json' -d '{"name": "Samwise Gamgee", "role": "ring bearer"}'
``