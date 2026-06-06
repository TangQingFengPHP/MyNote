### 简介

`Liquibase` 是一个数据库变更管理工具。

它解决的问题不是：

```text
Java 怎么查询数据库
SQL 怎么映射成对象
```

而是：

```text
数据库结构怎么跟着项目一起演进。
```

比如项目从第一个版本到第三个版本，数据库可能经历这些变化：

```text
V1：创建用户表
V2：给用户表增加 email 字段
V3：创建订单表
V4：给订单表增加索引
V5：初始化系统配置数据
```

如果靠手动执行 SQL，很容易出现这些问题：

```text
开发库执行了，测试库没执行
测试库执行了，生产库漏了一条
同一条 SQL 被重复执行
某个字段是谁加的、什么时候加的，查不清楚
回滚时不知道该撤哪几条 SQL
```

`Liquibase` 的做法是把数据库变更写成文件，并纳入 Git 管理。

应用启动或命令执行时，`Liquibase` 会检查哪些变更已经执行过，哪些还没执行，然后只执行未执行的部分。

一句话概括：

```text
Liquibase 用来管理数据库变更版本，让表结构、索引、初始化数据可以像代码一样被记录、评审、执行和回滚。
```

### Liquibase 适合什么场景

常见场景有这些：

* Spring Boot 项目需要自动初始化数据库结构
* 多个环境需要保持表结构一致
* 团队协作开发，数据库变更需要走 Git 评审
* 发布时需要知道数据库变更执行到哪一步
* 表结构变化需要可追踪、可回滚
* 不希望生产环境依赖 Hibernate `ddl-auto: update`
* 需要同时管理 DDL 和少量初始化数据

如果项目里已经有这些文件：

```text
001_create_user_table.sql
002_add_email_column.sql
003_create_order_table.sql
```

`Liquibase` 就是把这套脚本管理得更标准、更可追踪。

### Liquibase 和 ORM 的区别

`Liquibase` 不是 ORM。

它不负责：

* 把 Java 对象保存到数据库
* 把查询结果映射成实体
* 生成业务查询 SQL
* 管理 Repository 或 Mapper

这些事情通常由：

* `JdbcTemplate`
* `MyBatis`
* `MyBatis-Plus`
* `Spring Data JPA`

来完成。

`Liquibase` 负责的是数据库结构变更。

常见搭配方式：

```text
Liquibase 管表结构
MyBatis / JPA / JdbcTemplate 管业务读写
```

### 工作原理

`Liquibase` 的核心流程大致是：

```text
读取 changelog 文件
  |
  v
检查 DATABASECHANGELOG 表
  |
  v
找出还没执行的 changeSet
  |
  v
获取 DATABASECHANGELOGLOCK 锁
  |
  v
执行变更
  |
  v
记录执行历史
```

第一次运行时，`Liquibase` 会自动创建两张表。

### DATABASECHANGELOG

这张表记录已经执行过的变更。

常见字段有：

```text
ID
AUTHOR
FILENAME
DATEEXECUTED
ORDEREXECUTED
EXECTYPE
MD5SUM
DESCRIPTION
COMMENTS
TAG
LIQUIBASE
CONTEXTS
LABELS
```

`Liquibase` 会根据这些信息判断某个 `changeSet` 是否已经执行过。

### DATABASECHANGELOGLOCK

这张表用来加锁。

作用是防止多个应用实例同时执行数据库迁移。

在多实例部署时，这一点很重要。

### changeLog 和 changeSet

`changeLog` 是变更日志文件。

它可以包含很多个 `changeSet`。

`changeSet` 是最小执行单元。

例如：

```yaml
databaseChangeLog:
  - changeSet:
      id: 001-create-user-table
      author: feng
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: bigint
                  autoIncrement: true
                  constraints:
                    primaryKey: true
                    nullable: false
```

其中：

```text
id + author + 文件路径
```

共同定位一个 `changeSet`。

已经执行过的 `changeSet` 会记录到 `DATABASECHANGELOG`，后续不会重复执行。

### Maven 依赖

Spring Boot 项目里引入 `liquibase-core`。

```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
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

数据库驱动以 `MySQL` 为例：

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

Spring Boot 会根据 classpath 里的 `liquibase-core` 自动配置 `SpringLiquibase`。

启动时会自动执行未执行的数据库变更。

### Spring Boot 配置

`application.yml` 示例：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/liquibase_demo?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver

  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.yaml
    contexts: dev
```

