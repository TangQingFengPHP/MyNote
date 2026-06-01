### 简介

`MyBatis-Flex` 是一个基于 `MyBatis` 的增强框架。

它的定位很直接：

```text
保留 MyBatis 写 SQL 的灵活性，同时补上通用 CRUD、链式查询、分页、逻辑删除、乐观锁等常用能力。
```

普通 `MyBatis` 项目里，常见代码结构是：

```text
Entity
Mapper 接口
Mapper XML
Service
Controller
```

如果只是做一张表的增删改查，也要写不少重复 SQL。

`MyBatis-Flex` 的思路是：

```text
简单 CRUD 交给 BaseMapper
复杂条件交给 QueryWrapper
特别复杂的 SQL 仍然可以回到 MyBatis 原生 XML 或注解 SQL
```

一句话概括：

```text
MyBatis-Flex 是一个轻量的 MyBatis 增强工具，适合需要 SQL 控制感，又想减少重复 CRUD 代码的项目。
```

### MyBatis-Flex 解决什么问题

先看普通 `MyBatis` 的单表查询。

Mapper 接口：

```java
public interface UserMapper {
    User selectById(Long id);
}
```

XML：

```xml
<select id="selectById" resultType="com.example.entity.User">
    select id, username, email, age, status, created_at
    from tb_user
    where id = #{id}
</select>
```

新增、修改、删除、分页、条件查询都继续写 SQL。

这种方式很灵活，但简单操作会显得重复。

`MyBatis-Flex` 的写法是：

```java
public interface UserMapper extends BaseMapper<User> {
}
```

然后直接使用：

```java
User user = userMapper.selectOneById(1L);
```

这类基础方法由 `BaseMapper` 提供。

如果需要条件查询：

```java
QueryWrapper query = QueryWrapper.create()
        .where(USER.AGE.ge(18))
        .and(USER.STATUS.eq("ACTIVE"))
        .orderBy(USER.ID.desc());

List<User> users = userMapper.selectListByQuery(query);
```

这就是 `MyBatis-Flex` 的核心体验：

```text
单表 CRUD 少写 SQL
条件查询用链式 API
复杂 SQL 仍然保留 MyBatis 原生能力
```

### 核心概念

学习 `MyBatis-Flex` 时，先抓住几个关键词。

| 名称 | 作用 |
| --- | --- |
| `@Table` | 标记实体类对应的数据库表 |
| `@Id` | 标记主键字段 |
| `@Column` | 配置字段映射、逻辑删除、忽略字段等 |
| `BaseMapper<T>` | 通用 Mapper，提供基础增删改查 |
| `QueryWrapper` | 链式 SQL 构造器 |
| `Page<T>` | 分页结果对象 |
| `Db + Row` | 不依赖实体类的数据库操作方式 |
| APT | 编译期生成表定义类，比如 `USER`、`ACCOUNT` |

其中最常用的是：

```text
@Table + @Id + BaseMapper + QueryWrapper
```

### Maven 依赖

`MyBatis-Flex` 需要根据 `Spring Boot` 版本选择不同 starter。

`Spring Boot 2.x`：

```xml
<dependency>
    <groupId>com.mybatis-flex</groupId>
    <artifactId>mybatis-flex-spring-boot-starter</artifactId>
    <version>1.11.7</version>
</dependency>
```

`Spring Boot 3.x`：

```xml
<dependency>
    <groupId>com.mybatis-flex</groupId>
    <artifactId>mybatis-flex-spring-boot3-starter</artifactId>
    <version>1.11.7</version>
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

APT 处理器建议放到 `annotationProcessorPaths` 里：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.11.0</version>
    <configuration>
        <annotationProcessorPaths>
            <path>
                <groupId>com.mybatis-flex</groupId>
                <artifactId>mybatis-flex-processor</artifactId>
                <version>1.11.7</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

APT 用来生成表定义类。

比如实体类叫 `User`，编译后会生成类似这样的类：

```text
UserTableDef
```

代码里就可以静态导入：

```java
import static com.example.entity.table.UserTableDef.USER;
```

然后写出：

```java
USER.ID.eq(1L)
USER.USERNAME.like("张")
```

### 数据源配置

`application.yml` 示例：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/flex_demo?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver

mybatis-flex:
  print-banner: false
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

开发环境可以打开 SQL 日志，便于观察生成的 SQL。

生产环境一般会交给日志系统统一管理。

### 启动类配置

启动类需要扫描 Mapper。

```java
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@MapperScan("com.example.demo.mapper")
public class MyBatisFlexDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyBatisFlexDemoApplication.class, args);
    }
}
```

也可以在每个 Mapper 接口上加 `@Mapper`。

项目里 Mapper 较多时，`@MapperScan` 更省事。

### 准备演示表

下面用一个用户表做示例。

```sql
DROP TABLE IF EXISTS tb_user;

