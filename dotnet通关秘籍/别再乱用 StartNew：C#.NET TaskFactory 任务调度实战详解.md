### 简介

`TaskFactory` 是 `.NET` 里专门用来创建和调度 `Task` 的工厂类。

最常见的入口是：

```csharp
Task.Factory.StartNew(...)
```

很多代码里会把它当成 `Task.Run` 的高级版，甚至直接用它替代 `Task.Run`。

这种理解只对了一半。

一句话概括：

```text
TaskFactory 适合需要精细控制任务创建、调度器、取消令牌、创建选项和延续任务的场景。
```

普通后台任务，直接用 `Task.Run` 通常更清楚。

只有需要这些能力时，`TaskFactory` 才更合适：

* 指定 `TaskScheduler`
* 指定 `TaskCreationOptions`
* 传入 `CancellationToken`
* 使用 `LongRunning`
* 使用 `AttachedToParent`
* 使用 `ContinueWhenAll`
* 使用 `ContinueWhenAny`
* 把旧式 `Begin/End` 异步模型转成 `Task`

### Task.Factory 是什么？

`Task.Factory` 是 `Task` 类型上的静态属性，返回一个默认的 `TaskFactory`。

常见写法：

```csharp
Task task = Task.Factory.StartNew(() =>
{
    Console.WriteLine("任务执行中");
});
```

也可以自己创建一个 `TaskFactory`：

```csharp
var factory = new TaskFactory(
    CancellationToken.None,
    TaskCreationOptions.None,
    TaskContinuationOptions.None,
    TaskScheduler.Default);
```

手动创建 `TaskFactory` 的意义是统一默认配置。

例如一批任务都要使用同一个调度器、同一个取消令牌，就可以放进同一个工厂里。

### TaskFactory 能做什么？

常用方法如下：

| 方法 | 作用 |
| --- | --- |
| `StartNew` | 创建并启动一个任务 |
| `ContinueWhenAll` | 多个任务全部完成后执行延续任务 |
| `ContinueWhenAny` | 任意一个任务完成后执行延续任务 |
| `FromAsync` | 把旧式 APM 异步模型包装成 `Task` |

其中最常见的是 `StartNew`。

`StartNew` 比 `Task.Run` 参数更多，因此也更容易写错。

### Task.Run 和 Task.Factory.StartNew 的关系

`Task.Run` 可以理解成一种更安全、更固定配置的快捷写法。

这段代码：

```csharp
Task task = Task.Run(() => DoWork());
```

大致接近：

```csharp
Task task = Task.Factory.StartNew(
    () => DoWork(),
    CancellationToken.None,
    TaskCreationOptions.DenyChildAttach,
    TaskScheduler.Default);
```

重点有两个：

* `Task.Run` 使用 `TaskScheduler.Default`
* `Task.Run` 默认带 `DenyChildAttach`

而这个写法：

```csharp
Task task = Task.Factory.StartNew(() => DoWork());
```

使用的是当前默认调度器，不一定永远等于线程池调度器。

在普通控制台程序里，差异可能不明显。

但在 UI 程序、自定义调度器、延续任务链里，这个差异就可能影响执行线程。

### 两者怎么选？

| 场景 | 推荐 |
| --- | --- |
| 普通后台计算 | `Task.Run` |
| ASP.NET Core 里包装同步代码 | 通常不推荐额外包一层 |
| 需要指定 `LongRunning` | `Task.Factory.StartNew` |
| 需要指定自定义 `TaskScheduler` | `TaskFactory` |
| 需要 `ContinueWhenAll` / `ContinueWhenAny` | `TaskFactory` 或 `Task.WhenAll` / `Task.WhenAny` |
| 需要从 `Begin/End` 老 API 转成 Task | `Task.Factory.FromAsync` |

日常开发里，可以先记住这个规则：

```text
能用 Task.Run 说清楚的任务，不要强行换成 StartNew。
需要调度细节时，再使用 TaskFactory。
```

### Demo 目标

下面写一个控制台 Demo，模拟订单报表生成任务。

场景如下：

