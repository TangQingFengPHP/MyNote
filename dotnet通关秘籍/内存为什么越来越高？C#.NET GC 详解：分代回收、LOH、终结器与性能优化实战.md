### 简介

平时写 `C#`，很少会主动去碰内存管理。

因为 `.NET` 已经把最麻烦的一层包掉了：

* 对象分配不用手动 `malloc`
* 对象释放不用手动 `free`
* 大部分时候只管写业务，程序也能正常跑

但只要项目一上量，`GC` 这个词迟早会冒出来：

* 接口偶尔抖一下，排查看到 `Gen2 GC`
* 内存一直涨，不确定是正常波动还是泄漏
* 高频循环里 `new` 了很多对象，担心 `GC` 压力
* 大数组分配很多，发现 `LOH`、碎片、停顿时间开始难看
* 明明写了终结器，资源却没有按预期及时释放

很多资料一上来就讲一堆术语：

* `Gen0`
* `Gen1`
* `Gen2`
* `LOH`
* `SOH`
* `Finalizer`
* `Background GC`
* `Server GC`

概念不少，但真正难的不是背名词，而是把它们串成一张图。

一句话先说透：

> `.NET GC` 不是“定时清理垃圾”的后台线程，而是一套围绕托管堆、对象存活、分代回收、停顿时间和吞吐量做平衡的运行时机制。

这篇文章重点讲这些事：

* `GC` 到底回收什么，不回收什么
* 托管堆、引用、可达性到底怎么配合
* `Gen0 / Gen1 / Gen2 / LOH` 分别在解决什么问题
* 为什么有些对象会“活着活着就进了 `Gen2`”
* 终结器、`IDisposable`、手动 `GC.Collect()` 的边界是什么
* 遇到内存高、回收频繁、停顿明显时，应该从哪里下手
* 用几个能直接跑的 demo 把这些现象看出来

### 先把最容易搞混的一点说清楚

很多人一提 `GC`，脑子里冒出的第一反应是：

* 堆内存满了，系统帮忙删对象

这句话不能说完全错，但太粗了。

更准确的说法是：

> `GC` 回收的不是“没用了的变量名”，而是“已经不可达的托管对象”。

比如：

```csharp
var user = new User();
user = null;
```

这里不是 `user` 这个变量被回收了。

真正可能被回收的是原来那块 `User` 对象实例，只不过它在这之后已经没有有效引用能走到了。

所以 `GC` 关心的核心从来不是“变量还在不在”，而是：

* 这块对象是否还可达
* 是否还能从一组根对象一路引用过去

### GC 回收的到底是什么？

在 `.NET` 里，托管对象一般分配在托管堆上。

例如：

```csharp
var order = new Order();
var items = new List<OrderItem>();
var buffer = new byte[4096];
```

这些对象通常都会进入托管堆，由 `GC` 管理生命周期。

`GC` 会从一批“根”开始往下找仍然活着的对象，常见根包括：

* 当前线程栈上还能访问到的局部变量
* 静态字段
* `GCHandle` 固定或持有的对象
* CPU 寄存器里还保留着的托管引用

如果某个对象已经没有任何路径能从这些根走到它，那它就是垃圾候选。

可以粗略理解成这样：

```text
GC Roots
  -> static Cache
      -> Dictionary
          -> Product

GC Roots
  -> stack local variable
      -> Order
          -> OrderLines
```

只要路径还通，这些对象就不会被回收。

### 为什么 GC 不等于“自动解决内存问题”？

因为 `GC` 只负责回收“不可达的托管对象”。

它不自动解决下面这些问题：

* 静态集合一直持有对象，导致对象永远可达
* 事件订阅没解绑，导致对象被间接引用
* 非托管资源没及时释放，例如文件句柄、数据库连接句柄、原生内存
* 大对象频繁分配，导致回收成本和碎片上升

所以很多所谓“内存泄漏”，在托管世界里并不是对象永远无法回收，而是：

> 对象本来应该死，但还被某条引用链拽着。

### 为什么 GC 要分代？

这是 `.NET GC` 最核心的设计之一。

背后有个非常实用的假设：

> 绝大多数对象都活不久。

比如这些场景里的对象就很典型：

* 一次 JSON 反序列化产生的临时对象
* 循环里的字符串拼接中间结果
* 一次请求里的 DTO、临时列表、局部缓冲区
* `LINQ` 查询过程里的中间对象

如果每次回收都把整个堆从头扫到尾，成本会很高。

所以 `.NET` 用了分代回收：

