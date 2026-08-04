### 简介

接口最容易被写成两种极端。

一种是空壳：

```kotlin
interface UserService {
}
```

名字看起来很正规，里面没有任何契约。

另一种是万能开关：

```kotlin
interface UserService {
    fun login()
    fun logout()
    fun register()
    fun resetPassword()
    fun updateProfile()
    fun uploadAvatar()
    fun bindPhone()
    fun sendCoupon()
}
```

所有能力都塞进一个接口，实现类被迫实现一堆并不关心的方法。

Kotlin `interface` 真正适合做的事，是定义一组清楚的行为契约：

```text
能做什么
需要哪些属性
哪些行为有默认规则
不同实现之间如何替换
```

接口不是为了让代码看起来“更抽象”，而是为了把变化点隔离出来。支付方式会变、日志落地方式会变、缓存实现会变、消息发送渠道会变，这些地方就很适合用接口。

Kotlin 的接口比 Java 早期接口更灵活。它可以有抽象函数、默认实现、抽象属性、计算属性、接口继承、多接口实现，还能配合 `fun interface`、`by` 委托、`sealed interface` 做更干净的设计。

### 第一个接口：先定义能做什么

先看一个最小例子：通知发送。

```kotlin
interface Notifier {
    fun send(to: String, message: String)
}
```

这个接口只规定一件事：能发送消息。

邮件实现：

```kotlin
class EmailNotifier : Notifier {
    override fun send(to: String, message: String) {
        println("发送邮件到 $to：$message")
    }
}
```

短信实现：

```kotlin
class SmsNotifier : Notifier {
    override fun send(to: String, message: String) {
        println("发送短信到 $to：$message")
    }
}
```

调用方只依赖接口：

```kotlin
class RegisterService(
    private val notifier: Notifier
) {
    fun register(email: String) {
        println("创建账号：$email")
        notifier.send(email, "注册成功")
    }
}

fun main() {
    val service = RegisterService(EmailNotifier())
    service.register("tom@example.com")
}
```

输出：

```text
创建账号：tom@example.com
发送邮件到 tom@example.com：注册成功
```

后面要换成短信、站内信、测试假的通知器，`RegisterService` 不需要改。

```kotlin
val service = RegisterService(SmsNotifier())
```

这就是接口的核心价值：调用方依赖稳定契约，实现细节可以替换。

### 接口里的函数：没有函数体就是抽象函数

接口中的函数没有函数体时，实现类必须 `override`。

```kotlin
interface Payment {
    fun pay(amount: Int): Boolean
}

class BalancePayment : Payment {
    override fun pay(amount: Int): Boolean {
        println("余额支付：$amount")
        return true
    }
}

class BankCardPayment : Payment {
    override fun pay(amount: Int): Boolean {
        println("银行卡支付：$amount")
        return true
    }
}

fun checkout(payment: Payment, amount: Int) {
    val success = payment.pay(amount)
    println("支付结果：$success")
}

fun main() {
    checkout(BalancePayment(), 100)
    checkout(BankCardPayment(), 200)
}
```

输出：

```text
余额支付：100
支付结果：true
银行卡支付：200
支付结果：true
```

`checkout` 不关心具体支付方式，只关心 `Payment` 契约。

接口变量可以接收任意实现类：

```kotlin
val payment: Payment = BalancePayment()
```

这就是多态。真正运行哪个 `pay`，由实际对象决定。

### 默认实现：公共流程可以放进接口

Kotlin 接口函数可以有函数体。

```kotlin
interface Logger {
    fun log(level: String, message: String)

    fun info(message: String) {
        log("INFO", message)
    }

    fun error(message: String) {
        log("ERROR", message)
    }
}
```

实现类只需要完成最核心的 `log`：

```kotlin
class ConsoleLogger : Logger {
    override fun log(level: String, message: String) {
        println("[$level] $message")
    }
}

fun main() {
    val logger = ConsoleLogger()

    logger.info("服务启动")
    logger.error("连接失败")
}
```

