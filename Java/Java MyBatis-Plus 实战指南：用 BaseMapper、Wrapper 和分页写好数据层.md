### 简介

`MyBatis-Plus` 是一个基于 `MyBatis` 的增强工具。

它经常被简称为 `MP`。

它的核心定位是：

```text
只做增强，不改变 MyBatis 原有能力。
```

普通 `MyBatis` 项目里，哪怕只是做一张表的增删改查，也经常要写：

```text
Mapper 接口
Mapper XML
insert SQL
delete SQL
update SQL
select SQL
分页 SQL
条件 SQL
```

`MyBatis-Plus` 把这些单表常规操作封装成了通用方法。

最常见的写法是：

```java
public interface UserMapper extends BaseMapper<User> {
}
```

继承 `BaseMapper<User>` 后，就可以直接使用：

```java
userMapper.insert(user);
userMapper.selectById(1L);
userMapper.updateById(user);
userMapper.deleteById(1L);
```

一句话概括：

```text
MyBatis-Plus 用来减少 MyBatis 项目里的重复 CRUD 代码，同时保留 XML、自定义 SQL、Mapper 扩展这些原生能力。
```

### MyBatis-Plus 适合什么场景

它适合这些数据访问层场景：

* 单表 CRUD 很多
* 后台管理系统列表页很多
* 查询条件经常动态组合
* 需要分页、逻辑删除、自动填充、乐观锁
* 项目已经在使用 MyBatis
* 复杂 SQL 仍然希望写 XML

大致可以这样理解：

```text
简单单表操作交给 BaseMapper
动态条件交给 Wrapper
分页交给分页插件
通用 Service 交给 IService
复杂 SQL 继续写 MyBatis XML
```

### 核心概念

| 名称 | 作用 |
| --- | --- |
| `@TableName` | 指定实体类对应的表名 |
| `@TableId` | 指定主键字段和主键策略 |
| `@TableField` | 指定字段映射、自动填充、忽略字段等 |
| `@TableLogic` | 指定逻辑删除字段 |
| `@Version` | 指定乐观锁版本字段 |
| `BaseMapper<T>` | Mapper 层通用 CRUD |
| `IService<T>` | Service 层通用方法接口 |
| `ServiceImpl<M, T>` | Service 层通用实现 |
| `QueryWrapper` | 字符串字段名形式的查询条件构造器 |
| `LambdaQueryWrapper` | Lambda 方法引用形式的查询条件构造器 |
| `UpdateWrapper` | 字符串字段名形式的更新条件构造器 |
| `LambdaUpdateWrapper` | Lambda 方法引用形式的更新条件构造器 |
| `Page<T>` | 分页参数和分页结果 |

最常用的组合是：

```text
@TableName + @TableId + BaseMapper + LambdaQueryWrapper + Page
```

### Maven 依赖

`MyBatis-Plus` 需要根据 `Spring Boot` 版本选择 starter。

`Spring Boot 2.x`：

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.15</version>
</dependency>
```

`Spring Boot 3.x`：

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
    <version>3.5.15</version>
</dependency>
```

`Spring Boot 4.x`：

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot4-starter</artifactId>
    <version>3.5.15</version>
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

从 `3.5.9` 开始，分页插件相关依赖被拆成可选模块。

如果需要使用分页插件，还需要引入：

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-jsqlparser</artifactId>
    <version>3.5.15</version>
</dependency>
```

如果项目仍然运行在 `JDK 8`，可以按官方说明选择 `mybatis-plus-jsqlparser-4.9`。

### 数据源配置

`application.yml` 示例：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mp_demo?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver

mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
  global-config:
    db-config:
      id-type: auto
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
```

几个常见配置：

* `map-underscore-to-camel-case`：下划线字段映射驼峰属性
* `log-impl`：开发环境打印 SQL
* `id-type`：全局主键策略
* `logic-delete-field`：全局逻辑删除字段

开发环境打开 SQL 日志很方便。

生产环境通常交给日志系统统一控制。

### 启动类配置

启动类加上 `@MapperScan`：

```java
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@MapperScan("com.example.demo.mapper")
public class MyBatisPlusDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyBatisPlusDemoApplication.class, args);
    }
}
```

也可以在每个 Mapper 接口上加 `@Mapper`。

Mapper 较多时，`@MapperScan` 更省事。

### 分页插件配置

使用分页功能时，需要配置 `MybatisPlusInterceptor`。

