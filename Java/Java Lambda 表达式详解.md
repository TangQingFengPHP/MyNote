### 简介

`Lambda` 表达式是 `Java 8` 引入的一种简洁的语法，主要用于 简化匿名内部类 的写法，特别适用于 函数式接口（`Functional Interface`）。

### Lambda 表达式的语法

`Lambda` 表达式的基本格式如下：

```java
(参数列表) -> { 方法体 }
```

完整格式

```java
(参数1, 参数2, ...) -> { 方法体 }
```

简化规则

* 参数类型可省略（`Java` 编译器可自动推断）

* 只有一个参数时，可以省略括号 `(param) -> ...` 可写成 `param -> ...`

* 只有一条语句时，可以省略 `{}` 和 `return`

### 使用示例

#### 传统匿名内部类

创建一个接口的匿名实现类 并实例化：

```java
List<String> list = Arrays.asList("A", "B", "C");
list.forEach(new Consumer<String>() {
    @Override
    public void accept(String s) {
        System.out.println(s);
    }
});
```

等价于：

```java
class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Running...");
    }
}

Runnable task = new MyRunnable();
task.run();
```

#### 使用 Lambda

```java
list.forEach(s -> System.out.println(s));

或者：

list.forEach(System.out::println);
```

### Lambda 适用的接口

`Lambda` 适用于 函数式接口，即只包含一个抽象方法的接口。常见的有：

|  函数式接口   |  方法签名   |  作用   |
| --- | --- | --- |
|  Runnable   |  	void run()   |  无参数，无返回值   |
|  Callable<T>   |  T call()   |  无参数，有返回值   |
|  Comparator<T>   |  int compare(T o1, T o2)   |  比较两个对象   |
|  Consumer<T>   |  void accept(T t)   |  只消费数据，无返回值   |
|  Supplier<T>   |  T get()   |  只提供数据，有返回值   |
|  Function<T, R>   |  R apply(T t)   |  输入 T，输出 R   |
|  Predicate<T>   |  boolean test(T t)   |  断言（返回布尔值）   |

### 详细示例

#### Runnable（无参数、无返回值）

* 传统写法

```java
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello Lambda!");
    }
};
new Thread(r1).start();
```

* Lambda

```java
Runnable r2 = () -> System.out.println("Hello Lambda!");
new Thread(r2).start();
```

* 简化写法

```java
new Thread(() -> System.out.println("Hello Lambda!")).start();
```

#### Comparator<T>（多个参数，有返回值）

* 传统写法

```java
Comparator<Integer> comparator = new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return o2 - o1;
    }
};
```

* Lambda

```java
Comparator<Integer> comparator = (o1, o2) -> o2 - o1;
```

#### Consumer<T>（消费型接口）

* 传统写法

```java
Consumer<String> consumer = new Consumer<String>() {
    @Override
    public void accept(String s) {
        System.out.println(s);
    }
};
consumer.accept("Hello World!");
```

* Lambda

```java
Consumer<String> consumer = s -> System.out.println(s);
consumer.accept("Hello World!");
```

#### Supplier<T>（只提供数据，有返回值）

* 传统写法

```java
Supplier<String> supplier = new Supplier<String>() {
            @Override
            public String get() {
                return "Hello from anonymous class";
            }
        };

System.out.println(supplier.get());
```

* Lambda 

```java
Supplier<String> supplier = () -> "Hello from Lambda";
```

* 使用方法引用

```java
public class Demo {
    public static String greeting() {
        return "Hello!";
    }

    public static void main(String[] args) {
        Supplier<String> supplier = Demo::greeting;
        System.out.println(supplier.get());
    }
}
```

* 配合 `Optional` 使用

```java
import java.util.Optional;
import java.util.function.Supplier;

public class OptionalSupplierExample {
    public static void main(String[] args) {
        String name = null;

        Supplier<String> defaultSupplier = () -> "Default Name";
        String result = Optional.ofNullable(name).orElseGet(defaultSupplier);

        System.out.println(result); // 输出 Default Name
    }
}
```

