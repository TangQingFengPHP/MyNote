### 简介

项目代码写久了，很容易出现这种函数签名：

```kotlin
fun createOrder(
    userId: Long,
    productId: Long,
    logger: Logger,
    config: AppConfig,
    tx: Transaction
) {
    logger.info("create order")
    tx.execute("insert order")
}
```

真正的业务参数只有 `userId` 和 `productId`。

后面的 `logger`、`config`、`tx` 更像“运行环境”：

* 日志对象
* 配置对象
* 当前登录用户
* 数据库事务
* 权限能力
* 请求上下文
* 监控埋点对象

这些对象通常不会在每一层变化，但又经常被一层层传下去。

于是代码就变成了这样：

```kotlin
fun controller(logger: Logger, config: AppConfig, tx: Transaction) {
    service(logger, config, tx)
}

fun service(logger: Logger, config: AppConfig, tx: Transaction) {
    repository(logger, config, tx)
}

fun repository(logger: Logger, config: AppConfig, tx: Transaction) {
    logger.info(config.env)
    tx.execute("insert")
}
```

业务还没展开，参数先排了一长串。

Kotlin Context Parameters 就是为了解决这类问题：

> 函数可以声明自己需要哪些上下文，调用时只要当前作用域里有这些上下文，编译器会自动匹配，不用手动一层层传。

它不是全局变量，也不是运行时反射注入，而是编译期能检查的隐式参数。

### 版本说明

Context Parameters 从 Kotlin 2.2.0 开始进入预览，到了 Kotlin 2.4.0 已经稳定。

如果使用 Kotlin 2.4.0 及以上版本，普通 Context Parameters 不再需要额外开启实验参数。

如果使用 Kotlin 2.2.x 或 2.3.x，需要在 `build.gradle.kts` 里开启：

```kotlin
kotlin {
    compilerOptions {
        freeCompilerArgs.add("-Xcontext-parameters")
    }
}
```

注意一点：

```text
-Xcontext-receivers 和 -Xcontext-parameters 不要同时打开
```

`Context Receivers` 是旧实验特性，`Context Parameters` 是替代方案。

Kotlin 2.4.0 里还有一个能力叫显式上下文参数调用，用来解决部分重载歧义。这个能力仍是实验性的，需要额外开启：

```kotlin
kotlin {
    compilerOptions {
        freeCompilerArgs.add("-Xexplicit-context-arguments")
    }
}
```

普通使用先不用急着碰它，后面会单独讲。

### 第一个问题：Logger 到底还要传多少层

先定义一个日志接口：

```kotlin
interface Logger {
    fun info(message: String)
}

class ConsoleLogger : Logger {
    override fun info(message: String) {
        println("[INFO] $message")
    }
}
```

普通写法：

```kotlin
fun saveUser(name: String, logger: Logger) {
    logger.info("save user: $name")
    println("保存用户 $name")
}

fun main() {
    val logger = ConsoleLogger()
    saveUser("Tom", logger)
}
```

输出：

```text
[INFO] save user: Tom
保存用户 Tom
```

这段代码没问题。

问题出现在调用链变长之后：

```kotlin
fun createUser(name: String, logger: Logger) {
    validateUser(name, logger)
    saveUser(name, logger)
}

fun validateUser(name: String, logger: Logger) {
    logger.info("validate user: $name")
}

fun saveUser(name: String, logger: Logger) {
    logger.info("save user: $name")
}
```

每一层都要带着 `logger` 走。

换成 Context Parameters：

```kotlin
context(logger: Logger)
fun createUser(name: String) {
    validateUser(name)
    saveUser(name)
}

context(logger: Logger)
fun validateUser(name: String) {
    logger.info("validate user: $name")
}

context(logger: Logger)
fun saveUser(name: String) {
    logger.info("save user: $name")
}

fun main() {
    val logger = ConsoleLogger()

    context(logger) {
        createUser("Tom")
    }
}
```

输出：

```text
[INFO] validate user: Tom
[INFO] save user: Tom
```

关键语法只有两个：

```kotlin
context(logger: Logger)
fun saveUser(name: String) {
    logger.info("save user: $name")
}
```