```java
import com.baomidou.mybatisplus.annotation.DbType;
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MyBatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```

如果同时配置多个插件，分页插件通常放在最后。

### 准备演示表

下面用用户表做示例。

```sql
DROP TABLE IF EXISTS sys_user;

CREATE TABLE sys_user (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL,
  age INT NOT NULL,
  status VARCHAR(20) NOT NULL,
  deleted TINYINT NOT NULL DEFAULT 0,
  version INT NOT NULL DEFAULT 0,
  create_time DATETIME NOT NULL,
  update_time DATETIME NULL
);

INSERT INTO sys_user (username, email, age, status, deleted, version, create_time, update_time) VALUES
('张三', 'zhangsan@example.com', 20, 'ACTIVE', 0, 0, '2026-01-01 10:00:00', NULL),
('李四', 'lisi@example.com', 25, 'ACTIVE', 0, 0, '2026-01-02 10:00:00', NULL),
('王五', 'wangwu@example.com', 17, 'DISABLED', 0, 0, '2026-01-03 10:00:00', NULL);
```

### 实体类

```java
package com.example.demo.entity;

import com.baomidou.mybatisplus.annotation.FieldFill;
import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableLogic;
import com.baomidou.mybatisplus.annotation.TableName;
import com.baomidou.mybatisplus.annotation.Version;

import java.time.LocalDateTime;

@TableName("sys_user")
public class User {

    @TableId(type = IdType.AUTO)
    private Long id;

    private String username;

    private String email;

    private Integer age;

    private String status;

    @TableLogic
    private Integer deleted;

    @Version
    private Integer version;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

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

    public Integer getDeleted() {
        return deleted;
    }

    public void setDeleted(Integer deleted) {
        this.deleted = deleted;
    }

    public Integer getVersion() {
        return version;
    }

    public void setVersion(Integer version) {
        this.version = version;
    }

    public LocalDateTime getCreateTime() {
        return createTime;
    }

    public void setCreateTime(LocalDateTime createTime) {
        this.createTime = createTime;
    }

    public LocalDateTime getUpdateTime() {
        return updateTime;
    }

    public void setUpdateTime(LocalDateTime updateTime) {
        this.updateTime = updateTime;
    }
}
```

几个重点：

* `@TableName("sys_user")`：指定表名
* `@TableId(type = IdType.AUTO)`：主键使用数据库自增
* `@TableLogic`：逻辑删除字段
* `@Version`：乐观锁字段
* `@TableField(fill = FieldFill.INSERT)`：插入时自动填充
* `@TableField(fill = FieldFill.INSERT_UPDATE)`：插入和更新时自动填充

### 主键策略

常见主键策略有这些：

| 策略 | 说明 | 常见场景 |
| --- | --- | --- |
| `IdType.AUTO` | 数据库自增 | 单库单表、传统业务表 |
| `IdType.ASSIGN_ID` | 雪花算法生成 ID | 分布式系统、业务主键 |
| `IdType.ASSIGN_UUID` | UUID 字符串 | 字符串主键 |
| `IdType.INPUT` | 手动传入主键 | 外部系统同步数据 |

如果数据库字段是 `AUTO_INCREMENT`，实体里通常写：

```java
@TableId(type = IdType.AUTO)
private Long id;
```

### 自动填充

`createTime`、`updateTime` 这类字段经常需要自动填充。

实体字段上先配置：

```java
@TableField(fill = FieldFill.INSERT)
private LocalDateTime createTime;

@TableField(fill = FieldFill.INSERT_UPDATE)
private LocalDateTime updateTime;
```

再实现 `MetaObjectHandler`：

```java
package com.example.demo.config;

import com.baomidou.mybatisplus.core.handlers.MetaObjectHandler;
import org.apache.ibatis.reflection.MetaObject;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;

@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }
}
```

插入时自动填充：

```text
create_time
update_time
```

更新时自动填充：

```text
update_time
```

### Mapper 接口

```java
package com.example.demo.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.example.demo.entity.User;

public interface UserMapper extends BaseMapper<User> {
}
```

继承 `BaseMapper<User>` 后，单表常用方法都可以直接使用。

### 新增数据

```java
User user = new User();
user.setUsername("赵六");
user.setEmail("zhaoliu@example.com");
user.setAge(28);
user.setStatus("ACTIVE");

userMapper.insert(user);

System.out.println(user.getId());
```

如果主键是自增，插入后会回填 `id`。

常见 SQL 大致是：

