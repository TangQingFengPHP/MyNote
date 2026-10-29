# Kotlin Object Declarations and Expressions 详解与实战 Demo

Kotlin 的 Object Declarations and Expressions（对象声明与对象表达式） 是非常重要的语言特性，主要用于实现单例、匿名对象、伴生对象，以及临时的接口实现。

它与 Java、C# 中的 `static`、匿名内部类、单例模式有相似之处，但 Kotlin 的设计更加简洁。

官方文档：[Object declarations and expressions](https://kotlinlang.org/docs/object-declarations.html?utm_source=chatgpt.com)

本篇主要讲解：

1. `object declaration`：对象声明（单例）
2. `object expression`：对象表达式（匿名对象）
3. `companion object`：伴生对象
4. `data object`：数据对象
5. 对象初始化与线程安全
6. Java、C# 与 Kotlin 的区别
7. Spring Boot、策略模式、事件监听等实战场景

## 一、先理解 object 是什么

在 Kotlin 中，`object` 主要有三种使用形式。

| 语法                    | 名称         | 主要作用               |
| ----------------------- | ------------ | ---------------------- |
| `object Name {}`        | 对象声明     | 创建单例               |
| `object : Interface {}` | 对象表达式   | 创建匿名对象           |
| `companion object {}`   | 伴生对象     | 定义与类关联的对象成员 |
| `data object Name`      | 数据对象声明 | 带有数据对象语义的单例 |

最直观的区别：

```
// 1. 对象声明
object Logger {
    fun log(message: String) {
        println(message)
    }
}

// 2. 对象表达式
val callback = object : Runnable {
    override fun run() {
        println("Running")
    }
}

// 3. 伴生对象
class User {
    companion object {
        fun create(): User = User()
    }
}

// 4. 数据对象
data object Loading
```

调用：

```
Logger.log("Hello")

callback.run()

val user = User.create()

println(Loading)
```

需要注意：

`object` 不是一种普通的类实例化语法，而是 Kotlin 专门提供的对象构造机制。

# 二、Object Declaration：对象声明

对象声明最典型的用途是实现单例模式（Singleton Pattern）。

### 1. 最基本的单例

```
object AppConfig {

    val appName = "MyApplication"

    val port = 8080

    fun printConfig() {
        println("App: $appName")
        println("Port: $port")
    }
}

fun main() {
    println(AppConfig.appName)
    AppConfig.printConfig()
}
```

输出：

```
MyApplication
App: MyApplication
Port: 8080
```

这里：

```
object AppConfig
```

定义的是一个对象，而不是一个需要通过构造函数创建实例的普通类。

因此：

```
AppConfig.printConfig()
```

可以直接调用。

但不能：

```
// 编译错误
val config = AppConfig()
```

因为对象声明没有可供调用的构造函数。

### 2. Object Declaration 的本质

Java 中实现单例通常需要：

```
public final class AppConfig {

    private static final AppConfig INSTANCE = new AppConfig();

    private AppConfig() {
    }

    public static AppConfig getInstance() {
        return INSTANCE;
    }

    public void printConfig() {
        System.out.println("Config");
    }
}
```

Kotlin：

```
object AppConfig {

    fun printConfig() {
        println("Config")
    }
}
```

编译到 JVM 时，普通顶层对象声明通常会生成一个类和用于访问单例实例的静态字段。

概念上类似：

```
public final class AppConfig {

    public static final AppConfig INSTANCE = new AppConfig();

    private AppConfig() {
    }

    public void printConfig() {
        System.out.println("Config");
    }
}
```

这只是简化后的原理示意，实际字节码还涉及初始化方法、访问器和编译器优化。

### 3. 对象声明的引用相同

```
object CacheManager {

    var count = 0
}

fun main() {
    val a = CacheManager
    val b = CacheManager

    a.count++

    println(b.count)
    println(a === b)
}
```

输出：

```
1
true
```

因为 `a` 和 `b` 引用同一个对象。

### 4. 对象声明可以实现接口

```
interface Logger {
    fun log(message: String)
}

object ConsoleLogger : Logger {

    override fun log(message: String) {
        println("[LOG] $message")
    }
}

fun main() {
    val logger: Logger = ConsoleLogger

    logger.log("Hello Kotlin")
}
```

输出：

```
[LOG] Hello Kotlin
```

这里的 `ConsoleLogger` 既是单例，又实现了 `Logger` 接口。

### 5. 对象声明可以继承类

```
open class BaseHandler {

    open fun handle() {
        println("Base handle")
    }
}

object DefaultHandler : BaseHandler() {

    override fun handle() {
        println("Default handle")
    }
}

fun main() {
    DefaultHandler.handle()
}
```

输出：

```
Default handle
```

但对象声明不能声明主构造函数或次构造函数，也不能在每次访问时重新创建实例。

# 三、Object Declaration 的初始化时机

对象声明是延迟初始化的。

通常在第一次访问对象时初始化。

```
object DatabaseConfig {

    init {
        println("DatabaseConfig 初始化")
    }

    val host = "localhost"

    fun connect() {
        println("连接数据库：$host")
    }
}

fun main() {
    println("程序启动")

    DatabaseConfig.connect()

    DatabaseConfig.connect()
}
```

输出：

```
程序启动
DatabaseConfig 初始化
连接数据库：localhost
连接数据库：localhost
```

可以发现：

- 程序启动时，并没有立即初始化 `DatabaseConfig`。
- 第一次访问时，执行 `init`。
- 后续访问不会重复执行初始化。

对象声明的初始化是线程安全的。 但这并不意味着对象内部的可变属性自动具备线程安全性。

例如：

```
object Counter {
    var count = 0

    fun increment() {
        count++
    }
}
```

`count++` 不是原子操作。

多个线程并发调用时仍然可能发生数据竞争。

如果需要线程安全计数，可以使用：

```
import java.util.concurrent.atomic.AtomicInteger

object Counter {

    private val count = AtomicInteger(0)

    fun increment(): Int {
        return count.incrementAndGet()
    }

    fun current(): Int {
        return count.get()
    }
}
```

# 四、Object Expression：对象表达式

对象表达式用于创建匿名对象。

与对象声明最大的区别是：

对象表达式每次执行都会创建一个新的对象实例，而不是复用单例。

### 1. 最简单的匿名对象

```
fun main() {

    val user = object {
        val name = "Tom"
        val age = 30

        fun hello() {
            println("Hello $name")
        }
    }

    println(user.name)
    println(user.age)

    user.hello()
}
```

输出：

```
Tom
30
Hello Tom
```

这里没有定义：

```
class User
```

而是直接创建一个带有属性和方法的匿名对象。

可以理解为：

```
val user = object {
    // 匿名类的对象
}
```

与 Java 匿名内部类不同，Kotlin 的匿名对象不一定需要继承类或实现接口。

### 2. 每次创建不同实例

```
fun createUser() = object {
    val name = "Tom"
}

fun main() {
    val a = createUser()
    val b = createUser()

    println(a === b)
}
```

输出：

```
false
```

每次调用 `createUser()` 都创建新的匿名对象。

不过，这里还有一个重要的可见性问题，后面会专门讲解。

# 五、对象表达式实现接口

这是 Kotlin Object Expression 最常见的用途。

例如定义一个事件监听器：

```
interface EventListener {

    fun onSuccess(message: String)

    fun onError(error: String)
}
```

使用匿名对象实现：

```
fun main() {

    val listener = object : EventListener {

        override fun onSuccess(message: String) {
            println("成功：$message")
        }

        override fun onError(error: String) {
            println("失败：$error")
        }
    }

    listener.onSuccess("请求完成")
    listener.onError("网络异常")
}
```

输出：

```
成功：请求完成
失败：网络异常
```

这里：

```
object : EventListener
```

表示创建一个匿名对象，并实现 `EventListener` 接口。

注意：

```
object EventListenerImpl : EventListener
```

这是对象声明，创建单例。

而：

```
object : EventListener
```

这是对象表达式，创建匿名对象。

两者不能混淆。

# 六、匿名对象继承类

对象表达式还可以继承普通类或抽象类。

```
open class Animal(
    val name: String
) {

    open fun speak() {
        println("$name speak")
    }
}
```

创建匿名子类：

```
fun main() {

    val animal = object : Animal("Dog") {

        override fun speak() {
            println("$name bark")
        }
    }

    animal.speak()
}
```

输出：

```
Dog bark
```

这里：

```
object : Animal("Dog")
```

相当于：

1. 定义一个匿名子类。
2. 调用父类构造函数。
3. 创建该匿名子类的实例。

### 同时实现多个接口

```
interface Flyable {
    fun fly()
}

interface Swimmable {
    fun swim()
}

fun main() {

    val duck = object : Flyable, Swimmable {

        override fun fly() {
            println("飞行")
        }

        override fun swim() {
            println("游泳")
        }
    }

    duck.fly()
    duck.swim()
}
```

输出：

```
飞行
游泳
```

Kotlin 匿名对象也遵循普通类的继承规则：最多继承一个类，但可以实现多个接口。

# 七、匿名对象捕获外部变量

这一点与 Java 匿名内部类存在明显区别。

Kotlin 的匿名对象可以访问外部局部变量，并且可以修改被捕获的 `var`。

```
fun main() {

    var count = 0

    val counter = object {

        fun increment() {
            count++
        }

        fun current(): Int {
            return count
        }
    }

    counter.increment()
    counter.increment()
    counter.increment()

    println(counter.current())
}
```

输出：

```
3
```

Java 中，匿名内部类捕获的局部变量必须是 `final` 或 effectively final，因此不能直接修改被捕获的普通局部变量。

Kotlin 允许修改捕获的 `var`，编译器会根据目标平台采用适当机制保存和访问变量。

但是，这并不自动提供线程安全保证。

# 八、匿名对象的类型与可见性（重点）

这是 Kotlin 官方文档中非常重要、也很容易忽略的一部分。

考虑：

```
class UserService {

    private fun createPrivateObject() = object {
        val name = "Tom"
        val age = 30
    }

    fun test() {
        val user = createPrivateObject()

        println(user.name)
        println(user.age)
    }
}
```

因为 `createPrivateObject()` 是 `private`，匿名对象的具体类型可以在相应可见范围内保留。

因此可以访问：

```
user.name
user.age
```

但是如果把方法改成公开的：

```
class UserService {

    fun createUser() = object {
        val name = "Tom"
        val age = 30
    }
}
```

外部代码：

```
val user = UserService().createUser()

// 编译错误
println(user.name)
```

为什么？

因为公开函数不能将这个匿名对象的完整匿名类型作为公开 API 暴露。

在没有显式父类型的情况下，公开返回类型会被视为 `Any`。

所以：

```
user.name
```

无法编译。

### 正确方式：定义明确的接口或数据类

```
interface UserInfo {
    val name: String
    val age: Int
}

class UserService {

    fun createUser(): UserInfo {
        return object : UserInfo {

            override val name = "Tom"

            override val age = 30
        }
    }
}

fun main() {
    val user = UserService().createUser()

    println(user.name)
    println(user.age)
}
```

输出：

```
Tom
30
```

实际项目中，公开 API 通常应该使用明确的接口、类或 DTO，而不是依赖匿名对象的结构。

# 九、Companion Object：伴生对象

Kotlin 没有 Java/C# 那样的普通 `static` 成员声明机制。

但是可以使用伴生对象定义与类关联的成员。

### 1. 基本使用

```
class User(
    val name: String
) {

    companion object {

        fun create(name: String): User {
            return User(name)
        }
    }
}

fun main() {
    val user = User.create("Tom")

    println(user.name)
}
```

输出：

```
Tom
```

这里：

```
User.create("Tom")
```

看起来像 Java 的静态工厂方法：

```
User.create("Tom");
```

但 Kotlin 中的 `create()` 实际属于 `User` 的伴生对象。

### 2. 伴生对象可以命名

```
class User {

    companion object Factory {

        fun create(): User {
            return User()
        }
    }
}

fun main() {
    val a = User.create()

    val b = User.Factory.create()

    println(a === b)
}
```

输出：

```
false
```

因为 `create()` 每次返回新的 `User` 实例。

`User.Factory` 本身则是同一个伴生对象。

### 3. 伴生对象可以实现接口

```
interface Factory<T> {
    fun create(): T
}

class User(
    val name: String
) {

    companion object : Factory<User> {

        override fun create(): User {
            return User("Guest")
        }
    }
}

fun main() {
    val factory: Factory<User> = User

    val user = factory.create()

    println(user.name)
}
```

输出：

```
Guest
```

这里：

```
val factory: Factory<User> = User
```

是合法的。

因为 `User` 在表达式位置可以引用其伴生对象，而伴生对象实现了 `Factory<User>`。

这说明：

Companion Object 是真实的对象，不只是 static 语法糖。

# 十、伴生对象与 Java static 的区别

| 特性               | Kotlin Companion Object | Java static              |
| ------------------ | ----------------------- | ------------------------ |
| 属于什么           | 伴生对象实例            | 类级成员                 |
| 能实现接口         | 可以                    | 静态成员本身不可以       |
| 可以作为对象传递   | 可以                    | 静态成员不能作为对象实例 |
| 调用形式           | `User.create()`         | `User.create()`          |
| 每个类可以定义几个 | 最多一个伴生对象        | 不适用                   |
| 是否支持扩展函数   | 支持                    | 不支持相同机制           |

例如伴生对象扩展：

```
class User {

    companion object
}

fun User.Companion.createGuest(): User {
    return User()
}

fun main() {
    val user = User.createGuest()

    println(user)
}
```

这与你之前学习的 Kotlin Extensions 正好结合起来。

### Java 互操作：@JvmStatic

如果希望 Java 调用方直接使用：

```
User.create();
```

可以使用：

```
class User {

    companion object {

        @JvmStatic
        fun create(): User {
            return User()
        }
    }
}
```

否则，Java 通常通过：

```
User.Companion.create();
```

调用伴生对象方法。

`@JvmStatic` 会生成额外的 JVM 静态访问入口，但并不会把伴生对象本身变成 Java 静态类。

# 十一、Data Object：数据对象

Kotlin 还支持：

```
data object
```

它特别适合表示只有一个实例的状态。

例如：

```
sealed interface ApiResult {

    data class Success(
        val data: String
    ) : ApiResult

    data class Error(
        val message: String
    ) : ApiResult

    data object Loading : ApiResult
}
```

调用：

```
fun handle(result: ApiResult) {

    when (result) {
        is ApiResult.Success -> {
            println("成功：${result.data}")
        }

        is ApiResult.Error -> {
            println("失败：${result.message}")
        }

        ApiResult.Loading -> {
            println("加载中")
        }
    }
}

fun main() {
    handle(ApiResult.Loading)
    handle(ApiResult.Success("Hello"))
}
```

输出：

```
加载中
成功：Hello
```

### data object 与 object 的区别

```
object NormalState

data object DataState
```

测试：

```
fun main() {
    println(NormalState)
    println(DataState)
}
```

典型输出：

```
NormalState@某个哈希值
DataState
```

`data object` 会自动提供更适合数据对象的 `toString()`、`equals()` 和 `hashCode()` 行为。

其中 `equals()` 的设计能够让同一种数据对象类型的实例被视为相等，即使在某些反射等特殊场景中出现额外实例。

但 `data object` 不会生成：

```
copy()
component1()
```

因为它不包含需要复制或解构的主构造属性。

# 十二、实战 Demo：事件监听器

对象表达式在事件处理场景中非常常见。

```
interface DownloadListener {

    fun onStart()

    fun onProgress(progress: Int)

    fun onCompleted(fileName: String)

    fun onError(message: String)
}
```

定义下载器：

```
class Downloader {

    fun download(
        fileName: String,
        listener: DownloadListener
    ) {
        listener.onStart()

        for (progress in listOf(25, 50, 75, 100)) {
            listener.onProgress(progress)
        }

        listener.onCompleted(fileName)
    }
}
```

调用：

```
fun main() {

    val downloader = Downloader()

    downloader.download(
        "report.pdf",
        object : DownloadListener {

            override fun onStart() {
                println("开始下载")
            }

            override fun onProgress(progress: Int) {
                println("进度：$progress%")
            }

            override fun onCompleted(fileName: String) {
                println("下载完成：$fileName")
            }

            override fun onError(message: String) {
                println("下载失败：$message")
            }
        }
    )
}
```

输出：

```
开始下载
进度：25%
进度：50%
进度：75%
进度：100%
下载完成：report.pdf
```

这种方式的优势是：

不需要专门创建一个 `DownloadListenerImpl` 类。

适合一次性、局部使用的接口实现。

但如果监听器逻辑较复杂，或者需要多处复用，建议使用具名类。

# 十三、Object Expression 与 fun interface

例如：

```
fun interface Callback {
    fun onSuccess(message: String)
}
```

使用对象表达式：

```
val callback = object : Callback {

    override fun onSuccess(message: String) {
        println(message)
    }
}
```

可以简化为：

```
val callback = Callback { message ->
    println(message)
}
```

两者都能实现接口。

但只有符合函数式接口规则的接口，才能使用这种 SAM 转换语法。

例如一个接口有两个抽象方法：

```
interface Listener {
    fun onSuccess()
    fun onError()
}
```

就不能直接写成单个 Lambda。

需要使用对象表达式：

```
val listener = object : Listener {

    override fun onSuccess() {
        println("Success")
    }

    override fun onError() {
        println("Error")
    }
}
```

# 十四、Spring Boot 实战：对象声明与策略模式

假设开发一个订单折扣系统。

定义接口：

```
interface DiscountStrategy {

    fun calculate(
        originalPrice: Double
    ): Double
}
```

普通用户策略：

```
object NormalDiscount : DiscountStrategy {

    override fun calculate(originalPrice: Double): Double {
        return originalPrice
    }
}
```

VIP 用户策略：

```
object VipDiscount : DiscountStrategy {

    override fun calculate(originalPrice: Double): Double {
        return originalPrice * 0.9
    }
}
```

定义服务：

```
class OrderService(
    private val discountStrategy: DiscountStrategy
) {

    fun checkout(price: Double): Double {
        return discountStrategy.calculate(price)
    }
}
```

使用：

```
fun main() {

    val normalService = OrderService(NormalDiscount)

    val vipService = OrderService(VipDiscount)

    println(normalService.checkout(100.0))

    println(vipService.checkout(100.0))
}
```

输出：

```
100.0
90.0
```

为什么这里适合 `object`？

因为折扣策略本身没有每个请求独立的状态。

因此可以共享一个策略对象。

但需要注意：

如果策略需要注入数据库仓储、Redis 客户端、配置服务等依赖，Spring Boot 项目中通常更适合使用由 Spring 容器管理的 `@Component` / `@Service` Bean，而不是直接使用全局 `object` 保存依赖。

# 十五、实战 Demo：伴生对象工厂方法

伴生对象很适合创建复杂对象。

例如：

```
data class DatabaseConfig(
    val host: String,
    val port: Int,
    val username: String,
    val password: String
) {

    companion object {

        fun development(): DatabaseConfig {
            return DatabaseConfig(
                host = "localhost",
                port = 3306,
                username = "root",
                password = "dev-password"
            )
        }

        fun production(
            host: String,
            username: String,
            password: String
        ): DatabaseConfig {
            return DatabaseConfig(
                host = host,
                port = 3306,
                username = username,
                password = password
            )
        }
    }
}
```

调用：

```
fun main() {

    val dev = DatabaseConfig.development()

    val prod = DatabaseConfig.production(
        host = "mysql.example.com",
        username = "app",
        password = "secret"
    )

    println(dev.host)
    println(prod.host)
}
```

输出：

```
localhost
mysql.example.com
```

这个例子只是展示工厂方法；生产环境不要把数据库密码硬编码在源码中，应通过环境变量或密钥管理服务提供。

# 十六、object、class、data class、data object 的区别

| 类型                 | 是否可创建多个实例     | 是否支持构造参数   | 典型用途             |
| -------------------- | ---------------------- | ------------------ | -------------------- |
| `class`              | 可以                   | 可以               | 普通业务对象         |
| `data class`         | 可以                   | 可以               | DTO、数据模型        |
| `object`             | 单例                   | 不支持构造函数     | 无状态服务、常量集合 |
| `data object`        | 单例语义               | 不支持构造函数     | 状态枚举式对象       |
| `companion object`   | 每个所属类一个伴生对象 | 不支持构造函数     | 工厂方法、类级 API   |
| `object : Interface` | 每次求值创建对象       | 可调用父类构造函数 | 临时接口实现         |

## 十八、总结

Kotlin Object Declarations and Expressions 可以总结为四种最常见的写法：

```
// 1. 单例对象
object Logger {
    fun log(message: String) = println(message)
}

// 2. 匿名对象
val runnable = object : Runnable {
    override fun run() {
        println("Running")
    }
}

// 3. 伴生对象
class User {
    companion object {
        fun create() = User()
    }
}

// 4. 数据对象
data object Loading
```
对于 Kotlin 后端开发，最常用的组合通常是：

```
Interface + Object Expression
    → 临时策略和回调实现

Companion Object + Factory Method
    → 创建业务对象和 DTO

Sealed Interface + Data Object
    → 业务结果和状态建模

Object Declaration
    → 无状态共享对象
```