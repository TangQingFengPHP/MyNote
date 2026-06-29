### 简介

`Maven` 是 Java 生态里很常见的项目管理和构建工具。

它主要解决几类问题：

```text
项目目录怎么放
依赖 jar 怎么下载
源码怎么编译
测试怎么执行
jar / war 怎么打包
多模块项目怎么一起构建
构建产物怎么发布到仓库
```

简单理解：

```text
pom.xml 是项目说明书
Maven 仓库负责存放依赖和构建产物
Maven 生命周期负责按固定顺序执行构建流程
Maven 插件负责真正干活
```

一句话概括：

```text
Maven 用一套固定约定，把 Java 项目的依赖管理、编译、测试、打包和发布流程标准化。
```

### Maven 解决什么问题

没有 Maven 之前，一个 Java 项目经常需要手动处理很多事情：

```text
手动下载 jar
手动配置 classpath
手动处理 jar 版本冲突
手动编译代码
手动打包 jar 或 war
手动整理多模块依赖顺序
```

项目小的时候还能忍，项目大了以后会很容易失控。

Maven 的思路是：

```text
用标准目录约定源码位置
用 pom.xml 描述项目和依赖
用仓库保存依赖和构建产物
用生命周期串起编译、测试、打包、发布
用插件扩展具体构建动作
```

比如执行：

```bash
mvn package
```

Maven 会自动完成：

```text
读取 pom.xml
解析依赖
下载 jar
编译 main 代码
编译 test 代码
运行测试
打包 jar / war
```

### Maven 在项目里的位置

一个常见 Spring Boot 项目大致是这样：

```text
开发代码
  |
  v
pom.xml
  |
  v
Maven
  |
  +--> 下载依赖
  +--> 编译代码
  +--> 运行测试
  +--> 打包应用
  +--> 安装或发布构建产物
```

Maven 本身不负责业务逻辑。

它更像项目的构建管家：按照 `pom.xml` 和命令，把构建流程跑完。

### 安装和验证

Maven 需要 Java 环境。

常见准备：

```text
JDK
Maven
IDEA 或其他 IDE
```

验证 Java：

```bash
java -version
```

验证 Maven：

```bash
mvn -version
```

输出里通常会看到：

```text
Apache Maven ...
Java version ...
Java home ...
Default locale ...
OS name ...
```

### settings.xml

`settings.xml` 是 Maven 的用户或全局配置文件。

常见位置：

```text
用户级：~/.m2/settings.xml
全局级：Maven 安装目录/conf/settings.xml
```

常见用途：

* 配置本地仓库位置
* 配置镜像仓库
* 配置代理
* 配置私服账号
* 配置 profile

示例：

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0
                              https://maven.apache.org/xsd/settings-1.2.0.xsd">

    <localRepository>${user.home}/.m2/repository</localRepository>

    <mirrors>
        <mirror>
            <id>company-nexus</id>
            <name>Company Nexus</name>
            <url>https://nexus.example.com/repository/maven-public/</url>
            <mirrorOf>*</mirrorOf>
        </mirror>
    </mirrors>
