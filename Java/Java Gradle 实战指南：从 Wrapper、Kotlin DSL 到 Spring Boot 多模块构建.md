### 简介

Gradle 是 Java 生态里常见的构建工具。

它负责把一个项目从源码变成可运行、可测试、可发布的构建产物。

一个 Java 项目通常会涉及这些事情：

```text
下载依赖
编译源码
处理资源文件
运行测试
打包 jar / bootJar
发布到 Maven 仓库
组织多模块构建
```

Gradle 的核心特点是：

```text
用插件提供构建能力
用任务组织构建动作
用 DSL 描述构建配置
用 Wrapper 固定项目 Gradle 版本
```

一句话概括：

```text
Gradle 是一个灵活、可扩展、支持增量构建和多模块工程的现代构建工具。
```

### Gradle 解决什么问题

没有构建工具时，Java 项目需要手动处理很多事情：

```text
手动下载 jar
手动配置 classpath
手动执行 javac
手动运行测试
手动打包 jar
手动处理多模块编译顺序
```

项目小的时候还能接受，项目变大后很容易失控。

Gradle 的做法是：

```text
标准目录约定
声明依赖
应用插件
执行任务
自动计算任务顺序
复用构建缓存
```

比如执行：

```bash
./gradlew build
```

Gradle 会自动完成：

```text
读取 settings.gradle.kts
读取 build.gradle.kts
解析插件和依赖
编译 main 代码
编译 test 代码
运行测试
打包产物
生成构建结果
```

### Gradle 和 Maven 的关系

Gradle 和 Maven 都是构建工具，都能使用 Maven 仓库里的依赖。

区别主要在写法和构建模型上。

| 对比项 | Maven | Gradle |
| --- | --- | --- |
| 配置文件 | `pom.xml` | `build.gradle` / `build.gradle.kts` |
| 配置风格 | XML | Groovy DSL / Kotlin DSL |
| 生命周期 | 固定生命周期 | 任务图 DAG |
| 插件机制 | 插件绑定生命周期阶段 | 插件添加任务和扩展 |
| 灵活性 | 约定强，写法稳定 | 可编程能力更强 |
| 多模块 | 父 POM + modules | settings include + project dependency |
| 依赖仓库 | Maven 仓库 | Maven 仓库、Ivy 仓库等 |

Maven 更像：

```text
按照固定生命周期跑标准流程。
```

Gradle 更像：

```text
根据任务依赖关系生成任务图，再按任务图执行。
```

### Gradle Wrapper

实际项目里通常不直接使用本机的 `gradle` 命令，而是使用 Gradle Wrapper。

```bash
./gradlew build
```

Windows 下是：

```bash
gradlew.bat build
```

Wrapper 的作用是让项目固定使用某个 Gradle 版本。

常见文件结构：

```text
gradle-demo/
├── gradlew
├── gradlew.bat
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── settings.gradle.kts
└── build.gradle.kts
```

`gradle-wrapper.properties` 里会指定 Gradle 发行包地址：

```properties
distributionUrl=https\://services.gradle.org/distributions/gradle-9.6.1-bin.zip
```

这样无论本机是否安装 Gradle，只要有 JDK，就可以通过 `./gradlew` 执行构建。

生成 Wrapper：

```bash
gradle wrapper --gradle-version 9.6.1
```

升级 Wrapper：

```bash
./gradlew wrapper --gradle-version 9.6.1
```

项目里的 Wrapper 文件应该提交到 Git。

这样 CI、开发机、构建服务器都能使用同一个 Gradle 版本。

### Gradle 运行 JDK 和项目编译 JDK

Gradle 自己运行需要 JVM。

当前 Gradle 9.x 要求运行 Gradle 的 JVM 至少是 Java 17。

但项目代码编译使用哪个 JDK，可以通过 Java Toolchain 单独指定。

```kotlin
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

这表示：

```text
Gradle 自己可以运行在 Java 17+
项目源码使用 Java 21 工具链编译
```

Toolchain 的好处是构建更稳定。

不同机器上的 `JAVA_HOME` 不一致时，Gradle 仍然可以选择符合要求的 JDK 来编译项目。

### Gradle 项目目录

标准 Java 项目目录和 Maven 很接近。

```text
gradle-java-demo/
├── settings.gradle.kts
├── build.gradle.kts
├── gradle.properties
├── gradlew
├── gradlew.bat
├── gradle/
│   └── wrapper/
└── src/
    ├── main/
    │   ├── java/
    │   └── resources/
    └── test/
        ├── java/
        └── resources/
