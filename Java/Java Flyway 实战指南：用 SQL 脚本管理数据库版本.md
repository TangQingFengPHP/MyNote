### 简介

`Flyway` 是一个数据库迁移工具。

它解决的问题和 `Liquibase` 类似：

```text
数据库结构怎么跟着项目版本一起演进。
```

不过 `Flyway` 的风格更简单直接。

它主要通过 SQL 文件管理数据库变更。

比如：

```text
V1__create_users_table.sql
V2__add_user_email_column.sql
V3__create_orders_table.sql
V4__insert_init_data.sql
```

应用启动或命令执行时，`Flyway` 会检查哪些脚本已经执行过，哪些还没执行，然后按版本顺序执行新的脚本。

一句话概括：

```text
Flyway 用 SQL 文件管理数据库版本，让表结构、索引、初始化数据跟着代码一起提交、发布和追踪。
```

### Flyway 适合什么场景

常见场景有这些：

* Spring Boot 项目需要初始化数据库结构
* 团队希望直接用 SQL 管理表结构
* 多个环境需要保持数据库结构一致
* 发布时需要自动执行数据库变更
* 数据库变更需要纳入 Git 管理
* 不希望生产环境使用 Hibernate `ddl-auto: update`
* 项目不需要 YAML/XML 这种抽象迁移格式

如果团队本来就习惯写 SQL，`Flyway` 上手成本很低。

### Flyway 和 ORM 的关系

`Flyway` 不是 ORM。

它不负责：

* 查询数据库
* 保存 Java 对象
* 映射实体关系
* 生成业务 SQL

这些事情通常交给：

* `JdbcTemplate`
* `MyBatis`
* `MyBatis-Plus`
* `Spring Data JPA`

`Flyway` 只负责数据库结构和初始化数据的迁移。

常见搭配是：

```text
Flyway 管表结构
JPA / MyBatis / JdbcTemplate 管业务读写
```

### 工作原理

`Flyway` 的核心流程：

```text
扫描迁移脚本目录
  |
  v
检查 flyway_schema_history 表
  |
  v
找出未执行脚本
  |
  v
按版本顺序执行
  |
  v
记录执行结果和 checksum
```

第一次运行时，`Flyway` 会创建一张历史表。

默认表名是：

```text
flyway_schema_history
```

这张表记录：

```text
脚本版本
脚本描述
脚本文件名
checksum
执行时间
执行耗时
执行结果
```

后续启动时，`Flyway` 会通过这张表判断脚本是否执行过。

### Maven 依赖

Spring Boot 项目中，核心依赖是：

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

如果使用 MySQL，还需要数据库支持模块：

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
</dependency>
```

MySQL 驱动：

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

如果项目使用 JDBC：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

如果项目使用 JPA：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

Spring Boot 检测到 `flyway-core` 后，会自动配置 Flyway，并在应用启动时执行迁移。

### Spring Boot 配置

`application.yml` 示例：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/flyway_demo?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver

  flyway:
    enabled: true
    locations: classpath:db/migration
    validate-on-migrate: true
    out-of-order: false
```

常见配置：

| 配置 | 作用 |
| --- | --- |
| `spring.flyway.enabled` | 是否启用 Flyway |
| `spring.flyway.locations` | 迁移脚本目录 |
| `spring.flyway.table` | 历史表名 |
| `spring.flyway.baseline-on-migrate` | 非空库首次接入时是否自动 baseline |
| `spring.flyway.baseline-version` | baseline 版本 |
| `spring.flyway.validate-on-migrate` | 迁移前是否校验脚本 |
| `spring.flyway.out-of-order` | 是否允许乱序迁移 |
| `spring.flyway.clean-disabled` | 是否禁用 clean |

默认迁移目录是：

```text
classpath:db/migration
```

对应项目路径：

```text
src/main/resources/db/migration
```

### 推荐目录结构

```text
src/main/resources/
└── db/
    └── migration/
        ├── V1__create_users_table.sql
        ├── V2__create_orders_table.sql
        ├── V3__insert_init_data.sql
        ├── V4__add_user_status_column.sql
        └── R__create_user_order_summary_view.sql
```

`V` 开头的是版本迁移。

`R` 开头的是重复迁移。

### 文件命名规则

版本迁移格式：

```text
V版本号__描述.sql
```

注意中间是两个下划线：

```text
__
```

示例：

```text
V1__create_users_table.sql
V2__create_orders_table.sql
V3__insert_init_data.sql
V4__add_user_status_column.sql
```

也可以使用小版本：

```text
V1.0.0__init_schema.sql
V1.0.1__add_user_table.sql
V1.1.0__create_order_table.sql
```