</settings>
```

`settings.xml` 更适合放机器、账号、镜像、私服这类环境配置。

项目自己的依赖、插件、模块关系，应该放在 `pom.xml`。

### Maven Wrapper

除了本机提前安装 Maven，还有一种常见方式叫 `Maven Wrapper`。

它的作用是：

```text
项目自带 Maven 启动脚本
第一次执行时自动下载指定版本 Maven
后续构建都使用项目指定的 Maven 版本
```

常见文件结构：

```text
project
├── mvnw
├── mvnw.cmd
├── .mvn
│   └── wrapper
│       ├── maven-wrapper.jar
│       └── maven-wrapper.properties
└── pom.xml
```

其中：

| 文件 | 作用 |
| --- | --- |
| `mvnw` | Linux / macOS 下的 Maven Wrapper 启动脚本 |
| `mvnw.cmd` | Windows 下的 Maven Wrapper 启动脚本 |
| `.mvn/wrapper/maven-wrapper.properties` | 指定 Maven 下载地址和版本 |
| `.mvn/wrapper/maven-wrapper.jar` | Wrapper 启动逻辑 |

使用方式：

```bash
./mvnw clean package
```

Windows：

```bash
mvnw.cmd clean package
```

它和直接执行 `mvn clean package` 的区别：

```text
mvn      使用本机安装的 Maven
./mvnw   使用项目指定的 Maven
```

团队项目更常用 Wrapper。

好处是不同机器、CI 环境、开发环境都能使用同一个 Maven 版本，减少“本地能构建，流水线不能构建”的问题。

`.mvn/wrapper/maven-wrapper.properties` 示例：

```properties
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.9/apache-maven-3.9.9-bin.zip
```

### 标准目录结构

Maven 有一套约定目录。

```text
maven-demo
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   └── resources
│   └── test
│       ├── java
│       └── resources
└── target
```

说明：

| 目录 | 作用 |
| --- | --- |
| `src/main/java` | 业务源码 |
| `src/main/resources` | 业务配置、资源文件 |
| `src/test/java` | 测试源码 |
| `src/test/resources` | 测试配置、资源文件 |
| `target` | 编译和打包输出目录 |
| `pom.xml` | Maven 项目配置 |

`target` 是构建生成目录。

它通常不提交到 Git。

### 第一个 Maven 项目

目录：

```text
hello-maven
├── pom.xml
└── src
    └── main
        └── java
            └── com
                └── example
                    └── Main.java
```

`pom.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>hello-maven</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
</project>
```

`Main.java`：

```java
package com.example;

public class Main {

    public static void main(String[] args) {
        System.out.println("Hello Maven");
    }
}
```

编译：

```bash
mvn compile
```

打包：

```bash
mvn package
```

生成文件：

```text
target/hello-maven-1.0.0.jar
```

### POM 是什么

`POM` 全称是 `Project Object Model`。

Maven 项目的核心配置文件就是：

```text
pom.xml
```

一个 POM 可以描述：

* 项目坐标
* 打包方式
* 依赖
* 插件
* 父子关系
* 多模块关系
* 仓库
* 发布配置
* profile

最小 POM 通常需要这些内容：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>1.0.0</version>
</project>
```

### Maven 坐标

Maven 用坐标定位一个构建产物。

核心三元组：

```text
groupId + artifactId + version
```

示例：

```xml
<groupId>com.example</groupId>
<artifactId>order-service</artifactId>
<version>1.0.0</version>
```

常见含义：

| 字段 | 说明 |
| --- | --- |
| `groupId` | 组织或项目组标识，常用反向域名 |
| `artifactId` | 构建产物名称 |
| `version` | 版本号 |
| `packaging` | 打包方式，默认是 `jar` |

在仓库里通常会变成类似路径：

```text
com/example/order-service/1.0.0/order-service-1.0.0.jar
```

### packaging

`packaging` 表示项目最终打成什么类型。

常见值：

| packaging | 场景 |
| --- | --- |
| `jar` | 普通 Java 项目、Spring Boot 应用 |
| `war` | 传统 Web 应用，部署到外部 Servlet 容器 |
| `pom` | 父工程、聚合工程 |
| `maven-plugin` | Maven 插件项目 |

如果不写：

```xml
<packaging>jar</packaging>
```

Maven 默认按 `jar` 处理。

### properties

`properties` 常用来统一配置版本和编码。

```xml
<properties>
    <java.version>21</java.version>
    <maven.compiler.release>${java.version}</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <junit.jupiter.version>5.11.4</junit.jupiter.version>
</properties>
```

后面可以这样引用：

```xml
<version>${junit.jupiter.version}</version>
```

这比在多个地方重复写版本号更容易维护。

### dependencies

项目依赖写在 `<dependencies>` 里。

示例：添加 JUnit 5 测试依赖。

