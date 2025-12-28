### 简介

`Parallel` 并行编程是 `.NET` 中利用多核 `CPU` 进行并发执行的编程模型，主要通过 `System.Threading.Tasks` 命名空间中的 `Parallel` 类实现。它允许将任务分解成多个子任务，在多个线程上同时执行，以加速 `CPU` 密集型操作（如循环计算、数据处理）。

#### 核心组件：

* `Parallel` 类：提供静态方法如 `Parallel.For、Parallel.ForEach、Parallel.Invoke`，用于并行执行循环或方法。

* `PLINQ`（Parallel LINQ）：`LINQ` 的并行版本，通过 `AsParallel()` 扩展方法启用并行查询。

* 底层依赖：基于 Task Parallel Library (TPL)，利用线程池（`ThreadPool`）管理线程，避免手动创建线程的开销。

#### 关键概念：

* 并行度（`Degree of Parallelism`）：控制并发线程数，默认基于 CPU 核心（e.g., 4 核 CPU 可能 4 线程）。

* 数据并行：将数据分成块（`chunks`），每个线程处理一块（e.g., `Parallel.For`）。

* 任务并行：同时执行独立方法（e.g., `Parallel.Invoke`）。

### Parallel 的核心定位与价值

`Parallel` 类位于 `System.Threading.Tasks` 命名空间，是 `.NET` 提供的 “高层级并行工具”，核心价值：

* 自动线程调度：无需手动创建 / 管理线程，由 `TPL` 自动分配线程池线程，实现负载均衡；

* 简化并行逻辑：用类似串行循环的语法实现并行执行，降低并行编程门槛；

* 适配多核 `CPU`：默认根据 `CPU` 核心数调整并行度，最大化利用硬件资源；

* 支持取消 / 超时：可通过 `ParallelOptions` 控制并行过程，如取消执行、限制并行数。

> Parallel仅适合 CPU 密集型任务（如数据计算、图像处理、复杂逻辑运算）；I/O 密集型任务（文件读写、网络请求）应使用async/await，否则线程会阻塞，浪费资源。

### 核心概念与基础

#### 并行 vs. 并发 vs. 异步:

* 并发: 多个任务在重叠的时间段内执行（不一定同时）。单核上通过时间片切换实现。

* 并行: 多个任务真正同时执行，需要多核或多处理器支持。是并发的一种特例。

* 异步: 一种编程模式，允许启动一个操作后不阻塞当前线程，待操作完成后再处理结果。异步操作可以利用并发或并行（通常通过线程池），但其核心目标是非阻塞和响应性。

#### 线程 vs. 任务:

* 线程: 操作系统调度的基本单位。直接操作 `Thread` 类相对底层，需要手动管理生命周期、同步等，易出错且开销较大。

* 任务: `TPL` 引入的核心概念 `Task` 和 `Task<TResult>`。代表一个异步操作单元。任务通常由线程池线程执行，但提供了更高级别的抽象：

    * 任务调度: 由 `TaskScheduler` 管理，默认使用线程池。

    * 组合与延续: 使用 `ContinueWith, WhenAll, WhenAny` 等轻松组合任务。

    * 异常传播: 异常被封装在 `AggregateException` 中，便于统一处理。

    * 取消支持: 与 `CancellationTokenSource/CancellationToken` 集成。

    * 状态跟踪: 有 `Created, Running, RanToCompletion, Canceled, Faulted` 等状态。

* 线程池:

    * `.NET` 维护一个全局的工作线程池。

    * `TPL` 默认使用线程池来执行任务。

    * 优点：避免频繁创建销毁线程的开销，自动管理线程数量（根据负载增减）。
    
    * 使用 `ThreadPool.QueueUserWorkItem` 或 `Task.Run/Task.Factory.StartNew` 提交工作项。

### Parallel 核心 API 一览

| API              | 用途                |
| ---------------- | ------------------- |
| Parallel.For     | 并行 for 循环       |
| Parallel.ForEach | 并行 foreach        |
| Parallel.Invoke  | 并行执行多个 Action |
| ParallelOptions  | 控制并行度、取消等  |

#### Parallel.Invoke：并行执行多个独立任务

用于一次性并行执行多个无关联的方法（任务），适合 “多任务并行执行，等待全部完成” 的场景。

语法：

```csharp
public static void Invoke(params Action[] actions);
public static void Invoke(ParallelOptions options, params Action[] actions);
```

示例：并行执行三个独立的 `CPU` 密集型方法

