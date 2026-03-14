### 简介

如果说 `Span<T>` 是 `.NET` 高性能内存体系里最亮眼的类型，那么 `Memory<T>` 就是它最重要的搭档。

很多人学完 `Span<T>` 后，马上会遇到几个现实问题：

* 为什么 `Span<T>` 不能做类字段？
* 为什么 `Span<T>` 不能跨 `await`？
* 为什么异步 `IO` 场景里，很多 API 更喜欢 `Memory<T>` / `ReadOnlyMemory<T>`？

答案基本都指向同一个类型：

```csharp
Memory<T>
```

一句话先说透：

> `Span<T>` 更适合同步、短生命周期、高性能内存操作；`Memory<T>` 更适合需要跨异步边界、跨组件传递、或需要长期持有的内存视图。

如果你把 `Span<T>` 理解成“只能活在当前同步作用域里的高性能视图”，那 `Memory<T>` 就可以理解成：

> 更适合活在堆上、可以跨 `await` 传递的内存视图。

### 为什么需要 `Memory<T>`？

先看一个典型问题：

```csharp
async Task ReadAsync(Stream stream)
{
    Span<byte> buffer = stackalloc byte[1024]; // 这里就不行
    await stream.ReadAsync(buffer);
}
```

这类代码之所以不成立，不是因为缓冲区写法不优雅，而是因为：

* `Span<T>` 是 `ref struct`
* 它不能安全地跨异步状态机边界
* `await` 可能让方法状态机和局部变量提升到堆上

这时就需要一种“能表示连续内存，又能安全地跨异步边界传递”的类型。

这就是 `Memory<T>` 诞生的核心动机。

### `Memory<T>` 到底是什么？

你可以先用一句最直白的话理解：

> `Memory<T>` 是一段连续内存的可持久视图。

和 `Span<T>` 一样，它通常也不拥有底层数据本身，而是“指向”一段已有的连续内存。

但和 `Span<T>` 不同的是，它没有 `ref struct` 的那层严格限制，因此：

* 它可以做字段；
* 可以放到集合里；
* 可以作为返回值长期传递；
* 可以跨 `await` 使用。

对应的只读版本是：

```csharp
ReadOnlyMemory<T>
```

所以你可以把这两个类型先记成：

| 类型 | 定位 |
| --- | --- |
| `Span<T>` | 同步短生命周期高性能视图 |
| `Memory<T>` | 可跨异步、可长期持有的内存视图 |

### `Memory<T>` 和 `Span<T>` 到底是什么关系？

这是最核心的一组关系。

可以先记这张表：

| 类型 | 是否 `ref struct` | 能否跨 `await` | 能否做字段 | 典型用途 |
| --- | --- | --- | --- | --- |
| `Span<T>` | 是 | 否 | 否 | 同步高性能处理 |
| `ReadOnlySpan<T>` | 是 | 否 | 否 | 同步只读切片 |
| `Memory<T>` | 否 | 是 | 是 | 异步或长期持有 |
| `ReadOnlyMemory<T>` | 否 | 是 | 是 | 异步只读视图 |

一个非常实用的理解方式是：

* `Memory<T>` 负责“持有和传递”
* `Span<T>` 负责“实际操作”

所以实际代码里经常是这样：

```csharp
Memory<byte> memory = new byte[1024];
Span<byte> span = memory.Span;
```

也就是说：

* 用 `Memory<T>` 表达这段内存可以跨作用域存活；
* 真要读写它时，再通过 `.Span` 拿到高性能操作视图。

### `Memory<T>` 的本质长什么样？

概念上，`Memory<T>` 也可以理解成：

* 某个底层内存来源；
* 起始偏移；
* 长度。

你可以把它想象成：

```text
(object/reference, index, length)
```

它本质上仍然是视图，不是所有权本体。

这点非常重要，因为它意味着：

* `Memory<T>` 自己不负责发明一块新内存；
* 它只是对原始内存的一层包装；
* 你仍然要关心底层内存的生命周期。

### 最常见的创建方式

#### 1. 从数组创建

这是最常见的用法。

```csharp
byte[] bytes = new byte[1024];

Memory<byte> memory1 = bytes;
Memory<byte> memory2 = bytes.AsMemory();
Memory<byte> memory3 = bytes.AsMemory(100, 200);
```

这里：

* `memory1` 指向整个数组；
* `memory3` 指向数组中 `[100..300)` 那一段；
* 没有发生数据复制。

#### 2. 从字符串创建只读内存视图

字符串是不可变的，所以这里通常是：

```csharp
ReadOnlyMemory<char> memory = "Hello".AsMemory();
ReadOnlyMemory<char> world = "Hello World".AsMemory(6, 5);
```

和 `ReadOnlySpan<char>` 一样，它适合只读场景，但比 `ReadOnlySpan<char>` 更适合跨异步边界或长期存放。

#### 3. 从 `MemoryPool<T>` 获取

这在高性能场景里非常常见。

```csharp
using System.Buffers;

using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(4096);
Memory<byte> buffer = owner.Memory.Slice(0, 4096);
```

