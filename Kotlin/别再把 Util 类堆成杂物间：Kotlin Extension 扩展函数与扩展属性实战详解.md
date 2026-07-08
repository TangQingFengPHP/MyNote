### 简介

项目里经常会出现一堆工具类：

```kotlin
StringUtils.isEmail(value)
DateUtils.format(time)
UserUtils.displayName(user)
ListUtils.secondOrNull(items)
```

代码能跑，但读起来总像在翻工具箱。

Kotlin Extension 的目标不是“给类偷偷塞一个新方法”，而是把这类辅助逻辑放回更自然的位置：

```kotlin
value.isEmail()
time.toDisplayText()
user.displayName
items.secondOrNull()
```

调用方式像成员函数，实际却没有改动原来的类。

这点很关键。

Extension 看起来像“扩展了类”，本质上仍然是普通函数，只是第一个参数换成了点号前面的对象。理解这一点，很多坑都会变得清楚：为什么扩展不能覆盖成员函数，为什么扩展没有运行时多态，为什么扩展属性不能保存字段。

### 第一个扩展函数：让字符串自己判断邮箱

先看一个很小的需求：判断字符串是不是邮箱。

普通工具函数写法：

```kotlin
fun isEmail(value: String): Boolean {
    return value.contains("@") && value.contains(".")
}

fun main() {
    println(isEmail("tom@example.com"))
}
```

换成扩展函数：

```kotlin
fun String.isEmail(): Boolean {
    return contains("@") && contains(".")
}

fun main() {
    println("tom@example.com".isEmail())
    println("not-email".isEmail())
}
```

输出：

```text
true
false
```

`fun String.isEmail()` 里的 `String` 叫接收者类型，点号前面的 `"tom@example.com"` 叫接收者对象。

扩展函数内部可以直接访问接收者的公开成员：

```kotlin
fun String.isEmail(): Boolean {
    return this.contains("@") && this.contains(".")
}
```

`this` 指向点号前面的字符串。多数时候 `this` 可以省略，所以 `contains("@")` 等价于 `this.contains("@")`。

### Extension 本质：更顺手的静态函数

下面这个扩展：

```kotlin
fun String.maskPhone(): String {
    if (length != 11) {
        return this
    }

    return take(3) + "****" + takeLast(4)
}

fun main() {
    println("13800138000".maskPhone())
}
```

输出：

```text
138****8000
```

从调用体验看，`maskPhone()` 像是 `String` 自带的方法。

但它并没有真的进入 `String` 类。可以把它近似理解成：

```kotlin
fun maskPhone(receiver: String): String {
    if (receiver.length != 11) {
        return receiver
    }

    return receiver.take(3) + "****" + receiver.takeLast(4)
}
```

编译器只是允许下面这种写法：

```kotlin
"13800138000".maskPhone()
```

这种设计带来两个结果：

* 扩展不能访问接收者的 `private` 成员
* 扩展不是虚函数，没有运行时多态

### Demo：用扩展函数整理订单展示逻辑

业务代码里经常有一堆格式化逻辑。

先定义订单模型：

```kotlin
import java.math.BigDecimal
import java.math.RoundingMode
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter

data class Order(
    val id: Long,
    val buyerName: String?,
    val amount: BigDecimal,
    val createdAt: LocalDateTime,
    val status: OrderStatus
)

enum class OrderStatus {
    Pending,
    Paid,
    Canceled
}
```

没有扩展时，页面或接口层可能会散落这些判断：

```kotlin
fun buildOrderLine(order: Order): String {
    val buyer = if (order.buyerName.isNullOrBlank()) {
        "匿名客户"
    } else {
        order.buyerName
    }

    val amount = "¥" + order.amount.setScale(2, RoundingMode.HALF_UP)
    val time = order.createdAt.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm"))
    val status = when (order.status) {
        OrderStatus.Pending -> "待支付"
        OrderStatus.Paid -> "已支付"
        OrderStatus.Canceled -> "已取消"
    }

    return "#${order.id} $buyer $amount $time $status"
}
```

逻辑不复杂，但所有细节挤在一个函数里。

可以把稳定的展示规则拆成扩展：

