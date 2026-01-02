### 什么是 Interlocked.CompareExchange？

`Interlocked.CompareExchange` 是 `.NET` 中 `System.Threading.Interlocked` 类的最核心原子操作方法。它执行比较并交换（`Compare-And-Swap`，简称 `CAS`） 操作：在多线程环境下，安全地将变量的值与预期值比较，如果相等则替换为新值，整个过程原子不可中断。

关键特性

* 原子性：整个操作在CPU级别是原子的，不会被线程调度打断

* 无锁操作：无需使用锁即可实现线程安全

* 内存屏障：隐含完整内存屏障（`full fence`），确保操作前后内存访问顺序

返回值重要性：返回值是判断操作是否成功的关键

* 核心签名（常见重载）：

```csharp
public static int CompareExchange(ref int location, int value, int comparand);
public static long CompareExchange(ref long location, long value, long comparand);
public static T CompareExchange<T>(ref T location, T value, T comparand) where T : class;
public static float CompareExchange(ref float location, float value, float comparand);
public static double CompareExchange(ref double location, double value, double comparand);
```

* 参数解释：

    * `location`：要操作的共享变量（必须用 `ref` 传递）。

    * `value`：如果比较成功，要写入的新值。

    * `comparand`：预期旧值（用于比较）。

    * 返回值：操作前 `location` 的实际值（无论成功与否都返回旧值）。

`CAS` 是现代无锁（`lock-free`）并发编程的基础，许多高级结构（如 `ConcurrentDictionary`、`SpinLock`）内部都依赖它。

### 为什么使用 Interlocked.CompareExchange？

在多线程中，直接读取-修改-写入（如 `if (count == 0) count = 1;`）会产生竞争条件：多个线程可能同时读取相同值，导致覆盖更新。

* 传统锁的问题：`lock` 开销大（互斥锁、上下文切换），不适合高频轻量操作。

* `CAS` 的优势：

    * 无锁（`Lock-Free`）：不阻塞线程，高并发下性能极高。

    * 乐观并发：假设冲突少，先尝试操作，失败则重试（自旋）。

    * 原子性：硬件级保证（`x86 的 cmpxchg` 指令）。

* 典型应用场景：

    * 实现线程安全的单例模式（双检查锁）。

    * 自定义无锁队列/栈。

    * 原子更新复杂状态（标志位 + 计数器）。

    * 实现自定义同步原语（如 `SpinLock`、`SemaphoreSlim` 内部）。

传统的 “比较 + 赋值” 是两步非原子操作，多线程下会因竞态条件导致逻辑错误。

```csharp
// 非原子操作：多线程下可能同时通过if判断，导致赋值错误
private static int _value = 0;
public static void UnsafeUpdate(int oldVal, int newVal)
{
    if (_value == oldVal) // 步骤1：读取并比较
    {
        _value = newVal;  // 步骤2：赋值
    }
}
```

两个线程可能同时通过if判断（都读到 `_value=oldVal` ），最终都执行赋值，导致逻辑错误。

`Interlocked.CompareExchange` 将 “比较” 和 “赋值” 合并为不可中断的原子操作，从底层杜绝竞态条件，且无需阻塞线程（无锁）。

与 `lock` 的根本区别

| 维度           | lock       | CompareExchange |
| -------------- | ---------- | --------------- |
| 是否阻塞       | 是         | 否              |
| 是否上下文切换 | 是         | 否              |
| 粒度           | 任意代码块 | 单变量          |
| 性能           | 较低       | 极高            |
| 适合           | 复杂逻辑   | 状态切换        |

### 什么时候该用 CompareExchange？

| 场景         | 推荐 |
| ------------ | ---- |
| 简单状态切换 | ✔    |
| 计数、标志位 | ✔    |
| 高并发热点   | ✔    |
| 复杂业务流程 | ❌    |
| 多变量一致性 | ❌    |

### 如何使用 Interlocked.CompareExchange？

#### 最简单的 CAS 操作（判断是否交换成功）

```csharp
private static int _counter = 0;

public static void TestCas()
{
    // 目标：如果_counter是0，就改为100
    int expected = 0;    // 预期值
    int newValue = 100;  // 新值
    
    // 执行CAS操作
    int original = Interlocked.CompareExchange(ref _counter, newValue, expected);
    
    // 判断是否交换成功（原始值 == 预期值 → 成功）
    if (original == expected)
    {
        Console.WriteLine($"交换成功！原始值：{original}，新值：{_counter}");
    }
    else
    {
        Console.WriteLine($"交换失败！原始值：{original}，当前值：{_counter}");
    }
}
```

输出：交换成功！原始值：0，新值：100（若再次执行，会输出失败，因为`_counter` 已不是 0）。

#### 原子条件更新

```csharp
private int _status = 0;  // 0: 未初始化, 1: 初始化中, 2: 已完成

public void Initialize()
{
    if (Interlocked.CompareExchange(ref _status, 1, 0) == 0)
    {
        // 只有第一个线程进入这里
        try
        {
            // 执行初始化逻辑
            DoInitialize();
        }
        finally
        {
            // 标记完成（原子操作，无需 CAS）
            Interlocked.Exchange(ref _status, 2);
        }
    }
    else
    {
        // 其他线程等待完成
        while (_status != 2) Thread.SpinWait(10);
    }
}
```

