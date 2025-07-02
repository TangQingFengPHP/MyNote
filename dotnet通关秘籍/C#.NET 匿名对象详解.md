### 简介

匿名对象（`Anonymous Types`）是一种在编译时由编译器自动生成、但在源码中没有显式命名的引用类型，用来快速封装一组只读属性。它们最常见的场景是在 `LINQ` 查询中临时投影数据，但也可用于任何需要临时封装数据的地方。

### 基本语法

```csharp
// 最简单的匿名对象，两个属性：Name（string）、Age（int）
var person = new { Name = "Alice", Age = 30 };

// 访问属性
Console.WriteLine(person.Name);  // 输出 "Alice"
Console.WriteLine(person.Age);   // 输出 30
```

* 关键字：使用 `new { … }`，在大括号里列出“属性名 = 值”对。

* 类型：编译器在后台生成一个内部 `sealed` 类，属性带 `get` 访问器且只读。

* 推断：使用 `var` 才能声明匿名类型变量；不能写成 `SomeType person` = ...。

### 属性特性

* 只读：匿名类型的属性只带 `get`，没有 `set`。

* 自动 `Equals/GetHashCode`：编译器重写了这两个方法，使得如果两个匿名实例的属性名称、顺序、类型和值都相同，则它们相互 `Equals` 并且哈希值相同。

* `ToString()`：被重写以返回类似 `{ Name = Alice, Age = 30 }` 的字符串。

```csharp
var a = new { X = 1, Y = 2 };
var b = new { X = 1, Y = 2 };
Console.WriteLine(a.Equals(b));      // True
Console.WriteLine(a.GetHashCode());  // 与 b 相同
Console.WriteLine(a.ToString());     // "{ X = 1, Y = 2 }"
```

### 底层原理

编译器会自动生成类似如下的类：

```csharp
[CompilerGenerated]
internal sealed class <>f__AnonymousType0<<Name>j__TPar, <Age>j__TPar>
{
    public string Name { get; }
    public int Age { get; }
    
    public <>f__AnonymousType0(string name, int age)
    {
        this.Name = name;
        this.Age = age;
    }
    
    // 自动生成的 Equals, GetHashCode, ToString 方法
}
```

### 与命名类型的对比

| 特性                   | 匿名对象                      | 命名类/结构体                |
| ---------------------- | ----------------------------- | ---------------------------- |
| **定义**               | 编译器自动生成，无源码名称    | 必须手动定义                 |
| **属性可写性**         | 只读                          | 可读可写或只读               |
| **Equals/GetHashCode** | 自动基于所有属性实现          | 需手动重写                   |
| **ToString**           | 自动按属性输出                | 默认输出类型全名，或手动重写 |
| **可复用性**           | 仅在本方法/本表达式作用域使用 | 全局可见、可复用             |
| **用途**               | 临时封装、LINQ 投影           | 长期持有、跨层传递           |

### 在 LINQ 中的典型用法

匿名对象最常见于 `LINQ` 的 `select` 投影，将原始类型映射到只关心的字段集合：

```csharp
var people = new[]
{
    new { Name = "Alice", Age = 30, Country = "USA" },
    new { Name = "Bob",   Age = 25, Country = "UK" },
    new { Name = "Carol", Age = 28, Country = "USA" },
};

var query = people
    .Where(p => p.Country == "USA")
    .Select(p => new { p.Name, p.Age });  // 只投影 Name 和 Age

foreach (var item in query)
    Console.WriteLine($"{item.Name} is {item.Age} years old");
```

* `new { p.Name, p.Age }` 是属性名与源属性同名的简写，相当于 `new { Name = p.Name, Age = p.Age }`。

### 类型推断与作用域

* 匿名类型在同一程序集中同一属性名/顺序/类型出现时会被视为“同一类型”。

* 如果在不同方法或不同项目（不同程序集）里定义了形状相同的匿名对象，它们并不是相同的类型，不能互相赋值。

```csharp
// 方法 A
var p1 = new { X = 1, Y = 2 };
// 方法 B
var p2 = new { X = 3, Y = 4 };
// 即使属性完全一样，p1、p2 在不同方法范围内，也有相同编译时类型
// 但在不同项目或不同编译单元里，类型不同，不能跨程序集传递
```

### 限制与注意事项

* 只能有属性：匿名类型不能定义方法、字段或事件，只能有公有只读属性。

* 不可作为方法返回类型显式声明：只能用 `var` 接受，或返回 `object`/接口/动态，但这样会丢失静态类型安全。

* 不可用于公开 `API`：不宜在公共方法签名中使用匿名类型，因外部无法引用其实际类型。

* 不可修改：所有属性只读；如果需要改值，需创建新实例。

### 与 dynamic、元组对比

| 特性          | 匿名对象                 | `dynamic`                | 值元组 `(… )`                  |
| ------------- | ------------------------ | ------------------------ | ------------------------------ |
| **静态类型**  | 静态类型（有编译期检查） | 动态类型（仅运行期检查） | 静态类型                       |
| **属性命名**  | 自定义命名               | 任意命名                 | `Item1`/`Item2` 或具名元组字段 |
| **只读/可写** | 只读                     | 可读可写                 | 可读可写                       |
| **Equals**    | 基于属性自动实现         | 调用动态对象的 `Equals`  | 元组值相等比较                 |
| **用途**      | 临时封装、LINQ 投影      | 运行期灵活、COM/反射场景 | 临时打包多值返回               |

