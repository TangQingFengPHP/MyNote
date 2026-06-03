### 简介

`run` 是 Kotlin 标准库里的作用域函数。

作用域函数常见有 5 个：

* `let`
* `run`
* `with`
* `apply`
* `also`

`run` 的特点比较鲜明：

> 在对象作用域里执行一段逻辑，然后返回 Lambda 最后一行的结果。

常见写法：

```kotlin
val result = user.run {
    "$name-$age"
}
```

这里的 `name`、`age` 来自 `user` 对象，`result` 是 Lambda 最后一行的字符串。

`run` 适合这类场景：

* 在一个对象里连续读取多个属性
* 对对象做一段计算并返回结果
* 可空对象非空时执行并返回结果
* 创建一个临时作用域，收拢中间变量
* 配合 `apply` 完成“先配置，再计算”

### run 有两种形式

Kotlin 里有两种 `run`。

#### 对象扩展函数 run

```kotlin
val result = obj.run {
    // this 是 obj
    // 最后一行作为返回值
}
```

源码大致可以理解成：

```kotlin
public inline fun <T, R> T.run(block: T.() -> R): R {
    return block()
}
```

特点：

```text
Lambda 里用 this 访问对象
run 返回 Lambda 最后一行结果
```

#### 顶层函数 run

```kotlin
val result = run {
    // 没有接收者对象
    // 最后一行作为返回值
}
```

源码大致可以理解成：

```kotlin
public inline fun <R> run(block: () -> R): R {
    return block()
}
```

特点：

```text
没有上下文对象
主要用于创建临时作用域
返回 Lambda 最后一行结果
```

### 基础用法

先定义一个简单的数据类：

```kotlin
data class User(
    val name: String,
    val age: Int
)
```

使用 `run`：

```kotlin
fun main() {
    val user = User("Tom", 18)

    val info = user.run {
        "$name-$age"
    }

    println(info)
}
```

输出：

```text
Tom-18
```

在 `run` 代码块里，`this` 就是 `user`。

下面两种写法等价：

```kotlin
val info = user.run {
    "$name-$age"
}
```

```kotlin
val info = user.run {
    "${this.name}-${this.age}"
}
```

`this` 通常可以省略。

### run 的返回值

`run` 返回 Lambda 最后一行结果。

```kotlin
fun main() {
    val user = User("Tom", 18)

    val nextAge = user.run {
        age + 1
    }

    println(nextAge)
}
```

输出：

```text
19
```

再看一个多行示例：

```kotlin
fun main() {
    val user = User("Tom", 18)

    val result = user.run {
        val adult = age >= 18
        "name=$name, adult=$adult"
    }

    println(result)
}
```

输出：

```text
name=Tom, adult=true
```

最后一行：

```kotlin
"name=$name, adult=$adult"
```

就是整个 `run` 的返回值。

### this 接收者

`run` 的扩展函数版本使用带接收者的 Lambda：

```kotlin
T.() -> R
```

所以 Lambda 里可以直接访问对象成员。

```kotlin
data class Product(
    val name: String,
    val price: Double,
    val discountRate: Double
)

fun main() {
    val product = Product("Keyboard", 300.0, 0.8)

    val finalPrice = product.run {
        price * discountRate
    }

    println(finalPrice)
}
```

输出：

```text
240.0
```

如果使用 `let`，通常要通过 `it` 访问对象：

```kotlin
val finalPrice = product.let {
    it.price * it.discountRate
}
```

如果需要频繁访问同一个对象的属性，`run` 的 `this` 风格会更紧凑。

### 和 let 的区别

`run` 和 `let` 都返回 Lambda 结果。

区别在于上下文对象：

| 函数 | 上下文对象 | 返回值 | 常见用途 |
| --- | --- | --- | --- |
| `run` | `this` | Lambda 结果 | 在对象作用域里计算结果 |
| `let` | `it` | Lambda 结果 | 空安全、链式转换、显式参数 |

示例：

