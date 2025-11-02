### 简介

在许多应用程序中，缓存是提升性能的常见方法，尤其是在访问频繁且不经常变化的数据时。`MemoryCache` 是 `.NET` 提供的一个内存缓存实现，它允许在内存中存储数据，以减少对数据库、文件系统或其他远程服务的访问，进而提升系统响应速度。

`MemoryCache` 的核心优势是：

* 高效：内存操作非常快速，适合用于缓存短期有效的数据。

* 轻量：它是一个线程安全的缓存系统，且易于在 `.NET` 应用中配置和使用。

* 灵活：支持过期时间、优先级设置等多种功能，能够满足大多数缓存需求。

### 核心功能

* 线程安全

    * `MemoryCache` 是线程安全的，允许多个线程同时访问缓存中的数据。

* 过期策略

    * 绝对过期：指定一个具体的时间点，缓存项在该时间点后过期。

    * 滑动过期：缓存项最后访问后的指定时间段内过期。

* 缓存项优先级

    * 可以为缓存项设置优先级，允许缓存管理器在内存不足时根据优先级回收缓存项。

* 回调：

    * `PostEvictionCallback`：缓存项移除时触发回调。

    * `CacheEntryOptions`：支持自定义过期和移除逻辑。

* 依赖关系：

    * 使用 `ChangeToken` 支持基于外部信号的缓存失效（如文件更改）。

* 线程安全：内置并发控制，支持多线程访问。

* `DI` 集成：通过 `IMemoryCache` 接口与 `ASP.NET Core DI` 无缝集成。

* 数据大小限制

    * 可以设置缓存的最大容量，以防止占用过多内存。

* 支持绝对过期和滑动过期组合使用

    * 能灵活配置缓存失效的时间策略。

### 核心 API

`MemoryCache` 主要通过 `IMemoryCache` 接口操作，位于 `Microsoft.Extensions.Caching.Memory` 命名空间。核心 `API` 如下：

* 接口方法：

    * `ICacheEntry CreateEntry(object key)`：创建缓存项。

    * `bool TryGetValue(object key, out object value)`：尝试获取缓存值。

    * `void Remove(object key)`：移除缓存项。

    * `TItem Get<TItem>(object key)`：获取指定类型的缓存值。

    * `TItem GetOrCreate<TItem>(object key, Func<ICacheEntry, TItem> factory)`：获取或创建缓存值。

    * `Task<TItem> GetOrCreateAsync<TItem>(object key, Func<ICacheEntry, Task<TItem>> factory)`：异步获取或创建。

    * `TItem Set<TItem>(object key, TItem value, MemoryCacheEntryOptions options = null)`：设置缓存值。

* `MemoryCacheEntryOptions`：

    * `DateTimeOffset? AbsoluteExpiration`：绝对过期时间。

    * `TimeSpan? AbsoluteExpirationRelativeToNow`：相对当前时间的绝对过期。

    * `TimeSpan? SlidingExpiration`：滑动过期时间。

    * `CacheItemPriority Priority`：缓存项优先级。

    * `IChangeToken ExpirationTokens`：依赖的变更令牌。

    * `Action<ICacheEntry, EvictionReason, object> PostEvictionCallbacks`：移除回调。

    * `long? Size`：缓存项大小（用于大小限制，`.NET 6+`）。

* `MemoryCacheOptions`：

    * `TimeSpan ExpirationScanFrequency`：过期扫描间隔（默认 1 分钟）。

    * `long? SizeLimit`：缓存总大小限制（`.NET 6+`）。

    * `double CompactionPercentage`：内存压力下压缩比例（`.NET 6+`）。


> 注意：
> 旧版本.net 使用 `System.Runtime.Caching`
> 新版本.net 使用 `Microsoft.Extensions.Caching.Memory`


### API 用法

#### 创建与初始化 `MemoryCache`

```csharp
using System.Runtime.Caching;

// 创建默认的 MemoryCache 实例
MemoryCache cache = MemoryCache.Default;

// 或者创建带名称的实例
MemoryCache customCache = new MemoryCache("MyCache");
```

#### 添加缓存项

```csharp
CacheItemPolicy policy = new CacheItemPolicy
{
    AbsoluteExpiration = DateTimeOffset.Now.AddMinutes(10), // 设置绝对过期时间
    SlidingExpiration = TimeSpan.FromMinutes(5),           // 设置滑动过期时间
    Priority = CacheItemPriority.Default,                  // 设置优先级
    RemovedCallback = args => { Console.WriteLine("缓存项已移除"); } // 过期时执行回调
};

cache.Add("key", "value", policy); // 向缓存中添加项
```

#### 获取缓存项

```csharp
var value = cache.Get("key"); // 获取缓存项
Console.WriteLine(value); // 输出：value
```

#### 更新缓存项

```csharp
cache.Set("key", "newValue", DateTimeOffset.Now.AddMinutes(5)); // 更新缓存项
```

#### 移除缓存项

