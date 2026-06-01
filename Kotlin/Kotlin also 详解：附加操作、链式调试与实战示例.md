### 简介

`also` 是 Kotlin 标准库里的作用域函数。

作用域函数常见有 5 个：

* `let`
* `run`
* `with`
* `apply`
* `also`

`also` 主要用于附加操作，比如打印日志、调试中间结果、做校验、记录埋点、保存审计信息。

一句话概括：

> `also` 会把当前对象传进 Lambda，执行一段附加逻辑，然后返回对象本身。

常见写法：

```kotlin
val result = value.also {
    println(it)
}
```

这里的 `result` 仍然是 `value` 本身。

### also 的源码结构

Kotlin 标准库里，`also` 大致可以理解成这样：

```kotlin
public inline fun <T> T.also(block: (T) -> Unit): T {
    block(this)
    return this
}
```

这段代码有几个关键点：

* `T`：调用 `also` 的对象类型
* `block: (T) -> Unit`：接收当前对象、无返回值要求的 Lambda
* `block(this)`：把当前对象传给 Lambda
* `return this`：返回当前对象本身

所以 `also` 的重点是两个：

```text
Lambda 里用 it 访问对象
also 返回对象本身
```

### 基础用法

```kotlin
fun main() {
    val name = "Kotlin"

    val result = name.also {
        println("当前值: $it")
    }

    println(result)
}
```

输出：

```text
当前值: Kotlin
Kotlin
```

`also` 里面打印了当前对象，但返回值仍然是原来的字符串 `"Kotlin"`。

也可以显式命名参数：

```kotlin
fun main() {
    val name = "Kotlin"

    val result = name.also { value ->
        println("当前值: $value")
    }

    println(result)
}
```

逻辑简单时，用 `it` 就够了。逻辑变长时，命名参数更清楚。

### also 的返回值

`also` 返回调用对象本身，不返回 Lambda 最后一行。

```kotlin
fun main() {
    val result = "Kotlin".also {
        it.length
    }

    println(result)
}
```

输出：

```text
Kotlin
```

虽然 Lambda 最后一行是：

```kotlin
it.length
```

但 `also` 返回的是 `"Kotlin"`，不是 `6`。

如果目标是得到长度，可以使用 `let`：

```kotlin
val length = "Kotlin".let {
    it.length
}
```

### also 和 apply 的区别

`also` 和 `apply` 都返回对象本身。

区别主要在 Lambda 里访问对象的方式，以及表达的意图。

| 函数 | 上下文对象 | 返回值 | 常见用途 |
| --- | --- | --- | --- |
| `also` | `it` | 对象本身 | 日志、调试、校验、附加动作 |
| `apply` | `this` | 对象本身 | 对象初始化、属性配置 |

`apply` 更像“配置这个对象”：

```kotlin
data class User(
    var name: String = "",
    var age: Int = 0
)

val user = User().apply {
    name = "Tom"
    age = 18
}
```

`also` 更像“顺带对这个对象做点事”：

```kotlin
val user = User("Tom", 18).also {
    println("创建用户: $it")
}
```

两者可以组合使用：

```kotlin
val user = User()
    .apply {
        name = "Tom"
        age = 18
    }
    .also {
        println("用户配置完成: $it")
    }
```

### 链式调用中插入日志

`also` 很适合在链式调用中查看中间结果。

```kotlin
fun main() {
    val result = "  kotlin also  "
        .trim()
        .also {
            println("trim 后: $it")
        }
        .uppercase()
        .also {
            println("uppercase 后: $it")
        }

    println("最终结果: $result")
}
```

输出：

```text
trim 后: kotlin also
uppercase 后: KOTLIN ALSO
最终结果: KOTLIN ALSO
```

`also` 不改变链式调用里的主结果，只是在中间插入一段附加逻辑。

### 集合处理中查看中间结果

```kotlin
fun main() {
    val total = listOf(1, 2, 3, 4, 5, 6)
        .filter { it % 2 == 0 }
        .also {
            println("偶数: $it")
        }
        .map { it * it }
        .also {
            println("平方后: $it")
        }
        .sum()

    println("总和: $total")
}
```

输出：

```text
偶数: [2, 4, 6]
平方后: [4, 16, 36]
总和: 56
```

这种写法适合排查数据处理链路，尤其是 `filter`、`map`、`groupBy`、`sortedBy` 连在一起时。

### 配合可空对象

`also` 可以和安全调用符 `?.` 一起使用。

```kotlin
fun loadToken(): String? {
    return "abc123"
}

fun main() {
    val token = loadToken()
        ?.also {
            println("读取到 token，长度=${it.length}")
        }
        ?: "default-token"

    println(token)
}
```

输出：

```text
读取到 token，长度=6
abc123
```

如果 `loadToken()` 返回 `null`，`also` 不会执行，最终走 Elvis 操作符 `?:` 后面的默认值。

