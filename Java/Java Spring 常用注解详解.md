### 简介

`Spring` 框架中提供了大量注解用于简化开发、提升代码可读性、实现依赖注入、事务管理、`AOP、RESTful API` 等功能。

### 核心注解（IOC 容器管理）

* `@Component`：标注一个类为组件，由 `Spring` 容器自动扫描并管理（泛指 `Bean`）

* `@Service`：表示业务逻辑组件，功能等同于 `@Component`，语义更明确

```java
@Service
public class UserService {
    public void saveUser() {}
}
```

* `@Repository`：表示数据访问组件，功能同 `@Component`，`Spring` 会对其进行异常转换

```java
@Repository
public class UserRepository {
    // 数据访问逻辑
}
```

* `@Controller`：表示 `Spring MVC` 控制器

```java
@Controller
public class UserController {
    // 处理 HTTP 请求
}
```

* `@RestController`：等价于 `@Controller + @ResponseBody`，用于 `RESTful API`

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    @GetMapping
    public List<User> getUsers() {
        return Collections.emptyList();
    }
}
```

配套注解：

* `@Autowired`：自动注入依赖（默认按类型注入）

* `@Qualifier`：配合 `@Autowired` 按名称注入

* `@Value("${property}")`：注入配置文件中的值

* `@Primary`：多个 `Bean` 时，标记默认注入的 `Bean`

* `@Lazy`：延迟初始化 `Bean`

优化启动时间，处理资源密集型 `bean`。

```java
@Bean
@Lazy
public ExpensiveBean expensiveBean() {
    return new ExpensiveBean();
}
```

### 配置类注解

* `@Configuration`：标记配置类，等价于 `XML` 中的 `<beans>`

* `@Bean`：声明一个由 `Spring` 管理的 `Bean`，常用于第三方类注入

* `@ComponentScan`：指定包扫描组件（默认扫描当前类所在包及子包）

```java
@Configuration
@ComponentScan("com.example")
public class AppConfig {}
```

* `@Import`：引入其他配置类

* `@PropertySource`：加载指定（ `.properties` 配置文件）

* `@EnableAsync` 系列：启用异步功能

* `@EnableScheduling`：启用定时任务

### Web 层相关注解（Spring MVC）

* `@RequestMapping`：映射请求路径（可用于类/方法）

```java
@RequestMapping(value = "/hello", method = RequestMethod.GET)
public String sayHello() {
    return "Hello";
}
```

* `@GetMapping / @PostMapping / @PutMapping / @DeleteMapping`：请求方法快捷注解

* `@RequestParam`：获取 `URL` 参数（例如 `?name=abc`）

```java
@GetMapping("/search")
public List<User> search(@RequestParam String name) {
    return Collections.emptyList();
}
```

* `@PathVariable`：获取路径变量（如 `/user/{id}`）

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return new User(id);
}
```

* `@RequestBody`：获取请求体中的 `JSON` 数据并反序列化

```java
@PostMapping("/users")
public User createUser(@RequestBody User user) {
    return user;
}
```

* `@ModelAttribute`：方法参数绑定表单数据

* `@ResponseBody`：将返回值序列化为 `HTTP` 响应体（如 `JSON`）

```java
@GetMapping("/data")
@ResponseBody
public User getData() {
    return new User();
}
```

* `@ResponseStatus`：指定 HTTP 响应状态码

```java
@PostMapping("/users")
@ResponseStatus(HttpStatus.CREATED)
public User createUser() {
    return new User();
}
```

* `@CrossOrigin`：处理跨域请求（`CORS`）

### Spring Boot 相关注解

* `@SpringBootApplication`：核心注解，包含 `@Configuration、@EnableAutoConfiguration、@ComponentScan`

* `@EnableAutoConfiguration`：启用自动配置

* `@ConfigurationProperties`：绑定配置文件到 `Java` 对象

* `@RestControllerAdvice`：全局异常处理类

* `@SpringBootTest`：用于 `Spring Boot` 测试类

* `@TestConfiguration`：用于测试场景下的配置类

* `@Profile`：指定配置或 `Bean` 仅在某个环境（`profile`）生效

### 事务与异步等功能性注解

* `@Transactional`：事务控制，常用于 `Service` 层方法

* `@EnableTransactionManagement`：启用事务管理（如非 `Spring Boot` 时需显式开启）

* `@Async`：异步执行方法（配合 `@EnableAsync`）

* `@Scheduled`：定时任务执行（配合 `@EnableScheduling`）

### AOP 相关注解

* `@Aspect`：定义切面类

标记类为切面，定义横切关注点。

场景：日志、权限等横切逻辑。

```java
@Aspect
@Component
public class LoggingAspect {
    // 切面逻辑
}
```

* `@Before / @After / @Around`：通知方法

定义切点执行的时机。

```java
@Before("execution(* com.example.service.*.*(..))")
public void logBefore() {
    System.out.println("Method called");
}
```

