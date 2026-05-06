### 简介

很多项目一开始做缓存，通常都是这么写的：

```text
先查 IMemoryCache
-> 没有再查 Redis
-> 还没有就查数据库
-> 再把结果写回两层缓存
```

刚开始看起来没什么问题。

但只要项目一复杂，这套逻辑很快就会变得又长又散：

* 每个地方都在手写两级缓存
* 本地缓存和 Redis 的过期时间不好统一
* 并发一上来，容易缓存击穿
* 对象序列化、反序列化到处都是
* 删除缓存时，还得考虑本地层和分布式层一起失效

这时候，`.NET` 官方给了一个更现代的选择：

```csharp
HybridCache
```

一句话先说透：

> `HybridCache` 是 `.NET` 官方提供的混合缓存库，把进程内缓存和分布式缓存整合成一套统一 API，同时内置了缓存旁路、防击穿和序列化能力。

这篇文章重点讲这些事：

* `HybridCache` 到底解决了什么问题
* 它和 `IMemoryCache`、`IDistributedCache` 有什么关系
* 为什么很多人会把它看成官方版“两级缓存”
* `GetOrCreateAsync` 到底好用在哪
* 标签失效、删除缓存、多实例场景应该怎么理解
* 用几个更像真实项目的 demo，把常见写法串起来

### `HybridCache` 到底是什么？

可以先用最白话的方式理解：

> `HybridCache` 就是官方把“本地缓存 + 分布式缓存 + 回源逻辑”这套常见套路，统一收成了一层能力。

它最常见的结构通常是：

* `L1`：本地内存缓存
* `L2`：分布式缓存，例如 Redis

读取时的思路通常是：

```text
先查 L1
-> L1 没有再查 L2
-> L2 也没有才执行回源函数
-> 再把结果写回 L1 和 L2
```

这也是为什么很多人会把它看成：

> 官方版两级缓存方案。

### 为什么会有 `HybridCache`？

因为老写法虽然能用，但现实里会反复踩同一批坑。

最典型的几个痛点有这些：

#### 1. 缓存旁路逻辑重复

每个服务都在写：

* 先查缓存
* 没有就查数据库
* 再回填缓存

代码很容易一份份复制出去。

#### 2. 本地缓存和分布式缓存不好统一

有的地方只查内存，有的地方只查 Redis，有的地方先查内存再查 Redis。

时间一长，缓存访问方式很容易失控。

#### 3. 高并发下容易击穿

某个热点 key 一旦过期，大量请求可能会一起回源数据库。

#### 4. 序列化逻辑散落

手动用 `IDistributedCache` 时，很多项目都在自己写：

* JSON 序列化
* 反序列化
* 默认 TTL
* 扩展方法

这些东西本来就不该在业务层到处重复。

### `HybridCache` 和 `IMemoryCache`、`IDistributedCache` 到底是什么关系？

先把关系讲清楚，后面就不容易混。

#### `IMemoryCache`

* 本地内存缓存
* 只在当前进程内生效
* 速度极快
* 多实例之间不共享

#### `IDistributedCache`

* 分布式缓存抽象
* 通常接 Redis、SQL Server 这类外部缓存
* 多实例可共享
* 需要处理序列化

#### `HybridCache`

* 上层统一缓存库
* 默认用 `MemoryCache` 做本地层
* 如果项目里配置了 `IDistributedCache`，它就会把那一层作为分布式层

可以直接这样记：

> `HybridCache` 不是替代 `IMemoryCache` 或 `IDistributedCache` 的底层存储，而是把它们组织成一套更顺手的高层缓存方案。

### 它是不是只有接了 Redis 才有价值？

不是。

这是一个很容易误解的点。

即便没有配置分布式缓存，`HybridCache` 仍然可以工作：

* 依然有本地缓存
* 依然有 `GetOrCreateAsync`
* 依然有缓存击穿保护

只不过这时候它更像：

* 更高级的本地缓存访问层

如果项目里同时配置了 Redis 这类 `IDistributedCache` 实现，它才会自然升级成：

* 本地 `L1`
* 分布式 `L2`

### 先看最基础的注册方式

先装包：

```bash
dotnet add package Microsoft.Extensions.Caching.Hybrid
```

在 ASP.NET Core 里注册：

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHybridCache();

var app = builder.Build();
app.Run();
```

这就是最小接入方式。

如果项目里什么都不额外配，它会先用本地内存能力工作起来。

### 如果要接 Redis，怎么配？

先装 Redis 包：

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

然后这样注册：

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
});

builder.Services.AddHybridCache();

var app = builder.Build();
app.Run();
```