常见配置：

| 配置 | 作用 |
| --- | --- |
| `spring.liquibase.enabled` | 是否启用 Liquibase |
| `spring.liquibase.change-log` | 主 changelog 文件位置 |
| `spring.liquibase.contexts` | 指定执行哪些 context |
| `spring.liquibase.default-schema` | 默认 schema |
| `spring.liquibase.drop-first` | 执行前是否先删除对象 |
| `spring.liquibase.rollback-file` | 输出 rollback SQL 文件 |

`drop-first` 只适合极少数临时测试场景。

常规项目里通常保持默认关闭。

如果同时使用 JPA，生产环境一般这样配置：

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: none
```

表结构交给 `Liquibase` 管理，避免 ORM 自动改表和 changelog 冲突。

### 推荐目录结构

一种常见目录结构：

```text
src/main/resources/
└── db/
    └── changelog/
        ├── db.changelog-master.yaml
        ├── v1.0.0/
        │   ├── 001-create-users-table.yaml
        │   ├── 002-create-orders-table.yaml
        │   └── 003-init-user-data.yaml
        ├── v1.1.0/
        │   └── 001-add-user-status-column.yaml
        └── sql/
            └── 001-create-report-view.sql
```

主文件只负责组织。

具体变更放到版本目录里。

这样发布历史会比较清楚。

### 主 changelog

`db.changelog-master.yaml`：

```yaml
databaseChangeLog:
  - include:
      file: db/changelog/v1.0.0/001-create-users-table.yaml
  - include:
      file: db/changelog/v1.0.0/002-create-orders-table.yaml
  - include:
      file: db/changelog/v1.0.0/003-init-user-data.yaml
  - include:
      file: db/changelog/v1.1.0/001-add-user-status-column.yaml
```

也可以使用 `includeAll`：

```yaml
databaseChangeLog:
  - includeAll:
      path: db/changelog/v1.0.0/
```

`include` 的顺序更明确。

`includeAll` 更省事，但要注意文件名排序。

实际项目里，很多团队会选择：

```text
主文件 include 版本文件
版本文件 include 具体 changeSet 文件
```

### 创建用户表

`v1.0.0/001-create-users-table.yaml`：

```yaml
databaseChangeLog:
  - changeSet:
      id: 001-create-users-table
      author: feng
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: bigint
                  autoIncrement: true
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: username
                  type: varchar(50)
                  constraints:
                    nullable: false
              - column:
                  name: email
                  type: varchar(100)
                  constraints:
                    nullable: false
              - column:
                  name: age
                  type: int
                  constraints:
                    nullable: false
              - column:
                  name: created_at
                  type: datetime
                  constraints:
                    nullable: false
        - addUniqueConstraint:
            tableName: users
            columnNames: email
            constraintName: uk_users_email
      rollback:
        - dropTable:
            tableName: users
```

这个 `changeSet` 做了两件事：

* 创建 `users` 表
* 给 `email` 加唯一约束

回滚时删除 `users` 表。

### 创建订单表

`v1.0.0/002-create-orders-table.yaml`：

```yaml
databaseChangeLog:
  - changeSet:
      id: 002-create-orders-table
      author: feng
      changes:
        - createTable:
            tableName: orders
            columns:
              - column:
                  name: id
                  type: bigint
                  autoIncrement: true
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: user_id
                  type: bigint
                  constraints:
                    nullable: false
              - column:
                  name: order_no
                  type: varchar(50)
                  constraints:
                    nullable: false
              - column:
                  name: amount
                  type: decimal(10,2)
                  constraints:
                    nullable: false
              - column:
                  name: status
                  type: varchar(20)
                  constraints:
                    nullable: false
              - column:
                  name: created_at
                  type: datetime
                  constraints:
                    nullable: false
        - createIndex:
            tableName: orders
            indexName: idx_orders_user_id
            columns:
              - column:
                  name: user_id
        - addForeignKeyConstraint:
            baseTableName: orders
            baseColumnNames: user_id
            referencedTableName: users
            referencedColumnNames: id
            constraintName: fk_orders_user_id
      rollback:
        - dropTable:
            tableName: orders