CREATE TABLE tb_user (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL,
  age INT NOT NULL,
  status VARCHAR(20) NOT NULL,
  deleted TINYINT NOT NULL DEFAULT 0,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NULL
);

INSERT INTO tb_user (username, email, age, status, deleted, created_at, updated_at) VALUES
('张三', 'zhangsan@example.com', 20, 'ACTIVE', 0, '2026-01-01 10:00:00', NULL),
('李四', 'lisi@example.com', 25, 'ACTIVE', 0, '2026-01-02 10:00:00', NULL),
('王五', 'wangwu@example.com', 17, 'DISABLED', 0, '2026-01-03 10:00:00', NULL);
```

### 实体类

```java
package com.example.demo.entity;

import com.mybatisflex.annotation.Column;
import com.mybatisflex.annotation.Id;
import com.mybatisflex.annotation.Table;
import com.mybatisflex.core.keygen.KeyType;

import java.time.LocalDateTime;

@Table("tb_user")
public class User {

    @Id(keyType = KeyType.Auto)
    private Long id;

    private String username;

    private String email;

    private Integer age;

    private String status;

    @Column(isLogicDelete = true)
    private Boolean deleted;

    private LocalDateTime createdAt;

    private LocalDateTime updatedAt;

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

    public Boolean getDeleted() {
        return deleted;
    }