输出：

```text
[INFO] 服务启动
[ERROR] 连接失败
```

`info` 和 `error` 是接口里的默认实现。实现类可以直接复用，也可以覆盖。

```kotlin
class JsonLogger : Logger {
    override fun log(level: String, message: String) {
        println("""{"level":"$level","message":"$message"}""")
    }

    override fun error(message: String) {
        log("ERROR", "需要报警：$message")
    }
}
```

默认实现适合放“所有实现都大概率一致”的逻辑。不要把某个实现专属逻辑塞进接口默认方法。

### 接口属性：有名字，但没有字段

Kotlin 接口可以声明属性。

```kotlin
interface Account {
    val id: Long
    val name: String
}

data class UserAccount(
    override val id: Long,
    override val name: String
) : Account
```

接口属性也可以有默认 getter：

```kotlin
interface Account {
    val id: Long
    val name: String

    val displayName: String
        get() = "$id-$name"
}

data class UserAccount(
    override val id: Long,
    override val name: String
) : Account

fun main() {
    val account = UserAccount(1001, "Tom")
    println(account.displayName)
}
```

输出：

```text
1001-Tom
```

但接口属性不能这样写：

```kotlin
interface Account {
    val type = "user"
}
```

接口不能保存实例字段，也没有 backing field。属性只能是抽象属性，或者通过 getter 计算。

合法写法：

```kotlin
interface Account {
    val name: String

    val normalizedName: String
        get() = name.trim().lowercase()
}
```

需要保存状态时，状态必须放在实现类里：

```kotlin
class DefaultAccount(
    override val name: String
) : Account
```

### var 属性也可以声明，但仍然不保存字段

接口可以声明 `var`。

```kotlin
interface Switch {
    var enabled: Boolean

    fun turnOn() {
        enabled = true
    }

    fun turnOff() {
        enabled = false
    }
}

class LightSwitch : Switch {
    override var enabled: Boolean = false
}

fun main() {
    val switch = LightSwitch()

    println(switch.enabled)
    switch.turnOn()
    println(switch.enabled)
}
```

输出：

```text
false
true
```

`enabled` 的字段在 `LightSwitch` 里，不在接口里。

接口默认方法可以读写 `enabled`，但前提是实现类提供了这个属性。

### 接口继承接口：先拆能力，再组合能力

接口可以继承其他接口。

```kotlin
interface Readable {
    fun read(id: Long): String?
}

interface Writable {
    fun write(id: Long, value: String)
}

interface Repository : Readable, Writable
```

实现类需要补齐所有抽象成员：

```kotlin
class MemoryRepository : Repository {
    private val data = mutableMapOf<Long, String>()

    override fun read(id: Long): String? {
        return data[id]
    }

    override fun write(id: Long, value: String) {
        data[id] = value
    }
}

fun main() {
    val repository: Repository = MemoryRepository()

    repository.write(1, "Kotlin")
    println(repository.read(1))
}
```

输出：

```text
Kotlin
```

这种设计比一个大接口更灵活。

只读场景依赖 `Readable`：

```kotlin
class QueryService(
    private val readable: Readable
) {
    fun query(id: Long): String {
        return readable.read(id) ?: "空"
    }
}
```

写入场景依赖 `Writable`：

```kotlin
class SaveService(
    private val writable: Writable
) {
    fun save(id: Long, value: String) {
        writable.write(id, value)
    }
}
```

接口拆小之后，调用方只拿到真正需要的能力。

### 一个类实现多个接口

Kotlin 类只能继承一个父类，但可以实现多个接口。

```kotlin
interface Flyable {
    fun fly()
}

interface Swimmable {
    fun swim()
}

class Duck : Flyable, Swimmable {
    override fun fly() {
        println("低空飞行")
    }

    override fun swim() {
        println("水中游动")
    }
}

fun main() {
    val duck = Duck()

    duck.fly()
    duck.swim()
}
```

输出：

```text
低空飞行
水中游动
```