这里的关键点不是语法，而是：

* `MemoryPool<T>` 提供可复用内存块；
* `IMemoryOwner<T>` 表示这块内存的所有者；
* `owner.Memory` 给你的是对应的 `Memory<T>` 视图；
* 用完后必须释放 `owner`，否则这块内存不能正确归还。

### 常见操作

#### 切片 `Slice`

```csharp
Memory<byte> buffer = new byte[100];
Memory<byte> body = buffer.Slice(10, 20);
```

和 `Span<T>` 一样：

* `Slice` 只是创建新的视图；
* 不会复制底层数据。

#### 获取 `Span<T>`

```csharp
Memory<byte> memory = new byte[10];
Span<byte> span = memory.Span;
span[0] = 42;
```

这是 `Memory<T>` 最核心的桥梁。

很多代码的模式本质上都是：

```csharp
Memory<T> -> .Span -> 真正操作
```

#### 只读转换

```csharp
Memory<byte> writable = new byte[16];
ReadOnlyMemory<byte> readOnly = writable;
```

`Memory<T>` 可以很自然地转成只读版本。

### 一个最重要的结论：传递用 `Memory<T>`，处理用 `Span<T>`

这是最值得记住的一句话。

例如异步读取：

```csharp
async Task<int> ReadDataAsync(Stream stream, Memory<byte> buffer)
{
    return await stream.ReadAsync(buffer);
}
```

这里用 `Memory<byte>` 是因为：

* 这个参数要跨 `await`
* API 需要一个可以安全异步传递的内存视图

但如果你在同步代码里真正要处理数据：

```csharp
static void ProcessBuffer(Span<byte> buffer)
{
    buffer[0] = 1;
}
```

就更适合直接用 `Span<byte>`。

所以你会经常看到一种组合方式：

```csharp
async Task HandleAsync(Stream stream, Memory<byte> buffer)
{
    int bytesRead = await stream.ReadAsync(buffer);
    ProcessBuffer(buffer.Span[..bytesRead]);
}
```

这正是 `Memory<T>` 和 `Span<T>` 的典型协作方式。

### 为什么 `Memory<T>` 能跨 `await`？

因为它不是 `ref struct`。

更直白地说：

* `Memory<T>` 本身可以安全地存在堆上；
* 它可以被异步状态机持有；
* 因此它能作为 async 方法里的变量、参数、字段存在。

但这里要注意一件非常重要的事：

> 能跨 `await` 的是 `Memory<T>`，不是 `Memory<T>.Span`。

例如下面这种写法仍然是不对的：

```csharp
async Task BadAsync(Memory<byte> memory)
{
    Span<byte> span = memory.Span;
    await Task.Delay(1);
    span[0] = 1; // 不应该这样写
}
```

原因是：

* `span` 依旧是 `Span<byte>`
* `Span<byte>` 依旧不能跨 `await`

正确思路是：

* 先持有 `Memory<T>`
* 等需要同步处理时，再临时取 `.Span`

例如：

```csharp
async Task GoodAsync(Memory<byte> memory)
{
    await Task.Delay(1);
    memory.Span[0] = 1;
}
```

这里 `.Span` 的使用没有跨过 `await`，因此是安全的。

### 为什么 `Memory<T>` 能做字段？

因为它可以安全地活在堆上。

例如：

```csharp
public sealed class BufferHolder
{
    public Memory<byte> Buffer { get; }

    public BufferHolder(byte[] bytes)
    {
        Buffer = bytes;
    }
}
```

这在 `Span<T>` 里是不成立的。

这也是 `Memory<T>` 最典型的适用场景之一：

* 某个类需要持有一段内存视图；
* 但又不想总是暴露完整数组；
* 或者希望保留切片后的某段区域。

### `Memory<T>` 不等于“拥有内存”

这是一个很容易忽略，但非常关键的点。

`Memory<T>` 只是视图，不是所有者。

例如：

```csharp
byte[] bytes = new byte[100];
Memory<byte> memory = bytes.AsMemory(10, 20);
```

这里真正拥有数据的是：

```csharp
bytes
```

不是：

```csharp
memory
```

同理，如果你用的是内存池：

```csharp
using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(1024);
Memory<byte> memory = owner.Memory;
```

真正负责生命周期的是：

```csharp
owner
```

不是：

```csharp
memory
```

所以一旦 `owner.Dispose()`，你就不应该再继续使用这段 `Memory<byte>`。

### `MemoryPool<T>` 和 `IMemoryOwner<T>` 为什么经常和 `Memory<T>` 一起出现？

因为 `Memory<T>` 解决的是“视图表达”，而 `MemoryPool<T>` 解决的是“底层内存复用”。

它们组合起来非常适合这些场景：

* 网络服务器；
* 高吞吐文件处理；
* 大量临时缓冲区；
* 希望减少 `GC` 压力。

典型写法：

