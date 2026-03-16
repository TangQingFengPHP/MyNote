### 简介

如果说 `Span<T>` 解决的是“如何高效处理一段连续内存”，`Memory<T>` 解决的是“如何跨异步边界持有一段连续内存”，那么 `ReadOnlySequence<T>` 解决的就是另一个更真实的问题：

> 当数据根本不是一整块连续内存，而是分散在多段内存里时，怎么仍然零拷贝地把它当成一条完整数据流来处理？

这类场景在真实系统里非常常见：

* 网络收包往往是分段到达的；
* `PipeReader` 读取出来的数据经常不是单段连续缓冲区；
* 大文件、流式协议、消息边界解析也经常面对“多段内存拼成一个逻辑消息”的问题。

如果你只会 `Span<T>`，你很快就会遇到一个现实限制：

* `Span<T>` 只能表示一段连续内存；
* 但很多高性能 `IO` 场景下，数据本来就是多段的。

`ReadOnlySequence<T>` 的意义，就是把“多段只读内存”统一抽象成“一条可切片、可遍历、可定位的逻辑序列”。

### `ReadOnlySequence<T>` 到底是什么？

可以先用一句最直白的话理解：

> `ReadOnlySequence<T>` 是对一段逻辑连续、物理上可能分散的只读内存序列的抽象。

这里“逻辑连续”和“物理分散”这两个词非常关键。

例如下面两种情况：

#### 情况 1：单段内存

```csharp
byte[] bytes = [1, 2, 3, 4, 5];
ReadOnlySequence<byte> sequence = new ReadOnlySequence<byte>(bytes);
```

这里其实只有一段内存。

#### 情况 2：多段内存

```text
segment1: [1, 2]
segment2: [3, 4, 5]
segment3: [6, 7]
```

虽然底层分成三段，但 `ReadOnlySequence<byte>` 可以把它们看成逻辑上的：

```text
[1, 2, 3, 4, 5, 6, 7]
```

这就是它最核心的价值。

### 为什么 `ReadOnlySpan<T>` 不够用？

因为 `ReadOnlySpan<T>` 只能表示一段连续内存。

这没有问题，但它的能力边界非常明确：

* 适合字符串切片；
* 适合单个数组缓冲区；
* 适合单段同步处理。

可是一旦底层数据是多段的，你就必须面对这些问题：

* 要不要先把所有段拼成一个新数组？
* 要不要为了“方便处理”额外复制一次？
* 要不要在解析报文前先做一次合并？

这些额外复制，在高吞吐系统里往往就是不必要的性能损耗。

`ReadOnlySequence<T>` 的核心价值，就是尽量不复制，直接在多段视图上处理数据。

### 它和 `ReadOnlyMemory<T>` 又是什么关系？

可以先记这张表：

| 类型 | 表示什么 | 适合什么场景 |
| --- | --- | --- |
| `ReadOnlySpan<T>` | 单段只读连续内存视图 | 同步、单段处理 |
| `ReadOnlyMemory<T>` | 可跨异步持有的单段只读内存视图 | 单段异步场景 |
| `ReadOnlySequence<T>` | 单段或多段只读内存序列 | 流式 `IO`、多段缓冲区 |

所以更准确地说：

* `ReadOnlySpan<T>` / `ReadOnlyMemory<T>` 解决的是“单段”
* `ReadOnlySequence<T>` 解决的是“多段”

### 先看一个最小示例

从单段数组构造：

```csharp
using System.Buffers;

byte[] bytes = [1, 2, 3, 4, 5];
ReadOnlySequence<byte> sequence = new ReadOnlySequence<byte>(bytes);

Console.WriteLine(sequence.Length);    // 5
Console.WriteLine(sequence.IsSingleSegment); // True
Console.WriteLine(sequence.FirstSpan[0]);    // 1
```

这个例子看起来不惊艳，因为它只是单段。

但它先帮你建立几个最基础的认知：

* `ReadOnlySequence<T>` 可以表示序列长度；
* 它可能是单段，也可能是多段；
* 如果是单段，可以直接通过 `FirstSpan` 高效访问。

### 为什么它经常和 `PipeReader` 一起出现？

因为 `System.IO.Pipelines` 天然就是多段缓冲区模型。

典型代码长这样：

```csharp
ReadResult result = await reader.ReadAsync();
ReadOnlySequence<byte> buffer = result.Buffer;
```

这里的 `buffer` 不是 `byte[]`，也不是 `ReadOnlyMemory<byte>`，而是：

```csharp
ReadOnlySequence<byte>
```

原因很简单：

* 管道内部缓冲区可能分成多段；
* 编译器和运行时不想为了你“看起来方便”就强行拼成一段；
* 所以它直接把底层真实模型暴露出来。

这也是 `ReadOnlySequence<T>` 最核心的主场。

### `ReadOnlySequence<T>` 的几个关键成员

先把最常用的几个成员记住：

