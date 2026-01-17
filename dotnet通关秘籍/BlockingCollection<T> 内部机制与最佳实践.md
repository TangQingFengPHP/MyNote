### 简介

`BlockingCollection<T>` 是 `.NET` 中非常重要且实用的线程安全、阻塞式的生产者-消费者集合类，位于 `System.Collections.Concurrent` 命名空间。

> BlockingCollection 不是队列，
而是一个“带阻塞语义的并发管道（Blocking Producer–Consumer Abstraction）”。
> 在并发集合外面，加了一层“阻塞 + 容量控制 + 完成语义”

### 什么是生产者-消费者模式？

```
// 生产者线程 → [BlockingCollection] → 消费者线程
// 1. 生产者添加项目，如果集合已满则阻塞等待
// 2. 消费者取出项目，如果集合为空则阻塞等待
// 3. 自动的线程同步和资源管理
```

### 核心定位与价值

`BlockingCollection<T>` 是一个包装器，它可以基于以下几种底层集合来工作（默认使用 `ConcurrentQueue<T>`）：

| 底层集合类型                          | 默认 | 有界（Bounded） | 特点                |
| ------------------------------------- | ---- | --------------- | ------------------- |
| ConcurrentQueue<T>                    | 是   | 可选            | FIFO，性能最高      |
| ConcurrentStack<T>                    | 否   | 可选            | LIFO                |
| ConcurrentBag<T>                      | 否   | 可选            | 无序，插入/取出最快 |
| 自定义 IProducerConsumerCollection<T> | 否   | 可选            | 高度自定义          |

在多线程场景中，“生产者线程生产数据，消费者线程消费数据” 是高频场景（如日志收集、任务队列、消息处理）。若用普通集合（如`List<T>`）+ 手动锁实现，需处理：

* 线程安全（加 `lock` ）；

* 空集合时消费者等待（`Monitor.Wait`）；

* 满集合时生产者等待（`Monitor.Wait`）；

* 数据就绪时唤醒等待线程（`Monitor.Pulse`）。

`BlockingCollection<T>` 封装了上述所有逻辑，核心价值：

* 开箱即用的阻塞逻辑：空集合消费阻塞、满集合生产阻塞；

* 线程安全：所有操作（添加 / 移除 / 遍历）均线程安全；

* 支持边界限制：可设置集合最大容量（满则阻塞生产者）；

* 支持取消 / 完成：可优雅停止生产 / 消费，避免线程卡死；

* 灵活的底层存储：默认基于 `ConcurrentQueue<T>`（先进先出），也可指定 `ConcurrentStack<T>/ConcurrentBag<T>`。

### 最常用的几种创建方式

```csharp
// 1. 最常用：无界队列（推荐用于大多数场景）
var bc = new BlockingCollection<string>();

// 2. 有界队列（限制容量，生产者满时会阻塞）
var bcBounded = new BlockingCollection<string>(boundedCapacity: 100);

// 3. 指定底层集合 + 有界
var bcStack = new BlockingCollection<string>(
    new ConcurrentStack<string>(),
    boundedCapacity: 50);

// 4. 基于已有的集合（高级用法）
var queue = new ConcurrentQueue<string>();
var bcFromExisting = new BlockingCollection<string>(queue, 200);
```

### 核心 API 与基础使用

#### 核心构造函数

* `BlockingCollection<T>()`: 默认构造：无边界限制，底层用 `ConcurrentQueue<T>`

* `BlockingCollection<T>(int boundedCapacity)`: 指定最大容量（边界），满则生产者阻塞

* `BlockingCollection<T>(IProducerConsumerCollection<T>)`: 自定义底层存储（如`ConcurrentStack<T>`）

* `BlockingCollection<T>(IProducerConsumerCollection<T>, int)`: 自定义存储 + 最大容量

#### 核心方法 / 属性

* `Add(T item)`: 向集合添加元素：若集合满则阻塞，直到有空间

* `Add(T item, CancellationToken)`: 带取消令牌的 Add：可中途取消阻塞

* `Take()`: 从集合移除并返回元素：若集合空则阻塞，直到有元素

* `Take(CancellationToken)`: 带取消令牌的 Take：可中途取消阻塞