```csharp
using System.Buffers;

async Task<int> ReadOnceAsync(Stream stream)
{
    using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(4096);
    Memory<byte> buffer = owner.Memory.Slice(0, 4096);

    int bytesRead = await stream.ReadAsync(buffer);

    Process(buffer.Span[..bytesRead]);
    return bytesRead;
}

static void Process(ReadOnlySpan<byte> data)
{
    // 同步处理
}
```

这段代码的价值在于：

* 通过 `MemoryPool<T>` 避免反复分配新数组；
* 通过 `Memory<T>` 安全地跨异步传递；
* 通过 `.Span` 在同步处理阶段保持高性能。

### `ReadOnlyMemory<T>` 的定位

如果你的 API 本来就不希望调用方修改数据，那么应该优先考虑：

```csharp
ReadOnlyMemory<T>
```

例如：

```csharp
public sealed class Message
{
    public ReadOnlyMemory<byte> Payload { get; }

    public Message(ReadOnlyMemory<byte> payload)
    {
        Payload = payload;
    }
}
```

这个设计表达得更清楚：

* 这是一段需要长期持有的内存视图；
* 但调用方不应该修改它。

### 典型使用场景

#### 1. 异步 `IO`

这是 `Memory<T>` 最典型的主场。

例如：

* `Stream.ReadAsync(Memory<byte>)`
* `Socket.ReceiveAsync(Memory<byte>)`
* `PipeReader` / `Pipelines`

这些 API 本身就天然适合 `Memory<T>`。

#### 2. 需要长期持有某段缓冲区

比如：

* 某个对象内部保留消息体的一段视图；
* 某个组件缓存一段数据窗口；
* 某个解析器分阶段传递缓冲区。

#### 3. 只想暴露一段视图，不想暴露整个数组

这时 `Memory<T>` / `ReadOnlyMemory<T>` 往往比直接返回 `byte[]` 更合适。

#### 4. 与内存池配合减少分配

这在高性能系统里非常常见。

### `Memory<T>`、数组、`ArraySegment<T>` 怎么选？

这是一个很实际的问题。

#### 用数组

适合：

* 你明确拥有这块数据；
* 生命周期长；
* 只是普通业务代码；
* 不需要强调切片视图语义。

#### 用 `ArraySegment<T>`

它也是“数组的一段视图”，比 `Memory<T>` 更早。

但它的问题是：

* 只适用于数组；
* 表达能力比 `Memory<T>` 更窄；
* 和现代 `Span` / `Memory` API 体系配合不如后者自然。

#### 用 `Memory<T>`

适合：

* 你要表达“这是内存视图，不是完整数组”；
* 需要跨异步边界；
* 想和 `Span<T>` / `MemoryPool<T>` 统一协作。

所以今天的新代码里，如果场景允许，`Memory<T>` 往往比 `ArraySegment<T>` 更现代、更统一。

### `Memory<T>` 的几个常见坑

#### 1. 跨 `await` 持有 `.Span`

前面已经说过，这是最常见的问题之一。

记住：

* 可以跨 `await` 的是 `Memory<T>`
* 不是 `Memory<T>.Span`

#### 2. 忘记管理底层所有权

如果 `Memory<T>` 来自 `MemoryPool<T>` 或自定义 owner，那么真正要管理的是 owner 的生命周期。

#### 3. 把 `Memory<T>` 当成拥有者

它只是视图，不负责释放底层资源。

#### 4. 明明只读却还暴露 `Memory<T>`

如果不需要写权限，就优先用 `ReadOnlyMemory<T>`。

#### 5. 在纯同步高性能路径里滥用 `Memory<T>`

如果根本不需要跨异步边界，也不需要做字段，很多时候直接用 `Span<T>` / `ReadOnlySpan<T>` 更简单直接。

### 一套比较务实的实践建议

如果你准备在项目里正确使用 `Memory<T>`，下面这些建议很实用：

* 只在需要跨 `await`、做字段或长期持有时使用 `Memory<T>`；
* 真正做同步读写时，优先临时取 `.Span` 处理；
* 输入只读数据时，优先考虑 `ReadOnlyMemory<T>`；
* 需要内存复用时，优先考虑 `MemoryPool<T>` + `IMemoryOwner<T>`；
* 不要把 `Memory<T>` 和底层所有权混为一谈；
* 如果场景纯同步短生命周期，优先考虑 `Span<T>`。

### 总结

`Memory<T>` 的本质，不是“能跨异步的 Span<T> 这么简单”，而是：

> 它是 .NET 内存体系里那个负责“可持久视图表达”的类型。

你可以这样理解它：

* `Span<T>` 负责同步高性能处理；
* `ReadOnlySpan<T>` 负责同步只读处理；
* `Memory<T>` 负责跨异步、跨组件、跨生命周期地传递视图；
* `ReadOnlyMemory<T>` 负责只读的长期视图表达。

如果你在做这些事情：

* 异步 `IO`
* 缓冲区传递
* 内存池复用
* 需要字段持有某段内存
* 想统一 `Span` 和异步 API 的边界

那 `Memory<T>` 基本就是必须掌握的一项基础能力。