接口更像“能力标签 + 行为契约”。一个对象可以同时具备多个能力。

### 默认实现冲突：必须手动说清楚

多个接口里有相同签名的默认方法时，实现类必须覆盖。

```kotlin
interface Clickable {
    fun show() {
        println("Clickable show")
    }
}

interface Focusable {
    fun show() {
        println("Focusable show")
    }
}

class Button : Clickable, Focusable {
    override fun show() {
        super<Clickable>.show()
        super<Focusable>.show()
        println("Button show")
    }
}

fun main() {
    Button().show()
}
```

输出：

```text
Clickable show
Focusable show
Button show
```

`super<Clickable>.show()` 表示调用 `Clickable` 的默认实现。

`super<Focusable>.show()` 表示调用 `Focusable` 的默认实现。

如果不覆盖，编译器不知道该选哪一个，所以会直接报错。

属性默认 getter 也会冲突：

```kotlin
interface A {
    val title: String
        get() = "A"
}

interface B {
    val title: String
        get() = "B"
}

class C : A, B {
    override val title: String
        get() = super<A>.title + super<B>.title
}

fun main() {
    println(C().title)
}
```

输出：

```text
AB
```

冲突处理的原则很简单：多个来源都能提供实现时，最终类必须明确选择。

### 接口中的访问修饰符

接口成员默认是 `public`。

常规接口成员不适合写 `protected`。接口表达的是公开契约，受保护状态和继承层级细节更适合放到抽象类。

接口内部可以放私有辅助方法，用来服务默认实现：

```kotlin
interface TokenParser {
    fun parse(raw: String): Map<String, String> {
        return raw
            .split(";")
            .mapNotNull { parsePair(it) }
            .toMap()
    }

    private fun parsePair(text: String): Pair<String, String>? {
        val parts = text.split("=")
        if (parts.size != 2) {
            return null
        }

        return parts[0].trim() to parts[1].trim()
    }
}

class DefaultTokenParser : TokenParser

fun main() {
    val parser = DefaultTokenParser()
    println(parser.parse("id=100; name=Tom; broken"))
}
```

输出：

```text
{id=100, name=Tom}
```

私有方法不能成为接口契约，只是接口内部默认实现的工具。

### 匿名对象：临时实现一个接口

有些接口实现只在一个地方使用，单独建类反而啰嗦。

```kotlin
interface Task {
    fun run()
}

fun execute(task: Task) {
    println("任务开始")
    task.run()
    println("任务结束")
}

fun main() {
    execute(object : Task {
        override fun run() {
            println("清理临时文件")
        }
    })
}
```

输出：

```text
任务开始
清理临时文件
任务结束
```

`object : Task { ... }` 创建了一个匿名对象。

匿名对象适合一次性实现。复用次数多、逻辑复杂、需要单元测试时，应该抽成具名类。

### fun interface：一个抽象方法时可以用 Lambda

只有一个抽象方法的接口，可以声明为函数式接口。

```kotlin
fun interface Validator<T> {
    fun validate(value: T): Boolean
}
```

使用 Lambda 创建实例：

```kotlin
fun main() {
    val notBlank = Validator<String> { value ->
        value.isNotBlank()
    }

    println(notBlank.validate("Kotlin"))
    println(notBlank.validate("   "))
}
```

输出：

```text
true
false
```

函数式接口里可以有默认方法，但只能有一个抽象方法。

```kotlin
fun interface RetryPolicy {
    fun shouldRetry(error: Throwable): Boolean

    fun describe(): String {
        return "重试策略"
    }
}
```

使用：

```kotlin
fun request(policy: RetryPolicy) {
    val error = IllegalStateException("timeout")

    if (policy.shouldRetry(error)) {
        println("准备重试")
    } else {
        println("不再重试")
    }
}

fun main() {
    request { error ->
        error.message == "timeout"
    }
}
```

输出：

```text
准备重试
```

如果只是接收一个函数，普通函数类型通常更轻：

```kotlin
fun filterUsers(
    names: List<String>,
    predicate: (String) -> Boolean
): List<String> {
    return names.filter(predicate)
}
```

