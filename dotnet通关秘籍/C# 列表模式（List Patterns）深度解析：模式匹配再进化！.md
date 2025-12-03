### 简介

列表模式是一种模式匹配机制，允许检查一个集合（例如数组、`List<T>`、或任何实现了 `IEnumerable<T>` 的类型）的元素数量、顺序以及每个元素的内容。

在 `C# 10` 之前，模式匹配 (`Pattern Matching`) 已支持 `switch` 表达式、类型模式、属性模式等，但对列表或序列的匹配还不够直观。
`C# 11` 引入 列表模式（`List Patterns`），让开发者可以：

* 按顺序匹配数组、`Span<T>`、`IList<T>` 等列表。

* 检查长度、特定元素，甚至解构其中的值。

* 结合 `switch、is、when` 等模式匹配写出简洁、声明式的代码。

> ✅ 列表模式是模式匹配的扩展，不是新的集合类型。

### 语法

列表模式使用方括号 `[]` 来定义要匹配的模式：

基本形式：

```csharp
[ element-pattern-1, element-pattern-2, ... ]
```

```csharp
// 检查列表是否恰好包含三个元素：1, 2, 3
if (list is [1, 2, 3])
{
    Console.WriteLine("列表包含精确序列 1, 2, 3");
}
```

元素模式可以是：

* 常量（1、"abc"）

* 变量（var x）

* 类型模式（int x）

* 通配符（_）

* 切片模式（..，C# 11 引入）

* 嵌套模式（{}、[] 等）

可以用于：

* `is` 表达式

* `switch` 表达式或语句

* `when` 条件

### 匹配规则

* 匹配顺序严格一致。

* 元素数量需满足模式要求（可用切片模式放宽）。

* 支持 `IEnumerable`、数组、`Span<T>`、`IList<T>` 等实现 `Length/Count` 属性 + 索引器 的类型。

### 基础示例

#### 精确匹配

```csharp
int[] numbers = { 1, 2, 3 };

if (numbers is [1, 2, 3])
    Console.WriteLine("Exact match!");
```

> 只有当数组是 `{1,2,3}` 且长度为 3 时才匹配。

#### 通配符 _

```csharp
if (numbers is [1, _, 3])
    Console.WriteLine("Middle element can be anything");
```

> 忽略第二个元素，只要求首尾固定。

#### 切片模式 ..

* 匹配以 1 开头的任意长度数组。

```csharp
if (numbers is [1, ..])
    Console.WriteLine("Starts with 1");
```

* 匹配以 3 结尾的数组。

```csharp
if (numbers is [.., 3])
    Console.WriteLine("Ends with 3");
```

* 开头和结尾都可检查，中间元素数量不限。

```csharp
if (numbers is [1, .., 3])
    Console.WriteLine("Starts with 1 and ends with 3");
```

#### 变量绑定

```csharp
if (numbers is [1, .. var middle, 3])
    Console.WriteLine($"Middle elements count: {middle.Length}");
```

> 使用 var middle 捕获中间切片。

#### 类型模式

```csharp
object[] items = { "hello", 42, 99 };

if (items is ["hello", int x, ..])
    Console.WriteLine($"Second element is int: {x}");
```

> 结合类型模式和切片。

#### 空集合匹配

```csharp
   if (arr is []) 
   {
       Console.WriteLine("Array is empty");
   }
```

#### 嵌套列表模式

```csharp
object[] data = [1, [2, 3], 4];

string result = data switch
{
    [1, [2, 3], 4] => "Nested match: [1, [2, 3], 4]",
    _ => "No match"
};

Console.WriteLine(result); // 输出: Nested match: [1, [2, 3], 4]
```

#### 结合 switch 表达式

```csharp
int[] nums = { 1, 2, 3, 4 };

string result = nums switch
{
    [1, 2, 3, 4] => "Exact 1-2-3-4",
    [1, ..]      => "Starts with 1",
    [.., 4]      => "Ends with 4",
    []           => "Empty list",
    _            => "Other"
};
Console.WriteLine(result);
```

### 高级示例

#### 斐波那契检测

```csharp
int[] fib = { 0, 1, 1, 2, 3, 5, 8 };

if (fib is [0, 1, .. var rest])
    Console.WriteLine($"Fibonacci sequence with {rest.Length + 2} elements");
```

#### 解析命令行参数

```csharp
string[] args = { "add", "user", "Alice" };

var command = args switch
{
    ["add", "user", var name] => $"Add user {name}",
    ["remove", "user", var name] => $"Remove user {name}",
    [] => "No command",
    _ => "Invalid"
};

Console.WriteLine(command);
```

#### 带 when 条件

```csharp
if (numbers is [var first, .., var last] && first < last)
    Console.WriteLine($"First < Last ({first} < {last})");
```

### 适用场景

* 数据验证：检查输入的集合是否符合预期结构。

* 数据提取：从集合中提取特定元素到变量，减少手动索引操作。

* 简化控制流：替代复杂的 `if-else` 或循环逻辑。

* 递归处理：结合切片模式处理嵌套或不定长集合。