    public void setDeleted(Boolean deleted) {
        this.deleted = deleted;
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
}
```

几个重点：

* `@Table("tb_user")`：实体类对应 `tb_user` 表
* `@Id(keyType = KeyType.Auto)`：主键由数据库自增生成
* `@Column(isLogicDelete = true)`：标记逻辑删除字段
* `created_at` 会自动映射到 `createdAt`

### Mapper 接口

```java
package com.example.demo.mapper;

import com.example.demo.entity.User;
import com.mybatisflex.core.BaseMapper;

public interface UserMapper extends BaseMapper<User> {
}
```

继承 `BaseMapper<User>` 后，常见 CRUD 方法就可以直接使用。

### 新增数据

```java
User user = new User();
user.setUsername("赵六");
user.setEmail("zhaoliu@example.com");
user.setAge(28);
user.setStatus("ACTIVE");
user.setDeleted(false);
user.setCreatedAt(LocalDateTime.now());

userMapper.insert(user);

System.out.println(user.getId());
```

如果主键配置为自增，插入后主键会回填到实体对象。

常见生成 SQL 大致是：

```sql
insert into tb_user (username, email, age, status, deleted, created_at)
values (?, ?, ?, ?, ?, ?)
```

### 根据 ID 查询

```java
User user = userMapper.selectOneById(1L);
```

对应场景：

```text
通过主键查一条数据。
```

如果实体配置了逻辑删除字段，查询会自动带上未删除条件。

### 查询全部

```java
List<User> users = userMapper.selectAll();
```

实际项目中，`selectAll` 更适合数据量很小的字典表、配置表。

业务表数据量较大时，通常使用条件查询或分页查询。

### 修改数据

```java
User user = new User();
user.setId(1L);
user.setEmail("new-zhangsan@example.com");
user.setUpdatedAt(LocalDateTime.now());

userMapper.update(user);
```

这类写法适合根据实体主键更新。

如果只想更新某些字段，实体里只设置需要修改的字段更清楚。

### 删除数据

```java
userMapper.deleteById(1L);
```

如果实体类没有配置逻辑删除，这就是物理删除。

如果实体类配置了：

```java
@Column(isLogicDelete = true)
private Boolean deleted;
```

删除会变成更新逻辑删除字段。

大致 SQL 是：

```sql
update tb_user
set deleted = 1
where id = ?
  and deleted = 0
```

逻辑删除后的数据，普通查询会自动过滤。

### QueryWrapper 是什么

`QueryWrapper` 是 `MyBatis-Flex` 里非常核心的查询构造器。

它用来在 Java 代码里组装 SQL。

普通 SQL：

```sql
select *
from tb_user
where age >= 18
  and status = 'ACTIVE'
order by id desc
```

`QueryWrapper` 写法：

```java
QueryWrapper query = QueryWrapper.create()
        .where(USER.AGE.ge(18))
        .and(USER.STATUS.eq("ACTIVE"))
        .orderBy(USER.ID.desc());

List<User> users = userMapper.selectListByQuery(query);
```

这里的 `USER` 来自 APT 生成的表定义类。

常见静态导入：

```java
import static com.example.demo.entity.table.UserTableDef.USER;
```

### 常见条件写法

等于：

```java
USER.STATUS.eq("ACTIVE")
```

不等于：

```java
USER.STATUS.ne("DISABLED")
```

大于：

```java
USER.AGE.gt(18)
```

大于等于：

```java
USER.AGE.ge(18)
```

小于：

```java
USER.AGE.lt(60)
```

模糊查询：

```java
USER.USERNAME.like("张")
```

范围查询：

```java
USER.AGE.between(18, 30)
```

`IN` 查询：

```java
USER.ID.in(1L, 2L, 3L)
```

排序：

```java
orderBy(USER.ID.desc())
```

### 动态条件查询

业务接口里经常会有可选筛选条件。

比如：

```text
username 可传可不传
status 可传可不传
minAge 可传可不传
```

可以这样写：

```java
public List<User> search(String username, String status, Integer minAge) {
    QueryWrapper query = QueryWrapper.create()
            .select()
            .from(USER);

    if (username != null && !username.isBlank()) {
        query.and(USER.USERNAME.like(username));
    }

    if (status != null && !status.isBlank()) {
        query.and(USER.STATUS.eq(status));
    }

    if (minAge != null) {
        query.and(USER.AGE.ge(minAge));
    }

    query.orderBy(USER.ID.desc());

    return userMapper.selectListByQuery(query);
}
```

这种写法比手动拼 SQL 字符串更稳。

参数仍然会走预编译绑定，不需要把业务值直接拼到 SQL 里。

### 查询单个对象

按唯一字段查询：

```java
public User findByEmail(String email) {
    QueryWrapper query = QueryWrapper.create()
            .where(USER.EMAIL.eq(email));

    return userMapper.selectOneByQuery(query);
}
```

如果数据可能不存在，业务层可以包一层 `Optional`：

```java
public Optional<User> findOptionalByEmail(String email) {
    QueryWrapper query = QueryWrapper.create()
            .where(USER.EMAIL.eq(email));

    User user = userMapper.selectOneByQuery(query);
    return Optional.ofNullable(user);
}
```

### 查询部分字段

有些接口不需要返回整张表的所有字段。

```java
QueryWrapper query = QueryWrapper.create()
        .select(USER.ID, USER.USERNAME, USER.EMAIL)
        .from(USER)
        .where(USER.STATUS.eq("ACTIVE"));

List<User> users = userMapper.selectListByQuery(query);
```

这段 SQL 只会查询指定字段。

字段越少，网络传输和对象映射压力越小。

### 查询单个字段

统计数量：

```java
QueryWrapper query = QueryWrapper.create()
        .where(USER.STATUS.eq("ACTIVE"));

long count = userMapper.selectCountByQuery(query);
```

查询第一列：

```java
QueryWrapper query = QueryWrapper.create()
        .select(USER.EMAIL)
        .from(USER)
        .where(USER.ID.eq(1L));

String email = userMapper.selectObjectByQueryAs(query, String.class);
```

这类写法适合只取一个字段的场景。

### 分页查询

`BaseMapper` 提供了 `paginate` 方法。

```java
import com.mybatisflex.core.paginate.Page;

public Page<User> pageActiveUsers(int pageNumber, int pageSize) {
    QueryWrapper query = QueryWrapper.create()
            .where(USER.STATUS.eq("ACTIVE"))
            .orderBy(USER.ID.desc());

    return userMapper.paginate(pageNumber, pageSize, query);
}
```

返回值是 `Page<User>`。

常用字段：

```java
Page<User> page = pageActiveUsers(1, 10);

List<User> records = page.getRecords();
long totalRow = page.getTotalRow();
long totalPage = page.getTotalPage();
```

分页参数含义：

```text
pageNumber：当前页，从 1 开始
pageSize：每页条数
```

### 第二页以后复用总数

分页查询通常会查两次：

```text
一次查总数
一次查当前页数据
```

如果第一页已经拿到总数，第二页以后可以把总数传进去，减少一次 count 查询。

```java
long totalRow = 120L;

Page<User> page = userMapper.paginate(2, 10, totalRow, query);
```

这个写法适合对性能比较敏感的列表页。

### 多表查询

准备一张订单表：

```sql
DROP TABLE IF EXISTS tb_order;

CREATE TABLE tb_order (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  order_no VARCHAR(50) NOT NULL,
  amount DECIMAL(10, 2) NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at DATETIME NOT NULL
);
```

实体类：

```java
@Table("tb_order")
public class Order {

