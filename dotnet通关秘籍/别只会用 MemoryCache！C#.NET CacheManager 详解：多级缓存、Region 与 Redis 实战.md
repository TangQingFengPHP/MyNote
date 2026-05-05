### 简介

一提到缓存，很多项目里最先出现的通常是这几样东西：

* `IMemoryCache`
* `IDistributedCache`
* Redis
* 手写一套缓存工具类

刚开始看起来都能用。

但项目一复杂，问题就会慢慢冒出来：

* 有的地方直接查内存缓存
* 有的地方直接调 Redis
* 有的地方缓存键命名完全不统一
* 有的地方更新数据库以后，忘了删缓存
* 有的地方想做“本地内存 + Redis”两级缓存，结果代码越写越乱

这时候就会碰到一个库：

```csharp
CacheManager
```

一句话先说透：

> `CacheManager` 不是具体缓存中间件，而是一层缓存抽象和管理框架，用一套统一 API 把内存缓存、Redis、多级缓存、过期策略、区域清理这些事情串起来。

这篇文章重点讲这些事：

* `CacheManager` 到底解决了什么问题
* 它和直接用 `IMemoryCache`、Redis 有什么区别
* 基础增删改查、过期策略、`Region` 应该怎么用
* 多级缓存到底怎么配
* 业务里怎么写出更像真实项目的 demo，而不是只会 `Get/Put`
* 哪些场景适合它，哪些场景反而没必要上它

### `CacheManager` 到底是什么？

可以先用最白话的方式理解：

> `CacheManager` 就像一个缓存总控台。

底层缓存可以是：

* 本地内存
* Redis
* 多层组合

而业务代码面对的则是一套统一接口。

也就是说，业务层不用到处写：

* 这里调 `IMemoryCache`
* 那里调 Redis Client
* 另一个地方再手写过期和回填逻辑

而是尽量变成：

```csharp
cache.Get(...)
cache.Put(...)
cache.Remove(...)
cache.ClearRegion(...)
```

### 为什么项目里会需要它？

因为缓存一旦从“单点加速”升级成“系统级能力”，光有一个缓存容器已经不够了。

最典型的问题有这些：

#### 1. API 不统一

本地缓存一套写法，Redis 一套写法。

换一种缓存方案，就得改一片代码。

#### 2. 多级缓存难维护

想做：

* 一级缓存：进程内内存
* 二级缓存：Redis

结果读写、回填、删除、失效逻辑散得到处都是。

#### 3. 业务分组清理麻烦

比如商品详情、商品列表、商品价格缓存，本来就是一组。

如果没有分组能力，删除时往往只能：

* 手动拼很多 key
* 或者干脆粗暴清空一大片缓存

#### 4. 策略散落在代码里

有的缓存 5 分钟过期，有的滑动过期，有的永久保留。

如果每个地方都手写，后面维护起来会越来越乱。

### `CacheManager` 的核心价值是什么？

可以压缩成三点：

* 统一缓存操作入口
* 支持多级缓存
* 让缓存策略更集中

如果只记一句话，最适合记这个：

> `CacheManager` 真正值钱的，不是“能缓存”，而是“能把复杂缓存方案管理得像一套东西”。

### 它和直接用 `IMemoryCache` 有什么区别？

这个问题很关键。

因为很多项目其实并不需要 `CacheManager`。

先看最直观的区别：

| 方案 | 适合场景 |
| --- | --- |
| `IMemoryCache` | 单机、本地缓存、简单场景 |
| `IDistributedCache` / Redis | 多实例共享缓存 |
| `CacheManager` | 需要统一 API、分层缓存、集中策略管理 |

也就是说：

* 如果项目只是简单做个本地缓存，`IMemoryCache` 很可能就够了
* 如果项目主要就是 Redis，直接封装 Redis 也未必不行
* 只有当缓存层次和策略开始复杂起来，`CacheManager` 才会更有价值

### `CacheManager` 最常见的使用方式

最常见的落地方式通常是：

* 一个内存缓存句柄
* 一个 Redis 缓存句柄
* 业务通过统一 `ICacheManager<T>` 访问

于是就形成了很常见的两级缓存结构：

```text
请求
 -> 先查本地内存
 -> 本地没有，再查 Redis
 -> Redis 有，就回填本地
 -> Redis 也没有，再查数据库
```

这也是很多人接触 `CacheManager` 的主要原因。

### 先看最基础的内存缓存 Demo

先别急着上 Redis，先把最基础的用法跑通。

安装核心包和内存句柄包：

```bash
dotnet add package CacheManager.Core
dotnet add package CacheManager.Microsoft.Extensions.Caching.Memory
```

然后创建一个最小内存缓存：

```csharp
using CacheManager.Core;

var cache = CacheFactory.Build<string>("userCache", settings =>
{
    settings.WithMicrosoftMemoryCacheHandle("memory");
});

cache.Put("user:1", "Tom");

var value = cache.Get("user:1");
Console.WriteLine(value);
```

这个例子虽然简单，但已经把 `CacheManager` 的基本用法带出来了：

* 先建一个缓存实例
* 底层用什么 handle，在配置里说清楚
* 上层业务只面对统一 API

### 常见基础操作有哪些？

最常见的就是这几个：

* `Add`
* `Put`
* `Get`
* `TryGet`
* `Remove`
* `Exists`

看一个稍微完整点的 demo：

```csharp
using CacheManager.Core;

var cache = CacheFactory.Build<string>("demoCache", settings =>
{
    settings.WithMicrosoftMemoryCacheHandle("memory");
});

cache.Add("product:1", "iPhone");
cache.Put("product:2", "iPad");

Console.WriteLine(cache.Get("product:1"));
Console.WriteLine(cache.Get("product:2"));

if (cache.TryGet("product:3", out string missing))
{
    Console.WriteLine(missing);
}
else
{
    Console.WriteLine("product:3 不存在");
}

cache.Put("product:2", "iPad Pro");
Console.WriteLine(cache.Get("product:2"));

cache.Remove("product:1");
Console.WriteLine(cache.Exists("product:1"));
```

这里有个很实用的区分：

* `Add`：更偏“新增”，重复 key 通常不适合直接拿它做覆盖
* `Put`：更偏“放进去”，存在就更新，不存在就写入

实际业务里，大多数更新缓存的场景会更常用 `Put`。

### 缓存过期怎么配？

缓存离不开过期策略。

