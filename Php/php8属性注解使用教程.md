### 简介

`PHP 8` 引入了 属性（`Attributes`）作为新的元数据机制，用于替代传统的 `PHPDoc` 注解，使得代码更具类型安全性和结构化。

### 基本语法

`PHP 8` 的属性（`Attributes`）使用 `#[...]` 语法表示，并可以用于类、方法、属性、参数、常量等。

#### 定义属性

属性的本质是一个 `PHP` 类，通常以 `Attribute` 特性（`flag`）标记：

```php
#[Attribute] // 这是一个属性定义
class MyAttribute {
    public function __construct(public string $name) {}
}
```

#### 不带 __construct() 的空类

```php
#[Attribute]
class SimpleAttribute {}
```

使用示例：

```php
#[SimpleAttribute]
class AnotherClass {}
```

#### 使用属性

定义好 `MyAttribute` 之后，可以在类、方法、属性等地方使用：

```php
#[MyAttribute("Hello World")]
class MyClass {}
```

### 属性应用范围

`PHP 8` 允许在不同的地方使用属性，包括：

* 类

* 类的属性

* 类的方法

* 方法参数

* 常量

#### 应用到类

```php
#[MyAttribute("This is a class")]
class DemoClass {}
```

#### 应用到属性

```php
class User {
    #[MyAttribute("User ID")]
    public int $id;
}
```

#### 应用到方法

```php
class MyController {
    #[MyAttribute("This is a method")]
    public function myMethod() {}
}
```

#### 应用到方法参数

```php
class Test {
    public function greet(#[MyAttribute("Parameter annotation")] string $name) {
        echo "Hello, $name";
    }
}
```

#### 应用到类常量

```php
class Status {
    #[MyAttribute("Status Active")]
    public const ACTIVE = 1;
}
```

### 解析属性

`PHP` 提供了 `Reflection` 机制来获取属性信息。

#### 获取类的属性

```php
$reflection = new ReflectionClass(MyClass::class);
$attributes = $reflection->getAttributes();

foreach ($attributes as $attribute) {
    $instance = $attribute->newInstance();
    echo $instance->name; // 输出: Hello World
}
```

#### 获取常量的属性

```php
$reflectionConstant = new ReflectionClassConstant(MyClass::class, 'MY_CONST');
$attributesConstant = $reflectionConstant->getAttributes();
foreach ($attributesConstant as $attribute) {
    $instance = $attribute->newInstance();
    echo $instance->name . "\n";
}
```

#### 获取属性的属性

```php
$reflection = new ReflectionProperty(User::class, 'id');
$attributes = $reflection->getAttributes();

foreach ($attributes as $attribute) {
    $instance = $attribute->newInstance();
    echo $instance->name;
}
```

#### 获取方法的属性

```php
$reflection = new ReflectionMethod(MyController::class, 'myMethod');
$attributes = $reflection->getAttributes();

foreach ($attributes as $attribute) {
    $instance = $attribute->newInstance();
    echo $instance->name;
}
```

#### 获取方法参数的属性

```php
$reflectionMethod = new ReflectionMethod(MyClass::class, 'greet');
$parameters = $reflectionMethod->getParameters();
foreach ($parameters as $parameter) {
    $attributes = $parameter->getAttributes();
    foreach ($attributes as $attribute) {
        $instance = $attribute->newInstance();
        echo $instance->name . "\n";
    }
}
```

### 高级用法

#### 指定属性的适用范围

`PHP` 提供了 `Attribute::TARGET_*` 来限定属性可以应用的位置。

```php
#[Attribute(Attribute::TARGET_CLASS | Attribute::TARGET_METHOD)]
class OnlyForClassAndMethod {}
```

这样，`OnlyForClassAndMethod` 只能用于类和方法，如果用于属性，则会报错。

#### 应用同一个属性多次

```php
#[MyAttribute("First"), MyAttribute("Second")]
class Example {}
```

注意：同一个属性应用了多次，则需要属性本身支持 `Attribute::IS_REPEATABLE` 可重复应用的目标

#### 同时应用多个不同的属性

`PHP 8` 允许在同一元素（类、方法、属性等）上使用多个不同的属性，只需使用 多个 `#[...]` 语法 或 在 `#[...]` 内逗号分隔多个属性。

* 多个 `#[...]` 语法

```php
#[Attribute]
class FirstAttribute {}

#[Attribute]
class SecondAttribute {}

#[FirstAttribute]
#[SecondAttribute]
class MultiAttributeClass {}
```

* 在同一个 `#[...]` 里使用逗号分隔

```php
#[FirstAttribute, SecondAttribute]
class AnotherMultiAttributeClass {}
```

#### 带参数的属性

```php
#[Attribute]
class Route {
    public function __construct(public string $path, public string $method = "GET") {}
}

#[Route("/home", "GET")]
class HomeController {}
```

解析：

```php
$reflection = new ReflectionClass(HomeController::class);
$attributes = $reflection->getAttributes();

foreach ($attributes as $attribute) {
    $instance = $attribute->newInstance();
    echo "Path: " . $instance->path . ", Method: " . $instance->method;
}
```

#### 同时使用带参数和不带参数的属性

```php
#[Attribute]
class Role {
    public function __construct(public string $roleName) {}
}

#[Attribute]
class Loggable {}

#[Role("Admin"), Loggable]
class UserService {}
```

解析：

```php
$reflection = new ReflectionClass(UserService::class);
$attributes = $reflection->getAttributes();

foreach ($attributes as $attribute) {
    $instance = $attribute->newInstance();
    if ($instance instanceof Role) {
        echo "Role: " . $instance->roleName . PHP_EOL;
    } else {
        echo "Attribute: " . $attribute->getName() . PHP_EOL;
    }
}
```

输出：

```php
Role: Admin
Attribute: Loggable
```

### 实际应用场景

#### 路由映射（模拟 Laravel 路由）

```php
#[Attribute]
class Route {
    public function __construct(public string $path, public string $method = "GET") {}
}

class MyController {
    #[Route('/users', 'GET')]
    public function getUsers() {}

    #[Route('/users', 'POST')]
    public function createUser() {}
}

// 解析控制器的方法路由
$reflection = new ReflectionClass(MyController::class);
foreach ($reflection->getMethods() as $method) {
    foreach ($method->getAttributes(Route::class) as $attribute) {
        $route = $attribute->newInstance();
        echo "Method: {$route->method}, Path: {$route->path}" . PHP_EOL;
    }
}
```

输出：

```php
Method: GET, Path: /users
Method: POST, Path: /users
```