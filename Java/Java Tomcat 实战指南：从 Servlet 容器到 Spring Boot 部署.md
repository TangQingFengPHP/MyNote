### 简介

`Tomcat` 全名是 `Apache Tomcat`。

它是 Java Web 领域很常见的 Servlet 容器，也可以理解成轻量级 Java Web 服务器。

它主要负责这些事情：

```text
监听 HTTP 端口
接收浏览器或网关发来的请求
解析 HTTP 请求
找到对应的 Web 应用
调用 Servlet / Filter / Listener
管理 Session
返回 HTTP 响应
```

传统 Java Web 项目、Spring MVC 项目、打成 WAR 包的 Spring Boot 项目，都可以部署到 Tomcat。

一句话概括：

```text
Tomcat 是用来运行 Java Web 应用的 Servlet 容器，负责把 HTTP 请求交给 Java 代码处理。
```

### Tomcat 在 Java Web 里的位置

一个典型请求链路大致是：

```text
浏览器
  |
  v
Nginx
  |
  v
Tomcat
  |
  v
Servlet / Spring MVC / Spring Boot
  |
  v
Service
  |
  v
Repository / Mapper
  |
  v
数据库
```

`Nginx` 常用于静态资源、反向代理、负载均衡。

`Tomcat` 负责运行 Java Web 应用。

Spring Boot 默认内嵌 Tomcat。

所以本地运行一个 Spring Boot Web 项目时，控制台里看到的 `Tomcat started on port 8080`，其实就是启动了一个内嵌 Tomcat。

### Tomcat 和 Nginx 的区别

| 对比项 | Tomcat | Nginx |
| --- | --- | --- |
| 类型 | Servlet 容器 / Java Web 服务器 | Web 服务器 / 反向代理 |
| 主要职责 | 运行 Java Web 应用 | 静态资源、反向代理、负载均衡 |
| 是否能运行 Servlet | 可以 | 不可以 |
| 静态资源能力 | 可以处理，但不是强项 | 很强 |
| 常见位置 | 后端应用容器 | 入口网关或反向代理 |

常见部署结构：

```text
用户请求
  |
  v
Nginx: 80 / 443
  |
  v
Tomcat: 8080
  |
  v
Java 应用
```

### Tomcat 版本怎么选

Tomcat 版本和 Servlet / Jakarta 规范有关。

| Tomcat 版本线 | 包名 | Java 要求 | 常见场景 |
| --- | --- | --- | --- |
| Tomcat 9.0.x | `javax.servlet.*` | Java 8+ | 老项目、Spring Boot 2、Java EE 8 应用 |
| Tomcat 10.1.x | `jakarta.servlet.*` | Java 11+ | Spring Boot 3、Jakarta EE 10 相关项目 |
| Tomcat 11.0.x | `jakarta.servlet.*` | Java 17+ | 新项目、较新的 Jakarta 规范 |

最大的迁移点是包名变化：

```text
Tomcat 9：javax.servlet
Tomcat 10 / 11：jakarta.servlet
```

如果一个老项目代码里大量使用：

```java
import javax.servlet.http.HttpServlet;
```

直接放到 Tomcat 10 或 11 上通常会出问题。

需要迁移到：

```java
import jakarta.servlet.http.HttpServlet;
```

Spring Boot 2 常见搭配是 Tomcat 9。

Spring Boot 3 常见搭配是 Tomcat 10.1。

新项目如果基于 Java 17 和较新 Jakarta 规范，可以关注 Tomcat 11。

### 目录结构

下载并解压 Tomcat 后，常见目录如下：

```text
apache-tomcat/
├── bin/
├── conf/
├── lib/
├── logs/
├── temp/
├── webapps/
└── work/
```

| 目录 | 作用 |
| --- | --- |
| `bin` | 启动、停止脚本 |
| `conf` | 配置文件 |
| `lib` | Tomcat 自身和公共类库 |
| `logs` | 日志目录 |
| `temp` | 临时文件 |
| `webapps` | Web 应用部署目录 |
| `work` | JSP 编译后的缓存文件 |

几个常见配置文件：

| 文件 | 作用 |
| --- | --- |
| `conf/server.xml` | 端口、Connector、Engine、Host 等核心配置 |
| `conf/web.xml` | 全局 Web 配置 |
| `conf/context.xml` | 全局 Context 配置 |
| `conf/tomcat-users.xml` | Manager 用户和角色配置 |
| `conf/logging.properties` | Tomcat 日志配置 |