`CacheManager` 里最常见的是两种：

* 绝对过期
* 滑动过期

#### 绝对过期

到时间就过期，不管中间有没有被访问。

```csharp
cache.Put(
    "article:1001",
    "缓存内容",
    ExpirationMode.Absolute,
    TimeSpan.FromMinutes(10));
```

#### 滑动过期

只要持续访问，就往后顺延。

```csharp
cache.Put(
    "session:user:1",
    "在线状态",
    ExpirationMode.Sliding,
    TimeSpan.FromMinutes(20));
```

两者怎么选，可以直接这么理解：

* 数据必须定期刷新：优先绝对过期
* 热点数据希望“有人用就先留着”：可以考虑滑动过期

### Demo 1：商品详情缓存

下面这个 demo 更接近真实业务。

```csharp
using CacheManager.Core;

var cache = CacheFactory.Build<Product>("productCache", settings =>
{
    settings.WithMicrosoftMemoryCacheHandle("memory");
});

var product = new Product(1, "机械键盘", 399m);

cache.Put(
    "product:1",
    product,
    ExpirationMode.Absolute,
    TimeSpan.FromMinutes(5));

var cached = cache.Get("product:1");

Console.WriteLine($"{cached.Name} - {cached.Price}");

public sealed record Product(int Id, string Name, decimal Price);
```

这个例子想表达的重点不是 API，而是：

* 缓存的不一定只是字符串
* 业务对象本身也可以直接缓存
* 过期策略应该跟业务语义绑在一起看

### `Region` 是什么？为什么它很实用？

`Region` 可以理解成：

> 给一批相关缓存做分组。

比如商品模块里，可能有这些缓存：

* 商品详情
* 商品库存
* 商品价格
* 商品标签

如果都跟商品有关，那就可以放进同一个 `Region` 里管理。

看一个例子：

```csharp
using CacheManager.Core;

var cache = CacheFactory.Build<string>("regionCache", settings =>
{
    settings.WithMicrosoftMemoryCacheHandle("memory");
});

cache.Put("products", "product:1", "机械键盘");
cache.Put("products", "product:2", "无线鼠标");
cache.Put("products", "product:3", "显示器");

Console.WriteLine(cache.Get("products", "product:1"));

cache.ClearRegion("products");

Console.WriteLine(cache.Exists("products", "product:1"));
```

`Region` 最适合的场景通常是：

* 一个业务模块下面有很多相关 key
* 数据更新时需要批量失效
* 不想自己维护一堆 key 前缀和删除逻辑

### Demo 2：商品更新后，批量清理缓存

看一个更像项目代码的例子：

```csharp
using CacheManager.Core;

public sealed class ProductService
{
    private readonly ICacheManager<ProductDto> _detailCache;
    private readonly ICacheManager<string> _listCache;

    public ProductService(
        ICacheManager<ProductDto> detailCache,
        ICacheManager<string> listCache)
    {
        _detailCache = detailCache;
        _listCache = listCache;
    }

    public void UpdateProduct(ProductDto product)
    {
        // 1. 先更新数据库
        Console.WriteLine($"更新商品 {product.Id}");

        // 2. 删详情缓存
        _detailCache.Remove($"product:{product.Id}");

        // 3. 删列表区域缓存
        _listCache.ClearRegion("product-list");
    }
}

public sealed record ProductDto(int Id, string Name, decimal Price);
```

这个例子很接近真实项目里的缓存失效逻辑：

* 单条详情 key 精准删除
* 列表类缓存按区域整体失效

很多时候，缓存设计的难点不是“怎么读”，而是：

> 数据改了以后，到底该删哪一批缓存。

`Region` 在这个问题上会比纯手写 key 稳得多。

### 多级缓存到底怎么配？

这部分是 `CacheManager` 最核心也最容易被看中的地方。

安装 Redis 相关包：

```bash
dotnet add package CacheManager.StackExchange.Redis
dotnet add package CacheManager.Serialization.Json
```

然后创建一个“内存 + Redis”的两级缓存：

```csharp
using CacheManager.Core;

var cache = CacheFactory.Build<Product>("multiLevelCache", settings =>
{
    settings
        .WithMicrosoftMemoryCacheHandle("memory")
        .And
        .WithRedisConfiguration("redis", config =>
        {
            config.WithEndpoint("localhost", 6379);
        })
        .WithRedisCacheHandle("redis")
        .WithJsonSerializer();
});

cache.Put("product:1", new Product(1, "机械键盘", 399m));

var product = cache.Get("product:1");
Console.WriteLine(product.Name);

public sealed record Product(int Id, string Name, decimal Price);
```

这个结构可以先这样理解：

* 内存层负责快
* Redis 层负责共享和兜底

读取时的思路通常是：

```text
先查内存
-> 没有再查 Redis
-> Redis 命中后回填内存
```

写入时则通常希望各层都同步更新。

### 为什么多级缓存很有吸引力？

因为它刚好把两类缓存的优点拼在了一起：

#### 本地内存缓存的优点

* 访问极快
* 没有网络开销
* 适合热点数据

#### Redis 的优点

* 多实例共享
* 服务重启后不至于全丢
* 更适合做跨节点一致的数据层

如果直接只用 Redis，很多热点读取会多一层网络开销。

如果直接只用本地内存，多实例部署时又容易各存各的。

所以“本地内存 + Redis”在很多系统里会是一个很自然的折中。

### `CacheManager` 在 ASP.NET Core 里怎么注册到 DI？

控制台 demo 看懂以后，真正落地通常还是要进 ASP.NET Core。

最常见的做法是：

* 在 `Program.cs` 里创建缓存实例
* 把缓存实例注册进依赖注入容器
* 业务服务通过构造函数注入 `ICacheManager<T>`

先看一个最小可用的注册方式。

```csharp
using CacheManager.Core;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<ICacheManager<ProductDto>>(_ =>
    CacheFactory.Build<ProductDto>("productCache", settings =>
    {
        settings.WithMicrosoftMemoryCacheHandle("memory");
    }));

builder.Services.AddSingleton<ProductCacheService>();
builder.Services.AddSingleton<ProductRepository>();

var app = builder.Build();
app.Run();

public sealed record ProductDto(int Id, string Name, decimal Price);
```

这种写法适合：

* 单一业务类型缓存
* 先把接入方式跑通
* 项目里缓存规模还不算大

