### 简介

`C# 11` 和 `.NET 7` 引入了泛型数学（`Generic Math`）功能，这是一个革命性的特性，允许开发者编写适用于多种数值类型的通用数学算法。这是通过静态抽象接口成员实现的，解决了长期以来在泛型代码中处理数学运算的难题。

#### 为什么需要“泛型数学”？

* 以前无法对“数字类型集合”（`int/float/decimal/BigInteger/...`）做统一的泛型约束（只能 `where T : struct`），无法在泛型里使用 `+、*` 等运算符。

* `C# 11` 的静态抽象接口成员允许接口定义必须存在的静态成员和运算符，从而把运算符抽象为接口成员；`BCL` 利用了这个特性，定义了大量数值接口，称为 `.NET Generic Math`。

#### 核心思想

* 接口可以声明 `static abstract` 成员（例如 `static abstract T Self + T Self` 或 `static abstract T Zero`）。

* 数值类型（`int, double, decimal, BigInteger, Half, Int128...`）在 `.NET 7+` 中实现了这些接口。

* 因此：可以写 `where T : INumber<T>`，在方法体里直接写 `T result = a + b`; 或 `T.Zero、T.One`，或调用 `T.Sqrt(x)`（当 `T` 支持根函数时）。

### 主要接口

#### 基本操作接口

* `IAdditionOperators<TSelf, TOther, TResult>`：支持 `+`。

* `ISubtractionOperators<TSelf, TOther, TResult>`：支持 `-`。

* `IMultiplyOperators<TSelf, TOther, TResult>`：支持 `*`。

* `IDivisionOperators<TSelf, TOther, TResult>`：支持 `/`。

* `IModulusOperators、IBitwiseOperators、IShiftOperators、IComparisonOperators` 等。

#### 复合/高阶数值接口

* `INumberBase<TSelf>`：所有数字（甚至复数）共有的基础 `API`（包含 `Abs、CreateChecked/CreateTruncating/CreateSaturating` 等）。

* `INumber<TSelf>`：可比较（`ordered`）的“实数”类 `API`（实现它的类型可以比较大小）。

* `IBinaryInteger<TSelf>`：二进制整数专用（`int/long/BigInteger/UInt32/...`），提供 `DivRem、LeadingZeroCount、RotateLeft/Right` 等。

* `IFloatingPoint<TSelf> / IFloatingPointIeee754<TSelf>`：浮点专用接口（`float/double/half`），提供 `NaN/Infinity`、根/幂/三角/对数/舍入等。`IFloatingPointIeee754` 还包含 `IEEE-754` 特定 `API`（常量、特殊值等）。

### 核心概念

#### 静态抽象接口成员

```csharp
// 定义包含静态抽象成员的接口
public interface IAddable<T> where T : IAddable<T>
{
    static abstract T operator +(T left, T right);
}

// 实现接口
public struct MyNumber : IAddable<MyNumber>
{
    public int Value { get; }
    
    public MyNumber(int value) => Value = value;
    
    public static MyNumber operator +(MyNumber left, MyNumber right)
        => new MyNumber(left.Value + right.Value);
}
```

#### 数学接口体系

```csharp
// 基本数值接口
public interface INumber<TSelf> : 
    IAdditionOperators<TSelf, TSelf, TSelf>,
    ISubtractionOperators<TSelf, TSelf, TSelf>,
    IMultiplyOperators<TSelf, TSelf, TSelf>,
    IDivisionOperators<TSelf, TSelf, TSelf>,
    IComparisonOperators<TSelf, TSelf, bool>,
    IModulusOperators<TSelf, TSelf, TSelf>,
    IMinMaxValue<TSelf>
    where TSelf : INumber<TSelf>?
{
    static abstract TSelf Zero { get; }
    static abstract TSelf One { get; }
    static abstract TSelf Abs(TSelf value);
    static abstract TSelf Max(TSelf x, TSelf y);
    static abstract TSelf Min(TSelf x, TSelf y);
}
```

### 常见用法与示例

#### 简单的泛型数学函数

