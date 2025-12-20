### 简介

`record` 是 `C# 9` 引入的新引用类型（`Reference Type`），专门用于数据导向（`Data-Oriented`）的不可变对象。特别适合用于表示不可变的数据传输对象（`DTO`）、值对象和领域模型。

⚡ 主要特性：

* 内置值相等性：两个 `record` 实例如果属性值相同，则被认为相等（值相等）。

* 简洁语法：通过“主构造函数”直接定义属性。

* 不可变设计：推荐使用 `init` 访问器，实现只读属性。

* 模式匹配友好：可以用 `with`、解构等简化数据处理。

### 基本语法

#### 简单声明

```csharp
public record Person(string FirstName, string LastName);
```

* `record` 关键字定义一个不可变引用类型。

* `(string FirstName, string LastName)` 定义主构造函数和自动 `init` 属性。

等价于：

```csharp
public class Person
{
    public string FirstName { get; init; }
    public string LastName { get; init; }

    public Person(string firstName, string lastName)
        => (FirstName, LastName) = (firstName, lastName);

    // 自动生成值相等比较、Deconstruct 方法等
}
```

使用：

```csharp
var p1 = new Person("Alice", "Smith");
Console.WriteLine(p1.FirstName); // Alice
```

### 不可变性（init-only）

`Record` 的属性默认是只读的（`init-only`），这意味着它们只能在初始化时设置

```csharp
public record Person(string FirstName, string LastName, int Age);

// 创建 record 实例
var person = new Person("John", "Doe", 30);

// 编译错误：不能修改属性
// person.Age = 31;
```

### 值相等（Value Equality）

#### 引用类型 vs 值类型对比

| 类型     | 相等比较（默认）                     |
| -------- | ------------------------------------ |
| `class`  | **引用相等**（Reference Equality）   |
| `struct` | 值相等（字段逐一比较）               |
| `record` | **值相等**（自动生成 `Equals`/`==`） |

示例：

```csharp
var p1 = new Person("Alice", "Smith");
var p2 = new Person("Alice", "Smith");
Console.WriteLine(p1 == p2);      // True
Console.WriteLine(p1.Equals(p2)); // True
```

即使 `p1` 和 `p21` 是不同实例，只要属性值相同，就相等。

### with 表达式（非破坏性复制）

`record` 可以使用 `with` 表达式复制并修改部分属性：

```csharp
var p1 = new Person("Alice", "Smith");
var p2 = p1 with { LastName = "Johnson" };

Console.WriteLine(p2.FirstName); // Alice
Console.WriteLine(p2.LastName);  // Johnson
```

* `p1` 保持不变，`p2` 是新的实例。

* 这是不可变对象的核心优势。

### 解构（Deconstruction）

编译器会自动生成 `Deconstruct` 方法：

```csharp
var person = new Person("Alice", "Smith");
var (first, last) = person;
Console.WriteLine($"{first} {last}");
```

可直接用元组解构。

### 继承与多态

`record` 支持继承，且保留值相等语义：

```csharp
public record Person(string Name);
public record Student(string Name, int Grade) : Person(Name);

Person p = new Student("Alice", 5);
Console.WriteLine(p is Student); // True
```

自动生成的 `Equals` 会包含类型检查，子类与父类不同类型即使属性相同也不相等。

### 显式属性定义

可以不用主构造函数，像 `class` 一样定义：

```csharp
public record Car
{
    public string Brand { get; init; }
    public string Model { get; init; }
}
```

与 `class` 的唯一区别是自动生成了值相等比较。

### record struct（值类型记录，C# 10+）

`C# 10` 引入 `record struct`，结合了 `record` 的值相等 和 `struct` 的值类型特性。

```csharp
// Record struct（值类型）
public record struct Point(int X, int Y);

// 使用 record struct
var point1 = new Point(3, 4);
var point2 = new Point(3, 4);

Console.WriteLine(point1 == point2); // True
Console.WriteLine(point1); // Point { X = 3, Y = 4 }

// 修改 record struct（因为是值类型，所以可以修改）
point1.X = 5;
Console.WriteLine(point1); // Point { X = 5, Y = 4 }

// 只读 record struct
public readonly record struct ImmutablePoint(int X, int Y);

var immutablePoint = new ImmutablePoint(3, 4);
// immutablePoint.X = 5; // 编译错误：不能修改只读字段
```

### 记录的成员生成

编译器为 `record` 自动生成以下内容：

* `Equals(object?)` 和 `GetHashCode()`：基于属性值。

