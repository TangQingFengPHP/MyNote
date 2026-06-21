### 简介

一段 Lambda 看起来只有几行代码：

```kotlin
measure {
    loadUsers()
}
```

为什么 Kotlin 还专门提供了 `inline`？

如果只是为了“少调用一次函数”，JVM 自己也会做方法内联，似乎没必要再加一个关键字。

真正的原因在于 Kotlin 高阶函数。

普通高阶函数接收 Lambda 时，Lambda 需要以函数对象的形式传进去，再通过 `invoke()` 执行。某些无捕获 Lambda 可以被缓存，JVM 也可能继续优化，但从语言模型看，它仍然是一份需要传递和调用的函数值。

`inline` 会在编译期把函数体和可内联 Lambda 展开到调用处，常见收益有三个：

* 减少函数对象和间接调用带来的开销
* 允许 Lambda 使用非局部 `return`
* 配合 `reified` 在运行时使用泛型类型

所以 `inline` 不只是一个性能关键字，它还会改变 Lambda 的控制流能力和泛型能力。

这也是 `let`、`run`、`apply`、`also`、`use`、`forEach` 等标准库函数大量使用 `inline` 的原因。

### 先看普通高阶函数发生了什么

定义一个耗时统计函数：

```kotlin
fun <T> measure(name: String, block: () -> T): T {
    val start = System.nanoTime()
    val result = block()
    val cost = System.nanoTime() - start
    println("$name 耗时 ${cost}ns")
    return result
}
```

调用：

```kotlin
fun loadUsers(): List<String> {
    return listOf("Tom", "Lucy", "Jack")
}

fun main() {
    val users = measure("加载用户") {
        loadUsers()
    }

    println(users)
}
```

源码看起来像把几行代码传给了 `measure`，实际传入的是一个函数值。

在 JVM 上可以近似理解为：

```java
List<String> users = measure(
    "加载用户",
    new Function0<List<String>>() {
        @Override
        public List<String> invoke() {
            return loadUsers();
        }
    }
);
```

这里通常涉及：

```text
Lambda 对应的函数对象
block.invoke() 间接调用
Lambda 捕获外部变量时保存捕获值
```

不过不能简单下结论说“每次调用一定创建新对象”。

无捕获 Lambda 可能被编译成单例，JVM 的 JIT 也可能消除部分分配和调用。`inline` 的意义是让 Kotlin 编译器在编译阶段明确展开代码，而不是把所有希望都留给运行时优化。

### inline 到底内联了什么

给 `measure` 加上 `inline`：

```kotlin
inline fun <T> measure(name: String, block: () -> T): T {
    val start = System.nanoTime()
    val result = block()
    val cost = System.nanoTime() - start
    println("$name 耗时 ${cost}ns")
    return result
}
```

调用代码不变：

```kotlin
val users = measure("加载用户") {
    loadUsers()
}
```

编译后的效果可以近似看成：

```kotlin
val start = System.nanoTime()
val users = loadUsers()
val cost = System.nanoTime() - start
println("加载用户 耗时 ${cost}ns")
```

展开的内容有两部分：

* `measure` 的函数体
* 传给 `block` 的 Lambda 代码

原来的 `block()` 调用消失了，Lambda 的内容直接放到了对应位置。

这里的“展开”是帮助理解的近似说法，实际生成结果还会受到后端、调试信息和编译器优化影响。

### inline 不等于一定更快

`inline` 经常和性能放在一起讲，但不能把它当成“加了就快”的开关。

适合内联的函数通常有这些特征：

* 接收一个或多个 Lambda
* 函数体比较短
* 调用频率较高
* Lambda 通常立即执行，不需要保存
* 需要非局部返回
* 需要 `reified` 泛型

不适合随便内联的函数：

* 函数体很大
* 调用点很多
* 没有 Lambda，也不需要 `reified`
* 很少调用，性能收益没有实际意义
* 递归函数

内联函数体会复制到每个调用点。

假设一个 100 行函数被调用 50 次，内联后可能在不同位置重复生成大量代码，带来：

* 字节码体积增大
* 编译时间增加
* 指令缓存压力增大
* 调试栈和代码定位更复杂

给没有函数参数、也没有 `reified` 参数的普通函数加 `inline`，编译器通常还会给出 `NOTHING_TO_INLINE` 警告。

```kotlin
// 通常没有必要
inline fun add(a: Int, b: Int): Int {
    return a + b
}
```

### Demo：用 inline 封装统一执行流程

很多项目都会重复写日志、耗时统计和异常处理。

可以把固定流程放进一个内联函数：

