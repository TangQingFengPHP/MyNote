### 简介

`Thread` 是 .NET 里直接创建和管理线程的底层 API。

命名空间：

```csharp
using System.Threading;
```

最简单的写法：

```csharp
Thread thread = new(() =>
{
    Console.WriteLine("工作线程正在执行");
});

thread.Start();
```

一句话概括：

```text
Thread 表示一条独立执行路径，可以直接控制线程的启动、名称、前后台属性、优先级和等待过程。
```

不过，现代 .NET 项目里并不需要到处 `new Thread()`。

常见选择应该是：

```text
异步 I/O：async / await
普通后台计算：Task.Run
短小并发任务：Task / ThreadPool
长期独占、阻塞式工作线程：Thread
```

`Thread` 更底层，也更难管理。只有确实需要专用线程时，它的价值才真正体现出来。

### 线程到底是什么？

一个正在运行的进程里，可以有多条线程。

例如一个桌面程序可能同时存在：

```text
UI 线程
网络通信线程
日志写入线程
后台计算线程
运行时内部线程
```

这些线程共享同一个进程里的大部分资源：

* 堆内存
* 静态字段
* 打开的文件
* 数据库连接对象
* 进程级配置

但每条线程也有自己的执行状态：

* 调用栈
* 当前执行位置
* 寄存器上下文
* 线程局部数据

现代 .NET 运行时中的托管线程，通常由操作系统线程承载。操作系统负责调度它们在哪个 CPU 核心上运行。

### 并发不等于并行

创建两个线程，只能说明有两条执行路径，不代表它们一定同时占用两个 CPU 核心。

```text
并发：多个任务在一段时间内交替推进
并行：多个任务在同一时刻真正同时执行
```

是否能并行，取决于：

* CPU 核心数量
* 操作系统调度
* 线程是否正在阻塞
* 进程当前负载
* 运行环境限制

所以：

```text
线程多，不代表执行一定更快。
```

线程过多还会带来：

* 线程栈内存开销
* 上下文切换
* 锁竞争
* 缓存失效
* 调度延迟

### Thread、Task、ThreadPool 和 async 的区别

这几个概念经常混在一起。

| 技术 | 核心作用 | 是否直接代表线程 |
| --- | --- | --- |
| `Thread` | 创建和控制专用线程 | 是 |
| `ThreadPool` | 复用一组工作线程 | 是线程池 |
| `Task` | 表示一个工作的完成状态 | 不一定 |
| `async/await` | 编排异步流程 | 不会自动创建线程 |

`Task` 不等于线程。

例如：

```csharp
await Task.Delay(1000);
```

等待期间不需要专门占着一条线程睡觉。

而：

```csharp
Thread.Sleep(1000);
```

会让当前线程真实阻塞一秒。

### 什么情况下使用 Thread？

适合：

* 需要长期存在的专用工作线程
* 必须调用阻塞式老 API
* 需要设置线程优先级
* 需要设置 STA / MTA 单元状态
* 需要明确控制前台线程和后台线程
* 与某些原生库、COM 组件或线程亲和资源集成
* 线程必须长期维护自己的局部状态

不适合：

* 普通 HTTP 请求
* 数据库异步查询
* 文件异步读写
* 大量短小任务
* ASP.NET Core 里给每个请求创建线程
* 只是为了让方法看起来“异步”

### Thread 常用成员

| 成员 | 作用 |
| --- | --- |
| `Start()` | 启动线程 |
| `Join()` | 阻塞当前线程，等待目标线程结束 |
| `Sleep()` | 阻塞当前线程一段时间 |
| `Interrupt()` | 中断处于等待、休眠或 Join 状态的线程 |
| `Yield()` | 提示调度器让出当前时间片 |
| `CurrentThread` | 获取当前线程 |
| `IsAlive` | 判断线程是否仍在运行 |
| `IsBackground` | 设置前台或后台线程 |
| `Name` | 设置线程名称 |
| `Priority` | 设置调度优先级提示 |
| `ManagedThreadId` | 获取托管线程 ID |
| `ThreadState` | 查看线程状态 |

### Demo 目标

下面通过多个示例讲清楚：

