### 简介

在 `.NET` 里做性能优化时，很多人第一反应是：

* 少分配
* 少 `GC`
* 少临时对象

这个方向本身没有问题。

但问题在于，优化一旦开始，很容易走偏成另外一种极端：

* 看到对象创建就想池化
* 觉得池化一定比 `new` 更快
* 把对象池当成一种“万能缓存”

`ObjectPool` 这类工具，真正值钱的地方不是“把所有对象都放进池里”，而是：

> 对某些创建成本不算低、使用频率高、生命周期短、又能安全复用的对象，减少重复分配和回收的成本。

这篇文章重点讲清楚几件事：

* `ObjectPool` 到底是什么；
* 为什么会有它；
* `.NET` 里的官方实现怎么用；
* 怎么安装；
* 一个最小 demo 怎么跑；
* 自定义池化对象怎么写；
* 它适合什么场景，不适合什么场景。

一句话先给结论：

> `ObjectPool` 不是为了替代正常对象创建，而是为了在特定热点路径里复用“可重置的短期对象”。

### `ObjectPool` 到底是什么？

它位于：

```csharp
Microsoft.Extensions.ObjectPool
```

核心抽象很简单：

```csharp
ObjectPool<T>
```

最重要的两个动作也很简单：

* `Get()`：从池里取一个对象
* `Return(obj)`：把对象还回池里

它的思路不是：

* 预先把所有对象都准备好
* 永远保证对象不创建

而更像：

* 能复用就复用
* 复用不了就新建
* 还回来时如果池里放不下，也可以直接丢掉

所以一开始就要建立一个正确心智模型：

> 对象池是“尽量复用”，不是“绝不分配”。

### 为什么会有它？

先看一个最典型的例子：

```csharp
for (var i = 0; i < 100_000; i++)
{
    var sb = new StringBuilder();
    sb.Append("hello");
    _ = sb.ToString();
}
```

这段代码逻辑上没问题，但如果它出现在高频路径里，就会带来很直接的成本：

* 持续分配对象
* 增加 `GC` 压力
* 热点路径上吞吐变差

如果这个对象满足几件事：

* 创建本身有一定成本
* 用完之后能清理干净
* 不会被跨线程乱共享
* 高频重复出现

那就有池化的价值。

所以 `ObjectPool` 解决的不是“对象太多”这么宽泛的问题，而是更具体的这一类问题：

* 某些对象很适合复用
* 反复创建和回收它们不划算

### 它和缓存、连接池、数组池是什么关系？

这几个概念经常被混在一起，但其实不是一回事。

#### 1. 和缓存不一样

缓存关心的是：

* 数据值能不能复用
* 下次还能不能直接命中

对象池关心的是：

* 这个对象实例能不能再利用

也就是说，缓存复用的是“结果”，对象池复用的是“壳”。

#### 2. 和数据库连接池不一样

连接池里的资源通常更重，而且带有外部系统状态。

对象池更常见的目标通常是：

* `StringBuilder`
* 序列化缓冲对象
* 临时 parser
* 可重置的上下文对象

#### 3. 和 `ArrayPool<T>` 不一样

`ArrayPool<T>` 复用的是数组缓冲区。

`ObjectPool<T>` 复用的是一个完整对象。

两者都属于“减少分配”的工具，但粒度不同。

### 安装

官方包是：

```bash
dotnet add package Microsoft.Extensions.ObjectPool
```

如果你习惯看 `csproj`，对应就是：

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.ObjectPool" Version="*" />
</ItemGroup>
```

常用命名空间：

```csharp
using Microsoft.Extensions.ObjectPool;
```

### 怎么自己建一个最小 demo 跑起来？

先建一个控制台项目：

```bash
dotnet new console -n ObjectPoolDemo
cd ObjectPoolDemo
dotnet add package Microsoft.Extensions.ObjectPool
```

然后把 `Program.cs` 改成下面这样：

```csharp
using Microsoft.Extensions.ObjectPool;
using System.Text;

var provider = new DefaultObjectPoolProvider();
var pool = provider.CreateStringBuilderPool();

var sb = pool.Get();

