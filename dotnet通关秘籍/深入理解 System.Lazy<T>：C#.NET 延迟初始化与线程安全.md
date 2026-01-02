### 什么是 `Lazy<T>`？ 

`System.Lazy<T>` 是 `.NET Framework 4.0` 引入（位于 `System` 命名空间）的泛型类，用于实现线程安全的延迟初始化（`Lazy Initialization`）。它确保一个昂贵的对象或资源只在第一次真正需要时才被创建，并且在多线程环境下保证初始化只发生一次。

* 核心特性：

    * 延迟计算：值的创建被推迟到第一次访问 `.Value` 属性时。

    * 线程安全：内置多种线程安全模式，默认情况下完全线程安全。

    * 异常缓存：如果初始化过程中抛出异常，后续访问会重复抛出同一个异常（不重复执行工厂方法）。

    * 值缓存：一旦初始化完成，后续所有访问都返回同一个实例（单例行为）。

* 适用场景：

    * 昂贵资源初始化（如数据库连接、大文件加载、复杂计算）。

    * 配置对象、单例模式（比双检查锁更安全简洁）。

    * 避免启动时不必要的开销（尤其在某些代码路径可能不使用该对象时）。

### 为什么使用 `Lazy<T>`？

传统方式（如直接在字段初始化或手动双检查锁）存在问题：

* 提前初始化：浪费资源。

* 手动锁：易出错（死锁、竞争条件、`ABA` 问题）。

* 异常处理复杂：需手动缓存异常。

`Lazy<T>`性能优化：按需初始化。

* 线程安全：微软高度优化，无锁或轻量锁实现。

* 代码简洁：一行代码取代复杂的双检查锁。

* 防 `ABA`：内部使用复合状态 + `CAS` 机制，彻底避免 `ABA` 问题。 完美解决这些问题：

### 如何使用 `Lazy<T>`？

#### 核心语法与属性

| 成员                                 | 作用                                                                             |
| ------------------------------------ | -------------------------------------------------------------------------------- |
| `Lazy<T>()`                          | 构造函数：默认使用`T`的无参构造创建对象，线程安全模式为`ExecutionAndPublication` |
| `Lazy<T>(Func<T> valueFactory)`      | 构造函数：自定义对象创建逻辑（支持传参、复杂初始化）                             |
| `Lazy<T>(LazyThreadSafetyMode mode)` | 构造函数：指定线程安全模式                                                       |
| `Value`                              | 核心属性：首次访问触发对象初始化，后续访问返回已创建的实例（只读）               |
| `IsValueCreated`                     | 布尔属性：判断对象是否已完成初始化                                               |

#### 基础示例（默认构造）

```csharp
using System;
using System.Threading;

// 模拟创建成本高的对象（如加载配置、连接数据库）
public class ExpensiveObject
{
    public ExpensiveObject()
    {
        Console.WriteLine("ExpensiveObject 开始初始化（耗时操作）...");
        Thread.Sleep(2000); // 模拟2秒耗时初始化
        Console.WriteLine("ExpensiveObject 初始化完成！");
    }

    public void DoWork() => Console.WriteLine("ExpensiveObject 执行业务逻辑...");
}

class LazyBasicDemo
{
    static void Main()
    {
        Console.WriteLine("程序启动，创建Lazy<ExpensiveObject>实例...");
        // 此时仅创建Lazy<T>容器，ExpensiveObject并未实例化
        Lazy<ExpensiveObject> lazyObj = new Lazy<ExpensiveObject>();

        Console.WriteLine($"对象是否已创建：{lazyObj.IsValueCreated}"); // 输出：False

        // 首次访问Value：触发ExpensiveObject的构造函数
        Console.WriteLine("\n=== 首次访问Value ===");
        lazyObj.Value.DoWork();
        Console.WriteLine($"对象是否已创建：{lazyObj.IsValueCreated}"); // 输出：True

        // 再次访问Value：直接返回已创建的实例，不再初始化
        Console.WriteLine("\n=== 再次访问Value ===");
        lazyObj.Value.DoWork();
    }
}
```

输出结果：

```
程序启动，创建Lazy<ExpensiveObject>实例...
对象是否已创建：False

=== 首次访问Value ===
ExpensiveObject 开始初始化（耗时操作）...
ExpensiveObject 初始化完成！
ExpensiveObject 执行业务逻辑...
对象是否已创建：True

=== 再次访问Value ===
ExpensiveObject 执行业务逻辑...
```

#### 自定义工厂方法（带参数初始化）

若对象需要传参或复杂初始化逻辑，使用 `Func<T>` 工厂方法：

