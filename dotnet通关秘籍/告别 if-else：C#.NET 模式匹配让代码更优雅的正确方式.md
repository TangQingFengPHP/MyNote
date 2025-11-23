### 简介

模式匹配是 `C# 7.0` 开始引入的革命性特性，它提供了更简洁、更强大的方式来检查和提取数据中的信息。随着每个版本的更新，模式匹配功能不断强化，成为现代 `C#` 开发的核心特性。

模式匹配允许将输入表达式与各种特征进行匹配，支持多种模式类型。它主要用于：

* `is` 表达式：检查并可能声明变量。

* `switch` 语句：传统分支逻辑。

* `switch` 表达式：更简洁的表达式形式（`C# 8.0` 引入）。

### 模式匹配演进历程

| 版本    | 新增特性                                       |
| ------- | ---------------------------------------------- |
| C# 7.0  | 基本模式匹配：is 表达式、switch 语句增强       |
| C# 8.0  | 递归模式、属性模式、位置模式、switch 表达式    |
| C# 9.0  | 类型模式简化、逻辑模式（and/or/not）、关系模式 |
| C# 10.0 | 扩展属性模式                                   |
| C# 11.0 | 列表模式、切片模式                             |

### 基础模式类型

#### 声明模式

```csharp
// 检查类型并声明新变量
if (obj is string str)
{
    Console.WriteLine($"字符串长度: {str.Length}");
}

// 传统方式对比
if (obj is string)
{
    string str = (string)obj;
    Console.WriteLine($"字符串长度: {str.Length}");
}
```

#### 类型模式

```csharp
// 只检查类型，不声明变量
if (obj is string)
{
    Console.WriteLine("这是一个字符串");
}

// 在 switch 表达式中使用
var result = obj switch
{
    string => "它是字符串",
    int => "它是整数",
    _ => "其他类型"
};
```

#### 常量模式

```csharp
// 检查特定值
if (value is 5) { /* 值等于5 */ }
if (value is null) { /* 值为null */ }
if (value is "hello") { /* 字符串比较 */ }

// 在 switch 中
string message = value switch
{
    0 => "零",
    1 => "一",
    null => "空值",
    _ => "其他值"
};
```

### 高级模式匹配

#### 属性模式

```csharp
// 检查对象属性
if (person is { Name: "John", Age: > 18 })
{
    Console.WriteLine("John 已成年");
}

// 嵌套属性模式
if (order is { Customer: { Address: { City: "Beijing" } } })
{
    Console.WriteLine("北京客户的订单");
}

// C# 10 扩展属性模式
if (order is { Customer.Address.City: "Beijing" })
{
    Console.WriteLine("北京客户的订单（简化写法）");
}
```

#### 位置模式（元组模式）

```csharp
// 元组解构和匹配
var point = (x: 5, y: 10);
var quadrant = point switch
{
    (0, 0) => "原点",
    (var x, var y) when x > 0 && y > 0 => "第一象限",
    (var x, var y) when x < 0 && y > 0 => "第二象限",
    (var x, var y) when x < 0 && y < 0 => "第三象限",
    (var x, var y) when x > 0 && y < 0 => "第四象限",
    _ => "在坐标轴上"
};
```

#### 逻辑模式（C# 9.0+）

```csharp
// 使用逻辑运算符组合模式
bool IsValid(object obj) => obj is not null and (string or int);

// 复杂逻辑组合
var category = value switch
{
    > 0 and < 10 => "小的正数",
    < 0 or > 100 => "超出范围",
    not null => "非空值",
    _ => "其他"
};
```

#### 关系模式（C# 9.0+）

```csharp
// 使用关系运算符
string GetSize(int value) => value switch
{
    < 0 => "负数",
    >= 0 and < 10 => "小",
    >= 10 and < 100 => "中",
    >= 100 => "大"
};

// 与类型模式结合
string Describe(object obj) => obj switch
{
    int i when i < 0 => "负整数",
    int i when i > 0 => "正整数",
    int => "零",
    _ => "其他"
};
```

#### 列表模式（C# 11.0+）

```csharp
// 匹配数组或列表的特定模式
int[] numbers = { 1, 2, 3, 4 };

var result = numbers switch
{
    [1, 2, 3, 4] => "精确匹配",
    [1, 2, ..] => "以1,2开头",
    [.., 3, 4] => "以3,4结尾",
    [1, .. var middle, 4] => $"以1开头4结尾，中间有{middle.Length}个元素",
    [] => "空数组",
    _ => "其他模式"
};

// 切片模式
if (numbers is [var first, .. var rest])
{
    Console.WriteLine($"第一个元素: {first}, 剩余元素: {rest.Length}");
}
```

#### 字符串模式

```csharp
string text = "Hello, World";

if (text is ['H','e','l','l','o', ..])
{
    Console.WriteLine("以Hello开头");
}

// 与Span<char>模式匹配结合
ReadOnlySpan<char> span = text.AsSpan();
if (span is ['H','e','l','l','o', ..])
{
    Console.WriteLine("匹配Span");
}
```

#### 自定义模式匹配

```csharp
// 通过 Deconstruct 方法支持位置模式
public class Point
{
    public int X { get; }
    public int Y { get; }
    
    public void Deconstruct(out int x, out int y)
    {
        x = X;
        y = Y;
    }
}

// 现在可以对 Point 使用位置模式
var quadrant = point switch
{
    (0, 0) => "原点",
    (var x, var y) when x > 0 && y > 0 => "第一象限",
    // ...
};w
```

#### 表达式树中的模式匹配

```csharp
Expression<Func<object, bool>> expr = o => o switch
{
    int i when i > 0 => true,
    string s when s.Length > 5 => true,
    _ => false
};

// 解析表达式树
if (expr.Body is SwitchExpression switchExpr)
{
    foreach (SwitchCase @case in switchExpr.Cases)
    {
        // 处理每个case
    }
}
```

### 实际应用场景

#### 数据验证和处理

```csharp
public ValidationResult Validate(Order order)
{
    return order switch
    {
        null => ValidationResult.Failure("订单不能为空"),
        { TotalAmount: <= 0 } => ValidationResult.Failure("金额必须大于0"),
        { Customer: null } => ValidationResult.Failure("客户信息缺失"),
        { Items: [] } => ValidationResult.Failure("订单项不能为空"),
        { Status: OrderStatus.Cancelled } => ValidationResult.Failure("订单已取消"),
        _ => ValidationResult.Success()
    };
}
```

#### API 响应处理

```csharp
public string HandleResponse(ApiResponse response)
{
    return response switch
    {
        { StatusCode: 200, Data: not null } => "成功",
        { StatusCode: 404 } => "资源未找到",
        { StatusCode: >= 400 and < 500 } => "客户端错误",
        { StatusCode: >= 500 } => "服务器错误",
        { Error: not null } => $"错误: {response.Error.Message}",
        _ => "未知响应"
    };
}
```

