### 简介

`Spring Data JPA` 是 Spring Data 家族里专门用来简化 `JPA` 开发的模块。

它不是一个新的 ORM 规范。

更准确地说：

```text
JPA 是规范
Hibernate 是常见实现
Spring Data JPA 是 Spring 对 JPA Repository 的封装
```

在 Spring Boot 项目里，常见调用链大致是：

```text
Controller
  |
  v
Service
  |
  v
Repository
  |
  v
Spring Data JPA
  |
  v
Hibernate
  |
  v
JDBC
  |
  v
数据库
```

它的核心目标是：

```text
用 Repository 接口表达数据访问，让常见 CRUD、分页、排序、简单查询少写很多样板代码。
```

一句话概括：

```text
Spring Data JPA 适合用实体对象驱动数据库操作，尤其适合标准 CRUD、分页列表、简单条件查询和领域模型比较清楚的项目。
```

### JPA、Hibernate、Spring Data JPA 的关系

这几个概念经常一起出现。

### JPA

`JPA` 全称是 `Jakarta Persistence API`。

它是一套持久化规范。

常见注解有：

* `@Entity`
* `@Table`
* `@Id`
* `@GeneratedValue`
* `@Column`
* `@OneToMany`
* `@ManyToOne`

常见接口有：

* `EntityManager`
* `Query`
* `TypedQuery`

### Hibernate

`Hibernate` 是 `JPA` 的常见实现。

它负责真正执行对象映射、SQL 生成、脏检查、缓存、关联加载等工作。

### Spring Data JPA

`Spring Data JPA` 在 `JPA` 之上又封装了一层 Repository。

比如定义一个接口：

```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```

就能直接使用：

```java
userRepository.save(user);
userRepository.findById(1L);
userRepository.findAll();
userRepository.deleteById(1L);
```

简单理解：

```text
Hibernate 负责 ORM
Spring Data JPA 负责 Repository 抽象
Spring Boot 负责自动配置
```

### Maven 依赖

Spring Boot 项目里直接引入 starter：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

数据库驱动以 `MySQL` 为例：

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

如果需要写 Web 接口：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

`Spring Boot 3` 以后使用的是 `jakarta.persistence` 包。

也就是实体注解来自：

```java
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
```

不是旧的：

```java
import javax.persistence.Entity;
```

### 数据源和 JPA 配置

`application.yml` 示例：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/jpa_demo?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
```

常见配置说明：

| 配置 | 作用 |
| --- | --- |
| `ddl-auto` | 控制 Hibernate 是否自动处理表结构 |
| `show-sql` | 是否打印 SQL |
| `format_sql` | 是否格式化 SQL |

`ddl-auto` 常见取值：

| 值 | 含义 | 常见场景 |
| --- | --- | --- |
| `none` | 不处理表结构 | 生产环境 |
| `validate` | 校验实体和表结构 | 稳定环境 |
| `update` | 根据实体更新表结构 | 开发环境 |
| `create` | 启动时删除并重建表 | 本地临时测试 |
| `create-drop` | 启动创建，关闭删除 | 测试场景 |

生产环境通常使用：

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: none
```

表结构更适合交给 `Flyway`、`Liquibase` 或数据库变更流程管理。

### 准备演示表

下面用用户表和订单表做示例。

```sql
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS users;

CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL,
  age INT NOT NULL,
  status VARCHAR(20) NOT NULL,
  version INT NOT NULL DEFAULT 0,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NULL,
  UNIQUE KEY uk_users_email (email)
);

CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  order_no VARCHAR(50) NOT NULL,
  amount DECIMAL(10, 2) NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at DATETIME NOT NULL,
  INDEX idx_orders_user_id (user_id)
);

INSERT INTO users (username, email, age, status, version, created_at, updated_at) VALUES
('张三', 'zhangsan@example.com', 20, 'ACTIVE', 0, '2026-01-01 10:00:00', NULL),
('李四', 'lisi@example.com', 25, 'ACTIVE', 0, '2026-01-02 10:00:00', NULL),
('王五', 'wangwu@example.com', 17, 'DISABLED', 0, '2026-01-03 10:00:00', NULL);

INSERT INTO orders (user_id, order_no, amount, status, created_at) VALUES
(1, 'A001', 99.00, 'PAID', '2026-02-01 10:00:00'),
(1, 'A002', 260.00, 'PAID', '2026-02-02 10:00:00'),
(2, 'A003', 35.50, 'CANCELLED', '2026-02-03 10:00:00');
```