```text
创建线程
传递参数
获取结果
等待线程
前后台线程
协作取消
异常处理
共享数据同步
实现专用后台工作线程
```

创建控制台项目：

```bash
mkdir ThreadDemo
cd ThreadDemo

dotnet new console
```

### 创建并启动线程

```csharp
using System;
using System.Threading;

Console.WriteLine($"主线程 ID：{Environment.CurrentManagedThreadId}");

Thread worker = new(() =>
{
    Console.WriteLine($"工作线程 ID：{Environment.CurrentManagedThreadId}");
    Console.WriteLine("开始处理订单");
});

worker.Name = "OrderWorker";
worker.Start();

Console.WriteLine("主线程继续执行");

worker.Join();

Console.WriteLine("工作线程已经结束");
```

输出顺序可能类似：

```text
主线程 ID：1
主线程继续执行
工作线程 ID：4
开始处理订单
工作线程已经结束
```

工作线程和主线程会并发执行，因此中间几行的顺序不固定。

### 创建线程不等于启动线程

这段代码只创建线程对象：

```csharp
Thread worker = new(() => Console.WriteLine("执行任务"));
```

只有调用 `Start()` 后，线程才会开始运行：

```csharp
worker.Start();
```

同一个 `Thread` 实例只能启动一次。

错误示例：

```csharp
worker.Start();
worker.Join();
worker.Start();
```

第二次调用 `Start()` 会抛出 `ThreadStateException`。

线程结束后如果还要重新执行，需要创建新的 `Thread` 实例。

### 使用 Lambda 传递参数

推荐直接通过 Lambda 捕获强类型参数：

```csharp
string orderNo = "SO202606140001";
decimal amount = 199.80m;

Thread worker = new(() =>
{
    Console.WriteLine($"处理订单：{orderNo}");
    Console.WriteLine($"订单金额：{amount}");
});

worker.Start();
worker.Join();
```

这种写法比 `ParameterizedThreadStart` 更清楚，因为参数类型在编译期就能检查。

需要注意闭包变量可能在子线程读取前被修改：

```csharp
int taskId = 1;

Thread worker = new(() => Console.WriteLine(taskId));

taskId = 2;
worker.Start();
```

输出通常是：

```text
2
```

Lambda 捕获的是变量，不是创建线程那一刻的值。

需要固定值时，可以先复制：

```csharp
int taskId = 1;
int capturedTaskId = taskId;

Thread worker = new(() => Console.WriteLine(capturedTaskId));
```

### ParameterizedThreadStart

`Thread` 也支持一个 `object?` 参数：

```csharp
Thread worker = new(static state =>
{
    var args = (OrderArgs)state!;
    Console.WriteLine($"处理订单：{args.OrderNo}，金额：{args.Amount}");
});

worker.Start(new OrderArgs("SO202606140002", 299.00m));
worker.Join();

public sealed record OrderArgs(string OrderNo, decimal Amount);
```

这种写法存在运行时类型转换。

普通业务代码优先使用强类型 Lambda。

### Thread 如何返回结果？

`Thread` 没有 `Thread<T>`，也没有类似 `Task<T>` 的直接返回值。

通常需要通过共享变量接收结果：

```csharp
int result = 0;

Thread worker = new(() =>
{
    result = Enumerable.Range(1, 100).Sum();
});

worker.Start();
worker.Join();

Console.WriteLine(result);
```

这里必须先 `Join()`，再读取 `result`。

`Join()` 不只是等待线程结束，也建立了必要的同步关系，确保目标线程结束前完成的写入对当前线程可见。

如果任务天然需要返回值、取消和异常传播，`Task<T>` 通常更合适：

```csharp
int result = await Task.Run(() => Enumerable.Range(1, 100).Sum());
```

### Join 等待线程结束

`Join()` 会阻塞调用它的线程：

```csharp
Thread worker = new(() =>
{
    Thread.Sleep(1500);
    Console.WriteLine("任务完成");
});

worker.Start();

Console.WriteLine("开始等待");
worker.Join();
Console.WriteLine("等待结束");
```

也可以设置超时：

```csharp
bool completed = worker.Join(TimeSpan.FromMilliseconds(500));

if (!completed)
{
    Console.WriteLine("等待超时，线程仍在运行");
}
```

