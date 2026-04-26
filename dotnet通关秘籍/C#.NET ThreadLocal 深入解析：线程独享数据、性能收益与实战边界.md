### 简介

多线程代码里最麻烦的一个点，不是“怎么开线程”，而是“数据到底该不该共享”。

很多并发问题，本质上都不是线程太多，而是：

* 好几个线程同时改同一份数据
* 于是开始加锁
* 锁一多，性能又掉下去

这时候就会发现，有些数据其实根本没必要共享。

比如：

* 每个线程自己的计数器
* 每个线程自己的 `Random`
* 每个线程临时复用的 `StringBuilder`
* 每个线程自己的缓冲区

这种场景下，`ThreadLocal<T>` 就很合适。

一句话先说透：

> `ThreadLocal<T>` 的作用不是“让共享数据更安全”，而是“让数据干脆别共享”，直接给每个线程一份独立副本。

这篇文章重点会讲清楚几件事：

* `ThreadLocal<T>` 到底解决什么问题
* 它和 `lock`、`Interlocked`、`AsyncLocal<T>`、`[ThreadStatic]` 的边界是什么
* 为什么它在 `async/await` 里经常踩坑
* 实战里该怎么写，怎么收尾，怎么避坑

### `ThreadLocal<T>` 是什么？

`ThreadLocal<T>` 位于：

```csharp
System.Threading
```

它的核心语义很简单：

* 同一个 `ThreadLocal<T>` 实例
* 不同线程访问 `.Value`
* 拿到的是各自线程自己的值

也就是说，看起来像一个变量，实际上一份变量会被拆成很多份，按线程隔离保存。

可以把它想成“每个线程一个独立抽屉”：

* 线程 A 往抽屉里放了 `1`
* 线程 B 往抽屉里放了 `100`
* 彼此互相看不见，也互不影响

所以它不是同步工具，不负责协调线程之间的先后顺序。

它做的是另一件事：

> 既然这份状态本来就不该共享，那就直接按线程拆开存。

### 为什么需要它？

先看一个最常见的问题。

```csharp
int counter = 0;

Parallel.For(0, 100000, _ =>
{
    counter++;
});
```

这段代码有竞态条件，结果通常不对。

因为 `counter++` 不是原子操作，它至少包含：

* 读取旧值
* 加一
* 写回新值

多个线程同时做这件事，就会互相覆盖。

传统做法一般有两种：

#### 1. 加锁

```csharp
int counter = 0;
object sync = new();

Parallel.For(0, 100000, _ =>
{
    lock (sync)
    {
        counter++;
    }
});
```

优点是简单、正确。

缺点也很直接：

* 线程会竞争同一把锁
* 并发越高，等待越明显

#### 2. 用原子操作

```csharp
int counter = 0;

Parallel.For(0, 100000, _ =>
{
    Interlocked.Increment(ref counter);
});
```

这个写法通常比 `lock` 更轻。

但本质上，所有线程还是在争同一个计数器。

如果业务允许“先局部累计，最后再合并”，那还有第三种思路：

#### 3. 每个线程记自己的

```csharp
using var localCounter = new ThreadLocal<int>(() => 0, trackAllValues: true);

Parallel.For(0, 100000, _ =>
{
    localCounter.Value++;
});

int total = localCounter.Values.Sum();
```

这就是 `ThreadLocal<T>` 的典型用法：

* 运行时不争同一个变量
* 先在线程内部局部处理
* 最后统一汇总

很多高性能代码的思路，都是这一套。

### 它的工作方式是什么？

`ThreadLocal<T>` 有两个很关键的特点。

#### 1. 按线程隔离

同一个 `ThreadLocal<int>`，不同线程看到的是不同的 `.Value`。

```text
线程 A -> local.Value = 10
线程 B -> local.Value = 20
线程 C -> local.Value = 30
```

这三个值不会互相覆盖。

#### 2. 延迟初始化

通常会这样创建：

```csharp
var local = new ThreadLocal<Random>(() => new Random());
```

这里的工厂方法不是创建一次，而是：

* 哪个线程第一次访问 `.Value`
* 就在那个线程里执行一次工厂方法
* 后续该线程再次访问，直接拿缓存值

也就是说，初始化逻辑是“每线程一次”，不是“全局一次”。

### 最基础的 Demo：每个线程各用各的值