如果这个回调有明确业务含义，或者需要额外默认方法，`fun interface` 更合适：

```kotlin
fun interface UserNameRule {
    fun allow(name: String): Boolean

    fun errorMessage(): String {
        return "用户名不符合规则"
    }
}
```

### SAM 和普通函数类型的差别

下面两个类型看起来很像：

```kotlin
fun interface IntRule {
    fun accept(value: Int): Boolean
}

typealias IntPredicate = (Int) -> Boolean
```

使用时都能接收 Lambda：

```kotlin
val rule = IntRule { it > 0 }
val predicate: IntPredicate = { it > 0 }
```

但它们不是同一种东西。

`IntRule` 是一个接口类型，可以拥有名字、默认方法、继承其他接口，也可以被 Java 调用方当成接口实现。

`IntPredicate` 只是函数类型别名，本质仍然是 `(Int) -> Boolean`。

API 只是需要执行一个函数时，函数类型足够。

API 想表达领域契约时，函数式接口更清楚。

### 接口委托 by：少写一堆转发方法

接口常和委托一起使用。

先定义一个缓存接口：

```kotlin
interface Cache {
    fun get(key: String): String?
    fun put(key: String, value: String)
    fun remove(key: String)
}
```

内存实现：

```kotlin
class MemoryCache : Cache {
    private val data = mutableMapOf<String, String>()

    override fun get(key: String): String? {
        return data[key]
    }

    override fun put(key: String, value: String) {
        data[key] = value
    }

    override fun remove(key: String) {
        data.remove(key)
    }
}
```

如果要加日志，传统代理写法要手动转发所有方法：

```kotlin
class LoggingCache(
    private val cache: Cache
) : Cache {
    override fun get(key: String): String? {
        println("读取缓存：$key")
        return cache.get(key)
    }

    override fun put(key: String, value: String) {
        println("写入缓存：$key")
        cache.put(key, value)
    }

    override fun remove(key: String) {
        println("删除缓存：$key")
        cache.remove(key)
    }
}
```

使用 `by` 可以少写很多样板：

```kotlin
class LoggingCache(
    private val cache: Cache
) : Cache by cache {
    override fun get(key: String): String? {
        println("读取缓存：$key")
        return cache.get(key)
    }

    override fun put(key: String, value: String) {
        println("写入缓存：$key")
        cache.put(key, value)
    }
}
```

没有覆盖的 `remove` 会自动委托给 `cache`。

完整使用：

```kotlin
fun main() {
    val cache: Cache = LoggingCache(MemoryCache())

    cache.put("token", "abc")
    println(cache.get("token"))
    cache.remove("token")
    println(cache.get("token"))
}
```

输出：

```text
写入缓存：token
读取缓存：token
abc
读取缓存：token
null
```

`remove` 没有日志，因为它直接走了被委托对象。

委托适合代理、装饰器、组合复用。接口越稳定，委托越省事。

### 多接口委托：组合多个小能力

一个类可以把不同接口委托给不同对象。

```kotlin
interface Printer {
    fun print(text: String)
}

interface Scanner {
    fun scan(): String
}

class ConsolePrinter : Printer {
    override fun print(text: String) {
        println("打印：$text")
    }
}

class FileScanner : Scanner {
    override fun scan(): String {
        return "扫描内容"
    }
}

class OfficeMachine(
    printer: Printer,
    scanner: Scanner
) : Printer by printer, Scanner by scanner

fun main() {
    val machine = OfficeMachine(ConsolePrinter(), FileScanner())

    machine.print("合同")
    println(machine.scan())
}
```

输出：

```text
打印：合同
扫描内容
```

这种方式比继承层级更轻。对象能力来自组合，而不是来自又深又长的父类链。

### 实战 Demo：支付策略 + 风控 + 日志

下面用接口拆一个支付流程。

金额类型：