```

这里包含：

* 建表
* 创建索引
* 创建外键

实际项目里，如果外键由应用层保证，也可以不创建数据库外键，只保留索引。

### 初始化数据

`v1.0.0/003-init-user-data.yaml`：

```yaml
databaseChangeLog:
  - changeSet:
      id: 003-init-user-data
      author: feng
      context: dev,test
      changes:
        - insert:
            tableName: users
            columns:
              - column:
                  name: username
                  value: 张三
              - column:
                  name: email
                  value: zhangsan@example.com
              - column:
                  name: age
                  valueNumeric: 20
              - column:
                  name: created_at
                  valueDate: 2026-01-01T10:00:00
        - insert:
            tableName: users
            columns:
              - column:
                  name: username
                  value: 李四
              - column:
                  name: email
                  value: lisi@example.com
              - column:
                  name: age
                  valueNumeric: 25
              - column:
                  name: created_at
                  valueDate: 2026-01-02T10:00:00
      rollback:
        - delete:
            tableName: users
            where: email in ('zhangsan@example.com', 'lisi@example.com')
```

这里用了：

```yaml
context: dev,test
```

表示这批数据只在 `dev` 和 `test` 环境执行。

如果生产环境配置：

```yaml
spring:
  liquibase:
    contexts: prod
```

这组测试数据不会执行。

### 新增字段

`v1.1.0/001-add-user-status-column.yaml`：

```yaml
databaseChangeLog:
  - changeSet:
      id: 001-add-user-status-column
      author: feng
      changes:
        - addColumn:
            tableName: users
            columns:
              - column:
                  name: status
                  type: varchar(20)
                  defaultValue: ACTIVE
                  constraints:
                    nullable: false
      rollback:
        - dropColumn:
            tableName: users
            columnName: status
```

这类变更很常见：

```text
已有表增加一个业务字段。
```

如果表数据很多，新增非空字段时要谨慎。

常见做法是拆成几步：

```text
先加可空字段
回填数据
再加非空约束
```

### 使用 SQL 格式

有些场景直接写 SQL 更直观。

例如创建视图、函数、复杂索引。

`db/changelog/sql/001-create-report-view.sql`：

```sql
-- liquibase formatted sql

-- changeset feng:001-create-report-view
create view v_user_order_summary as
select
  u.id as user_id,
  u.username,
  count(o.id) as order_count,
  coalesce(sum(o.amount), 0) as total_amount
from users u
left join orders o on o.user_id = u.id
group by u.id, u.username;

-- rollback drop view v_user_order_summary;
```

主 changelog 引入：

```yaml
databaseChangeLog:
  - include:
      file: db/changelog/sql/001-create-report-view.sql
```

SQL 格式适合：

* 视图
* 存储过程
* 函数
* 触发器
* 数据库特有语法
* DBA 提供的变更脚本

### 使用 XML 格式

XML 格式也很常见。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
            http://www.liquibase.org/xml/ns/dbchangelog
            https://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <changeSet id="001-create-product-table" author="feng">
        <createTable tableName="products">
            <column name="id" type="bigint" autoIncrement="true">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="name" type="varchar(100)">
                <constraints nullable="false"/>
            </column>
            <column name="price" type="decimal(10,2)">
                <constraints nullable="false"/>
            </column>
        </createTable>
        <rollback>
            <dropTable tableName="products"/>
        </rollback>
    </changeSet>
</databaseChangeLog>
```

YAML、XML、JSON、SQL 都可以使用。

项目里最好统一主格式。

复杂 SQL 再单独使用 formatted SQL 文件。

### preConditions 前置条件

`preConditions` 可以在执行前检查数据库状态。

例如表不存在时才创建：

```yaml
databaseChangeLog:
  - changeSet:
      id: 004-create-config-table
      author: feng
      preConditions:
        - onFail: MARK_RAN
        - not:
            - tableExists:
                tableName: app_config
      changes:
        - createTable:
            tableName: app_config
            columns:
              - column:
                  name: id
                  type: bigint
                  autoIncrement: true
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: config_key
                  type: varchar(100)
                  constraints:
                    nullable: false
              - column:
                  name: config_value
                  type: varchar(500)
      rollback:
        - dropTable:
            tableName: app_config
```

`onFail: MARK_RAN` 的含义是：

```text
条件不满足时，把这个 changeSet 标记为已执行。
```

常见处理方式还有：

* `HALT`：停止执行
* `WARN`：输出警告
* `CONTINUE`：继续执行
* `MARK_RAN`：标记为已执行

### context 和 label

`context` 常用于区分环境。

```yaml
changeSet:
  id: init-dev-data
  author: feng
  context: dev
```

