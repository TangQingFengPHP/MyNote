# Kotlin Delegation（委托）详解：原理、语法、实战 Demo

Kotlin 的 Delegation（委托） 是一项非常实用的语言特性，尤其适合接口设计、代码复用、属性管理和框架开发。

Kotlin 的委托主要分为两大类：

1. 类委托（Class Delegation）：通过 `by` 将接口实现交给另一个对象。
2. 属性委托（Delegated Properties）：通过 `by` 将属性的读取、写入或初始化交给另一个对象。

例如：

```
// 类委托
class UserService(
    repository: Repository
) : Repository by repository

// 属性委托
val config by lazy {
    loadConfig()
}
```

两处都使用 `by`，但它们解决的问题完全不同。

## 一、先理解什么是委托

假设我们开发一个文件存储系统，有一个 `Storage` 接口：

```
interface Storage {
    fun upload(fileName: String)
    fun download(fileName: String)
}
```

本地存储实现：

```
class LocalStorage : Storage {

    override fun upload(fileName: String) {
        println("上传文件：$fileName")
    }

    override fun download(fileName: String) {
        println("下载文件：$fileName")
    }
}
```

现在我们需要增加一个日志存储包装器，在上传、下载时记录日志。

### 传统写法：手动委托

```
class LoggingStorage(
    private val storage: Storage
) : Storage {

    override fun upload(fileName: String) {
        println("开始上传")
        storage.upload(fileName)
    }

    override fun download(fileName: String) {
        storage.download(fileName)
    }
}
```

这里的 `LoggingStorage` 本身不负责真正存储文件，而是把工作交给 `storage`。

这就是委托的核心思想：

一个对象将部分或全部职责交给另一个对象执行。

但是存在一个问题：接口方法越来越多时，包装类需要编写大量转发代码。

### Kotlin 写法：使用 `by`

```
class LoggingStorage(
    private val storage: Storage
) : Storage by storage {

    override fun upload(fileName: String) {
        println("开始上传")
        storage.upload(fileName)
    }
}
```

使用：

```
fun main() {
    val storage = LoggingStorage(LocalStorage())

    storage.upload("hello.txt")
    storage.download("hello.txt")
}
```

输出：

```
开始上传
上传文件：hello.txt
下载文件：hello.txt
```

这里最关键的是：

```
: Storage by storage
```

它告诉编译器：

> 我实现了 Storage 接口，但没有显式重写的方法，交给 storage 对象处理。

因此，我们不需要手动实现 `download()`。

这就是 Kotlin 类委托的核心价值。

## 二、类委托的底层原理

Kotlin 的类委托不是运行时动态代理，而是由编译器生成接口方法的转发实现。

例如：

```
interface Printer {
    fun print(message: String)
}

class ConsolePrinter : Printer {
    override fun print(message: String) {
        println(message)
    }
}

class LoggingPrinter(
    private val delegate: Printer
) : Printer by delegate
```

编译器可以近似理解为生成了：

```
class LoggingPrinter(
    private val delegate: Printer
) : Printer {

    override fun print(message: String) {
        delegate.print(message)
    }
}
```

与 Java 动态代理的区别：

| 对比                       | Kotlin `by`        | Java 动态代理        |
| -------------------------- | ------------------ | -------------------- |
| 实现机制                   | 编译期生成转发方法 | 运行时代理机制       |
| 是否需要 InvocationHandler | 不需要             | JDK 动态代理通常需要 |
| 是否可覆盖指定方法         | 可以               | 可以通过拦截逻辑实现 |
| 主要用途                   | 代码复用、组合     | AOP、动态拦截        |
| 能否自动拦截任意内部调用   | 不能               | 也有调用边界限制     |

