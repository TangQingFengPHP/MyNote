### 简介

`TimeSpan` 是 `.NET` 中用于表示时间间隔或持续时间的重要结构体。它提供了丰富的方法和属性来处理时间跨度，从几毫秒到几百万天都可以精确表示。

### 概念与特性

`TimeSpan` 表示一个时间间隔（时间段），而不是具体的时间点。

| 特性     | 说明                                |
| -------- | ----------------------------------- |
| 命名空间 | `System`                            |
| 结构类型 | `struct`（值类型）                  |
| 表示范围 | 约 ±10,675,199 天（≈29,000 年）     |
| 精度     | 1 Tick = 100 纳秒                   |
| 单位表示 | 可用天、小时、分钟、秒、毫秒、Ticks |

与 `DateTime` 区别：

* `DateTime` 表示一个具体的时间点（例如：`2025-09-18 08:00`）。

* `TimeSpan` 表示一个时间间隔（例如：2 小时 30 分钟）。

### 创建 TimeSpan

#### 构造函数

```csharp
// 构造函数：TimeSpan(days, hours, minutes, seconds)
var span1 = new TimeSpan(1, 2, 30, 0); // 1天2小时30分0秒

// 构造函数：TimeSpan(hours, minutes, seconds)
var span2 = new TimeSpan(2, 30, 0);    // 2小时30分
```

#### 静态工厂方法

推荐使用静态方法，可读性更高：

```csharp
TimeSpan ts1 = TimeSpan.FromDays(1.5);       // 1.5天
TimeSpan ts2 = TimeSpan.FromHours(2.5);      // 2.5小时
TimeSpan ts3 = TimeSpan.FromMinutes(90);     // 90分钟
TimeSpan ts4 = TimeSpan.FromSeconds(45);     // 45秒
TimeSpan ts5 = TimeSpan.FromMilliseconds(500); // 500毫秒
TimeSpan ts6 = TimeSpan.FromTicks(5000);     // 5000 Ticks
```

#### 解析字符串

```csharp
TimeSpan ts = TimeSpan.Parse("1.02:30:00");  
// 格式：d.hh:mm:ss -> 1天2小时30分
```

或安全解析：

```csharp
if (TimeSpan.TryParse("02:15", out var result))
    Console.WriteLine(result); // 02:15:00
```

### 常用属性

| 属性                | 说明                   | 示例                   |
| ------------------- | ---------------------- | ---------------------- |
| `Days`              | 总天数(整数部分)       | `ts.Days`              |
| `Hours`             | 小时(0–23)             | `ts.Hours`             |
| `Minutes`           | 分钟(0–59)             | `ts.Minutes`           |
| `Seconds`           | 秒(0–59)               | `ts.Seconds`           |
| `Milliseconds`      | 毫秒(0–999)            | `ts.Milliseconds`      |
| `Ticks`             | 以 Tick（100ns）为单位 | `ts.Ticks`             |
| `TotalDays`         | 总天数(含小数)         | `ts.TotalDays`         |
| `TotalHours`        | 总小时数(含小数)       | `ts.TotalHours`        |
| `TotalMinutes`      | 总分钟数(含小数)       | `ts.TotalMinutes`      |
| `TotalSeconds`      | 总秒数(含小数)         | `ts.TotalSeconds`      |
| `TotalMilliseconds` | 总毫秒数               | `ts.TotalMilliseconds` |

区别

* `Days`、`Hours` 等返回整数部分（对应分量）。

* `TotalDays`、`TotalHours` 等返回完整总量。

示例：

```csharp
var ts = new TimeSpan(1, 2, 30, 0);
Console.WriteLine(ts.Days);        // 1
Console.WriteLine(ts.TotalHours);  // 26.5
```

### 运算操作

#### 加减

```csharp
var t1 = TimeSpan.FromHours(3);
var t2 = TimeSpan.FromMinutes(30);

TimeSpan sum = t1 + t2;  // 3:30:00
TimeSpan diff = t1 - t2; // 2:30:00
```

#### 乘除

```csharp
TimeSpan doubleTime = TimeSpan.FromMinutes(45) * 2; // 1:30:00
TimeSpan halfTime   = TimeSpan.FromHours(4) / 2;    // 2:00:00
```

> C# 11 之前不支持直接乘除，需要用 TimeSpan.FromTicks() 手动计算。
（.NET 7 / C# 11 起支持 * / 运算符）

#### 比较

```csharp
TimeSpan ts1 = TimeSpan.FromHours(2);
TimeSpan ts2 = TimeSpan.FromMinutes(120); // 也是2小时

// 比较
bool isEqual = ts1 == ts2;                // true
bool isNotEqual = ts1 != ts2;             // false
bool isGreater = ts1 > TimeSpan.FromHours(1); // true
bool isLess = ts1 < TimeSpan.FromHours(3);    // true

// 比较方法
int compareResult = ts1.CompareTo(ts2);   // 0 (相等)
bool equals = ts1.Equals(ts2);            // true
```

