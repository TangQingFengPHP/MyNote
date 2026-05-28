### 简介

Lambda 表达式是 Kotlin 里非常常见的语法。

集合操作里的 `map`、`filter`、`forEach`，作用域函数里的 `let`、`apply`、`also`，线程、回调、DSL 风格 API，背后都能看到 Lambda。

一句话概括：

> Lambda 就是一段没有名字的函数代码，可以赋值给变量，也可以传给另一个函数执行。

普通函数有名字：

```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}
```

Lambda 没有名字：

```kotlin
{ a: Int, b: Int -> a + b }
```

Lambda 的重点不是“少写几个字”，而是把一段行为当作数据传来传去。

### 基础语法

Lambda 的基本格式：

```kotlin
{ 参数列表 -> 函数体 }
```

示例：

```kotlin
val add = { a: Int, b: Int ->
    a + b
}

fun main() {
    println(add(10, 20))
}
```

输出：

```text
30
```

`->` 左边是参数，右边是执行逻辑。

如果函数体只有一行，可以写成：

```kotlin
val add = { a: Int, b: Int -> a + b }
```

### Lambda 的类型

Lambda 也有类型。

```kotlin
val add: (Int, Int) -> Int = { a, b ->
    a + b
}
```

这里的：

```kotlin
(Int, Int) -> Int
```

表示：

```text
接收两个 Int，返回一个 Int
```

常见函数类型：

```kotlin
() -> Unit
```

无参数、无返回值。

```kotlin
(String) -> Unit
```

接收一个 `String`，无返回值。

```kotlin
(Int) -> String
```

接收一个 `Int`，返回 `String`。

```kotlin
(String, Int) -> Boolean
```

接收 `String` 和 `Int`，返回 `Boolean`。

完整示例：

```kotlin
val printMessage: (String) -> Unit = { message ->
    println(message)
}

val isAdult: (Int) -> Boolean = { age ->
    age >= 18
}

fun main() {
    printMessage("Kotlin Lambda")
    println(isAdult(20))
}
```

输出：

```text
Kotlin Lambda
true
```

### 类型写左边还是右边

Lambda 的参数类型可以写在左边，也可以写在右边。

写在右边：

```kotlin
val add = { a: Int, b: Int ->
    a + b
}
```

写在左边：

```kotlin
val add: (Int, Int) -> Int = { a, b ->
    a + b
}
```

两种写法都可以。

如果左边已经声明了函数类型，右边参数就可以省略类型。

```kotlin
val square: (Int) -> Int = { number ->
    number * number
}
```

如果左边没有声明类型，右边参数通常要写清楚类型。

```kotlin
val square = { number: Int ->
    number * number
}
```

否则编译器不知道 `number` 是什么类型。

### 最后一行就是返回值

Lambda 里不需要写 `return`。

最后一行表达式就是返回值。

```kotlin
val formatName: (String) -> String = { name ->
    val trimmed = name.trim()
    trimmed.uppercase()
}

fun main() {
    println(formatName(" kotlin "))
}
```

输出：

```text
KOTLIN
```

这里的返回值是：

```kotlin
trimmed.uppercase()
```

如果最后一行是 `println`，返回值就是 `Unit`。

```kotlin
val log: (String) -> Unit = { message ->
    println("日志: $message")
}
```

### it 关键字

当 Lambda 只有一个参数，并且没有显式写参数名时，可以使用 `it`。

```kotlin
val numbers = listOf(1, 2, 3, 4)

val result = numbers.map {
    it * 2
}

println(result)
```

输出：

```text
[2, 4, 6, 8]
```

上面代码等价于：

```kotlin
val result = numbers.map { number ->
    number * 2
}
```

`it` 适合短逻辑：

```kotlin
users.filter { it.age >= 18 }
```

逻辑变复杂时，写出参数名更清楚：

```kotlin
users.filter { user ->
    user.age >= 18 && user.city == "上海" && user.balance > 1000
}
```

### 尾随 Lambda

如果函数的最后一个参数是 Lambda，可以把 Lambda 放到小括号外面。

先定义一个函数：

