### 简介

`with` 是 Kotlin 标准库里的作用域函数。

作用域函数常见有 5 个：

* `let`
* `run`
* `with`
* `apply`
* `also`

`with` 的定位很清楚：

> 把一个已有对象传进去，在这个对象的作用域里执行一段逻辑，然后返回 Lambda 最后一行的结果。

常见写法：

```kotlin
val result = with(user) {
    "$name-$age"
}
```

这里的 `name`、`age` 来自 `user` 对象，`result` 是 Lambda 最后一行的字符串。

`with` 适合这类场景：

* 对一个已经存在的对象连续读取多个属性
* 对一个已经存在的对象连续调用多个方法
* 在对象作用域里计算并返回结果
* 生成摘要、报表、格式化文本
* 测试代码中集中操作一个对象

### with 的源码结构

Kotlin 标准库里，`with` 大致可以理解成这样：

```kotlin
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    return receiver.block()
}
```

这段代码有几个关键点：

* `receiver: T`：传入的目标对象
* `block: T.() -> R`：带接收者的 Lambda
* `receiver.block()`：在目标对象作用域里执行 Lambda
* `R`：Lambda 的返回值类型

所以 `with` 的重点是两个：

```text
Lambda 里用 this 访问对象
with 返回 Lambda 最后一行结果
```

### with 不是扩展函数

`with` 和 `run` 很像，但调用方式不同。

`run` 是扩展函数：

```kotlin
val result = user.run {
    "$name-$age"
}
```

`with` 是普通函数：

```kotlin
val result = with(user) {
    "$name-$age"
}
```

区别可以概括成：

```text
run：对象.run { }
with：with(对象) { }
```

因为 `with` 不是扩展函数，所以不能写成：

```kotlin
// 错误写法
// user.with { }
```

也不能写成：

```kotlin
// 错误写法
// user?.with { }
```

可空对象更常用 `?.run { }`，后面会单独说明。

### 基础用法

先定义一个简单的数据类：

```kotlin
data class User(
    val name: String,
    val age: Int,
    val city: String
)
```

普通写法：

```kotlin
fun main() {
    val user = User("Tom", 18, "上海")

    val info = "${user.name}-${user.age}-${user.city}"

    println(info)
}
```

使用 `with`：

```kotlin
fun main() {
    val user = User("Tom", 18, "上海")

    val info = with(user) {
        "$name-$age-$city"
    }

    println(info)
}
```

输出：

```text
Tom-18-上海
```

在 `with(user) { }` 里面，`this` 就是 `user`。

下面两种写法等价：

```kotlin
val info = with(user) {
    "$name-$age-$city"
}
```

```kotlin
val info = with(user) {
    "${this.name}-${this.age}-${this.city}"
}
```

`this` 通常可以省略。

### with 的返回值

`with` 返回 Lambda 最后一行结果。

```kotlin
fun main() {
    val user = User("Tom", 18, "上海")

    val nextAge = with(user) {
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
    val user = User("Tom", 18, "上海")

    val result = with(user) {
        val adult = age >= 18
        "name=$name, city=$city, adult=$adult"
    }

    println(result)
}
```

输出：

```text
name=Tom, city=上海, adult=true
```

最后一行：

```kotlin
"name=$name, city=$city, adult=$adult"
```

就是整个 `with` 的返回值。

### with 和 run 的关系

`with` 可以理解成“普通函数版本的 `run`”。

两者都使用 `this`，都返回 Lambda 结果。

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

输出结果一样。

主要区别：

| 函数 | 调用方式 | 是否扩展函数 | 可空对象常见写法 |
| --- | --- | --- | --- |
| `run` | `obj.run { }` | 是 | `obj?.run { }` |
| `with` | `with(obj) { }` | 否 | 先判空，或改用 `?.run { }` |

当对象已经明确非空，并且需要集中访问它的成员时，`with` 表达很直接。

对象可能为空时，`?.run { }` 通常更顺手。

### 可空对象和 with

`with` 不是扩展函数，所以没有 `obj?.with { }` 这种写法。

可空对象可以先判空：

```kotlin
fun printUser(user: User?) {
    if (user != null) {
        with(user) {
            println(name)
            println(age)
        }
    }
}
```

也可以使用 `?.run { }`：

```kotlin
fun printUser(user: User?) {
    user?.run {
        println(name)
        println(age)
    }
}
```

`with(user)` 本身可以接收可空对象，但 Lambda 内的接收者类型也会是可空类型。

```kotlin
fun printNullableUser(user: User?) {
    with(user) {
        println(this?.name)
    }
}
```

这段代码能运行，但多数情况下不如先判空或使用 `?.run { }` 清楚。