```text
读取多个门店订单数据
  |
  v
并行计算每个门店销售额
  |
  v
汇总所有门店结果
  |
  v
取最快返回的门店结果
  |
  v
演示取消、异常、LongRunning、async 委托坑
```

这个 Demo 不依赖数据库，复制到控制台项目即可运行。

创建项目：

```bash
mkdir TaskFactoryDemo
cd TaskFactoryDemo

dotnet new console
```

### 准备模型

修改 `Program.cs`：

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

var stores = new[]
{
    new StoreOrder("上海店", new[] { 120m, 88m, 300m }),
    new StoreOrder("杭州店", new[] { 66m, 180m, 210m }),
    new StoreOrder("深圳店", new[] { 520m, 45m, 99m }),
};

Console.WriteLine($"主线程：{Environment.CurrentManagedThreadId}");

record StoreOrder(string StoreName, decimal[] Amounts);

record StoreReport(string StoreName, decimal TotalAmount, int ThreadId);
```

后面示例都基于这两个类型。

### 基础 StartNew

`StartNew` 会创建并启动一个任务。

```csharp
Task task = Task.Factory.StartNew(() =>
{
    Console.WriteLine($"任务线程：{Environment.CurrentManagedThreadId}");
    Console.WriteLine("开始生成订单报表");
});

await task;
```

有返回值时，返回 `Task<TResult>`：

```csharp
Task<StoreReport> reportTask = Task.Factory.StartNew(() =>
{
    var store = stores[0];

    decimal total = store.Amounts.Sum();

    return new StoreReport(
        store.StoreName,
        total,
        Environment.CurrentManagedThreadId);
});

StoreReport report = await reportTask;

Console.WriteLine($"{report.StoreName} 销售额：{report.TotalAmount}，线程：{report.ThreadId}");
```

这类简单场景用 `Task.Run` 也可以，而且更推荐：

```csharp
Task<StoreReport> reportTask = Task.Run(() =>
{
    var store = stores[0];
    return new StoreReport(store.StoreName, store.Amounts.Sum(), Environment.CurrentManagedThreadId);
});
```

### 传递 state，减少闭包

`StartNew` 支持传入 `object state`。

```csharp
Task<StoreReport> stateTask = Task.Factory.StartNew(static state =>
{
    var store = (StoreOrder)state!;

    return new StoreReport(
        store.StoreName,
        store.Amounts.Sum(),
        Environment.CurrentManagedThreadId);
}, stores[1]);

StoreReport stateReport = await stateTask;

Console.WriteLine($"{stateReport.StoreName} 销售额：{stateReport.TotalAmount}");
```

这类写法的好处是可以减少闭包捕获。

闭包写法是这样：

```csharp
var store = stores[1];

Task<StoreReport> taskWithClosure = Task.Factory.StartNew(() =>
{
    return new StoreReport(store.StoreName, store.Amounts.Sum(), Environment.CurrentManagedThreadId);
});
```

大多数业务代码不用纠结这点。

但在高频创建大量任务的场景里，`state` 参数可以少一些额外分配。

### CancellationToken 取消任务

`CancellationToken` 不会强制杀死线程。

它只是一个取消信号，任务内部要主动检查。

```csharp
using var cts = new CancellationTokenSource();

Task cancelTask = Task.Factory.StartNew(() =>
{
    for (int i = 1; i <= 10; i++)
    {
        cts.Token.ThrowIfCancellationRequested();

        Console.WriteLine($"处理第 {i} 批订单");
        Thread.Sleep(300);
    }
}, cts.Token);

await Task.Delay(900);
cts.Cancel();