```kotlin
data class Money(
    val cents: Long,
    val currency: String = "CNY"
) {
    fun text(): String {
        return "$currency ${cents / 100}.${(cents % 100).toString().padStart(2, '0')}"
    }
}
```

支付接口：

```kotlin
interface PaymentMethod {
    val name: String

    fun pay(orderId: Long, amount: Money): PaymentResult

    fun support(amount: Money): Boolean {
        return amount.cents > 0
    }
}
```

结果模型：

```kotlin
data class PaymentResult(
    val success: Boolean,
    val message: String
)
```

两个实现：

```kotlin
class BalancePay(
    private var balance: Long
) : PaymentMethod {
    override val name: String = "余额支付"

    override fun pay(orderId: Long, amount: Money): PaymentResult {
        if (balance < amount.cents) {
            return PaymentResult(false, "余额不足")
        }

        balance -= amount.cents
        return PaymentResult(true, "订单 $orderId 使用 $name 成功")
    }
}

class CardPay : PaymentMethod {
    override val name: String = "银行卡支付"

    override fun pay(orderId: Long, amount: Money): PaymentResult {
        return PaymentResult(true, "订单 $orderId 使用 $name 成功")
    }
}
```

风控接口：

```kotlin
interface RiskControl {
    fun check(orderId: Long, amount: Money): Boolean
}

class SimpleRiskControl : RiskControl {
    override fun check(orderId: Long, amount: Money): Boolean {
        return amount.cents <= 100_000
    }
}
```

支付服务只依赖接口：

```kotlin
class PaymentService(
    private val riskControl: RiskControl
) {
    fun pay(
        orderId: Long,
        amount: Money,
        method: PaymentMethod
    ): PaymentResult {
        if (!method.support(amount)) {
            return PaymentResult(false, "支付方式不支持该金额")
        }

        if (!riskControl.check(orderId, amount)) {
            return PaymentResult(false, "风控拦截")
        }

        return method.pay(orderId, amount)
    }
}
```

运行：

```kotlin
fun main() {
    val service = PaymentService(SimpleRiskControl())

    val small = service.pay(
        orderId = 1001,
        amount = Money(9_900),
        method = BalancePay(balance = 20_000)
    )

    val large = service.pay(
        orderId = 1002,
        amount = Money(200_000),
        method = CardPay()
    )

    println(small)
    println(large)
}
```

输出：

```text
PaymentResult(success=true, message=订单 1001 使用 余额支付 成功)
PaymentResult(success=false, message=风控拦截)
```

这里的变化点很清楚：

```text
支付方式会变，所以抽成 PaymentMethod
风控规则会变，所以抽成 RiskControl
支付编排流程相对稳定，所以放在 PaymentService
```

接口不是越多越好。只有变化点明确时，接口才有价值。

### 实战 Demo：Repository 接口不要只会 CRUD

很多项目会直接写：

```kotlin
interface UserRepository {
    fun save(user: User)
    fun delete(id: Long)
    fun update(user: User)
    fun findById(id: Long): User?
    fun findAll(): List<User>
}
```

这不一定错，但容易变成数据库操作清单。

更贴近业务的接口可以这样写：

```kotlin
data class User(
    val id: Long,
    val name: String,
    val enabled: Boolean
)

interface UserReader {
    fun findActiveUser(id: Long): User?
}

interface UserWriter {
    fun create(user: User)
    fun disable(id: Long)
}
```

内存实现：

```kotlin
class MemoryUserRepository : UserReader, UserWriter {
    private val users = mutableMapOf<Long, User>()

    override fun findActiveUser(id: Long): User? {
        return users[id]?.takeIf { it.enabled }
    }

    override fun create(user: User) {
        users[user.id] = user
    }

    override fun disable(id: Long) {
        val user = users[id] ?: return
        users[id] = user.copy(enabled = false)
    }
}
```

查询服务只依赖读接口：

```kotlin
class UserQueryService(
    private val userReader: UserReader
) {
    fun profile(id: Long): String {
        val user = userReader.findActiveUser(id) ?: return "用户不存在"
        return "${user.id}:${user.name}"
    }
}
```

