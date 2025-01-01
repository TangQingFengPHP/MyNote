### 简介

`Lambda` 表达式是 `C#` 中简洁表达匿名方法的一种方式，常用于函数式编程风格，例如 `LINQ`、委托、事件处理等场景。`Lambda` 表达式的语法紧凑，便于编写和阅读代码。

### 基础语法：

`(parameter_list) => expression`

* 参数列表：可以为空：如 `()`，也可以包含一个或多个参数。

* 箭头符号 `=>`：分隔参数和表达式。

* 表达式或代码块：右侧可以是单个表达式，也可以是包含多行代码的代码块。

### 示例用法

#### 基本示例

```csharp
// 带一个参数的 Lambda 表达式
Func<int, int> square = x => x * x;
Console.WriteLine(square(5)); // 输出 25

// 带两个参数的 Lambda 表达式
Func<int, int, int> add = (x, y) => x + y;
Console.WriteLine(add(3, 4)); // 输出 7
```

#### 多行代码的 `Lambda` 表达式

> 如果右侧是代码块，需要使用大括号 `{}` 并显式使用 `return` 返回值。

```csharp
Func<int, int, int> multiply = (x, y) => 
{
    Console.WriteLine($"Multiplying {x} and {y}");
    return x * y;
};
Console.WriteLine(multiply(3, 4)); // 输出 12
```

#### `Lambda` 表达式与委托

> `Lambda` 表达式可以直接赋值给委托类型。

1. 使用内置委托类型

```csharp
Func<int, int> doubleValue = x => x * 2;  // Func 委托
Console.WriteLine(doubleValue(10)); // 输出 20

Action<string> printMessage = message => Console.WriteLine(message);  // Action 委托
printMessage("Hello, Lambda!"); // 输出 Hello, Lambda!
```

2. 使用自定义委托

```csharp
delegate int Operation(int a, int b);

Operation subtract = (x, y) => x - y;
Console.WriteLine(subtract(10, 5)); // 输出 5
```

#### `Lambda` 表达式与 `LINQ`

> `Lambda` 表达式在 `LINQ` 查询中被广泛使用。

```csharp
// 简单的 LINQ 查询
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// 筛选出大于 3 的数字
var result = numbers.Where(x => x > 3);

foreach (var num in result)
{
    Console.WriteLine(num); // 输出 4 和 5
}

// 使用 Select 转换数据
var numbers = new List<int> { 1, 2, 3 };

// 将每个数字平方
var squares = numbers.Select(x => x * x);

foreach (var square in squares)
{
    Console.WriteLine(square); // 输出 1, 4, 9
}
```

#### 捕获外部变量

> `Lambda` 表达式可以捕获定义时的外部变量（称为“闭包”）。

```csharp
int multiplier = 3;
Func<int, int> multiply = x => x * multiplier;

Console.WriteLine(multiply(10)); // 输出 30

multiplier = 5;
Console.WriteLine(multiply(10)); // 输出 50
```

#### 使用 `Lambda` 表达式处理事件

```csharp
Button button = new Button();
button.Click += (sender, e) => Console.WriteLine("Button clicked!");
```

#### 结合 `Task` 和异步操作

```csharp
Func<int, Task<int>> asyncOperation = async x =>
{
    await Task.Delay(1000);
    return x * x;
};

var result = await asyncOperation(5);
Console.WriteLine(result); // 输出 25
```

### `Lambda` 表达式的优缺点

* 优点：

1. 简洁明了：代码更短、更清晰。

2. 功能强大：支持匿名函数、闭包、内联逻辑等功能。

3. 与 `LINQ` 无缝集成：在数据查询和操作中简化代码。

* 缺点：

1. 可读性：对于复杂的表达式，可能难以理解。

2. 调试困难：`Lambda` 表达式中的逻辑难以单步调试。

3. 性能开销：虽然开销很小，但 `Lambda` 表达式需要通过委托间接调用。

### 注意事项

1. 简化代码：尽量保持 `Lambda` 表达式简单。如果逻辑过于复杂，可以提取到单独的方法中。

2. 性能考虑：在高性能场景下，避免频繁创建 `Lambda` 表达式。

3. 调试技巧：通过 `Debug.WriteLine` 或将 `Lambda` 表达式拆分为多行辅助调试。