try
{
    await cancelTask;
}
catch (OperationCanceledException)
{
    Console.WriteLine("任务已取消");
}
```

这里有两个细节：

* `cts.Token` 传给 `StartNew`
* 任务内部调用 `ThrowIfCancellationRequested()`

如果只传 Token，但任务内部从不检查，正在执行的代码不会自动停下来。

### LongRunning 长任务

`TaskCreationOptions.LongRunning` 表示这是一个长时间运行的粗粒度任务。

它不是强制命令，而是给调度器的提示。

默认调度器通常会为它创建独立线程，避免长期占用线程池工作线程。

```csharp
Task longRunningTask = Task.Factory.StartNew(() =>
{
    Console.WriteLine($"LongRunning 线程：{Environment.CurrentManagedThreadId}");

    for (int i = 1; i <= 3; i++)
    {
        Console.WriteLine($"后台队列第 {i} 次轮询");
        Thread.Sleep(500);
    }
},
CancellationToken.None,
TaskCreationOptions.LongRunning,
TaskScheduler.Default);

await longRunningTask;
```

适合：

* 长时间阻塞任务
* 独立后台循环
* 持续消费队列
* 无法改造成真正异步的老代码

不适合：

* 普通接口请求
* 普通数据库查询
* 短小计算任务
* 已经有异步 API 的 I/O 操作

例如这类写法通常没有意义：

```csharp
await Task.Factory.StartNew(async () =>
{
    await httpClient.GetStringAsync(url);
}, TaskCreationOptions.LongRunning);
```

网络 I/O 本身就有异步 API，不需要用 `LongRunning` 占一个线程。

### ContinueWhenAll：全部完成后汇总

`ContinueWhenAll` 用来等一组任务全部完成后，再执行一个延续任务。

```csharp
Task<StoreReport>[] reportTasks = stores
    .Select(store => Task.Factory.StartNew(static state =>
    {
        var item = (StoreOrder)state!;

        Thread.Sleep(300);

        return new StoreReport(
            item.StoreName,
            item.Amounts.Sum(),
            Environment.CurrentManagedThreadId);
    }, store))
    .ToArray();

Task<decimal> totalTask = Task.Factory.ContinueWhenAll(reportTasks, completedTasks =>
{
    decimal total = completedTasks.Sum(t => t.Result.TotalAmount);

    Console.WriteLine("所有门店计算完成");

    return total;
});

decimal totalAmount = await totalTask;

Console.WriteLine($"总销售额：{totalAmount}");
```

这段代码的流程是：

```text
多个门店任务并行执行
  |
  v
全部完成
  |
  v
汇总销售额
```

现代代码里也可以写成：

```csharp
StoreReport[] reports = await Task.WhenAll(reportTasks);
decimal total = reports.Sum(x => x.TotalAmount);
```

`Task.WhenAll` 通常更适合配合 `async/await` 使用。

`ContinueWhenAll` 更像早期 TPL 风格，适合需要设置延续选项或调度器的场景。

### ContinueWhenAny：谁先完成先处理

`ContinueWhenAny` 用来处理最快完成的任务。

```csharp
Task<StoreReport>[] raceTasks = stores
    .Select((store, index) => Task.Factory.StartNew(static state =>
    {
        var data = ((StoreOrder Store, int Index))state!;

        Thread.Sleep((data.Index + 1) * 300);

        return new StoreReport(
            data.Store.StoreName,
            data.Store.Amounts.Sum(),
            Environment.CurrentManagedThreadId);
    }, (store, index)))
    .ToArray();

Task<StoreReport> fastestTask = Task.Factory.ContinueWhenAny(raceTasks, completedTask =>
{
    return completedTask.Result;
});

StoreReport fastest = await fastestTask;

Console.WriteLine($"最快返回：{fastest.StoreName}，销售额：{fastest.TotalAmount}");
```

典型场景：

* 多个缓存源谁先返回用谁
* 多个服务节点谁先响应用谁
* 多个计算方案先拿到一个可用结果

现代写法也可以使用：

```csharp
Task<StoreReport> first = await Task.WhenAny(raceTasks);
StoreReport fastestReport = await first;
```

### FromAsync：包装旧式 Begin/End API

早期 .NET 有一类异步 API 使用 `BeginXxx` / `EndXxx` 模式，也叫 `APM`。

`TaskFactory.FromAsync` 可以把它们包装成 `Task`。

示例：

```csharp
byte[] buffer = new byte[1024];

await File.WriteAllTextAsync("orders.txt", "hello task factory");