```sql
insert into sys_user (username, email, age, status, create_time, update_time)
values (?, ?, ?, ?, ?, ?)
```

### 根据 ID 查询

```java
User user = userMapper.selectById(1L);
```

如果配置了逻辑删除，查询会自动带上未删除条件。

### 查询全部

```java
List<User> users = userMapper.selectList(null);
```

`null` 表示没有额外查询条件。

业务表数据量较大时，不适合直接查全部。

更常见的是条件查询或分页查询。

### 批量查询

```java
List<Long> ids = Arrays.asList(1L, 2L, 3L);

List<User> users = userMapper.selectBatchIds(ids);
```

对应 SQL 大致是：

```sql
select id, username, email, age, status, deleted, version, create_time, update_time
from sys_user
where id in (?, ?, ?)
```

### 按 Map 查询

`selectByMap` 可以按字段和值做等值查询。

```java
Map<String, Object> params = new HashMap<>();
params.put("status", "ACTIVE");

List<User> users = userMapper.selectByMap(params);
```

注意这里的 key 是数据库字段名，不是 Java 属性名。

比如字段是：

```text
create_time
```

就写：

```java
params.put("create_time", value);
```

### 根据 ID 修改

```java
User user = new User();
user.setId(1L);
user.setEmail("new-zhangsan@example.com");

userMapper.updateById(user);
```

`updateById` 会根据主键更新。

未设置的字段通常不会参与更新。

### 根据 ID 删除

```java
userMapper.deleteById(1L);
```

如果没有配置逻辑删除，就是物理删除。

如果配置了 `@TableLogic`，会变成逻辑删除。

大致 SQL：

```sql
update sys_user
set deleted = 1
where id = ?
  and deleted = 0
```

逻辑删除后的数据，普通查询会自动过滤。

### BaseMapper 常用方法

| 方法 | 作用 |
| --- | --- |
| `insert(entity)` | 新增一条数据 |
| `deleteById(id)` | 按 ID 删除 |
| `delete(wrapper)` | 按条件删除 |
| `updateById(entity)` | 按 ID 更新 |
| `update(entity, wrapper)` | 按条件更新 |
| `selectById(id)` | 按 ID 查询 |
| `selectBatchIds(ids)` | 按 ID 集合查询 |
| `selectByMap(map)` | 按 Map 等值查询 |
| `selectOne(wrapper)` | 查询一条 |
| `selectList(wrapper)` | 查询列表 |
| `selectCount(wrapper)` | 查询数量 |
| `selectPage(page, wrapper)` | 分页查询 |

### Wrapper 是什么

`Wrapper` 用来构造 SQL 条件。

普通 SQL：

```sql
select *
from sys_user
where status = 'ACTIVE'
  and age >= 18
order by id desc
```

`LambdaQueryWrapper` 写法：

```java
LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();
wrapper.eq(User::getStatus, "ACTIVE")
       .ge(User::getAge, 18)
       .orderByDesc(User::getId);

List<User> users = userMapper.selectList(wrapper);
```

`LambdaQueryWrapper` 使用方法引用：

```java
User::getStatus
User::getAge
User::getId
```

这样字段改名时，编译期更容易发现问题。

### QueryWrapper 和 LambdaQueryWrapper

`QueryWrapper` 使用字符串字段名：

```java
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.eq("status", "ACTIVE")
       .ge("age", 18);
```

`LambdaQueryWrapper` 使用方法引用：

```java
LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();
wrapper.eq(User::getStatus, "ACTIVE")
       .ge(User::getAge, 18);
```

日常业务代码里，`LambdaQueryWrapper` 更常用。

原因是字段名不会写成字符串。

### 常见条件写法

等于：

```java
wrapper.eq(User::getStatus, "ACTIVE");
```

不等于：

```java
wrapper.ne(User::getStatus, "DISABLED");
```

大于：

```java
wrapper.gt(User::getAge, 18);
```

大于等于：

```java
wrapper.ge(User::getAge, 18);
```

小于：

```java
wrapper.lt(User::getAge, 60);
```

模糊查询：

```java
wrapper.like(User::getUsername, "张");
```

范围查询：

```java
wrapper.between(User::getAge, 18, 30);
```

`IN` 查询：

```java
wrapper.in(User::getId, Arrays.asList(1L, 2L, 3L));
```

排序：

```java
wrapper.orderByDesc(User::getId);
```

只查询部分字段：

