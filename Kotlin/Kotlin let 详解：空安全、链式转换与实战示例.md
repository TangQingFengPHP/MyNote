### 简介

`let` 是 Kotlin 标准库里的作用域函数。

作用域函数常见有 5 个：

* `let`
* `run`
* `with`
* `apply`
* `also`

`let` 的使用频率很高，尤其是在处理可空对象、临时变量、链式转换时。

一句话概括：

> `let` 会把当前对象传进 Lambda，Lambda 的最后一行作为整个 `let` 的返回值。

常见写法：

```kotlin
val result = value.let {
    // it 表示 value
    // 最后一行就是 result
}
```

如果配合安全调用符 `?.`：

```kotlin
nullableValue?.let {
    // 只有 nullableValue 不为 null 时才会执行
}
```

这也是 `let` 在实际项目中非常常见的写法。

### let 的源码结构

Kotlin 标准库里，`let` 大致可以理解成这样：

```kotlin
public inline fun <T, R> T.let(block: (T) -> R): R {
    return block(this)
}
```

这段代码有几个关键点：

* `T`：调用 `let` 的对象类型
* `R`：Lambda 的返回值类型
* `block: (T) -> R`：接收一个参数为 `T`、返回值为 `R` 的 Lambda
* `block(this)`：把当前对象传给 Lambda

所以：

```kotlin
val length = "Kotlin".let {
    it.length
}
```

可以理解成：

```kotlin
val value = "Kotlin"
val length = value.length
```

只是 `let` 把这个操作放进了一个独立的 Lambda 作用域。

### 基础用法

```kotlin
fun main() {
    val name = "Kotlin"

    name.let {
        println(it)
        println(it.length)
    }
}
```

输出：

```text
Kotlin
6
```

这里的 `it` 就是调用 `let` 的对象，也就是 `name`。

也可以显式命名参数：

```kotlin
fun main() {
    val name = "Kotlin"

    name.let { value ->
        println(value)
        println(value.length)
    }
}
```

逻辑较短时，用 `it` 足够清楚。逻辑较长时，命名参数通常更容易阅读。

### let 的返回值

`let` 返回的不是调用对象本身，而是 Lambda 最后一行的结果。

```kotlin
fun main() {
    val result = "Kotlin".let {
        it.length
    }

    println(result)
}
```

输出：

```text
6
```

`result` 的值是 `it.length`，不是 `"Kotlin"`。

再看一个多行 Lambda：

```kotlin
fun main() {
    val result = " kotlin ".let {
        val trimmed = it.trim()
        val upper = trimmed.uppercase()
        "处理结果: $upper"
    }

    println(result)
}
```

输出：

```text
处理结果: KOTLIN
```

最后一行：

```kotlin
"处理结果: $upper"
```

就是整个 `let` 的返回值。

### 空安全处理

`let` 常和 `?.` 一起使用。

普通判空写法：

```kotlin
fun printName(name: String?) {
    if (name != null) {
        println(name.length)
        println(name.uppercase())
    }
}
```

使用 `let`：

```kotlin
fun printName(name: String?) {
    name?.let {
        println(it.length)
        println(it.uppercase())
    }
}
```

含义是：

```text
name 不为 null 时，执行 let 里的代码
name 为 null 时，整个 let 不执行
```

示例：

```kotlin
fun main() {
    val name1: String? = "Kotlin"
    val name2: String? = null

    name1?.let {
        println("name1 length = ${it.length}")
    }

    name2?.let {
        println("name2 length = ${it.length}")
    }
}
```

输出：

```text
name1 length = 6
```

`name2` 是 `null`，所以第二个 `let` 不会执行。

### 配合 Elvis 操作符

`?.let` 经常和 Elvis 操作符 `?:` 一起用。

```kotlin
fun getNameLength(name: String?): Int {
    return name?.let {
        it.length
    } ?: 0
}

fun main() {
    println(getNameLength("Kotlin"))
    println(getNameLength(null))
}
```

输出：

```text
6
0
```

含义是：

```text
name 非空 -> 返回 name.length
name 为空 -> 返回 0
```

再看一个返回字符串的例子：

```kotlin
fun formatUsername(username: String?): String {
    return username?.let {
        "用户名: ${it.trim()}"
    } ?: "用户名为空"
}

fun main() {
    println(formatUsername("  tom  "))
    println(formatUsername(null))
}
```

