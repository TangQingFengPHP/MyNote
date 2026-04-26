### 简介

异步代码一多，参数传递很快就会开始变味。

最常见的场景是这样：

* 入口层拿到了 `TraceId`
* 服务层要打日志
* 仓储层也想拿到同一个 `TraceId`
* 调了好几层 `await` 以后，这个值还得一直跟着走

如果每一层都靠方法参数往下传，代码会越来越啰嗦。

这时候很多人会先想到：

* 放静态变量里，不行，会串请求
* 放 `ThreadLocal<T>` 里，不稳，`await` 之后线程可能早换了

`AsyncLocal<T>` 就是专门解决这类问题的。

一句话先说透：

> `AsyncLocal<T>` 绑定的不是线程，而是异步调用链的执行上下文。

所以这篇文章重点会放在几件事上：

* `AsyncLocal<T>` 到底解决什么问题
* 为什么它能跨 `await` 保持值不丢
* 它和 `ThreadLocal<T>`、静态变量的边界是什么
* 哪些场景适合用，哪些场景反而容易踩坑
* 怎么写出接近真实项目的 demo，而不是只背几个 API

### `AsyncLocal<T>` 是什么？

`AsyncLocal<T>` 位于：

```csharp
System.Threading
```

它的作用可以直接理解成：

* 给当前异步执行流挂一份上下文数据
* 这份数据可以穿过 `await`
* 后续方法不用显式传参，也能读到它

先别急着把它想得太玄乎。

它并不是全局变量，也不是线程变量，更不是锁。

更准确的说法是：

> `AsyncLocal<T>` 是一份“跟着当前逻辑调用链走”的上下文数据槽位。

### 为什么会需要它？

先看一个非常典型的需求。

接口请求进来时生成一个链路 ID：

```csharp
string traceId = Guid.NewGuid().ToString("N");
```

随后调用过程可能会经过：

* 控制器
* 应用服务
* 领域服务
* 仓储
* 日志组件

如果每一层都这样传：

```csharp
await service.CreateOrderAsync(orderDto, traceId);
```

再继续往下：

```csharp
await repository.SaveAsync(order, traceId);
```

很快就会出现两个问题：

* 很多方法参数只是为了透传上下文，业务本身并不关心
* 参数一多，真正有业务意义的输入反而被淹没了

这类场景下，`AsyncLocal<T>` 会比较顺手：

* 请求入口设置一次
* 后面整条异步调用链都能读取
* 不需要每层显式带着跑

### 它和 `ThreadLocal<T>` 的区别到底在哪？

这个点一定要先分清。

#### `ThreadLocal<T>`

绑定的是物理线程。

也就是说：

* 当前线程写进去的值
* 只能保证当前线程继续读到
* 一旦 `await` 后续体切到别的线程，就可能读不到原来的值

#### `AsyncLocal<T>`

绑定的是逻辑执行上下文。

也就是说：

* 即使 `await` 之后换了线程
* 只要还在同一条异步调用链上
* 值通常就还能继续拿到

一句话总结最方便记：

* `ThreadLocal<T>` 看线程
* `AsyncLocal<T>` 看异步调用链

### Demo 1：跨 `await` 保持上下文

先看最基础的例子。

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

static AsyncLocal<string> TraceId = new();

static async Task Main()
{
    TraceId.Value = "trace-1001";

    Console.WriteLine($"Main 开始，线程：{Thread.CurrentThread.ManagedThreadId}，值：{TraceId.Value}");

    await ProcessAsync();

    Console.WriteLine($"Main 结束，线程：{Thread.CurrentThread.ManagedThreadId}，值：{TraceId.Value}");
}