* 延迟对象创建（惰性加载）

```java
public class HeavyObject {
    public HeavyObject() {
        System.out.println("HeavyObject 构造函数被调用");
    }
}

public class LazyLoader {
    public static void load(Supplier<HeavyObject> supplier, boolean needed) {
        if (needed) {
            HeavyObject obj = supplier.get();
            System.out.println("对象已创建");
        } else {
            System.out.println("未使用对象，无需创建");
        }
    }

    public static void main(String[] args) {
        load(HeavyObject::new, false); // HeavyObject 不会被创建
        load(HeavyObject::new, true);  // 会创建
    }
}
```


#### Function<T, R>（转换型接口）

传统写法

```java
Function<String, Integer> function = new Function<String, Integer>() {
    @Override
    public Integer apply(String s) {
        return s.length();
    }
};
System.out.println(function.apply("Lambda"));
```

* Lambda

```java
Function<String, Integer> function = s -> s.length();
System.out.println(function.apply("Lambda"));
```

#### Predicate<T>（断言型接口）

* 传统写法

```java
Predicate<Integer> predicate = new Predicate<Integer>() {
    @Override
    public boolean test(Integer num) {
        return num > 10;
    }
};
System.out.println(predicate.test(15)); // true
```

* Lambda

```java
Predicate<Integer> predicate = num -> num > 10;
System.out.println(predicate.test(15)); // true
```

### 方法引用

`Lambda` 表达式还可以用 方法引用（`::`）来替代更简洁的写法。

|  类型   |  示例   |  等效 Lambda 表达式   |
| --- | --- | --- |
|  静态方法   |  Integer::parseInt   |  s -> Integer.parseInt(s)   |
|  实例方法   |  String::toUpperCase   |  s -> s.toUpperCase()   |
|  对象实例方法   |  System.out::println   |  s -> System.out.println(s)   |
|  构造方法   |  ArrayList::new   |  () -> new ArrayList<>()   |

示例：

```java
List<String> list = Arrays.asList("a", "b", "c");
list.forEach(System.out::println);
```

### Stream API 与 Lambda

`Lambda` 常配合 `Stream API` 处理集合：

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
List<String> filteredNames = names.stream()
    .filter(name -> name.length() > 3)
    .map(String::toUpperCase)
    .collect(Collectors.toList());
System.out.println(filteredNames); // [ALICE, CHARLIE]
```

### 自定义函数式接口

使用 `@FunctionalInterface` 自定义一个函数式接口：

```java
@FunctionalInterface
public interface MyFunction {
    int apply(int a, int b);
}
```

使用 `Lambda`：

```java
MyFunction add = (a, b) -> a + b;
System.out.println(add.apply(3, 5)); // 8
```

* `@FunctionalInterface` 不是必须的，但加上后，编译器会强制检查该接口是否符合 函数式接口 规范（即只能有一个抽象方法）。

* 可以添加 默认方法 和 静态方法，但不能有多个抽象方法：

```java
@FunctionalInterface
public interface MyFunction {
    int apply(int a, int b);

    default void printHello() {
        System.out.println("Hello");
    }
    
    static void staticMethod() {
        System.out.println("Static Method");
    }
}
```

### 方法引用（Method Reference）示例

方法引用是一种特殊的 `Lambda` 表达式写法，主要用来 简化代码。它可以看作 `Lambda` 表达式的简写形式，前提是 `Lambda` 只是调用一个已有的方法。

#### 静态方法引用

```java
Function<String, Integer> parseInt = Integer::parseInt;
System.out.println(parseInt.apply("123")); // 123
```

等同于

```java
Function<String, Integer> parseInt = s -> Integer.parseInt(s);
```

#### 实例方法引用

对特定对象调用实例方法

```java
String str = "hello";
Supplier<String> toUpper = str::toUpperCase;
System.out.println(toUpper.get()); // "HELLO"
```

等同于

```java
Supplier<String> toUpper = () -> str.toUpperCase();
```

#### 对象实例方法引用

应用在 `map()` 这样的场景

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
List<String> upperNames = names.stream()
    .map(String::toUpperCase)  // 等同于 name -> name.toUpperCase()
    .collect(Collectors.toList());

System.out.println(upperNames); // [ALICE, BOB, CHARLIE]
```

