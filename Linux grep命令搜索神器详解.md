## 一、命令介绍

Grep是“全局正则表达式打印”的缩写（global regular expression print），是一个用于搜索和匹配正则表达式中包含的文件中的文本模式的命令。此外，每个Linux发行版都预装了该命令。

可以使用通用正则表达式语法搜索和过滤文本。它无处不在，以至于动词“grep”已经成为“搜索”的同义词

## 二、语法

```shell
grep [options] pattern [FILE]
```

* grep：命令本身
* [options]：命令修饰符
* pattern：要找到的搜索查询
* [FILE]：命令将要搜索的文件

示例：`grep -i abc output.txt`

如果`FILE`是`-`，则从标准输入中读取数据（不递归），如果没有提供`FILE`，则在当前目录递归搜索。

## 三、常用选项

> 通用程序信息

```shell
--help：输出帮助信息
-V, --version：输出版本信息
```

> 模式语法

```shell
-E, --extended-regexp：把模式作为扩展的正则表达式
-F, --fixed-strings：把模式作为固定的字符串
-G, --basic-regexp：把模式作为正常的正则表达式，此为默认行为
-P, --perl-regexp：把模式作为`Pecl`兼容的正则表达式
```

> 匹配控制

```shell
-e [PATTERNS], --regexp=[PATTERNS]：指定给出的正则表达式字符串，此选项可使用多次，则会搜索给出的所有表达式，可结合`-f`一起使用

-f [FILE], --file=FILE：从文件中获取正则表达式，文件中每一行为一个正则表达式，此选项可使用多次，可结合`-e`一起使用，如果文件中包含0个表达式，则不匹配任何内容，如果[FILE]给出的是`-`，则从标准输入中读取数据

-i, --ignore-case：忽略大小写

---no-ignore-case：区分大小写，如果已经使用了`-i`，则使用此选项会取消`-i`的效果，两个选项会彼此覆盖

-v, --invert-match：反转匹配，即查找未匹配到的行

-w, --word-regexp：仅仅选择匹配到的包含整个单词的行

-x, --line-regexp：仅仅整行匹配
```

> 通用输出控制

```shell
-c, --count：抑制正常的输出，打印匹配到行的数量，如果接`-v, --invert-match`则统计未匹配到的数量

--color[=WHEN], --colour[=WHEN]：输出带颜色，WHEN有三个选项：never, always, auto

-L, --files-without-match：抑制正常的输出，打印未匹配到的文件名

-l, --files-with-matches：抑制正常的输出，打印匹配到的文件名

-m[NUM], --max-count=NUM：指定匹配查找读取的行数，如果[NUM]为0，则会立即停止读取，默认值为-1，表示无限读取

-o, --only-matching：仅仅打印匹配到的行

-q, --quiet, --silent：不输出任何东西，静音模式

-s, --no-messages：抑制错误信息，关于不存在或不可读的文件
```

> 输出行前缀控制

```shell
-b, --byte-offset：打印匹配到的内容之前基0的字节偏移内容

-H, --with-filename：打印匹配到的文件名，当指定超过一个文件时，此为默认行为

-h, --no-filename：抑制输出文件名，当仅指定一个文件时，或数据来源于标准输出时，此为默认行为

-n, --line-number：输出匹配行的基1的行号
```

> 上下文行控制

```shell
-A[NUM], --after-context=[NUM]：在匹配到的字符串之后打印指定的行数

-B[NUM], --before-context=[NUM]：在匹配到的字符串之前打印指定的行数

-C[NUM], -[NUM], --context=[NUM]：在匹配到的字符串之前和之后打印指定的行数

--group-separator=[SEP]：当使用`-A`, `-B`, `-C`时，使用指定的分隔符代替默认的`--`

--no-group-separator：当使用`-A`, `-B`, `-C`时，不显示分隔符
```

> 文件和目录选择

