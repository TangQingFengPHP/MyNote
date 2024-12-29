### 简介

`C#` 中的 `dynamic` 是一种特殊类型，它允许在运行时确定对象的类型和成员，而不是在编译时。

### `dynamic` 的定义

* `dynamic` 是一种类型，它告诉编译器对其进行“动态类型解析”。

* `dynamic` 类型的变量会跳过编译时类型检查，所有的操作会推迟到运行时进行。

* 适合处理未知类型的对象，或需要与动态语言（如 `Python`、`JavaScript`）互操作的场景。


### `dynamic` 的使用

#### 动态类型赋值

```csharp
dynamic obj = 10;  // 可以是整数
obj = "Hello";     // 可以变成字符串
obj = new { Name = "John", Age = 30 }; // 也可以是匿名类型
```

#### 访问成员

> 动态对象的成员在运行时解析，因此可以访问任意成员：
>
> 如果访问了不存在的成员，运行时会抛出 `RuntimeBinderException`。

```csharp
dynamic obj = new { Name = "John", Age = 30 };
Console.WriteLine(obj.Name); // 输出: John
```

#### 动态方法调用

```csharp
dynamic math = new { Add = (Func<int, int, int>)((x, y) => x + y) };
Console.WriteLine(math.Add(2, 3)); // 输出: 5
```

### `dynamic` 的核心特性

> 与编译时类型（静态类型）的区别

* 编译时检查：`dynamic` 不会在编译时检查类型或成员是否存在，所有操作推迟到运行时。

* 静态类型：`object` 和其他类型在编译时进行类型检查。

```csharp
object obj1 = 10;
// obj1.SomeMethod(); // 编译错误：object 没有 SomeMethod 方法

dynamic obj2 = 10;
// obj2.SomeMethod(); // 编译通过，但运行时抛出异常
```

> 类型推断

动态类型在运行时确定，而静态类型通过编译器推断：

```csharp
dynamic dynamicVariable = 123; // 编译器不检查类型
int staticVariable = 123;      // 编译器推断为 int

object obj = "Hello";
// Console.WriteLine(obj.Length); // 编译错误
Console.WriteLine(((string)obj).Length); // 强制转换

dynamic dyn = "Hello";
Console.WriteLine(dyn.Length); // 运行时解析，编译通过
```

### 使用场景

* 与动态语言交互：调用动态语言的 `API`，如 `COM` 对象、`IronPython` 等

* `JSON` 或 `XML` 数据处理：在处理结构未知的数据时动态解析。

* 匿名类型和动态扩展：快速访问动态创建的对象。

### 注意事项

* 性能开销：动态绑定会引入性能开销，因为解析是在运行时完成的。

* 类型安全：缺乏编译时类型检查，可能导致运行时错误。

* 调试困难：错误可能难以发现，尤其是在复杂场景中。

### `ExpandoObject` 与 `dynamic`

> `ExpandoObject` 是一个动态对象，可在运行时动态添加或删除成员：
>
> 常用于需要灵活扩展的场景，如 `JSON` 数据的解析

```csharp
using System.Dynamic;

dynamic expando = new ExpandoObject();
expando.Name = "John";
expando.Age = 30;

Console.WriteLine($"{expando.Name}, {expando.Age}");
```

#### `ExpandoObject` 内部实现机制

> `ExpandoObject` 实现了以下关键接口：

* `IDynamicMetaObjectProvider`：提供动态行为（如动态调用、成员访问）的核心接口。

* `IDictionary<string, object>`：内部使用一个 Dictionary `<string, object>` 存储动态添加的成员。

```csharp
using System.Dynamic;

dynamic expando = new ExpandoObject();
expando.Name = "John"; // 动态添加成员
Console.WriteLine(expando.Name); // 动态访问成员

// 等价于：
var expando = new ExpandoObject() as IDictionary<string, object>;
expando["Name"] = "John";
Console.WriteLine(expando["Name"]);
```

#### `ExpandoObject` 如何实现动态性？

> `ExpandoObject` 使用动态绑定和元对象来实现动态行为：

* 动态绑定：通过 `IDynamicMetaObjectProvider`，在运行时解析属性、方法等访问请求。

* 内部字典：通过 `Dictionary<string, object>` 存储成员。

* 元对象：`ExpandoObject` 的动态行为由一个元对象 `ExpandoMetaObject` 提供支持，它负责解释动态操作并将其映射到内部字典。

#### `ExpandoObject` 线程安全性

* `ExpandoObject` 本质上不是线程安全的，因为它允许动态修改成员。

* 在多线程场景下，需要通过显式锁定来确保线程安全。

#### `ExpandoObject` 使用示例

* 动态添加/删除成员

```csharp
dynamic expando = new ExpandoObject();
expando.Name = "John";
expando.Age = 30;

// 删除成员
var dict = (IDictionary<string, object>)expando;
dict.Remove("Age");

// 检查成员
Console.WriteLine(dict.ContainsKey("Name")); // True
Console.WriteLine(dict.ContainsKey("Age"));  // False
```

* 结合 `LINQ` 查询

> 因为 `ExpandoObject` 实现了 `IDictionary<string, object>`，可以结合 `LINQ` 操作：

```csharp
dynamic expando = new ExpandoObject();
expando.Name = "John";
expando.Age = 30;

var dict = (IDictionary<string, object>)expando;
var filtered = dict.Where(kv => kv.Key.StartsWith("N"));

foreach (var kv in filtered)
{
    Console.WriteLine($"{kv.Key}: {kv.Value}");
}
```

#### 综合考虑

1. 使用 `dynamic` 的场景：

* 与外部动态类型（如 `COM`、动态语言）交互。

* 动态调用方法或访问属性，无需构建明确的对象。

* 临时需要动态行为，但不需要动态修改成员。

2. 使用 `ExpandoObject` 的场景：

* 需要构建动态扩展的对象。

* 动态添加和删除属性。

* 构建轻量级、灵活的业务模型。

#### 选择建议

| **特性**              | **`dynamic`**                      | **`ExpandoObject`**                    |
| --------------------- | ---------------------------------- | -------------------------------------- |
| **动态成员解析**      | 支持任意动态成员，解析时运行时检查 | 只能操作明确添加到对象的动态成员       |
| **动态成员添加/删除** | 不支持                             | 支持（通过字典实现）                   |
| **类型检查**          | 编译时无类型检查，运行时解析       | 同样运行时解析                         |
| **适合的场景**        | 动态语言互操作、临时操作、反射     | 动态构建对象、领域模型扩展             |
| **性能**              | 比静态类型慢，因为运行时动态绑定   | 更高效，基于字典实现                   |
| **实现复杂度**        | 动态行为由 CLR 处理                | 由 `ExpandoObject` 自行实现动态行为    |
| **对字典的支持**      | 不支持                             | 内部就是 `IDictionary<string, object>` |
| **安全性**            | 运行时错误较多，调试复杂           | 动态但有一定约束                       |