但项目一旦复杂起来，通常不会只缓存一种类型。

这时候更常见的做法是按业务分别注册。

### Demo 3：在 ASP.NET Core 里注册多个缓存实例

例如：

* 商品详情缓存一个实例
* 商品列表缓存一个实例
* 用户资料缓存再一个实例

可以这样写：

```csharp
using CacheManager.Core;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton(provider =>
    CacheFactory.Build<ProductDto>("product-detail-cache", settings =>
    {
        settings.WithMicrosoftMemoryCacheHandle("memory");
    }));

builder.Services.AddSingleton(provider =>
    CacheFactory.Build<string>("product-list-cache", settings =>
    {
        settings.WithMicrosoftMemoryCacheHandle("memory");
    }));

builder.Services.AddSingleton<ProductCacheService>();
builder.Services.AddSingleton<ProductQueryService>();
builder.Services.AddSingleton<ProductRepository>();

var app = builder.Build();
app.Run();

public sealed record ProductDto(int Id, string Name, decimal Price);
```

然后在业务服务里直接注入：

```csharp
using CacheManager.Core;

public sealed class ProductCacheService
{
    private readonly ICacheManager<ProductDto> _detailCache;
    private readonly ICacheManager<string> _listCache;

    public ProductCacheService(
        ICacheManager<ProductDto> detailCache,
        ICacheManager<string> listCache)
    {
        _detailCache = detailCache;
        _listCache = listCache;
    }

    public ProductDto GetDetail(int id) => _detailCache.Get($"product:{id}");

    public void RemoveDetail(int id) => _detailCache.Remove($"product:{id}");

    public void ClearList() => _listCache.ClearRegion("product-list");
}
```

这里要注意一个工程化问题：

> 如果项目里注册了多个相同泛型参数的 `ICacheManager<T>`，光靠类型本身是区分不出来的。

例如同时有两个：

* `ICacheManager<string>`
* `ICacheManager<string>`

这时候就不适合直接裸注入了。

更稳的做法通常是：

* 给每个缓存再包一层业务服务
* 或者自己定义接口，例如 `IProductDetailCache`
* 让业务代码依赖“业务语义”，而不是依赖底层缓存实例本身

### 更接近生产的 DI 注册方式

如果要接 Redis，两级缓存的注册方式通常会长这样：

```csharp
using CacheManager.Core;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<ICacheManager<ProductDto>>(_ =>
    CacheFactory.Build<ProductDto>("productCache", settings =>
    {
        settings
            .WithMicrosoftMemoryCacheHandle("memory")
            .And
            .WithRedisConfiguration("redis", config =>
            {
                config.WithEndpoint("localhost", 6379);
            })
            .WithRedisCacheHandle("redis")
            .WithJsonSerializer();
    }));

builder.Services.AddSingleton<ProductQueryService>();
builder.Services.AddSingleton<ProductRepository>();

var app = builder.Build();
app.Run();

public sealed record ProductDto(int Id, string Name, decimal Price);
```

这段配置里最核心的点有两个：

* 内存缓存和 Redis 都在一个缓存实例里统一管理
* 业务层注入时，不需要关心底层到底有几层缓存

### 在 DI 里接入时，比较推荐的习惯

实际项目里，比较推荐这些做法：

* 缓存实例用单例注册
* key 生成规则统一收口
* 过期策略不要散落在控制器里
* 业务代码尽量依赖自己的缓存服务封装，而不是到处直接调 `Get/Put`

原因很简单：

* 缓存本来就是基础设施
* 基础设施越分散，后面越难改
* 一旦换 key 规则、换过期时间、换底层缓存，实现成本会非常高

### 多实例下，本地缓存为什么会不一致？

这也是两级缓存里最容易踩坑的地方。

先看一个很典型的部署场景：

```text
实例 A：本地内存 + Redis
实例 B：本地内存 + Redis
实例 C：本地内存 + Redis
```

现在假设商品 `product:1` 被缓存在三个实例的本地内存里。

这时候有个请求打到了实例 A，并且完成了商品更新。

如果只是：

* 更新数据库
* 删除 Redis 里的 key

那会发生什么？

答案是：

* 实例 A 可能已经拿到新值
* 实例 B 和实例 C 的本地内存里，旧值可能还活着

这就是多实例下最常见的一致性问题。

也就是说：

> Redis 一致，不代表每个实例里的本地缓存也一致。

### 为什么本地缓存一致性会变难？

因为本地缓存天然是“每个进程各有一份”。

它的优点是快，但代价就是：

* 数据副本变多
* 失效广播变难
* 更新传播不是天然就有的

单实例时，本地缓存只有一份，问题不大。

多实例一上来，这个问题就会立刻暴露。

### 处理多实例本地缓存一致性，常见有哪几种办法？

工程上最常见的是这几种。

#### 1. 更新数据库后，删 Redis，也删当前实例本地缓存

这是最基础的一步，但只能保证“当前实例”正确。

对于其他实例来说，还是不够。

#### 2. 缩短本地缓存过期时间

这是一种很常见的折中。

例如：

* Redis 过期 10 分钟
* 本地内存只保留 30 秒到 1 分钟

这样即便某个实例拿到了旧值，持续时间也不会太长。

这种方案的本质是：

> 不强求强一致，换取简单和高性能。

很多读多写少的场景，其实能接受这种做法。

#### 3. 更新后发失效通知，让其他实例一起删本地缓存

这是更像正经多实例方案的做法。

思路通常是：

* 某个实例更新数据
* 更新完成后，通过 Redis Pub/Sub、消息队列或 Backplane 发一个“缓存失效通知”
* 其他实例收到通知后，把对应本地缓存删掉

于是就能形成：

```text
实例 A 更新商品
-> 删除 Redis key
-> 发布 product:1 失效消息
-> 实例 B / C 收到消息后删除本地缓存
```

这类方案的核心价值是：

* 本地缓存还能保留性能优势
* 各实例不需要等自然过期
* 旧值停留时间能大幅缩短

#### 4. 干脆不做本地缓存，只用分布式缓存

如果业务对一致性非常敏感，又不想维护广播失效机制，那还有一个很直接的办法：

* 不做本地内存层
* 直接只用 Redis

这样做会少掉本地缓存那层极致性能，但一致性会简单很多。

### 哪种一致性策略更适合什么场景？

可以直接这样理解：

