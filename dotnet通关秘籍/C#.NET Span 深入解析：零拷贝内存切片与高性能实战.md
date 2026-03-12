### 简介

在 `.NET` 里，只要你开始关注性能，尤其是这些场景：

* 字符串解析；
* 网络协议处理；
* 文件读取和缓冲区操作；
* JSON、CSV、日志、报文解析；
* 高频数组切片；

你几乎一定会遇到 `Span<T>`。

它之所以重要，不是因为它“新”，而是因为它解决的是一个非常实际的问题：

> 如何在不额外分配内存、不额外复制数据的前提下，高效地操作一段连续内存？

过去很多代码为了拿到一段子数据，会写出这种逻辑：

```csharp
string part = text.Substring(0, 5);
byte[] header = bytes.Skip(0).Take(8).ToArray();
```

这些写法的问题不是“能不能用”，而是：

* 额外创建对象；
* 额外拷贝数据；
* 高频场景下带来 `GC` 压力；
* 本来只是想看一眼某段数据，结果却复制了一份。

`Span<T>` 的核心价值，就是把“复制一份再处理”，变成“直接在原始内存上切一片来处理”。

### Span 到底是什么？

可以先用一句最直白的话理解：

> `Span<T>` 是对一段连续内存的可写视图。

它自己通常不拥有数据，而只是“指向”一段已有的连续内存。

所以你可以把它理解成：

* 不是数组；
* 不是集合；
* 不是新的内存块；
* 而是某段连续内存的窗口。

这个窗口可以指向：

* 数组；
* `stackalloc` 分配的栈内存；
* 非托管内存；
* 其他可转换为连续内存的区域。

对应的只读版本是：

```csharp
ReadOnlySpan<T>
```

它最常见的应用，就是：

* 只读字符串处理；
* 不希望修改原始数据；
* API 只表达“读”，不表达“写”。

### 为什么需要 Span？

因为传统做法在很多高频场景下开销并不小。

例如字符串截取：

```csharp
string text = "Hello,World";
string left = text.Substring(0, 5);
```

`Substring` 的语义是：

* 创建一个新的字符串对象；
* 把对应字符复制过去。

而 `ReadOnlySpan<char>`：

```csharp
ReadOnlySpan<char> left = text.AsSpan(0, 5);
```

语义则是：

* 只是拿到原始字符串里前 5 个字符的视图；
* 不创建新字符串；
* 不复制字符数据。

这就是所谓的“零拷贝切片”。

### Span 的核心本质

`Span<T>` 从概念上可以理解成两部分：

* 一段内存的起始位置；
* 这段内存的长度。

也就是说，它更像：

```text
(pointer/reference, length)
```

而不是：

```text
真正拥有数据的一块新容器
```

这也是为什么：

* 对 `Span<T>` 的切片不会复制数据；
* 修改 `Span<T>` 的内容，本质上是在修改它指向的原始内存。

例如：

```csharp
int[] numbers = { 1, 2, 3, 4, 5 };
Span<int> span = numbers.AsSpan(1, 3);

span[0] = 99;

Console.WriteLine(numbers[1]); // 99
```

这里不是改了 `span` 的副本，而是直接改到了原数组。

### Span 最关键的特性：它是 `ref struct`

`Span<T>` 最关键的语言层特性，不是泛型，而是它是一个：

```csharp
ref struct
```

这会带来一系列很重要的限制，而这些限制并不是“设计缺陷”，而是为了安全。

你可以把这个设计理解成：

* `Span<T>` 很高性能；
* 但编译器必须严格限制它的生命周期，防止它逃逸到不安全的位置。

这也是为什么 `Span<T>`：

* 不能作为类字段；
* 不能装箱；
* 不能跨 `await`；
* 不能被闭包捕获；
* 不能随意长期存活在堆上。

这些限制的根源，基本都可以追溯到一句话：

> `Span<T>` 必须保证它引用的那段内存，在使用期间始终是安全有效的。