```kotlin
inline fun <T> executeTask(
    taskName: String,
    block: () -> T
): Result<T> {
    val start = System.nanoTime()
    println("[$taskName] 开始")

    return try {
        val result = block()
        val cost = System.nanoTime() - start
        println("[$taskName] 成功，耗时 ${cost}ns")
        Result.success(result)
    } catch (e: Exception) {
        val cost = System.nanoTime() - start
        println("[$taskName] 失败，耗时 ${cost}ns，原因：${e.message}")
        Result.failure(e)
    }
}
```

使用：

```kotlin
fun queryOrder(orderId: Long): String {
    require(orderId > 0) { "订单编号必须大于 0" }
    return "ORDER-$orderId"
}

fun main() {
    val success = executeTask("查询订单") {
        queryOrder(1001)
    }

    val failure = executeTask("查询订单") {
        queryOrder(-1)
    }

    println(success.getOrNull())
    println(failure.exceptionOrNull()?.message)
}
```

输出类似：

```text
[查询订单] 开始
[查询订单] 成功，耗时 350000ns
[查询订单] 开始
[查询订单] 失败，耗时 120000ns，原因：订单编号必须大于 0
ORDER-1001
订单编号必须大于 0
```

实际耗时会随运行环境变化。

这种小型流程函数很适合 `inline`：固定逻辑不复杂，Lambda 在函数内立即执行，也不需要保存到其他地方。

### inline 最容易忽略的能力：非局部返回

先看普通 Lambda：

```kotlin
fun runBlock(block: () -> Unit) {
    block()
}

fun findUser() {
    runBlock {
        // 编译失败：不能从 findUser 直接返回
        // return
    }
}
```

普通 Lambda 是一个函数对象，执行 `return` 时不能跨过 `runBlock`，直接退出外层的 `findUser`。

改成内联函数：

```kotlin
inline fun runBlock(block: () -> Unit) {
    block()
}
```

现在可以这样写：

```kotlin
fun printFirstAdmin(users: List<String>) {
    users.forEach { user ->
        if (user == "admin") {
            println("找到管理员")
            return
        }

        println("检查用户：$user")
    }

    println("没有找到管理员")
}

fun main() {
    printFirstAdmin(listOf("Tom", "admin", "Lucy"))
}
```

输出：

```text
检查用户：Tom
找到管理员
```

这里的 `return` 不是退出 `forEach` 的 Lambda，而是直接退出 `printFirstAdmin`。

原因是 `forEach` 本身就是内联函数，Lambda 被展开到 `printFirstAdmin` 内部，`return` 因而可以作用于外层函数。

这种行为叫非局部返回，Non-local Return。

### 只想跳过当前元素，要使用标签返回

非局部返回很方便，也很容易误伤外层流程。

如果需求只是跳过当前元素，不能直接写裸 `return`，应该写 `return@forEach`：

```kotlin
fun printNormalUsers(users: List<String>) {
    users.forEach { user ->
        if (user == "admin") {
            return@forEach
        }

        println(user)
    }

    println("处理完成")
}

fun main() {
    printNormalUsers(listOf("Tom", "admin", "Lucy"))
}
```

输出：

```text
Tom
Lucy
处理完成
```

两种返回的区别：

```kotlin
return           // 退出外层函数
return@forEach   // 只退出当前 Lambda 调用
```

代码审查时看到内联 Lambda 里的裸 `return`，需要特别留意它到底退出哪一层。

### noinline：这个 Lambda 需要作为对象保存

内联函数里的 Lambda 参数默认可以被内联。

可内联参数只能在允许内联的位置调用，不能随意保存、返回，或者传给普通非内联函数。

下面的代码会报错：

```kotlin
inline fun register(block: () -> Unit) {
    // 编译失败：非法使用内联参数
    // listeners += block
}
```

原因很直接：`block` 原本要在调用点展开，现在却要被保存到集合里，后面再执行。编译器无法同时把它当成“展开的代码”和“长期存在的函数对象”。

这时需要 `noinline`：

```kotlin
class EventBus {
    private val listeners = mutableListOf<(String) -> Unit>()

    fun addListener(listener: (String) -> Unit) {
        listeners += listener
    }

    fun publish(message: String) {
        listeners.forEach { listener ->
            listener(message)
        }
    }
}

inline fun EventBus.register(
    eventName: String,
    noinline listener: (String) -> Unit
) {
    println("注册事件：$eventName")
    addListener(listener)
}
```

使用：

```kotlin
fun main() {
    val eventBus = EventBus()

    eventBus.register("order-created") { message ->
        println("收到事件：$message")
    }

    eventBus.publish("订单 1001 已创建")
}
```