static async Task ProcessAsync()
{
    Console.WriteLine($"ProcessAsync 开始，线程：{Thread.CurrentThread.ManagedThreadId}，值：{TraceId.Value}");

    await Task.Delay(100);

    Console.WriteLine($"ProcessAsync 恢复后，线程：{Thread.CurrentThread.ManagedThreadId}，值：{TraceId.Value}");
}
```

输出通常类似这样：

```text
Main 开始，线程：1，值：trace-1001
ProcessAsync 开始，线程：1，值：trace-1001
ProcessAsync 恢复后，线程：7，值：trace-1001
Main 结束，线程：7，值：trace-1001
```

关键点不是线程号，而是：

* 线程可能变了
* `TraceId` 没丢

这就是 `AsyncLocal<T>` 的核心价值。

### Demo 2：和 `ThreadLocal<T>` 放在一起看，差别会非常直观

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

static ThreadLocal<string> ThreadTrace = new(() => "empty-thread");
static AsyncLocal<string> AsyncTrace = new();

static async Task Main()
{
    ThreadTrace.Value = "thread-trace";
    AsyncTrace.Value = "async-trace";

    await Task.Delay(100).ConfigureAwait(false);

    Console.WriteLine($"ThreadLocal：{ThreadTrace.Value}");
    Console.WriteLine($"AsyncLocal：{AsyncTrace.Value}");
}
```

这里的结果最常见的是：

* `ThreadLocal<T>` 可能读到默认值，或者读到当前线程自己的那份旧值
* `AsyncLocal<T>` 仍然能拿到 `async-trace`

原因就在于两者绑定对象根本不同。

### `AsyncLocal<T>` 为什么能做到这件事？

背后关键不是 `AsyncLocal<T>` 自己，而是 `.NET` 的：

```csharp
ExecutionContext
```

可以把它想成一份“当前执行环境的上下文包裹”。

这个包裹里可以带很多信息，其中就包括 `AsyncLocal<T>` 的值。

当异步方法遇到 `await` 时，运行时通常会做这些事：

* 先把当前执行上下文捕获下来
* 异步操作完成后，再把这个上下文恢复回来
* 于是后续代码还能继续看到原来的 `AsyncLocal<T>` 值

所以真正流动的不是线程，而是上下文。

### Demo 3：父流程设置值，子流程可以直接读取

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

static AsyncLocal<string> CurrentUser = new();

static async Task Main()
{
    CurrentUser.Value = "admin";
    await CreateOrderAsync();
}

static async Task CreateOrderAsync()
{
    Console.WriteLine($"CreateOrderAsync: {CurrentUser.Value}");
    await SaveOrderAsync();
}

static async Task SaveOrderAsync()
{
    await Task.Delay(50);
    Console.WriteLine($"SaveOrderAsync: {CurrentUser.Value}");
}
```

输出会保持一致：

```text
CreateOrderAsync: admin
SaveOrderAsync: admin
```

这正是它在请求上下文、租户上下文、日志作用域里被频繁使用的原因。

### Demo 4：子流程改值，父流程不会被永久污染

这是 `AsyncLocal<T>` 很容易让人误判的地方。

很多人第一次接触时，会以为它就是一份所有子流程共享的全局变量。实际上不是。

看例子：

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

static AsyncLocal<string> Context = new();

static async Task Main()
{
    Context.Value = "root";

    Console.WriteLine($"Main 调用前：{Context.Value}");
    await OuterAsync();
    Console.WriteLine($"Main 调用后：{Context.Value}");
}

static async Task OuterAsync()
{
    Console.WriteLine($"OuterAsync 开始：{Context.Value}");

    Context.Value = "outer";
    await InnerAsync();

    Console.WriteLine($"OuterAsync 结束：{Context.Value}");
}

static async Task InnerAsync()
{
    Console.WriteLine($"InnerAsync 开始：{Context.Value}");

    Context.Value = "inner";
    await Task.Delay(50);

    Console.WriteLine($"InnerAsync 结束：{Context.Value}");
}
```

一类常见输出会像这样：

```text
Main 调用前：root
OuterAsync 开始：root
InnerAsync 开始：outer
InnerAsync 结束：inner
OuterAsync 结束：outer
Main 调用后：root
```

这个现象最值得记住：

* 子流程能继承父流程当下的值
* 子流程里重新赋值，只会影响它自己的那段上下文
* 父流程恢复后，还是原来的值

所以更准确的理解应该是：

> `AsyncLocal<T>` 更像“上下文作用域”，不是单纯的一份共享变量。

### Demo 5：最贴近项目的场景，链路追踪 `TraceId`

这个例子最接近真实项目。

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

public static class TraceContext
{
    private static readonly AsyncLocal<string?> _traceId = new();

