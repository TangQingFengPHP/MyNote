# 别把 ImmutableArray 当成只读 List：C#.NET 不可变数组深入详解

很多代码会把集合暴露成 `IReadOnlyList<T>`：

```csharp
public IReadOnlyList<string> Names => _names;
```

这样可以限制调用方不能直接 `Add`，但它只是一个只读视图。只要内部的 `List<string>` 还在变化，外部枚举到的内容就可能变化。

`ImmutableArray<T>` 解决的是另一件事：创建完成后，数组中的元素和数组长度都不能通过这个集合实例改变。需要“修改”时，返回一个新的数组，旧数组继续保持原样。

它保留了数组最重要的优点：连续存储、按索引访问快、遍历开销低；同时提供不可变集合的快照语义，适合配置快照、规则表、路由表、编译器结果、插件列表等读多写少场景。

本文从创建、读取和修改开始，逐步说明 `Builder`、`default` 状态、线程安全、原子发布、性能取舍，以及什么时候该选择 `List<T>`、`ReadOnlyCollection<T>` 或 `ImmutableList<T>`。

## ImmutableArray<T> 是什么？

命名空间：

```csharp
System.Collections.Immutable
```

`ImmutableArray<T>` 是一个不可变数组结构。它本身是 `struct`，内部持有一个数组引用；集合实例创建后，不能通过 `Add`、索引赋值等方式改变原内容。

普通数组可以修改：

```csharp
int[] numbers = { 1, 2, 3 };
numbers[0] = 100;

Console.WriteLine(numbers[0]); // 100
```

`ImmutableArray<T>` 没有可写索引器：

```csharp
using System.Collections.Immutable;

ImmutableArray<int> numbers = ImmutableArray.Create(1, 2, 3);

// numbers[0] = 100; // 编译错误
```

如果确实需要替换第一个元素，使用 `SetItem`：

```csharp
ImmutableArray<int> replaced = numbers.SetItem(0, 100);

Console.WriteLine(string.Join(", ", numbers));  // 1, 2, 3
Console.WriteLine(string.Join(", ", replaced)); // 100, 2, 3
```

关键点是：`SetItem` 没有修改 `numbers`，而是返回了一个新数组。

## 为什么需要不可变数组？

### 只读接口不等于数据不变

下面的代码隐藏了写操作，但没有冻结数据：

```csharp
private readonly List<string> _names = new() { "Tom", "Jerry" };

public IReadOnlyList<string> Names => _names;

public void AddName(string name)
{
    _names.Add(name);
}
```

`Names` 不能直接调用 `Add`，但 `AddName` 仍然会让已经拿到的 `Names` 看到新元素。若某段逻辑需要一份稳定快照，`IReadOnlyList<T>` 不一定够用。

换成 `ImmutableArray<T>`：

```csharp
private ImmutableArray<string> _names =
    ImmutableArray.Create("Tom", "Jerry");

public ImmutableArray<string> Names => _names;

public void AddName(string name)
{
    _names = _names.Add(name);
}
```

原来的 `Names` 仍然指向旧版本，新读取会拿到新版本。每个版本都是稳定快照。

### 多线程读取更简单

可变集合需要协调读写：

```text
线程 A 正在枚举
线程 B 修改 List<T>
线程 A 可能抛出异常或读到不一致状态
```

不可变集合的读取不需要为了“集合自身会不会被修改”而加锁：

```text
发布一份 ImmutableArray 快照
多个线程并发读取同一快照
生成新版本时不影响旧版本
```

这不是说所有业务状态都自动线程安全。数组元素 `T` 如果本身是可变对象，元素对象的并发安全仍然需要单独处理。

## 安装和命名空间

在旧版 .NET、.NET Standard 或需要显式引用包的项目中安装：

```bash
dotnet add package System.Collections.Immutable
```

代码中引用：

```csharp
using System.Collections.Immutable;
```

