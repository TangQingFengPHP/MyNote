### 简介

`Stream` 是 `Java 8` 引入的一个 函数式编程特性，可以让我们用声明式的方式操作集合（如 `List、Set、Map` 等）。

核心作用是：

* 从集合中提取数据（流）

* 对数据做中间操作（`filter/map/sort...`）

* 最后做终端操作（`forEach/collect/count...`）

### Stream 基础结构

```java
collection.stream()
    .filter(...)       // 中间操作
    .map(...)          // 中间操作
    .collect(...)      // 终结操作
```

### 创建 Stream 的方式

```java
// 从集合创建
List<String> list = Arrays.asList("A", "B", "C");
Stream<String> stream1 = list.stream();

// 从数组创建
Stream<Integer> stream2 = Arrays.stream(new Integer[]{1, 2, 3});

// 使用 Stream.of()
Stream<String> stream3 = Stream.of("X", "Y", "Z");

// 生成无限流（需限制）
Stream<Integer> infiniteStream = Stream.iterate(0, n -> n + 2).limit(10);

// 数值范围
IntStream.range(1, 5);       // 生成 1,2,3,4
IntStream.rangeClosed(1,5);  // 生成 1,2,3,4,5

// 文件生成
Stream<String> lines = Files.lines(Paths.get("data.txt")); 

// Builder 构建
Stream<String> customStream = Stream.<String>builder()
    .add("Apple").add("Banana").build();
```

### 常用中间操作（返回 Stream）

* `filter(Predicate)`：条件过滤

```java
list.stream().filter(s -> s.startsWith("A"))
```

* `map(Function)`：映射/转换

```java
list.stream().map(String::toUpperCase)
```

* `flatMap(Function)`：拍平嵌套结构

```java
list.stream().flatMap(List::stream)
```

* `distinct()`：去重

```java
stream.distinct()
```

* `sorted()`：排序

```java
stream.sorted(Comparator.reverseOrder())
```

* `limit(n)`：取前 `n` 条

* `skip(n)`：跳过前 `n` 条

* `peek(Consumer)`：调试用，查看中间结果

```java
stream.peek(System.out::println)
```

示例

```java
List<String> result = list.stream()
    .filter(s -> s.startsWith("A"))
    .map(String::toLowerCase)
    .distinct()
    .collect(Collectors.toList());
```

### 常用终止操作（返回非 Stream）

* `collect(Collector)`：收集为集合、字符串等

```java
stream.collect(Collectors.toList())
```

* `forEach(Consumer)`：遍历每个元素

```java
stream.forEach(System.out::println)
```

* `count()`：统计数量

```java
stream.count()
```

* `anyMatch(Predicate)`：任一匹配

```java
stream.anyMatch(s -> s.contains("a"))
```

* `allMatch()`：全部匹配

* `noneMatch()`：都不匹配

* `findFirst()`：找第一个元素

```java
stream.findFirst()
```

* `findAny()`：找任意元素（并行时更快）

* `reduce()`：规约合并（累加、乘法等）

```java
stream.reduce(0, Integer::sum)
```

示例

```java
long count = list.stream().filter(s -> s.length() > 3).count();

Optional<String> any = list.stream().findAny();

String joined = list.stream().collect(Collectors.joining(", "));
```

### 收集器 Collectors 工具类

```java
List<String> names = people.stream()
    .map(Person::getName)
    .collect(Collectors.toList());

Set<String> set = list.stream().collect(Collectors.toSet());

Map<String, Integer> map = people.stream()
    .collect(Collectors.toMap(Person::getName, Person::getAge));

String result = list.stream().collect(Collectors.joining(", "));

double avg = people.stream().collect(Collectors.averagingInt(Person::getAge));
```

### 分组 & 分区

```java
// 分组
Map<String, List<Person>> groupByDept = people.stream()
    .collect(Collectors.groupingBy(Person::getDepartment));

// 多级分组
Map<String, Map<Integer, List<Person>>> complexGroup =
    people.stream().collect(Collectors.groupingBy(Person::getDept, 
        Collectors.groupingBy(Person::getAge)));

// 分区（true/false 分两组）
Map<Boolean, List<Person>> partition = people.stream()
    .collect(Collectors.partitioningBy(p -> p.getAge() > 30));
```

### 排序（sorted）

```java
// 自然排序
list.stream().sorted().forEach(System.out::println);

// 自定义排序
list.stream()
    .sorted((a, b) -> a.length() - b.length())
    .forEach(System.out::println);

// 对对象排序
people.stream()
    .sorted(Comparator.comparing(Person::getAge).reversed())
    .forEach(System.out::println);
```

### flatMap 的典型应用