* 解释：只有当 `_status` 为 0 时才设置为 1 并执行初始化。

#### 实现自旋锁

```csharp
public class SpinLock
{
    private int _lock = 0; // 0=未锁定, 1=已锁定
    
    public void Enter()
    {
        while (Interlocked.CompareExchange(ref _lock, 1, 0) != 0)
        {
            // 等待 - 可以使用Thread.SpinWait优化
            Thread.SpinWait(100);
        }
    }
    
    public void Exit()
    {
        Interlocked.Exchange(ref _lock, 0);
    }
}
```

#### 经典场景：无锁计数器（循环 CAS）

单次 `CAS` 可能因其他线程修改变量失败，需循环重试（自旋 `CAS`），实现线程安全的计数器：

```csharp
private static int _casCounter = 0;

/// <summary>
/// 无锁原子递增（替代Interlocked.Increment）
/// </summary>
public static int IncrementCounter()
{
    int current;   // 当前值
    int newValue;  // 新值
    do
    {
        // 1. 原子读取当前值（Interlocked保证可见性）
        current = _casCounter;
        // 2. 计算新值（仅本地计算，无竞态）
        newValue = current + 1;
        // 3. CAS操作：若当前值未被修改，就更新为新值
        // 返回值≠current → 被其他线程修改，重试
    } while (Interlocked.CompareExchange(ref _casCounter, newValue, current) != current);
    
    return newValue; // 返回递增后的值
}

// 测试：1000个线程各调用1次，最终结果必为1000
var tasks = Enumerable.Range(0, 1000)
    .Select(_ => Task.Run(IncrementCounter))
    .ToList();
Task.WaitAll(tasks.ToArray());
Console.WriteLine($"最终计数器值：{_casCounter}"); // 输出：1000
```

### 高级应用场景

#### 泛型版本：懒加载单例（双检查锁模式）

```csharp
public sealed class Singleton
{
    private static Singleton _instance = null;

    private Singleton() { }

    public static Singleton Instance
    {
        get
        {
            if (_instance == null)
            {
                var temp = new Singleton();
                Interlocked.CompareExchange(ref _instance, temp, null);
            }
            return _instance;
        }
    }
}
```

在 `.NET` 中需加 `Lazy<T>` 更安全：

```csharp
private static Lazy<Singleton> _instance = new Lazy<Singleton>(() => new Singleton());
public static Singleton Instance => _instance.Value;
```

#### 无锁状态机（原子状态转换）

控制对象状态的原子转换（如从 `Idle→Running` ，避免状态混乱）：

```csharp
// 定义状态枚举
public enum TaskState { Idle, Running, Completed, Failed }

public class TaskStateMachine
{
    private TaskState _state = TaskState.Idle;

    /// <summary>
    /// 原子转换：Idle → Running（仅当状态为Idle时成功）
    /// </summary>
    public bool TryStart()
    {
        TaskState original = Interlocked.CompareExchange(
            ref _state, 
            TaskState.Running,  // 新值
            TaskState.Idle      // 预期值
        );
        return original == TaskState.Idle; // 原始值=预期值 → 转换成功
    }

    /// <summary>
    /// 原子转换：Running → Completed
    /// </summary>
    public bool TryComplete()
    {
        TaskState original = Interlocked.CompareExchange(
            ref _state, 
            TaskState.Completed, 
            TaskState.Running
        );
        return original == TaskState.Running;
    }
}

// 使用
var stateMachine = new TaskStateMachine();
Console.WriteLine(stateMachine.TryStart());    // true（状态变为Running）
Console.WriteLine(stateMachine.TryStart());    // false（已不是Idle）
Console.WriteLine(stateMachine.TryComplete()); // true（状态变为Completed）
```

#### 自旋实现原子加法（模拟 Interlocked.Add）

```csharp
private int _counter = 0;

public int Increment()
{
    int oldValue, newValue;
    do
    {
        oldValue = _counter;
        newValue = oldValue + 1;
    } while (Interlocked.CompareExchange(ref _counter, newValue, oldValue) != oldValue);
    return newValue;
}
```

* 解释：循环直到 `CAS` 成功（自旋），实现无锁递增

#### 对象引用替换（线程安全赋值）

```csharp
private List<int> _cache = null;

public void UpdateCache(List<int> newCache)
{
    Interlocked.CompareExchange(ref _cache, newCache, _cache);  // 仅当未被其他线程修改时更新
}
```

### Interlocked.CompareExchange 的内部实现

* 硬件支持：

    * `x86/x64`：`LOCK CMPXCHG` 指令，总线锁保证原子性。

    * `ARM`：LDREX/STREX 指令对（`Load-Exclusive/Store-Exclusive`），失败重试。

同时，该操作会触发全内存屏障（`Full Memory Barrier`）：