输出：

```text
注册事件：order-created
收到事件：订单 1001 已创建
```

这里的 `listener` 必须继续存在，所以它不能被内联消掉。

`noinline` 参数具有普通函数值的能力：

* 保存到属性或集合
* 赋值给变量
* 作为返回值返回
* 传给普通高阶函数
* 稍后或异步执行

代价也很明确：它不再享受该 Lambda 参数的内联收益，而且不能使用非局部 `return`。

### 多个 Lambda 可以分别决定是否内联

同一个函数经常既有立即执行的 Lambda，也有需要保存的回调。

```kotlin
class TaskRegistry {
    private val callbacks = mutableMapOf<String, (String) -> Unit>()

    fun save(name: String, callback: (String) -> Unit) {
        callbacks[name] = callback
    }

    fun complete(name: String, result: String) {
        callbacks[name]?.invoke(result)
    }
}

inline fun <T> TaskRegistry.submit(
    name: String,
    action: () -> T,
    noinline onComplete: (String) -> Unit
): T {
    save(name, onComplete)
    return action()
}
```

调用：

```kotlin
fun main() {
    val registry = TaskRegistry()

    val orderId = registry.submit(
        name = "create-order",
        action = {
            println("创建订单")
            1001L
        },
        onComplete = { result ->
            println("任务完成：$result")
        }
    )

    println("订单编号：$orderId")
    registry.complete("create-order", "success")
}
```

输出：

```text
创建订单
订单编号：1001
任务完成：success
```

两个 Lambda 的角色不同：

```text
action      立即执行，可以内联
onComplete  保存起来以后执行，必须 noinline
```

### crossinline：仍然内联，但禁止跨出去 return

另一个常见问题是，Lambda 不在当前位置直接调用，而是放进另一个执行结构里。

例如：

```kotlin
inline fun runOnThread(block: () -> Unit) {
    Thread {
        block()
    }.start()
}
```

这段代码无法通过编译。

`block` 原本允许非局部返回，但真正执行它的是另一个线程里的 `Runnable`。等子线程执行时，外层函数早已可能结束，裸 `return` 根本没有合法的返回目标。

解决方式是添加 `crossinline`：

```kotlin
inline fun runOnThread(crossinline block: () -> Unit): Thread {
    return Thread {
        block()
    }.apply {
        start()
    }
}
```

使用：

```kotlin
fun main() {
    val thread = runOnThread {
        println("子线程执行任务")

        // 编译失败：crossinline 禁止非局部返回
        // return
    }

    thread.join()
    println("主线程结束")
}
```

输出：

```text
子线程执行任务
主线程结束
```

`crossinline` 的含义可以拆成两部分：

```text
cross：Lambda 会跨到另一层函数、对象或执行上下文中调用
inline：仍然保留内联参数的性质，但禁止非局部 return
```

需要注意，`crossinline` 不保证整个过程绝对不创建对象。

上面的 `Thread { ... }` 本身需要一个 `Runnable`，编译器可能生成包装对象。`crossinline` 表示参数没有被 `noinline` 化，不代表外围执行结构不需要对象。

### inline、noinline、crossinline 对比

| 写法 | Lambda 是否可内联 | 能否保存或稍后执行 | 能否非局部 return | 常见场景 |
| --- | --- | --- | --- | --- |
| 普通 Lambda 参数 | 否 | 可以 | 不可以 | 一般高阶函数 |
| `inline` 参数 | 可以 | 不可以 | 可以 | 立即执行的小型 Lambda |
| `noinline` 参数 | 否 | 可以 | 不可以 | 回调保存、返回、转交 |
| `crossinline` 参数 | 可以 | 不能直接当普通函数值保存 | 不可以 | 嵌套对象、线程、延迟执行结构 |

一句白话概括：

```text
inline      现场展开
noinline    留下一个函数对象以后用
crossinline 可以跨层调用，但不能从外层函数直接 return
```

### reified：inline 解决泛型类型擦除

普通泛型函数里不能直接判断 `value is T`：

```kotlin
fun <T> isType(value: Any): Boolean {
    // 编译失败：运行时不知道 T 是什么
    // return value is T
    return false
}
```

JVM 泛型通常存在类型擦除。运行时执行到函数内部时，`T` 的具体类型信息可能已经不存在。

传统做法是显式传入 `Class` 或 `KClass`：

```kotlin
fun <T : Any> isType(value: Any, type: kotlin.reflect.KClass<T>): Boolean {
    return type.isInstance(value)
}

fun main() {
    println(isType("Kotlin", String::class))
    println(isType(100, String::class))
}
```

输出：

```text
true
false
```