```kotlin
fun String?.orGuestName(): String {
    return if (isNullOrBlank()) {
        "匿名客户"
    } else {
        this
    }
}

fun BigDecimal.toMoneyText(): String {
    return "¥" + setScale(2, RoundingMode.HALF_UP).toPlainString()
}

fun LocalDateTime.toMinuteText(): String {
    val formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm")
    return format(formatter)
}

fun OrderStatus.toChineseText(): String {
    return when (this) {
        OrderStatus.Pending -> "待支付"
        OrderStatus.Paid -> "已支付"
        OrderStatus.Canceled -> "已取消"
    }
}

fun Order.toListLine(): String {
    return "#$id ${buyerName.orGuestName()} ${amount.toMoneyText()} ${createdAt.toMinuteText()} ${status.toChineseText()}"
}
```

使用：

```kotlin
fun main() {
    val order = Order(
        id = 1001,
        buyerName = null,
        amount = BigDecimal("99.8"),
        createdAt = LocalDateTime.of(2026, 7, 8, 10, 30),
        status = OrderStatus.Paid
    )

    println(order.toListLine())
}
```

输出：

```text
#1001 匿名客户 ¥99.80 2026-07-08 10:30 已支付
```

扩展函数不是为了把所有逻辑都塞到模型旁边，而是适合封装“围绕某个类型反复出现的、稳定的小动作”。

金额怎么展示，时间怎么展示，状态怎么展示，这些都很适合写成扩展。

### 可空接收者：null 也能直接点出来

Kotlin 扩展可以声明在可空类型上。

```kotlin
fun String?.orDash(): String {
    return if (this.isNullOrBlank()) {
        "-"
    } else {
        this
    }
}

fun main() {
    val nickname: String? = null
    val city: String? = "Shanghai"

    println(nickname.orDash())
    println(city.orDash())
}
```

输出：

```text
-
Shanghai
```

这里 `nickname` 是 `null`，但 `nickname.orDash()` 仍然可以调用，因为扩展的接收者类型是 `String?`。

可空扩展经常用于兜底展示：

```kotlin
fun String?.toSearchKeyword(): String {
    return this
        ?.trim()
        ?.takeIf { it.isNotEmpty() }
        ?: "*"
}

fun main() {
    println(null.toSearchKeyword())
    println("  kotlin  ".toSearchKeyword())
}
```

输出：

```text
*
kotlin
```

标准库里的 `Any?.toString()` 也是类似思路：即使接收者是 `null`，也能得到字符串 `"null"`。

### 泛型扩展：集合工具不用到处复制

扩展可以带泛型。集合类上尤其常见。

```kotlin
fun <T> List<T>.secondOrNull(): T? {
    return if (size >= 2) this[1] else null
}

fun <T> List<T>.headAndTail(): Pair<T, List<T>>? {
    if (isEmpty()) {
        return null
    }

    return first() to drop(1)
}

fun main() {
    println(listOf("A", "B", "C").secondOrNull())
    println(listOf(10, 20, 30).headAndTail())
    println(emptyList<Int>().headAndTail())
}
```

输出：

```text
B
(10, [20, 30])
null
```

这类扩展适合放在明确的包里，例如：

```text
com.example.common.collection
```

需要使用的地方再导入：

```kotlin
import com.example.common.collection.secondOrNull
```

不要把所有扩展都扔到一个 `Extensions.kt` 文件里。文件一大，查找和命名冲突都会变麻烦。

更常见的拆法：

```text
StringExtensions.kt
CollectionExtensions.kt
MoneyExtensions.kt
TimeExtensions.kt
ViewExtensions.kt
```

### 扩展属性：看起来像字段，其实只能计算

扩展不只能写函数，也能写属性。

```kotlin
data class User(
    val firstName: String,
    val lastName: String
)

val User.fullName: String
    get() = "$firstName $lastName"

fun main() {
    val user = User("Tom", "Green")
    println(user.fullName)
}
```

输出：

```text
Tom Green
```

扩展属性不能直接写成这样：

```kotlin
val User.fullName: String = "$firstName $lastName"
```