### 最常见的几种创建方式

#### 1. 从数组创建

```csharp
int[] array = { 1, 2, 3, 4, 5 };

Span<int> span1 = array;
Span<int> span2 = array.AsSpan();
Span<int> span3 = array.AsSpan(1, 3);
```

这里：

* `span1` 指向整个数组；
* `span3` 指向数组中 `[2,3,4]` 那一段；
* 没有发生新数组分配。

#### 2. 从字符串创建只读视图

字符串本身不可变，所以对应的是：

```csharp
ReadOnlySpan<char>
```

```csharp
string text = "Hello,World";

ReadOnlySpan<char> span = text.AsSpan();
ReadOnlySpan<char> left = text.AsSpan(0, 5);
```

这是 `Span` 系列在业务代码里最常见的入口之一。

#### 3. 从 `stackalloc` 创建栈上缓冲区

```csharp
Span<byte> buffer = stackalloc byte[256];
buffer.Clear();
```

这意味着：

* 缓冲区直接分配在栈上；
* 不经过堆；
* 不进入 `GC` 管理。

这在小型临时缓冲区场景里非常高效。

#### 4. 从非托管内存创建

`Span<T>` 也可以包装非托管内存，但这已经属于更底层的用法，通常要更谨慎。

### `Span<T>` 和 `ReadOnlySpan<T>` 怎么选？

这个选择其实很简单。

#### 用 `Span<T>`

当你需要：

* 修改数据；
* 把某段缓冲区传给下游写入；
* 做原地处理。

例如：

```csharp
Span<byte> bytes = stackalloc byte[4];
bytes[0] = 1;
```

#### 用 `ReadOnlySpan<T>`

当你只需要：

* 读取数据；
* 不允许修改；
* 想让 API 语义更清晰。

例如：

```csharp
static int CountComma(ReadOnlySpan<char> text)
{
    int count = 0;
    foreach (char c in text)
    {
        if (c == ',')
        {
            count++;
        }
    }
    return count;
}
```

在 API 设计上，一个很务实的建议是：

* 输入参数能只读就尽量用 `ReadOnlySpan<T>`；
* 真要修改时再用 `Span<T>`。

### 最常用的几个操作

#### 索引访问

```csharp
Span<int> span = new[] { 1, 2, 3 };
int value = span[1]; // 2
```

#### 切片 `Slice`

```csharp
Span<int> numbers = new[] { 1, 2, 3, 4, 5 };
Span<int> middle = numbers.Slice(1, 3); // 2,3,4
```

它的重点在于：

* 只是换了一个视图；
* 没有复制数据。

#### 复制 `CopyTo`

```csharp
Span<int> source = new[] { 1, 2, 3 };
Span<int> target = stackalloc int[3];

source.CopyTo(target);
```

只有你显式调用复制相关操作时，才真的会发生数据复制。

#### 填充和清空

```csharp
Span<byte> buffer = stackalloc byte[8];
buffer.Fill(0x20);
buffer.Clear();
```

#### 查找

```csharp
ReadOnlySpan<char> text = "a,b,c".AsSpan();
int index = text.IndexOf(',');
```

这类 API 在字符串解析里非常常见。

### 为什么 `Span<T>` 在字符串处理中这么有价值？

因为字符串处理是最容易不小心产生临时对象的地方。

例如以前很多代码会这样写：

```csharp
string line = "Alice,18,China";
string[] parts = line.Split(',');
```

问题在于：

* `Split` 会创建数组；
* 还会创建多个子字符串；
* 高并发高频场景下，这类分配非常可观。

而用 `ReadOnlySpan<char>` 可以把问题改写成“在原字符串上定位并切片”。

例如：

```csharp
ReadOnlySpan<char> line = "Alice,18,China".AsSpan();

int firstComma = line.IndexOf(',');
ReadOnlySpan<char> name = line[..firstComma];

ReadOnlySpan<char> rest = line[(firstComma + 1)..];
int secondComma = rest.IndexOf(',');
ReadOnlySpan<char> age = rest[..secondComma];
ReadOnlySpan<char> country = rest[(secondComma + 1)..];
```

