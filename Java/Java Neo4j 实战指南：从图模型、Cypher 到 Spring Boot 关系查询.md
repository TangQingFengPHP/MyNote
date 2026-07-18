### 简介

Neo4j 是一个图数据库。

关系型数据库习惯用表、行、列描述数据，复杂关系通常靠外键和 JOIN 拼起来。

Neo4j 换了一个角度：直接用节点和关系描述数据。

```text
(张三:User)-[:FRIEND_OF]->(李四:User)
(张三:User)-[:BOUGHT]->(手机:Product)
(手机:Product)-[:BELONGS_TO]->(数码:Category)
```

这种模型很适合关系密集的数据。

比如社交好友、知识图谱、推荐系统、权限关系、风控链路、组织架构、物流路径。数据之间的连接本身就是业务重点时，图数据库会比一堆 JOIN 更自然。

Neo4j 在 Java 项目里常见两种用法：

| 用法 | 适合场景 |
| --- | --- |
| Neo4j Java Driver | 直接写 Cypher，控制力强，适合复杂查询 |
| Spring Data Neo4j | Spring Boot 项目里做对象映射和 Repository 开发 |

### 图数据库解决什么问题

假设要查一个用户的三层好友。

关系型数据库里通常需要一张用户表，再加一张好友关系表：

```text
user
id | name
---|------
1  | 张三
2  | 李四
3  | 王五

friend
user_id | friend_id
--------|----------
1       | 2
2       | 3
```

查多层关系时，SQL 会变成多次自连接：

```sql
SELECT f3.friend_id
FROM friend f1
JOIN friend f2 ON f1.friend_id = f2.user_id
JOIN friend f3 ON f2.friend_id = f3.user_id
WHERE f1.user_id = 1;
```

层级越深，SQL 越难维护。

Neo4j 里可以直接表达路径：

```cypher
MATCH (:User {userId: 1})-[:FRIEND_OF*1..3]->(friend:User)
RETURN DISTINCT friend
```

`[:FRIEND_OF*1..3]` 表示沿着 `FRIEND_OF` 关系走 1 到 3 层。

路径查询、关系遍历、共同好友、最短路径这类需求，是 Neo4j 比较擅长的地方。

### 核心概念

Neo4j 的概念不多，关键是把它们组合起来。

| 概念 | 含义 | 示例 |
| --- | --- | --- |
| Node | 节点，表示一个实体 | 用户、商品、电影、公司 |
| Label | 标签，表示节点类型 | `User`、`Product`、`Movie` |
| Relationship | 关系，表示节点之间的连接 | `FRIEND_OF`、`BOUGHT` |
| Type | 关系类型 | `ACTED_IN`、`WORKS_AT` |
| Property | 属性，节点和关系都可以有 | `name`、`age`、`score` |
| Cypher | Neo4j 查询语言 | 类似 SQL 的声明式查询语言 |

一个节点可以有多个标签：

```cypher
CREATE (:User:Vip {userId: 1, name: "张三"})
```

关系有方向，也可以带属性：

```cypher
MATCH (u:User {userId: 1}), (p:Product {skuId: 1001})
CREATE (u)-[:BOUGHT {quantity: 2, amount: 1999.00}]->(p)
```

关系方向不是说业务只能单向理解，而是查询时需要清楚从哪里出发、沿着什么关系走。

### 本地启动 Neo4j

本地开发可以用 Docker 起一个 Neo4j。

```bash
docker run -d \
  --name neo4j-demo \
  -p 7474:7474 \
  -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/secretgraph \
  neo4j:latest
```

端口含义：

| 端口 | 作用 |
| --- | --- |
| `7474` | Neo4j Browser Web 控制台 |
| `7687` | Bolt 协议，Java 应用连接使用 |

浏览器访问：

```text
http://localhost:7474
```

连接信息：

```text
uri: bolt://localhost:7687
username: neo4j
password: secretgraph
```

### Cypher 基础语法

Cypher 是 Neo4j 的查询语言。