```kotlin
val user = User("Tom", 18)

val a = user.run {
    "$name-$age"
}

val b = user.let {
    "${it.name}-${it.age}"
}

println(a)
println(b)
```

输出：

```text
Tom-18
Tom-18
```

对象属性较多时，`run` 可以少写对象名或 `it`。

参数名能提升可读性时，`let` 更合适：

```kotlin
val result = user.let { currentUser ->
    "${currentUser.name}-${currentUser.age}"
}
```

### 和 apply 的区别

`run` 和 `apply` 都使用 `this`。

区别在于返回值。

| 函数 | 上下文对象 | 返回值 | 常见用途 |
| --- | --- | --- | --- |
| `run` | `this` | Lambda 结果 | 计算结果 |
| `apply` | `this` | 对象本身 | 初始化、配置对象 |

示例：

```kotlin
data class ServerConfig(
    var host: String = "",
    var port: Int = 0
)

fun main() {
    val config = ServerConfig().apply {
        host = "127.0.0.1"
        port = 8080
    }

    val address = config.run {
        "$host:$port"
    }

    println(address)
}
```

输出：

```text
127.0.0.1:8080
```

`apply` 负责把对象配置好，`run` 负责从对象中计算一个结果。

### 可空对象配合 run

`run` 可以和安全调用符 `?.` 配合使用。

```kotlin
data class Address(
    val city: String,
    val street: String
)

data class Customer(
    val name: String,
    val address: Address?
)

fun loadCustomer(id: Int): Customer? {
    return if (id == 1) {
        Customer("Tom", Address("上海", "人民路"))
    } else {
        null
    }
}

fun main() {
    val addressText = loadCustomer(1)?.run {
        val cityName = address?.city ?: "未知城市"
        "$name - $cityName"
    } ?: "客户不存在"

    println(addressText)
}
```

输出：

```text
Tom - 上海
```

含义是：

```text
客户非空 -> 进入 run 代码块并返回字符串
客户为空 -> 返回 客户不存在
```

### 顶层 run：临时作用域

顶层 `run` 没有接收者对象，主要用于创建一个临时作用域。

```kotlin
fun main() {
    val result = run {
        val a = 10
        val b = 20
        a + b
    }

    println(result)
}
```

输出：

```text
30
```

`a`、`b` 只在 `run` 代码块里有效，不会泄漏到外层作用域。

### 顶层 run：表达式化分支逻辑

多步计算可以用顶层 `run` 收成一个表达式。

```kotlin
fun formatInput(input: String?): String {
    return run {
        val value = input?.trim().orEmpty()

        if (value.isBlank()) {
            "EMPTY"
        } else {
            value.uppercase()
        }
    }
}

fun main() {
    println(formatInput("  kotlin  "))
    println(formatInput("   "))
}
```

输出：

```text
KOTLIN
EMPTY
```

这里的 `run` 只是为了收拢临时变量 `value`，让函数保持表达式风格。

### 链式调用中使用 run

`run` 可以放在链式调用末尾，把前面的结果转换成最终结果。

```kotlin
fun main() {
    val total = listOf(1, 2, 3, 4, 5, 6)
        .filter { it % 2 == 0 }
        .map { it * it }
        .run {
            println("平方后的偶数: $this")
            sum()
        }

    println(total)
}
```

输出：

```text
平方后的偶数: [4, 16, 36]
56
```

这里的 `this` 是前面链式调用得到的集合。

`run` 的最后一行 `sum()` 返回 `Int`，所以 `total` 是 `Int`。

### 实战 Demo：订单金额计算

```kotlin
data class OrderItem(
    val sku: String,
    val quantity: Int,
    val price: Double
)

data class Order(
    val id: String,
    val items: List<OrderItem>,
    val couponAmount: Double = 0.0
)

fun calculatePayAmount(order: Order): Double {
    return order.run {
        val total = items.sumOf { it.quantity * it.price }
        val afterCoupon = total - couponAmount

        if (afterCoupon < 0) {
            0.0
        } else {
            afterCoupon
        }
    }
}

fun main() {
    val order = Order(
        id = "A001",
        items = listOf(
            OrderItem("BOOK", 2, 59.0),
            OrderItem("PEN", 3, 5.0)
        ),
        couponAmount = 20.0
    )

    println(calculatePayAmount(order))
}
```

