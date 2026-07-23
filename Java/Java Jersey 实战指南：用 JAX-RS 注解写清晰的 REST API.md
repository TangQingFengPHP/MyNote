### 简介

Jersey 是 Jakarta RESTful Web Services 规范的一种实现。

老名字常叫 JAX-RS，新包名是：

```java
jakarta.ws.rs
```

Jersey 自己的核心包名通常是：

```java
org.glassfish.jersey
```

简单理解：

```text
Jakarta REST / JAX-RS 是规范
Jersey 是实现
Spring Boot 提供 Jersey 自动配置和 starter
```

Jersey 的开发方式是用注解把 Java 类声明成 HTTP 资源：

```java
@Path("/users")
public class UserResource {

    @GET
    @Path("/{id}")
    public UserView getById(@PathParam("id") Long id) {
        return userService.getById(id);
    }
}
```

这种写法和 Spring MVC 的 Controller 很像，只是注解来自 Jakarta REST 规范。

### Jersey 适合什么场景

Java 写 REST API 常见有几种方式：

| 方式 | 说明 |
| --- | --- |
| Servlet | 最底层，直接处理 request / response |
| Spring MVC | Spring 生态里最常见的 Web 框架 |
| Jersey | Jakarta REST / JAX-RS 实现 |
| RESTEasy | 另一种 Jakarta REST / JAX-RS 实现 |
| Apache CXF | 支持 JAX-RS、JAX-WS 等能力 |

Jersey 适合这些场景：

* 已经在使用 JAX-RS / Jakarta REST 注解的项目
* 传统 Java EE / Jakarta EE 项目
* 需要兼容标准 REST 资源模型的服务
* Spring Boot 项目里更偏好 JAX-RS 编程模型
* 需要 Jersey Client、Filter、ExceptionMapper 等能力的项目

如果项目已经深度使用 Spring MVC，继续用 Spring MVC 通常更顺。

如果团队习惯 `@Path`、`@GET`、`@Produces` 这套标准注解，Jersey 会更自然。

### Jersey、JAX-RS、Jakarta REST 的关系

这些名字容易混。

| 名称 | 含义 |
| --- | --- |
| JAX-RS | Java RESTful Web Services 老称呼 |
| Jakarta RESTful Web Services | JAX-RS 迁到 Jakarta 后的新名字 |
| Jersey | Jakarta REST / JAX-RS 的实现 |
| RESTEasy | Jakarta REST / JAX-RS 的另一种实现 |
| `javax.ws.rs` | 老包名，常见于 Java EE 8、Spring Boot 2 项目 |
| `jakarta.ws.rs` | 新包名，常见于 Jakarta EE、Spring Boot 3 项目 |

Spring Boot 3 相关项目里通常使用：

```java
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
```

老项目里可能会看到：

```java
import javax.ws.rs.GET;
import javax.ws.rs.Path;
```

迁移到 Spring Boot 3 或 Jakarta EE 新版本时，包名变化是主要改动之一。

### Jersey 和 Spring MVC 对比

同一个用户查询接口，用 Spring MVC 可能这样写：

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public UserView getById(@PathVariable Long id) {
        return userService.getById(id);
    }
}
```

Jersey 写法：

```java
@Path("/users")
public class UserResource {

    @GET
    @Path("/{id}")
    public UserView getById(@PathParam("id") Long id) {
        return userService.getById(id);
    }
}
```

对比：

| 维度 | Spring MVC | Jersey |
| --- | --- | --- |
| 路由类注解 | `@RestController`、`@RequestMapping` | `@Path` |
| GET | `@GetMapping` | `@GET` |
| POST | `@PostMapping` | `@POST` |
| 路径参数 | `@PathVariable` | `@PathParam` |
| 查询参数 | `@RequestParam` | `@QueryParam` |
| 请求体 | `@RequestBody` | 方法参数直接接收实体 |
| 响应对象 | 直接返回对象或 `ResponseEntity` | 直接返回对象或 `Response` |

两者都能写 REST API。

区别不在“能不能写”，而在项目选择的编程模型和生态集成方式。

### Spring Boot 集成 Jersey

Spring Boot 项目里使用 Jersey，直接引入 starter。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jersey</artifactId>
</dependency>
```