这里的 `name`、`age`、`country`：

* 都不是新字符串；
* 都只是原字符串上的只读切片。

如果后面再配合：

```csharp
int.TryParse(age, out int parsedAge);
```

就能把很多中间对象都省掉。

### `stackalloc` 和 `Span<T>` 是一对高频搭档

很多人学 `Span<T>`，真正开始觉得它强，是从 `stackalloc` 开始的。

例如：

```csharp
Span<char> buffer = stackalloc char[32];
```

这表示：

* 直接在当前方法栈帧里申请 32 个 `char`；
* 然后用 `Span<char>` 来安全访问它；
* 用完作用域结束自动回收。

这非常适合：

* 小块临时缓冲；
* 格式化；
* 协议解析；
* 数字转换；
* 避免反复申请小数组。

但这里也有一个很重要的务实建议：

* `stackalloc` 适合小块、短生命周期内存；
* 不要随手用它申请很大缓冲区；
* 过大的栈内存会带来栈压力。

### 为什么 `Span<T>` 不能跨 `await`？

这是最容易被问到的问题之一。

例如下面的代码通常就不行：

```csharp
async Task DemoAsync()
{
    Span<byte> buffer = stackalloc byte[128];
    await Task.Delay(1);
    buffer[0] = 1;
}
```

根本原因是：

* `await` 会把方法拆成状态机；
* 局部变量可能需要提升到堆上；
* 但 `Span<T>` 是 `ref struct`，不能安全地逃逸到堆上。

所以这类跨异步边界的场景，通常应该考虑：

```csharp
Memory<T>
```

而不是 `Span<T>`。

### `Span<T>` 和 `Memory<T>` 的边界要分清

这是理解 `Span` 体系最重要的一道坎。

可以先记这张简化表：

| 类型 | 可否跨 `await` | 可否做字段 | 典型场景 |
| --- | --- | --- | --- |
| `Span<T>` | 否 | 否 | 同步、短生命周期、高性能处理 |
| `ReadOnlySpan<T>` | 否 | 否 | 同步只读切片 |
| `Memory<T>` | 是 | 是 | 需要堆存活或异步边界 |
| `ReadOnlyMemory<T>` | 是 | 是 | 只读异步场景 |

一个简单判断规则：

* 只在当前同步方法里临时处理数据，用 `Span<T>`；
* 需要跨方法存活、跨 `await`、放字段里，用 `Memory<T>`。

例如：

```csharp
public sealed class BufferOwner
{
    public Memory<byte> Buffer { get; }

    public BufferOwner(byte[] bytes)
    {
        Buffer = bytes;
    }
}
```

这里可以存 `Memory<byte>`，但不能存 `Span<byte>`。

### `Span<T>`、数组、`ArraySegment<T>` 到底怎么选？

这是个非常实用的问题。

#### 直接用数组

适合：

* 你确实拥有这块数据；
* 生命周期长；
* 不在乎切片分配问题；
* API 兼容性优先。

#### `ArraySegment<T>`

它也是“数组的一段视图”，比 `Span<T>` 更早。

但它的问题是：

* 只适用于数组；
* API 生态不如 `Span<T>` 现代；
* 处理字符串、栈内存、非托管内存都不方便。

#### `Span<T>`

适合：

* 统一处理各种连续内存；
* 切片但不复制；
* 同步高性能路径。

所以今天的新代码里，如果场景允许，`Span<T>` 往往比 `ArraySegment<T>` 更自然。

### 和 `Substring`、`Split` 相比，Span 的价值到底在哪？

不是说 `Substring`、`Split` 不能用，而是要看场景。

如果只是偶尔处理一两次字符串，直接用普通 API 完全没问题。

但如果是这些场景：