原因是扩展没有幕后字段，也就是没有地方真正保存这个值。

只能通过 `get()` 计算：

```kotlin
val String.lastChar: Char?
    get() = if (isEmpty()) null else this[length - 1]
```

`var` 扩展属性也可以存在，但必须能把赋值动作转成已有对象的操作。

```kotlin
var StringBuilder.lastChar: Char
    get() = this[length - 1]
    set(value) {
        setCharAt(length - 1, value)
    }

fun main() {
    val builder = StringBuilder("abc")

    println(builder.lastChar)
    builder.lastChar = 'z'
    println(builder)
}
```

输出：

```text
c
abz
```

这里没有给 `StringBuilder` 新增字段，只是把 `lastChar = 'z'` 转成了 `setCharAt(...)`。

不建议用外部 `Map` 强行给对象挂状态：

```kotlin
private val tags = mutableMapOf<Any, String>()

var Any.tag: String?
    get() = tags[this]
    set(value) {
        if (value == null) {
            tags.remove(this)
        } else {
            tags[this] = value
        }
    }
```

这种写法会带来生命周期、内存泄漏、线程安全和对象相等性问题。除非非常清楚边界，否则扩展属性更适合做“计算属性”，不适合做“隐藏字段”。

### 成员函数优先：扩展盖不住原方法

类里已经有同名成员函数时，成员函数优先。

```kotlin
class Message {
    fun text(): String {
        return "member"
    }
}

fun Message.text(): String {
    return "extension"
}

fun main() {
    println(Message().text())
}
```

输出：

```text
member
```

扩展函数没有覆盖成员函数的能力。

所以不要通过扩展去“改造”已有 API 的语义。类本身已经提供的方法，优先相信成员函数；扩展更适合补充缺失能力。

成员函数优先还有一个现实影响：库升级后，原类新增了一个同名成员函数，原来调用扩展的位置可能变成调用成员函数。

例如旧版本库里没有 `summary()`，项目里写了扩展：

```kotlin
class Report(
    val title: String,
    val total: Int
)

fun Report.summary(): String {
    return "$title: $total"
}
```

如果后续 `Report` 类自身新增了 `summary()`，调用点会优先选择成员函数。这种变化可能影响行为。

公共扩展命名最好具体一点，少用 `run`、`parse`、`format`、`convert` 这种过宽名字。

### 静态解析：声明类型决定调用哪个扩展

扩展没有运行时多态。

```kotlin
open class Shape
class Rectangle : Shape()

fun Shape.name(): String {
    return "Shape"
}

fun Rectangle.name(): String {
    return "Rectangle"
}

fun printName(shape: Shape) {
    println(shape.name())
}

fun main() {
    val rectangle = Rectangle()

    println(rectangle.name())
    printName(rectangle)
}
```

输出：

```text
Rectangle
Shape
```

`rectangle.name()` 里变量声明类型是 `Rectangle`，所以调用 `Rectangle.name()`。

`printName(shape: Shape)` 里参数声明类型是 `Shape`，所以调用 `Shape.name()`。运行时传进来的是 `Rectangle`，也不会改成 `Rectangle.name()`。

真正需要多态时，应该使用成员函数：

```kotlin
open class Animal {
    open fun sound(): String {
        return "unknown"
    }
}

class Dog : Animal() {
    override fun sound(): String {
        return "wang"
    }
}

fun main() {
    val animal: Animal = Dog()
    println(animal.sound())
}
```

输出：

```text
wang
```

结论很直接：扩展适合“补工具”，不适合“做多态”。

### 扩展不能访问 private 和 protected

扩展函数不是类内部成员，所以不能访问私有属性。

```kotlin
class Account(
    private val balance: Int
)

fun Account.canPay(amount: Int): Boolean {
    // return balance >= amount
    return amount > 0
}
```

上面注释里的 `balance` 访问会编译失败。

需要暴露判断能力时，可以在类内部写成员函数：

```kotlin
class Wallet(
    private val balance: Int
) {
    fun canPay(amount: Int): Boolean {
        return balance >= amount
    }
}
```

扩展不是破坏封装的后门。它只能使用接收者公开出来的能力。

