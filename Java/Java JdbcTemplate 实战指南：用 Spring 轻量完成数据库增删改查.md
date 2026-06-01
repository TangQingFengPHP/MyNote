### 简介

`JdbcTemplate` 是 `Spring JDBC` 模块提供的数据库访问工具类。

它解决的是原生 `JDBC` 里很常见的一类重复工作：

```text
获取连接
创建 Statement
绑定 SQL 参数
遍历 ResultSet
关闭连接和结果集
处理 SQLException
```

原生 `JDBC` 能做的事情，`JdbcTemplate` 基本都能做。

区别在于，`JdbcTemplate` 把连接管理、资源释放、异常转换这些固定流程封装好了，业务代码只需要关注：

```text
执行什么 SQL
传什么参数
结果怎么映射
```

一句话概括：

```text
JdbcTemplate 是 Spring 提供的轻量级数据库操作工具，适合直接写 SQL，又不想写一堆 JDBC 模板代码的场景。
```

### 原生 JDBC 写法

先看一段原生 `JDBC` 查询代码：

```java
String sql = "select id, username, email, age from users where id = ?";

Connection connection = null;
PreparedStatement statement = null;
ResultSet resultSet = null;

try {
    connection = dataSource.getConnection();
    statement = connection.prepareStatement(sql);
    statement.setLong(1, 1L);
    resultSet = statement.executeQuery();

    if (resultSet.next()) {
        User user = new User();
        user.setId(resultSet.getLong("id"));
        user.setUsername(resultSet.getString("username"));
        user.setEmail(resultSet.getString("email"));
        user.setAge(resultSet.getInt("age"));
        return user;
    }

    return null;
} finally {
    if (resultSet != null) {
        resultSet.close();
    }
    if (statement != null) {
        statement.close();
    }
    if (connection != null) {
        connection.close();
    }
}
```

真正和业务有关的只有几件事：

* SQL 是什么
* 参数是什么
* 每一列怎么放进 `User`

其他部分基本都是固定流程。

### JdbcTemplate 写法

换成 `JdbcTemplate` 后：

```java
String sql = "select id, username, email, age from users where id = ?";

User user = jdbcTemplate.queryForObject(sql, (rs, rowNum) -> {
    User item = new User();
    item.setId(rs.getLong("id"));
    item.setUsername(rs.getString("username"));
    item.setEmail(rs.getString("email"));
    item.setAge(rs.getInt("age"));
    return item;
}, 1L);
```

代码明显少很多。

连接什么时候拿、什么时候关，`ResultSet` 怎么关闭，异常怎么转换，都交给 `JdbcTemplate` 处理。

### JdbcTemplate 做了什么

`JdbcTemplate` 大致处在这个位置：

```text
业务代码
  |
  v
JdbcTemplate
  |
  v
DataSource
  |
  v
数据库
```

它本身不负责连接池。

连接来自 `DataSource`。

在 `Spring Boot` 项目里，只要配置了数据源，并引入 `spring-boot-starter-jdbc`，通常可以直接注入：

```java
@Repository
public class UserRepository {

    private final JdbcTemplate jdbcTemplate;

    public UserRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
}
```

常见能力有这些：

* `update`：执行新增、修改、删除
* `queryForObject`：查询一条记录或一个值
* `query`：查询列表
* `batchUpdate`：批量执行
* `execute`：执行 DDL 或普通 SQL
* `RowMapper`：把一行结果映射成一个对象
* `NamedParameterJdbcTemplate`：使用命名参数写 SQL

### Maven 依赖

`Spring Boot` 项目常用依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>

<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

如果是本地演示，也可以使用 `H2` 数据库：

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

### 数据源配置

以 `MySQL` 为例：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo_db?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
```

`Spring Boot` 会根据这些配置创建 `DataSource`。

同时，`JdbcTemplate` 也会自动注册成 `Bean`。

### 准备演示表

下面的示例围绕用户表展开。

```sql
DROP TABLE IF EXISTS users;

CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL,
  age INT NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at DATETIME NOT NULL
);

INSERT INTO users (username, email, age, status, created_at) VALUES
('张三', 'zhangsan@example.com', 20, 'ACTIVE', '2026-01-01 10:00:00'),
('李四', 'lisi@example.com', 25, 'ACTIVE', '2026-01-02 10:00:00'),
('王五', 'wangwu@example.com', 17, 'DISABLED', '2026-01-03 10:00:00');
```

### 实体类

```java
import java.time.LocalDateTime;