```csharp
using System;
using System.Threading.Tasks;

class ParallelInvokeDemo
{
    static void Main()
    {
        // 记录开始时间
        var watch = System.Diagnostics.Stopwatch.StartNew();

        // 并行执行三个方法
        Parallel.Invoke(
            () => CalculateSum(1, 100000000), // 任务1：计算1~1亿的和
            () => CalculatePrimeCount(1, 100000), // 任务2：统计1~10万的质数数量
            () => GenerateRandomData(1000000) // 任务3：生成100万条随机数据
        );

        watch.Stop();
        Console.WriteLine($"并行执行耗时：{watch.ElapsedMilliseconds}ms");

        // 对比：串行执行（耗时远高于并行）
        watch.Restart();
        CalculateSum(1, 100000000);
        CalculatePrimeCount(1, 100000);
        GenerateRandomData(1000000);
        watch.Stop();
        Console.WriteLine($"串行执行耗时：{watch.ElapsedMilliseconds}ms");
    }

    // 模拟CPU密集型任务1：计算累加和
    static void CalculateSum(int start, int end)
    {
        long sum = 0;
        for (long i = start; i <= end; i++) sum += i;
        Console.WriteLine($"累加和：{sum}");
    }

    // 模拟CPU密集型任务2：统计质数数量
    static void CalculatePrimeCount(int start, int end)
    {
        int count = 0;
        for (int i = start; i <= end; i++)
        {
            if (IsPrime(i)) count++;
        }
        Console.WriteLine($"质数数量：{count}");
    }

    // 模拟CPU密集型任务3：生成随机数据
    static void GenerateRandomData(int count)
    {
        var random = new Random();
        double[] data = new double[count];
        for (int i = 0; i < count; i++) data[i] = random.NextDouble();
        Console.WriteLine($"随机数据生成完成，长度：{data.Length}");
    }

    // 辅助方法：判断是否为质数
    static bool IsPrime(int num)
    {
        if (num < 2) return false;
        for (int i = 2; i <= Math.Sqrt(num); i++)
        {
            if (num % i == 0) return false;
        }
        return true;
    }
}
```

关键说明：

* `Parallel.Invoke` 会等待所有传入的 `Action` 执行完成后才返回；

* 方法执行顺序不保证（由 `TPL` 调度），但最终会全部执行；

* 若其中一个方法抛出异常，其他方法仍会继续执行，最终所有异常会被包装为`AggregateException` 抛出。

#### Parallel.For：并行执行 for 循环

替代传统的 `for` 循环，将循环迭代分配到多个线程并行执行，适合 “固定次数的循环，迭代间无依赖” 的场景。

核心语法：

```csharp
// 基础版：从fromInclusive到toExclusive（不包含）的并行循环
public static ParallelLoopResult For(
    int fromInclusive, 
    int toExclusive, 
    Action<int> body
);

// 带配置版：支持取消、限制并行度
public static ParallelLoopResult For(
    int fromInclusive, 
    int toExclusive, 
    ParallelOptions options, 
    Action<int> body
);
```

示例：并行累加数组元素（线程安全版）

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class ParallelForDemo
{
    static void Main()
    {
        // 初始化1000万个元素的数组
        int[] numbers = new int[10_000_000];
        Random random = new Random();
        for (int i = 0; i < numbers.Length; i++) numbers[i] = random.Next(1, 100);

        long total = 0; // 共享累加变量（需保证线程安全）
        var watch = System.Diagnostics.Stopwatch.StartNew();

        // 并行循环累加
        ParallelLoopResult result = Parallel.For(
            0, // 起始索引（包含）
            numbers.Length, // 结束索引（不包含）
            // 循环体：i为当前迭代索引
            (i) => Interlocked.Add(ref total, numbers[i]) // 用Interlocked保证原子累加
        );

        watch.Stop();
        Console.WriteLine($"并行累加结果：{total}");
        Console.WriteLine($"耗时：{watch.ElapsedMilliseconds}ms");
        Console.WriteLine($"循环是否完成：{result.IsCompleted}");
    }
}
```

关键说明：

* `Parallel.For` 的迭代索引 `i` 是线程局部的，无需担心冲突，但共享变量（如 `total` ）必须保证线程安全（用 `Interlocked、lock或ConcurrentBag` 等）；

* 返回值 `ParallelLoopResult` 包含循环执行状态（`IsCompleted`：是否全部完成；`LowestBreakIteration`：是否提前中断）；

* 迭代间不能有依赖（如第 `i` 次迭代依赖第 `i-1` 次的结果），否则会导致结果错误。

#### Parallel.ForEach：并行执行 foreach 循环

替代传统的 `foreach` 循环，遍历 `IEnumerable<T>` 集合，将元素分配到多个线程并行处理。

核心语法：

```csharp
// 基础版：遍历IEnumerable<T>集合
public static ParallelLoopResult ForEach<TSource>(
    IEnumerable<TSource> source, 
    Action<TSource> body
);

