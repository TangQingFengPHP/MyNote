### 简介

在 `C#.NET` 里，只要碰到“生产者不断产出数据，消费者持续处理数据”这类场景，最终都会绕到一个核心问题：

* 如何解耦生产速度和消费速度；
* 如何在高并发下保证吞吐量；
* 如何在异步场景下避免把线程白白阻塞住；
* 如何在队列堆积时做背压、限流或丢弃。

`System.Threading.Channels` 提供的 `Channel<T>`，就是 .NET 为这类问题准备的现代答案。它本质上是一个高性能、异步友好的生产者-消费者通道，既能像队列一样传递数据，又比手写 `ConcurrentQueue<T> + SemaphoreSlim` 更完整，还比传统 `BlockingCollection<T>` 更适合 `async/await` 场景。

如果你的项目里有这些需求，`Channel<T>` 基本都值得优先考虑：

* 后台任务队列；
* 日志异步落库或落文件；
* 批量导入、消息处理、事件分发；
* 爬虫、采集、推送等“生产快、消费慢”的链路；
* `BackgroundService` 中的异步工作流。

### Channel 到底解决了什么问题？

先看一个很常见但不够完整的实现思路：

```csharp
var queue = new ConcurrentQueue<WorkItem>();
var signal = new SemaphoreSlim(0);

// 生产者
queue.Enqueue(workItem);
signal.Release();

// 消费者
await signal.WaitAsync(ct);
if (queue.TryDequeue(out var item))
{
    await ProcessAsync(item, ct);
}
```

它不是不能用，但很快你就会补一堆外围逻辑：

* 队列容量限制怎么做？
* 队列满了以后是等待、丢旧数据，还是丢新数据？
* 写入完成后怎么优雅通知消费者退出？
* 多生产者、多消费者时如何减少竞争？
* 异常、取消、关闭语义如何统一？

`Channel<T>` 把这些问题一次性封装好了。

### 为什么现代 .NET 更推荐 Channel？

和其他常见方案相比，`Channel<T>` 的定位非常清晰：

| 方案 | 特点 | 更适合什么场景 |
| --- | --- | --- |
| `ConcurrentQueue<T>` | 只是线程安全队列，不负责等待、完成、背压 | 简单缓存、轻量队列 |
| `BlockingCollection<T>` | 同步阻塞模型成熟稳定 | 老项目、同步代码、`async` 较少的场景 |
| `Channel<T>` | 异步友好、支持背压、支持完成语义、吞吐高 | 新项目、后台服务、异步生产消费模型 |

一句话概括：

* 如果你主要写同步阻塞代码，`BlockingCollection<T>` 仍然可用；
* 如果你主要写异步代码，`Channel<T>` 通常是更自然的选择。

### Channel 的核心模型

`Channel<T>` 可以理解成“一个带完整生命周期的异步队列”，核心分为两端：

* `ChannelWriter<T>`：写端，给生产者使用；
* `ChannelReader<T>`：读端，给消费者使用。

最常见的创建方式：

```csharp
using System.Threading.Channels;

var channel = Channel.CreateUnbounded<string>();

ChannelWriter<string> writer = channel.Writer;
ChannelReader<string> reader = channel.Reader;
```

你可以把它想象成：

```text
生产者 -> Writer -> Channel -> Reader -> 消费者
```

常用 API 如下：

| API | 作用 |
| --- | --- |
| `writer.WriteAsync(item, ct)` | 异步写入，必要时等待 |
| `writer.TryWrite(item)` | 立即尝试写入，不等待 |
| `reader.ReadAsync(ct)` | 异步读取一个元素 |
| `reader.TryRead(out item)` | 立即尝试读取一个元素 |
| `reader.WaitToReadAsync(ct)` | 等待“有数据可读” |
| `reader.ReadAllAsync(ct)` | 以 `IAsyncEnumerable<T>` 方式持续读取 |
| `writer.Complete(ex)` / `writer.TryComplete(ex)` | 告知读端：不会再有新数据了 |
| `reader.Completion` | 通道彻底结束后的完成任务 |

### 两种最重要的通道类型

#### 无界通道

```csharp
var channel = Channel.CreateUnbounded<Order>();
```

特点：

* 理论上不限制容量；
* 写入方通常不会因为“队列满”而等待；
* 代码最简单，吞吐也高。