注意：

```text
Join 超时只是不再等待，不会停止目标线程。
```

### 前台线程和后台线程

直接创建的 `Thread` 默认是前台线程：

```csharp
worker.IsBackground = false;
```

只要还有前台线程没有结束，进程通常就不会退出。

后台线程：

```csharp
worker.IsBackground = true;
```

当所有前台线程都结束后，进程可以直接退出，不会等待后台线程完整收尾。

例如：

```csharp
Thread background = new(() =>
{
    Thread.Sleep(2000);
    Console.WriteLine("后台线程完成");
})
{
    IsBackground = true
};

background.Start();

Console.WriteLine("主线程结束");
```

`后台线程完成` 可能不会输出。

所以后台线程不能依赖这些收尾动作：

* 最后一次日志写入
* 文件刷新
* 数据提交
* 网络消息发送
* 释放关键业务资源

重要任务应该提供明确的停止流程，并在退出前调用 `Join()`。

### Thread.Sleep 和 Task.Delay 的区别

```csharp
Thread.Sleep(1000);
```

会阻塞当前线程。

这一秒内，线程不能执行其他工作。

```csharp
await Task.Delay(1000);
```

表示异步等待，等待期间不会专门占用一条线程。

| 对比项 | `Thread.Sleep` | `Task.Delay` |
| --- | --- | --- |
| 是否阻塞线程 | 是 | 否 |
| 能否 `await` | 否 | 是 |
| 常见场景 | 专用线程轮询、测试、简单退避 | 异步流程、服务端请求、UI |

在异步方法里不要用：

```csharp
Thread.Sleep(1000);
```

应该使用：

```csharp
await Task.Delay(1000);
```

### 使用 CancellationToken 协作取消

.NET 不推荐强行杀死线程。

更合理的做法是发送取消信号，让线程自己结束。

```csharp
using var cts = new CancellationTokenSource();

Thread worker = new(() =>
{
    CancellationToken token = cts.Token;

    while (!token.IsCancellationRequested)
    {
        Console.WriteLine($"{DateTime.Now:HH:mm:ss} 检查新订单");

        if (token.WaitHandle.WaitOne(TimeSpan.FromMilliseconds(500)))
        {
            break;
        }
    }

    Console.WriteLine("工作线程正常退出");
});

worker.Start();

Thread.Sleep(1600);
cts.Cancel();

worker.Join();
```

这里没有直接使用：

```csharp
Thread.Sleep(500);
```

而是使用：

```csharp
token.WaitHandle.WaitOne(500)
```

好处是取消信号到达后，等待可以立刻结束，不必等满 500 毫秒。

### Interrupt 中断等待状态

`Interrupt()` 可以中断处于这些状态的线程：

* `Sleep`
* `Wait`
* `Join`

被中断的线程会抛出 `ThreadInterruptedException`：

```csharp
Thread worker = new(() =>
{
    try
    {
        Console.WriteLine("线程进入等待");
        Thread.Sleep(Timeout.Infinite);
    }
    catch (ThreadInterruptedException)
    {
        Console.WriteLine("线程等待被中断");
    }
});

worker.Start();

Thread.Sleep(500);
worker.Interrupt();
worker.Join();
```

`Interrupt()` 不是安全的“终止线程”方法。

如果线程正在运行普通计算代码，中断不会立即终止计算。它主要影响线程进入等待状态的过程。

业务代码通常优先使用 `CancellationToken`。

### Thread.Abort 为什么不能用？

`Thread.Abort()` 试图在任意位置强行终止线程。

这会产生严重问题：

* 锁可能没有释放
* 数据可能只写了一半
* 对象可能处于损坏状态
* `finally` 和资源清理难以可靠推断

在现代 .NET 中，`Thread.Abort()` 不受支持，调用通常会抛出 `PlatformNotSupportedException`。

`Suspend()` 和 `Resume()` 也不应该用于现代代码。

线程停止应该采用：

```text
发送取消信号
线程主动退出循环
执行 finally 清理
调用 Join 等待结束
```

### 线程异常不会自动传回主线程

看这段代码：

```csharp
Thread worker = new(() =>
{
    throw new InvalidOperationException("订单处理失败");
});

worker.Start();
worker.Join();
```