    @Id(keyType = KeyType.Auto)
    private Long id;

    private Long userId;

    private String orderNo;

    private BigDecimal amount;

    private String status;

    private LocalDateTime createdAt;

    // getter setter
}
```

Mapper：

```java
public interface OrderMapper extends BaseMapper<Order> {
}
```

定义一个 DTO 接收结果：

```java
public class OrderUserDTO {
    private Long orderId;
    private String orderNo;
    private BigDecimal amount;
    private String orderStatus;
    private Long userId;
    private String username;
    private String email;

    // getter setter
}
```

关联查询：

```java
import static com.example.demo.entity.table.OrderTableDef.ORDER;
import static com.example.demo.entity.table.UserTableDef.USER;

public List<OrderUserDTO> findOrderUsers(String orderStatus) {
    QueryWrapper query = QueryWrapper.create()
            .select(
                    ORDER.ID.as("orderId"),
                    ORDER.ORDER_NO.as("orderNo"),
                    ORDER.AMOUNT,
                    ORDER.STATUS.as("orderStatus"),
                    USER.ID.as("userId"),
                    USER.USERNAME.as("username"),
                    USER.EMAIL.as("email")
            )
            .from(ORDER)
            .leftJoin(USER).on(ORDER.USER_ID.eq(USER.ID))
            .where(ORDER.STATUS.eq(orderStatus))
            .orderBy(ORDER.ID.desc());

    return orderMapper.selectListByQueryAs(query, OrderUserDTO.class);
}
```

这种写法适合：

```text
SQL 不算特别复杂，但又不是简单单表查询。
```

如果 SQL 已经非常复杂，写 XML 也很正常。

`MyBatis-Flex` 不限制继续使用 MyBatis 原生 XML。

### 批量插入

```java
List<User> users = new ArrayList<>();

User user1 = new User();
user1.setUsername("用户1");
user1.setEmail("user1@example.com");
user1.setAge(18);
user1.setStatus("ACTIVE");
user1.setDeleted(false);
user1.setCreatedAt(LocalDateTime.now());
users.add(user1);

User user2 = new User();
user2.setUsername("用户2");
user2.setEmail("user2@example.com");
user2.setAge(19);
user2.setStatus("ACTIVE");
user2.setDeleted(false);
user2.setCreatedAt(LocalDateTime.now());
users.add(user2);

userMapper.insertBatch(users);
```

批量操作适合一次写入多条数据。

数据量很大时，可以按固定大小拆批。

```text
例如每 500 条或 1000 条执行一次。
```

### 条件删除

删除禁用状态用户：

```java
QueryWrapper query = QueryWrapper.create()
        .where(USER.STATUS.eq("DISABLED"));

userMapper.deleteByQuery(query);
```

如果配置了逻辑删除，这里会执行逻辑删除。

如果没有配置逻辑删除，就会执行物理删除。

### 条件更新

一些更新不方便先查出来再改，可以直接按条件更新。

```java
User updateEntity = new User();
updateEntity.setStatus("DISABLED");
updateEntity.setUpdatedAt(LocalDateTime.now());

QueryWrapper query = QueryWrapper.create()
        .where(USER.AGE.lt(18));

userMapper.updateByQuery(updateEntity, query);
```

含义是：

```text
把年龄小于 18 的用户状态改成 DISABLED。
```

### Active Record

`MyBatis-Flex` 也支持 `Active Record` 风格。

实体类继承 `Model` 后，可以直接调用保存、更新等方法。

```java
import com.mybatisflex.core.activerecord.Model;

@Table("tb_user")
public class User extends Model<User> {
    @Id(keyType = KeyType.Auto)
    private Long id;