它看起来不像 SQL，但思路很直观：用括号表示节点，用箭头表示关系。

### 创建节点

```cypher
CREATE (:User {userId: 1, name: "张三", age: 25});
CREATE (:User {userId: 2, name: "李四", age: 26});
CREATE (:User {userId: 3, name: "王五", age: 24});
```

### 查询节点

```cypher
MATCH (u:User)
RETURN u;
```

按属性查询：

```cypher
MATCH (u:User {userId: 1})
RETURN u.name AS name, u.age AS age;
```

### 创建关系

```cypher
MATCH (a:User {userId: 1}), (b:User {userId: 2})
CREATE (a)-[:FRIEND_OF {since: date("2024-01-01")}]->(b);
```

查询好友：

```cypher
MATCH (u:User {userId: 1})-[:FRIEND_OF]->(friend:User)
RETURN friend;
```

### MERGE：有就匹配，没有就创建

`CREATE` 每次都会创建新数据。

`MERGE` 更像“按模式查找，找不到再创建”。

```cypher
MERGE (u:User {userId: 1})
ON CREATE SET u.name = "张三", u.createdAt = datetime()
ON MATCH SET u.updatedAt = datetime()
RETURN u;
```

关系也可以用 `MERGE`：

```cypher
MATCH (a:User {userId: 1}), (b:User {userId: 2})
MERGE (a)-[:FRIEND_OF]->(b);
```

使用 `MERGE` 前，常见做法是先给业务唯一字段建唯一约束，避免并发写入时出现重复节点。

### 更新和删除

更新属性：

```cypher
MATCH (u:User {userId: 1})
SET u.age = 26, u.updatedAt = datetime()
RETURN u;
```

删除关系：

```cypher
MATCH (:User {userId: 1})-[r:FRIEND_OF]->(:User {userId: 2})
DELETE r;
```

删除节点和它的关系：

```cypher
MATCH (u:User {userId: 3})
DETACH DELETE u;
```

`DELETE` 只能删除没有关系的节点。

`DETACH DELETE` 会连同关系一起删除，使用时要确认业务含义。

### 索引和唯一约束

频繁按 `userId` 查用户时，可以建唯一约束：

```cypher
CREATE CONSTRAINT user_id_unique IF NOT EXISTS
FOR (u:User)
REQUIRE u.userId IS UNIQUE;
```

按商品编号查商品：

```cypher
CREATE CONSTRAINT product_sku_unique IF NOT EXISTS
FOR (p:Product)
REQUIRE p.skuId IS UNIQUE;
```

约束带来两个好处：

| 好处 | 说明 |
| --- | --- |
| 保证唯一性 | 避免业务唯一字段重复 |
| 提升查询性能 | `MATCH` 和 `MERGE` 可以走索引 |

`MERGE` 高频写入时，唯一约束尤其重要。

### Java Driver 方式

Neo4j 官方 Java Driver 适合直接执行 Cypher。

Maven 依赖：

```xml
<dependency>
    <groupId>org.neo4j.driver</groupId>
    <artifactId>neo4j-java-driver</artifactId>
    <version>${neo4j-java-driver.version}</version>
</dependency>
```

Neo4j Java Driver 6.x 需要 Java 17 或更高版本。

连接配置：

```java
package com.example.neo4j;

import org.neo4j.driver.AuthTokens;
import org.neo4j.driver.Driver;
import org.neo4j.driver.GraphDatabase;

public class Neo4jDriverFactory {

    public Driver createDriver() {
        return GraphDatabase.driver(
                "bolt://localhost:7687",
                AuthTokens.basic("neo4j", "secretgraph")
        );
    }
}
```

`Driver` 是线程安全对象，通常整个应用只创建一个。

`Session` 是轻量级会话，用完关闭。

### Java Driver 写入节点和关系

