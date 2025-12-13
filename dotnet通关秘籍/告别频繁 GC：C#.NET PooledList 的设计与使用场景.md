### 简介

`PooledList<T>` 是 高性能集合类型，由 `Collections.Pooled` 提供，用于替代 `List<T>`，通过 对象池 (`ArrayPool<T>`) 复用内部数组来减少 `GC`（垃圾回收）压力。

 ⚡ 核心目标：
在需要频繁创建/销毁 `List<T>` 的场景下，`PooledList<T>` 通过数组租借与归还的机制避免频繁分配内存，从而提升性能并降低 `GC` 负担。

### 安装

```shell
dotnet add package Collections.Pooled --version 1.0.82
```

添加命名空间

```csharp
using Collections.Pooled;
```

### 特点

* 数组池化：内部数组从 `ArrayPool<T>.Shared`（默认）或自定义池中租借，减少分配。

* `Span<T>` 支持：提供 `Span` 属性，直接访问内部数组的填充部分，支持零拷贝操作。

* `IDisposable` 实现：调用 `Dispose()` 时，返回内部数组到池中（不调用 `Dispose` 不会出错，但会降低池化效果）。

* 扩展方法：如 `TryFind`、`TryFindLast`（替换标准 `Find`、`FindLast`，返回 `bool` 以避免异常）。

* 添加/插入 `Span`：`AddSpan` 和 `InsertSpan` 方法返回一个 `Span<T>`，允许直接写入内部存储，而无需多次调用 `Add`。

* 构造函数选项：

    * 支持指定自定义 `ArrayPool<T>`。

    * `clearMode` 参数控制数组返回池时是否清除内容（默认自动）。

    * `sizeToCapacity` 参数使初始 `Count == Capacity`，适合值类型避免不必要的零初始化。

* `ToPooledList()` 扩展：从 `IEnumerable<T>` 快速创建 `PooledList<T>`。

* 性能提升：在高频操作中，减少 `GC` 触发，尤其适合循环中创建临时列表的场景。

### 内部原理

#### 普通 `List<T>` 内部

* `List<T>` 内部维护一个 `T[]` 数组。

* 当容量不足时会 申请更大数组 并 拷贝数据。

* 对象销毁后，这些数组最终交给 `GC` 回收。

#### `PooledList<T>` 内部

* 内部数组不是直接 `new` 出来的，而是从 `ArrayPool<T>.Shared` 租借。

* 使用结束时通过 `Dispose()` 方法 归还数组，供下次复用。

* 避免频繁分配和回收大数组，降低 `GC Gen2` 压力。

### 基本用法

#### 创建与释放

```csharp
using Microsoft.Toolkit.HighPerformance.Buffers;

using var list = new PooledList<int>(); // 自动使用 ArrayPool<int>
list.Add(1);
list.Add(2);
list.Add(3);

foreach (var item in list)
{
    Console.WriteLine(item);
}
// Dispose() 会自动归还数组
```

> 💡 推荐使用 using 确保 Dispose() 被调用，否则不会归还数组，造成内存浪费。

#### 初始容量

```csharp
using var list = new PooledList<int>(initialCapacity: 1024);
```

#### 转换为 `Span<T>` / `Memory<T>`

`PooledList<T>` 的优势之一是可以直接获取底层内存：

```csharp
Span<int> span = list.AsSpan();
Memory<int> memory = list.AsMemory();
```

这样可以高效地与 `Span/Memory API` 交互，避免额外拷贝。

### 常用操作

与 `List<T>` 基本一致：

```csharp
list.Add(10);
list.AddRange(new[] { 20, 30, 40 });
list.Insert(1, 15);
list.RemoveAt(0);
list.Clear();

Console.WriteLine(list.Count);
Console.WriteLine(list.Capacity);
```

### 与 `List<T>` 对比

| 特性                 | `List<T>`                     | `PooledList<T>`                              |
| -------------------- | ----------------------------- | -------------------------------------------- |
| 内存分配             | 每次扩容 `new` 新数组         | 从 `ArrayPool<T>` 租借，复用数组             |
| GC 压力              | 大量频繁创建/销毁时 GC 压力大 | 减少 GC Gen2 压力                            |
| 释放方式             | 依赖 GC                       | 必须 `Dispose()` 归还数组                    |
| 性能（频繁操作场景） | 可能产生大量堆分配            | 高性能、低分配                               |
| Span/Memory 支持     | 需要 `AsSpan()` 扩展          | 内置 `AsSpan`、`AsMemory`，零拷贝访问        |
| 适用场景             | 通用集合                      | 高性能、临时性数据集合（网络、序列化、算法） |

### 高级用法

#### 与 `ArrayPool<T>` 配合

```csharp
using var list = new PooledList<byte>(ArrayPool<byte>.Shared, 2048);
```

可以传入自定义池，比如为特殊场景优化的 `ArrayPool<T>`。

#### 与 `Span<T>` 高效处理

适合序列化/反序列化：

```csharp
Span<byte> buffer = list.AsSpan();
ProcessBuffer(buffer); // 直接操作底层数组，无需复制
```

#### 搜索和扩展

```csharp
var list = new PooledList<string> { "apple", "banana", "cherry" };
bool found = list.TryFind(s => s.StartsWith("b"), out string result);
Console.WriteLine(found ? result : "Not found"); // 输出: banana

var pooledFromEnumerable = Enumerable.Range(1, 5).ToPooledList(); // 扩展方法
Console.WriteLine(string.Join(", ", pooledFromEnumerable)); // 输出: 1, 2, 3, 4, 5

pooledFromEnumerable.Dispose();
```

* `TryFind` 和 `TryFindLast`：返回 `bool` 和 `out` 值，避免 `null` 检查。

### 注意事项与最佳实践

#### 必须调用 Dispose()

* 否则不会归还数组，导致内存泄漏。

* 推荐 `using` 块。

#### 不要长期持有 Span/Memory

* 因为数组归还池后可能被其他线程复用。

#### 适用场景

* 高频率、大数据临时集合。

* 网络协议解析、日志聚合、临时缓存。

* 需要 `Span` 访问的场景。

#### 不适合场景

* 长期持有的全局集合。

* 数据量小且生命周期长的普通集合。