    private String username;

    // getter setter
}
```

使用：

```java
User user = new User();
user.setUsername("张三");
user.save();
```

这种写法很短。

但在分层清晰的业务系统里，更常见的是：

```text
Controller -> Service -> Mapper
```

`Active Record` 更适合简单模块、脚本工具、管理后台里的少量操作。

### Db + Row

`Db + Row` 适合不想定义实体类的场景。

比如临时查询、后台工具、通用管理功能。

```java
import com.mybatisflex.core.row.Db;
import com.mybatisflex.core.row.Row;

Row row = Db.selectOneById("tb_user", "id", 1L);

String username = row.getString("username");
Integer age = row.getInt("age");
```

查询列表：

```java
List<Row> rows = Db.selectListBySql(
        "select id, username, email from tb_user where status = ?",
        "ACTIVE"
);
```

`Db + Row` 很灵活，但类型约束弱。

核心业务代码里，实体类和 Mapper 更容易维护。

### Service 层封装

下面是一个比较完整的用户服务示例。

```java
package com.example.demo.service;

import com.example.demo.entity.User;
import com.example.demo.mapper.UserMapper;
import com.mybatisflex.core.paginate.Page;
import com.mybatisflex.core.query.QueryWrapper;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

import static com.example.demo.entity.table.UserTableDef.USER;

@Service
public class UserService {

    private final UserMapper userMapper;

    public UserService(UserMapper userMapper) {
        this.userMapper = userMapper;
    }

    @Transactional
    public Long create(User user) {
        user.setStatus("ACTIVE");
        user.setDeleted(false);
        user.setCreatedAt(LocalDateTime.now());

        userMapper.insert(user);

        return user.getId();
    }

    public Optional<User> findById(Long id) {
        return Optional.ofNullable(userMapper.selectOneById(id));
    }

    public List<User> search(String username, String status, Integer minAge) {
        QueryWrapper query = QueryWrapper.create()
                .select()
                .from(USER);

        if (username != null && !username.isBlank()) {
            query.and(USER.USERNAME.like(username));
        }

        if (status != null && !status.isBlank()) {
            query.and(USER.STATUS.eq(status));
        }

        if (minAge != null) {
            query.and(USER.AGE.ge(minAge));
        }

        query.orderBy(USER.ID.desc());

        return userMapper.selectListByQuery(query);
    }

    public Page<User> page(String status, int pageNumber, int pageSize) {
        QueryWrapper query = QueryWrapper.create()
                .where(USER.STATUS.eq(status))
                .orderBy(USER.ID.desc());

        return userMapper.paginate(pageNumber, pageSize, query);
    }

    @Transactional
    public void updateEmail(Long id, String email) {
        User user = new User();
        user.setId(id);
        user.setEmail(email);
        user.setUpdatedAt(LocalDateTime.now());

        userMapper.update(user);
    }

    @Transactional
    public void disable(Long id) {
        User user = new User();
        user.setId(id);
        user.setStatus("DISABLED");
        user.setUpdatedAt(LocalDateTime.now());

        userMapper.update(user);
    }

