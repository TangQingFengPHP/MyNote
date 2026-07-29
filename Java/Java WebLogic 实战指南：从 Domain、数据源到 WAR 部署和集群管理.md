### 简介

WebLogic 全名是 `Oracle WebLogic Server`。

它是 Oracle 提供的企业级 Java 应用服务器，常见于银行、保险、电信、政务、传统大型企业系统。

Tomcat 更偏 Servlet 容器。

WebLogic 更像完整的企业级应用平台。

它除了能跑 Servlet、JSP、Spring MVC 这类 Web 应用，还提供：

```text
EJB
JMS
JTA 事务
JNDI
数据源连接池
集群
安全域
部署管理
监控诊断
```

一句话概括：

```text
WebLogic 是面向企业级 Java 应用的应用服务器，负责承载 Web 应用、企业组件、数据源、消息、事务、集群和安全管理。
```

### WebLogic 解决什么问题

传统企业 Java 系统经常不只是一个 Web 接口。

它可能还需要：

```text
统一数据源
分布式事务
JMS 消息
EJB 组件
多节点集群
会话复制
统一安全认证
集中部署和运维
```

如果每个应用都自己处理这些能力，系统会很难管。

WebLogic 的思路是：

```text
应用只关注业务代码
容器负责连接池、事务、消息、安全、集群和部署
```

比如应用里只写：

```java
@Resource(lookup = "jdbc/orderDS")
private DataSource dataSource;
```

真正的数据库地址、账号、连接池大小、测试 SQL、部署目标，都在 WebLogic 控制台里配置。

这也是很多传统企业项目喜欢 WebLogic 的原因：应用配置和运行资源可以集中交给应用服务器管理。

### WebLogic 和 Tomcat 的区别

| 对比项 | Tomcat | WebLogic |
| --- | --- | --- |
| 定位 | Servlet 容器 | 企业级应用服务器 |
| 所属 | Apache | Oracle |
| Servlet / JSP | 支持 | 支持 |
| EJB | 不支持 | 支持 |
| JMS | 不内置 | 内置 |
| JTA 分布式事务 | 能力有限 | 内置支持 |
| 数据源 | 应用内或容器配置 | 容器级数据源和连接池 |
| 集群 | 需要额外方案 | 内置集群能力 |
| 管理控制台 | 较轻 | 功能完整 |
| 授权 | 开源 | 商业产品 |

简单理解：

```text
Tomcat：跑 Web 应用很常见，轻量直接。
WebLogic：跑企业级 Java EE / Jakarta EE 系统，管理能力更完整。
```

新微服务项目通常更常用 Spring Boot 内嵌 Tomcat、Jetty、Undertow。

传统企业系统、历史 Java EE 系统、依赖 EJB/JMS/JTA 的系统，则经常会遇到 WebLogic。

### 版本和规范关系

WebLogic 版本和 Java/Jakarta 规范关系很重要。

| WebLogic 版本 | 规范支持 | 常见 JDK | 包名特点 |
| --- | --- | --- | --- |
| 12.2.1.4 | Java EE 7 | JDK 8 | `javax.*` |
| 14.1.1 | Java EE 8 | JDK 8 / 11 | `javax.*` |
| 14.1.2 | Jakarta EE 8 | JDK 17 / 21 | 仍以 `javax.*` 为主 |
| 15.1.1 | Jakarta EE 9.1 | JDK 17 / 21 | `jakarta.*` |

几个关键点：

```text
Java EE 8 / Jakarta EE 8：API 包名还是 javax.*
Jakarta EE 9.1：API 包名切到 jakarta.*
```

所以迁移时不能只看“Jakarta”这个名字。

WebLogic 14.1.2 支持 Jakarta EE 8，但应用代码仍然常见 `javax.servlet`、`javax.jms`、`javax.persistence`。

WebLogic 15.1.1 支持 Jakarta EE 9.1，应用代码要进入 `jakarta.servlet`、`jakarta.jms`、`jakarta.persistence` 这套包名体系。

### WebLogic 核心架构

WebLogic 的管理单位叫 Domain。

一个典型结构如下：

```text
Domain
  |
  +--> Admin Server
  |
  +--> Managed Server 1
  |
  +--> Managed Server 2
  |
  +--> Cluster
        |
        +--> Managed Server 1
        +--> Managed Server 2
```

