### 简介

内联数组是 `C# 12` 和 `.NET 8` 中引入的一个高级特性，它允许开发者创建固定大小的、在栈上分配或内联在结构体中的数组。这个特性主要用于高性能场景，可以避免堆分配和垃圾回收的开销。

#### 性能优势

内联数组的主要优势在于性能：

* 栈上分配：避免堆分配和垃圾回收

* 内存局部性：元素在内存中连续存储，提高缓存命中率

* 减少指针间接寻址：直接访问元素，不需要通过数组对象引用

#### 内联数组 vs 传统数组

| 特性       | 内联数组                 | 传统数组             |
| ---------- | ------------------------ | -------------------- |
| 内存位置   | 栈/包含结构体内存        | 托管堆               |
| 分配开销   | 无额外分配               | 需要堆分配           |
| 最大长度   | 受栈空间限制（通常≤1MB） | 受GC限制（通常≤2GB） |
| 适用场景   | 高性能计算/嵌入式开发    | 通用场景             |
| 灵活性     | 固定长度                 | 动态长度             |
| C#版本要求 | ≥ C# 12 (.NET 8+)        | 所有版本             |

### 基本语法

内联数组使用 `InlineArray` 特性和特定的模式来定义：

```csharp
using System.Runtime.CompilerServices;

[InlineArray(Size)]
struct InlineArrayStruct
{
    private T _element; // 单一字段，表示数组的起点
}
```

* `Size`：一个编译时常量，表示数组的固定长度。

* `T`：数组元素的类型（可以是值类型或引用类型）。

* 单一字段：结构体中只能有一个字段，表示数组的起点，编译器会将其扩展为固定大小的连续内存。

```csharp
[System.Runtime.CompilerServices.InlineArray(10)]
public struct MyInlineArray
{
    private int _element0; // 只需要定义一个字段，实际大小由特性指定
}
```

### 创建和使用内联数组

#### 基本定义和使用

```csharp
using System;
using System.Runtime.CompilerServices;

// 定义包含10个整数的内联数组
[InlineArray(10)]
public struct IntArray10
{
    private int _element0;
    
    // 可以添加方法或属性来增强功能
    public int Length => 10;
    
    public int Sum()
    {
        int sum = 0;
        for (int i = 0; i < Length; i++)
        {
            sum += this[i];
        }
        return sum;
    }
}

class Program
{
    static void Main()
    {
        IntArray10 array = new IntArray10();
        
        // 初始化数组
        for (int i = 0; i < array.Length; i++)
        {
            array[i] = i * 2;
        }
        
        // 访问元素
        for (int i = 0; i < array.Length; i++)
        {
            Console.WriteLine($"array[{i}] = {array[i]}");
        }
        
        Console.WriteLine($"总和: {array.Sum()}");
    }
}
```

#### 与 `Span<T>` 结合

```csharp
[InlineArray(5)]
struct FloatBuffer
{
    private float _element;
}

var buffer = new FloatBuffer();
buffer[0] = 1.5f;
buffer[1] = 2.5f;

Span<float> span = buffer;
Console.WriteLine(span[0]); // 输出: 1.5
Console.WriteLine(span.Length); // 输出: 5
```

#### 泛型内联数组

```csharp
[InlineArray(5)]
public struct GenericArray<T>
{
    private T _element0;
    
    public int Length => 5;
    
    public void Initialize(T initialValue)
    {
        for (int i = 0; i < Length; i++)
        {
            this[i] = initialValue;
        }
    }
}

// 使用示例
GenericArray<string> stringArray = new GenericArray<string>();
stringArray.Initialize("default");
```

#### 在性能敏感场景中使用

内联数组适合需要极致性能的场景，例如处理向量或缓冲区。

```csharp
[InlineArray(3)]
struct Vector3D
{
    private float _element;
}

Vector3D AddVectors(Vector3D a, Vector3D b)
{
    var result = new Vector3D();
    for (int i = 0; i < 3; i++)
    {
        result[i] = a[i] + b[i];
    }
    return result;
}

var v1 = new Vector3D { [0] = 1.0f, [1] = 2.0f, [2] = 3.0f };
var v2 = new Vector3D { [0] = 4.0f, [1] = 5.0f, [2] = 6.0f };
var sum = AddVectors(v1, v2);

Console.WriteLine($"({sum[0]}, {sum[1]}, {sum[2]}"); // 输出: (5, 7, 9)
```

* `Vector3D` 模拟一个三维向量，内联数组确保内存连续。

* 适合游戏开发或科学计算。

#### 与本机代码交互

内联数组的连续内存布局使其非常适合与本机代码（如 `C/C++` ）交互。

```csharp
using System.Runtime.InteropServices;

[InlineArray(8)]
struct CharBuffer
{
    private char _element;
}

[DllImport("someNativeLib.dll")]
extern static void ProcessBuffer(ref CharBuffer buffer);

var buffer = new CharBuffer();
buffer[0] = 'H';
buffer[1] = 'e';
buffer[2] = 'l';
buffer[3] = 'l';
buffer[4] = 'o';
ProcessBuffer(ref buffer);
```