如果需要参数校验：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

Spring Boot 会为 Jersey 做自动配置。

还需要提供一个 `ResourceConfig` Bean，用来注册资源类、过滤器、异常映射器等组件。

### ResourceConfig 配置

```java
package com.example.jersey.config;

import com.example.jersey.exception.ApiExceptionMapper;
import com.example.jersey.exception.ConstraintViolationMapper;
import com.example.jersey.filter.RequestLogFilter;
import com.example.jersey.resource.UserResource;
import org.glassfish.jersey.server.ResourceConfig;
import org.springframework.stereotype.Component;

@Component
public class JerseyConfig extends ResourceConfig {

    public JerseyConfig() {
        register(UserResource.class);
        register(ApiExceptionMapper.class);
        register(ConstraintViolationMapper.class);
        register(RequestLogFilter.class);
    }
}
```

Spring Boot 官方文档建议在可执行 jar 场景里显式 `register(...)` 端点。

原因是 Jersey 对可执行 jar 的包扫描支持有限，显式注册更稳定。

如果要给 Jersey 统一加路径前缀，可以在 `ResourceConfig` 上加 `@ApplicationPath`：

```java
package com.example.jersey.config;

import com.example.jersey.resource.UserResource;
import jakarta.ws.rs.ApplicationPath;
import org.glassfish.jersey.server.ResourceConfig;
import org.springframework.stereotype.Component;

@Component
@ApplicationPath("/api")
public class JerseyConfig extends ResourceConfig {

    public JerseyConfig() {
        register(UserResource.class);
    }
}
```

这样 `@Path("/users")` 最终访问路径就是：

```text
/api/users
```

### 第一个 Resource

```java
package com.example.jersey.resource;

import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.springframework.stereotype.Component;

@Component
@Path("/hello")
public class HelloResource {

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() {
        return "Hello Jersey";
    }
}
```

注册：

```java
register(HelloResource.class);
```

访问：

```text
GET http://localhost:8080/api/hello
```

如果 Resource 类加了 `@Component`，它可以交给 Spring 管理，也可以注入 Spring Bean。

### 用户 CRUD Demo

下面用一个用户接口串起常用注解。

请求对象：

```java
package com.example.jersey.user;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

public record CreateUserRequest(

        @NotBlank(message = "用户名不能为空")
        @Size(min = 2, max = 20, message = "用户名长度需要在2到20之间")
        String username,

        @NotBlank(message = "邮箱不能为空")
        @Email(message = "邮箱格式不正确")
        String email,

        @NotNull(message = "年龄不能为空")
        Integer age
) {
}
```

响应对象：

```java
package com.example.jersey.user;

public record UserView(
        Long id,
        String username,
        String email,
        Integer age
) {
}
```

业务服务：

```java
package com.example.jersey.user;

import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

@Service
public class UserService {

    private final AtomicLong idGenerator = new AtomicLong(1000);

    private final Map<Long, UserView> users = new ConcurrentHashMap<>();

    public List<UserView> findAll(String keyword) {
        return new ArrayList<>(users.values())
                .stream()
                .filter(user -> keyword == null || user.username().contains(keyword))
                .toList();
    }

    public UserView findById(Long id) {
        UserView user = users.get(id);
        if (user == null) {
            throw new ApiException(404, "用户不存在");
        }
        return user;
    }

    public UserView create(CreateUserRequest request) {
        Long id = idGenerator.incrementAndGet();
        UserView user = new UserView(id, request.username(), request.email(), request.age());
        users.put(id, user);
        return user;
    }

    public void delete(Long id) {
        users.remove(id);
    }
}
```

Resource：

