### 简介

`apply` 是 Kotlin 标准库里的作用域函数。

作用域函数常见有 5 个：

* `let`
* `run`
* `with`
* `apply`
* `also`

`apply` 主要用于对象初始化、属性赋值、链式配置。

一句话概括：

> `apply` 会在对象自己的作用域里执行一段代码，执行完成后返回对象本身。

常见写法：

```kotlin
val user = User().apply {
    name = "Tom"
    age = 18
}
```

这段代码的含义是：

```text
创建 User 对象
进入 User 对象的作用域
设置 name 和 age
返回这个 User 对象
```

### apply 的源码结构

Kotlin 标准库里，`apply` 大致可以理解成这样：

```kotlin
public inline fun <T> T.apply(block: T.() -> Unit): T {
    block()
    return this
}
```

这段代码有几个关键点：

* `T`：调用 `apply` 的对象类型
* `block: T.() -> Unit`：带接收者的 Lambda
* `block()`：在当前对象作用域里执行 Lambda
* `return this`：返回当前对象本身

所以 `apply` 的重点是两个：

```text
Lambda 里用 this 访问对象
apply 返回对象本身
```

### 基础用法

先定义一个简单的数据类：

```kotlin
data class User(
    var name: String = "",
    var age: Int = 0,
    var city: String = ""
)
```

普通写法：

```kotlin
fun main() {
    val user = User()
    user.name = "Tom"
    user.age = 18
    user.city = "上海"

    println(user)
}
```

使用 `apply`：

```kotlin
fun main() {
    val user = User().apply {
        name = "Tom"
        age = 18
        city = "上海"
    }

    println(user)
}
```

输出：

```text
User(name=Tom, age=18, city=上海)
```

在 `apply` 代码块里，可以直接访问 `User` 的属性。

下面两种写法等价：

```kotlin
val user = User().apply {
    name = "Tom"
    age = 18
}
```

```kotlin
val user = User().apply {
    this.name = "Tom"
    this.age = 18
}
```

`this` 通常可以省略。

### apply 的返回值

`apply` 返回的是调用对象本身。

```kotlin
fun main() {
    val user = User().apply {
        name = "Tom"
        age = 18
        "最后一行字符串"
    }

    println(user)
}
```

输出：

```text
User(name=Tom, age=18, city=)
```

虽然 Lambda 最后一行是字符串：

```kotlin
"最后一行字符串"
```

但 `apply` 不会返回这个字符串，而是返回 `User` 对象。

这是 `apply` 和 `let` 很重要的区别：

```kotlin
val length = "Kotlin".let {
    it.length
}
```

`let` 返回 Lambda 最后一行结果，也就是 `6`。

```kotlin
val value = "Kotlin".apply {
    uppercase()
}
```

`apply` 返回原对象，也就是 `"Kotlin"`。

### this 接收者

`apply` 的 Lambda 类型是：

```kotlin
T.() -> Unit
```

这叫带接收者的 Lambda。

普通 Lambda：

```kotlin
(User) -> Unit
```

内部一般通过参数访问对象：

```kotlin
val block: (User) -> Unit = { user ->
    user.name = "Tom"
}
```

带接收者的 Lambda：

```kotlin
User.() -> Unit
```

内部可以直接访问 `User` 的成员：

```kotlin
val block: User.() -> Unit = {
    name = "Tom"
}
```

`apply` 就是使用这种形式，所以代码块里可以直接写属性名和方法名。

### 对象初始化

`apply` 适合创建对象后连续设置多个属性。

```kotlin
data class ServerConfig(
    var host: String = "",
    var port: Int = 0,
    var timeoutSeconds: Int = 0,
    var debug: Boolean = false
)

fun main() {
    val config = ServerConfig().apply {
        host = "127.0.0.1"
        port = 8080
        timeoutSeconds = 30
        debug = true
    }

    println(config)
}
```

输出：

```text
ServerConfig(host=127.0.0.1, port=8080, timeoutSeconds=30, debug=true)
```

这种写法把对象创建和对象配置放在一起，适合配置项比较集中的场景。

### 集合初始化

可变集合初始化时，也可以使用 `apply`。

```kotlin
fun main() {
    val languages = mutableListOf<String>().apply {
        add("Java")
        add("Kotlin")
        add("Go")
    }

    println(languages)
}
```

输出：

```text
[Java, Kotlin, Go]
```

Map 也一样：

