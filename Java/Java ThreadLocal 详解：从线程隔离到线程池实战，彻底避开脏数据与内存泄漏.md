### 简介

`ThreadLocal` 直译是“线程本地变量”。它的特点不是让多个线程共享同一个变量，而是让每个线程都保存一份自己的变量。

可以把它理解成一组按线程分开的储物柜：柜子的编号由 `ThreadLocal` 对象决定，柜子属于当前线程。线程 A 放进去的数据，线程 B 看不到；即使两个线程操作的是同一个 `ThreadLocal` 对象，也不会读到对方的数据。

它最适合保存“只对当前线程有效”的上下文，例如：

* 当前请求的用户、租户和请求 ID
* 日志链路中的 `traceId`
* 同一线程内复用的数据库连接或事务上下文
* 每个线程独享的可变对象

但要先记住一句话：

```text
ThreadLocal 解决的是线程隔离，不是线程之间的通信，也不是共享变量的加锁方案。
```

### 先看一个真实问题：线程池为什么容易读到脏数据？

下面的代码看起来没有并发冲突，但第二个任务可能输出第一个任务留下的用户信息：

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadLocalPoolErrorDemo {
    private static final ThreadLocal<String> CURRENT_USER = new ThreadLocal<>();
    private static final ExecutorService POOL = Executors.newFixedThreadPool(1);

    public static void main(String[] args) throws InterruptedException {
        POOL.submit(() -> {
            CURRENT_USER.set("张三");
            System.out.println("任务一：" + CURRENT_USER.get());
            // 忘记 remove()
        });

        Thread.sleep(200);

        POOL.submit(() -> {
            // 线程被复用，这里可能还是“张三”
            System.out.println("任务二：" + CURRENT_USER.get());
        });

        POOL.shutdown();
    }
}
```

固定线程池里的工作线程不会随着任务结束而销毁。任务一把值放进了工作线程的本地变量，任务二又恰好使用同一个工作线程，于是旧值就被带了过来。

正确写法是把清理放到 `finally` 中：

```java
POOL.submit(() -> {
    try {
        CURRENT_USER.set("张三");
        // 业务代码
    } finally {
        CURRENT_USER.remove();
    }
});
```

这条规则非常重要：

```text
谁 set，谁负责在 finally 中 remove。
```

### ThreadLocal 到底保存在哪里？

很多资料会说“`ThreadLocal` 内部有一个 Map”，这句话不够准确。实际关系更接近下面这样：

```text
Thread
  └── ThreadLocalMap
        ├── ThreadLocal A -> value A
        ├── ThreadLocal B -> value B
        └── ThreadLocal C -> value C
```

数据放在 `Thread` 对象内部的 `ThreadLocalMap` 中，`ThreadLocal` 自身更像一把访问这张表的钥匙。

执行 `CURRENT_USER.set("张三")` 时，大致会发生这些事情：

1. 找到当前线程 `Thread.currentThread()`。
2. 取出当前线程里的 `ThreadLocalMap`。
3. 用当前 `ThreadLocal` 对象作为 key，保存对应的 value。
4. 后续 `get()` 仍然只从当前线程的 Map 中查值。

所以，同一个 `ThreadLocal` 对象在不同线程中对应的是不同的 value：

```text
线程 A 的 ThreadLocalMap：CURRENT_USER -> 张三
线程 B 的 ThreadLocalMap：CURRENT_USER -> 李四
```

这也是它天然具备线程隔离能力的原因。它没有通过锁让多个线程排队，而是让每个线程各用一份数据，属于“空间换时间”的思路。

### 一个完整的基础 Demo

```java
public class ThreadLocalBasicDemo {
    private static final ThreadLocal<String> USER = new ThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {
        Thread threadA = new Thread(() -> useThreadLocal("线程 A 的数据"), "Thread-A");
        Thread threadB = new Thread(() -> useThreadLocal("线程 B 的数据"), "Thread-B");

        threadA.start();
        threadB.start();
        threadA.join();
        threadB.join();

        // 主线程也有自己独立的副本
        useThreadLocal("主线程的数据");
    }

    private static void useThreadLocal(String value) {
        try {
            USER.set(value);
            System.out.println(Thread.currentThread().getName()
                    + " -> " + USER.get());
        } finally {
            USER.remove();
        }
    }
}
```

可能的输出如下，顺序不固定：

```text
Thread-A -> 线程 A 的数据
Thread-B -> 线程 B 的数据
main -> 主线程的数据
```

三个线程操作的是同一个 `USER`，但各自只能读到自己的值。

### 常用 API

#### `set(T value)`：设置当前线程的值

```java
ThreadLocal<String> local = new ThreadLocal<>();
local.set("hello");
```

这次设置只影响当前线程。其他线程调用 `get()` 时不会看到 `hello`。

#### `get()`：获取当前线程的值

```java
String value = local.get();
```

如果当前线程还没有设置过值，普通 `ThreadLocal` 返回 `null`。如果创建时提供了初始值，第一次 `get()` 会先创建初始值。

#### `remove()`：删除当前线程的值

```java
local.remove();
```

它只删除当前线程的副本，不会影响其他线程。删除后再次 `get()`，如果定义了初始值，会重新执行初始化逻辑。

#### `withInitial(Supplier)`：设置初始值

Java 8 以后，推荐使用 `withInitial`：

```java
private static final ThreadLocal<Integer> COUNTER =
        ThreadLocal.withInitial(() -> 0);
