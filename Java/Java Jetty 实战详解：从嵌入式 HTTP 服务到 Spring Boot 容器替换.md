### 简介

`Jetty` 是 Eclipse 基金会维护的 Java Web 服务器，也是一个 Servlet 容器。

它主要能做这些事：

```text
监听 HTTP 端口
处理 HTTP 请求和响应
运行 Servlet / Filter / Listener
部署 WAR 包
支持 WebSocket、HTTP/2、HTTP/3
作为库嵌入到 Java 程序里启动
作为 Spring Boot 的内嵌容器
```

一句话概括：

```text
Jetty 既能像 Tomcat 一样单独部署 Web 应用，也能像普通 Java 库一样嵌入到 main 方法里启动。
```

Jetty 最突出的特点是“嵌入式”和“可编程”。

很多中间件、网关、测试工具、微服务框架喜欢用 Jetty，原因很直接：启动快、模块化、代码里就能组装 HTTP 服务，不一定要准备一个完整应用服务器。

### Jetty 适合什么场景

常见场景如下：

| 场景 | 说明 |
| --- | --- |
| 嵌入式 HTTP 服务 | 普通 Java 程序里直接启动一个 Web 管理端口 |
| REST API | 不想引入完整 Spring MVC 时，可以直接用 Handler 或 Servlet |
| Spring Boot 内嵌容器 | 替换默认 Tomcat，用 Jetty 承载 Web 应用 |
| 传统 WAR 部署 | 独立 Jetty 服务部署标准 Web 应用 |
| WebSocket 服务 | 聊天、推送、实时状态 |
| 测试环境 | 用 Maven 插件快速跑 Web 项目 |
| 工具型程序 | 本地工具、桌面程序、采集程序暴露健康检查接口 |

简单理解：

```text
Tomcat 更常见于传统 Java Web 容器
Jetty 更常见于嵌入式服务、工具服务和高度可编程的 Web 底座
```

这不是说 Tomcat 不能嵌入，也不是说 Jetty 不能部署 WAR。

只是两个项目的气质不太一样。

### Jetty 和 Tomcat 的区别

| 对比项 | Jetty | Tomcat |
| --- | --- | --- |
| 类型 | HTTP 服务器 / Servlet 容器 | Servlet 容器 / Java Web 服务器 |
| 嵌入式体验 | 很强，API 设计偏可编程 | 支持，但日常感知没有 Jetty 强 |
| 独立部署 | 支持 | 支持 |
| WAR 部署 | 支持 | 支持 |
| Spring Boot | 可替换默认 Tomcat | Spring Boot Web 默认容器 |
| Handler API | Jetty 特色能力 | 没有同款核心模型 |
| 适合场景 | 嵌入式、微服务、工具、网关、长连接 | 传统 Web、Spring MVC、企业后台 |

Jetty 里很重要的概念是 `Handler`。

可以把它理解成：

```text
请求进来后，Jetty 先交给 Handler 树处理。
```

如果使用 Servlet，Servlet 也是挂在 Jetty 的 Handler 体系下面。

### 版本怎么选

Jetty 版本和 Java、Servlet / Jakarta EE 规范强相关。

| Jetty 版本线 | Java 要求 | EE 体系 | 包名 |
| --- | --- | --- | --- |
| Jetty 9.4.x | Java 8 | Java EE / Servlet 老项目 | `javax.servlet.*` |
| Jetty 10.0.x | Java 11 | Jakarta EE 8 | `javax.servlet.*` |
| Jetty 11.0.x | Java 11 | Jakarta EE 9 | `jakarta.servlet.*` |
| Jetty 12.0.x | Java 17 | Jakarta EE 8 / 9 / 10 | `javax.*` 和 `jakarta.*` 按环境区分 |
| Jetty 12.1.x | Java 17 | Jakarta EE 8 / 9 / 10 / 11 | `javax.*` 和 `jakarta.*` 按环境区分 |

新项目常见选择：

```text
Java 17+
Jetty 12.0.x 稳定线
Jakarta EE 10
jakarta.servlet.*
```

老项目如果还在使用：

```java
import javax.servlet.http.HttpServlet;
```

不能直接照搬 Jetty 12 的 EE10 示例。