配置：

```yaml
spring:
  liquibase:
    contexts: dev
```

只有匹配 `dev` 的 changeSet 会执行。

`label` 更适合按功能或发布范围筛选。

```yaml
changeSet:
  id: add-order-module
  author: feng
  labels: order-module
```

执行时可以只跑某些 label。

简单理解：

```text
context 更偏环境
label 更偏业务范围或发布范围
```

### rollback 回滚

`Liquibase` 支持为 changeSet 编写回滚逻辑。

YAML 示例：

```yaml
databaseChangeLog:
  - changeSet:
      id: 005-add-user-phone-column
      author: feng
      changes:
        - addColumn:
            tableName: users
            columns:
              - column:
                  name: phone
                  type: varchar(30)
      rollback:
        - dropColumn:
            tableName: users
            columnName: phone
```

formatted SQL 示例：

```sql
-- liquibase formatted sql

-- changeset feng:005-add-user-phone-column
alter table users add column phone varchar(30);

-- rollback alter table users drop column phone;
```

回滚前通常先生成 SQL 预览，再执行真正回滚。

### 常用命令

如果使用 Liquibase CLI，常见命令如下。

执行未执行变更：

```bash
liquibase update
```

查看未执行变更：

```bash
liquibase status
```

生成即将执行的 SQL：

```bash
liquibase update-sql
```

回滚最近 1 个 changeSet：

```bash
liquibase rollback-count 1
```

生成回滚 SQL：

```bash
liquibase rollback-count-sql 1
```

打标签：

```bash
liquibase tag v1.0.0
```

回滚到标签：

```bash
liquibase rollback v1.0.0
```

查看变更历史：

```bash
liquibase history
```

### Maven 插件

也可以使用 Maven 插件。

```xml
<plugin>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-maven-plugin</artifactId>
    <version>5.0.1</version>
    <configuration>
        <changeLogFile>src/main/resources/db/changelog/db.changelog-master.yaml</changeLogFile>
        <url>jdbc:mysql://localhost:3306/liquibase_demo</url>
        <username>root</username>
        <password>123456</password>
        <driver>com.mysql.cj.jdbc.Driver</driver>
    </configuration>
</plugin>
```

常用命令：

```bash
mvn liquibase:update
mvn liquibase:status
mvn liquibase:updateSQL
mvn liquibase:rollback -Dliquibase.rollbackCount=1
```

实际项目里，Spring Boot 启动自动执行和 CI/CD 命令执行二选一即可。

团队规模较大时，数据库迁移经常放在发布流水线里执行。

### 和 Flyway 的区别

`Flyway` 和 `Liquibase` 都是数据库迁移工具。

| 对比项 | Liquibase | Flyway |
| --- | --- | --- |
| 变更格式 | YAML、XML、JSON、SQL | 主要是 SQL |
| 抽象变更 | 支持 `createTable`、`addColumn` 等 | 以 SQL 脚本为主 |
| 回滚 | 社区版也支持手写 rollback | 常见做法是写反向迁移 |
| 学习成本 | 稍高 | 较低 |
| 灵活性 | 高 | 高 |
| 适合场景 | 复杂变更、多格式、需要回滚定义 | SQL 优先、简单直接 |

粗略理解：

```text
Flyway 更像按顺序执行 SQL 文件
Liquibase 更像用 changeSet 管理数据库变更记录
```

如果团队喜欢纯 SQL，`Flyway` 会很顺手。

如果需要 YAML/XML、precondition、context、label、rollback，`Liquibase` 更合适。

### 和 JPA ddl-auto 的关系

JPA 有：

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update
```

它可以根据实体自动改表。

这个能力适合本地快速验证。

但在多人协作和生产环境里，自动改表不容易审计。

使用 `Liquibase` 后，推荐让数据库结构由 changelog 管理：

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

这样表结构变化都能在 Git 里看到。

### 实战 Demo：配合 Spring Data JPA

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
        user.setCreatedAt(LocalDateTime.now());

        User saved = userRepository.save(user);
        return saved.getId();
    }
}
```

这里的表结构由 `Liquibase` changelog 创建。

业务读写由 `Spring Data JPA` 完成。

两者分工很清楚。

### 常见使用建议

### 已执行的 changeSet 保持稳定

`changeSet` 执行后，`Liquibase` 会记录 checksum。

如果直接修改已执行的 changeSet，后续运行可能出现 checksum 校验问题。