try
{
    sb.Append("Hello ");
    sb.Append("ObjectPool");

    Console.WriteLine(sb.ToString());
}
finally
{
    pool.Return(sb);
}
```

最后执行：

```bash
dotnet run
```

如果正常输出：

```text
Hello ObjectPool
```

那就说明这条最小链路已经通了。

这个最小 demo 里最重要的不是 API 名字，而是这个使用姿势：

* 先 `Get()`
* 用完以后一定 `Return()`
* 最稳妥的写法是放在 `try/finally`

### `ObjectPool` 的核心工作方式是什么？

不用先看源码，先抓住主线就够了。

你可以把它粗略理解成这样：

```text
Get:
  池里有空闲对象 -> 直接拿
  池里没有       -> 新建一个

Return:
  对象可复用 + 池里还有空间 -> 放回去
  否则                      -> 直接丢弃
```

这意味着一个很容易被忽略的事实：

> 池的大小限制的是“最多保留多少个可复用对象”，不是“程序一共最多只能创建多少个对象”。

如果并发一高，池子里不够用，`ObjectPool` 仍然会创建新对象。

所以它不是限流器，也不是容量控制器。

### `DefaultObjectPoolProvider`、`ObjectPool<T>`、`PooledObjectPolicy<T>` 分别是什么？

这几个类型建议一起理解。

#### 1. `ObjectPool<T>`

这是抽象池本身。

它定义的就是：

* `Get()`
* `Return()`

#### 2. `DefaultObjectPoolProvider`

这是默认的池工厂。

你通常不会每次都直接 new 某个内部池实现，而是用它来创建池：

```csharp
var provider = new DefaultObjectPoolProvider();
var pool = provider.CreateStringBuilderPool();
```

或者：

```csharp
var provider = new DefaultObjectPoolProvider();
var pool = provider.Create(new MyBufferPolicy());
```

#### 3. `PooledObjectPolicy<T>`

这个很关键。

它决定了两件事：

* 对象怎么创建
* 对象归还时能不能重新进入池

也就是说，池化不仅是“有个容器”，还要有一套规则。

### 自定义对象池怎么写？

如果你要池化的是自己的类型，通常会先写一个策略类。

先准备一个可复用对象：

```csharp
public sealed class ReusableBuffer
{
    public byte[] Buffer { get; } = new byte[4096];
    public int Length { get; set; }

    public void Reset()
    {
        Length = 0;
        Array.Clear(Buffer, 0, Buffer.Length);
    }
}
```

然后写一个策略：

```csharp
using Microsoft.Extensions.ObjectPool;

public sealed class ReusableBufferPolicy : PooledObjectPolicy<ReusableBuffer>
{
    public override ReusableBuffer Create()
    {
        return new ReusableBuffer();
    }

    public override bool Return(ReusableBuffer obj)
    {
        obj.Reset();
        return true;
    }
}
```

最后创建并使用池：

```csharp
var provider = new DefaultObjectPoolProvider();
var pool = provider.Create(new ReusableBufferPolicy());

var buffer = pool.Get();

try
{
    buffer.Buffer[0] = 100;
    buffer.Length = 1;
}
finally
{
    pool.Return(buffer);
}
```

这里最重要的是 `Return()` 里的逻辑。

因为对象要不要重新进入池，不是无条件的。

你完全可以在这里做判断：

* 对象状态不对，不回池
* 对象太大，不回池
* 对象被污染，不回池

例如：

```csharp
public override bool Return(ReusableBuffer obj)
{
    if (obj.Buffer.Length > 1024 * 1024)
    {
        return false;
    }

    obj.Reset();
    return true;
}
```

这类写法在工程上很有价值，因为有些对象一旦膨胀得太大，继续留在池里反而会浪费内存。

### `IResettable` 是什么？

官方实现还提供了一个很实用的思路：

```csharp
IResettable
```

如果对象本身就知道怎么把自己恢复到可复用状态，那它就更适合池化。

可以把它理解成：

* `PooledObjectPolicy<T>` 负责池规则
* `IResettable` 更像对象自己声明“我知道怎么重置自己”

这类设计的好处是职责更清楚：

* 池负责借还
* 对象负责恢复

### 在 ASP.NET Core 里怎么接到 DI？

如果你不想手动在每个地方 new `DefaultObjectPoolProvider`，更自然的方式通常是接进 `DI`。

例如：

```csharp
using Microsoft.Extensions.ObjectPool;

var builder = WebApplication.CreateBuilder(args);

