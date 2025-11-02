### 简介

`SemaphoreSlim` 是 `.NET` 中 `System.Threading` 命名空间提供的一个轻量级同步原语，用于限制对共享资源的并发访问。它是传统 `Semaphore` 类的轻量替代，专为高性能、异步场景设计，特别适合结合 `async/await` 的现代 `.NET` 应用（如 `ASP.NET Core`）。

在多线程或高并发应用中，共享资源的访问需要同步以避免竞争条件。传统的 `Semaphore` 类基于内核对象，性能开销较高，而 `SemaphoreSlim` 是一个用户态同步原语，专为以下场景设计：

* 限制并发访问：控制同时访问共享资源的线程数（如数据库连接、文件写入）。

* 异步友好：通过 `WaitAsync` 方法支持 `async/await`，适合异步编程。

* 轻量高效：相比 `Semaphore`，内存和性能开销更低。

* 跨平台支持：内置于 `.NET`，兼容所有 `.NET` 运行时。

解决的问题

| 问题    |  传统 Semaphore   |  SemaphoreSlim   |
| --- | --- | --- |
|  线程阻塞开销   |  高（内核模式）   |  低（用户模式旋转等待）   |
|  异步支持   | ❌ 不支持    |  ✅ 原生支持 async/await   |
|  内存占用   |  ~1KB 每实例   |  ~24 bytes   |
|  跨进程同步   |  ✅ 支持   |  ❌ 仅限进程内   |

### 核心功能

* 信号量计数：维护一个计数器，表示可用资源数。

* 限制并发：通过 `Wait` 或 `WaitAsync` 限制线程访问，`Release` 释放资源。

* 异步支持：提供 `WaitAsync` 方法，适合异步任务。

* 可取消性：通过 `CancellationToken` 支持取消等待。

* 动态调整：支持初始化和动态调整信号量计数。

* 线程安全：内置线程安全，适合多线程环境。

### 核心 API

`SemaphoreSlim` 位于 `System.Threading` 命名空间，核心 `API` 如下：

1. 构造函数

```csharp
// 初始化信号量，初始计数为 0，最大计数为 2
SemaphoreSlim semaphore = new SemaphoreSlim(0, 2);
```

* `SemaphoreSlim(int initialCount, int maxCount)`

    * `initialCount`：信号量的初始计数。

    * `maxCount`：信号量的最大计数。

2. `Wait()` 方法

`Wait()` 会请求一个信号量。如果当前没有可用的信号量，则线程会被阻塞直到信号量可用。

```csharp
semaphore.Wait();  // 请求信号量
// 执行受限资源的操作
```

3. `WaitAsync()` 方法

`WaitAsync()` 是异步方法，用于在异步上下文中请求信号量。

```csharp
await semaphore.WaitAsync();  // 异步请求信号量
// 执行异步操作
```

4. `Release()` 方法

`Release()` 用于释放一个信号量，增加信号量计数。通常在完成共享资源的访问后调用它。

```csharp
semaphore.Release();  // 释放一个信号量

semaphore.Release(3);  // 释放3个信号量
```

如果信号量的计数达到了最大值，`Release()` 会导致 `SemaphoreSlim` 的计数不会再增加。

5. `CurrentCount` 属性

`CurrentCount` 属性用于查看当前信号量的计数。

```csharp
int currentCount = semaphore.CurrentCount;  // 获取当前可用的信号量计数
```

6. 异常与超时处理

在信号量等待时，如果指定了超时，`Wait(int milliseconds)` 或 `WaitAsync(TimeSpan timeout)` 可以避免无限期阻塞：

```csharp
if (await semaphore.WaitAsync(TimeSpan.FromSeconds(5)))
{
    try
    {
        // 执行操作
    }
    finally
    {
        semaphore.Release();
    }
}
else
{
    // 超时逻辑
}
```

### 使用示例

#### 基本的线程限制

