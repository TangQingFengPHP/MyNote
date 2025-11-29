### 简介

`C# 11` 引入了原始字符串字面量（`Raw String Literals`），这是一个革命性的特性，极大地简化了包含大量特殊字符（如引号、反斜杠、换行符等）的字符串处理。

原始字符串字面量允许创建包含任意文本的字符串，而无需转义特殊字符。它们以至少三个双引号 `"""` 开始和结束，并且可以跨越多行。

### 核心特性与优势

#### 核心特点

* 无转义需求：不需要对引号、反斜杠等特殊字符进行转义

* 多行支持：原生支持多行字符串格式

* 空白保留：精确保留源代码中的缩进和格式化

* 灵活分隔符：使用可变数量的双引号作为分隔符

* 字符串插值：可与插值语法无缝结合

#### 与传统字符串对比

| 特性     | 常规字符串       | 逐字字符串(@)                  | 原始字符串字面量 |
| -------- | ---------------- | ------------------------------ | ---------------- |
| 转义需求 | 需要转义特殊字符 | 不需要转义，但对双引号仍需转义 | 完全不需要转义   |
| 多行支持 | 需使用`\n`等     | 支持                           | 原生支持         |
| 缩进保留 | 不保留           | 保留所有缩进                   | 智能缩进处理     |
| 引号处理 | `\"`             | `""`                           | 直接使用         |
| JSON/XML | 可读性差         | 可读性中等                     | 完美呈现         |

### 基本语法

```csharp
// 基本原始字符串
string basic = """这是一个原始字符串，不需要转义 "引号" 和 \反斜杠""";

// 多行原始字符串
string multiLine = """
    这是一个
    多行原始字符串
    可以包含 "引号" 和 \反斜杠
    """;

// 包含大括号的原始字符串
string withBraces = """
    { "name": "John", "age": 30 }
    """;
```

### 核心特性与优势

#### 无需转义特殊字符

```csharp
// 传统方式（需要转义）
string traditional = "这是一个包含\"引号\"和\\反斜杠的字符串";

// 原始字符串方式（无需转义）
string raw = """这是一个包含"引号"和\反斜杠的字符串""";

// 正则表达式示例
string regexPattern = """^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$""";
```

#### 多行字符串处理

```csharp
// 传统多行字符串（需要转义和连接）
string oldMultiLine = "第一行\n" +
                     "第二行\n" +
                     "第三行";

// 原始多行字符串（更简洁）
string newMultiLine = """
    第一行
    第二行
    第三行
    """;

// JSON 示例
string json = """
    {
        "name": "John Doe",
        "age": 30,
        "email": "john@example.com"
    }
    """;
```

#### 缩进处理

原始字符串字面量会自动处理缩进，使代码保持整洁：

```csharp
string GetFormattedMessage()
{
    return """
        Hello,
            This is an indented message.
        It preserves the formatting exactly as written.
        """;
}

// 输出结果：
// Hello,
//     This is an indented message.
// It preserves the formatting exactly as written.
```

#### 引号数量规则

当字符串内容包含三个或更多连续双引号时，需要使用更多引号作为分隔符：

```csharp
// 内容包含三个双引号 → 使用四个作为分隔符
string tripleQuotes = """" 这里可以包含 """ 三个引号 """";

// 内容包含四个双引号 → 使用五个作为分隔符
string fourQuotes = """"" 这里可以包含 """" 四个引号 """"";
```

引号数量选择原则

* 分隔符的双引号数量(N)必须大于内容中连续双引号的最大数量(M)

* 最小分隔符数量为3（即使内容没有三个引号）

* 公式：`N > M`

#### 多$符号规则

* `$` 的数量决定了插值表达式的边界

* 一个 `$` → 表达式使用 `{ }`

* 两个 `$$` → 表达式使用 `{{ }}`，以此类推

### 高级用法与技巧

#### 包含更多引号

如果需要字符串中包含三个或更多连续双引号，可以使用更多引号作为分隔符：