public class User {
    private Long id;
    private String username;
    private String email;
    private Integer age;
    private String status;
    private LocalDateTime createdAt;

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

    public LocalDateTime getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(LocalDateTime createdAt) {
        this.createdAt = createdAt;
    }
}
```

### RowMapper：结果集映射

`JdbcTemplate` 查询对象时，需要告诉它：

```text
ResultSet 的一行数据怎么变成 Java 对象。
```

这件事由 `RowMapper` 完成。

```java
import org.springframework.jdbc.core.RowMapper;

import java.sql.Timestamp;

private static final RowMapper<User> USER_ROW_MAPPER = (rs, rowNum) -> {
    User user = new User();
    user.setId(rs.getLong("id"));
    user.setUsername(rs.getString("username"));
    user.setEmail(rs.getString("email"));
    user.setAge(rs.getInt("age"));
    user.setStatus(rs.getString("status"));

    Timestamp createdAt = rs.getTimestamp("created_at");
    if (createdAt != null) {
        user.setCreatedAt(createdAt.toLocalDateTime());
    }

    return user;
};
```

`rowNum` 表示当前行号，从 `0` 开始。

### update：新增数据

`update` 可以执行 `insert`、`update`、`delete`。

新增用户：

```java
public int save(User user) {
    String sql = """
            insert into users (username, email, age, status, created_at)
            values (?, ?, ?, ?, ?)
            """;

    return jdbcTemplate.update(
            sql,
            user.getUsername(),
            user.getEmail(),
            user.getAge(),
            user.getStatus(),
            user.getCreatedAt()
    );
}
```

返回值是受影响行数。

```text
返回 1，表示插入成功 1 行。
```

如果项目还在使用低版本 Java，不支持文本块，可以写成普通字符串：

```java
String sql = "insert into users (username, email, age, status, created_at) values (?, ?, ?, ?, ?)";
```

### update：修改数据

```java
public int updateEmail(Long id, String email) {
    String sql = "update users set email = ? where id = ?";
    return jdbcTemplate.update(sql, email, id);
}
```

### update：删除数据

```java
public int deleteById(Long id) {
    String sql = "delete from users where id = ?";
    return jdbcTemplate.update(sql, id);
}
```

### queryForObject：查询单个值

统计用户数量：

```java
public long countAll() {
    String sql = "select count(*) from users";
    Long count = jdbcTemplate.queryForObject(sql, Long.class);
    return count == null ? 0L : count;
}
```

查询某个字段：

```java
public String findEmailById(Long id) {
    String sql = "select email from users where id = ?";
    return jdbcTemplate.queryForObject(sql, String.class, id);
}
```

`queryForObject` 查询单个值时，第二个参数可以写目标类型：

```java
Long.class
Integer.class
String.class
BigDecimal.class
```

### queryForObject：查询单个对象

```java
public User findById(Long id) {
    String sql = """
            select id, username, email, age, status, created_at
            from users
            where id = ?
            """;

    return jdbcTemplate.queryForObject(sql, USER_ROW_MAPPER, id);
}
```

这段代码适合“数据必须存在”的场景。

如果查不到数据，`queryForObject` 会抛出：

```text
EmptyResultDataAccessException
```

如果查出了多条数据，会抛出：

```text
IncorrectResultSizeDataAccessException
```

所以 `queryForObject` 的语义很明确：

```text
期望结果必须是一条。
```

### 查询可能不存在的数据

如果查询结果可能不存在，可以用 `query` 返回列表，再取第一条。

```java
import java.util.List;
import java.util.Optional;

public Optional<User> findOptionalById(Long id) {
    String sql = """
            select id, username, email, age, status, created_at
            from users
            where id = ?
            """;

    List<User> users = jdbcTemplate.query(sql, USER_ROW_MAPPER, id);

    if (users.isEmpty()) {
        return Optional.empty();
    }

    return Optional.of(users.get(0));
}
```

这种写法不会把“查不到”当成异常。

业务层可以这样处理：

```java
User user = userRepository.findOptionalById(id)
        .orElseThrow(() -> new IllegalArgumentException("用户不存在"));