现代 .NET 项目通常已经间接包含相关程序集，但明确写入 PackageReference 有助于保持项目依赖清晰。具体是否需要安装，取决于目标框架和当前项目文件。

## 创建 ImmutableArray

### 使用 `Create`

元素数量较少时，工厂方法最直观：

```csharp
ImmutableArray<int> numbers = ImmutableArray.Create(1, 2, 3, 4);

foreach (int number in numbers)
{
    Console.WriteLine(number);
}
```

也可以显式指定类型：

```csharp
ImmutableArray<string> names = ImmutableArray.Create<string>(
    "Tom",
    "Jerry",
    "Spike");
```

### 使用 `Empty`

创建空数组：

```csharp
ImmutableArray<string> empty = ImmutableArray<string>.Empty;
```

`Empty` 是已初始化的空数组，表达“明确存在，只是没有元素”。

### 从数组或集合转换

```csharp
int[] source = { 1, 2, 3 };
ImmutableArray<int> immutable = source.ToImmutableArray();
```

从 `IEnumerable<T>` 转换需要 `System.Linq` 中的扩展方法：

```csharp
using System.Collections.Immutable;

List<string> names = new() { "Tom", "Jerry" };
ImmutableArray<string> immutableNames = names.ToImmutableArray();
```

转换完成后，继续修改 `names` 不会改变 `immutableNames`：

```csharp
names.Add("Spike");

Console.WriteLine(immutableNames.Length); // 2
Console.WriteLine(names.Count);           // 3
```

这是因为转换会建立自己的数组存储，而不是继续暴露原 `List<T>` 的内部数组。

### 使用 `CreateRange`

不依赖 LINQ 扩展时，可以使用：

```csharp
IEnumerable<int> source = Enumerable.Range(1, 5);
ImmutableArray<int> numbers = ImmutableArray.CreateRange(source);
```

### 使用 Builder

动态构建大量元素时，使用 `Builder`：

```csharp
ImmutableArray<int>.Builder builder =
    ImmutableArray.CreateBuilder<int>();

for (int i = 1; i <= 5; i++)
{
    builder.Add(i * 10);
}

ImmutableArray<int> numbers = builder.ToImmutable();
```

指定初始容量可以减少扩容：

```csharp
ImmutableArray<int>.Builder builder =
    ImmutableArray.CreateBuilder<int>(capacity: 1000);
```

`Builder` 是可变的，只有调用 `ToImmutable()` 后才得到不可变数组。因此 Builder 不应该直接跨线程共享，也不应该在构建完成后继续修改它并把它当成已发布快照。

## 读取元素

### 索引访问

```csharp
ImmutableArray<int> numbers = ImmutableArray.Create(10, 20, 30);

int first = numbers[0];
int last = numbers[^1];

Console.WriteLine(first); // 10
Console.WriteLine(last);  // 30
```

索引访问是 O(1)，适合需要频繁按位置读取的场景。

### `Length` 和 `IsEmpty`

```csharp
Console.WriteLine(numbers.Length);
Console.WriteLine(numbers.IsEmpty);
```

`ImmutableArray<T>` 使用 `Length`，与普通数组保持一致。`IsEmpty` 只判断长度是否为零。

### 遍历

```csharp
foreach (int number in numbers)
{
    Console.WriteLine(number);
}
```

`ImmutableArray<T>` 提供自己的 Enumerator，适合直接 `foreach`。也可以作为 `IEnumerable<T>` 传给 LINQ：

```csharp
int total = numbers.Sum();
int max = numbers.Max();
```

如果热路径对分配非常敏感，优先使用直接 `foreach` 和索引循环，避免不必要的接口转换和 LINQ 链。

## “修改”ImmutableArray：返回新版本

不可变集合的修改方法不会改变原实例。

### 添加元素