适合：

* 数据量相对可控；
* 消费速度通常跟得上；
* 只是想在模块间解耦。

不适合：

* 上游可能瞬间打爆下游；
* 队列堆积会造成明显内存风险。

#### 有界通道

```csharp
var channel = Channel.CreateBounded<Order>(1000);
```

特点：

* 容量固定；
* 达到上限后会触发背压或丢弃策略；
* 更适合真实业务系统。

如果你的数据流来自网络请求、消息队列、定时批处理，通常应该优先考虑有界通道，而不是无界通道。

### 有界通道最关键：满了以后怎么办？

有界通道真正有价值的地方，不只是“限制容量”，而是你可以明确指定队列满时的行为。

```csharp
var channel = Channel.CreateBounded<string>(
    new BoundedChannelOptions(3)
    {
        FullMode = BoundedChannelFullMode.Wait
    });
```

`BoundedChannelFullMode` 常见有 4 种策略：

| 模式 | 行为 | 典型场景 |
| --- | --- | --- |
| `Wait` | 写入方等待，直到有空间 | 最稳妥，适合任务处理 |
| `DropNewest` | 丢弃通道里最新的数据，再写当前数据 | 更关注旧数据时 |
| `DropOldest` | 丢弃最旧的数据，保留最新数据 | 状态刷新、监控、行情流 |
| `DropWrite` | 直接丢弃这次写入 | 允许丢包的高频数据 |

这就是 `Channel<T>` 比“普通队列 + 锁”更完整的地方。

### 基础示例：最典型的异步生产者消费者

下面这段代码已经覆盖了日常 80% 的用法。

```csharp
using System.Threading.Channels;

var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(5)
{
    FullMode = BoundedChannelFullMode.Wait
});

var producer = Task.Run(async () =>
{
    try
    {
        for (int i = 1; i <= 20; i++)
        {
            await channel.Writer.WriteAsync(i);
            Console.WriteLine($"生产: {i}");
            await Task.Delay(100);
        }

        channel.Writer.Complete();
    }
    catch (Exception ex)
    {
        channel.Writer.TryComplete(ex);
    }
});

var consumer = Task.Run(async () =>
{
    await foreach (var item in channel.Reader.ReadAllAsync())
    {
        Console.WriteLine($"消费: {item}");
        await Task.Delay(300);
    }
});

await Task.WhenAll(producer, consumer);
await channel.Reader.Completion;
```

这个示例里最重要的不是语法，而是行为：

* 消费慢于生产时，队列会逐步堆积；
* 当队列满了，`WriteAsync` 会等待，而不是继续把内存打满；
* 生产结束后调用 `Complete()`，消费者在读完剩余数据后自然退出；
* 整个过程不需要自己维护锁、信号量和退出标记。

### `ReadAllAsync` 为什么这么常用？

很多人第一次用 `Channel<T>`，会写成下面这样：

```csharp
while (await reader.WaitToReadAsync(ct))
{
    while (reader.TryRead(out var item))
    {
        await ProcessAsync(item, ct);
    }
}
```

这没问题，而且在性能敏感的场景里很常见，因为它能一次性把当前可读的数据尽可能读空。

但大多数业务代码里，`ReadAllAsync` 更简洁：

```csharp
await foreach (var item in reader.ReadAllAsync(ct))
{
    await ProcessAsync(item, ct);
}
```

可以直接理解为：

* 有数据就读；
* 没数据就异步等；
* 写端完成且数据读空后，循环自动结束。

如果没有特殊性能诉求，优先写 `ReadAllAsync`。

### `TryWrite`、`WriteAsync` 该怎么选？

这是一个很实际的问题。

#### `WriteAsync`

```csharp
await writer.WriteAsync(item, ct);
```

特点：

* 通道满时会等待；
* 能自然形成背压；
* 最适合任务型数据。

典型场景：

* 工单处理；
* 数据同步；
* 批量导出；
* 任何“不允许随便丢数据”的业务。

#### `TryWrite`

```csharp
if (!writer.TryWrite(item))
{
    // 可以记录日志、降级、丢弃或改走其他路径
}
```

特点：

* 不等待；
* 成功就写，失败立刻返回；
* 适合允许降级的高频场景。

典型场景：