### 启动和关闭

macOS / Linux：

```bash
cd apache-tomcat/bin
./startup.sh
```

关闭：

```bash
./shutdown.sh
```

前台启动，方便看日志：

```bash
./catalina.sh run
```

Windows：

```bat
startup.bat
```

关闭：

```bat
shutdown.bat
```

默认访问地址：

```text
http://localhost:8080
```

如果能看到 Tomcat 首页，说明启动成功。

### 修改端口

端口配置在：

```text
conf/server.xml
```

默认 HTTP Connector：

```xml
<Connector port="8080"
           protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

改成 `9090`：

```xml
<Connector port="9090"
           protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

重启后访问：

```text
http://localhost:9090
```

如果端口被占用，日志里通常会看到 `Address already in use`。

### server.xml 里的核心结构

`server.xml` 大致结构如下：

```xml
<Server>
    <Service>
        <Connector />
        <Engine>
            <Host>
                <Context />
            </Host>
        </Engine>
    </Service>
</Server>
```

简单理解：

| 组件 | 作用 |
| --- | --- |
| `Server` | 整个 Tomcat 实例 |
| `Service` | 一组 Connector 和一个 Engine |
| `Connector` | 接收网络请求，比如 HTTP 8080 |
| `Engine` | 请求处理引擎 |
| `Host` | 虚拟主机，比如 localhost |
| `Context` | 一个 Web 应用 |
| `Wrapper` | 一个 Servlet |

请求进入 Tomcat 后，大致流转是：

```text
Connector
  |
  v
Engine
  |
  v
Host
  |
  v
Context
  |
  v
Wrapper / Servlet
```

### 第一个 Servlet Demo

下面示例适合 Tomcat 10 / 11。

因为它使用 `jakarta.servlet`。

Maven 依赖：

```xml
<dependency>
    <groupId>jakarta.servlet</groupId>
    <artifactId>jakarta.servlet-api</artifactId>
    <version>6.0.0</version>
    <scope>provided</scope>
</dependency>
```

`provided` 表示：

```text
编译时需要这个 API
运行时由 Tomcat 提供
```

Servlet：

```java
package com.example.servlet;

import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

@WebServlet("/hello")
public class HelloServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request,
                         HttpServletResponse response) throws ServletException, IOException {
        response.setContentType("text/plain;charset=UTF-8");
        response.getWriter().write("Hello Tomcat");
    }
}
```

如果应用名是 `demo`，访问地址：

```text
http://localhost:8080/demo/hello
```

### WAR 包结构

传统 Java Web 应用最终通常打成 `.war`。

一个 WAR 包内部结构大致是：

```text
demo.war
├── index.jsp
├── WEB-INF/
│   ├── web.xml
│   ├── classes/
│   └── lib/
```

重点：

| 路径 | 作用 |
| --- | --- |
| `WEB-INF/web.xml` | Web 应用配置文件 |
| `WEB-INF/classes` | 编译后的 class 文件 |
| `WEB-INF/lib` | 应用依赖 jar |
| 根目录 | 静态资源、JSP 等 |

`WEB-INF` 目录不能被浏览器直接访问。

### 部署 WAR 包

最常见方式是把 WAR 放到：

```text
apache-tomcat/webapps/
```

例如：

```text
webapps/demo.war
```

Tomcat 启动后会自动部署。

访问地址：

```text
http://localhost:8080/demo
```

如果希望根路径访问：

```text
http://localhost:8080/
```

可以把 WAR 命名为：

```text
ROOT.war
```

### Spring Boot 打 WAR 部署到外部 Tomcat

Spring Boot 默认是内嵌 Tomcat，直接打 jar 运行。

如果需要部署到外部 Tomcat，需要改成 WAR 包。

`pom.xml`：

```xml
<packaging>war</packaging>
```

`spring-boot-starter-tomcat` 设置为 `provided`：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>
```

启动类继承 `SpringBootServletInitializer`：

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.boot.web.servlet.support.SpringBootServletInitializer;

@SpringBootApplication
public class DemoApplication extends SpringBootServletInitializer {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(DemoApplication.class);
    }
}
```

测试接口：

```java
package com.example.demo.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/hello")
    public String hello() {
        return "Hello Tomcat";
    }
}
```

打包：

```bash
mvn clean package -DskipTests
```

假设生成：

```text
target/demo.war
```

