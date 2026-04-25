### 简介

在 `.NET` 高性能内存编程里，`Span<T>` 解决了一个很实际的问题：

> 不复制数据，也能操作一段连续内存。

但 `Span<T>` 本身还是比较“规矩”的。

比如：

* `Span<byte>` 就是按字节看；
* `Span<int>` 就是按整数看；
* `ReadOnlyMemory<T>` 背后到底是不是数组，不一定直接暴露；
* 想把一段字节当成结构体读出来，也不能直接强转。

这时候就会遇到 `MemoryMarshal`。

一句话先说清楚：

> `MemoryMarshal` 是一组底层内存工具，用来在 `Span<T>`、`Memory<T>`、结构体和字节之间做零拷贝转换、读取和桥接。

它的价值很直接：

* 少分配；
* 少复制；
* 直接按另一种视角看同一段内存；
* 在二进制协议、文件格式、网络缓冲区、序列化场景里减少中间对象。

但它不是日常业务代码里的默认工具。

因为 `MemoryMarshal` 做的很多事，本质上是在绕过一部分类型系统的保护。用对了，代码很干净，性能也好；用错了，问题通常比较隐蔽。

这篇文章主要讲清楚：

* `MemoryMarshal` 到底是什么；
* 它和 `Span<T>`、`Memory<T>` 是什么关系；
* `Cast`、`AsBytes`、`Read`、`Write` 怎么用；
* `GetReference`、`TryGetArray` 适合什么场景；
* 什么情况下不该用它；
* 使用时最容易踩哪些坑。

### 先把定位说清楚

`MemoryMarshal` 位于：

```csharp
System.Runtime.InteropServices
```

它是一个静态工具类，主要服务于这些类型：

* `Span<T>`
* `ReadOnlySpan<T>`
* `Memory<T>`
* `ReadOnlyMemory<T>`

普通代码里，`Span<T>` 负责表达“一段连续内存”。  
`MemoryMarshal` 则负责做更底层的事：

* 把同一段内存按另一种类型解释；
* 把结构体和字节互相读取、写入；
* 从 `Memory<T>` 里尝试拿到底层数组；
* 从 `Span<T>` 里拿到第一个元素的 `ref`。

可以这么理解：

* `Span<T>` 是安全的内存窗口；
* `MemoryMarshal` 是操作这个窗口的底层工具箱。

### 它解决的不是“能不能做”，而是“要不要复制”

先看一个普通的二进制解析场景。

假设有 4 个字节：

```csharp
byte[] data = { 1, 0, 0, 0 };
```

想把它解析成一个 `int`，常见写法是：

```csharp
int value = BitConverter.ToInt32(data, 0);
```

这没问题，代码也清楚。

但在更复杂的场景里，比如一大块网络缓冲区里有很多结构体、很多字段、很多连续数据，如果每一步都复制、转换、创建对象，开销就会慢慢堆起来。

`MemoryMarshal` 的思路不是“再造一份数据”，而是：

> 同一段内存，换一种类型视角来看。

比如：

```csharp
Span<byte> bytes = data;
Span<int> ints = MemoryMarshal.Cast<byte, int>(bytes);

Console.WriteLine(ints[0]); // 1
```

这里没有创建新的 `int[]`，也没有把字节复制到另一个地方。

只是把原来的 4 个字节，按一个 `int` 来看。

### `Cast<TFrom, TTo>`：把一段内存换个类型看

`Cast` 是 `MemoryMarshal` 里最常见的方法之一。

```csharp
Span<TTo> MemoryMarshal.Cast<TFrom, TTo>(Span<TFrom> span)
```

它的作用是：

> 把 `Span<TFrom>` 背后的同一段内存，重新解释成 `Span<TTo>`。

例如：

```csharp
byte[] buffer =
{
    1, 0, 0, 0,
    2, 0, 0, 0
};

Span<int> values = MemoryMarshal.Cast<byte, int>(buffer);

Console.WriteLine(values[0]); // 1
Console.WriteLine(values[1]); // 2
```

这里的关系是：

* 原始内存长度是 8 个字节；
* 一个 `int` 是 4 个字节；
* 所以转换后的 `Span<int>` 长度是 2。

反过来也可以：

```csharp
int[] numbers = { 1, 2 };

Span<byte> bytes = MemoryMarshal.Cast<int, byte>(numbers);

Console.WriteLine(bytes.Length); // 8
```

这类写法适合：

