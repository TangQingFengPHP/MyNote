### 简介

`DateTime` 是 `System` 命名空间中用于表示日期和时间的结构体，广泛用于处理时间相关的操作，如存储、计算、格式化等。

### DateTime 结构概述

* 定义：`System.DateTime` 是一个值类型（`struct`），表示自公元 0001 年 1 月 1 日午夜 00:00:00（`DateTime.MinValue`）起经过的“刻度”（ticks，1 tick = 100 纳秒）数。

* 内部存储：一个 `long` 值（ticks）+ 一个 `DateTimeKind` 枚举，指示时间的“种类”：

    * `Unspecified`：未指定是本地还是 `UTC`

    * `Utc`：协调世界时

    * `Local`：本地时区时间

* 范围：`DateTime.MinValue`（0001-01-01），`DateTime.MaxValue`（9999-12-31 23:59:59.9999999）

```csharp
// 时间单位转换
1 秒 = 10,000,000 Ticks
1 毫秒 = 10,000 Ticks
1 分钟 = 600,000,000 Ticks
```

### 主要功能

#### 表示日期和时间：

* 包含年、月、日、时、分、秒、毫秒。

* 支持 `Kind` 属性（`Utc、Local、Unspecified`）。

#### 时间操作：

* 比较：`Compare、CompareTo`。

* 运算：加减时间（`AddDays、AddHours` 等）。

* 时间差：`Subtract` 返回 `TimeSpan`。

#### 格式化：

* 使用 `ToString` 格式化（如 `"yyyy-MM-dd HH:mm:ss"`）。

* 支持区域性（`CultureInfo`）。

#### 转换：

* 与字符串、`Unix` 时间戳、数据库时间互转。

#### 静态属性：

* `DateTime.Now`：本地时间。

* `DateTime.UtcNow`：`UTC` 时间。

* `DateTime.Today`：当前日期（时间为 00:00:00）。

### 核心属性

| 属性                                      | 说明                                              |
| ----------------------------------------- | ------------------------------------------------- |
| `Now`                                     | 返回本地时区当前时间                              |
| `UtcNow`                                  | 返回 UTC 当前时间                                 |
| `Today`                                   | 返回本地时区当前日期（00:00:00 时分秒）           |
| `Date`                                    | 取当前实例的日期部分，时间部分置 00:00:00         |
| `Year`, `Month`, `Day`                    | 年、月、日                                        |
| `Hour`, `Minute`, `Second`, `Millisecond` | 小时、分钟、秒、毫秒                              |
| `DayOfWeek`                               | 星期枚举（`DayOfWeek.Monday`…）                   |
| `DayOfYear`                               | 年内天数（1–366）                                 |
| `Kind`                                    | 时间种类（`Utc`/`Local`/`Unspecified`）           |
| `Ticks`                                   | 自 0001-01-01 00:00:00 起的刻度数（100ns 为单位） |

```csharp
var dt = DateTime.Now;
Console.WriteLine($"{dt.Year}-{dt.Month}-{dt.Day} {dt.Hour}:{dt.Minute}:{dt.Second}, Kind={dt.Kind}");
```

### 构造与转换

#### 构造器

```csharp
// 指定年月日时分秒
var dt1 = new DateTime(2025, 8, 8, 14, 30, 0);

// 指定 Kind
var dt2 = new DateTime(2025,8,8,14,30,0, DateTimeKind.Utc);
```

#### 从 ticks

```csharp
var dtTicks = new DateTime(637632000000000000L);
```

#### 与 Unix 时间戳

```csharp
// DateTime <-> Unix seconds
var unixEpoch = DateTimeOffset.FromUnixTimeSeconds(0).UtcDateTime;
long toUnix = ((DateTimeOffset)DateTime.UtcNow).ToUnixTimeSeconds();
```

#### 与 DateTimeOffset

* `DateTimeOffset` 推荐用于跨时区场景，包含一个偏移量字段，且不朴素依赖 `Kind`。

* 从 `DateTime` 转换：

```csharp
var dto = new DateTimeOffset(dt);              // 按本地偏移
var dtoUtc = new DateTimeOffset(dt, TimeSpan.Zero);
```

