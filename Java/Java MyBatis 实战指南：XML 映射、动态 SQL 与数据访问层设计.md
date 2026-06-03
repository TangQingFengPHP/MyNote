### 简介

`MyBatis` 是 Java 里很常见的持久层框架。

它做的事情可以简单理解为：

```text
Java 方法
  |
  v
MyBatis
  |
  v
SQL
  |
  v
数据库
```

相比直接写 `JDBC`，`MyBatis` 省掉了大量重复代码：

```text
获取连接
创建 PreparedStatement
绑定参数
遍历 ResultSet
把结果塞进 Java 对象
关闭资源
处理异常
```

相比全自动 ORM，`MyBatis` 又保留了 SQL 的控制权。

SQL 写成什么样，执行计划怎么优化，字段怎么映射，都可以清楚地放在 Mapper XML 或注解里。

一句话概括：

```text
MyBatis 是一个以 SQL 为中心的持久层框架，适合需要清楚控制 SQL 和结果映射的项目。
```

### MyBatis 解决什么问题

先看一段普通 `JDBC` 查询代码：

```java
String sql = "select id, username, email, age from sys_user where id = ?";

Connection connection = dataSource.getConnection();
PreparedStatement statement = connection.prepareStatement(sql);
statement.setLong(1, 1L);

ResultSet resultSet = statement.executeQuery();
User user = null;

if (resultSet.next()) {
    user = new User();
    user.setId(resultSet.getLong("id"));
    user.setUsername(resultSet.getString("username"));
    user.setEmail(resultSet.getString("email"));
    user.setAge(resultSet.getInt("age"));
}

resultSet.close();
statement.close();
connection.close();
```

真正有业务含义的是：

```text
SQL 是什么
参数是什么
结果怎么映射成 User
```

`MyBatis` 把这些固定流程封装起来。

Mapper 接口：

```java
public interface UserMapper {
    User selectById(Long id);
}
```

Mapper XML：

```xml
<select id="selectById" resultType="User">
    select id, username, email, age
    from sys_user
    where id = #{id}
</select>
```

调用时只需要：

```java
User user = userMapper.selectById(1L);
```

这就是 `MyBatis` 的核心价值：

```text
SQL 自己写，参数绑定和结果映射交给框架处理。
```

### 核心概念

| 名称 | 作用 |
| --- | --- |
| `Mapper` 接口 | 定义数据库操作方法 |
| `Mapper XML` | 编写 SQL 和结果映射 |
| `SqlSessionFactory` | 创建 `SqlSession` 的工厂 |
| `SqlSession` | 一次数据库会话，负责执行 SQL |
| `MappedStatement` | XML 或注解里的一个 SQL 方法 |
| `ResultMap` | 把查询结果映射成 Java 对象 |
| `TypeHandler` | 处理 Java 类型和 JDBC 类型转换 |
| 动态 SQL | 根据条件拼接 SQL |
| 一级缓存 | `SqlSession` 级别缓存 |
| 二级缓存 | Mapper 命名空间级别缓存 |

在 `Spring Boot` 项目里，日常开发最常接触的是：

```text
Mapper 接口
Mapper XML
Entity
Service
Controller
```

`SqlSessionFactory` 和 `SqlSessionTemplate` 通常由 `mybatis-spring-boot-starter` 自动配置。

### Maven 依赖

`MyBatis Spring Boot Starter` 根据 `Spring Boot` 版本分支不同。

