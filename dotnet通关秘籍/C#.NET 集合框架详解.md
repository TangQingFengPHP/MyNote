### 简介

`C#` 集合框架是处理数据集合的核心组件，位于 `System.Collections` 和 `System.Collections.Generic` 命名空间。它提供了多种数据结构来高效存储和操作数据。

### 集合框架概览

```swift
System.Collections               (非泛型老版)
└─ System.Collections.Generic    (泛型现代版)
   ├─ IList<T> ← List<T>, LinkedList<T>, ObservableCollection<T>, …
   ├─ ICollection<T> ← HashSet<T>, SortedSet<T>, …
   ├─ IDictionary<TKey,TValue> ← Dictionary<TKey,TValue>, SortedDictionary<,>, …
   └─ IEnumerable<T>
System.Collections.Concurrent    (并发安全集合)
System.Collections.Immutable     (不可变集合)
System.Collections.ObjectModel   (通知模型集合)
```

* 非泛型：`ArrayList、Hashtable`（弃用，不推荐，新代码使用泛型集合）

* 泛型：类型安全、无装箱，性能更优

### 核心接口

| 接口                                | 功能                                           |
| ----------------------------------- | ---------------------------------------------- |
| `IEnumerable<T>`                    | 可枚举，支持 `foreach`                         |
| `ICollection<T>` : `IEnumerable<T>` | 增删改查，`Count`、`Add`、`Remove`、`Contains` |
| `IList<T>` : `ICollection<T>`       | 支持按索引访问，`Insert`、`IndexOf`            |
| `IDictionary<TKey,TValue>`          | 键/值对，`this[key]`、`TryGetValue`            |
| `IReadOnlyCollection<T>`            | 只读视图                                       |
| `IReadOnlyList<T>`                  | 只读索引                                       |
| `IReadOnlyDictionary<,>`            | 只读字典                                       |

### 主要实现与特性

#### 顺序访问集合

| 类型                        | 特点                                                      | 复杂度（N=Count）                    |
| --------------------------- | --------------------------------------------------------- | ------------------------------------ |
| **Array**                   | 固定大小，按索引快速访问                                  | 读/写 O(1)                           |
| **List<T>**                 | 可变大小的数组；增长时可能触发内部扩容                    | 读 O(1)，插尾 O(1) 摊销；插中间 O(N) |
| **LinkedList<T>**           | 双向链表，插入/删除任意节点 O(1)（已知节点）；按索引 O(N) | 访问 O(N)，插/删节点 O(1)            |
| **ObservableCollection<T>** | 在 UI/MVVM 中使用，修改时触发事件                         | 同 `List<T>`                         |

#### 键/值对集合

| 类型                               | 特点                                   | 复杂度                             |
| ---------------------------------- | -------------------------------------- | ---------------------------------- |
| **Dictionary\<TKey,TValue>**       | 哈希表，平均 O(1) 查找/添加/删除       | 查找/添加/删除 O(1)                |
| **SortedDictionary\<TKey,TValue>** | 基于红黑树，键有序，O(log N) 操作      | 查找/添加/删除 O(log N)            |
| **SortedList\<TKey,TValue>**       | 底层数组，按索引访问快，插入/删除 O(N) | 访问 O(log N) 二分查找；插/删 O(N) |

#### 集合语义的集合

| 类型                      | 特点                              | 复杂度                  |
| ------------------------- | --------------------------------- | ----------------------- |
| **HashSet<T>**            | 无序、不重复元素，基于哈希        | 查找/添加/删除 O(1)     |
| **SortedSet<T>**          | 有序、不重复，基于红黑树          | 查找/添加/删除 O(log N) |
| **Queue<T>**              | 先进先出，基于循环数组            | 入队/出队 O(1)          |
| **Stack<T>**              | 后进先出，基于数组                | 压栈/弹栈 O(1)          |
| **BlockingCollection<T>** | 有界/无界并发队列，支持阻塞和取消 | 入/出队 O(1)            |

### 并发集合（System.Collections.Concurrent）

| 类型                        | 特点                                    | 场景              |
| --------------------------- | --------------------------------------- | ----------------- |
| **ConcurrentDictionary<,>** | 线程安全的字典，分段锁或自旋锁          | 高并发读写字典    |
| **ConcurrentQueue<T>**      | 多生产者/多消费者队列                   | 并发任务调度      |
| **ConcurrentStack<T>**      | 并发栈                                  | 并发回退/撤销操作 |
| **ConcurrentBag<T>**        | 无序的线程本地存储集合                  | 并发任务结果收集  |
| **BlockingCollection<T>**   | 封装多个并发集合，支持生产者/消费者模式 | 流水线计算        |