```java
package com.example.jersey.resource;

import com.example.jersey.user.CreateUserRequest;
import com.example.jersey.user.UserService;
import com.example.jersey.user.UserView;
import jakarta.validation.Valid;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.DELETE;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.springframework.stereotype.Component;

import java.net.URI;
import java.util.List;

@Component
@Path("/users")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class UserResource {

    private final UserService userService;

    public UserResource(UserService userService) {
        this.userService = userService;
    }

    @GET
    public List<UserView> list(@QueryParam("keyword") String keyword) {
        return userService.findAll(keyword);
    }

    @GET
    @Path("/{id}")
    public UserView getById(@PathParam("id") Long id) {
        return userService.findById(id);
    }

    @POST
    public Response create(@Valid CreateUserRequest request) {
        UserView user = userService.create(request);
        return Response.created(URI.create("/api/users/" + user.id()))
                .entity(user)
                .build();
    }

    @DELETE
    @Path("/{id}")
    public Response delete(@PathParam("id") Long id) {
        userService.delete(id);
        return Response.noContent().build();
    }
}
```

接口效果：

| 请求 | 说明 |
| --- | --- |
| `GET /api/users` | 查询全部用户 |
| `GET /api/users?keyword=tom` | 按用户名关键字查询 |
| `GET /api/users/1001` | 根据 ID 查询 |
| `POST /api/users` | 创建用户 |
| `DELETE /api/users/1001` | 删除用户 |

POST 请求示例：

```json
{
  "username": "tom",
  "email": "tom@example.com",
  "age": 20
}
```

### 常用注解

Jersey 主要使用 Jakarta REST 注解。

| 注解 | 作用 |
| --- | --- |
| `@Path` | 定义类或方法路径 |
| `@GET` | 处理 GET 请求 |
| `@POST` | 处理 POST 请求 |
| `@PUT` | 处理 PUT 请求 |
| `@PATCH` | 处理 PATCH 请求 |
| `@DELETE` | 处理 DELETE 请求 |
| `@Produces` | 声明响应媒体类型 |
| `@Consumes` | 声明请求体媒体类型 |
| `@PathParam` | 获取路径参数 |
| `@QueryParam` | 获取查询参数 |
| `@HeaderParam` | 获取请求头 |
| `@CookieParam` | 获取 Cookie |
| `@FormParam` | 获取表单字段 |
| `@BeanParam` | 把多个参数封装到一个对象 |
| `@Context` | 注入上下文对象 |

`@Produces` 和 `@Consumes` 可以放在类上，也可以放在方法上。

方法上的配置优先级更高。

### 参数绑定

路径参数：

```java
@GET
@Path("/{id}")
public UserView getById(@PathParam("id") Long id) {
    return userService.findById(id);
}
```

查询参数：

```java
@GET
public List<UserView> list(@QueryParam("keyword") String keyword,
                           @QueryParam("page") @DefaultValue("1") Integer page) {
    return userService.findAll(keyword);
}
```

请求头：

```java
@GET
@Path("/profile")
public UserView profile(@HeaderParam("X-User-Id") Long userId) {
    return userService.findById(userId);
}
```

表单参数：

```java
@POST
@Path("/login")
@Consumes(MediaType.APPLICATION_FORM_URLENCODED)
public Response login(@FormParam("username") String username,
                      @FormParam("password") String password) {
    return Response.ok().build();
}
```

上下文对象：

```java
@GET
@Path("/request-info")
public String requestInfo(@Context jakarta.ws.rs.core.UriInfo uriInfo) {
    return uriInfo.getRequestUri().toString();
}
```

### BeanParam：封装查询条件

查询条件比较多时，可以用 `@BeanParam`。

```java
package com.example.jersey.user;

import jakarta.ws.rs.DefaultValue;
import jakarta.ws.rs.QueryParam;

public class UserQueryParam {

    @QueryParam("keyword")
    private String keyword;

    @QueryParam("page")
    @DefaultValue("1")
    private Integer page;

    @QueryParam("size")
    @DefaultValue("20")
    private Integer size;

    public String getKeyword() {
        return keyword;
    }

    public Integer getPage() {
        return page;
    }

    public Integer getSize() {
        return size;
    }
}
```

Resource：

```java
@GET
@Path("/search")
public List<UserView> search(@BeanParam UserQueryParam queryParam) {
    return userService.findAll(queryParam.getKeyword());
}
```

这种写法适合列表查询、分页查询、筛选条件较多的接口。

### Response：控制状态码和响应头

直接返回对象时，Jersey 会自动序列化。

需要精细控制状态码、响应头时，可以返回 `Response`。

创建成功：

