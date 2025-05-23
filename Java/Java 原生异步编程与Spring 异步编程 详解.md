### 简介

`Java` 异步编程是现代高性能应用开发的核心技术之一，它允许程序在执行耗时操作（如网络请求、文件 `IO`）时不必阻塞主线程，从而提高系统吞吐量和响应性。

### 异步 vs 同步

* 同步：任务按顺序执行，后续任务需等待前任务完成。

```java
public String syncTask() {
    // 模拟耗时操作
    Thread.sleep(1000);
    return "Result";
}
```

* 异步：任务并行或在后台执行，主线程立即返回。

```java
public CompletableFuture<String> asyncTask() {
    return CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        return "Result";
    });
}
```

### Java 原生异步支持

#### 手动创建线程

最基本的异步方式是创建 `Thread` 或实现 `Runnable`。

* 缺点：管理线程池困难，资源浪费，难以复用，缺乏结果处理机制。

```java
public class BasicAsync {
    public static void main(String[] args) {
        Thread thread = new Thread(() -> {
            try {
                Thread.sleep(1000);
                System.out.println("Task completed");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread.start();
        System.out.println("Main thread continues");
    }
}
```

#### 使用 ExecutorService

* 优点：提供线程池管理，复用线程，减少创建开销

* 缺点：`Future.get()` 是阻塞的，难以链式调用

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadPoolExample {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        executor.submit(() -> {
            Thread.sleep(1000);
            System.out.println("Task 1 completed");
        });
        executor.submit(() -> {
            Thread.sleep(500);
            System.out.println("Task 2 completed");
        });
        executor.shutdown();
    }
}
```

##### 常用方法：

* `submit(Runnable)`：提交无返回值的任务。

* `submit(Callable)`：提交有返回值的任务，返回 `Future`。

* `shutdown()`：关闭线程池，不接受新任务。

##### 线程池类型：

* `Executors.newFixedThreadPool(n)`：固定大小线程池。

* `Executors.newCachedThreadPool()`：动态调整线程数。

* `Executors.newSingleThreadExecutor()`：单线程执行。

线程池类型对比：

|  类型   |  特性   |  适用场景   |
| --- | --- | --- |
|  `FixedThreadPool`   |  固定线程数，无界队列   |  负载稳定的长期任务   |
|  `CachedThreadPool`   |  自动扩容，60秒闲置回收   |  短时突发任务   |
|  `ScheduledThreadPool`   |  支持定时/周期性任务   |  心跳检测、定时报表   |
|  `WorkStealingPool`   | 使用 `ForkJoinPool`，任务窃取算法    |  计算密集型并行任务   |

#### Future（Java 5+）

```java
import java.util.concurrent.*;

public class FutureExample {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(1);
        Future<String> future = executor.submit(() -> {
            Thread.sleep(1000);
            return "Task completed";
        });

        // 主线程继续
        System.out.println("Doing other work");

        // 阻塞获取结果
        String result = future.get(); // 等待任务完成
        System.out.println(result);

        executor.shutdown();
    }
}
```

##### 方法

* `get()`：阻塞获取结果。

* `isDone()`：检查任务是否完成。

* `cancel(boolean)`：取消任务。

##### 缺点

* `get()` 是阻塞的，不利于非阻塞编程。

* 难以组合多个异步任务。

#### CompletableFuture（Java 8+）

支持链式调用，真正现代化异步编程方式。

```java
import java.util.concurrent.CompletableFuture;