### 不可变集合（System.Collections.Immutable）

* `ImmutableList<T>, ImmutableDictionary<,>, ImmutableHashSet<T>` 等

* 特点：每次修改返回新版本，原版本不变；线程安全，适合多读少写场景

```csharp
var list1 = ImmutableList<int>.Empty;
var list2 = list1.Add(1);
```

### 性能选型

* 频繁随机访问 → `List<T>` 或 数组

* 频繁插中间 → `LinkedList<T>`（谨慎评估 `GC`/内存）

* 键/值查找 → `Dictionary<,>`；需要排序 → `SortedDictionary<,>`

* 唯一性 → `HashSet<T>`；需要有序 → `SortedSet<T>`

* 生产者—消费者 → `BlockingCollection<T>`（可指定底层并发队列）

* `UI` 数据绑定 → `ObservableCollection<T>`

* 高并发场景 → `System.Collections.Concurrent` 中的集合

* 按需 严格不可变 → `Immutable*`

### 常见陷阱

* `List<T>.Capacity vs Count：Capacity` 控制内部数组长度，`Add` 超出时会重新 `Array.Copy`；可通过构造函数预设 `Capacity` 避免多次扩容。

* `Dictionary` 键的 `GetHashCode`：自定义类型做键时要重写 `GetHashCode/Equals`，避免哈希冲突。

* `LinkedList<T>` 内存碎片：每个节点都是单独对象，`GC` 压力大时慎用。

* 并发集合的内存占用：`ConcurrentDictionary` 分段锁结构会消耗更多内存。

* 不可变集合：写操作开销高，适合读多写少。

### 常用集合详解

#### `List<T>` - 动态数组

* 特点：自动扩容、索引访问、支持快速随机访问

* 最佳场景：需要索引访问的集合，元素数量变化频繁

* 性能：

    * 添加：平均 O(1)，最坏 O(n)（扩容时）

    * 插入/删除：O(n)

    * 访问：O(1)

```csharp
// 创建与初始化
var numbers = new List<int> { 1, 2, 3 };

// 添加元素
numbers.Add(4);
numbers.AddRange(new[] { 5, 6 });

// 插入元素
numbers.Insert(0, 0); // 开头插入

// 删除元素
numbers.Remove(3);    // 按值删除
numbers.RemoveAt(0);  // 按索引删除

// 容量优化（减少内存占用）
numbers.TrimExcess();

// 预分配容量（提升性能）
var largeList = new List<int>(capacity: 1000);
```

#### `Dictionary<K,V>` - 哈希字典

* 特点：键值对存储、快速查找、键唯一

* 最佳场景：按键快速查找/更新

* 性能：

    * 查找/添加/删除：平均 O(1)，最坏 O(n)（哈希冲突时）

```csharp
var users = new Dictionary<int, string>
{
    [1] = "Alice",
    [2] = "Bob"
};

// 安全访问
if (users.TryGetValue(2, out string name))
{
    Console.WriteLine(name); // 输出 "Bob"
}

// 添加或更新
users[3] = "Charlie"; // 添加
users[2] = "Robert";  // 更新

// 遍历
foreach (var kvp in users)
{
    Console.WriteLine($"ID: {kvp.Key}, Name: {kvp.Value}");
}

// 自定义键类型（需实现GetHashCode和Equals）
public record UserId(int Id);
var userDict = new Dictionary<UserId, string>();
```

#### `HashSet<T>` - 唯一值集合

* 特点：元素唯一、快速存在性检查、集合运算

* 最佳场景：去重操作、集合运算（并集/交集等）

* 性能：添加/查找/删除 O(1)

```csharp
var set1 = new HashSet<int> { 1, 2, 3, 4 };
var set2 = new HashSet<int> { 3, 4, 5, 6 };

// 集合运算
set1.UnionWith(set2);      // 并集: {1,2,3,4,5,6}
set1.IntersectWith(set2);   // 交集: {3,4}
set1.ExceptWith(set2);      // 差集: {1,2}

// 存在性检查
bool contains = set1.Contains(3); // true
```

#### `SortedSet<T>` - 有序唯一集合

* 特点：元素按升序排列（需实现 `IComparable<T>`），插入 / 删除 / 查找 O (log n)。