### 伴生对象扩展：给类名补工厂方法

如果类有 `companion object`，可以给伴生对象写扩展。

```kotlin
data class User(
    val id: Long,
    val name: String
) {
    companion object
}

fun User.Companion.guest(): User {
    return User(id = 0, name = "Guest")
}

fun main() {
    val user = User.guest()
    println(user)
}
```

输出：

```text
User(id=0, name=Guest)
```

调用时像静态方法，但本质仍然是扩展。

伴生对象扩展适合补充外部工厂、测试数据构造、协议转换：

```kotlin
fun User.Companion.fromCsv(line: String): User {
    val parts = line.split(",")
    return User(
        id = parts[0].toLong(),
        name = parts[1]
    )
}

fun main() {
    println(User.fromCsv("100,Tom"))
}
```

输出：

```text
User(id=100, name=Tom)
```

### 成员扩展：把扩展限制在某个上下文里

扩展函数可以写在类内部。

```kotlin
class HtmlBuilder {
    private val lines = mutableListOf<String>()

    fun String.h1() {
        lines += "<h1>$this</h1>"
    }

    fun String.p() {
        lines += "<p>$this</p>"
    }

    fun build(block: HtmlBuilder.() -> Unit): String {
        block()
        return lines.joinToString("\n")
    }
}

fun main() {
    val html = HtmlBuilder().build {
        "Kotlin Extension".h1()
        "扩展让 DSL 写起来更自然".p()
    }

    println(html)
}
```

输出：

```text
<h1>Kotlin Extension</h1>
<p>扩展让 DSL 写起来更自然</p>
```

`String.h1()` 和 `String.p()` 只在 `HtmlBuilder` 的上下文里可见，外部不能直接调用。

这种写法常见于 DSL。它的好处是把扩展能力限制在特定范围，避免污染全局命名空间。

成员扩展里有两个接收者：

```kotlin
class Connection(
    val host: String,
    val port: Int
) {
    fun String.withPort(): String {
        return "$this:$port"
    }

    fun connect() {
        println(host.withPort())
    }
}

fun main() {
    Connection("kotlinlang.org", 443).connect()
}
```

输出：

```text
kotlinlang.org:443
```

这里有两个对象参与：

```text
String 是扩展接收者
Connection 是分发接收者
```

`this` 默认指向更近的接收者，也就是 `String`。需要访问外层对象时，可以使用标签：

```kotlin
class Connection(
    val host: String,
    val port: Int
) {
    fun String.fullAddress(): String {
        return "${this}@${this@Connection.port}"
    }
}
```

多接收者代码读起来容易绕，适合 DSL 和局部封装，不适合到处铺开。

### 扩展函数类型：标准库作用域函数背后的基础

Kotlin 里还有一种很重要的写法：

```kotlin
val block: StringBuilder.() -> Unit = {
    append("Hello")
    append(", ")
    append("Kotlin")
}
```

`StringBuilder.() -> Unit` 叫带接收者的函数类型。

使用：

```kotlin
fun buildText(block: StringBuilder.() -> Unit): String {
    val builder = StringBuilder()
    builder.block()
    return builder.toString()
}

fun main() {
    val text = buildText {
        append("Hello")
        append(", ")
        append("Kotlin")
    }

    println(text)
}
```

输出：

```text
Hello, Kotlin
```

这不是“扩展函数声明”，但和扩展接收者是同一套思路：代码块内部的 `this` 是 `StringBuilder`。

`apply` 的简化版可以这样理解：

```kotlin
inline fun <T> T.configure(block: T.() -> Unit): T {
    block()
    return this
}

data class ServerConfig(
    var host: String = "localhost",
    var port: Int = 8080,
    var enableSsl: Boolean = false
)

fun main() {
    val config = ServerConfig().configure {
        host = "api.example.com"
        port = 443
        enableSsl = true
    }

    println(config)
}
```

输出：

```text
ServerConfig(host=api.example.com, port=443, enableSsl=true)
```

这类写法是 Kotlin DSL、Gradle Kotlin DSL、HTML DSL、测试 DSL 的基础。

### Demo：用扩展写一个简单路由 DSL