需要选择对应的 EE8 模块，或者先把代码迁移到：

```java
import jakarta.servlet.http.HttpServlet;
```

### Jetty 的两种使用方式

Jetty 常见用法可以分成两类。

### 独立运行

先下载 Jetty，再部署应用。

大致像这样：

```text
jetty-home
jetty-base
webapps
start.jar
```

然后把 WAR 包放进去，由 Jetty 启动和管理。

这种方式适合：

* 传统 WAR 项目
* 多个 Web 应用统一部署
* 运维希望容器和应用分开管理

### 嵌入式运行

把 Jetty 当成 Maven 依赖引进来，在 `main` 方法里启动。

大致像这样：

```java
public static void main(String[] args) {
    Server server = new Server(8080);
    server.setHandler(...);
    server.start();
}
```

这种方式适合：

* 微服务
* 工具程序
* 单个 Jar 包部署
* 测试服务
* 内部管理端口

Spring Boot 内嵌 Jetty 本质上也是嵌入式模式。

### 核心架构

Jetty 处理请求的大致流程：

```text
Client
  |
  v
Connector
  |
  v
Server
  |
  v
Handler
  |
  v
Servlet / Filter / WebSocket / 自定义处理逻辑
```

几个核心对象：

| 对象 | 作用 |
| --- | --- |
| `Server` | Jetty 服务实例，负责管理生命周期 |
| `Connector` | 监听端口，接收客户端连接 |
| `ServerConnector` | 常见服务端连接器 |
| `Handler` | Jetty 请求处理核心 |
| `ContextHandler` | 给 Handler 增加上下文路径，比如 `/shop` |
| `ServletContextHandler` | 支持 Servlet / Filter，不需要 WAR 包 |
| `WebAppContext` | 支持标准 WAR 包和 `WEB-INF/web.xml` |
| `QueuedThreadPool` | Jetty 工作线程池 |

最小结构：

```text
Server
  |
  v
ServerConnector :8080
  |
  v
Handler
```

Servlet 结构：

```text
Server
  |
  v
ServletContextHandler /api
  |
  ├── UserServlet /users/*
  └── LogFilter /*
```

### Maven 依赖

下面示例基于 Jetty 12.0.x 和 Jakarta EE 10。

版本号可以统一放到 properties：

```xml
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <jetty.version>12.0.37</jetty.version>
    <jackson.version>2.17.2</jackson.version>
</properties>
```

最小 Handler 服务只需要 `jetty-server`：

```xml
<dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-server</artifactId>
    <version>${jetty.version}</version>
</dependency>
```

如果要写 Servlet，需要引入 EE10 Servlet 模块：

```xml
<dependency>
    <groupId>org.eclipse.jetty.ee10</groupId>
    <artifactId>jetty-ee10-servlet</artifactId>
    <version>${jetty.version}</version>
</dependency>
```

如果要部署 WAR，需要引入 WebApp 模块：

```xml
<dependency>
    <groupId>org.eclipse.jetty.ee10</groupId>
    <artifactId>jetty-ee10-webapp</artifactId>
    <version>${jetty.version}</version>
</dependency>
```

Demo 里为了输出 JSON，再加 Jackson：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>${jackson.version}</version>
</dependency>
```

完整依赖：

```xml
<dependencies>
    <dependency>
        <groupId>org.eclipse.jetty</groupId>
        <artifactId>jetty-server</artifactId>
        <version>${jetty.version}</version>
    </dependency>

    <dependency>
        <groupId>org.eclipse.jetty.ee10</groupId>
        <artifactId>jetty-ee10-servlet</artifactId>
        <version>${jetty.version}</version>
    </dependency>

    <dependency>
        <groupId>org.eclipse.jetty.ee10</groupId>
        <artifactId>jetty-ee10-webapp</artifactId>
        <version>${jetty.version}</version>
    </dependency>

    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>${jackson.version}</version>
    </dependency>
</dependencies>
```

### 第一个 Demo：不用 Servlet 的 HTTP 服务

Jetty 12 推荐直接使用 `Handler.Abstract` 写底层 HTTP 处理逻辑。

这个 Demo 只做一件事：

```text
访问 http://localhost:8080/hello
返回一段 JSON
```

启动类：

```java
package com.example.jettydemo;

