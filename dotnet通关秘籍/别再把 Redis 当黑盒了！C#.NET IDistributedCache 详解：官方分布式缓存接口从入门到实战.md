### 简介

很多项目一提到缓存，第一反应就是：

* 上 Redis
* 存个字符串
* 查不到就回数据库

刚开始这样写没什么问题。

但项目一复杂，几个现实问题就会很快冒出来：

* 多实例部署后，本地缓存各存各的
* 业务代码里到处散着 Redis key
* 对象序列化、反序列化重复写了一堆
* 过期时间全靠手抄
* 想切换缓存底层实现时，影响一大片代码

这时候就会碰到一个官方接口：

```csharp
IDistributedCache
```

一句话先说透：

> `IDistributedCache` 是 `.NET` 官方提供的分布式缓存抽象接口，用统一 API 屏蔽底层缓存实现差异，让业务代码不用直接绑定某个具体缓存组件。

这篇文章重点讲这些事：

* `IDistributedCache` 到底解决什么问题
* 它和 `IMemoryCache` 的边界是什么
* 为什么它只操作 `byte[]`
* 在 ASP.NET Core 里怎么注册和使用
* 字符串和对象缓存应该怎么封装
* 多实例场景里它真正值钱的地方在哪
* 用几个更像真实项目的 demo，把常见写法串起来

### `IDistributedCache` 到底是什么？

可以先用最白话的方式理解：

> `IDistributedCache` 不是某一种缓存产品，而是一层官方统一接口。

底层可以接的实现并不只一个，例如：

* Redis
* SQL Server
* 其他分布式缓存提供者

而业务代码看到的则是一套统一方法：

* `Get`
* `Set`
* `Refresh`
* `Remove`

也就是说，业务层更关心“要不要缓存这条数据”，而不是“底层到底是 Redis 还是别的东西”。

### 为什么有了 Redis，还要 `IDistributedCache`？

因为 Redis 是缓存产品，`IDistributedCache` 是缓存抽象。

两者不是替代关系，而是不同层级的东西。

可以这样理解：

* Redis：具体存储方案
* `IDistributedCache`：上层统一接口

如果业务代码直接 everywhere 写 Redis client，后面就会碰到这些问题：

* 序列化逻辑散落
* key 规则散落
* 过期策略散落
* 底层实现绑死

而 `IDistributedCache` 的价值就在于：

> 先把“缓存访问方式”统一下来，再决定底层到底接什么。

### 它和 `IMemoryCache` 的区别到底在哪？

这是最容易混的一点。

先看一句最短的区别：

* `IMemoryCache`：单实例本地缓存
* `IDistributedCache`：多实例可共享缓存

看表会更直观：

| 方案 | 数据范围 | 典型场景 |
| --- | --- | --- |
| `IMemoryCache` | 当前进程内 | 单机、本地热点缓存 |
| `IDistributedCache` | 多实例共享 | Web 集群、微服务、分布式部署 |

也就是说：

* 如果项目只有一个实例，本地缓存就能解决问题，`IMemoryCache` 往往更轻
* 如果项目部署成多实例，想让缓存共享，`IDistributedCache` 才更合适

### 为什么说 `AddDistributedMemoryCache` 不是真正的分布式缓存？

这个点必须先说清楚。

虽然名字里带了：

```csharp
DistributedMemoryCache
```

但它本质上还是当前进程内的内存实现。

也就是说：

* 实例 A 有自己的一份
* 实例 B 也有自己的一份
* 两边并不会天然同步

所以它更适合：

* 本地开发
* 单元测试
* 暂时不想接 Redis，但想先跑通 `IDistributedCache` 接口

而不适合真正的多实例生产共享缓存。

### `IDistributedCache` 的核心方法有哪些？

核心 API 非常少，甚至可以说少得有点朴素。

```csharp
public interface IDistributedCache
{
    byte[]? Get(string key);
    Task<byte[]?> GetAsync(string key, CancellationToken token = default);

    void Set(string key, byte[] value, DistributedCacheEntryOptions options);
    Task SetAsync(string key, byte[] value, DistributedCacheEntryOptions options, CancellationToken token = default);

    void Refresh(string key);
    Task RefreshAsync(string key, CancellationToken token = default);

    void Remove(string key);
    Task RemoveAsync(string key, CancellationToken token = default);
}
```

可以先抓住两个关键点：

* 操作很基础
* 它默认处理的是 `byte[]`

### 为什么它只操作 `byte[]`？

