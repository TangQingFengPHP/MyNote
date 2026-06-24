### 简介

同样一行 Kotlin 代码：

```kotlin
a += b
```

有时会直接修改 `a` 指向的对象，有时却会先计算 `a + b`，再把新对象赋值给 `a`。

再看几种熟悉的写法：

```kotlin
val total = price + freight
val allowed = userId in permissionSet
val cell = board[2, 3]
val result = rule(order)
val versions = startVersion..endVersion
```

这些符号看起来像 Kotlin 内置语法，背后其实都可能是普通函数调用：

```kotlin
price.plus(freight)
permissionSet.contains(userId)
board.get(2, 3)
rule.invoke(order)
startVersion.rangeTo(endVersion)
```

这就是 Kotlin Operator Overloading，运算符重载。

它并不是让符号可以随意发挥，而是建立了一套固定约定：

```text
固定运算符
固定函数名
固定参数形式
```

例如 `+` 只能映射到 `plus`，不能改成 `add`；`[]` 只能映射到 `get` 或 `set`；`in` 只能映射到 `contains`。

运算符重载用得合适，金额、向量、矩阵、区间和 DSL 会非常自然。用得随意，一个 `+` 号也可能藏住删除数据、网络请求甚至数据库写入。

### 第一个运算符：让坐标支持加法

先定义一个二维坐标：

```kotlin
data class Point(
    val x: Int,
    val y: Int
)
```

普通写法：

```kotlin
fun Point.add(other: Point): Point {
    return Point(
        x = x + other.x,
        y = y + other.y
    )
}

fun main() {
    val p1 = Point(10, 20)
    val p2 = Point(3, 5)

    println(p1.add(p2))
}
```

输出：

```text
Point(x=13, y=25)
```

把函数名改成 Kotlin 约定的 `plus`，再加上 `operator`：

```kotlin
data class Point(
    val x: Int,
    val y: Int
) {
    operator fun plus(other: Point): Point {
        return Point(
            x = x + other.x,
            y = y + other.y
        )
    }
}
```

调用时就能使用 `+`：

```kotlin
fun main() {
    val p1 = Point(10, 20)
    val p2 = Point(3, 5)

    val result = p1 + p2
    println(result)
}
```

输出：

```text
Point(x=13, y=25)
```

编译器按照约定解析：

```kotlin
p1 + p2
```

等价于：

```kotlin
p1.plus(p2)
```

去掉 `operator` 之后，`p1.plus(p2)` 仍然可以调用，但 `p1 + p2` 会编译失败。

### 运算符重载不是创建新符号

Kotlin 只允许重载语言规定好的运算符，不能发明一个新符号，也不能修改符号的优先级和结合方向。

例如：

```kotlin
val result = a + b * c
```

仍然会先计算 `b * c`，再计算 `a + ...`。

它近似展开成：

```kotlin
val result = a.plus(b.times(c))
```

即使 `plus` 和 `times` 都是自定义函数，也不能改变 `*` 比 `+` 优先的规则。

运算符函数可以定义成：

* 类的成员函数
* 扩展函数

成员函数示例：

```kotlin
data class Point(val x: Int, val y: Int) {
    operator fun plus(other: Point): Point {
        return Point(x + other.x, y + other.y)
    }
}
```

扩展函数示例：

```kotlin
operator fun Int.times(point: Point): Point {
    return Point(
        x = this * point.x,
        y = this * point.y
    )
}

fun main() {
    println(3 * Point(2, 4))
}
```

输出：

```text
Point(x=6, y=12)
```

需要注意，成员函数的优先级高于扩展函数。扩展函数无法覆盖类里已经存在的同签名成员函数。

### 常用运算符映射表

#### 一元运算符

| 表达式 | 对应函数 |
| --- | --- |
| `+a` | `a.unaryPlus()` |
| `-a` | `a.unaryMinus()` |
| `!a` | `a.not()` |
| `a++`、`++a` | `a.inc()` |
| `a--`、`--a` | `a.dec()` |

#### 二元算术运算符

