### 简介

`JDBC` 全称是 `Java Database Connectivity`。

它是 Java 官方提供的一套数据库访问 API。

简单理解：

```text
Java 程序
  |
  v
JDBC API
  |
  v
数据库驱动
  |
  v
MySQL / PostgreSQL / Oracle / SQL Server
```

`JdbcTemplate`、`MyBatis`、`MyBatis-Plus`、`MyBatis-Flex`、`Spring Data JPA`，底层最终都会通过 JDBC 和数据库通信。

一句话概括：

```text
JDBC 是 Java 访问关系型数据库的底层标准 API，负责连接数据库、执行 SQL、读取结果和控制事务。
```

### JDBC 解决什么问题

Java 程序想访问数据库，需要做几件事：

```text
连接数据库
发送 SQL
绑定参数
接收查询结果
把结果转成 Java 对象
提交或回滚事务
关闭连接资源
```

JDBC 就是这些动作的标准接口。

数据库厂商负责提供驱动。

比如：

* MySQL 驱动
* PostgreSQL 驱动
* Oracle 驱动
* SQL Server 驱动

Java 代码面向 JDBC API 编程，具体数据库由驱动完成适配。

### JDBC 和常见框架的关系

常见持久层框架最终都会走 JDBC。

```text
Spring Data JPA
MyBatis
MyBatis-Plus
JdbcTemplate
  |
  v
JDBC
  |
  v
数据库驱动
  |
  v
数据库
```

JDBC 更底层。

框架通常是在 JDBC 之上封装：

* 连接管理
* 事务管理
* SQL 映射
* 结果集映射
* 异常转换
* 分页插件
* 缓存

理解 JDBC 后，再看 MyBatis、JPA、JdbcTemplate，会更容易明白这些框架到底省掉了哪些重复代码。

### 核心接口

| 接口 / 类 | 作用 |
| --- | --- |
| `DriverManager` | 根据 URL 获取数据库连接 |
| `DataSource` | 数据源，通常由连接池实现 |
| `Connection` | 数据库连接，也负责事务控制 |
| `Statement` | 执行普通 SQL |
| `PreparedStatement` | 执行带参数的预编译 SQL |
| `CallableStatement` | 调用存储过程 |
| `ResultSet` | 查询结果集 |
| `SQLException` | JDBC 异常 |

日常最常用的是：

```text
Connection
PreparedStatement
ResultSet
```

### Maven 依赖

以 MySQL 为例：

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

如果不是 Spring Boot 项目，可以显式写版本：

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>${mysql-connector-j.version}</version>
</dependency>
```

如果需要连接池，可以使用 `HikariCP`：

```xml
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>${hikaricp.version}</version>
</dependency>
```

Spring Boot 默认连接池就是 `HikariCP`。

### 准备数据库

```sql
CREATE DATABASE jdbc_demo DEFAULT CHARACTER SET utf8mb4;

USE jdbc_demo;

DROP TABLE IF EXISTS users;

CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL,
  age INT NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at DATETIME NOT NULL,
  UNIQUE KEY uk_users_email (email)
);

INSERT INTO users (username, email, age, status, created_at) VALUES
('张三', 'zhangsan@example.com', 20, 'ACTIVE', '2026-01-01 10:00:00'),
('李四', 'lisi@example.com', 25, 'ACTIVE', '2026-01-02 10:00:00'),
('王五', 'wangwu@example.com', 17, 'DISABLED', '2026-01-03 10:00:00');
```

### JDBC URL

MySQL 常见 URL：

```text
jdbc:mysql://localhost:3306/jdbc_demo?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
```

结构说明：

```text
jdbc:mysql://主机:端口/数据库名?参数
```

常见参数：

| 参数 | 作用 |
| --- | --- |
| `useSSL=false` | 本地开发关闭 SSL |
| `serverTimezone=Asia/Shanghai` | 指定时区 |
| `characterEncoding=utf8` | 指定字符编码 |
| `allowPublicKeyRetrieval=true` | 某些 MySQL 认证场景需要 |

### 第一个查询 Demo

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;

public class JdbcFirstDemo {

    public static void main(String[] args) throws Exception {
        String url = "jdbc:mysql://localhost:3306/jdbc_demo?useSSL=false&serverTimezone=Asia/Shanghai";
        String username = "root";
        String password = "123456";

        try (Connection connection = DriverManager.getConnection(url, username, password);
             Statement statement = connection.createStatement();
             ResultSet resultSet = statement.executeQuery(
                     "select id, username, email, age, status from users"
             )) {

            while (resultSet.next()) {
                Long id = resultSet.getLong("id");
                String name = resultSet.getString("username");
                String email = resultSet.getString("email");
                Integer age = resultSet.getInt("age");
                String status = resultSet.getString("status");

                System.out.println(id + " " + name + " " + email + " " + age + " " + status);
            }
        }
    }
}
```