public class CompletableFutureExample {
    public static void main(String[] args) {
        CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return "Task result";
        })
        .thenApply(result -> result.toUpperCase()) // 转换结果
        .thenAccept(result -> System.out.println(result)) // 消费结果
        .exceptionally(throwable -> {
            System.err.println("Error: " + throwable.getMessage());
            return null;
        });

        System.out.println("Main thread continues");
    }
}
```

#### 虚拟线程（Java 21+，Project Loom）

虚拟线程是 `Java 21` 引入的轻量级线程，适合高并发 I/O 密集型任务。

```java
public class VirtualThreadExample {
    public static void main(String[] args) {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            executor.submit(() -> {
                try {
                    Thread.sleep(1000);
                    System.out.println("Task completed in virtual thread");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        System.out.println("Main thread continues");
    }
}
```

**优势**

* 轻量级，创建开销极低（相比传统线程）。

* 适合 I/O 密集型任务（如 HTTP 请求、数据库查询）。

**注意**

* 不适合 `CPU` 密集型任务（可能导致线程饥饿）。

* `Spring Boot 3.2+` 支持虚拟线程（需配置）。

#### 阻塞 vs 非阻塞

| 类型                                | 是否阻塞           | 获取结果方式           |
| ----------------------------------- | ------------------ | ---------------------- |
| `Future<T>`                         | ✅ 是               | `future.get()`（阻塞） |
| `CompletableFuture<T>`              | ✅（get） ❌（then） | 支持非阻塞链式处理     |
| `@Async + Future/CompletableFuture` | ✅                  | `get()` 或回调         |
| `WebFlux`                           | ❌ 完全非阻塞       | 响应式 `Mono` / `Flux` |


#### `Future<T>` vs `CompletableFuture<T>`：核心对比

|  功能   |  `Future<T>`   |  `CompletableFuture<T>`   |
| --- | --- | --- |
|  Java 版本   |  Java 5+   |  Java 8+   |
|  是否可组合   |  ❌ 不支持   | ✅ 支持链式组合、并行执行    |
|  支持异步回调   |  ❌ 无   |  ✅ 有 `.thenApply()`、`.thenAccept()` 等   |
|  支持异常处理   |  ❌ 无   |  ✅ 有 `.exceptionally()` 等   |
|  可取消   |  ✅ 支持 `cancel()`   |  ✅ 也支持   |
|  阻塞获取   |  ✅ `get()` 阻塞   |  ✅ `get()` 阻塞（也可非阻塞）   |
|  使用场景   |  简单线程任务   |  多异步任务组合、复杂控制流   |

### Spring 异步编程（基于 @Async）

#### 配置类或启动类启用异步支持

```java
@SpringBootApplication
@EnableAsync
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

```java
@Configuration
@EnableAsync
public class AsyncConfig {
}
```

#### 无返回值用法

```java
// 无返回值的异步方法
@Async
public void sendEmail(String to) {
    System.out.println("异步发送邮件给: " + to);
    try { Thread.sleep(2000); } catch (InterruptedException e) {}
    System.out.println("邮件发送完成");
}
```

#### 使用 `Future<T>`

**创建异步方法**

```java
@Service
public class AsyncService {
    @Async
    public Future<String> processTask() {
        // 模拟耗时操作
        return new AsyncResult<>("Task completed");
    }
}
```

**调用并获取结果：**

```java
@Autowired
private AsyncService asyncService;

public void executeTask() throws Exception {
    Future<String> future = asyncService.processTask();
    String result = future.get(); // 阻塞等待结果
}
```

#### 使用 `CompletableFuture<T>`

**创建异步方法**

```java
@Async
public CompletableFuture<String> asyncMethod() {
    return CompletableFuture.completedFuture("Async Result");
}
```

**调用方式：**

```java
CompletableFuture<String> result = asyncService.asyncMethod();
// 非阻塞，可以做其他事
String value = result.get(); // 阻塞获取
```

#### 线程池配置

##### 使用自定义配置类

```java
@Configuration
public class AsyncConfig {

    @Bean("taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);      // 核心线程数
        executor.setMaxPoolSize(20);      // 最大线程数
        executor.setQueueCapacity(100);   // 队列容量
        executor.setKeepAliveSeconds(30); // 空闲线程存活时间
        executor.setThreadNamePrefix("async-task-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

// 指定线程池
@Async("taskExecutor")
public Future<String> customPoolTask() { ... }
```

##### 使用配置文件

```yml
# application.yml
spring:
  task:
    execution:
      pool:
        core-size: 5
        max-size: 20
        queue-capacity: 100
        thread-name-prefix: async-
      shutdown:
        await-termination: true
        terminate-on-timeout: true
```

#### Spring WebFlux 示例

```java
@Service
public class UserService {
    public Mono<String> getUser() {
        return Mono.just("用户信息").delayElement(Duration.ofSeconds(2));
    }

    public Flux<String> getAllUsers() {
        return Flux.just("用户1", "用户2", "用户3").delayElements(Duration.ofSeconds(1));
    }
}
```

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/one")
    public Mono<String> getUser() {
        return userService.getUser();
    }

    @GetMapping("/all")
    public Flux<String> getAllUsers() {
        return userService.getAllUsers();
    }
}
```

调用时非阻塞行为体现

* `Mono<String>` 表示未来异步返回一个值；

* `Flux<String>` 表示异步返回多个值；

* 请求立即返回 `Publisher`，只有订阅时才开始执行（懒执行、非阻塞）；

* 它不占用线程，不会“卡死线程”等待值返回。

#### SpringBoot 集成示例

* 标记 `@Async` 注解：
  
`@Async` 标记方法为异步执行，`Spring` 在线程池中运行该方法。

```java
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

@Service
public class AsyncService {
    @Async
    public CompletableFuture<String> doAsyncTask() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        return CompletableFuture.completedFuture("Task completed");
    }
}
```

* 启用异步
 
在主类或配置类上添加 `@EnableAsync`。

```java
@SpringBootApplication
@EnableAsync
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

* 控制器调用异步方法

```java
@RestController
public class AsyncController {
    @Autowired
    private AsyncService asyncService;

    @GetMapping("/async")
    public String triggerAsync() {
        asyncService.doAsyncTask().thenAccept(result -> System.out.println(result));
        return "Task triggered";
    }
}
```

* 自定义线程池

`Spring` 默认使用 `SimpleAsyncTaskExecutor`，不适合生产环境。推荐配置自定义线程池。

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

@Configuration
public class AsyncConfig {
    @Bean(name = "taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        executor.setThreadNamePrefix("AsyncThread-");
        executor.initialize();
        return executor;
    }
}
```

* 指定线程池：

```java
@Async("taskExecutor")
public CompletableFuture<String> doAsyncTask() {
    // 异步逻辑
}
```

* 为 `@Async` 方法定义全局异常处理器

```java
@Component
public class AsyncExceptionHandler implements AsyncUncaughtExceptionHandler {
    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        System.err.println("Async error: " + ex.getMessage());
    }
}
```

* `Spring Boot` 测试：

```java
@SpringBootTest
public class AsyncServiceTest {
    @Autowired
    private AsyncService asyncService;