```

完整示例：

```java
public class ThreadLocalInitialDemo {
    private static final ThreadLocal<Integer> COUNTER =
            ThreadLocal.withInitial(() -> 0);

    public static void main(String[] args) {
        System.out.println(COUNTER.get()); // 0
        COUNTER.set(10);
        System.out.println(COUNTER.get()); // 10
        COUNTER.remove();
        System.out.println(COUNTER.get()); // 0，重新初始化
    }
}
```

### 最常见的实战：请求上下文

Web 请求经常需要在 Controller、Service、DAO 等多层代码中获取当前用户或请求 ID。如果每层都增加参数，方法签名会变得很臃肿。

可以把请求上下文放入 `ThreadLocal`，让同一个请求线程中的业务代码直接读取：

```java
final class RequestContext {
    private final String requestId;
    private final Long userId;

    RequestContext(String requestId, Long userId) {
        this.requestId = requestId;
        this.userId = userId;
    }

    public String getRequestId() {
        return requestId;
    }

    public Long getUserId() {
        return userId;
    }
}

final class RequestContextHolder {
    private static final ThreadLocal<RequestContext> CONTEXT = new ThreadLocal<>();

    private RequestContextHolder() {
    }

    public static void set(RequestContext context) {
        CONTEXT.set(context);
    }

    public static RequestContext get() {
        return CONTEXT.get();
    }

    public static void clear() {
        CONTEXT.remove();
    }
}
```

模拟过滤器和业务代码：

```java
public class RequestContextDemo {
    public static void handleRequest(Long userId) {
        try {
            RequestContext context = new RequestContext(
                    java.util.UUID.randomUUID().toString(), userId);
            RequestContextHolder.set(context);

            orderService();
        } finally {
            // Web 容器通常会复用线程，这里不能省略
            RequestContextHolder.clear();
        }
    }

    private static void orderService() {
        RequestContext context = RequestContextHolder.get();
        System.out.println("userId=" + context.getUserId()
                + ", requestId=" + context.getRequestId());
    }

    public static void main(String[] args) throws InterruptedException {
        Thread a = new Thread(() -> handleRequest(1001L));
        Thread b = new Thread(() -> handleRequest(1002L));
        a.start();
        b.start();
        a.join();
        b.join();
    }
}
```

在 Spring 项目中，过滤器、拦截器的入口和结束回调都适合放置 `set` 与 `remove`。`RequestContextHolder`、日志 MDC 等常见组件也使用了类似的线程上下文思路，但具体实现和清理责任仍需结合框架版本确认。

### 线程池中正确使用 ThreadLocal

线程池任务建议固定使用下面的模板：

```java
private static final ThreadLocal<String> TRACE_ID = new ThreadLocal<>();

static void runTask(String traceId) {
    try {
        TRACE_ID.set(traceId);
        doBusiness();
    } finally {
        TRACE_ID.remove();
    }
}

static void doBusiness() {
    System.out.println("当前 traceId：" + TRACE_ID.get());
}
```

`remove()` 不只是为了防止内存泄漏，还能避免两类业务问题：

* 下一个任务读到上一个任务的用户、租户或请求 ID。
* 线程长期存活时，ThreadLocal 中的大对象一直被线程持有。

如果是封装公共组件，最好让组件自己负责清理，而不是把清理要求留给每个调用方记忆。

### 为什么会有内存泄漏风险？

`ThreadLocalMap` 的 Entry 对 key 使用弱引用，但 value 仍然是强引用，关系大致如下：

```text
Thread
  -> ThreadLocalMap
       -> Entry
            -> 弱引用 ThreadLocal
            -> 强引用 value