* 保证操作前后的内存读写不会被 `CPU` 重排序；

* 确保所有线程能立即看到变量的最新值（避免 `CPU` 缓存导致的 “脏读”）。

* `.NET` 实现（`CoreCLR` 简化伪码）：

```csharp
public static int CompareExchange(ref int location, int value, int comparand)
{
    // 内联为平台特定原子指令
    fixed (int* ptr = &location)
    {
        return InterlockedCompareExchange(ptr, value, comparand);
    }
}
```

* 性能：

    * 无竞争：`O(1)`，极快。

    * 高竞争：自旋消耗 `CPU`，但仍远优于锁（无上下文切换）。

### 注意事项与最佳实践

* 自旋循环：总是用 `do-while` 包装 `CAS`，形成“乐观循环”。

* `ABA` 问题：

    * 线程 1 读取值为A，准备 `CAS` 为C；线程 2 将A改为B，又改回A；线程 1 的 `CAS` 会成功，但中间状态已变化，可能导致逻辑错误

    * 解决方法：版本号 + 引用 `CAS`。

* 内存可见性：`Interlocked` 操作自带全内存屏障（`full fence`），无需额外 `volatile`。

* 避免过度使用：简单计数用 `Interlocked.Increment`；集合用 `Concurrent` 类。

* 替代方案：

    * 高层：`ConcurrentDictionary、BlockingCollection`。

    * 现代：`System.Threading.Channels`（生产者-消费者）。

    * `.NET 8+`：考虑 `System.Threading.Lock`（`C# 12` 新特性，更简洁）。

### ABA 问题

#### 什么是 ABA 问题？

`ABA` 问题 是无锁（`lock-free`）编程中使用 `Compare-And-Swap` (`CAS`) 操作时的一种经典隐患。

场景描述：

* 线程 A 读取共享变量值：A

* 线程 B 先将值改为 B，然后又改回 A

* 线程 A 执行 `CAS`：期望值是 A，当前值也是 A → `CAS` 成功！

* 但实际上值已经被其他线程修改过，语义已经改变，线程 A 却误以为“一切未变”。

这会导致逻辑错误，尤其在无锁栈、队列等数据结构中，可能造成节点丢失、循环引用或内存泄漏。

#### 解决方案：版本号 + 引用 CAS

1. 定义「版本化状态对象」

```csharp
sealed class VersionedValue
{
    public readonly int Value;
    public readonly int Version;

    public VersionedValue(int value, int version)
    {
        Value = value;
        Version = version;
    }

    public override string ToString()
        => $"Value={Value}, Version={Version}";
}
```

关键点：

* 不可变（`readonly`）

* 每次修改 → 新对象

* `CAS` 比较的是 对象引用

2. 运行示例

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    static VersionedValue state = new VersionedValue(1, 0);

    static void Main()
    {
        var t1 = Task.Run(Thread1);
        var t2 = Task.Run(Thread2);

        Task.WaitAll(t1, t2);

        Console.WriteLine($"Final state: {state}");
    }

    static void Thread1()
    {
        var old = state; // 读 A(v0)
        Console.WriteLine($"T1 read: {old}");

        Thread.Sleep(200);

        var newState = new VersionedValue(3, old.Version + 1);

        var result = Interlocked.CompareExchange(
            ref state,
            newState,
            old
        );

        if (result == old)
        {
            Console.WriteLine("T1 CAS succeeded");
        }
        else
        {
            Console.WriteLine($"T1 CAS failed, current = {result}");
        }
    }

    static void Thread2()
    {
        Thread.Sleep(50);

        // A → B
        state = new VersionedValue(2, state.Version + 1);
        // B → A
        state = new VersionedValue(1, state.Version + 1);

        Console.WriteLine($"T2 changed state twice: {state}");
    }
}
```

3. 典型输出

```
T1 read: Value=1, Version=0
T2 changed state twice: Value=1, Version=2
T1 CAS failed, current = Value=1, Version=2
Final state: Value=1, Version=2
```

核心结果：

* 值还是 1

* 版本已经变了

* `CAS` 失败

* `ABA` 被成功识别

#### 为什么这个方案一定能防 ABA？

> CAS 比较的是“对象引用”，而不是值

即使：

```
Value: 1 → 2 → 1
```

但：

```
Reference: A → B → C
```

A ≠ C

`CAS` 不可能误判成功

#### 黄金组合

| 技术                        | 作用         |
| --------------------------- | ------------ |
| Interlocked.CompareExchange | 原子性       |
| 不可变对象                  | 消除中间态   |
| 版本号                      | 明确变化历史 |
| GC                          | 避免悬垂指针 |

> .NET 官方并发集合、Immutable 系列，都是这个思想

#### 什么时候该用这一套？

适合：

* 自定义状态机

* `CAS` + 状态切换

* 无锁缓存

* 高频并发更新

不适合：

* 普通计数

* 简单标志位

* 低并发业务


### 总结

> Interlocked.CompareExchange 是 .NET 无锁并发的“原子开关”
用它可以：
> * 不加锁
> * 不阻塞
> * 在竞争中安全修改状态