* 把一批 `byte` 看成 `int`、`short`、`long`；
* 把结构体数组看成字节；
* 做二进制序列化、哈希、校验、网络发送前的数据视图转换。

但 `Cast` 不是普通类型转换。

下面这种理解是错的：

```text
把 byte 转成 int
```

更准确的是：

```text
把同一段内存按 int 的布局重新解释
```

这意味着它强依赖：

* 类型大小；
* 内存布局；
* 当前机器的字节序；
* 数据是否真的符合目标类型的解释方式。

### `AsBytes<T>`：把结构体内存看成字节

如果目标只是把某段结构体内存变成字节视图，`AsBytes` 比 `Cast<T, byte>` 更直观。

```csharp
int[] numbers = { 1, 2, 3 };

Span<byte> bytes = MemoryMarshal.AsBytes(numbers.AsSpan());

Console.WriteLine(bytes.Length); // 12
```

它没有复制 `numbers`。

`bytes` 看到的就是 `numbers` 底层那块内存的字节形式。

修改 `bytes` 也会影响原始数据：

```csharp
int[] numbers = { 1 };

Span<byte> bytes = MemoryMarshal.AsBytes(numbers.AsSpan());
bytes[0] = 2;

Console.WriteLine(numbers[0]);
```

在常见小端机器上，输出通常是：

```text
2
```

这也顺手暴露了一个关键点：

> `MemoryMarshal` 操作的是内存原始表示，不会帮忙屏蔽字节序差异。

跨平台协议、网络协议、文件格式，只要规定了字节序，就不能只靠 `MemoryMarshal` 猜。

这种场景通常要配合：

```csharp
System.Buffers.Binary.BinaryPrimitives
```

例如：

```csharp
int value = BinaryPrimitives.ReadInt32LittleEndian(buffer);
```

或者：

```csharp
int value = BinaryPrimitives.ReadInt32BigEndian(buffer);
```

`MemoryMarshal` 适合“按当前内存布局读取”。  
`BinaryPrimitives` 更适合“按明确字节序读取”。

### `Read<T>`：从字节里读出一个结构体

`Read<T>` 用来从 `ReadOnlySpan<byte>` 中读取一个结构体：

```csharp
T value = MemoryMarshal.Read<T>(buffer);
```

例如定义一个协议头：

```csharp
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct MessageHeader
{
    public int Id;
    public short Length;
    public byte Type;
}
```

读取：

```csharp
public static MessageHeader ReadHeader(ReadOnlySpan<byte> buffer)
{
    const int HeaderSize = 7;

    if (buffer.Length < HeaderSize)
    {
        throw new ArgumentException("缓冲区长度不够。", nameof(buffer));
    }

    return MemoryMarshal.Read<MessageHeader>(buffer);
}
```

这里没有手动一个字段一个字段解析，而是直接把前 7 个字节按 `MessageHeader` 读出来。

这很快，也很简洁。

但条件也很明确：

* 结构体布局必须可控；
* 字段顺序、对齐、大小要和二进制数据一致；
* 结构体里不能包含引用类型字段；
* 字节序必须符合预期。

所以协议结构体通常要显式标注：

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]
```

`Pack = 1` 的意思是减少默认对齐填充，让结构体布局更贴近协议定义。

但这也不是所有场景都要写死。结构体如果要和 C 侧 ABI 对齐，`Pack` 要按对方的真实布局来，不是越小越好。

### `Write<T>`：把结构体写进字节缓冲区

和 `Read<T>` 对应的是 `Write<T>`。

```csharp
MemoryMarshal.Write(buffer, in value);
```

例如：

```csharp
public static void WriteHeader(Span<byte> buffer, MessageHeader header)
{
    const int HeaderSize = 7;

    if (buffer.Length < HeaderSize)
    {
        throw new ArgumentException("缓冲区长度不够。", nameof(buffer));
    }

    MemoryMarshal.Write(buffer, in header);
}
```

这会把 `header` 的内存表示写到 `buffer` 里。

还是同一句话：

> 写进去的是结构体的内存表示，不是某种自动跨平台协议格式。

如果协议要求大端序，或者字段需要固定编码方式，就不能简单 `Write` 完事。

这种场景更适合一个字段一个字段写：

```csharp
BinaryPrimitives.WriteInt32BigEndian(buffer, header.Id);
BinaryPrimitives.WriteInt16BigEndian(buffer.Slice(4), header.Length);
buffer[6] = header.Type;
```

可读性更强，也不会把平台字节序混进协议里。

### `AsRef<T>`：把字节视图当成结构体引用

`Read<T>` 会返回一个结构体值。

有时需要直接拿到这段字节对应的结构体引用，可以用：

```csharp
ref T value = ref MemoryMarshal.AsRef<T>(span);
```

例如：

```csharp
Span<byte> buffer = stackalloc byte[7];

