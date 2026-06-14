### 简介

`Spring WebFlux` 是 Spring Framework 5 引入的响应式 Web 框架。

它和常见的 `Spring MVC` 都能写 HTTP 接口，但底层处理模型不一样。

简单理解：

```text
Spring MVC
  |
  v
Servlet API
  |
  v
一个请求通常占用一个工作线程
```

```text
Spring WebFlux
  |
  v
Reactive Streams
  |
  v
少量事件循环线程处理大量异步 I/O
```

WebFlux 的核心不是把 Controller 返回值换成 `Mono`、`Flux` 这么简单。

更准确地说：

```text
WebFlux 适合把 HTTP、数据库、缓存、远程调用这些 I/O 操作串成非阻塞的数据流。
```

一句话概括：

```text
Spring WebFlux 是 Spring 的响应式 Web 框架，适合高并发 I/O、流式接口、网关转发和响应式数据访问场景。
```

### WebFlux 解决什么问题

传统阻塞式 Web 接口大致是这样：

```text
请求进来
  |
  v
分配线程
  |
  v
查询数据库 / 调用远程接口
  |
  v
线程等待结果
  |
  v
返回响应
```

如果请求里大量时间都花在等待 I/O，线程会被占着。

WebFlux 的思路是：

```text
请求进来
  |
  v
发起异步 I/O
  |
  v
线程释放出来处理其他请求
  |
  v
数据返回后继续处理
  |
  v
返回响应
```

适合的场景：

* 网关、BFF、聚合接口
* SSE 流式推送
* WebSocket
* 高并发远程接口调用
* 响应式数据库访问，比如 R2DBC
* 请求量很大、I/O 等待明显的服务

不太适合的场景：

* 主要是普通后台 CRUD
* 大量同步 JDBC、JPA 调用
* 大量 CPU 密集型计算
* 团队对 Reactor 操作符不熟

### WebFlux 和 Spring MVC 的区别

| 对比项 | Spring MVC | Spring WebFlux |
| --- | --- | --- |
| 编程模型 | 命令式、同步阻塞 | 响应式、异步非阻塞 |
| 常见返回值 | `User`、`List<User>` | `Mono<User>`、`Flux<User>` |
| 常见服务器 | Tomcat、Jetty、Undertow | Reactor Netty，也可运行在 Servlet 容器上 |
| 数据访问 | JDBC、JPA、MyBatis | R2DBC、Reactive Repository |
| HTTP 客户端 | `RestTemplate`、`RestClient` | `WebClient` |
| 典型场景 | 常规业务系统 | 高并发 I/O、流式数据、接口聚合 |

两个框架不是高低关系。

更像是两种不同工具：

```text
Spring MVC：写普通业务接口简单直接
Spring WebFlux：处理异步 I/O 和流式数据更自然
```

### 核心类型：Mono 和 Flux

WebFlux 基于 Project Reactor。

最常见的两个类型是：

| 类型 | 含义 | 常见场景 |
| --- | --- | --- |
| `Mono<T>` | 0 或 1 个元素 | 查询单条数据、创建结果、删除结果 |
| `Flux<T>` | 0 到 N 个元素 | 查询列表、流式推送、批量处理 |

`Mono<User>` 可以理解成：

```text
未来某个时间返回 0 个或 1 个 User。
```

`Flux<User>` 可以理解成：

```text
未来某个时间开始，陆续返回多个 User。
```

### 常用操作符

| 操作符 | 作用 |
| --- | --- |
| `map` | 同步转换 |
| `flatMap` | 异步转换 |
| `filter` | 过滤数据 |
| `switchIfEmpty` | 空结果兜底 |
| `defaultIfEmpty` | 空结果返回默认值 |
| `onErrorResume` | 异常兜底 |
| `doOnNext` | 数据经过时做附加动作 |
| `zip` | 合并多个异步结果 |
| `timeout` | 设置超时时间 |

示例：

```java
Mono<String> result = Mono.just("spring")
        .map(String::toUpperCase);
```

结果：

```text
SPRING
```

异步转换用 `flatMap`：

```java
Mono<UserProfile> profile = userRepository.findById(1L)
        .flatMap(user -> profileClient.findByUserId(user.id()));
```

查询列表用 `Flux`：

```java
Flux<String> names = userRepository.findAll()
        .filter(user -> user.age() >= 18)
        .map(User::username);
```

### Maven 依赖

Spring Boot 项目直接引入 WebFlux starter：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

测试依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

