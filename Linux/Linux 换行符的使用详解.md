### 简介

`Linux` 中的换行符对于格式化文本输出、修改文件和确保跨系统兼容性至关重要。

`Linux` 主要使用 `LF`（换行符，`\n`）来换行，而 `Windows` 使用 `CRLF`（回车符 + 换行符，`\r\n`）

### 检测文件中的换行符

#### 使用 cat -A 查看换行符

```shell
cat -A myfile.txt
```

输出 `Linux` 风格，`LF \n`：

```shell
Hello World$
```

输出 `Windows` 风格，`CRLF \r\n`:

```shell
Hello World^M$
```

* `$`：表示行的结束 (`LF`)

* `^M$`：表示文件有 `Windows` 换行符 (`\r\n`)

#### 使用 od -c 检查字符

```shell
od -c myfile.txt
```

输出：`Linux \n`

```shell
0000000   H   e   l   l   o       W   o   r   l   d  \n
```

输出：`Windows \r\n`

```shell
0000000   H   e   l   l   o       W   o   r   l   d  \r  \n
```

* `\r \n`： `Windows` 样式的行尾

* `\n`： `Linux` 风格的行尾

### 换行符格式转换

#### 将 Windows CRLF (\r\n) 转换为 Linux LF (\n)

* 使用 `dos2unix`

```shell
dos2unix myfile.txt
```

* 使用 `sed`

```shell
sed -i 's/\r$//' myfile.txt
```

* 使用 `tr`

```shell
cat myfile.txt | tr -d '\r' > newfile_unix.txt
```

#### 将 Linux LF (\n) 转换为 Windows CRLF (\r\n)

* `unix2dos`

```shell
unix2dos myfile.txt
```

* 使用 `sed`

```shell
sed -i 's/$/\r/' myfile.txt
```

* 使用 `awk`

```shell
awk '{print $0 "\r"}' myfile.txt > newfile_windows.txt
```

### 在输出中添加换行符

#### 打印多行文本

* 使用 `echo -e`

```shell
echo -e "Line 1\nLine 2\nLine 3"
```

输出：

```shell
Line 1
Line 2
Line 3
```

* 使用 `printf`

```shell
printf "Line 1\nLine 2\n"
```

### 在命令中插入换行符

#### 使用 `sed`

```shell
echo "Hello World" | sed 's/ / \n/g'
```

输出：

```shell
Hello
World
```

* 使用 `awk`

```shell
echo "Hello World" | awk '{print $1 "\n" $2}'
```

### 处理 Shell 脚本中的换行符

#### 循环遍历文件中的行

```shell
#!/bin/bash
while IFS= read -r line; do
    echo "Processing: $line"
done < myfile.txt
```

#### 从文件中删除空行

```shell
sed -i '/^$/d' myfile.txt

或

awk 'NF' myfile.txt > clean_file.txt
```

#### 计算文件中的换行符

```shell
grep -c '^' myfile.txt

或

wc -l myfile.txt
```

### 其他用法

#### 用新行追加文本

```shell
echo "New Entry" >> myfile.txt
```

#### 追加到文本不添加新行

```shell
echo -n "New Entry" >> myfile.txt
```

#### 检查文件是否以新行结尾

```shell
tail -c1 myfile.txt | od -c
```

* 如果输出显示 `\n`，则表示文件尾部有换行符。

* 如果没有输出，则文件缺少尾随换行符。

### 使用多行字符串

#### 将多行字符串分配给变量

```shell
mytext="Line 1
Line 2
Line 3"

echo "$mytext"
```

#### 使用 cat 读取多行输入

```shell
cat <<EOF > myfile.txt
This is line 1.
This is line 2.
EOF
```

### 在 Linux 中处理 Windows 格式的文件

#### 修复文件中的 ^M 字符

* 使用 `sed`

```shell
sed -i 's/\r$//' myfile.txt
```

* 使用 `vim`

```shell
vim myfile.txt

:set fileformat=unix
:wq
```