因为官方接口故意把自己设计得很薄。

这意味着：

* 不替业务决定对象序列化方式
* 不替业务决定 JSON、MessagePack 还是别的格式
* 不替业务决定对象怎么编码

从抽象角度看，这种设计其实很干净：

> `IDistributedCache` 只负责存和取，不负责理解业务对象。

也正因为这样，项目里几乎都会做一层扩展封装。

### 先看最基础的接入方式

如果项目用 Redis，先装包：

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

然后在 ASP.NET Core 里注册：

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "demo:";
});

var app = builder.Build();
app.Run();
```

这里的两个配置最常见：

* `Configuration`：Redis 连接字符串
* `InstanceName`：缓存 key 前缀

`InstanceName` 非常实用，因为它能让不同系统或不同模块的 key 自动带上统一前缀。

### Demo 1：最小可用的字符串缓存

下面先看一个最小 demo。

```csharp
using Microsoft.Extensions.Caching.Distributed;

public sealed class CacheService
{
    private readonly IDistributedCache _cache;

    public CacheService(IDistributedCache cache)
    {
        _cache = cache;
    }

    public async Task SetNameAsync()
    {
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
        };

        await _cache.SetStringAsync("user:name:1", "Tom", options);
    }

    public async Task<string> GetNameAsync()
    {
        return await _cache.GetStringAsync("user:name:1");
    }
}
```

这里用到的：

* `SetStringAsync`
* `GetStringAsync`

并不是 `IDistributedCache` 接口本身的方法，而是官方提供的扩展方法。

它们的价值很直接：

* 自动做字符串和 `byte[]` 之间的编码转换
* 比直接自己处理字节数组省事很多

### 什么时候该直接用 `GetAsync/SetAsync`？

如果缓存的是：

* 二进制内容
* 自定义序列化结果
* 压缩后的字节数据

就更适合直接操作：

```csharp
GetAsync
SetAsync
```

如果只是普通字符串和 JSON 文本，通常直接用：

```csharp
GetStringAsync
SetStringAsync
```

会更顺手。

### 过期策略怎么配？

`IDistributedCache` 最常见的配置对象是：

```csharp
DistributedCacheEntryOptions
```

最常见的两个过期策略是：

* 绝对过期
* 滑动过期

#### 绝对过期

到时间就失效，不管中间有没有访问。

```csharp
var options = new DistributedCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
};
```

#### 滑动过期

只要持续访问，就继续往后顺延。

```csharp
var options = new DistributedCacheEntryOptions
{
    SlidingExpiration = TimeSpan.FromMinutes(20)
};
```

可以先这样理解：

* 数据必须定时刷新：优先绝对过期
* 用户会话、活跃状态这类数据：滑动过期更常见

### `Refresh` 是干什么的？

这个方法很容易被忽略。

它的作用不是“重新查数据库刷新缓存”，而是：

> 对已经存在的缓存项，刷新它的滑动过期时间。

也就是说，如果缓存项配置了 `SlidingExpiration`，调用：

```csharp
RefreshAsync(key)
```

可以把它继续往后续命。

如果本来配的是绝对过期，那 `Refresh` 并不会把它变成永久有效。

### Demo 2：滑动过期加 `Refresh`

```csharp
using Microsoft.Extensions.Caching.Distributed;

public sealed class SessionCacheService
{
    private readonly IDistributedCache _cache;

    public SessionCacheService(IDistributedCache cache)
    {
        _cache = cache;
    }

    public async Task SetSessionAsync(string userId)
    {
        var options = new DistributedCacheEntryOptions
        {
            SlidingExpiration = TimeSpan.FromMinutes(30)
        };

        await _cache.SetStringAsync($"session:{userId}", "online", options);
    }

    public async Task KeepAliveAsync(string userId)
    {
        await _cache.RefreshAsync($"session:{userId}");
    }
}
```

这个例子更贴近登录态这类场景：

* 只要用户还活跃，就延长过期时间
* 用户长时间不活跃，再自然失效

### 真正写项目时，为什么一定要封装对象缓存？

因为只要业务里开始缓存对象，下面这些代码几乎就会重复出现：

* 序列化
* 反序列化
* 统一 `JsonSerializerOptions`
* 默认过期时间
* 空值处理

如果每个服务都自己手写一遍，后面维护会非常痛苦。

### Demo 3：先封一层对象缓存扩展

这个扩展几乎是项目里最常见的起手式。

```csharp
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