### 实体类

用户实体：

```java
package com.example.demo.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.OneToMany;
import jakarta.persistence.Table;
import jakarta.persistence.Version;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 50)
    private String username;

    @Column(nullable = false, unique = true, length = 100)
    private String email;

    @Column(nullable = false)
    private Integer age;

    @Column(nullable = false, length = 20)
    private String status;

    @Version
    private Integer version;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @OneToMany(mappedBy = "user")
    private List<Order> orders = new ArrayList<>();

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

    public Integer getVersion() {
        return version;
    }

    public void setVersion(Integer version) {
        this.version = version;
    }

    public LocalDateTime getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(LocalDateTime createdAt) {
        this.createdAt = createdAt;
    }

    public LocalDateTime getUpdatedAt() {
        return updatedAt;
    }

    public void setUpdatedAt(LocalDateTime updatedAt) {
        this.updatedAt = updatedAt;
    }

    public List<Order> getOrders() {
        return orders;
    }

    public void setOrders(List<Order> orders) {
        this.orders = orders;
    }
}
```

订单实体：

```java
package com.example.demo.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.FetchType;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.JoinColumn;
import jakarta.persistence.ManyToOne;
import jakarta.persistence.Table;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Column(name = "order_no", nullable = false, length = 50)
    private String orderNo;

    @Column(nullable = false)
    private BigDecimal amount;

    @Column(nullable = false, length = 20)
    private String status;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;

    // getter setter
}
```

几个重点：

* `@Entity`：标记 JPA 实体
* `@Table(name = "users")`：指定表名
* `@Id`：主键字段
* `@GeneratedValue(strategy = GenerationType.IDENTITY)`：数据库自增主键
* `@Column`：指定字段名、长度、是否可空等
* `@Version`：乐观锁版本字段
* `@OneToMany`、`@ManyToOne`：实体关联关系

### Repository 接口

```java
package com.example.demo.repository;

import com.example.demo.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;

import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long>,
        JpaSpecificationExecutor<User> {

    Optional<User> findByEmail(String email);
}
```

`JpaRepository<User, Long>` 里的两个泛型含义：

```text
User：实体类型
Long：主键类型
```

继承后可以直接使用：

```java
save
findById
findAll
deleteById
count
existsById
```

`JpaSpecificationExecutor<User>` 用来支持动态条件查询。

### 新增数据

```java
User user = new User();
user.setUsername("赵六");
user.setEmail("zhaoliu@example.com");
user.setAge(28);
user.setStatus("ACTIVE");
user.setCreatedAt(LocalDateTime.now());

User saved = userRepository.save(user);

System.out.println(saved.getId());
```

`save` 可以用于新增，也可以用于更新。

判断标准大致是：

```text
实体没有主键，通常执行 insert
实体有主键，通常执行 update 或 merge
```

实际 SQL 会由 Hibernate 根据实体状态生成。

### 根据 ID 查询

```java
Optional<User> user = userRepository.findById(1L);
```

返回 `Optional<User>`，表示数据可能存在，也可能不存在。

业务层可以这样处理：

```java
User user = userRepository.findById(id)
        .orElseThrow(() -> new IllegalArgumentException("用户不存在"));
```

### 查询全部

```java
List<User> users = userRepository.findAll();
```

业务表数据量较大时，`findAll()` 要谨慎使用。

更常见的是分页查询。

### 删除数据

按 ID 删除：

```java
userRepository.deleteById(1L);
```

按实体删除：

```java
userRepository.delete(user);
```

删除操作通常放在事务里。

### JpaRepository 常用方法

| 方法 | 作用 |
| --- | --- |
| `save(entity)` | 新增或更新 |
| `saveAll(entities)` | 批量保存 |
| `findById(id)` | 按 ID 查询 |
| `findAll()` | 查询全部 |
| `findAllById(ids)` | 按 ID 集合查询 |
| `deleteById(id)` | 按 ID 删除 |
| `delete(entity)` | 按实体删除 |
| `existsById(id)` | 判断 ID 是否存在 |
| `count()` | 统计总数 |
| `findAll(Pageable)` | 分页查询 |
| `findAll(Sort)` | 排序查询 |

### 方法名查询

`Spring Data JPA` 支持按方法名生成查询。

比如：