```

几个核心文件：

| 文件 | 作用 |
| --- | --- |
| `settings.gradle.kts` | 设置根项目名称、声明子模块、配置插件仓库 |
| `build.gradle.kts` | 当前项目或模块的构建脚本 |
| `gradle.properties` | Gradle 参数、项目属性 |
| `gradlew` | macOS / Linux Wrapper 命令 |
| `gradlew.bat` | Windows Wrapper 命令 |
| `gradle-wrapper.properties` | Wrapper 使用的 Gradle 版本 |

### settings.gradle.kts

单模块项目：

```kotlin
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        mavenCentral()
    }
}

rootProject.name = "gradle-java-demo"
```

多模块项目：

```kotlin
rootProject.name = "mall-system"

include("mall-common")
include("mall-domain")
include("mall-service")
include("mall-web")
```

`settings.gradle.kts` 是 Gradle 初始化阶段会读取的文件。

它决定这次构建包含哪些项目。

### build.gradle.kts 基础模板

下面是一个普通 Java 应用的 `build.gradle.kts`。

```kotlin
plugins {
    java
    application
}

group = "com.example"
version = "1.0.0-SNAPSHOT"

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.slf4j:slf4j-api:2.0.18")
    runtimeOnly("ch.qos.logback:logback-classic:1.5.15")

    testImplementation("org.junit.jupiter:junit-jupiter:5.11.4")
}

application {
    mainClass.set("com.example.demo.MainApplication")
}