* 新生对象先放在年轻代
* 大多数短命对象尽快在年轻代回收掉
* 存活时间越久的对象，越少去频繁扫描

这就是 `Gen0 / Gen1 / Gen2` 的背景。

### `Gen0`、`Gen1`、`Gen2` 到底是什么？

可以先用最白话的方式理解：

* `Gen0`：新生区，刚 `new` 出来的小对象大多先来这里
* `Gen1`：过渡区，熬过一次年轻代回收的对象，可能会来到这里
* `Gen2`：老年代，活得比较久的对象通常最后留在这里

一句话记忆：

> 越年轻，回收越频繁；越年老，回收越贵。

下面这个表最适合建立整体印象：

| 区域 | 典型对象 | 回收频率 | 回收成本 |
| --- | --- | --- | --- |
| `Gen0` | 短命临时对象 | 高 | 低 |
| `Gen1` | 过渡对象 | 中 | 中 |
| `Gen2` | 长生命周期对象 | 低 | 高 |

这里要特别注意一个常见误区：

> “对象一定会按 `Gen0 -> Gen1 -> Gen2` 一步一步升级”

实际理解时可以这么记：

* 新对象通常先在 `Gen0`
* 年轻代回收后，存活对象会被提升
* 老年代回收更少，但每次代价更高

至于某次具体回收后对象怎么移动、何时提升，属于运行时实现细节，不适合靠死记硬背一个机械流程来理解。

### Demo 1：看一眼对象现在大概在哪一代

先上一个最基础的例子。

```csharp
using System;

var obj = new object();
var bytes = new byte[1024];
var bigBytes = new byte[100_000];

Console.WriteLine($"obj      代：{GC.GetGeneration(obj)}");
Console.WriteLine($"bytes    代：{GC.GetGeneration(bytes)}");
Console.WriteLine($"bigBytes 代：{GC.GetGeneration(bigBytes)}");
Console.WriteLine($"MaxGeneration：{GC.MaxGeneration}");
```

通常会看到类似结果：

```text
obj      代：0
bytes    代：0
bigBytes 代：2
MaxGeneration：2
```

这个输出至少说明两件事：

* 普通小对象新建后通常在 `Gen0`
* 大对象很可能直接按老年代逻辑管理

这里的 `bigBytes`，就把话题带到了另一个重点：`LOH`。

### 什么是 `LOH`？

`LOH` 全称是：

```text
Large Object Heap
```

也就是大对象堆。

在 `.NET` 里，足够大的对象不会走普通小对象那条路，而是进入 `LOH`。

最常见的例子就是：

```csharp
var buffer = new byte[100_000];
```

这类对象为什么值得单独拿出来讲？

因为它们有几个非常关键的特点：

* 分配成本更高
* 回收通常要等更重的 GC
* 如果频繁申请和释放，容易带来更明显的内存压力
* 大对象场景下，更应该考虑池化和复用

项目里一旦频繁处理这些东西，就要提高警惕：

* 大数组
* 大字符串
* 图像、压缩、序列化缓冲区
* 一次性读入的大块二进制内容

### `85,000` 字节为什么经常被提到？

因为在 `.NET` 世界里，经常会看到一句经验话：

> 大于等于约 `85,000` 字节的对象，通常会进入 `LOH`。

这不是背面试题的重点，真正要记的是：

* 大对象分配路径不一样
* 大对象回收代价更高
* 高频大对象分配，比高频小对象分配更容易把程序拖慢

### Demo 2：用 `CollectionCount` 看 GC 次数变化

下面这个 demo 很适合用来建立直觉。

```csharp
using System;

static void PrintCounts(string title)
{
    Console.WriteLine(title);
    Console.WriteLine($"Gen0: {GC.CollectionCount(0)}");
    Console.WriteLine($"Gen1: {GC.CollectionCount(1)}");
    Console.WriteLine($"Gen2: {GC.CollectionCount(2)}");
    Console.WriteLine();
}

PrintCounts("开始前");

for (int i = 0; i < 1_000_000; i++)
{
    _ = new object();
}

PrintCounts("创建大量短命小对象后");

for (int i = 0; i < 2_000; i++)
{
    _ = new byte[100_000];
}

PrintCounts("继续创建大量大对象后");
```

这个例子要看的不是某个固定数字，而是趋势：

* 大量短命小对象，通常先把 `Gen0` 回收次数推高
* 大对象一多，`Gen2` 相关压力会更明显

这类 demo 非常适合配合 `dotnet-counters`、Visual Studio 诊断工具、`PerfView` 一起看。

### GC 一次回收时大概发生了什么？