```java
Optional<User> findByEmail(String email);
```

会根据方法名生成类似查询：

```sql
where email = ?
```

常见写法：

```java
List<User> findByStatus(String status);

List<User> findByAgeGreaterThan(Integer age);

List<User> findByUsernameContaining(String keyword);

List<User> findByStatusAndAgeGreaterThan(String status, Integer age);

List<User> findByStatusOrderByIdDesc(String status);

boolean existsByEmail(String email);

long countByStatus(String status);
```

常见关键词：

| 关键词 | 示例 | 含义 |
| --- | --- | --- |
| `And` | `findByStatusAndAge` | 并且 |
| `Or` | `findByStatusOrAge` | 或者 |
| `Between` | `findByAgeBetween` | 区间 |
| `LessThan` | `findByAgeLessThan` | 小于 |
| `GreaterThan` | `findByAgeGreaterThan` | 大于 |
| `Containing` | `findByUsernameContaining` | 包含，通常是 `%keyword%` |
| `StartingWith` | `findByUsernameStartingWith` | 前缀匹配 |
| `EndingWith` | `findByUsernameEndingWith` | 后缀匹配 |
| `In` | `findByIdIn` | IN 查询 |
| `OrderBy` | `findByStatusOrderByIdDesc` | 排序 |

方法名查询适合简单条件。

如果方法名变得很长，通常应该换成 `@Query` 或 `Specification`。

### 分页和排序

分页使用 `Pageable`。

```java
Pageable pageable = PageRequest.of(
        0,
        10,
        Sort.by(Sort.Direction.DESC, "id")
);

Page<User> page = userRepository.findAll(pageable);
```

注意页码从 `0` 开始。

```text
第 1 页：0
第 2 页：1
第 3 页：2
```

`Page<User>` 常用方法：

```java
List<User> content = page.getContent();
long totalElements = page.getTotalElements();
int totalPages = page.getTotalPages();
int number = page.getNumber();
int size = page.getSize();
boolean hasNext = page.hasNext();
```

如果只需要当前页数据，不需要总数，可以使用 `Slice`。

```java
Slice<User> findByStatus(String status, Pageable pageable);
```

`Page` 会查询总数。

`Slice` 通常只关心是否还有下一页。

### @Query 自定义 JPQL

方法名查询不适合复杂条件时，可以使用 `@Query`。

JPQL 面向实体和属性，不是直接面向表和字段。

```java
@Query("""
        select u
        from User u
        where u.status = :status
          and u.age >= :minAge
        order by u.id desc
        """)
List<User> findActiveUsers(@Param("status") String status,
                           @Param("minAge") Integer minAge);
```

这里的 `User` 是实体类名。

`status`、`age`、`id` 是实体属性名。

不是数据库表名和字段名。

### @Query 原生 SQL

如果需要直接写数据库 SQL，可以设置 `nativeQuery = true`。

```java
@Query(value = """
        select *
        from users
        where status = :status
          and age >= :minAge
        order by id desc
        """, nativeQuery = true)
List<User> findByNativeSql(@Param("status") String status,
                           @Param("minAge") Integer minAge);
```

原生 SQL 适合：

* 数据库特有语法
* 复杂报表
* 性能调优 SQL
* 很难用 JPQL 表达的查询

### 修改查询

`update`、`delete` 这类修改语句需要 `@Modifying`。

```java
@Modifying
@Query("""
        update User u
        set u.status = :status,
            u.updatedAt = :updatedAt
        where u.id = :id
        """)
int updateStatus(@Param("id") Long id,
                 @Param("status") String status,
                 @Param("updatedAt") LocalDateTime updatedAt);
```

调用这类方法时，需要事务。

通常放在 Service 层：

```java
@Transactional
public void disable(Long id) {
    userRepository.updateStatus(id, "DISABLED", LocalDateTime.now());
}
```

### Specification 动态查询

多条件筛选时，`Specification` 很常用。

Repository 需要继承：

```java
public interface UserRepository extends JpaRepository<User, Long>,
        JpaSpecificationExecutor<User> {
}
```

动态条件示例：