```csharp
cache.Remove("key"); // 移除缓存项
```

#### 使用 TryGetValue 方法检查缓存项

```csharp
object value;
if (cache.TryGetValue("key", out value))
{
    Console.WriteLine(value); // 如果存在，打印缓存值
}
else
{
    Console.WriteLine("缓存项不存在"); // 如果不存在，打印提示
}
```

#### 获取所有缓存项（遍历）

```csharp
foreach (var item in cache)
{
    Console.WriteLine($"Key: {item.Key}, Value: {item.Value}");
}
```

#### 自定义缓存实例

```csharp
// 创建带自定义配置的缓存
var cacheConfig = new NameValueCollection
{
    {"cacheMemoryLimitMegabytes", "100"}, // 100MB 内存限制
    {"physicalMemoryLimitPercentage", "50"}, // 物理内存50%
    {"pollingInterval", "00:05:00"} // 5分钟检查一次
};

var customCache = new MemoryCache("MyCustomCache", cacheConfig);

// 使用自定义缓存
customCache.Set("userData", userProfile, new CacheItemPolicy());
```

#### 优先级策略

```csharp
var highPriorityPolicy = new CacheItemPolicy
{
    Priority = CacheItemPriority.NotRemovable // 内存不足时不会被移除
};

var lowPriorityPolicy = new CacheItemPolicy
{
    Priority = CacheItemPriority.Default // 默认优先级
};
```

#### ASP.NET Core 中通过 DI 使用 IMemoryCache

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Caching.Memory;
using System;

[ApiController]
[Route("api/cache")]
public class CacheController : ControllerBase
{
    private readonly IMemoryCache _cache;

    public CacheController(IMemoryCache cache)
    {
        _cache = cache;
    }

    [HttpGet("set")]
    public IActionResult SetCache()
    {
        var key = "myKey";
        var value = $"Data at {DateTime.Now:HH:mm:ss}";
        var options = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5),
            SlidingExpiration = TimeSpan.FromSeconds(30)
        };

        _cache.Set(key, value, options);
        return Ok($"Cached: {value}");
    }

    [HttpGet("get")]
    public IActionResult GetCache()
    {
        if (_cache.TryGetValue("myKey", out string value))
            return Ok($"Cached value: {value}");
        return NotFound("Cache miss");
    }
}
```

注册服务：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddMemoryCache(); // 注册 MemoryCache
builder.Services.AddControllers();
```

#### 缓存依赖（ChangeToken）

基于外部信号失效（如文件更改）：

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Caching.Memory;
using Microsoft.Extensions.FileProviders;
using System;
using System.IO;
using System.Threading.Tasks;

[ApiController]
[Route("api/config")]
public class ConfigController : ControllerBase
{
    private readonly IMemoryCache _cache;
    private readonly PhysicalFileProvider _fileProvider;

    public ConfigController(IMemoryCache cache)
    {
        _cache = cache;
        _fileProvider = new PhysicalFileProvider(Directory.GetCurrentDirectory());
    }

    [HttpGet]
    public async Task<IActionResult> GetConfig()
    {
        var cacheKey = "configKey";
        var config = await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            var file = _fileProvider.GetFileInfo("config.txt");
            entry.AddExpirationToken(_fileProvider.Watch("config.txt"));
            entry.SlidingExpiration = TimeSpan.FromMinutes(5);
            // 读取文件
            string content = await File.ReadAllTextAsync(file.PhysicalPath);
            return content;
        });

        return Ok(config);
    }
}
```

说明：

* 使用 `PhysicalFileProvider.Watch` 创建 `IChangeToken`。

* 文件更改时缓存失效，重新加载。

### 性能优化

#### 避免过多缓存数据

设置合理的缓存大小和最大容量，避免 `MemoryCache` 占用过多内存。可以通过设置缓存的最大项数或缓存最大内存大小来控制。

```csharp
var policy = new CacheItemPolicy
{
    AbsoluteExpiration = DateTimeOffset.Now.AddMinutes(10)
};
var cache = new MemoryCache("MyCache", new NameValueCollection
{
    { "CacheMemoryLimitMegabytes", "100" },  // 限制最大内存为 100 MB
    { "PhysicalMemoryLimitPercentage", "80" }, // 限制物理内存占用为 80%
    { "PollingInterval", "00:01:00" } // 每 1 分钟检查一次缓存
});
```

#### 合理使用过期策略

  * 对于一些短时间有效的数据，建议使用滑动过期（`SlidingExpiration`），它可以保证数据的最新性。

   * 对于长期不变的数据，可以使用绝对过期（`AbsoluteExpiration`），避免内存被长时间占用。

#### 避免缓存穿透

   * 在向缓存写入数据之前，确保数据已经在某些存储中存在，避免因缓存项缺失导致每次访问都去请求外部数据库。

   * 如果缓存为空，可以缓存空值，以防止缓存穿透。

```csharp
public T GetOrCreate<T>(string key, Func<T> factory, TimeSpan expiration)
{
    if (cache.Get(key) is T result) 
        return result;
    
    // 特殊值标记缓存穿透
    if ("__NULL__".Equals(cache.Get(key)))
        return default;
    
    try
    {
        result = factory();
        if (result == null)
        {
            // 缓存空值避免反复查询
            cache.Set(key, "__NULL__", TimeSpan.FromMinutes(5));
            return default;
        }
        
        cache.Set(key, result, expiration);
        return result;
    }
    catch
    {
        // 异常时设置短期空缓存
        cache.Set(key, "__NULL__", TimeSpan.FromMinutes(1));
        throw;
    }
}
```

#### 合理设置回调函数

   * 使用 `RemovedCallback` 来处理过期的缓存项，执行资源清理或日志记录等操作。

```csharp
CacheItemPolicy policy = new CacheItemPolicy
{
    RemovedCallback = args =>
    {
        Console.WriteLine($"缓存项 {args.CacheItem.Key} 被移除");
    }
};
```

#### 定期清理

* 可以通过 `PollingInterval` 来定期检查缓存并进行清理，避免缓存中存放过期数据。

#### 缓存统计监控

```csharp
// 获取缓存统计信息
var stats = ((MemoryCache)cache).GetCacheStatistics();