* 读多写少，允许短时间旧值：本地缓存短 TTL + Redis
* 希望一致性更好，又想保留本地缓存性能：本地缓存 + Redis + 失效广播
* 数据一致性要求高，不想维护复杂同步：直接 Redis

也就是说，多实例下的关键问题不是“能不能加本地缓存”，而是：

> 本地缓存失效这件事，准备怎么同步给其他实例。

### Demo 4：更新后删除 Redis，并广播本地缓存失效

下面给一个简化版思路 demo。

```csharp
public sealed class ProductCommandService
{
    private readonly ICacheManager<ProductDto> _cache;
    private readonly ProductRepository _repository;
    private readonly ICacheInvalidationBus _bus;

    public ProductCommandService(
        ICacheManager<ProductDto> cache,
        ProductRepository repository,
        ICacheInvalidationBus bus)
    {
        _cache = cache;
        _repository = repository;
        _bus = bus;
    }

    public void Update(ProductDto product)
    {
        _repository.Update(product);

        var key = $"product:{product.Id}";

        // 当前实例先删缓存
        _cache.Remove(key);

        // 再通知其他实例一起删
        _bus.Publish(key);
    }
}

public interface ICacheInvalidationBus
{
    void Publish(string key);
}
```

其他实例拿到通知后，做的事情通常也很简单：

```csharp
public sealed class CacheInvalidationHandler
{
    private readonly ICacheManager<ProductDto> _cache;

    public CacheInvalidationHandler(ICacheManager<ProductDto> cache)
    {
        _cache = cache;
    }

    public void Handle(string key)
    {
        _cache.Remove(key);
    }
}
```

这个 demo 不是为了直接拿去上线，而是为了说明多实例一致性的本质：

* 更新数据库只是第一步
* 删除当前实例缓存也只是第二步
* 真正关键的是把“失效”传播出去

### 多实例缓存一致性里，最容易误判的一件事

很多人看到 Redis 里的 key 已经删掉了，就会以为缓存问题已经解决。

但如果本地缓存还在，其他节点照样可能继续返回旧数据。

所以排查这类问题时，一定要把缓存层次分开看：

* Redis 是否已经更新或删除
* 当前实例本地缓存是否已经失效
* 其他实例本地缓存是否也收到通知

只盯 Redis 一层，往往会误判。

### Demo 5：缓存查询的 Cache-Aside 写法

项目里最常见的缓存模式，通常不是写穿，而是：

```text
Cache Aside
```

也就是：

* 先查缓存
* 没有就查数据库
* 查到后回写缓存

看一个完整一点的服务例子：

```csharp
using CacheManager.Core;
using System;
using System.Collections.Generic;

var cache = CacheFactory.Build<ProductDto>("productCache", settings =>
{
    settings.WithMicrosoftMemoryCacheHandle("memory");
});

var repository = new ProductRepository();
var service = new ProductQueryService(cache, repository);

var p1 = service.GetById(1);
var p2 = service.GetById(1);

Console.WriteLine(p1.Name);
Console.WriteLine(p2.Name);

public sealed class ProductQueryService
{
    private readonly ICacheManager<ProductDto> _cache;
    private readonly ProductRepository _repository;

    public ProductQueryService(
        ICacheManager<ProductDto> cache,
        ProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public ProductDto GetById(int id)
    {
        var key = $"product:{id}";

        var cached = _cache.Get(key);
        if (cached is not null)
        {
            Console.WriteLine("命中缓存");
            return cached;
        }

        Console.WriteLine("查询数据库");
        var product = _repository.GetById(id);

        if (product is not null)
        {
            _cache.Put(
                key,
                product,
                ExpirationMode.Absolute,
                TimeSpan.FromMinutes(10));
        }

        return product;
    }
}

public sealed class ProductRepository
{
    private readonly Dictionary<int, ProductDto> _data = new()
    {
        [1] = new ProductDto(1, "机械键盘", 399m)
    };

    public ProductDto GetById(int id) => _data.GetValueOrDefault(id);
}

public sealed record ProductDto(int Id, string Name, decimal Price);
```

这个 demo 有两个非常重要的现实含义：

* 第一请求查数据库并写缓存
* 后续请求直接吃缓存

这就是大部分业务系统里缓存真正发挥作用的方式。

### 更新数据时，缓存该怎么处理？

这也是缓存最容易出 bug 的地方。

查询容易，失效难。

最常见的处理方式一般是：

* 先更新数据库
* 再删除相关缓存
* 让下次读取自动回填

看一个简单写法：

```csharp
public sealed class ProductCommandService
{
    private readonly ICacheManager<ProductDto> _cache;
    private readonly ProductRepository _repository;

    public ProductCommandService(
        ICacheManager<ProductDto> cache,
        ProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public void Update(ProductDto product)
    {
        _repository.Update(product);
        _cache.Remove($"product:{product.Id}");
    }
}
```

为什么很多场景更推荐“更新数据库后删缓存”，而不是“更新数据库后立刻改缓存”？

因为删除缓存更简单，也更不容易把脏数据写进去。

尤其是：

* 一个对象可能对应多个 key
* 一个对象可能影响多个列表页
* 更新逻辑里可能还有其他副作用

这时候“先删，让读取时重建”通常更稳。

### `CacheManager` 适合哪些场景？

比较适合这些场景：

* 已经明确需要多级缓存
* 既有本地缓存，又有 Redis
* 想把缓存配置和策略集中管理
* 业务里有明显的缓存分组、统一失效需求

### 哪些场景反而没必要上它？

也要把边界说清楚。

下面这些场景，直接用更简单的方案往往更合适：

* 只有单机本地缓存需求
* 只是偶尔缓存几个配置项
* 项目本身缓存逻辑很少
* 团队已经有一套稳定的缓存封装

也就是说，`CacheManager` 不是“有缓存就上”的库。

如果只是：

```text
查一下配置
缓存 10 分钟
结束
```

那大概率 `IMemoryCache` 就够了。

### 使用时最容易踩的几个坑

#### 1. 把缓存当数据库

缓存是加速层，不是真实数据源。

凡是最终正确性依赖缓存的设计，都要格外小心。

#### 2. 只管加缓存，不管删缓存

很多系统的问题不是缓存命中率低，而是脏数据太久不失效。

#### 3. 多级缓存做了，但一致性没想清楚

本地缓存和 Redis 都有数据时，删除和更新策略一定要提前想清楚。

不然很容易出现：

