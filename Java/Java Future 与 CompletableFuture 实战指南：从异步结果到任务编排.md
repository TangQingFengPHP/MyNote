### 简介

`Future` 和 `CompletableFuture` 都属于 Java 并发编程里的异步结果模型。

简单理解：

```text
Future
  |
  v
异步任务的结果占位符
```

```text
CompletableFuture
  |
  v
可以链式处理、组合编排、异常兜底的异步结果
```

`Future` 从 Java 5 开始出现，主要解决一个问题：

```text
任务提交到线程池后，后面还能拿到任务结果。
```

`CompletableFuture` 从 Java 8 开始出现，解决的问题更进一步：

```text
异步任务完成后，可以继续转换、消费、组合、兜底，不必只靠 get() 阻塞等待。
```

一句话概括：

```text
Future 适合接收一个异步任务的结果，CompletableFuture 更适合编排一组异步任务。
```

### Future 解决什么问题

普通 `Thread` 执行任务时，结果不好直接返回。

```java
Thread thread = new Thread(() -> {
    int result = 100 + 200;
});

thread.start();
```

这里的 `result` 只能留在线程内部。

如果希望任务在后台执行，同时主流程后面还能拿到结果，就可以使用 `ExecutorService` 和 `Future`。

执行流程：

```text
提交 Callable 任务
  |
  v
线程池异步执行
  |
  v
立即返回 Future
  |
  v
需要结果时调用 get()
```

### Future 第一个 Demo

```java
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class FutureFirstDemo {

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        Callable<Integer> task = () -> {
            Thread.sleep(1000);
            return 100 + 200;
        };

        Future<Integer> future = executor.submit(task);

        System.out.println("任务已提交");
        System.out.println("主流程继续执行");

        Integer result = future.get();
        System.out.println("异步结果：" + result);

        executor.shutdown();
    }
}
```

输出类似：

```text
任务已提交
主流程继续执行
异步结果：300
```

`submit` 会立即返回一个 `Future`。

`future.get()` 会阻塞当前线程，直到任务完成。

### Future 常用方法

| 方法 | 作用 |
| --- | --- |
| `get()` | 阻塞等待任务完成并返回结果 |
| `get(timeout, unit)` | 最多等待指定时间 |
| `isDone()` | 判断任务是否完成 |
| `cancel(mayInterruptIfRunning)` | 尝试取消任务 |
| `isCancelled()` | 判断任务是否已取消 |

### Future 超时和取消

`get()` 如果一直等不到结果，当前线程会一直卡住。

更稳妥的写法是带超时：

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

public class FutureTimeoutDemo {

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(1);

        Future<String> future = executor.submit(() -> {
            Thread.sleep(3000);
            return "远程接口结果";
        });

        try {
            String result = future.get(1, TimeUnit.SECONDS);
            System.out.println(result);
        } catch (TimeoutException e) {
            future.cancel(true);
            System.out.println("任务超时，已尝试取消");
        } finally {
            executor.shutdown();
        }
    }
}
```

`cancel(true)` 会尝试中断正在执行的任务。

任务内部如果捕获了 `InterruptedException`，需要恢复中断状态：

```java
try {
    Thread.sleep(3000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    return "任务被取消";
}
```

### Future 的局限

`Future` 能拿到异步任务结果，但表达能力比较基础。

常见痛点：

* `get()` 会阻塞当前线程
* 任务完成后不能直接注册回调
* 多个任务组合时需要手写等待和取值
* 异常处理集中在 `get()` 周围
* 不能方便地把一个异步结果继续接到另一个异步任务

比如下面这种接口聚合：

```text
查用户
查订单
查账户
合并结果
```

用 `Future` 能写，但代码会围绕多个 `get()` 展开。

任务越多，流程越散。

### CompletableFuture 是什么

`CompletableFuture<T>` 同时实现了：

```text
Future<T>
CompletionStage<T>
```

它既能像 `Future` 一样表示异步结果，也能像流水线一样继续编排后续动作。

它支持：

* 创建异步任务
* 转换结果
* 消费结果
* 串行异步调用
* 并行合并结果
* 等待多个任务完成
* 任意一个任务完成就继续
* 异常兜底
* 超时控制
* 手动完成结果

### 第一个 CompletableFuture Demo

```java
import java.util.concurrent.CompletableFuture;

public class CompletableFutureFirstDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            sleep(1000);
            return "Java";
        });

        String result = future
                .thenApply(String::toUpperCase)
                .thenApply(value -> "Hello " + value)
                .join();

        System.out.println(result);
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