### 和 apply 的区别

`with` 和 `apply` 都使用 `this`。

区别在返回值和调用意图：

| 函数 | 调用方式 | 返回值 | 常见用途 |
| --- | --- | --- | --- |
| `with` | `with(obj) { }` | Lambda 结果 | 集中操作已有对象并返回结果 |
| `apply` | `obj.apply { }` | 对象本身 | 初始化、配置对象 |

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

    val address = with(config) {
        "$host:$port"
    }

    println(address)
}
```

输出：

```text
127.0.0.1:8080
```

`apply` 返回配置好的对象，`with` 返回基于这个对象计算出来的字符串。

### 和 let 的区别

`with` 和 `let` 都能返回 Lambda 结果。

区别在于访问对象的方式。

| 函数 | 上下文对象 | 返回值 | 常见用途 |
| --- | --- | --- | --- |
| `with` | `this` | Lambda 结果 | 集中访问对象成员 |
| `let` | `it` | Lambda 结果 | 空安全、链式转换、显式参数 |

示例：

```kotlin
val text1 = with(user) {
    "$name-$age-$city"
}

val text2 = user.let {
    "${it.name}-${it.age}-${it.city}"
}
```

字段很多时，`with` 可以少写 `it`。

需要明确参数名时，`let` 更合适：

```kotlin
val text = user.let { currentUser ->
    "${currentUser.name}-${currentUser.age}"
}
```

### 实战 Demo：生成用户摘要

```kotlin
data class Order(
    val id: String,
    val amount: Double
)

data class UserSummary(
    val id: Long,
    val name: String,
    val city: String,
    val orders: List<Order>
)

fun buildUserSummary(user: UserSummary): String {
    return with(user) {
        val totalAmount = orders.sumOf { it.amount }
        val orderCount = orders.size

        "用户=$name, 城市=$city, 订单数=$orderCount, 总金额=$totalAmount"
    }
}

fun main() {
    val user = UserSummary(
        id = 1001,
        name = "Tom",
        city = "上海",
        orders = listOf(
            Order("A001", 120.0),
            Order("A002", 260.0)
        )
    )

    println(buildUserSummary(user))
}
```

输出：

```text
用户=Tom, 城市=上海, 订单数=2, 总金额=380.0
```

这里需要多次读取 `user` 的字段，并返回一段摘要文本，适合使用 `with`。

### 实战 Demo：矩形面积和周长

```kotlin
data class Rectangle(
    val width: Int,
    val height: Int
)

data class RectangleInfo(
    val area: Int,
    val perimeter: Int
)

fun calculate(rectangle: Rectangle): RectangleInfo {
    return with(rectangle) {
        RectangleInfo(
            area = width * height,
            perimeter = (width + height) * 2
        )
    }
}

fun main() {
    val info = calculate(Rectangle(width = 5, height = 10))
    println(info)
}
```

输出：

```text
RectangleInfo(area=50, perimeter=30)
```

`with(rectangle)` 让 `width`、`height` 可以直接参与计算。

### 实战 Demo：配置已有对象

`with` 也能修改已有对象。

```kotlin
data class TextStyle(
    var text: String = "",
    var size: Int = 14,
    var color: String = "black",
    var bold: Boolean = false
)

fun main() {
    val style = TextStyle()

    with(style) {
        text = "Kotlin with"
        size = 18
        color = "blue"
        bold = true
    }

    println(style)
}
```

输出：

```text
TextStyle(text=Kotlin with, size=18, color=blue, bold=true)
```

如果是创建对象时顺手配置，`apply` 更常见：

```kotlin
val style = TextStyle().apply {
    text = "Kotlin with"
    size = 18
    color = "blue"
    bold = true
}
```

如果对象已经存在，`with(style) { }` 也很自然。

### 实战 Demo：生成订单报表

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
* 返回汇总文字

代码：

```kotlin
fun buildReportSummary(orders: List<PayOrder>): String {
    val reports = orders
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

    return with(reports) {
        val totalCount = sumOf { it.orderCount }
        val totalAmount = sumOf { it.totalAmount }
        val cityNames = joinToString { it.city }

        "城市=$cityNames, 订单数=$totalCount, 总金额=$totalAmount"
    }
}

fun main() {
    println(buildReportSummary(orders))
}
```

输出：

```text
城市=上海, 深圳, 北京, 订单数=4, 总金额=860.0
```

`reports` 已经是一个确定存在的列表，后面需要围绕它计算多个值，`with(reports)` 能把这段汇总逻辑收在一起。

### 实战 Demo：用 StringBuilder 生成文本

```kotlin
data class Article(
    val title: String,
    val author: String,
    val tags: List<String>
)