```kotlin
fun calculate(a: Int, b: Int, operation: (Int, Int) -> Int): Int {
    return operation(a, b)
}
```

普通写法：

```kotlin
val result = calculate(10, 20, { x, y ->
    x + y
})
```

Kotlin 常用写法：

```kotlin
val result = calculate(10, 20) { x, y ->
    x + y
}
```

如果函数只有一个 Lambda 参数，小括号也可以省略。

```kotlin
fun runTask(task: () -> Unit) {
    task()
}

fun main() {
    runTask {
        println("任务执行")
    }
}
```

输出：

```text
任务执行
```

这就是很多 Kotlin API 看起来像 DSL 的原因。

### 忽略不用的参数

Lambda 有多个参数，但某些参数用不到，可以使用 `_` 忽略。

```kotlin
val scores = mapOf(
    "张三" to 90,
    "李四" to 86,
    "王五" to 95
)

scores.forEach { _, score ->
    println(score)
}
```

输出：

```text
90
86
95
```

只关心分数，不关心姓名时，`_` 比随便起一个变量名更清楚。

### Lambda 作为函数参数

Lambda 最常见的用法就是传给高阶函数。

```kotlin
fun checkAge(age: Int, rule: (Int) -> Boolean): Boolean {
    return rule(age)
}

fun main() {
    val adult = checkAge(20) { age ->
        age >= 18
    }

    val child = checkAge(12) {
        it < 18
    }

    println(adult)
    println(child)
}
```

输出：

```text
true
true
```

`checkAge` 不固定判断规则，规则由外部传入。

### Lambda 作为返回值

函数也可以返回一个 Lambda。

```kotlin
fun createDiscount(level: String): (Double) -> Double {
    return when (level) {
        "vip" -> { price -> price * 0.8 }
        "svip" -> { price -> price * 0.6 }
        else -> { price -> price }
    }
}

fun main() {
    val vipDiscount = createDiscount("vip")
    val normalDiscount = createDiscount("normal")

    println(vipDiscount(100.0))
    println(normalDiscount(100.0))
}
```

输出：

```text
80.0
100.0
```

这类写法适合做策略选择。先根据条件选出一段逻辑，再在后面执行。

### 闭包：Lambda 可以记住外部变量

Lambda 可以访问外部变量，这叫闭包。

```kotlin
fun createCounter(): () -> Int {
    var count = 0

    return {
        count++
        count
    }
}

fun main() {
    val counter = createCounter()

    println(counter())
    println(counter())
    println(counter())
}
```

输出：

```text
1
2
3
```

`createCounter` 执行结束后，`count` 并没有丢失。返回出去的 Lambda 仍然能访问它。

再看两个计数器：

```kotlin
fun main() {
    val counterA = createCounter()
    val counterB = createCounter()

    println(counterA())
    println(counterA())
    println(counterB())
}
```

输出：

```text
1
2
1
```

每次调用 `createCounter()`，都会产生一份新的 `count`。

### Lambda 和匿名函数

Lambda：

```kotlin
val add = { a: Int, b: Int ->
    a + b
}
```

匿名函数：

```kotlin
val add = fun(a: Int, b: Int): Int {
    return a + b
}
```

两者都没有函数名，都可以赋值给变量。

区别主要在返回值写法：

* Lambda 最后一行自动返回
* 匿名函数使用 `return`
* 匿名函数的参数和返回类型写得更像普通函数

匿名函数适合返回逻辑比较复杂、需要显式 `return` 的场景。

### return 和标签返回

Lambda 里的 `return` 容易让代码产生误解。

先看一个常见写法：

```kotlin
fun main() {
    val result = listOf(1, 2, 3, 4).map {
        if (it == 2) {
            return@map 0
        }
        it * 10
    }

    println(result)
}
```

输出：

```text
[10, 0, 30, 40]
```

`return@map 0` 的意思是：只从当前 `map` 的 Lambda 返回一个值，不是结束 `main`。

这里的 `0` 是当前元素转换后的结果。

`map` 会把每个元素转换成一个新值，所以 `return@map` 后面要跟一个返回值。

执行过程可以理解成：

```text
1 -> 10
2 -> 0
3 -> 30
4 -> 40
```