工作线程里的异常不会因为 `Join()` 自动在主线程重新抛出。

未处理异常还可能直接终止整个进程。

需要在线程入口处捕获：

```csharp
Exception? workerException = null;

Thread worker = new(() =>
{
    try
    {
        throw new InvalidOperationException("订单处理失败");
    }
    catch (Exception ex)
    {
        workerException = ex;
    }
});

worker.Start();
worker.Join();

if (workerException is not null)
{
    Console.WriteLine(workerException.Message);
}
```

相比之下，`Task` 会把异常保存在任务中，`await` 时可以自然传播：

```csharp
await Task.Run(() => throw new InvalidOperationException("订单处理失败"));
```

这也是普通后台任务更适合使用 `Task` 的原因之一。

### 共享变量为什么会出错？

下面启动两个线程，每个线程执行十万次自增：

```csharp
int count = 0;

Thread t1 = new(() =>
{
    for (int i = 0; i < 100_000; i++)
    {
        count++;
    }
});

Thread t2 = new(() =>
{
    for (int i = 0; i < 100_000; i++)
    {
        count++;
    }
});

t1.Start();
t2.Start();
t1.Join();
t2.Join();

Console.WriteLine(count);
```

预期结果是：

```text
200000
```

实际结果可能更小。

因为 `count++` 不是一个不可分割的动作，它包含：

```text
读取 count
加一
写回 count
```

两个线程可能同时读到旧值，然后互相覆盖。

### 使用 Interlocked 修复计数器

简单原子计数可以使用：

```csharp
Interlocked.Increment(ref count);
```

完整写法：

```csharp
int count = 0;

Thread t1 = new(Increment);
Thread t2 = new(Increment);

t1.Start();
t2.Start();
t1.Join();
t2.Join();

Console.WriteLine(count);

void Increment()
{
    for (int i = 0; i < 100_000; i++)
    {
        Interlocked.Increment(ref count);
    }
}
```

### 使用 lock 保护复合操作

如果临界区不只是一个简单自增，就应该使用锁：

```csharp
object gate = new();
decimal balance = 1000m;

void Withdraw(decimal amount)
{
    lock (gate)
    {
        if (balance < amount)
        {
            return;
        }

        balance -= amount;
    }
}
```

锁对象建议满足这些条件：

* 私有
* 只读引用
* 专门用于同步

类字段通常写成：

```csharp
private readonly object _gate = new();
```

不要锁这些对象：

```csharp
lock (this)
lock (typeof(SomeType))
lock ("固定字符串")
```

它们可能被外部代码共享，容易产生意料之外的锁竞争或死锁。

### volatile 能代替 lock 吗？

不能。

`volatile` 主要影响内存读写的可见性和顺序，不会让复合操作自动变成原子操作。

下面依然不安全：

```csharp
private volatile int _count;

_count++;
```

简单停止标记可以考虑 `volatile`，复杂状态更新仍然需要 `lock`、`Interlocked` 或其他同步原语。

### 线程名称和优先级

设置名称有助于日志和调试：

```csharp
Thread worker = new(ProcessOrders)
{
    Name = "OrderWorker"
};
```

当前线程名称：

```csharp
Console.WriteLine(Thread.CurrentThread.Name);
```

线程名称通常只能设置一次。重复设置可能抛出 `InvalidOperationException`。

优先级：

```csharp
worker.Priority = ThreadPriority.AboveNormal;
```

可选值：

```text
Lowest
BelowNormal
Normal
AboveNormal
Highest
```

优先级只是给操作系统调度器的提示，不保证高优先级线程一定先执行，也不应该用它实现业务顺序。

大多数项目保持默认 `Normal` 即可。

### ThreadState 只能用于观察

可以读取：

```csharp
Console.WriteLine(worker.ThreadState);
```

常见状态包括：

* `Unstarted`
* `Running`
* `WaitSleepJoin`
* `Stopped`
* `Background`

但 `ThreadState` 是一个瞬时快照。

读取完之后，线程状态可能马上变化，因此不适合用它编写关键同步逻辑。

需要协调线程时，应该使用：