表示这个函数需要一个 `Logger` 上下文，并且在函数体里用名字 `logger` 访问它。

调用时：

```kotlin
context(logger) {
    createUser("Tom")
}
```

表示这段代码块里提供了一个 `Logger` 上下文。

### 可以把它理解成“编译器帮忙传参数”

下面这段：

```kotlin
context(logger: Logger)
fun saveUser(name: String) {
    logger.info("save user: $name")
}
```

可以先粗略理解成：

```kotlin
fun saveUser(name: String, logger: Logger) {
    logger.info("save user: $name")
}
```

区别在于：

```text
普通参数：调用时手动传
Context Parameters：编译器从当前上下文里找
```

所以 Context Parameters 不是“凭空出现的依赖”。

没有上下文时，代码直接编译失败：

```kotlin
fun main() {
    // 编译失败：缺少 Logger 上下文
    saveUser("Tom")
}
```

这比运行时才发现缺 Bean、缺 Service、缺配置要早得多。

### 多个上下文：日志、配置、数据库一起用

实际项目不会只有一个 `Logger`。

再加两个对象：

```kotlin
data class AppConfig(
    val env: String,
    val enableAudit: Boolean
)

interface Database {
    fun insert(table: String, values: Map<String, Any>)
}

class MemoryDatabase : Database {
    override fun insert(table: String, values: Map<String, Any>) {
        println("INSERT INTO $table VALUES $values")
    }
}
```

一个函数可以声明多个上下文：

```kotlin
context(
    logger: Logger,
    config: AppConfig,
    db: Database
)
fun registerUser(name: String) {
    logger.info("env=${config.env}, register user: $name")

    db.insert(
        table = "users",
        values = mapOf(
            "name" to name,
            "audit" to config.enableAudit
        )
    )
}
```

调用：

```kotlin
fun main() {
    val logger = ConsoleLogger()
    val config = AppConfig(env = "dev", enableAudit = true)
    val db = MemoryDatabase()

    context(logger, config, db) {
        registerUser("Lucy")
    }
}
```

输出：

```text
[INFO] env=dev, register user: Lucy
INSERT INTO users VALUES {name=Lucy, audit=true}
```

编译器按类型查找当前上下文里的对象：

```text
Logger     -> logger
AppConfig  -> config
Database   -> db
```

函数签名里只保留真正的业务参数：

```kotlin
registerUser("Lucy")
```

这就是它最直接的价值。

### 用 context 块，还是用 with

提供上下文常见写法是 `context(...) { }`：

```kotlin
context(logger, config, db) {
    registerUser("Lucy")
}
```

有些示例也会看到 `with`：

```kotlin
with(logger) {
    registerUser("Lucy")
}
```

单个上下文时，`with(logger)` 能工作，因为 `logger` 成为当前接收者，编译器也能把它当作上下文候选。

多个上下文时，用 `context(...) { }` 更清楚：

```kotlin
context(logger, config, db) {
    registerUser("Lucy")
}
```

不需要写成多层嵌套：

```kotlin
with(logger) {
    with(config) {
        with(db) {
            registerUser("Lucy")
        }
    }
}
```

代码一多，`context(...) { }` 更适合表达“这里提供一组环境”。

### Demo：事务范围，不再到处传 tx

事务是 Context Parameters 很适合的场景。

先定义一个事务对象：

```kotlin
interface Transaction {
    fun begin()
    fun commit()
    fun rollback()
    fun execute(sql: String)
}

class ConsoleTransaction : Transaction {
    override fun begin() = println("BEGIN")
    override fun commit() = println("COMMIT")
    override fun rollback() = println("ROLLBACK")
    override fun execute(sql: String) = println("SQL: $sql")
}
```

再写几个需要事务的函数：

```kotlin
data class OrderItem(
    val productId: Long,
    val count: Int
)

context(tx: Transaction, logger: Logger)
fun createOrder(userId: Long, items: List<OrderItem>) {
    logger.info("create order for user=$userId")
    tx.execute("insert into orders(user_id) values($userId)")

    items.forEach { item ->
        createOrderItem(item)
    }
}

context(tx: Transaction, logger: Logger)
fun createOrderItem(item: OrderItem) {
    logger.info("create order item product=${item.productId}")
    tx.execute(
        "insert into order_items(product_id, count) values(${item.productId}, ${item.count})"
    )
}
```