```

### query：查询列表

查询全部用户：

```java
public List<User> findAll() {
    String sql = """
            select id, username, email, age, status, created_at
            from users
            order by id desc
            """;

    return jdbcTemplate.query(sql, USER_ROW_MAPPER);
}
```

按状态查询：

```java
public List<User> findByStatus(String status) {
    String sql = """
            select id, username, email, age, status, created_at
            from users
            where status = ?
            order by id desc
            """;

    return jdbcTemplate.query(sql, USER_ROW_MAPPER, status);
}
```

分页查询：

```java
public List<User> findPage(int page, int size) {
    int offset = (page - 1) * size;

    String sql = """
            select id, username, email, age, status, created_at
            from users
            order by id desc
            limit ? offset ?
            """;

    return jdbcTemplate.query(sql, USER_ROW_MAPPER, size, offset);
}
```

### BeanPropertyRowMapper：自动映射

手写 `RowMapper` 最清楚，也最可控。

如果表字段和实体属性比较规整，可以使用 `BeanPropertyRowMapper`。

```java
import org.springframework.jdbc.core.BeanPropertyRowMapper;

public List<User> findAllByBeanMapper() {
    String sql = "select id, username, email, age, status, created_at from users";

    return jdbcTemplate.query(
            sql,
            new BeanPropertyRowMapper<>(User.class)
    );
}
```

它可以处理常见的下划线转驼峰：

```text
created_at -> createdAt
```

不过它依赖字段名和属性名的匹配。

如果 SQL 字段比较复杂，或者结果来自多表关联，手写 `RowMapper` 通常更直观。

### BeanPropertyRowMapper 配合别名

如果字段名和属性名不一致，可以在 SQL 里使用别名。

```java
String sql = """
        select
          id,
          username as username,
          email as email,
          created_at as created_at
        from users
        """;
```

对于复杂查询，别名写清楚，可以减少映射问题。

### 返回自增主键

新增数据后，如果需要拿到数据库生成的主键，可以使用 `KeyHolder`。

```java
import org.springframework.jdbc.support.GeneratedKeyHolder;
import org.springframework.jdbc.support.KeyHolder;

import java.sql.PreparedStatement;
import java.sql.Statement;

public Long saveAndReturnId(User user) {
    String sql = """
            insert into users (username, email, age, status, created_at)
            values (?, ?, ?, ?, ?)
            """;

    KeyHolder keyHolder = new GeneratedKeyHolder();

    jdbcTemplate.update(connection -> {
        PreparedStatement ps = connection.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS);
        ps.setString(1, user.getUsername());
        ps.setString(2, user.getEmail());
        ps.setInt(3, user.getAge());
        ps.setString(4, user.getStatus());
        ps.setObject(5, user.getCreatedAt());
        return ps;
    }, keyHolder);

    Number key = keyHolder.getKey();
    return key == null ? null : key.longValue();
}
```

这种写法常用于：

```text
新增主表后，继续插入子表。
```

### batchUpdate：批量写入

批量插入可以减少多次数据库往返。

```java
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.util.List;

import org.springframework.jdbc.core.BatchPreparedStatementSetter;

public int[] batchSave(List<User> users) {
    String sql = """
            insert into users (username, email, age, status, created_at)
            values (?, ?, ?, ?, ?)
            """;

    return jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
        @Override
        public void setValues(PreparedStatement ps, int i) throws SQLException {
            User user = users.get(i);
            ps.setString(1, user.getUsername());
            ps.setString(2, user.getEmail());
            ps.setInt(3, user.getAge());
            ps.setString(4, user.getStatus());
            ps.setObject(5, user.getCreatedAt());
        }

        @Override
        public int getBatchSize() {
            return users.size();
        }
    });
}
```

返回值是每条 SQL 影响的行数。

还有一种更短的写法：

```java
public int[] batchUpdateStatus(List<Long> ids, String status) {
    String sql = "update users set status = ? where id = ?";

    List<Object[]> args = ids.stream()
            .map(id -> new Object[]{status, id})
            .toList();

    return jdbcTemplate.batchUpdate(sql, args);
}
```

如果项目使用 `Java 8`，可以改成：

```java
List<Object[]> args = ids.stream()
        .map(id -> new Object[]{status, id})
        .collect(Collectors.toList());
```

### execute：执行普通 SQL

`execute` 常用于执行不返回结果集的 SQL。

```java
jdbcTemplate.execute("""
        create table if not exists operation_logs (
          id bigint primary key auto_increment,
          content varchar(200) not null
        )
        """);