ref MessageHeader header = ref MemoryMarshal.AsRef<MessageHeader>(buffer);

header.Id = 1001;
header.Length = 32;
header.Type = 1;
```

这段代码的效果是：

* `header` 不是新对象；
* 它直接引用 `buffer` 里的那块内存；
* 修改 `header`，就是修改 `buffer`。

这类写法很底层，也更危险。

一般只有在非常明确的热路径里才值得这样做。普通解析代码里，`Read<T>` 和 `Write<T>` 已经够用。

### `TryGetArray`：从 `Memory<T>` 里尽量拿到底层数组

`Memory<T>` 比 `Span<T>` 更适合长期保存，也可以跨 `await`。

但它只是一个抽象。

`Memory<T>` 背后可能是：

* 数组；
* 字符串；
* 自定义 `MemoryManager<T>`；
* 其他内存来源。

有些老 API 只接受数组：

```csharp
void Send(byte[] buffer, int offset, int count);
```

如果手里是：

```csharp
ReadOnlyMemory<byte> memory
```

不一定能直接拿到数组。

这时可以用：

```csharp
if (MemoryMarshal.TryGetArray(memory, out ArraySegment<byte> segment))
{
    Send(segment.Array!, segment.Offset, segment.Count);
}
else
{
    byte[] rented = ArrayPool<byte>.Shared.Rent(memory.Length);

    try
    {
        memory.Span.CopyTo(rented);
        Send(rented, 0, memory.Length);
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(rented);
    }
}
```

这段代码表达的是：

* 如果 `Memory<byte>` 背后本来就是数组，就直接用；
* 如果不是数组，就走一次拷贝兜底。

这比盲目调用 `ToArray()` 更好，因为它保留了零拷贝机会。

### `GetReference`：拿到 `Span<T>` 的第一个元素引用

`GetReference` 会返回 `Span<T>` 第一个元素的 `ref`：

```csharp
Span<int> numbers = stackalloc int[3] { 1, 2, 3 };

ref int first = ref MemoryMarshal.GetReference(numbers);

first = 99;

Console.WriteLine(numbers[0]); // 99
```

这个方法常见于更底层的优化代码，比如和 `Unsafe.Add` 配合做无索引器访问：

```csharp
using System.Runtime.CompilerServices;

public static int Sum(ReadOnlySpan<int> values)
{
    ref int start = ref MemoryMarshal.GetReference(values);

    int sum = 0;

    for (int i = 0; i < values.Length; i++)
    {
        sum += Unsafe.Add(ref start, i);
    }

    return sum;
}
```

这类代码不是普通业务里的默认写法。

现代 JIT 对普通 `Span<T>` 循环已经做了很多优化。很多时候，下面这种代码已经足够好：

```csharp
public static int Sum(ReadOnlySpan<int> values)
{
    int sum = 0;

    foreach (int value in values)
    {
        sum += value;
    }

    return sum;
}
```

除非基准测试证明边界检查或索引器访问真的成了热点，否则没必要为了“看起来底层”而上 `GetReference + Unsafe.Add`。

另外，空 `Span<T>` 上调用 `GetReference` 时，返回的引用不能解引用。

所以这种代码必须先检查长度：

```csharp
if (values.IsEmpty)
{
    return 0;
}

ref int first = ref MemoryMarshal.GetReference(values);
```

### `CreateSpan`：从一个引用造出 Span

`CreateSpan` 可以从一个 `ref T` 和长度创建 `Span<T>`：

```csharp
Span<T> span = MemoryMarshal.CreateSpan(ref reference, length);
```

例如：

```csharp
int value = 10;

Span<int> one = MemoryMarshal.CreateSpan(ref value, 1);
one[0] = 20;