```xml
<dependencies>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>${junit.jupiter.version}</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

一个依赖通常由这些元素组成：

| 元素 | 作用 |
| --- | --- |
| `groupId` | 依赖所属组织 |
| `artifactId` | 依赖名称 |
| `version` | 依赖版本 |
| `scope` | 依赖作用范围 |
| `optional` | 是否可选传递 |
| `exclusions` | 排除传递依赖 |

### 依赖作用域 scope

`scope` 决定依赖在哪些阶段可见。

| scope | 编译 | 测试 | 运行 | 常见例子 |
| --- | --- | --- | --- | --- |
| `compile` | 是 | 是 | 是 | 普通业务依赖，默认值 |
| `provided` | 是 | 是 | 否 | Servlet API，运行环境提供 |
| `runtime` | 否 | 是 | 是 | JDBC 驱动 |
| `test` | 否 | 是 | 否 | JUnit、Mockito |
| `system` | 是 | 是 | 否 | 本地指定 jar，较少使用 |
| `import` | 否 | 否 | 否 | BOM 导入，只能用于 `dependencyManagement` |

示例：MySQL 驱动通常用 `runtime`。

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

### 依赖传递

Maven 会处理传递依赖。

例如：

```text
项目 A 依赖 B
B 依赖 C
```

项目 A 通常也能用到 C。

这就是传递依赖。

好处是不用手动把每个底层 jar 都写一遍。

问题是依赖树变复杂后，可能出现版本冲突。

### 依赖冲突和依赖仲裁

同一个依赖出现多个版本时，Maven 需要选一个最终版本。

常见规则：

```text
路径更短的版本优先
路径一样长时，先声明的版本优先
```

示意：

```text
当前项目
├── A
│   └── commons-lang3:3.12.0
└── B
    └── C
        └── commons-lang3:3.14.0
```

这里 `3.12.0` 路径更短，优先被选中。

查看依赖树：

```bash
mvn dependency:tree
```

只看某个依赖：

```bash
mvn dependency:tree -Dincludes=org.apache.commons:commons-lang3
```

### 排除依赖

如果某个传递依赖不想引入，可以使用 `exclusions`。

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>legacy-sdk</artifactId>
    <version>1.0.0</version>

    <exclusions>
        <exclusion>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

排除后，如果项目仍然需要另一个版本，可以显式添加：

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>jcl-over-slf4j</artifactId>
    <version>${jcl-over-slf4j.version}</version>
</dependency>
```

### dependencyManagement

`dependencyManagement` 不会直接引入依赖。

它只是管理依赖版本。

