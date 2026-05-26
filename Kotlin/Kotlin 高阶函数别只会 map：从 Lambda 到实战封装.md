# Kotlin 高阶函数别只会 map：从 Lambda 到实战封装

### 简介

Kotlin 高阶函数（Higher-order Function）一句话就能说清楚：

> 函数的参数是函数，或者函数的返回值是函数。

听起来像绕口令，实际开发里非常常见。

比如集合里的 `map`、`filter`、`forEach`，空值处理常用的 `let`，对象初始化常用的 `apply`，本质上都离不开高阶函数。

高阶函数的价值不在于“语法高级”，而在于把容易变化的逻辑抽出去。固定流程写在函数里，变化的部分通过 Lambda 传进去。

常见场景：

* 集合转换、过滤、统计
* 回调处理
* 日志、耗时统计、异常兜底
* 重试机制
* 点击防抖
* 策略逻辑切换
* DSL 风格 API

### 函数类型

Kotlin 里函数也有类型。

格式如下：

```kotlin
(参数类型1, 参数类型2) -> 返回值类型
```

常见写法：

```kotlin
() -> Unit
```

表示无参数、无返回值。

```kotlin
(String) -> Unit
```

表示接收一个 `String`，没有返回值。

```kotlin
(Int, Int) -> Int
```

表示接收两个 `Int`，返回一个 `Int`。

```kotlin
((Int, Int) -> Int)?
```

表示一个可空的函数类型。

示例：

```kotlin
val printName: (String) -> Unit = { name ->
    println(name)
}

val add: (Int, Int) -> Int = { a, b ->
    a + b
}

fun main() {
    printName("Kotlin")
    println(add(10, 20))
}
```

输出：

```text
Kotlin
30
```

### 第一个高阶函数

普通函数一般传数字、字符串、对象。

高阶函数可以传一段逻辑。

```kotlin
fun calculate(a: Int, b: Int, operation: (Int, Int) -> Int): Int {
    return operation(a, b)
}

fun main() {
    val r1 = calculate(10, 5) { x, y ->
        x + y
    }

    val r2 = calculate(10, 5) { x, y ->
        x * y
    }

    println(r1)
    println(r2)
}
```

输出：

```text
15
50
```

`calculate` 本身不关心到底是加法、乘法还是减法，只负责接收两个数字，然后执行传进来的 `operation`。

这种写法很像“模板”：

* 固定部分：两个数字参与计算
* 变化部分：具体怎么算

变化部分就用函数参数传进去。

### Lambda 表达式

高阶函数最常搭配 Lambda 使用。

基本格式：

```kotlin
{ 参数 -> 函数体 }
```

示例：

```kotlin
val square: (Int) -> Int = { number ->
    number * number
}

fun main() {
    println(square(6))
}
```

输出：

```text
36
```

如果 Lambda 只有一个参数，可以用 `it`。

```kotlin
val square: (Int) -> Int = {
    it * it
}
```

`it` 适合非常短的逻辑。逻辑一长，最好写出参数名，不然可读性会下降。

### Lambda 简写规则

Kotlin 的高阶函数看起来简洁，主要靠这几条规则。

#### 最后一个参数是 Lambda，可以放到括号外

完整写法：

```kotlin
calculate(10, 5, { x, y ->
    x + y
})
```

常用写法：

```kotlin
calculate(10, 5) { x, y ->
    x + y
}
```

#### 只有一个 Lambda 参数，括号可以省略

```kotlin
val names = listOf("Tom", "Jerry", "Spike")

names.forEach {
    println(it)
}
```

#### 未使用的参数可以用 `_`

```kotlin
val scores = mapOf("Tom" to 90, "Jerry" to 85)

scores.forEach { _, score ->
    println(score)
}
```

### 函数引用

如果已经有现成函数，可以用 `::` 直接引用。

```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}

fun calculate(a: Int, b: Int, operation: (Int, Int) -> Int): Int {
    return operation(a, b)
}

fun main() {
    val result = calculate(10, 20, ::add)
    println(result)
}
```

输出：

```text
30
```

成员函数也可以引用：

```kotlin
class User(val name: String) {
    fun sayHello() {
        println("hello, $name")
    }
}

fun main() {
    val action: (User) -> Unit = User::sayHello
    action(User("Kotlin"))
}
```

输出：

```text
hello, Kotlin
```

### 返回一个函数

高阶函数不只能接收函数，也能返回函数。

比如根据运算符返回不同的计算逻辑：