| 表达式 | 对应函数 |
| --- | --- |
| `a + b` | `a.plus(b)` |
| `a - b` | `a.minus(b)` |
| `a * b` | `a.times(b)` |
| `a / b` | `a.div(b)` |
| `a % b` | `a.rem(b)` |

#### 区间和成员判断

| 表达式 | 对应函数 |
| --- | --- |
| `a..b` | `a.rangeTo(b)` |
| `a..<b` | `a.rangeUntil(b)` |
| `a in b` | `b.contains(a)` |
| `a !in b` | `!b.contains(a)` |

#### 索引和调用

| 表达式 | 对应函数 |
| --- | --- |
| `a[i]` | `a.get(i)` |
| `a[i, j]` | `a.get(i, j)` |
| `a[i] = value` | `a.set(i, value)` |
| `a[i, j] = value` | `a.set(i, j, value)` |
| `a()` | `a.invoke()` |
| `a(x, y)` | `a.invoke(x, y)` |

#### 比较和相等

| 表达式 | 对应逻辑 |
| --- | --- |
| `a < b` | `a.compareTo(b) < 0` |
| `a > b` | `a.compareTo(b) > 0` |
| `a <= b` | `a.compareTo(b) <= 0` |
| `a >= b` | `a.compareTo(b) >= 0` |
| `a == b` | 空安全调用 `a?.equals(b)` |
| `a != b` | `a == b` 的结果取反 |

#### 复合赋值

| 表达式 | 原地修改函数 | 创建新值的后备写法 |
| --- | --- | --- |
| `a += b` | `a.plusAssign(b)` | `a = a.plus(b)` |
| `a -= b` | `a.minusAssign(b)` | `a = a.minus(b)` |
| `a *= b` | `a.timesAssign(b)` | `a = a.times(b)` |
| `a /= b` | `a.divAssign(b)` | `a = a.div(b)` |
| `a %= b` | `a.remAssign(b)` | `a = a.rem(b)` |

后面的章节会把最容易出错的几类单独拆开。

### 实战 Demo：金额为什么适合重载运算符

金额具有明确的加法、减法、倍数和比较语义，很适合做成不可变值对象。

为了避免 `Double` 精度问题，示例使用最小货币单位保存金额：人民币使用“分”。

```kotlin
import java.math.BigDecimal

data class Money private constructor(
    private val minor: Long,
    val currency: String
) : Comparable<Money> {

    companion object {
        fun cny(yuan: Long, fen: Int = 0): Money {
            require(fen in 0..99) { "分必须在 0 到 99 之间" }
            return Money(yuan * 100 + fen, "CNY")
        }

        fun usd(dollar: Long, cent: Int = 0): Money {
            require(cent in 0..99) { "美分必须在 0 到 99 之间" }
            return Money(dollar * 100 + cent, "USD")
        }
    }

    operator fun plus(other: Money): Money {
        requireSameCurrency(other)
        return copy(minor = minor + other.minor)
    }

    operator fun minus(other: Money): Money {
        requireSameCurrency(other)
        return copy(minor = minor - other.minor)
    }

    operator fun times(quantity: Int): Money {
        require(quantity >= 0) { "数量不能小于 0" }
        return copy(minor = minor * quantity)
    }

    operator fun unaryMinus(): Money {
        return copy(minor = -minor)
    }

    override fun compareTo(other: Money): Int {
        requireSameCurrency(other)
        return minor.compareTo(other.minor)
    }

    private fun requireSameCurrency(other: Money) {
        require(currency == other.currency) {
            "不同币种不能直接计算：$currency 与 ${other.currency}"
        }
    }

    override fun toString(): String {
        val amount = BigDecimal.valueOf(minor, 2)
        return "$currency $amount"
    }
}
```

再提供一个扩展函数，让数量可以写在左边：

```kotlin
operator fun Int.times(price: Money): Money {
    return price * this
}
```

使用：

```kotlin
fun main() {
    val unitPrice = Money.cny(yuan = 19, fen = 90)
    val freight = Money.cny(yuan = 6)

    val subtotal = unitPrice * 3
    val total = subtotal + freight

    println(subtotal)
    println(total)
    println(total > Money.cny(50))
    println(2 * unitPrice)
}
```

输出：