### also 可以修改可变对象

`also` 常用于附加操作，但它拿到的是对象本身。如果对象是可变的，`also` 里也能修改对象。

```kotlin
fun main() {
    val list = mutableListOf("A", "B").also {
        it.add("C")
        it.remove("A")
    }

    println(list)
}
```

输出：

```text
[B, C]
```

这段代码能正常工作。

不过如果主要目标是初始化或配置对象属性，`apply` 通常表达得更直接：

```kotlin
val user = User().apply {
    name = "Tom"
    age = 18
}
```

如果主要目标是记录、校验、调试、埋点，`also` 更贴近语义。

### 实战 Demo：创建对象后记录日志

```kotlin
data class User(
    val id: Long,
    val name: String,
    val age: Int
)

fun createUser(id: Long, name: String, age: Int): User {
    return User(id, name, age)
        .also {
            println("创建用户: id=${it.id}, name=${it.name}")
        }
}

fun main() {
    val user = createUser(1, "Tom", 18)
    println(user)
}
```

输出：

```text
创建用户: id=1, name=Tom
User(id=1, name=Tom, age=18)
```

`also` 在这里负责记录信息，函数返回值仍然是 `User`。

### 实战 Demo：配置对象后打印结果

```kotlin
data class ServerConfig(
    var host: String = "",
    var port: Int = 0,
    var timeoutSeconds: Int = 0
)

fun main() {
    val config = ServerConfig()
        .apply {
            host = "127.0.0.1"
            port = 8080
            timeoutSeconds = 30
        }
        .also {
            println("配置加载完成: ${it.host}:${it.port}, timeout=${it.timeoutSeconds}s")
        }

    println(config)
}
```

输出：

```text
配置加载完成: 127.0.0.1:8080, timeout=30s
ServerConfig(host=127.0.0.1, port=8080, timeoutSeconds=30)
```

`apply` 负责配置对象，`also` 负责配置完成后的附加动作。

### 实战 Demo：订单校验

```kotlin
data class OrderItem(
    val sku: String,
    val quantity: Int,
    val price: Double
)

data class Order(
    val id: String,
    val items: List<OrderItem>,
    val amount: Double
)

fun validateOrder(order: Order): Order {
    return order.also {
        require(it.id.isNotBlank()) { "订单 id 不能为空" }
        require(it.items.isNotEmpty()) { "订单商品不能为空" }
        require(it.amount > 0) { "订单金额必须大于 0" }
    }
}

fun main() {
    val order = Order(
        id = "A001",
        items = listOf(
            OrderItem("BOOK", 2, 59.0)
        ),
        amount = 118.0
    )

    val validOrder = validateOrder(order)
    println(validOrder)
}
```

输出：

```text
Order(id=A001, items=[OrderItem(sku=BOOK, quantity=2, price=59.0)], amount=118.0)
```

校验通过后，`validateOrder` 继续返回原订单对象。

### 实战 Demo：保存数据后记录结果

```kotlin
class Repository {
    fun save(content: String): Boolean {
        println("保存内容: $content")
        return content.isNotBlank()
    }
}

fun saveWithLog(repository: Repository, content: String): Boolean {
    return repository.save(content).also { success ->
        println("保存结果: ${if (success) "成功" else "失败"}")
    }
}

fun main() {
    val repository = Repository()
    val result = saveWithLog(repository, "Kotlin also")

    println("最终返回: $result")
}
```

输出：

```text
保存内容: Kotlin also
保存结果: 成功
最终返回: true
```

这里的调用对象是 `Boolean`。

`also` 打印保存结果后，返回值仍然是原来的 `Boolean`。

### 实战 Demo：文件创建后附加处理

下面示例会在系统临时目录创建文件。

```kotlin
import java.io.File

fun createTempLogFile(prefix: String): File {
    return File.createTempFile(prefix, ".log")
        .also { file ->
            file.writeText("log initialized\n")
        }
        .also { file ->
            println("临时日志文件: ${file.absolutePath}")
            file.deleteOnExit()
        }
}

fun main() {
    val file = createTempLogFile("kotlin-also-")
    println(file.name)
}
```

可能输出：

```text
临时日志文件: /var/folders/.../kotlin-also-123456.log
kotlin-also-123456.log
```

每个 `also` 都返回同一个 `File` 对象，所以可以连续追加附加动作。

### 实战 Demo：订单统计链路调试

准备数据：

```kotlin
data class PayOrder(
    val id: String,
    val city: String,
    val amount: Double,
    val paid: Boolean
)

val orders = listOf(
    PayOrder("A001", "上海", 120.0, true),
    PayOrder("A002", "北京", 80.0, false),
    PayOrder("A003", "上海", 260.0, true),
    PayOrder("A004", "深圳", 300.0, true),
    PayOrder("A005", "北京", 180.0, true)
)
```

需求：

