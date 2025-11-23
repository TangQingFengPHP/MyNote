### 简介

`C# 8.0` 引入了范围（`Ranges`）和索引（`Indices`）功能，提供了更简洁、更直观的语法来处理集合中的元素和子集。这些功能大大简化了数组、字符串、列表等数据结构的操作。

### 索引（Indices）

#### 从末尾开始的索引

使用 `^` 运算符表示从末尾开始的索引：

```csharp
int[] numbers = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };

// 传统方式获取最后一个元素
int last1 = numbers[numbers.Length - 1]; // 9

// 使用索引运算符获取最后一个元素
int last2 = numbers[^1]; // 9

// 获取倒数第二个元素
int secondLast = numbers[^2]; // 8

// 获取倒数第三个元素
int thirdLast = numbers[^3]; // 7
```

#### 索引的工作原理

索引实际上是 `System.Index` 结构体的语法糖：

```csharp
// 以下两行代码是等价的
int last = numbers[^1];
int last = numbers[new Index(1, fromEnd: true)];

// 从开头开始的索引
int first = numbers[0]; // 等价于 numbers[new Index(0, fromEnd: false)]
```

| 表达式 | 含义                    | 等同于                    |
| ------ | ----------------------- | ------------------------- |
| `^0`   | 序列结束后的位置        | `array.Length`            |
| `^1`   | 最后一个元素            | `array[array.Length - 1]` |
| `^2`   | 倒数第二个元素          | `array[array.Length - 2]` |
| `^n`   | 从末尾算起的第 n 个元素 | `array[array.Length - n]` |

### 范围（Ranges）

#### 基本范围操作

使用 `..` 运算符指定范围：

```csharp
int[] numbers = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };

// 获取索引1到3的元素（不包括索引3）
int[] sub1 = numbers[1..3]; // [1, 2]

// 获取从开始到索引3的元素
int[] sub2 = numbers[..3]; // [0, 1, 2]

// 获取从索引6到末尾的元素
int[] sub3 = numbers[6..]; // [6, 7, 8, 9]

// 获取所有元素
int[] all = numbers[..]; // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

// 使用从末尾开始的索引定义范围
int[] sub4 = numbers[^3..^1]; // [7, 8] - 从倒数第三个到倒数第一个（不包括）
```

#### 范围的工作原理

范围实际上是 `System.Range` 结构体的语法糖：

```csharp
// 以下两行代码是等价的
int[] sub = numbers[1..4];
int[] sub = numbers[new Range(1, 4)];

// Range 包含 Start 和 End 两个 Index
Range range = 1..4;
Console.WriteLine($"Start: {range.Start}, End: {range.End}");
// 输出: Start: 1, End: 4
```

| 语法           | 含义                 | 等同于                           |
| -------------- | -------------------- | -------------------------------- |
| `..`           | 整个范围             | `[0..^0]`                        |
| `start..`      | 从 start 到序列结束  | `[start..^0]`                    |
| `..end`        | 从开始到 end 之前    | `[0..end]`                       |
| `start..end`   | 从 start 到 end 之前 | `[start..end]`                   |
| `^start..^end` | 使用末尾索引指定范围 | `[length - start..length - end]` |

#### 范围表达式返回值

范围表达式返回的是原序列的视图(`view`)，而不是副本。对于数组、字符串等类型，它返回的是只读视图；对于 `Span<T>` 和 `Memory<T>`，它返回新的 `Span` 或 `Memory`。

```csharp
int[] original = [1, 2, 3, 4, 5];
int[] slice = original[1..4]; // [2, 3, 4]

// 修改原始数组会影响切片
original[2] = 100;
Console.WriteLine(string.Join(", ", slice)); // 2, 100, 4

// 修改切片也会影响原始数组
slice[1] = 200;
Console.WriteLine(string.Join(", ", original)); // 1, 2, 200, 4, 5
```

### 不同类型的使用示例

#### 数组

```csharp
int[] array = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };

// 获取前5个元素
int[] firstFive = array[..5]; // [0, 1, 2, 3, 4]

// 获取最后3个元素
int[] lastThree = array[^3..]; // [7, 8, 9]

// 获取中间部分
int[] middle = array[3..7]; // [3, 4, 5, 6]

// 获取除第一个和最后一个之外的所有元素
int[] withoutEnds = array[1..^1]; // [1, 2, 3, 4, 5, 6, 7, 8]
```

#### 字符串

```csharp
string text = "Hello, World!";

// 获取前5个字符
string hello = text[..5]; // "Hello"

// 获取最后6个字符
string world = text[^6..]; // "World!"

// 获取逗号后的部分（不包括逗号本身和空格）
string afterComma = text[7..^1]; // "World"

// 获取子字符串
string sub = text[7..12]; // "World"
```

#### 列表（`List<T>`）

```csharp
List<int> list = new List<int> { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };

// 获取范围（需要转换为数组或使用GetRange）
int[] subArray = list.ToArray()[2..6]; // [2, 3, 4, 5]

// 或者使用List的GetRange方法（不是基于范围的语法，但功能类似）
List<int> subList = list.GetRange(2, 4); // [2, 3, 4, 5]
```