```csharp
// 泛型求和函数
public static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T result = T.Zero;
    foreach (T value in values)
    {
        result += value;
    }
    return result;
}

// 泛型平均值函数
public static T Average<T>(IEnumerable<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    int count = 0;
    
    foreach (T value in values)
    {
        sum += value;
        count++;
    }
    
    return sum / T.CreateChecked(count);
}

// 使用示例
int[] ints = { 1, 2, 3, 4, 5 };
double[] doubles = { 1.1, 2.2, 3.3, 4.4, 5.5 };

Console.WriteLine(Sum(ints));     // 输出: 15
Console.WriteLine(Average(doubles)); // 输出: 3.3
```

#### 数学运算示例

```csharp
// 泛型数学运算
public static T Calculate<T>(T a, T b) where T : INumber<T>
{
    return (a + b) * (a - b) / T.CreateChecked(2);
}

// 使用不同的数值类型
Console.WriteLine(Calculate(10, 5));      // int: (10+5)*(10-5)/2 = 37
Console.WriteLine(Calculate(10.5, 5.5));  // double: (10.5+5.5)*(10.5-5.5)/2 = 40
```

#### 求平均值（`INumber<T>`，示例使用 CreateChecked 将 int 转为 T）

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Numerics;

static T Average<T>(IEnumerable<T> src) where T : INumber<T>
{
    if (src == null) throw new ArgumentNullException(nameof(src));
    T sum = T.Zero;
    long count = 0;
    foreach (var x in src) { sum += x; count++; }
    if (count == 0) throw new InvalidOperationException("sequence empty");
    // 将 long 转换为 T（CreateChecked/Truncating/Saturating 都可选）
    T countT = T.CreateChecked<long>(count);
    return sum / countT;
}
```

> `CreateChecked<TOther> / CreateTruncating<TOther> / CreateSaturating<TOther> 在 INumberBase<T>` 上定义，用于跨数值类型安全转换（会抛异常、截断或饱和）。

#### GCD（用在整数上：`IBinaryInteger<T>`）

```csharp
using System.Numerics;

static T Gcd<T>(T a, T b) where T : IBinaryInteger<T>
{
    a = T.Abs(a);
    b = T.Abs(b);
    while (b != T.Zero)
    {
        var r = a % b;
        a = b;
        b = r;
    }
    return a;
}
```

> `IBinaryInteger<T>` 提供 %、DivRem、LeadingZeroCount 等整型专用工具。

#### 浮点 sqrt / hypot（`IFloatingPointIeee754<T>`）

```csharp
using System.Numerics;

static T Hypot<T>(T x, T y) where T : IFloatingPointIeee754<T>
{
    // IFloatingPointIeee754 / IRootFunctions 提供 Hypot / Sqrt / Cbrt 等
    return T.Hypot(x, y);
    // 或者 return T.Sqrt(x * x + y * y);
}
```

> IFloatingPointIeee754 继承了 IRootFunctions、IPowerFunctions 等，支持 Sqrt, Pow, Hypot 等静态函数。

#### 泛型矩阵相乘

```csharp
public class Matrix<T> where T : INumber<T>
{
    T[,] _a;
    public int R => _a.GetLength(0);
    public int C => _a.GetLength(1);
    public Matrix(int r, int c) => _a = new T[r,c];
    public T this[int i,int j] { get => _a[i,j]; set => _a[i,j] = value; }

    public static Matrix<T> Multiply(Matrix<T> A, Matrix<T> B)
    {
        if (A.C != B.R) throw new ArgumentException("size");
        var C = new Matrix<T>(A.R, B.C);
        for (int i = 0; i < A.R; i++)
            for (int j = 0; j < B.C; j++)
            {
                T sum = T.Zero;
                for (int k = 0; k < A.C; k++)
                    sum += A[i,k] * B[k,j];
                C[i,j] = sum;
            }
        return C;
    }
}
```

#### 复数计算示例

```csharp
// 泛型复数计算
public record Complex<T>(T Real, T Imaginary) where T : INumber<T>
{
    public static Complex<T> operator +(Complex<T> left, Complex<T> right)
        => new Complex<T>(left.Real + right.Real, left.Imaginary + right.Imaginary);
    
    public static Complex<T> operator *(Complex<T> left, Complex<T> right)
        => new Complex<T>(
            left.Real * right.Real - left.Imaginary * right.Imaginary,
            left.Real * right.Imaginary + left.Imaginary * right.Real
        );
    
    public T Magnitude() where T : IRootFunctions<T>
    {
        return T.Sqrt(Real * Real + Imaginary * Imaginary);
    }
    