配合 `inline` 和 `reified`，类型参数可以实化：

```kotlin
inline fun <reified T> isType(value: Any): Boolean {
    return value is T
}
```

调用：

```kotlin
fun main() {
    println(isType<String>("Kotlin"))
    println(isType<String>(100))
    println(isType<Int>(100))
}
```

输出：

```text
true
false
true
```

为什么 `reified` 必须和 `inline` 一起使用？

调用处本来就知道具体类型：

```kotlin
isType<String>(value)
```

函数内联后，编译器可以把 `T` 替换成 `String`，效果近似：

```kotlin
value is String
```

如果函数没有内联，运行时只进入同一份通用函数体，类型擦除后就无法直接知道每次调用传入的 `T`。

### Demo：按类型过滤混合数据

实现一个简化版 `filterIsInstance`：

```kotlin
inline fun <reified T> Iterable<*>.onlyType(): List<T> {
    return buildList {
        for (item in this@onlyType) {
            if (item is T) {
                add(item)
            }
        }
    }
}
```

使用：

```kotlin
fun main() {
    val values: List<Any> = listOf(
        "Kotlin",
        100,
        "Java",
        true,
        200
    )

    val strings = values.onlyType<String>()
    val numbers = values.onlyType<Int>()

    println(strings)
    println(numbers)
}
```

输出：

```text
[Kotlin, Java]
[100, 200]
```

`reified` 常见用途：

* `value is T`
* `value as? T`
* `T::class`
* 按类型查找对象
* 序列化和反序列化入口
* 依赖容器按类型取对象
* 泛型路由或事件分发

### Demo：按类型读取简单容器

```kotlin
class ServiceContainer {
    private val services = mutableMapOf<kotlin.reflect.KClass<*>, Any>()

    fun <T : Any> register(type: kotlin.reflect.KClass<T>, service: T) {
        services[type] = service
    }

    fun <T : Any> get(type: kotlin.reflect.KClass<T>): T? {
        return type.java.cast(services[type])
    }
}

inline fun <reified T : Any> ServiceContainer.register(service: T) {
    register(T::class, service)
}

inline fun <reified T : Any> ServiceContainer.get(): T? {
    return get(T::class)
}
```

定义服务：

```kotlin
interface UserService {
    fun findName(id: Long): String
}

class DefaultUserService : UserService {
    override fun findName(id: Long): String {
        return "User-$id"
    }
}
```

使用：

```kotlin
fun main() {
    val container = ServiceContainer()

    container.register<UserService>(DefaultUserService())

    val userService = container.get<UserService>()
    println(userService?.findName(1001))
}
```

输出：

```text
User-1001
```

调用处不需要重复传 `UserService::class`，类型参数本身就能充当查找键。

### reified 也拿不到所有泛型细节

`reified` 能保留 `T` 对应的运行时类型，但 JVM 的嵌套泛型参数仍然可能被擦除。

例如：

```kotlin
inline fun <reified T> printType() {
    println(T::class)
}

fun main() {
    printType<List<String>>()
    printType<List<Int>>()
}
```

两次拿到的运行时类本质上都是 `List`，无法只靠 `T::class` 区分 `List<String>` 和 `List<Int>`。

所以：

```text
reified 解决了 T 本身的类型访问
不代表 JVM 泛型擦除彻底消失
```

需要完整保留嵌套泛型信息时，通常还要使用 `typeOf<T>()` 或序列化框架提供的类型令牌机制。

### inline 属性

除了函数，属性的访问器也可以内联。

```kotlin
class User(
    val firstName: String,
    val lastName: String
)

inline val User.fullName: String
    get() = "$firstName $lastName"
```

使用：

```kotlin
fun main() {
    val user = User("Tom", "Smith")
    println(user.fullName)
}
```

只有没有幕后字段的属性访问器才适合内联。

也可以单独修饰 getter 或 setter：

```kotlin
val User.displayName: String
    inline get() = "$lastName, $firstName"
```

实际项目里，内联属性远没有内联高阶函数和 `reified` 常见，了解语法即可。

### public inline 为什么不能随便访问 private 成员

下面的代码会报错：

```kotlin
class TokenStore {
    private val token = "secret"

    inline fun read(block: (String) -> Unit) {
        // 编译失败：public inline 不能访问 private 成员
        block(token)
    }
}
```

原因不是编译器故意找麻烦。

`read` 是 `public inline`，调用它的代码可能位于另一个模块。函数体会展开到调用模块里，而那个模块本来无权访问 `TokenStore` 的 `private token`。

常见解决方式：