import org.eclipse.jetty.http.HttpHeader;
import org.eclipse.jetty.io.Content;
import org.eclipse.jetty.server.Request;
import org.eclipse.jetty.server.Response;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.server.ServerConnector;
import org.eclipse.jetty.server.Handler;
import org.eclipse.jetty.util.Callback;

public class HelloHandlerServer {

    public static void main(String[] args) throws Exception {
        Server server = new Server();

        ServerConnector connector = new ServerConnector(server);
        connector.setHost("0.0.0.0");
        connector.setPort(8080);
        connector.setIdleTimeout(30_000);
        server.addConnector(connector);

        server.setHandler(new Handler.Abstract() {
            @Override
            public boolean handle(Request request, Response response, Callback callback) {
                String path = request.getHttpURI().getPath();

                if (!"/hello".equals(path)) {
                    response.setStatus(404);
                    response.getHeaders().put(HttpHeader.CONTENT_TYPE, "application/json;charset=UTF-8");
                    Content.Sink.write(response, true, "{\"message\":\"not found\"}", callback);
                    return true;
                }

                response.setStatus(200);
                response.getHeaders().put(HttpHeader.CONTENT_TYPE, "application/json;charset=UTF-8");
                Content.Sink.write(response, true, "{\"message\":\"Hello Jetty\"}", callback);
                return true;
            }
        });

        server.start();
        System.out.println("Jetty started: http://localhost:8080/hello");
        server.join();
    }
}
```

测试：

```shell
curl http://localhost:8080/hello
```

返回：

```json
{"message":"Hello Jetty"}
```

访问不存在的路径：

```shell
curl http://localhost:8080/abc
```

返回：

```json
{"message":"not found"}
```

这个例子没有 Servlet，没有 Spring MVC，也没有注解路由。

请求是直接交给 `Handler` 处理的。

这种方式适合做非常轻量的接口，比如：

* 健康检查
* 本地管理端口
* 内部指标接口
* 简单代理服务
* 不想引入 Web 框架的小工具

### Handler 为什么要返回 boolean

`handle` 方法返回 `true` 表示请求已经被当前 Handler 处理。

```java
return true;
```

如果是 Handler 组合结构，返回值会影响后续 Handler 是否继续处理。

同时，响应写完后需要通知 `callback`。

```java
Content.Sink.write(response, true, body, callback);
```

这行代码里，`Content.Sink.write` 会在写入完成后处理 callback。

如果手动处理异步写入，需要确保最后调用：

```java
callback.succeeded();
```

或异常时调用：

```java
callback.failed(error);
```

否则请求可能一直挂着。

### 第二个 Demo：Servlet 版 REST 接口

很多 Java Web 项目还是 Servlet 模型。

Jetty 可以不用 WAR 包，直接用 `ServletContextHandler` 在代码里注册 Servlet。

这个 Demo 实现三个接口：

| 方法 | 路径 | 作用 |
| --- | --- | --- |
| `GET` | `/api/users` | 查询用户列表 |
| `GET` | `/api/users?id=1` | 查询单个用户 |
| `POST` | `/api/users?name=张三` | 新增用户 |

### 返回对象

```java
package com.example.jettydemo.dto;

public record ApiResult<T>(int code, String message, T data) {

    public static <T> ApiResult<T> ok(T data) {
        return new ApiResult<>(200, "success", data);
    }

    public static <T> ApiResult<T> fail(int code, String message) {
        return new ApiResult<>(code, message, null);
    }
}
```

### Servlet

```java
package com.example.jettydemo.servlet;

import com.example.jettydemo.dto.ApiResult;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.List;
import java.util.Map;

public class UserServlet extends HttpServlet {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("application/json;charset=UTF-8");

        String id = request.getParameter("id");
        if (id == null || id.isBlank()) {
            ApiResult<List<Map<String, Object>>> result = ApiResult.ok(List.of(
                    Map.of("id", 1, "name", "张三"),
                    Map.of("id", 2, "name", "李四")
            ));
            objectMapper.writeValue(response.getWriter(), result);
            return;
        }