```shell
-a, --text：解析二进制文件作为一个正常文本，等价于`--binary-files=text`

-D[ACTION], --devices=[ACTION]：如果输出的文件是一个设备文件、FIFO或socket，则会使用[ACTION]来解析它，默认情况[ACTION]是read，把设备文件当做普通文件，如果[ACTION]是`skip`，设备文件将会被跳过

-d[ACTION], --directories=[ACTION]：如果输入的文件是一个目录，则会使用[ACTION]来解析它，默认情况[ACTION]是read，把目录当做普通文件，如果[ACTION]是`skip`, 则会跳过此目录，如果[ACTION]是`recurse`，则会在每一个目录下面读取所有文件

--exclude=[GLOB]：跳过匹配到文件名的文件

--exclude-from=[FILE]：从文件中读取匹配模式：跳过匹配到文件名的文件

--exclude-dir=[GLOB]：跳过匹配到的目录，当搜索时递归时，也会跳过子目录

--include=[GLOB]：从匹配到的文件中搜索

-r, --recursive：在给出的每个目录下递归读取所有的文件，与`-d recurse`等价
```

## 四、可用的正则表达式语法

```shell
?    # 前一项是可选的，最多匹配一次
^    # 锚定行的开始 如：'^grep'匹配所有以grep开头的行。    
$    # 锚定行的结束 如：'grep$' 匹配所有以grep结尾的行。
.    # 匹配一个非换行符的字符 如：'gr.p'匹配gr后接一个任意字符，然后是p。    
*    # 匹配零个或多个先前字符 如：'*grep'匹配所有一个或多个空格后紧跟grep的行。    
.*   # 一起用代表任意字符。   
[]   # 匹配一个指定范围内的字符，如'[Gg]rep'匹配Grep和grep。    
[^]  # 匹配一个不在指定范围内的字符，如：'[^A-Z]rep' 匹配不包含 A-Z 中的字母开头，紧跟 rep 的行
\(..\)  # 标记匹配字符，如'\(love\)'，love被标记为1。    
\<      # 锚定单词的开始，如:'\<grep'匹配包含以grep开头的单词的行。    
\>      # 锚定单词的结束，如'grep\>'匹配包含以grep结尾的单词的行。    
x\{m\}  # 重复字符x，m次，如：'0\{5\}'匹配包含5个o的行。    
x\{m,\}   # 重复字符x,至少m次，如：'o\{5,\}'匹配至少有5个o的行。    
x\{m,n\}  # 重复字符x，至少m次，不多于n次，如：'o\{5,10\}'匹配5--10个o的行。   
\w    # 匹配文字和数字字符，也就是[A-Za-z0-9]，如：'G\w*p'匹配以G后跟零个或多个文字或数字字符，然后是p。   
\W    # \w的反置形式，匹配一个或多个非单词字符，如点号句号等。   
\b    # 单词锁定符，如: '\bgrep\b'只匹配grep。  
```

## 六、应用实例

* 一般用法

```shell
grep abc message.log
```

* 匹配的结果添加颜色显示

```shell
grep --color abc message.log
```

* 递归搜索

```shell
grep -r abc /ssh
```

* 忽略大小写

```shell
grep -i abc message.log
```

* 统计匹配到的总行数

```shell
grep -c abc message.log
```

* 反转搜索，匹配不包含的行

```shell
grep -v abc message.log
```

* 打印匹配到的字符串行号

```shell
grep -n abc message.log
```

* 仅仅匹配包含整个单词的数据

```shell
grep -w linux message.log
```

* 结合其他命令一起使用，通过通道传递过来

```shell
dpkg -L | grep -i openssh

说明：以上搜索`openssh`包
```

* 打印匹配行之前或之后行的内容

```shell
grep abc message.log -A 3
grep abc message.log -B 3
grep abc message.log -C 3
```

* 使用正则表达式来匹配

```shell
grep ^abc message.log

grep abc$ message.log
```

* 打印匹配到的文件名

```shell
grep -l abc a.txt b.txt c.txt
```

* 仅仅打印匹配到的文本

```shell
grep -o abc a.txt b.txt c.txt
```

* 指定多个正则表达式字符串

```shell
grep -e abc -e Acb -e cbe message.log
```

* 从文件中读取模式

```shell
grep -f pattern1.txt pattern2.txt
```

* 使用`|`分割多个匹配模式

```shell
grep abc|def message.log
```

* 在当前目录下的所有文件中搜索匹配

```shell
grep abc * 
```

## 七、grep源码

![alt text](/images/grep-image.png)

## 八、官方文档

![alt text](/images/grep-image-1.png)

## 九、man pages

![alt text](/images/grep-image-2.png)