    @Test
    void testAsync() throws Exception {
        CompletableFuture<String> future = asyncService.doAsyncTask();
        assertEquals("Task completed", future.get(2, TimeUnit.SECONDS));
    }
}
```

#### 并行调用多个服务示例

并行调用 `getUser` 和 `getProfile`，总耗时接近较慢的任务（~1s）。

```java
@Service
public class UserService {
    @Async
    public CompletableFuture<User> getUser(Long id) {
        return CompletableFuture.supplyAsync(() -> {
            // 模拟远程调用
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return new User(id, "User" + id);
        });
    }

    @Async
    public CompletableFuture<Profile> getProfile(Long id) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return new Profile(id, "Profile" + id);
        });
    }
}

@RestController
public class UserController {
    @Autowired
    private UserService userService;

    @GetMapping("/user/{id}")
    public CompletableFuture<UserProfile> getUserProfile(@PathVariable Long id) {
        return userService.getUser(id)
            .thenCombine(userService.getProfile(id),
                (user, profile) -> new UserProfile(user, profile));
    }
}
```

#### 异步批量处理示例

并行处理 10 个任务，显著减少总耗时。

```java
@Service
public class BatchService {
    @Async
    public CompletableFuture<Void> processItem(int item) {
        return CompletableFuture.runAsync(() -> {
            try {
                Thread.sleep(100);
                System.out.println("Processed item: " + item);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });
    }
}

@RestController
public class BatchController {
    @Autowired
    private BatchService batchService;