```java
package com.example.neo4j;

import org.neo4j.driver.Driver;
import org.neo4j.driver.Session;

import static org.neo4j.driver.Values.parameters;

public class UserGraphRepository implements AutoCloseable {

    private final Driver driver;

    public UserGraphRepository(Driver driver) {
        this.driver = driver;
    }

    public void saveUser(Long userId, String name, Integer age) {
        try (Session session = driver.session()) {
            session.executeWrite(tx -> {
                tx.run("""
                        MERGE (u:User {userId: $userId})
                        SET u.name = $name,
                            u.age = $age,
                            u.updatedAt = datetime()
                        """, parameters(
                        "userId", userId,
                        "name", name,
                        "age", age
                )).consume();
                return null;
            });
        }
    }

    public void addFriend(Long userId, Long friendId) {
        try (Session session = driver.session()) {
            session.executeWrite(tx -> {
                tx.run("""
                        MATCH (a:User {userId: $userId})
                        MATCH (b:User {userId: $friendId})
                        MERGE (a)-[:FRIEND_OF]->(b)
                        """, parameters(
                        "userId", userId,
                        "friendId", friendId
                )).consume();
                return null;
            });
        }
    }

    @Override
    public void close() {
        driver.close();
    }
}
```

这里用参数传值，而不是把变量拼到 Cypher 字符串里。

参数化查询可以减少注入风险，也方便 Neo4j 复用查询计划。

### Java Driver 查询结果

```java
package com.example.neo4j;

import org.neo4j.driver.Driver;
import org.neo4j.driver.Record;
import org.neo4j.driver.Session;

import java.util.List;

import static org.neo4j.driver.Values.parameters;

public class UserGraphQueryRepository {

    private final Driver driver;

    public UserGraphQueryRepository(Driver driver) {
        this.driver = driver;
    }

    public List<UserFriendView> findFriends(Long userId) {
        try (Session session = driver.session()) {
            return session.executeRead(tx -> tx.run("""
                            MATCH (:User {userId: $userId})-[:FRIEND_OF]->(friend:User)
                            RETURN friend.userId AS userId,
                                   friend.name AS name,
                                   friend.age AS age
                            ORDER BY friend.name
                            """, parameters("userId", userId))
                    .list(this::toUserFriendView));
        }
    }

    private UserFriendView toUserFriendView(Record record) {
        return new UserFriendView(
                record.get("userId").asLong(),
                record.get("name").asString(),
                record.get("age").isNull() ? null : record.get("age").asInt()
        );
    }
}
```

返回对象：

```java
package com.example.neo4j;

public record UserFriendView(
        Long userId,
        String name,
        Integer age
) {
}
```

Java Driver 的优点是灵活。

复杂路径查询、动态图谱查询、性能调优时，直接写 Cypher 往往更清楚。

### Spring Boot 集成 Spring Data Neo4j

Spring Boot 项目里更常用的是 Spring Data Neo4j。

Maven 依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-neo4j</artifactId>
</dependency>
```

`application.yml`：

```yaml
spring:
  neo4j:
    uri: bolt://localhost:7687
    authentication:
      username: neo4j
      password: secretgraph
```

Spring Data Neo4j 的调用链大概是：

```text
Controller
  -> Service
      -> Neo4jRepository
          -> Spring Data Neo4j
              -> Neo4j Java Driver
                  -> Neo4j
```

### 节点实体映射

用户节点：

```java
package com.example.graph.user;

import org.springframework.data.neo4j.core.schema.GeneratedValue;
import org.springframework.data.neo4j.core.schema.Id;
import org.springframework.data.neo4j.core.schema.Node;
import org.springframework.data.neo4j.core.schema.Property;
import org.springframework.data.neo4j.core.schema.Relationship;

import java.util.HashSet;
import java.util.Set;

@Node("User")
public class UserNode {

    @Id
    @GeneratedValue
    private Long id;

    @Property("userId")
    private Long userId;

    private String name;

    private Integer age;

    @Relationship(type = "FRIEND_OF", direction = Relationship.Direction.OUTGOING)
    private Set<UserNode> friends = new HashSet<>();

    protected UserNode() {
    }