* 把内联函数改为 `internal` 或 `private`
* 不直接访问私有成员
* 将需要暴露给内联代码的成员设为 `internal`，并谨慎使用 `@PublishedApi`

示例：

```kotlin
class TokenStore {
    @PublishedApi
    internal val token = "secret"

    inline fun read(block: (String) -> Unit) {
        block(token)
    }
}
```

`@PublishedApi` 表示这个 `internal` 成员会成为公开内联实现的一部分。它不是简单的“绕过限制”，修改时需要考虑二进制兼容性。

### 内联函数会影响库的升级方式

普通函数调用通常保留为方法调用。库升级后，只要二进制签名兼容，调用方可以在不重新编译的情况下执行新版函数体。

内联函数的旧函数体已经复制进调用方产物。

如果一个库修改了 `public inline` 函数实现，但应用没有重新编译，应用里可能继续运行旧的内联代码。

所以公共库里的内联函数需要特别注意：

* 保持实现短小稳定
* 谨慎引用 `@PublishedApi` 成员
* 修改实现后重新编译使用方
* 不把复杂且频繁变化的业务逻辑做成公共内联 API

这也是“不要无脑 inline”的另一个原因。

### 如何在 IDEA 里确认是否内联

只看源码很难感受到内联结果，可以直接看字节码反编译结果。

在 IntelliJ IDEA 中打开 Kotlin 文件，然后执行：

```text
Tools
-> Kotlin
-> Show Kotlin Bytecode
-> Decompile
```

对比普通高阶函数和内联函数时，重点看：

* 调用处是否还有原函数调用
* 是否生成 `Function0`、`Function1` 等函数对象
* 是否还有 `invoke()` 调用
* Lambda 代码是否直接出现在调用方法中

不过反编译结果只是某次编译产物，最终运行性能还会受到 JVM JIT、逃逸分析、调用频率和运行环境影响。

真正比较性能时，应使用 JMH 或 `kotlinx-benchmark`，不要用只运行一次的 `System.nanoTime()` 测试下结论。

### 常见误区

#### inline 会把所有东西都变成零开销

不会。

内联可以减少高阶函数的部分开销，但 Lambda 内部创建的集合、字符串、线程和业务对象不会凭空消失。

#### Lambda 每次一定创建新对象

不一定。

无捕获 Lambda 可能复用实例，JVM 也可能优化掉部分分配。捕获变量、保存回调和跨作用域使用时，更容易产生实际对象。

#### noinline 只是“不优化”，其他行为不变

不完全对。

`noinline` 让参数恢复成普通函数值，因此可以保存和传递，但也失去了非局部返回能力。

#### crossinline 表示异步

不是。

`crossinline` 只限制非局部返回。它常出现在异步封装里，也可以用在同步的嵌套对象或另一个 Lambda 中。

#### reified 保留全部泛型信息

不是。

`reified` 让内联函数能使用具体的 `T`，嵌套泛型参数仍可能受 JVM 类型擦除影响。

#### 普通函数调用一定比 inline 慢

不一定。

JVM JIT 本身会根据运行情况内联普通方法。Kotlin `inline` 更重要的场景是高阶函数、非局部返回和 `reified`，不是给每个小函数手动贴性能标签。

### 什么时候该用哪一个

遇到 Lambda 参数时，可以按下面的顺序判断：

```text
Lambda 是否需要保存、返回或稍后调用？
是 -> noinline 或直接使用普通高阶函数

Lambda 是否会放进另一个 Lambda、对象或执行上下文？
是 -> crossinline

Lambda 是否立即执行，而且函数短小、调用频繁？
是 -> 可以考虑 inline

是否需要 T::class、is T、as? T？
是 -> inline + reified
```

还有一个更实际的判断标准：

```text
没有测量结果时，不要为了“可能更快”把大型函数全部内联。
```

### 总结

`inline` 最核心的代码：

```kotlin
inline fun <T> measure(block: () -> T): T {
    return block()
}
```

它让函数体和可内联 Lambda 在调用处展开，常用于小型高阶函数。

三个关键字的分工：

```text
inline      展开函数体和 Lambda，支持非局部返回
noinline    保留普通函数对象，允许保存、传递和稍后执行
crossinline 保持可内联，但禁止非局部返回
```

再加上一个重要组合：

```kotlin
inline fun <reified T> isType(value: Any): Boolean {
    return value is T
}
```

`reified` 借助内联，让泛型函数能在调用处使用具体类型。

一句话概括：

> `inline` 不是给所有函数准备的性能按钮，而是 Kotlin 为高阶函数提供的一套编译期展开机制；用得合适，可以减少抽象开销，也能表达普通函数做不到的控制流和类型操作。