输出：

```text
用户名: tom
用户名为空
```

### 作用域隔离

`let` 会创建一个小作用域。里面定义的变量不会影响外面。

```kotlin
fun main() {
    val result = "kotlin".let {
        val upper = it.uppercase()
        val message = "结果: $upper"
        message
    }

    println(result)
}
```

`upper` 和 `message` 只在 `let` 的 Lambda 内有效。

这种写法适合处理一些临时变量，避免外层作用域堆太多中间变量。

### 链式转换

`let` 的返回值是 Lambda 结果，所以适合放在链式调用中做转换。

```kotlin
fun main() {
    val result = "  kotlin let  "
        .trim()
        .uppercase()
        .let {
            "Result: $it"
        }

    println(result)
}
```

输出：

```text
Result: KOTLIN LET
```

也可以连续使用 `let`：

```kotlin
fun main() {
    val result = "  123  "
        .let { it.trim() }
        .let { it.toIntOrNull() }
        .let { number -> number?.times(2) }
        .let { "计算结果: $it" }

    println(result)
}
```

输出：

```text
计算结果: 246
```

连续 `let` 能表达一条处理管道，不过链条过长时可读性会下降。业务规则复杂时，拆成有名字的函数更合适。

### 配合 takeIf

`takeIf` 的作用是：满足条件时返回对象本身，不满足时返回 `null`。

```kotlin
fun main() {
    val age = 20

    val result = age
        .takeIf { it >= 18 }
        ?.let { "年龄 $it，允许访问" }
        ?: "年龄不足，拒绝访问"

    println(result)
}
```

输出：

```text
年龄 20，允许访问
```

如果年龄改成 16：

```kotlin
val age = 16
```

输出：

```text
年龄不足，拒绝访问
```

这类写法适合“先判断条件，再处理对象”的场景。

### 配合 mapNotNull

集合里有可空元素时，`let` 可以配合 `mapNotNull` 做过滤和转换。

```kotlin
fun main() {
    val names = listOf("Tom", null, "Jerry", "", "Spike")

    val result = names.mapNotNull { name ->
        name?.let {
            if (it.isBlank()) {
                null
            } else {
                it.uppercase()
            }
        }
    }

    println(result)
}
```

输出：

```text
[TOM, JERRY, SPIKE]
```

这段代码做了两件事：

* `null` 被过滤掉
* 空字符串返回 `null`，也会被 `mapNotNull` 过滤掉

### 和普通 if 判空的关系

`let` 不是用来替代所有 `if` 的。

下面这种写法很清楚：

```kotlin
fun printUser(user: User?) {
    if (user == null) {
        println("用户为空")
        return
    }

    println(user.name)
    println(user.age)
}
```

如果空值逻辑需要提前返回，普通 `if` 反而更直接。

`?.let` 更适合这种情况：

```kotlin
fun printUser(user: User?) {
    user?.let {
        println(it.name)
        println(it.age)
    }
}
```

也就是：对象非空时做一小段处理，对象为空时不需要特别处理。

### 实战 Demo：安全处理接口返回值

模拟一个接口，可能返回用户，也可能返回 `null`。

```kotlin
data class User(
    val id: Int,
    val name: String,
    val email: String?
)

fun fetchUser(id: Int): User? {
    return if (id == 1) {
        User(1, "Tom", "tom@example.com")
    } else {
        null
    }
}

fun saveUser(user: User) {
    println("保存用户: ${user.name}")
}

fun sendEmail(email: String) {
    println("发送邮件到: $email")
}

fun main() {
    fetchUser(1)?.let { user ->
        saveUser(user)

        user.email?.let { email ->
            sendEmail(email)
        }
    } ?: println("用户不存在")
}
```

输出：

```text
保存用户: Tom
发送邮件到: tom@example.com
```

这里有两层空安全：

* `fetchUser(1)?.let`：用户存在时才执行
* `user.email?.let`：邮箱存在时才发送邮件

如果嵌套层级继续增加，需要考虑拆函数，避免代码缩进过深。

### 实战 Demo：表单参数清洗

表单字段经常需要去空格、判空、转换格式。