* `@Pointcut`：定义切点表达式

简化切面配置。

```java
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceMethods() {}
```

### 注解元注解和自定义注解

* `@Retention`：注解生命周期（源码/编译期/运行时）

* `@Target`：注解使用范围（方法、字段、类等）

* `@Documented`：是否生成 `Javadoc`

* `@Inherited`：注解是否可被子类继承

### 用法示例

#### @CrossOrigin

用于处理 前后端分离 时的 跨域资源共享（`CORS`）问题。

浏览器基于安全考虑，默认会阻止来自不同源（协议 + 域名 + 端口）的请求。`@CrossOrigin` 可以显式允许跨域访问。

```java
@CrossOrigin(origins = "http://localhost:3000")
@RestController
public class MyController {
    
    @GetMapping("/hello")
    public String hello() {
        return "Hello from backend";
    }
}
```

常用属性：

* `origins`：允许的域名列表（支持多个）

* `methods`：允许的请求方式（如 `GET, POST`）

* `allowedHeaders`：允许的请求头

* `maxAge`：预检请求缓存时间，单位秒

推荐做法：

统一配置跨域规则而不是在控制器里逐个加 `@CrossOrigin`，例如通过配置类：

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("http://localhost:3000")
                .allowedMethods("*")
                .allowedHeaders("*")
                .allowCredentials(true);
    }
}
```

#### @Transactional — 声明式事务管理

用于开启数据库事务，控制方法在事务中执行，支持回滚与提交。

```java
@Service
public class UserService {

    @Transactional
    public void createUser(User user) {
        userRepository.save(user);
        // 如果这里抛出 RuntimeException，事务会回滚
    }
}
```

默认行为：

* 默认只对 `RuntimeException` 或其子类回滚。

* 对 `checked` 异常不会自动回滚，需手动指定。

注意事项：

* 必须运行在 `Spring` 管理的 代理对象 上，不能直接调用同类中的 `@Transactional` 方法。

* 常见失效场景：`private` 方法、`this.xxx()` 调用自身、未被 `Spring` 扫描等。

#### @Scheduled — 定时任务

用于计划任务调度，可配置任务在固定时间/间隔执行。

启用定时任务

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

创建定时任务类

```java
package com.example.demo.task;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;

@Component
public class MyTask {

    @Scheduled(fixedRate = 10000)
    public void task1() {
        System.out.println("每10秒执行一次: " + LocalDateTime.now());
    }

    @Scheduled(cron = "0/30 * * * * ?")
    public void task2() {
        System.out.println("每30秒执行一次（cron 表达式）: " + LocalDateTime.now());
    }
}
```

常用属性：

* `fixedRate`：每隔多久执行一次，单位毫秒（从上次开始算）

* `fixedDelay`：每隔多久执行一次（从上次结束算）

* `cron`：使用 `cron` 表达式执行任务

`Cron` 表达式格式：

```shell
秒 分 时 日 月 星期 年（可选）
```

#### @Configuration + @Bean

```java
@Configuration
public class AppConfig {

    @Bean
    public String defaultMessage() {
        return "默认消息 Bean";
    }
}
```

注入使用：

```java
@Autowired
private String defaultMessage;

@GetMapping("/bean")
public String beanMessage() {
    return defaultMessage;
}
```

#### @Value、@Component

```java
@Component
public class PropertyReader {

    @Value("${custom.welcome:Hello Default}")
    private String welcomeMessage;

    public String getWelcomeMessage() {
        return welcomeMessage;
    }
}
```

`application.yml` 配置：

```yaml
custom:
  welcome: 欢迎使用 Spring Boot
```

控制器调用：

```java
@Autowired
private PropertyReader reader;

@GetMapping("/welcome")
public String welcome() {
    return reader.getWelcomeMessage();
}
```

#### @Autowired + @Qualifier

目标：有两个实现类，手动指定注入其中一个

**接口**

```java
public interface MessageSender {
    String send();
}
```

**实现类 A**

```java
@Component("emailSender")
public class EmailSender implements MessageSender {
    public String send() {
        return "Email sent";
    }
}
```

**实现类 B**

```java
@Component("smsSender")
public class SmsSender implements MessageSender {
    public String send() {
        return "SMS sent";
    }
}
```

使用注入

```java
@RestController
public class SenderController {

    @Autowired
    @Qualifier("smsSender")
    private MessageSender sender;

