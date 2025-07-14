### 简介

泛型（`Generics`）是指在类型或方法定义时使用类型参数，以实现类型安全、可重用和高性能的数据结构与算法

**为什么需要泛型**

* 类型安全

    * 防止“装箱／拆箱”带来的性能损耗，并在编译时检测类型错误。

* 可重用

    * 同一份代码可应用于多种数据类型，无需重复实现。

* 高性能

    * 值类型无需装箱即可存入泛型容器，减少 `GC` 压力。

```csharp
// 非泛型示例（问题：类型不安全，需要装箱）
ArrayList list = new ArrayList();
list.Add(42);        // 装箱 int → object
int num = (int)list[0]; // 拆箱 + 强制转换

// 泛型解决方案
List<int> genericList = new List<int>();
genericList.Add(42);  // 直接存储 int
int num = genericList[0]; // 直接访问
```

### 泛型多种表现

#### 泛型类

```csharp
public class Repository<T>
{
    private readonly List<T> _items = new();

    public void Add(T item) => _items.Add(item);
    public T Get(int index) => _items[index];
    public IEnumerable<T> GetAll() => _items;
}
```

* `T` 是类型参数，可在实例化时指定具体类型：

```csharp
var intRepo   = new Repository<int>();
var userRepo  = new Repository<User>();
```

#### 泛型方法

```csharp
public static class Utils
{
    public static void Swap<T>(ref T a, ref T b)
    {
        T temp = a; a = b; b = temp;
    }
}

// 使用
int x = 1, y = 2;
Utils.Swap(ref x, ref y);  // T 推断为 int
```

* 方法级别的类型参数 `T` 与类级别独立：可在非泛型类型中定义泛型方法。

#### 泛型接口

```csharp
public interface IValidator<T>
{
    bool Validate(T entity);
}

public class UserValidator : IValidator<User>
{
    public bool Validate(User user) => !string.IsNullOrEmpty(user.Name);
}
```

#### 泛型结构体

```csharp
public struct Nullable<T> where T : struct
{
    private T _value;
    private bool _hasValue;
    
    public Nullable(T value)
    {
        _value = value;
        _hasValue = true;
    }
    
    public T Value => _hasValue ? _value : throw new InvalidOperationException();
}
```

#### 泛型委托

```csharp
public delegate TOutput Converter<TInput, TOutput>(TInput input);

// 使用
Converter<string, int> parser = int.Parse;
int num = parser("42");
```

### 泛型约束 (where 子句)

为类型参数指定约束，保证编译时可用的成员：

| 约束                        | 语义                                           |
| --------------------------- | ---------------------------------------------- |
| `where T : struct`          | 值类型（排除 Nullable<T>）                     |
| `where T : class`           | 引用类型                                       |
| `where T : new()`           | 必须有公开无参构造函数                         |
| `where T : BaseClass`       | 必须继承自 `BaseClass`                         |
| `where T : IInterface`      | 必须实现接口                                   |
| `where T : Enum`      | T 必须是枚举类型                                   |
| `where T : Delegate`      | T 必须是委托类型                                   |
| `where T : unmanaged`       | 非托管类型（C# 7.3+，可用于 `Span<T>`）        |
| `where T : notnull`         | C# 8.0+，排除可空引用与 Nullable<T>            |
| `where T : U`               | `T` 必须继承或实现 `U`                         |
| `where T : class?, struct?` | C# 8.0 nullable 注释上下文下的可空约束（少用） |

**示例**

```csharp
public class Factory<T>
    where T : IEntity, new()
{
    public T Create() => new T();          // new() 约束允许 new T()
    public void Save(T entity) where T : IEntity
    {
        Console.WriteLine(entity.Id);
    }
}
```

### 协变（Covariance）与逆变（Contravariance）

用于接口和委托类型参数，以在安全范围内放宽类型兼容规则。

| 术语 | 关键字 | 示例                 | 说明                                                 |
| ---- | ------ | -------------------- | ---------------------------------------------------- |
| 协变 | `out`  | `IEnumerable<out T>` | 可将 `IEnumerable<Derived>` 赋给 `IEnumerable<Base>` |
| 逆变 | `in`   | `IComparer<in T>`    | 可将 `IComparer<Base>` 赋给 `IComparer<Derived>`     |

```csharp
IEnumerable<string> strs = new List<string>();
IEnumerable<object> objs = strs;  // 协变：string → object

IComparer<object> cmpObj = Comparer<object>.Default;
IComparer<string> cmpStr = cmpObj;  // 逆变：object ← string
```

* 限制：只支持接口与委托，并且只能用在输入（逆变）或输出（协变）位置。

### 泛型与反射

可在运行时通过反射获取和构造泛型类型：

```csharp
Type listType = typeof(List<>);
Type intListType = listType.MakeGenericType(typeof(int));
object list = Activator.CreateInstance(intListType);
MethodInfo addMethod = intListType.GetMethod("Add");
addMethod.Invoke(list, new object[] { 42 });
```

### 泛型委托和事件

标准泛型委托类型：

* `Func<T1,T2,…,TResult>`

* `Action<T1,…,Tn>`

* `Predicate<T>`

* `Comparison<T>`

* `Converter<TSource,TDest>`

### JIT编译机制

