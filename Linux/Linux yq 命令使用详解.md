### 简介

`yq` 是一个轻量级、可移植的命令行 `YAML` 处理器，它允许使用类似于 `jq` 的语法读取、写入、更新、合并和过滤 `YAML` 数据。

主要有两个版本：

* 基于 `Python` 的并包装 `jq`，依赖 `jq` 语法

* 用 `Go` 写的（`mikefarah/yq`），目前最流行的版本，独立实现，功能更丰富，支持原地修改文件

### 安装

* `Debian/Ubuntu`

```shell
apt-get install yq
```

* `CentOS`

```shell
yum install yq

或

dnf install yq
```

* `macOS`

```shell
brew install yq
```

* 从二进制构建安装

```shell
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/local/bin/yq
chmod +x /usr/local/bin/yq
```

### 基础语法

```shell
yq [command] [flags] [expression] file.yaml
```

### 常用选项

* `-i`：直接修改原文件（类似 `sed -i`）

* `-o`：输出格式（`yaml、json、props、xml`）

* `-P`：指定输入格式，保留 `YAML` 格式（如注释、锚点）

* `-j`：输入为 `JSON` 格式（自动转换）

* `--prettyPrint`：美化输出（默认启用）

* `--colors`：启用颜色高亮

* `--xml-attr-prefix`：处理 `XML` 属性前缀（`XML` 格式专用）

* `-d <索引>` ：处理多文档时的文档索引（默认 0）

* `-s`：指定脚本

* `-r, --raw-output`: 输出原始字符串，而不是 `YAML/JSON` 编码的字符串

* `-N, --no-colors`: 禁用彩色输出

* `-I, --indent <N>`: 设置输出缩进（默认 2 个空格）

* `-v, --verbose`: 显示详细日志

* `-c, --compact-output`: 输出紧凑格式（单行 `JSON/YAML`）

### 基本表达式

`yq` 使用类似 `jq` 的语法（`.` 表示根节点，`.` 后接字段名访问数据）

* `.`: 表示整个输入 `YAML`。

* `.key`: 访问对象的字段（例如 `.name`）。

* `[]`: 访问数组元素（例如 `.items[0]`）。

* `. | select(condition)`: 过滤数据（例如 `.users[] | select(.age > 30)`）。

* `. | map(transform)`: 对数组元素应用转换（例如 `map(.price * 2)`）。

* `. | del(path)`: 删除指定路径的字段（例如 `del(.users[0])`）。

* `. | has("key")`: 检查字段是否存在。

* `. | length`: 获取数组或字符串长度。

* `. | sort_by(field)`: 按字段排序。

* `. | with_entries(...)`: 将对象转换为键值对数组。

* `. | += / =`: 更新或赋值字段（例如 `.version |= "2.0"`）。

* `. | env`: 访问环境变量（例如 `.env.DB_HOST`）。

### 基本操作

示例文件：`example.yaml`

```yaml
person:
  name: Alice
  age: 30
  hobbies:
    - reading
    - cycling
```
#### 读取 YAML

* 获取一个字段

```shell
yq '.person.name' example.yaml
# Output: Alice
```

* 获取数组项目

```shell
yq '.person.hobbies[0]' example.yaml
# Output: reading
```

#### 更新 YAML

* 变更一个值

```shell
yq '.person.age = 31' example.yaml
```

* 保存到源文件

```shell
yq -i '.person.age = 31' example.yaml
```

* 添加新的字段/键

```shell
yq '.person.job = "Engineer"' example.yaml

yq '.person += {"job": "Engineer"}' example.yaml
```

* 删除字段

```shell
yq 'del(.person.age)' example.yaml
```

* 数组操作

```shell
yq '.person.hobbies += gaming' example.yaml
```

#### 合并 YAML 文件

```shell
yq eval-all 'select(fileIndex == 0) * select(fileIndex == 1)' base.yaml override.yaml
```

#### 在 YAML 和 JSON 之间转换

* YAML → JSON

```shell
yq -o=json '.' example.yaml
```

* JSON → YAML

```shell
cat file.json | yq -P
```

#### 通过 select()

```shell
yq '.people[] | select(.age > 30)' people.yaml
```

#### 指定文档索引

```shell
yq -d1 'b.c' multi-doc.yaml  # 处理第二个文档（索引从0开始） 
```

### 示例用法

示例文件：

```yaml
server:
  host: "0.0.0.0"
  port: 8080
database:
  - name: "db1"
    type: "mysql"
  - name: "db2"
    type: "postgres"
```

