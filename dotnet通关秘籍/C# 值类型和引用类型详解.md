### 简介

在 `C#` 中，值类型和引用类型是两个基础的数据类型类别，它们的主要区别在于 存储位置 和 赋值方式。

### 值类型

值类型存储的是数据本身，分配在 栈 (`Stack`) 中。当一个值类型变量被赋值给另一个变量时，会复制值。

**值类型的特点**

* 内存分配：存储在栈上。

* 存储内容：直接保存数据。

* 复制行为：赋值时复制数据，两个变量互不影响。

* 默认值：初始化为类型的默认值（如 `int` 为 0）。

* 不可为 `null`（除非是可空类型，如 `int?`）。

**值类型的常见示例**

* 基本数据类型：`int、double、bool、char、float、decimal`

* 结构体（`struct`）：例如 `DateTime、Point`

* 枚举（`enum`）

**值类型示例**

```csharp
int a = 10;  // 值类型变量
int b = a;   // 将 a 的值赋值给 b
b = 20;      // 修改 b 的值，不影响 a

Console.WriteLine(a);  // 输出 10
Console.WriteLine(b);  // 输出 20
```

### 引用类型

引用类型存储的是数据的地址（引用），实际数据存储在 堆 (`Heap`) 中。当一个引用类型变量被赋值给另一个变量时，会复制引用，即两个变量指向同一个对象。

引用类型的特点

* 内存分配：引用存储在栈上，数据存储在堆上。

* 存储内容：保存对数据的引用地址，而不是数据本身。

* 复制行为：赋值时复制引用，多个变量指向同一块内存。

* 默认值：初始化为 `null`。

* 可以为 `null`。

**引用类型的常见示例**

* 类（`class`）

* 接口（`interface`）

* 委托（`delegate`）

* 字符串（`string`）

* 数组（`array`）

**引用类型示例**

```csharp
class MyClass
{
    public int Value;
}

MyClass obj1 = new MyClass();
obj1.Value = 10;

MyClass obj2 = obj1;  // 将 obj1 的引用赋值给 obj2
obj2.Value = 20;      // 修改 obj2 的值，会影响 obj1

Console.WriteLine(obj1.Value);  // 输出 20
Console.WriteLine(obj2.Value);  // 输出 20
```

### 值类型与引用类型的区别

|  特性   |  值类型   |  引用类型   |
| --- | --- | --- |
|  存储位置   |  栈   |  栈存储引用，堆存储数据   |
|  存储内容   |  保存数据本身   |  保存数据的地址（引用）   |
|  赋值行为  |  复制值   |  复制引用，多个引用指向同一个对象   |
|  默认值  |  默认值（如 int 为 0）   |  默认值为 null   |
|  性能  |  栈上分配，效率高   |  堆上分配，需垃圾回收   |


### 可空类型（`Nullable`）

> 值类型通常不能为 `null`，但可以通过 `Nullable<T>` 或 `T?` 使其可为空。

示例：

```csharp
int? a = null;
if (a.HasValue)
    Console.WriteLine(a.Value);
else
    Console.WriteLine("a is null");
```

### 特殊的引用类型：字符串

* 字符串（`string`）是引用类型，但具有值类型的行为。

* 因为字符串是 不可变的（`Immutable`），修改字符串实际上是创建了一个新对象。


```csharp
string str1 = "Hello";
string str2 = str1;
str2 = "World";

Console.WriteLine(str1);  // 输出 Hello
Console.WriteLine(str2);  // 输出 World
```

#### 值类型与引用类型的转换

* 装箱（`Boxing`）：将值类型转换为引用类型。

* 拆箱（`Unboxing`）：将引用类型转换为值类型。

示例：

```csharp
int a = 10;          // 值类型
object obj = a;      // 装箱，值类型变为引用类型
int b = (int)obj;    // 拆箱，将引用类型还原为值类型

Console.WriteLine(a);  // 输出 10
Console.WriteLine(b);  // 输出 10
```