    @PostMapping("/batch")
    public CompletableFuture<Void> processBatch() {
        List<CompletableFuture<Void>> futures = new ArrayList<>();
        for (int i = 1; i <= 10; i++) {
            futures.add(batchService.processItem(i));
        }
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));
    }
}
```

#### 响应式 WebFlux 示例

```java
@Service
public class ReactiveService {
    public Mono<String> fetchData() {
        return Mono.just("Data")
                   .delayElement(Duration.ofSeconds(1));
    }
}

@RestController
public class ReactiveController {
    @Autowired
    private ReactiveService reactiveService;

    @GetMapping("/data")
    public Mono<String> getData() {
        return reactiveService.fetchData();
    }
}
```

#### Spring Data JPA 集成示例

`JPA` 默认阻塞操作，可通过 `@Async` 包装异步调用。

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {}

@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    @Async
    public CompletableFuture<User> findUser(Long id) {
        return CompletableFuture.supplyAsync(() -> userRepository.findById(id).orElse(null));
    }
}
```

#### MyBatis Plus 集成示例

`MyBatis Plus` 默认阻塞，可通过 `@Async` 或线程池异步化。

```java
@Mapper
public interface UserMapper extends BaseMapper<User> {}

@Service
public class UserService {
    @Autowired
    private UserMapper userMapper;

    @Async
    public CompletableFuture<User> getUser(Long id) {
        return CompletableFuture.supplyAsync(() -> userMapper.selectById(id));
    }
}
```

#### 注意事项

* `@Async` 方法必须是 `public` 的。

* 不能在同一类内调用 `@Async` 方法（因 `Spring AOP` 代理机制）。

* 默认线程池由 `Spring` 提供，可自定义。


### CompletableFuture 所有核心 API 

* `supplyAsync()`：异步执行任务，返回值

* `runAsync()`：异步执行任务，无返回值

* `thenApply()`：接收前面任务结果并返回新结果

* `thenAccept()`：接收结果但无返回

* `thenRun()`：不接收结果也不返回，仅执行

* `thenCompose()`：嵌套异步任务

* `thenCombine()`：两个任务都完成后，合并结果

* `allOf()`：等多个任务全部完成

* `anyOf()`：任一任务完成即继续

* `exceptionally()`：捕获异常并处理

* `whenComplete()`：无论成功失败都执行

* `handle()`：可处理正常或异常结果

### `CompletableFuture<T>` 用法详解

#### 创建异步任务

##### `supplyAsync`：基本异步任务执行

```java
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> "Result");

```

##### `runAsync`：异步执行任务，无返回值

```java
CompletableFuture<Void> cf = CompletableFuture.runAsync(() -> System.out.println("Async run"));
```

#### 任务转换

##### `thenApply(Function)`：转换结果，对结果加工

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> "data")
    .thenApply(data -> data.toUpperCase());

System.out.println(future.get()); // DATA
```

##### `thenCompose(Function)`：扁平化链式异步

```java
CompletableFuture<String> composed = CompletableFuture
    .supplyAsync(() -> "A")
    .thenCompose(a -> CompletableFuture.supplyAsync(() -> a + "B"));

composed.thenAccept(System.out::println); // 输出 AB
```

##### `thenCombine(CompletionStage, BiFunction)`：两个任务完成后合并结果

```java
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> "World");

cf1.thenCombine(cf2, (a, b) -> a + " " + b).thenAccept(System.out::println);
```

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "A");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "B");

CompletableFuture<String> result = f1.thenCombine(f2, (a, b) -> a + b);
System.out.println(result.get()); // AB
```

#### 消费结果

##### `thenAccept(Consumer)`：消费结果

```java
CompletableFuture
    .supplyAsync(() -> "Result")
    .thenAccept(result -> System.out.println("Received: " + result));
```