* `==` 和 `!=` 运算符。

* `Deconstruct` 方法。

* `ToString()`：输出形如 `Person { FirstName = Alice, LastName = Smith }`。

这使得 `record` 非常适合作为 `DTO`、查询结果、配置数据。

### 与 class / struct 对比

| 特性          | `class`   | `struct`  | `record`（class） | `record struct`      |
| ------------- | --------- | --------- | ----------------- | -------------------- |
| 类型          | 引用类型  | 值类型    | 引用类型          | 值类型               |
| 相等性        | 引用相等  | 值相等    | **值相等**        | **值相等**           |
| 默认不可变    | ❌（可变） | ❌（可变） | ✅（推荐 `init`）  | ✅（推荐 `readonly`） |
| 内存分配      | 堆        | 栈/堆     | 堆                | 栈/堆                |
| 继承          | 支持      | 仅接口    | 支持              | 仅接口               |
| `with` 表达式 | ❌         | ❌         | ✅                 | ✅                    |

### 高级用法

#### 与模式匹配

```csharp
if (person is Person { FirstName: "Alice" })
    Console.WriteLine("Found Alice!");
```

```csharp
public abstract record Shape;
public record Circle(double Radius) : Shape;
public record Rectangle(double Width, double Height) : Shape;
public record Triangle(double Base, double Height) : Shape;

public static double CalculateArea(Shape shape)
{
    return shape switch
    {
        Circle c => Math.PI * c.Radius * c.Radius,
        Rectangle r => r.Width * r.Height,
        Triangle t => 0.5 * t.Base * t.Height,
        _ => throw new ArgumentException("Unknown shape")
    };
}

// 使用模式匹配
var circle = new Circle(5);
var rectangle = new Rectangle(4, 6);

Console.WriteLine(CalculateArea(circle)); // 78.53981633974483
Console.WriteLine(CalculateArea(rectangle)); // 24
```

#### 位置记录 + 解构

```csharp
public record Order(int Id, decimal Price);
var order = new Order(1001, 99.99m);
var (id, price) = order; // 自动解构
```

### 实际应用场景

#### DTO（数据传输对象）

```csharp
// API 响应 DTO
public record ApiResponse<T>(bool Success, string Message, T Data);

// API 请求 DTO
public record CreateUserRequest(string Username, string Email, string Password);

// 使用 record DTO
public class UserController : ControllerBase
{
    [HttpPost]
    public IActionResult CreateUser(CreateUserRequest request)
    {
        // 处理请求...
        var response = new ApiResponse<User>(true, "User created successfully", user);
        return Ok(response);
    }
}
```

#### 配置对象

```csharp
// 应用程序配置
public record AppSettings
{
    public string ConnectionString { get; init; }
    public int MaxRetryAttempts { get; init; } = 3;
    public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
    public LoggingSettings Logging { get; init; }
}

public record LoggingSettings
{
    public string Level { get; init; } = "Information";
    public string FilePath { get; init; } = "logs/app.log";
}

// 使用配置
var settings = new AppSettings
{
    ConnectionString = "Server=localhost;Database=MyDb",
    MaxRetryAttempts = 5,
    Logging = new LoggingSettings { Level = "Debug" }
};
```

### 适用场景

推荐使用 `record` 的场景：

* 不可变的数据模型：`DTO`、`API` 响应对象

* 配置项、设置类

* 数据库查询结果（`EF Core` 中很常见）

* 事件/消息模型（`Event Sourcing`、`CQRS`）

不推荐使用 `record` 的场景：

* 对象需要频繁修改。

* 需要严格的引用语义（例如缓存中的唯一实例）。

### 性能与注意事项

* `record` 是引用类型（除非是 `record struct`），所以在堆上分配，仍有 `GC` 开销。

* 值比较可能略微增加 `Equals/GetHashCode` 的计算成本。

* `with` 表达式会创建新对象，不要在高频场景频繁使用。

### 总结

| 特性     | 说明                                         |
| -------- | -------------------------------------------- |
| 核心目标 | **简化不可变数据类型**的声明和比较           |
| 类型     | 引用类型（默认）/值类型（`record struct`）   |
| 相等性   | 自动实现**基于值的相等性**（`Equals`、`==`） |
| 不可变性 | 推荐使用 `init` 或 `readonly`                |
| 复制     | `with` 表达式实现非破坏性复制                |
| 解构     | 自动生成 `Deconstruct`，支持元组解构         |
| 典型应用 | DTO、API 数据模型、配置对象、事件/消息对象   |
