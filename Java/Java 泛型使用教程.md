### 简介

`Java` 泛型是 `JDK 5` 引入的一项特性，它提供了编译时类型安全检测机制，允许在编译时检测出非法的类型。泛型的本质是参数化类型，也就是说所操作的数据类型被指定为一个参数。

泛型的好处：

* 编译期检查类型安全

* 避免强制类型转换（`cast`）

* 代码更通用，更易重用

### 泛型的基本使用

#### 泛型类

```java
public class Box<T> {
    private T content;

    public void set(T content) {
        this.content = content;
    }

    public T get() {
        return content;
    }
}
```

使用：

```java
Box<String> stringBox = new Box<>();
stringBox.set("Hello");
String value = stringBox.get(); // 无需强转
```

#### 泛型方法

```java
public class Util {
    public static <T> void printArray(T[] array) {
        for (T element : array) {
            System.out.println(element);
        }
    }
}
```

使用：

```java
String[] arr = {"A", "B", "C"};
Util.printArray(arr);
```

#### 泛型接口

```java
public interface Converter<F, T> {
    T convert(F from);
}

public class StringToIntegerConverter implements Converter<String, Integer> {
    public Integer convert(String from) {
        return Integer.parseInt(from);
    }
}
```

### 泛型中的通配符：?

#### ? 通配符

表示任意类型：

```java
public void printList(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}
```

#### ? extends T（上界通配符）

表示 `T` 或 `T` 的子类（只读，不能写）：

```java
public void printNumbers(List<? extends Number> list) {
    for (Number num : list) {
        System.out.println(num);
    }
}
```

不能添加元素：

```java
list.add(10); // ❌ 报错
```

#### ? super T（下界通配符）

表示 `T` 或 `T` 的父类（可以写入，但取出只能当作 `Object`）：

```java
public void addNumbers(List<? super Integer> list) {
    list.add(10); // ✅ OK
}
```

### 类型擦除（Type Erasure）

泛型在编译后会被擦除，`JVM` 不知道泛型类型，全部变成 `Object`：

```java
List<String> list = new ArrayList<>();
List<Integer> list2 = new ArrayList<>();
System.out.println(list.getClass() == list2.getClass()); // true
```

正因如此：

* 不能使用 `new T()`

* 不能使用 `T.class`

* 不能判断 `instanceof T`

### 常见的泛型类库例子

* `List<T>`：存储 T 类型的集合

* `Map<K, V>`：泛型键值对

* `Optional<T>`：包装返回值

* `Comparable<T>`：排序比较接口

* `Function<T, R>`：函数式接口

* `Callable<T>`：异步任务的返回类型

* `Future<T>`：异步计算的结果

### 进阶技巧

#### 泛型数组不允许：

```java
List<String>[] lists = new List<String>[10]; // ❌ 编译错误
```

#### 泛型静态方法必须声明 `<T>`：

```java
public static <T> void print(T value) {
    System.out.println(value);
}
```

### 实际项目中 Java 泛型的用法大全

#### 通用返回封装类（统一 API 响应格式）

```java
public class Result<T> {
    private int code;
    private String message;
    private T data;

    public Result(int code, String message, T data) {
        this.code = code;
        this.message = message;
        this.data = data;
    }

    public static <T> Result<T> success(T data) {
        return new Result<>(200, "Success", data);
    }

    public static <T> Result<T> fail(String message) {
        return new Result<>(500, message, null);
    }

    // getter/setter omitted
}
```

#### 通用分页结果类

```java
public class PageResult<T> {
    private List<T> records;
    private long total;

    public PageResult(List<T> records, long total) {
        this.records = records;
        this.total = total;
    }
}
```

### 结合 Spring 使用泛型的示例

#### 通用 Service 接口

```java
public interface BaseService<T, ID> {
    T findById(ID id);
    List<T> findAll();
    T save(T entity);
    void deleteById(ID id);
}
```

#### 抽象 Service 实现类

```java
public abstract class AbstractBaseService<T, ID> implements BaseService<T, ID> {

    @Autowired
    protected JpaRepository<T, ID> repository;

    @Override
    public T findById(ID id) {
        return repository.findById(id).orElse(null);
    }

    @Override
    public List<T> findAll() {
        return repository.findAll();
    }

    @Override
    public T save(T entity) {
        return repository.save(entity);
    }

    @Override
    public void deleteById(ID id) {
        repository.deleteById(id);
    }
}
```

#### 具体 Service

```java
@Service
public class UserService extends AbstractBaseService<User, Long> {
    // 可以扩展额外业务逻辑
}
```

### Java 泛型 与 C#.net 泛型比较

`Java` 泛型 与 `C# (.NET)` 泛型有很多相似之处，`C#` 泛型的设计部分参考了 `Java`。但它们在类型擦除、协变/逆变、约束、运行时行为等方面有显著的不同。


|  特性   |  Java 泛型   |  C# (.NET) 泛型   |
| --- | --- | --- |
|  类型擦除   |  是（编译期擦除）   |  否（保留类型信息）   |
|  运行时可反射获取泛型类型   |  否   |  是   |
|  基本类型支持   |  不直接支持（需使用包装类如 `Integer`）   |  支持，例如 `List<int>`   |
|  泛型约束   |  限制（只能 `extends`，不支持多个约束）   |  强大（支持 `where T : class, new()`, 多个接口）   |
|  协变/逆变支持   |  通过通配符 `? extends/super`   |  直接使用 `out/in` 关键字   |
|  泛型数组   |  不支持（如 `new T[]` 编译错误）   |  支持   |
|  泛型方法   |  支持   |  支持   |