下面用扩展函数类型做一个小型路由注册器。

```kotlin
class Router {
    private val routes = mutableMapOf<String, () -> String>()

    fun get(path: String, handler: () -> String) {
        routes["GET $path"] = handler
    }

    fun post(path: String, handler: () -> String) {
        routes["POST $path"] = handler
    }

    fun handle(method: String, path: String): String {
        return routes["$method $path"]?.invoke() ?: "404 Not Found"
    }
}

fun router(block: Router.() -> Unit): Router {
    val router = Router()
    router.block()
    return router
}

fun main() {
    val app = router {
        get("/health") {
            "OK"
        }

        post("/orders") {
            "order created"
        }
    }

    println(app.handle("GET", "/health"))
    println(app.handle("POST", "/orders"))
    println(app.handle("GET", "/missing"))
}
```

输出：

```text
OK
order created
404 Not Found
```

`router { ... }` 里面可以直接调用 `get` 和 `post`，原因是代码块的接收者是 `Router`。

这种写法比下面这种更接近声明式配置：

```kotlin
val router = Router()
router.get("/health") { "OK" }
router.post("/orders") { "order created" }
```

小型配置、构建器、测试数据生成，都很适合这种模式。

### 扩展和导入：不是定义了就全项目可见

顶层扩展受包和导入控制。

例如：

```kotlin
package com.example.text

fun String.toSlug(): String {
    return lowercase()
        .trim()
        .replace(Regex("\\s+"), "-")
}
```

另一个文件里使用：

```kotlin
package com.example.article

import com.example.text.toSlug

fun main() {
    println("Kotlin Extension Guide".toSlug())
}
```

输出：

```text
kotlin-extension-guide
```

扩展的可见性和普通函数一样，可以写 `private`、`internal`、`public`。

文件内专用扩展适合写成 `private`：

```kotlin
private fun String.normalizeKeyword(): String {
    return trim().lowercase()
}
```

模块内通用扩展适合 `internal`：

```kotlin
internal fun String.toCacheKey(): String {
    return trim().lowercase().replace(" ", ":")
}
```

公共 API 扩展要慎重命名，因为它会进入调用方的补全列表。

### 扩展函数也能是 operator 和 infix

扩展函数可以配合 `operator`。

```kotlin
data class Point(
    val x: Int,
    val y: Int
)

operator fun Point.plus(other: Point): Point {
    return Point(
        x = x + other.x,
        y = y + other.y
    )
}

fun main() {
    val result = Point(1, 2) + Point(3, 4)
    println(result)
}
```

输出：

```text
Point(x=4, y=6)
```

也可以配合 `infix` 写更接近自然语言的 DSL。

```kotlin
data class Header(
    val name: String,
    val value: String
)

infix fun String.toHeader(value: String): Header {
    return Header(this, value)
}

fun main() {
    val header = "Content-Type" toHeader "application/json"
    println(header)
}
```

输出：

```text
Header(name=Content-Type, value=application/json)
```

`operator` 和 `infix` 都不该为了炫技使用。符号或中缀表达式必须符合直觉，否则普通函数名更清楚。

### 典型实战：Repository 查询结果转换

后端代码里，经常要把数据库实体转成接口 DTO。

```kotlin
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter

data class UserEntity(
    val id: Long,
    val username: String,
    val email: String?,
    val enabled: Boolean,
    val createdAt: LocalDateTime
)

data class UserResponse(
    val id: Long,
    val username: String,
    val email: String,
    val status: String,
    val createdAt: String
)
```

转换逻辑可以写成扩展：

```kotlin
fun Boolean.toEnabledText(): String {
    return if (this) "启用" else "禁用"
}

fun LocalDateTime.toApiTime(): String {
    return format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"))
}

fun String?.toEmailText(): String {
    return this?.takeIf { it.isNotBlank() } ?: "-"
}

fun UserEntity.toResponse(): UserResponse {
    return UserResponse(
        id = id,
        username = username,
        email = email.toEmailText(),
        status = enabled.toEnabledText(),
        createdAt = createdAt.toApiTime()
    )
}

fun main() {
    val entity = UserEntity(
        id = 1,
        username = "tom",
        email = null,
        enabled = true,
        createdAt = LocalDateTime.of(2026, 7, 8, 9, 15, 30)
    )

    println(entity.toResponse())
}
```