```

实际业务里，建表更常交给迁移工具，比如 `Flyway`、`Liquibase`。

`execute` 更适合临时初始化、测试场景或少量管理类 SQL。

### NamedParameterJdbcTemplate

`JdbcTemplate` 默认使用 `?` 占位符。

参数少时很清楚：

```java
String sql = "select * from users where status = ? and age >= ?";
```

参数一多，可读性就会下降。

`NamedParameterJdbcTemplate` 支持命名参数：

```java
String sql = """
        select id, username, email, age, status, created_at
        from users
        where status = :status
          and age >= :minAge
        order by id desc
        """;
```

完整示例：

```java
import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;

public class UserQueryRepository {

    private final NamedParameterJdbcTemplate namedJdbcTemplate;

    public UserQueryRepository(NamedParameterJdbcTemplate namedJdbcTemplate) {
        this.namedJdbcTemplate = namedJdbcTemplate;
    }

    public List<User> findByCondition(String status, int minAge) {
        String sql = """
                select id, username, email, age, status, created_at
                from users
                where status = :status
                  and age >= :minAge
                order by id desc
                """;

        MapSqlParameterSource params = new MapSqlParameterSource()
                .addValue("status", status)
                .addValue("minAge", minAge);

        return namedJdbcTemplate.query(sql, params, USER_ROW_MAPPER);
    }
}
```

### IN 查询

`NamedParameterJdbcTemplate` 处理 `IN` 条件很方便。

```java
public List<User> findByIds(List<Long> ids) {
    String sql = """
            select id, username, email, age, status, created_at
            from users
            where id in (:ids)
            """;

    MapSqlParameterSource params = new MapSqlParameterSource()
            .addValue("ids", ids);

    return namedJdbcTemplate.query(sql, params, USER_ROW_MAPPER);
}
```

如果使用普通 `JdbcTemplate`，`IN` 参数需要自己拼出对应数量的 `?`，代码会更啰嗦。

### 事务控制

`JdbcTemplate` 可以直接配合 `@Transactional` 使用。

常见写法是：

```text
Repository 负责 SQL
Service 负责事务和业务流程
```

示例：

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Transactional
    public Long register(User user) {
        Long userId = userRepository.saveAndReturnId(user);

        userRepository.insertRegisterLog(userId, "用户注册成功");

        return userId;
    }
}
```

如果 `insertRegisterLog` 抛出运行时异常，前面的用户新增也会回滚。

### 完整实战 Demo：用户 Repository

下面是一份较完整的 `Repository` 示例，包含新增、返回主键、查询、分页、批量更新、删除。

```java
package com.example.demo.repository;

import com.example.demo.entity.User;
import org.springframework.dao.EmptyResultDataAccessException;
import org.springframework.jdbc.core.BatchPreparedStatementSetter;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.jdbc.support.GeneratedKeyHolder;
import org.springframework.jdbc.support.KeyHolder;
import org.springframework.stereotype.Repository;

import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.sql.Statement;
import java.sql.Timestamp;
import java.util.List;
import java.util.Optional;

@Repository
public class UserRepository {

    private final JdbcTemplate jdbcTemplate;

    public UserRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    private static final RowMapper<User> USER_ROW_MAPPER = (rs, rowNum) -> {
        User user = new User();
        user.setId(rs.getLong("id"));
        user.setUsername(rs.getString("username"));
        user.setEmail(rs.getString("email"));
        user.setAge(rs.getInt("age"));
        user.setStatus(rs.getString("status"));

        Timestamp createdAt = rs.getTimestamp("created_at");
        if (createdAt != null) {
            user.setCreatedAt(createdAt.toLocalDateTime());
        }

        return user;
    };

    public Long saveAndReturnId(User user) {
        String sql = """
                insert into users (username, email, age, status, created_at)
                values (?, ?, ?, ?, ?)
                """;

        KeyHolder keyHolder = new GeneratedKeyHolder();

        jdbcTemplate.update(connection -> {
            PreparedStatement ps = connection.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS);
            ps.setString(1, user.getUsername());
            ps.setString(2, user.getEmail());
            ps.setInt(3, user.getAge());
            ps.setString(4, user.getStatus());
            ps.setObject(5, user.getCreatedAt());
            return ps;
        }, keyHolder);

        Number key = keyHolder.getKey();
        return key == null ? null : key.longValue();
    }

    public int updateEmail(Long id, String email) {
        String sql = "update users set email = ? where id = ?";
        return jdbcTemplate.update(sql, email, id);
    }

    public int deleteById(Long id) {
        String sql = "delete from users where id = ?";
        return jdbcTemplate.update(sql, id);
    }

    public Optional<User> findById(Long id) {
        String sql = """
                select id, username, email, age, status, created_at
                from users
                where id = ?
                """;

        try {
            User user = jdbcTemplate.queryForObject(sql, USER_ROW_MAPPER, id);
            return Optional.ofNullable(user);
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }

    public List<User> findByStatus(String status) {
        String sql = """
                select id, username, email, age, status, created_at
                from users
                where status = ?
                order by id desc
                """;

        return jdbcTemplate.query(sql, USER_ROW_MAPPER, status);
    }

    public List<User> findPage(int page, int size) {
        int offset = (page - 1) * size;

        String sql = """
                select id, username, email, age, status, created_at
                from users
                order by id desc
                limit ? offset ?
                """;

        return jdbcTemplate.query(sql, USER_ROW_MAPPER, size, offset);
    }

    public int[] batchUpdateStatus(List<Long> ids, String status) {
        String sql = "update users set status = ? where id = ?";

        return jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                ps.setString(1, status);
                ps.setLong(2, ids.get(i));
            }

            @Override
            public int getBatchSize() {
                return ids.size();
            }
        });
    }
}
```