* 埋点；
* 高频监控；
* 非关键通知；
* 实时刷新类数据。

经验上：

* 业务任务流优先 `WriteAsync`；
* 遥测、监控、采样类数据再考虑 `TryWrite`。

### 单生产者/单消费者优化

如果你能明确确认通道只有一个写入者、一个读取者，可以显式打开优化选项：

```csharp
var channel = Channel.CreateUnbounded<int>(new UnboundedChannelOptions
{
    SingleWriter = true,
    SingleReader = true
});
```

这不是功能开关，而是性能提示。运行时知道并发模型更简单后，可以减少额外同步成本。

同理，有界通道也支持：

```csharp
var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(1024)
{
    SingleWriter = true,
    SingleReader = true,
    FullMode = BoundedChannelFullMode.Wait
});
```

什么时候值得开？

* 你非常确定只有一个写线程；
* 你非常确定只有一个读线程；
* 通道是性能链路的一部分。

如果不能确定，就别乱开。

### `AllowSynchronousContinuations` 要不要设置？

这个选项经常被文章一笔带过，但实际项目里最好谨慎一点。

```csharp
var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(100)
{
    AllowSynchronousContinuations = false
});
```

它的含义大致是：

* 某些等待中的 continuation，是否允许在当前线程上同步执行。

通常建议：

* 普通业务代码保持默认值即可；
* 只有你明确在压测、诊断延迟并理解调用链时，再考虑调整。

简单说，它是一个“性能微调项”，不是“常规必配项”。

### 在 `BackgroundService` 中使用 Channel

`Channel<T>` 和 `BackgroundService` 是非常自然的一组搭配。

一个典型模式是：

* Web 请求线程只负责把任务写进通道；
* 后台消费者持续从通道取任务并处理；
* 这样请求入口更轻，真正耗时的工作放到后台完成。

示例：

```csharp
public sealed class EmailQueue
{
    private readonly Channel<EmailMessage> _channel;

    public EmailQueue()
    {
        _channel = Channel.CreateBounded<EmailMessage>(new BoundedChannelOptions(500)
        {
            FullMode = BoundedChannelFullMode.Wait
        });
    }

    public ValueTask EnqueueAsync(EmailMessage message, CancellationToken ct)
        => _channel.Writer.WriteAsync(message, ct);

    public IAsyncEnumerable<EmailMessage> DequeueAllAsync(CancellationToken ct)
        => _channel.Reader.ReadAllAsync(ct);

    public void Complete(Exception? ex = null)
        => _channel.Writer.TryComplete(ex);
}

public sealed class EmailBackgroundService : BackgroundService
{
    private readonly EmailQueue _emailQueue;
    private readonly ILogger<EmailBackgroundService> _logger;

    public EmailBackgroundService(
        EmailQueue emailQueue,
        ILogger<EmailBackgroundService> logger)
    {
        _emailQueue = emailQueue;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        try
        {
            await foreach (var message in _emailQueue.DequeueAllAsync(stoppingToken))
            {
                await SendEmailAsync(message, stoppingToken);
            }
        }
        catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("邮件后台服务正在停止");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "邮件后台服务处理失败");
            throw;
        }
    }

    private static Task SendEmailAsync(EmailMessage message, CancellationToken ct)
    {
        return Task.Delay(200, ct);
    }
}
```

这个模式在实际项目里非常常见，因为它有几个直接好处：

* 请求线程不用自己做耗时 I/O；
* 有界通道天然能做削峰和限流；
* 停机时可以通过完成写端实现优雅退出。

### 多消费者场景怎么写？

如果你希望多个消费者并行处理同一条任务流，可以让多个任务同时读取同一个 `Reader`。

```csharp
var channel = Channel.CreateBounded<int>(100);

var workers = Enumerable.Range(0, 4)
    .Select(workerId => Task.Run(async () =>
    {
        await foreach (var item in channel.Reader.ReadAllAsync())
        {
            Console.WriteLine($"Worker {workerId} 处理 {item}");
            await Task.Delay(200);
        }
    }))
    .ToArray();
```

这里要注意一个关键语义：

* 多个消费者不是“广播消费”；
* 而是“竞争消费”；
* 同一条消息只会被其中一个消费者拿到。