    @Transactional
    public void remove(Long id) {
        userMapper.deleteById(id);
    }
}
```

这里把事务放在 `Service` 层。

Mapper 只负责数据库操作。

Service 负责业务规则和事务边界。

### Controller 示例

```java
package com.example.demo.controller;

import com.example.demo.entity.User;
import com.example.demo.service.UserService;
import com.mybatisflex.core.paginate.Page;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

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
        return userService.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("用户不存在"));
    }

    @GetMapping("/search")
    public List<User> search(@RequestParam(required = false) String username,
                             @RequestParam(required = false) String status,
                             @RequestParam(required = false) Integer minAge) {
        return userService.search(username, status, minAge);
    }

    @GetMapping
    public Page<User> page(@RequestParam(defaultValue = "ACTIVE") String status,
                           @RequestParam(defaultValue = "1") int pageNumber,
                           @RequestParam(defaultValue = "10") int pageSize) {
        return userService.page(status, pageNumber, pageSize);
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

这组接口覆盖了：

* 新增用户
* 按 ID 查询
* 多条件查询
* 分页查询
* 修改邮箱
* 禁用用户
* 删除用户

### APT 生成类找不到怎么办

`QueryWrapper` 里常见的：

```java
USER.ID.eq(1L)
```

依赖 APT 生成的表定义类。

如果 IDE 里找不到 `UserTableDef` 或 `USER`，一般检查这些地方：

* `mybatis-flex-processor` 是否配置到 `annotationProcessorPaths`
* Maven 是否执行过 `compile` 或 `package`
* IDE 是否开启 Annotation Processing
* `target/generated-sources/annotations` 是否被识别为 generated sources
* 静态导入路径是否正确

默认情况下，生成类通常在：

```text
target/generated-sources/annotations
```

实体类包名如果是：

```text
com.example.demo.entity
```

生成类包名通常类似：

```text
com.example.demo.entity.table
```

### 自定义 APT 配置

可以在项目根目录创建：

```text
mybatis-flex.config
```

示例：

```properties
processor.tableDef.propertiesNameStyle=upperCase
processor.tableDef.package=${entityPackage}.table
processor.mapper.generateEnable=false
```

常用配置含义：

| 配置 | 作用 |
| --- | --- |
| `processor.genPath` | 指定生成代码目录 |
| `processor.tableDef.package` | 指定表定义类包名 |
| `processor.tableDef.propertiesNameStyle` | 指定字段风格 |
| `processor.mapper.generateEnable` | 是否生成 Mapper |

默认字段风格通常是大写下划线：

```java
USER.ID
USER.USERNAME
USER.CREATED_AT
```

如果项目希望使用驼峰字段，也可以调整配置。

### 逻辑删除

逻辑删除是把删除动作变成更新状态。

实体字段：

```java
@Column(isLogicDelete = true)
private Boolean deleted;
```

执行：

```java
userMapper.deleteById(1L);
```

实际效果类似：

```sql
update tb_user
set deleted = 1
where id = ?
  and deleted = 0
```

查询时会自动过滤已删除数据。

也就是说：

```java
userMapper.selectOneById(1L);
```

会带上：

```sql
deleted = 0
```

逻辑删除适合用户、订单、文章这类需要保留历史记录的数据。

对于日志、临时数据、中间表，是否需要逻辑删除要看业务要求。

### 乐观锁

乐观锁适合防止并发修改覆盖。

表里增加版本字段：

```sql
ALTER TABLE tb_user ADD COLUMN version INT NOT NULL DEFAULT 0;
```

实体字段：

```java
@Column(version = true)
private Integer version;
```

更新时会根据版本号判断数据是否被其他事务改过。

大致逻辑是：

```text
更新条件里带上旧 version
更新成功后 version 增加
如果影响行数为 0，说明版本不匹配
```

这种能力常用于库存、余额、配置修改等场景。

### 数据填充

新增和修改时，经常要处理这些字段：

```text
created_at
updated_at
created_by
updated_by
```

可以在 `Service` 层手动设置：

```java
user.setCreatedAt(LocalDateTime.now());
user.setUpdatedAt(LocalDateTime.now());
```

也可以使用 `MyBatis-Flex` 的数据填充能力统一处理。

项目较小时，手动设置足够清楚。

项目字段规范较统一时，统一填充更省维护成本。

### 自定义 SQL

`MyBatis-Flex` 不影响 MyBatis 原生写法。

Mapper 接口里仍然可以写自定义方法：

```java
public interface UserMapper extends BaseMapper<User> {
    List<User> selectActiveUsersByKeyword(String keyword);
}
```

XML：

```xml
<select id="selectActiveUsersByKeyword" resultType="com.example.demo.entity.User">
    select id, username, email, age, status, deleted, created_at, updated_at
    from tb_user
    where deleted = 0
      and status = 'ACTIVE'
      and (
        username like concat('%', #{keyword}, '%')
        or email like concat('%', #{keyword}, '%')
      )
    order by id desc
</select>
```

所以不是所有 SQL 都要改成 `QueryWrapper`。

简单条件用 `QueryWrapper` 很舒服。

复杂报表、复杂统计、多层子查询，用 XML 仍然很合适。

### 和 JdbcTemplate、MyBatis、MyBatis-Plus 的区别

| 对比项 | JdbcTemplate | MyBatis | MyBatis-Plus | MyBatis-Flex |
| --- | --- | --- | --- | --- |
| SQL 控制 | 很直接 | 很直接 | 较直接 | 很直接 |
| 单表 CRUD | 手写 | 手写 | 内置 | 内置 |
| 链式查询 | 无 | 无 | 有 | 有 |
| 多表查询 | 手写 SQL | 手写 SQL | 通常手写 | `QueryWrapper` 支持较灵活 |
| XML 支持 | 无 | 支持 | 支持 | 支持 |
| 学习成本 | 低 | 中等 | 中等 | 中等 |
| 适合场景 | 少量 SQL、内部工具 | SQL 控制要求高 | 常规后台 CRUD | 需要轻量增强和灵活查询 |

粗略理解：

```text
JdbcTemplate 更接近原生 JDBC
MyBatis 更像 SQL 映射工具
MyBatis-Plus 更偏常规 CRUD 增强
MyBatis-Flex 更偏轻量增强 + 灵活 QueryWrapper
```

### 常见使用建议

### Mapper 只放数据库操作

Mapper 层适合放：

* 基础 CRUD
* 自定义查询方法
* 和数据库强相关的操作

业务判断、事务流程、多个 Mapper 的组合，更适合放在 Service 层。

### QueryWrapper 适合中等复杂度查询

适合写成 `QueryWrapper` 的查询：

* 单表条件查询
* 简单多条件筛选
* 简单 join
* 分页列表
* 动态条件查询

更适合写 XML 的查询：

* 大量聚合统计
* 多层嵌套子查询
* 复杂报表
* 数据库特有语法较多的 SQL

### 表定义类生成失败先查 APT

如果 `USER`、`ORDER` 这类表定义对象不存在，优先检查 APT。

这类问题通常不是 Mapper 或 SQL 问题，而是编译期生成代码没有生效。

### 逻辑删除字段保持统一

逻辑删除字段最好在项目里统一命名。

比如统一使用：

```text
deleted
```

或者：

```text
is_delete
```

字段名统一后，实体配置、全局配置、SQL 阅读都会更清楚。

### 批量操作注意分批

`insertBatch` 很方便，但大批量数据仍然需要分批。

如果一次性插入几十万条，容易带来 SQL 太长、事务太大、锁持有时间过长等问题。

常见做法是：

```text
每 500 条或 1000 条拆成一批。
```

### 保留 MyBatis 原生能力

`MyBatis-Flex` 是增强，不是替换所有写法。

项目里可以同时存在：

* `BaseMapper` 通用 CRUD
* `QueryWrapper` 条件查询
* XML 自定义 SQL
* 注解 SQL

按场景选择即可。

### 常用方法汇总

| 方法 | 作用 | 常见场景 |
| --- | --- | --- |
| `insert(entity)` | 新增数据 | 创建用户、创建订单 |
| `insertBatch(list)` | 批量新增 | 批量导入 |
| `update(entity)` | 根据实体主键更新 | 修改单条记录 |
| `updateByQuery(entity, query)` | 按条件更新 | 批量修改状态 |
| `deleteById(id)` | 按主键删除 | 删除单条记录 |
| `deleteByQuery(query)` | 按条件删除 | 批量删除或逻辑删除 |
| `selectOneById(id)` | 按主键查询 | 详情页 |
| `selectOneByQuery(query)` | 按条件查询一条 | 唯一字段查询 |
| `selectListByQuery(query)` | 按条件查询列表 | 列表页、筛选 |
| `selectAll()` | 查询全部 | 小字典表 |
| `selectCountByQuery(query)` | 查询数量 | 统计条数 |
| `paginate(pageNumber, pageSize, query)` | 分页查询 | 后台列表 |
| `selectListByQueryAs(query, DTO.class)` | 查询并映射 DTO | 多表关联查询 |

### 总结

`MyBatis-Flex` 的核心并不复杂。

最常用的组合就是：

```text
Entity 使用 @Table 和 @Id
Mapper 继承 BaseMapper
查询条件使用 QueryWrapper
分页使用 paginate
复杂 SQL 继续使用 MyBatis XML
```

它适合这些场景：

* 项目已经熟悉 MyBatis
* 希望保留 SQL 控制权
* 单表 CRUD 比较多
* 动态条件查询比较多
* 不想为简单查询写大量 XML
* 需要逻辑删除、分页、乐观锁等常用能力

落地时重点关注三件事：

* 依赖版本和 Spring Boot 版本要匹配
* APT 要正常生成表定义类
* `QueryWrapper` 和 XML 按复杂度分工

掌握这些内容后，`MyBatis-Flex` 已经可以覆盖大多数后台系统的数据访问层开发。