```java
wrapper.select(User::getId, User::getUsername, User::getEmail);
```

### 条件参数

很多 Wrapper 方法都有一个 `condition` 参数。

```java
wrapper.eq(status != null, User::getStatus, status);
wrapper.like(keyword != null && !keyword.isBlank(), User::getUsername, keyword);
wrapper.ge(minAge != null, User::getAge, minAge);
```

含义是：

```text
condition 为 true，才拼接这个条件。
condition 为 false，跳过这个条件。
```

动态查询时很方便。

完整示例：

```java
public List<User> search(String keyword, String status, Integer minAge) {
    LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();

    wrapper.like(keyword != null && !keyword.isBlank(), User::getUsername, keyword)
           .eq(status != null && !status.isBlank(), User::getStatus, status)
           .ge(minAge != null, User::getAge, minAge)
           .orderByDesc(User::getId);

    return userMapper.selectList(wrapper);
}
```

### 查询单个对象

按唯一字段查询：

```java
public User findByEmail(String email) {
    LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();
    wrapper.eq(User::getEmail, email);

    return userMapper.selectOne(wrapper);
}
```

如果可能查不到，可以返回 `Optional`：

```java
public Optional<User> findOptionalByEmail(String email) {
    LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();
    wrapper.eq(User::getEmail, email);

    return Optional.ofNullable(userMapper.selectOne(wrapper));
}
```

`selectOne` 适合结果最多一条的场景。

如果实际查出多条，会出现结果数量异常。

### 查询数量

```java
LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();
wrapper.eq(User::getStatus, "ACTIVE");

Long count = userMapper.selectCount(wrapper);
```

返回值表示满足条件的数据条数。

### 分页查询

分页查询使用 `Page<T>`。

```java
Page<User> page = new Page<>(1, 10);

LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();
wrapper.eq(User::getStatus, "ACTIVE")
       .orderByDesc(User::getId);

Page<User> result = userMapper.selectPage(page, wrapper);
```

常用字段：

```java
List<User> records = result.getRecords();
long total = result.getTotal();
long current = result.getCurrent();
long size = result.getSize();
long pages = result.getPages();
```

含义：

```text
records：当前页数据
total：总条数
current：当前页
size：每页条数
pages：总页数
```

分页插件配置和 `mybatis-plus-jsqlparser` 依赖缺一不可。

### 不查总数的分页

有些列表只需要下一页，不需要总条数。

可以关闭 count 查询：

```java
Page<User> page = new Page<>(1, 10, false);

Page<User> result = userMapper.selectPage(page, wrapper);
```

第三个参数是：

```text
searchCount
```

设置为 `false` 后，不再查询总数。

### 条件更新

`LambdaUpdateWrapper` 可以按条件更新。

```java
LambdaUpdateWrapper<User> wrapper = Wrappers.lambdaUpdate();
wrapper.set(User::getStatus, "DISABLED")
       .eq(User::getStatus, "ACTIVE")
       .lt(User::getAge, 18);

userMapper.update(null, wrapper);
```

大致 SQL：

```sql
update sys_user
set status = ?
where status = ?
  and age < ?
  and deleted = 0
```

这种写法适合批量改状态、批量打标记。

### 条件删除

删除禁用状态用户：

```java
LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();
wrapper.eq(User::getStatus, "DISABLED");

userMapper.delete(wrapper);
```

如果配置了逻辑删除，会执行逻辑删除。

### IService 和 ServiceImpl

除了 `BaseMapper`，`MyBatis-Plus` 还提供了通用 Service。

Service 接口：

```java
package com.example.demo.service;

import com.baomidou.mybatisplus.extension.service.IService;
import com.example.demo.entity.User;

public interface UserService extends IService<User> {
}
```

Service 实现：

```java
package com.example.demo.service.impl;

import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.example.demo.entity.User;
import com.example.demo.mapper.UserMapper;
import com.example.demo.service.UserService;
import org.springframework.stereotype.Service;

@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
}
```

这样就能在 Service 层直接使用：

```java
userService.save(user);
userService.getById(1L);
userService.updateById(user);
userService.removeById(1L);
userService.list();
userService.page(new Page<>(1, 10));
```

`ServiceImpl` 里面已经持有 Mapper。

简单业务可以直接复用通用方法。

复杂业务继续在 Service 里写自定义方法。

### 完整实战 Demo：UserService