### Controller 示例

配合 `Spring MVC` 可以很快写出一个简单接口。

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.time.LocalDateTime;
import java.util.List;

@RestController
@RequestMapping("/users")
public class UserController {

    private final UserRepository userRepository;

    public UserController(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @PostMapping
    public Long create(@RequestBody User user) {
        user.setStatus("ACTIVE");
        user.setCreatedAt(LocalDateTime.now());
        return userRepository.saveAndReturnId(user);
    }

    @GetMapping("/{id}")
    public User detail(@PathVariable Long id) {
        return userRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("用户不存在"));
    }

    @GetMapping
    public List<User> list() {
        return userRepository.findByStatus("ACTIVE");
    }
}
```

### 参数绑定和 SQL 注入

`JdbcTemplate` 支持参数绑定：

```java
String sql = "select * from users where email = ?";
User user = jdbcTemplate.queryForObject(sql, USER_ROW_MAPPER, email);
```

这里的 `email` 不会直接拼到 SQL 字符串里，而是作为参数传给数据库驱动。

这种写法比字符串拼接更稳妥：

```java
String sql = "select * from users where email = '" + email + "'";
```

常见原则：

```text
值用参数绑定
表名、字段名、排序字段不能直接接收外部输入
```

如果排序字段来自前端，可以做白名单映射：

```java
public String resolveSortColumn(String sort) {
    if ("age".equals(sort)) {
        return "age";
    }
    if ("createdAt".equals(sort)) {
        return "created_at";
    }
    return "id";
}
```

### 异常处理

`JdbcTemplate` 会把底层 `SQLException` 转成 Spring 的 `DataAccessException` 体系。

常见异常有：

| 异常 | 常见原因 |
| --- | --- |
| `EmptyResultDataAccessException` | `queryForObject` 没查到数据 |
| `IncorrectResultSizeDataAccessException` | 期望一条，实际多条 |
| `DuplicateKeyException` | 唯一键冲突 |
| `BadSqlGrammarException` | SQL 语法错误、表名字段名错误 |
| `DataIntegrityViolationException` | 非空约束、外键约束、字段长度等问题 |

在业务层通常不需要捕获所有数据库异常。

更常见的做法是：

```text
Repository 层处理“查不到”这种正常分支
全局异常处理统一处理其他数据库异常
```

### JdbcTemplate 和 MyBatis 的区别

`JdbcTemplate` 和 `MyBatis` 都是直接操作 SQL 的方案。

差异大致可以这样理解：

| 对比点 | JdbcTemplate | MyBatis |
| --- | --- | --- |
| SQL 位置 | 多数写在 Java 代码里 | 常写在 XML 或注解里 |
| 映射能力 | 轻量，常用 RowMapper | 更完整，支持复杂映射 |
| 学习成本 | 较低 | 中等 |
| 动态 SQL | 手写字符串或命名参数 | 动态 SQL 支持更成熟 |
| 适合场景 | 简单 CRUD、报表查询、小型模块 | 复杂 SQL、较多数据访问层代码 |

`JdbcTemplate` 的优势是直接、轻量、接近 JDBC。

如果项目里 SQL 很复杂、映射关系很多，`MyBatis` 通常会更省维护成本。

如果只是少量 SQL、内部工具、简单后台、数据修复脚本接口，`JdbcTemplate` 用起来很顺手。

### 常见使用建议

### Repository 层统一放 SQL

SQL 通常放在 `Repository` 或 `Dao` 层。

```java
@Repository
public class UserRepository {
}
```

`Service` 层负责业务流程，少直接拼 SQL。

### 复杂映射优先手写 RowMapper

`BeanPropertyRowMapper` 很省事，但它更适合简单对象。

多表关联、字段别名、枚举转换、时间转换、金额转换等场景，手写 `RowMapper` 会更清楚。

### 查询可能为空时返回 Optional

对于按 ID 查询、按唯一字段查询这类方法，可以返回 `Optional`。

```java
public Optional<User> findByEmail(String email) {
    String sql = """
            select id, username, email, age, status, created_at
            from users
            where email = ?
            """;

    List<User> users = jdbcTemplate.query(sql, USER_ROW_MAPPER, email);

    if (users.isEmpty()) {
        return Optional.empty();
    }

    return Optional.of(users.get(0));
}
```

方法签名本身就表达了：

```text
这个用户可能存在，也可能不存在。
```

### 大批量数据注意分批

`batchUpdate` 适合批量操作，但不是一次塞得越多越好。

如果一次处理几十万条数据，通常需要拆批：

```text
每 500 条或 1000 条提交一批。
```

具体大小要看数据库、连接池、SQL 复杂度和网络情况。

### 动态条件的参数绑定

动态查询经常会写成这样：

```java
String sql = "select * from users where 1 = 1";