这一点非常关键：

> `HybridCache` 自己不是 Redis 客户端，它会自动使用项目里已经注册好的 `IDistributedCache` 作为二级缓存。

也就是说，Redis 不是单独写进 `HybridCache` 配置里的，而是通过现有 `IDistributedCache` 接入。

### `HybridCache` 最核心的 API 是什么？

最核心的通常就是一个：

```csharp
GetOrCreateAsync
```

这个方法几乎把最常见的缓存旁路场景全包了。

它做的事情可以概括成：

* 先按 key 查缓存
* 如果命中，直接返回
* 如果没命中，执行工厂函数
* 再把结果写回缓存

### Demo 1：最小可用的 `GetOrCreateAsync`

```csharp
using Microsoft.Extensions.Caching.Hybrid;

public sealed class UserService
{
    private readonly HybridCache _cache;

    public UserService(HybridCache cache)
    {
        _cache = cache;
    }

    public async Task<UserDto> GetUserAsync(int id, CancellationToken cancellationToken = default)
    {
        return await _cache.GetOrCreateAsync(
            $"user:{id}",
            async cancel =>
            {
                await Task.Delay(100, cancel);
                return new UserDto(id, "Tom");
            },
            cancellationToken: cancellationToken);
    }
}

public sealed record UserDto(int Id, string Name);
```

这个例子虽然小，但已经把它和传统缓存写法的差别拉开了：

* 没有手动 `Get`
* 没有手动判断缓存命中
* 没有手动写回缓存

很多场景里，直接一个 `GetOrCreateAsync` 就够了。

### 它为什么能防缓存击穿？

这是 `HybridCache` 最有价值的点之一。

很多缓存问题，本质不是“缓存不会用”，而是：

> 热点 key 一过期，大量请求一起去回源。

`HybridCache` 内置了同 key 并发合并能力。

可以先这样理解：

* 同一个 key 同时来了很多请求
* 第一个请求负责真正执行回源函数
* 其他请求等待这个结果

也就是说：

> 对同一个 key，不会让所有并发请求都一起打数据库。

这就是很多资料里常说的：

```text
stampede protection
```

或者：

```text
single flight
```

### Demo 2：商品详情缓存的标准写法

```csharp
using Microsoft.Extensions.Caching.Hybrid;

public sealed class ProductQueryService
{
    private readonly HybridCache _cache;
    private readonly ProductRepository _repository;

    public ProductQueryService(
        HybridCache cache,
        ProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public async Task<ProductDto?> GetByIdAsync(int id, CancellationToken cancellationToken = default)
    {
        return await _cache.GetOrCreateAsync(
            $"product:{id}",
            async cancel => await _repository.GetByIdAsync(id, cancel),
            cancellationToken: cancellationToken);
    }
}

public sealed class ProductRepository
{
    public Task<ProductDto?> GetByIdAsync(int id, CancellationToken cancellationToken = default)
    {
        ProductDto? product = id == 1
            ? new ProductDto(1, "机械键盘", 399m)
            : null;

        return Task.FromResult(product);
    }
}

public sealed record ProductDto(int Id, string Name, decimal Price);
```

这个写法已经非常接近真实项目里的缓存查询服务了。

### 过期策略怎么配？

`HybridCache` 里最常见的过期配置在：

```csharp
HybridCacheEntryOptions
```

最值得先记住的是两个时间：

* `Expiration`
* `LocalCacheExpiration`

可以先用最白话的方式理解：

* `Expiration`：整体缓存项的有效时间，更偏向分布式层
* `LocalCacheExpiration`：本地缓存层的有效时间

### Demo 3：本地层短一点，分布式层长一点

```csharp
using Microsoft.Extensions.Caching.Hybrid;

public sealed class ProductService
{
    private readonly HybridCache _cache;

    public ProductService(HybridCache cache)
    {
        _cache = cache;
    }

    public async Task<ProductDto> GetAsync(int id, CancellationToken cancellationToken = default)
    {
        var options = new HybridCacheEntryOptions
        {
            Expiration = TimeSpan.FromMinutes(30),
            LocalCacheExpiration = TimeSpan.FromMinutes(5)
        };

        return await _cache.GetOrCreateAsync(
            $"product:{id}",
            async cancel =>
            {
                await Task.Delay(50, cancel);
                return new ProductDto(id, "机械键盘", 399m);
            },
            options,
            cancellationToken: cancellationToken);
    }
}

public sealed record ProductDto(int Id, string Name, decimal Price);
```