```java
package com.example.demo.service;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.core.conditions.update.LambdaUpdateWrapper;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.example.demo.entity.User;
import com.example.demo.mapper.UserMapper;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Optional;

@Service
public class UserService extends ServiceImpl<UserMapper, User> {

    @Transactional
    public Long create(User user) {
        user.setStatus("ACTIVE");

        save(user);

        return user.getId();
    }

    public Optional<User> findById(Long id) {
        return Optional.ofNullable(getById(id));
    }

    public Page<User> pageUsers(String keyword, String status, Integer minAge, long current, long size) {
        LambdaQueryWrapper<User> wrapper = Wrappers.lambdaQuery();

        wrapper.like(keyword != null && !keyword.isBlank(), User::getUsername, keyword)
               .eq(status != null && !status.isBlank(), User::getStatus, status)
               .ge(minAge != null, User::getAge, minAge)
               .orderByDesc(User::getId);

        return page(new Page<>(current, size), wrapper);
    }

    @Transactional
    public void updateEmail(Long id, String email) {
        User user = new User();
        user.setId(id);
        user.setEmail(email);

        updateById(user);
    }

    @Transactional
    public void disable(Long id) {
        LambdaUpdateWrapper<User> wrapper = Wrappers.lambdaUpdate();
        wrapper.set(User::getStatus, "DISABLED")
               .eq(User::getId, id);

        update(wrapper);
    }

    @Transactional
    public void remove(Long id) {
        removeById(id);
    }
}
```

这个 Service 包含：

* 新增用户
* 按 ID 查询
* 动态条件分页
* 修改邮箱
* 禁用用户
* 删除用户

### Controller 示例

```java
package com.example.demo.controller;

import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
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

    @GetMapping
    public Page<User> page(@RequestParam(required = false) String keyword,
                           @RequestParam(required = false) String status,
                           @RequestParam(required = false) Integer minAge,
                           @RequestParam(defaultValue = "1") long current,
                           @RequestParam(defaultValue = "10") long size) {
        return userService.pageUsers(keyword, status, minAge, current, size);
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

### 自定义 Mapper SQL

`MyBatis-Plus` 不影响原生 MyBatis。

Mapper 可以继续写自定义方法：

```java
package com.example.demo.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import com.example.demo.entity.User;
import org.apache.ibatis.annotations.Param;

public interface UserMapper extends BaseMapper<User> {