可以用一个简化版流程来理解：

```text
1. 找根
2. 标记仍然可达的对象
3. 回收不可达对象占用的空间
4. 视情况整理内存，让后续分配更高效
```

虽然底层细节远比这复杂，但工程上只要先吃透两件事就够了：

* `GC` 不是看“变量名还在不在”，而是看对象是否仍可达
* 回收范围越大、活对象越多，回收成本通常越高

### 为什么有时候 `GC` 会让程序“顿一下”？

因为 `GC` 某些阶段需要暂停托管线程，去做对象图分析和堆整理。

这就是常说的：

```text
STW（Stop The World）
```

当然，现代 `.NET GC` 已经做了很多优化，比如后台 GC、并发回收等，不是每次都“全停很久”。

但从现象上理解，依然可以先抓住这句话：

> 回收越重，停顿风险越大；活对象越多，扫描成本越高。

这也是为什么线上真正让人难受的，往往不是 `Gen0`，而是：

* `Gen2` 回收偏多
* `LOH` 压力偏大
* 大量对象被不必要地长期持有

### Demo 3：对象为什么会被“养大”到高代？

很多性能问题，不是分配太多，而是对象活得太久。

看这个例子：

```csharp
using System;
using System.Collections.Generic;

var cache = new List<byte[]>();

for (int i = 0; i < 10_000; i++)
{
    cache.Add(new byte[1024]);
}

Console.WriteLine($"缓存数量：{cache.Count}");
Console.WriteLine($"当前内存：{GC.GetTotalMemory(false) / 1024 / 1024.0:F2} MB");
Console.WriteLine($"Gen2 回收次数：{GC.CollectionCount(2)}");
```

这段代码的问题不在于 `byte[1024]` 很大，而在于：

* 这些对象被 `cache` 一直持有
* 对 `GC` 来说，它们始终可达
* 一旦活得足够久，就会逐渐成为高代对象

这也是很多“内存泄漏感”问题的根源。

代码里未必有真正意义上的泄漏，但对象就是下不来。

### 静态字段为什么危险？

因为静态字段本身就是一类典型 GC Root 链路入口。

例如：

```csharp
public static class AppCache
{
    public static List<Order> Orders { get; } = new();
}
```

只要这个进程没结束，`AppCache.Orders` 就一直在。

如果往里塞临时对象，又没有淘汰策略，那这些对象就会一直活着。

最常见的坑包括：

* 静态字典无限长大
* 单例服务里缓存了不该长期缓存的数据
* 事件总线一直持有订阅者
* 长生命周期对象引用了本来短命的对象

### 终结器到底是干什么的？

终结器就是很多人印象里的析构函数：

```csharp
~FileHolder()
{
    // 清理逻辑
}
```

但它的定位一定要讲清楚：

> 终结器不是“对象不用了就立刻执行”的释放机制，而是运行时在未来某个时刻补做清理的最后防线。

这意味着两件事：

* 执行时间不确定
* 带终结器的对象，回收路径会更重

也就是说，终结器不是性能优化工具，反而会增加回收复杂度。

### Demo 4：终结器为什么不能代替 `Dispose`

看一个简单例子：

```csharp
using System;
using System.Threading;

CreateObject();

Console.WriteLine("对象已离开作用域");
Thread.Sleep(1000);

GC.Collect();
GC.WaitForPendingFinalizers();

Console.WriteLine("终结器已执行完毕");

static void CreateObject()
{
    var demo = new FinalizerDemo();
}

sealed class FinalizerDemo
{
    ~FinalizerDemo()
    {
        Console.WriteLine("终结器执行了");
    }
}
```

这个 demo 最想说明的是：

* 对象离开作用域，不代表终结器马上执行
* 即便对象已经不可达，终结器也不是同步立刻跑

所以凡是“要尽快释放”的资源，都不应该把希望押在终结器上。

### 真正应该靠什么释放资源？

答案是：

```csharp
IDisposable
```

更准确地说：

> `GC` 负责托管内存，`Dispose` 负责确定性释放外部资源。

典型外部资源包括：

* 文件句柄
* 数据库连接
* Socket
* 非托管内存
* 操作系统句柄

看一个最小例子：

```csharp
using var stream = File.OpenRead("data.txt");
```

这里真正可靠的释放时机，不是“等 GC 高兴了来收”，而是离开作用域时执行 `Dispose`。

这也是为什么很多框架类型都实现了 `IDisposable`。

### `using` 和 GC 的关系到底是什么？

可以这样记：