输出：

```text
UserResponse(id=1, username=tom, email=-, status=启用, createdAt=2026-07-08 09:15:30)
```

这类扩展适合放在离业务边界较近的位置，例如：

```text
adapter/http/UserResponseMapper.kt
```

不要把它放到全局 `common` 包。因为 `UserEntity.toResponse()` 属于当前接口返回格式，不一定是所有场景的通用转换。

### 典型实战：Android View 防重复点击

Android 项目里，View 扩展很常见。下面用简化版 `View` 模拟防重复点击逻辑。

```kotlin
class View {
    private var clickListener: (() -> Unit)? = null

    fun setOnClickListener(listener: () -> Unit) {
        clickListener = listener
    }

    fun performClick() {
        clickListener?.invoke()
    }
}

fun View.setDebouncedClickListener(
    intervalMillis: Long = 800,
    nowProvider: () -> Long = System::currentTimeMillis,
    onClick: () -> Unit
) {
    var lastClickAt = 0L

    setOnClickListener {
        val now = nowProvider()
        if (now - lastClickAt >= intervalMillis) {
            lastClickAt = now
            onClick()
        }
    }
}

fun main() {
    val button = View()
    var now = 1000L

    button.setDebouncedClickListener(
        intervalMillis = 800,
        nowProvider = { now }
    ) {
        println("submit")
    }

    button.performClick()
    now += 300
    button.performClick()
    now += 800
    button.performClick()
}
```

输出：

```text
submit
submit
```

第二次点击距离第一次只有 300ms，被拦掉了。

这种扩展比继承一个 `DebouncedButton` 更轻。它不改变 View 的类型，只是在调用层补上常用行为。

真实 Android 代码里可以直接扩展 `android.view.View`：

```kotlin
fun View.setDebouncedClickListener(
    intervalMillis: Long = 800,
    onClick: (View) -> Unit
) {
    var lastClickAt = 0L

    setOnClickListener { view ->
        val now = System.currentTimeMillis()
        if (now - lastClickAt >= intervalMillis) {
            lastClickAt = now
            onClick(view)
        }
    }
}
```

### 典型实战：Result 结果处理

假设接口调用返回统一结果：

```kotlin
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Failure(val code: String, val message: String) : ApiResult<Nothing>()
}
```

可以给结果类型补一些链式操作：

```kotlin
inline fun <T, R> ApiResult<T>.mapData(transform: (T) -> R): ApiResult<R> {
    return when (this) {
        is ApiResult.Success -> ApiResult.Success(transform(data))
        is ApiResult.Failure -> this
    }
}

inline fun <T> ApiResult<T>.onFailure(block: (String, String) -> Unit): ApiResult<T> {
    if (this is ApiResult.Failure) {
        block(code, message)
    }

    return this
}

fun main() {
    val result: ApiResult<Int> = ApiResult.Success(100)

    val textResult = result
        .mapData { "score=$it" }
        .onFailure { code, message ->
            println("$code $message")
        }

    println(textResult)
}
```

输出：

```text
Success(data=score=100)
```

这类扩展能把通用的结果处理规则集中起来。调用点保留业务主线，不必反复写 `when`。

### 扩展、继承、成员函数怎么选

三者解决的问题不同。

| 方案 | 适合场景 | 不适合场景 |
| --- | --- | --- |
| 成员函数 | 类型自己的核心行为、需要访问私有状态、需要多态 | 第三方类、JDK 类、只在某个模块使用的展示转换 |
| 继承 | 明确的 is-a 关系、需要覆盖父类行为 | 只想补几个工具方法、原类是 final、组合更合适的场景 |
| 扩展函数 | 不改原类补充能力、类型相关工具、局部 DSL、格式化和转换 | 需要真正覆盖行为、需要保存状态、需要运行时多态 |

一个简单判断：

```text
这个行为是不是类型的核心能力？
```

是核心能力，优先成员函数。

