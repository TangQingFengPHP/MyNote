### 简介

`struct` 是 值类型（`Value Type`），用于封装一组相关的数据。
与类（`class`）相比，结构体通常更轻量，适用于小型、短生命周期的对象。

⚡ 关键特点：

* 存储在 栈（`stack`）上（也可能嵌套在堆中，但本质仍是值类型）。

* 按值传递（赋值/参数传递时会复制整个结构）。

* 无需垃圾回收（`GC`），生命周期由作用域决定。

* 可包含字段、属性、方法、构造函数、运算符重载等。

### 基本语法

```csharp
public struct Point
{
    public int X;
    public int Y;

    public Point(int x, int y)
    {
        X = x;
        Y = y;
    }

    public void Move(int dx, int dy)
    {
        X += dx;
        Y += dy;
    }
}
```

使用：

```csharp
Point p1 = new Point(3, 4);
p1.Move(1, 2);
Console.WriteLine($"{p1.X}, {p1.Y}"); // 4, 6
```

### 内存模型与分配

| 分配场景       | 存储位置 | 说明                                 |
| -------------- | -------- | ------------------------------------ |
| **局部变量**   | 栈       | 最典型场景，生命周期由方法作用域决定 |
| **作为类字段** | 堆       | 包含在类对象内部，但值本身在堆内存中 |
| **数组元素**   | 堆       | 数组本身在堆上，每个元素紧密排列     |

即使 `struct` 位于堆中（例如类字段、数组元素），它仍然是值类型，操作时是按值复制。

### 与 class 的区别

| 特性         | `class`(引用类型)      | `struct`(值类型)                   |
| ------------ | ---------------------- | ---------------------------------- |
| 内存分配     | 堆（GC 管理）          | 栈/堆中嵌入                        |
| 传递方式     | 按引用传递（复制引用） | 按值传递（复制整个对象）           |
| 默认构造函数 | 可定义无参/有参        | 系统提供无参构造，用户**不能定义** |
| 继承         | 支持继承/虚方法        | 不能继承，只能实现接口             |
| 空值（null） | 可以为 null            | 不可为 null（除非 `Nullable<T>`）  |
| 装箱/拆箱    | 不涉及                 | 赋值给 `object` 时会装箱           |
| 性能         | 分配慢（GC）           | 分配快，生命周期短时更高效         |


### 构造函数与初始化

#### 默认构造函数

* 所有结构体都有一个隐式的无参构造函数，将所有字段初始化为 默认值（如 `0`、`null`）。

* 用户不能显式定义无参构造函数（`C# 10` 开始支持，但有限制）。

#### 自定义构造函数

可以定义有参构造函数，但必须初始化所有字段。

```csharp
public Point(int x, int y)
{
    X = x;          // 必须全部赋值
    Y = y;
}
```

#### 结构体的成员

结构体可以包含：

* 字段（`Field`）

* 属性（`Property`）

* 方法（`Method`）

* 事件（`Event`）

* 运算符重载

* 静态成员

* 构造函数（但无参构造受限）

不支持：

* 继承其他类/结构体（只能实现接口）。

* 明确的析构函数（`~Destructor`）。

### 值类型的特性

#### 按值传递

结构体赋值时会复制整个值：

```csharp
Point p1 = new Point(1, 2);
Point p2 = p1;   // 复制
p2.X = 99;
Console.WriteLine(p1.X); // 1，不受 p2 修改影响
```

#### 装箱/拆箱

赋值给 `object` 或接口时，会将结构体复制到堆中：

```csharp
Point p = new Point(1, 2);
object obj = p;      // 装箱：复制到堆
Point p2 = (Point)obj; // 拆箱：从堆复制回栈
```

装箱/拆箱会带来性能开销，应尽量避免频繁发生。

#### 不可变结构体（Immutable struct）

为了避免复制带来的副作用，结构体推荐设计为不可变：

```csharp
public readonly struct Point
{
    public int X { get; }
    public int Y { get; }
    public Point(int x, int y) => (X, Y) = (x, y);
}
```

#### 实现接口

```csharp
public interface IDrawable
{
    void Draw();
}

public struct Circle : IDrawable
{
    public double Radius { get; }
    public Point Center { get; }
    
    public Circle(double radius, Point center)
    {
        Radius = radius;
        Center = center;
    }
    
    public void Draw()
    {
        Console.WriteLine($"Drawing circle at {Center} with radius {Radius}");
    }
    
    public double Area => Math.PI * Radius * Radius;
}
```