所以最终结果是：

```text
[10, 0, 30, 40]
```

`forEach` 里也经常这样写：

```kotlin
fun main() {
    listOf("A", "B", "C").forEach {
        if (it == "B") {
            return@forEach
        }
        println(it)
    }
}
```

输出：

```text
A
C
```

`return@forEach` 类似循环里的 `continue`，跳过当前这次 Lambda。

`forEach` 只是执行动作，不负责生成新集合，所以 `return@forEach` 后面不用跟返回值。

#### 隐式标签

`return@map`、`return@forEach` 里的 `map`、`forEach` 不是随便起的名字。

当 Lambda 传给某个函数时，Kotlin 会自动生成一个隐式标签。标签名默认就是接收这个 Lambda 的函数名。

```kotlin
val result = listOf(1, 2, 3).map {
    if (it == 2) {
        return@map 0
    }
    it * 10
}
```

这个 Lambda 是传给 `map` 的，所以可以使用 `return@map`。

```kotlin
listOf("A", "B", "C").forEach {
    if (it == "B") {
        return@forEach
    }
    println(it)
}
```

这个 Lambda 是传给 `forEach` 的，所以可以使用 `return@forEach`。

如果写一个不存在的标签，会直接编译失败。

```kotlin
listOf(1, 2, 3).forEach {
    return@abc
}
```

`abc` 没有定义，也不是当前 Lambda 的隐式标签，所以不能这样写。

#### 自定义标签

标签也可以自定义。

写法是在 Lambda 前面加上标签名：

```kotlin
listOf(1, 2, 3).forEach myLoop@{
    if (it == 2) {
        return@myLoop
    }
    println(it)
}
```

输出：

```text
1
3
```

这里的 `myLoop@` 是自定义标签，所以内部可以写 `return@myLoop`。

隐式标签和自定义标签可以这样区分：

```text
return@map      使用 map 的隐式标签
return@forEach  使用 forEach 的隐式标签
return@myLoop   使用 myLoop 自定义标签
```

### Lambda 接收者

Lambda 还可以带接收者。

普通 Lambda 类型：

```kotlin
(String) -> Int
```

带接收者的 Lambda 类型：

```kotlin
String.() -> Int
```

区别在于：带接收者的 Lambda 里面可以直接访问接收者成员。

```kotlin
val lengthGetter: String.() -> Int = {
    length
}

fun main() {
    println("Kotlin".lengthGetter())
}
```

输出：

```text
6
```

这里的 `length` 来自 `String` 对象本身。

`apply` 就是典型的接收者 Lambda。

```kotlin
data class ServerConfig(
    var host: String = "",
    var port: Int = 0,
    var debug: Boolean = false
)

fun main() {
    val config = ServerConfig().apply {
        host = "127.0.0.1"
        port = 8080
        debug = true
    }

    println(config)
}
```

在 `apply` 的 Lambda 里，可以直接写 `host`、`port`、`debug`。

### 集合实战：订单统计

准备数据：

```kotlin
data class Order(
    val id: String,
    val userId: Int,
    val city: String,
    val amount: Double,
    val paid: Boolean
)

val orders = listOf(
    Order("A001", 1, "上海", 120.0, true),
    Order("A002", 2, "北京", 80.0, false),
    Order("A003", 1, "上海", 260.0, true),
    Order("A004", 3, "深圳", 300.0, true),
    Order("A005", 2, "北京", 180.0, true)
)
```

需求：

* 只统计已支付订单
* 按城市分组
* 统计每个城市的订单数和总金额
* 按总金额倒序

代码：

```kotlin
data class CityOrderReport(
    val city: String,
    val count: Int,
    val totalAmount: Double
)

fun buildReport(orders: List<Order>): List<CityOrderReport> {
    return orders
        .filter { it.paid }
        .groupBy { it.city }
        .map { (city, cityOrders) ->
            CityOrderReport(
                city = city,
                count = cityOrders.size,
                totalAmount = cityOrders.sumOf { it.amount }
            )
        }
        .sortedByDescending { it.totalAmount }
}

fun main() {
    val reports = buildReport(orders)

    reports.forEach {
        println("${it.city} 订单数=${it.count}, 金额=${it.totalAmount}")
    }
}
```