```csharp
public class SemaphoreExample
{
    private static SemaphoreSlim semaphore = new SemaphoreSlim(3, 3);  // 最多允许 3 个线程同时执行

    public static async Task Main(string[] args)
    {
        var tasks = new List<Task>();

        for (int i = 0; i < 10; i++)  // 10 个任务，但最多 3 个同时执行
        {
            int taskId = i;
            tasks.Add(Task.Run(async () =>
            {
                await semaphore.WaitAsync();  // 请求信号量
                try
                {
                    Console.WriteLine($"Task {taskId} started.");
                    await Task.Delay(1000);  // 模拟任务执行
                    Console.WriteLine($"Task {taskId} completed.");
                }
                finally
                {
                    semaphore.Release();  // 释放信号量
                }
            }));
        }

        await Task.WhenAll(tasks);  // 等待所有任务完成
    }
}
```

在此示例中，最多只有 3 个任务可以同时执行。`SemaphoreSlim` 控制并发任务的数量。

#### 处理数据库连接池

```csharp
public class DatabaseConnectionPool
{
    private static SemaphoreSlim semaphore = new SemaphoreSlim(5, 5);  // 最大允许 5 个并发连接

    public static async Task AccessDatabaseAsync()
    {
        await semaphore.WaitAsync();  // 请求连接
        try
        {
            Console.WriteLine($"Accessing database on thread {Thread.CurrentThread.ManagedThreadId}");
            await Task.Delay(2000);  // 模拟数据库操作
        }
        finally
        {
            semaphore.Release();  // 释放连接
        }
    }
}
```

这里，`SemaphoreSlim` 控制对数据库连接的并发访问，最多允许 5 个线程同时访问数据库。

#### API 请求限流

```csharp
class RateLimitedHttpClient
{
    private readonly HttpClient _client = new();
    private readonly SemaphoreSlim _throttle = new(5); // 最大5个并发请求

    public async Task<string> GetAsync(string url)
    {
        await _throttle.WaitAsync();
        try
        {
            return await _client.GetStringAsync(url);
        }
        finally
        {
            _throttle.Release();
        }
    }
}
```

### 高级功能

#### 超时与取消控制

```csharp
async Task<bool> TryAccessAsync(TimeSpan timeout, CancellationToken ct)
{
    // 带超时和取消的等待
    if (await semaphore.WaitAsync(timeout, ct))
    {
        try { /* 操作资源 */ }
        finally { semaphore.Release(); }
        return true;
    }
    return false; // 超时未获取
}
```

#### 批量获取与释放

```csharp
// 批量获取3个许可
await semaphore.WaitAsync(3); 

try
{
    // 执行需要多个许可的操作
    ProcessBulkData();
}
finally
{
    // 批量释放3个许可
    semaphore.Release(3); 
}
```

#### 资源池实现

```csharp
class ResourcePool<T> where T : IDisposable
{
    private readonly ConcurrentQueue<T> _resources = new();
    private readonly SemaphoreSlim _semaphore;

    public ResourcePool(IEnumerable<T> resources)
    {
        foreach (var res in resources) _resources.Enqueue(res);
        _semaphore = new SemaphoreSlim(_resources.Count, _resources.Count);
    }

    public async Task<ResourceLease> AcquireAsync(CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        _resources.TryDequeue(out var resource);
        return new ResourceLease(resource, this);
    }

    private void Release(T resource)
    {
        _resources.Enqueue(resource);
        _semaphore.Release();
    }

    public readonly struct ResourceLease : IDisposable
    {
        private readonly ResourcePool<T> _pool;
        public T Resource { get; }

        public ResourceLease(T resource, ResourcePool<T> pool)
        {
            Resource = resource;
            _pool = pool;
        }

        public void Dispose() => _pool.Release(Resource);
    }
}
```

#### 避免常见陷阱