    public override string ToString() => $"{Real} + {Imaginary}i";
}

// 使用复数
var c1 = new Complex<double>(3, 4);
var c2 = new Complex<double>(1, 2);
var sum = c1 + c2;
var product = c1 * c2;

Console.WriteLine($"Sum: {sum}");           // 4 + 6i
Console.WriteLine($"Product: {product}");   // -5 + 10i
Console.WriteLine($"Magnitude: {c1.Magnitude()}"); // 5
```

### 实际应用场景

#### 通用数学库函数

```csharp
public static T StandardDeviation<T>(IEnumerable<T> values)
    where T : INumber<T>, IRootFunctions<T>
{
    var mean = Mean(values);
    var variance = T.Zero;
    var count = T.Zero;
    
    foreach (var value in values)
    {
        var diff = value - mean;
        variance += diff * diff;
        count++;
    }
    
    variance /= count;
    return T.Sqrt(variance);
}

public static T Mean<T>(IEnumerable<T> values) where T : INumber<T>
{
    var sum = T.Zero;
    var count = T.Zero;
    
    foreach (var value in values)
    {
        sum += value;
        count++;
    }
    
    return sum / count;
}
```

#### 几何计算

```csharp
public record Vector2D<T>(T X, T Y) where T : INumber<T>, IRootFunctions<T>
{
    public T Magnitude => T.Sqrt(X * X + Y * Y);
    
    public static Vector2D<T> operator +(Vector2D<T> a, Vector2D<T> b)
        => new(a.X + b.X, a.Y + b.Y);
    
    public static Vector2D<T> operator -(Vector2D<T> a, Vector2D<T> b)
        => new(a.X - b.X, a.Y - b.Y);
    
    public static T Dot(Vector2D<T> a, Vector2D<T> b)
        => a.X * b.X + a.Y * b.Y;
    
    public Vector2D<T> Normalize()
    {
        var mag = Magnitude;
        return mag == T.Zero 
            ? this 
            : new Vector2D<T>(X / mag, Y / mag);
    }
}
```

#### 财务计算

```csharp
public static T CalculateCompoundInterest<T>(
    T principal, 
    T annualRate, 
    int years, 
    int compoundingPeriods = 1)
    where T : IFloatingPoint<T>
{
    var ratePerPeriod = annualRate / T.CreateChecked(compoundingPeriods);
    var periods = years * compoundingPeriods;
    
    return principal * T.Pow(T.One + ratePerPeriod, T.CreateChecked(periods));
}
```

#### 数值积分和微分

```csharp
// 泛型数值积分
public static T Integrate<T>(
    Func<T, T> function, 
    T from, 
    T to, 
    int steps) where T : IFloatingPoint<T>
{
    T stepSize = (to - from) / T.CreateChecked(steps);
    T sum = T.Zero;
    
    for (int i = 0; i < steps; i++)
    {
        T x1 = from + T.CreateChecked(i) * stepSize;
        T x2 = from + T.CreateChecked(i + 1) * stepSize;
        T y1 = function(x1);
        T y2 = function(x2);
        
        // 梯形法则
        sum += (y1 + y2) * stepSize / T.CreateChecked(2);
    }
    
    return sum;
}

// 使用数值积分
Func<double, double> f = x => x * x; // f(x) = x²
double integral = Integrate(f, 0.0, 1.0, 1000);
Console.WriteLine($"∫x²dx from 0 to 1 = {integral}"); // 约等于 0.333...
```

#### 线性代数运算

```csharp
// 泛型向量类
public struct Vector<T> where T : INumber<T>
{
    private readonly T[] _components;
    
    public Vector(params T[] components)
    {
        _components = components;
    }
    
    public int Dimension => _components.Length;
    
    public T this[int index]
    {
        get => _components[index];
        set => _components[index] = value;
    }
    
    public static Vector<T> operator +(Vector<T> left, Vector<T> right)
    {
        if (left.Dimension != right.Dimension)
            throw new ArgumentException("Vectors must have the same dimension");
        
        T[] result = new T[left.Dimension];
        for (int i = 0; i < left.Dimension; i++)
        {
            result[i] = left[i] + right[i];
        }
        return new Vector<T>(result);
    }
    
