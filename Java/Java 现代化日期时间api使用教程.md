### 简介

在 `Java` 中，处理日期和时间对于许多应用程序都是必不可少的。`Java` 随着时间的推移而发展，随着 `Java 8` 的引入，引入了 `java.time` 包，为日期和时间操作提供了更现代、更全面的API。

### 旧版 java.util.Date 类（Java 8 之前）

在 `Java 8` 之前，`Java` 使用 `java.util.Date` 类来表示日期和时间。然而，它存在许多设计问题，并且不易于使用（例如，易变性、日期和时间之间的混淆）。

**使用java.util.Date的示例**

```java
import java.util.Date;

public class DateExample {
    public static void main(String[] args) {
        // Create a new Date object with the current date and time
        Date date = new Date();
        
        // Print the current date and time
        System.out.println("Current Date and Time: " + date);
        
        // Create a Date object with a specific time (milliseconds since epoch)
        Date specificDate = new Date(2025, 2, 7); // Deprecated way of creating Date (not recommended)
        System.out.println("Specific Date: " + specificDate);
        
        // Get the time in milliseconds since epoch
        long timeMillis = date.getTime();
        System.out.println("Milliseconds since epoch: " + timeMillis);
    }
}
```

### 现代 java.time 类（Java 8 及更高版本）

#### LocalDate – 表示不带时间的日期（年、月、日）

> 只需要日期（不需要时间或时区）时使用此类

```java
import java.time.LocalDate;

public class LocalDateExample {
    public static void main(String[] args) {
        // Current date
        LocalDate currentDate = LocalDate.now();
        System.out.println("Current Date: " + currentDate);
        
        // Specific date
        LocalDate specificDate = LocalDate.of(2025, 2, 7);
        System.out.println("Specific Date: " + specificDate);
        
        // Getting date information
        System.out.println("Year: " + currentDate.getYear());
        System.out.println("Month: " + currentDate.getMonth());
        System.out.println("Day of Month: " + currentDate.getDayOfMonth());
    }
}
```

* 获取当前日期并添加一天

```java
LocalDate tomorrow = LocalDate.now().plusDays(1);
```

* 获取当前日期并减去一个月

```java
LocalDate previousMonthSameDay = LocalDate.now().minus(1, ChronoUnit.MONTHS);
```

* 解析日期 `2016-06-12` 并分别获得星期几和第几个月份。返回值：第一个是一个表示 `DayOfWeek` 的对象，而第二个是一个表示月份序数的 `int`

```java
DayOfWeek sunday = LocalDate.parse("2016-06-12").getDayOfWeek();

int twelve = LocalDate.parse("2016-06-12").getDayOfMonth();
```

* 测试某个日期是否是闰年

```java
boolean leapYear = LocalDate.now().isLeapYear();
```

* 确定一个日期与另一个日期的关系是发生在另一个日期之前还是之后

```java
boolean notBefore = LocalDate.parse("2016-06-12")
  .isBefore(LocalDate.parse("2016-06-11"));

boolean isAfter = LocalDate.parse("2016-06-12")
  .isAfter(LocalDate.parse("2016-06-11"));
```

* 可以从给定日期获得日期边界

```java
LocalDateTime beginningOfDay = LocalDate.parse("2016-06-12").atStartOfDay();
LocalDate firstDayOfMonth = LocalDate.parse("2016-06-12")
  .with(TemporalAdjusters.firstDayOfMonth());
```

#### LocalTime – 表示不带日期或时区的时间（小时、分钟、秒、纳秒）

```java
import java.time.LocalTime;

public class LocalTimeExample {
    public static void main(String[] args) {
        // Current time
        LocalTime currentTime = LocalTime.now();
        System.out.println("Current Time: " + currentTime);
        
        // Specific time
        LocalTime specificTime = LocalTime.of(14, 30, 15);
        System.out.println("Specific Time: " + specificTime);
        
        // Getting time information
        System.out.println("Hour: " + currentTime.getHour());
        System.out.println("Minute: " + currentTime.getMinute());
        System.out.println("Second: " + currentTime.getSecond());
    }
}
```

* 解析字符串创建 `LocalTime`

```java
LocalTime sixThirty = LocalTime.parse("06:30");
```

* 工厂方法也可用于创建 `LocalTime`

```java
LocalTime sixThirty = LocalTime.of(6, 30);
```

* 联合 `plus` 方法一起使用

```java
LocalTime sevenThirty = LocalTime.parse("06:30").plus(1, ChronoUnit.HOURS);
```

* 获取小时数

```java
int six = LocalTime.parse("06:30").getHour();
```

* 检查特定时间是在另一个特定时间之前还是之后

```java
boolean isbefore = LocalTime.parse("06:30").isBefore(LocalTime.parse("07:30"));
```

* 通过 `LocalTime` 类中的常量获取一天的最大、最小和中午时间

```java
LocalTime maxTime = LocalTime.MAX

// 输出： 23:59:59.99
```