```kotlin
data class RegisterRequest(
    val username: String?,
    val email: String?,
    val age: String?
)

data class RegisterCommand(
    val username: String,
    val email: String,
    val age: Int
)

fun buildCommand(request: RegisterRequest): RegisterCommand? {
    val username = request.username
        ?.trim()
        ?.takeIf { it.isNotEmpty() }

    val email = request.email
        ?.trim()
        ?.lowercase()
        ?.takeIf { it.contains("@") }

    val age = request.age
        ?.trim()
        ?.toIntOrNull()
        ?.takeIf { it >= 18 }

    return username?.let { validUsername ->
        email?.let { validEmail ->
            age?.let { validAge ->
                RegisterCommand(
                    username = validUsername,
                    email = validEmail,
                    age = validAge
                )
            }
        }
    }
}

fun main() {
    val request = RegisterRequest(
        username = "  tom  ",
        email = " TOM@EXAMPLE.COM ",
        age = "20"
    )

    val command = buildCommand(request)
    println(command)
}
```

输出：

```text
RegisterCommand(username=tom, email=tom@example.com, age=20)
```

这段代码的流程：

* `username`：去空格，非空才保留
* `email`：去空格，转小写，包含 `@` 才保留
* `age`：转数字，大于等于 18 才保留
* 三个字段都有效时，才创建 `RegisterCommand`

如果字段更多，使用专门的校验器会更清晰。

### 实战 Demo：订单统计

准备订单数据：

```kotlin
data class Order(
    val id: String,
    val city: String?,
    val amount: Double,
    val paid: Boolean
)

data class CityReport(
    val city: String,
    val count: Int,
    val totalAmount: Double
)

val orders = listOf(
    Order("A001", "上海", 120.0, true),
    Order("A002", null, 80.0, true),
    Order("A003", "北京", 300.0, false),
    Order("A004", "上海", 260.0, true),
    Order("A005", "深圳", 180.0, true)
)
```

需求：

* 只统计已支付订单
* 城市为空的订单不统计
* 按城市分组
* 统计订单数和总金额
* 按总金额倒序

代码：

```kotlin
fun buildCityReport(orders: List<Order>): List<CityReport> {
    return orders
        .filter { it.paid }
        .mapNotNull { order ->
            order.city?.let { city ->
                order.copy(city = city)
            }
        }
        .groupBy { it.city!! }
        .map { (city, cityOrders) ->
            CityReport(
                city = city,
                count = cityOrders.size,
                totalAmount = cityOrders.sumOf { it.amount }
            )
        }
        .sortedByDescending { it.totalAmount }
}

fun main() {
    val reports = buildCityReport(orders)

    reports.forEach {
        println("${it.city} 订单数=${it.count}, 金额=${it.totalAmount}")
    }
}
```

输出：

```text
上海 订单数=2, 金额=380.0
深圳 订单数=1, 金额=180.0
```

这段代码里的 `let` 用来处理可空城市：

```kotlin
order.city?.let { city ->
    order.copy(city = city)
}
```

城市存在时返回订单，城市为空时返回 `null`，再由 `mapNotNull` 过滤掉。

上面的 `groupBy { it.city!! }` 能工作，但类型表达不够明确。可以把模型拆开，让后续流程里城市变成非空类型：

```kotlin
data class PaidOrder(
    val id: String,
    val city: String,
    val amount: Double
)

fun buildCityReportBetter(orders: List<Order>): List<CityReport> {
    return orders
        .filter { it.paid }
        .mapNotNull { order ->
            order.city?.let { city ->
                PaidOrder(
                    id = order.id,
                    city = city,
                    amount = order.amount
                )
            }
        }
        .groupBy { it.city }
        .map { (city, cityOrders) ->
            CityReport(
                city = city,
                count = cityOrders.size,
                totalAmount = cityOrders.sumOf { it.amount }
            )
        }
        .sortedByDescending { it.totalAmount }
}
```

经过 `PaidOrder` 转换后，`city` 已经是非空字符串，后面不需要再写 `!!`。

### 实战 Demo：读取配置并转换

很多配置来自字符串，需要先读取、再转换、再兜底。

```kotlin
fun readConfig(key: String): String? {
    val configs = mapOf(
        "server.port" to "8080",
        "server.timeout" to " 30 "
    )

    return configs[key]
}

fun main() {
    val port = readConfig("server.port")
        ?.let { it.toIntOrNull() }
        ?: 80

    val timeout = readConfig("server.timeout")
        ?.let { it.trim() }
        ?.let { it.toIntOrNull() }
        ?: 10

    println("port=$port")
    println("timeout=$timeout")
}
```