        ApiResult<Map<String, Object>> result = ApiResult.ok(
                Map.of("id", Long.parseLong(id), "name", "用户" + id)
        );
        objectMapper.writeValue(response.getWriter(), result);
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("application/json;charset=UTF-8");

        String name = request.getParameter("name");
        if (name == null || name.isBlank()) {
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            objectMapper.writeValue(response.getWriter(), ApiResult.fail(400, "name 不能为空"));
            return;
        }

        ApiResult<Map<String, Object>> result = ApiResult.ok(
                Map.of("id", System.currentTimeMillis(), "name", name)
        );
        objectMapper.writeValue(response.getWriter(), result);
    }
}
```

### 启动类

```java
package com.example.jettydemo;

import com.example.jettydemo.servlet.UserServlet;
import org.eclipse.jetty.ee10.servlet.ServletContextHandler;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.server.ServerConnector;

public class ServletApiServer {

    public static void main(String[] args) throws Exception {
        Server server = new Server();

        ServerConnector connector = new ServerConnector(server);
        connector.setPort(8080);
        connector.setIdleTimeout(30_000);
        server.addConnector(connector);

        ServletContextHandler context = new ServletContextHandler();
        context.setContextPath("/api");
        context.addServlet(UserServlet.class, "/users");

        server.setHandler(context);
        server.start();

        System.out.println("Jetty Servlet API started: http://localhost:8080/api/users");
        server.join();
    }
}
```

测试：

```shell
curl http://localhost:8080/api/users
```

返回：

```json
{
  "code": 200,
  "message": "success",
  "data": [
    {
      "id": 1,
      "name": "张三"
    },
    {
      "id": 2,
      "name": "李四"
    }
  ]
}
```

查询单个用户：

```shell
curl 'http://localhost:8080/api/users?id=100'
```

新增用户：

```shell
curl -X POST 'http://localhost:8080/api/users?name=王五'
```

### 加一个 Filter：接口耗时日志

Servlet 项目常见需求是记录请求日志。

可以写一个 Filter：

```java
package com.example.jettydemo.servlet;

import jakarta.servlet.Filter;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRequest;
import jakarta.servlet.ServletResponse;
import jakarta.servlet.http.HttpServletRequest;

import java.io.IOException;

public class AccessLogFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        long start = System.currentTimeMillis();

        try {
            chain.doFilter(request, response);
        } finally {
            HttpServletRequest httpRequest = (HttpServletRequest) request;
            long cost = System.currentTimeMillis() - start;
            System.out.println(httpRequest.getMethod() + " " + httpRequest.getRequestURI() + " " + cost + "ms");
        }
    }
}
```

注册 Filter：

```java
package com.example.jettydemo;

import com.example.jettydemo.servlet.AccessLogFilter;
import com.example.jettydemo.servlet.UserServlet;
import jakarta.servlet.DispatcherType;
import org.eclipse.jetty.ee10.servlet.FilterHolder;
import org.eclipse.jetty.ee10.servlet.ServletContextHandler;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.server.ServerConnector;

import java.util.EnumSet;

public class ServletApiServer {

    public static void main(String[] args) throws Exception {
        Server server = new Server();

        ServerConnector connector = new ServerConnector(server);
        connector.setPort(8080);
        server.addConnector(connector);

        ServletContextHandler context = new ServletContextHandler();
        context.setContextPath("/api");
        context.addServlet(UserServlet.class, "/users");
        context.addFilter(new FilterHolder(new AccessLogFilter()), "/*", EnumSet.of(DispatcherType.REQUEST));

        server.setHandler(context);
        server.start();
        server.join();
    }
}
```

访问接口后，控制台会打印：

```text
GET /api/users 12ms
```

这一套写法和 `web.xml` 里配置 Servlet / Filter 的效果类似，只是改成了 Java 代码注册。

### 第三个 Demo：用 WebAppContext 部署 WAR

如果项目已经是标准 WAR 包，可以用 `WebAppContext`。

目录假设如下：

```text
/opt/apps/user-center.war
```

启动类：

```java
package com.example.jettydemo;

import org.eclipse.jetty.ee10.webapp.WebAppContext;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.server.ServerConnector;

public class WarServer {