if (status != null) {
    sql += " and status = '" + status + "'";
}
```

这种写法不利于参数绑定。

更合适的是保留参数列表：

```java
StringBuilder sql = new StringBuilder("""
        select id, username, email, age, status, created_at
        from users
        where 1 = 1
        """);

List<Object> args = new ArrayList<>();

if (status != null) {
    sql.append(" and status = ?");
    args.add(status);
}

if (minAge != null) {
    sql.append(" and age >= ?");
    args.add(minAge);
}

List<User> users = jdbcTemplate.query(
        sql.toString(),
        USER_ROW_MAPPER,
        args.toArray()
);
```

如果动态条件很多，`NamedParameterJdbcTemplate` 会更容易维护。

### 常用方法汇总

| 方法 | 作用 | 常见场景 |
| --- | --- | --- |
| `update(sql, args...)` | 执行新增、修改、删除 | `insert/update/delete` |
| `queryForObject(sql, Class<T>, args...)` | 查询单个值 | 统计、单字段 |
| `queryForObject(sql, RowMapper<T>, args...)` | 查询单个对象 | 按主键查询且必须存在 |
| `query(sql, RowMapper<T>, args...)` | 查询列表 | 列表页、条件查询 |
| `batchUpdate(sql, batchArgs)` | 批量执行 | 批量插入、批量更新 |
| `execute(sql)` | 执行普通 SQL | DDL、初始化脚本 |
| `NamedParameterJdbcTemplate` | 命名参数 SQL | 多条件查询、`IN` 查询 |
| `RowMapper<T>` | 单行结果映射对象 | 自定义实体映射 |
| `BeanPropertyRowMapper<T>` | 自动属性映射 | 简单表和简单实体 |

### 总结

`JdbcTemplate` 的定位很清楚：

```text
保留 SQL 的直接控制感，同时减少 JDBC 模板代码。
```

它适合这些场景：

* SQL 数量不多
* 查询逻辑比较直接
* 不想引入完整 ORM 或 Mapper 框架
* 需要快速写内部工具、管理后台、数据处理逻辑
* 希望直接控制 SQL 和参数

实际使用时，重点放在几件事上：

* 用参数绑定，不拼接外部输入
* 用 `RowMapper` 管好结果映射
* 用 `query` 或 `Optional` 表达可能查不到的数据
* 用 `NamedParameterJdbcTemplate` 处理多参数和 `IN` 查询
* 用 `@Transactional` 管住多个 SQL 的一致性

掌握这些内容后，`JdbcTemplate` 已经可以覆盖大多数轻量数据库访问场景。
