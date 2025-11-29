### 简介

`Record Structs` 是一种值类型的记录（`record`），结合了 `struct` 的值语义和 `record` 的功能（如自动生成相等性比较、不可变性支持）。它们是 `C# 9.0` 中引入的引用类型 `record`（默认 `class`）的扩展，专为性能敏感场景设计，特别是在需要栈分配或避免 `GC` 压力的情况下。

#### 核心特性

* 值类型：存储在栈上（除非装箱），避免堆分配，适合小数据结构。

* 不可变性：默认鼓励不可变设计（通过 `init-only` 属性），但可选择可变。

* 值相等性：自动实现基于内容的相等性比较（== 和 `Equals`）。

* 自动 `ToString`：生成人类可读的字符串表示。

* 解构支持：自动提供 `Deconstruct` 方法，方便模式匹配和解构。

* `With` 表达式：支持非破坏性变异（创建新实例）。

* 继承支持：支持继承其他 `record structs`（但不能继承 `class` 或 `record class`）。

### 基本语法

```csharp
// 最简单的记录结构声明
public record struct Point(int X, int Y);

// 等价于传统的结构体声明（但功能更强大）
public struct Point
{
    public int X { get; init; }
    public int Y { get; init; }
    
    public Point(int x, int y)
    {
        X = x;
        Y = y;
    }
    
    // 自动生成的 Equals、GetHashCode、ToString 等方法
}
```

#### 可变性控制

```csharp
// 不可变记录结构 (推荐)
public readonly record struct ImmutablePoint(int X, int Y);

// 可变记录结构
public record struct MutablePoint
{
    public int X { get; set; }
    public int Y { get; set; }
}

// 混合可变性
public record struct MixedPoint(int X, int Y) // X 和 Y 默认只读
{
    public int Z { get; set; } // 额外的可变属性
}
```

### 记录结构 vs 记录类 vs 普通结构体

| 特性          | 记录结构 (record struct) | 记录类 (record class) | 普通结构体 (struct) |
| ------------- | ------------------------ | --------------------- | ------------------- |
| 类型          | 值类型                   | 引用类型              | 值类型              |
| 分配位置      | 栈（通常）               | 堆                    | 栈（通常）          |
| 默认不可变    | 是（属性为 `init`）      | 是（属性为 `init`）   | 否（可变）          |
| 值相等性      | 自动实现                 | 自动实现              | 需手动实现          |
| `with` 表达式 | 支持                     | 支持                  | 不支持              |
| 解构          | 自动支持                 | 自动支持              | 需手动实现          |
| 继承          | 不支持                   | 支持                  | 不支持              |

### 记录结构的核心特性

#### 位置记录结构（Positional Record Structs）

```csharp
// 位置记录结构 - 最简洁的形式
public record struct Person(string FirstName, string LastName, int Age);

// 使用示例
var person = new Person("John", "Doe", 30);
Console.WriteLine(person); // 输出: Person { FirstName = John, LastName = Doe, Age = 30 }

// 解构
var (firstName, lastName, age) = person;
Console.WriteLine($"{firstName} {lastName}, {age}岁");
```

#### 值相等性（Value Equality）

```csharp
public record struct Point(int X, int Y);

// 值相等性比较
var point1 = new Point(1, 2);
var point2 = new Point(1, 2);
var point3 = new Point(3, 4);

Console.WriteLine(point1 == point2); // True - 基于值比较
Console.WriteLine(point1 == point3); // False
Console.WriteLine(point1.Equals(point2)); // True
```

#### with 表达式（非破坏性变更）

```csharp
public record struct Person(string FirstName, string LastName, int Age);

var original = new Person("John", "Doe", 30);

// 创建修改后的副本（非破坏性变更）
var updated = original with { Age = 31 };
var renamed = original with { FirstName = "Jane" };

Console.WriteLine(original); // Person { FirstName = John, LastName = Doe, Age = 30 }
Console.WriteLine(updated);  // Person { FirstName = John, LastName = Doe, Age = 31 }
Console.WriteLine(renamed);  // Person { FirstName = Jane, LastName = Doe, Age = 30 }
```

#### 自定义行为

```csharp
// 自定义记录结构
public record struct Person(string FirstName, string LastName)
{
    // 添加计算属性
    public string FullName => $"{FirstName} {LastName}";
    
    // 添加方法
    public string GetFormattedName() => $"{LastName}, {FirstName}";
    
    // 重写ToString
    public override string ToString() => FullName;
    
    // 自定义相等性逻辑（可选）
    public bool Equals(Person other) => 
        FirstName == other.FirstName && LastName == other.LastName;
    
    // 自定义GetHashCode（可选）
    public override int GetHashCode() => 
        HashCode.Combine(FirstName, LastName);
}

// 使用自定义记录结构
var person = new Person("John", "Doe");
Console.WriteLine(person.FullName); // John Doe
Console.WriteLine(person.GetFormattedName()); // Doe, John
Console.WriteLine(person); // John Doe
```

### 高级用法和模式

#### 与模式匹配结合

```csharp
public record struct Point(int X, int Y);

// 在模式匹配中使用记录结构
string ClassifyPoint(Point point) => point switch
{
    (0, 0) => "原点",
    (var x, var y) when x == y => "在y=x线上",
    (var x, var y) when x > 0 && y > 0 => "第一象限",
    (var x, var y) when x < 0 && y > 0 => "第二象限",
    (var x, var y) when x < 0 && y < 0 => "第三象限",
    (var x, var y) when x > 0 && y < 0 => "第四象限",
    _ => "在坐标轴上"
};

// 使用示例
Console.WriteLine(ClassifyPoint(new Point(0, 0))); // 原点
Console.WriteLine(ClassifyPoint(new Point(3, 3))); // 在y=x线上
Console.WriteLine(ClassifyPoint(new Point(2, 4))); // 第一象限
```