这种配置思路非常常见：

* 本地层更短
* 分布式层更长

为什么经常这么做？

因为这样通常能兼顾两件事：

* 本地读性能好
* 多实例下旧数据停留时间不会太长

### 全局默认配置怎么配？

如果很多缓存项都差不多，可以在注册时直接配全局默认值。

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHybridCache(options =>
{
    options.MaximumPayloadBytes = 1024 * 1024;
    options.MaximumKeyLength = 1024;
    options.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(10),
        LocalCacheExpiration = TimeSpan.FromMinutes(5)
    };
});
```

这里几个点比较实用：

* `MaximumPayloadBytes`：限制单个缓存项大小
* `MaximumKeyLength`：限制 key 最大长度
* `DefaultEntryOptions`：全局默认过期策略

这类默认配置的价值在于：

* 避免每个地方都手写 TTL
* 降低超大缓存项把系统拖慢的风险

### 它默认怎么序列化对象？

如果项目里配置了分布式缓存层，那么对象写到二级缓存时就需要序列化。

`HybridCache` 默认的处理方式可以先记成这样：

* `string` 和 `byte[]` 会被特殊处理
* 其他对象默认走 `System.Text.Json`

这就意味着，大多数普通对象场景下：

* 不需要再像 `IDistributedCache` 那样手写一层 JSON 扩展方法

这是它非常省事的一点。

### 如果默认序列化不合适怎么办？

也可以配置自定义序列化器。

官方支持从 `AddHybridCache()` 链式配置：

* `AddSerializer`
* `AddSerializerFactory`

也就是说，如果项目有更明确的性能目标，比如：

* 想换成 protobuf
* 想对某个类型做更细粒度的序列化控制

`HybridCache` 也不是只能死用默认 JSON。

### `SetAsync` 和 `GetOrCreateAsync` 怎么选？

虽然 `GetOrCreateAsync` 很强，但不是所有场景都适合它。

如果已经有明确数据，想直接写缓存，可以用：

```csharp
SetAsync
```

例如：

* 后台预热缓存
* 主动刷新热点数据
* 某些事件驱动场景下直接覆盖缓存

### Demo 4：主动写入缓存

```csharp
using Microsoft.Extensions.Caching.Hybrid;

public sealed class ProductCacheWriter
{
    private readonly HybridCache _cache;

    public ProductCacheWriter(HybridCache cache)
    {
        _cache = cache;
    }

    public async Task WarmupAsync(ProductDto product, CancellationToken cancellationToken = default)
    {
        await _cache.SetAsync(
            $"product:{product.Id}",
            product,
            options: new HybridCacheEntryOptions
            {
                Expiration = TimeSpan.FromMinutes(30),
                LocalCacheExpiration = TimeSpan.FromMinutes(5)
            },
            cancellationToken: cancellationToken);
    }
}

public sealed record ProductDto(int Id, string Name, decimal Price);
```

### 更新数据时，缓存该怎么失效？

这也是缓存里最容易出问题的部分。

最常见、也最稳的一种思路通常还是：

* 先更新数据库
* 再删除相关缓存
* 让下一次查询自动重建

### Demo 5：更新后删除缓存

```csharp
using Microsoft.Extensions.Caching.Hybrid;

public sealed class ProductCommandService
{
    private readonly HybridCache _cache;
    private readonly ProductRepository _repository;

    public ProductCommandService(
        HybridCache cache,
        ProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public async Task UpdateAsync(ProductDto product, CancellationToken cancellationToken = default)
    {
        await _repository.UpdateAsync(product, cancellationToken);
        await _cache.RemoveAsync($"product:{product.Id}", cancellationToken);
    }
}
```

这个思路为什么常见？

因为它简单，而且通常更不容易把脏数据写回缓存。

### 标签失效是什么？为什么它很适合批量清理？

这是 `HybridCache` 相比很多旧缓存写法很有吸引力的一点。

可以先把标签理解成：

> 给一批缓存项打上逻辑分组。

例如：

* 商品详情都打上 `product`
* 某个分类商品再打上 `category:keyboard`

这样当分类更新时，就不需要手动删一堆 key，而是可以按标签失效。

### Demo 6：写缓存时带标签

```csharp
using Microsoft.Extensions.Caching.Hybrid;

public sealed class ProductQueryService
{
    private readonly HybridCache _cache;