JDBC 标准流程：

```text
获取 Connection
创建 Statement 或 PreparedStatement
执行 SQL
处理 ResultSet
关闭资源
```

这里使用了 `try-with-resources`。

资源会自动关闭。

关闭顺序是后创建的资源先关闭。

### Statement 和 PreparedStatement

`Statement` 适合执行固定 SQL。

```java
Statement statement = connection.createStatement();
ResultSet rs = statement.executeQuery("select * from users");
```

但带参数时，不适合用字符串拼接。

例如：

```java
String sql = "select * from users where email = '" + email + "'";
```

这种写法可读性差，也容易带来 SQL 注入风险。

更常见的是 `PreparedStatement`：

```java
String sql = "select id, username, email, age, status from users where email = ?";

try (PreparedStatement statement = connection.prepareStatement(sql)) {
    statement.setString(1, "zhangsan@example.com");

    try (ResultSet rs = statement.executeQuery()) {
        while (rs.next()) {
            System.out.println(rs.getString("username"));
        }
    }
}
```

`?` 是占位符。

参数通过：

```java
statement.setString(1, value);
statement.setInt(2, value);
statement.setLong(3, value);
```

绑定。

下标从 `1` 开始。

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

### 工具类

先写一个简单工具类。

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class JdbcUtil {

    private static final String URL = "jdbc:mysql://localhost:3306/jdbc_demo?useSSL=false&serverTimezone=Asia/Shanghai";
    private static final String USERNAME = "root";
    private static final String PASSWORD = "123456";

    public static Connection getConnection() throws SQLException {
        return DriverManager.getConnection(URL, USERNAME, PASSWORD);
    }
}
```

JDBC 4 之后，驱动通常可以自动加载。

传统写法里常见：

```java
Class.forName("com.mysql.cj.jdbc.Driver");
```

现代 MySQL 驱动一般不需要手动写。

### 结果集映射

把 `ResultSet` 转成 `User`：

```java
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Timestamp;

public class UserRowMapper {

    public static User map(ResultSet rs) throws SQLException {
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
    }
}
```

这段代码就是很多框架中 `RowMapper` 的雏形。

`JdbcTemplate` 里的 `RowMapper`，本质上也是做类似的事情。

### DAO 实战：新增数据

```java
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.Statement;

public class UserDao {