fun buildMarkdown(article: Article): String {
    return with(StringBuilder()) {
        appendLine("# ${article.title}")
        appendLine()
        appendLine("作者：${article.author}")
        appendLine()
        appendLine("标签：${article.tags.joinToString()}")
        toString()
    }
}

fun main() {
    val article = Article(
        title = "Kotlin with",
        author = "Tom",
        tags = listOf("Kotlin", "Scope Function")
    )

    println(buildMarkdown(article))
}
```

输出：

```text
# Kotlin with

作者：Tom

标签：Kotlin, Scope Function
```

这里的接收者是 `StringBuilder`，在 `with` 代码块里可以连续调用 `appendLine`。

最后一行 `toString()` 是返回值。

### 实战 Demo：测试代码里的集中操作

下面示例展示测试代码里常见的风格。

```kotlin
class ShoppingCart {
    private val items = mutableListOf<String>()

    fun add(item: String) {
        items.add(item)
    }

    fun remove(item: String) {
        items.remove(item)
    }

    fun size(): Int {
        return items.size
    }

    fun contains(item: String): Boolean {
        return items.contains(item)
    }
}

fun main() {
    val cart = ShoppingCart()

    val result = with(cart) {
        add("Book")
        add("Pen")
        remove("Pen")

        contains("Book") && size() == 1
    }

    println(result)
}
```

输出：

```text
true
```

`with(cart)` 把多个购物车操作放在同一个对象作用域里，最后返回断言结果。

### 嵌套 with 和 this 指向

嵌套 `with` 时，`this` 会指向最近一层接收者。

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
    val customer = Customer()

    with(customer) {
        name = "Tom"

        with(address) {
            city = "上海"
            street = "人民路"
        }
    }

    println(customer.name)
    println(customer.address.city)
}
```

外层 `with(customer)` 里，`this` 是 `Customer`。

内层 `with(address)` 里，`this` 是 `Address`。

如果需要在内层访问外层对象，可以使用标签：

```kotlin
fun main() {
    val customer = Customer()

    with(customer) customerBlock@{
        name = "Tom"

        with(address) {
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

### with、run、apply、let、also 的区别

作用域函数主要看两个问题：

```text
Lambda 里怎么访问对象
函数返回什么
```

| 函数 | 上下文对象 | 返回值 | 调用方式 | 常见用途 |
| --- | --- | --- | --- | --- |
| `with` | `this` | Lambda 结果 | `with(obj) { }` | 已有对象集中操作 |
| `run` | `this` | Lambda 结果 | `obj.run { }` | 对象作用域里计算结果 |
| `let` | `it` | Lambda 结果 | `obj.let { }` | 空安全、链式转换 |
| `apply` | `this` | 对象本身 | `obj.apply { }` | 初始化、配置对象 |
| `also` | `it` | 对象本身 | `obj.also { }` | 日志、调试、附加动作 |

简单选择方式：

```text
已有非空对象，集中访问成员并返回结果：with
对象可能为空，需要安全调用：run
需要初始化或配置对象并返回对象本身：apply
需要用显式参数名做转换：let
需要插入日志或附加动作：also
```

### 常见注意点

#### with 返回最后一行，不返回对象本身

```kotlin
fun main() {
    val user = User("Tom", 18, "上海")

    val result = with(user) {
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
val user = User("Tom", 18, "上海").apply {
    println(name)
}
```

#### with 不支持 obj?.with 形式

```kotlin
val user: User? = null

// 错误写法
// user?.with {
//     println(name)
// }
```

可空对象更常见的写法是：

```kotlin
user?.run {
    println(name)
}
```

#### with 适合已有对象，不适合强行包裹所有逻辑

```kotlin
val result = with(user) {
    "$name-$age"
}
```

这种写法很自然。

如果代码块和某个对象关系不强，顶层 `run { }` 可能更清楚：

```kotlin
val result = run {
    val a = 10
    val b = 20
    a + b
}
```

#### 嵌套 with 需要留意 this

```kotlin
with(customer) {
    with(address) {
        // this 是 address
    }
}
```

嵌套层级增加后，可以使用标签，或者提取函数减少歧义。

### 总结

`with` 的核心可以记成几句话：

* `with` 是普通函数，不是扩展函数
* 调用方式是 `with(obj) { }`
* Lambda 内部通过 `this` 访问传入对象
* `with` 返回 Lambda 最后一行结果
* `with` 适合对已有非空对象集中操作
* 可空对象通常使用 `?.run { }`
* 需要返回对象本身时，使用 `apply`

`with` 的定位是：把一个已经存在的对象放进作用域里，围绕它完成一段集中操作，然后把结果返回出来。