#### 类型擦除 vs 保留类型

* Java

```java
List<String> list = new ArrayList<>();
List<Integer> intList = new ArrayList<>();
System.out.println(list.getClass() == intList.getClass()); // true
```

`Java` 在编译时擦除了泛型信息，`List<String>` 和 `List<Integer>` 其实是同一个字节码类。

* C#

```csharp
List<string> list = new List<string>();
List<int> intList = new List<int>();
Console.WriteLine(list.GetType() == intList.GetType()); // false
```

`C#` 会为不同泛型参数生成不同的类实例，因此保留类型信息。

#### 泛型约束

* Java

```java
public class Repository<T extends BaseEntity> {
    public void save(T entity) { }
}
```

* C#

```csharp
public class Repository<T> where T : BaseEntity, new() {
    public void Save(T entity) { }
}
```

`C#` 支持更丰富的泛型约束，例如要求是引用类型 `class`、值类型 `struct`、必须有无参构造函数 `new()` 等。

#### 协变与逆变（Covariance & Contravariance）

* Java 使用通配符

```java
List<? extends Number> numbers;  // 只能读取，不能添加
List<? super Integer> integers;  // 只能添加 Integer
```

* C# 使用 in / out

```csharp
interface ICovariant<out T> { }
interface IContravariant<in T> { }
```

`C#` 在接口中用 `in/out` 明确支持协变逆变，且更强大、更类型安全。

#### 泛型数组

* Java

```java
T[] array = new T[10]; // 编译错误
```

* C#

```csharp
T[] array = new T[10]; // 合法
```

#### 基本类型泛型

* Java

```java
List<int> list = new ArrayList<>(); // 编译错误
List<Integer> list = new ArrayList<>(); // 正确
```

* C#

```csharp
List<int> list = new List<int>(); // 正确，泛型支持值类型
```

#### 总结

|  特性   |  Java   |  C#   |
| --- | --- | --- |
|  类型安全   |  ✔️   |  ✔️   |
|  灵活性   |  ❌（类型擦除限制）   |  ✔️（运行时保留泛型）   |
|  泛型数组   |  ❌   |  ✔️   |
|  基本类型支持   |  ❌（需包装）   |  ✔️   |
|  泛型约束   |  一般   |  强大   |
|  协变逆变   |  复杂、通配符语法   |  简洁、原生支持   |
|  性能   |  需装箱   |  无装箱（对值类型更快）   |

### 协变逆变详解

协变（Covariance）和逆变（Contravariance）是泛型类型系统中用于处理子类和父类之间的泛型关系的重要机制。

#### Java 的协变（Covariant）与逆变（Contravariant）

`Java` 使用 通配符（`wildcards`） 来支持：

> 协变（`? extends T`） — 只读

* 意思是：某个未知类型是 `T` 的子类

* 常用于“只能读”的场景

```java
public void readAnimals(List<? extends Animal> animals) {
    for (Animal animal : animals) {
        System.out.println(animal);
    }
}
```

可以传入 `List<Cat>、List<Dog> 或 List<Animal>`，但不能添加新元素。

```java
animals.add(new Cat()); // ❌ 编译错误
```

> 逆变（? super T） — 只写

* 意思是：某个未知类型是 `T` 的父类

* 常用于“只能写”的场景

```java
public void addCats(List<? super Cat> cats) {
    cats.add(new Cat());     // ✅ 合法
    // cats.add(new Animal()); // ❌ 不安全
}
```

可以传入 `List<Cat>、List<Animal>、List<Object>`，但不能安全地读取具体类型：

```java
Object obj = cats.get(0); // 只能作为 Object 使用
```

#### C# 中的协变与逆变

* C# 协变（out）

```csharp
interface IReadOnlyList<out T> {
    T Get(int index);
}

IReadOnlyList<Animal> animals = new List<Cat>(); // ✅ 协变成功
```

* C# 逆变（in）

```csharp
interface IWriter<in T> {
    void Write(T value);
}

IWriter<Cat> writer = new AnimalWriter(); // ✅ 逆变成功
```

#### 完整示例：协变 & 逆变

```java
import java.util.*;

class Animal { }
class Cat extends Animal { }
class Dog extends Animal { }

public class VarianceDemo {

    public static void main(String[] args) {
        List<Cat> cats = new ArrayList<>();
        cats.add(new Cat());

        readAnimals(cats); // ✅ 协变
        addCats(cats);     // ✅ 逆变
    }

    // 协变：读取
    public static void readAnimals(List<? extends Animal> animals) {
        for (Animal a : animals) {
            System.out.println("Animal: " + a);
        }
        // animals.add(new Dog()); // ❌ 编译错误
    }

    // 逆变：写入
    public static void addCats(List<? super Cat> animals) {
        animals.add(new Cat()); // ✅
    }
}
```

#### 总结

|  特性   |  协变   |  逆变   |
| --- | --- | --- |
|  Java 关键字   |  `? extends T`   |  `? super T`   |
|  C# 关键字   |  `out T`   |  `in T`   |
|  适合场景   |  读取数据   |  写入数据  |
|  是否可写   |  ❌   |  ✅   |
|  是否可读   |  ✅   |  ❌（只能当 Object）   |