```kotlin
fun getOperation(operator: String): (Int, Int) -> Int {
    return when (operator) {
        "+" -> { a, b -> a + b }
        "-" -> { a, b -> a - b }
        "*" -> { a, b -> a * b }
        "/" -> { a, b -> a / b }
        else -> { _, _ -> 0 }
    }
}

fun main() {
    val add = getOperation("+")
    val multiply = getOperation("*")

    println(add(8, 3))
    println(multiply(8, 3))
}
```

输出：

```text
11
24
```

这类写法适合做策略选择。外层函数负责决定使用哪套规则，返回值就是具体规则。

### 集合里的高阶函数

Kotlin 集合 API 是高阶函数最常见的使用场景。

准备数据：

```kotlin
data class User(
    val id: Int,
    val name: String,
    val age: Int,
    val city: String,
    val balance: Double
)

val users = listOf(
    User(1, "张三", 17, "杭州", 120.0),
    User(2, "李四", 22, "上海", 860.0),
    User(3, "王五", 28, "杭州", 1500.0),
    User(4, "赵六", 31, "深圳", 620.0),
    User(5, "钱七", 19, "上海", 300.0)
)
```

#### filter：过滤

```kotlin
val adults = users.filter { it.age >= 18 }

adults.forEach {
    println("${it.name} ${it.age}")
}
```

只保留满足条件的数据。

#### map：转换

```kotlin
val names = users.map { it.name }

println(names)
```

输出：

```text
[张三, 李四, 王五, 赵六, 钱七]
```

`map` 的重点是“把一种数据变成另一种数据”。

#### sortedBy：排序

```kotlin
val sortedUsers = users.sortedBy { it.age }

sortedUsers.forEach {
    println("${it.name} ${it.age}")
}
```

#### groupBy：分组

```kotlin
val usersByCity = users.groupBy { it.city }

usersByCity.forEach { city, list ->
    println("$city: ${list.map { it.name }}")
}
```

输出：

```text
杭州: [张三, 王五]
上海: [李四, 钱七]
深圳: [赵六]
```

#### fold：累加

```kotlin
val totalBalance = users.fold(0.0) { total, user ->
    total + user.balance
}

println(totalBalance)
```

`fold` 适合把一组数据累计成一个结果，比如总金额、总数量、拼接字符串。

### 实战 Demo：用户报表

需求：

* 只统计成年人
* 按城市分组
* 统计每个城市的人数和余额总和
* 按余额总和倒序排列

代码：

```kotlin
data class User(
    val id: Int,
    val name: String,
    val age: Int,
    val city: String,
    val balance: Double
)

data class CityReport(
    val city: String,
    val userCount: Int,
    val totalBalance: Double
)

fun buildCityReport(users: List<User>): List<CityReport> {
    return users
        .filter { it.age >= 18 }
        .groupBy { it.city }
        .map { (city, cityUsers) ->
            CityReport(
                city = city,
                userCount = cityUsers.size,
                totalBalance = cityUsers.sumOf { it.balance }
            )
        }
        .sortedByDescending { it.totalBalance }
}

fun main() {
    val users = listOf(
        User(1, "张三", 17, "杭州", 120.0),
        User(2, "李四", 22, "上海", 860.0),
        User(3, "王五", 28, "杭州", 1500.0),
        User(4, "赵六", 31, "深圳", 620.0),
        User(5, "钱七", 19, "上海", 300.0)
    )

    val reports = buildCityReport(users)

    reports.forEach {
        println("${it.city} 人数=${it.userCount}, 余额=${it.totalBalance}")
    }
}
```

输出：

```text
杭州 人数=1, 余额=1500.0
上海 人数=2, 余额=1160.0
深圳 人数=1, 余额=620.0
```

这段代码没有写临时变量、没有手动维护 Map、没有手动排序数组。每一步都只表达一件事：

* `filter`：先过滤
* `groupBy`：再分组
* `map`：转成报表对象
* `sortedByDescending`：最后排序

### 自定义高阶函数：安全执行

很多业务代码都需要 `try-catch`，如果每个地方都写一遍，代码会很散。

可以把固定的异常处理流程封装起来，把真正要执行的逻辑作为 Lambda 传进去。

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
    val result1 = safeRun(0) {
        100 / 2
    }

    val result2 = safeRun(0) {
        100 / 0
    }

    println(result1)
    println(result2)
}
```

输出：

```text
执行失败: / by zero
50
0
```

`safeRun` 的固定部分是异常捕获，变化部分是 `block()` 里的业务代码。

### 自定义高阶函数：耗时统计

耗时统计也是很适合高阶函数的场景。

```kotlin
fun <T> measure(name: String, block: () -> T): T {
    val start = System.currentTimeMillis()
    val result = block()
    val cost = System.currentTimeMillis() - start
    println("$name 耗时 ${cost}ms")
    return result
}