```java
List<String> lines = Arrays.asList("A B", "C D");
List<String> words = lines.stream()
    .flatMap(line -> Arrays.stream(line.split(" ")))
    .collect(Collectors.toList());
```

### 并行流（parallelStream）

```java
list.parallelStream().forEach(System.out::println);
```

### reduce 规约操作

```java
int sum = Arrays.asList(1, 2, 3, 4).stream()
    .reduce(0, Integer::sum); // 初始值 0，累加求和
```

### 应用案例

#### 数据过滤与筛选

在处理大量数据时，常常需要依据特定条件筛选出符合要求的数据。

```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

public class DataFiltering {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        // 筛选出所有偶数
        List<Integer> evenNumbers = numbers.stream()
                .filter(n -> n % 2 == 0)
                .collect(Collectors.toList());
        System.out.println("偶数列表: " + evenNumbers);
    }
}
```

#### 数据映射与转换

有时候需要把集合中的元素转换为其他类型或格式。

```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }
}

public class DataMapping {
    public static void main(String[] args) {
        List<Person> people = Arrays.asList(
                new Person("Alice", 25),
                new Person("Bob", 30),
                new Person("Charlie", 35)
        );
        // 将 Person 对象转换为他们的名字列表
        List<String> names = people.stream()
                .map(Person::getName)
                .collect(Collectors.toList());
        System.out.println("名字列表: " + names);
    }
}
```

运用 `map` 中间操作，把 `Person` 对象列表转换为包含每个人名字的字符串列表。

#### 数据排序

利用 `Streams` 对集合中的元素进行排序。

```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

public class DataSorting {
    public static void main(String[] args) {
        List<String> words = Arrays.asList("banana", "apple", "cherry");
        // 按字母顺序排序
        List<String> sortedWords = words.stream()
                .sorted()
                .collect(Collectors.toList());
        System.out.println("排序后的单词列表: " + sortedWords);
    }
}
```

使用 `sorted` 中间操作对字符串列表按字母顺序进行排序。

#### 数据统计

`Stream` 提供了一些方法用于统计数据，如求和、平均值、最大值、最小值等。

```java
import java.util.Arrays;
import java.util.IntSummaryStatistics;
import java.util.List;

public class DataStatistics {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
        IntSummaryStatistics stats = numbers.stream()
                .mapToInt(Integer::intValue)
                .summaryStatistics();
        System.out.println("总和: " + stats.getSum());
        System.out.println("平均值: " + stats.getAverage());
        System.out.println("最大值: " + stats.getMax());
        System.out.println("最小值: " + stats.getMin());
    }
}
```

通过 `summaryStatistics` 方法，能够获取整数列表的总和、平均值、最大值和最小值等统计信息。

#### 分组与分区

可以按照特定条件对数据进行分组或分区。

```java
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

class Product {
    private String name;
    private double price;

    public Product(String name, double price) {
        this.name = name;
        this.price = price;
    }

    public String getName() {
        return name;
    }

    public double getPrice() {
        return price;
    }
}

public class GroupingAndPartitioning {
    public static void main(String[] args) {
        List<Product> products = Arrays.asList(
                new Product("Apple", 1.5),
                new Product("Banana", 0.5),
                new Product("Cherry", 2.0),
                new Product("Date", 0.8)
        );
        // 按价格是否大于 1 进行分区
        Map<Boolean, List<Product>> partitionedByPrice = products.stream()
                .collect(Collectors.partitioningBy(p -> p.getPrice() > 1));
        System.out.println("价格大于 1 的产品: " + partitionedByPrice.get(true));
        System.out.println("价格小于等于 1 的产品: " + partitionedByPrice.get(false));
    }
}
```

### 最佳实践

* 避免副作用：`Stream` 操作应是无状态的，避免修改外部变量。

* 优先使用方法引用：使代码更简洁（如 `String::length`）。

* 谨慎使用并行流：根据数据量和操作复杂度评估是否使用。

* 链式操作顺序优化：将过滤操作（`filter`）放在前面，减少后续处理的数据量。

### 常见问题

* `Stream` 不会修改源数据，操作是惰性的。

* `Stream` 只能使用一次，一旦执行终端操作，流就被消费，不可重复使用。

* 在 `Lambda` 中处理 `Stream` 异常，或使用 `try-catch` 包裹终端操作。

### Java Stream 与 C# LINQ

#### 核心目标一致

|  特性   |  `Java Stream API`   |  `C# LINQ`   |
| --- | --- | --- |
|  面向语言   |  Java 8+   |  C# 3.0+   |
|  编程范式   |  函数式编程   |  集合查询式 + 函数式编程   |
|  处理方式   |  面向流（Stream）处理   |  面向集合（Enumerable/IQueryable）处理   |
|  核心思想   |  用流水线的方式处理集合   |  像 SQL 一样写集合操作   |