public static class DistributedCacheExtensions
{
    private static readonly JsonSerializerOptions JsonOptions = new(JsonSerializerDefaults.Web);

    public static async Task SetObjectAsync<T>(
        this IDistributedCache cache,
        string key,
        T value,
        TimeSpan? absoluteExpire = null,
        CancellationToken cancellationToken = default)
    {
        var bytes = JsonSerializer.SerializeToUtf8Bytes(value, JsonOptions);

        var options = new DistributedCacheEntryOptions();

        if (absoluteExpire.HasValue)
        {
            options.AbsoluteExpirationRelativeToNow = absoluteExpire;
        }

        await cache.SetAsync(key, bytes, options, cancellationToken);
    }

    public static async Task<T?> GetObjectAsync<T>(
        this IDistributedCache cache,
        string key,
        CancellationToken cancellationToken = default)
    {
        var bytes = await cache.GetAsync(key, cancellationToken);

        if (bytes is null || bytes.Length == 0)
        {
            return default;
        }

        return JsonSerializer.Deserialize<T>(bytes, JsonOptions);
    }
}
```

这个扩展虽然不复杂，但价值非常高：

* 业务层不再直接碰 `byte[]`
* JSON 规则统一了
* 后面要换序列化策略时，改动集中在一处

### Demo 4：用 `IDistributedCache` 做标准的 Cache-Aside

项目里最常见的缓存模式通常还是：

```text
Cache Aside
```

也就是：

* 先查缓存
* 没有就查数据库
* 查到后回写缓存

看一个更像真实项目的例子：

```csharp
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

public sealed class ProductQueryService
{
    private readonly IDistributedCache _cache;
    private readonly ProductRepository _repository;

    public ProductQueryService(
        IDistributedCache cache,
        ProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public async Task<ProductDto?> GetByIdAsync(int id)
    {
        var key = $"product:{id}";

        var cached = await _cache.GetObjectAsync<ProductDto>(key);
        if (cached is not null)
        {
            Console.WriteLine("命中缓存");
            return cached;
        }

        Console.WriteLine("查询数据库");
        var product = await _repository.GetByIdAsync(id);

        if (product is not null)
        {
            await _cache.SetObjectAsync(key, product, TimeSpan.FromMinutes(10));
        }

        return product;
    }
}

public sealed class ProductRepository
{
    public Task<ProductDto?> GetByIdAsync(int id)
    {
        ProductDto? product = id == 1
            ? new ProductDto(1, "机械键盘", 399m)
            : null;

        return Task.FromResult(product);
    }
}

public sealed record ProductDto(int Id, string Name, decimal Price);
```

这个例子里真正重要的不是 API，而是节奏：

* 请求先走缓存
* 缓存没命中才回数据库
* 命中数据库后再把结果写回缓存

这就是绝大多数业务系统里缓存最常见的使用方式。

### 更新数据时，为什么很多场景更推荐“删缓存”？

这是缓存设计里最容易踩坑的地方。

很多人第一反应是：

* 改数据库
* 顺手把缓存也更新掉

听起来合理，但项目一复杂，问题就来了：

* 一个对象可能对应多个 key
* 一个对象可能影响详情页和列表页
* 更新逻辑里可能还有其他副作用

所以很多时候更稳的做法反而是：

* 先更新数据库
* 再删除相关缓存
* 让下次读取自动重建

### Demo 5：更新数据库后删除缓存

```csharp
using Microsoft.Extensions.Caching.Distributed;

public sealed class ProductCommandService
{
    private readonly IDistributedCache _cache;
    private readonly ProductRepository _repository;