* `using` 不负责回收托管对象本身
* `using` 负责及时调用 `Dispose`
* 对象本体什么时候被 GC 回收，仍然由运行时决定

所以这两者不是替代关系，而是分工关系。

### 手动调用 `GC.Collect()` 到底该不该？

大多数业务代码里，答案都很简单：

> 不该，或者说不该把它当常规优化手段。

原因是：

* `GC` 自己比业务代码更了解当前堆状态
* 强制回收可能会打断正常节奏
* 本来只是年轻代压力，手动一脚可能把更重的回收提前做了

真正更值得优化的通常不是“手动收”，而是：

* 为什么分配这么多
* 为什么对象活这么久
* 为什么大对象这么频繁

只有少数非常明确的场景，才会认真评估手动触发，例如：

* 已知某一阶段刚释放了大量对象，且后面马上进入内存敏感阶段
* 测试或 benchmark 需要尽量减少噪音

即便如此，也应该谨慎。

### Demo 5：循环里分配模式不同，GC 压力差别会很大

下面这组代码比单纯讲概念更有感觉。

#### 写法一：每次都新建大缓冲区

```csharp
for (int i = 0; i < 10_000; i++)
{
    var buffer = new byte[100_000];
    buffer[0] = 1;
}
```

问题很直接：

* 每次都分配大对象
* 高频进入大对象分配路径
* `GC` 和内存系统压力都不小

#### 写法二：复用缓冲区

```csharp
var buffer = new byte[100_000];

for (int i = 0; i < 10_000; i++)
{
    buffer[0] = 1;
}
```

这两段业务含义可能差不多，但内存行为完全不是一个量级。

如果场景允许，再进一步一点，通常会考虑：

* `ArrayPool<T>`
* `ObjectPool<T>`
* 流式处理，而不是一次性把大块数据全搬上来

### Demo 6：`ArrayPool<T>` 优化前后对比

如果业务里反复用到大缓冲区，`ArrayPool<T>` 往往是非常直接的优化点。

先看一个典型的“分配型写法”。

#### 优化前：每次都 `new byte[]`

```csharp
using System;
using System.Diagnostics;

const int BufferSize = 100_000;
const int Iterations = 20_000;

Run("new byte[]", () =>
{
    for (int i = 0; i < Iterations; i++)
    {
        var buffer = new byte[BufferSize];
        buffer[0] = 1;
        buffer[^1] = 2;
    }
});

static void Run(string title, Action action)
{
    GC.Collect();
    GC.WaitForPendingFinalizers();
    GC.Collect();

    var gen0Before = GC.CollectionCount(0);
    var gen1Before = GC.CollectionCount(1);
    var gen2Before = GC.CollectionCount(2);
    var memoryBefore = GC.GetTotalMemory(true);

    var sw = Stopwatch.StartNew();
    action();
    sw.Stop();

    var memoryAfter = GC.GetTotalMemory(false);

    Console.WriteLine($"==== {title} ====");
    Console.WriteLine($"耗时: {sw.ElapsedMilliseconds} ms");
    Console.WriteLine($"Gen0: {GC.CollectionCount(0) - gen0Before}");
    Console.WriteLine($"Gen1: {GC.CollectionCount(1) - gen1Before}");
    Console.WriteLine($"Gen2: {GC.CollectionCount(2) - gen2Before}");
    Console.WriteLine($"内存变化: {(memoryAfter - memoryBefore) / 1024.0 / 1024.0:F2} MB");
    Console.WriteLine();
}
```

这段代码的问题很明显：

* 每次循环都创建一个 `100_000` 字节数组
* 这种大小已经很容易进入大对象分配路径
* 次数一多，`GC` 压力和 `LOH` 压力都会比较明显

再看池化版。

#### 优化后：用 `ArrayPool<byte>.Shared`