```csharp
using System;
using System.Threading;

using var localNumber = new ThreadLocal<int>(() =>
{
    Console.WriteLine($"线程 {Thread.CurrentThread.ManagedThreadId} 初始化");
    return 0;
});

Thread t1 = new(() =>
{
    localNumber.Value++;
    localNumber.Value++;
    Console.WriteLine($"线程 {Thread.CurrentThread.ManagedThreadId} 的值：{localNumber.Value}");
});

Thread t2 = new(() =>
{
    localNumber.Value += 10;
    Console.WriteLine($"线程 {Thread.CurrentThread.ManagedThreadId} 的值：{localNumber.Value}");
});

t1.Start();
t2.Start();
t1.Join();
t2.Join();
```

输出大致会是这样：

```text
线程 8 初始化
线程 9 初始化
线程 8 的值：2
线程 9 的值：10
```

这里要注意两点：

* 初始化逻辑分别在两个线程里各跑了一次
* 两个线程虽然访问的是同一个 `localNumber` 变量，但值完全独立

### 常用 API 先过一遍

#### 构造函数

```csharp
new ThreadLocal<T>()
new ThreadLocal<T>(Func<T> valueFactory)
new ThreadLocal<T>(Func<T> valueFactory, bool trackAllValues)
```

最常见的是后两种。

#### `Value`

```csharp
local.Value
```

读取或设置当前线程对应的值。

#### `IsValueCreated`

```csharp
local.IsValueCreated
```

表示“当前线程”是否已经创建过值。

这个判断不是全局的，而是当前线程视角。

#### `Values`

```csharp
local.Values
```

获取所有已跟踪线程的值。

但要注意，只有在构造时传了：

```csharp
trackAllValues: true
```

这个属性才有意义。

#### `Dispose()`

```csharp
local.Dispose();
```

释放 `ThreadLocal<T>` 持有的资源和跟踪结构。

只要生命周期不是跟进程同生共死，通常都应该在不用时及时释放。

### Demo 2：高并发计数，先局部累加再汇总

这个例子最能说明 `ThreadLocal<T>` 的实际价值。

```csharp
using System;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

using var localCounter = new ThreadLocal<int>(() => 0, trackAllValues: true);

Parallel.For(0, 1_000_000, _ =>
{
    localCounter.Value++;
});

int total = localCounter.Values.Sum();

Console.WriteLine($"总数：{total}");
Console.WriteLine($"线程份数：{localCounter.Values.Count}");
Console.WriteLine($"每个线程的局部计数：{string.Join(", ", localCounter.Values)}");
```

这种模式的优点是：

* 运行过程中几乎没有共享写竞争
* 每个线程只改自己的局部值
* 最后再做一次集中合并

这类场景通常包括：

* 批量统计
* 并行累加
* 分片处理后的结果归并

但这里也要把边界说清楚：

> 如果业务要求“每次加一之后，全局值立刻可见”，那就不能用 `ThreadLocal<T>` 代替 `Interlocked`。

因为它本来就不是共享计数器。

### Demo 3：给每个线程一个专属 `Random`

`Random` 这个场景很经典。

老代码里经常会看到两种不太理想的写法：

#### 写法 1：多个线程共用一个 `Random`

```csharp
Random random = new();
```

这会有线程安全问题。

#### 写法 2：每次都 `new Random()`

```csharp
int value = new Random().Next();
```

这种写法会反复创建对象，而且在某些旧代码场景里还可能出现种子过近的问题。

用 `ThreadLocal<Random>` 会更稳：

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

using var localRandom = new ThreadLocal<Random>(() =>
    new Random(HashCode.Combine(Environment.TickCount, Thread.CurrentThread.ManagedThreadId)));

Parallel.For(0, 8, i =>
{
    int value = localRandom.Value.Next(1, 100);
    Console.WriteLine($"线程 {Thread.CurrentThread.ManagedThreadId} -> {value}");
});
```

这种模式的价值是：

* 每个线程一个 `Random`
* 不需要对同一个实例加锁
* 也不用每次临时创建

当然，如果只是普通场景下拿随机数，现代 `.NET` 里也常常可以直接用：

```csharp
Random.Shared
```

但如果场景明确需要“线程私有状态”，`ThreadLocal<Random>` 依然有意义。

### Demo 4：线程私有对象复用，减少临时分配

除了计数器，`ThreadLocal<T>` 另一个很常见的用途，就是对象复用。

比如每个线程都频繁拼接字符串，如果每次都 `new StringBuilder()`，分配会比较多。

这时候可以给每个线程准备一个自己的 `StringBuilder`：

```csharp
using System;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

using var localBuilder = new ThreadLocal<StringBuilder>(() => new StringBuilder(256));