输出：

```text
Hello JAVA
```

这里没有在中间步骤里手动 `get()`。

任务完成后会自动进入 `thenApply`。

### runAsync 和 supplyAsync

`CompletableFuture` 创建异步任务常用两个静态方法。

### runAsync

没有返回值。

```java
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    System.out.println("刷新缓存");
});
```

适合：

* 发送通知
* 写日志
* 刷缓存
* 执行不需要返回结果的后台动作

### supplyAsync

有返回值。

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "商品详情";
});
```

适合：

* 查询远程接口
* 查询数据库
* 计算一个结果
* 读取文件内容

### 默认线程池

没有指定 `Executor` 时，`runAsync` 和 `supplyAsync` 默认使用 `ForkJoinPool.commonPool()`。

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return Thread.currentThread().getName();
});
```

输出类似：

```text
ForkJoinPool.commonPool-worker-1
```

默认池适合简单 Demo。

业务项目里更常见的是传入自定义线程池：

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "订单数据";
}, orderExecutor);
```

原因很简单：

```text
不同业务任务隔离
可以控制队列长度
可以设置线程名称
可以观察拒绝策略
可以避免所有异步任务挤在 commonPool
```

### 自定义线程池

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class AsyncExecutors {

    public static ExecutorService orderExecutor() {
        AtomicInteger counter = new AtomicInteger(1);

        ThreadFactory threadFactory = runnable -> {
            Thread thread = new Thread(runnable);
            thread.setName("order-async-" + counter.getAndIncrement());
            return thread;
        };

        return new ThreadPoolExecutor(
                8,
                16,
                60,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(500),
                threadFactory,
                new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
}
```

使用：

```java
ExecutorService executor = AsyncExecutors.orderExecutor();

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "订单查询结果";
}, executor);

System.out.println(future.join());

executor.shutdown();
```

线程池参数需要结合业务压测调整。

如果任务主要是 I/O 等待，也可以结合虚拟线程：

```java
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return callRemoteService();
}, executor);
```

### get 和 join

`Future` 常用 `get()`。

`CompletableFuture` 也有 `get()`，但更多代码会使用 `join()`。

| 方法 | 异常形式 | 说明 |
| --- | --- | --- |
| `get()` | `InterruptedException`、`ExecutionException` | 受检异常，需要显式处理 |
| `get(timeout, unit)` | 额外抛 `TimeoutException` | 带等待时间 |
| `join()` | `CompletionException` | 非受检异常，链式代码里更常见 |
| `getNow(defaultValue)` | 不阻塞 | 未完成时返回默认值 |

示例：

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "OK");

String result = future.join();
System.out.println(result);
```

如果任务异常，`join()` 会抛出 `CompletionException`。

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    throw new IllegalStateException("库存服务异常");
});

try {
    future.join();
} catch (CompletionException e) {
    System.out.println(e.getCause().getMessage());
}
```

### thenApply：转换结果

`thenApply` 用来把上一步结果转换成另一个值。

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "java")
        .thenApply(String::toUpperCase)
        .thenApply(value -> "语言：" + value);

System.out.println(future.join());
```

输出：

```text
语言：JAVA
```

类比同步写法：

```java
String value = "java";
String upper = value.toUpperCase();
String result = "语言：" + upper;
```

`thenApply` 适合纯内存转换，不适合在里面继续发起新的异步任务。

### thenAccept：消费结果

`thenAccept` 接收上一步结果，但不返回新值。

```java
CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> "订单创建成功")
        .thenAccept(message -> {
            System.out.println("发送通知：" + message);
        });

future.join();
```

适合：

* 打印结果
* 发送通知
* 写入日志
* 推送消息

### thenRun：只关心完成信号

`thenRun` 不接收上一步结果，也不返回新值。

```java
CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> "任务结果")
        .thenRun(() -> {
            System.out.println("任务已经完成");
        });

future.join();
```

适合只关心“完成了”这件事的场景。

### thenCompose：串行异步任务

如果第二个任务依赖第一个任务的结果，并且第二个任务本身也是异步的，使用 `thenCompose`。

示例场景：

```text
先查用户
再根据用户 ID 查订单
```

```java
CompletableFuture<User> userFuture = findUser(1001L);