运行：

```kotlin
fun main() {
    val repository = MemoryUserRepository()
    repository.create(User(1, "Tom", enabled = true))

    val service = UserQueryService(repository)

    println(service.profile(1))
    repository.disable(1)
    println(service.profile(1))
}
```

输出：

```text
1:Tom
用户不存在
```

接口按读写能力拆开后，调用方不会拿到不该使用的写权限。

### 实战 Demo：用 sealed interface 表达有限状态

普通接口表示开放扩展，任何地方都可能新增实现。

有些场景只允许有限几种状态，例如页面加载状态。

```kotlin
sealed interface UiState<out T>

data object Loading : UiState<Nothing>

data class Success<T>(
    val data: T
) : UiState<T>

data class Failure(
    val message: String
) : UiState<Nothing>
```

处理状态：

```kotlin
fun render(state: UiState<List<String>>): String {
    return when (state) {
        Loading -> "加载中"
        is Success -> "数据：${state.data.joinToString()}"
        is Failure -> "错误：${state.message}"
    }
}

fun main() {
    println(render(Loading))
    println(render(Success(listOf("Kotlin", "Java"))))
    println(render(Failure("网络异常")))
}
```

输出：

```text
加载中
数据：Kotlin, Java
错误：网络异常
```

`sealed interface` 的直接实现受限制，编译器知道有哪些分支。`when` 可以做穷尽检查，不容易漏掉状态。

普通接口适合开放插件式扩展。

密封接口适合有限状态、有限事件、有限错误类型。

### 接口和抽象类怎么选

接口和抽象类都能写抽象函数，也都能提供默认实现。差别主要在状态和继承模型。

| 对比点 | interface | abstract class |
| --- | --- | --- |
| 构造函数 | 没有构造函数 | 可以有构造函数 |
| 实例字段 | 不能保存字段 | 可以保存字段 |
| 实现数量 | 一个类可实现多个接口 | 一个类只能继承一个父类 |
| 主要表达 | 能力、契约、角色 | 共同基类、共享状态、模板流程 |
| 适合场景 | 支付方式、通知渠道、缓存能力、回调规则 | 带公共字段的基类、固定生命周期、模板方法 |

接口适合：

```text
只关心行为契约
实现之间没有共同状态
一个类需要组合多个能力
调用方只需要依赖最小能力
```

抽象类适合：

```text
需要构造参数
需要共享字段
需要受保护方法
需要固定一套继承模板
```

例子：

```kotlin
abstract class BaseController(
    protected val logger: Logger
) {
    fun handle(name: String) {
        logger.info("开始处理：$name")
        doHandle(name)
        logger.info("处理结束：$name")
    }

    protected abstract fun doHandle(name: String)
}
```

这里需要构造参数和 `protected` 成员，抽象类更合适。

### 接口不是 DTO，也不是标记贴纸

空接口有时会被当成标记：

```kotlin
interface DomainEvent
```

如果只是为了给类型贴标签，要确认后面是否真的会基于这个标签做约束。

更清楚的写法可能是密封接口：

```kotlin
sealed interface DomainEvent

data class UserRegistered(
    val userId: Long
) : DomainEvent

data class OrderPaid(
    val orderId: Long
) : DomainEvent
```

事件处理：

```kotlin
fun handle(event: DomainEvent): String {
    return when (event) {
        is UserRegistered -> "用户注册：${event.userId}"
        is OrderPaid -> "订单支付：${event.orderId}"
    }
}
```

空接口不是完全不能用，但要有明确目的。只是为了“看起来有架构”，通常没有意义。

### 接口不要一上来就抽

代码里只有一个实现，而且短期没有第二个实现时，接口不一定有价值。

下面这种结构很常见：

```kotlin
interface UserService {
    fun register(name: String)
}

class UserServiceImpl : UserService {
    override fun register(name: String) {
        println("注册：$name")
    }
}
```

如果项目里永远只通过 `UserServiceImpl` 工作，接口只是多了一层文件。