输出：

```text
113.0
```

`run` 把订单相关计算放在 `Order` 对象作用域里，最后返回应付金额。

### 实战 Demo：生成用户展示信息

```kotlin
data class UserProfile(
    val id: Long,
    val nickname: String?,
    val email: String?,
    val vipLevel: Int
)

fun buildDisplayName(profile: UserProfile): String {
    return profile.run {
        val name = nickname?.takeIf { it.isNotBlank() }
            ?: email?.substringBefore("@")
            ?: "用户$id"

        val vipText = if (vipLevel > 0) "VIP$vipLevel" else "普通用户"

        "$name ($vipText)"
    }
}

fun main() {
    val profile = UserProfile(
        id = 1001,
        nickname = null,
        email = "tom@example.com",
        vipLevel = 2
    )

    println(buildDisplayName(profile))
}
```

输出：

```text
tom (VIP2)
```

这里需要读取 `profile` 的多个字段，并返回一个新字符串，适合使用 `run`。

### 实战 Demo：配置后返回连接地址

```kotlin
data class DbConfig(
    var host: String = "",
    var port: Int = 3306,
    var database: String = "",
    var username: String = ""
)

fun main() {
    val jdbcUrl = DbConfig()
        .apply {
            host = "127.0.0.1"
            port = 3306
            database = "demo"
            username = "root"
        }
        .run {
            "jdbc:mysql://$host:$port/$database?user=$username"
        }

    println(jdbcUrl)
}
```

输出：

```text
jdbc:mysql://127.0.0.1:3306/demo?user=root
```

`apply` 返回配置好的对象，`run` 返回基于对象生成的字符串。

### 实战 Demo：读取配置并转换

```kotlin
fun readConfig(key: String): String? {
    val configs = mapOf(
        "server.port" to "8080",
        "server.timeout" to " 30 ",
        "feature.debug" to "true"
    )

    return configs[key]
}

data class ServerOptions(
    val port: Int,
    val timeoutSeconds: Int,
    val debug: Boolean
)

fun loadOptions(): ServerOptions {
    return run {
        val port = readConfig("server.port")?.toIntOrNull() ?: 80
        val timeout = readConfig("server.timeout")?.trim()?.toIntOrNull() ?: 10
        val debug = readConfig("feature.debug")?.toBooleanStrictOrNull() ?: false

        ServerOptions(
            port = port,
            timeoutSeconds = timeout,
            debug = debug
        )
    }
}

fun main() {
    println(loadOptions())
}
```

输出：

```text
ServerOptions(port=8080, timeoutSeconds=30, debug=true)
```

顶层 `run` 让 `port`、`timeout`、`debug` 这些临时变量只存在于配置加载代码块里。

### 实战 Demo：报表统计

```kotlin
data class PayOrder(
    val id: String,
    val city: String,
    val amount: Double,
    val paid: Boolean
)

data class CityReport(
    val city: String,
    val orderCount: Int,
    val totalAmount: Double
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
* 生成城市报表
* 返回报表摘要字符串

代码：

```kotlin
fun buildSummary(orders: List<PayOrder>): String {
    return orders
        .filter { it.paid }
        .groupBy { it.city }
        .map { (city, cityOrders) ->
            CityReport(
                city = city,
                orderCount = cityOrders.size,
                totalAmount = cityOrders.sumOf { it.amount }
            )
        }
        .sortedByDescending { it.totalAmount }
        .run {
            val totalOrders = sumOf { it.orderCount }
            val totalAmount = sumOf { it.totalAmount }
            val cityNames = joinToString { it.city }

            "城市=$cityNames, 订单数=$totalOrders, 总金额=$totalAmount"
        }
}