CompletableFuture<Order> orderFuture = userFuture
        .thenCompose(user -> findLatestOrder(user.id()));
```

完整 Demo：

```java
import java.util.concurrent.CompletableFuture;

public class ThenComposeDemo {

    public static void main(String[] args) {
        CompletableFuture<Order> future = findUser(1001L)
                .thenCompose(user -> findLatestOrder(user.id()));

        System.out.println(future.join());
    }

    private static CompletableFuture<User> findUser(Long userId) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(300);
            return new User(userId, "张三");
        });
    }

    private static CompletableFuture<Order> findLatestOrder(Long userId) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(500);
            return new Order(9001L, userId, "PAID");
        });
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }

    record User(Long id, String username) {
    }

    record Order(Long id, Long userId, String status) {
    }
}
```

`thenCompose` 可以避免这种嵌套：

```java
CompletableFuture<CompletableFuture<Order>> nested = findUser(1001L)
        .thenApply(user -> findLatestOrder(user.id()));
```

### thenCombine：合并两个独立任务

如果两个任务互不依赖，可以并行发起，再合并结果。

示例场景：

```text
查用户信息
查账户余额
合并成用户首页
```

```java
CompletableFuture<User> userFuture = findUser(1001L);
CompletableFuture<Account> accountFuture = findAccount(1001L);

CompletableFuture<UserHome> homeFuture = userFuture.thenCombine(
        accountFuture,
        (user, account) -> new UserHome(user, account)
);
```

完整 Demo：

```java
import java.math.BigDecimal;
import java.util.concurrent.CompletableFuture;

public class ThenCombineDemo {

    public static void main(String[] args) {
        CompletableFuture<UserHome> future = findUser(1001L)
                .thenCombine(findAccount(1001L), UserHome::new);

        System.out.println(future.join());
    }

    private static CompletableFuture<User> findUser(Long userId) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(500);
            return new User(userId, "张三");
        });
    }

    private static CompletableFuture<Account> findAccount(Long userId) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(500);
            return new Account(userId, new BigDecimal("99.80"));
        });
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }

    record User(Long id, String username) {
    }

    record Account(Long userId, BigDecimal balance) {
    }

    record UserHome(User user, Account account) {
    }
}
```

两个任务都完成后，才会执行合并函数。

### allOf：等待全部任务完成

`allOf` 适合等待一组任务全部完成。

```java
import java.util.List;
import java.util.concurrent.CompletableFuture;

public class AllOfDemo {

    public static void main(String[] args) {
        List<Long> productIds = List.of(1L, 2L, 3L);

        List<CompletableFuture<Product>> futures = productIds.stream()
                .map(AllOfDemo::findProduct)
                .toList();

        CompletableFuture<Void> allFuture = CompletableFuture.allOf(
                futures.toArray(new CompletableFuture[0])
        );

        List<Product> products = allFuture
                .thenApply(ignore -> futures.stream()
                        .map(CompletableFuture::join)
                        .toList())
                .join();

        System.out.println(products);
    }

    private static CompletableFuture<Product> findProduct(Long productId) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(300);
            return new Product(productId, "商品-" + productId);
        });
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }

    record Product(Long id, String name) {
    }
}
```

`allOf` 返回的是 `CompletableFuture<Void>`。

所以需要在全部完成后，从原来的 `futures` 里逐个 `join()` 取结果。

### anyOf：等待任意一个任务完成

`anyOf` 适合“谁先返回用谁”的场景。

比如同一份配置可以从多个中心读取：

```java
import java.util.concurrent.CompletableFuture;

public class AnyOfDemo {

    public static void main(String[] args) {
        CompletableFuture<String> nacosFuture = loadFromNacos();
        CompletableFuture<String> apolloFuture = loadFromApollo();

        CompletableFuture<Object> fastest = CompletableFuture.anyOf(
                nacosFuture,
                apolloFuture
        );

        System.out.println(fastest.join());
    }

    private static CompletableFuture<String> loadFromNacos() {
        return CompletableFuture.supplyAsync(() -> {
            sleep(500);
            return "nacos-config";
        });
    }