    public static string? TraceId
    {
        get => _traceId.Value;
        set => _traceId.Value = value;
    }
}

public sealed class OrderService
{
    public async Task CreateAsync()
    {
        Console.WriteLine($"[Service] TraceId={TraceContext.TraceId}");
        await new OrderRepository().SaveAsync();
    }
}

public sealed class OrderRepository
{
    public async Task SaveAsync()
    {
        await Task.Delay(50);
        Console.WriteLine($"[Repository] TraceId={TraceContext.TraceId}");
    }
}

static async Task Main()
{
    TraceContext.TraceId = Guid.NewGuid().ToString("N");

    try
    {
        await new OrderService().CreateAsync();
    }
    finally
    {
        TraceContext.TraceId = null;
    }
}
```

这种模式非常适合：

* 链路追踪 ID
* 当前租户 ID
* 当前请求用户
* 日志作用域

最关键的收益是：

* 业务方法签名不会为了透传上下文越来越臃肿
* 底层组件也能直接读取当前上下文

### Demo 6：引用类型是个大坑

这点必须单独讲。

很多人知道 `AsyncLocal<T>` 能隔离上下文，就误以为里面放引用类型也天然安全。其实并不是。

看这个例子：

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

public sealed class RequestInfo
{
    public string TraceId { get; set; } = string.Empty;
}

static AsyncLocal<RequestInfo> RequestContext = new();

static async Task Main()
{
    RequestContext.Value = new RequestInfo { TraceId = "root-trace" };

    await Task.Run(async () =>
    {
        Console.WriteLine($"子任务开始：{RequestContext.Value.TraceId}");

        RequestContext.Value.TraceId = "child-updated";
        await Task.Delay(50);
    });

    Console.WriteLine($"主流程恢复后：{RequestContext.Value.TraceId}");
}
```

这里很可能输出：

```text
子任务开始：root-trace
主流程恢复后：child-updated
```

原因不是 `AsyncLocal<T>` 失效了，而是：

* `AsyncLocal<T>` 传的是 `RequestInfo` 这个引用
* 子流程和父流程拿到的是同一个对象引用
* 改的是对象内部属性，不是重新给 `.Value` 赋新对象

所以实战里最好优先遵循这个原则：

* 传简单值类型
* 传字符串
* 传不可变对象
* 尽量少传可变大对象

如果非要放复杂对象，最好按“整体替换”来写，而不是到处改内部属性。

### Demo 7：变化通知

`AsyncLocal<T>` 还有一个不算常用但挺有意思的能力：值变化通知。

```csharp
using System;
using System.Threading;

var local = new AsyncLocal<string?>(args =>
{
    Console.WriteLine(
        $"上下文变化：旧值={args.PreviousValue ?? "<null>"}，新值={args.CurrentValue ?? "<null>"}，" +
        $"线程切换={args.ThreadContextChanged}");
});

local.Value = "A";
local.Value = "B";
local.Value = null;
```

这个能力更适合：

* 调试上下文传播
* 验证链路是否被改写
* 观察异步恢复时值变化

业务代码里一般不需要到处用，但排查问题时很有帮助。

### 它和静态变量的边界是什么？

这一点也很容易混。

#### 静态变量

是全局共享的。

如果多个请求同时进来，都改同一个静态字段，数据就会互相覆盖。

#### `AsyncLocal<T>`

通常是“静态字段 + 每条异步调用链各自一份值”的组合。

也就是说：

* 静态的是容器入口
* 真正的值不是全局共享那一份
* 每条执行流看到的是自己的上下文值

这也是为什么框架里很多上下文访问器，会把 `AsyncLocal<T>` 写成 `static readonly` 字段。

### 什么时候适合用 `AsyncLocal<T>`？

比较适合：

* `TraceId`
* `CorrelationId`
* 当前租户信息
* 当前用户标识
* 日志上下文
* 需要跨 `await` 透传、但不想层层传参的控制类信息

不太适合：

* 大对象缓存
* 大量频繁写入的临时业务数据
* 需要长期保活的资源对象
* 复杂可变对象图
* 真正的全局共享状态

### 为什么不建议往里面塞大对象？

原因有两个。

