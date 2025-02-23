### 持久化注解

`Jakarta Persistence` 注解是 `Jakarta EE` 规范（以前是 `Java EE`）的一部分，用于 `Java` 应用程序中的对象关系映射（ `Object-Relational Mapping， ORM` ）。这些注解允许将 `Java` 对象映射到关系数据库表，并支持各种持久化操作。

#### `@Entity`

* 包：`jakarta.persistence`

* 用法：将一个类标记为 `JPA` 实体，这意味着它将被映射到数据库中的表。

* 示例：

```java
import jakarta.persistence.Entity;
import jakarta.persistence.Id;

@Entity
public class Customer {
    @Id
    private Long id;
    private String name;

    // Getters and setters
}
```

#### `@Id`

此注解用于指示实体的主键。每个实体都必须有一个 `@Id` 字段。

* 包：`jakarta.persistence`

* 用法：标记实体的主键

* 示例：

```java
@Id
private Long id;
```

#### `@GeneratedValue`

该注解用于定义主键值的生成策略，常见的策略有 `AUTO、IDENTITY、SEQUENCE、TABLE`

* 包：`jakarta.persistence`

* 用法：指定如何生成主键

* 示例：

```java
@Id
@GeneratedValue(strategy = GenerationType.AUTO)
private Long id;
```

**策略解释**

* `AUTO`：持久层提供器（例如 `Hibernate`）将根据底层数据库选择最合适的生成策略，如果没有提供策略，则这是默认策略

* `IDENTITY`：主键由数据库本身生成，通常使用关系数据库中的自动增量字段（例如 `MySQL` 或 `PostgreSQL`），当数据库列的值自动增加时通常使用（例如，`MySQL` 中的 `AUTO_INCREMENT`）

* `SEQUENCE`：使用数据库序列生成主键值，在 `Oracle、PostgreSQL` 和其他一些支持序列的关系数据库中很常见


#### `@Column`

`@Column` 注释用于定义自定义列名或为实体的字段指定附加列属性（如可空、唯一、长度等）

* 包：`jakarta.persistence`

* 用法：指定实体字段的列映射

* 示例：

```java
@Column(name = "customer_name")
private String name;
```

#### `@Table`

`@Table` 注解指定实体应映射到数据库中的表。如果不指定，则默认表名与实体类名相同

* 包：`jakarta.persistence`

* 用法：指定实体映射到的表

* 示例：

```java
@Entity
@Table(name = "customers")
public class Customer {
    @Id
    private Long id;

    @Column(name = "customer_name")
    private String name;
}
```

#### `@OneToOne`

例如，每个客户实例都与一个地址相关联

* 包：`jakarta.persistence`

* 用法：定义两个实体之间的一对一关系

* 示例：

```java
@OneToOne
private Address address;
```

#### `@OneToMany`

此注解指定一个 `Customer` 实例与多个 `Order` 实例相关联

* 包：`jakarta.persistence`

* 用法：定义两个实体之间的一对多关系

* 示例：

```java
@OneToMany(mappedBy = "customer")
private List<Order> orders;
```

#### `@ManyToOne`

此注解指定多个订单实体可以与单个客户关联

* 包：`jakarta.persistence`

* 用法：定义两个实体之间的多对一关系

* 示例：

```java
@ManyToOne
private Customer customer;
```

#### `@ManyToMany`

此注解定义了两个实体之间的多对多关系，使用连接表来存储关联

* 包：`jakarta.persistence`

* 用法：定义两个实体之间的多对多关系

* 示例：

```java
@ManyToMany
@JoinTable(
    name = "customer_order",
    joinColumns = @JoinColumn(name = "customer_id"),
    inverseJoinColumns = @JoinColumn(name = "order_id")
)
private List<Order> orders;
```

#### `@JoinColumn`

`@JoinColumn` 注解指定用于关系中的外键的列

* 包：`jakarta.persistence`

* 用法：指定关系中的外键列

* 示例：

```java
@ManyToOne
@JoinColumn(name = "customer_id")
private Customer customer;
```

#### `@Embeddable`

`@Embeddable` 用于定义可以嵌入到实体中的类。`Address` 类可以嵌入到 `Customer` 实体中

* 包：`jakarta.persistence`

* 用法：将一个类标记为可嵌入，这意味着它可以嵌入到其他实体中

* 示例：

```java
@Embeddable
public class Address {
    private String street;
    private String city;
    private String zipCode;
}
```

#### `@Embedded`

* 包：`jakarta.persistence`

* 用法：将字段标记为嵌入对象，即作为实体一部分的对象

* 示例：

```java
@Embedded
private Address address;
```

#### `@Transient`

`@Transient` 注解将字段标记为非持久性，这意味着它不会保存到数据库中

* 包：`jakarta.persistence`

* 用法：指定字段未映射到数据库列

* 示例：