如果需要响应式数据库访问，可以引入 Spring Data R2DBC。

以 MySQL 为例：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>

<dependency>
    <groupId>io.asyncer</groupId>
    <artifactId>r2dbc-mysql</artifactId>
    <scope>runtime</scope>
</dependency>
```

常见配置：

```yaml
spring:
  r2dbc:
    url: r2dbc:mysql://localhost:3306/webflux_demo
    username: root
    password: 123456

server:
  port: 8080
```

如果项目里同时引入了：

```text
spring-boot-starter-web
spring-boot-starter-webflux
```

Spring Boot 通常会按 Servlet Web 应用启动。

如果目标是纯 WebFlux 应用，依赖里保留 `spring-boot-starter-webflux` 更清晰。

### 准备数据库

```sql
CREATE DATABASE webflux_demo DEFAULT CHARACTER SET utf8mb4;

USE webflux_demo;

DROP TABLE IF EXISTS users;

CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL,
  age INT NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at DATETIME NOT NULL,
  UNIQUE KEY uk_users_email (email)
);

INSERT INTO users (username, email, age, status, created_at) VALUES
('张三', 'zhangsan@example.com', 20, 'ACTIVE', '2026-01-01 10:00:00'),
('李四', 'lisi@example.com', 25, 'ACTIVE', '2026-01-02 10:00:00'),
('王五', 'wangwu@example.com', 17, 'DISABLED', '2026-01-03 10:00:00');
```

### 启动类

```java
package com.example.webfluxdemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class WebFluxDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(WebFluxDemoApplication.class, args);
    }
}
```

### 实体类

Spring Data R2DBC 使用 `@Table` 和 `@Id` 映射表。

```java
package com.example.webfluxdemo.user;

import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;

import java.time.LocalDateTime;

@Table("users")
public class User {

    @Id
    private Long id;
    private String username;
    private String email;
    private Integer age;
    private String status;
    private LocalDateTime createdAt;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }

    public LocalDateTime getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(LocalDateTime createdAt) {
        this.createdAt = createdAt;
    }
}
```

### DTO

接口入参和出参单独定义，避免直接把数据库实体暴露给接口。

```java
package com.example.webfluxdemo.user;

public record UserCreateRequest(
        String username,
        String email,
        Integer age
) {
}
```

```java
package com.example.webfluxdemo.user;

public record UserResponse(
        Long id,
        String username,
        String email,
        Integer age,
        String status
) {

    public static UserResponse from(User user) {
        return new UserResponse(
                user.getId(),
                user.getUsername(),
                user.getEmail(),
                user.getAge(),
                user.getStatus()
        );
    }
}
```

### Repository

`ReactiveCrudRepository` 返回的是 `Mono` 和 `Flux`。

```java
package com.example.webfluxdemo.user;

import org.springframework.data.r2dbc.repository.Query;
import org.springframework.data.repository.reactive.ReactiveCrudRepository;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

public interface UserRepository extends ReactiveCrudRepository<User, Long> {

    Mono<User> findByEmail(String email);

    Flux<User> findByStatus(String status);

    @Query("""
            select *
            from users
            where (:status is null or status = :status)
              and (:keyword is null or username like concat('%', :keyword, '%'))
            order by id desc
            limit :size offset :offset
            """)
    Flux<User> search(String status, String keyword, int size, long offset);
}
```

### Service

业务层负责把 Repository 返回的响应式类型继续组合。

```java
package com.example.webfluxdemo.user;

import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.time.LocalDateTime;

@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public Mono<UserResponse> create(UserCreateRequest request) {
        return userRepository.findByEmail(request.email())
                .flatMap(user -> Mono.<User>error(new EmailAlreadyExistsException(request.email())))
                .switchIfEmpty(Mono.defer(() -> {
                    User user = new User();
                    user.setUsername(request.username());
                    user.setEmail(request.email());
                    user.setAge(request.age());
                    user.setStatus("ACTIVE");
                    user.setCreatedAt(LocalDateTime.now());
                    return userRepository.save(user);
                }))
                .map(UserResponse::from);
    }

    public Mono<UserResponse> findById(Long id) {
        return userRepository.findById(id)
                .switchIfEmpty(Mono.error(new UserNotFoundException(id)))
                .map(UserResponse::from);
    }

    public Flux<UserResponse> findAll() {
        return userRepository.findAll()
                .map(UserResponse::from);
    }

    public Flux<UserResponse> search(String status, String keyword, int page, int size) {
        long offset = (long) Math.max(page - 1, 0) * size;
        return userRepository.search(status, keyword, size, offset)
                .map(UserResponse::from);
    }

    public Mono<UserResponse> disable(Long id) {
        return userRepository.findById(id)
                .switchIfEmpty(Mono.error(new UserNotFoundException(id)))
                .flatMap(user -> {
                    user.setStatus("DISABLED");
                    return userRepository.save(user);
                })
                .map(UserResponse::from);
    }

    public Mono<Void> deleteById(Long id) {
        return userRepository.findById(id)
                .switchIfEmpty(Mono.error(new UserNotFoundException(id)))
                .flatMap(userRepository::delete);
    }
}
```

异常类：

```java
package com.example.webfluxdemo.user;