tasks.test {
    useJUnitPlatform()
}
```

这个文件做了几件事：

| 配置 | 作用 |
| --- | --- |
| `plugins` | 应用 Java 和 Application 插件 |
| `group` | 项目组织名 |
| `version` | 项目版本 |
| `java.toolchain` | 指定 Java 编译工具链 |
| `repositories` | 依赖从哪里下载 |
| `dependencies` | 项目依赖 |
| `application.mainClass` | 可运行应用入口类 |
| `tasks.test` | 配置测试任务 |

### Groovy DSL 和 Kotlin DSL

Gradle 支持两种常见 DSL：

| 文件 | DSL |
| --- | --- |
| `build.gradle` | Groovy DSL |
| `build.gradle.kts` | Kotlin DSL |

Groovy DSL：

```groovy
plugins {
    id 'java'
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

Kotlin DSL：

```kotlin
plugins {
    java
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
}
```

Kotlin DSL 的类型提示和 IDE 补全通常更好，新项目可以优先使用 `build.gradle.kts`。

老项目里遇到 `build.gradle` 也很正常。

### 常用命令

日常命令通常使用 Wrapper。

```bash
./gradlew tasks
```

查看所有任务。

```bash
./gradlew clean
```

清理 `build` 目录。

```bash
./gradlew compileJava
```

编译 main 源码。

```bash
./gradlew test
```

运行测试。

```bash
./gradlew build
```

完整构建，通常包含编译、测试、打包。

```bash
./gradlew jar
```

打普通 jar。

```bash
./gradlew dependencies
```

查看依赖树。

```bash
./gradlew dependencyInsight --dependency logback-classic
```

查看某个依赖为什么被引入。

```bash
./gradlew build -x test
```

构建时跳过测试。

### 构建生命周期

Gradle 构建有三个阶段：

| 阶段 | 做什么 |
| --- | --- |
| 初始化 | 读取 `settings.gradle.kts`，确定参与构建的项目 |
| 配置 | 读取各模块 `build.gradle.kts`，配置 Project 和 Task |
| 执行 | 根据任务图执行被请求的任务 |

执行：

```bash
./gradlew build
```

Gradle 不会简单地从上到下跑脚本。

它会先生成任务图，再执行任务图里的任务。

比如 `build` 可能依赖：

```text
compileJava
processResources
classes
compileTestJava
test
jar
assemble
check
```

插件会向项目添加任务，任务之间可以声明依赖关系。

### Task 是 Gradle 的执行单元

查看任务：

```bash
./gradlew tasks
```

自定义一个简单任务：

```kotlin
tasks.register("hello") {
    group = "demo"
    description = "打印一条 Gradle 消息"

    doLast {
        println("Hello Gradle")
    }
}
```

执行：

```bash
./gradlew hello
```

任务依赖：

```kotlin
tasks.register("packageInfo") {
    dependsOn("test")

    doLast {
        println("测试通过后生成打包信息")
    }
}
```

`dependsOn("test")` 表示执行 `packageInfo` 前先执行 `test`。

### Plugin 是能力入口

Gradle 很多能力都来自插件。

比如 Java 插件：

```kotlin
plugins {
    java
}
```

它会添加：

```text
compileJava
processResources
classes
jar
compileTestJava
test
build
```

Application 插件：

```kotlin
plugins {
    application
}
```

它会添加：

```text
run
installDist
distZip
distTar
```

Spring Boot 插件：

```kotlin
plugins {
    id("org.springframework.boot") version "3.5.0"
    id("io.spring.dependency-management") version "1.1.7"
    java
}
```

它会添加：

```text
bootRun
bootJar
bootBuildImage
```

简单理解：

```text
插件负责把某类构建能力接入项目。
任务负责真正执行构建动作。
```

### 依赖配置怎么选

Gradle 依赖不是只有一个 `compile`。

常见配置如下：

| 配置 | 说明 |
| --- | --- |
| `implementation` | 编译和运行需要，但不暴露给依赖当前模块的其他模块 |
| `api` | 编译和运行需要，并暴露给依赖当前模块的其他模块 |
| `compileOnly` | 编译需要，运行时由环境提供 |
| `runtimeOnly` | 编译不需要，运行时需要 |
| `testImplementation` | 测试编译和运行需要 |
| `testRuntimeOnly` | 测试运行时需要 |
| `annotationProcessor` | 注解处理器，比如 Lombok、MapStruct |

示例：

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")

    runtimeOnly("com.mysql:mysql-connector-j")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

`implementation` 和 `api` 的区别在多模块里很重要。

假设：

```text
mall-web -> mall-service -> mall-domain
```

如果 `mall-service` 的公开方法签名里暴露了 `mall-domain` 类型，就可以用 `api(project(":mall-domain"))`。

如果只是内部使用，就用 `implementation(project(":mall-domain"))`。

使用 `api` 需要应用 `java-library` 插件：

```kotlin
plugins {
    `java-library`
}
```

### 依赖排除和版本冲突

排除传递依赖：

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web") {
        exclude(group = "org.springframework.boot", module = "spring-boot-starter-tomcat")
    }
}
```

查看依赖来源：

```bash
./gradlew dependencyInsight --dependency spring-boot-starter-tomcat
```

刷新依赖缓存：

```bash
./gradlew --refresh-dependencies build
```

Gradle 会做版本冲突解析，通常选择较高版本。

Spring Boot 项目一般交给 Boot 的 BOM 和依赖管理控制版本，业务模块少手写三方依赖版本。

### Version Catalog：集中管理依赖版本

多模块项目里，依赖版本散落在各个 `build.gradle.kts` 中会很难维护。

Gradle 推荐使用 Version Catalog。

文件位置：

```text
gradle/libs.versions.toml
```

示例：

```toml
[versions]
spring-boot = "3.5.0"
junit-jupiter = "5.11.4"
mapstruct = "1.6.3"

[libraries]
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit-jupiter" }
mapstruct = { module = "org.mapstruct:mapstruct", version.ref = "mapstruct" }
mapstruct-processor = { module = "org.mapstruct:mapstruct-processor", version.ref = "mapstruct" }

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
```

`build.gradle.kts` 中使用：

```kotlin
plugins {
    alias(libs.plugins.spring.boot)
    java
}

dependencies {
    implementation(libs.mapstruct)
    annotationProcessor(libs.mapstruct.processor)
    testImplementation(libs.junit.jupiter)
}
```

这样版本集中在一个文件里，多模块升级更方便。

### Spring Boot Gradle 项目

一个常见 Spring Boot 单模块配置如下：

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.5.0"
    id("io.spring.dependency-management") version "1.1.7"
}

group = "com.example"
version = "1.0.0"

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    runtimeOnly("com.mysql:mysql-connector-j")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

tasks.test {
    useJUnitPlatform()
}
```

开发阶段运行：

```bash
./gradlew bootRun
```

打生产包：

```bash
./gradlew clean bootJar
```

产物一般在：

```text
build/libs/
```

运行：

```bash
java -jar build/libs/demo-1.0.0.jar
```