Console.WriteLine($"缓存命中率: {stats.CacheHitRatio:P}");
Console.WriteLine($"总缓存项: {stats.TotalCount}");
Console.WriteLine($"缓存大小: {stats.CacheSizeKB} KB");
Console.WriteLine($"内存限制: {stats.MemoryLimitMB} MB");
```

### 优缺点

#### 优点

* 高性能：进程内缓存，访问速度快。

* 轻量简单：无外部依赖，易于集成。

* 灵活过期：支持绝对、滑动和依赖失效。

* 异步支持：`GetOrCreateAsync` 适合异步加载。

* `DI` 集成：与 `ASP.NET Core` 无缝结合。

* 内存管理：`.NET 6+` 支持大小限制和压缩。

#### 缺点

* 进程内限制：数据不跨进程或实例共享，重启丢失。

* 内存占用：大数据量可能导致内存压力。

* 无分布式支持：不适合多实例部署（需用 `Redis`）。

* 手动管理：需显式设置键和过期策略。

* 有限功能：无高级功能（如标签、区域缓存）。

### 常见使用场景

#### 适用场景

* 数据缓存

    * 在 `Web` 应用中缓存数据库查询结果、`API` 请求结果等，减少数据库查询的压力，提高响应速度。

    * 示例：缓存用户信息、商品列表等。

* 会话存储

    * 使用缓存来存储用户会话信息，避免每次请求都查询数据库。

    * 示例：存储用户的登录状态或临时数据。

* 频繁访问的数据

    * 在频繁读取但不常更新的数据上使用缓存来提高性能。

    * 示例：热点新闻、热销商品、排行榜等。

* `API` 限流

    * 用于控制 `API` 调用频率，存储用户请求的时间戳，避免过多的请求对后台服务产生压力。

    * 示例：`API` 请求次数的缓存，防止暴力请求。

* 耗时操作结果缓存

    * 对于计算成本较高的操作，可以将结果缓存起来，减少重复计算。

    * 示例：图片处理后的结果、文件读取操作的缓存等。

#### 不适用场景

* 分布式系统共享数据

* 大型数据集（大于内存限制）

* 需要持久化的数据

* 严格的实时数据一致性要求

### 与相关技术对比

|  特性   |  MemoryCache   |  HttpRuntime.Cache   |  IDistributedCache   |  Redis   |
| --- | --- | --- | --- | --- |
|  存储位置   |  内存  |  内存   |  分布式存储   |  分布式   |
|  应用范围   |  通用  |  通用   |  分布式系统|  分布式  |
|  性能   |  最高   |  高   |  中   |  中高   |
|  持久化   |  ❌   |  ❌   |  ✅   |  ✅   |
|  过期策略	   |  丰富   | 丰富    |  基础   |  丰富   |
|  集群支持   |  ❌   |  ❌   |   ✅  |  ✅   |
|  .NET Core   |  ✅   |  ❌   |  ✅   |  ✅   |

### 总结

`MemoryCache` 是 `.NET` 中轻量高效的进程内缓存组件，适合高频读取、低更新频率的场景，如热点数据、配置或计算结果。它提供灵活的过期策略、回调和依赖失效，支持异步操作和 `ASP.NET Core DI`。相比 `IDistributedCache`（如 `Redis`），它性能更高但限于单实例。

### 资源和文档

* `MSDN` 文档: https://learn.microsoft.com/zh-cn/dotnet/api/system.runtime.caching.memorycache?view=net-9.0-pp

* 缓存模式指南: https://learn.microsoft.com/zh-cn/azure/architecture/patterns/cache-aside

* `MemoryCache` 源码: https://github.com/dotnet/runtime/blob/main/src/libraries/System.Runtime.Caching/src/System/Runtime/Caching/MemoryCache.cs