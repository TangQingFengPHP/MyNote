### 简介

`Interlocked.Increment` 是 `.NET` 中一个重要的线程安全操作方法，用于以原子方式递增变量的值。它位于 `System.Threading` 命名空间中，提供了一种轻量级的线程同步机制。

这些方法包括：

| 方法                                                    | 作用                         |
| ------------------------------------------------------- | ---------------------------- |
| `Increment(ref int location)`                           | 原子 +1                      |
| `Decrement(ref int location)`                           | 原子 -1                      |
| `Add(ref int location, int value)`                      | 原子加指定值                 |
| `Exchange(ref T location, T value)`                     | 原子交换值                   |
| `CompareExchange(ref T location, T value, T comparand)` | CAS 操作（Compare-And-Swap） |
| `Read(ref long location)`                               | 原子读取 long 值             |

这些操作都是 线程安全且无锁 (`lock-free`) 的。

### 核心概念与作用

#### 原子操作定义

* 不可分割性：操作要么完全执行，要么完全不执行

* 线程安全：多线程环境下保证操作完整性

* 无锁机制：避免传统锁带来的性能开销

#### 核心特性

| 特性         | 说明                   |
| ------------ | ---------------------- |
| 线程安全     | 无需额外同步机制       |
| 硬件级原子性 | 使用 CPU 原子指令实现  |
| 内存屏障     | 确保操作前后内存一致性 |
| 高性能       | 比锁机制快 10-50 倍    |

### 底层实现原理

#### x86/x64 架构实现

```Assembly
; x86 实现
lock xadd [location], eax

; x64 实现
lock xadd [location], rax
```

* `lock` 前缀：锁定内存总线，确保操作原子性

* `xadd` 指令：交换并相加寄存器与内存值

### 基本用法

```csharp
using System;
using System.Threading;

class Program
{
    private static int _counter = 0;

    static void Main()
    {
        var threads = new Thread[10];

        for (int i = 0; i < 10; i++)
        {
            threads[i] = new Thread(IncrementCounter);
            threads[i].Start();
        }

        foreach (var t in threads)
            t.Join();

        Console.WriteLine($"最终计数值: {_counter}");
    }

    static void IncrementCounter()
    {
        for (int i = 0; i < 1000; i++)
        {
            Interlocked.Increment(ref _counter);
        }
    }
}
```

输出：

```
最终计数值: 10000
```

即使多个线程并发操作同一个变量，也不会出现竞争问题。

### 为什么不能直接用 _counter++

`_counter++` 实际上是三步操作：

```csharp
int temp = _counter;
temp = temp + 1;
_counter = temp;
```

在多线程环境中，这三步可能被打断，比如：

| 线程A               | 线程B               |
| ------------------- | ------------------- |
| 读取 `_counter = 0` | 读取 `_counter = 0` |
| +1 → `1`            | +1 → `1`            |
| 写回 `_counter = 1` | 写回 `_counter = 1` |

最终 `_counter = 1`，而不是 2。
这就是竞争条件（`race condition`）。

`Interlocked.Increment` 内部使用 `CPU` 的原子指令（如 `x86` 的 `LOCK INC`），
确保操作不可中断，因此绝对线程安全。

### 返回值

`Interlocked.Increment` 会返回自增后的值：

```csharp
int count = 0;
int newValue = Interlocked.Increment(ref count);

Console.WriteLine(newValue); // 1
Console.WriteLine(count);    // 1
```

所以它可以直接用于生成唯一 `ID`、计数等逻辑。

### 支持的类型

| 方法                          | 支持的类型                                 |
| ----------------------------- | ------------------------------------------ |
| `Interlocked.Increment`       | `int`, `long`                              |
| `Interlocked.Decrement`       | `int`, `long`                              |
| `Interlocked.Add`             | `int`, `long`                              |
| `Interlocked.Exchange`        | `int`, `long`, `float`, `double`, `object` |
| `Interlocked.CompareExchange` | 同上                                       |

### 性能分析

相比于使用 `lock` 的同步方式：

```csharp
lock(_lockObj)
{
    _counter++;
}
```

`Interlocked.Increment`：

✅ 无锁操作（`Lock-free`）
✅ 内核级原子性（基于 `CPU` 指令）
✅ 极高性能（纳秒级）

实测在多线程计数场景中性能提升 5~10 倍以上。
非常适合高频次计数（如统计请求量、日志写入次数、对象实例数）。

### 完整的原子操作方法集

```csharp
using System;
using System.Threading;

class InterlockedCompleteExample
{
    private static int _counter = 0;
    private static long _bigCounter = 0;
    private static int _value = 10;
    private static object _syncObject = new object();

    static void DemonstrateAllMethods()
    {
        // 1. Increment - 递增
        int result1 = Interlocked.Increment(ref _counter);
        Console.WriteLine($"After Increment: {_counter}, Returned: {result1}");

        // 2. Decrement - 递减
        int result2 = Interlocked.Decrement(ref _counter);
        Console.WriteLine($"After Decrement: {_counter}, Returned: {result2}");

        // 3. Add - 加法
        int result3 = Interlocked.Add(ref _counter, 5);
        Console.WriteLine($"After Add(5): {_counter}, Returned: {result3}");

        // 4. Exchange - 交换
        int original = Interlocked.Exchange(ref _value, 20);
        Console.WriteLine($"After Exchange: {_value}, Original: {original}");

        // 5. CompareExchange - 比较并交换
        int comparand = 20;
        int newValue = 30;
        int result5 = Interlocked.CompareExchange(ref _value, newValue, comparand);
        Console.WriteLine($"After CompareExchange: {_value}, Returned: {result5}");

        // 6. Read - 读取 long 类型（确保在32位系统上原子读取）
        long readResult = Interlocked.Read(ref _bigCounter);
        Console.WriteLine($"Read long value: {readResult}");

        // 7. And - 位与操作（.NET 5+）
        // Interlocked.And(ref _value, 0x0F);

        // 8. Or - 位或操作（.NET 5+）
        // Interlocked.Or(ref _value, 0xF0);
    }
}
```

