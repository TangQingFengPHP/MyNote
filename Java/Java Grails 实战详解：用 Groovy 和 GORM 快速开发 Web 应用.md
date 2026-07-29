### 简介

`Grails` 是一个运行在 JVM 上的全栈 Web 开发框架。

它基于这些技术：

```text
Groovy
Spring Boot
Spring Framework
Hibernate
GORM
Gradle
```

一句话概括：

```text
Grails 用 Groovy 的简洁语法，把 Spring Boot、Hibernate、Web MVC、校验、脚手架、视图和插件整合成一套高效率 Web 开发框架。
```

它的核心思想是：

```text
约定优于配置。
```

也就是很多事情不需要手写配置，只要按约定放到对应目录、使用对应命名，框架就知道该怎么处理。

比如：

```text
Book.groovy 放在 grails-app/domain
BookController.groovy 放在 grails-app/controllers
book/index.gsp 放在 grails-app/views
```

Grails 会自动把这些东西串起来。

### Grails 适合什么场景

Grails 适合快速做 Web 系统。

常见场景有：

| 场景 | 说明 |
| --- | --- |
| 后台管理系统 | 表单、列表、分页、增删改查很多 |
| CRUD 系统 | 领域模型清楚，主要围绕数据库操作 |
| REST API | 用 GORM 快速做接口 |
| 原型项目 | 先快速跑通业务，再逐步优化 |
| 内部工具 | 权限不复杂、开发效率优先 |
| 中小型业务系统 | 希望少写样板代码 |

Grails 不太适合所有场景。

如果项目里需要大量底层定制、团队主要写 Java、国内招聘和维护成本是重点，Spring Boot 可能更稳。

如果项目更看重开发效率、领域模型清楚、团队接受 Groovy，Grails 能少写很多重复代码。

### Grails 和 Spring Boot 的关系

Grails 不是 Spring Boot 的反面。

它更像是：

```text
Grails = Spring Boot + Groovy + GORM + 约定 + 脚手架 + 插件体系
```

Spring Boot 提供底层启动、自动配置、Web 容器、Spring 生态集成。

Grails 在上面加了一层高生产力开发模型：

```text
Controller
  |
  v
Service
  |
  v
GORM
  |
  v
Hibernate
  |
  v
Database
```

Spring Boot 项目里经常需要手写：

* Entity
* Repository
* Service
* Controller
* DTO
* 校验
* 分页
* 视图或 JSON 返回

Grails 会用 Domain、GORM、Controller、Scaffolding、JSON Views 等能力把这些事情简化。

### Grails 和 Spring Boot 对比

| 对比项 | Grails | Spring Boot |
| --- | --- | --- |
| 主要语言 | Groovy | Java / Kotlin |
| 底层 | 基于 Spring Boot | Spring Boot 本身 |
| ORM | GORM，常基于 Hibernate | JPA、MyBatis、JOOQ 等 |
| 开发风格 | 约定强，代码少 | 更显式，更自由 |
| CRUD 效率 | 很高 | 需要自己组织代码 |
| 国内生态 | 相对小 | 非常大 |
| 学习重点 | Grails 约定、Groovy、GORM | Spring、配置、组件选择 |
| 适合项目 | 后台、CRUD、原型、内部系统 | 通用后端服务 |

Grails 更像“完整套餐”。

Spring Boot 更像“底座 + 自选组件”。

### 版本怎么选

Grails 版本和 JDK、Spring Boot、Groovy、Jakarta EE 关系很紧。

当前官方 `latest` 文档已经指向 Grails 8 里程碑版本。Grails 8 面向更新的技术栈，官方指南中已经出现基于 Spring Boot 4 和 JDK 21 的示例。

常见版本线可以这样理解：

| Grails 版本 | JDK 要求 | 说明 |
| --- | --- | --- |
| Grails 5 | JDK 8+ | 老项目常见 |
| Grails 6 | JDK 11+ | Spring Boot 2.x 体系 |
| Grails 7 | JDK 17+ | Spring Boot 3.x / Jakarta 迁移线 |
| Grails 8 | JDK 21 更常见 | 面向 Spring Boot 4 的新版本线 |