#### 实现接口

```csharp
public record struct Vector2D(double X, double Y) : IFormattable
{
    public double Magnitude => Math.Sqrt(X * X + Y * Y);
    
    public string ToString(string format, IFormatProvider formatProvider)
    {
        return format?.ToUpper() switch
        {
            "M" => $"({X}, {Y}) with magnitude {Magnitude:F2}",
            _ => $"({X}, {Y})"
        };
    }
}

// 使用接口实现
var vector = new Vector2D(3, 4);
Console.WriteLine(vector.ToString("M", CultureInfo.InvariantCulture));
// 输出: (3, 4) with magnitude 5.00
```

#### 集合中使用 Record Structs

```csharp
// 高性能点集处理
var points = new Point[1000];
var sum = new Point(0, 0);

for (int i = 0; i < points.Length; i++)
{
    points[i] = new Point(i, i * 2);
    sum = sum with 
    { 
        X = sum.X + points[i].X, 
        Y = sum.Y + points[i].Y 
    };
}
```

#### 与 Span 和 Memory 结合

```csharp
Span<Point> points = stackalloc Point[4];
points[0] = new(0, 0);
points[1] = new(0, 1);
points[2] = new(1, 1);
points[3] = new(1, 0);

// 高性能几何计算
double area = CalculatePolygonArea(points);
```

### 实际应用场景

#### 数学和几何计算

```csharp 
public readonly record struct Rectangle(Point TopLeft, Point BottomRight)
{
    public int Width => BottomRight.X - TopLeft.X;
    public int Height => BottomRight.Y - TopLeft.Y;
    public int Area => Width * Height;
    
    public bool Contains(Point point) =>
        point.X >= TopLeft.X && point.X <= BottomRight.X &&
        point.Y >= TopLeft.Y && point.Y <= BottomRight.Y;
    
    public Rectangle Inflate(int delta) =>
        this with 
        { 
            TopLeft = new Point(TopLeft.X - delta, TopLeft.Y - delta),
            BottomRight = new Point(BottomRight.X + delta, BottomRight.Y + delta)
        };
}

// 使用几何记录结构
var rect = new Rectangle(new Point(0, 0), new Point(10, 10));
Console.WriteLine($"面积: {rect.Area}"); // 面积: 100
Console.WriteLine($"包含点 (5,5): {rect.Contains(new Point(5, 5))}"); // True

var largerRect = rect.Inflate(2);
Console.WriteLine($"新面积: {largerRect.Area}"); // 新面积: 196
```

#### 数据传输对象（DTO）

```csharp
// API 响应DTO
public readonly record struct ApiResponse<T>(T Data, string Error, DateTime Timestamp)
{
    public bool IsSuccess => string.IsNullOrEmpty(Error);
    
    public static ApiResponse<T> Success(T data) => 
        new ApiResponse<T>(data, null, DateTime.UtcNow);
    
    public static ApiResponse<T> Failure(string error) => 
        new ApiResponse<T>(default, error, DateTime.UtcNow);
}

// 使用DTO记录结构
var successResponse = ApiResponse<string>.Success("操作成功");
var errorResponse = ApiResponse<string>.Failure("发生错误");

Console.WriteLine(successResponse.IsSuccess); // True
Console.WriteLine(errorResponse.IsSuccess);   // False
```

#### 领域模型中的值对象

```csharp
// 货币值对象
public readonly record struct Money(decimal Amount, string Currency)
{
    public static Money operator +(Money left, Money right)
    {
        if (left.Currency != right.Currency)
            throw new InvalidOperationException("货币类型不匹配");
        
        return new Money(left.Amount + right.Amount, left.Currency);
    }
    
    public static Money operator *(Money money, decimal factor) =>
        new Money(money.Amount * factor, money.Currency);
    
    public override string ToString() => $"{Amount:F2} {Currency}";
}

// 使用值对象
var price1 = new Money(100.50m, "USD");
var price2 = new Money(50.25m, "USD");
var total = price1 + price2;
var discounted = total * 0.9m;

Console.WriteLine($"总价: {total}");       // 总价: 150.75 USD
Console.WriteLine($"折扣价: {discounted}"); // 折扣价: 135.68 USD
```

### 最佳实践和注意事项

#### 何时使用记录结构

```csharp
// ✅ 适合使用记录结构的场景：
// 1. 小型、简单的数据结构
public record struct Point(int X, int Y);

// 2. 值语义重要的场景
public record struct Money(decimal Amount, string Currency);

// 3. 性能敏感的场景（避免堆分配）
public record struct Measurement(double Value, string Unit);

// 4. 需要值相等性的场景
public record struct KeyValuePair<TKey, TValue>(TKey Key, TValue Value);

// ❌ 不适合使用记录结构的场景：
// 1. 大型数据结构（>16字节）
// 2. 需要继承的场景
// 3. 需要身份标识的场景
```

### 适用场景

* 高性能游戏开发：`3D` 坐标、向量、颜色

* 科学计算：矩阵、复数、测量单位

* 金融系统：货币金额、汇率

* 数据处理管道：中间数据结构

* 设备通信：协议数据包结构

* 地理空间计算：坐标点、边界框

### 总结

`C# 10` 的记录结构是一个强大的特性，它结合了结构体的性能优势和记录的简洁性。关键要点：

* 值类型语义：记录结构是值类型，分配在栈上，性能更好

* 不可变性：默认提供不可变属性（使用 `init` 访问器）

* 值相等性：自动实现基于值的相等性比较

* 简洁语法：提供位置语法、`with` 表达式和解构功能

* 适用场景：小型数据结构、值对象、性能敏感场景