```csharp
ImmutableArray<int> original = ImmutableArray.Create(1, 2, 3);
ImmutableArray<int> appended = original.Add(4);

Console.WriteLine(string.Join(", ", original));  // 1, 2, 3
Console.WriteLine(string.Join(", ", appended)); // 1, 2, 3, 4
```

添加多个元素：

```csharp
ImmutableArray<int> result = original.AddRange(new[] { 4, 5, 6 });
```

### 替换元素

```csharp
ImmutableArray<int> replaced = original.SetItem(1, 200);

Console.WriteLine(string.Join(", ", original));  // 1, 2, 3
Console.WriteLine(string.Join(", ", replaced)); // 1, 200, 3
```

如果不确定索引是否存在，可以使用 `SetItem` 前先判断：

```csharp
int index = 10;
ImmutableArray<int> result = index < original.Length
    ? original.SetItem(index, 100)
    : original;
```

索引超出范围时，`SetItem` 会抛出 `ArgumentOutOfRangeException`。

### 删除元素

```csharp
ImmutableArray<int> removedByValue = original.Remove(2);
ImmutableArray<int> removedByIndex = original.RemoveAt(0);

Console.WriteLine(string.Join(", ", removedByValue)); // 1, 3
Console.WriteLine(string.Join(", ", removedByIndex)); // 2, 3
```

`Remove` 删除第一个匹配元素；没有找到时会返回内容相同的数组或按当前实现处理，业务代码不应依赖“是否返回同一个实例”这种实现细节。

### 插入和移除范围

```csharp
ImmutableArray<int> inserted = original.Insert(1, 99);
ImmutableArray<int> rangeRemoved = original.RemoveRange(0, 2);

Console.WriteLine(string.Join(", ", inserted));     // 1, 99, 2, 3
Console.WriteLine(string.Join(", ", rangeRemoved)); // 3
```

插入和删除会移动后续元素，因此通常是 O(n) 操作。

### 排序、反转和过滤

```csharp
ImmutableArray<int> numbers = ImmutableArray.Create(3, 1, 2);

ImmutableArray<int> sorted = numbers.Sort();
ImmutableArray<int> reversed = numbers.Reverse();
ImmutableArray<int> filtered = numbers.Where(x => x > 1)
    .ToImmutableArray();
```

这些方法都不会改动 `numbers`。

## Builder：大量写入的正确姿势

下面的写法逻辑上正确，但性能通常很差：

```csharp
ImmutableArray<int> numbers = ImmutableArray<int>.Empty;

for (int i = 0; i < 100_000; i++)
{
    numbers = numbers.Add(i);
}
```

每次 `Add` 都可能创建并复制一个新数组，累计复制量会非常大。

改用 Builder：

```csharp
ImmutableArray<int>.Builder builder =
    ImmutableArray.CreateBuilder<int>(100_000);

for (int i = 0; i < 100_000; i++)
{
    builder.Add(i);
}

ImmutableArray<int> numbers = builder.ToImmutable();
```

Builder 的工作方式更接近 `List<T>`：内部数组可以扩容，元素可以修改；转换成不可变数组时，尽量复用现有存储，避免多一次完整复制。

### 从旧数组变更得到 Builder

```csharp
ImmutableArray<int> original = ImmutableArray.Create(1, 2, 3);

ImmutableArray<int>.Builder builder = original.ToBuilder();
builder.Add(4);
builder[0] = 100;

ImmutableArray<int> updated = builder.ToImmutable();
```

`ToBuilder()` 本身需要把原内容复制到可变 Builder 中，因此适合“后面要连续做多次修改”的场景，不适合只改一个元素却频繁来回转换。

### Builder 不是线程安全集合

```csharp
ImmutableArray<int>.Builder builder =
    ImmutableArray.CreateBuilder<int>();

// 不要让多个线程同时修改同一个 builder
```

Builder 只负责构建阶段。构建结束后调用 `ToImmutable()`，再把结果交给并发读取代码。

## default、Empty 和 IsDefault

`ImmutableArray<T>` 是值类型，因此可以处于“默认值”状态：