    public UserNode(Long userId, String name, Integer age) {
        this.userId = userId;
        this.name = name;
        this.age = age;
    }

    public Long getId() {
        return id;
    }

    public Long getUserId() {
        return userId;
    }

    public String getName() {
        return name;
    }

    public Integer getAge() {
        return age;
    }

    public Set<UserNode> getFriends() {
        return friends;
    }
}
```

商品节点：

```java
package com.example.graph.product;

import org.springframework.data.neo4j.core.schema.GeneratedValue;
import org.springframework.data.neo4j.core.schema.Id;
import org.springframework.data.neo4j.core.schema.Node;
import org.springframework.data.neo4j.core.schema.Property;

@Node("Product")
public class ProductNode {

    @Id
    @GeneratedValue
    private Long id;

    @Property("skuId")
    private Long skuId;

    private String name;

    private String category;

    protected ProductNode() {
    }

    public ProductNode(Long skuId, String name, String category) {
        this.skuId = skuId;
        this.name = name;
        this.category = category;
    }

    public Long getId() {
        return id;
    }

    public Long getSkuId() {
        return skuId;
    }

    public String getName() {
        return name;
    }

    public String getCategory() {
        return category;
    }
}
```

`@Node` 对应节点标签。

`@Property` 可以指定 Neo4j 里的属性名。

`@Relationship` 表示节点之间的关系。

### Repository 查询

```java
package com.example.graph.user;

import org.springframework.data.neo4j.repository.Neo4jRepository;
import org.springframework.data.neo4j.repository.query.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;
import java.util.Optional;

public interface UserNodeRepository extends Neo4jRepository<UserNode, Long> {

    Optional<UserNode> findByUserId(Long userId);

    List<UserNode> findByNameContaining(String keyword);

    @Query("""
            MATCH (u:User {userId: $userId})-[:FRIEND_OF]->(friend:User)
            RETURN friend
            ORDER BY friend.name
            """)
    List<UserNode> findFriends(@Param("userId") Long userId);

    @Query("""
            MATCH (u:User {userId: $userId})
            MATCH (friend:User {userId: $friendId})
            MERGE (u)-[:FRIEND_OF]->(friend)
            """)
    void addFriend(@Param("userId") Long userId, @Param("friendId") Long friendId);

    @Query("""
            MATCH (u:User {userId: $userId})-[r:FRIEND_OF]->(friend:User {userId: $friendId})
            DELETE r
            """)
    void removeFriend(@Param("userId") Long userId, @Param("friendId") Long friendId);
}
```

`Neo4jRepository` 提供基础 CRUD。

简单查询可以用方法名派生。

复杂关系查询直接写 `@Query`，可读性通常更好。

### Service 示例

```java
package com.example.graph.user;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class UserGraphService {

    private final UserNodeRepository userNodeRepository;

    public UserGraphService(UserNodeRepository userNodeRepository) {
        this.userNodeRepository = userNodeRepository;
    }

    @Transactional
    public UserNode create(UserCreateRequest request) {
        UserNode user = new UserNode(request.userId(), request.name(), request.age());
        return userNodeRepository.save(user);
    }

    @Transactional
    public void addFriend(Long userId, Long friendId) {
        userNodeRepository.addFriend(userId, friendId);
    }

    @Transactional(readOnly = true)
    public List<UserNode> findFriends(Long userId) {
        return userNodeRepository.findFriends(userId);
    }

    @Transactional
    public void removeFriend(Long userId, Long friendId) {
        userNodeRepository.removeFriend(userId, friendId);
    }
}
```

请求对象：

```java
package com.example.graph.user;

public record UserCreateRequest(
        Long userId,
        String name,
        Integer age
) {
}
```

### Controller 示例

接口层不直接返回复杂的节点对象，避免关系对象递归序列化。

```java
package com.example.graph.user;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/api/graph/users")
public class UserGraphController {

    private final UserGraphService userGraphService;

    public UserGraphController(UserGraphService userGraphService) {
        this.userGraphService = userGraphService;
    }