```csharp
// 自定义带参数的对象构造
public class ConfigObject
{
    private string _configPath;
    public ConfigObject(string configPath)
    {
        _configPath = configPath;
        Console.WriteLine($"加载配置文件：{configPath}");
    }

    public string GetConfig() => $"配置内容（来自{_configPath}）";
}

class LazyFactoryDemo
{
    static void Main()
    {
        // 自定义工厂方法：创建ConfigObject并传入参数
        Lazy<ConfigObject> lazyConfig = new Lazy<ConfigObject>(() => 
        {
            Console.WriteLine("执行自定义工厂方法...");
            return new ConfigObject("appsettings.json");
        });

        Console.WriteLine("首次访问配置：");
        Console.WriteLine(lazyConfig.Value.GetConfig()); // 触发工厂方法
    }
}
```

#### `Lazy<T>` 的线程安全模式

`Lazy<T>` 的关键特性是支持多线程场景，通过 `LazyThreadSafetyMode` 枚举控制线程安全行为

| 线程安全模式                      | 适用场景                | 核心行为                                                             | 性能         |
| --------------------------------- | ----------------------- | -------------------------------------------------------------------- | ------------ |
| `None`                            | 单线程环境              | 无线程安全保障，多线程同时访问`Value`可能创建多个实例                | 最高（无锁） |
| `PublicationOnly`                 | 多线程 + 对象创建成本低 | 多线程可同时初始化，但最终仅保留一个实例（其他实例被丢弃），无锁阻塞 | 高           |
| `ExecutionAndPublication`（默认） | 多线程 + 对象创建成本高 | 加锁保证只有一个线程执行初始化，其他线程等待，绝对单实例             |

示例：多线程下的线程安全模式

```csharp
class LazyThreadSafetyDemo
{
    // 默认模式：ExecutionAndPublication（加锁保证单实例）
    static Lazy<ExpensiveObject> lazySafeObj = new Lazy<ExpensiveObject>();

    static void AccessLazyObj()
    {
        Console.WriteLine($"线程{Thread.CurrentThread.ManagedThreadId} 尝试访问Value...");
        var obj = lazySafeObj.Value;
        Console.WriteLine($"线程{Thread.CurrentThread.ManagedThreadId} 获取对象HashCode：{obj.GetHashCode()}");
    }

    static void Main()
    {
        // 启动5个线程同时访问
        for (int i = 0; i < 5; i++)
        {
            new Thread(AccessLazyObj).Start();
        }
        Thread.Sleep(3000); // 等待所有线程完成
    }
}
```

输出结果

所有线程最终获取的 `HashCode` 完全相同，且仅触发一次初始化（证明单实例）：

```csharp
线程3 尝试访问Value...
ExpensiveObject 开始初始化（耗时操作）...
线程4 尝试访问Value...
线程5 尝试访问Value...
线程6 尝试访问Value...
线程7 尝试访问Value...
ExpensiveObject 初始化完成！
线程3 获取对象HashCode：46104728
线程4 获取对象HashCode：46104728
线程5 获取对象HashCode：46104728
线程6 获取对象HashCode：46104728
线程7 获取对象HashCode：46104728
```

### 高级应用场景

#### 懒加载单例模式

`Lazy<T>` 是 `.NET` 中实现线程安全、懒加载单例的最优方式：

```csharp
public sealed class LazySingleton
{
    // 私有静态Lazy实例：延迟初始化，默认线程安全
    private static readonly Lazy<LazySingleton> _lazyInstance = new Lazy<LazySingleton>(() => new LazySingleton());

    // 私有构造函数：禁止外部实例化
    private LazySingleton() 
    {
        Console.WriteLine("单例对象初始化...");
    }

    // 公共属性：首次访问时创建实例
    public static LazySingleton Instance => _lazyInstance.Value;

    public void DoBusiness() => Console.WriteLine("单例对象执行业务逻辑...");
}

// 使用
class SingletonDemo
{
    static void Main()
    {
        Console.WriteLine("程序启动，未访问单例...");
        // 首次访问Instance：触发单例初始化
        LazySingleton.Instance.DoBusiness();
        // 再次访问：直接返回已创建的实例
        LazySingleton.Instance.DoBusiness();
    }
}
```

#### 处理初始化异常

若 `Lazy<T>` 的工厂方法抛出异常，后续访问 `Value` 会缓存并重新抛出同一异常

```csharp
Lazy<ExpensiveObject> lazyErrorObj = new Lazy<ExpensiveObject>(() =>
{
    throw new InvalidOperationException("初始化失败：配置文件不存在！");
});

try
{
    // 首次访问：抛出异常
    lazyErrorObj.Value.DoWork();
}
catch (AggregateException ex)
{
    Console.WriteLine($"初始化异常：{ex.InnerException.Message}");
}

try
{
    // 再次访问：仍抛出同一异常（异常被缓存）
    lazyErrorObj.Value.DoWork();
}
catch (AggregateException ex)
{
    Console.WriteLine($"再次访问异常：{ex.InnerException.Message}");
}
```