```csharp
struct Rectangle : IEquatable<Rectangle>
{
    public int Width, Height;

    public bool Equals(Rectangle other)
    {
        return Width == other.Width && Height == other.Height;
    }

    public override bool Equals(object obj) => obj is Rectangle other && Equals(other);
    public override int GetHashCode() => HashCode.Combine(Width, Height);
}

var r1 = new Rectangle { Width = 10, Height = 20 };
var r2 = new Rectangle { Width = 10, Height = 20 };
Console.WriteLine(r1.Equals(r2)); // 输出: True
```

#### 运算符重载示例

```csharp
struct Point
{
    public int X, Y;

    public Point(int x, int y)
    {
        X = x;
        Y = y;
    }

    public static Point operator +(Point a, Point b)
    {
        return new Point(a.X + b.X, a.Y + b.Y);
    }

    public override string ToString() => $"({X}, {Y})";
}

var p1 = new Point(1, 2);
var p2 = new Point(3, 4);
var sum = p1 + p2;
Console.WriteLine(sum); // 输出: (4, 6)
```

### 结构体的新特性

#### readonly struct

* 表示完全不可变。

* 编译器强制所有字段为 `readonly`。

* 防止无意修改。

```csharp
public readonly struct Vector
{
    public readonly double X;
    public readonly double Y;
}
```

#### ref struct

* 只能在栈上分配。

* 用于高性能场景，如 `Span<T>`。

#### record struct 

值类型的 记录类型，支持内建的值比较、`with` 表达式。

```csharp
public record struct Point(int X, int Y);
var p1 = new Point(1,2);
var p2 = p1 with { X = 3 }; // 基于值的复制
```

### 高级用法

#### 与模式匹配结合使用

```csharp
public struct RGBColor
{
    public byte R;
    public byte G;
    public byte B;
    
    public RGBColor(byte r, byte g, byte b)
    {
        R = r;
        G = g;
        B = b;
    }
    
    public void Deconstruct(out byte r, out byte g, out byte b)
    {
        r = R;
        g = G;
        b = B;
    }
}

class Program
{
    static string GetColorName(RGBColor color)
    {
        return color switch
        {
            (255, 0, 0) => "Red",
            (0, 255, 0) => "Green",
            (0, 0, 255) => "Blue",
            (255, 255, 255) => "White",
            (0, 0, 0) => "Black",
            _ => "Unknown"
        };
    }
    
    static void Main()
    {
        RGBColor red = new RGBColor(255, 0, 0);
        Console.WriteLine(GetColorName(red)); // 输出: Red
    }
}
```

### 适用场景

✅ 适合：

* 轻量级、小数据对象（坐标、颜色、矩形、数值容器等）。

* 高性能计算（避免 `GC` 开销）。

* 频繁创建销毁的对象。

❌ 不适合：

* 对象非常大（复制开销高）。

* 需要继承层次。

* 需要引用语义（共享同一对象）。

> 如果类型大小超过 16 字节，且频繁传递，建议使用 `class` 而不是 `struct`。

### 实战示例

#### 高性能点计算

```csharp
public readonly struct Point
{
    public readonly double X;
    public readonly double Y;
    public Point(double x, double y) => (X, Y) = (x, y);

    public double Distance(Point other) =>
        Math.Sqrt(Math.Pow(X - other.X, 2) + Math.Pow(Y - other.Y, 2));
}

void Demo()
{
    Point p1 = new(1, 2);
    Point p2 = new(4, 6);
    Console.WriteLine(p1.Distance(p2)); // 5
}
```

* 无 `GC` 压力。

* 值传递保证线程安全。

### 总结

| 特性       | 说明                                             |
| ---------- | ------------------------------------------------ |
| 类型       | 值类型（Value Type）                             |
| 内存分配   | 栈分配或嵌入堆中                                 |
| 继承       | 不能继承其他类/结构体，只能实现接口              |
| 构造函数   | 系统提供无参构造；用户可定义有参构造             |
| 默认初始化 | 所有字段自动初始化为默认值                       |
| 传递方式   | 按值传递（复制整个对象）                         |
| 典型场景   | 小型数据对象（Point、Color）、高性能计算         |
| 高级特性   | `readonly struct`、`ref struct`、`record struct` |
| 设计建议   | 尽量保持不可变，避免装箱，避免过大对象           |
