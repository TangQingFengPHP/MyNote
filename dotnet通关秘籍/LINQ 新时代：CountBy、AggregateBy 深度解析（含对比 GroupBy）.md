### 简介

在 `.NET 8` 之前，`LINQ` 没有内置 `CountBy` 和 `AggregateBy` 方法，但在 `.NET 9（C# 13）` 中，`LINQ` 正式引入了这两个新扩展方法，极大简化了数据分组和聚合的写法。

#### 背景

传统的分组统计一般使用 `GroupBy`：

```csharp
var query = list.GroupBy(x => x.Category)
                .Select(g => new { Category = g.Key, Count = g.Count() });
```

但 `GroupBy`：

* 代码冗长

* 对简单的计数/聚合任务过于复杂

为此，`.NET 9` 引入：

* `CountBy` → 按键快速计数

* `AggregateBy` → 按键快速聚合

#### 什么是 CountBy 和 AggregateBy？

* `CountBy`：用于按键（`key`）对集合进行分组并统计每个键的出现次数，返回一个键值对集合，其中键是分组依据，值是该键的计数。

* `AggregateBy`：用于按键对集合进行分组并对每个分组应用自定义聚合函数，返回一个键值对集合，其中键是分组依据，值是聚合结果。

这两个方法类似于 `GroupBy` 后接 `Count` 或 `Aggregate`，但它们更高效、更简洁，减少了中间分组对象的创建，优化了性能。

关键特点：

* 高效性：直接生成键值对结果，避免 `GroupBy` 创建中间 `IGrouping` 对象的开销。

* 简洁性：将分组和统计/聚合合并为一步操作。

* 灵活性：支持自定义键选择器和聚合逻辑。

* 返回类型：返回 `IEnumerable<KeyValuePair<TKey, TValue>>`，便于进一步处理。

### CountBy

作用：按键分组并统计数量。
类似 `GroupBy(...).Select(...g.Count())` 的简化版。

方法签名

```csharp
public static IEnumerable<KeyValuePair<TKey, int>> CountBy<TSource, TKey>(
    this IEnumerable<TSource> source,
    Func<TSource, TKey> keySelector,
    IEqualityComparer<TKey>? comparer = null)
```

* `source`：输入的集合（实现 `IEnumerable<TSource>`）。

* `keySelector`：一个函数，从每个元素提取分组的键。

* `comparer`：可选的键比较器，用于自定义键的相等性判断（默认使用 `EqualityComparer<TKey>.Default`）。

* 返回：`IEnumerable<KeyValuePair<TKey, int>>`，每个键值对包含键和该键的计数。

#### 基础用法

```csharp
var fruits = new[] { "apple", "banana", "apple", "orange", "banana", "apple" };

var result = fruits.CountBy(f => f);

foreach (var kv in result)
{
    Console.WriteLine($"{kv.Key}: {kv.Value}");
}
```

输出：

```
apple: 3
banana: 2
orange: 1
```

等价于：

```csharp
fruits.GroupBy(f => f).Select(g => new KeyValuePair<string,int>(g.Key, g.Count()));
```

* `fruit => fruit` 按水果名称分组并计数。

* 结果是一个键值对集合，键是水果名称，值是出现次数。

#### 自定义键

```csharp
var numbers = new[] { 1, 2, 3, 4, 5, 6 };
var result = numbers.CountBy(n => n % 2 == 0 ? "Even" : "Odd");
```

输出：

```
Odd: 3
Even: 3
```

#### 使用比较器：忽略大小写

```csharp
var names = new[] { "apple", "Apple", "APPLE", "banana" };
var counts = names.CountBy(name => name, StringComparer.OrdinalIgnoreCase);

foreach (var kvp in counts)
{
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
}
// 输出:
// apple: 3
// banana: 1
```

* `StringComparer.OrdinalIgnoreCase` 忽略键的大小写。

### AggregateBy

作用：按键分组并在分组中执行自定义聚合逻辑（不仅仅是计数）。
类似 `GroupBy(...).Aggregate(...)` 的简化版。

方法签名

