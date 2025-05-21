### 简介

在 `PHP 7.4` 中，官方引入了 箭头函数（`Arrow Functions`），语法简洁、行为类似 `JavaScript` 的箭头函数，同时保留了 `PHP` 的闭包特性。

### 基本语法

```php
fn(parameter_list) => expression;
```

* 使用 `fn` 关键字。

* `parameter_list`：函数的参数，可以为空或包含多个参数

* `=>`：箭头符号，分隔参数和函数体

* `expression`：函数的返回值，通常是单行表达式（不支持多行语句块）

* 只能用于单表达式体，不能有语句块、`return` 关键字或多行语句。

* 自动从父作用域继承变量（不需要 `use`）。

### 特性说明

#### 自动变量绑定（`Lexical Scoping`）

箭头函数中使用外部变量时，不需要使用 `use()`：

```php
$a = 5;
$fn = fn($b) => $a + $b;
echo $fn(3); // 8
```

#### 不支持多行或复杂逻辑

```php
$fn = fn($x) => {
    $y = $x * 2;
    return $y;
}; // ❌ 错误
```

应该使用传统匿名函数：

```php
$fn = function($x) {
    $y = $x * 2;
    return $y;
};
```

#### 不能修改外部变量

由于箭头函数捕获外部变量是值传递，修改函数内部的变量不会影响外部变量。

```php
$x = 10;
$fn = fn($a) => $a + ($x += 5); // 错误：不能修改 $x
echo $fn(5); // 抛出错误
```

如果需要修改外部变量，需使用传统匿名函数并通过 `use (&$x)` 传递引用。

### 箭头函数 vs 匿名函数

#### 普通匿名函数

```php
$factor = 10;

$square = function($n) use ($factor) {
    return $n * $factor;
};

echo $square(2); // 20
```

#### 箭头函数（更简洁）

```php
$factor = 10;

$square = fn($n) => $n * $factor;

echo $square(2); // 20
```

#### 与传统匿名函数的对比

| 特性          | 箭头函数 (`fn`)          | 传统匿名函数 (`function`)                         |
| ------------- | ---------------------- | ----------------------------------------------- |
| 语法          | 简洁：`fn($x) => $x * 2` | 冗长：`function ($x) use ($y) { return $x * 2; }` |
| 变量捕获      | 自动按值捕获外部变量   | 需显式使用 `use` 捕获变量（支持引用）             |
| 函数体        | 仅支持单行表达式       | 支持多行语句块                                  |
| return 关键字 | 隐式返回               | 需显式使用 `return`                               |
| 性能          | 略高（更轻量级）       | 略低（更灵活但稍重）                            |

何时使用箭头函数：

* 需要简洁的单行逻辑（如数组操作的回调）。

* 不需要修改外部变量。

* 用于简单的闭包或回调函数（如 `array_map、array_filter` 等）。

何时使用传统匿名函数：

* 需要多行逻辑。

* 需要通过引用修改外部变量。

* 需要更复杂的控制结构（如循环、条件语句）。

### 示例用法

#### 数组映射

```php
$nums = [1, 2, 3];
$result = array_map(fn($x) => $x * 2, $nums);
// [2, 4, 6]
```

#### 数组过滤

```php
$users = [
    ['id' => 1, 'active' => true],
    ['id' => 2, 'active' => false],
];

$activeUsers = array_filter($users, fn($u) => $u['active']);
// 只保留 active 为 true 的用户
```

#### 闭包作为参数

简化事件处理器、中间件等场景

```php
// 模拟一个中间件
$middleware = function ($request, $next) {
    // 处理请求...
    return $next($request);
};

// 使用箭头函数简化
$middleware = fn ($request, $next) => $next($request);
```

#### 排序回调

简化 `usort` 或 `uasort` 的比较逻辑

```php 
$users = [
    ['name' => 'Alice', 'age' => 30],
    ['name' => 'Bob', 'age' => 25],
];

// 按年龄升序排序
usort($users, fn ($a, $b) => $a['age'] <=> $b['age']);
```

#### 动态生成条件过滤器

```php
$minPrice = 100;
$products = [
    ['name' => 'A', 'price' => 80],
    ['name' => 'B', 'price' => 150],
];

// 过滤价格 >= $minPrice 的商品
$filtered = array_filter(
    $products,
    fn ($product) => $product['price'] >= $minPrice
);
```

#### 链式调用中的闭包

```php
$data = [1, 2, 3];
$processed = array_map(
    fn ($x) => $x * 2,
    array_filter(
        $data,
        fn ($x) => $x > 1
    )
);
// 结果： [4, 6]
```

#### 嵌套箭头函数

```php
$multiplier = fn($x) => fn($y) => $x * $y;
$double = $multiplier(2);
echo $double(5); // 输出：10
```

#### 结合高阶函数

箭头函数可以作为参数传递给其他函数。

```php
function applyCallback(array $data, callable $callback) {
    return array_map($callback, $data);
}

$result = applyCallback([1, 2, 3], fn($x) => $x ** 2);
print_r($result); // 输出：[1, 4, 9]
```