#### LocalDateTime – 表示日期和时间，但不包含时区

```java
import java.time.LocalDateTime;

public class LocalDateTimeExample {
    public static void main(String[] args) {
        // Current date and time
        LocalDateTime currentDateTime = LocalDateTime.now();
        System.out.println("Current Date and Time: " + currentDateTime);
        
        // Specific date and time
        LocalDateTime specificDateTime = LocalDateTime.of(2025, 2, 7, 14, 30);
        System.out.println("Specific Date and Time: " + specificDateTime);
    }
}
```

* 使用 `parse` 解析字符串创建日期时间

```java
LocalDateTime.parse("2015-02-20T06:30:00");
```

* `plusDays` 添加天数

```java
localDateTime.plusDays(1);
```

* `minusHours` 减小时数

```java
localDateTime.minusHours(2);
```

* 获取月份

```java
localDateTime.getMonth();
```

### ZonedDateTime – 表示带时区的日期和时间

> 当需要考虑时区（例如 UTC、PST 等）时使用此类

```java
import java.time.ZonedDateTime;
import java.time.ZoneId;

public class ZonedDateTimeExample {
    public static void main(String[] args) {
        // Current date and time in a specific timezone
        ZonedDateTime zonedDateTime = ZonedDateTime.now(ZoneId.of("America/New_York"));
        System.out.println("Current Date and Time in New York: " + zonedDateTime);
        
        // Specific date and time in a timezone
        ZonedDateTime specificZonedDateTime = ZonedDateTime.of(2025, 2, 7, 14, 30, 0, 0, ZoneId.of("Europe/London"));
        System.out.println("Specific Date and Time in London: " + specificZonedDateTime);
    }
}
```

* 获取所有可用的时区

```java
Set<String> allZoneIds = ZoneId.getAvailableZoneIds();
```

* 使用 `parse` 方法解析特定时区的日期时间

```java
ZonedDateTime.parse("2015-05-03T10:15:30+01:00[Europe/Paris]");
```

#### Instant – 表示 UTC 中的某个时间点（时间戳）

> 此类用于时间戳，即自 Unix 纪元 (1970-01-01T00:00:00Z) 以来的秒数或毫秒数

```java
import java.time.Instant;

public class InstantExample {
    public static void main(String[] args) {
        // Current timestamp
        Instant currentInstant = Instant.now();
        System.out.println("Current Timestamp: " + currentInstant);
        
        // Convert Instant to milliseconds since epoch
        long millisecondsSinceEpoch = currentInstant.toEpochMilli();
        System.out.println("Milliseconds since epoch: " + millisecondsSinceEpoch);
    }
}
```

#### Duration 和 Period - 测量两个 java.time 对象之间的时间

* `Duration`：用于以秒和纳秒为单位测量时间

* `Period`：用于测量年、月、日的时间

```java
import java.time.Duration;
import java.time.LocalDateTime;

public class DurationExample {
    public static void main(String[] args) {
        LocalDateTime startTime = LocalDateTime.of(2025, 2, 7, 14, 0, 0);
        LocalDateTime endTime = LocalDateTime.of(2025, 2, 7, 16, 0, 0);
        
        Duration duration = Duration.between(startTime, endTime);
        System.out.println("Duration: " + duration.toHours() + " hours");
    }
}
```

* `Period` 的用法

```java
int five = Period.between(initialDate, finalDate).getDays();
```

#### 格式化和解析日期

> java.time.format.DateTimeFormatter 类用于格式化和解析日期和时间

```java
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

public class DateFormatExample {
    public static void main(String[] args) {
        // Formatting
        LocalDate date = LocalDate.now();
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
        String formattedDate = date.format(formatter);
        System.out.println("Formatted Date: " + formattedDate);
        
        // Parsing
        String dateString = "2025-02-07";
        LocalDate parsedDate = LocalDate.parse(dateString, formatter);
        System.out.println("Parsed Date: " + parsedDate);
    }
}
```

#### 在传统日期和 java.time 之间转换

**将 java.util.Date 转换为 java.time.LocalDate**

```java
import java.util.Date;
import java.time.LocalDate;
import java.time.ZoneId;

public class ConvertDateExample {
    public static void main(String[] args) {
        Date legacyDate = new Date();
        LocalDate localDate = legacyDate.toInstant().atZone(ZoneId.systemDefault()).toLocalDate();
        System.out.println("Converted LocalDate: " + localDate);
    }
}
```

**将 java.time.LocalDate 转换为 java.util.Date**

```java
import java.time.LocalDate;
import java.time.ZoneId;
import java.util.Date;

public class ConvertLocalDateExample {
    public static void main(String[] args) {
        LocalDate localDate = LocalDate.now();
        Date legacyDate = Date.from(localDate.atStartOfDay(ZoneId.systemDefault()).toInstant());
        System.out.println("Converted java.util.Date: " + legacyDate);
    }
}
```