```csharp
public static IEnumerable<KeyValuePair<TKey, TResult>> AggregateBy<TSource, TKey, TAccumulate, TResult>(
    this IEnumerable<TSource> source,
    Func<TSource, TKey> keySelector,
    TAccumulate seed,
    Func<TAccumulate, TSource, TAccumulate> func,
    Func<TKey, TAccumulate, TResult> resultSelector,
    IEqualityComparer<TKey>? comparer = null)
```

参数说明：

* `source`：输入的集合（实现 `IEnumerable<TSource>`）。

* `keySelector`：从每个元素提取分组的键。

* `seed`：聚合的初始值（每个分组从此值开始）。

* `func`：聚合函数，定义如何将元素累加到当前累积值。

* `resultSelector`：结果选择器，将键和最终累积值转换为结果。

* `comparer`：可选的键比较器。

* 返回：`IEnumerable<KeyValuePair<TKey, TResult>>`，每个键值对包含键和聚合结果。

#### 求和

```csharp
var orders = new[]
{
    new { Category = "Book", Price = 10 },
    new { Category = "Book", Price = 20 },
    new { Category = "Food", Price = 5 },
    new { Category = "Food", Price = 7 },
};

var result = orders.AggregateBy(
    keySelector: o => o.Category,
    seed: 0m,
    accumulator: (sum, item) => sum + item.Price
);

foreach (var kv in result)
{
    Console.WriteLine($"{kv.Key}: {kv.Value}");
}
```

输出：

```
Book: 30
Food: 12
```

等价于：

```csharp
orders.GroupBy(o => o.Category)
      .Select(g => new KeyValuePair<string,decimal>(g.Key, g.Sum(x => x.Price)));
```

#### 拼接字符串

```csharp
var names = new[]
{
    new { Group = "A", Name = "Alice" },
    new { Group = "A", Name = "Alex" },
    new { Group = "B", Name = "Bob" },
};

var result = names.AggregateBy(
    keySelector: n => n.Group,
    seed: "",
    accumulator: (s, n) => s == "" ? n.Name : $"{s}, {n.Name}"
);

foreach (var kv in result)
{
    Console.WriteLine($"{kv.Key}: {kv.Value}");
}
```

输出：

```csharp
A: Alice, Alex
B: Bob
```

#### 使用自定义结果：统计最大值

```csharp
var items = new[]
{
    new { Category = "A", Value = 10 },
    new { Category = "B", Value = 20 },
    new { Category = "A", Value = 15 }
};

var maxValues = items.AggregateBy(
    item => item.Category,          // 按类别分组
    seed: int.MinValue,             // 初始值为最小整数
    (max, item) => Math.Max(max, item.Value), // 取最大值
    (key, max) => max);             // 返回最大值

foreach (var kvp in maxValues)
{
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
}
// 输出:
// A: 15
// B: 20
```

### CountBy vs AggregateBy

| 特性     | CountBy                               | AggregateBy                                   |
| -------- | ------------------------------------- | --------------------------------------------- |
| 功能     | 仅计数                                | 自定义任何聚合操作                            |
| 返回类型 | `IEnumerable<KeyValuePair<TKey,int>>` | `IEnumerable<KeyValuePair<TKey,TAccumulate>>` |
| 复杂度   | 更简洁                                | 更灵活，但需提供 seed 和 accumulator          |
| 适用场景 | 频率统计                              | 求和、平均值、拼接字符串、自定义聚合等        |

### 性能优势

与 `GroupBy` 相比：

* `CountBy / AggregateBy` 只执行一次遍历

* 内部使用 哈希表累积，减少对象创建

* 对大数据集统计效率更高

### 实战示例：日志统计

```csharp
record Log(string Level, int Size);

var logs = new[]
{
    new Log("Info", 10),
    new Log("Error", 5),
    new Log("Info", 20),
    new Log("Error", 15),
    new Log("Warning", 7)
};

// 统计不同 Level 的日志数量
var count = logs.CountBy(l => l.Level);

// 统计不同 Level 的总 Size
var size = logs.AggregateBy(l => l.Level, 0, (sum, log) => sum + log.Size);
```