核心概念：

| 概念 | 说明 |
| --- | --- |
| `Domain` | 域，一个完整的 WebLogic 管理环境 |
| `Admin Server` | 管理服务器，提供控制台和配置管理 |
| `Managed Server` | 受管服务器，真正运行业务应用 |
| `Cluster` | 多个 Managed Server 组成的集群 |
| `Machine` | 物理机或虚拟机映射 |
| `Node Manager` | 远程启动、停止、监控 Managed Server |
| `Data Source` | 数据源和连接池 |
| `JMS Server` | JMS 消息服务 |
| `Security Realm` | 用户、组、角色、安全认证 |

Admin Server 负责管理。

Managed Server 负责跑业务。

生产环境里，应用通常部署到 Managed Server 或 Cluster，而不是只部署到 Admin Server。

### Domain 目录结构

一个 Domain 目录大概是这样：

```text
base_domain/
├── bin/
├── config/
├── lib/
├── logs/
├── security/
├── servers/
│   ├── AdminServer/
│   └── ManagedServer1/
└── startWebLogic.sh
```

常见目录：

| 目录 | 说明 |
| --- | --- |
| `bin` | 启动、停止脚本 |
| `config` | Domain 配置文件 |
| `lib` | Domain 级别公共 jar |
| `logs` | 日志 |
| `security` | 安全相关文件 |
| `servers` | 各 Server 的运行目录 |

核心配置文件：

```text
config/config.xml
```

它保存 Domain 里的 Server、数据源、JMS、集群、部署等配置。

一般不手工改 `config.xml`，更常见的是通过控制台、WLST 或 REST 管理接口修改。

### 安装和创建 Domain

WebLogic 安装包通常从 Oracle 官网或 Oracle Software Delivery Cloud 获取。

安装后常见目录变量：

| 变量 | 说明 |
| --- | --- |
| `ORACLE_HOME` | Oracle Middleware 安装目录 |
| `DOMAIN_HOME` | 具体 Domain 目录 |
| `JAVA_HOME` | JDK 目录 |

创建 Domain 可以使用配置向导：

```bash
$ORACLE_HOME/oracle_common/common/bin/config.sh
```

Windows：

```bash
%ORACLE_HOME%\oracle_common\common\bin\config.cmd
```

创建时常见配置：

```text
Domain 名称
管理员账号和密码
Admin Server 端口，默认常见 7001
是否创建 Managed Server
是否创建 Cluster
是否配置 Node Manager
```

开发环境可以先创建一个单机 Domain。

生产环境通常会创建 Admin Server、多个 Managed Server、Machine、Node Manager 和 Cluster。

### 启动和关闭

进入 Domain 目录：

```bash
cd $DOMAIN_HOME
```

启动 Admin Server：

```bash
./startWebLogic.sh
```

Windows：

```bash
startWebLogic.cmd
```

控制台地址通常是：

```text
http://localhost:7001/console
```

启动 Managed Server：

```bash
cd $DOMAIN_HOME/bin
./startManagedWebLogic.sh ManagedServer1 http://localhost:7001
```

关闭脚本：

```bash
./stopWebLogic.sh
./stopManagedWebLogic.sh ManagedServer1
```

生产环境里更常用 Node Manager 来管理 Managed Server 生命周期。

### 第一个 Servlet WAR Demo

先写一个最小 Web 应用，打成 WAR 部署到 WebLogic。

项目结构：

```text
weblogic-servlet-demo/
├── pom.xml
└── src/
    └── main/
        ├── java/
        │   └── com/example/weblogic/HelloServlet.java
        └── webapp/
            ├── index.jsp
            └── WEB-INF/
                ├── web.xml
                └── weblogic.xml
```

### pom.xml

如果目标是 WebLogic 14.1.x 或更早的 `javax.*` 体系，可以使用：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>weblogic-servlet-demo</artifactId>
    <version>1.0.0</version>
    <packaging>war</packaging>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>4.0.1</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <finalName>weblogic-servlet-demo</finalName>
    </build>