##### `thenRun(Runnable)`：继续执行下一个任务，无需前面结果

```java
CompletableFuture
    .supplyAsync(() -> "X")
    .thenRun(() -> System.out.println("Next step executed"));
```

#### 异常处理

##### `exceptionally(Function<Throwable, T>)`：异常处理

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("Oops!");
    return "ok";
}).exceptionally(ex -> "Fallback: " + ex.getMessage());

System.out.println(future.get());
```


##### `handle(BiFunction<T, Throwable, R>)`：同时处理正常与异常结果

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    throw new RuntimeException("Error!");
}).handle((result, ex) -> {
    if (ex != null) return "Handled: " + ex.getMessage();
    return result;
});

System.out.println(future.get());
```

##### `whenComplete(BiConsumer<T, Throwable>)`：类似 finally

* 在 `CompletableFuture` 执行完毕后执行一个回调，无论是成功还是异常。

* 不会改变原来的结果或异常，仅用于处理副作用（如日志）。

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Final Result")
    .whenComplete((result, ex) -> {
        System.out.println("Completed with: " + result);
    });
```

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("出错了");
    return "成功";
}).whenComplete((result, exception) -> {
    if (exception != null) {
        System.out.println("发生异常：" + exception.getMessage());
    } else {
        System.out.println("执行结果：" + result);
    }
});
```

#### 并发组合

##### allOf / anyOf：组合任务

```java
CompletableFuture<Void> all = CompletableFuture.allOf(task1, task2);
CompletableFuture<Object> any = CompletableFuture.anyOf(task1, task2);
```

##### allOf(...)：等待全部任务完成

需要单独从每个任务中再 `.get()` 拿到结果

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "A");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "B");

CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2);
all.thenRun(() -> System.out.println("All done")).get();
```

```java
CompletableFuture<String> userFuture = CompletableFuture.supplyAsync(() -> fetchUser());
CompletableFuture<String> orderFuture = CompletableFuture.supplyAsync(() -> fetchOrder());

// 两个任务都完成后执行
CompletableFuture<Void> bothDone = CompletableFuture.allOf(userFuture, orderFuture);

bothDone.thenRun(() -> {
    try {
        String user = userFuture.get();
        String order = orderFuture.get();
        System.out.println("用户: " + user + ", 订单: " + order);
    } catch (Exception e) {
        e.printStackTrace();
    }
});
```

##### anyOf(...)：任一完成即触发

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) {}
    return "fast";
});
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "slow");

CompletableFuture<Object> any = CompletableFuture.anyOf(f1, f2);
System.out.println(any.get()); // 输出最快那个
```

#### 超时控制

##### `orTimeout(long timeout, TimeUnit unit)`：超时异常

如果在指定时间内没有完成，就抛出 `TimeoutException` 异常。

```java
CompletableFuture<String> f = CompletableFuture.supplyAsync(() -> {
    try { Thread.sleep(2000); } catch (Exception e) {}
    return "late result";
}).orTimeout(1, TimeUnit.SECONDS);

try {
    System.out.println(f.get());
} catch (Exception e) {
    System.out.println("Timeout: " + e.getMessage());
}
```

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "执行完成";
}).orTimeout(2, TimeUnit.SECONDS)
  .exceptionally(ex -> "捕获到异常：" + ex.getClass().getSimpleName());

System.out.println("结果：" + future.join()); // 打印“捕获到异常：TimeoutException”
```

##### `completeOnTimeout(T value, long timeout, TimeUnit unit)`：超时默认值

如果在指定时间内没有完成，则返回一个默认值，并完成该任务。

```java
CompletableFuture<String> f = CompletableFuture.supplyAsync(() -> {
    try { Thread.sleep(2000); } catch (Exception e) {}
    return "slow";
}).completeOnTimeout("timeout default", 1, TimeUnit.SECONDS);

System.out.println(f.get()); // timeout default
```

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(3000); // 模拟耗时任务
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "正常返回结果";
}).completeOnTimeout("超时默认值", 2, TimeUnit.SECONDS);

System.out.println("最终结果：" + future.join()); // 会打印“超时默认值”
```

