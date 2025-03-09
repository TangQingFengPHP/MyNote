### 简介

`Linux` 中的 `expr` 命令用于计算表达式的值，包括算术运算、字符串操作和逻辑比较。它常用于 `shell` 脚本。

### 基本算术运算

`expr` 支持基本算术运算，例如加、减、乘、除和模数

* 加(`+`)

```shell
expr 5 + 3
# Output: 8
```

* 减(`-`)

```shell
expr 10 - 4
# Output: 6
```

* 乘(`*`)

`*` 运算符必须进行转义（`\*`）以防止 `shell` 解释

```shell
expr 6 \* 3
# Output: 18
```

* 除(`/`)

```shell
expr 12 / 4
# Output: 3
```

* 取模(`%`)

```shell
expr 10 % 3
# Output: 1
```

### 字符串操作

#### 计算字符串长度

```shell
expr length "Hello"
# Output: 5
```

#### 提取字符串

```shell
expr substr "HelloWorld" 1 5
# Output: Hello
```

#### 查找字符位置

```shell
expr index "LinuxShell" 'S'
# Output: 6
```

### 比较操作

#### 检查是否相等(=)

```shell
expr "apple" = "apple"
# Output: 1 (true)
```

#### 检查是否不相等(!=)

```shell
expr "apple" != "banana"
# Output: 1 (true)
```

#### 比大小

`>` 和 `<` 必须用 `\` 进行转义，以防止被 `shell` 解释

```shell
expr 10 \> 5
# Output: 1 (true)

expr 2 \< 8
# Output: 1 (true)
```

### 逻辑操作

#### AND(`&`)

```shell
expr 4 \> 2 \& 3 \> 1
# Output: 1 (true)
```

#### OR(`|`)

```shell
expr 4 \< 2 \| 3 \> 1
# Output: 1 (true)
```

### 在 `shell` 脚本中使用 `expr`

```shell
#!/bin/bash
a=10
b=5
sum=$(expr $a + $b)
echo "Sum: $sum"

# Output: Sum: 15
```

### expr 的替代方案

* 使用 `$(())` 进行算术运算

```shell
echo $((5 + 3))
```

* 使用 `bc` 进行浮点计算

```shell
echo "scale=2; 10 / 3" | bc
```