</project>
```

`scope=provided` 表示 Servlet API 由 WebLogic 容器提供，不打进 WAR。

如果目标是 WebLogic 15.1.1，需要切换到 `jakarta.servlet-api`，代码包名也要改成 `jakarta.servlet.*`。

### HelloServlet

WebLogic 14.1.x 示例：

```java
package com.example.weblogic;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.PrintWriter;

@WebServlet("/hello")
public class HelloServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request,
                         HttpServletResponse response) throws ServletException, IOException {
        response.setContentType("text/html;charset=UTF-8");

        PrintWriter writer = response.getWriter();
        writer.println("<h2>Hello WebLogic</h2>");
        writer.println("<p>Server: " + System.getProperty("weblogic.Name") + "</p>");
        writer.println("<p>Java: " + System.getProperty("java.version") + "</p>");
    }
}
```

### web.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                             http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <display-name>weblogic-servlet-demo</display-name>

    <welcome-file-list>
        <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>
</web-app>
```

### weblogic.xml

`weblogic.xml` 是 WebLogic 专用部署描述符，放在 `WEB-INF` 下。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<weblogic-web-app xmlns="http://xmlns.oracle.com/weblogic/weblogic-web-app">
    <context-root>/demo</context-root>

    <container-descriptor>
        <prefer-web-inf-classes>false</prefer-web-inf-classes>
    </container-descriptor>
</weblogic-web-app>
```

`context-root` 决定访问根路径。

打包：

```bash
mvn clean package
```

生成：

```text
target/weblogic-servlet-demo.war
```

部署后访问：

```text
http://localhost:7001/demo/hello
```

### 控制台部署 WAR

常见步骤：

```text
登录控制台
  |
  v
Deployments
  |
  v
Install
  |
  v
选择 WAR / EAR
  |
  v
选择部署目标 Server 或 Cluster
  |
  v
Finish
  |
  v
Start servicing all requests
```

开发环境可以部署到 Admin Server。

生产环境通常部署到 Cluster 或 Managed Server。

### 命令行部署

WebLogic 提供 `weblogic.Deployer`。

部署：

```bash
java weblogic.Deployer \
  -adminurl t3://localhost:7001 \
  -username weblogic \
  -password welcome1 \
  -deploy \
  -name weblogic-servlet-demo \
  -source target/weblogic-servlet-demo.war \
  -targets AdminServer
```

停止：

```bash
java weblogic.Deployer \
  -adminurl t3://localhost:7001 \
  -username weblogic \
  -password welcome1 \
  -stop \
  -name weblogic-servlet-demo
```

卸载：

```bash
java weblogic.Deployer \
  -adminurl t3://localhost:7001 \
  -username weblogic \
  -password welcome1 \
  -undeploy \
  -name weblogic-servlet-demo
```

命令行部署适合自动化脚本和 CI/CD。

账号密码不适合写死在脚本里，生产环境一般通过凭据文件、环境变量或部署平台注入。

### 配置 JDBC 数据源

WebLogic 数据源通常在控制台里创建。

常见步骤：

```text
Services
  |
  v
Data Sources
  |
  v
New Generic Data Source
  |
  v
填写名称和 JNDI
  |
  v
选择数据库类型和驱动
  |
  v
填写 URL、用户名、密码
  |
  v
测试连接
  |
  v
选择部署目标
```

示例配置：

| 配置项 | 示例 |
| --- | --- |
| Name | `OrderDataSource` |
| JNDI Name | `jdbc/orderDS` |
| Driver | MySQL / Oracle JDBC Driver |
| URL | `jdbc:mysql://localhost:3306/order_db` |
| Target | `OrderCluster` |

应用代码通过 JNDI 获取数据源。

### JNDI 数据源 Demo

Servlet 示例：

```java
package com.example.weblogic;

import javax.annotation.Resource;
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.sql.DataSource;
import java.io.IOException;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

@WebServlet("/orders/count")
public class OrderCountServlet extends HttpServlet {

    @Resource(lookup = "jdbc/orderDS")
    private DataSource dataSource;

    @Override
    protected void doGet(HttpServletRequest request,
                         HttpServletResponse response) throws ServletException, IOException {
        response.setContentType("text/plain;charset=UTF-8");

        try (Connection connection = dataSource.getConnection();
             PreparedStatement statement = connection.prepareStatement("select count(*) from orders");
             ResultSet resultSet = statement.executeQuery()) {

            resultSet.next();
            response.getWriter().write("订单数量：" + resultSet.getInt(1));
        } catch (Exception e) {
            throw new ServletException(e);
        }
    }
}
```