新项目如果追求稳定，优先看当前稳定发布线。

如果是学习新技术栈，可以关注 Grails 8 文档和指南。

如果是老系统维护，版本选择要先看项目用的是：

```text
javax.servlet.*
```

还是：

```text
jakarta.servlet.*
```

Spring Boot 3 以后整体进入 Jakarta 命名空间，老项目迁移时要特别注意。

### 安装 Grails

macOS / Linux 推荐用 SDKMAN 管理 Grails。

安装 SDKMAN：

```shell
curl -s https://get.sdkman.io | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
```

安装 Grails：

```shell
sdk install grails
```

指定版本：

```shell
sdk install grails 8.0.0-M4
```

查看版本：

```shell
grails --version
```

Grails 项目通常也会带 `grailsw` 和 `gradlew`。

实际项目里，更推荐使用项目自带 Wrapper：

```shell
./grailsw --version
./gradlew bootRun
```

这样可以避免本机 Grails / Gradle 版本和项目要求不一致。

### 创建项目

创建一个普通 Web 项目：

```shell
grails create-app library-demo
cd library-demo
```

Grails 8 里同时存在 Shell CLI 和 Forge CLI。

普通开发命令通常直接用：

```shell
grails create-app library-demo
```

如果需要更细的生成参数，可以使用 Forge：

```shell
grails -t forge create-app library-demo
```

运行项目：

```shell
./gradlew bootRun
```

默认访问：

```text
http://localhost:8080
```

改端口：

```shell
./gradlew bootRun -Dgrails.server.port=8090
```

也可以用 Grails 命令：

```shell
grails run-app
```

### 项目目录结构

典型 Grails 项目结构：

```text
library-demo/
├── grails-app/
│   ├── conf/
│   ├── controllers/
│   ├── domain/
│   ├── services/
│   ├── views/
│   ├── i18n/
│   ├── init/
│   └── taglib/
├── src/
│   ├── main/
│   │   ├── groovy/
│   │   └── resources/
│   ├── test/
│   └── integration-test/
├── build.gradle
├── gradlew
└── grailsw
```

常见目录说明：

| 目录 | 作用 |
| --- | --- |
| `grails-app/domain` | Domain 类，也就是领域模型和 GORM 实体 |
| `grails-app/controllers` | Controller，处理 Web 请求 |
| `grails-app/services` | Service，放业务逻辑和事务 |
| `grails-app/views` | GSP 或 JSON View |
| `grails-app/conf` | 配置、URL 映射 |
| `grails-app/init` | 启动类、BootStrap 初始化数据 |
| `grails-app/i18n` | 国际化消息 |
| `src/main/groovy` | 普通 Groovy 类 |
| `src/main/java` | 普通 Java 类 |
| `src/test/groovy` | 单元测试 |
| `src/integration-test/groovy` | 集成测试 |

Grails 的约定会根据目录和类名自动识别组件。

比如：

```text
BookService.groovy -> 自动作为 Service Bean
BookController.groovy -> 自动作为 Controller
Book.groovy -> 自动作为 GORM Domain
```

### 第一个 Controller

创建 Controller：

```shell
grails create-controller hello
```

生成文件：

```text
grails-app/controllers/library/demo/HelloController.groovy
```

代码：

```groovy
package library.demo

class HelloController {

    def index() {
        render "Hello Grails"
    }
}
```

访问：

```text
http://localhost:8080/hello
```

Grails 默认 URL 约定：

```text
/控制器名/动作名/id
```

比如：

```text
/book/index
/book/show/1
/book/create
```

如果只访问 `/hello`，默认会执行 `index` 动作。

### 实战 Demo：图书馆 REST API

下面做一个图书馆管理接口。

目标接口：

| 方法 | 路径 | 作用 |
| --- | --- | --- |
| `GET` | `/v1/authors` | 作者列表 |
| `GET` | `/v1/authors/1` | 作者详情 |
| `POST` | `/v1/authors` | 新增作者 |
| `GET` | `/v1/books` | 图书列表 |
| `GET` | `/v1/books/1` | 图书详情 |
| `POST` | `/v1/books` | 新增图书 |
| `PUT` | `/v1/books/1` | 修改图书 |
| `DELETE` | `/v1/books/1` | 删除图书 |