```csharp
ImmutableArray<int> defaultArray = default;
ImmutableArray<int> emptyArray = ImmutableArray<int>.Empty;

Console.WriteLine(defaultArray.IsDefault); // True
Console.WriteLine(emptyArray.IsDefault);   // False
Console.WriteLine(defaultArray.IsEmpty);   // True
Console.WriteLine(emptyArray.IsEmpty);     // True
```

两者长度都为零，但语义不同：

| 状态 | 含义 |
| --- | --- |
| `default` | 尚未初始化的数组值 |
| `Empty` | 已明确初始化，但没有元素 |
| `IsDefaultOrEmpty` | 两种空状态都视为空 |

业务 API 通常可以返回 `Empty`，避免调用方额外判断默认状态：

```csharp
public ImmutableArray<string> GetNames()
{
    return ImmutableArray<string>.Empty;
}
```

对于可能来自字段默认值、序列化或可选参数的内部状态，可以使用：

```csharp
if (items.IsDefaultOrEmpty)
{
    Console.WriteLine("没有数据");
}
```

某些 API 在默认状态下调用枚举、索引或扩展方法时可能抛出异常。对外传递前先规范为 `Empty`，通常更稳妥：

```csharp
ImmutableArray<int> normalized = value.IsDefault
    ? ImmutableArray<int>.Empty
    : value;
```

## 浅不可变：元素对象仍然可能变化

`ImmutableArray<T>` 保护的是数组结构和元素引用，不会自动冻结 `T` 对象。

```csharp
sealed class User
{
    public string Name { get; set; } = string.Empty;
}

User user = new() { Name = "Tom" };
ImmutableArray<User> users = ImmutableArray.Create(user);

user.Name = "Jerry";

Console.WriteLine(users[0].Name); // Jerry
```

数组没有被修改，但数组中的 `User` 对象被外部修改了。

需要深层不可变时，应让元素类型本身不可变：

```csharp
public sealed record User(string Name);

ImmutableArray<User> users = ImmutableArray.Create(new User("Tom"));
users = users.SetItem(0, users[0] with { Name = "Jerry" });
```

也可以使用只读属性、`record`、不可变 DTO 或其他不可变对象模型。集合不可变和元素不可变是两层不同的保证。

## 线程安全：读取安全不等于更新自动安全

### 多线程读取

一份已经构建好的数组可以被多个线程读取：

```csharp
ImmutableArray<string> names = ImmutableArray.Create(
    "Tom",
    "Jerry",
    "Spike");

Parallel.For(0, 100, _ =>
{
    foreach (string name in names)
    {
        _ = name.Length;
    }
});
```

读取过程中没有线程会改变 `names` 的内容，因此不需要为集合本身加锁。

### 使用 ImmutableInterlocked 发布新版本

如果一个字段会被多个线程读取，同时由后台线程整体替换，需要使用适合不可变集合的原子操作：

```csharp
using System.Collections.Immutable;

public sealed class RulesSnapshot
{
    private ImmutableArray<string> _rules =
        ImmutableArray<string>.Empty;

    public ImmutableArray<string> Current => _rules;

    public void Replace(IEnumerable<string> rules)
    {
        ImmutableArray<string> next = rules.ToImmutableArray();
        ImmutableInterlocked.InterlockedExchange(
            ref _rules,
            next);
    }
}
```

读取时先保存局部快照：

```csharp
public bool Contains(string rule)
{
    ImmutableArray<string> snapshot = _rules;
    return snapshot.Contains(rule, StringComparer.OrdinalIgnoreCase);
}
```

一次方法调用会基于同一份数组快照完成，不会读到“半更新”的集合内容。

需要注意：

* `ImmutableArray<T>` 保证的是集合内容不可变。
* `ImmutableInterlocked` 负责多个线程之间发布新版本时的原子交换。
* 元素对象 `T` 若可变，仍然需要自己的线程安全策略。