await using FileStream stream = new(
    "orders.txt",
    FileMode.Open,
    FileAccess.Read,
    FileShare.Read,
    bufferSize: 4096,
    useAsync: true);

Task<int> readTask = Task.Factory.FromAsync(
    stream.BeginRead,
    stream.EndRead,
    buffer,
    0,
    buffer.Length,
    state: null);

int bytesRead = await readTask;

Console.WriteLine($"读取字节数：{bytesRead}");
```

现在大多数新 API 已经直接提供 `ReadAsync`、`WriteAsync`、`SendAsync`。

所以 `FromAsync` 更多用于兼容老接口。

### async 委托的大坑

`Task.Factory.StartNew` 遇到 `async` 委托时，很容易写出嵌套任务。

看这段代码：

```csharp
Task<Task<string>> nestedTask = Task.Factory.StartNew(async () =>
{
    await Task.Delay(500);
    return "异步结果";
});
```

返回类型不是：

```csharp
Task<string>
```

而是：

```csharp
Task<Task<string>>
```

原因是：

```text
StartNew 只负责启动委托。
async 委托本身又会返回一个 Task。
```

如果只等待外层任务：

```csharp
Task<string> innerTask = await nestedTask;
```

只表示 `async` 委托已经返回了内部任务，不代表内部异步操作已经完成。

正确处理方式有两个。

第一种，使用 `Unwrap()`：

```csharp
Task<string> unwrappedTask = Task.Factory.StartNew(async () =>
{
    await Task.Delay(500);
    return "异步结果";
}).Unwrap();

string result = await unwrappedTask;
```

第二种，直接用 `Task.Run`：

```csharp
Task<string> runTask = Task.Run(async () =>
{
    await Task.Delay(500);
    return "异步结果";
});

string result = await runTask;
```

这也是日常代码更推荐 `Task.Run` 的原因之一。

`Task.Run` 对 `Func<Task>` 和 `Func<Task<TResult>>` 有专门重载，使用起来更自然。

### 异常处理

`TaskFactory` 创建的任务如果抛异常，异常会存到 `Task.Exception` 里。

使用 `await` 时，异常会按原始异常抛出：

```csharp
Task errorTask = Task.Factory.StartNew(() =>
{
    throw new InvalidOperationException("报表生成失败");
});

try
{
    await errorTask;
}
catch (InvalidOperationException ex)
{
    Console.WriteLine(ex.Message);
}
```

使用 `Wait()` 或 `Result` 时，异常会被包进 `AggregateException`：

```csharp
Task<int> errorResultTask = Task.Factory.StartNew<int>(() =>
{
    throw new InvalidOperationException("计算失败");
});

try
{
    int result = errorResultTask.Result;
}
catch (AggregateException ex)
{
    Console.WriteLine(ex.InnerException?.Message);
}
```

所以异步代码里更推荐：

```csharp
await task;
```

而不是：

```csharp
task.Wait();
task.Result;
```

`Wait()` 和 `Result` 会阻塞当前线程，在 UI 程序和某些同步上下文里还可能引发死锁。

### 父子任务 AttachedToParent

`AttachedToParent` 可以让子任务附加到父任务。

父任务会等附加的子任务完成后才算完成。

```csharp
Task parent = Task.Factory.StartNew(() =>
{
    Console.WriteLine("父任务开始");

    Task.Factory.StartNew(() =>
    {
        Thread.Sleep(500);
        Console.WriteLine("子任务完成");
    }, TaskCreationOptions.AttachedToParent);

    Console.WriteLine("父任务委托结束");
});

await parent;

Console.WriteLine("父任务真正完成");
```

输出顺序通常类似：

```text
父任务开始
父任务委托结束
子任务完成
父任务真正完成
```

但要注意，`Task.Run` 默认带 `DenyChildAttach`。

也就是说，用 `Task.Run` 创建的父任务，子任务不能随便附加进去。

这能避免一些隐式父子关系带来的等待问题。

### 自定义 TaskFactory

如果一批任务都要使用相同配置，可以创建自己的 `TaskFactory`。

下面示例统一使用一个 `CancellationToken` 和 `TaskScheduler.Default`：

```csharp
using var source = new CancellationTokenSource();