    @GetMapping("/send")
    public String send() {
        return sender.send();
    }
}
```

#### @EnableAsync + @Async

在 `Spring Boot` 中使用异步方法，标准步骤：

* 启用异步功能：添加 `@EnableAsync`（一般加在主类或配置类上）

* 将方法标记为异步：加上 `@Async`

* 返回类型为 `void`、`Future<T>`、或更推荐的 `CompletableFuture<T>`

|  返回类型   |  使用方式   |  是否阻塞   |  获取结果方式   |
| --- | --- | --- | --- |
|  `void`   |  只执行，不关心返回   |  否   |  	无   |
|  `CompletableFuture<T>`   |  可选择异步/阻塞等待   |  否 / 是   |  `.thenApply() / .get()`  |
|  `Future<T>`   |  必须调用 `.get()` 获取结果（阻塞）   |  是   |  `.get()`   |

用 `CompletableFuture<T>`，更灵活，支持异步链式调用或 `.get()` 阻塞等待。


**和 .NET 中 async/await 对比**

|  特性   | Java Spring (@Async)    | .NET C# (async/await)    |
| --- | --- | --- |
|  语法简洁性   | 稍繁琐，要显式 .get()    |  极简洁，直接用 `await`   |
|  支持异步链式调用   |  `CompletableFuture` 支持   |  `Task` 和 `LINQ` 查询语法完美融合   |
|  线程池控制   |  可配置线程池 `Executor`   |  默认线程池，支持自定义   |
| 方法声明限制    |  必须是 `Spring Bean` 的方法，不能内部调用   |  可自由组合（类方法、静态方法等）   |
|  异常传播   |  需要手动处理 `.get()` 抛出的异常   |  可以用 `try-catch` 正常处理   |
|  用于 `Web` 控制器时效果   |  不能直接异步响应 `Web`（除非改为 `WebFlux`）   |  完整异步模型，控制器方法可 `async` 返回响应   |

> `@Async` 更适合“后台异步任务处理”场景，不适合异步 `Web` 返回响应（需要用 `Spring WebFlux`）。

**示例一：返回void**

```java
@Service
public class AsyncService {
    @Async
    public void runAsync() {
        System.out.println("异步线程执行中：" + Thread.currentThread().getName());
    }
}
```

```java
@EnableAsync
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

调用异步方法：

```java
@GetMapping("/async")
public String async() {
    asyncService.runAsync();
    return "触发异步调用";
}
```

**示例二：使用 `CompletableFuture<T>` 返回结果**

可以像 `.NET async/await` 一样“一步获取结果”，只不过 `Spring` 的 `.get()` 是阻塞的。

```java
@Async
public CompletableFuture<String> doTask() {
    return CompletableFuture.completedFuture("结果：" + LocalDateTime.now());
}

@GetMapping("/result")
public String getResult() throws Exception {
    return asyncService.doTask().get(); // 阻塞等待结果返回
}
```

**示例三：使用 `Future<T>`**

使用 `ExecutorService` 提交任务并获取结果

```java
ExecutorService executor = Executors.newFixedThreadPool(1);

Future<String> future = executor.submit(() -> {
    Thread.sleep(1000);
    return "任务完成";
});

// 阻塞等待
String result = future.get(); // throws InterruptedException, ExecutionException
System.out.println(result);
executor.shutdown();
```

局限：

* `.get()` 会阻塞；

* 无法设置回调；

* 无法链式执行；

* 无异常链式处理。

**示例四：使用`CompletableFuture<T>` 的 `supplyAsync` 方法**

##### 基本异步任务执行

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "Hello";
});
String result = future.get(); // 阻塞等待
```

##### 异步任务 + 消费结果 

```java
CompletableFuture<Void> future = CompletableFuture
    .supplyAsync(() -> "hello")
    .thenAccept(result -> System.out.println("结果是：" + result));
```

##### 异步任务 + 转换结果（链式调用）

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> "5")
    .thenApply(Integer::parseInt)
    .thenApply(num -> num * 2)
    .thenApply(Object::toString);
```

##### 异常处理

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

##### 多任务并发组合（allOf / anyOf）

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "A");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "B");

// 等待全部完成
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2);
all.join();

System.out.println("结果：" + f1.join() + ", " + f2.join());
```

##### 合并两个任务结果

```java
CompletableFuture<Integer> f1 = CompletableFuture.supplyAsync(() -> 100);
CompletableFuture<Integer> f2 = CompletableFuture.supplyAsync(() -> 200);

CompletableFuture<Integer> result = f1.thenCombine(f2, Integer::sum);
System.out.println(result.get()); // 输出 300
```

##### 自定义线程池

```java
ExecutorService pool = Executors.newFixedThreadPool(4);

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "线程池中的任务";
}, pool);

System.out.println(future.get());
pool.shutdown();
```

##### 模拟后台任务进度查询

```java
@RestController
public class TaskController {

    @Autowired
    private TaskService taskService;

    @GetMapping("/run")
    public String run() throws Exception {
        CompletableFuture<String> task = taskService.doTask();
        return task.get(); // 等待完成
    }
}

@Service
public class TaskService {

    @Async
    public CompletableFuture<String> doTask() throws InterruptedException {
        Thread.sleep(3000);
        return CompletableFuture.completedFuture("任务完成！");
    }
}
```