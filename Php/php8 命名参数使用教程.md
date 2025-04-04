### 简介

`PHP 8` 引入 命名参数（`Named Arguments`），允许在调用函数时按参数名传递值，而不是按照参数位置。这增强了代码的可读性、灵活性，并减少参数顺序依赖。

### 基本用法

传统位置参数（`Positional Arguments`）：

```php
function greet($name, $greeting) {
    echo "$greeting, $name!";
}

greet("Alice", "Hello"); // 输出：Hello, Alice!
```

* 调用时必须按顺序传递参数，否则逻辑错误。

* 代码不直观，开发者可能不清楚 `Hello` 是 `greeting` 还是 `name`。

`PHP 8` 命名参数：

```php
greet(name: "Alice", greeting: "Hello"); // 输出：Hello, Alice!
```

* 直接指定参数名，不必记住顺序。

* 代码可读性更强，易维护。

### 位置参数 + 命名参数混合

`PHP 8` 允许 位置参数（`Positional Arguments`） 和 命名参数（`Named Arguments`） 混合使用：

```php
greet("Alice", greeting: "Hi"); // 输出：Hi, Alice!
```

位置参数必须在前，命名参数必须在后！

```php
greet(greeting: "Hi", "Alice"); // 语法错误：位置参数不能放在命名参数之后
```

### 省略默认参数

传统写法（必须按顺序传递所有参数）：

```php
function createUser($name, $age = 18, $city = "Unknown") {
    echo "Name: $name, Age: $age, City: $city";
}

// 只想传 `city`，但必须提供 `age`
createUser("Alice", 25, "New York"); // Name: Alice, Age: 25, City: New York
```

`PHP 8` 命名参数（可以省略默认参数）：

```php
createUser(name: "Alice", city: "New York"); // Name: Alice, Age: 18, City: New York
```

* 只提供 `name` 和 `city`，省略 `age`（使用默认值 `18`）。

* 避免传递不需要的参数，调用更灵活。

### 适用于函数、方法、构造函数

#### 用于类方法

```php
class Person {
    public function setInfo($name, $age = 18, $city = "Unknown") {
        echo "Name: $name, Age: $age, City: $city";
    }
}

$person = new Person();
$person->setInfo(name: "Bob", city: "Los Angeles"); 
// Name: Bob, Age: 18, City: Los Angeles
```

#### 用于构造函数

```php
class Car {
    public function __construct($brand, $color = "black", $price = 10000) {
        echo "Brand: $brand, Color: $color, Price: $price";
    }
}

$car = new Car(brand: "Toyota", price: 15000);
// Brand: Toyota, Color: black, Price: 15000
```

### 不适用于变长参数（Variadic Parameters）

```php
function addNumbers(int ...$numbers) {
    return array_sum($numbers);
}

echo addNumbers(numbers: 1, 2, 3, 4); // 语法错误
```

正确用法：

```php
echo addNumbers(1, 2, 3, 4); // 输出 10
```

### 用于魔术方法 __call()

```php
class Test {
    public function __call($name, $arguments) {
        print_r($arguments);
    }
}

$obj = new Test();
$obj->someMethod(param1: "Hello", param2: "World"); 
```

### 命名参数 vs 关联数组

在 `PHP 8` 之前，可以用 关联数组 传递参数：

```php
function registerUser($data) {
    echo "Name: {$data['name']}, Age: {$data['age']}";
}

registerUser(['name' => 'Alice', 'age' => 25]);
```

`PHP 8` 命名参数更优雅：

```php
function registerUser($name, $age) {
    echo "Name: $name, Age: $age";
}

registerUser(name: "Alice", age: 25);
```

命名参数更直观，避免数组拼写错误，减少 `isset()` 检查