```java
public Specification<User> buildSpec(String keyword, String status, Integer minAge) {
    return (root, query, criteriaBuilder) -> {
        List<Predicate> predicates = new ArrayList<>();

        if (keyword != null && !keyword.isBlank()) {
            Predicate usernameLike = criteriaBuilder.like(
                    root.get("username"),
                    "%" + keyword + "%"
            );
            Predicate emailLike = criteriaBuilder.like(
                    root.get("email"),
                    "%" + keyword + "%"
            );
            predicates.add(criteriaBuilder.or(usernameLike, emailLike));
        }

        if (status != null && !status.isBlank()) {
            predicates.add(criteriaBuilder.equal(root.get("status"), status));
        }

        if (minAge != null) {
            predicates.add(criteriaBuilder.greaterThanOrEqualTo(root.get("age"), minAge));
        }

        return criteriaBuilder.and(predicates.toArray(new Predicate[0]));
    };
}
```

调用：

```java
Specification<User> spec = buildSpec(keyword, status, minAge);
Pageable pageable = PageRequest.of(0, 10, Sort.by("id").descending());

Page<User> page = userRepository.findAll(spec, pageable);
```

`Specification` 适合：

```text
查询条件很多
每个条件都可选
列表页筛选项较多
需要复用条件片段
```

### Projection 投影

有些接口只需要返回部分字段。

可以使用接口投影：

```java
public interface UserSummary {
    Long getId();
    String getUsername();
    String getEmail();
}
```

Repository：

```java
List<UserSummary> findByStatus(String status);
```

这样接口返回的不是完整 `User` 实体，而是只包含部分字段的视图。

也可以使用 JPQL 构造 DTO。

DTO：

```java
package com.example.demo.dto;

public class UserSummaryDTO {
    private Long id;
    private String username;
    private String email;

    public UserSummaryDTO(Long id, String username, String email) {
        this.id = id;
        this.username = username;
        this.email = email;
    }

    // getter
}
```

Repository：

```java
@Query("""
        select new com.example.demo.dto.UserSummaryDTO(u.id, u.username, u.email)
        from User u
        where u.status = :status
        """)
List<UserSummaryDTO> findSummaryByStatus(@Param("status") String status);
```

投影适合：

* 列表页
* 下拉选项
* 只读接口
* 不希望暴露完整实体的接口

### 审计字段

常见审计字段：

```text
created_at
updated_at
created_by
updated_by
```

Spring Data JPA 可以自动填充创建时间和更新时间。

实体类先加监听器：

```java
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import jakarta.persistence.EntityListeners;
import jakarta.persistence.MappedSuperclass;
import java.time.LocalDateTime;

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    public LocalDateTime getCreatedAt() {
        return createdAt;
    }

    public LocalDateTime getUpdatedAt() {
        return updatedAt;
    }
}
```

启动类启用审计：

```java
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

@EnableJpaAuditing
@SpringBootApplication
public class JpaDemoApplication {
}
```

实体继承：

```java
public class User extends BaseEntity {
}
```

这样保存和更新时，时间字段会自动处理。

### 乐观锁

JPA 支持乐观锁。

实体字段：

```java
@Version
private Integer version;
```

更新时会带上版本条件。

大致逻辑：

```text
where id = ? and version = ?
```

更新成功后版本号增加。

如果版本不一致，会抛出乐观锁相关异常。

这种机制适合防止并发修改覆盖。

### 关联关系和懒加载

实体关联常见有：

* `@OneToOne`
* `@OneToMany`
* `@ManyToOne`
* `@ManyToMany`

订单和用户的关系通常是：

```text
多个订单属于一个用户。
```

订单实体：

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "user_id", nullable = false)
private User user;
```

`FetchType.LAZY` 表示懒加载。

只有访问 `order.getUser()` 时，才加载用户。

懒加载要注意事务边界。

如果事务已经关闭，再访问懒加载属性，可能出现懒加载异常。

### EntityGraph

`EntityGraph` 可以在查询时指定需要一起加载的关联对象。

Repository：

```java
@EntityGraph(attributePaths = "orders")
Optional<User> findWithOrdersById(Long id);
```

查询用户时同时加载订单。

这类写法常用于处理 `N + 1` 查询问题。

所谓 `N + 1`，大致是：

```text
先查 1 次用户列表
再为每个用户各查 1 次订单
```

如果用户有 100 条，就可能变成 101 次查询。

可以用 `EntityGraph`、`fetch join`、DTO 查询等方式处理。

### Service 实战

```java
package com.example.demo.service;

