### 简介

`PHP 8` 引入了 `?->`（`Nullsafe` 操作符），用于简化 `null` 检查，减少繁琐的 `if` 语句或 `isset()` 代码，提高可读性。

### ?-> Nullsafe 操作符的作用

在 `PHP 7` 及以下，访问对象的属性或方法时，如果对象是 `null`，会导致致命错误 (`Fatal error`)：

```php
$person = null;
echo $person->name; // Fatal error: Uncaught Error: Trying to get property of non-object
```

解决方案（传统写法）：

```php
$person = null;
echo isset($person) ? $person->name : null;
```

`PHP 8` 解决方案（`?->`）：

```php
$person = null;
echo $person?->name; // 不会报错，直接返回 null
```

### ?-> 基本用法

#### 访问对象的属性

```php
class Person {
    public string $name = "John";
}

$person = new Person();
echo $person?->name; // 输出 "John"

$person = null;
echo $person?->name; // 输出 null，不会报错
```

#### 访问对象的方法

```php
class User {
    public function getName() {
        return "Alice";
    }
}

$user = new User();
echo $user?->getName(); // 输出 "Alice"

$user = null;
echo $user?->getName(); // 输出 null，不会报错
```

#### 访问嵌套对象

```php
class Address {
    public string $city = "New York";
}

class Person {
    public ?Address $address = null;
}

$person = new Person();
echo $person->address?->city; // 输出 null，不会报错

$person->address = new Address();
echo $person->address?->city; // 输出 "New York"
```

#### ?-> 结合数组

不能用于数组索引（`[]`），但可以用于 `ArrayAccess` 对象

```php
$data = null;
echo $data?['key']; // 语法错误：不能用于数组
```

解决方案：使用 `ArrayAccess` 对象

```php
class Collection implements ArrayAccess {
    private array $items = ['name' => 'Alice'];

    public function offsetExists($offset) { return isset($this->items[$offset]); }
    public function offsetGet($offset) { return $this->items[$offset] ?? null; }
    public function offsetSet($offset, $value) { $this->items[$offset] = $value; }
    public function offsetUnset($offset) { unset($this->items[$offset]); }
}

$collection = new Collection();
echo $collection?->offsetGet('name'); // 输出 "Alice"

$collection = null;
echo $collection?->offsetGet('name'); // 输出 null，不会报错
```

#### ?-> 结合函数返回值

```php
function getUser() {
    return null;
}

echo getUser()?->name; // 输出 null，不会报错
```

#### ?-> 结合链式调用

`PHP 8` 允许链式 `?->` 操作，简化复杂的 `null` 检查：

```php
class Department {
    public ?Person $manager = null;
}

$department = new Department();

// 传统写法
echo isset($department->manager) ? $department->manager->name : null;

// PHP 8 `?->`
echo $department?->manager?->name; // 输出 null，不会报错
```

#### ?-> 结合赋值

`?->` 不能用于赋值，只能用于访问！

```php
$person = null;

// 不能用 `?->` 进行赋值
$person?->name = "John"; // 语法错误
```

解决方案：

```php
if ($person !== null) {
    $person->name = "John";
}
```

#### ?-> 不能用于静态方法

```php
class Test {
    public static function hello() {
        return "Hello";
    }
}

echo Test?->hello(); // ❌ 语法错误
```

静态方法必须用 `::` 访问，不支持 `?->`

解决方案：

```php
echo isset(Test::hello) ? Test::hello() : null;
```

#### ?-> 和 ?? 的区别

`?->` 用于对象，`??` 用于 `null` 合并

```php
$person = null;

// `?->` 适用于对象
echo $person?->name; // 返回 null

// `??` 适用于变量为空时提供默认值
echo $person?->name ?? "Default Name"; // 输出 "Default Name"
```

* `?->` 用于安全访问对象的属性或方法。

* `??` 用于 `null` 合并，提供默认值。