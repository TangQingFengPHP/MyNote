### 简介

`IEnumerable<T>` 是 `.NET` 中最核心的接口之一，位于 `System.Collections.Generic` 命名空间中。它代表一个可枚举的集合，支持在集合上进行迭代操作。

#### `IEnumerable<T>` 是什么？

```csharp
public interface IEnumerable<out T> : IEnumerable
{
    IEnumerator<T> GetEnumerator();
}
```

* 它定义了一个可枚举对象的契约；

* 任何实现了 `IEnumerable<T>` 的类型都能被 `foreach` 循环遍历；

* 泛型版 `IEnumerable<T>` 是非泛型 `IEnumerable` 的类型安全扩展；

* 它的核心方法只有一个：`GetEnumerator()`。

### IEnumerable 与 IEnumerator 的关系

要理解 `IEnumerable<T>`，必须知道它依赖另一个接口：

```csharp
public interface IEnumerator<out T> : IDisposable, IEnumerator
{
    T Current { get; }
    bool MoveNext();
    void Reset(); // 通常不实现
}
```

关系示意图：

```scss
IEnumerable<T>
    └── GetEnumerator()
          └── 返回 IEnumerator<T>
                  ├── MoveNext() → 是否还有元素
                  ├── Current → 当前元素
                  └── Reset() → 重置（可选）
```

### 基本用法

```csharp
using System;
using System.Collections.Generic;

class Program
{
    static void Main()
    {
        List<string> fruits = new List<string> { "Apple", "Banana", "Cherry" };
        
        // 使用 foreach 遍历（推荐）
        foreach (string fruit in fruits)
        {
            Console.WriteLine(fruit);
        }
        
        // 等价的手动迭代方式
        IEnumerator<string> enumerator = fruits.GetEnumerator();
        try
        {
            while (enumerator.MoveNext())
            {
                string fruit = enumerator.Current;
                Console.WriteLine(fruit);
            }
        }
        finally
        {
            enumerator?.Dispose();
        }
    }
}
```

### 手写一个 `IEnumerable<T>` 示例

```csharp
public class NumberCollection : IEnumerable<int>
{
    private readonly int[] _numbers = { 1, 2, 3, 4, 5 };

    public IEnumerator<int> GetEnumerator()
    {
        foreach (var n in _numbers)
            yield return n;
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

使用：

```csharp
var numbers = new NumberCollection();
foreach (var n in numbers)
{
    Console.WriteLine(n);
}
```

输出：

```
1
2
3
4
5
```

### yield return 的魔法

等价于下面的展开版：

```csharp
public IEnumerator<int> GetEnumerator()
{
    return new Enumerator();
}

private class Enumerator : IEnumerator<int>
{
    private int _index = -1;
    private readonly int[] _numbers = { 1, 2, 3, 4, 5 };

    public int Current => _numbers[_index];
    object IEnumerator.Current => Current;

    public bool MoveNext()
    {
        _index++;
        return _index < _numbers.Length;
    }

    public void Reset() => _index = -1;
    public void Dispose() { }
}
```

`yield` 编译后自动生成状态机类，就是这种效果。

这也是 `LINQ` 延迟执行的基础机制。

### foreach 的底层原理

当写：

```csharp
foreach (var item in collection)
{
    Console.WriteLine(item);
}
```

编译器实际上生成：

```csharp
using (var enumerator = collection.GetEnumerator())
{
    while (enumerator.MoveNext())
    {
        var item = enumerator.Current;
        Console.WriteLine(item);
    }
}
```

### 延迟执行（Lazy Evaluation）

`LINQ` 查询（`Where`, `Select` 等）通常返回 `IEnumerable<T>`。
这些操作是延迟执行的：只有在遍历时才真正运行。

```csharp
var numbers = new[] { 1, 2, 3, 4, 5 };

var query = numbers.Where(n => n > 2).Select(n => n * 10);

Console.WriteLine("Query created.");
foreach (var n in query)
{
    Console.WriteLine(n);
}
```

`Where / Select` 只是定义查询，不会立即执行。
直到 `foreach` 时，才会真正迭代并执行逻辑。

### 常见实现 `IEnumerable<T>` 的类型

| 类型                      | 说明                             |
| ------------------------- | -------------------------------- |
| `List<T>`                 | 基于数组实现的集合               |
| `T[]`                     | 数组                             |
| `Dictionary<TKey,TValue>` | 键值对集合（枚举键值对）         |
| `HashSet<T>`              | 不重复元素集合                   |
| `Queue<T>` / `Stack<T>`   | 队列与栈                         |
| `string`                  | 实现了非泛型 `IEnumerable<char>` |
| `LINQ` 查询结果           | 延迟执行序列                     |

### 手写一个支持过滤的 `IEnumerable<T>`

```csharp
public class FilteredCollection<T> : IEnumerable<T>
{
    private readonly IEnumerable<T> _source;
    private readonly Func<T, bool> _predicate;

    public FilteredCollection(IEnumerable<T> source, Func<T, bool> predicate)
    {
        _source = source;
        _predicate = predicate;
    }