    private static CompletableFuture<String> loadFromApollo() {
        return CompletableFuture.supplyAsync(() -> {
            sleep(300);
            return "apollo-config";
        });
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

返回类型是 `CompletableFuture<Object>`。

使用时通常需要做类型转换，或者把结果统一包装成相同类型。

### applyToEither：两个任务谁快用谁

`applyToEither` 和 `anyOf` 类似，但它可以继续转换结果，并保留泛型。

```java
CompletableFuture<String> primary = CompletableFuture.supplyAsync(() -> {
    sleep(500);
    return "primary";
});

CompletableFuture<String> backup = CompletableFuture.supplyAsync(() -> {
    sleep(300);
    return "backup";
});

CompletableFuture<String> result = primary.applyToEither(
        backup,
        value -> "使用结果：" + value
);

System.out.println(result.join());
```

适合主备接口、同城双活读取等场景。

### exceptionally：异常恢复

`exceptionally` 只在上游异常时执行。

```java
import java.util.concurrent.CompletableFuture;

public class ExceptionallyDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            throw new IllegalStateException("库存服务不可用");
        }).exceptionally(ex -> {
            return "库存状态未知";
        });

        System.out.println(future.join());
    }
}
```

输出：

```text
库存状态未知
```

适合返回兜底值。

### handle：正常和异常都处理

`handle` 不管上游成功还是失败，都会执行。

```java
import java.util.concurrent.CompletableFuture;

public class HandleDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            if (System.currentTimeMillis() > 0) {
                throw new IllegalStateException("支付服务异常");
            }
            return "支付成功";
        }).handle((result, ex) -> {
            if (ex != null) {
                return "支付状态待确认";
            }
            return result;
        });

        System.out.println(future.join());
    }
}
```

适合把成功和失败统一转换成同一种返回结构。

### whenComplete：完成后观察结果

`whenComplete` 可以拿到结果或异常，但通常不改变最终结果。

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "OK")
        .whenComplete((result, ex) -> {
            if (ex != null) {
                System.out.println("记录异常：" + ex.getMessage());
            } else {
                System.out.println("记录结果：" + result);
            }
        });

System.out.println(future.join());
```

适合：

* 记录日志
* 打点统计
* 释放资源
* 观察任务结果

如果 `whenComplete` 里继续抛异常，后续链路也会受到影响。

### exceptionally、handle、whenComplete 对比

| 方法 | 是否只处理异常 | 是否能改变结果 | 常见用途 |
| --- | --- | --- | --- |
| `exceptionally` | 是 | 可以 | 异常兜底 |
| `handle` | 否 | 可以 | 成功和失败统一转换 |
| `whenComplete` | 否 | 通常不改 | 日志、监控、资源释放 |

### 超时控制

Java 9 开始，`CompletableFuture` 提供了更方便的超时方法。

### orTimeout

超时后以异常完成。

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class OrTimeoutDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            sleep(3000);
            return "远程接口结果";
        }).orTimeout(1, TimeUnit.SECONDS)
          .exceptionally(ex -> "接口超时");

        System.out.println(future.join());
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

输出：

```text
接口超时
```

### completeOnTimeout

超时后用默认值完成。

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    sleep(3000);
    return "慢接口结果";
}).completeOnTimeout("默认结果", 1, TimeUnit.SECONDS);

System.out.println(future.join());
```

`completeOnTimeout` 适合可以接受默认值的场景。

`orTimeout` 适合需要显式进入异常处理的场景。

### 手动完成 CompletableFuture

`CompletableFuture` 可以先创建出来，后面再由其他线程填入结果。

```java
import java.util.concurrent.CompletableFuture;

public class ManualCompleteDemo {

    public static void main(String[] args) {
        CompletableFuture<String> future = new CompletableFuture<>();

        Thread thread = new Thread(() -> {
            try {
                Thread.sleep(500);
                future.complete("回调结果");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                future.completeExceptionally(e);
            }
        });

        thread.start();

        System.out.println(future.join());
    }
}
```

更常见的写法是把 `completeExceptionally` 放在回调桥接代码里：

```java
CompletableFuture<String> future = new CompletableFuture<>();

legacyClient.callAsync(new LegacyCallback() {
    @Override
    public void onSuccess(String result) {
        future.complete(result);
    }

    @Override
    public void onError(Throwable throwable) {
        future.completeExceptionally(throwable);
    }
});
```

这种写法适合把老式回调 API 包装成 `CompletableFuture`。

### 实战：用户首页聚合接口

场景：

```text
用户首页需要同时展示：
用户信息
账户余额
最近订单
优惠券数量
```

这些数据来自不同服务，互相之间没有依赖关系，适合并发查询。

领域模型：

```java
import java.math.BigDecimal;
import java.util.List;

public record UserInfo(Long userId, String username) {
}

public record AccountInfo(Long userId, BigDecimal balance) {
}

public record OrderInfo(Long orderId, String title) {
}

public record CouponInfo(Integer count) {
}

public record UserHomeResponse(
        UserInfo user,
        AccountInfo account,
        List<OrderInfo> orders,
        CouponInfo coupon
) {
}
```

远程服务模拟：

```java
import java.math.BigDecimal;
import java.util.List;

public class RemoteClients {

    public UserInfo findUser(Long userId) {
        sleep(300);
        return new UserInfo(userId, "张三");
    }

    public AccountInfo findAccount(Long userId) {
        sleep(500);
        return new AccountInfo(userId, new BigDecimal("88.60"));
    }

    public List<OrderInfo> findRecentOrders(Long userId) {
        sleep(700);
        return List.of(
                new OrderInfo(9001L, "Java 并发编程"),
                new OrderInfo(9002L, "Spring Boot 实战")
        );
    }

    public CouponInfo findCoupon(Long userId) {
        sleep(200);
        return new CouponInfo(3);
    }

    private void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

聚合服务：

```java
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class UserHomeService {

    private final RemoteClients remoteClients = new RemoteClients();
    private final ExecutorService executor = Executors.newFixedThreadPool(8);

    public UserHomeResponse loadHome(Long userId) {
        CompletableFuture<UserInfo> userFuture = CompletableFuture.supplyAsync(
                () -> remoteClients.findUser(userId),
                executor
        );

        CompletableFuture<AccountInfo> accountFuture = CompletableFuture.supplyAsync(
                () -> remoteClients.findAccount(userId),
                executor
        );

        CompletableFuture<List<OrderInfo>> orderFuture = CompletableFuture.supplyAsync(
                () -> remoteClients.findRecentOrders(userId),
                executor
        );

        CompletableFuture<CouponInfo> couponFuture = CompletableFuture.supplyAsync(
                () -> remoteClients.findCoupon(userId),
                executor
        ).completeOnTimeout(new CouponInfo(0), 300, TimeUnit.MILLISECONDS);

        return CompletableFuture
                .allOf(userFuture, accountFuture, orderFuture, couponFuture)
                .thenApply(ignore -> new UserHomeResponse(
                        userFuture.join(),
                        accountFuture.join(),
                        orderFuture.join(),
                        couponFuture.join()
                ))
                .orTimeout(2, TimeUnit.SECONDS)
                .join();
    }

    public void shutdown() {
        executor.shutdown();
    }
}
```

测试入口：

```java
public class UserHomeDemo {