```csharp
using System;
using System.Buffers;
using System.Diagnostics;

const int BufferSize = 100_000;
const int Iterations = 20_000;

Run("ArrayPool<byte>.Shared", () =>
{
    for (int i = 0; i < Iterations; i++)
    {
        var buffer = ArrayPool<byte>.Shared.Rent(BufferSize);

        try
        {
            buffer[0] = 1;
            buffer[BufferSize - 1] = 2;
        }
        finally
        {
            ArrayPool<byte>.Shared.Return(buffer);
        }
    }
});

static void Run(string title, Action action)
{
    GC.Collect();
    GC.WaitForPendingFinalizers();
    GC.Collect();

    var gen0Before = GC.CollectionCount(0);
    var gen1Before = GC.CollectionCount(1);
    var gen2Before = GC.CollectionCount(2);
    var memoryBefore = GC.GetTotalMemory(true);

    var sw = Stopwatch.StartNew();
    action();
    sw.Stop();

    var memoryAfter = GC.GetTotalMemory(false);

    Console.WriteLine($"==== {title} ====");
    Console.WriteLine($"耗时: {sw.ElapsedMilliseconds} ms");
    Console.WriteLine($"Gen0: {GC.CollectionCount(0) - gen0Before}");
    Console.WriteLine($"Gen1: {GC.CollectionCount(1) - gen1Before}");
    Console.WriteLine($"Gen2: {GC.CollectionCount(2) - gen2Before}");
    Console.WriteLine($"内存变化: {(memoryAfter - memoryBefore) / 1024.0 / 1024.0:F2} MB");
    Console.WriteLine();
}
```

这个版本的核心变化只有一件事：

* 不再反复创建新数组
* 改成从池里租一个，用完再还回去

在高频场景里，通常会带来几个直接收益：

* 更少的堆分配
* 更少的 GC 次数
* 更低的 `LOH` 压力
* 更平稳的吞吐和延迟表现

不过 `ArrayPool<T>` 也不是白送的，几个边界要先记住：

* `Rent` 回来的数组长度可能大于请求值
* 数组内容不保证是干净的
* 用完一定要 `Return`
* 如果数组里放过敏感数据，归还前要考虑是否清空

所以它最适合的场景一般是：

* 高频临时缓冲区
* 大小相对稳定的字节数组
* 网络、IO、序列化、压缩这类热点路径

### 高频 GC 的常见根因，不是 GC 太差，而是分配模式太差

线上一旦看到 GC 频繁，优先怀疑这些地方：

* 热点循环里频繁 `new`
* 字符串反复拼接
* 大量 `ToList()`、`ToArray()` 产生中间对象
* 临时大数组频繁申请
* 不必要的装箱拆箱
* 缓存只进不出

很多时候，真正的优化不是“研究某个 GC 参数”，而是把分配行为改健康。

### `struct` 能不能减少 GC 压力？

可以，但这里也很容易被讲歪。

正确理解应该是：

* 值类型在某些场景下可以减少额外堆分配
* 但值类型不是“永远都在栈上”
* 如果值类型成为引用类型对象的一部分，或者发生装箱，照样会进入托管内存体系

所以别把“用 `struct`”理解成万能性能药。

### `Span<T>`、池化、复用，为什么总和 GC 绑在一起讲？

因为这些手段本质上都在做一件事：

> 少分配，少制造垃圾，少让 GC 干重活。

比如：

* `Span<T>`：减少切片复制和中间对象
* `ArrayPool<T>`：减少大数组频繁创建
* `ObjectPool<T>`：复用可重复使用的对象
* `StringBuilder`：减少字符串拼接产生的大量临时字符串

所以 GC 优化的核心思路，几乎永远不是“怎么更快回收”，而是：

> 怎么少制造必须被回收的东西。

### 怎么判断 GC 到底有没有成为瓶颈？

可以先从这些指标和工具入手。

#### 常见 API

```csharp
GC.GetTotalMemory(false)
GC.CollectionCount(0)
GC.CollectionCount(1)
GC.CollectionCount(2)
GC.GetGeneration(obj)
```

#### 常见工具

* `dotnet-counters`
* `dotnet-trace`
* `PerfView`
* Visual Studio Diagnostic Tools
* JetBrains `dotMemory`

比如看运行中进程：

```bash
dotnet-counters monitor --process-id <pid> System.Runtime
```

重点一般先盯这些方向：

* 分配速率是不是过高
* `Gen0` 回收是不是过于频繁
* `Gen2` 回收是不是偏多
* 托管堆总大小是不是持续走高

### `dotnet-counters` 实战排查怎么下手？

很多时候，线上第一步不是上来抓 dump，也不是先看代码，而是先确认：

* 现在到底是不是 GC 在忙
* 忙的是年轻代，还是老年代
* 是分配太猛，还是对象留得太久

`dotnet-counters` 很适合干这个事，因为它足够轻，拿来先看趋势很方便。

先找到进程：

```bash
dotnet-counters ps
```

再盯目标进程：

```bash
dotnet-counters monitor --process-id <pid> System.Runtime
```

通常会重点看这几类指标：

* `alloc-rate`
* `gc-heap-size`
* `gen-0-gc-count`
* `gen-1-gc-count`
* `gen-2-gc-count`
* `loh-size`
* `time-in-gc`

可以先用下面这种方式理解它们。