* `TryAdd(T item, int millisecondsTimeout)`: 尝试添加：超时返回 false（非阻塞）

* `TryTake(out T item, int millisecondsTimeout)`: 尝试获取：超时返回 false（非阻塞）

* `CompleteAdding()`: 标记 “添加完成”：后续 Add 会抛异常，Take 在集合空后退出

* `IsAddingCompleted`: 判断是否已调用 `CompleteAdding()`

* `IsCompleted`: 判断是否 “添加完成且集合为空”

* `BoundedCapacity`: 集合最大容量（-1 表示无限制）

#### 核心操作方法

```csharp
public class CoreOperations
{
    public static void DemonstrateOperations()
    {
        var collection = new BlockingCollection<string>(boundedCapacity: 3);
        
        // 1. 添加项目
        collection.Add("项目1"); // 阻塞直到有空间
        
        // 2. 尝试添加（不阻塞）
        bool added = collection.TryAdd("项目2", millisecondsTimeout: 0);
        Console.WriteLine($"尝试添加结果: {added}");
        
        // 3. 带超时的添加
        bool addedWithTimeout = collection.TryAdd("项目3", 
            millisecondsTimeout: 1000); // 最多等待1秒
        Console.WriteLine($"带超时添加结果: {addedWithTimeout}");
        
        // 4. 取出项目（阻塞）
        string item1 = collection.Take(); // 阻塞直到有项目可取
        Console.WriteLine($"取出: {item1}");
        
        // 5. 尝试取出（不阻塞）
        bool taken = collection.TryTake(out string item2, millisecondsTimeout: 0);
        Console.WriteLine($"尝试取出结果: {taken}, 项目: {item2}");
        
        // 6. 查看但不移除
        bool peeked = collection.TryPeek(out string item3);
        Console.WriteLine($"查看结果: {peeked}, 项目: {item3}");
        
        // 7. 完成添加
        collection.CompleteAdding();
        Console.WriteLine($"IsAddingCompleted: {collection.IsAddingCompleted}");
        Console.WriteLine($"IsCompleted: {collection.IsCompleted}");
        
        // 8. 获取当前所有项目（不阻塞）
        string[] allItems = collection.ToArray();
        Console.WriteLine($"当前项目数: {allItems.Length}");
    }
}
```

#### 基础示例：简单生产者 - 消费者

```csharp
using System;
using System.Collections.Concurrent;
using System.Threading;
using System.Threading.Tasks;

class BlockingCollectionBasicDemo
{
    static void Main()
    {
        // 创建阻塞集合，最大容量为5（满则生产者阻塞）
        var bc = new BlockingCollection<int>(5);

        // 1. 生产者线程：生产1-10的数字
        Task producer = Task.Run(() =>
        {
            for (int i = 1; i <= 10; i++)
            {
                bc.Add(i); // 满则阻塞
                Console.WriteLine($"生产者：添加 {i}，当前集合数量：{bc.Count}");
                Thread.Sleep(100); // 模拟生产耗时
            }
            // 标记添加完成：消费者知道不会有新数据了
            bc.CompleteAdding();
            Console.WriteLine("生产者：完成所有生产，标记添加完成");
        });

        // 2. 消费者线程：消费所有数字
        Task consumer = Task.Run(() =>
        {
            // GetConsumingEnumerable()：遍历集合，空则阻塞，直到CompleteAdding且空
            foreach (int item in bc.GetConsumingEnumerable())
            {
                Console.WriteLine($"消费者：消费 {item}，当前集合数量：{bc.Count}");
                Thread.Sleep(500); // 模拟消费耗时（比生产慢，会导致集合堆积）
            }
            Console.WriteLine("消费者：所有数据消费完成");
        });

        // 等待所有任务完成
        Task.WaitAll(producer, consumer);
        bc.Dispose(); // 释放资源
    }
}
```

输出结果