```text
CNY 59.70
CNY 65.70
true
CNY 39.80
```

代码读起来接近业务公式：

```kotlin
val total = unitPrice * 3 + freight
```

运算符函数仍然负责守住业务规则。不同币种相加会直接失败：

```kotlin
val cny = Money.cny(10)
val usd = Money.usd(10)

// 抛出 IllegalArgumentException
println(cny + usd)
```

这里的 `+` 符合直觉：相同币种相加得到新金额，原对象不变。

### 运算符不要求左右两边类型相同

`plus`、`times` 等函数的参数类型和返回类型可以按业务需要设计。

例如：

```kotlin
data class Basket(
    val items: List<String>
) {
    operator fun plus(product: String): Basket {
        return copy(items = items + product)
    }
}

fun main() {
    val basket = Basket(emptyList())
    val result = basket + "Keyboard" + "Mouse"

    println(result.items)
}
```

输出：

```text
[Keyboard, Mouse]
```

这里是：

```text
Basket + String -> Basket
```

但是交换顺序不会自动成立：

```kotlin
// 没有 String.plus(Basket)，所以不能编译
// "Keyboard" + basket
```

运算符重载不会自动获得交换律。需要反方向语法时，必须再提供对应的成员函数或扩展函数。

### 一元运算符：负号、正号和逻辑取反

一元运算符没有额外参数。

```kotlin
data class Vector(
    val x: Int,
    val y: Int
) {
    operator fun unaryMinus(): Vector {
        return Vector(-x, -y)
    }

    operator fun unaryPlus(): Vector {
        return this
    }
}

fun main() {
    val vector = Vector(3, -5)

    println(-vector)
    println(+vector)
}
```

输出：

```text
Vector(x=-3, y=5)
Vector(x=3, y=-5)
```

`!` 对应 `not()`，返回类型不强制必须是 `Boolean`，但实际设计最好符合“取反”直觉。

```kotlin
data class FeatureFlag(
    val enabled: Boolean
) {
    operator fun not(): FeatureFlag {
        return copy(enabled = !enabled)
    }
}

fun main() {
    val flag = FeatureFlag(enabled = true)
    println(!flag)
}
```

输出：

```text
FeatureFlag(enabled=false)
```

### `++` 和 `--` 不是普通的原地修改

`++` 对应 `inc()`，`--` 对应 `dec()`。

```kotlin
data class VersionCode(val value: Int) {
    operator fun inc(): VersionCode {
        return VersionCode(value + 1)
    }

    operator fun dec(): VersionCode {
        return VersionCode(value - 1)
    }
}
```

使用时变量必须能够重新赋值：

```kotlin
fun main() {
    var version = VersionCode(10)

    val old = version++

    println(old)
    println(version)
}
```

输出：

```text
VersionCode(value=10)
VersionCode(value=11)
```

后置自增可以近似理解为：

```kotlin
val old = version
version = version.inc()
```

前置自增：

```kotlin
val current = ++version
```

可以近似理解为：

```kotlin
version = version.inc()
val current = version
```

`inc()` 和 `dec()` 应返回新值，不应偷偷修改原对象。最终变量更新由编译器完成。

因此下面这种写法不能工作：

```kotlin
val version = VersionCode(10)

// 编译失败：val 不能重新赋值
// version++
```

### 比较运算符只需要一个 compareTo

`<`、`>`、`<=`、`>=` 都映射到 `compareTo`。

实现版本号比较：

```kotlin
data class SemanticVersion(
    val major: Int,
    val minor: Int,
    val patch: Int
) : Comparable<SemanticVersion> {

    override fun compareTo(other: SemanticVersion): Int {
        return compareValuesBy(
            this,
            other,
            SemanticVersion::major,
            SemanticVersion::minor,
            SemanticVersion::patch
        )
    }

    override fun toString(): String {
        return "$major.$minor.$patch"
    }
}
```

使用：

```kotlin
fun main() {
    val current = SemanticVersion(2, 3, 10)
    val required = SemanticVersion(2, 2, 20)

    println(current > required)
    println(current >= SemanticVersion(2, 3, 10))
}
```

输出：

```text
true
true
```

