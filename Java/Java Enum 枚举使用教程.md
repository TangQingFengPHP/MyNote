### 简介

`Java` 枚举（`enum`）是 `Java 5` 引入的一种特殊类，用于表示一组固定的常量（如状态、类型等）。它结合了类型安全、代码可读性和面向对象特性，广泛应用于 `Java` 项目中（如 `Spring Boot`、`MyBatis Plus` 等）

###  枚举的核心概念

* 所有枚举实例必须在类首行显式声明，且默认被 `public static final` 修饰 

* 继承自 `java.lang.Enum` 类，无法被其他类继承 

* 构造器强制私有化，防止外部实例化 

### 基础：定义与使用

```java
public enum Status {
    SUCCESS, // 枚举实例（本质是 public static final）
    FAILURE,
    PENDING
}
```

使用：

```java
Status status = Status.SUCCESS;
System.out.println(status);          // SUCCESS
System.out.println(status.name());   // SUCCESS
System.out.println(status.ordinal()); // 0

// 遍历枚举
for (Status status : Status.values()) {
    System.out.println(status);
}
```

#### values() 方法 

`values()` 方法会返回一个包含枚举类型所有常量的数组。

```java
enum Weekday {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}

public class EnumValuesExample {
    public static void main(String[] args) {
        Weekday[] allWeekdays = Weekday.values();
        for (Weekday weekday : allWeekdays) {
            System.out.println(weekday);
        }
    }
}
```

#### ordinal() 方法

`ordinal()` 方法返回枚举常量在枚举类型中的索引，索引从 `0` 开始。

```java
enum Planet {
    MERCURY, VENUS, EARTH, MARS, JUPITER, SATURN, URANUS, NEPTUNE
}

public class EnumOrdinalExample {
    public static void main(String[] args) {
        Planet earth = Planet.EARTH;
        System.out.println("地球的索引是: " + earth.ordinal());
    }
}
```

#### valueOf() 方法

`valueOf()` 方法可以根据枚举常量的名称返回对应的枚举常量

```java
enum Color {
    RED, GREEN, BLUE
}

public class EnumValueOfExample {
    public static void main(String[] args) {
        Color green = Color.valueOf("GREEN");
        System.out.println("获取到的颜色是: " + green);
    }
}
```  

### 自定义字段与方法

```java
public enum ResultCode {
    SUCCESS(200, "成功"),
    FAILURE(500, "失败");

    private final int code;
    private final String message;

    ResultCode(int code, String message) {
        this.code = code;
        this.message = message;
    }

    public int getCode() { return code; }
    public String getMessage() { return message; }
}
```

使用：

```java
System.out.println(ResultCode.SUCCESS.getCode());     // 200
System.out.println(ResultCode.FAILURE.getMessage());  // 失败
```

### 枚举实现接口

```java
public interface PriceCalculator {
    double calculatePrice();
}

public enum Coffee implements PriceCalculator {
    ESPRESSO(2.5) {
        @Override
        public double calculatePrice() {
            return basePrice * 1.2;
        }
    },
    LATTE(3.0) {
        @Override
        public double calculatePrice() {
            return basePrice * 1.5;
        }
    };

    protected final double basePrice;

    Coffee(double basePrice) {
        this.basePrice = basePrice;
    }
}
```

### 设计模式应用

#### 枚举中定义抽象方法（策略模式）

```java
public enum Operation {
    PLUS {
        public int apply(int x, int y) { return x + y; }
    },
    MINUS {
        public int apply(int x, int y) { return x - y; }
    };

    public abstract int apply(int x, int y);
}
```

使用：

```java
int result = Operation.PLUS.apply(5, 3); // 8
```

#### 状态机实现

```java
public enum OrderState {
    CREATED {
        @Override
        public OrderState nextState() {
            return PAID;
        }
    },
    PAID {
        @Override
        public OrderState nextState() {
            return SHIPPED;
        }
    };
    SHIPPED {
        @Override
        public OrderState nextState() {
            throw new IllegalStateException("Order has already been shipped.");
        }
    };

    public abstract OrderState nextState();
}
```

### 根据 code 值反查枚举

```java
public static ResultCode fromCode(int code) {
    for (ResultCode rc : ResultCode.values()) {
        if (rc.getCode() == code) {
            return rc;
        }
    }
    throw new IllegalArgumentException("未知code: " + code);
}
```

### 枚举实现单例模式

枚举天然线程安全且防反射攻击，是实现单例的最佳方式：

```java
public enum Singleton {
    INSTANCE;

    private int counter = 0;

    public void increment() {
        counter++;
    }

    public int getCount() {
        return counter;
    }
}

// 使用
Singleton.INSTANCE.increment();
System.out.println(Singleton.INSTANCE.getCount());
```

### 枚举工具类

#### EnumSet 高效存储枚举集合

```java
EnumSet<Day> weekend = EnumSet.of(Day.SATURDAY, Day.SUNDAY);
if (weekend.contains(today)) {
    System.out.println("休息日");
}
```

#### EnumMap 键为枚举的专用 Map

```java
EnumMap<Day, String> schedule = new EnumMap<>(Day.class);
schedule.put(Day.MONDAY, "会议");
String task = schedule.get(Day.MONDAY); // "会议"
```

### 枚举在 switch 语句中的使用

枚举非常适合在 `switch` 语句中使用，因为枚举常量是固定的，编译器可以进行类型检查。

```java
enum Day {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}

public class EnumSwitchExample {
    public static void main(String[] args) {
        Day today = Day.MONDAY;
        switch (today) {
            case MONDAY:
                System.out.println("今天是周一，开始新的一周工作");
                break;
            case FRIDAY:
                System.out.println("今天是周五，快到周末啦");
                break;
            case SATURDAY:
            case SUNDAY:
                System.out.println("今天是周末，可以好好休息");
                break;
            default:
                System.out.println("工作日继续加油");
        }
    }
}
```

### 数据库映射

```java
@Converter(autoApply = true)
public class StatusConverter implements AttributeConverter<Status, String> {
    @Override
    public String convertToDatabaseColumn(Status status) {
        return status.getCode();
    }

    @Override
    public Status convertToEntityAttribute(String code) {
        return Status.fromCode(code);
    }
}
```

### 枚举反射操作

```java
// 反射获取枚举信息
Class<Day> enumClass = Day.class;
if (enumClass.isEnum()) {
    for (Field field : enumClass.getDeclaredFields()) {
        if (field.isEnumConstant()) {
            System.out.println("枚举常量: " + field.getName());
        }
    }
}
```

### 枚举的底层原理

枚举会被编译为继承 `java.lang.Enum` 的 `final` 类：

```java
// 反编译 Day.class
public final class Day extends Enum<Day> {
    public static final Day MONDAY = new Day("MONDAY", 0);
    // 其他枚举实例...
    private static final Day[] VALUES = values();

    private Day(String name, int ordinal) {
        super(name, ordinal);
    }

    public static Day[] values() { /* 返回克隆数组 */ }
    public static Day valueOf(String name) { /* 查找实例 */ }
}
```