### 创建 Domain

创建作者：

```shell
grails create-domain-class library.demo.Author
```

创建图书：

```shell
grails create-domain-class library.demo.Book
```

作者 Domain：

```groovy
package library.demo

class Author {

    String name
    String biography
    Date dateOfBirth

    Date dateCreated
    Date lastUpdated

    static hasMany = [books: Book]

    static constraints = {
        name blank: false, unique: true, maxSize: 100
        biography nullable: true, maxSize: 2000
        dateOfBirth nullable: true
    }

    static mapping = {
        books cascade: 'all-delete-orphan'
    }

    String toString() {
        name
    }
}
```

图书 Domain：

```groovy
package library.demo

class Book {

    String title
    String isbn
    Integer pageCount
    Date publishedOn

    Date dateCreated
    Date lastUpdated

    static belongsTo = [author: Author]

    static constraints = {
        title blank: false, maxSize: 200
        isbn blank: false, unique: true, matches: /\d{10}|\d{13}/
        pageCount nullable: true, min: 1
        publishedOn nullable: true
        author nullable: false
    }

    static mapping = {
        table 'book'
    }

    String toString() {
        title
    }
}
```

这两个类没有写 `@Entity`、`@Table`、`@Column`。

Grails 会把 `grails-app/domain` 下面的类当作 Domain 处理，并通过 GORM 映射到数据库。

几个关键点：

| 写法 | 作用 |
| --- | --- |
| `static constraints` | 校验规则 |
| `static hasMany` | 一对多关系 |
| `static belongsTo` | 从属关系，也会影响级联 |
| `static mapping` | 数据库映射定制 |
| `dateCreated` | 自动维护创建时间 |
| `lastUpdated` | 自动维护更新时间 |

### 常用 constraints

`constraints` 是 Grails 里非常常见的校验写法。

常用规则：

| 规则 | 说明 |
| --- | --- |
| `blank: false` | 字符串不能是空白 |
| `nullable: false` | 不能为 `null` |
| `unique: true` | 唯一 |
| `maxSize: 100` | 最大长度 |
| `size: 3..20` | 长度范围 |
| `min: 1` | 最小值 |
| `max: 120` | 最大值 |
| `email: true` | 邮箱格式 |
| `url: true` | URL 格式 |
| `matches: /regex/` | 正则匹配 |
| `inList: ['A', 'B']` | 必须在列表中 |

Domain 保存时会自动校验。

```groovy
def book = new Book(title: '', isbn: 'abc')
book.validate()
println book.errors
```

如果保存失败不想静默返回 `null`，可以使用：

```groovy
book.save(failOnError: true)
```

这样校验或持久化失败会直接抛异常，开发阶段更容易发现问题。

### 配置数据库

开发阶段可以先用 H2。

`grails-app/conf/application.yml` 示例：

```yaml
dataSource:
  pooled: true
  jmxExport: true
  driverClassName: org.h2.Driver
  username: sa
  password: ''

environments:
  development:
    dataSource:
      dbCreate: update
      url: jdbc:h2:mem:devDb;LOCK_TIMEOUT=10000;DB_CLOSE_ON_EXIT=FALSE
  test:
    dataSource:
      dbCreate: update
      url: jdbc:h2:mem:testDb;LOCK_TIMEOUT=10000;DB_CLOSE_ON_EXIT=FALSE
  production:
    dataSource:
      dbCreate: none
      url: jdbc:h2:./prodDb;LOCK_TIMEOUT=10000;DB_CLOSE_ON_EXIT=FALSE
```

MySQL 示例：

```yaml
dataSource:
  pooled: true
  driverClassName: com.mysql.cj.jdbc.Driver
  username: root
  password: 123456
  url: jdbc:mysql://127.0.0.1:3306/library_demo?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8

environments:
  development:
    dataSource:
      dbCreate: update
  production:
    dataSource:
      dbCreate: none
```

MySQL 驱动依赖放到 `build.gradle`：