#### 异步懒加载（伪 AsyncLazy）

```csharp
private readonly Lazy<Task<ExpensiveObject>> _asyncResource = new(async () =>
{
    await Task.Delay(2000);
    return new ExpensiveObject();
});

public async Task<ExpensiveObject> GetResourceAsync() => await _asyncResource.Value;
```

#### 支持刷新（Reset）的自定义包装

```csharp
public class RefreshableLazy<T>
{
    private Lazy<T> _current;

    public RefreshableLazy(Func<T> factory)
    {
        _current = new Lazy<T>(factory);
    }

    public T Value => _current.Value;

    public void Refresh()
    {
        var factory = _current.Value; // 保留旧工厂？或传入新工厂
        _current = new Lazy<T>(_current.Factory); // 重新创建
    }
}
```

#### 与依赖注入结合（ASP.NET Core）

```csharp
services.AddSingleton<ExpensiveService>();
services.AddSingleton<Lazy<ExpensiveService>>(provider => 
    new Lazy<ExpensiveService>(provider.GetRequiredService<ExpensiveService>));
```

#### 配置文件加载

```csharp
public class AppConfig
{
    private static readonly Lazy<AppConfig> _instance = 
        new Lazy<AppConfig>(LoadConfiguration);
    
    public static AppConfig Instance => _instance.Value;
    
    public string ConnectionString { get; }
    public int Timeout { get; }
    
    private AppConfig(string config)
    {
        // 解析配置
    }
    
    private static AppConfig LoadConfiguration()
    {
        string configData = File.ReadAllText("config.json");
        return new AppConfig(configData);
    }
}
```

### `Lazy<T>` vs 手动懒加载

手动实现线程安全的懒加载需要写双重检查锁定（易出错），而 `Lazy<T>` 已封装所有逻辑：

```csharp
// 手动实现（线程安全，需双重检查+volatile）
private volatile ExpensiveObject _manualObj;
private readonly object _lockObj = new object();
public ExpensiveObject ManualObj
{
    get
    {
        if (_manualObj == null)
        {
            lock (_lockObj)
            {
                if (_manualObj == null)
                {
                    _manualObj = new ExpensiveObject();
                }
            }
        }
        return _manualObj;
    }
}

// Lazy<T>实现（一行搞定，线程安全）
private readonly Lazy<ExpensiveObject> _lazyObj = new Lazy<ExpensiveObject>();
public ExpensiveObject LazyObj => _lazyObj.Value;
```

### `Lazy<T>` 内部实现原理（简要）

* 使用一个私有字段（如 `object? _box`）存储状态：

    * `null`：未初始化

    * `Box<T>`：已完成（包含值）

    * 异常对象：初始化失败

* 初始化时使用 `Interlocked.CompareExchange` 进行原子状态转换。

* 内部状态机结合版本机制，彻底防止 `ABA` 问题。

* `ExecutionAndPublication` 模式下使用轻量自旋 + 等待机制，确保只有一个线程执行工厂方法。

### 优点与缺点

| 方面         | 优点                                  | 缺点                                         |
| ------------ | ------------------------------------- | -------------------------------------------- |
| **线程安全** | 内置完美支持，防 ABA、防重复初始化    | None 模式下需手动注意                        |
| **易用性**   | 代码极简，一行搞定                    | 不支持主动 Reset（需自定义包装）             |
| **性能**     | 高度优化，无锁路径极快                | 第一次访问可能阻塞（但这是延迟初始化的本质） |
| **功能**     | 异常缓存、值缓存、IsValueCreated 查询 | 不支持异步初始化（需用 Lazy<Task<T>>）       |
| **适用性**   | 几乎所有懒加载场景的首选              | 若需支持刷新/重置，需额外封装                |

### 最佳实践

* 优先使用 `Lazy<T>` 替代手动双检查锁。

* 始终使用默认线程安全模式（除非明确需要 `PublicationOnly`）。

* 工厂方法应无副作用（尤其在 `PublicationOnly` 模式下）。

* 异常处理：捕获 `.Value` 抛出的异常。

* 不要在工厂方法中访问同一个 `Lazy` 实例（可能死锁）。

### 总结

* `System.Lazy<T>` 是 `.NET` 官方的延迟初始化工具，核心是首次访问 `Value` 时创建对象，提升程序启动速度和内存效率；

* 核心属性：`Value`（触发初始化）、`IsValueCreated`（判断是否已创建）；

* 线程安全模式需按需选择：单线程用 `None`，多线程高成本对象用默认的`ExecutionAndPublication`；

* 最优场景：创建成本高 / 不一定使用的对象、懒加载单例模式；
