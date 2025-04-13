### 简介

在 `.NET` 中，值拷贝（`Value Copy`）主要指的是将一个 值类型 的实例或对象的值复制到另一个变量中，使两个变量之间互不影响。我们可以从几个维度来详细理解：

### 值拷贝的本质

`.NET` 中的类型分为：

* 值类型（`Value Type`）：例如 `int、float、bool、DateTime、struct` 自定义结构体

* 引用类型（`Reference Type`）：例如 `string`（特殊值行为）、`class、object、array、List<T>` 等

值类型在赋值时是拷贝值本身，引用类型则是拷贝引用（地址）。

#### 值类型是值拷贝

```csharp
int a = 10;
int b = a; // 值拷贝
b = 20;

Console.WriteLine(a); // 输出 10，不受 b 更改影响
```

#### 引用类型是引用拷贝

```csharp
class Person
{
    public string Name;
}

Person p1 = new Person { Name = "Alice" };
Person p2 = p1; // 引用拷贝，指向同一对象

p2.Name = "Bob";

Console.WriteLine(p1.Name); // 输出 Bob，p1 也被修改了
```

### 自定义结构体也是值类型（值拷贝）

```csharp
struct Point
{
    public int X;
    public int Y;
}

Point p1 = new Point { X = 1, Y = 2 };
Point p2 = p1;  // 值拷贝
p2.X = 100;

Console.WriteLine(p1.X); // 1
Console.WriteLine(p2.X); // 100
```

结构体之间的赋值是完全拷贝一份内存，不影响原来的结构体变量。

### 浅拷贝 vs 深拷贝（类场景）

如果需要让 引用类型 的对象像 值类型 一样“复制”，就要实现：

* 浅拷贝（`Shallow Copy`）：复制对象的字段引用

* 深拷贝（`Deep Copy`）：复制整个对象图结构

#### 使用 MemberwiseClone() 实现浅拷贝（只能在类内部使用）

```csharp
public class Person
{
    public string Name;
    public Address Address;

    public Person ShallowCopy()
    {
        return (Person)this.MemberwiseClone();
    }
}

public class Address
{
    public string City;
}

// 使用
var original = new Person { Name = "Alice", Address = new Address { City = "New York" } };
var copy = original.ShallowCopy();

copy.Name = "Bob";
copy.Address.City = "London";

Console.WriteLine(original.Name);      // 输出: Alice
Console.WriteLine(original.Address.City); // 输出: London（共享引用）
```

#### 实现深拷贝的常用方式

* 自己 `new` 一个新对象，手动复制字段

```csharp
public class Person
{
    public string Name;
    public Address Address;

    public Person DeepCopy()
    {
        return new Person
        {
            Name = this.Name,
            Address = new Address { City = this.Address.City }
        };
    }
}

// 使用
var original = new Person { Name = "Alice", Address = new Address { City = "New York" } };
var copy = original.DeepCopy();

copy.Name = "Bob";
copy.Address.City = "London";

Console.WriteLine(original.Name);      // 输出: Alice
Console.WriteLine(original.Address.City); // 输出: New York（独立）
```

* 使用序列化反序列化（如 `Newtonsoft.Json` 或 `System.Text.Json`）

```csharp
public static T DeepCopy<T>(T obj)
{
    string json = JsonSerializer.Serialize(obj);
    return JsonSerializer.Deserialize<T>(json);
}

// 使用
var original = new Person { Name = "Alice", Address = new Address { City = "New York" } };
var copy = DeepCopy(original);

copy.Address.City = "London";
Console.WriteLine(original.Address.City); // 输出: New York
```

* 反射或表达式树：动态生成深拷贝逻辑。

```csharp
public static object DeepCopyReflection(object original) {
    if (original == null) return null;
    Type type = original.GetType();
    object copy = Activator.CreateInstance(type);
    foreach (FieldInfo field in type.GetFields(BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance)) {
        object value = field.GetValue(original);
        if (value is ICloneable cloneable) 
            field.SetValue(copy, cloneable.Clone());
        else 
            field.SetValue(copy, DeepCopyReflection(value));
    }
    return copy;
}
```

* 使用第三方库：`DeepCloner` 实现深拷贝

```csharp
using System;
using Force.DeepCloner;

public class Person
{
    public string Name { get; set; }
    public Address Address { get; set; }
}

public class Address
{
    public string City { get; set; }
}

class Program
{
    static void Main()
    {
        var original = new Person
        {
            Name = "Alice",
            Address = new Address { City = "New York" }
        };

        // 深拷贝对象
        var copy = original.DeepClone();

        copy.Name = "Bob";
        copy.Address.City = "London";

        Console.WriteLine(original.Name);       // 输出: Alice
        Console.WriteLine(original.Address.City); // 输出: New York（独立）
    }
}
```

* 使用第三方库：`DeepCloner` 处理复杂对象图

`DeepCloner` 支持嵌套对象、集合和循环引用的深拷贝。

```csharp
public class Company
{
    public List<Employee> Employees { get; set; }
}

public class Employee
{
    public string Name { get; set; }
    public Company Company { get; set; }
}

class Program
{
    static void Main()
    {
        var company = new Company();
        var employee = new Employee { Name = "Alice", Company = company };
        company.Employees = new List<Employee> { employee };

        // 深拷贝（处理循环引用）
        var companyCopy = company.DeepClone();

        companyCopy.Employees[0].Name = "Bob";
        Console.WriteLine(company.Employees[0].Name); // 输出: Alice（独立）
    }
}
```

> 性能优化

`DeepCloner` 通过缓存和 `IL` 代码生成实现高性能深拷贝，适合高频调用场景。

**基准对比**

|  方法   |  耗时（1000次深拷贝）   |  内存占用   |
| --- | --- | --- |
|  `DeepCloner`   |  ~10 ms   |  低   |
|  `Json序列化`   |  ~200 ms   |  高   |
|  `手动深拷贝`   |  ~5 ms（简单对象）   |  低   |

* 使用 `AutoMapper` 进行深度拷贝

```csharp
using AutoMapper;
using System;

// 定义源对象类
public class Source
{
    public int Id { get; set; }
    public string Name { get; set; }
}

// 定义目标对象类
public class Destination
{
    public int Id { get; set; }
    public string Name { get; set; }
}

class Program
{
    static void Main()
    {
        // 配置映射
        var config = new MapperConfiguration(cfg => cfg.CreateMap<Source, Destination>());
        var mapper = config.CreateMapper();

        // 创建源对象
        var source = new Source { Id = 1, Name = "Example" };

        // 进行映射（拷贝）
        var destination = mapper.Map<Destination>(source);

        Console.WriteLine($"Id: {destination.Id}, Name: {destination.Name}");
    }
}
```

* 使用 `ExpressMapper` 进行深度拷贝

`ExpressMapper` 是一个高性能的对象映射库，它利用表达式树来生成映射代码，从而实现快速的对象映射和拷贝。它支持复杂的映射规则和自定义转换，并且具有良好的性能表现。

```csharp
using ExpressMapper;
using System;

// 定义源对象类
public class SourceClass
{
    public int Value { get; set; }
}

// 定义目标对象类
public class DestinationClass
{
    public int Value { get; set; }
}

class Program
{
    static void Main()
    {
        // 配置映射
        Mapper.Register<SourceClass, DestinationClass>();

        // 创建源对象
        var source = new SourceClass { Value = 10 };

        // 进行映射（拷贝）
        var destination = Mapper.Map<SourceClass, DestinationClass>(source);

        Console.WriteLine($"Value: {destination.Value}");
    }
}
```