#### `Span<T>` 和 `Memory<T>`

```csharp
int[] array = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };

// 创建Span并使用范围
Span<int> span = array.AsSpan();
Span<int> spanSlice = span[2..6]; // [2, 3, 4, 5]

// 创建Memory并使用范围
Memory<int> memory = array.AsMemory();
Memory<int> memorySlice = memory[3..7]; // [3, 4, 5, 6]
```

### 高级用法和模式

#### 与模式匹配结合使用

```csharp
int[] numbers = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };

// 使用范围进行模式匹配
if (numbers is [0, 1, 2, .., 8, 9])
{
    Console.WriteLine("数组以0,1,2开头，以8,9结尾");
}

// 在switch表达式中使用
string result = numbers switch
{
    [0, 1, 2, ..] => "以0,1,2开头",
    [.., 7, 8, 9] => "以7,8,9结尾",
    [0, .., 9] => "以0开头，以9结尾",
    _ => "其他模式"
};
```

#### 自定义类型支持范围

要使自定义类型支持范围操作，需要实现以下方法之一：

```csharp
public class MyCollection<T>
{
    private T[] _items;
    
    public MyCollection(T[] items)
    {
        _items = items;
    }
    
    // 方法1：实现Slice方法
    public MyCollection<T> Slice(int start, int length)
    {
        T[] slice = new T[length];
        Array.Copy(_items, start, slice, 0, length);
        return new MyCollection<T>(slice);
    }
    
    // 方法2：实现索引器接受Range参数
    public MyCollection<T> this[Range range]
    {
        get
        {
            var (start, length) = GetStartAndLength(range);
            return Slice(start, length);
        }
    }
    
    // 辅助方法：将Range转换为(start, length)
    private (int start, int length) GetStartAndLength(Range range)
    {
        int start = range.Start.IsFromEnd ? 
            _items.Length - range.Start.Value : range.Start.Value;
        
        int end = range.End.IsFromEnd ? 
            _items.Length - range.End.Value : range.End.Value;
        
        int length = end - start;
        return (start, length);
    }
    
    // 其他成员...
}

// 使用自定义集合的范围操作
var collection = new MyCollection<int>(new[] { 0, 1, 2, 3, 4, 5 });
var subCollection = collection[1..4]; // 包含元素[1, 2, 3]
```

### 实际应用场景

#### 字符串处理

```csharp
// 提取文件扩展名
string GetFileExtension(string filename)
{
    int dotIndex = filename.LastIndexOf('.');
    return dotIndex >= 0 ? filename[(dotIndex + 1)..] : string.Empty;
}

// 提取域名
string GetDomain(string url)
{
    int protocolEnd = url.IndexOf("://");
    if (protocolEnd < 0) return url;
    
    int domainStart = protocolEnd + 3;
    int pathStart = url.IndexOf('/', domainStart);
    
    return pathStart < 0 ? 
        url[domainStart..] : 
        url[domainStart..pathStart];
}

// 处理CSV行
string[] ParseCsvLine(string line)
{
    List<string> fields = new List<string>();
    int start = 0;
    
    while (start < line.Length)
    {
        int end = line.IndexOf(',', start);
        if (end < 0) end = line.Length;
        
        fields.Add(line[start..end].Trim());
        start = end + 1;
    }
    
    return fields.ToArray();
}
```

#### 数据分页

```csharp
// 使用范围实现分页
public IEnumerable<T> GetPage<T>(T[] data, int pageNumber, int pageSize)
{
    int startIndex = (pageNumber - 1) * pageSize;
    if (startIndex >= data.Length)
        return Enumerable.Empty<T>();
    
    int endIndex = Math.Min(startIndex + pageSize, data.Length);
    return data[startIndex..endIndex];
}

// 使用Span<T>提高性能
public ReadOnlySpan<T> GetPageSpan<T>(T[] data, int pageNumber, int pageSize)
{
    int startIndex = (pageNumber - 1) * pageSize;
    if (startIndex >= data.Length)
        return ReadOnlySpan<T>.Empty;
    
    int endIndex = Math.Min(startIndex + pageSize, data.Length);
    return data.AsSpan()[startIndex..endIndex];
}
```

#### 数组操作

```csharp
// 数组旋转
void RotateArrayLeft<T>(T[] array, int positions)
{
    positions %= array.Length;
    if (positions == 0) return;
    
    // 创建临时数组保存前positions个元素
    T[] temp = array[..positions];
    
    // 将剩余元素向左移动
    Array.Copy(array, positions, array, 0, array.Length - positions);
    
    // 将临时数组中的元素放回末尾
    Array.Copy(temp, 0, array, array.Length - positions, positions);
}

// 数组分割
(T[] left, T[] right) SplitArray<T>(T[] array, int splitIndex)
{
    return (array[..splitIndex], array[splitIndex..]);
}
```