    public static void main(String[] args) {
        UserHomeService service = new UserHomeService();

        try {
            long start = System.currentTimeMillis();
            UserHomeResponse response = service.loadHome(1001L);
            long cost = System.currentTimeMillis() - start;

            System.out.println(response);
            System.out.println("耗时：" + cost + " ms");
        } finally {
            service.shutdown();
        }
    }
}
```

这里四个查询并发执行，总耗时主要取决于最慢的那个任务，而不是四个任务耗时相加。

优惠券接口使用了 `completeOnTimeout`。

如果优惠券服务慢了，首页仍然可以返回，只是优惠券数量按 `0` 处理。

### 实战：下单流程串并结合

并不是所有任务都能并发。

下单流程通常既有依赖关系，也有独立查询。

示例流程：

```text
校验订单
  |
  v
并发查询用户、库存、价格
  |
  v
扣减库存
  |
  v
创建订单
```

代码示例：

```java
import java.math.BigDecimal;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class OrderCreateService {

    private final ExecutorService executor = Executors.newFixedThreadPool(8);

    public CompletableFuture<OrderCreateResult> create(OrderCreateCommand command) {
        return validate(command)
                .thenCompose(validCommand -> {
                    CompletableFuture<UserInfo> userFuture = findUser(validCommand.userId());
                    CompletableFuture<StockInfo> stockFuture = checkStock(validCommand.productId());
                    CompletableFuture<PriceInfo> priceFuture = findPrice(validCommand.productId());

                    return CompletableFuture
                            .allOf(userFuture, stockFuture, priceFuture)
                            .thenCompose(ignore -> deductStock(stockFuture.join()))
                            .thenApply(stockResult -> new OrderCreateResult(
                                    validCommand.userId(),
                                    validCommand.productId(),
                                    priceFuture.join().price(),
                                    stockResult.success()
                            ));
                });
    }

    private CompletableFuture<OrderCreateCommand> validate(OrderCreateCommand command) {
        return CompletableFuture.supplyAsync(() -> {
            if (command.count() <= 0) {
                throw new IllegalArgumentException("商品数量需要大于 0");
            }
            return command;
        }, executor);
    }

    private CompletableFuture<UserInfo> findUser(Long userId) {
        return CompletableFuture.supplyAsync(() -> new UserInfo(userId, "张三"), executor);
    }

    private CompletableFuture<StockInfo> checkStock(Long productId) {
        return CompletableFuture.supplyAsync(() -> new StockInfo(productId, 100), executor);
    }

    private CompletableFuture<PriceInfo> findPrice(Long productId) {
        return CompletableFuture.supplyAsync(() -> new PriceInfo(productId, new BigDecimal("59.90")), executor);
    }

    private CompletableFuture<StockResult> deductStock(StockInfo stockInfo) {
        return CompletableFuture.supplyAsync(() -> {
            if (stockInfo.available() <= 0) {
                throw new IllegalStateException("库存不足");
            }
            return new StockResult(true);
        }, executor);
    }

    public void shutdown() {
        executor.shutdown();
    }

    public record OrderCreateCommand(Long userId, Long productId, Integer count) {
    }

    public record StockInfo(Long productId, Integer available) {
    }

    public record PriceInfo(Long productId, BigDecimal price) {
    }

    public record StockResult(Boolean success) {
    }

    public record OrderCreateResult(
            Long userId,
            Long productId,
            BigDecimal price,
            Boolean stockDeducted
    ) {
    }
}
```

这段代码里：

```text
validate -> 串行前置
findUser / checkStock / findPrice -> 并行
deductStock -> 依赖库存检查结果
thenApply -> 组装最终结果
```

### Spring Boot 中返回 CompletableFuture

Spring MVC Controller 可以直接返回 `CompletableFuture<T>`。

Maven 依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

异步线程池：

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("orderAsyncExecutor")
    public Executor orderAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("order-async-");
        executor.initialize();
        return executor;
    }
}
```

Service：

```java
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

@Service
public class SpringOrderService {

    @Async("orderAsyncExecutor")
    public CompletableFuture<String> createOrder(Long userId, Long productId) {
        try {
            Thread.sleep(500);
            return CompletableFuture.completedFuture(
                    "订单创建成功，userId=" + userId + ", productId=" + productId
            );
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return CompletableFuture.failedFuture(e);
        }
    }
}
```

Controller：

```java
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.concurrent.CompletableFuture;

@RestController
@RequestMapping("/api/orders")
public class SpringOrderController {

    private final SpringOrderService orderService;

    public SpringOrderController(SpringOrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public CompletableFuture<String> create(
            @RequestParam Long userId,
            @RequestParam Long productId
    ) {
        return orderService.createOrder(userId, productId);
    }
}
```

访问：

```text
POST http://localhost:8080/api/orders?userId=1001&productId=2001
```

### CompletableFuture 和虚拟线程

Java 21 以后，I/O 密集型异步任务可以考虑使用虚拟线程执行器。

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CompletableFutureVirtualThreadDemo {

    public static void main(String[] args) {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
                sleep(500);
                return "虚拟线程执行结果：" + Thread.currentThread();
            }, executor);