* `CharBuffer` 的内存布局与 `C` 语言中的 `char[8]` 兼容。

* 通过 `ref` 传递，确保本机代码可以直接操作内存。

### 高级用法

#### 与不安全代码结合

```csharp
[InlineArray(8)]
public unsafe struct DoubleArray
{
    private double _element0;
    
    public fixed int Length => 8;
    
    public double* GetPointer()
    {
        fixed (double* ptr = &this[0])
        {
            return ptr;
        }
    }
}
```

#### 模拟多维数组

```csharp
// 使用一维内联数组模拟二维数组
[InlineArray(16)] // 4x4 矩阵
public struct Matrix4x4
{
    private float _element0;
    
    public int Rows => 4;
    public int Columns => 4;
    
    public float this[int row, int col]
    {
        get => this[row * Columns + col];
        set => this[row * Columns + col] = value;
    }
    
    public static Matrix4x4 Identity()
    {
        var matrix = new Matrix4x4();
        for (int i = 0; i < 4; i++)
        {
            matrix[i, i] = 1.0f;
        }
        return matrix;
    }
}
```

#### 与 Span 和 Memory 互操作

```csharp
[InlineArray(100)]
public struct Buffer100
{
    private byte _element0;
    
    public int Length => 100;
    
    public Span<byte> AsSpan()
    {
        return MemoryMarshal.CreateSpan(ref this[0], Length);
    }
    
    public ReadOnlySpan<byte> AsReadOnlySpan()
    {
        return MemoryMarshal.CreateReadOnlySpan(ref this[0], Length);
    }
}
```

### 底层原理

#### 编译后代码结构

```csharp
// 原始代码
[InlineArray(5)]
public struct Buffer5 { private int _element; }

// 近似编译结果
public struct Buffer5
{
    private int _element0;
    private int _element1;
    private int _element2;
    private int _element3;
    private int _element4;

    public ref int this[int index]
    {
        get
        {
            if ((uint)index >= 5)
                throw new IndexOutOfRangeException();
            return ref Unsafe.Add(ref _element0, index);
        }
    }
}
```

### 适用场景

* 高性能计算：游戏引擎、图形处理或科学计算中需要连续内存的场景。

* 缓冲区管理：处理固定大小的缓冲区，如网络数据包或文件读写。

* 与本机代码交互：与 `C/C++` 或其他本机库交互，传递连续内存块。

* 替代小型数组：在结构体中替代 `T[]` 字段，减少堆分配。

* 向量/矩阵操作：表示数学向量或矩阵，优化内存访问。

### 注意事项

#### 仅限结构体：

* 只能定义在 `struct` 中，不能用于 `class`。

* 这是因为结构体是值类型，内存分配更可控。

#### 固定大小：

* 内联数组的大小在编译时必须是常量，无法动态调整。

* 不适合需要变长数组的场景。

#### 单一字段限制：

* 带有 `[InlineArray]` 的结构体只能包含一个字段。

* 如果需要其他字段，必须使用嵌套结构体。

```csharp
[InlineArray(10)]
struct InvalidBuffer
{
    private int _element;
    private int _otherField; // 错误：只能有一个字段
}
```

#### 性能优势：

* 内联数组分配在栈上（对于局部变量）或嵌入结构体中，减少堆分配。

* 连续内存布局减少缓存未命中（`cache miss`），提高性能。

#### 版本要求：

* 内联数组是 `C# 12（.NET 8）`的新特性，需确保项目目标框架为 `.NET 8.0` 或更高。

* 需要 `System.Runtime.CompilerServices.InlineArray` 特性。

#### 索引越界：

* 内联数组支持索引访问，但不会自动检查越界。

* 使用 `Span<T>` 操作时，`Span<T>` 会提供边界检查。

```csharp
[InlineArray(2)]
struct SmallBuffer
{
    private int _element;
}

var buffer = new SmallBuffer();
buffer[2] = 1; // 运行时异常：索引越界
```

#### 与普通数组的对比：

* 普通数组（`T[]`）：分配在托管堆上，动态大小，支持垃圾回收。

* 内联数组：固定大小，嵌入结构体，连续内存，适合高性能场景。

### 与其他特性的对比

#### 与普通数组（T[]）的对比：

* 普通数组是引用类型，分配在堆上，大小可动态调整。

* 内联数组是值类型的一部分，固定大小，内存连续，性能更高。

#### 与固定大小缓冲区（fixed）的对比：

* `C#` 的 `fixed` 关键字用于在 `unsafe` 上下文中创建固定大小缓冲区。

* 内联数组是类型安全的，无需 `unsafe`，更易用。

#### 与 `Span<T>/Memory<T>` 的对比：

* 内联数组是数据的存储结构，而 `Span<T>和Memory<T>`是访问视图。

* 内联数组可直接转换为 `Span<T>`，结合使用效率更高。

#### 与栈分配（stackalloc）的对比：

* `stackalloc` 在栈上分配临时内存，生命周期短。

* 内联数组嵌入结构体，支持更灵活的生命周期和传递。