#### 1. `alloc-rate` 很高

这通常说明程序正在疯狂分配对象。

常见原因有：

* 热点循环频繁 `new`
* 大量中间集合和中间字符串
* 每次请求都在造临时大缓冲区

这时候先别急着盯 `Gen2`，通常先从“哪里在分配”入手更有效。

#### 2. `gen-0-gc-count` 涨得很快

这通常是短命对象很多。

单看这个不一定是坏事，因为年轻代本来就是拿来高频回收的。

真正该继续追的是：

* `alloc-rate` 是否也很高
* CPU 是否被 GC 明显吃掉
* 请求延迟是否已经被拖慢

#### 3. `gen-2-gc-count` 明显上涨

这个通常更值得警惕。

因为它常常意味着：

* 长生命周期对象偏多
* 对象提升比较明显
* 大对象分配频繁
* 缓存、静态引用、事件引用可能有问题

如果这里开始持续增长，就该继续往“谁在留住对象”这个方向查。

#### 4. `loh-size` 持续偏大

这说明大对象堆压力不小。

这时候优先看这些代码：

* 大 `byte[]`
* 大字符串
* 大块序列化和反序列化
* 图片、压缩、网络传输缓冲区

如果 `loh-size` 跟业务高峰同步抬升，又迟迟下不来，往往就要考虑池化、分块和流式处理了。

#### 5. `time-in-gc` 偏高

这个指标更接近“GC 对吞吐和延迟的真实影响”。

如果这里只是偶发波动，未必说明有问题。

但如果持续偏高，通常就说明：

* 回收已经不是背景噪音
* GC 正在明显抢 CPU
* 或者某些回收停顿已经开始影响业务

可以先记一个很实用的排查顺序：

1. 先看 `alloc-rate` 高不高。
2. 再看 `gen-0-gc-count` 和 `gen-2-gc-count` 谁更突出。
3. 如果 `gen-2-gc-count` 和 `loh-size` 都难看，优先怀疑长生命周期对象和大对象分配。
4. 如果 `time-in-gc` 也上来了，说明问题已经不只是“分配多”，而是开始真实影响运行时表现了。

也就是说，`dotnet-counters` 最适合回答的不是“到底哪一行代码有问题”，而是：

> GC 压力现在主要来自短命对象、大对象，还是对象存活时间过长。

### `dotnet-counters` 看完以后，怎么继续定位到代码？

这一步是很多人最容易卡住的地方。

因为 `dotnet-counters` 非常适合看趋势，却不适合直接点名具体方法。

它更像体检报告，能告诉当前问题像什么：

* 分配过多
* 大对象过多
* 老年代压力偏大
* GC 停顿已经开始影响业务

但它不会直接告诉“第 328 行代码有问题”。

所以真正的排查，通常要接着往下走。

一句话先记住：

> `dotnet-counters` 负责发现现象，`dotnet-trace` 负责找分配热点，`dotnet-dump` 负责找对象为什么下不来。

### 第一步：先判断这是“分配太猛”，还是“对象留太久”

看到指标后，先做分类。

#### 更像“分配太猛”的信号

通常表现为：

* `alloc-rate` 很高
* `gen-0-gc-count` 增长很快
* `gc-heap-size` 不一定长期居高不下

这种情况更像代码在持续制造大量短命对象。

常见代码模式包括：

* 热点循环里频繁 `new`
* 大量 `ToList()`、`ToArray()`
* 高频字符串拼接
* 频繁序列化、反序列化产生中间对象
* 装箱拆箱

这类问题下一步通常应该去抓“谁在分配”。

#### 更像“对象留太久”的信号

通常表现为：

* `gen-2-gc-count` 偏高
* `gc-heap-size` 长时间下不来
* `loh-size` 偏大且不容易回落

这种情况更像：

* 有对象一直被引用
* 缓存只进不出
* 静态字段或单例把对象挂住了
* 事件订阅没解绑

这类问题下一步通常应该去看“谁在引用它”。

### 第二步：怀疑高分配时，用 `dotnet-trace` 找分配热点

如果初步判断是“分配太猛”，下一步一般会用：

```text
dotnet-trace
```

先抓 trace：

```bash
dotnet-trace collect --process-id <pid>
```

如果想更明确盯运行时事件，也可以指定 provider：

```bash
dotnet-trace collect --process-id <pid> --providers Microsoft-Windows-DotNETRuntime:0x1C000080018:5
```

抓完以后，通常配合这些工具分析：

* `PerfView`
* Visual Studio
* 其他支持 trace 的分析工具