    public IEnumerator<T> GetEnumerator()
    {
        foreach (var item in _source)
        {
            if (_predicate(item))
                yield return item;
        }
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

使用：

```csharp
var list = new List<int> { 1, 2, 3, 4, 5 };
var filtered = new FilteredCollection<int>(list, x => x % 2 == 0);

foreach (var n in filtered)
    Console.WriteLine(n);
```

### `IEnumerable<T>` vs `IQueryable<T>`

| 特性       | IEnumerable                        | IQueryable                 |
| ---------- | ---------------------------------- | -------------------------- |
| 执行时机   | 本地内存中                         | 可翻译为远程查询（如 SQL） |
| 适用场景   | 内存集合                           | 数据库 ORM（EF Core 等）   |
| 表达式类型 | 委托（Func）                       | 表达式树（Expression）     |
| 可延迟执行 | ✅                                  | ✅                          |
| 例子       | `List<T>`, `Array`, `yield return` | `DbSet<T>`                 |

示例：

```csharp
// IEnumerable：在内存中过滤
var result1 = list.Where(x => x > 10);

// IQueryable：生成 SQL 查询
var result2 = db.Users.Where(x => x.Age > 10);
```

### IEnumerable 的扩展方法分类（LINQ 常用）

| 分类 | 示例方法                                 |
| ---- | ---------------------------------------- |
| 过滤 | `Where`, `Distinct`, `Skip`, `Take`      |
| 投影 | `Select`, `SelectMany`                   |
| 聚合 | `Count`, `Sum`, `Average`, `Aggregate`   |
| 元素 | `First`, `Last`, `Single`, `ElementAt`   |
| 组合 | `Concat`, `Union`, `Intersect`, `Except` |
| 排序 | `OrderBy`, `ThenBy`, `Reverse`           |
| 转换 | `ToList`, `ToArray`, `ToDictionary`      |

### 高级特性

#### 自定义 LINQ 扩展方法

```csharp
public static class MyLinqExtensions
{
    // 自定义 Where 方法
    public static IEnumerable<T> Where<T>(
        this IEnumerable<T> source, 
        Func<T, bool> predicate)
    {
        foreach (T item in source)
        {
            if (predicate(item))
            {
                yield return item;
            }
        }
    }
    
    // 自定义 Select 方法
    public static IEnumerable<TResult> Select<TSource, TResult>(
        this IEnumerable<TSource> source,
        Func<TSource, TResult> selector)
    {
        foreach (TSource item in source)
        {
            yield return selector(item);
        }
    }
    
    // 自定义扩展方法
    public static IEnumerable<T> SkipEveryOther<T>(this IEnumerable<T> source)
    {
        bool take = true;
        foreach (T item in source)
        {
            if (take)
            {
                yield return item;
            }
            take = !take;
        }
    }
    
    // 带索引的扩展方法
    public static IEnumerable<TResult> SelectWithIndex<TSource, TResult>(
        this IEnumerable<TSource> source,
        Func<TSource, int, TResult> selector)
    {
        int index = 0;
        foreach (TSource item in source)
        {
            yield return selector(item, index);
            index++;
        }
    }
}

// 使用自定义扩展方法
var numbers = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8 };
var result = numbers.SkipEveryOther(); // 返回 1, 3, 5, 7

var indexed = numbers.SelectWithIndex((num, idx) => $"Index {idx}: {num}");
```

#### 无限序列

```csharp
public static class InfiniteSequences
{
    // 无限数字序列
    public static IEnumerable<int> InfiniteNumbers()
    {
        int i = 0;
        while (true)
        {
            yield return i++;
        }
    }
    
    // 斐波那契数列
    public static IEnumerable<long> Fibonacci()
    {
        long a = 0, b = 1;
        while (true)
        {
            yield return a;
            long temp = a;
            a = b;
            b = temp + b;
        }
    }
    
    // 随机数序列
    public static IEnumerable<int> RandomNumbers(int min, int max)
    {
        Random rnd = new Random();
        while (true)
        {
            yield return rnd.Next(min, max);
        }
    }
}

// 使用无限序列（一定要结合 Take 等方法使用）
var firstTenFibonacci = Fibonacci().Take(10);
var randomNumbers = RandomNumbers(1, 100).Take(5);
```

#### 数据分页

```csharp
public static class PagingExtensions
{
    public static IEnumerable<IEnumerable<T>> Page<T>(this IEnumerable<T> source, int pageSize)
    {
        var page = new List<T>(pageSize);
        foreach (T item in source)
        {
            page.Add(item);
            if (page.Count == pageSize)
            {
                yield return page;
                page = new List<T>(pageSize);
            }
        }
        
        if (page.Count > 0)
        {
            yield return page;
        }
    }
}

// 使用分页
var bigCollection = Enumerable.Range(1, 1000);
foreach (var page in bigCollection.Page(100))
{
    Console.WriteLine($"Page with {page.Count()} items");
    // 处理当前页
}
```