public class UserNotFoundException extends RuntimeException {

    public UserNotFoundException(Long id) {
        super("用户不存在，id=" + id);
    }
}
```

```java
package com.example.webfluxdemo.user;

public class EmailAlreadyExistsException extends RuntimeException {

    public EmailAlreadyExistsException(String email) {
        super("邮箱已存在，email=" + email);
    }
}
```

### 注解式 Controller

WebFlux 支持和 Spring MVC 很像的注解式写法。

区别主要在返回值：

```text
单个结果：Mono<T>
多个结果：Flux<T>
```

```java
package com.example.webfluxdemo.user;

import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.time.Duration;

@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<UserResponse> create(@RequestBody UserCreateRequest request) {
        return userService.create(request);
    }

    @GetMapping("/{id}")
    public Mono<UserResponse> findById(@PathVariable Long id) {
        return userService.findById(id);
    }

    @GetMapping
    public Flux<UserResponse> findAll() {
        return userService.findAll();
    }

    @GetMapping("/search")
    public Flux<UserResponse> search(
            @RequestParam(required = false) String status,
            @RequestParam(required = false) String keyword,
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "10") int size
    ) {
        return userService.search(status, keyword, page, size);
    }

    @PutMapping("/{id}/disable")
    public Mono<UserResponse> disable(@PathVariable Long id) {
        return userService.disable(id);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public Mono<Void> deleteById(@PathVariable Long id) {
        return userService.deleteById(id);
    }

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<UserResponse> stream() {
        return userService.findAll()
                .delayElements(Duration.ofSeconds(1));
    }
}
```

### 接口测试

创建用户：

```text
POST http://localhost:8080/api/users
Content-Type: application/json

{
  "username": "赵六",
  "email": "zhaoliu@example.com",
  "age": 28
}
```

查询单个用户：

```text
GET http://localhost:8080/api/users/1
```

查询列表：

```text
GET http://localhost:8080/api/users
```

分页条件查询：

```text
GET http://localhost:8080/api/users/search?status=ACTIVE&keyword=张&page=1&size=10
```

流式接口：

```text
GET http://localhost:8080/api/users/stream
```

`stream` 接口返回的是 `text/event-stream`。

浏览器或支持 SSE 的客户端可以持续接收服务端推送的数据。

### 函数式路由

WebFlux 还支持函数式端点。

这种写法把路由和处理逻辑分开。

```text
RouterFunction：负责路由
Handler：负责处理请求
```

Handler：

```java
package com.example.webfluxdemo.user;

import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.server.ServerRequest;
import org.springframework.web.reactive.function.server.ServerResponse;
import reactor.core.publisher.Mono;

@Component
public class UserHandler {

    private final UserService userService;

    public UserHandler(UserService userService) {
        this.userService = userService;
    }

    public Mono<ServerResponse> findById(ServerRequest request) {
        Long id = Long.valueOf(request.pathVariable("id"));
        return userService.findById(id)
                .flatMap(user -> ServerResponse.ok().bodyValue(user));
    }

    public Mono<ServerResponse> create(ServerRequest request) {
        return request.bodyToMono(UserCreateRequest.class)
                .flatMap(userService::create)
                .flatMap(user -> ServerResponse
                        .created(request.uriBuilder().path("/{id}").build(user.id()))
                        .bodyValue(user));
    }

    public Mono<ServerResponse> stream(ServerRequest request) {
        return ServerResponse.ok()
                .contentType(MediaType.TEXT_EVENT_STREAM)
                .body(userService.findAll(), UserResponse.class);
    }
}
```

Router：

```java
package com.example.webfluxdemo.user;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.server.RouterFunction;
import org.springframework.web.reactive.function.server.RouterFunctions;
import org.springframework.web.reactive.function.server.ServerResponse;

@Configuration
public class UserRouter {