```

当 `ThreadLocal` 对象没有其他强引用时，GC 可能回收 Entry 的 key，形成一个 key 已经为空、value 还存在的“脏 Entry”。JDK 会在线程后续操作 `ThreadLocalMap` 时顺便清理一部分脏 Entry，但这不是可靠的业务清理机制。

如果线程是线程池中的长期存活线程，value 可能比业务需要存活更久。因此不能把希望寄托在弱引用或 GC 上，正确做法仍然是：

```java
try {
    LOCAL.set(value);
    // 使用 value
} finally {
    LOCAL.remove();
}
```

另外，`ThreadLocal` 通常声明为 `static final`，保证作为 key 的实例长期可达；但这并不能替代任务结束时的 `remove()`。

### `ThreadLocal` 和 `synchronized` 有什么区别？

| 对比项 | `ThreadLocal` | `synchronized` |
| --- | --- | --- |
| 解决的问题 | 线程之间互相隔离数据 | 多线程安全访问共享数据 |
| 数据关系 | 每个线程一份 | 多个线程共享一份 |
| 是否加锁 | 不加锁 | 可能发生锁竞争 |
| 常见用途 | 请求上下文、线程私有对象 | 共享计数器、共享缓存 |
| 能否让线程看到彼此的修改 | 不能 | 可以，取决于同步方式 |

例如，多个线程累加同一个总数，需要使用 `AtomicInteger`、锁或其他并发工具；把总数放进 `ThreadLocal` 只会得到多个线程各自的总数，无法得到全局结果。

### `InheritableThreadLocal` 能解决子线程传值吗？

`InheritableThreadLocal` 会在创建子线程时，把父线程的值传给子线程：

```java
public class InheritableDemo {
    private static final InheritableThreadLocal<String> CONTEXT =
            new InheritableThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {
        CONTEXT.set("父线程数据");

        Thread child = new Thread(() ->
                System.out.println("子线程：" + CONTEXT.get()));
        child.start();
        child.join();

        CONTEXT.remove();
    }
}
```

但它只在“创建线程”时复制。线程池里的线程通常早已创建，后续提交任务不会重新复制父线程的数据，因此不能把它当成线程池上下文传递方案。

`CompletableFuture`、异步线程池、消息消费等场景同样要注意：`ThreadLocal` 默认不会自动跨线程传递。可靠做法包括显式传参、在任务包装器中设置并清理上下文，或使用专门的上下文传递组件。

### 一个容易被忽略的例子：`SimpleDateFormat`

`SimpleDateFormat` 不是线程安全的。旧项目中有时会使用 `ThreadLocal` 为每个线程保存一个实例：

```java
import java.text.SimpleDateFormat;
import java.util.Date;

public class DateFormatHolder {
    private static final ThreadLocal<SimpleDateFormat> FORMATTER =
            ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    public static String format(Date date) {
        try {
            return FORMATTER.get().format(date);
        } finally {
            FORMATTER.remove();
        }
    }
}
```

不过新代码更推荐使用线程安全的 `java.time.format.DateTimeFormatter`。只有在维护旧 API 或确实需要复用 `SimpleDateFormat` 时，才考虑这种写法。

### Java 25 之后：可以关注 `ScopedValue`

`ThreadLocal` 的值可以被任意代码修改，而且生命周期容易失控。Java 25 将 `ScopedValue` 正式纳入平台，用于在一个明确的动态作用域中读取不可变上下文，特别适合结构化并发和虚拟线程场景。

示例：

```java
import java.lang.ScopedValue;

public class ScopedValueDemo {
    private static final ScopedValue<String> USER = ScopedValue.newInstance();

    public static void main(String[] args) {
        ScopedValue.where(USER, "张三").run(() -> {
            System.out.println(USER.get());
            printUser();
        });
    }

    private static void printUser() {
        System.out.println("业务代码读取：" + USER.get());
    }
}
```

`ScopedValue` 不是 `ThreadLocal` 的简单改名：它强调“绑定在作用域内、读取为主、离开作用域自动失效”。现有 Spring 版本、线程模型和部署 JDK 需要一起评估，不能因为 JDK 版本较新就直接全量替换。传统线程池请求上下文仍然大量使用 `ThreadLocal`。

### 使用清单

* 把 `ThreadLocal` 当作当前线程的上下文容器，不要当作全局变量。
* 在线程池、Web 容器、定时任务中，始终使用 `try-finally` 清理。
* 存放用户、租户、权限等信息时，避免跨请求残留。
* 不要依赖弱引用和 GC 自动清理脏数据。
* 不要指望 `ThreadLocal` 自动跨越 `CompletableFuture` 或线程池。
* 能通过普通参数清晰传递的数据，不必强行放进 `ThreadLocal`。
* 需要在线程之间共享结果时，使用并发集合、原子类、锁或其他并发工具。
* 使用 `InheritableThreadLocal` 前，先确认线程是临时创建还是线程池复用。

### 总结

`ThreadLocal` 的核心并不复杂：数据属于线程，而不是属于 `ThreadLocal` 对象本身。

它非常适合保存当前请求、当前事务、当前链路等线程上下文，能够减少无意义的参数层层传递；但线程池让线程长期存活，也让清理责任变得不可回避。

实际开发中只要记住两件事，绝大多数坑都能避开：

```text
ThreadLocal 用来隔离线程数据，不用来在线程之间通信。
set 之后放进 finally 调用 remove，尤其是在可复用线程中。
```