也可以手动查找：

```java
Context context = new InitialContext();
DataSource dataSource = (DataSource) context.lookup("jdbc/orderDS");
```

容器数据源的好处是：

```text
连接池参数集中管理
数据库账号不写在应用包里
应用和数据库配置解耦
生产运维可以统一调整连接池
```

### Spring Boot WAR 部署到 WebLogic

Spring Boot 默认打可执行 jar，内嵌 Tomcat。

部署到 WebLogic 时，通常要改成 WAR，让 WebLogic 作为外部 Servlet 容器。

`pom.xml`：

```xml
<packaging>war</packaging>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

启动类继承 `SpringBootServletInitializer`：

```java
package com.example.boot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.boot.web.servlet.support.SpringBootServletInitializer;

@SpringBootApplication
public class WebLogicBootApplication extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(WebLogicBootApplication.class);
    }

    public static void main(String[] args) {
        SpringApplication.run(WebLogicBootApplication.class, args);
    }
}
```

Controller：

```java
package com.example.boot;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HealthController {

    @GetMapping("/health")
    public String health() {
        return "UP";
    }
}
```

`src/main/webapp/WEB-INF/weblogic.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<weblogic-web-app xmlns="http://xmlns.oracle.com/weblogic/weblogic-web-app">
    <context-root>/boot-demo</context-root>

    <container-descriptor>
        <prefer-application-packages>
            <package-name>org.springframework.*</package-name>
            <package-name>org.slf4j.*</package-name>
            <package-name>ch.qos.logback.*</package-name>
            <package-name>com.fasterxml.jackson.*</package-name>
        </prefer-application-packages>
    </container-descriptor>
</weblogic-web-app>
```

打包：

```bash
mvn clean package
```

部署后访问：

```text
http://localhost:7001/boot-demo/health
```

版本注意：

```text
Spring Boot 2.x：javax.* 体系，适合 WebLogic 14.1.x 及更早版本
Spring Boot 3.x：jakarta.* 体系，更适合 WebLogic 15.1.1 这类 Jakarta EE 9.1 版本
```

如果 Spring Boot 3 应用部署到只支持 `javax.*` 的 WebLogic 版本，通常会出现类找不到、Servlet API 不匹配等问题。

### Spring 使用 WebLogic 数据源

Spring Boot 可以直接使用 JNDI 数据源。

`application.yml`：

```yaml
spring:
  datasource:
    jndi-name: jdbc/orderDS
```

这样 Spring Boot 会从 WebLogic 容器查找 `jdbc/orderDS`。

如果用 Java 配置：

```java
package com.example.boot.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jndi.JndiObjectFactoryBean;

import javax.sql.DataSource;

@Configuration
public class DataSourceConfig {

    @Bean
    public JndiObjectFactoryBean orderDataSource() {
        JndiObjectFactoryBean bean = new JndiObjectFactoryBean();
        bean.setJndiName("jdbc/orderDS");
        bean.setProxyInterface(DataSource.class);
        return bean;
    }
}
```

使用容器数据源后，应用包里不需要放数据库连接 URL 和密码。

### JMS 资源和代码示例

WebLogic 内置 JMS 能力。

常见配置顺序：

```text
创建 JMS Server
创建 JMS Module
创建 Connection Factory
创建 Queue 或 Topic
绑定 JNDI 名称
部署到 Server 或 Cluster
```

假设配置：

| 资源 | JNDI |
| --- | --- |
| Connection Factory | `jms/orderConnectionFactory` |
| Queue | `jms/orderQueue` |

发送消息：

```java
package com.example.weblogic.jms;

import javax.annotation.Resource;
import javax.jms.ConnectionFactory;
import javax.jms.JMSContext;
import javax.jms.Queue;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@WebServlet("/jms/send")
public class OrderMessageServlet extends HttpServlet {

    @Resource(lookup = "jms/orderConnectionFactory")
    private ConnectionFactory connectionFactory;

    @Resource(lookup = "jms/orderQueue")
    private Queue orderQueue;