```java
return Response.created(URI.create("/api/users/" + user.id()))
        .entity(user)
        .build();
```

无内容：

```java
return Response.noContent().build();
```

自定义响应头：

```java
return Response.ok(user)
        .header("X-Request-Source", "jersey")
        .build();
```

常见状态码：

| 场景 | 状态码 |
| --- | --- |
| 查询成功 | `200 OK` |
| 创建成功 | `201 Created` |
| 删除成功且无响应体 | `204 No Content` |
| 参数错误 | `400 Bad Request` |
| 未登录 | `401 Unauthorized` |
| 无权限 | `403 Forbidden` |
| 资源不存在 | `404 Not Found` |
| 服务端异常 | `500 Internal Server Error` |

### 异常处理：ExceptionMapper

Jersey 用 `ExceptionMapper` 统一处理异常。

先定义业务异常：

```java
package com.example.jersey.user;

public class ApiException extends RuntimeException {

    private final int status;

    public ApiException(int status, String message) {
        super(message);
        this.status = status;
    }

    public int getStatus() {
        return status;
    }
}
```

错误响应：

```java
package com.example.jersey.exception;

public record ApiErrorResponse(
        String code,
        String message
) {
}
```

异常映射器：

```java
package com.example.jersey.exception;

import com.example.jersey.user.ApiException;
import jakarta.ws.rs.ext.ExceptionMapper;
import jakarta.ws.rs.ext.Provider;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

@Provider
public class ApiExceptionMapper implements ExceptionMapper<ApiException> {

    @Override
    public Response toResponse(ApiException exception) {
        ApiErrorResponse body = new ApiErrorResponse(
                "BUSINESS_ERROR",
                exception.getMessage()
        );

        return Response.status(exception.getStatus())
                .type(MediaType.APPLICATION_JSON)
                .entity(body)
                .build();
    }
}
```

注册：

```java
register(ApiExceptionMapper.class);
```

这样 Resource 里可以直接抛业务异常，统一由 Mapper 转成 JSON 响应。

### 参数校验

Jersey 可以结合 Jakarta Validation。

请求对象上写约束：

```java
public record CreateUserRequest(

        @NotBlank(message = "用户名不能为空")
        String username,

        @Email(message = "邮箱格式不正确")
        String email
) {
}
```

Resource 参数上加 `@Valid`：

```java
@POST
public Response create(@Valid CreateUserRequest request) {
    UserView user = userService.create(request);
    return Response.ok(user).build();
}
```

校验异常也可以通过 `ExceptionMapper` 统一处理。

```java
package com.example.jersey.exception;

import jakarta.validation.ConstraintViolation;
import jakarta.validation.ConstraintViolationException;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.ExceptionMapper;
import jakarta.ws.rs.ext.Provider;

import java.util.List;

@Provider
public class ConstraintViolationMapper implements ExceptionMapper<ConstraintViolationException> {

    @Override
    public Response toResponse(ConstraintViolationException exception) {
        List<String> errors = exception.getConstraintViolations()
                .stream()
                .map(ConstraintViolation::getMessage)
                .toList();

        return Response.status(Response.Status.BAD_REQUEST)
                .entity(errors)
                .build();
    }
}
```

### Filter：请求日志和鉴权入口

Jersey 里常用过滤器处理请求日志、鉴权、TraceId 等逻辑。

```java
package com.example.jersey.filter;

import jakarta.ws.rs.container.ContainerRequestContext;
import jakarta.ws.rs.container.ContainerRequestFilter;
import jakarta.ws.rs.ext.Provider;

import java.io.IOException;

@Provider
public class RequestLogFilter implements ContainerRequestFilter {

    @Override
    public void filter(ContainerRequestContext requestContext) throws IOException {
        System.out.printf(
                "Jersey 请求：%s %s%n",
                requestContext.getMethod(),
                requestContext.getUriInfo().getPath()
        );
    }
}
```

注册：

```java
register(RequestLogFilter.class);
```

简单鉴权示例：