    public ProductQueryService(HybridCache cache)
    {
        _cache = cache;
    }

    public async Task<ProductDto> GetAsync(int id, CancellationToken cancellationToken = default)
    {
        var options = new HybridCacheEntryOptions
        {
            Expiration = TimeSpan.FromMinutes(20),
            LocalCacheExpiration = TimeSpan.FromMinutes(5)
        };

        var tags = new[] { "product", $"product:{id}", "category:keyboard" };

        return await _cache.GetOrCreateAsync(
            $"product:{id}",
            async cancel =>
            {
                await Task.Delay(50, cancel);
                return new ProductDto(id, "机械键盘", 399m);
            },
            options,
            tags,
            cancellationToken: cancellationToken);
    }
}

public sealed record ProductDto(int Id, string Name, decimal Price);
```

### 为什么标签要写成数组？第一个 `product` 又是什么意思？

这里的：

```csharp
var tags = new[] { "product", $"product:{id}", "category:keyboard" };
```

本质上是在给同一条缓存打上多个“逻辑分组标签”。

先抓住一个核心点：

> 标签不是 key，本质上是为了后续按不同维度做失效控制。

之所以写成数组，是因为一条缓存通常不只属于一个维度。

比如一个商品详情缓存，同时可能属于：

* 商品模块
* 某一个具体商品
* 某一个商品分类

所以会需要一次挂多个标签，而不是只挂一个。

可以把这三个标签拆开看：

* `"product"`：大类标签
* `$"product:{id}"`：单对象标签
* `"category:keyboard"`：业务分类标签

#### 1. `"product"` 是大类标签

这个标签最适合表达：

* 这条缓存属于商品模块

它的价值在于后面可以直接按模块整体失效：

```csharp
await _cache.RemoveByTagAsync("product", cancellationToken);
```

这句话的效果可以理解成：

* 让所有打过 `product` 标签的缓存都失效

所以第一个 `product` 不是特殊关键字，只是一个普通标签，只不过通常会把它当成“商品模块总标签”来用。

#### 2. `$"product:{id}"` 是单对象标签

如果 `id = 12`，那它就会变成：

```text
product:12
```

它的意义更像：

* 这条缓存属于“商品 12”这个对象维度

这样以后如果要按“某一个对象”维度做逻辑失效，就会更精确。

#### 3. `"category:keyboard"` 是业务维度标签

这个标签最适合表达：

* 这条商品缓存同时属于“键盘分类”

这样当分类维度发生变化时，就可以直接按分类清：

```csharp
await _cache.RemoveByTagAsync("category:keyboard", cancellationToken);
```

而不需要自己去维护这一分类下所有商品 key 的清单。

### 标签设计的思路，通常可以分成这三层

比较常见的设计方式就是：

* 模块级标签：例如 `product`
* 对象级标签：例如 `product:12`
* 业务级标签：例如 `category:keyboard`

这样做的好处是很直接的：

* 想清整个模块，可以按模块标签
* 想清某个对象，可以按对象标签
* 想清某个业务范围，可以按业务标签

也就是说，标签最值钱的地方，不是“多挂几个字符串”，而是：

> 先把后面的失效维度设计出来。

### 标签不要乱挂，维度要能落到真实失效场景

标签并不是越多越好。

如果一条缓存随手挂一堆没有明确意义的标签，后面维护反而会乱。

更稳的做法通常是先问自己：

* 以后可能按什么维度清缓存
* 是按模块清，按对象清，还是按分类清
* 哪些维度是真实会发生的失效场景

想清这几个问题后，再决定挂哪些标签，设计会稳很多。

### Demo 7：按标签失效

```csharp
using Microsoft.Extensions.Caching.Hybrid;

public sealed class CategoryCommandService
{
    private readonly HybridCache _cache;

    public CategoryCommandService(HybridCache cache)
    {
        _cache = cache;
    }

    public async Task UpdateCategoryAsync(CancellationToken cancellationToken = default)
    {
        await _cache.RemoveByTagAsync("category:keyboard", cancellationToken);
    }
}
```

这里有个非常关键的技术事实一定要说清楚：

> `RemoveByTagAsync` 是逻辑失效，不是立刻去把所有底层缓存值物理删掉。

更准确地说：

* 带这些标签的数据，后续会被当成缓存未命中处理
* 但底层已有值通常还是按自己的生命周期自然过期

这个差别很重要。

### 还有一个多实例下非常容易忽略的点

按 key 删除和按标签失效时，当前节点和二级分布式缓存会被处理。

但其他服务器进程里的本地内存层，并不会被主动扫一遍物理清掉。

也就是说，多实例场景下仍然要理解一个现实：

* 本地层本身就是每个节点各有一份

这也是为什么前面经常会建议：

* `LocalCacheExpiration` 不要太长

### Demo 8：更接近真实项目的商品服务

```csharp
using Microsoft.Extensions.Caching.Hybrid;

public sealed class ProductService
{
    private readonly HybridCache _cache;
    private readonly ProductRepository _repository;