#### 语法对比

基础例子：从字符串列表中过滤出以 A 开头的字母，转成小写后收集

* Java Stream：

```java
List<String> result = list.stream()
    .filter(s -> s.startsWith("A"))
    .map(String::toLowerCase)
    .collect(Collectors.toList());
```

* C# LINQ：

```csharp
List<string> result = list
    .Where(s => s.StartsWith("A"))
    .Select(s => s.ToLower())
    .ToList();
```

对比说明：两者几乎一致，`Java` 是 `stream()` 后链接操作，`C#` 是直接链式调用。

#### 常用操作对应表

|  功能  |  Java Stream   |  C# LINQ   |
| --- | --- | --- |
|  过滤   |  filter(Predicate)   |  Where(Func<T, bool>)   |
|  映射   |  map(Function)   |  Select(Func<T, TResult>)   |
|  拍平   |  flatMap(Function)   |  SelectMany()   |
|  排序   |  sorted() / Comparator   |  OrderBy() / ThenBy()   |
|  去重   |  distinct()   |  Distinct()   |
|  计数   |  count()   |  Count()   |
|  取前n条   |  limit(n)   |  Take(n)   |
|  跳过前n条   |  skip(n)   |  Skip(n)   |
|  查找元素   |  findFirst() / findAny()   |  FirstOrDefault() / First()   |
|  是否匹配   |  anyMatch() / allMatch()   |  Any() / All()   |
|  聚合   |  reduce()   |  Aggregate()   |
|  收集   |  collect(Collectors)   |  ToList() / ToDictionary()   |
|  分组   |  Collectors.groupingBy()   |  GroupBy()   |
|  分区   |  Collectors.partitioningBy()   |  GroupBy(bool) + ToLookup()   |
|  遍历   |  forEach()   |  foreach 或 .ForEach()（List 扩展）   |

#### 使用方式差异

|  特性   |  Java Stream   |  C# LINQ   |
| --- | --- | --- |
|  是否懒加载   |  是，中间操作不执行直到终止操作   |  是，延迟执行   |
|  多线程   |  支持 `.parallelStream()`（需小心）   |  可用 `PLINQ（Parallel LINQ）`并行处理   |
|  `SQL` 风格语法   |  ❌ 不支持   |  ✅ 支持 `from ... where ... select` 查询语法   |
|  集合类型支持   |  Collection、数组、Map 等   |  IEnumerable、IQueryable、List、Array 等   |
|  返回类型   |  Stream → collect 后得集合   |  LINQ 直接链式调用后转集合   |

#### 示例对比：复杂操作

* Java

```java
Map<String, Map<Integer, List<Person>>> group =
    people.stream().collect(Collectors.groupingBy(
        Person::getDept,
        Collectors.groupingBy(Person::getAge)
    ));
```

* C#

```csharp
var group = people
    .GroupBy(p => p.Dept)
    .ToDictionary(
        g => g.Key,
        g => g.GroupBy(p => p.Age).ToDictionary(gg => gg.Key, gg => gg.ToList())
    );
```

#### 并行处理实现

* Java Streams：

通过 `parallelStream()` 或 `stream().parallel()` 快速启用并行流，但需注意线程安全问题。

```java
list.parallelStream().forEach(s -> process(s)); // 自动分配线程
```

* C# PLINQ：

通过 `AsParallel()` 启用并行查询，可自定义并行度。

```csharp
list.AsParallel().WithDegreeOfParallelism(4).ForAll(s => Process(s));
```

#### 空值处理

* Java Streams：

默认不支持 `null` 元素（可能抛出 `NullPointerException`），需显式处理。

* C# LINQ：

允许集合中包含 `null`，但某些操作（如 `First()`）可能需处理空值。

#### 总结对比

|  维度   |  Java Stream API   |  C# LINQ   |
| --- | --- | --- |
|  可读性   |  简洁，但不支持 SQL 风格   |  支持 SQL 风格，阅读更直观   |
|  灵活性   |  借助 `Collectors` 可以做很多操作   |  `LINQ` 本身功能更丰富   |
|  多线程处理   |  `.parallelStream()`（粗粒度）   |  `PLINQ`（细粒度）   |
|  数据源支持   |  Java 集合体系   |  .NET 集合体系 + 数据库 IQueryable   |
|  底层机制   |  基于中间操作链和终结操作   | 基于延迟计算迭代器   |
|  功能扩展性   |  使用 `Collectors` 辅助函数   |  使用 `LINQ Extension Methods`   |
|  框架集成度   |  和 Spring/Java EE 无缝结合   |  和 Entity Framework、ASP.NET 配合良好   |