Parallel.For(0, 10, i =>
{
    StringBuilder sb = localBuilder.Value;
    sb.Clear();
    sb.Append("任务编号：").Append(i)
      .Append("，线程：").Append(Thread.CurrentThread.ManagedThreadId);

    Console.WriteLine(sb.ToString());
});
```

这种用法很适合：

* 文本拼接缓冲区
* 小对象缓存
* 线程私有临时上下文
* 高并发下减少短命对象分配

不过也别走偏：

> `ThreadLocal<T>` 适合放“线程用完即可复用”的对象，不适合随手塞很大的长期对象。

原因后面会讲。

### 为什么它在 `async/await` 里容易出事？

这是 `ThreadLocal<T>` 最常见的误区之一。

先看例子：

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

using var localText = new ThreadLocal<string>(() => "未设置");

async Task DemoAsync()
{
    localText.Value = "开始阶段";
    Console.WriteLine($"await 前，线程 {Thread.CurrentThread.ManagedThreadId}，值：{localText.Value}");

    await Task.Delay(100);

    Console.WriteLine($"await 后，线程 {Thread.CurrentThread.ManagedThreadId}，值：{localText.Value}");
}

await DemoAsync();
```

很多人第一次看到这里会困惑：

* `await` 前明明赋值了
* 为什么 `await` 后可能读不到刚才那个值？

原因很简单：

* `ThreadLocal<T>` 绑定的是线程
* `async/await` 绑定的是异步执行流
* `await` 之后，续体不一定还回到原来的线程

线程一旦变了，`ThreadLocal<T>` 对应的那份值也就变了。

所以一定要记住：

> `ThreadLocal<T>` 适合线程上下文，不适合异步调用链上下文。

如果需求是“跨 `await` 继续带着这份值走”，应该看的是：

```csharp
AsyncLocal<T>
```

### `ThreadLocal<T>` 和 `AsyncLocal<T>` 到底怎么选？

先把一句话讲明白：

* `ThreadLocal<T>`：按线程隔离
* `AsyncLocal<T>`：按异步调用链隔离

举个直观点的判断方式：

#### 适合 `ThreadLocal<T>` 的场景

* 每个线程自己的缓存对象
* 每个线程自己的临时缓冲区
* 并行局部统计，最后汇总
* 依赖线程隔离，而不是调用链传递

#### 适合 `AsyncLocal<T>` 的场景

* 请求上下文
* 链路追踪 ID
* 日志作用域
* 需要跨 `await` 传递的上下文信息

如果是 ASP.NET Core 请求上下文、日志上下文之类的场景，基本都不该先想到 `ThreadLocal<T>`。

### 它和 `[ThreadStatic]` 有什么区别？

`[ThreadStatic]` 也是线程本地存储，但两者不是一回事。

先看 `[ThreadStatic]` 的样子：

```csharp
[ThreadStatic]
private static int _value;
```

它的特点是：

* 只能修饰 `static` 字段
* 每个线程各有一份字段值
* 用法很轻，但能力比较原始

而 `ThreadLocal<T>` 的优势在于：

* 支持泛型
* 支持工厂方法延迟初始化
* 支持 `IsValueCreated`
* 支持在 `trackAllValues: true` 时拿到所有线程值
* 支持 `Dispose()`

可以简单理解成：

* `[ThreadStatic]` 更接近“语法级线程字段”
* `ThreadLocal<T>` 更接近“功能完整的线程本地容器”

