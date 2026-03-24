### 简介

在 `.NET` 里提到线程同步，很多人的第一反应还是：

```csharp
private readonly object _sync = new();

lock (_sync)
{
    // 临界区
}
```

这套写法本身没有问题，而且已经用了很多年。

但从 `.NET 9` 和 `C# 13` 开始，官方开始明确给出一个更推荐的方向：

```csharp
System.Threading.Lock
```

也就是说，互斥锁这件事，不再只是“拿个 `object` 凑合锁一下”，而是开始有了一个专用类型。

这篇文章重点讲清楚几件事：

* `System.Threading.Lock` 到底是什么；
* 它和传统 `lock(object)` / `Monitor` 是什么关系；
* 编译器为什么会对它做专门处理；
* 它到底解决了什么问题，又没有解决什么问题；
* 实战里什么时候该用它，什么时候仍然该看 `SemaphoreSlim`、`Interlocked` 或其他同步原语。

一句话先说结论：

> 在支持 `.NET 9 + C# 13` 的同步代码里，`System.Threading.Lock` 可以理解成“官方更推荐的新一代默认互斥锁写法”。

### `System.Threading.Lock` 到底是什么？

它位于：

```csharp
System.Threading
```

本质上，它是一个专门用来做线程内进程级互斥的锁类型。

最值得先记住的不是 API，而是它的定位：

* 它解决的仍然是“临界区互斥”问题；
* 它不是分布式锁；
* 它不是异步锁；
* 它也不是为了替代所有同步原语。

更准确地说，它是在替代这样一种历史习惯：

```csharp
用一个普通 object 充当锁对象
```

所以它的核心价值不是“发明了新的并发模型”，而是：

* 用专用类型表达“这就是锁”；
* 让编译器和运行时更容易识别并优化；
* 降低把普通对象误当成锁对象的概率。

### 为什么会有它？

传统写法的问题，不在于不能用，而在于太宽泛。

例如下面这些写法，大家都见过：

```csharp
lock (this) { }
lock (typeof(MyType)) { }
lock ("global-lock") { }
```

这些都可能工作，但都很容易埋坑。

根源就在于：

> 传统 `lock` 语句锁的是“某个对象”，而不是“某个明确的锁类型”。

这会带来几个现实问题：

* 锁对象语义不够清晰；
* 容易把可见对象暴露给外部代码一起锁；
* 编译器很难确认“这一定是拿来同步的对象”；
* 一些优化空间天然受限。

`System.Threading.Lock` 的出现，本质上就是把“锁”从一种约定俗成的用法，提升成一个明确的一等公民。

### 最常见的用法长什么样？

最推荐的写法其实很简单：

```csharp
using System.Threading;

public sealed class Account
{
    private readonly Lock _balanceLock = new();
    private decimal _balance;

    public void Deposit(decimal amount)
    {
        lock (_balanceLock)
        {
            _balance += amount;
        }
    }

    public decimal GetBalance()
    {
        lock (_balanceLock)
        {
            return _balance;
        }
    }
}
```

你会发现，表面语法和以前几乎一样：

* 还是 `lock (...)`
* 还是尽量缩小临界区
* 还是保护共享状态

变化在于：

* 锁字段不再是 `object`
* 而是专用的 `Lock`

这就是它最现实的迁移价值：

> 很多场景下，你只需要把锁字段从 `object` 换成 `Lock`，代码表达力就已经更强了。

### 它和传统 `lock(object)` / `Monitor` 到底是什么关系？

先说最重要的一句：

> `System.Threading.Lock` 没有改变“互斥”的本质，但改变了“锁对象的类型”和“编译器如何生成代码”。

传统写法：

```csharp
private readonly object _sync = new();

lock (_sync)
{
    // 临界区
}
```

背后大家熟悉的是：

```csharp
Monitor.Enter(...)
try
{
    // 临界区
}
finally
{
    Monitor.Exit(...)
}
```

而当 `lock` 语句的目标表达式精确是 `System.Threading.Lock` 时，编译器会走专门路径，而不是普通 `Monitor` 路径。

这也是它和传统锁最核心的差异。

### 编译器到底做了什么？

这是理解 `System.Threading.Lock` 的关键。

如果你写：

```csharp
private readonly Lock _gate = new();

public void Update()
{
    lock (_gate)
    {
        // 临界区
    }
}
```

在 `C# 13` 下，这不再只是“老的 `lock` 语法配新对象”。

编译器会识别出：

* 目标表达式就是 `System.Threading.Lock`
* 可以走它的专用进入/退出模式

等价理解可以近似成：