### 用锁也可以，但语义不同

以下方式同样可以保护字段：

```csharp
private readonly object _gate = new();
private ImmutableArray<string> _rules =
    ImmutableArray<string>.Empty;

public void Replace(IEnumerable<string> rules)
{
    ImmutableArray<string> next = rules.ToImmutableArray();

    lock (_gate)
    {
        _rules = next;
    }
}
```

如果只是替换一个不可变快照，`ImmutableInterlocked` 更贴合这个模型；如果更新还涉及多个字段、外部资源或复杂事务，锁或其他同步方案仍然有价值。

## 实战 Demo：线程安全的配置快照

下面实现一个简单的配置仓库：

* 配置读取不加锁。
* 后台更新时先构建完整数组。
* 完整数组构建完成后一次性发布。
* 每次读取拿到稳定快照。

### 配置模型

```csharp
public sealed record AppSetting(
    string Key,
    string Value);
```

使用 `record` 让配置项本身具备值语义和不可变属性。

### 配置仓库

```csharp
using System.Collections.Immutable;

public sealed class ConfigurationSnapshot
{
    private ImmutableArray<AppSetting> _settings =
        ImmutableArray<AppSetting>.Empty;

    public void Replace(IEnumerable<AppSetting> settings)
    {
        ImmutableArray<AppSetting> next =
            settings.ToImmutableArray();

        ImmutableInterlocked.InterlockedExchange(
            ref _settings,
            next);
    }

    public bool TryGetValue(
        string key,
        out string? value)
    {
        ImmutableArray<AppSetting> snapshot = _settings;

        foreach (AppSetting setting in snapshot)
        {
            if (string.Equals(
                setting.Key,
                key,
                StringComparison.OrdinalIgnoreCase))
            {
                value = setting.Value;
                return true;
            }
        }

        value = null;
        return false;
    }

    public ImmutableArray<AppSetting> GetAll()
    {
        return _settings;
    }
}
```

### 运行示例

```csharp
using System.Collections.Immutable;

var configuration = new ConfigurationSnapshot();

configuration.Replace(new[]
{
    new AppSetting("ConnectionString", "Server=db01"),
    new AppSetting("LogLevel", "Information")
});

if (configuration.TryGetValue("LogLevel", out string? level))
{
    Console.WriteLine(level); // Information
}

ImmutableArray<AppSetting> snapshot = configuration.GetAll();

configuration.Replace(new[]
{
    new AppSetting("ConnectionString", "Server=db02"),
    new AppSetting("LogLevel", "Debug")
});

Console.WriteLine(snapshot[0].Value); // Server=db01
Console.WriteLine(configuration.GetAll()[0].Value); // Server=db02
```

旧快照仍然可以继续使用，新配置不会影响正在执行的读取逻辑。这种模式很适合定时加载配置、刷新路由、替换规则表和热更新插件清单。

## ImmutableArray 与 List 的性能取舍

`ImmutableArray<T>` 不是“所有场景都更快”的 `List<T>` 替代品。

| 操作 | `ImmutableArray<T>` | `List<T>` |
| --- | --- | --- |
| 按索引读取 | O(1) | O(1) |
| 遍历 | 快，连续数组 | 快，连续数组 |
| 末尾添加一次 | 通常 O(n)，生成新数组 | 摊销 O(1) |
| 中间插入/删除 | O(n) | O(n) |
| 多次连续构建 | 使用 Builder | 直接 Add |
| 多线程读取 | 快照安全 | 修改时需同步 |
| 内存结构 | 数组 + 值类型包装 | List + 内部数组 |

重点不在于“不可变集合没有成本”，而在于把修改成本集中到版本生成阶段，换取读取阶段的稳定性和更简单的并发模型。

### 什么时候适合 ImmutableArray？

适合：