更稳妥的方式是：

```text
新增一个 changeSet 修正前面的结构。
```

例如字段长度从 `varchar(50)` 改成 `varchar(100)`：

```yaml
databaseChangeLog:
  - changeSet:
      id: 006-modify-user-username-length
      author: feng
      changes:
        - modifyDataType:
            tableName: users
            columnName: username
            newDataType: varchar(100)
      rollback:
        - modifyDataType:
            tableName: users
            columnName: username
            newDataType: varchar(50)
```

### changeSet 保持小而清楚

一个 `changeSet` 最好表达一个明确变更。

例如：

```text
创建 users 表
给 users 添加 status 字段
创建 orders 表
创建 idx_orders_user_id 索引
初始化字典数据
```

这样出问题时更容易定位，也更容易写 rollback。

### 生产环境先预览 SQL

正式执行前可以先生成 SQL：

```bash
liquibase update-sql
```

或者 Maven：

```bash
mvn liquibase:updateSQL
```

生成的 SQL 可以给 DBA 或团队评审。

### 初始化数据区分环境

测试账号、演示数据、开发配置，建议加 `context`。

```yaml
context: dev,test
```

生产环境只执行：

```yaml
spring:
  liquibase:
    contexts: prod
```

这样可以避免测试数据进入生产库。

### 大表变更要拆步骤

大表上直接执行这些操作可能耗时较长：

* 增加非空字段
* 修改字段类型
* 添加唯一索引
* 大批量 update
* 大批量 delete

更常见的做法是拆分：

```text
先加可空字段
应用兼容新旧字段
分批回填数据
再加约束或切换读取逻辑
```

`Liquibase` 可以记录每一步，但具体执行策略仍然要结合数据库规模和业务窗口。

### 多实例启动注意锁

`DATABASECHANGELOGLOCK` 会防止多个实例同时迁移。

如果某次迁移异常中断，锁可能没有释放。

这时需要先确认没有迁移任务正在执行，再处理锁表。

处理锁表前，先确认没有迁移任务仍在执行。

### 常用 change 类型汇总

| change 类型 | 作用 |
| --- | --- |
| `createTable` | 创建表 |
| `dropTable` | 删除表 |
| `addColumn` | 新增字段 |
| `dropColumn` | 删除字段 |
| `modifyDataType` | 修改字段类型 |
| `renameColumn` | 重命名字段 |
| `createIndex` | 创建索引 |
| `dropIndex` | 删除索引 |
| `addUniqueConstraint` | 添加唯一约束 |
| `addForeignKeyConstraint` | 添加外键 |
| `insert` | 插入数据 |
| `update` | 更新数据 |
| `delete` | 删除数据 |
| `sql` | 执行原生 SQL |
| `loadData` | 从 CSV 加载数据 |

### 常用配置汇总

| 配置 | 作用 |
| --- | --- |
| `spring.liquibase.enabled` | 是否启用 |
| `spring.liquibase.change-log` | 主 changelog 路径 |
| `spring.liquibase.contexts` | 执行指定 context |
| `spring.liquibase.label-filter` | 执行指定 label |
| `spring.liquibase.default-schema` | 默认 schema |
| `spring.liquibase.liquibase-schema` | Liquibase 记录表所在 schema |
| `spring.liquibase.drop-first` | 执行前先删除数据库对象 |
| `spring.liquibase.rollback-file` | 生成 rollback SQL 文件 |

### 总结

`Liquibase` 的重点是把数据库变更纳入工程化管理。

常见使用流程是：

```text
创建 master changelog
按版本拆分 changelog 文件
每次表结构变化新增 changeSet
启动应用或流水线执行 update
通过 DATABASECHANGELOG 追踪执行历史
必要时使用 rollback 或新增修复 changeSet
```

它适合这些场景：

* Spring Boot 项目需要稳定初始化表结构
* 多环境数据库结构需要保持一致
* 数据库变更需要代码评审
* 发布过程需要自动执行数据库迁移
* 表结构变化需要可追踪、可回滚

落地时重点关注几件事：

* changelog 文件纳入 Git
* 已执行 changeSet 不直接修改
* 大表变更拆步骤
* 生产环境先预览 SQL
* 测试数据使用 context 隔离
* JPA `ddl-auto` 不和 Liquibase 抢表结构控制权

掌握这些内容后，`Liquibase` 已经可以覆盖大多数 Java 项目的数据库版本管理需求。