`Spring Boot 3.x` 常用：

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>3.0.5</version>
</dependency>
```

`Spring Boot 4.x` 可使用：

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>4.0.0</version>
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

如果需要 Web 接口，再引入：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### 数据源和 MyBatis 配置

`application.yml` 示例：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mybatis_demo?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver

mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.example.demo.entity
  configuration:
    map-underscore-to-camel-case: true
    default-statement-timeout: 30
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

几个常见配置：

* `mapper-locations`：Mapper XML 文件位置
* `type-aliases-package`：实体类别名扫描包
* `map-underscore-to-camel-case`：下划线转驼峰
* `default-statement-timeout`：SQL 默认超时时间
* `log-impl`：开发环境打印 SQL

`map-underscore-to-camel-case` 很常用。

数据库字段：

```text
create_time
```

Java 属性：

```text
createTime
```

开启后可以自动映射。

### 启动类配置

启动类加上 `@MapperScan`：

```java
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@MapperScan("com.example.demo.mapper")
public class MyBatisDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyBatisDemoApplication.class, args);
    }
}
```

也可以在每个 Mapper 接口上加 `@Mapper`。

Mapper 数量比较多时，`@MapperScan` 更省事。

### 准备演示表

下面用用户表和订单表做示例。

```sql
DROP TABLE IF EXISTS sys_order;
DROP TABLE IF EXISTS sys_user;

CREATE TABLE sys_user (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL,
  age INT NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NULL
);

CREATE TABLE sys_order (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  order_no VARCHAR(50) NOT NULL,
  amount DECIMAL(10, 2) NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at DATETIME NOT NULL,
  INDEX idx_user_id (user_id)
);

INSERT INTO sys_user (username, email, age, status, created_at, updated_at) VALUES
('张三', 'zhangsan@example.com', 20, 'ACTIVE', '2026-01-01 10:00:00', NULL),
('李四', 'lisi@example.com', 25, 'ACTIVE', '2026-01-02 10:00:00', NULL),
('王五', 'wangwu@example.com', 17, 'DISABLED', '2026-01-03 10:00:00', NULL);

INSERT INTO sys_order (user_id, order_no, amount, status, created_at) VALUES
(1, 'A001', 99.00, 'PAID', '2026-02-01 10:00:00'),
(1, 'A002', 260.00, 'PAID', '2026-02-02 10:00:00'),
(2, 'A003', 35.50, 'CANCELLED', '2026-02-03 10:00:00');
```

### 实体类

用户实体：

```java
package com.example.demo.entity;

import java.time.LocalDateTime;