```java
@Transient
private String temporaryData;
```

#### `@Version`

`@Version` 注解用于通过添加版本字段来处理乐观锁定。这可确保如果另一个事务修改同一实体，则更改不会被覆盖

* 包：`jakarta.persistence`

* 用法：用于通过跟踪版本列来实现乐观锁

* 示例：

```java
@Version
private Long version;
```

#### 联合使用示例

```java
import jakarta.persistence.*;
import java.util.List;

@Entity
@Table(name = "customers")
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "customer_name", nullable = false, length = 100)
    private String name;

    @OneToMany(mappedBy = "customer")
    private List<Order> orders;

    @Embedded
    private Address address;

    // Getters and setters
}
```

### 验证注解

`Jakarta Validation` 是用于验证 `Java bean` 约束的规范。它包括几个内置注释，用于验证输入并确保数据符合指定的规则。这些注释通常用于基于 `Java` 的框架，如 `Jakarta EE`（以前的 `Java EE` ）和 `Spring Boot`。

#### `@NotNull`

确保字段不为空。它不检查空字符串或集合，只检查空值

```java
@NotNull(message = "Name cannot be null")
private String name;
```

#### `@NotEmpty`

确保字段不为 `null` 且不为空。它用于字符串、集合和数组

```java
@NotEmpty(message = "Description cannot be empty")
private String description;
```

#### `@NotBlank`

确保字段不为空且至少包含一个非空白字符。这主要用于字符串

```java
@NotBlank(message = "Username cannot be blank")
private String username;
```


#### `@Size`

确保字段的长度（字符串、数组或集合）介于指定的最小长度和最大长度之间

```java
@Size(min = 5, max = 15, message = "Password must be between 5 and 15 characters")
private String password;
```

#### `@Min 和 @Max`

用于数字字段，以确保值在指定范围内

```java
@Min(18)
@Max(100)
private int age;
```

#### `@Email`

确保该字段是一个有效的电子邮件地址

```java
@Email(message = "Invalid email format")
private String email;
```

#### `@Pattern`

确保该字段与指定的正则表达式 (`regex`) 匹配

```java
@Pattern(regexp = "^[A-Za-z0-9]{5,10}$", message = "Username must be between 5 and 10 alphanumeric characters")
private String username;
```

#### `@Future`

确保日期或时间是将来的

```java
@Future(message = "The event date must be in the future")
private LocalDate eventDate;
```

#### `@Past`

确保日期或时间是过去时间

```java
@Past(message = "The date of birth must be in the past")
private LocalDate dateOfBirth;
```

#### `@Valid`

用于触发嵌套对象的验证。当对象的属性本身是另一个具有约束的对象时，`@Valid` 用于触发该嵌套对象的验证。

```java
public class Person {
    @NotNull
    private String name;

    @Valid
    private Address address;
}

public class Address {
    @NotNull
    private String street;
}
```

#### `@AssertTrue 和 @AssertFalse`

确保注解的布尔字段或方法分别返回 true 或 false

```java
@AssertTrue(message = "The account must be active")
private boolean active;
```

#### `@DecimalMin 和 @DecimalMax`

用于数字字段（特别是 `BigDecimal` ）以确保值在特定范围内

```java
@DecimalMin(value = "10.5", message = "Value must be greater than or equal to 10.5")
private BigDecimal price;
```

#### `@CreditCardNumber`

确保该字段是有效的信用卡号

```java
@CreditCardNumber(message = "Invalid credit card number")
private String creditCardNumber;
```

#### `@URL`

确保该字段包含有效的 URL

```java
@URL(message = "Invalid URL")
private String websiteUrl;
```

#### 联合示例用法

```java
import javax.validation.constraints.*;

public class User {
    @NotNull(message = "Username is required")
    @NotBlank(message = "Username cannot be blank")
    @Size(min = 3, max = 15, message = "Username must be between 3 and 15 characters")
    private String username;

    @Email(message = "Email should be valid")
    private String email;

    @Min(18)
    @Max(100)
    private int age;

    @Valid
    private Address address;  // Nested object validation

    // Getters and setters
}

class Address {
    @NotNull(message = "Street cannot be null")
    private String street;

    @NotNull(message = "City cannot be null")
    private String city;

    // Getters and setters
}
```

#### 在 Spring Boot 中使用 Jakarta 验证

```java
@RestController
public class UserController {

    @PostMapping("/users")
    public ResponseEntity<String> createUser(@Valid @RequestBody User user) {
        // If validation fails, a MethodArgumentNotValidException will be thrown
        return ResponseEntity.ok("User is valid");
    }
}
```

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<String> handleValidationException(MethodArgumentNotValidException ex) {
        String errorMessage = "Validation failed: " + ex.getBindingResult().getAllErrors().toString();
        return ResponseEntity.badRequest().body(errorMessage);
    }
}
```