```kotlin
fun main() {
    val scores = mutableMapOf<String, Int>().apply {
        put("Kotlin", 100)
        put("Java", 90)
        put("Go", 85)
    }

    println(scores)
}
```

可能输出：

```text
{Kotlin=100, Java=90, Go=85}
```

### 链式配置

因为 `apply` 返回对象本身，所以可以继续链式调用。

```kotlin
data class Product(
    var name: String = "",
    var price: Double = 0.0,
    var enabled: Boolean = true
)

fun main() {
    val product = Product()
        .apply {
            name = "Kotlin 入门课程"
            price = 99.0
        }
        .apply {
            enabled = price > 0
        }

    println(product)
}
```

输出：

```text
Product(name=Kotlin 入门课程, price=99.0, enabled=true)
```

多个 `apply` 可以串起来，不过如果配置逻辑属于同一阶段，放在一个 `apply` 里通常更清楚。

### 配合 also

`apply` 和 `also` 都返回对象本身。

区别是：

* `apply` 通过 `this` 访问对象，适合配置对象
* `also` 通过 `it` 访问对象，适合日志、调试、附加动作

示例：

```kotlin
data class Order(
    var id: String = "",
    var amount: Double = 0.0,
    var status: String = "CREATED"
)

fun main() {
    val order = Order()
        .apply {
            id = "A001"
            amount = 199.0
            status = "PAID"
        }
        .also {
            println("订单创建完成: ${it.id}, 金额=${it.amount}")
        }

    println(order)
}
```

输出：

```text
订单创建完成: A001, 金额=199.0
Order(id=A001, amount=199.0, status=PAID)
```

这里 `apply` 负责设置属性，`also` 负责打印日志。

### 配合可空对象

`apply` 可以和安全调用符 `?.` 配合使用。

```kotlin
data class Profile(
    var nickname: String = "",
    var email: String = "",
    var verified: Boolean = false
)

fun loadProfile(id: Int): Profile? {
    return if (id == 1) Profile() else null
}

fun main() {
    val profile = loadProfile(1)?.apply {
        nickname = "Tom"
        email = "tom@example.com"
        verified = true
    }

    println(profile)
}
```

输出：

```text
Profile(nickname=Tom, email=tom@example.com, verified=true)
```

如果 `loadProfile` 返回 `null`，`apply` 不会执行，最终结果也是 `null`。

### apply 不适合做值转换

`apply` 返回对象本身，不返回 Lambda 最后一行。

```kotlin
fun main() {
    val result = "Kotlin".apply {
        length
    }

    println(result)
}
```

输出：

```text
Kotlin
```

如果目标是拿到长度，可以使用 `let` 或 `run`：

```kotlin
val length1 = "Kotlin".let {
    it.length
}

val length2 = "Kotlin".run {
    length
}
```

`apply` 更适合“配置这个对象，然后继续得到这个对象”。

### 实战 Demo：构建请求对象

后端接口、HTTP 请求、数据库查询，经常需要先组装一个对象。

```kotlin
data class SearchRequest(
    var keyword: String = "",
    var pageIndex: Int = 1,
    var pageSize: Int = 20,
    var sortBy: String = "createdAt",
    var descending: Boolean = true
)

fun buildSearchRequest(keyword: String?, page: Int?): SearchRequest {
    return SearchRequest().apply {
        this.keyword = keyword?.trim().orEmpty()
        pageIndex = page?.takeIf { it > 0 } ?: 1
        pageSize = 20
        sortBy = "createdAt"
        descending = true
    }
}

fun main() {
    val request = buildSearchRequest("  kotlin  ", 2)
    println(request)
}
```

输出：

```text
SearchRequest(keyword=kotlin, pageIndex=2, pageSize=20, sortBy=createdAt, descending=true)
```

`apply` 把请求对象的组装过程放在一个作用域里，返回值仍然是 `SearchRequest`。

### 实战 Demo：订单创建

```kotlin
data class OrderItem(
    val sku: String,
    val quantity: Int,
    val price: Double
)

data class CreateOrderCommand(
    var userId: Long = 0,
    var items: MutableList<OrderItem> = mutableListOf(),
    var totalAmount: Double = 0.0,
    var remark: String = ""
)

fun createOrderCommand(userId: Long, items: List<OrderItem>, remark: String?): CreateOrderCommand {
    return CreateOrderCommand().apply {
        this.userId = userId
        this.items.addAll(items)
        totalAmount = items.sumOf { it.quantity * it.price }
        this.remark = remark?.trim().orEmpty()
    }
}

fun main() {
    val command = createOrderCommand(
        userId = 1001,
        items = listOf(
            OrderItem("BOOK", 2, 59.0),
            OrderItem("PEN", 3, 5.0)
        ),
        remark = "  尽快发货  "
    )

    println(command)
}
```