    @Override
    protected void doPost(HttpServletRequest request,
                          HttpServletResponse response) throws IOException {
        try (JMSContext context = connectionFactory.createContext()) {
            context.createProducer().send(orderQueue, "order-created-10001");
        }

        response.getWriter().write("sent");
    }
}
```

如果目标是 WebLogic 15.1.1，需要把 `javax.jms` 换成 `jakarta.jms`，`javax.annotation` 换成 `jakarta.annotation`。

### EAR 包是什么

WebLogic 里除了 WAR，还经常能看到 EAR。

| 包类型 | 说明 |
| --- | --- |
| WAR | Web 应用包，通常包含 Servlet、JSP、Spring MVC |
| JAR | 普通 Java 库或 EJB Jar |
| EAR | 企业应用包，可以包含多个 WAR、EJB Jar、公共 lib |

EAR 结构示例：

```text
order-system.ear
├── META-INF/
│   └── application.xml
├── order-web.war
├── order-ejb.jar
└── lib/
    └── common-utils.jar
```

老式 Java EE 大系统常用 EAR，把多个模块作为一个企业应用一起部署。

新一些的 Spring Boot 应用更多是单 WAR 或可执行 jar。

### WLST 自动化管理

WLST 全称是 `WebLogic Scripting Tool`。

它是 WebLogic 的脚本工具，语法基于 Jython。

连接：

```python
connect('weblogic', 'welcome1', 't3://localhost:7001')
```

部署应用：

```python
connect('weblogic', 'welcome1', 't3://localhost:7001')

deploy(
    'weblogic-servlet-demo',
    '/opt/apps/weblogic-servlet-demo.war',
    targets='AdminServer'
)

disconnect()
exit()
```

启动 Managed Server：

```python
connect('weblogic', 'welcome1', 't3://localhost:7001')
start('ManagedServer1')
disconnect()
exit()
```

执行：

```bash
$ORACLE_HOME/oracle_common/common/bin/wlst.sh deploy.py
```

WLST 适合：

```text
自动部署
批量创建资源
环境初始化
CI/CD
配置巡检
```

生产脚本里同样不适合明文写密码。

### 集群和高可用

生产环境常见结构：

```text
Nginx / Apache / F5
  |
  v
WebLogic Cluster
  |
  +--> ManagedServer1
  +--> ManagedServer2
  +--> ManagedServer3
```

WebLogic 集群能提供：

* 应用部署到多个节点
* 会话复制
* 故障转移
* 负载分摊
* 集群级 JMS / 数据源目标

常见配置步骤：

```text
创建 Machine
配置 Node Manager
创建 Managed Server
创建 Cluster
把 Managed Server 加入 Cluster
配置数据源和 JMS 目标
把应用部署到 Cluster
前端负载均衡转发流量
```

集群不是只创建多个 Server。

还要考虑会话、JMS、事务日志、数据源目标、健康检查、负载均衡和故障恢复。

### 日志和排查

常见日志位置：

```text
$DOMAIN_HOME/servers/AdminServer/logs/
$DOMAIN_HOME/servers/ManagedServer1/logs/
```

常见日志：

| 日志 | 说明 |
| --- | --- |
| `AdminServer.log` | 管理服务器日志 |
| `ManagedServer1.log` | 受管服务器日志 |
| `access.log` | HTTP 访问日志 |
| `domain.log` | Domain 级别日志 |
| 应用日志 | 应用自己输出的日志 |

排查部署失败时，通常先看：

```text
Managed Server 日志
应用启动异常堆栈
类冲突信息
数据源测试结果
JNDI 查找异常
```

### 类加载和依赖冲突

WebLogic 自带很多 Java EE / Jakarta EE 相关库。

应用自己也可能带类似库。

这就容易出现类冲突。

常见现象：

```text
ClassNotFoundException
NoSuchMethodError
ClassCastException
Servlet API 包名不匹配
Jackson / SLF4J / Spring 版本冲突
```

`weblogic.xml` 可以控制部分类加载行为：

```xml
<container-descriptor>
    <prefer-application-packages>
        <package-name>org.springframework.*</package-name>
        <package-name>org.slf4j.*</package-name>
        <package-name>com.fasterxml.jackson.*</package-name>
    </prefer-application-packages>