    @PostMapping
    public UserView create(@RequestBody UserCreateRequest request) {
        UserNode user = userGraphService.create(request);
        return UserView.from(user);
    }

    @PostMapping("/{userId}/friends/{friendId}")
    public void addFriend(@PathVariable Long userId, @PathVariable Long friendId) {
        userGraphService.addFriend(userId, friendId);
    }

    @GetMapping("/{userId}/friends")
    public List<UserView> findFriends(@PathVariable Long userId) {
        return userGraphService.findFriends(userId)
                .stream()
                .map(UserView::from)
                .toList();
    }
}
```

返回对象：

```java
package com.example.graph.user;

public record UserView(
        Long userId,
        String name,
        Integer age
) {

    public static UserView from(UserNode user) {
        return new UserView(user.getUserId(), user.getName(), user.getAge());
    }
}
```

这个接口已经能完成：

| 接口 | 作用 |
| --- | --- |
| `POST /api/graph/users` | 创建用户节点 |
| `POST /api/graph/users/{userId}/friends/{friendId}` | 创建好友关系 |
| `GET /api/graph/users/{userId}/friends` | 查询好友列表 |

### 关系属性：用户给商品评分

Neo4j 的关系也可以带属性。

比如用户给商品评分：

```text
(User)-[:RATED {score: 5, comment: "好用"}]->(Product)
```

Spring Data Neo4j 可以用 `@RelationshipProperties` 映射这种关系。

```java
package com.example.graph.rating;

import com.example.graph.product.ProductNode;
import org.springframework.data.neo4j.core.schema.RelationshipId;
import org.springframework.data.neo4j.core.schema.RelationshipProperties;
import org.springframework.data.neo4j.core.schema.TargetNode;

import java.time.LocalDateTime;

@RelationshipProperties
public class RatingRelation {

    @RelationshipId
    private Long id;

    @TargetNode
    private ProductNode product;

    private Integer score;

    private String comment;

    private LocalDateTime createdAt;

    protected RatingRelation() {
    }

    public RatingRelation(ProductNode product, Integer score, String comment) {
        this.product = product;
        this.score = score;
        this.comment = comment;
        this.createdAt = LocalDateTime.now();
    }

    public ProductNode getProduct() {
        return product;
    }

    public Integer getScore() {
        return score;
    }

    public String getComment() {
        return comment;
    }

    public LocalDateTime getCreatedAt() {
        return createdAt;
    }
}
```

在 `UserNode` 上增加关系：

```java
@Relationship(type = "RATED", direction = Relationship.Direction.OUTGOING)
private Set<RatingRelation> ratings = new HashSet<>();
```

这种写法适合关系本身有业务含义的场景。

好友关系只有连接意义时，可以直接用节点集合。

评分、购买、转账、任职、参与项目这类关系，通常都需要关系属性。

### 路径查询：共同好友和最短路径

共同好友：

```cypher
MATCH (a:User {userId: $userId})-[:FRIEND_OF]->(common:User)<-[:FRIEND_OF]-(b:User {userId: $otherUserId})
RETURN common.userId AS userId, common.name AS name
ORDER BY common.name
```

好友推荐：

```cypher
MATCH (u:User {userId: $userId})-[:FRIEND_OF]->(:User)-[:FRIEND_OF]->(candidate:User)
WHERE NOT (u)-[:FRIEND_OF]->(candidate)
  AND candidate.userId <> u.userId
RETURN candidate.userId AS userId,
       candidate.name AS name,
       count(*) AS score
ORDER BY score DESC, name
LIMIT 10
```

最短路径：

```cypher
MATCH path = shortestPath(
  (start:User {userId: $startUserId})-[:FRIEND_OF*1..6]-(end:User {userId: $endUserId})
)
RETURN path
```

路径查询要控制最大深度。

没有上限的变长关系查询，数据量大时容易产生非常高的遍历成本。

### 用 Neo4jClient 写灵活查询

Spring Data Neo4j 除了 Repository，也提供 `Neo4jClient`。

它适合查询结果不是完整实体，而是报表、推荐列表、路径统计这类 DTO。

```java
package com.example.graph.recommend;