    public ProductService(
        HybridCache cache,
        ProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public async Task<ProductDto?> GetByIdAsync(int id, CancellationToken cancellationToken = default)
    {
        return await _cache.GetOrCreateAsync(
            $"product:{id}",
            async cancel => await _repository.GetByIdAsync(id, cancel),
            options: new HybridCacheEntryOptions
            {
                Expiration = TimeSpan.FromMinutes(30),
                LocalCacheExpiration = TimeSpan.FromMinutes(5)
            },
            tags: ["product", $"product:{id}"],
            cancellationToken: cancellationToken);
    }

    public async Task UpdateAsync(ProductDto product, CancellationToken cancellationToken = default)
    {
        await _repository.UpdateAsync(product, cancellationToken);
        await _cache.RemoveAsync($"product:{product.Id}", cancellationToken);
        await _cache.RemoveByTagAsync("product", cancellationToken);
    }
}

public sealed class ProductRepository
{
    public Task<ProductDto?> GetByIdAsync(int id, CancellationToken cancellationToken = default)
    {
        ProductDto? product = id == 1
            ? new ProductDto(1, "机械键盘", 399m)
            : null;

        return Task.FromResult(product);
    }

    public Task UpdateAsync(ProductDto product, CancellationToken cancellationToken = default)
        => Task.CompletedTask;
}

public sealed record ProductDto(int Id, string Name, decimal Price);
```

这个例子里，几乎已经能看到 `HybridCache` 在真实业务里的完整轮廓了：

* 查询走 `GetOrCreateAsync`
* 更新后删 key
* 需要时按标签做一批逻辑失效

### `HybridCache`、`IDistributedCache`、`CacheManager` 怎么选？

真正做选型时，先别急着问哪个更火。

更该先问的是：

* 有没有本地 `L1` 加分布式 `L2` 的需求
* 是不是多实例部署
* 缓存旁路逻辑是不是已经写得到处都是
* 热点 key 并发回源是不是已经成了现实问题
* 项目更偏官方路线，还是历史缓存体系已经很重

可以直接按下面这张表看：

| 方案 | 更适合什么场景 |
| --- | --- |
| `IMemoryCache` | 单机、本地、轻量缓存 |
| `IDistributedCache` | 多实例共享缓存，但不强调两级缓存封装 |
| `HybridCache` | 官方两级缓存、旁路逻辑统一、防击穿诉求明确 |
| `CacheManager` | 历史项目、缓存层复杂、策略集中管理需求更重 |

可以直接把边界记成这样：

* 只有本地缓存需求：优先 `IMemoryCache`
* 只需要跨节点共享缓存：优先 `IDistributedCache`
* 已经明确需要两级缓存，又不想再手写样板代码：优先评估 `HybridCache`
* 历史项目缓存层已经很重，还需要更强的集中管理能力：再看 `CacheManager`

如果项目里已经开始频繁写这种逻辑：

```text
先查内存
-> 没有再查 Redis
-> 没有再查数据库
-> 再写回两层缓存
-> 还要自己想怎么防击穿
```

那真正该考虑的就不再只是“Redis 怎么配”，而是：

> 这套两级缓存样板代码，值不值得交给 `HybridCache` 这种更高层的官方能力来接管。

### 总结

`HybridCache` 的价值，不只是“把缓存 API 包得更短”。

它真正值钱的地方通常是：

* 官方两级缓存方案
* 内置缓存旁路
* 内置并发防击穿
* 默认序列化和更顺手的对象缓存体验
* 更适合现代 ASP.NET Core 项目的缓存写法

如果项目已经开始频繁手写“本地缓存 + Redis + 回源 + 防击穿”这套逻辑，那 `HybridCache` 基本就是一个很自然的升级方向。

如果只记一句话，最值得记的是：

> `HybridCache` 最大的意义，不是多一个缓存库，而是把两级缓存、回源逻辑和并发保护收成了一套官方统一能力。