* Redis 已经更新
* 某个实例的本地缓存还没失效
* 结果读到了旧值

#### 4. Key 设计混乱

建议从一开始就统一命名，例如：

```text
product:1
product:list:page:1
user:profile:1001
order:detail:9001
```

Key 一旦失控，后面删除和排查会越来越痛苦。

#### 5. 缓存值过大

大对象本身就会带来更多内存和序列化开销。

尤其是多级缓存里，还要考虑：

* 本地内存占用
* Redis 网络传输
* 序列化和反序列化成本

### 先记一个工程化原则

实际项目里，越到后面越会发现一件事：

> 最麻烦的往往不是“怎么把数据塞进缓存”，而是 key、TTL、失效规则到底由谁统一管理。

所以从一开始就更推荐：

* 按业务封装缓存访问
* 不要在业务代码里到处直接拼 key
* 不要把过期策略散落在控制器和服务里

后面会专门讲一段“统一缓存封装”该怎么做。

### 缓存穿透、缓存击穿、缓存雪崩，在 `CacheManager` 场景下怎么处理？

这是缓存专题里绕不开的三件事。

很多文章会把这几个词讲得很像面试题，但项目里真正重要的不是背定义，而是：

* 现象长什么样
* 为什么会把数据库打爆
* 用 `CacheManager` 这类统一缓存层时，应该怎么落地

先把最白话的区别说清楚：

* 缓存穿透：查一个根本不存在的数据，缓存永远没有，每次都打到数据库
* 缓存击穿：某个热点 key 刚好失效，大量请求同时回源数据库
* 缓存雪崩：大批 key 在同一时间失效，或者缓存层整体不可用，导致流量一起砸向下游

这三个问题看起来都像“缓存没命中”，但处理方式并不一样。

### 什么是缓存穿透？

缓存穿透最典型的场景是：

* 请求查的 key 根本不存在
* 缓存里没有
* 数据库里也没有
* 结果每次请求都会重复查数据库

例如：

```text
product:999999
```

这个商品压根不存在。

如果每次都这样走：

```text
查缓存
-> 没有
-> 查数据库
-> 数据库也没有
-> 直接返回空
```

那下一次再来，还是同样一套流程。

请求一多，数据库就会被这种“无效查询”白白消耗掉。

### `CacheManager` 场景下怎么处理缓存穿透？

最常见的办法是：

* 缓存空值
* 给空值一个较短过期时间

也就是说，查数据库发现不存在以后，不是直接什么都不做，而是把“查无此数据”这件事也缓存起来。

### Demo 6：用空值缓存挡住穿透

```csharp
using CacheManager.Core;
using System;
using System.Collections.Generic;

var cache = CacheFactory.Build<string>("productCache", settings =>
{
    settings.WithMicrosoftMemoryCacheHandle("memory");
});

var repository = new ProductRepository();
var service = new ProductQueryService(cache, repository);

Console.WriteLine(service.GetName(1));
Console.WriteLine(service.GetName(999));
Console.WriteLine(service.GetName(999));

public sealed class ProductQueryService
{
    private const string NullValue = "__NULL__";

    private readonly ICacheManager<string> _cache;
    private readonly ProductRepository _repository;

    public ProductQueryService(
        ICacheManager<string> cache,
        ProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public string GetName(int id)
    {
        var key = $"product:name:{id}";

        var cached = _cache.Get(key);
        if (cached is not null)
        {
            Console.WriteLine("命中缓存");
            return cached == NullValue ? null : cached;
        }

        Console.WriteLine("查询数据库");
        var name = _repository.GetName(id);

        if (name is null)
        {
            _cache.Put(
                key,
                NullValue,
                ExpirationMode.Absolute,
                TimeSpan.FromMinutes(1));

            return null;
        }

        _cache.Put(
            key,
            name,
            ExpirationMode.Absolute,
            TimeSpan.FromMinutes(10));

        return name;
    }
}

public sealed class ProductRepository
{
    private readonly Dictionary<int, string> _data = new()
    {
        [1] = "机械键盘"
    };

    public string GetName(int id) => _data.GetValueOrDefault(id);
}
```

这个写法的重点有两个：

* 不存在的数据也做一次短期缓存
* 空值缓存时间通常要比正常数据更短

为什么空值缓存时间不要太长？

因为：

* 数据今天不存在，不代表过一会儿还不存在
* 空值缓存太久，可能会把刚新增的数据挡住

### 什么是缓存击穿？

缓存击穿通常不是“很多 key 同时失效”，而是：

> 某一个特别热点的 key 失效了，瞬间大量请求一起回源。

比如首页正在疯狂访问商品 `product:1`。

它刚好过期的那一刻，几千个请求同时进来，都发现缓存没了，于是一起查数据库。

结果就是：

* 明明只是一个 key 过期
* 却把数据库瞬间压出尖峰

### `CacheManager` 场景下怎么处理缓存击穿？

最常见的做法通常是：

* 给热点 key 加单飞控制
* 让同一时刻只有一个请求去回源
* 其他请求等待结果或复用已回填的数据

也就是常说的：

```text
single flight
```

`CacheManager` 本身负责缓存管理，但“只允许一个请求回源”这件事，通常还是要在业务层自己做。

### Demo 7：用锁避免热点 key 击穿

下面给一个简化版思路。

```csharp
using CacheManager.Core;
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;

var cache = CacheFactory.Build<ProductDto>("productCache", settings =>
{
    settings.WithMicrosoftMemoryCacheHandle("memory");
});

var repository = new ProductRepository();
var service = new ProductQueryService(cache, repository);

var product = service.GetById(1);
Console.WriteLine(product.Name);

public sealed class ProductQueryService
{
    private static readonly ConcurrentDictionary<string, object> Locks = new();

    private readonly ICacheManager<ProductDto> _cache;
    private readonly ProductRepository _repository;

    public ProductQueryService(
        ICacheManager<ProductDto> cache,
        ProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public ProductDto GetById(int id)
    {
        var key = $"product:{id}";

        var cached = _cache.Get(key);
        if (cached is not null)
        {
            return cached;
        }

        var sync = Locks.GetOrAdd(key, _ => new object());

        lock (sync)
        {
            cached = _cache.Get(key);
            if (cached is not null)
            {
                return cached;
            }

            Console.WriteLine("回源数据库");
            var product = _repository.GetById(id);

            if (product is not null)
            {
                _cache.Put(
                    key,
                    product,
                    ExpirationMode.Absolute,
                    TimeSpan.FromMinutes(10));
            }

            return product;
        }
    }
}

public sealed class ProductRepository
{
    private readonly Dictionary<int, ProductDto> _data = new()
    {
        [1] = new ProductDto(1, "机械键盘", 399m)
    };

    public ProductDto GetById(int id) => _data.GetValueOrDefault(id);
}

public sealed record ProductDto(int Id, string Name, decimal Price);
```