import org.springframework.data.neo4j.core.Neo4jClient;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public class UserRecommendRepository {

    private final Neo4jClient neo4jClient;

    public UserRecommendRepository(Neo4jClient neo4jClient) {
        this.neo4jClient = neo4jClient;
    }

    public List<UserRecommendView> recommendFriends(Long userId) {
        return neo4jClient.query("""
                        MATCH (u:User {userId: $userId})-[:FRIEND_OF]->(:User)-[:FRIEND_OF]->(candidate:User)
                        WHERE NOT (u)-[:FRIEND_OF]->(candidate)
                          AND candidate.userId <> u.userId
                        RETURN candidate.userId AS userId,
                               candidate.name AS name,
                               count(*) AS score
                        ORDER BY score DESC, name
                        LIMIT 10
                        """)
                .bind(userId).to("userId")
                .fetchAs(UserRecommendView.class)
                .mappedBy((typeSystem, record) -> new UserRecommendView(
                        record.get("userId").asLong(),
                        record.get("name").asString(),
                        record.get("score").asLong()
                ))
                .all()
                .stream()
                .toList();
    }
}
```

返回对象：

```java
package com.example.graph.recommend;

public record UserRecommendView(
        Long userId,
        String name,
        Long score
) {
}
```

Repository 适合实体 CRUD。

`Neo4jClient` 适合复杂查询和自定义结果映射。

### 事务处理

Spring Data Neo4j 可以直接使用 Spring 的 `@Transactional`。

```java
@Transactional
public void createUserAndFriend(UserCreateRequest userRequest, UserCreateRequest friendRequest) {
    UserNode user = userNodeRepository.save(
            new UserNode(userRequest.userId(), userRequest.name(), userRequest.age())
    );
    UserNode friend = userNodeRepository.save(
            new UserNode(friendRequest.userId(), friendRequest.name(), friendRequest.age())
    );
    userNodeRepository.addFriend(user.getUserId(), friend.getUserId());
}
```

Java Driver 方式则使用 `executeWrite`、`executeRead` 管理读写事务。

```java
try (Session session = driver.session()) {
    session.executeWrite(tx -> {
        tx.run("MERGE (:User {userId: $userId})", parameters("userId", 1L)).consume();
        tx.run("MERGE (:User {userId: $userId})", parameters("userId", 2L)).consume();
        tx.run("""
                MATCH (a:User {userId: $aId}), (b:User {userId: $bId})
                MERGE (a)-[:FRIEND_OF]->(b)
                """, parameters("aId", 1L, "bId", 2L)).consume();
        return null;
    });
}
```

读写事务分开写，语义更清楚，也方便驱动做重试和路由。

### 建模建议

图数据库建模的重点不是“把表搬成节点”，而是把业务问题里的关系表达出来。

| 业务问题 | 图模型思路 |
| --- | --- |
| 查共同好友 | `User` 节点 + `FRIEND_OF` 关系 |
| 商品推荐 | `User`、`Product`、`Category` + `BOUGHT`、`BELONGS_TO` |
| 权限判断 | `User`、`Role`、`Permission` + `HAS_ROLE`、`HAS_PERMISSION` |
| 风控排查 | `Account`、`Device`、`Phone`、`Ip` + 登录、绑定、使用关系 |
| 知识图谱 | 实体节点 + 语义关系 |

几个建模原则：

| 原则 | 说明 |
| --- | --- |
| 节点表示实体 | 用户、商品、公司、账号、设备 |
| 关系表示动作或连接 | 关注、购买、登录、绑定、属于 |
| 高频过滤字段建索引 | `userId`、`skuId`、`accountId` |
| 关系有业务属性时建关系属性 | 评分、金额、时间、角色 |
| 控制变长路径深度 | 避免无上限遍历 |

### Neo4j 和 MySQL 怎么选

Neo4j 不是 MySQL 的替代品，更像是补充。

| 场景 | 更适合 |
| --- | --- |
| 标准订单 CRUD | MySQL |
| 财务流水、强报表聚合 | MySQL |
| 多层关系查询 | Neo4j |
| 好友推荐、共同关系 | Neo4j |
| 知识图谱查询 | Neo4j |
| 风控关系网络 | Neo4j |

常见架构是：

```text
MySQL 保存核心业务数据
Neo4j 保存关系网络和图查询模型
```

比如订单主数据仍然在 MySQL，用户、商品、类目、购买关系同步到 Neo4j，用于推荐和关系分析。

### 常见问题

#### 查询越来越慢

常见原因有三个：

| 原因 | 处理方式 |
| --- | --- |
| 按属性过滤但没有索引 | 给高频查询字段建索引或唯一约束 |
| 变长路径没有限制深度 | 使用 `*1..3`、`*1..6` 这类上限 |
| 返回了过多节点和关系 | 只返回需要的字段，配合 `LIMIT` |

可以用 `EXPLAIN` 或 `PROFILE` 看执行计划：

```cypher
PROFILE
MATCH (u:User {userId: 1})-[:FRIEND_OF*1..3]->(friend:User)
RETURN DISTINCT friend
LIMIT 20;
```

#### MERGE 创建了重复数据

常见原因是 `MERGE` 的模式没有覆盖唯一业务字段，或者缺少唯一约束。

推荐先创建约束：

```cypher
CREATE CONSTRAINT user_id_unique IF NOT EXISTS
FOR (u:User)
REQUIRE u.userId IS UNIQUE;
```

再按唯一字段 `MERGE`：

```cypher
MERGE (u:User {userId: $userId})
SET u.name = $name;
```

#### 返回实体导致 JSON 无限嵌套

节点之间互相关联时，直接把实体返回给接口层，可能出现 JSON 循环或数据过大。

处理方式是返回 DTO：

```java
public record UserView(Long userId, String name, Integer age) {
}
```

Controller 里把实体转换成 DTO，再返回给前端。

#### Spring Data Neo4j 保存关系时覆盖了旧关系

Spring Data Neo4j 以聚合对象为单位保存数据。

如果实体里的关系集合不是完整状态，直接 `save` 可能让关系结果和预期不一致。

关系增删比较复杂时，可以用自定义 Cypher 精确控制：

```java
@Query("""
        MATCH (u:User {userId: $userId})
        MATCH (friend:User {userId: $friendId})
        MERGE (u)-[:FRIEND_OF]->(friend)
        """)