#### 构造方法引用

如果 `Lambda` 表达式的目标是创建对象，可以使用 构造方法引用：

```java
Supplier<ArrayList<String>> listSupplier = ArrayList::new;
ArrayList<String> list = listSupplier.get(); // 创建 ArrayList 实例
```

等价于

```java
Supplier<ArrayList<String>> listSupplier = () -> new ArrayList<>();
```

应用场景：

```java
Function<Integer, int[]> arrayCreator = int[]::new;
int[] arr = arrayCreator.apply(5); // 创建长度为5的数组
System.out.println(Arrays.toString(arr)); // [0, 0, 0, 0, 0]
```

#### 方法引用作为方法参数

```java
public class Demo {
    public static void printSomething(Supplier<String> supplier) {
        System.out.println(supplier.get());
    }

    public static void main(String[] args) {
        printSomething(() -> "Hello World");
        printSomething("Hello"::toUpperCase); // 方法引用
    }
}
```

#### .map(String::toUpperCase) 传入 Function？

在 `.map(String::toUpperCase)` 中，`String::toUpperCase` 是符合 `Function<T, R>` 类型的，而不是 `Supplier`。

* `.map()` 方法的参数是 `Function<T, R>`：

```java
public interface Function<T, R> {
    R apply(T t);
}

// 它接收一个参数 T，返回 R。
```

* `String::toUpperCase` 等价于：

```java
(String s) -> s.toUpperCase()
```

#### "Hello"::toUpperCase 适用于 Supplier<String>

因为 `"Hello"` 是一个已经存在的 `String` 对象，不需要参数。

```java
Supplier<String> supplier = "Hello"::toUpperCase;
System.out.println(supplier.get()); // 输出：HELLO
```

等价于：

```java
Supplier<String> supplier = () -> "Hello".toUpperCase();
```

#### String::toUpperCase 与 "Hello"::toUpperCase

|  形式   |  绑定情况   |  适用函数式接口   |  是否需要参数   |  示例   |
| --- | --- | --- | --- | --- |
|  String::toUpperCase   | 未绑定（Unbound）    |  Function<String, String>   |  需要 String 参数   |  s -> s.toUpperCase()   |
|  "Hello"::toUpperCase   |  已绑定（Bound）   |  Supplier<String>   |  不需要参数   |  () -> "Hello".toUpperCase()
   |

#### Callable<T> 的使用示例

`Callable<T>` 是 `Java` 并发包中的一个接口，类似 `Runnable`，但它可以有返回值，并且可以抛出异常。

示例：使用 `ExecutorService.submit()` 提交 `Callable<T>`

```java
import java.util.concurrent.*;

public class CallableExample {
    public static void main(String[] args) throws Exception {
        // 1. 创建线程池
        ExecutorService executor = Executors.newFixedThreadPool(2);

        // 2. 提交 Callable 任务
        Callable<Integer> task = () -> {
            Thread.sleep(2000);
            return 42;
        };

        Future<Integer> future = executor.submit(task);

        // 3. 执行其他任务
        System.out.println("正在执行其他任务...");

        // 4. 获取返回结果
        Integer result = future.get(); // 阻塞等待返回值
        System.out.println("任务结果：" + result); // 输出：42

        // 5. 关闭线程池
        executor.shutdown();
    }
}
```

* `Callable<Integer>` 任务 延迟 2 秒，然后返回 42

* `submit(task)` 返回 `Future<Integer>`，表示异步计算的结果。

* `future.get()` 会阻塞，直到 `Callable` 执行完成并返回结果。