接口值得出现的信号通常是：

```text
存在多个真实实现
测试需要替换实现
模块边界需要隔离
第三方依赖需要包一层
领域变化点已经明确
框架要求依赖接口
```

没有这些信号时，直接写类往往更简单。

### 常见坑一：接口太大

大接口会强迫实现类实现不需要的能力。

```kotlin
interface Worker {
    fun code()
    fun design()
    fun test()
    fun deploy()
}
```

如果有的角色只负责测试，也必须实现 `code`、`design`、`deploy`，这就是接口设计过大。

拆成小接口：

```kotlin
interface Coder {
    fun code()
}

interface Designer {
    fun design()
}

interface Tester {
    fun test()
}

interface Deployer {
    fun deploy()
}
```

组合使用：

```kotlin
class FullStackEngineer : Coder, Designer, Tester, Deployer {
    override fun code() = println("编码")
    override fun design() = println("设计")
    override fun test() = println("测试")
    override fun deploy() = println("部署")
}
```

接口越小，调用方依赖越准。

### 常见坑二：默认实现塞太多业务

默认实现应该表达通用逻辑。

下面这种接口不太合适：

```kotlin
interface OrderProcessor {
    fun process(orderId: Long) {
        // 查询订单
        // 扣库存
        // 发起支付
        // 写日志
        // 推送消息
    }
}
```

接口默认方法里藏一整套业务流程，实现类很难知道哪些步骤可覆盖，哪些步骤不能碰。

复杂编排更适合放到服务类或抽象类。接口保留关键变化点。

### 常见坑三：接口命名过虚

这些名字很容易变成垃圾桶：

```kotlin
interface Manager
interface Handler
interface Processor
interface Helper
```

不是完全不能用，但单独看不出契约。

更好的名字要表达能力：

```kotlin
interface TokenVerifier
interface OrderPriceCalculator
interface InvoiceSender
interface UserReader
interface CacheStore
```

接口名最好能让函数列表变得可预测。

### 常见坑四：为了测试强行给所有类加接口

测试替换实现是接口的常见用途，但不代表所有类都要配一个接口。

纯计算类通常可以直接测试：

```kotlin
class PriceCalculator {
    fun discount(price: Int, rate: Int): Int {
        return price * (100 - rate) / 100
    }
}
```

更需要接口的是外部依赖：

```kotlin
interface Clock {
    fun nowMillis(): Long
}

class SystemClock : Clock {
    override fun nowMillis(): Long {
        return System.currentTimeMillis()
    }
}
```

测试时替换时间：

```kotlin
class FixedClock(
    private val fixed: Long
) : Clock {
    override fun nowMillis(): Long {
        return fixed
    }
}
```

外部系统、时间、随机数、网络、数据库、消息队列，这些地方更值得抽接口。

### Java 互操作和 JVM default 方法

Kotlin 接口编译到 JVM 后，默认方法会按编译器配置生成。

常见配置在 Gradle Kotlin DSL 里：

```kotlin
kotlin {
    compilerOptions {
        jvmDefault = JvmDefaultMode.NO_COMPATIBILITY
    }
}
```

常见模式：

```text
enable：默认模式，生成接口默认方法，同时保留兼容桥接
no-compatibility：只生成接口默认方法，适合新项目
disable：不生成接口默认方法，只走兼容实现
```

新项目通常更关注生成结果简洁；旧项目或库项目更关注二进制兼容。

和 Java 混写时，还要注意 Kotlin `internal` 在字节码层面可能变成 public 加名称改写。公共库接口不要随便暴露内部语义。

### 一个完整 Demo：通知中心的接口设计

下面把普通接口、默认实现、函数式接口、接口委托串起来。

消息模型：

```kotlin
data class Notice(
    val receiver: String,
    val title: String,
    val content: String
)

data class SendResult(
    val success: Boolean,
    val detail: String
)
```

通道接口：