输出：

```
---Count---
Info: 2
Error: 2
Warning: 1

---Size---
Info: 30
Error: 20
Warning: 7
```

#### 统计单词频率并排序

```csharp
var text = "the quick brown fox jumps over the lazy dog the quick fox";
var words = text.Split(' ');
var wordCounts = words.CountBy(word => word, StringComparer.OrdinalIgnoreCase)
                      .OrderByDescending(kvp => kvp.Value);

foreach (var kvp in wordCounts)
{
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
}
// 输出:
// the: 3
// quick: 2
// fox: 2
// brown: 1
// jumps: 1
// over: 1
// lazy: 1
// dog: 1
```

#### 复杂聚合（构建对象）

```csharp
var orders = new[]
{
    new { Customer = "Alice", Amount = 100, Item = "Laptop" },
    new { Customer = "Bob", Amount = 50, Item = "Mouse" },
    new { Customer = "Alice", Amount = 200, Item = "Phone" }
};

var summaries = orders.AggregateBy(
    order => order.Customer,
    seed: new { Total = 0, Items = new List<string>() },
    (acc, order) => new { Total = acc.Total + order.Amount, Items = acc.Items.Append(order.Item).ToList() },
    (key, acc) => new { Customer = key, acc.Total, acc.Items });

foreach (var summary in summaries)
{
    Console.WriteLine($"{summary.Customer}: Total = {summary.Total}, Items = {string.Join(", ", summary.Items)}");
}
// 输出:
// Alice: Total = 300, Items = Laptop, Phone
// Bob: Total = 50, Items = Mouse
```

### 适用场景

#### CountBy

* 统计频率：统计集合中元素的出现次数（如单词计数、类别统计）。

* 分组分析：快速生成键值对形式的计数结果，适合数据分析。

* 替代 `GroupBy + Count`：在需要简单计数时，`CountBy` 更高效。

#### AggregateBy

* 分组聚合：对分组数据执行求和、最大值、最小值、平均值等操作。

* 复杂聚合：如连接字符串、构建复杂对象等。

* 高性能场景：需要高效处理大集合，避免中间分组对象的开销。

### 总结

| 方法          | 用途                     | 替代旧写法                                   | 场景示例                 |
| ------------- | ------------------------ | -------------------------------------------- | ------------------------ |
| `CountBy`     | 按键分组计数             | `GroupBy().Select(g => g.Count())`           | 商品销量、用户角色人数   |
| `AggregateBy` | 按键分组并执行自定义聚合 | `GroupBy().Aggregate()` 或 `GroupBy().Sum()` | 日志大小总和、字符串拼接 |

### 注意事项：

#### 版本要求：

* `CountBy` 和 `AggregateBy` 是 `C# 13（.NET 9）`的新特性，需目标框架为 `.NET 9.0` 或更高。

* 在较低版本中，可使用 `GroupBy + Count` 或 `Aggregate` 替代，但性能稍差。

#### 性能优势：

* 两者直接生成键值对，避免 `GroupBy` 的中间 `IGrouping` 对象，减少内存分配。

* 对于大集合或高频操作，性能提升显著。

#### 键比较器：

* 默认使用 `EqualityComparer<TKey>.Default`，适合大多数场景。

* 对于自定义类型或特殊相等性逻辑，需提供 `IEqualityComparer<TKey>`。

#### 不可变性：

* 返回的 `IEnumerable<KeyValuePair<TKey, TValue>>` 是延迟求值的。

* 如果需要持久化结果，调用 `ToList()` 或 `ToDictionary()`。

#### 错误处理：

* 如果 `source` 为 `null`，会抛出 `ArgumentNullException`。

* 如果 `keySelector` 或 `func` 抛出异常，需在调用代码中处理。

#### 与 GroupBy 的选择：

* 如果需要访问分组中的所有元素，使用 `GroupBy`。

* 如果只需要键和聚合结果（如计数、总和），优先使用 `CountBy` 或 `AggregateBy`。