```csharp
// 使用四个引号作为分隔符，以包含三个引号
string withTripleQuotes = """"
    这个字符串包含三个连续引号: """
    """";

// 使用五个引号作为分隔符，以包含四个引号
string withFourQuotes = """""
    这个字符串包含四个连续引号: """"
    """"";
```

#### 与字符串插值结合使用

原始字符串可以与字符串插值结合使用，提供更强大的功能：

```csharp
string name = "John";
int age = 30;

// 基本插值
string interpolated = $$"""
    {
        "name": "{{name}}",
        "age": {{age}}
    }
    """;

// 复杂插值（使用多个$符号）
string complex = $$$"""
    {
        "name": "{{{name}}}",  // 注意：这里需要三个大括号
        "age": {{age}}
    }
    """;
```

#### 最小化缩进

原始字符串会自动去除与结束引号对齐的缩进：

```csharp
string minimalIndent = """
    First line
    Second line
    Third line
    """; // 结束引号的缩进决定了最小缩进

// 等效于：
// "First line\nSecond line\nThird line"
```

### 实际应用场景

#### JSON 和 XML 处理

```csharp
// JSON 配置
string configJson = """
    {
        "appSettings": {
            "timeout": 30,
            "retryCount": 3,
            "apiUrl": "https://api.example.com"
        },
        "logging": {
            "level": "Information",
            "filePath": "logs/app.log"
        }
    }
    """;

// XML 文档
string xmlDocument = """
    <?xml version="1.0" encoding="UTF-8"?>
    <catalog>
        <book id="bk101">
            <author>Gambardella, Matthew</author>
            <title>XML Developer's Guide</title>
            <genre>Computer</genre>
            <price>44.95</price>
            <publish_date>2000-10-01</publish_date>
            <description>An in-depth look at creating applications with XML.</description>
        </book>
    </catalog>
    """;
```

#### 正则表达式模式

```csharp
// 复杂的正则表达式（无需转义反斜杠）
string emailPattern = """^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$""";
string phonePattern = """^(\+\d{1,3}\s?)?(\(\d{1,4}\)\s?)?\d{1,4}[\s.-]?\d{1,4}[\s.-]?\d{1,9}$""";

// 使用正则表达式
bool isValidEmail = Regex.IsMatch("test@example.com", emailPattern);
```

#### SQL 查询

```csharp
string sqlQuery = """
    SELECT 
        u.Id,
        u.Name,
        u.Email,
        COUNT(o.Id) AS OrderCount
    FROM Users u
    LEFT JOIN Orders o ON u.Id = o.UserId
    WHERE u.IsActive = 1
    GROUP BY u.Id, u.Name, u.Email
    HAVING COUNT(o.Id) > 0
    ORDER BY OrderCount DESC
    """;
```

#### 代码生成

```csharp
string GenerateCSharpClass(string className, IEnumerable<string> properties)
{
    return $$"""
        public class {{className}}
        {
            {{string.Join("\n    ", properties.Select(p => $"public string {p} {{ get; set; }}"))}}
            
            public {{className}}()
            {
            }
            
            public override string ToString()
            {
                return $"{{
                    {{string.Join(", ", properties.Select(p => $"{p}={{{p}}}"))}}
                }}";
            }
        }
        """;
}

// 使用示例
string classCode = GenerateCSharpClass("Person", new[] { "FirstName", "LastName", "Age" });
```

### 总结

`C# 11` 的原始字符串字面量是一个强大的特性，它：

* 消除转义需求：无需转义引号、反斜杠等特殊字符

* 支持多行内容：轻松创建多行字符串，保持格式

* 智能缩进处理：自动处理缩进，使代码更整洁

* 与插值结合：支持字符串插值，创建动态内容

* 编译时处理：无运行时性能开销

适用场景：

* `JSON/XML/HTML` 内容：处理结构化数据

* 正则表达式：编写复杂的模式匹配

`SQL` 查询：编写多行数据库查询

* 代码生成：创建模板化代码

* 文档字符串：编写格式化的文档

