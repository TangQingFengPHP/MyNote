### 简介

运算符重载是 `C#` 提供的一种特性，允许开发者为 自定义类型（类/结构体） 定义运算符的行为。
例如，可以让 `Vector` 对象支持 + 运算，而不是仅限于基本类型（`int`、`double` 等）。

💡 本质：运算符重载是一个 带有 `operator` 关键字的静态方法，通过自定义方法改变运算符的操作行为。

### 适用范围与限制

| 特性               | 说明                                                                                                          |
| ------------------ | ------------------------------------------------------------------------------------------------------------- |
| 可重载的类型       | **类（class）** 和 **结构体（struct）**                                                                       |
| 不可重载的类型     | 接口、枚举、委托                                                                                              |
| 方法修饰符         | 必须是 **public static**                                                                                      |
| 至少一个自定义类型 | 运算符的参数中至少有一个必须是用户自定义类型                                                                  |
| 不能重载的运算符   | `.`（成员访问）、`?:`（条件运算符）、`new`、`is`、`as`、`typeof`、`sizeof`、`=`, `+=`, `-=`（但可以间接重载） |

### 支持重载的运算符

| 分类       | 运算符                                                            |
| ---------- | ----------------------------------------------------------------- |
| 一元运算符 | `+` `-` `!` `~` `++` `--` `true` `false`                          |
| 二元运算符 | `+` `-` `*` `/` `%` `&` \`                                        |
| 比较运算符 | `==` `!=` `<` `>` `<=` `>=`（必须成对重载，如重载==则必须重载!=） |
| 转换运算符 | `implicit`（隐式转换） `explicit`（显式转换）                     |

### 基本语法

```csharp
public static 返回类型 operator 运算符(参数列表)
{
    // 自定义逻辑
}
```

* `operator` 关键字定义运算符。

* 参数中至少有一个是当前类/结构体。

* 建议返回新的对象，保持不可变性。

### 常见示例

#### 重载二元运算符（+）

创建一个二维向量类：

```csharp
public struct Vector
{
    public double X { get; }
    public double Y { get; }

    public Vector(double x, double y) => (X, Y) = (x, y);

    public static Vector operator +(Vector a, Vector b)
        => new Vector(a.X + b.X, a.Y + b.Y);

    public override string ToString() => $"({X}, {Y})";
}

// 使用
var v1 = new Vector(1, 2);
var v2 = new Vector(3, 4);
Console.WriteLine(v1 + v2); // 输出: (4, 6)
```

#### 重载一元运算符（-）

```csharp
public static Vector operator -(Vector v)
    => new Vector(-v.X, -v.Y);

var v = new Vector(5, -3);
Console.WriteLine(-v); // 输出: (-5, 3)
```

#### 重载比较运算符（==, !=）

比较向量是否相等：

```csharp
public static bool operator ==(Vector a, Vector b)
    => a.X == b.X && a.Y == b.Y;

public static bool operator !=(Vector a, Vector b)
    => !(a == b);

// 建议同时重写 Equals 和 GetHashCode
public override bool Equals(object? obj)
    => obj is Vector v && this == v;
public override int GetHashCode()
    => HashCode.Combine(X, Y);
```

* 重载 `==` 时 必须 同时重载 `!=`。

* `Equals` 和 `GetHashCode` 也要同步实现，保证一致性。

#### 重载递增/递减运算符（++/--）

```csharp
public static Vector operator ++(Vector v)
    => new Vector(v.X + 1, v.Y + 1);

public static Vector operator --(Vector v)
    => new Vector(v.X - 1, v.Y - 1);
```

#### 转换运算符（implicit/explicit）

在 `Vector` 和 `double` 之间转换：

```csharp
public static implicit operator double(Vector v)
    => Math.Sqrt(v.X * v.X + v.Y * v.Y); // 隐式转换为长度

public static explicit operator Vector(double d)
    => new Vector(d, d); // 需要强制转换
```

使用：

```csharp
Vector v = new Vector(3, 4);
double len = v; // 隐式转换
Vector v2 = (Vector)5.0; // 显式转换
```

#### 逻辑运算符（true/false）

用于自定义布尔逻辑：

```csharp
public static bool operator true(Vector v) => v.X != 0 || v.Y != 0;
public static bool operator false(Vector v) => v.X == 0 && v.Y == 0;

Vector v = new Vector(0, 0);
if (v) // 自动调用 operator true
    Console.WriteLine("非零向量");
else
    Console.WriteLine("零向量");
```

### 运算符与方法的关系

运算符重载只是语法糖，编译器会将运算符转换为静态方法调用：

```csharp
var c = a + b;
// 等价于
var c = Vector.op_Addition(a, b);
```

常用方法映射：

| 运算符 | 生成的方法名   |
| ------ | -------------- |
| `+`    | op_Addition    |
| `-`    | op_Subtraction |
| `*`    | op_Multiply    |
| `/`    | op_Division    |
| `==`   | op_Equality    |
| `!=`   | op_Inequality  |

### 综合示例：复数类

```csharp
public struct Complex
{
    public double Real { get; }
    public double Imag { get; }

    public Complex(double real, double imag)
        => (Real, Imag) = (real, imag);

    public static Complex operator +(Complex a, Complex b)
        => new Complex(a.Real + b.Real, a.Imag + b.Imag);

    public static Complex operator -(Complex a, Complex b)
        => new Complex(a.Real - b.Real, a.Imag - b.Imag);

    public static Complex operator *(Complex a, Complex b)
        => new Complex(a.Real * b.Real - a.Imag * b.Imag,
                       a.Real * b.Imag + a.Imag * b.Real);

    public static bool operator ==(Complex a, Complex b)
        => a.Real == b.Real && a.Imag == b.Imag;

    public static bool operator !=(Complex a, Complex b)
        => !(a == b);

    public override string ToString() => $"{Real} + {Imag}i";
}
```

### 总结

| 特性     | 说明                                             |
| -------- | ------------------------------------------------ |
| 适用场景 | 数学计算类（向量、矩阵、复数）、日期时间、坐标类 |
| 关键规则 | `public static`、至少一个参数为自定义类型        |
| 搭配使用 | `Equals`、`GetHashCode`、`IComparable`           |
| 设计建议 | 遵循语义一致性、返回新对象、与方法重载保持协调   |