`compareTo` 返回值只看正负号：

```text
小于 0：左边小于右边
等于 0：两边排序位置相同
大于 0：左边大于右边
```

不要假设它只能返回 `-1`、`0`、`1`。

也不要直接用减法实现所有比较：

```kotlin
// 不推荐，极端数值可能溢出
return age - other.age
```

更稳妥的写法：

```kotlin
return age.compareTo(other.age)
```

### compareTo 和 equals 最好保持一致

如果 `a.compareTo(b) == 0`，通常也应该让 `a == b` 成立。

否则把对象放进排序集合时容易出现意外结果。

例如 `TreeSet` 主要依赖排序结果判断元素位置。如果两个对象 `compareTo` 返回 `0`，但 `equals` 返回 `false`，集合行为会很难理解。

值对象可以优先使用 `data class`，让 `equals`、`hashCode` 和属性值保持一致，再让 `compareTo` 使用同一组关键属性。

### `==`、`===` 根本不是一回事

Kotlin 的 `==` 检查结构相等，底层使用 `equals`。

```kotlin
data class User(
    val id: Long,
    val name: String
)

fun main() {
    val a = User(1, "Tom")
    val b = User(1, "Tom")

    println(a == b)
    println(a === b)
}
```

输出：

```text
true
false
```

原因：

```text
a == b   属性值相等
a === b  是否为同一个对象引用
```

`data class` 自动生成 `equals`，所以两个不同对象也能通过 `==` 判断为相等。

普通类需要重写 `equals` 和 `hashCode`：

```kotlin
class Account(
    val number: String
) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is Account) return false
        return number == other.number
    }

    override fun hashCode(): Int {
        return number.hashCode()
    }
}
```

`equals` 是一个特殊约定。`Any.equals` 本身已经具有运算符能力，重写时使用 `override` 即可，不需要额外写 `operator`。

```kotlin
override fun equals(other: Any?): Boolean
```

`==` 还是空安全的。表达式：

```kotlin
a == b
```

可以近似理解为：

```kotlin
a?.equals(b) ?: (b === null)
```

`===` 和 `!==` 是引用比较，不能重载。

### 下标访问：让二维棋盘像数组一样使用

`get` 和 `set` 可以接收多个索引参数，很适合矩阵、棋盘、表格和坐标容器。

```kotlin
class Board(
    private val rows: Int,
    private val columns: Int
) {
    private val cells = Array(rows) {
        Array(columns) { "." }
    }

    operator fun get(row: Int, column: Int): String {
        checkPosition(row, column)
        return cells[row][column]
    }

    operator fun set(row: Int, column: Int, value: String) {
        checkPosition(row, column)
        cells[row][column] = value
    }

    private fun checkPosition(row: Int, column: Int) {
        require(row in 0 until rows) { "行下标越界：$row" }
        require(column in 0 until columns) { "列下标越界：$column" }
    }

    override fun toString(): String {
        return cells.joinToString("\n") { row ->
            row.joinToString(" ")
        }
    }
}
```

使用：

```kotlin
fun main() {
    val board = Board(rows = 3, columns = 3)

    board[0, 0] = "X"
    board[1, 1] = "O"
    board[2, 2] = "X"

    println(board[1, 1])
    println(board)
}
```

输出：

```text
O
X . .
. O .
. . X
```

编译器转换规则：

```kotlin
board[1, 1]
```

对应：

```kotlin
board.get(1, 1)
```

而：

```kotlin
board[1, 1] = "O"
```

对应：

```kotlin
board.set(1, 1, "O")
```

`set` 的最后一个参数是赋进去的值，前面的参数都是索引。

### `in` 的调用方向是反的

看到：

```kotlin
userId in permissionSet
```

很容易误以为会调用：

```kotlin
userId.contains(permissionSet)
```

实际方向正好相反：

```kotlin
permissionSet.contains(userId)
```

定义一个权限集合：

```kotlin
class PermissionSet(
    private val allowedUserIds: Set<Long>
) {
    operator fun contains(userId: Long): Boolean {
        return userId in allowedUserIds
    }
}

fun main() {
    val permissions = PermissionSet(
        allowedUserIds = setOf(1001L, 1002L)
    )

    println(1001L in permissions)
    println(2001L !in permissions)
}
```