import com.example.demo.dto.UserSummaryDTO;
import com.example.demo.entity.User;
import com.example.demo.repository.UserRepository;
import jakarta.persistence.EntityNotFoundException;
import jakarta.persistence.criteria.Predicate;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Service
@Transactional(readOnly = true)
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Transactional
    public Long create(User user) {
        user.setStatus("ACTIVE");
        user.setCreatedAt(LocalDateTime.now());

        User saved = userRepository.save(user);

        return saved.getId();
    }

    public User detail(Long id) {
        return userRepository.findById(id)
                .orElseThrow(() -> new EntityNotFoundException("用户不存在"));
    }

    public Page<User> page(String keyword, String status, Integer minAge, int pageNumber, int pageSize) {
        Specification<User> spec = buildSpec(keyword, status, minAge);

        Pageable pageable = PageRequest.of(
                pageNumber - 1,
                pageSize,
                Sort.by(Sort.Direction.DESC, "id")
        );

        return userRepository.findAll(spec, pageable);
    }

    @Transactional
    public void updateEmail(Long id, String email) {
        User user = detail(id);
        user.setEmail(email);
        user.setUpdatedAt(LocalDateTime.now());
    }

    @Transactional
    public void disable(Long id) {
        userRepository.updateStatus(id, "DISABLED", LocalDateTime.now());
    }

    @Transactional
    public void remove(Long id) {
        userRepository.deleteById(id);
    }

    private Specification<User> buildSpec(String keyword, String status, Integer minAge) {
        return (root, query, criteriaBuilder) -> {
            List<Predicate> predicates = new ArrayList<>();

            if (keyword != null && !keyword.isBlank()) {
                Predicate usernameLike = criteriaBuilder.like(root.get("username"), "%" + keyword + "%");
                Predicate emailLike = criteriaBuilder.like(root.get("email"), "%" + keyword + "%");
                predicates.add(criteriaBuilder.or(usernameLike, emailLike));
            }

            if (status != null && !status.isBlank()) {
                predicates.add(criteriaBuilder.equal(root.get("status"), status));
            }

            if (minAge != null) {
                predicates.add(criteriaBuilder.greaterThanOrEqualTo(root.get("age"), minAge));
            }

            return criteriaBuilder.and(predicates.toArray(new Predicate[0]));
        };
    }
}
```

这里有一个 JPA 常见点：

```java
User user = detail(id);
user.setEmail(email);
```

在事务里查出来的实体是托管状态。

修改属性后，不一定需要显式调用 `save`。

事务提交时，Hibernate 会做脏检查并生成更新 SQL。

### Controller 示例

```java
package com.example.demo.controller;

import com.example.demo.entity.User;
import com.example.demo.service.UserService;
import org.springframework.data.domain.Page;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    public Long create(@RequestBody User user) {
        return userService.create(user);
    }

    @GetMapping("/{id}")
    public User detail(@PathVariable Long id) {
        return userService.detail(id);
    }

    @GetMapping
    public Page<User> page(@RequestParam(required = false) String keyword,
                           @RequestParam(required = false) String status,
                           @RequestParam(required = false) Integer minAge,
                           @RequestParam(defaultValue = "1") int pageNumber,
                           @RequestParam(defaultValue = "10") int pageSize) {
        return userService.page(keyword, status, minAge, pageNumber, pageSize);
    }

    @PutMapping("/{id}/email")
    public void updateEmail(@PathVariable Long id, @RequestParam String email) {
        userService.updateEmail(id, email);
    }

    @PutMapping("/{id}/disable")
    public void disable(@PathVariable Long id) {
        userService.disable(id);
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        userService.remove(id);
    }
}
```

实际接口不一定直接返回实体。

更常见的是返回 DTO，避免把关联字段、内部字段、懒加载对象直接暴露出去。

### 事务边界

JPA 非常依赖事务边界。

常见写法：

```java
@Service
@Transactional(readOnly = true)
public class UserService {