* `Join`
* `CancellationToken`
* `ManualResetEventSlim`
* `AutoResetEvent`
* `lock`
* `SemaphoreSlim`

### 不要把 async Lambda 直接交给 Thread

下面的写法很危险：

```csharp
Thread worker = new(async () =>
{
    await Task.Delay(1000);
    throw new InvalidOperationException("处理失败");
});
```

`ThreadStart` 要求返回 `void`。

所以这个异步 Lambda 会变成 `async void`：

```text
Thread 只负责执行到第一个未完成的 await
await 后续逻辑不再由这条专用线程保证
异常也无法通过 Task 观察
```

需要异步流程时，直接使用：

```csharp
Task worker = Task.Run(async () =>
{
    await Task.Delay(1000);
});
```

专用 `Thread` 更适合同步、阻塞式循环。

### 实战 Demo：专用订单工作线程

下面实现一个完整的后台工作线程。

它负责：

```text
接收订单任务
按顺序处理
捕获单个任务异常
停止接收新任务
处理完队列后退出
主线程等待它正常收尾
```

使用 `BlockingCollection<T>` 作为阻塞队列：

```csharp
using System.Collections.Concurrent;

public sealed class OrderWorker : IDisposable
{
    private readonly BlockingCollection<OrderJob> _queue = new();
    private readonly Thread _thread;
    private bool _started;
    private bool _stopped;
    private bool _disposed;

    public OrderWorker()
    {
        _thread = new Thread(Run)
        {
            Name = "OrderWorker",
            IsBackground = true
        };
    }

    public void Start()
    {
        ThrowIfDisposed();

        if (_started)
        {
            throw new InvalidOperationException("工作线程不能重复启动");
        }

        if (_stopped)
        {
            throw new InvalidOperationException("已经停止的工作线程不能重新启动");
        }

        _started = true;
        _thread.Start();
    }

    public void Enqueue(OrderJob job)
    {
        ThrowIfDisposed();

        if (_stopped || _queue.IsAddingCompleted)
        {
            throw new InvalidOperationException("工作线程已经停止接收任务");
        }

        _queue.Add(job);
    }

    public void Stop()
    {
        if (_disposed || _stopped)
        {
            return;
        }

        _stopped = true;
        _queue.CompleteAdding();

        if (_started)
        {
            _thread.Join();
        }
    }

    private void Run()
    {
        Console.WriteLine($"工作线程启动，ID：{Environment.CurrentManagedThreadId}");

        foreach (OrderJob job in _queue.GetConsumingEnumerable())
        {
            try
            {
                Process(job);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"订单 {job.OrderNo} 处理失败：{ex.Message}");
            }
        }

        Console.WriteLine("订单队列处理完成，工作线程退出");
    }

    private static void Process(OrderJob job)
    {
        Console.WriteLine($"开始处理订单：{job.OrderNo}，金额：{job.Amount}");

        Thread.Sleep(300);

        if (job.Amount < 0)
        {
            throw new InvalidOperationException("订单金额不能小于 0");
        }

        Console.WriteLine($"订单处理完成：{job.OrderNo}");
    }

    public void Dispose()
    {
        if (_disposed)
        {
            return;
        }

        Stop();
        _queue.Dispose();
        _disposed = true;
    }

    private void ThrowIfDisposed()
    {
        if (_disposed)
        {
            throw new ObjectDisposedException(nameof(OrderWorker));
        }
    }
}

public sealed record OrderJob(string OrderNo, decimal Amount);
```

调用代码：

```csharp
using var worker = new OrderWorker();

worker.Start();

worker.Enqueue(new OrderJob("SO001", 99.00m));
worker.Enqueue(new OrderJob("SO002", 268.00m));
worker.Enqueue(new OrderJob("SO003", -1.00m));
worker.Enqueue(new OrderJob("SO004", 520.00m));

worker.Stop();
```

输出类似：

```text
工作线程启动，ID：4
开始处理订单：SO001，金额：99.00
订单处理完成：SO001
开始处理订单：SO002，金额：268.00
订单处理完成：SO002
开始处理订单：SO003，金额：-1.00
订单 SO003 处理失败：订单金额不能小于 0
开始处理订单：SO004，金额：520.00
订单处理完成：SO004
订单队列处理完成，工作线程退出
```