输出：

```text
port=8080
timeout=30
```

`let` 在这里负责把字符串转换成目标类型。

### let、also、apply、run 的区别

作用域函数容易混淆的地方是：Lambda 里用 `it` 还是 `this`，返回对象本身还是返回 Lambda 结果。

| 函数 | 上下文对象 | 返回值 | 常见用途 |
| --- | --- | --- | --- |
| `let` | `it` | Lambda 结果 | 空安全、转换结果 |
| `also` | `it` | 对象本身 | 日志、调试、附加动作 |
| `apply` | `this` | 对象本身 | 对象初始化、属性赋值 |
| `run` | `this` | Lambda 结果 | 对象内部计算并返回结果 |
| `with` | `this` | Lambda 结果 | 对已有对象集中调用 |

#### let 返回 Lambda 结果

```kotlin
val length = "Kotlin".let {
    it.length
}

println(length)
```

输出：

```text
6
```

#### also 返回对象本身

```kotlin
val value = "Kotlin".also {
    println(it.length)
}

println(value)
```

输出：

```text
6
Kotlin
```

#### apply 返回对象本身

```kotlin
data class Config(
    var host: String = "",
    var port: Int = 0
)

val config = Config().apply {
    host = "127.0.0.1"
    port = 8080
}

println(config)
```

输出：

```text
Config(host=127.0.0.1, port=8080)
```

#### run 返回 Lambda 结果

```kotlin
val length = "Kotlin".run {
    length
}

println(length)
```

输出：

```text
6
```

简单选择方式：

```text
需要转换成另一个结果：let
需要顺手打印日志并继续返回原对象：also
需要初始化对象属性：apply
需要在对象内部计算一个结果：run
```

### 常见注意点

#### let 不会修改原对象引用

```kotlin
fun main() {
    val result = "kotlin".let {
        it.uppercase()
    }

    println(result)
}
```

输出：

```text
KOTLIN
```

`uppercase()` 返回了一个新字符串，原字符串本身没有被修改。

如果对象是可变对象，在 `let` 里修改属性，修改的是对象内部状态：

```kotlin
data class User(var name: String)

fun main() {
    val user = User("tom")

    user.let {
        it.name = it.name.uppercase()
    }

    println(user)
}
```

输出：

```text
User(name=TOM)
```

这里修改的是 `user` 对象的属性，不是把 `user` 变量重新赋值。

#### let 嵌套过深会影响阅读

```kotlin
user?.let { u ->
    u.address?.let { address ->
        address.city?.let { city ->
            println(city)
        }
    }
}
```

可以改成安全调用链：

```kotlin
val city = user?.address?.city

city?.let {
    println(it)
}
```

也可以提前返回：

```kotlin
fun printCity(user: User?) {
    val city = user?.address?.city ?: return
    println(city)
}
```

#### it 过多时改成命名参数

```kotlin
user?.let {
    println(it.name)
    println(it.email)
}
```

逻辑简单时没问题。

业务代码变长时，命名参数更清楚：

```kotlin
user?.let { userInfo ->
    println(userInfo.name)
    println(userInfo.email)
}
```

#### 不关心返回值时可以考虑 also

`let` 返回 Lambda 结果。

```kotlin
val result = "Kotlin".let {
    println(it)
}
```

这里 `result` 的类型是 `Unit`。

如果只是打印日志，并且还要继续使用原对象，`also` 更合适：

```kotlin
val result = "Kotlin"
    .also { println(it) }
    .uppercase()

println(result)
```

输出：

```text
Kotlin
KOTLIN
```

### 总结

`let` 的核心可以记成几句话：

* `let` 是作用域函数
* Lambda 内部通过 `it` 访问调用对象
* `let` 返回 Lambda 的最后一行
* `?.let { }` 常用于可空对象的非空执行
* `let` 适合做链式转换、临时作用域、结果包装
* 不关心返回值且需要继续返回原对象时，`also` 往往更合适

`let` 的重点不是把所有判空都改成链式写法，而是在合适的地方减少重复判断、收拢临时变量，并让数据转换过程更清楚。