* 日志逐行解析；
* 协议报文解析；
* 网关、代理、中间件；
* 高频文本处理；
* 框架级基础组件；

那 `ReadOnlySpan<char>` 的优势会非常明显，因为它可以：

* 避免大量中间字符串；
* 避免大量临时数组；
* 降低 `GC` 压力；
* 提高吞吐和稳定性。

### 一个更接近实战的例子：解析请求行

假设有一行简单数据：

```text
GET /api/users HTTP/1.1
```

我们想拿到方法、路径、协议。

```csharp
static (ReadOnlySpan<char> method, ReadOnlySpan<char> path, ReadOnlySpan<char> protocol)
    ParseRequestLine(ReadOnlySpan<char> line)
{
    int firstSpace = line.IndexOf(' ');
    ReadOnlySpan<char> method = line[..firstSpace];

    line = line[(firstSpace + 1)..];
    int secondSpace = line.IndexOf(' ');
    ReadOnlySpan<char> path = line[..secondSpace];
    ReadOnlySpan<char> protocol = line[(secondSpace + 1)..];

    return (method, path, protocol);
}
```

这个例子想表达的重点不是协议解析本身，而是：

* 全程都在原始字符数据上切片；
* 没有 `Split(' ')`；
* 没有生成多个中间字符串；
* 非常适合高频解析场景。

### `Span<T>` 的典型限制，一定要记住

这些限制几乎都是 `ref struct` 带来的。

#### 1. 不能作为类字段

```csharp
public class BadBuffer
{
    public Span<byte> Buffer; // 编译错误
}
```

#### 2. 不能装箱

```csharp
Span<int> span = stackalloc int[4];
object obj = span; // 编译错误
```

#### 3. 不能作为泛型集合元素

```csharp
// List<Span<int>> list = new(); // 编译错误
```

#### 4. 不能被闭包捕获

```csharp
Span<int> span = stackalloc int[4];
// Action action = () => Console.WriteLine(span[0]); // 编译错误
```

#### 5. 不能跨异步和迭代器边界

这前面已经解释过，是生命周期安全问题。

### 性能该怎么理解？不要神化 Span

`Span<T>` 很强，但它不是“写了就一定更快”的魔法。

你需要先分清它优化的是什么：

* 避免分配；
* 避免拷贝；
* 统一连续内存访问方式。

如果你的瓶颈根本不在这些地方，那改成 `Span<T>` 也未必有明显收益。

所以更务实的结论是：

* 高频内存切片、解析、缓冲区处理，`Span<T>` 很值得；
* 普通业务代码，如果只是为了“显得高级”而强上 `Span<T>`，通常没必要。

### 一套比较稳妥的实践建议

如果你准备在项目里真正用 `Span<T>`，下面这些建议比较实用：

* 输入只读数据时，优先用 `ReadOnlySpan<T>`；
* 需要修改缓冲区时，再用 `Span<T>`；
* 只在同步短生命周期场景用 `Span<T>`；
* 跨 `await`、跨字段、跨长期生命周期，用 `Memory<T>`；
* 小块临时缓冲可优先考虑 `stackalloc`；
* 处理字符串解析时，优先考虑 `AsSpan()` + `Slice/IndexOf`；
* 不要为了炫技把所有普通逻辑都改成 `Span<T>`。

### 总结

`Span<T>` 的本质，不是“另一个数组类型”，而是对连续内存的一层高性能视图抽象。

你可以这样理解它：

* 数组是数据拥有者；
* `Span<T>` 是数据视图；
* `ReadOnlySpan<T>` 是只读数据视图；
* `Memory<T>` 是适合跨堆和异步边界的长期视图。

在今天的 `.NET` 项目里，只要你在做这些事情：

* 高频字符串解析；
* 协议和报文处理；
* 缓冲区切片；
* 零拷贝优化；
* 减少中间对象分配；

那 `Span<T>` 几乎都是值得优先掌握的基础能力。