```groovy
dependencies {
    runtimeOnly 'com.mysql:mysql-connector-j'
}
```

`dbCreate` 常见值：

| 值 | 说明 | 常见场景 |
| --- | --- | --- |
| `create-drop` | 启动建表，关闭删除 | 临时测试 |
| `create` | 每次启动重建 | 本地演示 |
| `update` | 根据 Domain 更新表结构 | 开发环境 |
| `validate` | 只校验结构 | 测试 / 预发 |
| `none` | 不处理表结构 | 生产环境 |

生产环境不建议依赖 `dbCreate: update` 管表结构。

更合适的方式是数据库迁移工具，比如 Grails Database Migration 插件、Flyway 或 Liquibase。

### 初始化演示数据

Grails 可以用 `BootStrap` 初始化数据。

文件：

```text
grails-app/init/library/demo/BootStrap.groovy
```

代码：

```groovy
package library.demo

import grails.util.Environment

class BootStrap {

    def init = { servletContext ->
        if (Environment.current == Environment.PRODUCTION) {
            log.info 'production environment, skip bootstrap data'
            return
        }

        if (Author.count() > 0) {
            return
        }

        Author.withTransaction {
            Author tolkien = new Author(
                    name: 'J.R.R. Tolkien',
                    biography: 'English writer and philologist',
                    dateOfBirth: Date.parse('yyyy-MM-dd', '1892-01-03')
            ).save(failOnError: true)

            Author leGuin = new Author(
                    name: 'Ursula K. Le Guin',
                    biography: 'American author',
                    dateOfBirth: Date.parse('yyyy-MM-dd', '1929-10-21')
            ).save(failOnError: true)

            new Book(
                    author: tolkien,
                    title: 'The Hobbit',
                    isbn: '9780547928227',
                    pageCount: 310,
                    publishedOn: Date.parse('yyyy-MM-dd', '1937-09-21')
            ).save(failOnError: true)

            new Book(
                    author: leGuin,
                    title: 'A Wizard of Earthsea',
                    isbn: '9780547773742',
                    pageCount: 205,
                    publishedOn: Date.parse('yyyy-MM-dd', '1968-01-01')
            ).save(failOnError: true)
        }
    }

    def destroy = {
    }
}
```

这里特意跳过生产环境，避免生产启动时插入演示数据。

### GORM 基础查询

GORM 是 Grails 的数据访问核心。

它可以直接在 Domain 类上调用方法：

```groovy
Book.get(1L)
Book.list()
Book.count()
Book.exists(1L)
```

新增：

```groovy
def author = Author.findByName('J.R.R. Tolkien')

new Book(
        title: 'The Lord of the Rings',
        isbn: '9780618640157',
        pageCount: 1216,
        author: author
).save(failOnError: true, flush: true)
```

修改：

```groovy
def book = Book.get(1L)
book.pageCount = 320
book.save(failOnError: true)
```

删除：

```groovy
Book.get(1L)?.delete(flush: true)
```

动态查询：

```groovy
Book.findByIsbn('9780547928227')
Book.findAllByTitleLike('%Ring%')
Book.findAllByPageCountGreaterThan(300)
Book.countByAuthor(author)
```

分页排序：

```groovy
Book.list(max: 20, offset: 0, sort: 'id', order: 'desc')
```

Where 查询：

```groovy
def query = Book.where {
    pageCount > 300 && title =~ '%Ring%'
}

List<Book> books = query.list(sort: 'publishedOn', order: 'desc')
```

HQL 查询：

```groovy
Book.executeQuery(
        'select b from Book b where b.author.name = :name',
        [name: 'J.R.R. Tolkien']
)
```

GORM 的价值在于：简单查询不用写 Repository，复杂查询也有多种表达方式。

### Service 层

业务逻辑不要都塞到 Controller。

创建 Service：

```shell
grails create-service library.demo.BookService
```

代码：