</container-descriptor>
```

但这不是万能解法。

更好的方式是先确认：

```text
WebLogic 版本
JDK 版本
Java EE / Jakarta EE 版本
Spring / Hibernate 版本
应用依赖里是否打入了容器已提供的 API
```

Servlet、JSP、JMS、JPA 这类容器 API 通常使用 `provided`，避免重复打入 WAR。

### 容器化和 Kubernetes

WebLogic 也可以运行在容器和 Kubernetes 里。

Oracle 提供了 WebLogic Kubernetes Operator、WebLogic Deploy Tooling、WebLogic Image Tool 等工具。

常见思路：

```text
用镜像承载 WebLogic 运行时
用模型文件描述 Domain 配置
用 Operator 管理 Domain 生命周期
用 Kubernetes 管理 Pod、Service、存储和滚动更新
```

这类部署更偏运维和平台工程。

传统脚本部署项目可以先掌握 Domain、Managed Server、数据源、部署和 WLST。

现代化迁移项目再进一步关注 Kubernetes 工具链。

### 常见问题

#### Spring Boot jar 能不能直接部署到 WebLogic

通常不直接部署可执行 jar。

WebLogic 是应用服务器，常规部署对象是 WAR 或 EAR。

Spring Boot 应用部署到 WebLogic 时，一般改成 WAR，并继承 `SpringBootServletInitializer`。

#### javax 和 jakarta 怎么选

看 WebLogic 和应用框架版本。

| 场景 | 包名 |
| --- | --- |
| WebLogic 14.1.x、Spring Boot 2 | `javax.*` |
| WebLogic 15.1.1、Spring Boot 3 | `jakarta.*` |

两套包名不能随意混用。

#### 数据源 JNDI 找不到

常见原因：

| 原因 | 处理方式 |
| --- | --- |
| JNDI 名称写错 | 对照控制台配置 |
| 数据源没有 Target 到当前 Server | 把数据源部署到应用所在 Server 或 Cluster |
| 数据源测试失败 | 检查驱动、URL、账号、网络 |
| 应用里 res-ref 配置不一致 | 检查 `web.xml`、`weblogic.xml` |

#### 部署后访问 404

常见原因：

| 原因 | 处理方式 |
| --- | --- |
| context-root 和访问路径不一致 | 检查 `weblogic.xml` |
| 应用没有启动 | 查看 Deployments 状态 |
| 部署目标不对 | 确认部署到当前访问的 Server |
| Controller / Servlet 映射不对 | 检查应用路由 |

#### 本地能跑，WebLogic 部署失败

常见方向：

```text
JDK 版本不一致
Servlet API 包名不一致
容器自带依赖和应用依赖冲突
Spring Boot jar 没改 WAR
数据库驱动没有放好
数据源没有 Target
weblogic.xml 配置不匹配
```

### 实践建议

| 场景 | 建议 |
| --- | --- |
| 新建 WebLogic 项目 | 先确认 WebLogic、JDK、Java/Jakarta 规范版本 |
| Web 应用部署 | 打 WAR，容器 API 使用 `provided` |
| Spring Boot 部署 | 改 WAR，继承 `SpringBootServletInitializer` |
| 数据库连接 | 优先使用 WebLogic 数据源和 JNDI |
| 部署自动化 | 使用 `weblogic.Deployer` 或 WLST |
| 生产部署 | 应用部署到 Managed Server 或 Cluster |
| 类冲突 | 优先理清依赖范围，再考虑 `prefer-application-packages` |
| 密码管理 | 使用凭据文件、密钥系统或部署平台注入 |
| 容器化 | 关注 WebLogic Kubernetes Operator 和 WDT |

### 小结

WebLogic 的核心价值在企业级运行时能力。

它不只是一个 Servlet 容器，还提供 Domain 管理、Admin Server、Managed Server、集群、数据源、JMS、JTA、安全域、部署和监控能力。

日常开发和部署里，最需要先掌握几件事：版本和包名关系、Domain 架构、WAR/EAR 部署、JNDI 数据源、JMS 资源、Spring Boot WAR 改造、WLST 自动化和类加载冲突处理。

如果项目仍然运行在传统企业 Java 技术栈上，WebLogic 依然是很重要的一类应用服务器。