Console.WriteLine(value); // 20
```

这看起来很方便，但它是 `MemoryMarshal` 里风险更高的一类方法。

因为它相信传入的 `ref` 后面真的有 `length` 个连续元素。

如果只有一个变量，却写成：

```csharp
int value = 10;
Span<int> broken = MemoryMarshal.CreateSpan(ref value, 10);
```

那就相当于告诉运行时：

> 从这个 `int` 开始，后面还有 10 个连续 `int` 可以访问。

事实并不是这样。

这种错误不会像普通数组越界那样容易理解，后果也更难排查。

所以 `CreateSpan` 更适合框架代码、底层库代码，普通业务代码能不用就不用。

### 几个更贴近实际的使用示例

上面的例子主要是说明 API 行为，下面几个场景更接近真实代码。

#### 示例 1：解析固定协议头，再切出消息体

很多二进制协议都会有一个固定头部，比如：

* 4 字节消息长度；
* 2 字节消息类型；
* 2 字节标志位；
* 后面跟着消息体。

如果协议明确使用大端序，字段级解析更适合用 `BinaryPrimitives`：

```csharp
using System.Buffers.Binary;

public readonly record struct PacketHeader(
    int BodyLength,
    ushort MessageType,
    ushort Flags);

public static bool TryReadPacket(
    ReadOnlySpan<byte> buffer,
    out PacketHeader header,
    out ReadOnlySpan<byte> body)
{
    const int HeaderSize = 8;

    header = default;
    body = default;

    if (buffer.Length < HeaderSize)
    {
        return false;
    }

    int bodyLength = BinaryPrimitives.ReadInt32BigEndian(buffer);
    ushort messageType = BinaryPrimitives.ReadUInt16BigEndian(buffer.Slice(4));
    ushort flags = BinaryPrimitives.ReadUInt16BigEndian(buffer.Slice(6));

    if (bodyLength < 0 || buffer.Length < HeaderSize + bodyLength)
    {
        return false;
    }

    header = new PacketHeader(bodyLength, messageType, flags);
    body = buffer.Slice(HeaderSize, bodyLength);
    return true;
}
```

这个例子里没有直接用 `MemoryMarshal.Read<T>`，原因很简单：

* 协议字段有明确大端序；
* `MemoryMarshal.Read<T>` 按当前机器内存表示读；
* 字段级解析更清楚，也更稳定。

这也是 `MemoryMarshal` 的一个重要使用边界。

有些场景里，不用它反而更合适。

#### 示例 2：批量把结构体写入网络缓冲区

如果一批数据只在同一套系统内部传输，结构体布局也完全可控，可以把结构体数组直接看成字节。

例如一批传感器采样数据：

```csharp
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct SensorSample
{
    public int DeviceId;
    public long Timestamp;
    public float Value;
}
```

发送前拿到字节视图：

```csharp
public static ReadOnlySpan<byte> AsPayload(ReadOnlySpan<SensorSample> samples)
{
    return MemoryMarshal.AsBytes(samples);
}
```

如果要写入已有缓冲区：

```csharp
public static int WriteSamples(
    ReadOnlySpan<SensorSample> samples,
    Span<byte> destination)
{
    ReadOnlySpan<byte> payload = MemoryMarshal.AsBytes(samples);

    if (destination.Length < payload.Length)
    {
        throw new ArgumentException("目标缓冲区长度不够。", nameof(destination));
    }

    payload.CopyTo(destination);
    return payload.Length;
}
```

这类写法的好处是：

* 不需要逐个字段写入；
* 不需要为每条数据创建临时 `byte[]`；
* 一批结构体可以直接形成连续字节视图。

但前提也很硬：

* 发送方和接收方都认可这个结构体布局；
* 没有跨语言、跨架构、跨版本的布局差异；
* 不要求独立于平台字节序。

只要数据要落盘长期保存，或者要跨系统传输，通常更推荐定义明确的协议格式，而不是直接裸写结构体内存。

#### 示例 3：从文件块中批量读取结构体记录

某些内部文件格式会把固定大小记录连续写入文件。

例如文件内容是一批 `SensorSample`：

```csharp
public static ReadOnlySpan<SensorSample> ReadSamples(ReadOnlySpan<byte> fileBlock)
{
    int size = Marshal.SizeOf<SensorSample>();

    if (fileBlock.Length % size != 0)
    {
        throw new InvalidDataException("文件块长度不是记录大小的整数倍。");
    }

    return MemoryMarshal.Cast<byte, SensorSample>(fileBlock);
}
```

这段代码没有把文件块复制成 `SensorSample[]`，而是直接拿到一个结构体视图。

遍历时就可以这样写：

```csharp
ReadOnlySpan<SensorSample> samples = ReadSamples(fileBlock);

