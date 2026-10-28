# Kotlin Inheritance（继承）详解：语法、原理、实战 Demo

Kotlin 的继承（Inheritance）和 Java、C# 的面向对象继承思想基本一致，但在语法和设计理念上存在几个重要区别。

其中最核心的一点是：

Kotlin 的类和成员默认都是 `final`，必须显式使用 `open` 才能允许继承或重写。

例如：

```
open class Animal {
    open fun speak() {
        println("动物发出声音")
    }
}

class Dog : Animal() {
    override fun speak() {
        println("汪汪")
    }
}
```

本篇结合 Kotlin 官方文档，详细讲解继承的语法、构造函数、方法重写、属性重写、多态、抽象类、多继承冲突、初始化顺序，以及 Spring Boot 实战。

官方文档：[Kotlin Inheritance](https://kotlinlang.org/docs/inheritance.html?utm_source=chatgpt.com)

## 一、Kotlin 继承的基本概念

继承允许一个类复用另一个类的可继承成员，并在此基础上扩展或修改行为。

例如，我们有一个动物类：

```
open class Animal {
    fun eat() {
        println("动物正在吃东西")
    }
}
```

定义子类：

```
class Dog : Animal() {
    fun bark() {
        println("汪汪")
    }
}
```

调用：

```
fun main() {
    val dog = Dog()

    dog.eat()
    dog.bark()
}
```

输出：

```
动物正在吃东西
汪汪
```

这里的继承关系是：

在 Kotlin 中：

- `Animal` 是父类（Superclass）。
- `Dog` 是子类（Subclass）。
- `open` 表示允许继承。
- `:` 表示继承或实现接口。
- `Animal()` 表示调用父类构造函数。

与 Java 不同，Kotlin 不使用 `extends` 关键字。

## 二、为什么 Kotlin 默认禁止继承？

先看下面的代码：

```
class Animal {
    fun eat() {
        println("eat")
    }
}

// 编译错误
class Dog : Animal()
```

原因是 Kotlin 的普通类默认不可继承。

必须修改为：

```
open class Animal {
    fun eat() {
        println("eat")
    }
}

class Dog : Animal()
```

Java、C#、Kotlin 对比：

| 特性           | Kotlin         | Java                  | C#                     |
| -------------- | -------------- | --------------------- | ---------------------- |
| 类默认能否继承 | 否             | 可以                  | 可以                   |
| 允许继承       | `open class`   | 默认允许              | 默认允许               |
| 禁止继承       | 默认或 `final` | `final`               | `sealed`               |
| 方法默认可重写 | 否             | 通常可以              | 否                     |
| 允许方法重写   | `open`         | 非 `final` 的实例方法 | `virtual` / `abstract` |
| 重写方法       | `override`     | `@Override`           | `override`             |

Kotlin 采用的是 默认封闭，显式开放 的设计思想。

好处是开发者必须明确哪些类、哪些成员属于可扩展的 API，减少无意继承和重写造成的问题。

## 三、open、override、final 关键字

这是 Kotlin 继承最重要的三个关键字。

### 1. open：允许重写

```
open class Animal {

    open fun speak() {
        println("Animal speak")
    }

    fun eat() {
        println("Animal eat")
    }
}
```

其中：

```
open fun speak()
```

可以被子类重写。

但：

```
fun eat()
```

默认不能重写。

### 2. override：重写父类方法

```
class Dog : Animal() {

    override fun speak() {
        println("Dog bark")
    }
}
```

调用：

```
fun main() {
    val dog = Dog()

    dog.speak()
    dog.eat()
}
```

输出：

```
Dog bark
Animal eat
```

### 3. final：禁止进一步重写

需要注意一个细节：

Kotlin 中，被 `override` 重写的方法默认仍然允许继续重写。

例如：

```
open class Animal {
    open fun speak() {
        println("Animal")
    }
}

open class Dog : Animal() {
    override fun speak() {
        println("Dog")
    }
}

class Husky : Dog() {
    override fun speak() {
        println("Husky")
    }
}
```

如果希望 `Dog.speak()` 不再允许子类重写：

```
open class Dog : Animal() {

    final override fun speak() {
        println("Dog")
    }
}
```

此时：

```
class Husky : Dog() {

    // 编译错误
    override fun speak() {
        println("Husky")
    }
}
```

因为 `Dog.speak()` 已经被声明为 `final`。

## 四、继承与构造函数

Kotlin 的构造函数与 Java、C# 不太一样。

Kotlin 支持：

- 主构造函数（Primary Constructor）。
- 次构造函数（Secondary Constructor）。

继承时必须正确初始化父类。

### 1. 父类有主构造函数

```
open class Animal(
    val name: String,
    val age: Int
) {
    fun info() {
        println("name=$name, age=$age")
    }
}
```

子类：

```
class Dog(
    name: String,
    age: Int,
    val breed: String
) : Animal(name, age)
```

使用：

```
fun main() {
    val dog = Dog(
        name = "旺财",
        age = 3,
        breed = "柴犬"
    )

    dog.info()
    println(dog.breed)
}
```

输出：

```
name=旺财, age=3
柴犬
```

重点：

```
class Dog(
    name: String,
    age: Int,
    val breed: String
) : Animal(name, age)
```

`name`、`age` 是传给父类的构造参数。

因为它们没有在 `Dog` 构造参数前使用 `val` 或 `var`，所以不会为 `Dog` 额外声明同名属性。

父类已经声明了：

```
val name: String
val age: Int
```

子类可以直接使用继承来的属性。

### 2. 子类没有主构造函数

如果子类只有次构造函数，可以使用 `super(...)` 调用父类构造函数。

```
open class Person(
    val name: String
)

class Student : Person {

    val score: Int

    constructor(name: String, score: Int) : super(name) {
        this.score = score
    }
}

fun main() {
    val student = Student("Tom", 95)

    println(student.name)
    println(student.score)
}
```

输出：

```
Tom
95
```

这里：

```
constructor(name: String, score: Int) : super(name)
```

表示在次构造函数中初始化父类。

规则：如果子类声明了主构造函数，父类的主构造函数调用通常写在类头部；如果子类没有主构造函数，则每个次构造函数必须直接或间接完成父类初始化。

## 五、super：调用父类方法

子类重写父类方法时，有时候希望保留父类原有逻辑。

```
open class Animal {

    open fun speak() {
        println("Animal speak")
    }
}

class Dog : Animal() {

    override fun speak() {
        super.speak()
        println("Dog bark")
    }
}

fun main() {
    Dog().speak()
}
```

输出：

```
Animal speak
Dog bark
```

这里：

```
super.speak()
```

调用父类的实现。

### 实战：统一日志处理

```
open class BaseService {

    open fun execute() {
        println("基础业务逻辑")
    }
}

class OrderService : BaseService() {

    override fun execute() {
        println("开始处理订单")

        super.execute()

        println("订单处理完成")
    }
}
```

这种写法在需要扩展父类行为时很常见。

不过，在复杂业务系统中，如果父类承担过多逻辑，可能会造成继承层级过深，因此不应为了复用几行代码就建立继承关系。

## 六、属性重写（Property Overriding）

Kotlin 不仅可以重写方法，还可以重写属性。

### 1. 重写 val 属性

```
open class Animal {

    open val name: String = "Animal"
}

class Dog : Animal() {

    override val name: String = "Dog"
}

fun main() {
    val animal: Animal = Dog()

    println(animal.name)
}
```

输出：

```
Dog
```

这里虽然变量声明为：

```
val animal: Animal
```

但实际对象是 `Dog`，访问开放属性时会发生动态分派。

### 2. 使用 getter 重写

```
open class Shape {

    open val area: Double
        get() = 0.0
}

class Rectangle(
    val width: Double,
    val height: Double
) : Shape() {

    override val area: Double
        get() = width * height
}

fun main() {
    val shape: Shape = Rectangle(10.0, 5.0)

    println(shape.area)
}
```

输出：

```
50.0
```

### 3. 可以用 var 重写 val 吗？

可以。

```
open class User {

    open val name: String = "Guest"
}

class Admin : User() {

    override var name: String = "Admin"
}
```

原因是：

- `val` 只要求提供读取能力。
- `var` 同时提供读取和写入能力。

因此，`var` 可以满足 `val` 的契约。

反过来则不行：

```
open class User {
    open var name: String = "Guest"
}

class Admin : User() {

    // 编译错误
    override val name: String = "Admin"
}
```

因为父类要求可写，而子类却移除了写入能力。

## 七、继承与多态（Polymorphism）

这是面向对象编程的核心。

假设有一个支付系统：

```
open class Payment {

    open fun pay(amount: Double) {
        println("支付金额：$amount")
    }
}
```

支付宝：

```
class AliPay : Payment() {

    override fun pay(amount: Double) {
        println("支付宝支付：$amount")
    }
}
```

微信支付：

```
class WeChatPay : Payment() {

    override fun pay(amount: Double) {
        println("微信支付：$amount")
    }
}
```

业务方法：

```
fun checkout(payment: Payment, amount: Double) {
    payment.pay(amount)
}
```

调用：

```
fun main() {
    checkout(AliPay(), 100.0)
    checkout(WeChatPay(), 200.0)
}
```

输出：

```
支付宝支付：100.0
微信支付：200.0
```

`checkout()` 并不关心实际对象是 `AliPay` 还是 `WeChatPay`。

运行时根据对象实际类型调用对应的 `pay()`。

这就是动态分派（Dynamic Dispatch）。

不过，对于支付能力这样的业务抽象，实际开发中通常更适合使用 `interface Payment`，而不是继承一个带有默认业务实现的普通类。

## 八、抽象类（Abstract Class）

抽象类不能直接实例化，可以定义抽象方法和普通方法。

```
abstract class Animal {

    abstract fun speak()

    fun sleep() {
        println("动物正在睡觉")
    }
}
```

子类：

```
class Cat : Animal() {

    override fun speak() {
        println("喵喵")
    }
}
```

调用：

```
fun main() {
    val cat = Cat()

    cat.speak()
    cat.sleep()
}
```

输出：

```
喵喵
动物正在睡觉
```

抽象类具有以下特点：

- 不能直接实例化。
- 可以有构造函数。
- 可以有成员变量。
- 可以有普通方法和抽象方法。
- 抽象成员不需要额外声明 `open`。
- 非抽象成员仍然遵循 Kotlin 的 `open` / `final` 规则。

### 实战：模板方法模式

假设开发一个数据导入系统，支持 CSV、Excel 等文件。

导入过程固定：

1. 读取文件。
2. 解析数据。
3. 校验数据。
4. 保存数据。

但不同格式的解析逻辑不同。

可以使用抽象类：

```
abstract class DataImporter {

    fun import(filePath: String) {
        val content = readFile(filePath)
        val records = parse(content)

        validate(records)
        save(records)
    }

    protected open fun readFile(filePath: String): String {
        println("读取文件：$filePath")
        return "mock-data"
    }

    protected abstract fun parse(content: String): List<String>

    protected open fun validate(records: List<String>) {
        println("校验 ${records.size} 条数据")
    }

    protected open fun save(records: List<String>) {
        println("保存 ${records.size} 条数据")
    }
}
```

CSV 实现：

```
class CsvImporter : DataImporter() {

    override fun parse(content: String): List<String> {
        println("解析 CSV")
        return listOf("Tom", "Jack")
    }
}
```

Excel 实现：

```
class ExcelImporter : DataImporter() {

    override fun parse(content: String): List<String> {
        println("解析 Excel")
        return listOf("Alice", "Bob", "Lucy")
    }
}
```

调用：

```
fun main() {
    val csv = CsvImporter()
    csv.import("users.csv")

    println("----------")

    val excel = ExcelImporter()
    excel.import("users.xlsx")
}
```

输出：

```
读取文件：users.csv
解析 CSV
校验 2 条数据
保存 2 条数据
----------
读取文件：users.xlsx
解析 Excel
校验 3 条数据
保存 3 条数据
```

这就是经典的 Template Method Pattern（模板方法模式）。

在 Java、C# 中也经常使用这种设计模式。

## 九、Kotlin 支持多继承吗？

Kotlin 不支持同时继承多个类。

例如：

```
open class A
open class B

// 编译错误
class C : A(), B()
```

但 Kotlin 支持：

继承一个类，同时实现多个接口。

例如：

```
open class Animal {
    fun eat() {
        println("eat")
    }
}

interface Flyable {
    fun fly()
}

interface Swimmable {
    fun swim()
}

class Duck : Animal(), Flyable, Swimmable {

    override fun fly() {
        println("飞行")
    }

    override fun swim() {
        println("游泳")
    }
}
```

## 十、接口默认方法冲突

如果两个接口定义了相同签名且有默认实现的方法，子类必须明确处理冲突。

```
interface A {

    fun hello() {
        println("A")
    }
}

interface B {

    fun hello() {
        println("B")
    }
}
```

错误写法：

```
class C : A, B
```

编译器无法判断应该继承哪个默认实现。

正确写法：

```
class C : A, B {

    override fun hello() {
        super<A>.hello()
        super<B>.hello()
    }
}
```

调用：

```
fun main() {
    C().hello()
}
```

输出：

```
A
B
```

这里：

```
super<A>.hello()
```

表示调用接口 `A` 的默认实现。

```
super<B>.hello()
```

表示调用接口 `B` 的默认实现。

如果父类和接口都提供了同名、同签名的可继承实现，也需要根据继承规则解决冲突。

## 十一、初始化顺序：继承中最容易踩坑的问题

Kotlin 对象初始化顺序很重要。

例如：

```
open class Base(
    val name: String
) {

    init {
        println("Base init: $name")
    }
}

class Derived(
    name: String,
    val age: Int
) : Base(name) {

    init {
        println("Derived init: $age")
    }
}

fun main() {
    Derived("Tom", 20)
}
```

输出：

```
Base init: Tom
Derived init: 20
```

通常可以理解为：

1. 计算传递给父类构造函数的参数。
2. 执行父类初始化。
3. 执行子类属性初始化和 `init` 块，按源码声明顺序进行。

### 危险场景：父类初始化时访问 open 属性

```
open class Base {

    open val name: String = "Base"

    init {
        println("Base init: $name")
    }
}

class Derived : Base() {

    override val name: String = "Derived"
}

fun main() {
    Derived()
}
```

这里可能输出：

```
Base init: null
```

原因是：

父类初始化时访问 `name`，发生动态分派，调用了子类属性的 getter。

但是此时子类的属性初始化还没有完成，因此可能读取到 JVM 字段的默认值 `null`。

这是一个很危险的问题。

最佳实践：不要在父类构造函数、属性初始化器或 `init` 块中调用可能被子类重写的成员。

即使属性在 Kotlin 类型系统中声明为非空，也不能保证这种初始化顺序下不会观察到未初始化状态。

## 十二、protected、private、internal 与继承

Kotlin 的可见性修饰符：

| 修饰符      | 类外访问   | 子类访问       | 说明         |
| ----------- | ---------- | -------------- | ------------ |
| `public`    | 可以       | 可以           | 默认         |
| `private`   | 不可以     | 不可以直接访问 | 仅声明范围内 |
| `protected` | 不可以     | 可以           | 当前类及子类 |
| `internal`  | 同模块可以 | 同模块可以     | 模块级可见   |

示例：

```
open class BaseService {

    private val secret = "private"

    protected val token = "protected"

    internal val config = "internal"

    fun execute() {
        println("public")
    }
}
```

子类：

```
class UserService : BaseService() {

    fun test() {
        println(token)
        println(config)

        // println(secret) // 编译错误
    }
}
```

注意：`internal` 是模块可见性，不是“包可见性”，与 Java 的 package-private 不同。

## 十三、继承与扩展函数的区别

继承的重写方法支持动态分派：

```
open class Animal {

    open fun speak() {
        println("Animal")
    }
}

class Dog : Animal() {

    override fun speak() {
        println("Dog")
    }
}

fun main() {
    val animal: Animal = Dog()

    animal.speak()
}
```

输出：

```
Dog
```

但是扩展函数采用静态解析：

```
open class Animal
class Dog : Animal()

fun Animal.speak() {
    println("Animal Extension")
}

fun Dog.speak() {
    println("Dog Extension")
}

fun main() {
    val animal: Animal = Dog()

    animal.speak()
}
```

输出：

```
Animal Extension
```

因为编译器根据变量的声明类型 `Animal` 选择扩展函数，而不是运行时对象的 `Dog` 类型。

| 对比             | 成员方法重写    | 扩展函数           |
| ---------------- | --------------- | ------------------ |
| 分派方式         | 动态分派        | 静态解析           |
| 是否需要继承     | 通常需要        | 不需要             |
| 能否重写已有成员 | 可以，需 `open` | 不可以             |
| 适合场景         | 多态业务行为    | 工具方法、API 扩展 |

## 十四、继承与 Delegation（委托）的区别

### 继承

```
open class Logger {

    open fun log(message: String) {
        println(message)
    }
}

class FileLogger : Logger() {

    override fun log(message: String) {
        println("File: $message")
    }
}
```

### 委托

```
interface Logger {
    fun log(message: String)
}

class ConsoleLogger : Logger {

    override fun log(message: String) {
        println(message)
    }
}

class FileLogger(
    logger: Logger
) : Logger by logger
```

区别：

| 特性                   | 继承             | 委托               |
| ---------------------- | ---------------- | ------------------ |
| 复用方式               | 继承父类实现     | 组合并转发给对象   |
| 耦合程度               | 通常较高         | 通常较低           |
| 能否动态更换实现       | 不方便           | 可通过组合设计实现 |
| 适合场景               | 稳定的 is-a 关系 | 可组合的行为       |
| 是否支持父类虚方法分派 | 支持             | 不自动实现相同语义 |

例如：

- `Dog` 是一种 `Animal`：适合继承。
- `OrderService` 使用一个 `Logger`：适合组合或委托。

实际业务系统中，优先考虑接口 + 组合；当确实存在稳定的类型层次或模板行为时，再考虑继承。

## 十五、Spring Boot 实战：抽象 Service 基类

在 Kotlin + Spring Boot 项目中，可以使用抽象类复用通用业务逻辑。

例如开发一个基础 CRUD Service。

### 1. 定义实体

```
data class User(
    val id: Long,
    val name: String
)
```

### 2. 定义 Repository 接口

```
interface CrudRepository<T, ID> {

    fun findById(id: ID): T?

    fun save(entity: T): T

    fun deleteById(id: ID)
}
```

这里是演示用的自定义接口，不是 Spring Data 自带的 `CrudRepository`。

### 3. 定义抽象 Service

```
abstract class BaseCrudService<T, ID>(
    protected val repository: CrudRepository<T, ID>
) {

    open fun findById(id: ID): T? {
        return repository.findById(id)
    }

    open fun save(entity: T): T {
        return repository.save(entity)
    }

    open fun deleteById(id: ID) {
        repository.deleteById(id)
    }
}
```

### 4. 定义 UserService

```
class UserService(
    repository: CrudRepository<User, Long>
) : BaseCrudService<User, Long>(repository) {

    override fun save(entity: User): User {
        require(entity.name.isNotBlank()) {
            "用户名不能为空"
        }

        return super.save(entity)
    }
}
```

### 5. 定义内存 Repository

```
class InMemoryUserRepository : CrudRepository<User, Long> {

    private val data = mutableMapOf<Long, User>()

    override fun findById(id: Long): User? {
        return data[id]
    }

    override fun save(entity: User): User {
        data[entity.id] = entity
        return entity
    }

    override fun deleteById(id: Long) {
        data.remove(id)
    }
}
```

### 6. 测试

```
fun main() {
    val repository = InMemoryUserRepository()
    val service = UserService(repository)

    service.save(User(1, "Tom"))
    service.save(User(2, "Jack"))

    println(service.findById(1))

    service.deleteById(2)

    println(service.findById(2))
}
```

输出：

```
User(id=1, name=Tom)
null
```

这个例子将以下 Kotlin 特性组合起来：

- `abstract class`：抽象类。
- `<T, ID>`：泛型。
- `protected`：受保护成员。
- `open`：允许重写。
- `override`：重写业务方法。
- `super`：调用父类逻辑。
- `interface`：仓储契约。

需要特别注意：Spring Boot 中的 `@Transactional`、`@Cacheable` 等基于代理的功能可能涉及类和方法是否可代理。Kotlin 类默认 `final`，因此 Spring 项目通常使用 `kotlin-spring`（all-open）编译器插件，为特定 Spring 注解类及成员生成适合代理的开放声明。

官方参考：[Kotlin all-open Compiler Plugin](https://kotlinlang.org/docs/all-open-plugin.html?utm_source=chatgpt.com)。

## 十六、Kotlin 的 sealed class 和继承

还有一个值得学习的知识点：`sealed class`（密封类）。

它与 `open class` 不同。

```
sealed class ApiResult {

    data class Success(
        val data: String
    ) : ApiResult()

    data class Error(
        val message: String
    ) : ApiResult()

    data object Loading : ApiResult()
}
```

使用：

```
fun handle(result: ApiResult) {

    when (result) {
        is ApiResult.Success -> {
            println(result.data)
        }

        is ApiResult.Error -> {
            println(result.message)
        }

        ApiResult.Loading -> {
            println("加载中")
        }
    }
}
```

因为 `ApiResult` 是密封类，编译器能够了解受限制的直接继承层次，帮助检查 `when` 是否穷尽。

需要注意：`sealed` 不是彻底禁止继承，而是限制直接子类的声明位置和模块边界；符合规则的非密封子类仍可能继续被继承。

这非常适合：

- API 请求结果。
- 业务状态。
- 订单状态机。
- UI 状态。
- 错误类型建模。

官方参考：[Kotlin Sealed Classes](https://kotlinlang.org/docs/sealed-classes.html?utm_source=chatgpt.com)。

## 十七、常见错误总结

| 错误写法或理解                                  | 原因                  | 正确做法                 |
| ----------------------------------------------- | --------------------- | ------------------------ |
| `class Dog : Animal()`，但 `Animal` 没有 `open` | 父类默认 final        | `open class Animal`      |
| 重写普通方法                                    | 方法默认 final        | 父类方法加 `open`        |
| `override` 后认为不能再重写                     | override 默认仍开放   | 使用 `final override`    |
| `var` 属性用 `val` 重写                         | 丢失写入能力          | 用 `var` 重写            |
| 同时继承两个普通类                              | Kotlin 不支持类多继承 | 一个父类 + 多个接口      |
| 父类 `init` 访问开放属性                        | 子类可能尚未初始化    | 避免构造期间调用开放成员 |
| 把 Extension 当作多态                           | 扩展函数静态解析      | 使用真正的成员重写       |
| 认为继承总比组合方便                            | 继承容易形成强耦合    | 优先根据业务关系选择     |