多人协作时，也可以使用时间戳版本：

```text
V202606070001__create_users_table.sql
V202606070002__create_orders_table.sql
V202606070003__add_user_status_column.sql
```

时间戳版本不容易和其他分支撞版本号。

### 第一个迁移脚本

`V1__create_users_table.sql`：

```sql
create table users (
    id bigint primary key auto_increment,
    username varchar(50) not null,
    email varchar(100) not null,
    age int not null,
    created_at datetime not null,
    constraint uk_users_email unique (email)
) engine = InnoDB default charset = utf8mb4;
```

启动 Spring Boot 后，`Flyway` 会执行这个脚本。

执行成功后，`flyway_schema_history` 里会记录：

```text
version: 1
description: create users table
script: V1__create_users_table.sql
success: true
```

再次启动应用时，这个脚本不会重复执行。

### 创建订单表

`V2__create_orders_table.sql`：

```sql
create table orders (
    id bigint primary key auto_increment,
    user_id bigint not null,
    order_no varchar(50) not null,
    amount decimal(10, 2) not null,
    status varchar(20) not null,
    created_at datetime not null,
    index idx_orders_user_id (user_id),
    constraint fk_orders_user_id foreign key (user_id) references users (id)
) engine = InnoDB default charset = utf8mb4;
```

这类脚本适合放结构变更。

例如：

* 建表
* 新增字段
* 创建索引
* 修改字段类型
* 创建约束

### 插入初始化数据

`V3__insert_init_data.sql`：

```sql
insert into users (username, email, age, created_at)
values
('张三', 'zhangsan@example.com', 20, '2026-01-01 10:00:00'),
('李四', 'lisi@example.com', 25, '2026-01-02 10:00:00');

insert into orders (user_id, order_no, amount, status, created_at)
values
(1, 'A001', 99.00, 'PAID', '2026-02-01 10:00:00'),
(1, 'A002', 260.00, 'PAID', '2026-02-02 10:00:00');
```

初始化数据也可以交给 Flyway。

但测试数据和生产数据要区分。

如果只是开发环境用的测试数据，可以放到单独目录，再通过 profile 控制执行。

### 新增字段

`V4__add_user_status_column.sql`：

```sql
alter table users
add column status varchar(20) not null default 'ACTIVE';
```

已有大表加非空字段时要谨慎。

常见拆分方式：

```text
先加可空字段
分批回填数据
再加非空约束
```

Flyway 负责记录每一步。

具体执行策略要结合表数据量和业务窗口。

### 重复迁移

重复迁移文件以 `R__` 开头。

示例：

```text
R__create_user_order_summary_view.sql
```

`R__create_user_order_summary_view.sql`：

```sql
create or replace view v_user_order_summary as
select
    u.id as user_id,
    u.username,
    count(o.id) as order_count,
    coalesce(sum(o.amount), 0) as total_amount
from users u
left join orders o on o.user_id = u.id
group by u.id, u.username;
```

重复迁移的特点：

```text
没有版本号
脚本 checksum 改变时会重新执行
版本迁移执行完后再执行
按描述排序执行
```

它适合管理：

* 视图
* 存储过程
* 函数
* 触发器
* 可重复刷新的参考数据

重复迁移脚本最好写成可重复执行。

例如：

```sql
create or replace view ...
```

而不是：

```sql
create view ...
```

### schema history 表

默认历史表名是：

```text
flyway_schema_history
```

可以配置：

```yaml
spring:
  flyway:
    table: flyway_schema_history
```

这张表很重要。

它记录了哪些脚本执行过。

常见字段有：

```text
installed_rank
version
description
type
script
checksum
installed_by
installed_on
execution_time
success
```

`checksum` 用来检测已执行脚本是否被修改。

如果某个已执行的 `V1__create_users_table.sql` 后来被改了，Flyway 校验时会发现 checksum 不一致，并阻止迁移继续执行。

### 已执行脚本保持稳定

版本迁移脚本执行后，不建议直接修改。

比如 `V1__create_users_table.sql` 已经在测试环境或生产环境执行过。

此时发现少了一个字段，更合适的方式是新增脚本：

```text
V5__add_user_phone_column.sql
```

内容：

```sql
alter table users
add column phone varchar(30);
```

这样所有环境都能按同样顺序执行。

### baseline

`baseline` 用来把一个已经存在的数据库接入 Flyway。

比如数据库里已经有很多表，但还没有 `flyway_schema_history` 表。

如果直接启用 Flyway，可能会出现非空 schema 没有历史表的问题。

配置：

```yaml
spring:
  flyway:
    baseline-on-migrate: true
    baseline-version: 1
```

含义是：

