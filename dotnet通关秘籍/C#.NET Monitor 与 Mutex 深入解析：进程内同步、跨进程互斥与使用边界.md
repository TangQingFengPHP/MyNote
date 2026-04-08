### 简介

在 `.NET` 里提到同步，很多人第一反应通常是：

```csharp
lock (_gate)
{
    // 临界区
}
```

这没问题。

但只要你继续往下挖，很快就会碰到两个更底层的名字：

```csharp
Monitor
Mutex
```

它们都能做“互斥”，但解决的问题并不是同一类。

一句话先说透：

> `Monitor` 是进程内线程同步的默认基础设施，`Mutex` 则更偏跨进程互斥的操作系统级工具。

所以这篇文章重点不是只列 API，而是讲清楚：

* `lock` 和 `Monitor` 到底是什么关系；
* `Monitor.Wait / Pulse` 真正解决什么问题；
* `Mutex` 为什么不能简单理解成“更高级的锁”；
* 什么时候该用 `Monitor`，什么时候才值得上 `Mutex`；
* 它们和 `System.Threading.Lock`、`SemaphoreSlim`、`ReaderWriterLockSlim` 的边界是什么。

### 先别急着比，先看它们各自解决什么问题

假设你现在有两个线程同时改一个共享字段：

```csharp
_count++;
```

如果不做同步，问题大家都知道：

* 竞态条件
* 数据错乱
* 最终结果不稳定

这时候你要的是：

* 同一时刻只让一个线程进入临界区

这就是 `Monitor` 最常解决的问题。

但如果你的问题变成：

* 两个不同进程不能同时写同一个全局资源
* 或者你想防止程序多开

那你要解决的已经不是“进程内线程同步”了。

这时候更常见的答案才会是：

* `Mutex`

所以这两者的第一层差异不是 API，而是：

* 作用范围不一样

### `Monitor` 到底是什么？

可以先用一句最直白的话理解：

> `Monitor` 是 `.NET` 里围绕对象锁实现的进程内互斥机制。

它位于：

```csharp
System.Threading
```

而且有一个特别重要的事实：

* C# 里的 `lock` 基本就是它的语法糖

也就是说，下面这段代码：

```csharp
lock (_gate)
{
    // 临界区
}
```

近似可以理解成：

```csharp
Monitor.Enter(_gate);
try
{
    // 临界区
}
finally
{
    Monitor.Exit(_gate);
}
```

所以如果你过去一直在用 `lock`，其实你已经在用 `Monitor` 了，只是平时没有直接写它的名字。

### 为什么大多数进程内同步场景优先想到 `Monitor`？

因为它够直接，也够高效。

它适合的就是最普通的互斥需求：

* 保护一段共享状态
* 同一时刻只允许一个线程进入
* 不涉及跨进程

例如：

```csharp
private readonly object _gate = new();
private int _count;

public void Increment()
{
    lock (_gate)
    {
        _count++;
    }
}
```

这是最典型、也最常见的用法。

你真正应该先记住的是：

* `Monitor` 不神秘
* 它就是 `lock` 背后的基础机制

### `Monitor.Enter / Exit` 和 `lock` 到底怎么选？

绝大多数时候，直接用：

```csharp
lock
```

就够了。

因为：

* 语法更短
* 异常安全更自然
* 不容易漏掉 `Exit`

只有在你需要这些能力时，才更可能直接碰 `Monitor`：

* `TryEnter`
* `Wait`
* `Pulse`
* `PulseAll`

也就是说，`Monitor` 真正值钱的地方，不是拿它替代 `lock`，而是它提供了更细粒度的同步和线程协作能力。

### `Monitor.TryEnter` 为什么值得单独讲？

因为它解决的不是“加锁”本身，而是：

* 不想无限等锁

最常见的写法大概是：

```csharp
var lockTaken = false;
try
{
    Monitor.TryEnter(_gate, TimeSpan.FromSeconds(1), ref lockTaken);

    if (!lockTaken)
    {
        return;
    }

    // 临界区
}
finally
{
    if (lockTaken)
    {
        Monitor.Exit(_gate);
    }
}
```

这类写法适合：

* 你不能接受线程无限期等待
* 你更希望超时失败，而不是一直挂着

所以 `TryEnter` 的价值，不是“更底层”，而是：

* 它给了你一个超时和失败分支

### `Monitor.Wait / Pulse / PulseAll` 真正是干什么的？

这是最容易被讲得很抽象的一组 API。

其实可以先把它理解成一句话：

> 它们不是用来“再加一层锁”，而是用来让持有同一把锁的线程彼此协调时机。

一个最典型的场景就是：

* 生产者-消费者

例如队列为空时，消费者不要一直空转，而是等生产者通知。