    public static T operator *(Vector<T> left, Vector<T> right) // 点积
    {
        if (left.Dimension != right.Dimension)
            throw new ArgumentException("Vectors must have the same dimension");
        
        T result = T.Zero;
        for (int i = 0; i < left.Dimension; i++)
        {
            result += left[i] * right[i];
        }
        return result;
    }
    
    public T Magnitude() where T : IRootFunctions<T>
    {
        T sumOfSquares = T.Zero;
        foreach (T component in _components)
        {
            sumOfSquares += component * component;
        }
        return T.Sqrt(sumOfSquares);
    }
    
    public override string ToString() => 
        $"[{string.Join(", ", _components)}]";
}

// 使用向量
var v1 = new Vector<double>(1, 2, 3);
var v2 = new Vector<double>(4, 5, 6);
var sum = v1 + v2;
var dotProduct = v1 * v2;

Console.WriteLine($"v1 + v2 = {sum}");           // [5, 7, 9]
Console.WriteLine($"v1 · v2 = {dotProduct}");    // 32
Console.WriteLine($"|v1| = {v1.Magnitude()}");   // 3.741...
```

#### 统计计算

```csharp
// 泛型统计函数
public static class Statistics<T> where T : INumber<T>, IFloatingPoint<T>
{
    public static T Mean(IEnumerable<T> values)
    {
        T sum = T.Zero;
        int count = 0;
        
        foreach (T value in values)
        {
            sum += value;
            count++;
        }
        
        return sum / T.CreateChecked(count);
    }
    
    public static T Variance(IEnumerable<T> values)
    {
        T mean = Mean(values);
        T sumOfSquares = T.Zero;
        int count = 0;
        
        foreach (T value in values)
        {
            T deviation = value - mean;
            sumOfSquares += deviation * deviation;
            count++;
        }
        
        return sumOfSquares / T.CreateChecked(count);
    }
    
    public static T StandardDeviation(IEnumerable<T> values)
    {
        return T.Sqrt(Variance(values));
    }
    
    public static (T Min, T Max, T Median) DescriptiveStats(IEnumerable<T> values)
    {
        var sorted = values.OrderBy(v => v).ToArray();
        int count = sorted.Length;
        
        T min = sorted[0];
        T max = sorted[count - 1];
        
        T median = count % 2 == 0
            ? (sorted[count / 2 - 1] + sorted[count / 2]) / T.CreateChecked(2)
            : sorted[count / 2];
        
        return (min, max, median);
    }
}

// 使用统计函数
double[] data = { 1.2, 2.3, 3.4, 4.5, 5.6 };
Console.WriteLine($"Mean: {Statistics<double>.Mean(data)}");
Console.WriteLine($"Variance: {Statistics<double>.Variance(data)}");
Console.WriteLine($"Standard Deviation: {Statistics<double>.StandardDeviation(data)}");

var (min, max, median) = Statistics<double>.DescriptiveStats(data);
Console.WriteLine($"Min: {min}, Max: {max}, Median: {median}");
```

### 自定义数值类型

#### 创建支持泛型数学的自定义类型

```csharp
// 自定义分数类型
public readonly struct Fraction : 
    INumber<Fraction>,
    IComparisonOperators<Fraction, Fraction, bool>,
    IModulusOperators<Fraction, Fraction, Fraction>
{
    public long Numerator { get; }
    public long Denominator { get; }
    
    public Fraction(long numerator, long denominator)
    {
        if (denominator == 0)
            throw new DivideByZeroException("Denominator cannot be zero");
        
        // 简化分数
        long gcd = Gcd(Math.Abs(numerator), Math.Abs(denominator));
        Numerator = numerator / gcd;
        Denominator = denominator / gcd;
        
        // 确保分母为正
        if (Denominator < 0)
        {
            Numerator = -Numerator;
            Denominator = -Denominator;
        }
    }
    
    private static long Gcd(long a, long b) => b == 0 ? a : Gcd(b, a % b);
    
    // INumber<Fraction> 实现
    public static Fraction Zero => new Fraction(0, 1);
    public static Fraction One => new Fraction(1, 1);
    
    public static Fraction operator +(Fraction left, Fraction right)
    {
        long numerator = left.Numerator * right.Denominator + right.Numerator * left.Denominator;
        long denominator = left.Denominator * right.Denominator;
        return new Fraction(numerator, denominator);
    }
    
    public static Fraction operator -(Fraction left, Fraction right)
    {
        long numerator = left.Numerator * right.Denominator - right.Numerator * left.Denominator;
        long denominator = left.Denominator * right.Denominator;
        return new Fraction(numerator, denominator);
    }
    
    public static Fraction operator *(Fraction left, Fraction right)
    {
        return new Fraction(
            left.Numerator * right.Numerator,
            left.Denominator * right.Denominator
        );
    }
    
    public static Fraction operator /(Fraction left, Fraction right)
    {
        return new Fraction(
            left.Numerator * right.Denominator,
            left.Denominator * right.Numerator
        );
    }
    
    // 其他接口实现...
    
    public override string ToString() => $"{Numerator}/{Denominator}";
}