// 带索引版：获取元素的索引
public static ParallelLoopResult ForEach<TSource>(
    IEnumerable<TSource> source, 
    Action<TSource, ParallelLoopState, long> body
);
```

示例：并行处理文件列表（ `CPU` 密集型的文件内容解析）

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Threading.Tasks;

class ParallelForEachDemo
{
    static void Main()
    {
        // 获取指定目录下的所有文本文件
        string[] files = Directory.GetFiles(@"D:\test", "*.txt");
        // 存储解析结果（线程安全集合）
        var parseResults = new System.Collections.Concurrent.ConcurrentDictionary<string, int>();

        var watch = System.Diagnostics.Stopwatch.StartNew();

        // 并行遍历文件列表
        Parallel.ForEach(
            files, // 要遍历的集合
            (file, state, index) => // 循环体：file=当前文件，state=循环状态，index=当前索引
            {
                try
                {
                    // 解析文件：统计文件中的数字数量（CPU密集型）
                    int numberCount = CountNumbersInFile(file);
                    // 将结果存入线程安全字典
                    parseResults.TryAdd(file, numberCount);
                    Console.WriteLine($"已处理第{index+1}个文件：{file}，数字数量：{numberCount}");
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"处理文件{file}失败：{ex.Message}");
                    // 可选：终止所有迭代
                    // state.Stop();
                }
            }
        );

        watch.Stop();
        Console.WriteLine($"\n全部处理完成，共{parseResults.Count}个文件，耗时：{watch.ElapsedMilliseconds}ms");
    }

    // 模拟CPU密集型任务：统计文件中的数字数量
    static int CountNumbersInFile(string filePath)
    {
        string content = File.ReadAllText(filePath);
        int count = 0;
        foreach (char c in content)
        {
            if (char.IsDigit(c)) count++;
        }
        // 模拟复杂计算（放大CPU消耗）
        for (int i = 0; i < 100000; i++) { Math.Sqrt(i); }
        return count;
    }
}
```

关键说明：

* `ParallelLoopState`：用于控制循环（ `Stop()` 终止所有迭代、`Break()` 终止后续迭代、`IsStopped` 判断是否终止）；

* 推荐使用线程安全集合（如 `ConcurrentDictionary、ConcurrentBag` ）存储并行处理的结果，避免共享集合的线程安全问题；

* 若集合元素数量少、循环体执行时间极短，并行开销可能超过收益，此时应使用串行循环。

#### ParallelOptions：配置并行行为

`ParallelOptions` 用于自定义并行执行的规则，核心属性：

| `MaxDegreeOfParallelism` | 限制最大并行度（线程数），默认值为`-1`（自动适配 CPU 核心数），可设置为具体数值（如`4`表示最多 4 个线程并行） |
| ------------------------ | ------------------------------------------------------------------------------------------------------------- |
| `CancellationToken`      | 取消令牌，用于取消并行执行                                                                                    |
| `TaskScheduler`          | 指定任务调度器（默认使用线程池调度器）                                                                        |

示例：限制并行度 + 取消并行执行

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class ParallelOptionsDemo
{
    static void Main()
    {
        CancellationTokenSource cts = new CancellationTokenSource();
        // 5秒后取消执行
        cts.CancelAfter(5000);

        ParallelOptions options = new ParallelOptions
        {
            MaxDegreeOfParallelism = 4, // 最多4个线程并行
            CancellationToken = cts.Token // 绑定取消令牌
        };

        try
        {
            Parallel.For(
                0,
                1000000,
                options,
                (i) =>
                {
                    // 模拟耗时操作
                    Thread.Sleep(10);
                    if (i % 100000 == 0) Console.WriteLine($"已处理{i}次");
                    // 检查取消令牌（可选，TPL会自动检查，但手动检查更及时）
                    options.CancellationToken.ThrowIfCancellationRequested();
                }
            );
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("并行执行被取消");
        }
        catch (AggregateException ex)
        {
            foreach (var innerEx in ex.InnerExceptions)
            {
                Console.WriteLine($"异常：{innerEx.Message}");
            }
        }
        finally
        {
            cts.Dispose();
        }
    }
}
```

并发度说明：

* 默认：≈ CPU 核心数

* 并不是越大越好

* 过大 → 线程切换成本上升

> CPU 密集型 ≈ 核心数
轻计算 ≈ 核心数 × 1.5

#### PLINQ（Parallel LINQ）

并行查询：

```csharp
var query = from num in data.AsParallel()  // 启用并行
            where num % 2 == 0
            select num * 2;