```groovy
package library.demo

import grails.gorm.transactions.Transactional

@Transactional
class BookService {

    Book createBook(Map data) {
        Author author = Author.get(data.authorId as Long)
        if (!author) {
            throw new IllegalArgumentException('作者不存在')
        }

        Book book = new Book(
                title: data.title,
                isbn: data.isbn,
                pageCount: data.pageCount as Integer,
                publishedOn: parseDate(data.publishedOn),
                author: author
        )

        book.save(failOnError: true)
        return book
    }

    Book updateBook(Long id, Map data) {
        Book book = Book.get(id)
        if (!book) {
            return null
        }

        if (data.title != null) {
            book.title = data.title
        }
        if (data.isbn != null) {
            book.isbn = data.isbn
        }
        if (data.pageCount != null) {
            book.pageCount = data.pageCount as Integer
        }
        if (data.publishedOn != null) {
            book.publishedOn = parseDate(data.publishedOn)
        }
        if (data.authorId != null) {
            Author author = Author.get(data.authorId as Long)
            if (!author) {
                throw new IllegalArgumentException('作者不存在')
            }
            book.author = author
        }

        book.save(failOnError: true)
        return book
    }

    List<Book> search(String keyword, Integer max, Integer offset) {
        int safeMax = Math.min(max ?: 20, 100)
        int safeOffset = offset ?: 0

        if (!keyword) {
            return Book.list(max: safeMax, offset: safeOffset, sort: 'id', order: 'desc')
        }

        Book.where {
            title =~ "%${keyword}%" || author.name =~ "%${keyword}%"
        }.list(max: safeMax, offset: safeOffset, sort: 'id', order: 'desc')
    }

    void deleteBook(Long id) {
        Book.get(id)?.delete(flush: true)
    }

    private Date parseDate(Object value) {
        if (!value) {
            return null
        }
        if (value instanceof Date) {
            return value
        }
        Date.parse('yyyy-MM-dd', value.toString())
    }
}
```

`@Transactional` 表示 Service 方法在事务里执行。

如果方法抛出运行时异常，事务会回滚。

### REST Controller

创建 Controller：

```shell
grails create-controller library.demo.Book
grails create-controller library.demo.Author
```

作者 Controller：

```groovy
package library.demo

class AuthorController {

    static responseFormats = ['json']

    def index(Integer max, Integer offset) {
        int safeMax = Math.min(max ?: 20, 100)
        respond Author.list(max: safeMax, offset: offset ?: 0, sort: 'id', order: 'asc')
    }

    def show(Long id) {
        Author author = Author.get(id)
        if (!author) {
            render status: 404, text: '作者不存在'
            return
        }

        respond author
    }

    def save() {
        Author author = new Author(request.JSON)

        if (!author.validate()) {
            respond author.errors, status: 422
            return
        }

        author.save(flush: true)
        respond author, status: 201
    }
}
```

图书 Controller：

```groovy
package library.demo

class BookController {

    static responseFormats = ['json']
    static allowedMethods = [
            save  : 'POST',
            update: 'PUT',
            delete: 'DELETE'
    ]

    BookService bookService

    def index(String keyword, Integer max, Integer offset) {
        respond bookService.search(keyword, max, offset)
    }

    def show(Long id) {
        Book book = Book.get(id)
        if (!book) {
            render status: 404, text: '图书不存在'
            return
        }

        respond book
    }

    def save() {
        try {
            Book book = bookService.createBook(request.JSON as Map)
            respond book, status: 201
        } catch (IllegalArgumentException e) {
            render status: 400, text: e.message
        }
    }

    def update(Long id) {
        try {
            Book book = bookService.updateBook(id, request.JSON as Map)
            if (!book) {
                render status: 404, text: '图书不存在'
                return
            }
            respond book
        } catch (IllegalArgumentException e) {
            render status: 400, text: e.message
        }
    }

    def delete(Long id) {
        bookService.deleteBook(id)
        render status: 204
    }
}
```

关键点：

| 写法 | 作用 |
| --- | --- |
| `static responseFormats = ['json']` | 默认返回 JSON |
| `static allowedMethods` | 限制动作使用的 HTTP 方法 |
| `request.JSON` | 获取 JSON 请求体 |
| `respond object` | 按内容协商返回响应 |
| `render status: 404` | 直接返回状态码 |

### URL 映射

默认 URL 是：

```text
/$controller/$action?/$id?
```