| 成员 | 作用 |
| --- | --- |
| `Length` | 逻辑总长度 |
| `IsEmpty` | 是否为空 |
| `IsSingleSegment` | 是否只有一段 |
| `First` | 第一段 `ReadOnlyMemory<T>` |
| `FirstSpan` | 第一段 `ReadOnlySpan<T>` |
| `Start` / `End` | 序列边界位置 |
| `Slice(...)` | 按位置或长度切片 |
| `ToArray()` | 复制为数组 |
| `TryGet(...)` | 逐段遍历 |

这几个 API 基本就覆盖了日常使用的核心操作。

### `SequencePosition` 是什么？

这是学 `ReadOnlySequence<T>` 时最容易陌生的一个概念。

`ReadOnlySequence<T>` 既然可以跨多段内存，那显然不能再用一个简单的 `int index` 表示任意位置。

因为同样的逻辑位置可能对应：

* 第 1 段偏移 2；
* 或第 3 段偏移 0；

所以它用的是：

```csharp
SequencePosition
```

你可以把它理解成：

> 一个用于标识“多段序列中某个位置”的游标。

它不是普通整数索引，而是能描述“位于哪一段、该段偏移多少”的抽象位置。

这也是为什么很多 API 都长这样：

```csharp
sequence.Slice(position)
sequence.Slice(start, end)
sequence.GetPosition(10)
```

### 最常见的遍历方式：`TryGet`

如果你要处理多段序列，一个非常常见的方式就是逐段遍历。

```csharp
using System.Buffers;

ReadOnlySequence<byte> sequence = new ReadOnlySequence<byte>(new byte[] { 1, 2, 3, 4 });

var position = sequence.Start;
while (sequence.TryGet(ref position, out ReadOnlyMemory<byte> memory))
{
    ReadOnlySpan<byte> span = memory.Span;
    foreach (byte b in span)
    {
        Console.WriteLine(b);
    }
}
```

这里的逻辑其实很简单：

* 从 `Start` 开始；
* 每次拿到一段 `ReadOnlyMemory<byte>`；
* 处理这一段；
* `position` 自动前进到下一段。

也就是说：

* `TryGet` 是按段遍历；
* 不是按元素遍历。

这在多段缓冲区场景里非常重要。

### `FirstSpan` 很方便，但不要误会它

很多人第一次看到：

```csharp
sequence.FirstSpan
```

会下意识以为它代表整个序列。

不是。

它只代表：

* 第一段的 `ReadOnlySpan<T>`

如果序列是多段的，那么：

* `FirstSpan` 只覆盖第一段；
* 后面的段你仍然需要继续处理。

所以：

* 如果 `sequence.IsSingleSegment == true`，`FirstSpan` 往往非常方便；
* 如果是多段，不能只靠 `FirstSpan` 处理完整数据。

### `Slice` 是 `ReadOnlySequence<T>` 的高频能力

和 `Span<T>` / `Memory<T>` 一样，`ReadOnlySequence<T>` 也支持切片。

例如：

```csharp
ReadOnlySequence<byte> sequence = new ReadOnlySequence<byte>(new byte[] { 1, 2, 3, 4, 5 });
ReadOnlySequence<byte> body = sequence.Slice(1, 3);
```

这表示逻辑上的：

```text
[2, 3, 4]
```

这里的重点仍然是：

* 切片只是视图变化；
* 不代表发生了底层复制。

在多段场景里，这个能力尤其重要，因为它允许你：

* 不拼接整个消息；
* 直接在原始多段缓冲区上切出 header、body、payload。

### 一个更贴近实战的例子：按换行切协议帧

假设你在处理文本协议，消息以 `\n` 结尾。

在 `PipeReader` 里经常会写出类似逻辑：

```csharp
using System.Buffers;
using System.IO.Pipelines;

static bool TryReadLine(ref ReadOnlySequence<byte> buffer, out ReadOnlySequence<byte> line)
{
    SequencePosition? position = buffer.PositionOf((byte)'\n');
    if (position is null)
    {
        line = default;
        return false;
    }

    line = buffer.Slice(0, position.Value);
    buffer = buffer.Slice(buffer.GetPosition(1, position.Value));
    return true;
}
```

这段代码的价值非常大，因为它说明了 `ReadOnlySequence<T>` 最经典的用途：

* 在多段数据里查找边界；
* 找到一条完整消息；
* 用 `Slice` 切出来；
* 不做中间拷贝；
* 再把剩余未消费数据继续保留。

如果你用普通数组思维做这件事，往往很容易走向：

* 先拼包；
* 再复制；
* 再切；
* 再复制。

这恰恰是高性能系统最想避免的。

### 为什么很多人最后会配合 `SequenceReader<T>` 一起用？

因为直接操作 `ReadOnlySequence<T>` 虽然很强，但对于“逐字节、逐字符解析”来说，有时候还是偏底层。

这时通常会进一步使用：

```csharp
SequenceReader<byte>
```

它建立在 `ReadOnlySequence<T>` 之上，提供更适合协议解析的 API，例如：

* `TryRead`
* `TryPeek`
* `TryAdvanceTo`
* `UnreadSpan`