var results = query.ToArray();  // 强制执行
```

* 选项：`WithDegreeOfParallelism(4)` 控制并行度；`WithExecutionMode(ParallelExecutionMode.ForceParallelism)` 强制并行。

* 有序 vs. 无序：默认无序；用 `AsOrdered()` 保持顺序（性能稍低）。

### 底层原理：Parallel 的执行机制

`Parallel` 的高效性源于 `TPL` 的核心设计：

* 分区策略：将循环拆分为多个 “分区”（`Partitioner`），每个分区由一个线程处理，避免单个线程处理过多迭代；

* 工作窃取算法（`Work-Stealing`）：若某个线程完成自身分区后，会从其他线程的分区中 “窃取” 未处理的迭代，实现负载均衡；

* 线程池复用：使用线程池的工作线程，避免频繁创建 / 销毁线程的开销；

* `PLINQ` 内部：用 `ParallelQuery<T>` 包装，查询树上附加并行运算符；合并结果时用 `Barrier` 同步。

* 性能开销：启动线程 ~1ms；适合 >100ms 任务；小任务可能负优化（开销 > 收益）。

* 异常聚合：所有迭代抛出的异常会被包装为 `AggregateException`，需遍历`InnerExceptions` 处理。

### Parallel 与线程安全

#### 错误示例：共享变量

```csharp
int sum = 0;

Parallel.For(0, 1000, i =>
{
    sum += i; // ❌ 线程不安全
});
```

结果：不确定

#### 正确方式一：Interlocked

```csharp
int sum = 0;

Parallel.For(0, 1000, i =>
{
    Interlocked.Add(ref sum, i);
});
```

#### 正确方式二：局部变量 + 聚合（推荐）

```csharp
int sum = 0;

Parallel.For(0, 1000,
    () => 0,
    (i, state, local) => local + i,
    local => Interlocked.Add(ref sum, local)
);
```
高性能、低竞争

### 替代方案：

* 小任务：用 `Task.Run` 或 `Parallel LINQ`。

* 数据并行：`System.Numerics (SIMD)` 或 `GPU (CUDA.NET)`。

* 高并发：`Actor` 模型 (`Akka.NET`) 或 `Channels`。

### Parallel.ForEachAsync

在受控并发度下，并行执行异步操作

它本质是：

* `async / await` 友好

* 支持并发限制

* 内置 `CancellationToken`

* 自动调度，不用手写 `SemaphoreSlim`

#### 基本用法

##### 最简单示例

```csharp
await Parallel.ForEachAsync(items, async (item, ct) =>
{
    await ProcessAsync(item, ct);
});
```

* 必须 `await`

* `lambda` 参数里有 `CancellationToken`

* 返回 `ValueTask`

##### 限制并发度

```csharp
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = 5
};

await Parallel.ForEachAsync(items, options, async (item, ct) =>
{
    await CallApiAsync(item, ct);
});
```

这相当于：

```csharp
SemaphoreSlim(5) + Task.WhenAll
```

但 更简洁、更安全

### Parallel.ForEachAsync vs Task.WhenAll

#### Task.WhenAll（无并发控制）

```csharp
await Task.WhenAll(items.Select(item =>
    ProcessAsync(item)
));
```

特点：

* 一次性创建所有 `Task`

* 不限制并发

* 数量大 → 容易压垮资源

#### Parallel.ForEachAsync（有并发控制）

```csharp
await Parallel.ForEachAsync(items,
    new ParallelOptions { MaxDegreeOfParallelism = 5 },
    async (item, ct) =>
    {
        await ProcessAsync(item, ct);
    });
```

特点：

* 滑动窗口式并发

* 同时最多 `N` 个任务

* 更适合：

    * HTTP

    * DB

    * 文件 IO

    * 调用第三方接口

#### 什么时候选哪个？

| 场景            | 推荐                  |
| --------------- | --------------------- |
| 少量任务（<20） | Task.WhenAll          |
| 大量任务        | Parallel.ForEachAsync |
| 需要限流        | Parallel.ForEachAsync |
| CPU 密集        | Parallel.For / PLINQ  |

### 典型实战场景

#### 批量调用第三方 API

```csharp
await Parallel.ForEachAsync(userIds,
    new ParallelOptions { MaxDegreeOfParallelism = 10 },
    async (id, ct) =>
    {
        await apiClient.SyncUserAsync(id, ct);
    });
```

#### 批量文件处理（IO）

```csharp
await Parallel.ForEachAsync(files,
    async (file, ct) =>
    {
        var content = await File.ReadAllTextAsync(file, ct);
        await SaveAsync(content, ct);
    });
```