具体化（`Reification`）：`C#` 泛型在运行时会为每个类型参数生成独立的原生代码。例如：

`List<int>` 和 `List<string>` 会被 `JIT` 编译为不同的本地代码，直接操作 `int` 和 `string` 类型。

### 高级泛型技巧

#### 泛型委托

```csharp
// 泛型委托定义
Func<T, bool> Predicate<T>;      // 接收T参数，返回bool
Action<T> Action<T>;              // 接收T参数，无返回值
Func<T1, T2, TResult> Func<T1, T2, TResult>; // 多参数泛型委托

// 使用示例
List<int> numbers = new List<int> { 1, 2, 3, 4 };
// 筛选偶数：Predicate<int> 等价于 Func<int, bool>
IEnumerable<int> evenNumbers = numbers.Where(n => n % 2 == 0);

// 自定义泛型委托
delegate T Converter<in T, out U>(T input); // 协变与逆变（in/out关键字）
```

#### 泛型反射

```csharp
Type openType = typeof(List<>);
Type closedType = openType.MakeGenericType(typeof(int));
object list = Activator.CreateInstance(closedType);  // 创建 List<int>
```

#### 默认值表达式

```csharp
public T GetDefault<T>() => default(T);  // int:0, string:null
```

#### 泛型缓存（静态字段隔离）

```csharp
public class Cache<T>
{
    public static readonly DateTime Created = DateTime.Now;
}
// 每种类型有独立副本
Console.WriteLine(Cache<int>.Created); 
Console.WriteLine(Cache<string>.Created);
```

#### 条件泛型方法

```csharp
public T Process<T>(T input)
{
    if (input is int i) return (T)(object)(i * 2);
    if (input is string s) return (T)(object)(s.ToUpper());
    return input;
}
```

#### 递归泛型约束

用于定义类型参数与自身的继承关系（如实现链式调用）：

```csharp
// T必须继承自BaseClass<T>
public abstract class BaseClass<T> where T : BaseClass<T>
{
    public T SetValue(T value)
    {
        // 链式调用：返回this的派生类型
        return this as T;
    }
}

// 派生类示例
public class DerivedClass : BaseClass<DerivedClass>
{
    // ...
}
```

#### 命名规范

```csharp
public class Repository<TEntity>         // TEntity 描述性名称
public delegate TOutput Mapper<TInput, TOutput>  // TInput/TOutput 清晰
public TValue GetValue<TKey, TValue>     // 多参数区分
```

#### 接口静态抽象成员

```csharp
public interface IAddable<T> where T : IAddable<T>
{
    static abstract T operator +(T left, T right);
}

public T Sum<T>(T a, T b) where T : IAddable<T>
{
    return a + b;
}
```

### 泛型的实际应用场景

#### 集合类

```csharp
List<int> numbers = new();
Dictionary<string, DateTime> events = new();
HashSet<Guid> ids = new();
```

#### 委托系统

```csharp
Action<int> printNumber = Console.WriteLine;
Func<double, double> square = x => x * x;
Predicate<string> isLong = s => s.Length > 10;
```

#### Nullable值类型

```csharp
Nullable<int> nullableInt = 42; // 等价于 int?
if (nullableInt.HasValue)
    Console.WriteLine(nullableInt.Value);
```

#### 异步编程

```csharp
Task<int> FetchDataAsync() { ... }
ValueTask<DateTime> GetTimestampAsync() { ... }
```

#### 仓储模式与数据访问

```csharp
// 泛型仓储接口
public interface IGenericRepository<T> where T : class
{
    T GetById(int id);
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
}

// 实现类
public class EfGenericRepository<T> : IGenericRepository<T> where T : class
{
    private readonly DbContext _context;
    private readonly DbSet<T> _dbSet;
    
    public EfGenericRepository(DbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }
    
    // 实现接口方法...
}
```

#### 通用工具类与扩展方法

```csharp
// 泛型空值检查工具
public static class Guard
{
    public static T NotNull<T>(T value, string parameterName) where T : class
    {
        if (value == null)
            throw new ArgumentNullException(parameterName);
        return value;
    }
}

// 使用示例
public void ProcessData(string data)
{
    string processed = Guard.NotNull(data, nameof(data));
    // ...
}
```

#### 泛型工厂模式

```csharp
public interface ILogger
{
    void Log(string message);
}

public class FileLogger : ILogger { ... }
public class DatabaseLogger : ILogger { ... }

// 泛型工厂
public static class LoggerFactory
{
    public static T CreateLogger<T>() where T : ILogger, new()
    {
        var logger = new T();
        logger.Initialize(); // 约束确保有Initialize方法
        return logger;
    }
}

// 使用
var fileLogger = LoggerFactory.CreateLogger<FileLogger>();
```

#### 依赖注入

```csharp
services.AddScoped(typeof(IRepository<>), typeof(MemoryRepository<>));
```

### 泛型限制与解决方案

* 不能直接实例化枚举 => 使用 `Enum` 约束（ `C# 7.3+` ）

* 不能使用算术运算符 => 通过 `dynamic` 或表达式树计算

* 不能创建泛型数组 => 使用 `Array.CreateInstance`

* 不能重载仅泛型不同的方法 => 添加额外参数或使用不同方法名