    public static void main(String[] args) throws Exception {
        Server server = new Server();

        ServerConnector connector = new ServerConnector(server);
        connector.setPort(8080);
        server.addConnector(connector);

        WebAppContext webAppContext = new WebAppContext();
        webAppContext.setContextPath("/user-center");
        webAppContext.setWar("/opt/apps/user-center.war");

        server.setHandler(webAppContext);
        server.start();

        System.out.println("WAR started: http://localhost:8080/user-center");
        server.join();
    }
}
```

访问：

```text
http://localhost:8080/user-center
```

`ServletContextHandler` 和 `WebAppContext` 的区别：

| 对比项 | ServletContextHandler | WebAppContext |
| --- | --- | --- |
| 是否需要 WAR | 不需要 | 通常需要 |
| 是否读取 web.xml | 不读取 | 会读取 |
| 组件注册方式 | Java 代码注册 | 按 WAR / web.xml / 注解 |
| 适合场景 | 嵌入式服务、小应用 | 标准 Web 应用、旧项目迁移 |

### 独立 Jetty 部署 WAR

Jetty 也可以像传统应用服务器一样独立运行。

常见目录概念：

```text
JETTY_HOME：Jetty 安装目录
JETTY_BASE：业务配置和应用目录
```

`JETTY_HOME` 放 Jetty 自身文件。

`JETTY_BASE` 放项目配置、模块启用文件、webapps、日志等。

这种设计方便升级 Jetty：

```text
升级时替换 JETTY_HOME
业务配置继续留在 JETTY_BASE
```

创建 base 目录：

```shell
mkdir jetty-base
cd jetty-base
```

启用 HTTP 和 EE10 部署模块：

```shell
java -jar $JETTY_HOME/start.jar --add-modules=http,ee10-deploy
```

这会创建类似目录：

```text
jetty-base/
├── resources/
├── start.d/
│   ├── http.ini
│   └── ee10-deploy.ini
└── webapps/
```

复制 WAR：

```shell
cp /opt/apps/user-center.war webapps/
```

启动：

```shell
java -jar $JETTY_HOME/start.jar
```

访问：

```text
http://localhost:8080/user-center
```

WAR 文件名通常会影响访问路径。

比如：

```text
webapps/user-center.war -> /user-center
webapps/root.war        -> /
```

如果需要自定义 context path，可以使用 Jetty context XML。

示例 `webapps/user-center.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "https://jetty.org/configure_10_0.dtd">
<Configure class="org.eclipse.jetty.ee10.webapp.WebAppContext">
    <Set name="contextPath">/uc</Set>
    <Set name="war">/opt/apps/user-center.war</Set>
</Configure>
```

这样访问路径就是：

```text
http://localhost:8080/uc
```

### Jetty Maven 插件

开发传统 Web 项目时，可以用 Jetty Maven 插件快速启动。

Jetty 12 以后，Maven 插件也按 EE 版本拆包。

Jakarta EE 10 项目使用：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.eclipse.jetty.ee10</groupId>
            <artifactId>jetty-ee10-maven-plugin</artifactId>
            <version>12.0.37</version>
            <configuration>
                <scan>2</scan>
                <webApp>
                    <contextPath>/</contextPath>
                </webApp>
            </configuration>
        </plugin>
    </plugins>
</build>
```

启动：

```shell
mvn jetty:run
```

常见目标：

| 目标 | 说明 |
| --- | --- |
| `jetty:run` | 直接运行未打包 Web 应用，开发最常用 |
| `jetty:run-war` | 运行已经打好的 WAR |
| `jetty:start` | 启动后不阻塞 Maven，适合绑定构建生命周期 |
| `jetty:start-war` | 以 WAR 方式启动后不阻塞 |
| `jetty:stop` | 停止 |

`scan` 表示扫描变更的间隔秒数。

```xml
<scan>2</scan>
```

表示每 2 秒检查一次变更，发现变化后重新部署。

Jetty Maven 插件适合开发和测试，不适合生产部署。

生产环境更适合：

* 独立 Jetty
* 嵌入式 Jetty
* Spring Boot 可执行 Jar
* 容器镜像

### Spring Boot 替换 Tomcat 为 Jetty