fun main() {
    val result = measure("加载用户") {
        Thread.sleep(300)
        listOf("张三", "李四", "王五")
    }

    println(result)
}
```

输出：

```text
加载用户 耗时 300ms
[张三, 李四, 王五]
```

实际耗时会有少量波动。

### 自定义高阶函数：重试机制

请求接口、读写外部资源时，偶发失败很常见。重试流程可以封装成高阶函数。

```kotlin
fun <T> retry(times: Int = 3, block: () -> T): T {
    var lastError: Exception? = null

    repeat(times) { index ->
        try {
            return block()
        } catch (e: Exception) {
            lastError = e
            println("第 ${index + 1} 次执行失败: ${e.message}")
        }
    }

    throw lastError ?: RuntimeException("执行失败")
}

fun main() {
    var count = 0

    val result = retry(times = 3) {
        count++
        if (count < 3) {
            throw RuntimeException("网络异常")
        }
        "请求成功"
    }

    println(result)
}
```

输出：

```text
第 1 次执行失败: 网络异常
第 2 次执行失败: 网络异常
请求成功
```

重试次数、异常记录、失败后抛出，这些流程都固定下来。具体执行什么，由 `block` 决定。

### 返回函数 Demo：防抖

防抖函数的特点是：传入一个动作，返回一个新动作。

新动作里带着时间判断，短时间重复调用会被忽略。

```kotlin
fun <T> debounce(intervalMillis: Long, action: (T) -> Unit): (T) -> Unit {
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
    val search = debounce<String>(1000) { keyword ->
        println("搜索: $keyword")
    }

    search("kotlin")
    search("kotlin 高阶函数")

    Thread.sleep(1100)

    search("kotlin lambda")
}
```

可能输出：

```text
搜索: kotlin
搜索: kotlin lambda
```

第二次调用距离第一次太近，所以没有执行。

### 作用域函数也是高阶函数

`let`、`run`、`with`、`apply`、`also` 都接收 Lambda，所以它们也是高阶函数的典型应用。

#### let

常用于非空处理和结果转换。

```kotlin
val name: String? = "Kotlin"

val length = name?.let {
    it.length
}

println(length)
```

#### apply

常用于对象初始化，返回对象本身。

```kotlin
data class Config(
    var host: String = "",
    var port: Int = 0,
    var debug: Boolean = false
)

val config = Config().apply {
    host = "127.0.0.1"
    port = 8080
    debug = true
}

println(config)
```

#### also

常用于插入日志、调试、附加操作，返回对象本身。

```kotlin
val numbers = mutableListOf(1, 2, 3)
    .also { println("添加前: $it") }
    .apply { add(4) }
    .also { println("添加后: $it") }
```

### inline 内联函数

高阶函数虽然好用，但 Lambda 在某些情况下会产生函数对象，频繁调用时可能有额外开销。

Kotlin 提供 `inline`，可以让编译器把函数体展开到调用处。

```kotlin
inline fun repeatTimes(times: Int, action: (Int) -> Unit) {
    for (i in 0 until times) {
        action(i)
    }
}

fun main() {
    repeatTimes(3) {
        println("第 $it 次执行")
    }
}
```

输出：

```text
第 0 次执行
第 1 次执行
第 2 次执行
```

`inline` 适合短小、频繁调用、参数里带 Lambda 的函数。

不要所有高阶函数都加 `inline`。函数体太大时，内联会让生成的代码变多。

### noinline

`inline` 函数里的 Lambda 参数默认都会被内联。

如果某个 Lambda 需要被保存、返回、传给其它非内联函数，就要加 `noinline`。

```kotlin
inline fun runTask(
    mainTask: () -> Unit,
    noinline finishTask: () -> Unit
) {
    mainTask()
    saveCallback(finishTask)
}

fun saveCallback(callback: () -> Unit) {
    callback()
}
```

`mainTask` 会被内联，`finishTask` 不会被内联。

### crossinline

`crossinline` 用来限制 Lambda 里的非局部返回。

看一个容易踩坑的例子：

```kotlin
inline fun execute(block: () -> Unit) {
    block()
}