### 使用限制

```csharp
// 1. 不能作为方法参数或返回值
public void Process(/* 错误：var data */) { } 

// 2. 不能在类中定义字段
class MyClass {
    // private var _data; // 错误
}

// 3. 属性只读
var item = new { Value = 10 };
// item.Value = 20; // 编译错误

// 4. 不能添加方法
var calculator = new { 
    // int Add(int a, int b) => a + b; // 不允许
};
```

### 高级用法

#### 进阶：在方法间传递匿名对象

虽然不能在方法签名里写明匿名类型，但可以利用 泛型 与 类型推断：

```csharp
T Echo<T>(T obj) => obj;

// 调用时：
var anon = new { Name = "X", Age = 99 };
var same = Echo(anon);   // T 推断为该匿名类型
Console.WriteLine(same.Name);  // 依然可访问
```

#### 嵌套匿名对象

```csharp
var company = new {
    Name = "TechCorp",
    Address = new {
        Street = "123 Main St",
        City = "Seattle"
    },
    Employees = new[] {
        new { Name = "Alice", Position = "Developer" },
        new { Name = "Bob", Position = "Manager" }
    }
};

Console.WriteLine(company.Address.City); // Seattle
```

#### 动态类型转换

```csharp
// 匿名对象转动态类型
dynamic dynamicPerson = new {
    FirstName = "John",
    LastName = "Doe"
};

Console.WriteLine(dynamicPerson.FirstName); // John

// 动态类型转匿名对象（需类型匹配）
var anonymousPerson = (object)dynamicPerson as dynamic;
```

#### JSON 序列化

```csharp
using System.Text.Json;

var data = new {
    Timestamp = DateTime.UtcNow,
    Event = "UserLogin",
    Details = new {
        UserId = 12345,
        IP = "192.168.1.1"
    }
};

string json = JsonSerializer.Serialize(data);
// 输出: 
// {"Timestamp":"2023-07-15T12:30:45Z","Event":"UserLogin","Details":{"UserId":12345,"IP":"192.168.1.1"}}
```

#### Web API 临时响应

```csharp
[HttpGet("user/{id}")]
public IActionResult GetUserSummary(int id)
{
    var user = _dbContext.Users.Find(id);
    if (user == null) return NotFound();
    
    // 使用匿名对象构建响应
    return Ok(new {
        user.Id,
        user.Username,
        JoinDate = user.CreatedAt.ToString("yyyy-MM-dd"),
        PostCount = _dbContext.Posts.Count(p => p.UserId == id)
    });
}
```

#### 数据转换管道

```csharp
public IEnumerable<dynamic> TransformData(IEnumerable<DataRecord> records)
{
    return records
        .Where(r => r.IsValid)
        .Select(r => new {
            Key = r.Id.ToString("D5"),
            Value = r.Amount * r.ConversionRate,
            Category = GetCategory(r.Type)
        })
        .GroupBy(x => x.Category)
        .Select(g => new {
            Category = g.Key,
            Total = g.Sum(x => x.Value),
            Items = g.ToList()
        });
}
```

#### 单元测试数据准备

```csharp
[Test]
public void CalculateTotal_ShouldReturnCorrectSum()
{
    // 准备测试数据
    var items = new[] {
        new { Name = "Item1", Price = 10.99m, Quantity = 2 },
        new { Name = "Item2", Price = 5.49m, Quantity = 3 },
        new { Name = "Item3", Price = 15.00m, Quantity = 1 }
    };
    
    // 执行测试
    decimal total = Calculator.CalculateTotal(items);
    
    // 验证结果
    Assert.AreEqual(10.99m*2 + 5.49m*3 + 15.00m, total);
}
```

#### 结合动态类型

```csharp
dynamic dynObj = new { Message = "Hello", Code = 200 };
Console.WriteLine(dynObj.Message); // 运行时解析

// 匿名类型转动态 ExpandoObject
dynamic expando = new ExpandoObject();
var anon = new { A = 1, B = 2 };
foreach (var prop in anon.GetType().GetProperties())
{
    ((IDictionary<string, object>)expando)[prop.Name] = prop.GetValue(anon);
}
```

#### 反射操作

```csharp
var anon = new { Title = "C# Guide", Pages = 300 };

// 读取属性
PropertyInfo prop = anon.GetType().GetProperty("Title");
string value = (string)prop.GetValue(anon); // "C# Guide"

// 动态创建匿名类型
Type anonType = RuntimeHelpers.GetAnonymousType(
    new { Name = default(string), Age = default(int) }.GetType()
);
object newObj = Activator.CreateInstance(anonType, "Tom", 25);
```

#### 动态查询构建器

```csharp
public static IQueryable<object> BuildDynamicQuery(
    IQueryable<User> source, 
    params string[] fields)
{
    var parameter = Expression.Parameter(typeof(User), "u");
    var bindings = fields.Select(field => 
        Expression.Bind(
            typeof(User).GetProperty(field),
            Expression.Property(parameter, field)
        )
    );
    
    var body = Expression.MemberInit(
        Expression.New(typeof(User)), 
        bindings
    );
    
    var selector = Expression.Lambda(body, parameter);
    return source.Select((dynamic)selector);
}

// 使用：仅查询指定字段
var result = BuildDynamicQuery(db.Users, "Id", "Username").ToList();
```