#### 特殊日期获取

```csharp
DateTime today = DateTime.Today; // 今天的午夜时间
DateTime min = DateTime.MinValue;
DateTime max = DateTime.MaxValue;
DateTime unixEpoch = new DateTime(1970, 1, 1, 0, 0, 0, DateTimeKind.Utc);
```

### DateTimeKind 与时区处理

#### DateTimeKind 枚举

```csharp
public enum DateTimeKind
{
    Unspecified = 0, // 未指定时区（默认）
    Utc = 1,         // UTC时间
    Local = 2        // 本地时间
}
```

#### 时区转换最佳实践

```csharp
// 创建带时区信息的时间
DateTime utcTime = new DateTime(2023, 10, 1, 0, 0, 0, DateTimeKind.Utc);
DateTime localTime = new DateTime(2023, 10, 1, 8, 0, 0, DateTimeKind.Local);

// 时区转换
TimeZoneInfo tz = TimeZoneInfo.FindSystemTimeZoneById("China Standard Time");

// UTC转本地时间
DateTime utcToChina = TimeZoneInfo.ConvertTimeFromUtc(utcTime, tz);

// 本地时间转UTC
DateTime chinaToUtc = TimeZoneInfo.ConvertTimeToUtc(localTime, tz);

// 跨时区转换
TimeZoneInfo nyTz = TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time");
DateTime chinaToNy = TimeZoneInfo.ConvertTime(localTime, tz, nyTz);
```

### 格式化与解析

#### 标准格式化字符串

|  格式符   |  说明   |  示例输出   |
| --- | --- | --- |
| "d"    |  短日期   |  2023-06-15   |
|  "D"	   |  长日期   |  2023年6月15日   |
|  "t"   |  短时间   |  14:30   |
|  "T"   |  长时间   |  14:30:45   |
|  "f"   |  完整日期/时间	   |  2023年6月15日 14:30   |
|  "F"   |  完整格式   |  2023年6月15日 14:30:45   |
|  "g"   |  常规短格式   |  2023-06-15 14:30   |
|  "G"   |  常规长格式   |  2023-06-15 14:30:45   |
|  "s"   |  可排序格式   |  2023-06-15T14:30:45   |
|  "u"   |  通用可排序   |  2023-06-15 14:30:45Z   |
|  "U"   |  UTC完整格式   |  2023年6月15日 6:30:45   |

#### 格式化

* 标准格式

```csharp
dt.ToString("o");  // ISO 8601 完整格式：2025-08-08T14:30:00.0000000Z
dt.ToString("s");  // 可排序格式：2025-08-08T14:30:00
dt.ToString("G");  // 默认长日期+长时间
dt.ToString("d");       // "2023/10/1"（短日期）
dt.ToString("D");       // "2023年10月1日"（长日期）
dt.ToString("t");       // "14:30"（短时间）
dt.ToString("T");       // "14:30:00"（长时间）
```

* 自定义格式

```csharp
dt.ToString("yyyy-MM-dd HH:mm:ss");      // 2025-08-08 14:30:00
dt.ToString("yyyy-MM-dd HH:mm:ss.fff");  // 带毫秒
dt.ToString("dddd, MMM d yyyy");         // 星期几, 月 日 年
```

#### 解析

* `Parse / TryParse`

```csharp
DateTime.Parse("2025-08-08 14:30");
DateTime.TryParse("2025-08-08", out var dt);
```

* 指定格式解析

```csharp
DateTime.ParseExact("08/08/2025", "MM/dd/yyyy", CultureInfo.InvariantCulture);
DateTime.TryParseExact(input, new[] {"yyyy-MM-dd","MM/dd/yyyy"}, 
                       CultureInfo.InvariantCulture, DateTimeStyles.None, out dt);
```

### 日期时间组件访问

#### 属性访问

```csharp
DateTime dt = new DateTime(2023, 10, 5, 14, 30, 45, 123);

int year = dt.Year;       // 2023
int month = dt.Month;     // 10
int day = dt.Day;         // 5
int hour = dt.Hour;       // 14
int minute = dt.Minute;   // 30
int second = dt.Second;   // 45
int millisecond = dt.Millisecond; // 123
long ticks = dt.Ticks;    // 638,000,000,000,000,000 + ...

DayOfWeek dayOfWeek = dt.DayOfWeek; // DayOfWeek.Thursday
int dayOfYear = dt.DayOfYear;       // 278 (10月5日是2023年第278天)
```