    @Transactional
    public Long create(User user) {
        return userRepository.save(user).getId();
    }
}
```

查询方法默认只读事务。

写入方法单独加普通事务。

这样做的好处：

* 查询语义更清楚
* 写操作有事务保护
* 懒加载和脏检查行为更稳定

### Spring Data JPA 和 MyBatis 的区别

| 对比项 | Spring Data JPA | MyBatis |
| --- | --- | --- |
| 思路 | 面向对象和实体关系 | 面向 SQL |
| 常规 CRUD | Repository 自动提供 | 通常手写 SQL |
| 动态查询 | 方法名、Specification、Criteria | XML 动态 SQL |
| 复杂 SQL | JPQL / 原生 SQL | XML SQL 更直接 |
| 关联关系 | 实体注解表达 | SQL join + ResultMap |
| 性能控制 | 关注懒加载、N+1、事务 | 关注 SQL 和执行计划 |
| 适合场景 | 领域模型清楚、CRUD 多 | SQL 复杂、报表多 |

简单理解：

```text
Spring Data JPA 更适合实体关系清楚的业务系统
MyBatis 更适合 SQL 控制要求高的系统
```

### 常见使用建议

### 生产环境谨慎使用 ddl-auto update

`ddl-auto: update` 很适合本地开发。

生产环境里，表结构变更通常需要审批、回滚方案和版本记录。

更常见的是：

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```

或者：

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: none
```

表结构交给数据库迁移工具。

### Repository 方法名不要过长

方法名查询很方便。

比如：

```java
findByStatusAndAgeGreaterThan
```

但如果变成：

```java
findByStatusAndAgeGreaterThanAndEmailContainingAndUsernameStartingWithOrderByCreatedAtDesc
```

可读性会下降。

这时更适合使用：

* `@Query`
* `Specification`
* 自定义 Repository 实现

### 控制实体返回范围

实体类通常包含很多数据库字段和关联关系。

接口层直接返回实体，可能带来：

* 字段暴露过多
* 懒加载触发异常
* JSON 循环引用
* 不必要的关联查询

更常见的方式是返回 DTO 或 Projection。

### 关联查询注意 N + 1

JPA 关联映射很方便，但也容易触发额外查询。

列表页如果要展示关联字段，可以优先考虑：

* `@EntityGraph`
* JPQL `fetch join`
* DTO 查询
* Projection

### 复杂报表优先考虑原生 SQL

Spring Data JPA 很适合常规业务数据访问。

但复杂报表、窗口函数、数据库特有语法、大量聚合统计，原生 SQL 通常更清楚。

可以使用：

```java
@Query(nativeQuery = true)
```

或者把复杂查询放到 MyBatis、JdbcTemplate、jOOQ 等更直接的 SQL 工具里。

### 常用注解汇总

| 注解 | 作用 |
| --- | --- |
| `@Entity` | 标记 JPA 实体 |
| `@Table` | 指定表名 |
| `@Id` | 指定主键 |
| `@GeneratedValue` | 指定主键生成策略 |
| `@Column` | 指定字段映射 |
| `@Version` | 乐观锁版本字段 |
| `@OneToMany` | 一对多关系 |
| `@ManyToOne` | 多对一关系 |
| `@JoinColumn` | 指定外键字段 |
| `@Query` | 自定义 JPQL 或原生 SQL |
| `@Modifying` | 标记更新或删除语句 |
| `@EntityGraph` | 指定查询时加载的关联属性 |
| `@CreatedDate` | 自动填充创建时间 |
| `@LastModifiedDate` | 自动填充更新时间 |
| `@EnableJpaAuditing` | 启用审计 |

### 常用接口和类汇总

| 名称 | 作用 |
| --- | --- |
| `JpaRepository<T, ID>` | 通用 Repository |
| `JpaSpecificationExecutor<T>` | Specification 动态查询 |
| `Pageable` | 分页参数 |
| `PageRequest` | 创建分页参数 |
| `Page<T>` | 分页结果，带总数 |
| `Slice<T>` | 分页切片，不一定查总数 |
| `Sort` | 排序参数 |
| `Specification<T>` | 动态条件 |
| `EntityManager` | JPA 原生入口 |

### 总结

`Spring Data JPA` 的重点是用 Repository 和实体关系来组织数据访问。

常见开发流程是：

```text
建表
写 Entity
写 Repository
写 Service
写 Controller
根据复杂度选择方法名查询、@Query、Specification、Projection
```

适合它的场景：

* 实体关系清楚
* 标准 CRUD 多
* 分页列表多
* 业务更关注对象模型
* SQL 不需要处处手写控制

需要重点关注的地方：

* 事务边界
* 懒加载
* N + 1 查询
* DTO 和 Projection
* `ddl-auto` 的环境差异
* 复杂 SQL 的处理边界

掌握这些内容后，`Spring Data JPA` 已经可以覆盖大多数常规业务系统的数据访问层开发。