var factory = new TaskFactory(
    source.Token,
    TaskCreationOptions.None,
    TaskContinuationOptions.None,
    TaskScheduler.Default);

Task<StoreReport>[] tasks = stores
    .Select(store => factory.StartNew(static state =>
    {
        var item = (StoreOrder)state!;

        return new StoreReport(
            item.StoreName,
            item.Amounts.Sum(),
            Environment.CurrentManagedThreadId);
    }, store))
    .ToArray();

StoreReport[] reports = await Task.WhenAll(tasks);

foreach (StoreReport item in reports)
{
    Console.WriteLine($"{item.StoreName}: {item.TotalAmount}");
}
```

这种写法的价值不是“少写几行代码”，而是把一批任务的默认策略固定下来。

### 常见错误

### 1. 用 StartNew 包异步 I/O

不推荐：

```csharp
await Task.Factory.StartNew(async () =>
{
    await File.ReadAllTextAsync("orders.txt");
});
```

推荐：

```csharp
string text = await File.ReadAllTextAsync("orders.txt");
```

异步 I/O 本身不需要额外占用线程池线程。

### 2. 忘记指定 TaskScheduler.Default

在某些上下文里：

```csharp
Task.Factory.StartNew(() => DoWork());
```

可能会使用当前调度器。

如果明确希望在线程池执行，可以写完整：

```csharp
Task.Factory.StartNew(
    () => DoWork(),
    CancellationToken.None,
    TaskCreationOptions.None,
    TaskScheduler.Default);
```

普通场景更简单：

```csharp
Task.Run(() => DoWork());
```

### 3. async 委托忘记 Unwrap

错误：

```csharp
Task<Task<string>> task = Task.Factory.StartNew(async () =>
{
    await Task.Delay(1000);
    return "ok";
});
```

正确：

```csharp
Task<string> task = Task.Factory.StartNew(async () =>
{
    await Task.Delay(1000);
    return "ok";
}).Unwrap();
```

更简单：

```csharp
Task<string> task = Task.Run(async () =>
{
    await Task.Delay(1000);
    return "ok";
});
```

### 4. 把 LongRunning 当成性能优化开关

`LongRunning` 不是“更快”开关。

它更适合长时间阻塞的粗粒度任务。

大量短任务都加 `LongRunning`，反而可能创建过多线程，增加上下文切换和内存开销。

### 5. 用 Result / Wait 阻塞异步代码

不推荐：

```csharp
var result = task.Result;
task.Wait();
```

推荐：

```csharp
var result = await task;
```

`await` 不会阻塞当前线程，异常处理也更自然。

### TaskFactory 适合哪些场景？

适合：

* 需要指定 `TaskCreationOptions`
* 需要指定 `TaskScheduler`
* 需要把旧式 `Begin/End` API 转成 `Task`
* 需要父子任务关系
* 需要统一创建一批任务的默认策略
* 维护早期 TPL 风格代码

不适合：

* 普通异步 I/O
* 普通后台计算
* 只是为了“让方法变异步”
* ASP.NET Core 请求里随手包同步代码
* 大量短小任务无脑 `StartNew`

### 总结

`TaskFactory` 不是过时 API，但它也不是日常异步编程的首选入口。

更合理的分工是：

```text
普通后台任务：Task.Run
等待多个任务：Task.WhenAll / Task.WhenAny
真正异步 I/O：直接 await 原生异步 API
需要精细控制调度：TaskFactory
```

`Task.Factory.StartNew` 的强大之处在于可配置。

但可配置也意味着更容易踩坑：

* 调度器可能不是预期的
* `async` 委托会产生嵌套 `Task`
* `LongRunning` 可能被滥用
* `Wait` / `Result` 可能阻塞线程
* 取消不会自动杀死正在执行的代码

掌握这些边界后，`TaskFactory` 更适合放在工具箱里处理特殊任务调度，而不是替代 `Task.Run` 成为默认写法。
