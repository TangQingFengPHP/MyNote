### 简介

`jq` 是一个命令行 `JSON` 处理器，允许解析、过滤、转换和格式化 `JSON` 数据，提取特定字段或重构 `JSON`，高效使用 `JSON` 中的 `API` 或配置文件。

### 安装

* `Debian/Ubuntu    `

```shell
sudo apt install jq
```

* `CentOS/RHEL`

```shell
sudo yum install jq

或

sudo dnf install jq
```

* `macOS（Homebrew）`

```shell
brew install jq
```

### 基础语法

```shell
jq '<filter>' <file_or_input>
```

### 常用选项

* `-r`：输出原始字符串（去除 `JSON` 引号）

* `-c`：紧凑输出（不格式化），即去除不必要空格

* `--slurp`：将输入合并为单个 `JSON` 数组

* `--arg`：传递外部变量到过滤器

* `--raw-input`：将输入视为原始文本（非 `JSON`）

* `-n`：不使用输入数据（需手动读取）

* `-S`：按键名排序对象字段

* `--tab`：使用制表符缩进（美化输出）

* `--arg name value`：定义变量，供过滤器使用

* `-M, --monochrome-output`：禁用彩色输出

* `-j, --join-output`：输出不换行，适合处理多行输出

### 基本过滤器

* `.` : 表示整个输入 `JSON`。

* `.key`: 访问对象的字段（例如 `.name` 提取字段 `name` 的值）。

* `[]`: 访问数组元素（例如 `.items[0]` 提取数组第一个元素）。

* `. | select(condition)`: 过滤数据，基于条件选择（例如 `.[] | select(.age > 30)`）。

* `. | map(transform)`: 对数组中的每个元素应用转换（例如 `map(.price * 2)`）。

* `. | length`: 获取数组长度或字符串长度。

* `. | keys`: 获取对象的键列表。

* `. | sort_by(field)`: 按指定字段排序数组。

* `. | group_by(field)`: 按字段分组。

* `. | to_entries`: 将对象转换为键值对数组。

* `. | join(",")`: 将数组元素连接为字符串。

### 过滤器语法详解

#### 基础操作

* 格式化输出：

```shell
cat data.json | jq '.'  # 美化输出 JSON
```

输出效果：

```json
{
  "name": "Alice",
  "age": 30,
  "hobbies": ["reading", "hiking"]
}
```  

* 选择字段：

```shell
jq '.[].name' data.json  # 提取数组所有元素的 name 字段
```

* 条件过滤：

```shell
jq '.[] | select(.age > 25)' data.json  # 过滤年龄大于25的对象
```

* 多字段组合：

```shell
jq '.[] | {name, city}' data.json  # 组合 name 和 city 字段
```

* 新增字段：

```shell
jq '.[] | .country = "USA"' data.json  # 添加 country 字段
```

* 删除字段：

```shell
jq 'del(.[].city)' data.json  # 删除所有元素的 city 字段
```

* 数组映射：

```shell
jq 'map(.age * 2)' data.json  # 所有年龄翻倍
```

* 排序与切片：

```shell
jq 'sort_by(.age) | .[:2]' data.json  # 按年龄升序取前2个元素
```

* 逻辑运算：and, or, not

```shell
jq 'select(.age > 20 and .city == "Beijing")' data.json
```

* 内置函数：

```shell
jq 'length' data.json        # 数组长度
jq 'keys' data.json          # 对象键名列表
jq 'group_by(.city)' data.json  # 按城市分组
```

* 字段访问：

```shell
echo '{"user": {"name": "Alice", "age": 30}}' | jq '.user.name'
# 输出："Alice"
```

* 数组索引：

```shell
echo '[1, 2, 3]' | jq '.[1]'  # 输出：2
```

* 迭代数组：

```shell
echo '[{"id":1}, {"id":2}]' | jq '.[].id'  # 输出：1 和 2
```

* 多字段选择：

```shell
echo '{"name": "Alice", "age": 30}' | jq '{name, age}'
# 输出：{"name": "Alice", "age": 30}
```

#### 条件与逻辑

* 条件过滤（`if-else`）：

```shell
echo '30' | jq 'if . > 18 then "Adult" else "Child" end'
# 输出："Adult"
```

* 逻辑运算符：

```shell
echo '{"age": 20}' | jq 'select(.age >= 18 and .age <= 60)'
# 输出：{"age": 20}
```

#### 字符串与数学操作

* 字符串拼接：

```shell
echo '{"name": "Alice"}' | jq '"Hello, " + .name'  # 输出："Hello, Alice"
```

* 数学运算：