输出：

```text
CreateOrderCommand(userId=1001, items=[OrderItem(sku=BOOK, quantity=2, price=59.0), OrderItem(sku=PEN, quantity=3, price=5.0)], totalAmount=133.0, remark=尽快发货)
```

这里需要设置多个字段，还要计算总金额。`apply` 能让对象构建过程集中在一起。

### 实战 Demo：配置对象校验

`apply` 也可以和普通函数配合，在配置完成后做校验。

```kotlin
data class DatabaseConfig(
    var host: String = "",
    var port: Int = 3306,
    var username: String = "",
    var password: String = "",
    var database: String = ""
)

fun DatabaseConfig.validate(): DatabaseConfig {
    require(host.isNotBlank()) { "host 不能为空" }
    require(port in 1..65535) { "port 不合法" }
    require(username.isNotBlank()) { "username 不能为空" }
    require(database.isNotBlank()) { "database 不能为空" }
    return this
}

fun main() {
    val config = DatabaseConfig()
        .apply {
            host = "127.0.0.1"
            port = 3306
            username = "root"
            password = "123456"
            database = "demo"
        }
        .validate()

    println(config)
}
```

输出：

```text
DatabaseConfig(host=127.0.0.1, port=3306, username=root, password=123456, database=demo)
```

`apply` 负责配置，`validate()` 负责校验。职责分开后，代码更容易维护。

### 实战 Demo：简单 Builder 风格 API

`apply` 很适合实现简单的 Builder 风格 API。

```kotlin
class DialogConfig {
    var title: String = ""
    var message: String = ""
    var positiveText: String = "确定"
    var negativeText: String = "取消"
    var cancelable: Boolean = true
    var onPositive: (() -> Unit)? = null
}

fun dialog(block: DialogConfig.() -> Unit): DialogConfig {
    return DialogConfig().apply(block)
}

fun main() {
    val config = dialog {
        title = "删除确认"
        message = "确认删除这条记录吗？"
        positiveText = "删除"
        negativeText = "取消"
        cancelable = false
        onPositive = {
            println("执行删除")
        }
    }

    println(config.title)
    println(config.message)
    config.onPositive?.invoke()
}
```

输出：

```text
删除确认
确认删除这条记录吗？
执行删除
```

`dialog { ... }` 内部其实就是：

```kotlin
DialogConfig().apply(block)
```

这个模式在配置对象、构建 DSL 风格 API 时很常见。

### 实战 Demo：文件写入配置

下面示例只展示写法，运行时会在当前目录创建文件。

```kotlin
import java.io.File
import java.nio.charset.Charset

data class FileWriteOptions(
    var charset: String = "UTF-8",
    var append: Boolean = false,
    var createParentDirs: Boolean = true
)

fun writeTextFile(
    path: String,
    content: String,
    options: FileWriteOptions = FileWriteOptions()
) {
    val file = File(path)

    options.apply {
        if (createParentDirs) {
            file.parentFile?.mkdirs()
        }

        if (append) {
            file.appendText(content, Charset.forName(charset))
        } else {
            file.writeText(content, Charset.forName(charset))
        }
    }
}

fun main() {
    val options = FileWriteOptions().apply {
        append = false
        createParentDirs = true
    }

    writeTextFile("output/apply-demo.txt", "hello apply", options)
}
```

`options.apply { ... }` 里读取配置项，`FileWriteOptions().apply { ... }` 里设置配置项。

### 嵌套 apply 和 this 指向

嵌套 `apply` 时，`this` 容易指向内层对象。

```kotlin
class Address {
    var city: String = ""
    var street: String = ""
}

class Customer {
    var name: String = ""
    var address: Address = Address()
}

fun main() {
    val customer = Customer().apply {
        name = "Tom"

        address.apply {
            city = "上海"
            street = "人民路"
        }
    }

    println(customer.name)
    println(customer.address.city)
}
```

在外层 `apply` 里，`this` 是 `Customer`。

在内层 `address.apply` 里，`this` 是 `Address`。

如果需要在内层访问外层对象，可以使用标签：