#### 复杂查询

```shell
# 遍历数组并提取字段
yq '.database[].name' config.yaml
# 输出：db1 db2
```

#### 条件逻辑

```shell
yq '.database[] | select(.type == "mysql" or .name == "db2")' config.yaml
```

#### 数学运算

```shell
yq '.server.port * 2' config.yaml
# 输出：16160（假设原值为 8080）
```

#### 变量传递

```shell
yq --arg new_port 9000 '.server.port = $new_port' config.yaml
```

#### 多文件操作

```shell
yq eval-all '.common * .env' common.yaml env.yaml > merged.yaml
```

#### 处理标准输入

```shell
echo '{"name": "Alice"}' | yq -P
# 输出：
# name: Alice
```

#### 提取 Kubernetes 资源信息

```shell
kubectl get pod my-pod -o yaml | yq '.metadata.labels'
```

#### 批量修改 CI/CD 配置

```shell
yq -i '.jobs.build.steps += {"name": "test", "run": "make test"}' .github/workflows/ci.yaml
```

#### 验证 YAML 语法

```shell
yq 'tag == "!!map"' file.yaml  # 检查是否为合法的 YAML 对象
```

#### 生成动态配置

```shell
yq -n '.env = "prod" | .replicas = 3' > deployment.yaml
```

#### 通配符匹配

```shell
yq '.bob.*.cats' data.yaml  # 提取所有子项的 `cats` 字段 
```

#### 环境变量注入

动态替换值

```shell
NAME=John yq -i '.user.name = strenv(NAME)' profile.yaml
```

#### 脚本批量修改

使用脚本文件批量更新（`update_script.yaml`）

```yaml
.user.addresses[0].city: "Paris"
.log.level: "info"
```

执行脚本：

```shell
yq -s update_script.yaml config.yaml
```

#### 复杂路径处理

* 键含特殊字符（如 `.` 或 `-`）

```shell
yq '.["key.with.dots"]' data.yaml  # 引用特殊键名 
```

#### 计算数组长度 

```shell
yq '.user.orders | length' user.yaml  # 计算数组长度 
```

#### 日志分析与过滤

```shell
cat log.yaml | yq 'select(.level == "error") | {time: .timestamp, msg: .message}'
```

#### 自动化脚本集成

```shell
# 生成随机端口并写入配置
PORT=$((RANDOM%1000+3000)) yq -i '.service.port = env(PORT)' service.yaml
```

#### 修改嵌套数组元素

```shell
yq -i '(.servers[] | select(.name=="db") | .ip) = "192.168.1.100"' infra.yam
```

#### 使用正则匹配

```shell
yq '.[] | select(.email | test("@example.com$"))' contacts.yaml
```

#### 数学运算

```shell
yq '.metrics.traffic | map(.value * 1024)' stats.yaml
```

#### 条件判断

```shell
yq 'if .enabled then .config else {} end' settings.yaml
```

#### 配置文件修改

修改 `Kubernetes Deployment` 的镜像版本

```shell
yq -i '(.spec.template.spec.containers[] | select(.name=="app").image) = "nginx:1.25"' deployment.yaml
```

#### 数据清洗与转换

提取 `CSV` 中的特定列并转为 `JSON`

```shell
yq -i -o=json '.[] | {"name": .Name, "age": .Age}' data.csv
```

#### 自动化运维脚本

批量禁用失效服务配置

```shell
for file in *.yaml; do
  yq -i '(.services[] | select(.status=="inactive")).enabled = false' "$file"
done
```

#### 处理多文档 YAML

示例文件

```yaml
---
name: Doc1
value: 1
---
name: Doc2
value: 2
```

提取所有文档的 `name`

```shell
yq '.. | .name?' multi.yaml
```

输出：

```yaml
Doc1
Doc2
```

#### 使用环境变量

```shell
export MY_VERSION=2.0
yq '.version = env.MY_VERSION' data.yaml
```

#### 合并多个 YAML 文件

`file1.yaml` 和 `file2.yaml` 会被合并，字段冲突以最后一个文件为准

```shell
yq '.[0] * .[1]' file1.yaml file2.yaml
```

#### 创建新 YAML

```shell
yq -n '{"new_field": "value", "numbers": [1, 2, 3]}'
```

输出：

```yaml
new_field: value
numbers:
  - 1
  - 2
  - 3
```

#### 递归查找

```shell
yq '.. | .name?'
```