* 核心方法：同 `HashSet<T>`，支持 `Min、Max` 属性。

```csharp
SortedSet<int> sortedSet = new SortedSet<int> { 3, 1, 2 };
// sortedSet 元素顺序：1, 2, 3

int min = sortedSet.Min;  // 最小元素：1
int max = sortedSet.Max;  // 最大元素：3
```

#### `Queue<T>` - 先进先出队列

* 特点：`FIFO`（先进先出）处理

* 最佳场景：任务调度、消息处理

```csharp
var queue = new Queue<string>();
queue.Enqueue("First");
queue.Enqueue("Second");
queue.Enqueue("Third");

while (queue.Count > 0)
{
    string item = queue.Dequeue();
    Console.WriteLine(item); // 输出 First → Second → Third
}
```

#### `Stack<T>` - 后进先出栈

* 特点：`LIFO`（后进先出）处理

* 最佳场景：撤销操作、递归算法

```csharp
var stack = new Stack<int>();
stack.Push(1);
stack.Push(2);
stack.Push(3);

while (stack.Count > 0)
{
    int top = stack.Pop();
    Console.WriteLine(top); // 输出 3 → 2 → 1
}
```

#### `LinkedList<T>` - 双向链表

* 特点：高效插入/删除、顺序访问

* 最佳场景：频繁在中间插入/删除元素

```csharp
var list = new LinkedList<string>();
var node1 = list.AddFirst("First");
var node3 = list.AddLast("Third");
var node2 = list.AddAfter(node1, "Second");

list.Remove(node2);  // 高效删除中间节点

// 顺序遍历
var current = list.First;
while (current != null)
{
    Console.WriteLine(current.Value);
    current = current.Next;
}
```

#### `SortedDictionary<K,V>` 与 `SortedList<K,V>`

* 特点：按键排序的字典

* 区别：

    * `SortedDictionary`：基于红黑树，插入删除更快 (O(log n))

    * `SortedList`：基于数组，内存更紧凑，随机访问更快

### 并发集合（System.Collections.Concurrent）

#### `ConcurrentDictionary<K,V>`

```csharp
var concurrentDict = new ConcurrentDictionary<int, string>();
concurrentDict.TryAdd(1, "One");
concurrentDict.AddOrUpdate(1, k => "New", (k, v) => "Updated");
```

#### `ConcurrentQueue<T>`

```csharp
var queue = new ConcurrentQueue<int>();
queue.Enqueue(1);
if (queue.TryDequeue(out int result)) { ... }
```

#### `ConcurrentBag<T>`

无序集合，适用于生产者-消费者场景

```csharp
var bag = new ConcurrentBag<int>();
bag.Add(42);
if (bag.TryTake(out int item)) { ... }
```

#### `BlockingCollection` - 生产者消费者模型

```csharp
BlockingCollection<int> buffer = new BlockingCollection<int>(boundedCapacity: 5);

// 生产者
Task.Run(() => {
    for (int i = 0; i < 10; i++) {
        buffer.Add(i);
        Thread.Sleep(100);
    }
    buffer.CompleteAdding();
});

// 消费者
foreach (int item in buffer.GetConsumingEnumerable()) {
    Console.WriteLine(item);
}
```

### 不可变集合（System.Collections.Immutable）

```csharp
// 创建不可变集合
var immutableList = ImmutableList.Create(1, 2, 3);

// 所有"修改"操作返回新集合
var newList = immutableList.Add(4).Remove(2);

// 原集合保持不变
Console.WriteLine(immutableList.Count); // 输出 3
Console.WriteLine(newList.Count);       // 输出 3 (添加1个移除1个)
```

#### 内存安全集合

性能特点：

* 结构共享：修改操作复用大部分现有内存

* 线程绝对安全

* 适合配置对象、历史记录等场景

```csharp
var builder = ImmutableArray.CreateBuilder<int>();
builder.Add(1);
builder.Add(2);
ImmutableArray<int> immutable = builder.ToImmutable();

// 修改操作返回新实例
ImmutableArray<int> newArray = immutable.Add(3);
```

### 性能优化技巧

#### 预分配容量：对已知大小的集合初始化时指定容量

```csharp
var list = new List<int>(1000); // 避免多次扩容
```

#### 避免装箱拆箱：优先使用泛型集合而非非泛型集合

```csharp
// 推荐
List<int> intList = new();

// 避免
ArrayList oldList = new(); // 导致装箱
```

#### 批量操作：使用 AddRange() 替代循环中的单个 Add()