REST API 通常需要更清晰的路径。

修改：

```text
grails-app/controllers/library/demo/UrlMappings.groovy
```

示例：

```groovy
package library.demo

class UrlMappings {

    static mappings = {
        group "/v1", {
            "/authors"(resources: "author")
            "/books"(resources: "book")
        }

        "/"(view: "/index")
        "404"(view: '/notFound')
        "500"(view: '/error')
    }
}
```

这样就能使用：

```text
GET    /v1/books
GET    /v1/books/1
POST   /v1/books
PUT    /v1/books/1
DELETE /v1/books/1
```

### 测试 REST API

启动：

```shell
./gradlew bootRun
```

查询图书：

```shell
curl http://localhost:8080/v1/books
```

查询作者：

```shell
curl http://localhost:8080/v1/authors
```

新增作者：

```shell
curl -X POST http://localhost:8080/v1/authors \
  -H 'Content-Type: application/json' \
  -d '{"name":"Isaac Asimov","biography":"Science fiction writer"}'
```

新增图书：

```shell
curl -X POST http://localhost:8080/v1/books \
  -H 'Content-Type: application/json' \
  -d '{
        "title":"Foundation",
        "isbn":"9780553293357",
        "pageCount":255,
        "publishedOn":"1951-01-01",
        "authorId":3
      }'
```

搜索图书：

```shell
curl 'http://localhost:8080/v1/books?keyword=Foundation&max=10'
```

修改图书：

```shell
curl -X PUT http://localhost:8080/v1/books/1 \
  -H 'Content-Type: application/json' \
  -d '{"pageCount":320}'
```

删除图书：

```shell
curl -X DELETE http://localhost:8080/v1/books/1
```

### 用 RestfulController 少写 CRUD

如果接口是标准 CRUD，可以继承 `RestfulController`。

```groovy
package library.demo

import grails.rest.RestfulController

class BookRestController extends RestfulController<Book> {

    static responseFormats = ['json']

    BookRestController() {
        super(Book)
    }

    @Override
    protected List<Book> listAllResources(Map params) {
        params.max = Math.min((params.int('max') ?: 20), 100)
        Book.list(params)
    }
}
```

这种写法会自动提供常见 REST 动作。

适合简单资源。

如果业务逻辑比较多，显式写 Controller + Service 更清楚。

### GSP 页面 Demo

Grails 不只能写 REST API，也能写服务端页面。

下面加一个简单图书列表页。

Controller：

```groovy
package library.demo

class BookPageController {

    def index() {
        [
                books: Book.list(max: 20, sort: 'id', order: 'desc'),
                total: Book.count()
        ]
    }
}
```

URL 映射：

```groovy
"/books-page"(controller: "bookPage", action: "index")
```

视图文件：

```text
grails-app/views/bookPage/index.gsp
```

代码：

```gsp
<!doctype html>
<html>
<head>
    <meta name="layout" content="main"/>
    <title>图书列表</title>
</head>
<body>
    <h1>图书列表</h1>

    <p>总数：${total}</p>

    <table>
        <thead>
        <tr>
            <th>ID</th>
            <th>书名</th>
            <th>作者</th>
            <th>ISBN</th>
            <th>页数</th>
        </tr>
        </thead>
        <tbody>
        <g:each in="${books}" var="book">
            <tr>
                <td>${book.id}</td>
                <td>${book.title}</td>
                <td>${book.author?.name}</td>
                <td>${book.isbn}</td>
                <td>${book.pageCount}</td>
            </tr>
        </g:each>
        </tbody>
    </table>
</body>
</html>
```

访问：

```text
http://localhost:8080/books-page
```

常见 GSP 标签：

| 标签 | 作用 |
| --- | --- |
| `<g:if>` | 条件判断 |
| `<g:else>` | 否则分支 |
| `<g:each>` | 遍历 |
| `<g:link>` | 生成链接 |
| `<g:form>` | 生成表单 |
| `<g:fieldError>` | 显示字段错误 |
| `<g:paginate>` | 分页 |

GSP 适合后台页面、内部系统、服务端渲染页面。

前后端分离项目通常用 JSON API。