Spring Boot Web 默认使用 Tomcat。

替换成 Jetty，只需要排除 Tomcat，再引入 Jetty starter。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

Controller 不需要改：

```java
package com.example.bootjetty.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
@RequestMapping("/api")
public class HealthController {

    @GetMapping("/health")
    public Map<String, Object> health() {
        return Map.of(
                "status", "UP",
                "container", "Jetty"
        );
    }
}
```

启动后，控制台里会看到 Jetty 相关日志。

访问：

```shell
curl http://localhost:8080/api/health
```

返回：

```json
{
  "status": "UP",
  "container": "Jetty"
}
```

Spring Boot 里常见 Jetty 配置：

```yaml
server:
  port: 8080
  jetty:
    connection-idle-timeout: 30s
    max-connections: 10000
    max-http-form-post-size: 2MB
    threads:
      min: 8
      max: 200
      acceptors: -1
      selectors: -1
      idle-timeout: 60s
```

说明：

| 配置 | 作用 |
| --- | --- |
| `server.jetty.connection-idle-timeout` | 连接空闲多久关闭 |
| `server.jetty.max-connections` | 最大连接数 |
| `server.jetty.max-http-form-post-size` | 表单 POST 最大大小 |
| `server.jetty.threads.min` | 最小线程数 |
| `server.jetty.threads.max` | 最大线程数 |
| `server.jetty.threads.acceptors` | 接收连接线程数 |
| `server.jetty.threads.selectors` | I/O 选择器线程数 |

`acceptors` 和 `selectors` 通常先保持默认 `-1`。

默认值会按运行环境计算。除非已经有压测数据，否则不建议一上来手动调很大。

### 自定义 Spring Boot 内嵌 Jetty

如果配置文件不够，可以写 `WebServerFactoryCustomizer`。

比如调整 Connector 的空闲超时：

```java
package com.example.bootjetty.config;

import org.eclipse.jetty.server.Connector;
import org.eclipse.jetty.server.ServerConnector;
import org.springframework.boot.web.embedded.jetty.JettyServletWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class JettyConfig {

    @Bean
    public WebServerFactoryCustomizer<JettyServletWebServerFactory> jettyCustomizer() {
        return factory -> factory.addServerCustomizers(server -> {
            for (Connector connector : server.getConnectors()) {
                if (connector instanceof ServerConnector serverConnector) {
                    serverConnector.setIdleTimeout(30_000);
                }
            }
        });
    }
}
```

这种方式适合配置文件覆盖不到的 Jetty 原生能力。

### 线程模型怎么理解

Jetty 不是“一个请求一个新线程”这么简单。

可以粗略理解成：

```text
Acceptor 负责接收连接
Selector 负责监听 I/O 事件
Worker 负责执行请求处理逻辑
```

`QueuedThreadPool` 是 Jetty 常见线程池。

简单嵌入式配置示例：

```java
package com.example.jettydemo;

import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.server.ServerConnector;
import org.eclipse.jetty.util.thread.QueuedThreadPool;

public class ThreadPoolServer {

    public static void main(String[] args) throws Exception {
        QueuedThreadPool threadPool = new QueuedThreadPool();
        threadPool.setName("jetty-worker");
        threadPool.setMinThreads(8);
        threadPool.setMaxThreads(200);
        threadPool.setIdleTimeout(60_000);

        Server server = new Server(threadPool);

        ServerConnector connector = new ServerConnector(server);
        connector.setPort(8080);
        server.addConnector(connector);

        server.start();
        server.join();
    }
}
```

线程数不是越大越好。

需要看这些因素：

* CPU 核数
* 接口是否阻塞
* 数据库连接池大小
* 下游接口耗时
* 请求峰值
* JVM 内存

如果接口里有大量慢 SQL、远程 HTTP 调用、文件 I/O，Jetty 线程再多也只是把阻塞堆起来。

更有效的优化往往是：

```text
缩短接口耗时
控制下游超时
限制队列长度
做好限流
给慢任务单独线程池
```

### Handler 和 Servlet 怎么选