如果你要的是广播，那 `Channel<T>` 不是直接答案，你需要自己做 fan-out，或者为每个消费者维护独立通道。

### 异常和关闭语义怎么处理？

这是 `Channel<T>` 最容易写乱的地方。

推荐记住 3 条规则：

#### 1. 正常结束时，写端要完成

```csharp
writer.Complete();
```

或者：

```csharp
writer.TryComplete();
```

如果你不完成写端，读端的 `ReadAllAsync()` 很可能会一直等下去。

#### 2. 生产者异常时，把异常传给通道

```csharp
catch (Exception ex)
{
    writer.TryComplete(ex);
}
```

这样消费者在等待完成时就能感知错误，而不是悄悄卡住。

#### 3. 取消不等于异常

消费者常见写法：

```csharp
try
{
    await foreach (var item in reader.ReadAllAsync(ct))
    {
        await ProcessAsync(item, ct);
    }
}
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    // 正常取消
}
```

如果是应用关闭触发的取消，通常不应该当作错误日志刷满屏。

### Channel 和 BlockingCollection 到底怎么选？

这个问题最好直接按场景回答。

#### 更适合 `Channel<T>` 的场景

* 整个调用链本身就是异步的；
* 你要把工作从请求线程卸载到后台；
* 你需要背压和容量限制；
* 你希望优雅处理取消、关闭、异常传播；
* 你在写 `BackgroundService`、Worker、消息处理程序。

#### 更适合 `BlockingCollection<T>` 的场景

* 老项目大量同步代码，不打算改成异步；
* 团队整体就是阻塞式编程模型；
* 运行环境较老，历史包袱比较重。

很多新项目里，如果你拿不准，默认优先选 `Channel<T>`，通常问题不大。

### 常见误区与踩坑点

#### 1. 无界通道不是“更省事”，而是“更容易堆内存”

如果上游入口很猛，下游处理很慢，无界通道只会把问题推迟成内存问题。

生产环境里，默认优先考虑有界通道。

#### 2. 忘记 `Complete()`，消费者会一直挂着

这是最常见的问题之一。

只要你的消费端使用了 `ReadAllAsync()`，就必须认真设计“什么时候关闭写端”。

#### 3. `Channel<T>` 不是持久化消息队列

它只是进程内通道，不会帮你落盘，不保证进程崩溃后数据还在。

如果你需要的是可靠消息投递、断电恢复、跨进程消费，那应该看 Kafka、RabbitMQ、Azure Queue 之类的系统。

#### 4. 多消费者不是广播

前面已经提过，再强调一次：

* 一个元素被一个消费者取走后，其他消费者就看不到了。

#### 5. 不要把 `TryWrite` 当成“高性能版 `WriteAsync`”

它们的语义不同，不只是同步异步区别。

* `WriteAsync` 是“必要时等待，保证尽量写进去”；
* `TryWrite` 是“此刻能写就写，不能写就算了”。

### 一套比较稳妥的实战建议

如果你只是想在项目里把 `Channel<T>` 用对，下面这套默认策略基本够用了：

* 默认优先用有界通道；
* 默认 `FullMode = Wait`；
* 默认消费者用 `ReadAllAsync()`；
* 生产完成后一定调用 `TryComplete()`；
* 入口请求只负责入队，耗时逻辑放后台；
* 需要限流时调小容量，需要削峰时调大容量；
* 允许丢弃的数据，再考虑 `TryWrite` 或 `DropOldest/DropWrite`。

一个非常常见、也非常稳的配置如下：

```csharp
var channel = Channel.CreateBounded<Job>(new BoundedChannelOptions(1000)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleWriter = false,
    SingleReader = false
});
```

如果你不知道从哪里开始，就从这个配置开始。

### 总结

`Channel<T>` 的本质，不只是“线程安全队列”，而是一个带有异步等待、背压控制、完成语义和并发优化的生产者-消费者基础设施。

你可以这样理解它：

* 想要简单线程安全存取，用并发集合；
* 想要同步阻塞式生产消费，用 `BlockingCollection<T>`；
* 想要现代异步、可限流、可优雅关闭的生产消费模型，用 `Channel<T>`。

在今天的 `.NET` 项目里，尤其是 `ASP.NET Core`、Worker Service、后台任务系统中，`Channel<T>` 已经是非常值得优先掌握的一项能力。