```
生产者：添加 1，当前集合数量：1
生产者：添加 2，当前集合数量：2
生产者：添加 3，当前集合数量：3
生产者：添加 4，当前集合数量：4
生产者：添加 5，当前集合数量：5
消费者：消费 1，当前集合数量：4
生产者：添加 6，当前集合数量：5  // 消费后腾出空间，生产者继续添加
生产者：添加 7，当前集合数量：5  // 集合再次满，生产者阻塞
消费者：消费 2，当前集合数量：4
生产者：添加 8，当前集合数量：5
...（后续依次消费和生产）
生产者：完成所有生产，标记添加完成
消费者：消费 10，当前集合数量：0
消费者：所有数据消费完成
```

核心现象：

* 集合容量设为 5，生产者添加到 5 个后阻塞，直到消费者消费 1 个腾出空间；

* `GetConsumingEnumerable()` 自动处理阻塞逻辑，无需手动判断集合是否为空；

* `CompleteAdding()` 后，消费者遍历完剩余数据即退出，不会无限阻塞。

### 高级用法详解

#### 边界限制（Bounded Capacity）

通过构造函数指定 `boundedCapacity`，实现 “生产者限流”：

```csharp
// 最大容量3，满则生产者阻塞
var bc = new BlockingCollection<string>(3);

// 生产者1：快速添加3个元素，第4个会阻塞
Task.Run(() =>
{
    bc.Add("A");
    bc.Add("B");
    bc.Add("C");
    Console.WriteLine("生产者1：已添加3个，准备添加第4个（会阻塞）");
    bc.Add("D"); // 阻塞，直到消费者消费一个
    Console.WriteLine("生产者1：第4个元素添加成功");
});

// 消费者1：2秒后消费一个元素
Task.Run(() =>
{
    Thread.Sleep(2000);
    var item = bc.Take();
    Console.WriteLine($"消费者1：消费 {item}");
});
```

#### 取消阻塞（CancellationToken）

用 `CancellationToken` 中断阻塞的 `Add/Take` 操作，避免线程永久阻塞：

```csharp
var cts = new CancellationTokenSource();
// 3秒后取消
cts.CancelAfter(3000);

var bc = new BlockingCollection<int>();

// 生产者：尝试添加，3秒后取消
Task.Run(() =>
{
    try
    {
        // 集合无边界，此处不会阻塞，但演示取消逻辑
        for (int i = 1; ; i++)
        {
            bc.Add(i, cts.Token);
            Console.WriteLine($"添加 {i}");
            Thread.Sleep(500);
        }
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("生产者：添加操作被取消");
        bc.CompleteAdding();
    }
});

// 消费者：尝试消费，3秒后取消
Task.Run(() =>
{
    try
    {
        while (true)
        {
            int item = bc.Take(cts.Token);
            Console.WriteLine($"消费 {item}");
        }
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("消费者：消费操作被取消");
    }
});
```

#### 自定义底层存储

默认底层是 `ConcurrentQueue<T>`（FIFO），可指定 `ConcurrentStack<T>`（LIFO）或 `ConcurrentBag<T>`（无序）：

```csharp
// 底层用ConcurrentStack（栈：后进先出）
var bc = new BlockingCollection<int>(new ConcurrentStack<int>());

bc.Add(1);
bc.Add(2);
bc.Add(3);

// Take会获取最后添加的3（栈顶）
Console.WriteLine(bc.Take()); // 输出：3
Console.WriteLine(bc.Take()); // 输出：2
Console.WriteLine(bc.Take()); // 输出：1
```

#### 多生产者 / 多消费者

`BlockingCollection<T>` 天然支持多生产者、多消费者并发操作，无需额外同步：

```csharp
var bc = new BlockingCollection<int>(10);

// 3个生产者线程
for (int i = 0; i < 3; i++)
{
    int producerId = i + 1;
    Task.Run(() =>
    {
        for (int j = 1; j <= 5; j++)
        {
            int value = producerId * 100 + j;
            bc.Add(value);
            Console.WriteLine($"生产者{producerId}：添加 {value}");
            Thread.Sleep(100);
        }
    });
}

// 2个消费者线程
for (int i = 0; i < 2; i++)
{
    int consumerId = i + 1;
    Task.Run(() =>
    {
        foreach (var item in bc.GetConsumingEnumerable())
        {
            Console.WriteLine($"消费者{consumerId}：消费 {item}");
            Thread.Sleep(200);
        }
    });
}

// 等待所有生产者完成后标记添加完成
Task.Delay(2000).ContinueWith(_ => bc.CompleteAdding());
```