这一步最关键的是看两类东西：

* 哪些方法分配最多
* 哪些类型分配最多

也就是常说的：

* allocation stack
* allocation hot path

比如如果看到类似调用链：

```text
OrderAppService.GetOrders
 -> Mapper.Map
 -> Enumerable.ToList
 -> List<T>..ctor
```

那就说明问题的核心不是“GC 很忙”，而是：

* 这条业务路径在频繁创建中间集合
* 每次请求都在复制数据
* 分配模式本身就偏重

这种时候回到代码里，通常会查这些点：

* 能不能少一次 `ToList()`
* 能不能改成流式枚举
* 能不能复用缓冲区
* 能不能少创建中间 DTO

### 第三步：怀疑对象留存时，用 `dotnet-dump` 看引用链

如果更像“对象留太久”，`dotnet-trace` 就不是最优先的了。

因为这时候真正要回答的问题不是“谁在分配”，而是：

> 这些对象为什么还活着？

这时候更常见的做法是抓 dump：

```bash
dotnet-dump collect -p <pid>
```

然后进入分析：

```bash
dotnet-dump analyze <dump-file>
```

常用命令一般是这几个：

```text
dumpheap -stat
dumpheap -type Namespace.TypeName
gcroot <object-address>
```

可以这样理解它们的作用：

* `dumpheap -stat`：先看哪类对象最多、占内存最大
* `dumpheap -type`：再看某个具体类型的实例
* `gcroot`：最后看对象为什么还活着

其中最关键的往往是：

```text
gcroot
```

它会把引用链拉出来。

例如可能看到这种链路：

```text
static AppCache._orders
 -> List<Order>
 -> Order
```

或者：

```text
OrderEventBus._handlers
 -> SomeSubscriber
 -> LargeBufferHolder
```

这时候问题就能直接落回代码设计：

* 静态缓存没有淘汰策略
* 单例对象持有请求级数据
* 事件订阅没有解绑
* 集合只加不清

### 第四步：把指标翻译成最可能的代码味道

这一步很实用，适合排查时快速建立方向感。

#### `alloc-rate` 高

优先怀疑：

* 循环里反复 `new`
* 频繁 `Select().ToList()`
* 字符串大量拼接
* 热点路径上创建临时集合
* 临时大缓冲区反复申请

#### `gen-0-gc-count` 高

优先怀疑：

* 短命对象特别多
* 请求处理链上中间对象偏多
* 小对象分配非常密集

#### `gen-2-gc-count` 高

优先怀疑：

* 缓存设计有问题
* 对象被长期引用
* 静态字段或单例挂住了对象
* 对象生命周期本来很短，却被长生命周期对象持有

#### `loh-size` 高

优先怀疑：

* 大 `byte[]`
* 图片、压缩、序列化缓冲区
* 一次性整块读入大文件
* 没做池化的大对象分配

#### `time-in-gc` 高

优先怀疑：

* 前面几类问题已经叠加
* GC 不再只是“正常工作”，而是已经开始影响业务吞吐和延迟

### 一条最常用的实战排查链路

线上排查时，通常可以按这个顺序走：

1. 先用 `dotnet-counters` 确认问题属于高分配、对象留存，还是大对象压力。
2. 如果像高分配，继续用 `dotnet-trace` 或 `PerfView` 找分配热点方法。
3. 如果像对象留存，继续用 `dotnet-dump` 看 `dumpheap` 和 `gcroot`。
4. 拿到热点方法或引用链之后，再回到代码里改分配模式、缓存策略和对象生命周期。

整个过程里，最容易跑偏的一件事就是：

> 一看到 GC 指标难看，就马上怀疑 GC 自己有问题。

多数时候，GC 只是把代码里的分配模式和引用关系老老实实反映出来了。

真正该改的，往往还是：

* 哪些对象被造得太多
* 哪些对象被留得太久
* 哪些大对象本来可以复用却没有复用

### 一个更接近排查现场的 Demo

下面这个控制台程序可以直接跑，适合用来边看输出边观察计数器。