void addFriend(Long userId, Long friendId);
```

### 实践建议

| 场景 | 建议 |
| --- | --- |
| Spring Boot 标准项目 | 优先使用 `spring-boot-starter-data-neo4j` |
| 复杂图查询 | 直接写 Cypher 或使用 `Neo4jClient` |
| 高频唯一字段 | 创建唯一约束 |
| 路径查询 | 限制关系层数 |
| 接口返回 | 返回 DTO，避免直接暴露节点实体 |
| 写入关系 | 简单关系用对象映射，复杂关系用 `@Query` |
| 性能分析 | 使用 `EXPLAIN`、`PROFILE` |
| 数据同步 | MySQL 负责主数据，Neo4j 负责图查询模型 |

### 小结

Neo4j 的核心不是“另一种存表方式”，而是把关系当成一等公民。

节点表示实体，关系表示实体之间的连接，属性描述节点和关系的细节，Cypher 用图模式表达查询。

Java 项目里，简单直接的方式是 Neo4j Java Driver，适合手写 Cypher 和复杂查询。Spring Boot 项目里更常见的是 Spring Data Neo4j，适合 Repository、对象映射、事务管理和常规 CRUD。

如果业务重点是多层关系、路径、推荐、知识图谱、风控网络，Neo4j 能把模型和查询写得更贴近业务本身。