封装一个事务入口：

```kotlin
context(tx: Transaction, logger: Logger)
fun <T> transactional(block: context(Transaction, Logger) () -> T): T {
    tx.begin()

    return try {
        val result = block()
        tx.commit()
        result
    } catch (e: Exception) {
        tx.rollback()
        logger.info("transaction rollback: ${e.message}")
        throw e
    }
}
```

完整调用：

```kotlin
fun main() {
    val tx = ConsoleTransaction()
    val logger = ConsoleLogger()

    context(tx, logger) {
        transactional {
            createOrder(
                userId = 1001,
                items = listOf(
                    OrderItem(productId = 10, count = 2),
                    OrderItem(productId = 20, count = 1)
                )
            )
        }
    }
}
```

输出：

```text
BEGIN
[INFO] create order for user=1001
SQL: insert into orders(user_id) values(1001)
[INFO] create order item product=10
SQL: insert into order_items(product_id, count) values(10, 2)
[INFO] create order item product=20
SQL: insert into order_items(product_id, count) values(20, 1)
COMMIT
```

这里的重点不是少写几个参数，而是事务边界变清楚了：

```kotlin
context(tx, logger) {
    transactional {
        createOrder(...)
    }
}
```

这段代码表达的含义很直接：

```text
在这个事务和日志上下文里，执行一组订单逻辑
```

### `block: context(...) () -> T` 是什么

上面这段可能有点陌生：

```kotlin
fun <T> transactional(block: context(Transaction, Logger) () -> T): T
```

它表示：

```text
block 是一个函数
这个函数执行时需要 Transaction 和 Logger 上下文
这个函数最终返回 T
```

普通函数类型长这样：

```kotlin
() -> T
```

带上下文的函数类型长这样：

```kotlin
context(Transaction, Logger) () -> T
```

所以 `transactional { ... }` 里的代码可以直接调用需要 `Transaction`、`Logger` 的函数。

这种写法很适合封装：

* 事务
* 权限范围
* 请求范围
* 临时配置范围
* DSL 构建范围

### Demo：权限不是 if，而是一种编译期能力

很多系统里，删除用户必须有管理员权限。

常见写法：

```kotlin
fun deleteUser(id: Long, currentUser: CurrentUser) {
    if (!currentUser.isAdmin) {
        error("无权限")
    }

    println("delete user $id")
}
```

这种写法是运行时检查。

Context Parameters 可以把“能不能调用”提前到编译期。

先定义一个能力类型：

```kotlin
interface AdminPermission

object Admin : AdminPermission
```

删除函数声明自己需要管理员能力：

```kotlin
context(permission: AdminPermission, logger: Logger)
fun deleteUser(id: Long) {
    logger.info("delete user: $id")
    println("用户 $id 已删除")
}
```

调用：

```kotlin
fun main() {
    val logger = ConsoleLogger()

    context(Admin, logger) {
        deleteUser(1001)
    }
}
```

输出：

```text
[INFO] delete user: 1001
用户 1001 已删除
```

没有 `AdminPermission` 上下文时：

```kotlin
fun main() {
    val logger = ConsoleLogger()

    context(logger) {
        // 编译失败：缺少 AdminPermission
        deleteUser(1001)
    }
}
```

这类写法常被称为 Capability-Based Design，能力驱动设计。

`deleteUser` 的函数签名已经把限制写出来了：

```kotlin
context(permission: AdminPermission, logger: Logger)
fun deleteUser(id: Long)
```

含义是：

```text
只有拿到 AdminPermission 能力的代码区域，才能调用 deleteUser
```

### Demo：小型 HTML DSL

Context Parameters 不只适合依赖注入，也适合 DSL。

先定义一个构建器：

```kotlin
class HtmlBuilder {
    private val lines = mutableListOf<String>()

    fun tag(name: String, content: String) {
        lines += "<$name>$content</$name>"
    }

    fun build(): String {
        return lines.joinToString("\n")
    }
}
```