foreach (ref readonly SensorSample sample in samples)
{
    // 处理 sample
}
```

这种写法适合内部格式、临时文件、缓存文件。

如果文件格式需要长期兼容，字段级解析会更稳。结构体一旦加字段、改字段类型、调整 `Pack`，直接 `Cast` 的格式就可能不兼容。

#### 示例 4：分块计算校验值

有时一段字节数据可以按更大的整数单位批量处理。

例如一个简单的 16 位累加校验：

```csharp
public static uint SumUInt16(ReadOnlySpan<byte> data)
{
    ReadOnlySpan<ushort> words = MemoryMarshal.Cast<byte, ushort>(
        data.Slice(0, data.Length / 2 * 2));

    uint sum = 0;

    foreach (ushort word in words)
    {
        sum += word;
    }

    if ((data.Length & 1) != 0)
    {
        sum += data[^1];
    }

    return sum;
}
```

这里通过 `Cast<byte, ushort>`，把偶数字节部分按 `ushort` 批量处理。

不过这类校验通常也会涉及字节序定义。

如果校验算法规定了网络字节序，仍然要按算法要求显式处理，而不能默认使用本机内存表示。

#### 示例 5：和 `ArrayPool<byte>` 配合减少临时分配

在高频构造二进制消息时，常见做法是从 `ArrayPool<byte>` 租一个缓冲区，然后在这块缓冲区上写数据。

```csharp
using System.Buffers;
using System.Buffers.Binary;