如果只是极简单、极底层的线程静态字段，[`ThreadStatic]` 可以考虑。

如果需要初始化逻辑、生命周期管理、值跟踪，通常 `ThreadLocal<T>` 更合适。

### 它和 `lock`、`Interlocked` 的边界是什么？

这三个东西经常被拿来一起比较，但职责并不一样。

#### `ThreadLocal<T>` 不是锁

它不负责保护共享状态。

如果多线程真的要改同一个 `Dictionary`、同一个对象、同一个队列，还是得靠：

* `lock`
* `Monitor`
* `ReaderWriterLockSlim`
* 并发集合

`ThreadLocal<T>` 只能解决“别共享”这一类问题。

#### 它也不是原子操作替代品

`Interlocked` 解决的是：

* 一份共享变量
* 需要原子读改写

而 `ThreadLocal<T>` 解决的是：

* 多份局部变量
* 线程先各算各的
* 最后再归并

所以两者往往不是谁替代谁，而是看业务语义。

#### 一个简单判断标准

如果问题是：

* “同一个值必须被所有线程共同看到”

优先看共享同步方案。

如果问题是：

* “每个线程其实只需要自己的那份状态”

这才是 `ThreadLocal<T>` 的地盘。

### `trackAllValues: true` 有什么代价？

这个参数很有用，但不是白给的。

```csharp
var local = new ThreadLocal<int>(() => 0, trackAllValues: true);
```

开启后，`ThreadLocal<T>` 会额外跟踪所有线程创建过的值，方便最后通过 `Values` 做汇总。

代价是：

* 会增加一些管理开销
* 会让这些线程本地值更容易被整体持有和观察

所以如果根本不需要 `Values`，就别顺手开着。

能不用就不用，只有在“确实需要全量汇总”时再开启。

### 生命周期问题一定要重视

`ThreadLocal<T>` 很容易被误用成“顺手一放就完事”。

这在短命线程里可能没什么感觉，但在线程池环境里，问题就会明显很多。

因为线程池线程不是用完就销毁，它们会长期存在。

如果 `ThreadLocal<T>` 里放的是：

* 大对象
* 缓冲区
* 数据库连接
* 大集合

那这些对象可能会在线程上挂很久。

所以实战里要注意几条：

#### 1. 不要把它当万能缓存

线程本地不等于免费缓存。

线程数一多，内存占用会按线程倍增。

#### 2. 不再使用时及时 `Dispose()`

```csharp
local.Dispose();
```

特别是临时创建的 `ThreadLocal<T>`，别让它一直悬着。

#### 3. 对象越大，越要谨慎

一个线程放 `1MB` 缓冲区不算什么，32 个线程就是 `32MB`，64 个线程就是 `64MB`。

这还只是单个 `ThreadLocal<T>`。

#### 4. 线程池线程复用会放大问题

本来以为“任务结束对象就没了”，实际可能只是任务结束了，线程还在，值也还在。

### 常见误区

#### 误区 1：`ThreadLocal<T>` 能替代所有锁

不能。

如果状态本身必须共享，它就帮不上忙。

#### 误区 2：`ThreadLocal<T>` 适合请求上下文

在同步、固定线程模型里也许还凑合，但现代 `.NET` 大量场景是 `async/await`，这时通常应该先看 `AsyncLocal<T>`。

#### 误区 3：`ThreadLocal<T>` 一定更快

不一定。

如果：

* 数据量很小
* 线程数不多
* 初始化对象很贵
* 最后合并本身也有成本

那 `ThreadLocal<T>` 未必真的占优。

性能结论要看场景，不看想象。

#### 误区 4：在线程池任务里放任何对象都没问题

这类代码最容易把小优化写成长期内存占用。

特别是缓存数组、缓冲区、连接对象这类资源，要非常克制。

### 什么时候适合用 `ThreadLocal<T>`？

比较适合：

* 并行计算中的线程局部累计
* 每线程独立的 `Random`
* 每线程复用的 `StringBuilder`
* 每线程的临时缓冲区
* 某些依赖线程模型的底层组件状态

不太适合：

* ASP.NET Core 请求上下文
* 需要跨 `await` 传递的数据
* 真正的全局共享状态
* 大对象长期缓存
* 可以直接用参数传递的数据

### 一段更贴近实战的综合示例

下面这个例子演示一种很常见的模式：

* 并行处理大量输入
* 每个线程复用自己的 `StringBuilder`
* 每个线程维护自己的成功计数
* 最后统一汇总

```csharp
using System;
using System.Collections.Concurrent;
using System.Linq;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

var inputs = Enumerable.Range(1, 10000).ToArray();
var outputs = new ConcurrentBag<string>();

using var localBuilder = new ThreadLocal<StringBuilder>(() => new StringBuilder(128));
using var localSuccessCount = new ThreadLocal<int>(() => 0, trackAllValues: true);

Parallel.ForEach(inputs, item =>
{
    StringBuilder sb = localBuilder.Value;
    sb.Clear();

    sb.Append("item=").Append(item)
      .Append(", thread=").Append(Thread.CurrentThread.ManagedThreadId);

    outputs.Add(sb.ToString());

    localSuccessCount.Value++;
});

int totalSuccess = localSuccessCount.Values.Sum();

Console.WriteLine($"输出数量：{outputs.Count}");
Console.WriteLine($"成功数量：{totalSuccess}");
Console.WriteLine($"参与线程数：{localSuccessCount.Values.Count}");
```

这段代码里：

* `ConcurrentBag<string>` 负责承接真正共享的输出结果
* `ThreadLocal<StringBuilder>` 负责减少线程内临时分配
* `ThreadLocal<int>` 负责线程内局部累计

这就是比较典型的组合拳：

> 该共享的共享，该局部的局部，不要什么都硬塞进同一种并发手段里。

### 总结

`ThreadLocal<T>` 最核心的价值，不是“线程安全”这四个字本身，而是换一种思路：

* 不是想办法保护共享
* 而是尽量减少共享

只要场景满足“每个线程维护自己的状态”，它通常会比锁更自然，也往往更高效。

但边界也很明确：

* 它不保护共享数据
* 它不适合跨 `await`
* 它可能带来线程级内存常驻