#### 自定义线程池

```java
ExecutorService pool = Executors.newFixedThreadPool(2);

CompletableFuture<String> f = CompletableFuture.supplyAsync(() -> "pooled", pool);
System.out.println(f.get());
pool.shutdown();
```

#### 异步任务 + 消费结果

```java
CompletableFuture<Void> future = CompletableFuture
    .supplyAsync(() -> "hello")
    .thenAccept(result -> System.out.println("结果是：" + result));
```

#### 异步任务 + 转换结果（链式调用）

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> "5")
    .thenApply(Integer::parseInt)
    .thenApply(num -> num * 2)
    .thenApply(Object::toString);
```

#### 异常处理

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> {
        if (true) throw new RuntimeException("出错了！");
        return "success";
    })
    .exceptionally(ex -> {
        System.out.println("异常: " + ex.getMessage());
        return "默认值";
    });
```

#### 多任务并发组合（allOf / anyOf）

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "A");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "B");

// 等待全部完成
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2);
all.join();

System.out.println("结果：" + f1.join() + ", " + f2.join());
```

#### 合并两个任务结果

```java
CompletableFuture<Integer> f1 = CompletableFuture.supplyAsync(() -> 100);
CompletableFuture<Integer> f2 = CompletableFuture.supplyAsync(() -> 200);

CompletableFuture<Integer> result = f1.thenCombine(f2, Integer::sum);
System.out.println(result.get()); // 输出 300
```

#### 自定义线程池

```java
ExecutorService pool = Executors.newFixedThreadPool(4);

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "线程池中的任务";
}, pool);

System.out.println(future.get());
pool.shutdown();
```

#### 链式异步处理

```java
CompletableFuture.supplyAsync(() -> "Step 1")
    .thenApply(s -> s + " -> Step 2")
    .thenCompose(s -> CompletableFuture.supplyAsync(() -> s + " -> Step 3"))
    .thenAccept(System.out::println)
    .exceptionally(ex -> {
        ex.printStackTrace();
        return null;
    });
```

#### 订单处理示例

```java
public class OrderSystem {
    @Async("dbExecutor")
    public CompletableFuture<Order> saveOrder(Order order) {
        // 数据库写入操作
        return CompletableFuture.completedFuture(order);
    }

    @Async("httpExecutor")
    public CompletableFuture<String> notifyLogistics(Order order) {
        // 调用物流API
        return CompletableFuture.completedFuture("SUCCESS");
    }

    public void processOrder(Order order) {
        CompletableFuture<Order> saveFuture = saveOrder(order);
        saveFuture.thenCompose(savedOrder -> 
            notifyLogistics(savedOrder)
        ).exceptionally(ex -> {
            log.error("物流通知失败", ex);
            return "FALLBACK";
        });
    }
}
```

#### 总结图谱

```scss
CompletableFuture
├─ 创建任务
│  ├─ runAsync() -> 无返回值
│  └─ supplyAsync() -> 有返回值
├─ 处理结果
│  ├─ thenApply() -> 转换
│  ├─ thenAccept() -> 消费
│  ├─ thenRun() -> 执行新任务
│  ├─ thenCombine() -> 合并结果
│  └─ thenCompose() -> 链式调用
├─ 异常处理
│  ├─ exceptionally()
│  ├─ handle()
│  └─ whenComplete()
├─ 组合任务
│  ├─ allOf()
│  └─ anyOf()
└─ 超时控制
   ├─ orTimeout()
   └─ completeOnTimeout()