    public ProductCommandService(
        IDistributedCache cache,
        ProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public async Task UpdateAsync(ProductDto product)
    {
        await _repository.UpdateAsync(product);
        await _cache.RemoveAsync($"product:{product.Id}");
    }
}
```

这种写法的核心优点是简单。

简单通常意味着：

* 更少的脏数据风险
* 更少的遗漏 key 风险
* 更容易和 `Cache-Aside` 模式配合

### 多实例场景里，`IDistributedCache` 真正值钱的地方在哪？

这个问题必须讲透。

因为很多系统上缓存，不是因为本地内存不够快，而是因为部署已经不是单实例了。

先看一个场景：

```text
实例 A
实例 B
实例 C
```

如果这里只用 `IMemoryCache`，会发生什么？

答案是：

* A 的缓存 A 自己知道
* B 的缓存 B 自己知道
* C 的缓存 C 自己知道

彼此并不共享。

而 `IDistributedCache` 背后如果接的是 Redis，这三个实例看到的就是同一份缓存数据。

这就是它在 Web 集群里真正有价值的地方：

> 多个节点共享同一份缓存，而不是各存各的。

### Demo 6：商品详情接口更像真实项目的写法

看一个 ASP.NET Core 控制器里的最小实践。

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Caching.Distributed;

[ApiController]
[Route("api/products")]
public sealed class ProductsController : ControllerBase
{
    private readonly ProductQueryService _service;

    public ProductsController(ProductQueryService service)
    {
        _service = service;
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<ProductDto>> Get(int id)
    {
        var product = await _service.GetByIdAsync(id);
        if (product is null)
        {
            return NotFound();
        }

        return Ok(product);
    }
}
```

真正上线时，比较推荐的结构一般是：

* 控制器不直接操作缓存
* 业务服务决定缓存 key 和缓存策略
* `IDistributedCache` 只作为基础设施能力被注入

### 只会 `Set/Get` 还不够，项目里还要想清楚什么？

真正把 `IDistributedCache` 用稳，通常还要提前想清楚这几件事：

* key 命名规则
* 序列化方式
* 过期策略
* 删除策略
* 空值是否缓存
* 热点数据是否要预热

也就是说，官方接口虽然简单，但缓存设计本身并不简单。

### Key 命名为什么一定要统一？

因为缓存 key 一旦乱起来，后面会很难删，也很难查。

比较常见的命名方式一般是：

```text
product:1
product:list:page:1
user:profile:1001
order:detail:9001
```

直接看就能知道：

* 模块是什么
* 这条数据是什么类型
* 对应的是哪个对象

### Demo 7：把 key 规则集中收口

```csharp
public static class CacheKeys
{
    public static string Product(int id) => $"product:{id}";

    public static string ProductList(int page) => $"product:list:page:{page}";

    public static string UserProfile(int userId) => $"user:profile:{userId}";
}
```

别小看这一步。

很多项目后面一团乱，就是因为 key 从第一天开始就是手写散着拼的。

### `IDistributedCache` 最常见的几个坑

#### 1. 以为 `DistributedMemoryCache` 就是分布式缓存

它不是。

它只是为了让 `IDistributedCache` 能跑起来的内存实现。

#### 2. 业务代码到处自己做 JSON 序列化

这样写几天没问题，写几个月就会很难维护。

#### 3. 把缓存当数据库

缓存是加速层，不是真实数据源。

最终正确性依赖数据库或真正的数据存储，而不是缓存。

#### 4. 过期时间一刀切

所有 key 全都 10 分钟过期，看起来省事，后面很容易出问题。

不同数据通常应该有不同 TTL。

#### 5. 忘了处理缓存空值和热点 key

缓存穿透、击穿、雪崩这些问题，并不会因为用了官方接口就自动消失。

### `IDistributedCache` 适合哪些场景？

比较适合这些场景：

* 项目已经是多实例部署
* 想让缓存跨节点共享
* 希望先用官方统一抽象把缓存层接起来
* 缓存逻辑不需要太重的多级封装

### 哪些场景反而未必非它不可？

也要把边界说清楚。

下面这些情况，未必一定要上它：

* 只有单实例本地缓存需求
* 项目只是缓存几个配置项
* 业务里其实更需要的是本地热点缓存

这种场景下：

* `IMemoryCache`
* 或者更轻的业务封装

往往就够了。

### `IDistributedCache`、Redis Client、`HybridCache` 怎么看？

可以直接这样理解：

* `IDistributedCache`：官方标准抽象，足够通用
* Redis Client：更底层、更灵活，但业务更容易直接绑到底层实现
* `HybridCache`：更现代，能力更完整，适合评估较新的官方路线

也就是说，如果只想用官方抽象先把分布式缓存接起来，`IDistributedCache` 很合适。

如果还想要更强的两级缓存和更高层的能力，再去看 `HybridCache` 会更自然。

### 总结

`IDistributedCache` 的价值，不在于 API 有多花哨。

它真正值钱的地方是：

* 官方标准接口
* 多实例共享缓存
* 业务代码不用直接绑死具体缓存实现

如果项目已经进入多实例部署，又希望缓存访问方式保持统一，那 `IDistributedCache` 就是非常自然的一层抽象。

如果只记一句话，最值得记的是：

> `IDistributedCache` 最大的意义，不是“帮忙上 Redis”，而是把分布式缓存访问收敛成一套官方统一接口，让业务代码更容易演进。
