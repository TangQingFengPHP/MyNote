### 简介

`Volatile` 是 `C#` 中处理内存可见性和指令重排序的关键机制，它提供了对内存访问的精细控制。在并发编程中，`volatile` 关键字和 `Volatile` 类都是解决共享变量可见性问题的重要工具。

### 为什么需要volatile？

#### CPU 缓存导致的 “内存可见性” 问题

现代 `CPU` 为提升性能，会将频繁访问的变量缓存到核心专属的缓存（`L1/L2/L3`）中，而非每次都读写主内存。这会导致：

* 线程 A 修改了共享字段的值（仅写入自己的 `CPU` 缓存，未同步到主内存）；

* 线程 B 读取该字段时，从自己的 `CPU` 缓存读取（仍是旧值），无法看到线程 A 的修改。

#### 编译器 / CPU 的 “指令重排序” 优化

编译器（`C#` 编译器）和 `CPU` 为提升执行效率，会在不改变单线程逻辑的前提下，调整指令的执行顺序

```csharp
// 原始代码
bool _isReady = false;
int _data = 100;

// 编译器/CPU可能重排序为：先赋值_data，再赋值_isReady（单线程无影响）
// 但多线程下，线程B可能看到_isReady=true，但_data还是旧值
```

`volatile` 的核心作用就是：禁止缓存 + 禁止指令重排序，保证多线程对字段的访问 “所见即所得”。

* 插入内存屏障（`memory barrier`）：

    * `Acquire Fence`：读取 `volatile` 字段前，禁止将后续读取提前。

    * `Release Fence`：写入 `volatile` 字段后，禁止将之前写入推迟。

* 强制每次读写都直接访问主内存，绕过缓存优化。

### 核心定义与语法

#### 语法规则

`volatile` 只能修饰字段，且有严格的类型限制，语法如下：

```csharp
// 正确：修饰实例字段
private volatile bool _isRunning;

// 正确：修饰静态字段
private static volatile int _counter;

// 错误：不能修饰方法/参数/局部变量/属性/常量
public volatile void DoWork() { } // 编译错误
private int VolatileProperty { get; set; } // 编译错误（属性不能加volatile）
```

#### 支持的类型

`volatile` 仅支持以下类型（避免 `CPU` 操作的原子性问题）：

* 引用类型（如 `object`、`string`、自定义类）；

* 值类型：`byte、sbyte、short、ushort、int、uint、long、ulong、char、float、bool`；

* 上述类型的指针（如 `int*` ）。

> 注意：不支持double、decimal、struct（自定义值类型）、DateTime等，这些类型的读写不是原子的，volatile无法保证正确性。

#### 等效方法：Volatile.Read/Volatile.Write

除了关键字，`.NET` 还提供 `Volatile` 静态类的 `Read/Write` 方法，功能与 `volatile` 关键字一致，但更灵活（可动态控制读写）：

```csharp
// 等价于 volatile 修饰的 _isRunning = true
Volatile.Write(ref _isRunning, true);

// 等价于读取 volatile 修饰的 _isRunning
bool current = Volatile.Read(ref _isRunning);
```

### 核心原理：内存屏障（Memory Barrier）

`volatile` 的底层是通过插入内存屏障（`Memory Barrier`） 实现的：

* 读屏障（`Load Barrier`）：读取 `volatile` 字段时，插入读屏障，强制 `CPU` 从主内存读取值，而非缓存；同时禁止将读指令重排序到屏障之前。

* 写屏障（`Store Barrier`）：写入 `volatile` 字段时，插入写屏障，强制 `CPU` 将值写入主内存，而非缓存；同时禁止将写指令重排序到屏障之后。

### 基础使用示例

#### 关键字用法

```csharp
public class ThreadSafeFlag
{
    private volatile bool _isRunning = true;

    public void Run()
    {
        // 线程1：循环直到标志关闭
        while (_isRunning)
        {
            // 执行工作
            Thread.SpinWait(1000);
        }
        Console.WriteLine("线程停止");
    }

    public void Stop()
    {
        // 线程2：设置标志
        _isRunning = false;
        Console.WriteLine("停止信号已发送");
    }
}
```

使用示例：

```csharp
var flag = new ThreadSafeFlag();
var worker = new Thread(flag.Run);
worker.Start();

Thread.Sleep(100);
flag.Stop();  // 另一个线程能立即看到变化
worker.Join();
```

不加 `volatile`：可能导致 `_isRunning` 被缓存，线程永远不退出。

#### Volatile 类静态方法（.NET 4.5+ 推荐）

```csharp
using System.Threading;

private int _value;

public int ReadValue() => Volatile.Read(ref _value);
public void WriteValue(int newValue) => Volatile.Write(ref _value, newValue);
```

* `Volatile.Read`：带 `Acquire` 屏障的读取。

* `Volatile.Write`：带 `Release` 屏障的写入。

* 优势：更精确控制屏障方向，比关键字更灵活。

#### 双检查锁单例

```csharp
public sealed class Singleton
{
    private static volatile Singleton? _instance;

    public static Singleton Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (typeof(Singleton))
                {
                    if (_instance == null)
                        _instance = new Singleton();
                }
            }
            return _instance!;
        }
    }

    private Singleton() { }
}
```

### 优点与缺点

| 方面       | 优点                                   | 缺点                           |
| ---------- | -------------------------------------- | ------------------------------ |
| **性能**   | 极低开销（仅内存屏障），远高于锁       | 仍比普通变量慢（禁用部分优化） |
| **易用性** | 简单关键字或方法调用                   | 语义复杂，易误用               |
| **适用性** | 完美用于简单标志位、状态切换、双检查锁 | **不能**用于计数器、复合操作   |
| **安全性** | 提供必要内存模型保证                   | 不足以实现复杂同步             |

### 推荐场景

#### 推荐使用 `volatile` 的场景：

* 布尔标志（如停止信号 `_isRunning`）。

* 状态枚举（如 `Ready/Running/Stopped`）。

* 引用类型字段的双检查锁单例。

* 一写多读（`one writer, multiple readers`）模式。

#### 不推荐使用 `volatile` 的场景：

* 计数器、累加操作 → 用 `Interlocked`。

* 复杂状态 → 用 `lock` 或无锁结构。

* 64位值（long/double）在32位进程 → 用 `Interlocked`。

### Volatile vs Interlocked

| 对比项   | Volatile          | Interlocked |
| -------- | ----------------- | ----------- |
| 原子性   | ❌                 | ✅           |
| 内存屏障 | Acquire / Release | Full Fence  |
| 返回旧值 | ❌                 | ✅           |
| 适用场景 | 状态观察          | 状态修改    |
| 性能     | 更快              | 稍慢        |

### 总结

`volatile` 是 `.NET` 多线程编程中一个低级但关键的工具，适合简单的一写多读标志场景。但绝不能滥用，大多数线程安全需求应优先选择 `Interlocked、lock、Lazy<T>` 或并发集合。

```csharp
// 读：Volatile
var state = Volatile.Read(ref _state);

// 写：CAS / Exchange
if (state == A)
    Interlocked.CompareExchange(ref _state, B, A);
```

> Volatile 是并发程序的“观察者协议”，
> Interlocked 才是“修改者协议”。