| 需求 | 建议 |
| --- | --- |
| 极简 HTTP 接口 | `Handler` |
| 健康检查 / 指标暴露 | `Handler` |
| 已经熟悉 Servlet | `ServletContextHandler` |
| 需要 Filter / Session | `ServletContextHandler` |
| 标准 WAR 包 | `WebAppContext` |
| Spring MVC / Spring Boot | 交给 Spring Boot Jetty starter |

`Handler` 更轻，但更底层。

`Servlet` 更熟悉，生态也更完整。

如果只是写业务 REST API，Spring Boot + Jetty 更省心。

如果是做网关、中间件、内部控制面，直接嵌入 Jetty Handler 会更轻。

### 静态资源

Jetty 可以处理静态资源。

纯静态文件服务可以直接用 `ResourceHandler`。

示例：

```java
package com.example.jettydemo;

import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.server.ServerConnector;
import org.eclipse.jetty.server.handler.ResourceHandler;
import org.eclipse.jetty.util.resource.ResourceFactory;

import java.nio.file.Path;
import java.util.List;

public class StaticFileServer {

    public static void main(String[] args) throws Exception {
        Server server = new Server();

        ServerConnector connector = new ServerConnector(server);
        connector.setPort(8080);
        server.addConnector(connector);

        ResourceHandler resourceHandler = new ResourceHandler();
        resourceHandler.setBaseResource(ResourceFactory.of(resourceHandler).newResource(Path.of("public")));
        resourceHandler.setDirAllowed(false);
        resourceHandler.setWelcomeFiles(List.of("index.html"));
        resourceHandler.setAcceptRanges(true);

        server.setHandler(resourceHandler);
        server.start();
        server.join();
    }
}
```

目录：

```text
project/
├── public/
│   └── index.html
└── src/
```

访问：

```text
http://localhost:8080/index.html
```

如果是 Servlet Web 应用，也可以使用 `DefaultServlet` 或 `ResourceServlet`，它们更适合挂在 `ServletContextHandler` 里。

生产环境里，静态资源通常还是交给 Nginx、CDN 或对象存储。

Jetty 处理静态资源没问题，但业务系统里更常见的结构是：

```text
Nginx / CDN 处理静态资源
Jetty 处理动态接口
```

### HTTPS 简单配置

嵌入式 Jetty 配置 HTTPS 时，需要准备 keystore。

生成本地测试证书：

```shell
keytool -genkeypair \
  -alias jetty-demo \
  -keyalg RSA \
  -keysize 2048 \
  -storetype PKCS12 \
  -keystore jetty-demo.p12 \
  -validity 365
```

Jetty HTTPS Connector 示例：

```java
package com.example.jettydemo;

import org.eclipse.jetty.http.HttpVersion;
import org.eclipse.jetty.server.HttpConfiguration;
import org.eclipse.jetty.server.HttpConnectionFactory;
import org.eclipse.jetty.server.SecureRequestCustomizer;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.server.ServerConnector;
import org.eclipse.jetty.server.SslConnectionFactory;
import org.eclipse.jetty.util.ssl.SslContextFactory;

public class HttpsServer {

    public static void main(String[] args) throws Exception {
        Server server = new Server();

        HttpConfiguration httpsConfig = new HttpConfiguration();
        httpsConfig.addCustomizer(new SecureRequestCustomizer());

        SslContextFactory.Server sslContextFactory = new SslContextFactory.Server();
        sslContextFactory.setKeyStorePath("jetty-demo.p12");
        sslContextFactory.setKeyStorePassword("changeit");

        ServerConnector httpsConnector = new ServerConnector(
                server,
                new SslConnectionFactory(sslContextFactory, HttpVersion.HTTP_1_1.asString()),
                new HttpConnectionFactory(httpsConfig)
        );
        httpsConnector.setPort(8443);

        server.addConnector(httpsConnector);
        server.start();
        server.join();
    }
}
```

访问：

```text
https://localhost:8443
```

真实生产环境里，HTTPS 常见做法是：

```text
公网 HTTPS 终止在 Nginx / 网关 / SLB
内网 HTTP 转发到 Jetty
```

如果 Jetty 直接暴露公网 HTTPS，需要认真配置证书、协议版本、加密套件和证书续期流程。

### 日志和访问日志

Spring Boot 里可以直接开 Jetty access log：