    public Long insert(User user) {
        String sql = """
                insert into users (username, email, age, status, created_at)
                values (?, ?, ?, ?, ?)
                """;

        try (Connection connection = JdbcUtil.getConnection();
             PreparedStatement statement = connection.prepareStatement(
                     sql,
                     Statement.RETURN_GENERATED_KEYS
             )) {
            statement.setString(1, user.getUsername());
            statement.setString(2, user.getEmail());
            statement.setInt(3, user.getAge());
            statement.setString(4, user.getStatus());
            statement.setObject(5, user.getCreatedAt());

            statement.executeUpdate();

            try (ResultSet keys = statement.getGeneratedKeys()) {
                if (keys.next()) {
                    return keys.getLong(1);
                }
            }

            return null;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

这里用到了：

```java
Statement.RETURN_GENERATED_KEYS
```

用于获取数据库生成的自增主键。

### DAO 实战：按 ID 查询

```java
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.Optional;

public Optional<User> findById(Long id) {
    String sql = """
            select id, username, email, age, status, created_at
            from users
            where id = ?
            """;

    try (Connection connection = JdbcUtil.getConnection();
         PreparedStatement statement = connection.prepareStatement(sql)) {

        statement.setLong(1, id);

        try (ResultSet rs = statement.executeQuery()) {
            if (rs.next()) {
                return Optional.of(UserRowMapper.map(rs));
            }
        }

        return Optional.empty();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

返回 `Optional<User>`，表示可能查到，也可能查不到。

### DAO 实战：条件查询

```java
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.ArrayList;
import java.util.List;

public List<User> search(String status, Integer minAge) {
    StringBuilder sql = new StringBuilder("""
            select id, username, email, age, status, created_at
            from users
            where 1 = 1
            """);

    List<Object> args = new ArrayList<>();

    if (status != null && !status.isBlank()) {
        sql.append(" and status = ?");
        args.add(status);
    }

    if (minAge != null) {
        sql.append(" and age >= ?");
        args.add(minAge);
    }

    sql.append(" order by id desc");

    try (Connection connection = JdbcUtil.getConnection();
         PreparedStatement statement = connection.prepareStatement(sql.toString())) {

        for (int i = 0; i < args.size(); i++) {
            statement.setObject(i + 1, args.get(i));
        }

        List<User> users = new ArrayList<>();

        try (ResultSet rs = statement.executeQuery()) {
            while (rs.next()) {
                users.add(UserRowMapper.map(rs));
            }
        }

        return users;
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

动态条件里，SQL 结构可以用 `StringBuilder` 拼接。

业务值仍然用 `?` 绑定。

### DAO 实战：分页查询

```java
public List<User> findPage(String status, int pageNumber, int pageSize) {
    int offset = (pageNumber - 1) * pageSize;

    String sql = """
            select id, username, email, age, status, created_at
            from users
            where status = ?
            order by id desc
            limit ? offset ?
            """;

    try (Connection connection = JdbcUtil.getConnection();
         PreparedStatement statement = connection.prepareStatement(sql)) {

        statement.setString(1, status);
        statement.setInt(2, pageSize);
        statement.setInt(3, offset);

        List<User> users = new ArrayList<>();

        try (ResultSet rs = statement.executeQuery()) {
            while (rs.next()) {
                users.add(UserRowMapper.map(rs));
            }
        }

        return users;
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

统计总数：

```java
public long countByStatus(String status) {
    String sql = "select count(*) from users where status = ?";

    try (Connection connection = JdbcUtil.getConnection();
         PreparedStatement statement = connection.prepareStatement(sql)) {

        statement.setString(1, status);

        try (ResultSet rs = statement.executeQuery()) {
            if (rs.next()) {
                return rs.getLong(1);
            }
        }

        return 0;
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

### DAO 实战：更新数据

```java
public int updateEmail(Long id, String email) {
    String sql = "update users set email = ? where id = ?";

    try (Connection connection = JdbcUtil.getConnection();
         PreparedStatement statement = connection.prepareStatement(sql)) {

        statement.setString(1, email);
        statement.setLong(2, id);

        return statement.executeUpdate();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

`executeUpdate()` 用于执行：

* `insert`
* `update`
* `delete`

返回受影响行数。

### DAO 实战：删除数据

```java
public int deleteById(Long id) {
    String sql = "delete from users where id = ?";

    try (Connection connection = JdbcUtil.getConnection();
         PreparedStatement statement = connection.prepareStatement(sql)) {

        statement.setLong(1, id);

        return statement.executeUpdate();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

### 事务控制

默认情况下，JDBC 连接是自动提交的。

也就是每执行一条 SQL，数据库自动提交一次。

事务需要手动关闭自动提交：

```java
connection.setAutoCommit(false);
```

示例：账户转账。

```java
import java.math.BigDecimal;
import java.sql.Connection;
import java.sql.PreparedStatement;

public void transfer(Long fromAccountId, Long toAccountId, BigDecimal amount) {
    String decreaseSql = "update accounts set balance = balance - ? where id = ?";
    String increaseSql = "update accounts set balance = balance + ? where id = ?";

    try (Connection connection = JdbcUtil.getConnection()) {
        connection.setAutoCommit(false);

        try (PreparedStatement decrease = connection.prepareStatement(decreaseSql);
             PreparedStatement increase = connection.prepareStatement(increaseSql)) {

            decrease.setBigDecimal(1, amount);
            decrease.setLong(2, fromAccountId);
            decrease.executeUpdate();

            increase.setBigDecimal(1, amount);
            increase.setLong(2, toAccountId);
            increase.executeUpdate();

            connection.commit();
        } catch (Exception e) {
            connection.rollback();
            throw e;
        } finally {
            connection.setAutoCommit(true);
        }
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

事务关键点：

```text
setAutoCommit(false)
执行多条 SQL
成功 commit
失败 rollback
```

### 事务隔离级别

可以通过 `Connection` 设置事务隔离级别。

```java
connection.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
```

常见隔离级别：

| 常量 | 含义 |
| --- | --- |
| `TRANSACTION_READ_UNCOMMITTED` | 读未提交 |
| `TRANSACTION_READ_COMMITTED` | 读已提交 |
| `TRANSACTION_REPEATABLE_READ` | 可重复读 |
| `TRANSACTION_SERIALIZABLE` | 串行化 |

实际项目里，通常使用数据库默认隔离级别。

只有明确出现并发一致性问题时，才单独调整。

### 批处理

批量插入时，可以使用 `addBatch` 和 `executeBatch`。

```java
public int[] batchInsert(List<User> users) {
    String sql = """
            insert into users (username, email, age, status, created_at)
            values (?, ?, ?, ?, ?)
            """;

    try (Connection connection = JdbcUtil.getConnection();
         PreparedStatement statement = connection.prepareStatement(sql)) {

        connection.setAutoCommit(false);

        try {
            for (User user : users) {
                statement.setString(1, user.getUsername());
                statement.setString(2, user.getEmail());
                statement.setInt(3, user.getAge());
                statement.setString(4, user.getStatus());
                statement.setObject(5, user.getCreatedAt());
                statement.addBatch();
            }

            int[] result = statement.executeBatch();
            connection.commit();

            return result;
        } catch (Exception e) {
            connection.rollback();
            throw e;
        } finally {
            connection.setAutoCommit(true);
        }
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

如果使用 MySQL，批量插入常会配合连接参数：

```text
rewriteBatchedStatements=true
```

示例：

```text
jdbc:mysql://localhost:3306/jdbc_demo?rewriteBatchedStatements=true
```

大批量数据建议分批处理。

比如每 `500` 或 `1000` 条执行一次。

### CallableStatement

`CallableStatement` 用来调用存储过程。

示例：

```java
try (Connection connection = JdbcUtil.getConnection();
     CallableStatement statement = connection.prepareCall("{call refresh_user_stats(?)}")) {

    statement.setLong(1, 1L);
    statement.execute();
}
```

存储过程使用不多，但在一些数据库重逻辑系统中仍然存在。

### 连接池

直接用 `DriverManager.getConnection`，每次都会创建数据库连接。

数据库连接是比较重的资源。

实际项目通常使用连接池。

连接池负责：

* 提前创建连接
* 复用连接
* 控制最大连接数
* 检测连接可用性
* 归还连接

常见连接池：

* `HikariCP`
* `Druid`
* `Tomcat JDBC Pool`

Spring Boot 默认使用 `HikariCP`。

### HikariCP Demo

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;

public class DataSourceFactory {

    private static final HikariDataSource DATA_SOURCE;

    static {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/jdbc_demo?useSSL=false&serverTimezone=Asia/Shanghai");
        config.setUsername("root");
        config.setPassword("123456");
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(2);
        config.setConnectionTimeout(30000);

        DATA_SOURCE = new HikariDataSource(config);
    }

    public static Connection getConnection() throws SQLException {
        return DATA_SOURCE.getConnection();
    }

    public static DataSource getDataSource() {
        return DATA_SOURCE;
    }
}
```

使用连接池时：

```java
try (Connection connection = DataSourceFactory.getConnection()) {
    // 使用连接
}
```

这里的 `close()` 不是真的关闭物理连接。

它表示把连接归还给连接池。

### JDBC 和 JdbcTemplate 的区别

`JdbcTemplate` 是 Spring 对 JDBC 的封装。

| 对比项 | JDBC | JdbcTemplate |
| --- | --- | --- |
| 获取连接 | 手动处理 | DataSource 管理 |
| 关闭资源 | 手动或 try-with-resources | 自动处理 |
| SQL 执行 | 手动写模板代码 | 简化封装 |
| 结果映射 | 手动从 ResultSet 取值 | RowMapper |
| 异常 | SQLException | DataAccessException |
| 适合场景 | 学习底层、少量工具代码 | Spring 项目常规 JDBC 操作 |

`JdbcTemplate` 的很多能力，本质上就是帮 JDBC 去掉模板代码。

### JDBC 和 MyBatis 的区别

| 对比项 | JDBC | MyBatis |
| --- | --- | --- |
| SQL 位置 | Java 代码里 | XML 或注解 |
| 参数绑定 | 手动 set 参数 | 框架处理 |
| 结果映射 | 手动 ResultSet 映射 | resultType / resultMap |
| 动态 SQL | 手动拼接 | XML 标签 |
| 事务 | Connection 手动控制 | Spring 管理 |
| 适合场景 | 底层学习、简单脚本 | 业务系统数据访问层 |

MyBatis 底层仍然基于 JDBC。

只是把 SQL 组织、参数绑定、结果映射这些事情封装得更适合项目开发。

### 常见使用建议

### 优先使用 PreparedStatement

带参数的 SQL 优先使用：

```java
PreparedStatement
```

业务值用 `?` 占位符绑定。

字段名、表名、排序字段这类 SQL 结构不能用 `?` 绑定时，先做白名单映射。

### 使用 try-with-resources 管理资源

JDBC 资源都需要关闭。

常见资源：

* `Connection`
* `Statement`
* `PreparedStatement`
* `ResultSet`

推荐写法：

```java
try (Connection connection = JdbcUtil.getConnection();
     PreparedStatement statement = connection.prepareStatement(sql);
     ResultSet rs = statement.executeQuery()) {
    // 处理结果集
}
```

### 业务项目优先使用连接池

`DriverManager` 适合学习和简单工具。

Web 项目、后台服务、批处理任务更适合使用连接池。

连接池可以减少频繁创建和销毁数据库连接的开销。

### 大批量数据分批提交

批处理不是一次塞得越多越好。

如果一次提交几十万条数据，可能带来：

* SQL 包太大
* 内存占用过高
* 事务过大
* 锁持有时间过长

更常见的是分批执行。

```text
每 500 条或 1000 条提交一次。
```

### 异常需要明确处理

学习 Demo 里经常看到：

```java
e.printStackTrace();
```

实际项目里，更常见的是：

```java
throw new RuntimeException(e);
```

或者转换成项目里的业务异常、数据访问异常。

### 常用方法汇总

| 方法 | 作用 |
| --- | --- |
| `DriverManager.getConnection(...)` | 获取数据库连接 |
| `connection.prepareStatement(sql)` | 创建预编译 SQL |
| `statement.executeQuery()` | 执行查询 |
| `statement.executeUpdate()` | 执行增删改 |
| `statement.addBatch()` | 添加批处理 |
| `statement.executeBatch()` | 执行批处理 |
| `statement.getGeneratedKeys()` | 获取自增主键 |
| `resultSet.next()` | 移动到下一行 |
| `resultSet.getString(...)` | 获取字符串字段 |
| `resultSet.getInt(...)` | 获取整数字段 |
| `connection.setAutoCommit(false)` | 关闭自动提交 |
| `connection.commit()` | 提交事务 |
| `connection.rollback()` | 回滚事务 |
| `connection.setTransactionIsolation(...)` | 设置事务隔离级别 |

### 总结

`JDBC` 是 Java 数据库访问的基础。

它的核心流程非常固定：

```text
获取连接
准备 SQL
绑定参数
执行 SQL
处理结果集
提交或回滚事务
关闭资源
```

实际项目里，通常不会在每个业务方法里直接写大量 JDBC 模板代码。

更常见的是使用：

* `JdbcTemplate`
* `MyBatis`
* `MyBatis-Plus`
* `Spring Data JPA`

但这些框架底层都离不开 JDBC。

掌握 `Connection`、`PreparedStatement`、`ResultSet`、事务和连接池之后，再看上层持久层框架，会更容易理解它们到底封装了什么。