    @Bean
    public RouterFunction<ServerResponse> userRoutes(UserHandler handler) {
        return RouterFunctions.route()
                .GET("/fn/users/{id}", handler::findById)
                .POST("/fn/users", handler::create)
                .GET("/fn/users/stream", handler::stream)
                .build();
    }
}
```

函数式路由适合：

* 网关类接口
* 路由很多、需要集中管理的接口
* 更偏函数组合风格的项目

普通业务项目使用注解式 Controller 也很常见。

### WebClient

`WebClient` 是 Spring 提供的响应式 HTTP 客户端。

它适合在 WebFlux 项目里调用远程 HTTP 服务。

配置：

```java
package com.example.webfluxdemo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class WebClientConfig {

    @Bean
    public WebClient userApiClient(WebClient.Builder builder) {
        return builder
                .baseUrl("https://user-api.example.com")
                .defaultHeader("X-App-Name", "webflux-demo")
                .build();
    }
}
```

调用单个接口：

```java
package com.example.webfluxdemo.remote;

import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.time.Duration;

@Component
public class RemoteUserClient {

    private final WebClient userApiClient;

    public RemoteUserClient(WebClient userApiClient) {
        this.userApiClient = userApiClient;
    }

    public Mono<RemoteUser> findById(Long id) {
        return userApiClient.get()
                .uri("/users/{id}", id)
                .retrieve()
                .bodyToMono(RemoteUser.class)
                .timeout(Duration.ofSeconds(2));
    }

    public Flux<RemoteUser> findAll() {
        return userApiClient.get()
                .uri("/users")
                .retrieve()
                .bodyToFlux(RemoteUser.class);
    }
}
```

远程 DTO：

```java
package com.example.webfluxdemo.remote;

public record RemoteUser(
        Long id,
        String username,
        String email
) {
}
```

### WebClient 错误处理

远程接口返回 `4xx`、`5xx` 时，可以使用 `onStatus` 转成业务异常。

```java
public Mono<RemoteUser> findById(Long id) {
    return userApiClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .onStatus(
                    status -> status.value() == 404,
                    response -> Mono.error(new RemoteUserNotFoundException(id))
            )
            .onStatus(
                    status -> status.is5xxServerError(),
                    response -> response.bodyToMono(String.class)
                            .defaultIfEmpty("")
                            .flatMap(body -> Mono.error(new RemoteServiceException(body)))
            )
            .bodyToMono(RemoteUser.class)
            .timeout(Duration.ofSeconds(2));
}
```

异常类：

```java
package com.example.webfluxdemo.remote;

public class RemoteUserNotFoundException extends RuntimeException {

    public RemoteUserNotFoundException(Long id) {
        super("远程用户不存在，id=" + id);
    }
}
```

```java
package com.example.webfluxdemo.remote;

public class RemoteServiceException extends RuntimeException {

    public RemoteServiceException(String body) {
        super("远程服务异常：" + body);
    }
}
```

### 并发调用多个接口

`Mono.zip` 可以合并多个异步结果。

比如一个用户详情页需要：

```text
用户基础信息
账户信息
最近订单
```

可以这样组合：

```java
public Mono<UserDetailResponse> findDetail(Long userId) {
    Mono<UserResponse> userMono = userService.findById(userId);
    Mono<AccountResponse> accountMono = accountClient.findByUserId(userId);
    Mono<OrderSummaryResponse> orderMono = orderClient.findRecentSummary(userId);

    return Mono.zip(userMono, accountMono, orderMono)
            .map(tuple -> new UserDetailResponse(
                    tuple.getT1(),
                    tuple.getT2(),
                    tuple.getT3()
            ));
}
```

只要三个调用之间没有依赖关系，就可以并发发起。

### 全局异常处理

WebFlux 也可以使用 `@RestControllerAdvice` 处理异常。

```java
package com.example.webfluxdemo.common;

import com.example.webfluxdemo.remote.RemoteServiceException;
import com.example.webfluxdemo.remote.RemoteUserNotFoundException;
import com.example.webfluxdemo.user.EmailAlreadyExistsException;
import com.example.webfluxdemo.user.UserNotFoundException;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import reactor.core.publisher.Mono;