```csharp
public sealed class SimpleQueue<T>
{
    private readonly Queue<T> _queue = new();
    private readonly object _gate = new();

    public void Enqueue(T item)
    {
        lock (_gate)
        {
            _queue.Enqueue(item);
            Monitor.Pulse(_gate);
        }
    }

    public T Dequeue()
    {
        lock (_gate)
        {
            while (_queue.Count == 0)
            {
                Monitor.Wait(_gate);
            }

            return _queue.Dequeue();
        }
    }
}
```

这段代码真正发生的事是：

* 消费者发现队列空了
* `Wait` 释放当前锁并进入等待
* 生产者入队后 `Pulse`
* 被唤醒的消费者再重新竞争锁并继续执行

这里最关键的不是 API 名字，而是这个语义：

* `Wait` 会释放锁
* `Pulse` 只是发通知，不会替对方执行
* 被唤醒线程要重新拿到锁之后，才会继续往下走

### 为什么 `Wait` 外面通常要写 `while`，不是 `if`？

这是一个很实用的细节。

因为被唤醒不等于条件一定满足。

更稳的写法总是：

```csharp
while (_queue.Count == 0)
{
    Monitor.Wait(_gate);
}
```

而不是：

```csharp
if (_queue.Count == 0)
{
    Monitor.Wait(_gate);
}
```

原因很简单：

* 线程被唤醒后，条件可能已经又变了
* 或者被唤醒的线程不止一个

所以 `while` 本质上是在做：

* 条件重检

这不是语法细节，而是正确性保证的一部分。

### `Monitor` 最容易踩哪些坑？

#### 1. 锁 `this`

```csharp
lock (this)
{
}
```

这几乎总不是一个好主意。

因为外部代码也可能拿到 `this` 来锁。

#### 2. 锁字符串

```csharp
lock ("my-lock")
{
}
```

这更危险，因为字符串驻留会让锁对象共享得超出你的预期。

#### 3. 手写 `Monitor.Enter` 却忘了 `finally`

这会让异常路径直接变成死锁制造机。

#### 4. 把 `Wait/Pulse` 当成跨锁通信

它们必须围绕同一个同步对象使用，不是随便找两把锁互相通知。

### `Mutex` 到底是什么？

如果说 `Monitor` 更像 `.NET` 运行时里偏进程内的同步机制，那 `Mutex` 更像：

> 操作系统级的互斥体。

它同样位于：

```csharp
System.Threading
```

但它的定位明显更重。

最值得先记住的是：

* `Mutex` 不只是给线程用的
* 它最有价值的场景通常是跨进程

最常见的使用方式大概是：

```csharp
using var mutex = new Mutex();

mutex.WaitOne();
try
{
    // 临界区
}
finally
{
    mutex.ReleaseMutex();
}
```

这个语法上看起来也像“加锁”，但别被表面骗了。

它和 `Monitor` 最大的区别，不是方法名，而是：

* 它是内核对象
* 它可以做命名互斥
* 它可以跨进程

### `Mutex` 真正值钱的场景是什么？

最经典的就是：

* 防止程序多开

例如：

```csharp
using var mutex = new Mutex(true, @"Global\MyAppName", out var createdNew);

if (!createdNew)
{
    Console.WriteLine("应用已经在运行");
    return;
}

Console.ReadLine();
```

这里的关键不是“锁住一段代码”，而是：

* 操作系统范围内有了一个带名字的互斥体

只要别的进程也用同一个名字，它们拿到的就是同一个系统级同步对象。

这就是 `Monitor` 做不到、而 `Mutex` 能做的事。

### 为什么大多数日常同步不推荐优先用 `Mutex`？

因为它太重了。

更务实地说：

* `Monitor` 更适合进程内短临界区
* `Mutex` 更适合跨进程互斥

如果你只是想保护内存里的一个字段、一段集合操作、一段缓存更新逻辑，却直接上 `Mutex`，通常就是：

* 能用
* 但不值

原因很现实：

* 系统调用开销更高
* 上下文切换成本更重
* 使用复杂度也更高

所以绝大多数进程内同步需求，还是应优先回到：

* `lock`
* `Monitor`

### `Mutex` 最容易踩哪些坑？

#### 1. 忘记 `ReleaseMutex`

这个和忘记释放锁一样致命。

#### 2. 跨线程释放

`Mutex` 有线程所有权语义，不是哪个线程都能随便放。

#### 3. 忽略 `AbandonedMutexException`

如果持有 `Mutex` 的线程异常退出，后续线程可能会遇到被遗弃互斥体异常。

这时候真正危险的不是“抛了个异常”，而是：

* 共享资源状态可能已经不一致了

所以它不是一个可以随便吞掉的信号。

### `Monitor` 和 `Mutex` 到底怎么选？

可以先看这张压缩表：