#### 数据流水线（Pipeline）模式

```csharp
public class DataPipelineExample
{
    public static void RunPipeline()
    {
        // 创建三个阶段的流水线
        var stage1 = new BlockingCollection<string>(boundedCapacity: 10);
        var stage2 = new BlockingCollection<string>(boundedCapacity: 10);
        var stage3 = new BlockingCollection<string>(boundedCapacity: 10);
        
        CancellationTokenSource cts = new CancellationTokenSource();
        
        // 阶段1：数据源
        var sourceTask = Task.Run(() =>
        {
            try
            {
                for (int i = 1; i <= 20; i++)
                {
                    string data = $"原始数据{i}";
                    stage1.Add(data, cts.Token);
                    Console.WriteLine($"阶段1: 产生 {data}");
                    Thread.Sleep(50);
                }
                
                stage1.CompleteAdding();
                Console.WriteLine("阶段1完成");
            }
            catch (OperationCanceledException)
            {
                Console.WriteLine("阶段1被取消");
            }
        });
        
        // 阶段2：数据处理
        var processorTask = Task.Run(() =>
        {
            try
            {
                foreach (var item in stage1.GetConsumingEnumerable(cts.Token))
                {
                    string processed = $"处理过的[{item}]";
                    stage2.Add(processed, cts.Token);
                    Console.WriteLine($"阶段2: 处理 {item} -> {processed}");
                    Thread.Sleep(100);
                }
                
                stage2.CompleteAdding();
                Console.WriteLine("阶段2完成");
            }
            catch (OperationCanceledException)
            {
                Console.WriteLine("阶段2被取消");
            }
        });
        
        // 阶段3：数据输出
        var outputTask = Task.Run(() =>
        {
            try
            {
                foreach (var item in stage2.GetConsumingEnumerable(cts.Token))
                {
                    string result = $"最终结果<{item}>";
                    stage3.Add(result, cts.Token);
                    Console.WriteLine($"阶段3: 输出 {item} -> {result}");
                    Thread.Sleep(80);
                }
                
                stage3.CompleteAdding();
                Console.WriteLine("阶段3完成");
            }
            catch (OperationCanceledException)
            {
                Console.WriteLine("阶段3被取消");
            }
        });
        
        // 监控输出
        var monitorTask = Task.Run(() =>
        {
            int count = 0;
            foreach (var item in stage3.GetConsumingEnumerable())
            {
                count++;
                Console.WriteLine($"监控: 收到第{count}个结果: {item}");
            }
            
            Console.WriteLine($"监控: 总共收到 {count} 个结果");
        });
        
        // 运行5秒后取消
        Task.Run(() =>
        {
            Thread.Sleep(5000);
            Console.WriteLine("\n流水线运行5秒，发送取消信号...");
            cts.Cancel();
        });
        
        try
        {
            Task.WaitAll(sourceTask, processorTask, outputTask, monitorTask, 10000);
        }
        catch (AggregateException ex)
        {
            Console.WriteLine($"任务异常: {ex.Flatten().Message}");
        }
        
        Console.WriteLine("流水线运行结束");
    }
}
```

### 使用场景

#### 适合场景

* `CPU` 线程池任务

* 后台 `Worker`

* 批处理系统

* `ETL` 管道

* 传统 `Producer–Consumer`

#### 不适合场景

* `async/await`

* 高吞吐低延迟网络 IO

* `UI` 线程

* 实时系统

### 总结

* `BlockingCollection<T>` 是 `.NET` 官方的阻塞式线程安全集合，核心适配 “生产者 - 消费者” 模型；

* 核心特性：空集合消费阻塞、满集合生产阻塞，支持边界限制、取消操作、自定义底层存储；

* 核心 API：`Add()`（生产）、`Take()`（消费）、`CompleteAdding()`（标记生产完成）、`GetConsumingEnumerable()`（遍历消费）；

* 关键坑点：必须调用 `CompleteAdding()` 避免消费者永久阻塞，使用后需 `Dispose` 释放资源；

* 适用场景：日志收集、任务队列、消息分发、多线程数据处理等生产者 - 消费者场景，优先使用而非手动实现。