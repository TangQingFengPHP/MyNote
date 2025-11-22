### 简介

集合表达式（`Collection Expressions`）是 `C# 12.0`（随 `.NET 8.0` 发布于 2023 年）引入的一项新特性，用于以简洁、声明式的方式创建和初始化集合（如数组、列表、字典等）。集合表达式通过 [...] 语法提供了一种更直观的方式来定义集合，减少样板代码并提高可读性。

### 背景和作用

集合表达式旨在解决传统集合初始化（如 `new List<T> { ... }` 或 `new[] { ... }`）的冗长和不一致问题，灵感来源于 `Python` 和 `JavaScript` 的列表推导式。它在以下场景中特别有用：

* 数据初始化：快速定义数组、列表或字典。

* 批量操作：简化 `MySqlBulkLoader` 的数据表填充。

* `API` 输入/输出：处理 `JSON` 数据或响应集合。

* 缓存管理：初始化 `MemoryCache` 的键值对。

* 定时任务：结合 `Cronos` 处理批量数据。

### 核心概念

#### 传统初始化 vs 集合表达式

```csharp
// 传统方式  
List<int> list1 = new List<int> { 1, 2, 3 };  
int[] array1 = new int[] { 1, 2, 3 };  

// C# 12 集合表达式  
List<int> list2 = [1, 2, 3];  
int[] array2 = [1, 2, 3];  
Span<int> span = [1, 2, 3];  
```

#### 语法结构

```csharp
// 基础形式  
[element1, element2, ..., elementN]  

// 空集合  
var empty = [];  

// 嵌套集合  
int[][] matrix = [[1, 2], [3, 4], [5, 6]];  
```

### 支持的所有集合类型

| 集合类型          | 示例                                | 编译器转换机制                                 |
| ----------------- | ----------------------------------- | ---------------------------------------------- |
| 数组              | int[] arr = [1, 2, 3];              | 直接优化为 new int[] { ... }                   |
| List<T>           | List<int> list = [1, 2, 3];         | 调用 List<T>.CollectionsMarshalHelper.Create() |
| Span<T>           | Span<int> span = [1, 2, 3];         | 栈分配或缓冲区重用                             |
| ReadOnlySpan<T>   | ReadOnlySpan<char> ros = ['a','b']; | 同 Span<T>                                     |
| ImmutableArray<T> | ImmutableArray<int> ia = [1,2,3];   | 调用 ImmutableArray.Create()                   |
| 自定义集合        | 实现 IEnumerable<T> + Add 方法      | 编译器生成 Add 调用序列                        |

> ✅ 所有实现 IEnumerable<T> 且有 Add 方法的类型都支持

### 主要功能

#### 简洁语法：

* 使用 `[...]` 替代 `new List<T> { ... }` 或 `new[] { ... }`。

* 支持多种集合类型（数组、`List<T>`、`Span<T>`、字典等）。

#### 扩展运算符：

* 使用 `..` 展开现有集合（如合并数组或列表）。

#### 类型推断：

* 自动推断目标集合类型，减少显式类型声明。

#### 多场景支持：

* 初始化集合、传递参数、返回结果。

* 支持 `LINQ`、异步操作和并发场景。

#### 与现有集合兼容：

* 无缝转换到 `IEnumerable<T>`、`ICollection<T>` 等。

### 核心语法

集合表达式使用 `[...]` 语法，支持以下形式：

* 基本初始化：

```csharp
int[] numbers = [1, 2, 3]; // 等价于 new[] { 1, 2, 3 }
List<string> names = ["Alice", "Bob"]; // 等价于 new List<string> { "Alice", "Bob" }
```

* 扩展运算符（..）：

```csharp
int[] moreNumbers = [1, 2, ..numbers, 4]; // 展开 numbers，生成 [1, 2, 1, 2, 3, 4]
```

* 空集合：

```csharp
int[] empty = []; // 等价于 Array.Empty<int>()
```

* 嵌套集合：

```csharp
int[][] jagged = [[1, 2], [3, 4]]; // 锯齿状数组
```

* 字典初始化：

```csharp
Dictionary<string, int> scores = ["Alice": 90, "Bob": 85]; // 等价于 new Dictionary<string, int> { ["Alice"] = 90, ["Bob"] = 85 }
```

* 目标类型：

    * 支持 `IEnumerable<T>、ICollection<T>、List<T>、数组、Span<T>` 等。

    * 编译器根据上下文推断类型：

    ```csharp
    IEnumerable<int> enumerable = [1, 2, 3]; // 推断为数组
    ```

* 自定义集合支持

```csharp
public class CustomCollection<T> : IEnumerable<T>  
{  
    private readonly List<T> _items = new();  

    public void Add(T item) => _items.Add(item);  

    public IEnumerator<T> GetEnumerator() => _items.GetEnumerator();  
}  

// 使用集合表达式初始化  
CustomCollection<string> tags = ["dotnet", "csharp"];  
```