            System.out.println(future.join());
        }
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

需要注意的是，`CompletableFuture` 负责任务编排，虚拟线程负责承载阻塞任务。

二者解决的问题不同：

```text
CompletableFuture：表达异步依赖关系
虚拟线程：降低阻塞任务的线程成本
```

普通同步代码如果只是想并发跑多个阻塞任务，也可以直接使用虚拟线程执行器和 `Future`。

如果需要复杂编排、异常兜底、任务组合，`CompletableFuture` 仍然很合适。

### 常见使用建议

### 为异步任务指定 Executor

业务代码里建议显式传入线程池。

```java
CompletableFuture.supplyAsync(() -> loadData(), executor);
```

这样可以避免多个业务共用默认 `ForkJoinPool.commonPool()` 后互相影响。

### 在线程池里设置有界队列

异步任务不是越多越好。

有界队列和拒绝策略可以让系统在压力过大时更早暴露问题。

```java
new ThreadPoolExecutor(
        8,
        16,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(500),
        threadFactory,
        new ThreadPoolExecutor.CallerRunsPolicy()
);
```

### 区分 thenApply 和 thenCompose

返回普通值时用 `thenApply`。

```java
CompletableFuture<String> nameFuture = userFuture
        .thenApply(UserInfo::username);
```

返回 `CompletableFuture` 时用 `thenCompose`。