```csharp
using System;
using System.Collections.Generic;

PrintSnapshot("开始");

RunSmallObjectPressure();
PrintSnapshot("制造小对象压力后");

RunLargeObjectPressure();
PrintSnapshot("制造大对象压力后");

RunRetentionPressure();
PrintSnapshot("制造长生命周期对象后");

static void RunSmallObjectPressure()
{
    for (int i = 0; i < 2_000_000; i++)
    {
        _ = new SmallPayload(i, $"name-{i}");
    }
}

static void RunLargeObjectPressure()
{
    for (int i = 0; i < 1_000; i++)
    {
        _ = new byte[100_000];
    }
}

static void RunRetentionPressure()
{
    var cache = new List<byte[]>();

    for (int i = 0; i < 5_000; i++)
    {
        cache.Add(new byte[4096]);
    }

    Console.WriteLine($"cache 持有对象数：{cache.Count}");
}

static void PrintSnapshot(string title)
{
    Console.WriteLine($"==== {title} ====");
    Console.WriteLine($"TotalMemory: {GC.GetTotalMemory(false) / 1024 / 1024.0:F2} MB");
    Console.WriteLine($"Gen0 Count : {GC.CollectionCount(0)}");
    Console.WriteLine($"Gen1 Count : {GC.CollectionCount(1)}");
    Console.WriteLine($"Gen2 Count : {GC.CollectionCount(2)}");
    Console.WriteLine();
}

sealed record SmallPayload(int Id, string Name);
```

这个 demo 不适合拿来做严格 benchmark，但很适合帮助建立三种直觉：

* 小对象多了，年轻代会很忙
* 大对象多了，老年代压力会上来
* 对象只要一直被引用，GC 就没法替忙

### 什么时候该优先怀疑 `LOH`？

如果项目里经常有这些行为，就该重点看大对象堆：

* 反复分配几十 KB 到几百 KB 的字节数组
* 大 JSON、大图片、大压缩包一次性整块读入
* 频繁做大块序列化和反序列化
* 视频、音频、网络协议缓冲区反复创建

常见应对思路一般是：

* 复用缓冲区
* 分块处理
* 避免一次性把整份数据全部载入内存
* 检查是否可以用流式 API

### GC 模式要不要关心？

要，但通常是在服务级别，而不是日常业务代码里随手折腾。

常见会看到这些词：

* `Workstation GC`
* `Server GC`
* 后台 GC
* 延迟模式

先记结论就够了：

* 客户端和服务端负载模型不同，GC 策略也会不同
* 大多数项目先用默认配置，只有出现明确指标问题再调
* 没有监控数据支撑时，不建议拍脑袋改 GC 模式

### 关于“GC 之后内存为什么没立刻掉下去”这件事

这是非常常见的疑问。

看到托管堆或进程内存没有立刻明显下降，不一定就代表 GC 失效了。

原因可能包括：

* 对象其实还活着
* 运行时保留了一部分堆空间，后续继续复用
* 回收了对象，但操作系统层面的工作集不一定立刻按想象变化

所以判断有没有问题，不能只看任务管理器里某一瞬间的数字。

更应该结合：

* 托管堆趋势
* 对象存活趋势
* `Gen2` 次数
* 分配速率
* 是否有明显引用链把对象长期挂住

### 一组非常实用的 GC 经验

#### 1. `Gen0` 多，不一定是坏事

如果程序分配很多短命小对象，`Gen0` 回收本来就会比较活跃。

真正值得紧张的是：

* 分配速率高到把 CPU 吃掉
* `Gen2` 也开始频繁
* 请求延迟被回收停顿明显拉高

#### 2. `Gen2` 多，通常更值得关注

因为这往往说明：

* 长生命周期对象很多
* 对象提升明显
* 大对象压力偏大
* 缓存、静态引用、事件引用可能有问题

#### 3. 不要把 `GC.Collect()` 当救火按钮

真问题通常不是“没收”，而是：

* 还被引用
* 分配太猛
* 大对象太多
* 生命周期设计不合理

#### 4. `Dispose` 很重要，但它不等于 GC

这两者负责的是不同问题：

* `Dispose`：尽快释放外部资源
* `GC`：回收托管对象占用的内存

#### 5. GC 优化的第一原则几乎永远是“减少分配”

这条比记任何 API 都更值钱。

### 总结

把 `GC` 真正理解透，关键不是记住多少术语，而是抓住这几条主线：

* `GC` 通过可达性判断对象是否还能活
* 分代回收的核心目标，是让短命对象尽快在年轻代死掉
* `Gen2` 和 `LOH` 一旦压力变大，停顿和成本通常都会更明显
* 终结器不是及时释放机制，外部资源应靠 `IDisposable`
* 真正有效的优化，大多发生在“减少分配、缩短不必要的存活时间、避免大对象频繁创建”这几个层面

如果只记一句话，最值得记的是：

> `.NET GC` 最大的价值，不是替代码“自动兜底一切”，而是让大多数对象生命周期管理变得足够便宜；真正决定程序稳不稳的，依然是对象分配和引用设计本身。