```csharp
using (_gate.EnterScope())
{
    // 临界区
}
```

这里的重点有两个：

* `EnterScope()` 返回的是一个作用域对象；
* 这个作用域结束时会自动释放锁。

所以从使用者角度看，`lock (_gate)` 仍然是最自然的写法；但从编译器角度看，它已经不是传统 `Monitor` 的那套展开方式了。

### `EnterScope()` 是什么？

如果你不想依赖 `lock` 语句，也可以直接这样写：

```csharp
private readonly Lock _gate = new();

public void Update()
{
    using (_gate.EnterScope())
    {
        // 临界区
    }
}
```

这段代码的意义很直接：

* 进入 `using` 时拿锁；
* 离开作用域时自动释放。

它的风格其实很像：

* `RAII`
* 作用域守卫
* “进作用域即持有，出作用域即释放”

如果你本来就喜欢更显式的资源作用域写法，这种形式会很顺手。

### 从源码心智模型看，`EnterScope()` 到底在表达什么？

如果从源码和编译器视角理解，`EnterScope()` 最重要的不是“多了一个 API”，而是它把锁获取和释放变成了一个很清晰的作用域模型。

你可以把它近似理解成：

* 进入作用域时获取锁；
* 持有一个只在当前作用域内有效的句柄；
* 离开作用域时自动释放锁。

也就是说，编译器针对 `lock (_gate)` 的专门处理，本质上是在把“加锁”这件事从：

* 传统 `Monitor.Enter/Exit` 式样的通用展开

转成：

* 一个更明确的“作用域持有锁”模型

这也是为什么 `System.Threading.Lock` 会让人联想到：

* `RAII`
* scope guard
* `using` 作用域资源管理

它不是让互斥的本质变了，而是让“锁的生命周期”表达得更干净。

### 为什么官方还会配一个 `IDE0330`？

这也是很有代表性的信号。

`.NET` 团队不只是加了一个新类型，还专门配了分析器规则：

```text
IDE0330 Prefer System.Threading.Lock
```

这说明官方想推动的并不只是“知道有这个 API”，而是：

* 在新平台上，把它当成默认推荐写法；
* 在代码审查和静态分析阶段就把旧习惯往新方向拉；
* 让“专锁专用”这件事逐步变成团队共识。

更务实地说，`IDE0330` 的意义有两层：

* 技术层面：提醒你项目已经具备迁移条件；
* 工程层面：帮助团队统一同步代码风格。

所以如果你的项目已经是 `.NET 9 + C# 13`，并且大量使用：

```csharp
private readonly object _sync = new();
```

那这条规则其实很值得纳入代码质量体系。

### 它和 `Monitor` 能不能混着当同一把锁用？

不能把它们当成同一套机制来依赖。

这是一个非常容易忽略，但很重要的点。

官方文档明确强调：

> 只有当 `lock` 语句目标表达式的类型精确是 `System.Threading.Lock` 时，编译器才会走专用实现；如果表达式类型变成了 `object`、泛型 `T` 等其他类型，可能就会走另一套实现，例如 `Monitor`。

这意味着什么？

这意味着下面这类写法是有风险的：

```csharp
Lock gate = new();
object boxed = gate;

lock (boxed)
{
    // 这里不再是你以为的那条专用 Lock 路径
}
```

所以它的使用原则很明确：

* 不要把 `Lock` 当成普通 `object` 四处传；
* 不要靠“反正都是 lock”这种思路混用；
* 要让表达式类型保持为精确的 `System.Threading.Lock`。

### `System.Threading.Lock` 可重入吗？

可以。

根据官方 API 文档：

* 同一个线程可以多次进入同一个 `Lock`；
* 递归进入是允许的；
* 退出次数必须和进入次数匹配。

也就是说，从语义上看，它和 `Monitor` 一样，属于线程持有型互斥锁，而不是像某些低级自旋锁那样天然不支持重入。

但要注意两个现实判断：

* “可重入”不等于“鼓励你写复杂锁递归”；
* 锁层级一复杂，死锁风险依然存在。

所以它支持重入，但这不是你放松锁设计纪律的理由。

### 从源码和运行时视角看，它在优化什么？

如果只站在业务代码层面看，`System.Threading.Lock` 好像只是把：

```csharp
object _sync
```

换成了：

```csharp
Lock _sync
```

但从运行时设计角度看，真正的价值在于“专用化”。

可以粗略理解成：

* 以前：任意引用类型都可能被拿来锁
* 现在：有一个专门表达互斥意图的锁类型

这种专用化会带来两个直接好处：