输出：

```text
true
true
```

`!in` 不需要单独实现函数，它就是对 `contains` 的结果取反。

### 自定义区间：版本号也能使用 `..`

前面已经让 `SemanticVersion` 实现了 `Comparable`。

再增加 `rangeTo`：

```kotlin
operator fun SemanticVersion.rangeTo(
    end: SemanticVersion
): ClosedRange<SemanticVersion> {
    return object : ClosedRange<SemanticVersion> {
        override val start: SemanticVersion = this@rangeTo
        override val endInclusive: SemanticVersion = end
    }
}
```

使用：

```kotlin
fun main() {
    val supported = SemanticVersion(2, 1, 0)..SemanticVersion(2, 5, 0)

    println(SemanticVersion(2, 3, 1) in supported)
    println(SemanticVersion(3, 0, 0) in supported)
}
```

输出：

```text
true
false
```

这里组合了三层约定：

```text
..          调用 rangeTo
in          调用 contains
范围比较    最终依赖 compareTo
```

区间只表示“从哪里到哪里”时，`ClosedRange` 已经够用。

如果还希望写：

```kotlin
for (version in supported) {
    // ...
}
```

那就需要额外定义如何产生下一个版本，并让返回对象支持迭代。一个区间能做成员判断，不代表它天然能够遍历。

### `invoke`：对象为什么能像函数一样调用

`invoke` 对应圆括号调用：

```kotlin
object(argument)
```

例如订单折扣规则：

```kotlin
data class Order(
    val userLevel: Int,
    val total: Money
)

class DiscountRule(
    private val minimumLevel: Int,
    private val discount: Money
) {
    operator fun invoke(order: Order): Money {
        return if (order.userLevel >= minimumLevel) {
            order.total - discount
        } else {
            order.total
        }
    }
}
```

使用：

```kotlin
fun main() {
    val vipDiscount = DiscountRule(
        minimumLevel = 3,
        discount = Money.cny(10)
    )

    val order = Order(
        userLevel = 4,
        total = Money.cny(99)
    )

    val payable = vipDiscount(order)
    println(payable)
}
```

输出：

```text
CNY 89.00
```

这行：

```kotlin
vipDiscount(order)
```

等价于：

```kotlin
vipDiscount.invoke(order)
```

`invoke` 适合表达“这个对象本身就是一条可执行规则”：

* 校验规则
* 价格规则
* 路由匹配器
* 数据转换器
* DSL 节点

如果对象主要职责不是执行，普通命名函数通常更清楚。例如 `repository(user)` 很难看出是在查询、保存还是删除，写成 `repository.find(user)` 会更明确。

### `+=` 为什么有时改对象，有时换对象

这是运算符重载最容易踩坑的地方。

#### 只有 plusAssign：直接修改对象

```kotlin
class ShoppingCart {
    private val products = mutableListOf<String>()

    operator fun plusAssign(product: String) {
        products += product
    }

    override fun toString(): String {
        return products.toString()
    }
}

fun main() {
    val cart = ShoppingCart()

    cart += "Keyboard"
    cart += "Mouse"

    println(cart)
}
```

输出：

```text
[Keyboard, Mouse]
```

这里调用：

```kotlin
cart.plusAssign("Keyboard")
```

`cart` 是 `val` 也能使用 `+=`，因为变量没有重新赋值，只是对象内部状态发生了变化。

#### 只有 plus：计算新对象后重新赋值

```kotlin
data class ImmutableCart(
    val products: List<String>
) {
    operator fun plus(product: String): ImmutableCart {
        return copy(products = products + product)
    }
}

fun main() {
    var cart = ImmutableCart(emptyList())

    cart += "Keyboard"
    cart += "Mouse"

    println(cart.products)
}
```

这时 `cart += product` 近似展开为：

```kotlin
cart = cart.plus(product)
```

因此变量必须是 `var`。

如果写成 `val`：

```kotlin
val cart = ImmutableCart(emptyList())

// 编译失败：需要重新给 cart 赋值
// cart += "Keyboard"
```