```kotlin
fun main() {
    val customer = Customer().apply customerBlock@{
        name = "Tom"

        address.apply {
            city = "上海"
            street = "${this@customerBlock.name} 的地址"
        }
    }

    println(customer.address.street)
}
```

输出：

```text
Tom 的地址
```

`this@customerBlock` 表示外层 `apply` 的接收者，也就是 `Customer`。

### 自定义 applyIf

有时只想在满足条件时配置对象，可以定义一个扩展函数。

```kotlin
inline fun <T> T.applyIf(condition: Boolean, block: T.() -> Unit): T {
    if (condition) {
        apply(block)
    }
    return this
}

data class ApiOptions(
    var enableCache: Boolean = false,
    var timeoutSeconds: Int = 10,
    var retryTimes: Int = 0
)

fun main() {
    val debug = true

    val options = ApiOptions()
        .apply {
            timeoutSeconds = 30
        }
        .applyIf(debug) {
            enableCache = false
            retryTimes = 1
        }

    println(options)
}
```

输出：

```text
ApiOptions(enableCache=false, timeoutSeconds=30, retryTimes=1)
```

`applyIf` 保持了 `apply` 的特点：执行配置后仍然返回对象本身。

### apply、let、also、run 的区别

作用域函数主要看两个问题：

```text
Lambda 里怎么访问对象
函数返回什么
```

| 函数 | 上下文对象 | 返回值 | 常见用途 |
| --- | --- | --- | --- |
| `apply` | `this` | 对象本身 | 初始化、配置对象 |
| `also` | `it` | 对象本身 | 日志、调试、附加动作 |
| `let` | `it` | Lambda 结果 | 空安全、数据转换 |
| `run` | `this` | Lambda 结果 | 在对象作用域里计算结果 |
| `with` | `this` | Lambda 结果 | 对已有对象集中调用 |

#### apply 返回对象本身

```kotlin
val user = User().apply {
    name = "Tom"
}
```

#### let 返回 Lambda 结果

```kotlin
val name = User().apply {
    this.name = "Tom"
}.let {
    it.name
}
```

#### also 返回对象本身

```kotlin
val user = User().apply {
    name = "Tom"
}.also {
    println(it.name)
}
```

#### run 返回 Lambda 结果

```kotlin
val name = User().apply {
    this.name = "Tom"
}.run {
    name
}
```

简单选择方式：

```text
需要初始化或配置对象：apply
需要返回原对象并顺手做日志：also
需要把对象转换成另一个结果：let
需要在对象作用域里计算一个结果：run
```

### 常见注意点

#### apply 返回原对象，不返回最后一行

```kotlin
fun main() {
    val result = StringBuilder().apply {
        append("Kotlin")
        length
    }

    println(result)
}
```

输出：

```text
Kotlin
```

虽然最后一行是 `length`，但 `result` 仍然是 `StringBuilder`。

#### 不可变对象调用 apply 不会改变值本身

```kotlin
fun main() {
    val text = "kotlin".apply {
        uppercase()
    }

    println(text)
}
```

输出：

```text
kotlin
```

`String` 是不可变对象，`uppercase()` 会返回新字符串，但 `apply` 返回的仍然是原字符串。

如果目标是拿到大写结果，可以写成：

```kotlin
val text = "kotlin".uppercase()
```

或者：

```kotlin
val text = "kotlin".let {
    it.uppercase()
}
```

#### apply 里适合放配置逻辑

```kotlin
val user = User().apply {
    name = "Tom"
    age = 18
}
```

这类属性设置很适合放在 `apply` 里。

如果代码块里开始出现大量业务判断、数据库访问、网络请求，提取成普通函数通常更清楚。

#### 嵌套 apply 需要留意 this

```kotlin
outer.apply {
    inner.apply {
        // this 是 inner
    }
}
```

嵌套层级增加后，可以使用命名函数、局部变量或标签减少歧义。

### 总结

`apply` 的核心可以记成几句话：

* `apply` 是作用域函数
* Lambda 内部通过 `this` 访问调用对象
* `this` 通常可以省略
* `apply` 返回调用对象本身
* `apply` 适合对象初始化、属性赋值、链式配置
* 需要返回 Lambda 结果时，通常使用 `let` 或 `run`
* 嵌套 `apply` 时，需要关注 `this` 指向

`apply` 的定位很明确：围绕一个对象做配置，然后继续得到这个对象。只要目标是“配置对象本身”，`apply` 通常就是合适的选择。