输出：

```text
上海 订单数=2, 金额=380.0
深圳 订单数=1, 金额=300.0
北京 订单数=1, 金额=180.0
```

这段代码里每个 Lambda 都只做一件事：

* `filter { it.paid }`：过滤已支付订单
* `groupBy { it.city }`：按城市分组
* `map { ... }`：转换成报表对象
* `sortedByDescending { it.totalAmount }`：按总金额排序

### 实战 Demo：参数校验器

业务参数校验经常会出现一堆 `if`。

可以用 Lambda 把规则抽出来。

```kotlin
data class RegisterForm(
    val username: String,
    val password: String,
    val age: Int
)

class Validator<T>(private val value: T) {
    private val errors = mutableListOf<String>()

    fun rule(message: String, predicate: (T) -> Boolean): Validator<T> {
        if (!predicate(value)) {
            errors.add(message)
        }
        return this
    }

    fun result(): List<String> {
        return errors
    }
}

fun <T> validate(value: T, block: Validator<T>.() -> Unit): List<String> {
    val validator = Validator(value)
    validator.block()
    return validator.result()
}

fun main() {
    val form = RegisterForm(
        username = "tom",
        password = "123",
        age = 16
    )

    val errors = validate(form) {
        rule("用户名长度不能小于 4") { it.username.length >= 4 }
        rule("密码长度不能小于 6") { it.password.length >= 6 }
        rule("年龄不能小于 18") { it.age >= 18 }
    }

    println(errors)
}
```

输出：

```text
[用户名长度不能小于 4, 密码长度不能小于 6, 年龄不能小于 18]
```

这里有两种 Lambda：

```kotlin
predicate: (T) -> Boolean
```

普通 Lambda，用来判断规则是否通过。

```kotlin
block: Validator<T>.() -> Unit
```

带接收者的 Lambda，用来写出接近配置的调用方式。

### 实战 Demo：安全执行

异常处理也适合用 Lambda 封装。

```kotlin
fun <T> safeRun(defaultValue: T, block: () -> T): T {
    return try {
        block()
    } catch (e: Exception) {
        println("执行失败: ${e.message}")
        defaultValue
    }
}

fun main() {
    val value1 = safeRun(0) {
        "100".toInt()
    }

    val value2 = safeRun(0) {
        "abc".toInt()
    }

    println(value1)
    println(value2)
}
```

输出：

```text
执行失败: For input string: "abc"
100
0
```

`safeRun` 负责异常兜底，传入的 Lambda 负责具体业务。

### 实战 Demo：耗时统计

```kotlin
fun <T> measure(name: String, block: () -> T): T {
    val start = System.currentTimeMillis()
    val result = block()
    val cost = System.currentTimeMillis() - start
    println("$name 耗时 ${cost}ms")
    return result
}

fun main() {
    val names = measure("加载用户名单") {
        Thread.sleep(200)
        listOf("张三", "李四", "王五")
    }

    println(names)
}
```

输出：

```text
加载用户名单 耗时 200ms
[张三, 李四, 王五]
```

实际耗时会有少量波动。

### 实战 Demo：防重复点击

防重复点击的核心是：返回一个带时间判断的新 Lambda。

```kotlin
fun <T> throttle(intervalMillis: Long, action: (T) -> Unit): (T) -> Unit {
    var lastTime = 0L

    return { value: T ->
        val now = System.currentTimeMillis()
        if (now - lastTime >= intervalMillis) {
            lastTime = now
            action(value)
        }
    }
}

fun main() {
    val submit = throttle<String>(1000) { text ->
        println("提交: $text")
    }

    submit("第一次")
    submit("第二次")

    Thread.sleep(1100)

    submit("第三次")
}
```

可能输出：

```text
提交: 第一次
提交: 第三次
```

第二次调用和第一次距离太近，所以被忽略。

### SAM 转换

SAM 是 Single Abstract Method 的缩写，表示只有一个抽象方法的接口。

Java 里很多回调接口都属于 SAM。