```shell
echo '{"x": 5, "y": 3}' | jq '.x * .y'  # 输出：15
```

#### 高级操作

* 递归下降（..）：

```shell
echo '{"a": {"b": [1, 2]}}' | jq '.. | numbers?'  # 输出：1 和 2
```

* 自定义函数：

```shell
echo '5' | jq 'def pow(n): . ^ n; pow(3)'  # 输出：125
```

* 合并数据（+）：

```shell
echo '{"a":1} {"b":2}' | jq -s '.[0] + .[1]'  # 输出：{"a":1, "b":2}
```

### 用法示例

#### 格式化打印 JSON（彩色、缩进）

```shell
cat data.json | jq .

jq '.' fruit.json

curl http://api.open-notify.org/iss-now.json | jq '.'
```

#### 提取指定字段

```shell
cat data.json | jq '.name'
```

#### 提取嵌套字段

```shell
cat data.json | jq '.user.address.city'
```

#### 从数组里面选择对象

```shell
cat data.json | jq '.users[] | select(.age > 30)'
```

#### 获取对象数组中特定键的所有值

```shell
cat data.json | jq '.users[].name'
```

#### 过滤并输出为新的 JSON 数组

```shell
cat data.json | jq '[.users[] | select(.active == true)]'
```

#### 修改 JSON（添加或更改键）

```shell
cat data.json | jq '.users[] += {"role": "guest"}'
```

#### 将输出格式化为原始字符串（删除引号）

```shell
cat data.json | jq -r '.users[].name'
```

#### 从命令行使用（无需文件）

```shell
echo '{"key": "value"}' | jq '.key'
```

#### 使用管道组合多个过滤器

```shell
cat data.json | jq '.users[] | select(.active == true) | .name'
```

#### 将多个键放入一个新对象中

```shell
cat data.json | jq '.users[] | {name, email}'
```

#### 使用变量

```shell
name="John"
cat data.json | jq --arg name "$name" '.users[] | select(.name == $name)'
```

#### 统计元素

```shell
cat data.json | jq '.users | length'
```

#### 结合 diff 比较 json 文件

```shell
diff <(jq -S . file1.json) <(jq -S . file2.json)
```

#### 合并多个 JSON 对象

```shell
echo '{"a":1} {"b":2}' | jq --slurp '.'  

# 输出：{"key": "new_value"}
```

#### 遍历数组

```shell
jq '.items[]'
```

#### 根据条件选择项目

```shell
jq '.users[] | select(.age > 30)'
```

#### 使用 reduce 进行聚合

```shell
jq 'reduce .[] as $item (0; . + $item.value)'
```

#### 使用walk进行递归遍历

```shell
jq 'walk(if type == "string" then ascii_downcase else . end)'   
```

#### 传递变量

```shell
jq --arg var "new_value" '.key = $var' <<< '{"key": "old_value"}' 

# 输出：{"key": "new_value"}
```

#### 提取 API 响应的特定字段

```shell
curl -s https://api.example.com/users | jq '.[].email'
```

#### 过滤日志中的错误信息

```shell
cat logs.json | jq 'select(.level == "ERROR") | .message'
```

#### 格式化 JSON 文件

```shell
jq '.' input.json > formatted.json
```

#### 重命名字段

```shell
echo '{"old_name": "Alice"}' | jq '{new_name: .old_name}'
# 输出：{"new_name": "Alice"}
```

#### 处理流式 JSON（--stream）

解析大型或流式 `JSON` 文件：

```shell
jq --stream 'select(length==2)' large.json
```

#### 定义可复用的函数库：

```shell
# math.jq
def sqrt: . ^ 0.5;

# 使用
echo '16' | jq 'import "math" as math; math::sqrt'
```

#### 正则表达式匹配（test）

```shell
echo '"alice@example.com"' | jq 'test("@example.com$")'  # 输出：true
```

#### 调试过滤器（debug）

```shell
echo '{"data": [1, 2]}' | jq '.data | debug | .[]'
```

#### 删除指定字段

```shell
jq 'del(.address)' data.json
```

#### 切片

```shell
echo '[1,2,3,4,5,6,7,8,9,10]' | jq '.[6:9]'
```

#### 返回长度

```shell
jq '.fruit | length' fruit.json
```

#### 批量修改配置文件

```shell
jq '.config.timeout = 30' settings.json > tmp.json && mv tmp.json settings.json
```

#### 生成新的 JSON 对象

```shell
jq -n '{"new_field": "value"}'
```

#### 将用户数据转换为新的结构

```shell
jq '.users | map({"full_name": .name, "details": { "age": .age, "location": .city }})' data.json
```