```csharp
// ❌ 错误：忘记释放
semaphore.Wait();
DoWork(); // 若异常则永远不释放

// ✅ 正确：使用using模式
public struct SemaphoreGuard : IDisposable
{
    private SemaphoreSlim _semaphore;
    public SemaphoreGuard(SemaphoreSlim semaphore)
    {
        _semaphore = semaphore;
        semaphore.Wait();
    }
    public void Dispose() => _semaphore?.Release();
}

using (new SemaphoreGuard(semaphore))
{
    // 受保护操作
}
```

#### 与Channel结合实现生产者-消费者

```csharp
var channel = Channel.CreateBounded<int>(capacity: 100);
var semaphore = new SemaphoreSlim(0, 100); // 初始无许可

// 生产者
public async Task ProduceAsync(int item)
{
    await channel.Writer.WriteAsync(item);
    semaphore.Release(); // 增加可用许可
}

// 消费者
public async Task ConsumeAsync()
{
    await semaphore.WaitAsync(); // 等待数据
    var item = await channel.Reader.ReadAsync();
    ProcessItem(item);
}
```

### 性能优化

* 适合线程内同步

`SemaphoreSlim` 是轻量级的，仅在同一个进程内工作，如果需要跨进程的同步，可以使用 `Semaphore` 类。

* 最小化信号量请求与释放的频率

每次调用 `Wait()` 和 `Release()` 都有一定的开销。在高并发场景中，频繁地请求和释放信号量可能会影响性能，因此尽量将资源使用放在一个操作内进行。

* 避免死锁

使用 `SemaphoreSlim` 时，要确保所有 `Wait()` 操作都有对应的 `Release()` 调用，避免死锁。

* 超时机制

为了避免某些线程永远等待信号量，考虑为 `WaitAsync` 和 `Wait` 设置超时，或者在等待时使用 `CancellationToken` 来支持取消操作。

### 常见使用场景

* 数据库连接池
控制并发访问数据库的连接数，防止连接池耗尽。

* 并发访问共享资源
限制同时访问某些资源（例如文件、缓存、API）的线程数。

* 限制高并发请求
在高并发场景下，限制客户端请求数，防止系统过载。

* 任务调度与执行
控制任务的并发执行数量，保证系统负载均衡。

* 信号量与生产者消费者模型
使用 `SemaphoreSlim` 配合队列实现生产者消费者模型，限制队列中同时处理的任务数。

```mermaid
graph LR
    A[客户端请求] --> B{信号量可用？}
    B -->|是| C[处理请求]
    B -->|否| D[加入等待队列]
    C --> E[释放信号量]
    D --> F[超时/取消处理]
    E --> G[通知下一个请求]
```

### 与相关同步原语对比

|  特性   |  SemaphoreSlim   |  Semaphore   |  Mutex   |  Monitor   |  ReaderWriterLockSlim   |
| --- | --- | --- | --- | --- | --- |
| 跨进程    |  ❌   |  ✅   |  ✅   |  ❌   |  ❌   |
|  递归获取   |  ❌   |  ❌   | ✅    |   ✅  |  ✅   |
|  异步支持|  ✅|   ❌  |  ❌   |  ❌   |  ❌   |
|  读写分离   |  ❌   |  ❌   |  ❌|  ❌   |  ✅   |
|  轻量级   |  ✅   |   ❌  |   ❌  |  ✅   |  ❌   |
|  超时支持   |  ✅   |  ✅   |  ✅   |  ✅   |  ✅   |

### 总结

`SemaphoreSlim` 是一种高效的线程同步原语，能够有效控制并发线程数，广泛应用于限制资源访问、调度任务执行等场景。相比于 `Semaphore`，它在线程内的同步场景中更加轻量、性能更好。在使用时，应注意避免死锁、合理设置超时以及优化信号量的请求与释放频率，以确保系统性能与稳定性。

### 资源和文档

* 官方文档：

    * `Microsoft Learn`：https://learn.microsoft.com/en-us/dotnet/api/system.threading.semaphoreslim

    * `.NET Threading`：https://learn.microsoft.com/en-us/dotnet/standard/threading

* `GitHub`：https://github.com/dotnet/runtime