* 初始化后长期读取的静态数据。
* 配置、规则、路由和权限快照。
* 编译器、分析器、代码生成器中的结果集合。
* 多线程读、低频整体替换。
* 需要快速索引访问的不可变数据。
* API 返回值需要明确表达“调用方不能修改”。

### 什么时候应该使用 List？

适合：

* 单线程内部临时构建。
* 高频增加、删除、排序和修改。
* 集合生命周期只存在于一个方法内部。
* 不需要跨线程共享稳定快照。

常见组合是：内部用 `List<T>` 构建，完成后转换成 `ImmutableArray<T>` 对外发布：

```csharp
List<string> buffer = new();
buffer.Add("A");
buffer.Add("B");

ImmutableArray<string> published = buffer.ToImmutableArray();
```

## ImmutableArray 与 ReadOnlyCollection 的区别

`ReadOnlyCollection<T>` 是对现有 `IList<T>` 的只读包装：

```csharp
List<int> list = new() { 1, 2, 3 };
ReadOnlyCollection<int> view = list.AsReadOnly();

list.Add(4);
Console.WriteLine(view.Count); // 4
```

它适合“限制当前引用的写操作”，但不会创建独立快照。

`ImmutableArray<T>` 则把内容固定下来：

```csharp
List<int> list = new() { 1, 2, 3 };
ImmutableArray<int> snapshot = list.ToImmutableArray();

list.Add(4);
Console.WriteLine(snapshot.Length); // 3
```

选择依据：

```text
需要只读视图，原集合变化也应该可见 → ReadOnlyCollection<T>
需要独立稳定快照 → ImmutableArray<T>
```

## ImmutableArray 与 ImmutableList 的区别

两者都不可变，但底层结构和修改成本不同：

| 特性 | `ImmutableArray<T>` | `ImmutableList<T>` |
| --- | --- | --- |
| 底层结构 | 连续数组 | 树形结构 |
| 索引读取 | O(1) | 通常 O(log n) |
| 添加/删除 | 通常需要复制数组，O(n) | 结构共享，通常 O(log n) |
| 遍历局部性 | 好 | 相对复杂 |
| 适合场景 | 读多写少、频繁索引 | 需要多个版本并频繁修改 |

如果数据主要是读取和遍历，`ImmutableArray<T>` 往往更合适；如果每次修改都要生成新版本，且集合较大、修改频率高，可以评估 `ImmutableList<T>`。

## API 设计：返回 ImmutableArray

### 对外返回不可变结果

```csharp
public sealed class PluginCatalog
{
    private readonly ImmutableArray<string> _plugins;

    public PluginCatalog(IEnumerable<string> plugins)
    {
        _plugins = plugins.ToImmutableArray();
    }

    public ImmutableArray<string> GetPlugins()
    {
        return _plugins;
    }
}
```

调用方可以遍历、索引和 LINQ 查询，但不能直接修改集合结构。

### 不要把内部 Builder 暴露出去

```csharp
public ImmutableArray<int>.Builder GetBuilder()
{
    return _builder;
}
```

这种 API 会把可变构建状态泄露给外部，破坏封装和线程安全。应返回 `ImmutableArray<T>`，或者返回只读接口：

```csharp
public ImmutableArray<int> GetItems()
{
    return _builder.ToImmutable();
}
```

### 参数接收

如果方法需要稳定快照，可以直接接收 `ImmutableArray<T>`：

```csharp
public void RegisterRules(ImmutableArray<string> rules)
{
    _rules = rules;
}
```

如果方法只需要枚举，不关心集合具体类型，使用 `IEnumerable<T>` 更灵活；如果需要避免调用方在执行过程中改变数据，入口处可以转换一次：

```csharp
public void RegisterRules(IEnumerable<string> rules)
{
    _rules = rules.ToImmutableArray();
}
```

## 常见误区

### 误区一：ImmutableArray 中的对象也自动不可变

它只保证数组结构不变，不能自动冻结引用类型元素。元素类需要使用 `record`、只读属性或其他不可变设计。