```text
这个行为是不是某个调用场景的辅助能力？
```

是辅助能力，可以考虑扩展。

```text
这个行为是不是多个子类型要有不同实现？
```

需要成员函数和多态，不适合扩展。

### 常见坑一：扩展太多，补全列表变成垃圾场

扩展写多了，IDE 补全里会出现大量函数。

例如给 `String` 写几十个扩展：

```kotlin
fun String.toUserId(): Long = trim().toLong()
fun String.toOrderId(): Long = trim().toLong()
fun String.toSkuId(): Long = trim().toLong()
fun String.toTenantId(): Long = trim().toLong()
```

这些函数看似方便，实际会污染所有字符串调用点。

更稳的方式是让领域类型承担语义：

```kotlin
@JvmInline
value class UserId(val value: Long)

fun String.toUserId(): UserId {
    return UserId(trim().toLong())
}
```

甚至直接把转换放在更靠近输入解析的位置，而不是让全项目所有 `String` 都拥有业务含义。

### 常见坑二：扩展名字过大

这种名字很危险：

```kotlin
fun String.parse(): Any {
    return this
}

fun String.convert(): String {
    return trim()
}

fun Any.format(): String {
    return toString()
}
```

问题不是代码不能运行，而是含义太宽。调用点很难看出具体规则。

更好的命名要带上业务语义：

```kotlin
fun String.parseOrderId(): Long {
    return trim().toLong()
}

fun String.toSearchKeyword(): String {
    return trim().lowercase()
}

fun BigDecimal.toCnyText(): String {
    return "¥" + setScale(2, RoundingMode.HALF_UP).toPlainString()
}
```

扩展函数名要回答一个问题：点号前面的对象，经过这个动作后变成什么。

### 常见坑三：把复杂业务流程藏进扩展

下面这种扩展不太合适：

```kotlin
fun Order.payAndSendMessage() {
    // 扣库存
    // 调支付
    // 写订单
    // 发短信
    // 推送 MQ
}
```

调用点看起来只是一个普通方法：

```kotlin
order.payAndSendMessage()
```

但实际背后可能有事务、网络、消息、重试和状态变更。扩展函数的语法太轻，容易隐藏重量。

复杂业务流程更适合放进服务对象：

```kotlin
class OrderPaymentService {
    fun pay(orderId: Long) {
        // 编排支付流程
    }
}
```

扩展可以做小而确定的转换，不适合伪装成领域服务。

### 常见坑四：扩展和成员同名导致误判

成员函数优先，所以这种代码容易误导：

```kotlin
class Text {
    fun trim(): String {
        return "member trim"
    }
}

fun Text.trim(): String {
    return "extension trim"
}
```

调用：

```kotlin
println(Text().trim())
```

输出一定是：

```text
member trim
```

扩展函数名尽量不要和接收者已有成员同名。即使当前版本没有同名成员，公共库升级后也可能出现。

### 常见坑五：把扩展当 Monkey Patch

有些语言可以在运行时给类补方法，甚至修改已有方法。Kotlin Extension 不是这类机制。

它没有这些能力：

```text
不能修改原类字节码
不能覆盖成员函数
不能访问 private 成员
不能给对象增加字段
不能根据运行时类型动态选择扩展
```

所以 Kotlin 扩展更像“带接收者的静态工具函数”，不是运行时补丁。

### 和 Java 互操作时长什么样

Kotlin 顶层扩展编译到 JVM 后，本质是静态方法。

假设文件名是 `TextExtensions.kt`：

```kotlin
package com.example.text

fun String.maskPhone(): String {
    if (length != 11) {
        return this
    }

    return take(3) + "****" + takeLast(4)
}
```

Java 调用时类似：

```java
String result = TextExtensionsKt.maskPhone("13800138000");
```

可以通过 `@file:JvmName` 改生成类名：

```kotlin
@file:JvmName("TextExt")

package com.example.text

fun String.maskPhone(): String {
    if (length != 11) {
        return this
    }

    return take(3) + "****" + takeLast(4)
}
```

Java 调用变成：

```java
String result = TextExt.maskPhone("13800138000");
```

Kotlin 调用仍然是：