```yaml
server:
  jetty:
    accesslog:
      enabled: true
      filename: logs/jetty-access.log
      retention-period: 14
      append: true
```

普通嵌入式 Jetty 可以通过 Handler / Filter 自己记录访问日志。

如果是 Servlet 应用，用前面的 `AccessLogFilter` 就够处理简单场景。

生产日志至少要包含：

* 请求方法
* 请求路径
* 状态码
* 耗时
* 客户端 IP
* User-Agent
* TraceId

不要在访问日志里直接打印密码、Token、身份证号、手机号等敏感信息。

### 常见坑

### Jetty 12 里找不到 org.eclipse.jetty.servlet.ServletContextHandler

Jetty 12 的 Servlet 模块按 EE 环境拆开了。

Jakarta EE 10 应该使用：

```java
import org.eclipse.jetty.ee10.servlet.ServletContextHandler;
```

Maven 依赖：

```xml
<dependency>
    <groupId>org.eclipse.jetty.ee10</groupId>
    <artifactId>jetty-ee10-servlet</artifactId>
    <version>${jetty.version}</version>
</dependency>
```

老文章里的：

```java
import org.eclipse.jetty.servlet.ServletContextHandler;
```

多半是 Jetty 9 / 10 / 11 时代的写法。

### javax.servlet 和 jakarta.servlet 混用

常见报错：

```text
ClassNotFoundException: javax.servlet.http.HttpServlet
```

或：

```text
ClassCastException: ... jakarta.servlet.Servlet
```

原因通常是依赖和代码包名不匹配。

判断方式：

```text
代码 import javax.servlet.* -> 使用 EE8 / 老版本相关模块
代码 import jakarta.servlet.* -> 使用 EE9 / EE10 / EE11 相关模块
```

Spring Boot 3 使用 `jakarta.servlet.*`。

Spring Boot 2 使用 `javax.servlet.*`。

### Handler 写了响应但请求一直不结束

检查 callback 是否完成。

正确示例：

```java
Content.Sink.write(response, true, body, callback);
return true;
```

或者手动：

```java
callback.succeeded();
return true;
```

异常时：

```java
callback.failed(error);
return true;
```

### Spring Boot 已经引入 Jetty 但还是 Tomcat

检查是否排除了 Tomcat：

```xml
<exclusions>
    <exclusion>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId>
    </exclusion>
</exclusions>
```

再检查依赖树：

```shell
mvn dependency:tree
```

如果还能看到 `spring-boot-starter-tomcat`，说明某个 starter 又间接带了 Tomcat。

### Maven 插件启动失败

Jetty 12 插件要选 EE 对应版本。

Jakarta EE 10：

```xml
<groupId>org.eclipse.jetty.ee10</groupId>
<artifactId>jetty-ee10-maven-plugin</artifactId>
```

不是旧写法：

```xml
<groupId>org.eclipse.jetty</groupId>
<artifactId>jetty-maven-plugin</artifactId>
```

### 端口被占用

报错类似：

```text
java.net.BindException: Address already in use
```

说明端口已经被占用。

查看端口：

```shell
lsof -i :8080
```

换端口：

```java
connector.setPort(8081);
```

或 Spring Boot 配置：

```yaml
server:
  port: 8081
```

### 总结

Jetty 可以按这条线理解：

```text
Server 管生命周期
Connector 管端口和连接
Handler 管请求处理
ServletContextHandler 管 Servlet / Filter
WebAppContext 管 WAR 包
Spring Boot starter 负责把 Jetty 接到 Boot 启动流程里
```

日常使用可以按场景选择：

* 极简服务：`Server + Handler`
* Servlet 接口：`Server + ServletContextHandler`
* WAR 项目：`WebAppContext` 或独立 Jetty
* Spring Boot 项目：排除 Tomcat，引入 `spring-boot-starter-jetty`
* 开发调试：`jetty-ee10-maven-plugin`

Jetty 的优势不在于“概念多”，而在于组合灵活。简单接口可以很轻，标准 Web 应用也能部署，Spring Boot 里还能直接替换容器。把版本、EE 环境、Handler 和 Servlet 的边界理清楚，Jetty 用起来就不会乱。