* 编译器更容易识别并生成更有针对性的代码；
* 运行时不必总背着“任何对象都可能被拿来做锁”的历史包袱。

所以它不是“换个类名”，而是把同步语义收得更明确。

这也是为什么官方在 `C# lock` 文档里已经明确建议：

* 在 `.NET 9 + C# 13` 上，优先锁一个专用的 `System.Threading.Lock` 实例。

### 它比 `lock(object)` 快多少？

这个问题最好别答得太绝对。

更稳妥的说法是：

* 官方推荐它作为更优性能路径；
* 具体收益和运行时版本、争用程度、临界区长度、CPU 架构都有关系；
* 不要把某个固定百分比当成通用结论。

工程上更应该这样理解：

> `System.Threading.Lock` 的价值，除了可能更快，更重要的是“语义更清晰、误用更少、编译器可识别”。

如果你的场景是高频短临界区，它的收益通常会更明显。

如果你的临界区里本来就做了慢操作，那换成它也不会 magically 解决核心问题。

### 从面试和选型视角看，它和其他同步原语怎么区分？

这是最常见的追问之一。

如果只背定义，很容易把这些原语混成一团；但如果按“解决什么等待模型”去分，边界就很清楚。

#### `System.Threading.Lock` vs `Monitor`

两者都解决同步互斥。

区别在于：

* `Monitor` 是通用对象锁机制；
* `System.Threading.Lock` 是专用锁类型；
* 当前者更像历史通用路径，后者更像新平台上的推荐默认路径。

#### `System.Threading.Lock` vs `SemaphoreSlim`

这两个不是一类问题。

* `Lock` 解决的是同步互斥，一次只允许一个线程进入临界区；
* `SemaphoreSlim` 解决的是计数型并发控制，并且支持 `WaitAsync`。

所以只要场景里出现：

* `await`
* 异步限流
* 多许可并发

通常就应该优先想到 `SemaphoreSlim`，而不是 `Lock`。

#### `System.Threading.Lock` vs `ReaderWriterLockSlim`

这个对比的关键是读写比例。

* `Lock` 更适合普通互斥；
* `ReaderWriterLockSlim` 更适合读多写少、并且读写分离价值明显的场景。

如果你的共享资源本质上就是“大家都在改”，那上读写锁往往只会把模型搞复杂。

#### `System.Threading.Lock` vs `Interlocked`

这是另一个边界非常明确的对比。

* `Interlocked` 适合单个原子操作；
* `Lock` 适合多步临界区互斥。

所以如果只是：

* 计数器加减
* 引用交换
* CAS 比较交换

第一选择通常都不该是 `Lock`。

一句话压缩这组对比：

> `System.Threading.Lock` 是新一代默认同步互斥工具，但它仍然只回答“同步临界区互斥”这一类问题。

### `lock` 语句是不是从此都该换成 `Lock`？

如果你满足这些前提，答案通常是“值得优先考虑”：

* 项目已经是 `.NET 9+`
* 语言版本支持 `C# 13+`
* 代码是同步互斥，不是异步互斥
* 你本来就在用专用私有锁对象

这时候从：

```csharp
private readonly object _gate = new();
```

改成：

```csharp
private readonly Lock _gate = new();
```

往往是一个很自然的升级。

但下面这些情况，就别机械迁移：

* 目标框架还没到 `.NET 9`
* 团队还没有统一语言版本
* 同步模型本身就设计得很乱
* 问题根本不在锁类型，而在锁粒度或临界区内容

也就是说，`System.Threading.Lock` 是更好的默认选项，但不是一键治百病。

### 在 `async/await` 代码里能不能用？

不要把它当成异步锁来用。

原因和传统 `lock` 一样：

* 锁是由线程持有的；
* `await` 之后代码可能在另一个线程继续执行；
* 所以不能在持锁区间跨 `await`。

不管你是这样写：

```csharp
lock (_gate)
{
    await Task.Delay(1);
}
```

还是这样写：

```csharp
using (_gate.EnterScope())
{
    await Task.Delay(1);
}
```

都不属于正确用法。

如果你需要的是异步互斥，通常应该看的是：

* `SemaphoreSlim`
* `AsyncLock`

而不是 `System.Threading.Lock`。

### 它适合哪些场景？

下面这些场景很适合优先考虑它：

* 保护一个对象内部的少量共享状态；
* 典型同步临界区；
* 原本就会写 `private readonly object _sync = new();`
* 想明确表达“这个字段就是锁”；
* 想在支持新平台的前提下用更推荐的默认写法。

最典型的例子包括：

* 余额更新
* 本地缓存状态切换
* 集合读写保护
* 组件内部的小粒度同步