#### 日期计算辅助

```csharp
DateTime dt = new DateTime(2023, 2, 15);

// 获取月份天数
int daysInFeb = DateTime.DaysInMonth(2023, 2); // 28

// 判断闰年
bool isLeap2023 = DateTime.IsLeapYear(2023); // false
bool isLeap2024 = DateTime.IsLeapYear(2024); // true

// 获取月份首日和末日
DateTime firstDayOfMonth = new DateTime(dt.Year, dt.Month, 1);
DateTime lastDayOfMonth = new DateTime(dt.Year, dt.Month, 
    DateTime.DaysInMonth(dt.Year, dt.Month));
```

### 日期时间运算与比较

#### 算术运算

```csharp
DateTime now = DateTime.Now;

// 加法
DateTime tomorrow = now.AddDays(1);
DateTime nextHour = now.AddHours(1);
DateTime nextMinute = now.AddMinutes(1);

// 减法
DateTime yesterday = now.AddDays(-1);
DateTime lastWeek = now.AddDays(-7);

// 时间跨度计算
DateTime start = new DateTime(2023, 1, 1);
DateTime end = new DateTime(2023, 12, 31);
TimeSpan duration = end - start; // 364天

// Ticks运算
DateTime preciseTime = now.AddTicks(50000); // 增加5毫秒
```

#### 比较操作

```csharp
DateTime dateA = new DateTime(2023, 6, 15);
DateTime dateB = new DateTime(2023, 6, 20);

// 比较方法
int compareResult = DateTime.Compare(dateA, dateB); 
// <0: dateA < dateB
// =0: 相等
// >0: dateA > dateB

// 运算符重载
bool isBefore = dateA < dateB; // true
bool isAfter = dateA > dateB;  // false
bool isSame = dateA == dateB;  // false

// 范围检查
bool isInRange = dateA >= start && dateA <= end;
```

### DateTime 与相关类型对比

| 特性     | DateTime  | DateTimeOffset | TimeOnly    | DateOnly  | NodaTime |
| -------- | --------- | -------------- | ----------- | --------- | -------- |
| 时区支持 | 弱        | 强（偏移量）   | 无          | 无        | 强       |
| 日期部分 | 需要计算  | 需要计算       | 无          | 有        | 有       |
| 时间部分 | 有        | 有             | 有          | 无        | 有       |
| 精度     | 100ns     | 100ns          | 100ns       | 天        | 1ns      |
| 序列化   | 有问题    | 安全           | 安全        | 安全      | 安全     |
| 范围     | 0001-9999 | 0001-9999      | 00:00-24:00 | 0001-9999 | 无限     |
| .NET版本 | 1.0+      | 2.0+           | 6.0+        | 6.0+      | NuGet包  |

### 最佳实践总结

#### 存储原则：

* 始终以 `UTC` 时间存储

* 仅在显示时转换为本地时间

#### 序列化准则：

```csharp
// 使用ISO 8601格式
DateTime.UtcNow.ToString("O"); // 2023-10-05T06:30:45.1234567Z
```

#### 时区处理：

* 使用 `DateTimeOffset` 替代 `DateTime`

* 复杂时区需求使用 `NodaTime` 或 `TimeZoneInfo`

#### 日期/时间分离：

```csharp
// .NET 6+ 推荐
DateOnly birthday = new DateOnly(1990, 5, 15);
TimeOnly meetingTime = new TimeOnly(14, 30);
```

#### 性能敏感场景：

* 避免重复调用 `DateTime.Now/UtcNow`

* 预计算静态日期值

* 高精度计时用 `Stopwatch`

### 资源和文档

* `DateTime`：https://learn.microsoft.com/en-us/dotnet/api/system.datetime

* `TimeZoneInfo`：https://learn.microsoft.com/en-us/dotnet/api/system.timezoneinfo

* `ASP.NET Core`：https://learn.microsoft.com/en-us/aspnet/core/