定义几个 DSL 函数：

```kotlin
context(html: HtmlBuilder)
fun h1(text: String) {
    html.tag("h1", text)
}

context(html: HtmlBuilder)
fun p(text: String) {
    html.tag("p", text)
}

context(html: HtmlBuilder)
fun button(text: String) {
    html.tag("button", text)
}
```

再写一个入口函数：

```kotlin
fun html(block: context(HtmlBuilder) () -> Unit): String {
    val builder = HtmlBuilder()

    context(builder) {
        block()
    }

    return builder.build()
}
```

使用：

```kotlin
fun main() {
    val page = html {
        h1("Kotlin Context Parameters")
        p("把共享上下文放进作用域，让业务参数保持干净。")
        button("开始")
    }

    println(page)
}
```

输出：

```text
<h1>Kotlin Context Parameters</h1>
<p>把共享上下文放进作用域，让业务参数保持干净。</p>
<button>开始</button>
```

这个 DSL 的关键点是：

```kotlin
fun html(block: context(HtmlBuilder) () -> Unit): String
```

`html { }` 块里的函数都需要 `HtmlBuilder` 上下文，而入口函数负责创建并提供这个上下文。

调用方不用看到 `HtmlBuilder`，也不用手动传它。

### 带上下文的属性

Context Parameters 不只支持函数，也支持属性。

示例：

```kotlin
data class RequestContext(
    val requestId: String,
    val userId: Long
)

context(request: RequestContext)
val requestTag: String
    get() = "requestId=${request.requestId}, userId=${request.userId}"
```

使用：

```kotlin
fun main() {
    val request = RequestContext(
        requestId = "req-001",
        userId = 1001
    )

    context(request) {
        println(requestTag)
    }
}
```

输出：

```text
requestId=req-001, userId=1001
```

不过带上下文的属性有几个限制：

* 不能有幕后字段
* 不能写初始化器
* 不能使用委托

所以这种属性一般写成计算属性：

```kotlin
context(request: RequestContext)
val requestTag: String
    get() = "requestId=${request.requestId}, userId=${request.userId}"
```

不要写成：

```kotlin
// 错误写法
context(request: RequestContext)
val requestTag: String = request.requestId
```

### 匿名上下文参数 `_`

有些函数不直接使用上下文对象，只是要求调用者处在某种能力范围内。

例如只要求有管理员权限，但函数体不关心权限对象本身：

```kotlin
interface AuditPermission

object AuditAdmin : AuditPermission

context(_: AuditPermission, logger: Logger)
fun exportAuditLog() {
    logger.info("export audit log")
    println("导出审计日志")
}
```

`_` 表示：

```text
需要这个上下文，但函数体内不通过名字访问它
```

调用：

```kotlin
fun main() {
    val logger = ConsoleLogger()

    context(AuditAdmin, logger) {
        exportAuditLog()
    }
}
```

这种写法常见于“能力证明”：

```text
只要能进入这个上下文，就说明具备调用资格
```

### 同类型上下文会歧义

Context Parameters 按类型匹配上下文。

如果同一层作用域里有两个同类型对象，就会产生歧义：

```kotlin
interface MessageSender {
    fun send(message: String)
}

class EmailSender : MessageSender {
    override fun send(message: String) {
        println("email: $message")
    }
}

class SmsSender : MessageSender {
    override fun send(message: String) {
        println("sms: $message")
    }
}

context(sender: MessageSender)
fun notifyUser(message: String) {
    sender.send(message)
}

fun main() {
    val email = EmailSender()
    val sms = SmsSender()

    context(email, sms) {
        // 编译失败：两个 MessageSender 都能匹配
        notifyUser("hello")
    }
}
```

编译器不会随便选一个。

解决方式之一是使用更具体的上下文类型：

```kotlin
context(sender: EmailSender)
fun sendEmail(message: String) {
    sender.send(message)
}

context(sender: SmsSender)
fun sendSms(message: String) {
    sender.send(message)
}
```

调用：

