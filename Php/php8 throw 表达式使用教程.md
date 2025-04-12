### 简介

在 `PHP 8` 中，`throw` 可以作为一个 表达式（`expression`） 来使用，而不再仅仅是语句（`statement`）。这是一项非常实用的新特性，能够让 `throw` 更加灵活，尤其适用于 三元运算符、箭头函数、空合并运算符 (`??`) 等表达式中。

### 基本语法

```php
throw new Exception("Something went wrong");
```

这是 `PHP 7` 及之前的写法，只能单独作为语句使用。

### 用法示例

`PHP 8` 开始，`throw` 可以写在如下位置：

* 三元运算符中

* 空合并运算符中（`??`）

* 箭头函数（`fn`）中

* 直接作为返回值

#### 用于三元表达式中

```php
$value = $input !== null ? $input : throw new InvalidArgumentException("input is required");
```

传统写法要写成多行：

```php
if ($input !== null) {
    $value = $input;
} else {
    throw new InvalidArgumentException("input is required");
}
```

#### 用于 null 合并运算符中 (??)

```php
$username = $_GET['username'] ?? throw new Exception("username is required");
```

当没有传 `username` 参数时，直接抛出异常。

等价于：

```php
if (!isset($_GET['username'])) {
    throw new Exception("username is required");
}
$username = $_GET['username'];
```

#### 用于箭头函数中

```php
$fn = fn($value) => $value ?? throw new Exception("Missing value");

echo $fn("Hello"); // 输出 Hello
echo $fn(null);    // 抛出异常
```

之前的箭头函数无法写 `throw`，`PHP 8` 中可以了！

#### 用作函数返回值

```php
function getUser($id) {
    return $id > 0 ? "User $id" : throw new Exception("Invalid ID");
}
```

#### 用于逻辑表达式中

```php
is_string($name) || throw new InvalidArgumentException("Name must be a string");
```

类似于断言语法，用 `||` 来做“验证失败就抛异常”。

> 注意事项

* `throw` 表达式的返回类型是 `never`（从 `PHP 8.1` 起支持 `never` 类型，表示函数不会返回值，只会中止执行）。

* 表达式中的 `throw` 不能用于返回非异常类（必须是 `Throwable` 或其子类）。

* 在 `match`、`array_map()` 等表达式中也可以使用。

#### 用于 match() 结合 throw

```php
echo match($type) {
    'json' => 'application/json',
    'html' => 'text/html',
    default => throw new Exception("Unsupported type: $type"),
};
```

#### array_map 结合 throw 表达式

* 基本示例：抛出非法元素异常

`throw` 表达式可以与 `array_map()` 搭配使用，用来对数组元素进行验证、转换或过滤时抛出异常。这在处理用户输入、配置项、批量数据时特别有用。


```php
$data = [1, 2, 'x', 4];

$result = array_map(fn($item) =>
    is_int($item)
        ? $item * 2
        : throw new InvalidArgumentException("Invalid item: $item"),
    $data
);
```

* 高级示例：检查字段存在性

```php
$users = [
    ['name' => 'Alice'],
    ['name' => 'Bob'],
    ['email' => 'carol@example.com'], // 不合法
];

$names = array_map(fn($user) =>
    $user['name'] ?? throw new Exception("Missing 'name' in user: " . json_encode($user)),
    $users
);
```

* `DTO` / 数据结构映射中很有用

```php
class UserDTO {
    public function __construct(public string $name) {}
}

$raw = [['name' => 'Tom'], ['foo' => 'bar']];

$users = array_map(fn($item) =>
    isset($item['name'])
        ? new UserDTO($item['name'])
        : throw new Exception("Invalid user item: " . json_encode($item)),
    $raw
);
```

* 包装为验证函数增强复用性

```php
function requireKey(array $arr, string $key) {
    return $arr[$key] ?? throw new InvalidArgumentException("Missing key: $key");
}

$data = [['a' => 1], ['b' => 2]];

$values = array_map(fn($item) => requireKey($item, 'a'), $data);
```

### 写一个可复用的 SafeArrayMapper 类 

支持在批量数据处理时安全验证字段是否存在、类型是否匹配，并在异常情况下中断处理。

> 特性

* `requireKey()`：字段必须存在

* `requireType()`：类型必须符合

* `map()`：安全地映射数组，自动抛出异常

* 支持 `DTO` 构建等场景

代码实现

```php
<?php

class SafeArrayMapper
{
    /**
     * 验证字段是否存在
     */
    public static function requireKey(array $item, string $key): mixed
    {
        return $item[$key] ?? throw new InvalidArgumentException("Missing required key: '$key'");
    }

    /**
     * 验证字段存在且类型正确
     */
    public static function requireType(array $item, string $key, string $type): mixed
    {
        $value = $item[$key] ?? throw new InvalidArgumentException("Missing required key: '$key'");
        if (gettype($value) !== $type) {
            throw new InvalidArgumentException("Key '$key' must be of type $type, got " . gettype($value));
        }
        return $value;
    }

    /**
     * 批量映射数据并自动验证
     *
     * @param array $data 多项数据
     * @param callable $callback 映射函数，接收单项
     * @return array 处理结果
     */
    public static function map(array $data, callable $callback): array
    {
        return array_map(function ($item) use ($callback) {
            if (!is_array($item)) {
                throw new InvalidArgumentException("Each item must be an array");
            }
            return $callback($item);
        }, $data);
    }
}
```

示例用法

* 字段校验 + `DTO` 构建

```php
class UserDTO {
    public function __construct(public string $name, public int $age) {}
}

$rawUsers = [
    ['name' => 'Alice', 'age' => 28],
    ['name' => 'Bob', 'age' => 35],
    ['age' => 22], // 错误：缺少 name
];

$users = SafeArrayMapper::map($rawUsers, fn($item) =>
    new UserDTO(
        SafeArrayMapper::requireType($item, 'name', 'string'),
        SafeArrayMapper::requireType($item, 'age', 'integer')
    )
);
```

* 快速获取字段数组（e.g. 所有 `email`）

```php
$raw = [
    ['email' => 'a@example.com'],
    ['email' => 'b@example.com'],
    ['name' => 'no_email'], // 错误项
];

$emails = SafeArrayMapper::map($raw, fn($item) =>
    SafeArrayMapper::requireKey($item, 'email')
);
```

* 注册全局 `helper` 函数（方便使用）

```php
function safe_key(array $arr, string $key) {
    return SafeArrayMapper::requireKey($arr, $key);
}
```

`composer` 注册为自动加载：

```json
{
  "autoload": {
    "files": [
      "helpers.php"
    ]
  }
}
```

或：

```php
require_once __DIR__ . '/path/to/helpers.php';
```

用法：

```php
$names = array_map(fn($item) => safe_key($item, 'name'), $items);
```