fun main() {
    println(buildSummary(orders))
}
```

输出：

```text
城市=上海, 深圳, 北京, 订单数=4, 总金额=860.0
```

`run` 接收前面链式处理后的报表列表，并把它转换成摘要字符串。

### 自定义 runIfNotNull

可空对象经常使用 `?.run { }`，也可以封装成扩展函数。

```kotlin
inline fun <T, R> T?.runIfNotNull(block: T.() -> R): R? {
    return this?.run(block)
}

data class Account(
    val id: Long,
    val email: String?
)

fun main() {
    val account: Account? = Account(1, "tom@example.com")

    val domain = account.runIfNotNull {
        email?.substringAfter("@")
    }

    println(domain)
}
```

输出：

```text
example.com
```

`runIfNotNull` 的返回值是 `R?`，对象为空时返回 `null`。

### run、let、apply、also 的区别

作用域函数主要看两个问题：

```text
Lambda 里怎么访问对象
函数返回什么
```

| 函数 | 上下文对象 | 返回值 | 常见用途 |
| --- | --- | --- | --- |
| `run` | `this` | Lambda 结果 | 在对象作用域里计算结果 |
| `let` | `it` | Lambda 结果 | 空安全、链式转换、显式参数 |
| `apply` | `this` | 对象本身 | 初始化、配置对象 |
| `also` | `it` | 对象本身 | 日志、调试、校验、附加动作 |
| `with` | `this` | Lambda 结果 | 对已有对象集中调用 |

简单选择方式：

```text
需要在对象作用域里返回一个结果：run
需要用显式参数名做转换：let
需要初始化或配置对象：apply
需要附加动作并继续返回原对象：also
```

### run 和 with 的关系

`run` 和 `with` 都使用 `this`，都返回 Lambda 结果。

区别是调用方式：

```kotlin
val result1 = user.run {
    "$name-$age"
}
```

```kotlin
val result2 = with(user) {
    "$name-$age"
}
```

`run` 是扩展函数，可以配合安全调用：

```kotlin
val result = userOrNull?.run {
    "$name-$age"
}
```

`with` 是普通函数，不适合直接写成 `userOrNull?.with { }` 这种形式。

### 常见注意点

#### run 返回最后一行，不返回对象本身

```kotlin
fun main() {
    val result = User("Tom", 18).run {
        age + 1
    }

    println(result)
}
```

输出：

```text
19
```

如果需要返回对象本身，使用 `apply`。

```kotlin
val user = User("Tom", 18).apply {
    println(name)
}
```

#### 只做日志时 also 更贴近语义

```kotlin
val result = listOf(1, 2, 3)
    .also { println(it) }
    .sum()
```

如果只是插入日志，并且要继续返回原对象，`also` 更合适。

#### 顶层 run 不宜包住整段函数

```kotlin
fun createUser(): User {
    return run {
        val name = "Tom"
        val age = 18
        User(name, age)
    }
}
```

这段代码能运行。

如果整个函数只有这一段逻辑，直接写通常更简单：

```kotlin
fun createUser(): User {
    val name = "Tom"
    val age = 18
    return User(name, age)
}
```

顶层 `run` 更适合收拢局部临时变量，或者让某段逻辑作为表达式参与赋值。

#### 嵌套 run 需要留意 this

```kotlin
outer.run {
    inner.run {
        // this 是 inner
    }
}
```

嵌套层级增加后，可以使用标签或者提取函数减少歧义。

```kotlin
outer.run outerBlock@{
    inner.run {
        println(this@outerBlock)
        println(this)
    }
}
```

### 总结

`run` 的核心可以记成几句话：

* `run` 有两种形式：`obj.run { }` 和 `run { }`
* `obj.run { }` 里通过 `this` 访问对象
* `run` 返回 Lambda 最后一行结果
* `?.run { }` 适合可空对象非空时计算结果
* 顶层 `run { }` 适合创建临时作用域
* 需要返回对象本身时，使用 `apply`
* 只是插入日志或调试时，使用 `also`

`run` 的定位是：把一段对象相关逻辑收进作用域里，然后把计算结果作为表达式带出来。