```text
第一次迁移时，把当前数据库标记为 baseline version 1。
```

后续只执行高于 baseline 的版本脚本。

例如：

```text
V1__init_existing_schema.sql
V2__add_user_status_column.sql
```

baseline 为 `1` 时，`V1` 不会执行，`V2` 会继续执行。

已有库接入时，baseline 很有用。

新项目空库一般不需要打开 `baseline-on-migrate`。

### validate

`validate` 用来校验迁移脚本和历史记录是否一致。

常见校验内容：

* 已执行脚本是否还存在
* 已执行脚本 checksum 是否变化
* 是否存在版本冲突
* 是否存在未按规则命名的脚本

Spring Boot 默认迁移时通常会进行校验。

也可以显式配置：

```yaml
spring:
  flyway:
    validate-on-migrate: true
```

手动执行：

```bash
flyway validate
```

### repair

`repair` 用来修复 schema history 表里的某些状态。

常见场景：

* 开发环境里脚本 checksum 改过，需要同步历史表
* 删除了已经不再存在的失败记录
* 修复已删除迁移脚本对应的记录状态

命令：

```bash
flyway repair
```

`repair` 不会自动修改业务表结构。

它主要修复的是 `flyway_schema_history`。

生产环境使用前需要先确认原因和影响。

### clean

`clean` 会删除 schema 里的数据库对象。

包括：

* 表
* 视图
* 存储过程
* 函数
* 触发器

命令：

```bash
flyway clean
```

它适合本地开发、集成测试里快速重置数据库。

生产环境通常会禁用：

```yaml
spring:
  flyway:
    clean-disabled: true
```

### out-of-order

默认情况下，Flyway 按版本顺序执行。

如果数据库已经执行到 `V5`，后来又出现一个 `V4`，默认不会继续执行这个较低版本。

配置：

```yaml
spring:
  flyway:
    out-of-order: false
```

多人协作时，建议提前统一版本号规则。

常见方案：

```text
使用时间戳版本号
每个分支合并前检查 migration 文件
发布前统一整理版本顺序
```

### 常用命令

如果使用 Flyway CLI，常见命令如下。

执行迁移：

```bash
flyway migrate
```

查看状态：

```bash
flyway info
```

校验：

```bash
flyway validate
```

修复历史表：

```bash
flyway repair
```

建立基线：

```bash
flyway baseline
```

清理数据库：

```bash
flyway clean
```

### Maven 插件

可以使用 Maven 插件执行 Flyway 命令。

```xml
<plugin>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-maven-plugin</artifactId>
    <configuration>
        <url>jdbc:mysql://localhost:3306/flyway_demo</url>
        <user>root</user>
        <password>123456</password>
    </configuration>
</plugin>
```

常用命令：

```bash
mvn flyway:migrate
mvn flyway:info
mvn flyway:validate
mvn flyway:repair
mvn flyway:baseline
mvn flyway:clean
```

Spring Boot 启动自动迁移和 Maven/CLI 迁移二选一即可。

团队规模较大时，迁移经常放到 CI/CD 流水线里执行。

### 配合 Spring Data JPA

如果项目使用 JPA，建议让 Flyway 管表结构。

JPA 只做校验：

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```

实体类：

```java
package com.example.demo.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

import java.time.LocalDateTime;

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

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;

    // getter setter
}
```

Repository：

```java
package com.example.demo.repository;

import com.example.demo.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}
```

Service：

```java
package com.example.demo.service;

import com.example.demo.entity.User;
import com.example.demo.repository.UserRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Transactional
    public Long create(String username, String email, Integer age) {
        User user = new User();
        user.setUsername(username);
        user.setEmail(email);
        user.setAge(age);
        user.setStatus("ACTIVE");
        user.setCreatedAt(LocalDateTime.now());

        User saved = userRepository.save(user);
        return saved.getId();
    }
}
```

这里的分工是：

```text
Flyway 创建 users 表
JPA 实体映射 users 表
Repository 负责业务读写
```

### 多环境脚本

可以按环境配置不同目录。

开发环境：

```yaml
spring:
  flyway:
    locations: classpath:db/migration,classpath:db/dev
```

生产环境：

```yaml
spring:
  flyway:
    locations: classpath:db/migration
```

目录：

```text
src/main/resources/
└── db/
    ├── migration/
    │   ├── V1__create_users_table.sql
    │   └── V2__create_orders_table.sql
    └── dev/
        └── V1000__insert_dev_test_data.sql
```

这样开发测试数据不会进入生产环境。

### Java Migration

除了 SQL，Flyway 也支持 Java 迁移。

适合这些场景：

* 复杂数据转换
* 需要调用 Java 逻辑
* 处理大字段或文件
* SQL 很难表达的迁移

示例：

```java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;