```java
CompletableFuture<List<OrderInfo>> orderFuture = userFuture
        .thenCompose(user -> findRecentOrders(user.userId()));
```

### join 放在边界处

`join()` 会阻塞当前线程。

更清晰的写法是：

```text
中间流程继续返回 CompletableFuture
最外层边界再 join
```

例如：

```java
public CompletableFuture<UserHomeResponse> loadHomeAsync(Long userId) {
    return buildHomeFuture(userId);
}

public UserHomeResponse loadHome(Long userId) {
    return loadHomeAsync(userId).join();
}
```

### 异常需要进入链路

异步任务里吞掉异常，会让调用方误以为任务成功。

更常见的是抛出异常，让 `CompletableFuture` 异常完成：

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    if (remoteFailed()) {
        throw new IllegalStateException("远程服务失败");
    }
    return "OK";
});
```

然后在链路后面统一处理：

```java
future.exceptionally(ex -> "fallback");
```

### 设置超时

远程接口、数据库、RPC 调用都适合设置超时。

```java
CompletableFuture<String> future = CompletableFuture
        .supplyAsync(() -> callRemoteService(), executor)
        .orTimeout(2, TimeUnit.SECONDS);
```

### 常用 API 汇总

| API | 作用 |
| --- | --- |
| `ExecutorService.submit(...)` | 提交任务并返回 `Future` |
| `Future.get()` | 阻塞等待结果 |
| `Future.get(timeout, unit)` | 带超时等待结果 |
| `Future.cancel(true)` | 尝试取消任务 |
| `CompletableFuture.runAsync(...)` | 异步执行无返回任务 |
| `CompletableFuture.supplyAsync(...)` | 异步执行有返回任务 |
| `CompletableFuture.completedFuture(value)` | 创建已完成结果 |
| `CompletableFuture.failedFuture(error)` | 创建已失败结果 |
| `complete(value)` | 手动完成结果 |
| `completeExceptionally(error)` | 手动以异常完成 |
| `thenApply(...)` | 转换结果 |
| `thenAccept(...)` | 消费结果 |
| `thenRun(...)` | 完成后执行动作 |
| `thenCompose(...)` | 串行异步编排 |
| `thenCombine(...)` | 合并两个独立任务 |
| `allOf(...)` | 等待全部任务完成 |
| `anyOf(...)` | 等待任意一个任务完成 |
| `applyToEither(...)` | 两个任务谁先成功用谁 |
| `exceptionally(...)` | 异常兜底 |
| `handle(...)` | 成功和异常统一处理 |
| `whenComplete(...)` | 完成后观察结果 |
| `orTimeout(...)` | 超时后异常完成 |
| `completeOnTimeout(...)` | 超时后返回默认值 |
| `join()` | 阻塞获取结果，抛非受检异常 |
| `getNow(defaultValue)` | 不阻塞获取结果 |

### 总结

`Future` 和 `CompletableFuture` 都围绕“异步任务结果”展开。

`Future` 更基础：

```text
提交任务
拿到 Future
需要结果时 get
```

`CompletableFuture` 更适合任务编排：

```text
异步执行
结果转换
串行依赖
并行合并
异常兜底
超时控制
```

普通单个异步任务，用 `Future` 就能解决。

如果一个接口需要同时查多个服务、组合多个结果、设置兜底和超时，`CompletableFuture` 更顺手。

真正落到工程里，重点不是把所有代码都改成异步，而是把适合并发等待的 I/O 任务拆出来，再配好线程池、超时、异常和监控。