| 维度 | `Monitor` | `Mutex` |
| --- | --- | --- |
| 主要作用域 | 进程内 | 可跨进程 |
| 常见写法 | `lock` / `Monitor.Enter` | `WaitOne` / `ReleaseMutex` |
| 性能和开销 | 更轻 | 更重 |
| 线程协作 | `Wait/Pulse` | 不擅长 |
| 典型场景 | 内存状态保护 | 单实例程序、跨进程互斥 |

如果只记一句话：

> 进程内同步先想 `Monitor`，跨进程互斥才认真考虑 `Mutex`。

### 它们和 `System.Threading.Lock`、`SemaphoreSlim`、`ReaderWriterLockSlim` 的边界是什么？

这组对比也很重要。

#### `Monitor` vs `System.Threading.Lock`

在 `.NET 9 + C# 13` 下，官方更推荐新的 `System.Threading.Lock` 作为默认同步写法。

但这不等于 `Monitor` 失效了。

更准确地说：

* 普通互斥写法，优先考虑 `System.Threading.Lock`
* 理解 `lock` 背后的传统机制，还是离不开 `Monitor`
* 真要用 `Wait/Pulse/TryEnter`，你还是会回到 `Monitor`

#### `Monitor` vs `SemaphoreSlim`

`Monitor` 更像：

* 一次只允许一个线程进

`SemaphoreSlim` 更像：

* 允许 `N` 个并发
* 并且支持异步等待

所以只要场景里出现：

* `await`
* 限制并发数而不是单纯互斥

通常就别继续往 `Monitor` 上想了。

#### `Monitor` vs `ReaderWriterLockSlim`

前者适合普通互斥。

后者适合：

* 读多写少

也就是说，`ReaderWriterLockSlim` 是在更复杂的读写模型上优化，而 `Monitor` 仍然是最普通的互斥基础设施。

### 一个更贴近项目的案例怎么理解？

可以看两个非常典型的场景。

#### 场景一：进程内缓存更新

```csharp
private readonly object _gate = new();
private Dictionary<int, string> _cache = new();

public string? Get(int id)
{
    lock (_gate)
    {
        return _cache.TryGetValue(id, out var value) ? value : null;
    }
}
```

这里最自然的答案就是：

* `Monitor` / `lock`

因为它只是进程内普通互斥。

#### 场景二：桌面程序只能开一个实例

```csharp
using var mutex = new Mutex(true, @"Global\DemoApp", out var createdNew);

if (!createdNew)
{
    return;
}

// 应用主逻辑
```

这里最自然的答案就是：

* `Mutex`

因为你已经进入了跨进程同步。

### 一个非常务实的选择顺序

如果你在做同步原语选型，可以先按这个顺序判断：

1. 只是普通进程内互斥吗？
2. 如果是，优先考虑 `System.Threading.Lock` / `lock`
3. 如果需要 `Wait/Pulse/TryEnter` 这类能力，再看 `Monitor`
4. 如果是异步互斥或并发限制，看 `SemaphoreSlim`
5. 如果是读多写少，看 `ReaderWriterLockSlim`
6. 只有在明确需要跨进程互斥时，再考虑 `Mutex`

这个顺序很重要。

因为很多时候不是“不会用 `Mutex`”，而是一开始就把问题想重了。

### 面试里怎么答比较到位？

如果面试官问：

“`Monitor` 和 `Mutex` 的区别是什么？”

一个比较自然的回答可以是：

> `Monitor` 更适合进程内线程同步，也是 C# `lock` 背后的基础机制；它轻量、常用，而且支持 `TryEnter`、`Wait`、`Pulse` 这类线程协作能力。`Mutex` 是更重的操作系统级互斥体，最大的价值是跨进程同步，比如防止程序多开或控制多个进程访问同一个全局资源。大多数日常同步场景优先考虑 `Monitor`，而不是 `Mutex`。

如果继续追问“那 `Wait/Pulse` 是干什么的”，可以答：

> 它们不是单纯再加一层锁，而是让持有同一把锁的线程之间做条件协调。最典型的就是生产者-消费者：队列为空时消费者 `Wait`，生产者入队后 `Pulse` 唤醒等待线程。

如果再追问“最大的坑是什么”，优先答这三个：

* 手写 `Monitor.Enter` 却忘记 `finally`
* `Wait` 外面用 `if` 而不是 `while`
* 进程内同步却误用 `Mutex`

### 总结

`Monitor` 和 `Mutex` 最值得记住的，不是它们都能“加锁”，而是它们解决的问题层级不一样：

> `Monitor` 解决的是进程内线程如何互斥和协作，`Mutex` 解决的是更重、更偏系统级的互斥问题，尤其是跨进程。

如果你只想记住几句话，可以记这几条：

* `lock` 背后基本就是 `Monitor`；
* `Monitor` 是进程内同步默认答案之一；
* `Wait/Pulse` 真正值钱的地方是线程协作，不只是互斥；
* `Mutex` 最大价值是跨进程，不是“更高级的 lock”；
* 进程内问题别轻易上 `Mutex`，通常不值。