```kotlin
interface NoticeChannel {
    val channelName: String

    fun send(notice: Notice): SendResult

    fun format(notice: Notice): String {
        return "[$channelName] ${notice.title} - ${notice.content}"
    }
}
```

两个通道：

```kotlin
class EmailChannel : NoticeChannel {
    override val channelName: String = "Email"

    override fun send(notice: Notice): SendResult {
        return SendResult(
            success = true,
            detail = "邮件发送给 ${notice.receiver}：${format(notice)}"
        )
    }
}

class SmsChannel : NoticeChannel {
    override val channelName: String = "Sms"

    override fun send(notice: Notice): SendResult {
        return SendResult(
            success = true,
            detail = "短信发送给 ${notice.receiver}：${notice.content}"
        )
    }
}
```

发送前过滤规则：

```kotlin
fun interface NoticeRule {
    fun allow(notice: Notice): Boolean
}
```

日志装饰器：

```kotlin
class LoggingNoticeChannel(
    private val target: NoticeChannel
) : NoticeChannel by target {
    override fun send(notice: Notice): SendResult {
        println("开始发送：${target.channelName}")
        val result = target.send(notice)
        println("发送结束：${result.success}")
        return result
    }
}
```

通知中心：

```kotlin
class NoticeCenter(
    private val channel: NoticeChannel,
    private val rule: NoticeRule
) {
    fun publish(notice: Notice): SendResult {
        if (!rule.allow(notice)) {
            return SendResult(false, "规则拦截")
        }

        return channel.send(notice)
    }
}
```

运行：

```kotlin
fun main() {
    val center = NoticeCenter(
        channel = LoggingNoticeChannel(EmailChannel()),
        rule = NoticeRule { notice ->
            notice.receiver.isNotBlank() && notice.content.length <= 50
        }
    )

    val result = center.publish(
        Notice(
            receiver = "tom@example.com",
            title = "订单通知",
            content = "订单已经支付成功"
        )
    )

    println(result)
}
```

输出：

```text
开始发送：Email
发送结束：true
SendResult(success=true, detail=邮件发送给 tom@example.com：[Email] 订单通知 - 订单已经支付成功)
```

这个例子里：

```text
NoticeChannel 表示可发送通知的通道
EmailChannel 和 SmsChannel 是不同实现
NoticeRule 是单方法规则，适合 fun interface
LoggingNoticeChannel 使用 by 委托增强行为
NoticeCenter 只依赖接口，不绑定具体通道
```

接口把变化点切开之后，新增飞书、钉钉、站内信，只需要加新的 `NoticeChannel` 实现。

### 写 interface 的几条实用规则

接口写得舒服，通常符合这些规则：

* 名字表达能力，不表达空泛身份
* 方法数量少，调用方只依赖必要能力
* 默认实现只放稳定通用逻辑
* 属性只表达契约，不假装保存字段
* 有多个真实实现、边界隔离、测试替换时再抽接口
* 单抽象方法且有业务语义时使用 `fun interface`
* 有限状态和事件优先考虑 `sealed interface`
* 多个默认实现冲突时显式覆盖并说明选择
* 需要共享状态、构造参数、受保护成员时使用抽象类
* 代理增强优先考虑接口委托 `by`

### 总结

Kotlin `interface` 不是“空壳文件”，也不是“万能抽象层”。

它适合表达清楚的能力契约：

```text
PaymentMethod 能支付
RiskControl 能风控
UserReader 能读取用户
NoticeChannel 能发送通知
Cache 能读写缓存
```

Kotlin 接口的几个核心点：

```text
没有函数体的成员必须由实现类 override
有函数体的成员是默认实现
接口可以声明属性，但不能保存字段
一个类可以实现多个接口
多接口默认实现冲突必须显式覆盖
fun interface 适合单方法业务回调
sealed interface 适合有限状态建模
by 委托适合代理和组合复用
```

接口最好的状态，是调用方只看见稳定契约，实现类负责变化细节。抽象刚好挡住变化，代码才会变清楚；抽象挡住了阅读，接口就成了噪音。