import java.sql.PreparedStatement;

public class V5__normalize_user_email extends BaseJavaMigration {

    @Override
    public void migrate(Context context) throws Exception {
        try (PreparedStatement statement = context.getConnection()
                .prepareStatement("update users set email = lower(email)")) {
            statement.executeUpdate();
        }
    }
}
```

类名也遵守版本迁移命名规则：

```text
V5__normalize_user_email
```

常规 DDL 优先用 SQL。

复杂数据转换再考虑 Java Migration。

### 和 Liquibase 的区别

| 对比项 | Flyway | Liquibase |
| --- | --- | --- |
| 主要格式 | SQL | YAML、XML、JSON、SQL |
| 上手成本 | 较低 | 中等 |
| 变更模型 | 按版本脚本执行 | changeSet 模型 |
| 回滚 | 常见做法是写新迁移向前修复 | 支持 rollback 定义 |
| 适合场景 | SQL 优先、简单直接 | 复杂流程、多格式、强元数据 |
| 历史表 | `flyway_schema_history` | `DATABASECHANGELOG`、`DATABASECHANGELOGLOCK` |

粗略理解：

```text
Flyway 更像按顺序执行 SQL 文件
Liquibase 更像用 changeSet 描述数据库变更
```

如果项目以 SQL 为主，`Flyway` 很顺手。

如果需要 YAML/XML、precondition、context、label、rollback 等能力，`Liquibase` 更合适。

### 常见使用建议

### 已执行的版本脚本保持稳定

版本迁移脚本执行后，尽量保持稳定。

如果线上已经执行：

```text
V1__create_users_table.sql
```

后续需要加字段，就新增：

```text
V2__add_user_status_column.sql
```

这样所有环境都能按同样顺序演进。

### 版本号规则提前统一

小项目可以使用：

```text
V1
V2
V3
```

多人协作项目更适合：

```text
V202606070001
V202606070002
V202606070003
```

版本号冲突会少很多。

### 生产环境禁用 clean

`clean` 会删除数据库对象。

生产环境建议配置：

```yaml
spring:
  flyway:
    clean-disabled: true
```

本地环境可以按需打开。

### 先在测试库验证迁移

发布前建议至少执行：

```bash
flyway validate
flyway migrate
```

测试库通过后，再进入生产发布流程。

如果迁移涉及大表，还需要评估锁表时间、执行耗时和回滚方案。

### 初始化数据保持可重复

版本迁移只会执行一次。

如果初始化数据未来可能调整，可以考虑：

```text
使用新的 V 脚本修正数据
或把视图、函数、配置刷新放到 R 脚本
```

重复迁移适合可重复执行的对象。

### 常用配置汇总

| 配置 | 作用 |
| --- | --- |
| `spring.flyway.enabled` | 是否启用 |
| `spring.flyway.locations` | 脚本目录 |
| `spring.flyway.table` | 历史表名 |
| `spring.flyway.baseline-on-migrate` | 非空库是否自动 baseline |
| `spring.flyway.baseline-version` | baseline 版本 |
| `spring.flyway.validate-on-migrate` | 迁移前是否校验 |
| `spring.flyway.out-of-order` | 是否允许乱序执行 |
| `spring.flyway.clean-disabled` | 是否禁用 clean |
| `spring.flyway.schemas` | 指定 schema |
| `spring.flyway.default-schema` | 默认 schema |

### 常用命令汇总

| 命令 | 作用 |
| --- | --- |
| `migrate` | 执行未执行迁移 |
| `info` | 查看迁移状态 |
| `validate` | 校验脚本和历史记录 |
| `repair` | 修复历史表状态 |
| `baseline` | 建立基线 |
| `clean` | 清理数据库对象 |

### 总结

`Flyway` 的核心非常简单：

```text
把数据库变更写成 SQL 文件
按 V 版本号排序执行
用 flyway_schema_history 记录执行历史
用 checksum 防止已执行脚本被悄悄修改
用 R 脚本管理可重复刷新的对象
```

它适合这些场景：

* 团队偏好直接写 SQL
* 数据库变更希望纳入 Git
* Spring Boot 启动时自动迁移
* 多环境结构需要保持一致
* 不希望 JPA 自动改表
* 希望工具简单、规则清楚、维护成本低

落地时重点关注：

* 脚本命名规范
* 版本号规则
* 已执行脚本保持稳定
* baseline 的使用边界
* clean 的环境隔离
* 大表迁移的执行窗口

掌握这些内容后，`Flyway` 已经可以覆盖大多数 Java 项目的数据库版本管理需求。