#### 1. 它的定位不是缓存容器

`AsyncLocal<T>` 最适合装的是“小而关键的上下文信息”，比如 ID、名称、轻量上下文对象。

#### 2. 上下文传播本身也有成本

异步链路越复杂，传播越频繁，塞的对象越重，排查问题和控制生命周期就越麻烦。

尤其是在高并发服务里，`AsyncLocal<T>` 应该尽量保持轻量。

### 后台任务场景一定要格外小心

还有一个很容易踩坑的点：

`Task.Run` 默认会捕获当前 `ExecutionContext`，也就意味着会把当前 `AsyncLocal<T>` 一起带过去。

这有时是好事，有时反而是坏事。

例如某个请求里启动了一个后台任务：

```csharp
_ = Task.Run(() =>
{
    Console.WriteLine(TraceContext.TraceId);
});
```

如果本意只是“顺手丢个后台工作”，却不希望把请求上下文一起传过去，那这个默认行为就可能造成误判甚至污染。

这种时候要意识到一件事：

> `AsyncLocal<T>` 默认是会随执行上下文流动的，不是只在当前方法里生效。

### 高级一点的控制：禁止上下文流动

如果确实明确不想把当前上下文传给后台任务，可以用：

```csharp
using System.Threading;

using (ExecutionContext.SuppressFlow())
{
    _ = Task.Run(() =>
    {
        Console.WriteLine(TraceContext.TraceId); // 大概率拿不到原上下文值
    });
}
```

这个 API 不算日常高频，但在基础设施代码里很有用。

它适合那种语义非常明确的场景：

* 就是不希望继承当前请求上下文
* 就是要启动一个“脱离当前链路”的后台任务

### 一个更完整的实战写法：带作用域的上下文包装

项目里直接到处写：

```csharp
TraceContext.TraceId = xxx;
```

时间长了很容易忘记恢复。

更稳一点的方式是做一个小作用域包装：

```csharp
using System;
using System.Threading;

public static class TraceContext
{
    private static readonly AsyncLocal<string?> _traceId = new();

    public static string? TraceId
    {
        get => _traceId.Value;
        private set => _traceId.Value = value;
    }

    public static IDisposable BeginScope(string traceId)
    {
        string? previous = TraceId;
        TraceId = traceId;
        return new RestoreScope(previous);
    }

    private sealed class RestoreScope : IDisposable
    {
        private readonly string? _previous;

        public RestoreScope(string? previous)
        {
            _previous = previous;
        }

        public void Dispose()
        {
            TraceId = _previous;
        }
    }
}
```

使用时：

```csharp
using (TraceContext.BeginScope(Guid.NewGuid().ToString("N")))
{
    await service.CreateAsync();
}
```

这种写法的好处很直接：

* 进入作用域时设置值
* 离开作用域时自动恢复
* 比手动清理更不容易漏

### 它和 `HttpContext.Items`、方法参数传递怎么选？

这几个工具也经常被放在一起比较。

#### 方法参数传递

优点是显式、清楚、最容易追踪。

缺点是透传链路一长，签名会越来越臃肿。

#### `HttpContext.Items`

适合明确绑定在 ASP.NET Core 请求对象上的数据。

但它依赖 `HttpContext`，脱离 Web 请求上下文就不好用了。

#### `AsyncLocal<T>`

适合那种：

* 需要跨很多层 `await`
* 不想层层传参
* 又不应该做成全局共享变量

最实用的判断标准可以这么记：

* 业务核心输入，优先参数传递
* 请求附属上下文，适合 `AsyncLocal<T>`
* 只在 ASP.NET Core 请求里临时挂点数据，`HttpContext.Items` 也可以

### 总结

`AsyncLocal<T>` 最核心的价值，不是“存个值”这么简单，而是：

* 让上下文跟着异步调用链走
* 跨 `await` 也不容易丢
* 不用为了透传上下文把方法签名写得越来越重

但边界也必须记清楚：

* 它不是线程变量
* 它不是全局共享变量
* 它不适合装大对象和复杂可变对象
* 它默认会随着执行上下文继续传播

> `AsyncLocal<T>` 不是把数据绑在线程上，而是把数据挂在当前异步调用链上。