### 脚手架

Grails 的脚手架可以快速生成 CRUD。

给 Domain 生成 Controller 和 GSP：

```shell
grails generate-all library.demo.Book
```

通常会生成：

```text
grails-app/controllers/library/demo/BookController.groovy
grails-app/views/book/index.gsp
grails-app/views/book/show.gsp
grails-app/views/book/create.gsp
grails-app/views/book/edit.gsp
```

也可以只生成 Controller：

```shell
grails generate-controller library.demo.Book
```

脚手架适合：

* 快速原型
* 后台 CRUD
* 学习 Grails 约定
* 生成后再手动调整

不建议长期完全依赖脚手架不改代码。

实际项目里通常是：

```text
先 generate-all 跑通
再把业务逻辑挪到 Service
再整理权限、校验、异常、页面和接口返回
```

### Command Object

表单或接口参数比较复杂时，可以用命令对象。

比如搜索图书：

```groovy
package library.demo

class BookSearchCommand {

    String keyword
    Integer max = 20
    Integer offset = 0

    static constraints = {
        keyword nullable: true, maxSize: 100
        max min: 1, max: 100
        offset min: 0
    }
}
```

Controller：

```groovy
class BookController {

    BookService bookService

    def search(BookSearchCommand cmd) {
        if (cmd.hasErrors()) {
            respond cmd.errors, status: 422
            return
        }

        respond bookService.search(cmd.keyword, cmd.max, cmd.offset)
    }
}
```

好处：

* 参数绑定集中
* 参数校验集中
* Controller 少写 `params.xxx`
* 复杂表单更清楚

### JSON Views

`respond book` 虽然方便，但复杂项目里经常需要控制 JSON 输出格式。

比如不想直接暴露 Domain 的全部字段。

可以使用 JSON Views。

示例文件：

```text
grails-app/views/book/show.gson
```

代码：

```groovy
import library.demo.Book

model {
    Book book
}

json {
    id book.id
    title book.title
    isbn book.isbn
    pageCount book.pageCount
    author {
        id book.author.id
        name book.author.name
    }
}
```

列表：

```text
grails-app/views/book/index.gson
```

```groovy
import library.demo.Book

model {
    Iterable<Book> bookList
}

json books: bookList.collect { Book book ->
    [
            id       : book.id,
            title    : book.title,
            isbn     : book.isbn,
            pageCount: book.pageCount,
            author   : [
                    id  : book.author.id,
                    name: book.author.name
            ]
    ]
}
```

JSON Views 适合正式 API。

它可以避免 Domain 结构和接口响应强绑定。

### 测试

Grails 常用 Spock 写测试。

Service 单元测试示例：

```groovy
package library.demo

import grails.testing.gorm.DataTest
import grails.testing.services.ServiceUnitTest
import spock.lang.Specification

class BookServiceSpec extends Specification
        implements ServiceUnitTest<BookService>, DataTest {

    void setupSpec() {
        mockDomains Book, Author
    }

    void "创建图书时作者不存在应该抛异常"() {
        when:
        service.createBook([
                title   : 'Foundation',
                isbn    : '9780553293357',
                authorId: 999
        ])

        then:
        thrown(IllegalArgumentException)
    }
}
```

Controller 测试示例：

```groovy
package library.demo

import grails.testing.web.controllers.ControllerUnitTest
import spock.lang.Specification

class AuthorControllerSpec extends Specification
        implements ControllerUnitTest<AuthorController> {

    void "show 找不到作者时返回 404"() {
        when:
        controller.show(1L)

        then:
        response.status == 404
    }
}
```

运行测试：

```shell
./gradlew test
```

或：

```shell
grails test-app
```

集成测试通常放在：

```text
src/integration-test/groovy
```

适合测试真实 Spring 上下文、数据库访问、Controller 到 Service 的完整链路。

### 打包和部署

Grails 底层是 Spring Boot 应用。

常见打包：

```shell
./gradlew assemble
```

运行 Jar：

```shell
java -jar build/libs/library-demo-0.1.jar
```

也可以打 WAR：

```shell
./gradlew war
```

生产环境常见配置：