这个 Demo 展示了专用线程比较合理的使用方式：

* 线程长期存在
* 工作是同步阻塞式的
* 队列负责等待和唤醒
* 停止过程明确
* 退出前调用 `Join()`
* 单个任务异常不会打崩整个工作线程

示例默认由一个组件统一负责 `Start()`、`Stop()` 和 `Dispose()`。如果多个线程可能同时控制生命周期，还需要用锁保护 `_started`、`_stopped` 和 `_disposed` 状态。

如果任务处理本身是异步 I/O，更适合改用：

```text
Channel<T> + BackgroundService + async/await
```

### OrderWorker 还能怎么改进？

上面的 Demo 为了突出线程模型，保持了较简单的结构。

实际项目还可以增加：

* 有界队列，避免任务无限堆积
* 取消超时
* 重试策略
* 死信队列
* 处理耗时指标
* 更完善的日志
* 启动和停止状态检查
* 多调用方并发启停保护

有界队列示例：

```csharp
private readonly BlockingCollection<OrderJob> _queue = new(boundedCapacity: 1000);
```

队列满时，`Add()` 会阻塞生产者，形成背压。

### STA 线程是什么？

某些 Windows 技术要求线程运行在单线程单元中，例如部分：

* COM 组件
* 剪贴板操作
* OLE
* WPF / WinForms 相关能力

可以在线程启动前设置：

```csharp
Thread thread = new(() =>
{
    // Windows STA 工作
});

thread.SetApartmentState(ApartmentState.STA);
thread.Start();
thread.Join();
```

这属于平台相关能力，不适合跨平台通用业务逻辑。

### 常见错误

### 1. 每个请求创建一条线程

错误思路：

```csharp
app.MapGet("/orders", () =>
{
    new Thread(ProcessOrders).Start();
    return "ok";
});
```

请求一多，就会创建大量线程，带来严重调度和内存压力。

ASP.NET Core 中应该优先使用异步 API、后台队列和托管服务。

### 2. 用 Sleep 等待异步 I/O

错误：

```csharp
while (!completed)
{
    Thread.Sleep(100);
}
```

这会浪费线程，还可能出现数据可见性问题。

应该使用任务、事件、信号量或异步 API 等明确的通知机制。

### 3. 依赖后台线程完成关键工作

后台线程可能随进程退出直接终止。

关键数据必须有明确的停止、刷新和等待流程。

### 4. 忽略线程入口异常

每条手工线程都应该有清楚的异常边界。

线程入口处没有捕获的异常可能终止整个进程。

### 5. 创建太多专用线程

`Thread` 不会像线程池一样自动复用。

大量短任务应交给：

* `Task.Run`
* `ThreadPool`
* `Parallel`
* PLINQ

### 6. 用 Priority 控制业务顺序

线程优先级不是任务排序工具。

需要先后顺序时，应该使用队列、锁、信号量或任务依赖。

### Thread 的合理使用边界

可以用 `Thread`：

```text
长期阻塞的专用循环
需要固定线程属性
需要 STA
原生组件要求固定线程
维护传统同步线程模型
```

优先使用 `Task`：

```text
需要返回结果
需要自然传播异常
需要组合多个任务
短期 CPU 计算
大多数后台工作
```

优先使用 `async/await`：

```text
HTTP 调用
数据库访问
文件 I/O
消息队列异步客户端
定时等待
```

### 总结

`Thread` 是 .NET 并发体系里最直接的线程控制工具。

它提供了：

* 独立线程生命周期
* 前台和后台线程控制
* 名称和优先级设置
* `Join` 阻塞等待
* `Interrupt` 中断等待状态
* STA / MTA 单元配置

但它没有 `Task` 那些更现代的能力：

* 没有直接返回值
* 没有自然的异常传播
* 没有任务组合
* 没有自动线程复用
* 没有内置的协作取消模型

实战里的选择可以归纳为：

```text
真正需要专用线程时使用 Thread。
普通并发任务交给 Task 和线程池。
异步 I/O 直接使用 async/await。
```

掌握 `Thread` 的重点不是创建更多线程，而是理解线程的生命周期、共享状态、退出方式和使用边界。