这个写法最核心的点是“双重检查”：

* 第一次先查缓存
* 拿到锁以后再查一次缓存

因为很可能在等待锁的过程中，前一个请求已经把缓存填回去了。

### 什么是缓存雪崩？

缓存雪崩和击穿最容易混。

击穿通常是一个热点 key 的问题。

雪崩通常是：

* 一大批 key 同时过期
* 或者整个缓存层突然不可用
* 结果请求集体回源

最常见的诱因包括：

* 所有缓存都设成同一个过期时间
* 应用启动时同一批 key 一起预热
* Redis 故障
* 缓存节点网络抖动

### `CacheManager` 场景下怎么处理缓存雪崩？

工程上最常见的是几种手段一起上：

* 过期时间加随机抖动
* 本地缓存和 Redis 分层兜底
* 热点数据提前续期或异步预热
* 限流、降级、熔断，避免数据库被打穿

其中最容易落地的一件事是：

> 不要把一批缓存全设成同一个 TTL。

### Demo 8：给缓存 TTL 加随机抖动

```csharp
using CacheManager.Core;

var cache = CacheFactory.Build<ProductDto>("productCache", settings =>
{
    settings.WithMicrosoftMemoryCacheHandle("memory");
});

var product = new ProductDto(1, "机械键盘", 399m);

var random = Random.Shared.Next(30, 180);
var ttl = TimeSpan.FromMinutes(10).Add(TimeSpan.FromSeconds(random));

cache.Put(
    "product:1",
    product,
    ExpirationMode.Absolute,
    ttl);

public sealed record ProductDto(int Id, string Name, decimal Price);
```

这件事看起来很小，但很实用。

因为它能把原本集中在同一时刻失效的一批 key，打散到不同时间点。

### 多级缓存对雪崩有什么帮助？

如果只有 Redis 一层，Redis 一旦抖一下，所有请求都可能直接去打数据库。

如果有本地缓存层，哪怕 Redis 短时间波动，本地热点数据也还能顶一会儿。

也就是说，多级缓存除了加速，本身也能起到一部分缓冲作用。

但这里一定要注意：

* 本地缓存能缓冲，不代表能解决所有一致性问题
* Redis 挂了，本地缓存也只能兜住热点，不可能兜住全部数据

所以雪崩问题的本质，依然不是“缓存技术选型”本身，而是：

* TTL 策略是否合理
* 热点数据是否做了保护
* 下游数据库是否有降级手段

### 这三类问题最容易混淆的地方

可以直接用一句话区分：

* 穿透：查的是不存在的数据
* 击穿：炸的是单个热点 key
* 雪崩：炸的是一批 key，或者整层缓存

从 `CacheManager` 落地角度看，常见应对可以压缩成这样：

* 缓存穿透：空值缓存、短 TTL
* 缓存击穿：单飞控制、热点 key 加锁回填
* 缓存雪崩：TTL 打散、多级缓存、限流降级

也就是说，`CacheManager` 负责把缓存层组织起来，但真正决定系统能不能扛住高并发异常流量的，还是这些围绕它的策略设计。

### 缓存预热、热点数据续期、后台刷新，为什么经常一起出现？

走到这一步，缓存已经不只是“查了就存、改了就删”这么简单了。

很多线上系统真正难受的，不是平时缓存命不中，而是这些时间点：

* 应用刚启动，缓存全空
* 某批热点 key 集中过期
* 一个热点 key 反复被删除后重建
* 高峰期正好遇到缓存回源

这时候，常见优化动作通常就是三类：

* 缓存预热
* 热点数据续期
* 后台刷新

这三件事本质上都在做同一类工作：

> 尽量不要把“重新查数据库并回填缓存”这件事，留到真正的用户请求路径上。

### 什么是缓存预热？

缓存预热可以直接理解成：

> 在真实流量打进来之前，先把大概率会用到的数据放进缓存。

最常见的时机有这些：

* 应用启动后
* 定时任务执行时
* 大促、活动开始前
* 已知热点商品、热点配置、热点字典数据发布后

这样做的目的很直接：

* 避免应用启动后一波请求全去查数据库
* 避免热点数据第一次访问时产生明显延迟

### Demo 9：应用启动后做一轮缓存预热

下面给一个简化版思路。

```csharp
using CacheManager.Core;
using Microsoft.Extensions.Hosting;

public sealed class ProductCacheWarmupService : IHostedService
{
    private readonly ICacheManager<ProductDto> _cache;
    private readonly ProductRepository _repository;

    public ProductCacheWarmupService(
        ICacheManager<ProductDto> cache,
        ProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        var hotProducts = _repository.GetHotProducts();

        foreach (var product in hotProducts)
        {
            _cache.Put(
                $"product:{product.Id}",
                product,
                ExpirationMode.Absolute,
                TimeSpan.FromMinutes(10));
        }

        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
        => Task.CompletedTask;
}

public sealed record ProductDto(int Id, string Name, decimal Price);
```

如果放到 ASP.NET Core 里，通常会这样注册：

```csharp
builder.Services.AddHostedService<ProductCacheWarmupService>();
```

不过预热不是“数据越多越好”。

比较常见也更合理的做法是只预热：

* 热门商品
* 基础字典
* 首页高频配置
* 登录后必查的数据

如果上来就把全表都灌进缓存，通常只会把启动时间和内存占用一起搞难看。

### 什么是热点数据续期？

热点数据续期可以理解成：

> 某些 key 明明一直很热，那就别等它硬过期以后再让请求回源，而是在快过期时主动延长寿命。

这个思路特别适合：

* 首页推荐
* 爆款商品
* 高频配置项
* 热门用户画像

因为这类数据最大的特点通常是：

* 访问特别频繁
* 更新并不算特别频繁
* 一旦失效，回源代价很高

### `CacheManager` 里做热点续期，常见怎么落地？

最常见的方式不是让每个请求都无脑 `Put` 一次，而是：

