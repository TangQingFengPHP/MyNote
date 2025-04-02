### 简介

`PHP 8` 引入了 `match()` 表达式，用来替代传统的 `switch` 语句，提供更简洁、更安全的方式进行条件匹配。与 `switch` 不同，`match()` 是一个表达式，它会返回值，并且使用 严格比较（`===`）。

### 基本语法

```php
$result = match (表达式) {
    值1 => 结果1,
    值2 => 结果2,
    值3, 值4 => 结果3, // 多个值匹配同一个结果
    default => 默认结果
};
```

`match()` 直接返回值，并且必须匹配到一个值，否则会抛出 `UnhandledMatchError`。

### match() VS switch()

|  特性   |  switch   |  match   |
| --- | --- | --- |
|  语法   |  需要 `case` 和 `break`   |  直接使用 `=>`   |
|  比较方式   |  宽松比较 (`==`)   |  严格比较 (`===`)   |
|  返回值   |  需要 `return`   |  直接返回值   |
|  `fall-through`   |  可能（如果缺少 `break`）   |  不会（默认不会执行下一个 `case`）   |
|  未匹配情况   |  不会抛异常   |  会抛异常   |

### match() 的基本用法

#### 基本示例

```php
$number = 2;

$result = match ($number) {
    1 => 'One',
    2 => 'Two',
    3 => 'Three',
    default => 'Unknown'
};

echo $result; // 输出 "Two"
```

* `match()` 自动返回值，不需要 `return`

* 严格比较，`2 == "2"` 在 `switch` 里会匹配，但在 `match()` 里不会

#### 多个值匹配同一个结果

```php
$fruit = "apple";

$color = match ($fruit) {
    "apple", "cherry", "strawberry" => "red",
    "banana", "lemon" => "yellow",
    "grape", "blueberry" => "purple",
    default => "unknown"
};

echo $color; // 输出 "red"
```

* 可以在 `match()` 的 单个 `case` 里写多个匹配值

#### 严格比较 (===)

```php
$value = "2";

$result = match ($value) {
    2 => "Matched as integer",
    "2" => "Matched as string",
    default => "No match"
};

echo $result; // 输出 "Matched as string"
```

* `match()` 严格比较 (`===`)，不会把 `"2"` 误认为 `2`。

#### 变量匹配

```php
$input = 100;

$result = match ($input) {
    $a = 10 => "Ten",
    $b = 100 => "Hundred",
    default => "Other"
};

echo $result; // 输出 "Hundred"
```

* `match()` 支持 变量赋值（`$a = 10`），但是 `$a` 不能在 `match()` 外使用。

### match() 结合表达式

#### match() 结合函数

```php
function getCategory(string $food): string {
    return match ($food) {
        "apple", "banana", "cherry" => "Fruit",
        "carrot", "potato" => "Vegetable",
        default => "Unknown"
    };
}

echo getCategory("apple"); // 输出 "Fruit"
```

* `match()` 可以直接用在函数返回值，使代码更清晰。

#### match() 结合数组

```php
$code = 404;

$messages = [
    200 => "OK",
    301 => "Moved Permanently",
    404 => "Not Found",
    500 => "Internal Server Error"
];

$message = match ($code) {
    default => $messages[$code] ?? "Unknown Status"
};

echo $message; // 输出 "Not Found"
```

* `match()` 可以结合数组 实现动态映射

#### match() 结合 throw

```php
$role = "guest";

$permission = match ($role) {
    "admin" => "Access All",
    "editor" => "Edit Content",
    "user" => "View Content",
    default => throw new Exception("Invalid role: $role")
};

echo $permission;
```

* `match()` 可以直接抛出异常，增强安全性。

### match() 进阶用法

#### match() 结合 fn()（箭头函数）

```php
$input = "php";

$result = fn() => match ($input) {
    "php" => "Hypertext Preprocessor",
    "js" => "JavaScript",
    default => "Unknown"
};

echo $result(); // 输出 "Hypertext Preprocessor"
```

* `match()` 可以结合箭头函数 `fn()`，使代码更加简洁。

#### match() 结合 true 模拟 switch 的 case 逻辑

```php
$age = 25;

$category = match (true) {
    $age < 18 => "Minor",
    $age >= 18 && $age < 60 => "Adult",
    default => "Senior"
};

echo $category; // 输出 "Adult"
```

* 这样可以避免 `switch(true)` 的写法，让 `match()` 处理 范围匹配。

### match() 的错误和注意点

#### 必须匹配到一个值

```php
$value = 5;

$result = match ($value) {
    1 => "One",
    2 => "Two",
    3 => "Three"
};

echo $result;
```

错误：`Uncaught UnhandledMatchError: Unhandled match case '5'`

解决方法：加上 `default`

```php
$result = match ($value) {
    1 => "One",
    2 => "Two",
    3 => "Three",
    default => "Unknown"
};
```