`bootJar` 会生成 Spring Boot 可执行 jar。

普通 `jar` 任务生成的是普通 jar，通常不包含 Spring Boot 启动所需的嵌套依赖结构。

### bootRun、bootJar、jar 的区别

| 任务 | 作用 | 常见命令 |
| --- | --- | --- |
| `bootRun` | 直接从源码和 runtime classpath 启动 Spring Boot 应用 | `./gradlew bootRun` |
| `bootJar` | 打 Spring Boot 可执行 jar | `./gradlew bootJar` |
| `jar` | 打普通 jar | `./gradlew jar` |
| `build` | 执行完整构建，通常会包含测试和打包 | `./gradlew build` |

开发阶段常用：

```bash
./gradlew bootRun
```

CI 或发版阶段常用：

```bash
./gradlew clean build
```

或者只打 Spring Boot 可执行包：

```bash
./gradlew clean bootJar
```

### 多模块项目

一个典型多模块结构：

```text
mall-system/
├── settings.gradle.kts
├── build.gradle.kts
├── mall-common/
│   └── build.gradle.kts
├── mall-domain/
│   └── build.gradle.kts
├── mall-service/
│   └── build.gradle.kts
└── mall-web/
    └── build.gradle.kts
```

`settings.gradle.kts`：

```kotlin
rootProject.name = "mall-system"

include("mall-common")
include("mall-domain")
include("mall-service")
include("mall-web")
```

根 `build.gradle.kts`：

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.5.0" apply false
    id("io.spring.dependency-management") version "1.1.7" apply false
}

allprojects {
    group = "com.example"
    version = "1.0.0"

    repositories {
        mavenCentral()
    }
}

subprojects {
    apply(plugin = "java")

    java {
        toolchain {
            languageVersion = JavaLanguageVersion.of(21)
        }
    }

    tasks.withType<Test>().configureEach {
        useJUnitPlatform()
    }
}
```

`mall-service/build.gradle.kts`：

```kotlin
plugins {
    `java-library`
}

dependencies {
    api(project(":mall-domain"))
    implementation(project(":mall-common"))
}
```

`mall-web/build.gradle.kts`：

```kotlin
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":mall-service"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

构建整个项目：

```bash
./gradlew clean build
```

只构建某个模块：

```bash
./gradlew :mall-web:build
```

只运行 Web 模块：

```bash
./gradlew :mall-web:bootRun
```

### gradle.properties

`gradle.properties` 常用于放 Gradle 构建参数和项目属性。

```properties
org.gradle.jvmargs=-Xmx2g -Dfile.encoding=UTF-8
org.gradle.parallel=true
org.gradle.caching=true

projectVersion=1.0.0
```

在 `build.gradle.kts` 中读取：

```kotlin
version = providers.gradleProperty("projectVersion").get()
```

常见配置：

| 配置 | 作用 |
| --- | --- |
| `org.gradle.jvmargs` | Gradle Daemon JVM 参数 |
| `org.gradle.parallel` | 多项目并行构建 |
| `org.gradle.caching` | 启用构建缓存 |
| 自定义属性 | 项目版本、仓库地址等 |

账号、Token、私服密码不适合直接提交到项目仓库。

这类信息可以放到用户目录的 `~/.gradle/gradle.properties`，或者由 CI 环境变量注入。

### 发布到 Maven 仓库

Gradle 可以用 `maven-publish` 插件发布构建产物。

```kotlin
plugins {
    `java-library`
    `maven-publish`
}

group = "com.example"
version = "1.0.0"

publishing {
    publications {
        create<MavenPublication>("mavenJava") {
            from(components["java"])
        }
    }

    repositories {
        maven {
            name = "localRepo"
            url = uri(layout.buildDirectory.dir("repo"))
        }
    }
}
```

发布到本地 Maven 仓库：

```bash
./gradlew publishToMavenLocal
```

发布到配置的仓库：

```bash
./gradlew publish
```

公共库、内部 SDK、多模块基础组件，经常会用到发布能力。

### 自定义任务示例：生成构建信息

下面定义一个任务，生成 `build/generated/build-info.txt`。