例如 Java 常见写法：

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("run");
    }
}).start();
```

Kotlin 可以直接写 Lambda：

```kotlin
Thread {
    println("run")
}.start()
```

这是因为 `Runnable` 只有一个抽象方法，Kotlin 可以把 Lambda 转成对应接口对象。

自定义 Kotlin `fun interface` 也支持这种写法：

```kotlin
fun interface ClickListener {
    fun onClick(id: Int)
}

fun setClickListener(listener: ClickListener) {
    listener.onClick(100)
}

fun main() {
    setClickListener { id ->
        println("点击 id=$id")
    }
}
```

输出：

```text
点击 id=100
```

### Lambda 和 inline

Lambda 传给函数时，可能会生成函数对象。

对于短小、频繁调用的高阶函数，可以使用 `inline`。

```kotlin
inline fun repeatTimes(times: Int, action: (Int) -> Unit) {
    for (i in 0 until times) {
        action(i)
    }
}

fun main() {
    repeatTimes(3) {
        println("第 $it 次")
    }
}
```

输出：

```text
第 0 次
第 1 次
第 2 次
```

`inline` 会让编译器把函数体展开到调用处，减少部分函数对象开销。

不过 `inline` 不是越多越好。函数体太大时，内联会让生成代码变多。

### 常见写法对比

#### 单参数短逻辑

```kotlin
users.filter { it.age >= 18 }
```

#### 单参数复杂逻辑

```kotlin
users.filter { user ->
    user.age >= 18 &&
        user.city == "上海" &&
        user.balance >= 1000
}
```

#### 多参数 Lambda

```kotlin
map.forEach { key, value ->
    println("$key = $value")
}
```

#### 忽略参数

```kotlin
map.forEach { _, value ->
    println(value)
}
```

#### 函数引用

```kotlin
fun isPaid(order: Order): Boolean {
    return order.paid
}

val paidOrders = orders.filter(::isPaid)
```

如果已有函数刚好能匹配 Lambda 类型，函数引用更干净。

### 常见注意点

#### 不要把 it 用到看不懂

下面这种写法阅读成本很高：

```kotlin
orders.map {
    it.copy(id = it.id.trim(), city = it.city.trim())
}.filter {
    it.paid && it.amount > 100
}
```

拆出参数名会好一些：

```kotlin
orders.map { order ->
    order.copy(id = order.id.trim(), city = order.city.trim())
}.filter { order ->
    order.paid && order.amount > 100
}
```

#### Lambda 太长就提取函数

```kotlin
fun isValidOrder(order: Order): Boolean {
    return order.paid && order.amount > 100 && order.city.isNotBlank()
}

val result = orders.filter(::isValidOrder)
```

有名字的函数可以解释业务含义，比一大段 Lambda 更直观。

#### 闭包里修改 var 要谨慎

```kotlin
var total = 0.0

orders.forEach {
    total += it.amount
}
```

能写，但更推荐使用集合函数表达累计逻辑：

```kotlin
val total = orders.sumOf { it.amount }
```

状态越少，代码越稳。

#### map 和 forEach 不要混用目的

`map` 用来转换并产生新集合。

```kotlin
val names = users.map { it.name }
```

`forEach` 用来执行动作。

```kotlin
users.forEach {
    println(it.name)
}
```

不要为了打印日志使用 `map`。

### 总结

Lambda 是一段可以传递的匿名函数代码。

重点掌握这些内容：

* 基础格式：`{ 参数 -> 函数体 }`
* 函数类型：`(Int, Int) -> Int`
* 单参数简写：`it`
* 最后一行自动作为返回值
* 尾随 Lambda：最后一个 Lambda 参数可以放到括号外
* 标签返回：`return@map`、`return@forEach`
* 闭包：Lambda 可以访问外部变量
* 接收者 Lambda：`String.() -> Int`
* SAM 转换：Lambda 可以转换成单抽象方法接口
* 性能优化：合适时使用 `inline`

Lambda 写得好，代码会更短、更清楚。Lambda 写得太满，代码也会变绕。简单逻辑直接写 Lambda，复杂业务提取成有名字的函数，通常是更稳的写法。