fun main() {
    execute {
        println("before return")
        return
    }

    println("after execute")
}
```

这里的 `return` 会直接从 `main` 返回，`after execute` 不会执行。

如果 Lambda 会被放到另一个对象或线程里执行，就不允许这种返回方式，需要 `crossinline`。

```kotlin
inline fun postTask(crossinline block: () -> Unit) {
    val runnable = Runnable {
        block()
    }
    runnable.run()
}
```

加了 `crossinline` 后，`block` 里不能直接 `return` 到外层函数。

### 高阶函数和接口回调的区别

Java 里常见写法：

```java
button.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        submit();
    }
});
```

Kotlin 高阶函数可以写得更轻：

```kotlin
fun setClickListener(onClick: () -> Unit) {
    onClick()
}

fun submit() {
    println("submit")
}

fun main() {
    setClickListener {
        submit()
    }
}
```

如果只是传一段行为，Lambda 往往比接口更直接。

不过接口也不是没用。回调方法很多、需要表达一组能力、需要面向 Java 暴露 API 时，接口依然合适。

### 完整 Demo：小型数据处理管道

下面写一个简单的数据处理管道。每一步都是一个函数，上一部的结果交给下一步。

```kotlin
class Pipeline<T>(private val value: T) {

    fun <R> then(transform: (T) -> R): Pipeline<R> {
        return Pipeline(transform(value))
    }

    fun get(): T {
        return value
    }
}

fun <T> pipeline(value: T): Pipeline<T> {
    return Pipeline(value)
}

fun main() {
    val result = pipeline("  kotlin higher-order function  ")
        .then { it.trim() }
        .then { it.replace("-", " ") }
        .then { it.split(" ") }
        .then { words -> words.filter { it.isNotBlank() } }
        .then { words -> words.joinToString(separator = "_") { it.uppercase() } }
        .get()

    println(result)
}
```

输出：

```text
KOTLIN_HIGHER_ORDER_FUNCTION
```

这个 Demo 里，`then` 是高阶函数：

```kotlin
fun <R> then(transform: (T) -> R): Pipeline<R>
```

每次调用 `then`，都传入一个转换规则。管道本身不关心具体怎么转换，只负责把数据交给下一个规则。

### 常见注意点

#### it 不要滥用

短逻辑可以用 `it`：

```kotlin
users.filter { it.age >= 18 }
```

逻辑变长时，命名参数更清楚：

```kotlin
users.filter { user ->
    user.age >= 18 && user.balance > 500
}
```

#### Lambda 最后一行就是返回值

```kotlin
val result = listOf(1, 2, 3).map {
    val double = it * 2
    double + 1
}

println(result)
```

输出：

```text
[3, 5, 7]
```

`double + 1` 是 Lambda 的返回值。

#### reduce 和 fold 不一样

`reduce` 没有初始值，会从集合第一个元素开始。

```kotlin
val sum1 = listOf(1, 2, 3).reduce { total, item ->
    total + item
}
```

`fold` 有初始值，更适合空集合或需要指定初始状态的场景。

```kotlin
val sum2 = emptyList<Int>().fold(0) { total, item ->
    total + item
}
```

空集合调用 `reduce` 会报错，调用 `fold` 更稳。

#### 高阶函数别写得太绕

高阶函数可以让代码很简洁，也可能让代码变得很难读。

下面这种链式写法已经不太友好：

```kotlin
val result = users
    .filter { it.age > 18 }
    .map { it.copy(name = it.name.trim()) }
    .groupBy { it.city }
    .mapValues { it.value.sortedByDescending { user -> user.balance } }
    .filterValues { it.isNotEmpty() }
```

业务规则复杂时，可以拆成有名字的函数：

```kotlin
fun isValidAdult(user: User): Boolean {
    return user.age > 18 && user.name.isNotBlank()
}

fun normalizeUser(user: User): User {
    return user.copy(name = user.name.trim())
}

val result = users
    .filter(::isValidAdult)
    .map(::normalizeUser)
    .groupBy { it.city }
```

可读性通常比一长串 Lambda 更重要。

### 总结

Kotlin 高阶函数的核心就是把函数当成普通值使用。

重点掌握这几块：

* 函数类型：`(Int, Int) -> Int`
* Lambda：`{ a, b -> a + b }`
* 函数引用：`::add`
* 函数作为参数：把变化逻辑传进去
* 函数作为返回值：根据条件返回不同策略
* 集合高阶函数：`map`、`filter`、`groupBy`、`fold`
* 作用域函数：`let`、`run`、`with`、`apply`、`also`
* 性能优化：合适时使用 `inline`

高阶函数不是为了炫技，而是为了把重复流程收起来，把真正变化的业务逻辑露出来。写得好，代码会更短、更清楚，也更容易复用。
