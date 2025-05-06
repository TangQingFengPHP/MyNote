### 简介

`Lombok` 是 `Java` 的一个 编译器插件，用于简化 `Java` 中常见样板代码（如 `getter/setter`、构造函数、`toString`、`equals/hashCode` 等）的编写，提高开发效率。

### Lombok 安装与配置

#### Maven 依赖：

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.30</version>
    <scope>provided</scope>
</dependency>
```

#### IDEA 插件支持：

* 安装插件：`Lombok Plugin`

* 启用注解处理：
    * Preferences → Build, Execution, Deployment → Compiler → Annotation Processors → ✅ Enable

### 常用注解

#### @Getter / @Setter

作用：自动生成字段的 `getter` 和 `setter` 方法。

```java
@Getter
@Setter
public class User {
    private String name;
    private int age;
}
```

#### @ToString

作用：生成 `toString()` 方法。

参数：`exclude`（排除字段）、`includeFieldNames`（是否包含字段名）。

```java
@ToString
public class User {
    private String name;
    private int age;
}
```

#### @EqualsAndHashCode

作用：生成 `equals()` 和 `hashCode()` 方法。

参数：`exclude`（排除字段）、`callSuper`（是否包含父类字段）。

```java
@EqualsAndHashCode
public class User {
    private String name;
    private int age;
}
```

#### @NoArgsConstructor / @AllArgsConstructor / @RequiredArgsConstructor

* `@NoArgsConstructor`：无参构造函数；

* `@AllArgsConstructor`：全参数构造函数；

* `@RequiredArgsConstructor`：包含 `final` 字段和带 `@NonNull` 字段的构造函数，自动注入依赖

```java
@NoArgsConstructor
@AllArgsConstructor
@RequiredArgsConstructor
public class User {
    private String name;
    private final int age;
}
```

#### @Data

相当于 `@Getter + @Setter + @ToString + @EqualsAndHashCode + @RequiredArgsConstructor`。

```java
@Data
public class User {
    private String name;
    private int age;
}
```

#### @Builder

构建者模式（`Builder Pattern`），支持链式构建对象。

```java
@Builder
public class User {
    private String name;
    private int age;
}

// 使用方式：
User user = User.builder()
                .name("Tom")
                .age(25)
                .build();
```

#### @Value

用于不可变对象，相当于 `@Data + final + private` + 构造函数。

将类标记为不可变（所有字段自动生成 `final` 和 `getter`，无 `setter`）。

```java
@Value
public class User {
    String name;
    int age;
}
```

#### @SneakyThrows

用于忽略受检异，不需写 `try-catch` 或 `throws`。

```java
@SneakyThrows
public void test() {
    throw new IOException("Error");
}
```

#### @Slf4j

自动生成日志对象 `log`，用于日志打印。

为类自动生成 `private static final Logger log = LoggerFactory.getLogger(...)`;

还有其他日志注解如 `@Log4j2, @CommonsLog, @JBossLog` 等。

```java
@Slf4j
public class Demo {
    public void test() {
        log.info("This is a log");
    }
}
```

#### @Cleanup

自动关闭资源，相当于 `try-with-resources`。

```java
import lombok.Cleanup;

public class Example {
    public void readFile() throws IOException {
        @Cleanup InputStream in = new FileInputStream("file.txt");
        // use the input stream
    }
}
```

#### @Accessors

* 配置 `Getter/Setter` 的生成规则
    * `chain=true：Setter` 返回当前对象，支持链式调用（`user.setName("A").setAge(20)`)
    * `fluent=true`：生成无前缀的方法（如 `user.name()` 代替 `user.getName()`）

#### @NonNull

自动生成空值检查逻辑    

编译后自动抛出 `NullPointerException` 如果 name 为 null。

```java
public void setName(@NonNull String name) {
    this.name = name;
}
```