* 查询时顺便判断是否快过期
* 快过期时，异步触发一次续期或刷新
* 用户请求先拿当前缓存值，不阻塞主链路

也就是说，重点不是“强同步更新”，而是：

> 在不拖慢请求的前提下，让热点 key 尽量别掉下来。

### Demo 10：热点 key 快过期时，触发续期

下面给一个思路型 demo。

```csharp
using CacheManager.Core;
using System.Collections.Concurrent;

public sealed class HotProductCacheService
{
    private static readonly ConcurrentDictionary<string, byte> RefreshingKeys = new();

    private readonly ICacheManager<CacheEnvelope<ProductDto>> _cache;
    private readonly ProductRepository _repository;

    public HotProductCacheService(
        ICacheManager<CacheEnvelope<ProductDto>> cache,
        ProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public ProductDto Get(int id)
    {
        var key = $"product:{id}";
        var envelope = _cache.Get(key);

        if (envelope is null)
        {
            return RefreshNow(id, key).Value;
        }

        if (envelope.ExpireAt <= DateTimeOffset.UtcNow.AddMinutes(1))
        {
            _ = TryRefreshInBackground(id, key);
        }

        return envelope.Value;
    }

    private CacheEnvelope<ProductDto> RefreshNow(int id, string key)
    {
        var product = _repository.GetById(id);
        var envelope = new CacheEnvelope<ProductDto>(
            product,
            DateTimeOffset.UtcNow.AddMinutes(10));

        _cache.Put(
            key,
            envelope,
            ExpirationMode.Absolute,
            TimeSpan.FromMinutes(10));

        return envelope;
    }

    private Task TryRefreshInBackground(int id, string key)
    {
        if (!RefreshingKeys.TryAdd(key, 0))
        {
            return Task.CompletedTask;
        }

        return Task.Run(() =>
        {
            try
            {
                RefreshNow(id, key);
            }
            finally
            {
                RefreshingKeys.TryRemove(key, out _);
            }
        });
    }
}

public sealed record CacheEnvelope<T>(T Value, DateTimeOffset ExpireAt);
public sealed record ProductDto(int Id, string Name, decimal Price);
```

这个 demo 的关键点有三个：

* 缓存值里额外记录一个业务层可见的过期时间
* 快过期时，不阻塞当前请求，改为后台续期
* 用一个字典防止同一个 key 被重复续期

这不是唯一写法，但思路很常见。

### 什么是后台刷新？

后台刷新和“热点续期”很像，但关注点稍微不同。

热点续期更像：

* 快过期了，延一下寿命

后台刷新更像：

* 定时把一批关键数据重新拉取并覆盖缓存

例如：

* 每 5 分钟刷新热门商品
* 每 10 分钟刷新汇率配置
* 每 1 分钟刷新首页推荐位

这类数据最大的特点通常是：

* 有明显的刷新周期
* 数据源改动不算实时强一致
* 能接受“稍晚一点更新，但别在请求链路里回源”

### Demo 11：定时后台刷新热点商品缓存

这个例子会更接近生产里的后台任务。

```csharp
using CacheManager.Core;
using Microsoft.Extensions.Hosting;

public sealed class ProductCacheRefreshService : BackgroundService
{
    private readonly ICacheManager<ProductDto> _cache;
    private readonly ProductRepository _repository;

    public ProductCacheRefreshService(
        ICacheManager<ProductDto> cache,
        ProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var hotProducts = _repository.GetHotProducts();

            foreach (var product in hotProducts)
            {
                _cache.Put(
                    $"product:{product.Id}",
                    product,
                    ExpirationMode.Absolute,
                    TimeSpan.FromMinutes(10));
            }

            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}

public sealed record ProductDto(int Id, string Name, decimal Price);
```

这类后台刷新的优点很明显：

* 请求链路更稳
* 热点数据更不容易掉空
* 数据库压力更可控

但也有边界：

* 刷新频率太高，会变成定时打数据库
* 刷新范围太大，会把缓存变成批处理任务
* 多实例下如果每个节点都在刷，可能还会造成重复工作

### 多实例下做预热和刷新，要注意什么？

这也是很容易被忽略的点。

如果有 10 个实例，每个实例启动时都去预热同一批热点数据，那很可能会发生：

* 缓存还没热起来
* 数据库先被 10 倍流量打了一轮

所以多实例场景下，常见做法通常是：

* 只让一个实例负责预热或刷新
* 或者把预热、刷新放到独立后台任务里
* 或者通过分布式锁控制同一时刻只有一个节点执行

也就是说，预热和后台刷新一旦进了集群环境，就不能再按单机思路直接复制。

### 这三种策略分别适合什么场景？

可以直接这样记：

* 缓存预热：适合应用启动、活动开始前、已知热点数据
* 热点数据续期：适合访问很频繁、回源代价高、但更新没那么频繁的数据
* 后台刷新：适合有固定刷新节奏、可以接受轻微延迟的数据

它们之间不是互斥关系，很多项目里反而会组合使用：

* 启动时先预热一批
* 运行中对超热点 key 做续期
* 再用后台任务定时刷新核心数据

### 一个很实用的判断标准

如果某条数据：

* 每次失效都很容易打爆数据库
* 大多数时候又并不要求秒级强一致

那就应该认真考虑：

* 能不能预热
* 能不能续期
* 能不能后台刷新

因为这类数据最怕的不是“缓存了会旧一点”，而是：

> 每次过期都让真实请求来承担重新构建缓存的成本。

### 项目里怎么做统一缓存封装？

文章前面已经多次提到一个观点：

> 不建议在业务代码里到处直接散着写 `cache.Get`、`cache.Put`、`cache.Remove`。

原因很现实：

* key 容易乱
* TTL 容易乱
* 失效策略容易乱
* 后面要调整缓存方案时，改动面会非常大

所以在真实项目里，更稳的做法通常不是“把 `CacheManager` 直接扔给所有业务代码”，而是：

* 统一封装缓存访问入口
* 统一 key 规则
* 统一过期策略
* 统一日志和异常处理

也就是说，业务层应该依赖的是：

* `IProductCache`
* `IUserProfileCache`
* `IOrderCache`

而不是直接依赖一堆裸的 `ICacheManager<T>`。

### Demo 12：一个比较实用的统一缓存封装

下面给一个偏实战的骨架写法。