#### plus 和 plusAssign 不要同时定义成同样适用

如果同一个类型同时提供可用的 `plus` 和 `plusAssign`，表达式 `a += b` 可能产生重载解析歧义，编译器不会稳定地替业务代码猜测“修改原对象”还是“创建新对象”。

更稳妥的设计：

```text
不可变类型：提供 plus，返回新对象
可变类型：按需提供 plusAssign，修改当前对象
同一组参数不要让 plus 和 plusAssign 同时竞争
```

这也是 Kotlin 集合里 `List` 和 `MutableList` 行为容易让人困惑的根源之一。看到 `+=` 时，必须结合变量类型、`val`/`var` 和可用运算符函数判断真实行为。

### 解构声明背后也是 operator 约定

下面的写法叫解构声明：

```kotlin
val (name, age) = user
```

它会调用：

```kotlin
val name = user.component1()
val age = user.component2()
```

`data class` 会自动生成 `componentN`：

```kotlin
data class User(
    val name: String,
    val age: Int
)

fun main() {
    val user = User("Tom", 20)
    val (name, age) = user

    println(name)
    println(age)
}
```

普通类也能手动支持解构：

```kotlin
class Coordinate(
    private val longitude: Double,
    private val latitude: Double
) {
    operator fun component1(): Double = longitude
    operator fun component2(): Double = latitude
}

fun main() {
    val coordinate = Coordinate(120.15, 30.28)
    val (longitude, latitude) = coordinate

    println("$longitude, $latitude")
}
```

`component1`、`component2` 没有对应的可见符号，但仍属于 Kotlin 的运算符约定体系。

返回顺序必须稳定且符合对象含义。解构位置没有字段名，一旦顺序设计混乱，调用处很难发现错误。

### for 循环也依赖约定函数

`for` 循环并不要求对象必须实现 Java 的 `Iterable` 接口，只要能提供符合约定的迭代器即可。

```kotlin
class Countdown(
    private val start: Int
) {
    operator fun iterator(): Iterator<Int> {
        return object : Iterator<Int> {
            private var current = start

            override fun hasNext(): Boolean {
                return current >= 0
            }

            override fun next(): Int {
                return current--
            }
        }
    }
}

fun main() {
    for (number in Countdown(3)) {
        println(number)
    }
}
```

输出：

```text
3
2
1
0
```

循环会使用 `iterator()`，再反复调用 `hasNext()` 和 `next()`。

实现标准 `Iterator` 或 `Iterable` 通常更清楚，也更容易和集合 API 配合。约定函数适合轻量 DSL 或无法修改原类型的场景。

### 属性委托也使用 operator 函数

属性委托语法：

```kotlin
val token by delegate
```

读取属性时会调用委托对象的 `getValue`。

```kotlin
import kotlin.reflect.KProperty

class EnvironmentValue(
    private val values: Map<String, String>,
    private val key: String
) {
    operator fun getValue(
        thisRef: Any?,
        property: KProperty<*>
    ): String {
        return values[key]
            ?: error("缺少配置：$key，对应属性：${property.name}")
    }
}

class AppConfig(environment: Map<String, String>) {
    val apiUrl: String by EnvironmentValue(environment, "API_URL")
    val appName: String by EnvironmentValue(environment, "APP_NAME")
}
```

使用：

```kotlin
fun main() {
    val config = AppConfig(
        mapOf(
            "API_URL" to "https://example.com",
            "APP_NAME" to "Order Center"
        )
    )

    println(config.apiUrl)
    println(config.appName)
}
```

输出：

```text
https://example.com
Order Center
```

可变委托属性还需要实现 `setValue`：

```kotlin
operator fun setValue(
    thisRef: Any?,
    property: KProperty<*>,
    value: String
)
```

`getValue`、`setValue` 和 `provideDelegate` 没有 `+`、`[]` 这样的符号，但同样通过 `operator` 约定驱动语言语法。

### Demo：用 unaryPlus 做一个小型文本 DSL

Kotlin DSL 里经常看到这种写法：

```kotlin
+"订单创建成功"
```

它不是特殊字符串语法，而是 `unaryPlus()`。