```kotlin
context(email, sms) {
    sendEmail("hello")
    sendSms("hello")
}
```

如果需要同名函数重载，Kotlin 2.4.0 提供了显式上下文参数调用，但这个能力仍是实验性的。

示例：

```kotlin
context(email: EmailSender)
fun sendNotification() {
    email.send("邮件通知")
}

context(sms: SmsSender)
fun sendNotification() {
    sms.send("短信通知")
}

context(defaultEmail: EmailSender, defaultSms: SmsSender)
fun notifyAllChannels() {
    sendNotification(email = defaultEmail)
    sendNotification(sms = defaultSms)
}
```

需要开启：

```kotlin
freeCompilerArgs.add("-Xexplicit-context-arguments")
```

普通业务代码里，更建议先用清晰的函数名或更具体的类型，少制造重载歧义。

### 和 Context Receivers 的区别

旧 Context Receivers 写法大致是这样：

```kotlin
context(Logger)
fun saveUser(name: String) {
    info("save user: $name")
}
```

`Logger` 像一个隐式 receiver，函数体里可以直接调用 `info`。

Context Parameters 写法：

```kotlin
context(logger: Logger)
fun saveUser(name: String) {
    logger.info("save user: $name")
}
```

最大区别：

```text
Context Receivers：上下文像隐式 this
Context Parameters：上下文是有名字的参数
```

新写法看起来稍微啰嗦一点，但好处很明显：

```kotlin
context(logger: Logger, auditLogger: Logger)
fun saveUser(name: String) {
    logger.info("save user: $name")
    auditLogger.info("audit user: $name")
}
```

两个 `Logger` 角色不同，名字直接写清楚。

旧写法里，多个 receiver 混在同一个作用域，很容易看不出方法到底来自哪里。

### 和 DI 框架是什么关系

Context Parameters 很像轻量版依赖注入，但它不是 Spring、Koin、Dagger 的替代品。

可以这样理解：

```text
DI 框架：负责创建对象、管理生命周期、组装依赖图
Context Parameters：负责把已经存在的上下文带进一段代码
```

适合 Context Parameters 的场景：

* 日志、事务、配置等横切对象
* 请求范围内的用户、租户、traceId
* 小型模块内部的依赖传递
* DSL 作用域
* 权限能力约束
* 测试时替换上下文实现

不适合直接替代 DI 框架的场景：

* 大型对象图自动装配
* 复杂生命周期管理
* 多环境 Bean 扫描
* AOP、拦截器、代理增强
* 框架级组件管理

更实际的用法是两者配合。

例如 Spring 负责创建 `UserRepository`、`Logger`、`TransactionManager`，业务模块内部用 Context Parameters 减少重复透传。

### 和全局单例的区别

全局单例也能少传参数：

```kotlin
object GlobalLogger {
    fun info(message: String) {
        println(message)
    }
}

fun saveUser(name: String) {
    GlobalLogger.info("save user: $name")
}
```

看起来简单，但问题不少：

* 测试替换麻烦
* 并发场景容易混乱
* 多租户、多请求上下文不好处理
* 函数签名看不出依赖
* 代码耦合到具体实现

Context Parameters 的依赖写在函数签名上：

```kotlin
context(logger: Logger)
fun saveUser(name: String)
```

调用处必须提供上下文：

```kotlin
context(TestLogger()) {
    saveUser("Tom")
}
```

依赖不是全局到处可见，而是只在指定作用域内可见。

### 什么时候适合用

适合用 Context Parameters 的代码，通常有这些特征：

```text
依赖在一段调用链里经常使用
依赖本身不是业务输入
依赖应该受作用域限制
缺少依赖时希望编译期报错
```

典型例子：

* `Logger`
* `Transaction`
* `RequestContext`
* `CurrentUser`
* `TenantContext`
* `AppConfig`
* `Clock`
* `CoroutineScope`
* `HtmlBuilder`
* `AdminPermission`

不太适合用的场景：

```text
只在一个函数里用一次的普通参数
业务含义很强的输入参数
每次调用都明显不同的值
读代码时必须明确看到的核心数据
```

例如 `userId`、`amount`、`productId`、`pageSize` 这类值，通常还是普通参数更清楚。