```kotlin
"13800138000".maskPhone()
```

这也再次说明：扩展在 JVM 层面不是接收者类的成员方法。

### 一个完整小 Demo：订单列表接口转换

下面把前面的概念串起来：可空扩展、金额扩展、时间扩展、实体转 DTO、集合扩展。

```kotlin
import java.math.BigDecimal
import java.math.RoundingMode
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter

data class OrderEntity(
    val id: Long,
    val buyerName: String?,
    val amount: BigDecimal,
    val status: String,
    val createdAt: LocalDateTime
)

data class OrderItemResponse(
    val id: Long,
    val buyerName: String,
    val amount: String,
    val status: String,
    val createdAt: String
)

fun String?.orAnonymous(): String {
    return this?.takeIf { it.isNotBlank() } ?: "匿名客户"
}

fun BigDecimal.toYuanText(): String {
    return "¥" + setScale(2, RoundingMode.HALF_UP).toPlainString()
}

fun LocalDateTime.toSecondText(): String {
    return format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"))
}

fun String.toOrderStatusText(): String {
    return when (this) {
        "PENDING" -> "待支付"
        "PAID" -> "已支付"
        "CANCELED" -> "已取消"
        else -> "未知"
    }
}

fun OrderEntity.toItemResponse(): OrderItemResponse {
    return OrderItemResponse(
        id = id,
        buyerName = buyerName.orAnonymous(),
        amount = amount.toYuanText(),
        status = status.toOrderStatusText(),
        createdAt = createdAt.toSecondText()
    )
}

fun List<OrderEntity>.toItemResponses(): List<OrderItemResponse> {
    return map { it.toItemResponse() }
}

fun main() {
    val orders = listOf(
        OrderEntity(
            id = 1001,
            buyerName = null,
            amount = BigDecimal("19.9"),
            status = "PAID",
            createdAt = LocalDateTime.of(2026, 7, 8, 10, 0, 0)
        ),
        OrderEntity(
            id = 1002,
            buyerName = "Lucy",
            amount = BigDecimal("5"),
            status = "PENDING",
            createdAt = LocalDateTime.of(2026, 7, 8, 11, 30, 0)
        )
    )

    orders.toItemResponses().forEach(::println)
}
```

输出：

```text
OrderItemResponse(id=1001, buyerName=匿名客户, amount=¥19.90, status=已支付, createdAt=2026-07-08 10:00:00)
OrderItemResponse(id=1002, buyerName=Lucy, amount=¥5.00, status=待支付, createdAt=2026-07-08 11:30:00)
```

这个 Demo 里，每个扩展都只做一件小事：

```text
String? 负责空名称兜底
BigDecimal 负责金额文本
LocalDateTime 负责时间文本
String 负责状态文本
OrderEntity 负责转 DTO
List<OrderEntity> 负责批量转换
```

调用点读起来就很直：

```kotlin
orders.toItemResponses()
```

### 写扩展的几条实用规则

扩展用得舒服，通常符合这些规则：

* 函数短，行为清楚，一眼能看懂
* 接收者类型和函数语义强相关
* 不隐藏网络请求、数据库写入、事务、消息发送
* 不依赖隐式全局状态
* 名字具体，避免和标准库、成员函数冲突
* 公共扩展放到清晰包名下，局部扩展使用 `private`
* 扩展属性只做计算，不伪装成对象字段
* 需要多态时使用成员函数，不使用扩展函数

### 总结

Kotlin Extension 最大的价值，是把“围绕某个类型的小能力”写到更贴近调用语义的位置。

它能减少工具类噪音，也能让格式化、转换、DSL、集合辅助函数变得更自然。

但它不是继承，不是覆盖，也不是运行时补丁。

核心规则可以记成几句话：

```text
扩展像成员一样调用，本质还是静态函数
成员函数优先于扩展函数
扩展按声明类型静态解析，没有运行时多态
扩展属性没有幕后字段，只能计算或转发
可空接收者可以直接处理 null
```

真正好用的扩展，通常都很小、很具体、很贴近类型。把它当作“更自然的工具函数”，而不是“万能改造器”，代码会清爽很多。