```shell
java -Dgrails.env=prod -jar build/libs/library-demo-0.1.jar
```

外部配置可以放到环境变量或外部配置文件里。

常见生产关注点：

* 数据库迁移
* 日志级别
* 连接池配置
* 健康检查
* JVM 参数
* 静态资源
* 反向代理
* 安全配置

### 常用命令

| 命令 | 作用 |
| --- | --- |
| `grails create-app app-name` | 创建项目 |
| `grails create-domain-class package.Book` | 创建 Domain |
| `grails create-controller package.Book` | 创建 Controller |
| `grails create-service package.BookService` | 创建 Service |
| `grails generate-all package.Book` | 生成 CRUD Controller 和视图 |
| `grails run-app` | 运行应用 |
| `grails test-app` | 运行测试 |
| `./gradlew bootRun` | 通过 Gradle 运行 |
| `./gradlew test` | 通过 Gradle 测试 |
| `./gradlew assemble` | 打包 |
| `./gradlew war` | 打 WAR 包 |

Grails 8 以后 CLI 体系有 Shell CLI 和 Forge CLI 的区别。

日常开发优先用项目里的：

```shell
./grailsw
./gradlew
```

### 常见坑

### Domain 目录写错

正确目录通常是：

```text
grails-app/domain
```

不是：

```text
grails-app/domains
```

目录写错后，Grails 不会按 Domain 类处理，GORM 方法也不会按预期工作。

### save 失败但没有报错

GORM 的 `save()` 校验失败时可能返回 `null`。

容易出现这种代码：

```groovy
book.save()
println book.id
```

如果校验没过，`id` 还是空。

开发阶段建议：

```groovy
book.save(failOnError: true)
```

或者显式检查：

```groovy
if (!book.save()) {
    println book.errors
}
```

### 生产环境使用 dbCreate update

开发环境可以用：

```yaml
dbCreate: update
```

生产环境不建议这样做。

生产库表结构应该走迁移脚本和发布流程。

### Controller 里写太多业务逻辑

不推荐：

```groovy
class BookController {
    def save() {
        // 参数校验、查作者、创建图书、复杂事务全部写这里
    }
}
```

更适合：

```text
Controller 处理请求和响应
Service 处理业务和事务
Domain 表达模型和约束
```

### 直接返回 Domain 导致 JSON 失控

简单 Demo 可以：

```groovy
respond book
```

正式 API 更建议使用：

* JSON Views
* DTO
* 明确字段白名单

这样可以避免敏感字段、懒加载关系、循环引用等问题。

### GORM 懒加载导致 N+1

作者列表里如果循环访问 `author.books`，可能触发很多 SQL。

可以按需要使用 fetch join：

```groovy
Author.list(fetch: [books: 'join'])
```

或者专门为列表接口设计查询和响应字段。

### javax 和 jakarta 混用

Grails 新版本跟随 Spring Boot 3 / 4 后进入 Jakarta 体系。

老代码里如果大量使用：

```groovy
import javax.servlet.http.HttpServletRequest
```

迁移时要改成：

```groovy
import jakarta.servlet.http.HttpServletRequest
```

依赖也要跟着换，否则容易出现类找不到或类型不匹配。

### 总结

Grails 可以按这条线理解：

```text
Groovy 提供简洁语法
Spring Boot 提供运行底座
GORM 负责数据库访问
Domain 定义数据模型和校验
Controller 处理 Web 请求
Service 承载业务逻辑和事务
GSP / JSON Views 负责响应展示
Gradle 负责构建和运行
```

日常开发顺序通常是：

* 创建 Domain
* 写 constraints 和关系
* 配置数据源
* 初始化演示数据
* 写 Service
* 写 Controller
* 配 URL 映射
* 用 curl 或页面测试
* 补测试和异常处理
* 最后考虑部署和数据库迁移

Grails 的优势在于把常见 Web 应用的固定套路收进框架约定里。CRUD、校验、分页、关系映射、脚手架、视图、REST API 都有现成路径。项目越偏业务表单和数据管理，Grails 的效率优势越明显；项目越偏底层基础设施和强定制，Spring Boot 的显式组合方式往往更合适。