```csharp
// 高效方式
list.AddRange(items);

// 低效方式
foreach (var item in items) list.Add(item);
```

#### 选择合适相等比较器：自定义字典键时实现 IEquatable<T> 和重写 GetHashCode()

```csharp
public class CustomKey : IEquatable<CustomKey>
{
    public int Id { get; set; }

    public bool Equals(CustomKey other) => Id == other?.Id;
    
    public override int GetHashCode() => Id.GetHashCode();
}
```

#### 枚举优化：避免在热路径中创建枚举器

```csharp
// 低效
for (int i = 0; i < list.Count; i++) { ... }

// 高效
int count = list.Count;
for (int i = 0; i < count; i++) { ... }
```

#### 接口返回只读视图

```csharp
public IReadOnlyList<string> GetItems()
{
    return _internalList.AsReadOnly();
}
```

#### 使用集合初始化器：简洁的初始化语法

```csharp
var dict = new Dictionary<int, string>
{
    [1] = "One",
    [2] = "Two"
};
```

#### LINQ结合使用：利用LINQ进行复杂查询

```csharp
var recentOrders = orders
    .Where(o => o.Date > DateTime.Now.AddDays(-7))
    .OrderByDescending(o => o.Total)
    .ToList();
```

#### 避免修改遍历中的集合：使用 ToList() 创建副本

```csharp
foreach (var item in collection.ToList())
{
    if (condition) collection.Remove(item);
}
```

### 实践场景

#### 使用 Try 方法

使用 `TryGetValue、TryAdd` 提高效率

```csharp
if (dict.TryGetValue(key, out var value))
{
    // 使用 value
}
```

#### 相等性实现

```csharp
public class Person : IEquatable<Person>
{
    public int Id { get; set; }
    
    public bool Equals(Person? other) => other?.Id == Id;
    
    public override int GetHashCode() => Id.GetHashCode();
}

// 自定义比较器
class CaseInsensitiveComparer : IEqualityComparer<string>
{
    public bool Equals(string? x, string? y) 
        => string.Equals(x, y, StringComparison.OrdinalIgnoreCase);
    
    public int GetHashCode(string obj) 
        => obj.ToUpperInvariant().GetHashCode();
}
```

#### 枚举安全

```csharp
// 错误：在枚举中修改集合
foreach (var item in list) {
    if (item.ShouldRemove) 
        list.Remove(item); // 抛出InvalidOperationException
}

// 正确：使用临时集合
List<Item> toRemove = list.Where(item => item.ShouldRemove).ToList();
foreach (var item in toRemove) {
    list.Remove(item);
}
```

#### 值类型集合优化

```csharp
// 避免装箱
List<Point> points = new List<Point>(); // 结构体集合
points.Add(new Point(1, 2));

// 优先考虑Span<T>处理内存连续数据
Span<int> span = CollectionsMarshal.AsSpan(numbers);
```

#### LRU缓存实现

```csharp
public class LRUCache<TKey, TValue>
{
    private readonly Dictionary<TKey, LinkedListNode<CacheItem>> _dict;
    private readonly LinkedList<CacheItem> _list;
    private readonly int _capacity;

    public LRUCache(int capacity)
    {
        _capacity = capacity;
        _dict = new Dictionary<TKey, LinkedListNode<CacheItem>>(capacity);
        _list = new LinkedList<CacheItem>();
    }

    public TValue Get(TKey key)
    {
        if (_dict.TryGetValue(key, out var node))
        {
            _list.Remove(node);
            _list.AddFirst(node);
            return node.Value.Value;
        }
        return default;
    }

    public void Add(TKey key, TValue value)
    {
        if (_dict.Count >= _capacity)
        {
            var last = _list.Last;
            _dict.Remove(last.Value.Key);
            _list.RemoveLast();
        }

        var newNode = new LinkedListNode<CacheItem>(
            new CacheItem { Key = key, Value = value });
        
        _list.AddFirst(newNode);
        _dict.Add(key, newNode);
    }

    private class CacheItem
    {
        public TKey Key { get; init; }
        public TValue Value { get; init; }
    }
}
```

#### 数据批处理

```csharp
const int BatchSize = 100;
List<Data> batch = new List<Data>(BatchSize);

foreach (var item in massiveCollection)
{
    batch.Add(item);
    
    if (batch.Count == BatchSize)
    {
        ProcessBatch(batch);
        batch.Clear();
    }
}

ProcessBatch(batch); // 处理剩余项
```