官方文档：[Kotlin Delegation](https://kotlinlang.org/docs/delegation.html?utm_source=chatgpt.com)。

## 三、类委托可以重写指定方法

这是非常重要的能力。

```
interface MessageService {
    fun send(message: String)
    fun status(): String
}

class EmailMessageService : MessageService {

    override fun send(message: String) {
        println("邮件发送：$message")
    }

    override fun status(): String = "READY"
}

class AuditMessageService(
    private val target: MessageService
) : MessageService by target {

    override fun send(message: String) {
        println("[AUDIT] 准备发送")
        target.send(message)
    }
}

fun main() {
    val service = AuditMessageService(EmailMessageService())

    service.send("Hello Kotlin")
    println(service.status())
}
```

输出：

```
[AUDIT] 准备发送
邮件发送：Hello Kotlin
READY
```

这里有两种行为：

- `send()`：由 `AuditMessageService` 自己实现。
- `status()`：由编译器自动转发给 `target`。

需要特别注意：委托对象内部调用自己的方法时，不会自动调用外层类重写的方法。

例如：

```
interface Greeter {
    val message: String
    fun greet()
}

class DefaultGreeter : Greeter {
    override val message = "Hello"

    override fun greet() {
        println(message)
    }
}

class CustomGreeter(
    greeter: Greeter
) : Greeter by greeter {

    override val message = "你好"
}

fun main() {
    val greeter = CustomGreeter(DefaultGreeter())

    println(greeter.message) // 你好
    greeter.greet()          // Hello
}
```

为什么 `greet()` 输出 `Hello`？

因为调用被转发给了 `DefaultGreeter`，它读取的是自己的 `message`，而不是 `CustomGreeter.message`。

这是类委托与继承中的虚方法重写最容易混淆的地方。

## 四、多个接口委托

Kotlin 支持同时将不同接口委托给不同对象。

```
interface Reader {
    fun read(): String
}

interface Writer {
    fun write(text: String)
}

class FileReader : Reader {
    override fun read() = "文件内容"
}

class FileWriter : Writer {
    override fun write(text: String) {
        println("写入：$text")
    }
}

class FileManager(
    reader: Reader,
    writer: Writer
) : Reader by reader, Writer by writer

fun main() {
    val manager = FileManager(FileReader(), FileWriter())

    println(manager.read())
    manager.write("Kotlin Delegation")
}
```

输出：

```
文件内容
写入：Kotlin Delegation
```

这就是常说的 组合优于继承（Composition over Inheritance）。

与继承不同，`FileManager` 不需要成为 `FileReader` 或 `FileWriter` 的子类。

# 五、属性委托（Delegated Properties）

接下来是 Kotlin 委托的第二个核心部分。

语法：

```
val property: Type by delegate

var property: Type by delegate
```

这里的 `by` 不再是接口方法转发，而是将属性的访问行为交给委托对象。

它的核心机制是：

```
operator fun getValue(...)
operator fun setValue(...)
```

其中：

| 属性类型 | 必须提供                     |
| -------- | ---------------------------- |
| `val`    | `getValue()`                 |
| `var`    | `getValue()` 和 `setValue()` |

## 六、手写属性委托 Demo

先从最基础的例子开始。

```
import kotlin.reflect.KProperty

class StringDelegate {

    private var value = "默认值"

    operator fun getValue(
        thisRef: Any?,
        property: KProperty<*>
    ): String {
        println("读取属性：${property.name}")
        return value
    }

    operator fun setValue(
        thisRef: Any?,
        property: KProperty<*>,
        newValue: String
    ) {
        println("修改属性：${property.name} = $newValue")
        value = newValue
    }
}

class User {
    var name: String by StringDelegate()
}

fun main() {
    val user = User()

    println(user.name)

    user.name = "Tom"

    println(user.name)
}
```

输出：

```
读取属性：name
默认值
修改属性：name = Tom
读取属性：name
Tom
```

调用过程：

### 参数解释

```
operator fun getValue(
    thisRef: Any?,
    property: KProperty<*>
): String
```

`thisRef`：属性所属的对象。在例子中是 `User` 实例。

`property`：当前属性的元信息，例如：

```
property.name
```

可以获取属性名称 `name`。

`getValue()`：读取属性时调用。

`setValue()`：修改属性时调用。

这里的 `operator` 正是你之前学习的 运算符重载，因此委托和 Operator Overloading 是直接关联的。

## 七、属性委托的编译原理

例如：

```
class User {
    var name: String by StringDelegate()
}
```

编译器可以近似理解为生成：

```
class User {

    private val nameDelegate = StringDelegate()

    var name: String
        get() = nameDelegate.getValue(this, this::name)
        set(value) {
            nameDelegate.setValue(this, this::name, value)
        }
}
```

这是用于理解的简化示意，并非精确反编译结果；编译器通常会生成隐藏的委托字段和属性元数据，并对某些场景进行优化。

# 八、lazy：延迟初始化委托

`lazy` 是 Kotlin 开发中最常见的属性委托之一。

```
val value by lazy {
    println("初始化")
    "Hello Kotlin"
}
```

使用：

```
fun main() {
    println("程序启动")

    println(value)
    println(value)
}
```

输出：

```
程序启动
初始化
Hello Kotlin
Hello Kotlin
```

可以发现：

只有第一次读取属性时才执行初始化代码。

第二次读取直接返回缓存结果。

### 实战：延迟加载配置

```
data class AppConfig(
    val host: String,
    val port: Int
)

class ConfigService {

    val config: AppConfig by lazy {
        println("正在加载配置")

        AppConfig(
            host = "localhost",
            port = 8080
        )
    }
}

fun main() {
    val service = ConfigService()

    println("ConfigService 已创建")

    println(service.config.host)
    println(service.config.port)
}
```

输出：

```
ConfigService 已创建
正在加载配置
localhost
8080
```

`ConfigService` 创建时，并没有立即加载配置。

### lazy 的线程安全模式

Kotlin 提供三种模式：

| 模式           | 含义                                           | 适用场景       |
| -------------- | ---------------------------------------------- | -------------- |
| `SYNCHRONIZED` | 默认，初始化同步                               | 多线程共享     |
| `PUBLICATION`  | 允许多个线程竞争执行初始化，最终只发布一个结果 | 初始化允许重复 |
| `NONE`         | 不提供线程安全保证                             | 确定单线程访问 |

例如：

```
val config by lazy(LazyThreadSafetyMode.SYNCHRONIZED) {
    loadConfig()
}

fun loadConfig(): String {
    return "application.yml"
}
```

注意：`lazy` 的初始化线程安全，不代表初始化后对象内部的可变状态也是线程安全的。

与 C# 对比：

```
var config = new Lazy<AppConfig>(() => LoadConfig());
```

Kotlin 则是：

```
val config by lazy { loadConfig() }
```

语法更加简洁。

# 九、Delegates.observable：监听属性变化

例如开发用户资料管理功能，需要监听用户名的修改。

```
import kotlin.properties.Delegates

class User {

    var name: String by Delegates.observable("Guest") {
        property, oldValue, newValue ->

        println("属性：${property.name}")
        println("$oldValue -> $newValue")
    }
}

fun main() {
    val user = User()

    user.name = "Tom"
    user.name = "Jack"
}
```

输出：

```
属性：name
Guest -> Tom
属性：name
Tom -> Jack
```

这里：

```
Delegates.observable(initialValue) { property, old, new ->
    // ...
}
```

回调会在属性赋值完成后执行。

即使新旧值相同，只要发生赋值，也会调用回调。

### 实战：订单状态监听

```
import kotlin.properties.Delegates

enum class OrderStatus {
    CREATED,
    PAID,
    SHIPPED,
    COMPLETED
}

class Order {

    var status by Delegates.observable(OrderStatus.CREATED) {
        _, old, new ->

        if (old != new) {
            println("订单状态变化：$old -> $new")
        }
    }
}

fun main() {
    val order = Order()

    order.status = OrderStatus.PAID
    order.status = OrderStatus.SHIPPED
    order.status = OrderStatus.COMPLETED
}
```

输出：

```
订单状态变化：CREATED -> PAID
订单状态变化：PAID -> SHIPPED
订单状态变化：SHIPPED -> COMPLETED
```

这种写法适合简单的属性变化通知。

不过，生产环境中涉及订单状态流转、事务一致性、审计日志等复杂业务时，不应只依赖属性委托来保证业务正确性。

# 十、Delegates.vetoable：拦截属性修改

`observable` 是修改后通知。

`vetoable` 则可以在修改前判断是否允许赋值。

```
import kotlin.properties.Delegates

class Account {

    var balance: Int by Delegates.vetoable(100) {
        _, oldValue, newValue ->

        println("尝试修改余额：$oldValue -> $newValue")

        newValue >= 0
    }
}

fun main() {
    val account = Account()

    account.balance = 50
    println(account.balance)

    account.balance = -100
    println(account.balance)
}
```

输出：

```
尝试修改余额：100 -> 50
50
尝试修改余额：50 -> -100
50
```

因为：

```
newValue >= 0
```

当返回 `false` 时，赋值被拒绝。

两者区别：

| 特性         | observable | vetoable   |
| ------------ | ---------- | ---------- |
| 执行时机     | 修改后     | 修改前     |
| 可以拒绝修改 | 否         | 是         |
| 回调返回值   | `Unit`     | `Boolean`  |
| 常见用途     | 状态通知   | 属性值校验 |

# 十一、Map 属性委托

这是 Kotlin 非常实用的一个功能。

假设后端接口返回：

```
{
  "name": "Tom",
  "age": 30,
  "email": "tom@example.com"
}
```

通常需要将数据映射到 DTO。

Kotlin 可以直接通过 Map 委托：

```
class User(
    private val map: Map<String, Any?>
) {
    val name: String by map
    val age: Int by map
    val email: String by map
}

fun main() {
    val data = mapOf(
        "name" to "Tom",
        "age" to 30,
        "email" to "tom@example.com"
    )

    val user = User(data)

    println(user.name)
    println(user.age)
    println(user.email)
}
```

输出：

```
Tom
30
tom@example.com
```

这里的：

```
val name: String by map
```

会根据属性名 `name` 从 Map 中取值。

### MutableMap 支持写入

```
class Settings(
    private val data: MutableMap<String, Any?>
) {
    var theme: String by data
    var fontSize: Int by data
}

fun main() {
    val data = mutableMapOf<String, Any?>(
        "theme" to "light",
        "fontSize" to 14
    )

    val settings = Settings(data)

    settings.theme = "dark"
    settings.fontSize = 18

    println(data)
}
```

输出：

```
{theme=dark, fontSize=18}
```

需要注意，Map 委托不等于完整的 JSON 反序列化：

- 缺少必需键可能抛出异常。
- 值的类型不符合声明可能产生类型转换异常。
- 不会自动处理复杂的嵌套 DTO、日期转换或校验。

在 Spring Boot 项目中，正式的 JSON DTO 映射仍建议使用 Jackson、kotlinx.serialization 等序列化框架。

# 十二、属性委托给另一个属性

Kotlin 还支持属性引用委托：

```
class User {

    var username: String = "Tom"

    @Deprecated(
        "Use username instead",
        ReplaceWith("username")
    )
    var name: String by this::username
}

fun main() {
    val user = User()

    user.name = "Jack"

    println(user.username)
}
```

输出：

```
Jack
```

这在 API 重构时非常实用。

例如旧代码使用：

```
user.name
```

新代码改为：

```
user.username
```

你希望旧代码暂时继续运行，但提醒开发人员迁移。

使用属性委托可以避免维护两份状态。

# 十三、自定义泛型属性委托：校验属性值

前面的 `StringDelegate` 只能处理字符串。

实际开发可以利用泛型实现可复用的属性校验委托。

```
import kotlin.reflect.KProperty

class ValidatedProperty<T>(
    initialValue: T,
    private val validator: (T) -> Boolean
) {
    private var value: T = initialValue

    operator fun getValue(
        thisRef: Any?,
        property: KProperty<*>
    ): T {
        return value
    }

    operator fun setValue(
        thisRef: Any?,
        property: KProperty<*>,
        newValue: T
    ) {
        require(validator(newValue)) {
            "属性 ${property.name} 的值不合法：$newValue"
        }

        value = newValue
    }
}
```

定义业务类：

```
class Product {

    var price: Double by ValidatedProperty(0.0) {
        it >= 0
    }

    var stock: Int by ValidatedProperty(0) {
        it >= 0
    }
}
```

使用：

```
fun main() {
    val product = Product()

    product.price = 99.9
    product.stock = 100

    println(product.price)
    println(product.stock)

    try {
        product.stock = -10
    } catch (e: IllegalArgumentException) {
        println(e.message)
    }
}
```

输出：

```
99.9
100
属性 stock 的值不合法：-10
```

# 十四、使用 ReadOnlyProperty 和 ReadWriteProperty

Kotlin 标准库提供了两个接口，帮助我们规范自定义属性委托：

```
ReadOnlyProperty<in T, out V>
ReadWriteProperty<in T, V>
```

`ReadOnlyProperty` 负责读取。

`ReadWriteProperty` 继承 `ReadOnlyProperty`，同时提供写入能力。

例如：

```
import kotlin.properties.ReadWriteProperty
import kotlin.reflect.KProperty

class UppercaseDelegate(
    initialValue: String = ""
) : ReadWriteProperty<Any?, String> {

    private var value = initialValue

    override fun getValue(
        thisRef: Any?,
        property: KProperty<*>
    ): String {
        return value
    }

    override fun setValue(
        thisRef: Any?,
        property: KProperty<*>,
        value: String
    ) {
        this.value = value.uppercase()
    }
}

class UserProfile {
    var code: String by UppercaseDelegate()
}

fun main() {
    val user = UserProfile()

    user.code = "abc123"

    println(user.code)
}
```

输出：

```
ABC123
```

这种实现比直接定义 `operator fun getValue()`、`operator fun setValue()` 更容易统一接口和复用。

但 Kotlin 并不要求所有委托类必须实现这两个接口。只要提供符合要求的运算符函数即可。

# 十五、provideDelegate：高级委托机制

除了：

```
getValue()
setValue()
```

Kotlin 还有：

```
provideDelegate()
```

作用是：

在属性与委托对象建立绑定时，执行额外逻辑。

例如要求配置属性必须以 `app` 开头：

```
import kotlin.properties.ReadOnlyProperty
import kotlin.reflect.KProperty

class ConfigProvider {

    operator fun provideDelegate(
        thisRef: Any?,
        property: KProperty<*>
    ): ReadOnlyProperty<Any?, String> {

        require(property.name.startsWith("app")) {
            "配置属性必须以 app 开头"
        }

        return ReadOnlyProperty { _, prop ->
            "value-of-${prop.name}"
        }
    }
}

class AppSettings {
    val appName: String by ConfigProvider()
    val appVersion: String by ConfigProvider()
}

fun main() {
    val settings = AppSettings()

    println(settings.appName)
    println(settings.appVersion)
}
```

输出：

```
value-of-appName
value-of-appVersion
```

这里的执行顺序：

1. 创建 `AppSettings` 实例。
2. 为委托属性执行 `provideDelegate()`，完成绑定检查。
3. 第一次读取属性时调用 `getValue()`。

注意：`provideDelegate()` 不是每次读取属性都会调用，而是在建立委托关系时调用。

# 十六、Spring Boot 实战：通过委托实现缓存包装器

假设我们有一个用户查询服务。

接口：

```
data class UserDTO(
    val id: Long,
    val name: String
)

interface UserRepository {
    fun findById(id: Long): UserDTO?
    fun save(user: UserDTO)
}
```

数据库实现（这里用内存模拟）：

```
class DatabaseUserRepository : UserRepository {

    private val users = mutableMapOf<Long, UserDTO>()

    override fun findById(id: Long): UserDTO? {
        println("查询数据库：$id")
        return users[id]
    }

    override fun save(user: UserDTO) {
        println("保存数据库：${user.id}")
        users[user.id] = user
    }
}
```

现在希望添加缓存，但不修改原有 `DatabaseUserRepository`。

```
class CachedUserRepository(
    private val target: UserRepository
) : UserRepository by target {

    private val cache = mutableMapOf<Long, UserDTO>()

    override fun findById(id: Long): UserDTO? {
        cache[id]?.let {
            println("命中缓存：$id")
            return it
        }

        val user = target.findById(id)

        if (user != null) {
            cache[id] = user
        }

        return user
    }

    override fun save(user: UserDTO) {
        target.save(user)
        cache.remove(user.id)
    }
}
```

使用：

```
fun main() {
    val repository = CachedUserRepository(
        DatabaseUserRepository()
    )

    repository.save(UserDTO(1, "Tom"))

    println(repository.findById(1))
    println(repository.findById(1))
}
```

输出：

```
保存数据库：1
查询数据库：1
UserDTO(id=1, name=Tom)
命中缓存：1
UserDTO(id=1, name=Tom)
```

这里 `CachedUserRepository` 只重写了需要增强的两个方法。以后 `UserRepository` 增加其他方法时，符合委托规则的方法可以自动转发。

实际生产环境还需要考虑线程安全、缓存过期、缓存一致性、空值缓存等问题；这个 Demo 主要演示委托如何实现装饰器模式。

## 十七、委托、继承、代理和扩展函数的区别

| 特性               | 委托 `by`  | 继承     | 动态代理     | 扩展函数     |
| ------------------ | ---------- | -------- | ------------ | ------------ |
| 代码复用           | 是         | 是       | 是           | 是           |
| 需要继承父类       | 否         | 是       | 否           | 否           |
| 可包装接口实现     | 是         | 有条件   | 是           | 否           |
| 自动转发接口方法   | 是         | 不需要   | 通过代理处理 | 否           |
| 能增加实际成员状态 | 包装类可以 | 可以     | 代理对象可以 | 不可以       |
| 主要用途           | 组合、转发 | 类型层次 | 运行时拦截   | 增加便捷 API |

还有一个需要区分的概念：

Kotlin 的类委托 `by` 与 Spring 的 `@Autowired`、`@Transactional` 不是同一回事。

`by` 负责生成接口方法转发；Spring DI 负责创建、管理和注入对象，Spring AOP 则可以负责方法拦截。它们可以组合使用，但不能互相直接替代。

# 十八、常见问题和注意事项

1\. `by` 后面必须是对象吗？

类委托的 `by` 后面需要一个能够提供所实现接口的表达式：

```
class Service(
    repository: Repository
) : Repository by repository
```

属性委托的 `by` 后面则需要提供符合委托运算符约定的表达式：

```
val name by lazy { "Tom" }
```

2\. 委托是否会复制对象？

不会因为使用 `by` 就自动深拷贝委托对象。通常是持有委托对象引用，并将调用转发过去。

3\. `val` 委托一定不能改变值吗？

`val` 代表不能通过这个属性赋值，不意味着委托对象返回的内容永远不变。

例如，一个 `val` 的 `getValue()` 每次都可以计算并返回不同的结果。

4\. `lazy` 和 `lateinit` 有什么区别？

| 特性         | `lazy`         | `lateinit`         |
| ------------ | -------------- | ------------------ |
| 常见声明     | `val`          | `var`              |
| 初始化方式   | 首次读取时计算 | 后续手动赋值       |
| 自动缓存     | 是             | 不涉及懒计算       |
| 未初始化读取 | 会触发初始化   | 抛出异常           |
| 典型用途     | 配置、昂贵对象 | 框架注入、延迟赋值 |

5\. `Delegates.observable` 是线程安全的吗？

不能把它当作线程安全的状态容器。需要多线程协调时，应该额外使用同步机制、原子类型，或者适合协程的状态管理工具。

# 十九、知识总结

| 委托类型        | 语法                             | 主要用途             | 重要程度 |
| --------------- | -------------------------------- | -------------------- | -------- |
| 类委托          | `Interface by obj`               | 接口转发、装饰器     | 高       |
| 自定义属性委托  | `var x by Delegate()`            | 统一读写逻辑         | 高       |
| lazy            | `val x by lazy {}`               | 延迟初始化           | 很高     |
| observable      | `Delegates.observable()`         | 监听属性变化         | 高       |
| vetoable        | `Delegates.vetoable()`           | 拦截属性赋值         | 中       |
| Map 委托        | `val x by map`                   | 动态属性映射         | 中       |
| 属性引用委托    | `by this::other`                 | 属性迁移、兼容旧 API | 中       |
| provideDelegate | `operator fun provideDelegate()` | 属性绑定检查         | 进阶     |

你可以把 Kotlin 委托总结为两个公式：

类委托：

```
class A(b: B) : B by b
```

含义是：

> A 实现 B 接口，但把未重写的接口成员实现交给 b。

属性委托：

```
var value by delegate
```

含义是：

> value 的读取与赋值行为由 delegate 的 getValue/setValue 负责。

类比：Java 和 C#：

- 类委托：类似装饰器模式 + 编译器自动生成转发代码。
- 属性委托：类似可复用的属性 getter/setter 逻辑，但比普通属性访问器更容易抽象和复用。

官方参考：

[Kotlin Delegation — 类委托](https://kotlinlang.org/docs/delegation.html?utm_source=chatgpt.com)[Kotlin Delegated Properties — 属性委托](https://kotlinlang.org/docs/delegated-properties.html?utm_source=chatgpt.com)