```java
package com.example.jersey.filter;

import jakarta.ws.rs.container.ContainerRequestContext;
import jakarta.ws.rs.container.ContainerRequestFilter;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.Provider;

import java.io.IOException;

@Provider
public class TokenAuthFilter implements ContainerRequestFilter {

    @Override
    public void filter(ContainerRequestContext requestContext) throws IOException {
        String token = requestContext.getHeaderString("Authorization");
        if (token == null || token.isBlank()) {
            requestContext.abortWith(
                    Response.status(Response.Status.UNAUTHORIZED)
                            .entity("未登录")
                            .build()
            );
        }
    }
}
```

复杂安全逻辑更适合交给 Spring Security 或网关统一处理。

### Jersey Client

Jersey 不只可以写服务端，也提供 Client API 调用 HTTP 服务。

Maven 依赖：

```xml
<dependency>
    <groupId>org.glassfish.jersey.core</groupId>
    <artifactId>jersey-client</artifactId>
    <version>${jersey.version}</version>
</dependency>

<dependency>
    <groupId>org.glassfish.jersey.media</groupId>
    <artifactId>jersey-media-json-jackson</artifactId>
    <version>${jersey.version}</version>
</dependency>
```

Client 示例：

```java
package com.example.jersey.client;

import jakarta.ws.rs.client.Client;
import jakarta.ws.rs.client.ClientBuilder;
import jakarta.ws.rs.client.Entity;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

public class JerseyClientDemo {

    public static void main(String[] args) {
        Client client = ClientBuilder.newClient();

        CreateUserRequest request = new CreateUserRequest(
                "tom",
                "tom@example.com",
                20
        );

        Response response = client.target("http://localhost:8080/api")
                .path("/users")
                .request(MediaType.APPLICATION_JSON)
                .post(Entity.entity(request, MediaType.APPLICATION_JSON));

        System.out.println(response.getStatus());
        System.out.println(response.readEntity(String.class));

        response.close();
        client.close();
    }
}
```

Spring 项目里也可以使用 `RestClient`、`WebClient`、OpenFeign 等客户端。

Jersey Client 更适合已经在 JAX-RS/Jersey 体系里的项目。

### 独立运行：Grizzly Demo

Jersey 也可以不依赖 Spring Boot，用 Grizzly 启动一个轻量 HTTP 服务。

依赖：

```xml
<dependency>
    <groupId>org.glassfish.jersey.containers</groupId>
    <artifactId>jersey-container-grizzly2-http</artifactId>
    <version>${jersey.version}</version>
</dependency>

<dependency>
    <groupId>org.glassfish.jersey.inject</groupId>
    <artifactId>jersey-hk2</artifactId>
    <version>${jersey.version}</version>
</dependency>

<dependency>
    <groupId>org.glassfish.jersey.media</groupId>
    <artifactId>jersey-media-json-jackson</artifactId>
    <version>${jersey.version}</version>
</dependency>
```

启动类：

```java
package com.example.jersey.standalone;

import org.glassfish.grizzly.http.server.HttpServer;
import org.glassfish.jersey.grizzly2.httpserver.GrizzlyHttpServerFactory;
import org.glassfish.jersey.server.ResourceConfig;

import java.io.IOException;
import java.net.URI;

public class JerseyStandaloneApplication {

    public static void main(String[] args) throws IOException {
        ResourceConfig config = new ResourceConfig()
                .register(HelloResource.class);

        HttpServer server = GrizzlyHttpServerFactory.createHttpServer(
                URI.create("http://localhost:8080/api/"),
                config
        );

        System.out.println("Jersey Grizzly 服务已启动");
        System.in.read();
        server.shutdownNow();
    }
}
```

这种方式适合 Demo、测试工具、轻量 HTTP 服务。

正式业务系统更常见的方式还是 Spring Boot 或应用服务器部署。

### 和 Spring MVC 共存

Spring Boot 项目里，Jersey 和 Spring MVC 可以共存，但需要明确请求由谁处理。

一种常见方式是让 Jersey 走 Filter 模式，并把 404 转发给其他 Web 框架。

`application.yml`：

```yaml
spring:
  jersey:
    type: filter
```

`ResourceConfig`：