复制到：

```text
apache-tomcat/webapps/demo.war
```

访问：

```text
http://localhost:8080/demo/hello
```

### Spring Boot 内嵌 Tomcat

Spring Boot Web 默认使用内嵌 Tomcat。

依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

运行：

```bash
java -jar demo.jar
```

控制台通常能看到：

```text
Tomcat started on port 8080
```

常见配置：

```yaml
server:
  port: 8081
  servlet:
    context-path: /demo
  tomcat:
    threads:
      max: 200
      min-spare: 10
    accept-count: 100
    connection-timeout: 20s
```

访问地址：

```text
http://localhost:8081/demo/hello
```

### 外部 Tomcat 和内嵌 Tomcat 怎么选

| 对比项 | 外部 Tomcat | 内嵌 Tomcat |
| --- | --- | --- |
| 打包方式 | WAR | JAR |
| 启动方式 | 启动 Tomcat | `java -jar` |
| 配置位置 | Tomcat `conf` 目录 | Spring Boot 配置 |
| 运维方式 | 多应用共用一个容器 | 每个应用独立进程 |
| 常见场景 | 传统部署、存量系统 | 微服务、容器化、云原生 |

现代 Spring Boot 项目里，内嵌 Tomcat 更常见。

传统企业环境里，外部 Tomcat + WAR 部署仍然很多。

### Connector 常用配置

`Connector` 是请求入口。

常见配置：

```xml
<Connector port="8080"
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="200"
           minSpareThreads="10"
           acceptCount="100"
           connectionTimeout="20000"
           redirectPort="8443" />
```

常用属性：

| 属性 | 作用 |
| --- | --- |
| `port` | 监听端口 |
| `protocol` | 协议实现 |
| `maxThreads` | 最大请求处理线程数 |
| `minSpareThreads` | 最小空闲线程数 |
| `acceptCount` | 线程都忙时的等待队列长度 |
| `connectionTimeout` | 连接超时时间 |
| `redirectPort` | HTTP 跳转 HTTPS 时使用的端口 |

调优时不能只看 Tomcat。

还要结合：

* CPU 核数
* JVM 内存
* 数据库连接池
* 接口耗时
* 网关超时
* 下游服务能力

### Executor 线程池

Tomcat 可以定义共享线程池。

```xml
<Executor name="tomcatThreadPool"
          namePrefix="catalina-exec-"
          maxThreads="300"
          minSpareThreads="20" />

<Connector port="8080"
           protocol="HTTP/1.1"
           executor="tomcatThreadPool"
           connectionTimeout="20000"
           redirectPort="8443" />
```

多个 Connector 可以共享同一个 Executor。

实际项目里，一个 HTTP Connector 通常已经足够。

### HTTPS 配置

Tomcat 可以直接配置 HTTPS。

示例：

```xml
<Connector port="8443"
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           SSLEnabled="true">
    <SSLHostConfig>
        <Certificate certificateKeystoreFile="conf/keystore.p12"
                     certificateKeystorePassword="changeit"
                     type="RSA" />
    </SSLHostConfig>
</Connector>
```

也可以让 Nginx 负责 HTTPS，Tomcat 只监听内网 HTTP。

常见结构：

```text
浏览器 HTTPS
  |
  v
Nginx 443
  |
  v
Tomcat 8080
```

这种方式更常见，也更方便统一管理证书。

### Nginx 反向代理 Tomcat

Nginx 配置示例：

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Spring Boot 内嵌 Tomcat 识别反向代理头：

```yaml
server:
  forward-headers-strategy: framework
```

外部 Tomcat 也可以通过 Valve 处理代理头。

### JNDI 数据源

传统 Tomcat 项目可能会把数据源放到 Tomcat 里。

`conf/context.xml`：

```xml
<Context>
    <Resource name="jdbc/AppDataSource"
              auth="Container"
              type="javax.sql.DataSource"
              factory="org.apache.tomcat.jdbc.pool.DataSourceFactory"
              driverClassName="com.mysql.cj.jdbc.Driver"
              url="jdbc:mysql://localhost:3306/app_db"
              username="root"
              password="123456"
              maxActive="50"
              maxIdle="20"
              minIdle="5"
              maxWait="10000" />
</Context>
```

应用里通过 JNDI 查找：

```java
Context context = new InitialContext();
DataSource dataSource = (DataSource) context.lookup("java:comp/env/jdbc/AppDataSource");
```