public class User {
    private Long id;
    private String username;
    private String email;
    private Integer age;
    private String status;
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

订单实体：

```java
package com.example.demo.entity;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class Order {
    private Long id;
    private Long userId;
    private String orderNo;
    private BigDecimal amount;
    private String status;
    private LocalDateTime createdAt;

    // getter setter
}
```

### Mapper 接口

```java
package com.example.demo.mapper;

import com.example.demo.entity.User;
import org.apache.ibatis.annotations.Param;

import java.util.List;
import java.util.Optional;

public interface UserMapper {

    int insert(User user);

    int update(User user);

    int deleteById(@Param("id") Long id);

    User selectById(@Param("id") Long id);

    List<User> selectAll();

    List<User> search(@Param("keyword") String keyword,
                      @Param("status") String status,
                      @Param("minAge") Integer minAge);

    List<User> selectByIds(@Param("ids") List<Long> ids);

    List<User> selectPage(@Param("status") String status,
                          @Param("offset") int offset,
                          @Param("size") int size);

    long countByStatus(@Param("status") String status);
}
```

Mapper 接口只定义方法。

具体 SQL 放在 XML 里。

### Mapper XML 基本结构

文件路径：

```text
src/main/resources/mapper/UserMapper.xml
```

基础结构：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "https://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.demo.mapper.UserMapper">

</mapper>
```

`namespace` 必须对应 Mapper 接口全限定名：

```text
com.example.demo.mapper.UserMapper
```

XML 里的 SQL 标签 `id` 要和接口方法名对应。

```java
User selectById(Long id);
```

对应：

```xml
<select id="selectById">
</select>
```

### resultType 查询

简单查询可以使用 `resultType`。

```xml
<select id="selectById" resultType="User">
    select id, username, email, age, status, created_at, updated_at
    from sys_user
    where id = #{id}
</select>
```

如果开启了下划线转驼峰：

```yaml
mybatis:
  configuration:
    map-underscore-to-camel-case: true
```

`created_at` 可以自动映射到 `createdAt`。

### resultMap 映射

复杂映射更适合使用 `resultMap`。

```xml
<resultMap id="UserResultMap" type="User">
    <id property="id" column="id"/>
    <result property="username" column="username"/>
    <result property="email" column="email"/>
    <result property="age" column="age"/>
    <result property="status" column="status"/>
    <result property="createdAt" column="created_at"/>
    <result property="updatedAt" column="updated_at"/>
</resultMap>
```

查询时引用：

```xml
<select id="selectById" resultMap="UserResultMap">
    select id, username, email, age, status, created_at, updated_at
    from sys_user
    where id = #{id}
</select>
```

`resultType` 适合简单映射。

`resultMap` 适合这些场景：

* 字段名和属性名差异较大
* 多表关联查询
* 一对一、一对多映射
* 枚举、嵌套对象、复杂对象结构

### 新增数据

Mapper 接口：

```java
int insert(User user);
```

XML：

```xml
<insert id="insert" useGeneratedKeys="true" keyProperty="id">
    insert into sys_user (username, email, age, status, created_at, updated_at)
    values (#{username}, #{email}, #{age}, #{status}, #{createdAt}, #{updatedAt})
</insert>
```

`useGeneratedKeys="true"` 表示使用数据库生成的主键。

`keyProperty="id"` 表示把生成的主键回填到实体对象的 `id` 属性。

调用：

```java
User user = new User();
user.setUsername("赵六");
user.setEmail("zhaoliu@example.com");
user.setAge(28);
user.setStatus("ACTIVE");
user.setCreatedAt(LocalDateTime.now());

userMapper.insert(user);

System.out.println(user.getId());
```

### 修改数据

Mapper 接口：

```java
int update(User user);
```

XML：

```xml
<update id="update">
    update sys_user
    <set>
        <if test="username != null and username != ''">
            username = #{username},
        </if>
        <if test="email != null and email != ''">
            email = #{email},
        </if>
        <if test="age != null">
            age = #{age},
        </if>
        <if test="status != null and status != ''">
            status = #{status},
        </if>
        updated_at = #{updatedAt}
    </set>
    where id = #{id}
</update>
```

`<set>` 会自动处理末尾多余的逗号。

调用：

```java
User user = new User();
user.setId(1L);
user.setEmail("new-zhangsan@example.com");
user.setUpdatedAt(LocalDateTime.now());

userMapper.update(user);
```

### 删除数据

Mapper 接口：

```java
int deleteById(@Param("id") Long id);
```

XML：

```xml
<delete id="deleteById">
    delete from sys_user
    where id = #{id}
</delete>
```

如果业务需要保留历史记录，可以把删除改成逻辑删除：

```xml
<update id="deleteById">
    update sys_user
    set status = 'DELETED', updated_at = now()
    where id = #{id}
</update>
```

### 查询列表

```xml
<select id="selectAll" resultMap="UserResultMap">
    select id, username, email, age, status, created_at, updated_at
    from sys_user
    order by id desc
</select>
```

业务表数据量较大时，查询全部要谨慎。

更常见的是条件查询或分页查询。

### 动态 SQL：where 和 if

动态查询是 `MyBatis` 很重要的能力。

比如三个条件都是可选的：

```text
keyword：用户名或邮箱关键字
status：用户状态
minAge：最小年龄
```

XML：

```xml
<select id="search" resultMap="UserResultMap">
    select id, username, email, age, status, created_at, updated_at
    from sys_user
    <where>
        <if test="keyword != null and keyword != ''">
            and (
                username like concat('%', #{keyword}, '%')
                or email like concat('%', #{keyword}, '%')
            )
        </if>
        <if test="status != null and status != ''">
            and status = #{status}
        </if>
        <if test="minAge != null">
            and age &gt;= #{minAge}
        </if>
    </where>
    order by id desc
</select>
```

`<where>` 的作用是：

```text
有条件时自动加 where
自动处理开头多余的 and 或 or
没有条件时不生成 where
```

调用：

```java
List<User> users = userMapper.search("张", "ACTIVE", 18);
```

### 动态 SQL：choose

`<choose>` 类似 Java 里的 `if / else if / else`。

```xml
<select id="searchByKeyword" resultMap="UserResultMap">
    select id, username, email, age, status, created_at, updated_at
    from sys_user
    <where>
        <choose>
            <when test="username != null and username != ''">
                username like concat('%', #{username}, '%')
            </when>
            <when test="email != null and email != ''">
                email = #{email}
            </when>
            <otherwise>
                status = 'ACTIVE'
            </otherwise>
        </choose>
    </where>
</select>
```

含义是：

```text
优先按 username 查
username 没传，再按 email 查
都没传，则查 ACTIVE 状态
```

### 动态 SQL：foreach

`<foreach>` 常用于 `IN` 查询和批量插入。

按 ID 集合查询：

```xml
<select id="selectByIds" resultMap="UserResultMap">
    select id, username, email, age, status, created_at, updated_at
    from sys_user
    where id in
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
```

Mapper 接口：

```java
List<User> selectByIds(@Param("ids") List<Long> ids);
```

调用：

```java
List<User> users = userMapper.selectByIds(Arrays.asList(1L, 2L, 3L));
```

批量插入：

```xml
<insert id="batchInsert">
    insert into sys_user (username, email, age, status, created_at)
    values
    <foreach collection="users" item="user" separator=",">
        (#{user.username}, #{user.email}, #{user.age}, #{user.status}, #{user.createdAt})
    </foreach>
</insert>
```

### sql 片段复用

常用字段可以抽成 SQL 片段。

```xml
<sql id="UserColumns">
    id, username, email, age, status, created_at, updated_at
</sql>
```

使用：

```xml
<select id="selectById" resultMap="UserResultMap">
    select
    <include refid="UserColumns"/>
    from sys_user
    where id = #{id}
</select>
```

字段较多时，这样能减少重复。

### 分页查询

MyBatis 本身不内置分页插件。

最直接的写法是 SQL 里写 `limit offset`。

Mapper 接口：

```java
List<User> selectPage(@Param("status") String status,
                      @Param("offset") int offset,
                      @Param("size") int size);
```

XML：

```xml
<select id="selectPage" resultMap="UserResultMap">
    select id, username, email, age, status, created_at, updated_at
    from sys_user
    <where>
        <if test="status != null and status != ''">
            and status = #{status}
        </if>
    </where>
    order by id desc
    limit #{size} offset #{offset}
</select>
```

统计总数：

```xml
<select id="countByStatus" resultType="long">
    select count(*)
    from sys_user
    <where>
        <if test="status != null and status != ''">
            and status = #{status}
        </if>
    </where>
</select>
```

Service 层计算偏移量：

```java
int offset = (pageNumber - 1) * pageSize;
List<User> records = userMapper.selectPage(status, offset, pageSize);
long total = userMapper.countByStatus(status);
```

也可以使用 `PageHelper` 这类分页插件。

插件适合多数据库、多列表统一分页的项目。

手写分页适合 SQL 较少、分页逻辑很直接的项目。

### 参数绑定：#{} 和 ${}

`#{}` 是参数绑定。

```xml
where id = #{id}
```

它会使用 `PreparedStatement`。

效果类似：

```sql
where id = ?
```

业务值一般都应该使用 `#{}`。

`${}` 是字符串替换。

```xml
order by ${sortColumn}
```

它会把内容直接拼到 SQL 里。

如果外部输入直接进入 `${}`，容易产生 SQL 注入风险。

常见原则：

```text
普通值用 #{}
表名、字段名、排序方向确实不能用 #{} 时，使用白名单后再放入 ${}
```

排序字段白名单示例：

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

### 多参数方法和 @Param

Mapper 方法只有一个简单参数时：

```java
User selectById(Long id);
```

XML 里通常可以直接写：

```xml
where id = #{id}
```

多个参数时，推荐加 `@Param`。

```java
List<User> search(@Param("keyword") String keyword,
                  @Param("status") String status,
                  @Param("minAge") Integer minAge);
```

XML 里就能直接使用：

```xml
#{keyword}
#{status}
#{minAge}
```

这样可读性更好，也减少参数名不一致的问题。

### 注解方式

简单 SQL 可以直接写在 Mapper 注解上。

```java
package com.example.demo.mapper;

import com.example.demo.entity.User;
import org.apache.ibatis.annotations.Delete;
import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Options;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;
import org.apache.ibatis.annotations.Update;

public interface UserAnnotationMapper {

    @Insert("""
            insert into sys_user (username, email, age, status, created_at)
            values (#{username}, #{email}, #{age}, #{status}, #{createdAt})
            """)
    @Options(useGeneratedKeys = true, keyProperty = "id")
    int insert(User user);

    @Select("""
            select id, username, email, age, status, created_at, updated_at
            from sys_user
            where id = #{id}
            """)
    User selectById(@Param("id") Long id);

    @Update("""
            update sys_user
            set email = #{email}, updated_at = now()
            where id = #{id}
            """)
    int updateEmail(@Param("id") Long id, @Param("email") String email);

    @Delete("delete from sys_user where id = #{id}")
    int deleteById(@Param("id") Long id);
}
```

注解方式适合：

```text
SQL 很短
没有复杂动态条件
没有复杂结果映射
```

XML 方式适合：

```text
SQL 较长
动态条件较多
多表关联较多
需要 ResultMap
需要统一维护 SQL
```

### 一对多 ResultMap

查询用户及订单列表时，可以使用 `collection`。

DTO：

```java
public class UserOrderDTO {
    private Long id;
    private String username;
    private String email;
    private List<Order> orders;

    // getter setter
}
```

ResultMap：

```xml
<resultMap id="UserOrderResultMap" type="com.example.demo.dto.UserOrderDTO">
    <id property="id" column="user_id"/>
    <result property="username" column="username"/>
    <result property="email" column="email"/>

    <collection property="orders" ofType="com.example.demo.entity.Order">
        <id property="id" column="order_id"/>
        <result property="userId" column="user_id"/>
        <result property="orderNo" column="order_no"/>
        <result property="amount" column="amount"/>
        <result property="status" column="order_status"/>
        <result property="createdAt" column="order_created_at"/>
    </collection>
</resultMap>
```

查询：

```xml
<select id="selectUserOrders" resultMap="UserOrderResultMap">
    select
      u.id as user_id,
      u.username,
      u.email,
      o.id as order_id,
      o.user_id,
      o.order_no,
      o.amount,
      o.status as order_status,
      o.created_at as order_created_at
    from sys_user u
    left join sys_order o on o.user_id = u.id
    where u.id = #{userId}
    order by o.id desc
</select>
```

多表查询时，字段别名很重要。

比如用户 ID 和订单 ID 都叫 `id`，如果不取别名，映射容易混乱。

### TypeHandler

`TypeHandler` 用来处理 Java 类型和数据库类型之间的转换。

常见场景：

```text
枚举 <-> varchar
JSON 字符串 <-> Java 对象
逗号字符串 <-> List
```

示例：状态枚举。

```java
public enum UserStatus {
    ACTIVE,
    DISABLED
}
```

实体字段：

```java
private UserStatus status;
```

如果数据库字段是 `varchar`，可以写自定义 `TypeHandler`。

项目里枚举字段较多时，统一处理类型转换会更清楚。

### 缓存

MyBatis 有一级缓存和二级缓存。

一级缓存：

```text
SqlSession 级别
默认开启
同一个 SqlSession 内重复查询可能命中缓存
```

二级缓存：

```text
Mapper namespace 级别
需要显式配置
跨 SqlSession
```

XML 中开启二级缓存：

```xml
<cache/>
```

实际业务里，二级缓存要谨慎开启。

原因是数据库数据可能被其他服务、其他 SQL、其他系统修改。

如果缓存失效策略没设计好，容易读到旧数据。

更常见的做法是：

```text
本地缓存或 Redis 由业务层统一管理。
```

### Service 层实战

```java
package com.example.demo.service;

import com.example.demo.entity.User;
import com.example.demo.mapper.UserMapper;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

@Service
public class UserService {

    private final UserMapper userMapper;

    public UserService(UserMapper userMapper) {
        this.userMapper = userMapper;
    }

    @Transactional
    public Long create(User user) {
        user.setStatus("ACTIVE");
        user.setCreatedAt(LocalDateTime.now());

        userMapper.insert(user);

        return user.getId();
    }

    public Optional<User> findById(Long id) {
        return Optional.ofNullable(userMapper.selectById(id));
    }

    public List<User> search(String keyword, String status, Integer minAge) {
        return userMapper.search(keyword, status, minAge);
    }

    public PageResult<User> page(String status, int pageNumber, int pageSize) {
        int offset = (pageNumber - 1) * pageSize;

        List<User> records = userMapper.selectPage(status, offset, pageSize);
        long total = userMapper.countByStatus(status);

        return new PageResult<>(records, total, pageNumber, pageSize);
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
    public void remove(Long id) {
        userMapper.deleteById(id);
    }
}
```

分页结果类：

```java
package com.example.demo.dto;

import java.util.List;

public class PageResult<T> {
    private List<T> records;
    private long total;
    private int pageNumber;
    private int pageSize;

    public PageResult(List<T> records, long total, int pageNumber, int pageSize) {
        this.records = records;
        this.total = total;
        this.pageNumber = pageNumber;
        this.pageSize = pageSize;
    }

    public List<T> getRecords() {
        return records;
    }

    public long getTotal() {
        return total;
    }

    public int getPageNumber() {
        return pageNumber;
    }

    public int getPageSize() {
        return pageSize;
    }
}
```

### Controller 示例

```java
package com.example.demo.controller;

import com.example.demo.dto.PageResult;
import com.example.demo.entity.User;
import com.example.demo.service.UserService;
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
    public List<User> search(@RequestParam(required = false) String keyword,
                             @RequestParam(required = false) String status,
                             @RequestParam(required = false) Integer minAge) {
        return userService.search(keyword, status, minAge);
    }

    @GetMapping
    public PageResult<User> page(@RequestParam(required = false) String status,
                                 @RequestParam(defaultValue = "1") int pageNumber,
                                 @RequestParam(defaultValue = "10") int pageSize) {
        return userService.page(status, pageNumber, pageSize);
    }

    @PutMapping("/{id}/email")
    public void updateEmail(@PathVariable Long id, @RequestParam String email) {
        userService.updateEmail(id, email);
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
* 动态条件查询
* 分页查询
* 修改邮箱
* 删除用户

### 事务控制

`MyBatis` 通常和 Spring 事务一起使用。

业务层方法加 `@Transactional`：

```java
@Transactional
public Long create(User user) {
    userMapper.insert(user);
    logMapper.insert("创建用户：" + user.getUsername());
    return user.getId();
}
```

如果第二个插入失败，第一个插入也会回滚。

常见原则：

```text
Mapper 负责 SQL
Service 负责业务流程和事务边界
```

### Executor 类型

MyBatis 有几种执行器：

| 执行器 | 说明 |
| --- | --- |
| `SIMPLE` | 默认执行器，每次执行都创建新的 Statement |
| `REUSE` | 复用 PreparedStatement |
| `BATCH` | 批量执行更新语句 |

配置示例：

```yaml
mybatis:
  executor-type: batch
```

批量插入、批量更新较多时，可以考虑 `BATCH`。

不过批量模式下要关注事务大小、内存占用和异常处理。

### XML 和注解怎么选

| 场景 | 推荐方式 |
| --- | --- |
| 简单按 ID 查询 | 注解或 XML 都可以 |
| 简单新增、删除 | 注解或 XML 都可以 |
| 动态条件较多 | XML |
| 多表关联 | XML |
| 复杂 ResultMap | XML |
| 报表 SQL | XML |
| 临时小查询 | 注解 |

实际项目里常见组合是：

```text
主流程 SQL 写 XML
少量简单 SQL 用注解
```

### 和 JdbcTemplate、MyBatis-Plus、MyBatis-Flex 的区别

| 对比项 | JdbcTemplate | MyBatis | MyBatis-Plus | MyBatis-Flex |
| --- | --- | --- | --- | --- |
| SQL 控制 | 很直接 | 很直接 | 支持 XML 和 Wrapper | 支持 XML 和 QueryWrapper |
| 结果映射 | 手写 RowMapper | XML / 注解映射 | 自动 CRUD + 映射 | 自动 CRUD + 映射 |
| 单表 CRUD | 手写 | 手写 | 内置 | 内置 |
| 动态 SQL | 手写拼接 | XML 标签 | Wrapper / XML | QueryWrapper / XML |
| 分页 | 手写 | 手写或插件 | 分页插件 | 内置分页 |
| 适合场景 | 少量 SQL、工具类项目 | SQL 控制要求高 | 常规后台 CRUD | 轻量增强和灵活查询 |

粗略理解：

```text
JdbcTemplate 更接近 JDBC
MyBatis 更强调 SQL 映射
MyBatis-Plus 更强调通用 CRUD 和成熟生态
MyBatis-Flex 更强调轻量增强和链式查询
```

### 常见使用建议

### Mapper XML 路径要对齐

配置：

```yaml
mybatis:
  mapper-locations: classpath:mapper/*.xml
```

文件就应该放在：

```text
src/main/resources/mapper/
```

如果 XML 放在多级目录，可以配置：

```yaml
mybatis:
  mapper-locations: classpath*:mapper/**/*.xml
```

### namespace 和接口全限定名一致

XML：

```xml
<mapper namespace="com.example.demo.mapper.UserMapper">
```

接口：

```java
package com.example.demo.mapper;

public interface UserMapper {
}
```

两边不一致时，Mapper 方法找不到对应 SQL。

### 动态 SQL 优先用标签

动态条件不要直接字符串拼接。

更适合使用：

* `<where>`
* `<if>`
* `<choose>`
* `<foreach>`
* `<set>`
* `<trim>`

这样可以减少多余 `and`、多余逗号、空条件等问题。

### 普通参数优先使用 #{}

普通业务值使用：

```xml
#{value}
```

只有字段名、排序字段、表名这类 SQL 结构片段才可能用：

```xml
${value}
```

使用 `${}` 前要先做白名单校验。

### 复杂查询优先写 ResultMap

简单表可以靠自动映射。

复杂查询建议写清楚 `resultMap`。

尤其是：

* 多表 join
* 字段重名
* 一对多
* 一对一
* DTO 查询

清晰的 `resultMap` 比隐式映射更容易维护。

### 常用标签汇总

| 标签 | 作用 | 常见场景 |
| --- | --- | --- |
| `<select>` | 查询 SQL | 列表、详情、统计 |
| `<insert>` | 新增 SQL | 创建数据 |
| `<update>` | 修改 SQL | 更新字段、逻辑删除 |
| `<delete>` | 删除 SQL | 物理删除 |
| `<resultMap>` | 结果映射 | 复杂对象映射 |
| `<sql>` | SQL 片段 | 字段列表复用 |
| `<include>` | 引入 SQL 片段 | 复用字段列表 |
| `<where>` | 动态 where | 可选查询条件 |
| `<if>` | 条件判断 | 动态条件 |
| `<choose>` | 多分支判断 | 类似 if/else |
| `<foreach>` | 循环 | IN 查询、批量插入 |
| `<set>` | 动态 set | 动态更新 |
| `<trim>` | 自定义前后缀处理 | 复杂动态 SQL |

### 总结

`MyBatis` 的重点不是自动生成 SQL，而是让 SQL 和 Java 方法建立清晰关系。

最常见的开发流程是：

```text
建表
写 Entity
写 Mapper 接口
写 Mapper XML
写 Service
写 Controller
```

核心能力可以归纳为：

```text
SQL 自己控制
参数用 #{} 绑定
结果用 resultType 或 resultMap 映射
动态条件用 XML 标签组织
事务交给 Spring Service 层管理
复杂 SQL 保持在 XML 里
```

对于复杂业务查询、报表、性能优化要求较高的数据访问层，原生 `MyBatis` 依然是很稳妥的选择。