// 使用自定义分数类型
Fraction f1 = new Fraction(1, 2);
Fraction f2 = new Fraction(3, 4);
Fraction sum = f1 + f2; // 5/4
Fraction product = f1 * f2; // 3/8

Console.WriteLine($"{f1} + {f2} = {sum}");
Console.WriteLine($"{f1} × {f2} = {product}");
```

### 高级技巧

#### 类型转换处理

```csharp
public static TResult ConvertSafely<TInput, TResult>(TInput value)
    where TInput : INumber<TInput>
    where TResult : INumber<TResult>
{
    try
    {
        return TResult.CreateChecked(value);
    }
    catch (OverflowException)
    {
        return value < TInput.Zero 
            ? TResult.NegativeInfinity 
            : TResult.PositiveInfinity;
    }
}
```

#### 性能优化

```csharp
// 使用泛型数学的向量化操作
public static T[] VectorAdd<T>(T[] a, T[] b) 
    where T : IAdditionOperators<T, T, T>, IAdditiveIdentity<T, T>
{
    if (a.Length != b.Length)
        throw new ArgumentException("Arrays must be same length");
    
    var result = new T[a.Length];
    
    // 使用 SIMD 优化（如果可用）
    if (Vector.IsHardwareAccelerated && 
        Vector<T>.IsSupported)
    {
        int vectorSize = Vector<T>.Count;
        int i = 0;
        
        for (; i <= a.Length - vectorSize; i += vectorSize)
        {
            var va = new Vector<T>(a, i);
            var vb = new Vector<T>(b, i);
            (va + vb).CopyTo(result, i);
        }
        
        // 处理剩余元素
        for (; i < a.Length; i++)
        {
            result[i] = a[i] + b[i];
        }
    }
    else
    {
        // 回退到常规循环
        for (int i = 0; i < a.Length; i++)
        {
            result[i] = a[i] + b[i];
        }
    }
    
    return result;
}
```

#### 条件约束组合

```csharp
public static T SafeDivide<T>(T dividend, T divisor)
    where T : IDivisionOperators<T, T, T>, 
              IComparisonOperators<T, T, bool>,
              IAdditiveIdentity<T, T>
{
    if (divisor == T.AdditiveIdentity)
        throw new DivideByZeroException();
    
    return dividend / divisor;
}
```

### 类型/接口选择建议

* 想要通用数（能 + - * /、有 Zero、能比较大小）→ 用 `INumber<T>`。

* 只针对整数算法（GCD、位操作等）→ 用 `IBinaryInteger<T>`。

* 只针对浮点/IEEE754 特性（NaN、Infinity、Sqrt、Hypot 等）→ 用 `IFloatingPointIeee754<T>`（或更窄的 `IFloatingPoint<T>`）。

### 总结

`C# 11` 和 `.NET 7` 的泛型数学功能是一个重大突破，它：

* 解决了长期痛点：终于可以在泛型代码中方便地进行数学运算

* 提供了完整的数学接口体系：覆盖基本运算、比较、三角函数等

* 支持自定义数值类型：可以创建自己的数值类型并集成到数学生态中

* 保持高性能：通过静态抽象接口避免装箱拆箱开销

* 增强类型安全：编译时类型检查，减少运行时错误

适用场景：

* 数学库开发：创建通用的数学算法库

* 科学计算：物理模拟、数值分析等

* 游戏开发：向量、矩阵运算

* 金融计算：高精度数值计算

* 数据处理：统计分析、数据转换