Spring Boot 项目里，更常见的是直接在应用配置里使用 HikariCP。

### 日志和排查

常看日志目录：

```text
logs/
```

常见文件：

| 文件 | 作用 |
| --- | --- |
| `catalina.out` | 控制台输出，Linux 常见 |
| `catalina.YYYY-MM-DD.log` | Catalina 日志 |
| `localhost.YYYY-MM-DD.log` | Web 应用相关日志 |
| `manager.YYYY-MM-DD.log` | Manager 应用日志 |
| `host-manager.YYYY-MM-DD.log` | Host Manager 日志 |
| `localhost_access_log.*.txt` | 访问日志 |

常见排查顺序：

```text
应用是否部署成功
端口是否启动
访问路径是否正确
日志是否有异常堆栈
依赖 jar 是否缺失
Tomcat 版本和 Servlet 包名是否匹配
```

### 常见问题

### 端口被占用

现象：

```text
Address already in use
```

处理思路：

```text
换端口
或者停止占用端口的进程
```

macOS / Linux 查询端口：

```bash
lsof -i :8080
```

### 访问 404

常见原因：

* 应用没有部署成功
* URL 上下文路径写错
* Controller 或 Servlet 路径写错
* WAR 包名和访问路径不一致
* Spring Boot 配置了 `context-path`

如果 WAR 叫：

```text
demo.war
```

默认访问前缀就是：

```text
/demo
```

### javax 和 jakarta 不匹配

Tomcat 9 使用 `javax.servlet`。

Tomcat 10 / 11 使用 `jakarta.servlet`。

如果应用和 Tomcat 版本不匹配，可能出现：

```text
ClassNotFoundException
NoClassDefFoundError
```

处理思路：

```text
老项目继续使用 Tomcat 9
新项目迁移到 jakarta 后使用 Tomcat 10 / 11
```

### WAR 部署失败

常见原因：

* JDK 版本不匹配
* Servlet API 包名不匹配
* 依赖冲突
* 数据库连接失败
* 配置文件缺失
* Tomcat 用户没有目录读写权限

优先查看：

```text
logs/catalina.out
logs/localhost.*.log
```

### 安全相关建议

生产环境里，常见做法有这些：

* 删除不需要的默认应用，比如 `docs`、`examples`
* Manager 后台限制访问来源
* Manager 账号使用强密码
* 不使用的 Connector 从 `server.xml` 移除
* AJP 只在可信网络里使用，或者关闭
* Tomcat 进程使用普通用户运行
* 定期升级到受支持版本
* 反向代理统一处理 HTTPS
* 不把管理端口暴露到公网

Tomcat 默认已经有不少安全配置。

生产环境还要结合网络边界、运维规范和应用权限一起处理。

### 常用命令汇总

| 命令 | 作用 |
| --- | --- |
| `startup.sh` | 后台启动 |
| `shutdown.sh` | 停止 |
| `catalina.sh run` | 前台启动 |
| `tail -f logs/catalina.out` | 查看日志 |
| `lsof -i :8080` | 查看端口占用 |
| `jps -l` | 查看 Java 进程 |
| `jstack <pid>` | 查看线程栈 |
| `jmap -heap <pid>` | 查看堆信息 |

### 常用配置汇总

| 配置 | 作用 |
| --- | --- |
| `server.xml` | Tomcat 主配置 |
| `Connector port` | HTTP 监听端口 |
| `maxThreads` | 最大处理线程数 |
| `acceptCount` | 等待队列长度 |
| `connectionTimeout` | 连接超时时间 |
| `Host appBase` | 应用部署目录 |
| `Context path` | 应用访问路径 |
| `webapps` | 默认部署目录 |
| `logs` | 日志目录 |

### 总结

`Tomcat` 的核心定位很清楚：

```text
运行 Java Web 应用
管理 Servlet 生命周期
处理 HTTP 请求和响应
提供应用部署、Session、线程池、日志等基础能力
```

实际项目里常见两种方式：

```text
Spring Boot 内嵌 Tomcat，打成 jar 运行
外部 Tomcat 部署 WAR 包
```

新项目通常更偏向内嵌 Tomcat。

存量系统、传统企业部署、集中式运维环境，外部 Tomcat 仍然很常见。

掌握这些内容后，基本就能完成 Tomcat 的安装启动、端口配置、WAR 部署、Spring Boot 外置部署、日志排查和常见生产配置。