* 只统计已支付订单
* 按城市分组
* 统计每个城市总金额
* 按总金额倒序

代码：

```kotlin
fun main() {
    val reports = orders
        .filter { it.paid }
        .also {
            println("已支付订单: $it")
        }
        .groupBy { it.city }
        .also {
            println("按城市分组: $it")
        }
        .map { (city, cityOrders) ->
            city to cityOrders.sumOf { it.amount }
        }
        .sortedByDescending { it.second }
        .also {
            println("排序结果: $it")
        }

    reports.forEach { (city, amount) ->
        println("$city: $amount")
    }
}
```

输出示例：

```text
已支付订单: [PayOrder(id=A001, city=上海, amount=120.0, paid=true), PayOrder(id=A003, city=上海, amount=260.0, paid=true), PayOrder(id=A004, city=深圳, amount=300.0, paid=true), PayOrder(id=A005, city=北京, amount=180.0, paid=true)]
按城市分组: {上海=[PayOrder(id=A001, city=上海, amount=120.0, paid=true), PayOrder(id=A003, city=上海, amount=260.0, paid=true)], 深圳=[PayOrder(id=A004, city=深圳, amount=300.0, paid=true)], 北京=[PayOrder(id=A005, city=北京, amount=180.0, paid=true)]}
排序结果: [(上海, 380.0), (深圳, 300.0), (北京, 180.0)]
上海: 380.0
深圳: 300.0
北京: 180.0
```

这类代码适合临时排查数据处理过程。稳定后，如果日志太多，可以删除部分 `also`，或者提取成专门的日志函数。

### 自定义 alsoIf

有时只想在满足条件时执行附加动作，可以定义一个扩展函数。

```kotlin
inline fun <T> T.alsoIf(condition: Boolean, block: (T) -> Unit): T {
    if (condition) {
        also(block)
    }
    return this
}

fun main() {
    val debug = true

    val result = listOf(1, 2, 3, 4)
        .filter { it % 2 == 0 }
        .alsoIf(debug) {
            println("debug: $it")
        }
        .map { it * 10 }

    println(result)
}
```

输出：

```text
debug: [2, 4]
[20, 40]
```

`alsoIf` 的返回值仍然是调用对象本身，所以不会打断链式调用。

### also、apply、let、run 的区别

作用域函数主要看两个问题：

```text
Lambda 里怎么访问对象
函数返回什么
```

| 函数 | 上下文对象 | 返回值 | 常见用途 |
| --- | --- | --- | --- |
| `also` | `it` | 对象本身 | 日志、调试、校验、附加动作 |
| `apply` | `this` | 对象本身 | 初始化、配置对象 |
| `let` | `it` | Lambda 结果 | 空安全、数据转换 |
| `run` | `this` | Lambda 结果 | 在对象作用域里计算结果 |
| `with` | `this` | Lambda 结果 | 对已有对象集中调用 |

简单选择方式：

```text
需要附加动作并继续返回原对象：also
需要初始化或配置对象：apply
需要把对象转换成另一个结果：let
需要在对象作用域里计算一个结果：run
```

### 常见注意点

#### also 返回原对象，不返回最后一行

```kotlin
fun main() {
    val result = "Kotlin".also {
        it.length
    }

    println(result)
}
```

输出：

```text
Kotlin
```

#### also 里可以修改对象，但语义要清楚

```kotlin
val list = mutableListOf<String>().also {
    it.add("A")
    it.add("B")
}
```

这段代码可以运行。

不过如果主要目标是初始化集合，`apply` 更常见：

```kotlin
val list = mutableListOf<String>().apply {
    add("A")
    add("B")
}
```

#### 复杂逻辑可以提取函数

```kotlin
fun logOrder(order: Order) {
    println("订单: ${order.id}, amount=${order.amount}")
}

val order = Order("A001", emptyList(), 100.0)
    .also(::logOrder)
```

当 `also` 里的逻辑超过几行，提取成有名字的函数通常更容易阅读。

#### 多个 also 适合调试，长期代码需要克制

```kotlin
val result = orders
    .filter { it.paid }
    .also { println("paid=$it") }
    .groupBy { it.city }
    .also { println("grouped=$it") }
    .mapValues { it.value.sumOf { order -> order.amount } }
    .also { println("total=$it") }
```

调试时这样写很方便。业务代码稳定后，可以保留关键日志，或者改成更统一的日志封装。

### 总结

`also` 的核心可以记成几句话：

* `also` 是作用域函数
* Lambda 内部通过 `it` 访问调用对象
* `also` 返回调用对象本身
* `also` 适合日志、调试、校验、埋点等附加动作
* `apply` 更适合初始化和配置对象
* `let` 更适合把对象转换成另一个结果
* `also` 可以插入链式调用中，不改变主流程返回值

`also` 的定位很清晰：围绕当前对象做一段附加动作，然后继续把当前对象交给后续流程。
