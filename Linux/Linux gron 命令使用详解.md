### 简介

`gron` 是一个独特的命令行工具，用于将 `JSON` 数据转换为离散的、易于 `grep` 处理的赋值语句格式。它的名字来源于 "`grepable on`" 或 "`grepable JSON`"，主要解决在命令行中处理复杂 `JSON` 数据的难题。

核心价值

`gron` 的核心是将 `JSON` 数据展平为类似 `json.path.to.key = value`; 的格式

* 简化 `JSON` 处理：将嵌套的 `JSON` 结构扁平化为易于搜索的格式

* 增强 `grep` 能力：使标准文本工具能高效处理 `JSON` 数据

* 可逆转换：可将处理后的数据还原为原始 `JSON`

### 安装

* `Ubuntu/Debian`

```shell
sudo apt install gron
```

* `CentOS/RHEL`

```shell
sudo yum install epel-release
sudo yum install gron
```

* `macOS`

```shell
brew install gron
```

* 从源码安装 (`Go`)

```shell
go install github.com/tomnomnom/gron@latest
```

### 常用选项

* `-c, --color`：强制彩色输出（即使非终端环境）。

* `-i, --indent`：指定缩进空格数（默认 2）。

* `-n, --no-sort`：不排序输出结果。

* `-u, --ungron`：将 `gron` 格式转回 `JSON`。

* `--json`：等同于 `--ungron`，但更符合语义。

* `-v, --values`: 仅输出值的部分（不包括路径）

* `-s, --stream`: 将每行输入视为单独的 `JSON` 对象处理


### 示例用法

`data.json` 文件示例：

```json
{
  "name": "Alice",
  "age": 30,
  "pets": [
    {"name": "Rex", "type": "dog"},
    {"name": "Whiskers", "type": "cat"}
  ],
  "address": {
    "city": "New York",
    "zip": "10001"
  }
}
```

#### 转换 JSON 为 gron 格式

```shell
gron data.json
```

输出：

```javascript
json = {};
json.name = "Alice";
json.age = 30;
json.pets = [];
json.pets[0] = {};
json.pets[0].name = "Rex";
json.pets[0].type = "dog";
json.pets[1] = {};
json.pets[1].name = "Whiskers";
json.pets[1].type = "cat";
json.address = {};
json.address.city = "New York";
json.address.zip = "10001";
```

#### 搜索特定值

```shell
gron data.json | grep "zip"
```

输出：

```javascript
json.address.zip = "10001";
```

#### 恢复为 JSON 格式 (--ungron)

```shell
gron data.json | grep "pets" | gron --ungron
```

输出：

```javascript
{
  "pets": [
    {
      "name": "Rex",
      "type": "dog"
    },
    {
      "name": "Whiskers",
      "type": "cat"
    }
  ]
}
```

#### 使用自定义变量名 (-s)

```shell
gron -s data data.json
```

输出：

```javascript
data = {};
data.name = "Alice";
```

#### 流式处理大型文件 (--stream)

```shell
curl -s https://api.example.com/large-data | gron --stream
```

#### 指定输出格式 (-j, --json)

```shell
gron data.json -j | grep "name"
```

输出：

```javascript
$.name = "Alice";
$.pets[0].name = "Rex";
$.pets[1].name = "Whiskers";
```

#### 组合 awk 处理数据

```shell
gron data.json | awk '/pets/ && /type/ {print $3}'
```

输出：

```
"dog"
"cat"
```

#### 修改并恢复数据

```shell
gron data.json | sed 's/"New York"/"Boston"/' | gron --ungron
```

#### 处理多个文件

```shell
gron file1.json file2.json | grep "error"
```

#### 使用 jq 风格路径

```shell
gron -j data.json | grep 'pets.*name'
```

输出：

```javascript
$.pets[0].name = "Rex";
$.pets[1].name = "Whiskers";
```

#### 调试 API 响应

```shell
curl -s https://api.github.com/users/octocat | gron | grep "company"
```

#### 处理复杂配置文件

```shell
gron config.json | grep "database.password"
```

#### 搜索嵌套值

```shell
gron data.json | grep "pets.*cat"
```

#### 提取所有键名

```shell
gron data.json | awk -F '.' '{print $2}' | sort | uniq
```

#### 处理压缩数据

```shell
zcat large.json.gz | gron --stream | grep "error"
```

#### 颜色高亮输出

```shell
gron data.json | grep --color=auto "name"
```