    Page<User> selectActiveUserPage(Page<User> page, @Param("keyword") String keyword);
}
```

XML：

```xml
<select id="selectActiveUserPage" resultType="com.example.demo.entity.User">
    select id, username, email, age, status, deleted, version, create_time, update_time
    from sys_user
    where deleted = 0
      and status = 'ACTIVE'
      and (
        username like concat('%', #{keyword}, '%')
        or email like concat('%', #{keyword}, '%')
      )
    order by id desc
</select>
```

调用：

```java
Page<User> page = new Page<>(1, 10);
Page<User> result = userMapper.selectActiveUserPage(page, "张");
```

分页插件会对自定义 SQL 生效。

### 逻辑删除

逻辑删除字段：

```java
@TableLogic
private Integer deleted;
```

全局配置：

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
```

调用删除：

```java
userMapper.deleteById(1L);
```

实际变成：

```sql
update sys_user
set deleted = 1
where id = ?
  and deleted = 0
```

普通查询会自动过滤：

```sql
deleted = 0
```

逻辑删除适合用户、订单、文章这类需要保留历史记录的数据。

### 乐观锁

乐观锁字段：

```java
@Version
private Integer version;
```

需要配置乐观锁插件：

```java
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.OptimisticLockerInnerInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MyBatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```

更新时会带上版本条件。

大致逻辑：

```text
where id = ? and version = ?
```

更新成功后，版本号增加。

如果影响行数为 `0`，说明数据已经被其他事务改过。

### 防止全表更新和删除

可以配置 `BlockAttackInnerInterceptor`。

```java
import com.baomidou.mybatisplus.extension.plugins.inner.BlockAttackInnerInterceptor;

@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
    return interceptor;
}
```

它可以拦截没有条件的全表更新和删除。

比如：

```sql
delete from sys_user
```

或：

```sql
update sys_user set status = 'DISABLED'
```

这类操作在业务系统里通常风险很高。

### 插件顺序

多个插件同时使用时，顺序需要注意。

常见建议：

```text
多租户、动态表名
分页、乐观锁
SQL 规范、防全表更新删除
```

分页插件一般放在靠后位置。

原因是分页需要基于前面已经改写后的 SQL 再处理。

### 和 MyBatis、JdbcTemplate、MyBatis-Flex 的区别

| 对比项 | JdbcTemplate | MyBatis | MyBatis-Plus | MyBatis-Flex |
| --- | --- | --- | --- | --- |
| SQL 控制 | 很直接 | 很直接 | 支持 XML 和 Wrapper | 支持 XML 和 QueryWrapper |
| 单表 CRUD | 手写 | 手写 | 内置 | 内置 |
| Service 封装 | 无 | 无 | `IService` | `IService` |
| Lambda 查询 | 无 | 无 | 支持 | 支持 APT 表定义 |
| 分页 | 手写或插件 | 插件 | 内置插件 | 内置分页 |
| 生态成熟度 | 简单稳定 | 成熟 | 成熟 | 较新 |
| 适合场景 | 少量 SQL、工具类项目 | SQL 控制要求高 | 常规后台 CRUD | 轻量增强、灵活查询 |

粗略理解：

```text
JdbcTemplate 更接近 JDBC
MyBatis 更强调 SQL 映射
MyBatis-Plus 更强调通用 CRUD 和成熟生态
MyBatis-Flex 更强调轻量和灵活查询构造
```

### 常见使用建议

### Mapper 层保持简单

Mapper 层适合放：

* `BaseMapper` 基础能力
* 少量自定义 SQL 方法
* 和数据库强相关的查询

业务流程、事务、跨表组合，更适合放在 Service 层。

### Lambda Wrapper 优先

相比字符串字段名：

```java
wrapper.eq("username", "张三");
```

Lambda 写法更容易维护：

```java
wrapper.eq(User::getUsername, "张三");
```

字段重命名后，编译器可以帮忙发现问题。

### 查询列表尽量带条件或分页

```java
userMapper.selectList(null);
```

这会查询所有未逻辑删除的数据。

对于业务表，数据量增长后很容易变慢。

更常见的做法：

```java
userMapper.selectList(wrapper);
userMapper.selectPage(page, wrapper);
```

### 复杂 SQL 回到 XML

`Wrapper` 适合中等复杂度条件查询。

如果 SQL 包含大量聚合、窗口函数、复杂子查询、多层动态条件，XML 通常更清楚。

`MyBatis-Plus` 不限制原生 MyBatis 写法。

### 批量操作注意分批

`saveBatch` 很方便，但数据量很大时仍然要拆批。

常见做法：

```text
每 500 条或 1000 条执行一次。
```

这样可以减少 SQL 太长、事务太大、锁持有时间过长等问题。

### 常用方法汇总

| 方法 | 作用 | 常见场景 |
| --- | --- | --- |
| `insert(entity)` | 新增数据 | 创建用户 |
| `deleteById(id)` | 按 ID 删除 | 删除单条记录 |
| `delete(wrapper)` | 按条件删除 | 批量删除、逻辑删除 |
| `updateById(entity)` | 按 ID 更新 | 修改单条记录 |
| `update(entity, wrapper)` | 按条件更新 | 批量改状态 |
| `selectById(id)` | 按 ID 查询 | 详情页 |
| `selectBatchIds(ids)` | 批量 ID 查询 | 批量加载 |
| `selectList(wrapper)` | 条件列表查询 | 列表页 |
| `selectOne(wrapper)` | 查询单条 | 唯一字段查询 |
| `selectCount(wrapper)` | 查询数量 | 统计 |
| `selectPage(page, wrapper)` | 分页查询 | 后台列表 |
| `save(entity)` | Service 新增 | 业务层新增 |
| `saveBatch(list)` | Service 批量新增 | 批量导入 |
| `page(page, wrapper)` | Service 分页 | 分页接口 |
| `removeById(id)` | Service 删除 | 删除接口 |

### 总结

`MyBatis-Plus` 的重点不是替代 `MyBatis`，而是把常规 CRUD 和条件查询封装得更顺手。

落地时抓住这条线就够了：

```text
实体类用 @TableName、@TableId、@TableLogic
Mapper 继承 BaseMapper
查询条件用 LambdaQueryWrapper
分页配置 MybatisPlusInterceptor
Service 复用 IService 和 ServiceImpl
复杂 SQL 继续写 XML
```

它适合后台管理系统、业务中台、内部系统、常规 CRUD 较多的项目。

只要控制好 Wrapper 和 XML 的边界，数据访问层会比纯 MyBatis 少很多重复代码。