```csharp
using CacheManager.Core;

public interface IProductCache
{
    ProductDto GetDetail(int id);
    void SetDetail(ProductDto product);
    void RemoveDetail(int id);
    void ClearListRegion();
}

public sealed class ProductCache : IProductCache
{
    private static readonly TimeSpan DetailTtl = TimeSpan.FromMinutes(10);

    private readonly ICacheManager<ProductDto> _detailCache;
    private readonly ICacheManager<string> _listCache;

    public ProductCache(
        ICacheManager<ProductDto> detailCache,
        ICacheManager<string> listCache)
    {
        _detailCache = detailCache;
        _listCache = listCache;
    }

    public ProductDto GetDetail(int id)
        => _detailCache.Get(GetDetailKey(id));

    public void SetDetail(ProductDto product)
    {
        _detailCache.Put(
            GetDetailKey(product.Id),
            product,
            ExpirationMode.Absolute,
            DetailTtl);
    }

    public void RemoveDetail(int id)
        => _detailCache.Remove(GetDetailKey(id));

    public void ClearListRegion()
        => _listCache.ClearRegion(ProductListRegion);

    private static string GetDetailKey(int id) => $"product:detail:{id}";

    private const string ProductListRegion = "product-list";
}

public sealed record ProductDto(int Id, string Name, decimal Price);
```

这个封装的重点其实不是代码量，而是边界清晰：

* key 规则被收口了
* TTL 被收口了
* 列表区域失效策略也被收口了

后面如果要改成：

* 详情 15 分钟过期
* 列表改成另一个 `Region`
* 详情缓存写入时多打一条日志

都只需要动这一层。

### 再往前走一步：把 Cache-Aside 也一起封起来

很多项目里更推荐的做法，是连“查不到就回源并回填”这套逻辑也一起封进去。

因为这部分逻辑如果散在业务层，也很容易到处复制粘贴。

看一个例子：

```csharp
using CacheManager.Core;

public interface IProductCacheFacade
{
    ProductDto GetOrLoad(int id);
    void Invalidate(int id);
}

public sealed class ProductCacheFacade : IProductCacheFacade
{
    private readonly IProductCache _cache;
    private readonly ProductRepository _repository;

    public ProductCacheFacade(
        IProductCache cache,
        ProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public ProductDto GetOrLoad(int id)
    {
        var cached = _cache.GetDetail(id);
        if (cached is not null)
        {
            return cached;
        }

        var product = _repository.GetById(id);
        if (product is not null)
        {
            _cache.SetDetail(product);
        }

        return product;
    }

    public void Invalidate(int id)
    {
        _cache.RemoveDetail(id);
        _cache.ClearListRegion();
    }
}
```

这样做的好处更直接：

* 查询缓存的套路统一了
* 失效逻辑统一了
* 业务代码会更像“调用领域服务”，而不是在拼缓存脚本

### `CacheManager`、`IMemoryCache`、Redis、`HybridCache` 怎么选？

文章写到这里，真正该落下来的问题已经不是“会不会用 `CacheManager`”，而是：

> 这个项目到底该不该用它？

这个问题没有一个放之四海而皆准的答案，但工程上可以按下面这个思路判断。

### 1. 只有单机本地缓存需求

比如：

* 缓存一些配置
* 缓存少量热点查询结果
* 项目就是单实例，或者本地缓存只做轻量加速

这种场景下，通常直接用：

```csharp
IMemoryCache
```

就够了。

原因也很简单：

* 依赖少
* 学习成本低
* 代码更直接

如果这时候硬上 `CacheManager`，很多时候只是把简单问题做复杂。

### 2. 多实例共享缓存，但不需要多级封装

比如：

* 主要依赖 Redis
* 本地缓存并不是核心诉求
* 只是想让多个节点共享数据

这种场景下，直接用 Redis 封装往往就够了。

也可以是：

* `IDistributedCache`
* 自己基于 `StackExchange.Redis` 做轻量封装

如果项目对缓存层管理要求不高，直接用这些方案会更轻。

### 3. 需要统一 API 和多级缓存管理

如果项目已经进入这种状态：

* 既有本地缓存，又有 Redis
* 需要统一 key、TTL、失效策略
* 需要 `Region`
* 需要把多级缓存方案集中管理

那 `CacheManager` 的价值就出来了。

它更适合：

* 现有项目缓存层已经变复杂
* 需要收敛多种缓存访问方式
* 希望缓存策略配置更集中

### 4. 新项目，而且已经在较新的 .NET 生态里

这时候就不能不提：

```csharp
HybridCache
```

如果是较新的 `.NET` 项目，并且更希望走官方生态路线，那么 `HybridCache` 会是一个很值得认真评估的方向。

原因通常有这些：

* 官方体系内能力
* 更贴近现代 ASP.NET Core 依赖注入和配置方式
* 对新项目更自然

但这不代表 `CacheManager` 没价值。

更准确的理解应该是：

* 老项目、已有 `CacheManager` 体系：继续用并不奇怪
* 新项目、希望更贴近官方路线：优先评估 `HybridCache`

### 可以直接这样做选型判断

如果要快速落一个判断，通常可以按这个表来看：

| 方案 | 更适合什么场景 |
| --- | --- |
| `IMemoryCache` | 单机、本地、轻量缓存 |
| Redis / `IDistributedCache` | 多实例共享缓存，但策略不复杂 |
| `CacheManager` | 多级缓存、统一封装、策略集中管理 |
| `HybridCache` | 新项目、偏官方路线、希望和现代 .NET 生态更一致 |

### 总结

`CacheManager` 的价值，从来不只是“再包一层缓存 API”。

它真正适合的，是这类问题：

* 缓存后端不止一个
* 想做本地内存加分布式缓存
* 想把缓存策略、分组、失效逻辑收敛起来

如果项目的缓存需求已经开始从“偶尔加速一下”变成“系统层能力”，那 `CacheManager` 的思路就很有参考价值。

但真正决定这套方案能不能长期稳住的，通常不是某个 API 名字，而是这些基础问题有没有先想清楚：

* key 怎么命名
* TTL 怎么分层
* 更新后删哪些缓存
* 多实例下怎么同步失效
* 热点数据怎么预热、续期、刷新

如果只记一句话，最值得记的是：

> `CacheManager` 最擅长的，不是替代某一个缓存组件，而是把多级缓存、统一 API 和缓存管理策略整合成一套更容易维护的方案。
