### 简介

`ref struct` 是 `C# 7.2` 引入的一种特殊结构体类型，
它与普通 `struct` 的最大区别是 严格限制其分配位置：

`ref struct` 只能分配在栈（`stack`）上，不能分配在堆（`heap`）上。

⚡ 设计初衷

* 提高性能：栈分配比堆分配快，并且无需 `GC` 回收。

* 提供安全的内存访问：保证生命周期受控，防止内存泄漏和悬空引用。

* 适用于需要直接操作内存的场景，例如 `Span<T>`、`ReadOnlySpan<T>`。

#### 关键特性

* 只能分配在栈上，不能分配在堆上

* 不能作为类的字段

* 不能实现接口

* 不能装箱

* 不能作为异步方法或迭代器的局部变量

### 基本语法

```csharp
public ref struct MyStruct
{
    public int X;
    public int Y;

    public void Print() => Console.WriteLine($"{X}, {Y}");
}
```

### 与普通 struct 的区别

| 特性            | `struct`                     | `ref struct`                  |
| --------------- | ---------------------------- | ----------------------------- |
| 分配位置        | 栈或堆（例如在类中或装箱时） | **只能栈分配**                |
| 装箱（boxing）  | 支持（可转为 `object`）      | ❌ 禁止                        |
| 接口实现        | 支持                         | ❌ 禁止（不能实现接口）        |
| 异步方法/迭代器 | 支持                         | ❌ 不能被 `async`/`yield` 捕获 |
| 闭包捕获        | 支持                         | ❌ 禁止                        |
| 泛型约束        | 可作为泛型参数               | ❌ 禁止用作类泛型参数          |
| 生命周期        | 受 GC 管理                   | **完全受栈作用域约束**        |

`ref struct` 的限制确保它 不会被错误地提升到堆中，保证其生命周期安全。

### 使用场景

`ref struct` 非常适合以下 高性能、低开销 的场景：

| 场景               | 示例                                               |
| ------------------ | -------------------------------------------------- |
| **内存切片**       | `Span<T>`、`ReadOnlySpan<T>`                       |
| **避免 GC**        | 高频分配和释放的临时数据结构                       |
| **非托管资源访问** | 指针操作、`stackalloc` 分配的缓冲区                |
| **网络与数据解析** | 高性能序列化/反序列化（如 JSON、Protocol Buffers） |

### 典型示例

#### `Span<T>`：最常见的 ref struct

`Span<T>` 是一个表示连续内存区域的类型：

```csharp
Span<int> numbers = stackalloc int[5] { 1, 2, 3, 4, 5 };
numbers[2] = 99;

foreach (var n in numbers)
    Console.Write($"{n} "); // 输出: 1 2 99 4 5
```

* `stackalloc` 在栈上分配内存。

* `Span<T>` 只能存在于当前方法栈中，离开作用域自动回收。

#### 自定义 ref struct

```csharp
public ref struct Point
{
    public int X;
    public int Y;

    public double Length => Math.Sqrt(X * X + Y * Y);
}

void Demo()
{
    var p = new Point { X = 3, Y = 4 };
    Console.WriteLine(p.Length); // 5
}
```

#### 与 stackalloc 配合

```csharp
public static Span<byte> CreateBuffer()
{
    Span<byte> buffer = stackalloc byte[1024]; // 栈上分配 1KB
    buffer[0] = 42;
    return buffer; // ❌ 错误：不能返回 ref struct
}
```

返回 `Span<T>` 会导致栈内存逃逸，因此编译器会报错。

### 编译器施加的约束

`ref struct` 的安全限制主要有以下几点：

#### 不能装箱

```csharp
ref struct MyStruct { }
object o = new MyStruct(); // ❌ 编译错误
```

因为装箱会将值类型复制到堆上。

#### 不能实现接口

```csharp
ref struct MyStruct : IDisposable { } // ❌ 编译错误
```

接口调用可能导致提升到堆，破坏生命周期安全。

#### 不能作为类字段

```csharp
class MyClass
{
    public Span<int> SpanField; // ❌ 编译错误
}
```

因为类实例在堆上，而 `ref struct` 只能存在栈上。

#### 不能用作泛型参数

```csharp
List<Span<int>> list = new(); // ❌ 编译错误
```

#### 不能捕获到闭包

```csharp
Span<int> span = stackalloc int[10];
Action action = () => Console.WriteLine(span[0]); // ❌ 编译错误
```

闭包会将变量提升到堆中，破坏生命周期。

#### 不能用于异步方法/迭代器

```csharp
async Task Demo()
{
    Span<int> span = stackalloc int[10]; // ❌ 编译错误
    await Task.Delay(1000);
}
```

异步状态机会导致变量在堆上存储。

### 与其他类型对比

| 特性      | `class`         | `struct`    | `ref struct`                 |
| --------- | --------------- | ----------- | ---------------------------- |
| 分配位置  | 堆              | 栈/堆       | **仅栈**                     |
| 内存回收  | GC              | 自动回收/GC | 自动回收（方法退出时）       |
| 接口实现  | ✅               | ✅           | ❌                            |
| 装箱/拆箱 | ❌（本身是引用） | ✅           | ❌                            |
| 异步/闭包 | ✅               | ✅           | ❌                            |
| 典型代表  | `String`        | `DateTime`  | `Span<T>`, `ReadOnlySpan<T>` |

### 性能优势

| 场景           | 普通 `struct`  | `ref struct`         |
| -------------- | -------------- | -------------------- |
| 分配/释放速度  | 快             | **最快（仅栈操作）** |
| GC 压力        | 可能有（装箱） | **无 GC**            |
| 内存局部性     | 较好           | **最佳**             |
| 生命周期可控性 | GC 管理        | **作用域结束即释放** |

### 实战示例：高性能字符串切片

```csharp
public static int ParseDigits(ReadOnlySpan<char> span)
{
    int value = 0;
    foreach (var c in span)
    {
        if (!char.IsDigit(c)) break;
        value = value * 10 + (c - '0');
    }
    return value;
}

void Demo()
{
    string input = "12345abc";
    var slice = input.AsSpan(0, 5); // 直接操作原字符串内存
    Console.WriteLine(ParseDigits(slice)); // 输出 12345
}
```

优势：

* 不会产生 `Substring` 带来的额外堆分配。

* 内存安全且性能接近指针操作。

### 总结

| 方面     | 说明                                                                |
| -------- | ------------------------------------------------------------------- |
| 核心特性 | 只能分配在栈上，生命周期由作用域严格控制，无 GC 压力                |
| 主要限制 | 不能装箱、不能作为类字段、不能捕获闭包、不能异步/迭代、不能实现接口 |
| 典型应用 | `Span<T>`、`ReadOnlySpan<T>`、高性能内存处理、网络数据解析          |
| 最佳实践 | 使用 `using` 范围、`readonly` 修饰、避免逃逸、短生命周期            |