```java
package com.example.jersey.config;

import org.glassfish.jersey.server.ResourceConfig;
import org.glassfish.jersey.servlet.ServletProperties;
import org.springframework.stereotype.Component;

@Component
public class JerseyConfig extends ResourceConfig {

    public JerseyConfig() {
        register(UserResource.class);
        property(ServletProperties.FILTER_FORWARD_ON_404, true);
    }
}
```

这样 Jersey 没匹配到的请求，可以继续交给 Spring MVC。

如果项目没有共存需求，Jersey 独立处理接口会更简单。

### Spring Security 集成注意点

Jersey 和 Spring Security 结合时，如果使用方法级安全，Spring Boot 官方文档建议设置：

```java
setProperties(Map.of(
        "jersey.config.server.response.setStatusOverSendError",
        true
));
```

完整示例：

```java
import com.example.jersey.resource.UserResource;
import org.glassfish.jersey.server.ResourceConfig;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
public class JerseyConfig extends ResourceConfig {

    public JerseyConfig() {
        register(UserResource.class);
        setProperties(Map.of(
                "jersey.config.server.response.setStatusOverSendError",
                true
        ));
    }
}
```

这个配置可以避免 Jersey 过早提交错误响应，让 Spring Security 有机会正确处理认证和授权失败。

### 常见问题

#### Resource 没有生效

常见原因：

| 原因 | 处理方式 |
| --- | --- |
| 没有注册 Resource | 在 `ResourceConfig` 里 `register(...)` |
| 只使用 `packages(...)` 扫描 | 可执行 jar 场景优先显式注册 |
| 缺少 `@Component` | 需要 Spring 管理时加 `@Component` |
| 路径前缀理解错误 | 检查 `@ApplicationPath` 和 `@Path` |

#### JSON 不能正常序列化

常见原因是缺少 Jackson 适配。

Spring Boot starter 通常会处理常见 JSON 场景。

独立 Jersey 项目可以加入：

```xml
<dependency>
    <groupId>org.glassfish.jersey.media</groupId>
    <artifactId>jersey-media-json-jackson</artifactId>
    <version>${jersey.version}</version>
</dependency>
```

#### @RequestBody 为什么不能用

`@RequestBody` 是 Spring MVC 注解。

Jersey 使用 JAX-RS 模型，请求体通常直接放在方法参数里：

```java
@POST
public Response create(CreateUserRequest request) {
    return Response.ok().build();
}
```

路径参数使用 `@PathParam`，查询参数使用 `@QueryParam`。

#### javax.ws.rs 和 jakarta.ws.rs 能不能混用

不适合混用。

Spring Boot 2、老 Java EE 项目常见 `javax.ws.rs`。

Spring Boot 3、Jakarta EE 新项目常见 `jakarta.ws.rs`。

同一个项目里混用两套包名，容易出现注解不识别、类加载冲突、依赖版本不匹配等问题。

### 实践建议

| 场景 | 建议 |
| --- | --- |
| Spring Boot 3 项目 | 使用 `spring-boot-starter-jersey` |
| Resource 注册 | 优先在 `ResourceConfig` 显式 `register` |
| 路径前缀 | 使用 `@ApplicationPath("/api")` |
| JSON 接口 | 类上统一 `@Produces`、`@Consumes` |
| 业务异常 | 使用 `ExceptionMapper` |
| 请求日志 | 使用 `ContainerRequestFilter` |
| 参数校验 | 结合 Jakarta Validation |
| 和 Spring MVC 共存 | 使用 filter 模式和 404 转发 |
| 新项目包名 | 使用 `jakarta.ws.rs` |

### 小结

Jersey 的核心是 JAX-RS / Jakarta REST 编程模型。

`@Path` 定义资源路径，`@GET`、`@POST` 等注解定义 HTTP 方法，`@Produces` 和 `@Consumes` 控制媒体类型，`@PathParam`、`@QueryParam` 等注解完成参数绑定。

Spring Boot 项目里，`spring-boot-starter-jersey` 加 `ResourceConfig` 就能把 Jersey 接入应用。业务接口用 Resource 编写，异常交给 `ExceptionMapper`，横切逻辑交给 Filter，外部 HTTP 调用可以用 Jersey Client。

如果项目需要标准 JAX-RS 风格的 REST API，Jersey 是一套成熟、清晰、可落地的选择。