#### 取绝对值

```csharp
var negative = TimeSpan.FromHours(-5);
var positive = negative.Duration(); // 05:00:00
```

#### 取负值

```csharp
var neg = -TimeSpan.FromMinutes(30); // -00:30:00
```

### 与 DateTime 配合

`TimeSpan` 常用于计算两个时间点的差值：

```csharp
DateTime start = DateTime.Now;
// 模拟一些操作
System.Threading.Thread.Sleep(1500);
DateTime end = DateTime.Now;

TimeSpan elapsed = end - start;
Console.WriteLine($"执行耗时: {elapsed.TotalMilliseconds} ms");
```

也可以用来加减时间：

```csharp
DateTime tomorrow = DateTime.Now + TimeSpan.FromDays(1);
```

### 格式化输出

#### 默认格式

```csharp
Console.WriteLine(ts.ToString()); // "1.02:30:00" (d.hh:mm:ss)
```

#### 标准格式

```csharp
Console.WriteLine(ts.ToString("c"));     // 同上 (常数格式)
Console.WriteLine(ts.ToString("g"));     // 1:2:30:45.5 (常规短格式)
Console.WriteLine(ts.ToString("G"));     // 1:02:30:45.5000000 (常规长格式)
```

#### 自定义格式

`ToString` 支持标准和自定义格式：

```csharp
var ts = new TimeSpan(1, 2, 30, 45, 500);
Console.WriteLine(ts.ToString(@"hh\:mm\:ss"));        // 02:30:45
Console.WriteLine(ts.ToString(@"d\.hh\:mm\:ss\.fff")); // 1.02:30:45.500
```

> 需要转义 : 和 .

### 常用静态字段

| 字段                | 说明         |
| ------------------- | ------------ |
| `TimeSpan.Zero`     | 表示 0       |
| `TimeSpan.MaxValue` | 最大可表示值 |
| `TimeSpan.MinValue` | 最小可表示值 |

### 典型使用场景

#### 定时任务/超时控制

```csharp
var timeout = TimeSpan.FromSeconds(30);
var cts = new CancellationTokenSource(timeout);
```

#### 统计程序耗时

```csharp
var watch = System.Diagnostics.Stopwatch.StartNew();
// do work
watch.Stop();
Console.WriteLine(watch.Elapsed); // TimeSpan
```

### 实际应用示例

```csharp
using System;

class Program
{
    static void Main()
    {
        DateTime start = DateTime.Now;

        // 模拟任务
        System.Threading.Thread.Sleep(1200);

        DateTime end = DateTime.Now;
        TimeSpan span = end - start;

        Console.WriteLine($"耗时: {span.TotalMilliseconds} 毫秒");
        Console.WriteLine($"格式化: {span.ToString(@"hh\:mm\:ss\.fff")}");
        Console.WriteLine($"天: {span.Days}, 小时: {span.Hours}, 总小时: {span.TotalHours:F2}");
    }
}
```

输出示例：

```
耗时: 1203.45 毫秒
格式化: 00:00:01.203
天: 0, 小时: 0, 总小时: 0.00
```

### 高级用法

#### 自定义 TimeSpan 扩展方

```csharp
public static class TimeSpanExtensions
{
    public static string ToHumanReadableString(this TimeSpan timeSpan)
    {
        if (timeSpan.TotalDays >= 1)
            return $"{(int)timeSpan.TotalDays} 天 {timeSpan.Hours} 小时";
        
        if (timeSpan.TotalHours >= 1)
            return $"{(int)timeSpan.TotalHours} 小时 {timeSpan.Minutes} 分钟";
        
        if (timeSpan.TotalMinutes >= 1)
            return $"{(int)timeSpan.TotalMinutes} 分钟 {timeSpan.Seconds} 秒";
        
        if (timeSpan.TotalSeconds >= 1)
            return $"{(int)timeSpan.TotalSeconds} 秒";
        
        return $"{timeSpan.Milliseconds} 毫秒";
    }
}

// 使用扩展方法
TimeSpan ts = TimeSpan.FromHours(2.5);
Console.WriteLine(ts.ToHumanReadableString()); // 输出: 2 小时 30 分钟
```

### 注意事项和最佳实践

* 不可变性：`TimeSpan` 是值类型且不可变，所有操作都返回新的 `TimeSpan` 实例

* 精度考虑：`TimeSpan` 使用 `ticks`（100 纳秒）作为内部存储，提供高精度但需要注意浮点运算的精度问题

* 文化敏感性：解析和格式化 `TimeSpan` 时，注意当前线程的文化设置

* 性能考虑：对于高性能场景，考虑使用 `Stopwatch` 而不是 `DateTime` 减法来计算时间间隔