常见于父 POM：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>${spring-boot.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

子模块使用时可以省略版本：

```xml
<dependencies>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

简单区分：

```text
dependencies：真正引入依赖
dependencyManagement：只管理依赖版本
```

### 生命周期

Maven 有三套生命周期：

| 生命周期 | 作用 |
| --- | --- |
| `clean` | 清理构建输出 |
| `default` | 编译、测试、打包、安装、发布 |
| `site` | 生成项目站点文档 |

最常用的是 `default` 生命周期。

核心阶段：

```text
validate
compile
test
package
verify
install
deploy
```

阶段是有顺序的。

执行：

```bash
mvn package
```

会依次执行：

```text
validate -> compile -> test -> package
```

执行：

```bash
mvn install
```

会依次执行：

```text
validate -> compile -> test -> package -> verify -> install
```

### 常用命令

| 命令 | 作用 |
| --- | --- |
| `mvn clean` | 删除 `target` |
| `mvn compile` | 编译主代码 |
| `mvn test` | 执行测试 |
| `mvn package` | 打包 |
| `mvn verify` | 运行检查并验证包 |
| `mvn install` | 安装到本地仓库 |
| `mvn deploy` | 发布到远程仓库 |
| `mvn dependency:tree` | 查看依赖树 |
| `mvn help:effective-pom` | 查看最终生效的 POM |
| `mvn help:effective-settings` | 查看最终生效的 settings |

常见组合：

```bash
mvn clean package
```

跳过测试执行，但仍编译测试代码：

```bash
mvn clean package -DskipTests
```

跳过测试编译和测试执行：

```bash
mvn clean package -Dmaven.test.skip=true
```

强制更新快照依赖：

```bash
mvn clean package -U
```

使用指定 profile：

```bash
mvn clean package -Pprod
```

### 插件和目标

Maven 真正执行动作的是插件。

比如：

```text
maven-compiler-plugin 负责编译
maven-surefire-plugin 负责运行单元测试
maven-jar-plugin 负责打 jar
maven-install-plugin 负责安装到本地仓库
maven-deploy-plugin 负责发布到远程仓库
```

命令里的 `dependency:tree` 可以拆成：

```text
dependency 插件
tree 目标
```

格式是：

```text
mvn 插件前缀:目标
```

例如：

```bash
mvn dependency:tree
```

```bash
mvn help:effective-pom
```

### build 配置

`build` 用来配置构建过程。

常见配置：

* 打包文件名
* 插件
* 资源过滤
* 源码目录
* 测试目录

示例：

```xml
<build>
    <finalName>hello-maven</finalName>

    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>${maven-compiler-plugin.version}</version>
            <configuration>
                <release>${java.version}</release>
            </configuration>
        </plugin>

        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>${maven-surefire-plugin.version}</version>
        </plugin>
    </plugins>
</build>
```

### 资源文件和打包

`src/main/resources` 下的文件会被复制到 `target/classes`。

示例：

```text
src/main/resources
├── application.properties
└── banner.txt
```

打包后会进入 jar：

```text
application.properties
banner.txt
```

Java 代码读取资源：

```java
import java.io.InputStream;
import java.nio.charset.StandardCharsets;

public class ResourceDemo {

    public static void main(String[] args) throws Exception {
        try (InputStream inputStream = ResourceDemo.class
                .getClassLoader()
                .getResourceAsStream("banner.txt")) {

            if (inputStream == null) {
                throw new IllegalStateException("资源文件不存在");
            }

            String content = new String(inputStream.readAllBytes(), StandardCharsets.UTF_8);
            System.out.println(content);
        }
    }
}
```

### 资源过滤

资源过滤可以把 Maven 属性替换到配置文件里。

`pom.xml`：

```xml
<properties>
    <app.env>dev</app.env>
</properties>

<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
        </resource>
    </resources>
</build>
```

`src/main/resources/app.properties`：

```properties
app.env=${app.env}
app.version=${project.version}
```

打包后：

```properties
app.env=dev
app.version=1.0.0
```

如果项目里有二进制资源，比如图片、字体、证书，过滤时需要谨慎配置，避免内容被当成文本替换。

### 可运行 jar Demo

普通 jar 默认没有启动类信息。

如果希望用下面命令运行：

```bash
java -jar target/hello-maven.jar
```

可以配置 `maven-jar-plugin`：

```xml
<build>
    <finalName>hello-maven</finalName>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <version>${maven-jar-plugin.version}</version>
            <configuration>
                <archive>
                    <manifest>
                        <mainClass>com.example.Main</mainClass>
                    </manifest>
                </archive>
            </configuration>
        </plugin>
    </plugins>
</build>
```

打包：

```bash
mvn clean package
```

运行：

```bash
java -jar target/hello-maven.jar
```

如果项目有第三方依赖，普通 jar 不会自动把依赖打进去。

这类场景可以使用 Spring Boot 插件、Shade 插件或 Assembly 插件生成可执行包。

### Spring Boot 可执行 jar

普通 Maven `jar` 通常只包含当前项目自己的 class 和资源。

第三方依赖不会自动打进去。

Spring Boot 项目常见的可执行 jar 是另一种结构：

```text
boot-demo.jar
├── BOOT-INF
│   ├── classes
│   │   └── 当前项目编译后的 class 和资源
│   └── lib
│       ├── spring-core-xxx.jar
│       ├── jackson-databind-xxx.jar
│       └── 其他依赖 jar
└── org
    └── springframework
        └── boot
            └── loader
```

也就是说：

```text
外层是一个 jar
里面放当前项目 class
里面还放一批依赖 jar
```

依赖 jar 通常不是被打散混进业务 class 目录，而是原样放在 `BOOT-INF/lib`。

这个结构由 `spring-boot-maven-plugin:repackage` 生成。

普通 jar 和 Boot jar 的差异可以这样看：

```bash
jar tf target/boot-demo.jar
```

如果看到：

```text
BOOT-INF/classes/
BOOT-INF/lib/
org/springframework/boot/loader/
```

说明它是 Spring Boot 可执行 jar。

### Maven 打包和 Spring Boot repackage

执行：

```bash
mvn clean package
```

对于 `jar` 项目，Maven 的默认流程大致是：

```text
maven-resources-plugin    复制资源
maven-compiler-plugin     编译 Java
maven-surefire-plugin     执行单元测试
maven-jar-plugin          打普通 jar
```

如果 Spring Boot 的 `repackage` 已经绑定到生命周期，还会继续执行：

```text
spring-boot-maven-plugin:repackage
```

完整链路可以理解成：

```text
.java
  |
  v
maven-compiler-plugin 编译
  |
  v
target/classes
  |
  v
maven-jar-plugin 打普通 jar
  |
  v
普通 jar
  |
  v
spring-boot-maven-plugin:repackage
  |
  v
可执行 Spring Boot jar
```

`maven-compiler-plugin` 只负责编译。

`maven-jar-plugin` 负责生成普通 jar。

`spring-boot-maven-plugin` 负责 Spring Boot 特有能力，其中最常见的是把普通 jar 重新包装成可执行 Boot jar。

如果使用 `spring-boot-starter-parent`，并且配置了：

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```

`mvn clean package` 通常会执行 `repackage`。

如果没有使用 Spring Boot parent，或者希望配置更明确，可以显式绑定：

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

打包后常见文件：

```text
target/demo.jar
target/demo.jar.original
```

含义：

```text
demo.jar           Spring Boot 可执行 jar
demo.jar.original  Maven 原始普通 jar
```

### maven-compiler-plugin 是否需要配置

`maven-compiler-plugin` 不写，`mvn clean package` 也能编译。

因为 Maven 的 `jar` 项目默认会在 `compile` 阶段调用编译插件。

显式配置它通常是为了控制编译细节：

* Java 编译版本
* 编码
* 编译参数
* 注解处理器

例如 Lombok 和 MapStruct：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <release>21</release>
        <annotationProcessorPaths>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version>
            </path>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>${mapstruct.version}</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

这里的重点是：

```text
编译阶段启用 Lombok 和 MapStruct 的注解处理器
```

`maven-jar-plugin` 也可以不写。

只要项目打包方式是 `jar`，`mvn package` 默认会调用 `maven-jar-plugin:jar`。

显式配置 `maven-jar-plugin`，通常是为了设置：

* `Main-Class`
* manifest 内容
* 打包排除项
* jar 文件名

### 安装到本地仓库

`mvn install` 会把构建产物安装到本地仓库。

例如项目坐标：

```xml
<groupId>com.example</groupId>
<artifactId>common-utils</artifactId>
<version>1.0.0</version>
```

执行：

```bash
mvn clean install
```

本地仓库里会出现：

```text
~/.m2/repository/com/example/common-utils/1.0.0/
```

另一个本地项目就可以依赖它：

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>common-utils</artifactId>
    <version>1.0.0</version>
</dependency>
```

### 发布到远程仓库

`mvn deploy` 会把构建产物发布到远程仓库。

常见于公司私服，比如 Nexus、Artifactory。

POM 里配置发布地址：

```xml
<distributionManagement>
    <repository>
        <id>company-releases</id>
        <url>https://nexus.example.com/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>company-snapshots</id>
        <url>https://nexus.example.com/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

账号密码通常放在 `settings.xml`：

```xml
<servers>
    <server>
        <id>company-releases</id>
        <username>${env.NEXUS_USERNAME}</username>
        <password>${env.NEXUS_PASSWORD}</password>
    </server>
</servers>
```

这里的 `id` 要和 `distributionManagement` 里的 `id` 对上。

发布：

```bash
mvn clean deploy
```

### SNAPSHOT 和 Release

Maven 版本里常见两类：

```text
1.0.0-SNAPSHOT
1.0.0
```

`SNAPSHOT` 表示开发中的快照版本。

同一个 `1.0.0-SNAPSHOT` 可以反复发布，Maven 会按快照策略检查更新。

`1.0.0` 表示正式版本。

正式版本发布后通常不再覆盖。

常见约定：

```text
开发阶段：1.0.0-SNAPSHOT
发布阶段：1.0.0
下一个开发阶段：1.0.1-SNAPSHOT
```

从包体看，`SNAPSHOT` 和 Release 没有天然结构差异。

如果源码、依赖、构建参数完全一样，包内容可以非常接近。

但实际项目里常会出现这些差异：

* 文件名不同
* manifest 里的版本不同
* `build-info` 里的版本不同
* 资源过滤写入的 `${project.version}` 不同
* `SNAPSHOT` 依赖可能解析到不同时间戳版本

真正稳定的差异在仓库行为：

```text
SNAPSHOT 可以反复发布和更新
Release 发布后通常不再覆盖
```

### profile

`profile` 用来按环境切换配置。

示例：

```xml
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <app.env>dev</app.env>
        </properties>
    </profile>

    <profile>
        <id>prod</id>
        <properties>
            <app.env>prod</app.env>
        </properties>
    </profile>
</profiles>
```

使用：

```bash
mvn clean package -Pprod
```

配合资源过滤，`app.properties` 里的 `${app.env}` 会被替换成 `prod`。

### effective-pom

实际构建时，Maven 看到的 POM 不只是当前文件。

它还会合并：

* Super POM
* 父 POM
* 当前 POM
* profile
* 默认插件绑定

查看最终生效的 POM：

```bash
mvn help:effective-pom
```

输出到文件：

```bash
mvn help:effective-pom -Doutput=effective-pom.xml
```

适合排查：

* 依赖版本从哪里来
* 插件版本是否生效
* profile 是否启用
* 父 POM 是否覆盖配置

### effective-settings

查看最终生效的 settings：

```bash
mvn help:effective-settings
```

输出到文件：

```bash
mvn help:effective-settings -Doutput=effective-settings.xml
```

适合排查：

* 本地仓库路径
* 镜像是否生效
* 私服账号配置
* profile 是否来自 settings

### 多模块项目

多模块项目常见于中大型系统。

示例结构：

```text
mall-system
├── pom.xml
├── mall-common
│   └── pom.xml
├── mall-domain
│   └── pom.xml
├── mall-service
│   └── pom.xml
└── mall-web
    └── pom.xml
```

父工程 `pom.xml`：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>mall-system</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <modules>
        <module>mall-common</module>
        <module>mall-domain</module>
        <module>mall-service</module>
        <module>mall-web</module>
    </modules>

    <properties>
        <java.version>21</java.version>
        <maven.compiler.release>${java.version}</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
</project>
```

子模块 `mall-service/pom.xml`：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>mall-system</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <artifactId>mall-service</artifactId>

    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>mall-domain</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>
</project>
```

在父工程目录执行：

```bash
mvn clean install
```

Maven 会根据模块关系计算构建顺序。

### 继承和聚合的区别

这两个概念经常放在一起，但不是一回事。

### 继承

子 POM 使用 `<parent>` 继承父 POM。

作用：

* 继承 `groupId`
* 继承 `version`
* 继承 `properties`
* 继承 `dependencyManagement`
* 继承 `pluginManagement`

### 聚合

父 POM 使用 `<modules>` 聚合多个模块。

作用：

* 一次构建多个模块
* 自动计算模块构建顺序
* 统一执行 `clean install`

简单区分：

```text
继承：配置复用
聚合：一起构建
```

实际项目里，父工程经常同时承担这两个角色。

### pluginManagement

`pluginManagement` 和 `dependencyManagement` 很像。

它只管理插件版本和默认配置，不会主动执行插件。

父 POM：

```xml
<build>
    <pluginManagement>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>${maven-compiler-plugin.version}</version>
                <configuration>
                    <release>${java.version}</release>
                </configuration>
            </plugin>
        </plugins>
    </pluginManagement>
</build>
```

子模块启用插件：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

简单区分：

```text
plugins：真正启用插件
pluginManagement：统一管理插件版本和默认配置
```

### Spring Boot Maven Demo

Spring Boot 项目常见 POM：

下面版本只是示例，实际项目按团队使用的 Spring Boot 版本替换。

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.0</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>boot-maven-demo</artifactId>
    <version>1.0.0-SNAPSHOT</version>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

打包：

```bash
mvn clean package
```

运行：

```bash
java -jar target/boot-maven-demo-1.0.0-SNAPSHOT.jar
```

Spring Boot 父 POM 或 BOM 会管理大量依赖版本。

所以很多 Spring Boot starter 不需要显式写版本。

### Spring Boot 开发和生产命令

开发阶段常见方式：

```bash
mvn spring-boot:run
```

或者直接在 IDE 里运行启动类的 `main` 方法。

打包阶段：

```bash
mvn clean package
```

跳过测试执行：

```bash
mvn clean package -DskipTests
```

跳过测试编译和测试执行：

```bash
mvn clean package -Dmaven.test.skip=true
```

生产运行：

```bash
java -jar target/boot-maven-demo-1.0.0-SNAPSHOT.jar
```

如果要发布到公司私服：

```bash
mvn clean deploy
```

### spring-boot:run 和 spring-boot:start

`spring-boot:run` 用于前台启动应用。

```bash
mvn spring-boot:run
```

特点：

```text
适合开发调试
Maven 进程持续运行
控制台持续输出日志
按 Ctrl + C 停止
```

`spring-boot:start` 更多用于集成测试流程。

```bash
mvn spring-boot:start
```

特点：

```text
启动应用
不长期阻塞 Maven 后续阶段
通常配合 spring-boot:stop 使用
```

典型流程：

```text
pre-integration-test   spring-boot:start
integration-test       执行接口或集成测试
post-integration-test  spring-boot:stop
```

简单区分：

```text
spring-boot:run    开发时前台启动
spring-boot:start  集成测试前启动
spring-boot:stop   集成测试后停止
```

### Spring Boot 插件是否需要配置

`spring-boot-maven-plugin` 不是普通 Maven 编译必需项。

不写它，`mvn clean package` 仍然可以：

```text
编译代码
执行测试
打普通 jar
```

但不写它，通常不会生成标准 Spring Boot 可执行 jar。

Spring Boot 项目配置它，主要是为了：

* `spring-boot:run` 开发启动
* `spring-boot:repackage` 生成可执行 Boot jar
* `spring-boot:build-image` 构建镜像
* `build-info` 生成构建信息
* AOT 相关处理

所以可以这样理解：

```text
普通 Maven 构建：不一定需要 spring-boot-maven-plugin
Spring Boot 可执行 jar：通常需要 spring-boot-maven-plugin
```

### Java 和 .NET 发布形态对比

Java 普通 jar 和 .NET dll 都可以理解成编译产物容器。

普通 Java 发布也可以是：

```text
app.jar
lib/a.jar
lib/b.jar
lib/c.jar
```

.NET 常见发布目录类似：

```text
App.dll
DependencyA.dll
DependencyB.dll
DependencyC.dll
```

Spring Boot 可执行 jar 则更像：

```text
app.jar
  ├── 当前项目 class
  └── 依赖 jar
```

外面看是一个 jar。

里面看仍然是“主程序 + 依赖包”。

所以更准确的说法是：

```text
Spring Boot 把依赖 jar 放进外层 jar 内部
.NET 默认发布常见是多个 dll 放在同一个目录
二者本质都是主程序加依赖包
```

### IDEA 中使用 Maven

IDEA 里常见位置：

```text
Settings
  |
  v
Build, Execution, Deployment
  |
  v
Build Tools
  |
  v
Maven
```

常见配置：

* Maven home path
* User settings file
* Local repository
* JDK for importer
* Runner JRE

常见操作：

* 刷新 Maven 项目
* 展开 Lifecycle 执行 `clean`、`package`、`install`
* 展开 Dependencies 查看依赖
* 使用 Reload 重新导入 POM

如果改了 `pom.xml` 后依赖没有生效，通常先刷新 Maven 项目。

### IDEA 里的 Lifecycle 和 Plugins

IDEA Maven 面板里常见两块：

```text
Lifecycle
Plugins
```

`Lifecycle` 执行的是 Maven 阶段。

比如点 `package`，相当于：

```bash
mvn package
```

它会按生命周期顺序执行前面的阶段：

```text
validate
compile
test
package
```

中间每个阶段再由对应插件完成具体动作。

`Plugins` 执行的是某个插件的某个目标。

例如：

```bash
mvn compiler:compile
mvn jar:jar
mvn spring-boot:run
mvn spring-boot:repackage
```

简单区分：

```text
Lifecycle：按 Maven 标准流程构建
Plugins：直接执行某个插件功能
```

日常构建通常点 `Lifecycle` 里的 `clean`、`package`、`install`。

只想执行某个特殊能力时，再点 `Plugins` 里的具体 goal。

### 常见排查命令

### 查看依赖树

```bash
mvn dependency:tree
```

### 查看某个依赖来源

```bash
mvn dependency:tree -Dincludes=org.slf4j:slf4j-api
```

### 查看最终 POM

```bash
mvn help:effective-pom
```

### 查看最终 settings

```bash
mvn help:effective-settings
```

### 查看插件帮助

```bash
mvn help:describe -Dplugin=compiler -Ddetail
```

### 离线构建

```bash
mvn -o clean package
```

离线构建要求依赖已经存在于本地仓库。

### 常见使用建议

### 版本集中管理

多模块项目里，依赖版本适合放在父 POM 的 `dependencyManagement`。

子模块只声明依赖，不到处重复写版本。

### 插件版本明确声明

核心插件建议统一声明版本。

比如：

* `maven-compiler-plugin`
* `maven-surefire-plugin`
* `maven-jar-plugin`
* `maven-resources-plugin`

这样可以减少构建结果随 Maven 默认插件版本变化而波动。

### 依赖冲突先看 dependency:tree

遇到 `ClassNotFoundException`、`NoSuchMethodError`、日志实现冲突时，先看依赖树。

```bash
mvn dependency:tree
```

再决定是否使用：

* 排除传递依赖
* 显式声明版本
* 使用 BOM
* 调整模块依赖关系

### 配置和代码分开

`pom.xml` 管项目构建。

应用运行配置放在：

```text
src/main/resources
```

比如：

* `application.yml`
* `application.properties`
* `logback-spring.xml`

### 减少 system scope

`system` 依赖绑定本地文件路径，不利于团队协作和 CI 构建。

更合适的方式是把 jar 安装到本地仓库或发布到公司私服。

本地安装第三方 jar：

```bash
mvn install:install-file \
  -Dfile=vendor-sdk.jar \
  -DgroupId=com.vendor \
  -DartifactId=vendor-sdk \
  -Dversion=1.0.0 \
  -Dpackaging=jar
```

项目里再用普通依赖引用：

```xml
<dependency>
    <groupId>com.vendor</groupId>
    <artifactId>vendor-sdk</artifactId>
    <version>1.0.0</version>
</dependency>
```

### 常用标签汇总

| 标签 | 作用 |
| --- | --- |
| `<groupId>` | 项目或依赖的组织标识 |
| `<artifactId>` | 项目或依赖名称 |
| `<version>` | 版本号 |
| `<packaging>` | 打包类型 |
| `<properties>` | 属性变量 |
| `<dependencies>` | 真正引入依赖 |
| `<dependencyManagement>` | 管理依赖版本 |
| `<build>` | 构建配置 |
| `<plugins>` | 启用插件 |
| `<pluginManagement>` | 管理插件版本和默认配置 |
| `<resources>` | 资源文件配置 |
| `<profiles>` | 环境配置切换 |
| `<modules>` | 聚合子模块 |
| `<parent>` | 继承父 POM |
| `<distributionManagement>` | 发布仓库配置 |

### 常用命令汇总

| 命令 | 作用 |
| --- | --- |
| `mvn -version` | 查看 Maven 和 Java 环境 |
| `mvn clean` | 清理构建输出 |
| `mvn compile` | 编译主代码 |
| `mvn test` | 执行测试 |
| `mvn package` | 打包 |
| `mvn verify` | 验证构建结果 |
| `mvn install` | 安装到本地仓库 |
| `mvn deploy` | 发布到远程仓库 |
| `mvn dependency:tree` | 查看依赖树 |
| `mvn dependency:analyze` | 分析依赖使用情况 |
| `mvn help:effective-pom` | 查看最终生效的 POM |
| `mvn help:effective-settings` | 查看最终生效的 settings |
| `mvn help:describe` | 查看插件目标说明 |

### 总结

Maven 的核心不是 XML 本身，而是一套约定。

它把 Java 项目拆成几个固定部分：

```text
目录结构
POM
坐标
仓库
依赖
生命周期
插件
多模块
```

日常开发最常用的是：

```text
mvn clean package
mvn clean install
mvn dependency:tree
mvn help:effective-pom
```

理解 `dependencies`、`dependencyManagement`、生命周期、插件、多模块之后，Maven 的大部分问题都可以顺着依赖树和 effective POM 排查出来。