### 误区二：每次 Add 都很便宜

单次 `Add` 使用简单，但在循环中不断 `array = array.Add(item)` 会反复分配和复制。大量写入使用 Builder。

### 误区三：IReadOnlyList 就等于 ImmutableArray

`IReadOnlyList<T>` 只是接口能力限制，不保证底层集合不变化。需要快照时使用 `ToImmutableArray()`。

### 误区四：ImmutableArray 的字段赋值就自动完成并发发布

不可变内容和版本发布是两个问题。并发整体替换字段时，使用 `ImmutableInterlocked` 或锁，明确同步语义。

### 误区五：default 和 Empty 完全一样

两者都没有元素，但 `default` 表示未初始化状态，`Empty` 表示已初始化的空集合。公共 API 更推荐返回 `Empty`。

### 误区六：所有线程安全问题都消失了

数组本身不会被修改，但元素对象、外部资源、多个字段之间的组合状态仍可能需要同步。

## 一个简单的基准测试思路

比较集合性能时，不能只测一次 `Add`。应该分开测读取和构建：

```csharp
using System.Collections.Immutable;
using System.Diagnostics;

const int count = 100_000;

Stopwatch stopwatch = Stopwatch.StartNew();
ImmutableArray<int>.Builder builder =
    ImmutableArray.CreateBuilder<int>(count);

for (int i = 0; i < count; i++)
{
    builder.Add(i);
}

ImmutableArray<int> immutable = builder.ToImmutable();
stopwatch.Stop();

Console.WriteLine($"Builder 构建：{stopwatch.ElapsedMilliseconds} ms");

stopwatch.Restart();
long total = 0;
for (int i = 0; i < immutable.Length; i++)
{
    total += immutable[i];
}

stopwatch.Stop();
Console.WriteLine($"索引读取：{stopwatch.ElapsedMilliseconds} ms");
Console.WriteLine(total);
```

正式性能结论应使用 BenchmarkDotNet，并结合真实数据量、元素类型、读写比例和内存分配结果判断。不要因为某个集合在单项测试中更快，就直接替换所有业务集合。

## 总结

`ImmutableArray<T>` 可以理解为“拥有数组读取性能的不可变快照”：

```text
创建完成
   ↓
内容和长度固定
   ↓
多个线程安全读取
   ↓
修改时生成新版本
```

使用时记住几个关键原则：

* 少量固定数据使用 `ImmutableArray.Create`。
* 大量动态构建使用 `ImmutableArray<T>.Builder`。
* 修改操作返回新数组，旧数组不会改变。
* `Empty` 和 `default` 都是空，但语义不同。
* 数组不可变不代表元素对象不可变。
* 并发整体替换使用 `ImmutableInterlocked` 或锁。
* 高频写入优先考虑 `List<T>`，需要不可变版本时再转换。

真正适合 `ImmutableArray<T>` 的场景，是“读取远多于写入，并且读取方需要稳定快照”。如果核心需求是频繁增删，`List<T>` 或 `ImmutableList<T>` 往往更合适。

参考资料：

* [Microsoft Learn：ImmutableArray<T> 结构](https://learn.microsoft.com/en-us/dotnet/api/system.collections.immutable.immutablearray-1)
* [Microsoft Learn：System.Collections.Immutable 命名空间](https://learn.microsoft.com/en-us/dotnet/api/system.collections.immutable)
* [Microsoft Learn：ImmutableArray<T>.Builder](https://learn.microsoft.com/en-us/dotnet/api/system.collections.immutable.immutablearray-1.builder)
* [Microsoft Learn：ImmutableArray<T>.ToBuilder](https://learn.microsoft.com/en-us/dotnet/api/system.collections.immutable.immutablearray-1.tobuilder)
* [Microsoft Learn：ImmutableInterlocked](https://learn.microsoft.com/en-us/dotnet/api/system.collections.immutable.immutableinterlocked)