### 典型应用场景

#### 多线程计数器

```csharp
private static int _activeConnections;

public static void OnClientConnect()
{
    var current = Interlocked.Increment(ref _activeConnections);
    Console.WriteLine($"连接数增加到 {current}");
}

public static void OnClientDisconnect()
{
    var current = Interlocked.Decrement(ref _activeConnections);
    Console.WriteLine($"连接数减少到 {current}");
}
```

#### 分配唯一自增 ID

```csharp
private static int _nextId = 0;

public static int GetNextId()
{
    return Interlocked.Increment(ref _nextId);
}
```

#### 控制并发访问次数

```csharp
private static int _running = 0;

public async Task ProcessAsync()
{
    if (Interlocked.Increment(ref _running) > 5)
    {
        Console.WriteLine("超过并发限制，拒绝执行");
        Interlocked.Decrement(ref _running);
        return;
    }

    try
    {
        await DoWorkAsync();
    }
    finally
    {
        Interlocked.Decrement(ref _running);
    }
}
```

### 高级用法

#### 无锁栈实现

```csharp
public class LockFreeStack<T>
{
    private class Node
    {
        public T Value { get; }
        public Node Next { get; set; }

        public Node(T value)
        {
            Value = value;
        }
    }

    private Node _head;

    public void Push(T value)
    {
        var newNode = new Node(value);
        Node oldHead;
        do
        {
            oldHead = _head;
            newNode.Next = oldHead;
        }
        while (Interlocked.CompareExchange(ref _head, newNode, oldHead) != oldHead);
    }

    public bool TryPop(out T value)
    {
        Node oldHead;
        do
        {
            oldHead = _head;
            if (oldHead == null)
            {
                value = default;
                return false;
            }
        }
        while (Interlocked.CompareExchange(ref _head, oldHead.Next, oldHead) != oldHead);

        value = oldHead.Value;
        return true;
    }

    public bool IsEmpty => _head == null;
}
```

#### 线程安全的对象池

```csharp
public class ObjectPool<T> where T : class, new()
{
    private class Node
    {
        public T Value { get; }
        public Node Next { get; set; }

        public Node(T value)
        {
            Value = value;
        }
    }

    private Node _head;
    private int _count = 0;
    private readonly int _maxSize;

    public ObjectPool(int maxSize = 100)
    {
        _maxSize = maxSize;
    }

    public void Return(T item)
    {
        if (Interlocked.Read(ref _count) >= _maxSize)
        {
            return; // 池已满，丢弃对象
        }

        var newNode = new Node(item);
        Node oldHead;
        do
        {
            oldHead = _head;
            newNode.Next = oldHead;
        }
        while (Interlocked.CompareExchange(ref _head, newNode, oldHead) != oldHead);

        Interlocked.Increment(ref _count);
    }

    public T Get()
    {
        Node oldHead;
        do
        {
            oldHead = _head;
            if (oldHead == null)
            {
                Interlocked.Increment(ref _count);
                return new T();
            }
        }
        while (Interlocked.CompareExchange(ref _head, oldHead.Next, oldHead) != oldHead);

        Interlocked.Decrement(ref _count);
        return oldHead.Value;
    }

    public int Count => (int)Interlocked.Read(ref _count);
}
```


### 与 lock、Monitor 的区别

| 特性         | Interlocked        | lock / Monitor   |
| ------------ | ------------------ | ---------------- |
| 是否锁住线程 | 否                 | 是               |
| 原子性       | ✅                  | ✅                |
| 是否可中断   | 不可               | 可被其他线程等待 |
| 性能         | 极高               | 一般             |
| 适用场景     | 简单数值、指针操作 | 多步复杂逻辑     |

* 如果只是“计数/标志位更新”，用 `Interlocked`；

* 如果涉及多个变量或逻辑组合，用 `lock`。

### CompareExchange (CAS) 补充理解

`Interlocked.CompareExchange` 是构建更复杂无锁结构的核心：

```csharp
int location = 0;
int newValue = 1;
int expected = 0;

int old = Interlocked.CompareExchange(ref location, newValue, expected);
```

* 如果 `location == expected` → 把 `location` 改成 `newValue`；

* 否则什么都不做；

* 返回修改前的旧值。

这就是经典的 `Compare-And-Swap` (CAS)，
是无锁队列、无锁堆栈、无锁缓存等的基础原语。

### 与 async/await 的结合注意事项

`Interlocked` 是同步原子操作，适用于多线程并发，不涉及异步上下文。
即使在 `async` 方法中使用，也完全没问题：

```csharp
private static int _count;

public async Task LogAsync()
{
    Interlocked.Increment(ref _count);
    await File.AppendAllTextAsync("log.txt", $"{_count}\n");
}
```

* 它不会保证异步顺序；

* 它保证计数线程安全。

### 总结

| 特性     | Interlocked.Increment                |
| -------- | ------------------------------------ |
| 命名空间 | `System.Threading`                   |
| 返回值   | 自增后的值                           |
| 线程安全 | ✅ 原子操作                           |
| 锁机制   | 无锁                                 |
| 支持类型 | `int`, `long`                        |
| 性能     | 极高                                 |
| 应用场景 | 计数、ID生成、并发控制、原子状态更新 |
| 替代方案 | lock、Monitor、Volatile、CAS         |