```

### 什么场景适合用 Java 异步（@Async / CompletableFuture）？

|  场景   |  是否适合异步？   |
| --- | --- |
|  调用多个远程服务并行   |  ✅ 很适合   |
|  复杂 CPU 运算耗时任务   |  ✅ 可以放到异步线程池   |
| 简单业务逻辑、数据库操作    |  ❌ 不建议，同步更可控   |
|  非主流程的日志、打点操作   |  ✅ 合适异步处理   |

### Java 和 .NET 异步处理对比

> 并行调用两个服务，提高响应速度

#### Spring Boot 示例（@Async + CompletableFuture）

**项目结构**

```java
└── src
    └── main
        ├── java
        │   ├── demo
        │   │   ├── controller
        │   │   │   └── AggregateController.java
        │   │   ├── service
        │   │   │   ├── RemoteService.java
        │   │   │   └── RemoteServiceImpl.java
        │   │   └── DemoApplication.java
```

**RemoteService.java**

```java
public interface RemoteService {
    @Async
    CompletableFuture<String> getUserInfo();

    @Async
    CompletableFuture<String> getAccountInfo();
}
```

**RemoteServiceImpl.java**

```java
@Service
public class RemoteServiceImpl implements RemoteService {

    @Override
    public CompletableFuture<String> getUserInfo() {
        try {
            Thread.sleep(2000); // 模拟耗时
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        return CompletableFuture.completedFuture("UserInfo");
    }

    @Override
    public CompletableFuture<String> getAccountInfo() {
        try {
            Thread.sleep(3000); // 模拟耗时
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        return CompletableFuture.completedFuture("AccountInfo");
    }
}
```

**AggregateController.java**

```java
@RestController
@RequestMapping("/api")
public class AggregateController {

    @Autowired
    private RemoteService remoteService;

    @GetMapping("/aggregate")
    public ResponseEntity<String> aggregate() throws Exception {
        CompletableFuture<String> userFuture = remoteService.getUserInfo();
        CompletableFuture<String> accountFuture = remoteService.getAccountInfo();

        // 等待所有完成
        CompletableFuture.allOf(userFuture, accountFuture).join();

        // 获取结果
        String result = userFuture.get() + " + " + accountFuture.get();
        return ResponseEntity.ok(result);
    }
}
```

**DemoApplication.java**

```java
@SpringBootApplication
@EnableAsync
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

#### .NET 示例（async/await）

**项目结构**

```csharp
└── Controllers
    └── AggregateController.cs
└── Services
    └── IRemoteService.cs
    └── RemoteService.cs
```

**IRemoteService.cs**

```csharp
public interface IRemoteService {
    Task<string> GetUserInfoAsync();
    Task<string> GetAccountInfoAsync();
}
```

**RemoteService.cs**

```csharp
public class RemoteService : IRemoteService {
    public async Task<string> GetUserInfoAsync() {
        await Task.Delay(2000); // 模拟耗时
        return "UserInfo";
    }

    public async Task<string> GetAccountInfoAsync() {
        await Task.Delay(3000); // 模拟耗时
        return "AccountInfo";
    }
}
```

**AggregateController.cs**

```csharp
[ApiController]
[Route("api/[controller]")]
public class AggregateController : ControllerBase {
    private readonly IRemoteService _remoteService;

    public AggregateController(IRemoteService remoteService) {
        _remoteService = remoteService;
    }

    [HttpGet("aggregate")]
    public async Task<IActionResult> Aggregate() {
        var userTask = _remoteService.GetUserInfoAsync();
        var accountTask = _remoteService.GetAccountInfoAsync();

        await Task.WhenAll(userTask, accountTask);

        var result = $"{userTask.Result} + {accountTask.Result}";
        return Ok(result);
    }
}
```

#### Java vs .NET 异步用法对比总结

| 方面         | Java（Spring Boot）            | .NET Core（ASP.NET） |
| ------------ | ------------------------------ | -------------------- |
| 异步声明方式 | `@Async` + `CompletableFuture` | `async/await`        |
| 返回值类型   | `CompletableFuture<T>`         | `Task<T>`            |
| 等待多个任务 | `CompletableFuture.allOf()`    | `Task.WhenAll()`     |
| 是否阻塞     | `.get()` 会阻塞，链式不阻塞    | `await` 非阻塞       |
| 简洁性       | 稍复杂（需要注解和线程池配置） | 极简、天然异步支持   |