```kotlin
abstract class GenerateBuildInfoTask : DefaultTask() {

    @get:OutputFile
    abstract val outputFile: RegularFileProperty

    @TaskAction
    fun generate() {
        outputFile.get().asFile.writeText(
            """
            name=${project.name}
            version=${project.version}
            time=${java.time.Instant.now()}
            """.trimIndent()
        )
    }
}

tasks.register<GenerateBuildInfoTask>("generateBuildInfo") {
    outputFile.set(layout.buildDirectory.file("generated/build-info.txt"))
}
```

执行：

```bash
./gradlew generateBuildInfo
```

Gradle 任务可以声明输入和输出。

输入输出声明清楚后，Gradle 才能更好地做增量构建和缓存。

### IDEA 里 Gradle 面板怎么理解

IDEA 的 Gradle 面板通常会展示：

```text
Tasks
Dependencies
Plugins
```

`Tasks` 是可以执行的任务。

比如：

```text
build
clean
test
bootRun
bootJar
dependencies
```

双击任务，本质上类似执行：

```bash
./gradlew bootRun
```

`Plugins` 是当前项目应用的插件。

插件会贡献任务和配置项。

比如 Spring Boot 插件贡献 `bootRun`、`bootJar`，Java 插件贡献 `compileJava`、`test`、`jar`。

所以 IDEA 里真正执行的是任务，不是插件本身。

### 常见问题

#### gradle 和 gradlew 用哪个

项目内优先使用：

```bash
./gradlew build
```

它会使用项目指定的 Gradle 版本。

本机 `gradle` 更适合初始化 Wrapper 或做临时实验。

#### build 和 bootJar 有什么区别

`bootJar` 只负责生成 Spring Boot 可执行 jar。

`build` 是生命周期任务，会执行检查、测试、打包等一组任务。Spring Boot 项目里，`build` 通常也会触发 `bootJar`。

#### 为什么依赖下载很慢

常见原因是网络、仓库源、公司代理。

可以在 `settings.gradle.kts` 或企业统一 Gradle 配置里设置仓库。

```kotlin
repositories {
    mavenCentral()
    maven("https://maven.aliyun.com/repository/public")
}
```

企业项目更推荐使用内部 Nexus、Artifactory 这类私服。

#### 为什么修改代码后没有重新编译全部内容

Gradle 支持增量构建。

如果输入和输出没有变化，对应任务可能显示 `UP-TO-DATE`。

```text
> Task :compileJava UP-TO-DATE
```

这表示 Gradle 判断该任务不需要重新执行。

#### implementation 和 api 怎么选

默认优先用 `implementation`。

只有当前模块的公开 API 暴露了某个依赖类型时，才考虑 `api`。

这样可以减少无关依赖传递，提高编译隔离效果。

#### Spring Boot 项目能不能只执行 jar

可以执行 `jar`，但它生成的是普通 jar。

Spring Boot 应用通常使用：

```bash
./gradlew bootJar
```

`bootJar` 生成的 jar 才包含 Spring Boot 可执行应用所需的启动结构。

### 实践建议

| 场景 | 建议 |
| --- | --- |
| 项目构建 | 使用 `./gradlew` |
| 新项目 DSL | 优先使用 Kotlin DSL |
| Java 版本 | 使用 `java.toolchain` 固定 |
| Spring Boot 运行 | 开发阶段用 `bootRun` |
| Spring Boot 打包 | 发版阶段用 `clean bootJar` 或 `clean build` |
| 多模块依赖 | 默认 `implementation`，必要时 `api` |
| 依赖版本 | 多模块使用 Version Catalog |
| 依赖排查 | 使用 `dependencies` 和 `dependencyInsight` |
| 构建性能 | 开启缓存、并行构建，避免低质量自定义任务 |
| 敏感配置 | 放到用户级 Gradle 配置或 CI 环境变量 |

### 小结

Gradle 的核心不是某个单独命令，而是一套任务驱动的构建模型。

Wrapper 负责固定 Gradle 版本，`settings.gradle.kts` 决定构建包含哪些项目，`build.gradle.kts` 描述插件、依赖和任务，插件贡献具体构建能力，任务组成最终的执行图。

普通 Java 项目关注 `java`、`application`、`test`、`jar`。Spring Boot 项目还要关注 `bootRun` 和 `bootJar`。多模块项目则要进一步关注 `settings.gradle.kts`、模块依赖、`api` / `implementation`、Version Catalog 和统一构建配置。

把这些概念串起来后，Gradle 就不只是“能打包”的工具，而是可以支撑复杂工程构建、依赖治理和持续交付的构建系统。