builder.Services.TryAddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
builder.Services.TryAddSingleton<ObjectPool<StringBuilder>>(sp =>
{
    var provider = sp.GetRequiredService<ObjectPoolProvider>();
    return provider.CreateStringBuilderPool();
});
```

后面在服务里直接注入：

```csharp
public sealed class MessageBuilderService
{
    private readonly ObjectPool<StringBuilder> _pool;

    public MessageBuilderService(ObjectPool<StringBuilder> pool)
    {
        _pool = pool;
    }

    public string Build(string name)
    {
        var sb = _pool.Get();

        try
        {
            sb.Append("hello ");
            sb.Append(name);
            return sb.ToString();
        }
        finally
        {
            _pool.Return(sb);
        }
    }
}
```

这种写法更适合真实项目，因为池本身通常应该是：

* 单例
* 长生命周期
* 全局复用

### 从源码心智模型看，它内部大致是什么样？

不用先盯实现细节，先记住一个更重要的事实：

> `ObjectPool` 追求的是“尽量低成本地复用少量对象”，不是“做一个严格、复杂、功能很全的资源管理器”。

所以它的实现思路也很务实：

* 尽量快速取到对象
* 尽量快速还回对象
* 放不下就算了

也就是说，它不是那种：

* 排队非常严格
* 绝不超量创建
* 生命周期管理很复杂

的重型池模型。

这也是为什么它在热点路径里更好用：

* 逻辑简单
* 开销可控
* 行为可预测

### 它适合什么场景？

优先在这些场景里考虑它：

* 高频创建、短生命周期对象
* 对象可以明确重置
* 热点路径对分配和 `GC` 比较敏感
* 对象复用收益明显高于管理成本

典型例子包括：

* `StringBuilder`
* 文本拼接缓冲对象
* 临时解析器
* 可重复使用的请求上下文辅助对象
* 某些序列化 / 反序列化辅助对象

### 它不适合什么场景？

这部分比“怎么用”更重要，因为对象池很容易被滥用。

#### 1. 对象本身很轻，直接 `new` 成本极低

比如一个只有几个字段的小对象，直接池化很可能得不偿失。

因为你引入了：

* 借还成本
* 重置成本
* 代码复杂度

#### 2. 对象状态复杂，很难可靠重置

如果对象里带着很多内部状态、外部引用、事件、句柄、线程上下文，那池化风险会明显上升。

这时候最大的风险不是“没优化到”，而是：

* 脏状态泄漏到下一次使用

#### 3. 对象会长时间被占用

对象池最适合的是：

* 短借短还

如果对象拿走以后要用很久，那池化收益会越来越低。

#### 4. 想拿它当缓存

对象池不是为了让对象一直留着给未来业务命中，它只是复用实例。

#### 5. 想靠它解决并发限制

它不会因为池大小是 32，就保证系统同时最多只存在 32 个对象。

不够用的时候，它还是会继续创建。

### 使用时最容易踩的坑

#### 1. 忘记归还

最直接，也最常见。

所以建议形成固定写法：

```csharp
var item = pool.Get();
try
{
    // use item
}
finally
{
    pool.Return(item);
}
```

#### 2. 归还前没重置干净

这会直接导致脏数据串到下一次调用。

#### 3. 归还后继续使用对象

这是很危险的一类 bug。

一旦对象已经回池，它理论上随时可能被别人再次借走。

#### 4. 池化大对象，但不控制膨胀

有些对象会随着业务输入越长越大。

这时如果不做策略控制，池里可能慢慢堆满“已经膨胀过的大对象”，反而拉高常驻内存。

### 一个比较务实的判断标准

要不要上 `ObjectPool`，一般先看四件事：

1. 这个对象是不是热点路径里高频创建？
2. 创建和回收成本是不是已经值得关注？
3. 它能不能被可靠重置？
4. 池化带来的复杂度，值不值得这点收益？

如果这四个问题里，有两个以上答不上来，那通常先别急着池化。

### 总结

`ObjectPool` 最值得理解的，不是 API 有多简单，而是它背后的取舍：

* 用少量额外管理成本
* 换热点路径上的更少分配和更低 `GC` 压力

所以它真正适合的是：

* 高频
* 短生命周期
* 可重置
* 复用收益明显

一句话收尾：

> `ObjectPool` 不是“让对象永远不创建”，而是“让适合复用的对象别老是重复创建”。 