```kotlin
class MessageBuilder {
    private val lines = mutableListOf<String>()

    operator fun String.unaryPlus() {
        lines += this
    }

    fun build(): String {
        return lines.joinToString("\n")
    }
}

fun message(block: MessageBuilder.() -> Unit): String {
    return MessageBuilder()
        .apply(block)
        .build()
}
```

使用：

```kotlin
fun main() {
    val text = message {
        +"订单编号：1001"
        +"支付状态：已支付"
        +"配送状态：待发货"
    }

    println(text)
}
```

输出：

```text
订单编号：1001
支付状态：已支付
配送状态：待发货
```

这里的：

```kotlin
+"订单编号：1001"
```

会在 `MessageBuilder` 作用域内调用：

```kotlin
"订单编号：1001".unaryPlus()
```

这种写法适合边界明确的 DSL。普通业务代码如果大量出现孤立的 `+"text"`，反而会增加理解成本。

### 哪些运算符不能重载

Kotlin 不允许所有符号都参与重载。

常见不可重载语法包括：

* `&&`、`||`
* `?:`
* `===`、`!==`
* `is`、`!is`
* `as`、`as?`
* `=`
* `.`、`?.`
* `::`

`&&` 和 `||` 不能重载，一个重要原因是短路求值。

```kotlin
conditionA() && conditionB()
```

如果 `conditionA()` 为 `false`，`conditionB()` 根本不会执行。

普通函数调用通常要先计算参数。如果允许把 `&&` 简单映射成函数，短路语义就容易被破坏。

`===` 也不能重载，因为它明确表示引用是否相同，不应该由业务类型重新解释。

### 运算符重载几乎不等于额外反射开销

表达式：

```kotlin
val result = a + b
```

由编译器静态解析为对应函数调用，不需要运行时反射查找。

因此运算符写法和直接调用 `a.plus(b)` 通常没有本质性能差异。

真正需要关注的是运算符函数内部做了什么：

* 是否创建大量新对象
* 是否复制大集合
* 是否执行数据库或网络操作
* 是否存在复杂算法
* 是否修改共享状态

符号很短，不代表执行成本很低。

例如下面的设计虽然语法允许，但非常不合适：

```kotlin
operator fun User.plus(role: Role): User {
    database.insertUserRole(id, role.id)
    return this
}
```

看到 `user + role`，很难想到它会写数据库。

更清楚的写法：

```kotlin
fun User.assignRole(role: Role) {
    database.insertUserRole(id, role.id)
}
```

### 运算符设计的五条原则

#### 符号语义必须符合直觉

适合：

```kotlin
moneyA + moneyB
vector * scalar
item in container
matrix[row, column]
```

不适合：

```kotlin
user + role       // 实际写数据库
order - customer  // 实际取消订单
!service          // 实际重启服务
```

#### `plus` 优先返回新对象

`+` 通常给人“计算一个新结果”的预期。

```kotlin
val c = a + b
```

如果这行代码偷偷修改 `a`，后续逻辑很容易出错。

不可变值对象最适合重载算术运算符。

#### 可变操作要明显

确实需要原地修改时，可以使用 `plusAssign`，但类型本身应该清楚地表现出可变性，例如 `MutableCart`、`MutableMatrix`。

#### compareTo、equals、hashCode 保持一致

相等、排序和哈希集合使用的是不同约定。三者含义冲突时，`Set`、`Map`、`TreeSet` 和排序结果都可能变得反直觉。

#### 不要为了少写几个字符牺牲业务含义

```kotlin
repository(user)
```

虽然比：

```kotlin
repository.save(user)
```

短，但动作含义消失了。

符号适合稳定、通用、接近数学或容器直觉的操作。带副作用、成本高、失败模式复杂的业务动作，命名函数通常更可靠。

### 一个完整综合 Demo

下面用库存对象串起 `plus`、`minus`、`contains`、`get` 和 `invoke`。