不要为了少写参数，把所有东西都塞进上下文。

### 一个完整小案例：请求上下文 + 日志 + 订单服务

最后串一个更接近业务的示例。

定义上下文：

```kotlin
data class RequestContext(
    val requestId: String,
    val userId: Long,
    val tenantId: String
)

interface OrderRepository {
    fun save(userId: Long, productId: Long)
}

class MemoryOrderRepository : OrderRepository {
    override fun save(userId: Long, productId: Long) {
        println("save order: userId=$userId, productId=$productId")
    }
}
```

定义业务函数：

```kotlin
context(
    request: RequestContext,
    logger: Logger,
    repo: OrderRepository
)
fun placeOrder(productId: Long) {
    logger.info(
        "requestId=${request.requestId}, tenant=${request.tenantId}, place order"
    )

    checkProduct(productId)
    repo.save(request.userId, productId)
}

context(request: RequestContext, logger: Logger)
fun checkProduct(productId: Long) {
    logger.info("user=${request.userId}, check product=$productId")
}
```

调用：

```kotlin
fun main() {
    val request = RequestContext(
        requestId = "req-20260610-001",
        userId = 1001,
        tenantId = "cn"
    )
    val logger = ConsoleLogger()
    val repo = MemoryOrderRepository()

    context(request, logger, repo) {
        placeOrder(productId = 9527)
    }
}
```

输出：

```text
[INFO] requestId=req-20260610-001, tenant=cn, place order
[INFO] user=1001, check product=9527
save order: userId=1001, productId=9527
```

`placeOrder` 的业务参数只有一个：

```kotlin
productId: Long
```

请求信息、日志、仓储对象都来自上下文。

函数签名仍然保留了依赖说明：

```kotlin
context(
    request: RequestContext,
    logger: Logger,
    repo: OrderRepository
)
fun placeOrder(productId: Long)
```

读签名就能知道：

```text
这段逻辑必须在请求上下文、日志上下文、订单仓储上下文里运行
```

### 常见坑

#### 不是运行时注入

Context Parameters 是编译期解析。

缺少上下文时，编译失败，不会等到运行时报错。

#### 不是全局变量

上下文只在当前作用域和内部调用链里可见。

```kotlin
context(logger) {
    saveUser("Tom")
}

// 这里已经离开 logger 上下文
```

#### 同类型对象别放太多

多个同类型上下文容易歧义。

如果一个模块里经常同时需要两个 `Logger`，最好拆成不同接口：

```kotlin
interface BusinessLogger : Logger
interface AuditLogger : Logger
```

然后分别声明：

```kotlin
context(
    businessLogger: BusinessLogger,
    auditLogger: AuditLogger
)
fun saveUser(name: String) {
    businessLogger.info("save user: $name")
    auditLogger.info("audit user: $name")
}
```

比两个 `Logger` 混在一起更清楚。

#### 不要隐藏核心业务参数

下面这种写法不推荐：

```kotlin
data class OrderInput(
    val userId: Long,
    val productId: Long
)

context(input: OrderInput)
fun placeOrder() {
    println(input.productId)
}
```

`userId`、`productId` 是业务输入，不是环境。

更清楚的写法：

```kotlin
context(logger: Logger, repo: OrderRepository)
fun placeOrder(userId: Long, productId: Long) {
    logger.info("place order")
    repo.save(userId, productId)
}
```

上下文应该放“环境”和“能力”，不要拿来藏业务数据。

### 总结

Context Parameters 的核心很简单：

```kotlin
context(logger: Logger)
fun saveUser(name: String) {
    logger.info("save user: $name")
}
```

它解决的是这类问题：

```text
Logger、Config、Transaction、RequestContext 这类对象到处透传
```

它带来的效果是：

* 业务参数更干净
* 共享上下文不用层层传
* 依赖仍然写在函数签名上
* 缺少上下文时编译期报错
* DSL、事务、权限范围更容易表达

一句话概括：

> Context Parameters 不是为了把参数藏起来，而是把“业务输入”和“运行环境”分开，让函数签名更像真正的业务动作。