public static void SendMessage(int id, ReadOnlySpan<byte> payload)
{
    const int HeaderSize = 8;
    int totalLength = HeaderSize + payload.Length;

    byte[] buffer = ArrayPool<byte>.Shared.Rent(totalLength);

    try
    {
        Span<byte> message = buffer.AsSpan(0, totalLength);

        BinaryPrimitives.WriteInt32BigEndian(message, payload.Length);
        BinaryPrimitives.WriteInt32BigEndian(message.Slice(4), id);
        payload.CopyTo(message.Slice(HeaderSize));

        SendToTransport(message);
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

这段代码的重点是：

* 只使用 `buffer.AsSpan(0, totalLength)` 这段有效区域；
* 写完后把有效区域交给发送逻辑；
* 不管发送是否成功，都在 `finally` 里归还数组。

如果缓冲区里可能包含敏感数据，归还时可以考虑清理：

```csharp
ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
```

如果头部是本地内部结构体，也可以这样写：

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct LocalHeader
{
    public int Length;
    public int Id;
}

public static void WriteLocalHeader(Span<byte> destination, LocalHeader header)
{
    if (destination.Length < Marshal.SizeOf<LocalHeader>())
    {
        throw new ArgumentException("目标缓冲区长度不够。", nameof(destination));
    }

    MemoryMarshal.Write(destination, in header);
}
```

两种写法的选择点还是一样：

* 明确协议格式，优先 `BinaryPrimitives`；
* 明确本地布局，才考虑 `MemoryMarshal.Write`。

### 和 `BitConverter`、`BinaryPrimitives` 怎么选？

这几个 API 很容易混在一起。

可以按场景区分。

#### 用 `BitConverter`

适合简单转换：

```csharp
int value = BitConverter.ToInt32(bytes);
```

特点是：

* 用法简单；
* 适合不在热点路径里的普通转换；
* 字节序跟当前机器相关。

#### 用 `BinaryPrimitives`

适合协议、文件格式、网络字节序：

```csharp
int value = BinaryPrimitives.ReadInt32BigEndian(bytes);
```

特点是：

* 明确指定大小端；
* 代码语义清楚；
* 很适合字段级解析。

#### 用 `MemoryMarshal`

适合把一段内存整体按某个结构体或类型视图处理：

```csharp
MessageHeader header = MemoryMarshal.Read<MessageHeader>(bytes);
```

特点是：

* 少复制；
* 更底层；
* 强依赖布局和字节序；
* 更适合高频、批量、明确布局的场景。

简单说：

* 单个字段解析，优先 `BinaryPrimitives`；
* 整块结构体重解释，再考虑 `MemoryMarshal`；
* 不在性能热点，普通写法更稳。

### 它和 `unsafe` 是什么关系？

`MemoryMarshal` 不要求代码块写 `unsafe`。

但它提供的很多能力，本质上已经很接近 `unsafe`：

* 直接拿 `ref`；
* 重解释内存；
* 从引用创建一段连续视图；
* 把字节当成结构体。

区别在于：

* `Span<T>` 仍然保留长度信息；
* 很多访问仍然有边界检查；
* 编译器和运行时仍然能做一部分保护。

所以它不是“完全安全的魔法工具”，更像：

> 在不直接写指针的前提下，开放一部分底层内存能力。

这也是它的使用边界。

### 常见坑 1：把重解释当成类型转换

`MemoryMarshal.Cast<byte, int>` 不是把每个 `byte` 转成一个 `int`。

它不是这样：

```text
{ 1, 2, 3, 4 } -> { 1, 2, 3, 4 }
```

而是这样：

```text
{ 1, 0, 0, 0 } -> 一个 int：1
```

它看的是同一段内存，只是换了目标类型的大小和布局。

所以数据长度、类型大小、字节序都要对得上。

### 常见坑 2：忽略字节序

本机是小端序时：

```csharp
byte[] data = { 1, 0, 0, 0 };
int value = MemoryMarshal.Read<int>(data);
```

读出来通常是：

```text
1
```

但如果协议规定的是大端序，这么读就是错的。

协议解析不要靠“当前机器刚好这样存”。

更稳的写法是：

```csharp
int value = BinaryPrimitives.ReadInt32BigEndian(data);
```

### 常见坑 3：结构体里放引用类型字段

`MemoryMarshal.Read<T>`、`Write<T>`、`AsBytes<T>` 这类方法并不适合随便拿任意结构体来用。

下面这种结构体就不适合：

```csharp
public struct UserInfo
{
    public int Id;
    public string Name;
}
```

`string` 是引用类型。

它的内存里放的不是字符串内容本身，而是对象引用。

把这种结构体直接写成字节，不会得到一个可跨进程、可落盘、可传输的稳定格式。

适合 `MemoryMarshal` 直接读写的结构体，通常只包含：

* 基础数值类型；
* 其他同样明确布局的结构体；
* 固定大小、布局稳定的数据。

### 常见坑 4：以为零拷贝一定更好

零拷贝不是免费午餐。

它减少的是分配和复制，但会增加：

* 对布局的依赖；
* 对字节序的依赖；
* 对调用方输入合法性的要求；
* 代码理解成本。

很多业务代码根本不在热点路径。

这种情况下，清楚、稳定、容易维护的写法更划算。

`MemoryMarshal` 更适合出现在这些地方：

* 明确的性能热点；
* 二进制协议解析；
* 序列化框架；
* 网络库；
* 文件格式处理；
* 已经用 `Span<T>` / `Memory<T>` 组织起来的底层缓冲区代码。

### 常见坑 5：长期保存底层引用

`GetReference` 拿到的是 `ref`，不是一个可以长期保存的稳定地址。

托管对象仍然可能被 `GC` 移动。

如果需要把地址交给原生代码长期使用，就要进入固定内存、`GCHandle`、`fixed`、`MemoryHandle` 这些话题。

`MemoryMarshal` 本身不等于“固定内存”。

这点很容易混淆。

### 一张图看懂 `MemoryMarshal` 的位置

可以把它放在这条链路里理解：

```mermaid
flowchart LR
    A["原始数据"] --> B["Span / Memory 连续内存视图"]
    B --> C["MemoryMarshal 重解释 / 读写 / 获取引用"]
    C --> D["结构体 / 字节视图 / ref / ArraySegment"]
```

普通代码多数时候停在 `Span<T>` / `Memory<T>` 就够了。

`MemoryMarshal` 是继续往下一层走，处理“怎么按底层内存表示理解这段数据”。

### 总结

`MemoryMarshal` 的核心不是“神奇转换”，而是：

> 在不复制数据的前提下，把同一段内存换一种方式读取、写入或暴露出来。

它最常用的几类能力是：

* `Cast`：把一段内存按另一种元素类型看；
* `AsBytes`：把结构体内存看成字节；
* `Read` / `Write`：从字节缓冲区读写结构体；
* `TryGetArray`：从 `Memory<T>` 里尽量拿到底层数组；
* `GetReference` / `CreateSpan`：做更底层的 `ref` 级操作。

真正需要记住的是边界：

* 它不负责处理协议字节序；
* 它不负责保证结构体布局一定符合外部数据；
* 它不适合带引用类型字段的结构体直接落盘或传输；
* 它不等于固定内存；
* 它也不是普通业务代码的默认选择。

`MemoryMarshal` 更像一把薄刀。

适合在明确知道内存布局、明确知道数据来源、明确需要减少复制的地方使用。

如果只是为了“看起来高性能”而使用它，通常不值得。