你可以把它理解成：

* `ReadOnlySequence<T>` 负责表示多段数据；
* `SequenceReader<T>` 负责更方便地消费这条多段数据流。

所以很多真正的协议解析代码里，这两个类型经常一起出现。

### 单段优化和多段处理，应该怎么写？

一个很务实的写法是：

* 如果是单段，优先走 `FirstSpan` 的快路径；
* 如果是多段，再走逐段遍历或 `SequenceReader<T>`。

例如：

```csharp
static int CountComma(ReadOnlySequence<byte> sequence)
{
    int count = 0;

    if (sequence.IsSingleSegment)
    {
        foreach (byte b in sequence.FirstSpan)
        {
            if (b == (byte)',')
            {
                count++;
            }
        }

        return count;
    }

    var position = sequence.Start;
    while (sequence.TryGet(ref position, out ReadOnlyMemory<byte> memory))
    {
        foreach (byte b in memory.Span)
        {
            if (b == (byte)',')
            {
                count++;
            }
        }
    }

    return count;
}
```

这种写法的思路很常见：

* 单段路径尽量简单直接；
* 多段路径再使用更完整的遍历逻辑。

### 如果我就想要一个数组，能不能直接 `ToArray()`？

当然可以：

```csharp
byte[] bytes = sequence.ToArray();
```

但这里要非常清楚：

* 这会发生复制；
* 也意味着你放弃了 `ReadOnlySequence<T>` 最核心的零拷贝价值。

所以 `ToArray()` 不是不能用，而是应该明确知道：

* 这是主动选择“为了方便，接受一次复制”。

如果你本来就在做高性能流式解析，那通常应该尽量延后甚至避免这一步。

### `ReadOnlySequence<T>` 的几个典型使用场景

#### 1. `Pipelines`

这是它最典型的主场，没有之一。

#### 2. 协议解析

例如：

* HTTP
* Redis 协议
* 自定义 TCP 文本协议
* 二进制帧协议

#### 3. 流式处理

例如：

* 按行解析
* 分隔符切包
* 多段缓冲区拼装逻辑消息

#### 4. 避免拼包复制

如果你不想每次都先拼成一大块连续数组，`ReadOnlySequence<T>` 就非常适合。

### `ReadOnlySequence<T>` 的几个常见坑

#### 1. 把 `FirstSpan` 当成完整数据

这前面已经强调过了。

* `FirstSpan` 只是一段；
* 不是整个序列。

#### 2. 过早 `ToArray()`

一旦你第一时间就 `ToArray()`，多段零拷贝的优势基本就没了。

#### 3. 用“单段索引思维”理解它

`ReadOnlySequence<T>` 天生就是为多段设计的。

如果你脑子里一直只有“一个数组 + 一个下标”的模型，那会很容易写得别扭。

#### 4. 不理解 `SequencePosition`

它不是普通下标，而是多段序列位置游标。

#### 5. 在多段场景里强行用 `ReadOnlySpan<T>` 解决所有问题

`ReadOnlySpan<T>` 很快，但它只适合单段。

多段就是多段，不要硬把问题降维成单段。

### 和 `Span<T>`、`Memory<T>` 这组类型怎么配合理解？

如果把这几个类型放在一起看，会更容易形成整体认知。

| 类型 | 解决什么问题 |
| --- | --- |
| `Span<T>` | 如何高效处理一段连续内存 |
| `Memory<T>` | 如何跨异步边界持有一段连续内存 |
| `ReadOnlySequence<T>` | 如何表示并处理多段逻辑连续内存 |

也就是说，它们不是竞争关系，而是层次递进关系：

* 单段同步 -> `Span<T>`
* 单段异步/长期持有 -> `Memory<T>`
* 多段流式数据 -> `ReadOnlySequence<T>`

### 一套比较务实的使用建议

如果你准备在项目里正确使用 `ReadOnlySequence<T>`，下面这些建议比较实用：

* 只要面对 `PipeReader` 或多段缓冲区，就优先接受它的多段模型；
* 单段场景优先走 `IsSingleSegment` + `FirstSpan` 快路径；
* 多段遍历优先考虑 `TryGet` 或 `SequenceReader<T>`；
* 只有在确实需要连续内存时，再考虑 `ToArray()`；
* 先按“视图切片”思维写代码，不要一上来就想着“先拼成大数组”。

### 总结

`ReadOnlySequence<T>` 的本质，不是“另一个复杂集合类型”，而是 `.NET` 为多段内存数据流提供的零拷贝抽象。

你可以这样理解它：

* `Span<T>` 解决单段；
* `Memory<T>` 解决单段异步持有；
* `ReadOnlySequence<T>` 解决多段流式处理。

在今天的 `.NET` 高性能体系里，只要你接触这些内容：

* `PipeReader`
* 协议解析
* 网络报文处理
* 流式分段数据
* 零拷贝设计

那 `ReadOnlySequence<T>` 基本都是绕不过去的一项基础能力。