### 它不适合哪些场景？

先把边界说透：

* 它不是异步锁
* 它不是读写锁
* 它不是跨进程锁
* 它不是分布式锁
* 它不是高竞争长耗时任务的万能解药

如果你的需求是：

* 大量读、少量写：更该看 `ReaderWriterLockSlim`
* 异步限流或异步互斥：更该看 `SemaphoreSlim`
* 单个数值的原子增减：更该看 `Interlocked`
* 分布式节点协调：应该看数据库锁、Redis 锁或其他分布式协调方案

同步原语的选择，核心从来不是“哪个新就用哪个”，而是“它到底解决哪一类等待模型”。

### 一个很容易踩的坑：把它暴露成别的类型

如果你已经决定用 `System.Threading.Lock`，就尽量保持它的专用身份。

例如这类写法不推荐：

```csharp
private readonly Lock _gate = new();

public object SyncRoot => _gate;
```

或者：

```csharp
void M<T>(T gate) where T : class
{
    lock (gate)
    {
    }
}
```

原因不是“代码一定编不过”，而是：

* 你会丢失它的专用语义；
* 编译器未必还能按 `Lock` 的专门路径处理；
* 后续维护者也更容易搞混这把锁到底是什么。

### 一个非常务实的选择顺序

如果你在做并发控制，可以先按这个顺序判断：

1. 单个共享变量能不能用 `Interlocked` 解决？
2. 如果不能，是否只是普通同步互斥？
3. 如果是，并且项目在 `.NET 9 + C# 13`，优先考虑 `System.Threading.Lock`
4. 如果是异步等待，换成 `SemaphoreSlim` / `AsyncLock`
5. 如果是读多写少，再看 `ReaderWriterLockSlim`

这个顺序的意义在于：

* `Lock` 是新的默认同步互斥选项；
* 但它仍然只是“同步互斥”这一类问题的答案。

### 面试里怎么答比较到位？

如果面试官问：

“`System.Threading.Lock` 和传统 `lock(object)` 有什么区别？”

一个比较稳的回答可以是：

> `System.Threading.Lock` 是 .NET 9 引入的专用互斥锁类型。它解决的还是线程间临界区互斥问题，但不再依赖普通 object 充当锁对象。当 `lock` 语句的目标表达式精确是 `System.Threading.Lock` 时，C# 13 编译器会走专门路径，等价理解可以近似看成用 `EnterScope()` 返回的作用域对象自动释放锁，而不是老的通用 `Monitor` 展开方式。它的价值不仅在性能，也在于语义更清晰、误用更少。不过它依然是同步锁，不能跨 `await`，也不适合替代 `SemaphoreSlim` 这类异步同步原语。`

如果继续追问“它支不支持重入”，可以补一句：

> 支持。同一个线程可以重复进入同一个 `Lock`，但进入和退出次数要匹配。

如果继续追问“编译器为什么能优化它”，可以补一句：

> 因为 `lock` 语句在看到目标表达式精确是 `System.Threading.Lock` 时，可以不再按通用对象锁去展开，而是走这个专用类型的作用域进入/退出路径。这个前提非常关键，所以不要把它先转成 `object`、接口或泛型再去锁。

如果追问“`IDE0330` 是干什么的”，可以这样答：

> 它是 Roslyn 的一个建议规则，核心是在支持 `.NET 9 + C# 13` 的条件下，提示你把传统专用 `object` 锁迁移成 `System.Threading.Lock`，从而统一到新的推荐写法。

如果追问“什么时候值得迁移”，就答：

* 项目已在 `.NET 9 + C# 13`
* 原本就是同步互斥
* 本来就在用专用私有锁对象
* 希望统一成更清晰的新默认写法

### 总结

`System.Threading.Lock` 最值得记住的，不是它“多新”，而是它把一件老事情做得更明确了：

> 互斥锁就应该是一个专门的锁类型，而不是任意 `object` 的兼职工作。

如果你只想记住几句话，可以记这几条：

* 它是 `.NET 9 + C# 13` 下更推荐的同步互斥写法；
* 当 `lock` 目标精确是 `System.Threading.Lock` 时，编译器会走专用路径；
* 它不是异步锁，不能跨 `await`；
* 它支持重入，但仍然要严格控制锁顺序和临界区长度；
* 如果只是普通同步互斥，并且平台支持，优先用它通常比 `object` 锁更清晰。

---

参考资料：

* Microsoft Learn: `System.Threading.Lock` API
* Microsoft Learn: C# `lock` statement
* Microsoft Learn: `IDE0330 Prefer System.Threading.Lock`