import java.time.LocalDateTime;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Mono<ApiError> handleUserNotFound(UserNotFoundException exception) {
        return Mono.just(ApiError.of(404, exception.getMessage()));
    }

    @ExceptionHandler(RemoteUserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Mono<ApiError> handleRemoteUserNotFound(RemoteUserNotFoundException exception) {
        return Mono.just(ApiError.of(404, exception.getMessage()));
    }

    @ExceptionHandler(EmailAlreadyExistsException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public Mono<ApiError> handleEmailAlreadyExists(EmailAlreadyExistsException exception) {
        return Mono.just(ApiError.of(409, exception.getMessage()));
    }

    @ExceptionHandler(RemoteServiceException.class)
    @ResponseStatus(HttpStatus.BAD_GATEWAY)
    public Mono<ApiError> handleRemoteService(RemoteServiceException exception) {
        return Mono.just(ApiError.of(502, exception.getMessage()));
    }

    public record ApiError(
            Integer code,
            String message,
            LocalDateTime timestamp
    ) {

        public static ApiError of(Integer code, String message) {
            return new ApiError(code, message, LocalDateTime.now());
        }
    }
}
```

### 响应式链路里的异常兜底

局部兜底可以使用 `onErrorResume`。

```java
public Mono<UserResponse> findByIdWithFallback(Long id) {
    return userService.findById(id)
            .onErrorResume(UserNotFoundException.class, exception -> {
                UserResponse fallback = new UserResponse(
                        -1L,
                        "默认用户",
                        "default@example.com",
                        0,
                        "UNKNOWN"
                );
                return Mono.just(fallback);
            });
}
```

如果只是记录日志，可以使用 `doOnError`：

```java
public Mono<UserResponse> findById(Long id) {
    return userService.findById(id)
            .doOnError(exception -> log.error("查询用户失败，id={}", id, exception));
}
```

`doOnError` 不会吞掉异常。

异常仍会继续向后传播。

### 阻塞代码的处理方式

WebFlux 的价值来自非阻塞链路。

如果在响应式链路里直接执行 JDBC、JPA、文件读取、老 SDK 同步调用，就会占用事件循环线程。

临时接入阻塞代码时，可以把它放到 `boundedElastic` 调度器。

```java
public Mono<UserResponse> findFromOldJdbcService(Long id) {
    return Mono.fromCallable(() -> oldJdbcUserService.findById(id))
            .subscribeOn(Schedulers.boundedElastic())
            .map(UserResponse::from);
}
```

需要导入：

```java
import reactor.core.scheduler.Schedulers;
```

这只是兼容方式。

如果核心链路大量依赖 JDBC、JPA、MyBatis，Spring MVC 往往更直接。

### block 的使用边界

`block()` 会把响应式调用转成同步等待。

示例：

```java
UserResponse user = userService.findById(1L).block();
```

它适合出现在：

* 命令行程序
* 初始化脚本
* 少量测试代码

业务接口里频繁使用 `block()`，会把非阻塞链路重新变成阻塞等待。

### SSE 流式推送

SSE 适合服务端持续推送单向消息。

Controller 写法：

```java
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> events() {
    return Flux.interval(Duration.ofSeconds(1))
            .map(index -> "event-" + index)
            .take(10);
}
```

浏览器访问：

```text
GET http://localhost:8080/api/users/events
```

每秒会收到一条数据。

适合场景：

* 任务进度
* 监控指标
* 通知消息
* 日志流

### WebTestClient 测试

WebFlux 常用 `WebTestClient` 测试接口。

```java
package com.example.webfluxdemo.user;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.reactive.server.WebTestClient;
import reactor.core.publisher.Mono;

import static org.mockito.Mockito.when;

@WebFluxTest(UserController.class)
class UserControllerTest {

    @Autowired
    private WebTestClient webTestClient;

    @MockitoBean
    private UserService userService;

    @Test
    void shouldFindUserById() {
        UserResponse response = new UserResponse(
                1L,
                "张三",
                "zhangsan@example.com",
                20,
                "ACTIVE"
        );

        when(userService.findById(1L)).thenReturn(Mono.just(response));

        webTestClient.get()
                .uri("/api/users/1")
                .exchange()
                .expectStatus().isOk()
                .expectBody()
                .jsonPath("$.id").isEqualTo(1)
                .jsonPath("$.username").isEqualTo("张三")
                .jsonPath("$.status").isEqualTo("ACTIVE");
    }
}
```

如果项目使用的 Spring Boot 版本还没有 `@MockitoBean`，可以使用同类测试能力里的 `@MockBean`。

### StepVerifier 测试 Mono 和 Flux

`StepVerifier` 用来测试 Reactor 流。

```java
package com.example.webfluxdemo.user;

import org.junit.jupiter.api.Test;
import reactor.core.publisher.Flux;
import reactor.test.StepVerifier;

class ReactorTest {

    @Test
    void shouldFilterActiveUserNames() {
        Flux<String> names = Flux.just(
                        new UserResponse(1L, "张三", "zhangsan@example.com", 20, "ACTIVE"),
                        new UserResponse(2L, "李四", "lisi@example.com", 25, "DISABLED"),
                        new UserResponse(3L, "王五", "wangwu@example.com", 18, "ACTIVE")
                )
                .filter(user -> "ACTIVE".equals(user.status()))
                .map(UserResponse::username);

        StepVerifier.create(names)
                .expectNext("张三")
                .expectNext("王五")
                .verifyComplete();
    }
}
```

### 常见使用建议

### 保持链路非阻塞

WebFlux 项目里，HTTP、数据库、缓存、消息队列都尽量使用响应式客户端。

常见搭配：

| 类型 | 响应式选择 |
| --- | --- |
| HTTP | `WebClient` |
| 数据库 | R2DBC |
| Redis | Reactive Redis |
| MongoDB | Reactive MongoDB |
| 消息处理 | Reactor、响应式驱动 |

### 区分 map 和 flatMap

同步转换使用 `map`。

```java
Mono<String> username = userService.findById(1L)
        .map(UserResponse::username);
```

返回值本身还是 `Mono` 时，使用 `flatMap`。

```java
Mono<AccountResponse> account = userService.findById(1L)
        .flatMap(user -> accountClient.findByUserId(user.id()));
```

### 空结果使用 switchIfEmpty

查询不到数据时，可以转成异常。

```java
public Mono<UserResponse> findById(Long id) {
    return userRepository.findById(id)
            .switchIfEmpty(Mono.error(new UserNotFoundException(id)))
            .map(UserResponse::from);
}
```

也可以返回默认值。

```java
public Mono<String> findDisplayName(Long id) {
    return userRepository.findById(id)
            .map(User::getUsername)
            .defaultIfEmpty("匿名用户");
}
```

### 控制并发数量

`flatMap` 可以并发处理多个异步任务。

第二个参数可以限制并发数量。

```java
public Flux<UserResponse> enrichUsers(Flux<UserResponse> users) {
    return users.flatMap(
            user -> profileClient.fillProfile(user),
            8
    );
}
```

这类限制适合远程服务保护、批量任务处理等场景。

### 设置超时

远程调用建议设置超时。

```java
public Mono<RemoteUser> findRemoteUser(Long id) {
    return remoteUserClient.findById(id)
            .timeout(Duration.ofSeconds(2));
}
```

结合兜底：

```java
public Mono<RemoteUser> findRemoteUser(Long id) {
    return remoteUserClient.findById(id)
            .timeout(Duration.ofSeconds(2))
            .onErrorResume(exception -> Mono.empty());
}
```

### 常用方法汇总

| 方法 | 作用 |
| --- | --- |
| `Mono.just(value)` | 创建单值流 |
| `Mono.empty()` | 创建空流 |
| `Mono.error(error)` | 创建异常流 |
| `Flux.just(...)` | 创建多值流 |
| `Flux.fromIterable(list)` | 从集合创建流 |
| `map(...)` | 同步转换 |
| `flatMap(...)` | 异步转换 |
| `filter(...)` | 过滤元素 |
| `switchIfEmpty(...)` | 空结果处理 |
| `defaultIfEmpty(...)` | 空结果默认值 |
| `onErrorResume(...)` | 异常兜底 |
| `timeout(...)` | 超时控制 |
| `Mono.zip(...)` | 合并多个单值异步结果 |
| `delayElements(...)` | 延迟发送元素 |
| `subscribeOn(...)` | 指定订阅执行调度器 |
| `WebClient.retrieve()` | 发起请求并提取响应体 |
| `WebTestClient.exchange()` | 执行测试请求 |

### 总结

`Spring WebFlux` 的重点不是语法新，而是处理模型变了。

它把一次接口请求拆成一条响应式数据流：

```text
接收请求
  |
  v
读取参数
  |
  v
查询数据或调用远程接口
  |
  v
转换结果
  |
  v
处理异常
  |
  v
返回响应
```

适合 WebFlux 的项目，通常有明显的异步 I/O、流式响应或接口聚合需求。

如果只是普通 CRUD，Spring MVC 依然是简单直接的选择。

如果使用 WebFlux，数据库、缓存、HTTP 客户端也尽量选择响应式版本，这样才能把非阻塞链路真正串起来。