```kotlin
data class Product(
    val sku: String,
    val name: String
)

data class StockItem(
    val product: Product,
    val quantity: Int
)

class Inventory private constructor(
    private val quantities: Map<Product, Int>
) {
    companion object {
        fun empty(): Inventory {
            return Inventory(emptyMap())
        }
    }

    operator fun plus(item: StockItem): Inventory {
        require(item.quantity > 0) { "入库数量必须大于 0" }

        val current = quantities[item.product] ?: 0
        return Inventory(
            quantities + (item.product to (current + item.quantity))
        )
    }

    operator fun minus(item: StockItem): Inventory {
        require(item.quantity > 0) { "出库数量必须大于 0" }

        val current = quantities[item.product] ?: 0
        require(current >= item.quantity) {
            "库存不足：${item.product.sku}，当前 $current，需要 ${item.quantity}"
        }

        val remaining = current - item.quantity
        val newQuantities = if (remaining == 0) {
            quantities - item.product
        } else {
            quantities + (item.product to remaining)
        }

        return Inventory(newQuantities)
    }

    operator fun contains(product: Product): Boolean {
        return get(product) > 0
    }

    operator fun get(product: Product): Int {
        return quantities[product] ?: 0
    }

    operator fun invoke(): List<StockItem> {
        return quantities.map { (product, quantity) ->
            StockItem(product, quantity)
        }
    }
}
```

使用：

```kotlin
fun main() {
    val keyboard = Product("K001", "Keyboard")
    val mouse = Product("M001", "Mouse")

    var inventory = Inventory.empty()

    inventory += StockItem(keyboard, 10)
    inventory += StockItem(mouse, 5)
    inventory -= StockItem(keyboard, 3)

    println(keyboard in inventory)
    println(inventory[keyboard])
    println(inventory[mouse])
    println(inventory())
}
```

输出：

```text
true
7
5
[StockItem(product=Product(sku=K001, name=Keyboard), quantity=7), StockItem(product=Product(sku=M001, name=Mouse), quantity=5)]
```

这里没有定义 `plusAssign` 和 `minusAssign`。

所以：

```kotlin
inventory += item
inventory -= item
```

分别退化成：

```kotlin
inventory = inventory + item
inventory = inventory - item
```

`Inventory` 自身保持不可变，每次入库和出库都返回新对象。变量使用 `var`，只是为了接住新结果。

### 常见误区

#### `operator` 可以加在任意函数名上

不可以。

函数名和参数形式必须符合 Kotlin 已定义的约定。`operator fun add(...)` 不会获得 `+` 语法。

#### `a in b` 调用 `a.contains(b)`

正好相反，它调用 `b.contains(a)`。

#### `==` 和 `===` 都能重载

只有 `==` 会使用可重写的 `equals`。`===` 固定比较对象引用，不能重载。

#### compareTo 必须返回 -1、0、1

不需要。只要负数、零、正数的方向正确即可。

#### `a += b` 一定调用 plusAssign

不一定。没有适用的 `plusAssign` 时，可以退化成 `a = a + b`。两套函数同时适用时还可能产生歧义。

#### 运算符函数一定没有副作用

语言没有强制限制，但 API 设计应该遵守直觉。`plus`、`minus`、`times` 最好返回新值，明显的可变操作再使用 `plusAssign` 等函数。

#### 区间能判断 in，就一定能 for 遍历

不一定。成员判断依赖 `contains`，遍历依赖 `iterator`、`hasNext` 和 `next`，是两套不同约定。

### 总结

Kotlin 运算符重载的本质不是符号魔法，而是编译器约定的函数调用：

```kotlin
a + b       -> a.plus(b)
a[i]        -> a.get(i)
a in b      -> b.contains(a)
a()         -> a.invoke()
a..b        -> a.rangeTo(b)
a < b       -> a.compareTo(b) < 0
```

最容易混淆的是复合赋值：

```text
plusAssign  修改当前对象
plus        返回新对象，再由变量接住新值
```

适合运算符重载的类型通常具备稳定、直观的运算含义：

* 金额
* 向量和矩阵
* 区间
* 容器
* 索引结构
* 可执行规则
* 小型 DSL

一句话概括：

> 运算符重载的价值不是把函数名变短，而是让类型的行为更接近它所表达的概念；符号无法准确表达业务动作时，普通命名函数反而更清楚。
