### 简介

在 `Bash` 脚本中，`shift` 命令用于将命令行参数向左移动，有效地丢弃第一个参数并将其他参数向下移动。

### 基础语法

```shell
shift [N]
```

N（可选）→ 要移动的位置数。默认值为 1

### 示例用法

#### 移动参数

```shell
#!/bin/bash
echo "Before shift: $1 $2 $3"

shift  # Shift once

echo "After shift: $1 $2 $3"
```

运行脚本示例：

```shell
./script.sh a b c d
```

输出如下：

```shell
Before shift: a b c
After shift: b c d
```

* `shift` 删除 `$1 (a)`

* `$2` 变成 `$1`, `$3` 变成 `$2` 等等。

#### 在循环中使用 shift

```shell
#!/bin/bash
while [[ $# -gt 0 ]]; do
    echo "Argument: $1"
    shift
done
```

运行：

```shell
./script.sh one two three
```

输出：

```shell
Argument: one
Argument: two
Argument: three
```

#### 将 shift 与 N 结合使用

```shell
#!/bin/bash
echo "Before shift: $1 $2 $3 $4"

shift 2  # Shift by 2 places

echo "After shift: $1 $2"
```

运行：

```shell
./script.sh a b c d
```

输出：

```shell
Before shift: a b c d
After shift: c d
```

* `$1` 和 `$2` 都被移除

#### 检查参数是否有剩余

```shell
#!/bin/bash
while [[ $# -gt 0 ]]; do
    echo "Processing: $1"
    shift
done
echo "No more arguments."
```

* `$#`：剩余参数的数量

* 当 `$#` 达到0时，循环结束

#### 循环遍历成对的参数

```shell
#!/bin/bash
while [[ $# -gt 1 ]]; do
    echo "Key: $1, Value: $2"
    shift 2
done
```

运行：

```shell
./script.sh --name Alice --age 30 --city Parisss
```

输出：

```shell
Key: --name, Value: Alice
Key: --age, Value: 30
Key: --city, Value: Paris
```