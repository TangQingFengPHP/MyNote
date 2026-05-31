### 简介

`Optional` 是 `Java 8` 引入的一个容器类，包路径是：

```java
java.util.Optional
```

它用来表达一种很常见的情况：

```text
这个结果可能有值，也可能没有值。
```

比如按 ID 查询用户：

```java
User user = userRepository.findById(1L);
```

如果找到了，返回 `User`。

如果没找到，传统写法通常返回 `null`。

`Optional` 的写法是：

```java
Optional<User> user = userRepository.findById(1L);
```

这段代码的含义更明确：

```text
findById 可能查到用户，也可能查不到用户。
```

一句话概括：

```text
Optional 用来把“可能为空”这件事写进方法返回值里，让空值处理变得更清楚。
```

### 为什么需要 Optional

先看一段普通判空代码：

```java
public String getUserCity(User user) {
    if (user != null) {
        Address address = user.getAddress();
        if (address != null) {
            String city = address.getCity();
            if (city != null) {
                return city;
            }
        }
    }

    return "未知城市";
}
```

代码本身不复杂，但层级一多，判空会越来越长。

如果中间漏掉一次判空：

```java
return user.getAddress().getCity();
```

只要 `user` 或 `address` 是 `null`，就会出现：

```text
NullPointerException
```

使用 `Optional` 后，可以把“取值、转换、兜底”连在一起：

```java
public String getUserCity(User user) {
    return Optional.ofNullable(user)
            .map(User::getAddress)
            .map(Address::getCity)
            .orElse("未知城市");
}
```

这段代码表达的是：

```text
user 有值就取 address
address 有值就取 city
city 有值就返回 city
中间任何一步为空，就返回“未知城市”
```

### Optional 的基本结构

`Optional<T>` 里的 `T` 表示实际包装的数据类型。

```java
Optional<String> name;
Optional<User> user;
Optional<Order> order;
```

`Optional` 只有两种状态：

* 有值：里面包装了一个非 `null` 对象
* 空：里面没有值

创建一个有值的 `Optional`：

```java
Optional<String> name = Optional.of("Java");
```

创建一个空的 `Optional`：

```java
Optional<String> name = Optional.empty();
```

### 创建 Optional

常用创建方式有 3 个：

* `Optional.of(value)`
* `Optional.ofNullable(value)`
* `Optional.empty()`

### Optional.of

`Optional.of` 用来包装一个确定不为 `null` 的值。

```java
Optional<String> name = Optional.of("Java");

System.out.println(name);
```

输出：

```text
Optional[Java]
```

如果传入 `null`：

```java
Optional<String> name = Optional.of(null);
```

会直接抛出：

```text
NullPointerException
```

所以 `of` 适合这种场景：

```text
值一定不为空，只是需要包装成 Optional。
```

### Optional.ofNullable

`Optional.ofNullable` 是最常见的创建方式。

它允许传入 `null`。

```java
String value = null;

Optional<String> name = Optional.ofNullable(value);

System.out.println(name);
```

输出：

```text
Optional.empty
```

如果传入非空值：

```java
String value = "Java";

Optional<String> name = Optional.ofNullable(value);

System.out.println(name);
```

输出：

```text
Optional[Java]
```

日常项目里，包装一个来源不确定的对象时，通常使用 `ofNullable`。

### Optional.empty

`Optional.empty` 用来创建一个空的 `Optional`。

```java
Optional<User> user = Optional.empty();
```

它常用于方法返回空结果：

```java
public Optional<User> findUserById(Long id) {
    if (id == null) {
        return Optional.empty();
    }

    User user = queryUser(id);
    return Optional.ofNullable(user);
}
```

### 判断是否有值

`Optional` 提供了两个常见判断方法：

* `isPresent()`
* `isEmpty()`

`isPresent()` 表示是否有值：

```java
Optional<String> name = Optional.of("Java");

System.out.println(name.isPresent());
```

输出：

```text
true
```

`isEmpty()` 表示是否为空，它是 `Java 11` 新增的方法：

```java
Optional<String> name = Optional.empty();

System.out.println(name.isEmpty());
```

输出：

```text
true
```

判断后再 `get` 的写法可以用，但不算 `Optional` 最有价值的写法：

```java
Optional<String> name = Optional.of("Java");

if (name.isPresent()) {
    System.out.println(name.get());
}
```

更常见的写法是直接使用 `ifPresent`、`map`、`orElse` 等方法。

### get 取值

`get()` 可以直接取出 `Optional` 里的值。

```java
Optional<String> name = Optional.of("Java");

System.out.println(name.get());
```

输出：

```text
Java
```

但空 `Optional` 调用 `get()` 会抛异常：

```java
Optional<String> name = Optional.empty();

System.out.println(name.get());
```

异常：

```text
NoSuchElementException: No value present
```

所以实际项目里更常见的是先明确空值处理方式，而不是直接写：

```java
optional.get()
```

更合适的方式是：

```java
optional.orElse(...)
optional.orElseGet(...)
optional.orElseThrow(...)
optional.ifPresent(...)
```

### ifPresent：有值才执行

`ifPresent` 适合处理“有值就做一件事，没值就不处理”的场景。

```java
Optional<String> name = Optional.ofNullable("Java");

name.ifPresent(value -> System.out.println("name = " + value));
```

输出：

```text
name = Java
```

如果是空值：

```java
Optional<String> name = Optional.empty();

name.ifPresent(value -> System.out.println("name = " + value));
```

不会输出任何内容。

### ifPresentOrElse

`ifPresentOrElse` 是 `Java 9` 新增的方法。

它可以同时处理有值和无值两种情况。

```java
Optional<String> name = Optional.empty();

name.ifPresentOrElse(
        value -> System.out.println("name = " + value),
        () -> System.out.println("name 不存在")
);
```

输出：

```text
name 不存在
```

### orElse：为空时返回默认值

`orElse` 用来给空值设置默认值。

```java
String name = Optional.ofNullable(null)
        .orElse("默认名称");

System.out.println(name);
```

输出：

```text
默认名称
```

有值时返回原值：

```java
String name = Optional.ofNullable("Java")
        .orElse("默认名称");

System.out.println(name);
```

输出：

```text
Java
```

### orElseGet：为空时再生成默认值

`orElseGet` 接收的是一个 `Supplier`。

只有 `Optional` 为空时，才会执行里面的逻辑。

```java
String name = Optional.ofNullable(null)
        .orElseGet(() -> "默认名称");

System.out.println(name);
```

输出：

```text
默认名称
```

### orElse 和 orElseGet 的区别

这两个方法很像，但执行时机不一样。

准备一个方法：

```java
private static String createDefaultName() {
    System.out.println("生成默认名称");
    return "默认名称";
}
```

使用 `orElse`：

```java
String name = Optional.of("Java")
        .orElse(createDefaultName());

System.out.println(name);
```

输出：

```text
生成默认名称
Java
```

虽然 `Optional` 里已经有 `"Java"`，但 `createDefaultName()` 还是执行了。

再看 `orElseGet`：

```java
String name = Optional.of("Java")
        .orElseGet(() -> createDefaultName());

System.out.println(name);
```

输出：

```text
Java
```

`createDefaultName()` 没有执行。

简单理解：

```text
orElse：默认值先准备好，不管最后用不用
orElseGet：真正为空时，才去生成默认值
```

如果默认值只是一个普通字符串，用 `orElse` 很自然：

```java
String name = optionalName.orElse("匿名用户");
```

如果默认值需要查询数据库、调用接口、创建复杂对象，用 `orElseGet` 更合适：

```java
User user = optionalUser.orElseGet(() -> userRepository.createGuestUser());
```

### orElseThrow：为空时抛异常

有些场景下，空值不是正常结果，而是业务异常。

比如订单必须存在：

```java
Order order = orderRepository.findById(orderId)
        .orElseThrow(() -> new IllegalArgumentException("订单不存在"));
```

`Java 10` 开始，`orElseThrow()` 可以不传参数。

```java
String name = Optional.<String>empty()
        .orElseThrow();
```

空值时会抛出：

```text
NoSuchElementException
```

业务代码里通常更推荐带上明确的异常类型和提示信息：

```java
User user = userRepository.findById(userId)
        .orElseThrow(() -> new IllegalStateException("用户不存在"));
```

### map：转换里面的值

`map` 用来把 `Optional` 里的值转换成另一个值。

```java
Optional<String> name = Optional.of("Java");

Optional<Integer> length = name.map(String::length);

System.out.println(length.orElse(0));
```

输出：

```text
4
```

如果原来的 `Optional` 是空的，`map` 不会执行：

```java
Optional<String> name = Optional.empty();

Optional<Integer> length = name.map(value -> {
    System.out.println("计算长度");
    return value.length();
});

System.out.println(length.orElse(0));
```

输出：

```text
0
```

不会输出 `计算长度`。

### map 处理对象属性

`map` 很适合用来安全获取对象属性。

```java
String city = Optional.ofNullable(user)
        .map(User::getAddress)
        .map(Address::getCity)
        .orElse("未知城市");
```

这段代码里：

* `user` 为空，直接返回默认值
* `address` 为空，直接返回默认值
* `city` 为空，直接返回默认值
* 三者都有值，返回真实城市

不用写多层 `if`，也不用担心中间某一步出现空指针。

### flatMap：处理返回 Optional 的方法

如果转换方法本身返回的就是 `Optional`，应该使用 `flatMap`。

先看一个类：

```java
class User {
    private final Address address;

    User(Address address) {
        this.address = address;
    }

    public Optional<Address> getAddress() {
        return Optional.ofNullable(address);
    }
}
```

此时 `getAddress()` 返回的是：

```java
Optional<Address>
```

如果使用 `map`：

```java
Optional<Optional<Address>> address = Optional.of(user)
        .map(User::getAddress);
```

结果会变成嵌套结构：

```text
Optional<Optional<Address>>
```

使用 `flatMap` 可以把嵌套压平：

```java
Optional<Address> address = Optional.of(user)
        .flatMap(User::getAddress);
```

再配合城市字段：

```java
String city = Optional.of(user)
        .flatMap(User::getAddress)
        .flatMap(Address::getCity)
        .orElse("未知城市");
```

简单区分：

```text
方法返回普通值，用 map
方法返回 Optional，用 flatMap
```

### filter：按条件保留值

`filter` 用来给 `Optional` 里的值加条件。

条件满足，值保留。

条件不满足，变成空 `Optional`。

```java
Optional<Integer> age = Optional.of(20);

Optional<Integer> adultAge = age.filter(value -> value >= 18);

System.out.println(adultAge.isPresent());
```

输出：

```text
true
```

如果条件不满足：

```java
Optional<Integer> age = Optional.of(16);

Optional<Integer> adultAge = age.filter(value -> value >= 18);

System.out.println(adultAge.isPresent());
```

输出：

```text
false
```

常见业务写法：

```java
Optional<User> activeUser = Optional.ofNullable(user)
        .filter(User::isActive);
```

表示：

```text
user 不为空，并且是启用状态，才继续保留。
```

### or：为空时切换到另一个 Optional

`or` 是 `Java 9` 新增的方法。

它适合处理多个来源的查找逻辑。

```java
Optional<User> user = findByEmail(email)
        .or(() -> findByPhone(phone))
        .or(() -> findByUsername(username));
```

含义是：

```text
先按邮箱查
查不到再按手机号查
还查不到再按用户名查
```

注意，`or` 返回的还是 `Optional`。

```java
User result = findByEmail(email)
        .or(() -> findByPhone(phone))
        .or(() -> findByUsername(username))
        .orElseThrow(() -> new IllegalArgumentException("用户不存在"));
```

### stream：把 Optional 转成 Stream

`stream` 是 `Java 9` 新增的方法。

它可以把一个 `Optional` 转成包含 0 个或 1 个元素的 `Stream`。

```java
Optional<String> name = Optional.of("Java");

long count = name.stream().count();

System.out.println(count);
```

输出：

```text
1
```

空 `Optional`：

```java
Optional<String> name = Optional.empty();

long count = name.stream().count();

System.out.println(count);
```

输出：

```text
0
```

它常用于集合处理。

比如有一组用户 ID，需要查询用户，但有些 ID 查不到：

```java
List<User> users = userIds.stream()
        .map(userRepository::findById)
        .flatMap(Optional::stream)
        .collect(Collectors.toList());
```

这里的 `findById` 返回 `Optional<User>`。

`flatMap(Optional::stream)` 会自动丢掉空结果，只留下查到的用户。

### 实战 Demo：订单查询与金额展示

下面这组代码可以直接复制运行，演示 `Optional` 在业务代码里的常见用法。

```java
import java.math.BigDecimal;
import java.util.Arrays;
import java.util.List;
import java.util.Optional;

public class OptionalOrderDemo {

    public static void main(String[] args) {
        OrderService orderService = new OrderService();

        System.out.println(orderService.getBuyerName("A001"));
        System.out.println(orderService.getBuyerName("A002"));
        System.out.println(orderService.getBuyerName("A404"));

        System.out.println(orderService.getPaidAmountText("A001"));
        System.out.println(orderService.getPaidAmountText("A002"));
        System.out.println(orderService.getPaidAmountText("A003"));

        orderService.findOrder("A001")
                .filter(Order::isPaid)
                .ifPresent(order -> System.out.println("已支付订单: " + order.getOrderNo()));

        try {
            Order order = orderService.findOrder("A404")
                    .orElseThrow(() -> new IllegalArgumentException("订单不存在"));
        } catch (IllegalArgumentException e) {
            System.out.println("异常信息: " + e.getMessage());
        }
    }

    static class OrderService {
        private final List<Order> orders = Arrays.asList(
                new Order("A001", new Buyer("张三", new Address("上海")), new BigDecimal("99.00"), "PAID"),
                new Order("A002", new Buyer("李四", null), new BigDecimal("35.50"), "PAID"),
                new Order("A003", null, new BigDecimal("88.00"), "CANCELLED")
        );

        public Optional<Order> findOrder(String orderNo) {
            return orders.stream()
                    .filter(order -> order.getOrderNo().equals(orderNo))
                    .findFirst();
        }

        public String getBuyerName(String orderNo) {
            return findOrder(orderNo)
                    .map(Order::getBuyer)
                    .map(Buyer::getName)
                    .orElse("未知买家");
        }

        public String getPaidAmountText(String orderNo) {
            return findOrder(orderNo)
                    .filter(Order::isPaid)
                    .map(Order::getAmount)
                    .map(amount -> "支付金额: " + amount)
                    .orElse("未支付或订单不存在");
        }

        public String getBuyerCity(String orderNo) {
            return findOrder(orderNo)
                    .map(Order::getBuyer)
                    .map(Buyer::getAddress)
                    .map(Address::getCity)
                    .orElse("未知城市");
        }
    }

    static class Order {
        private final String orderNo;
        private final Buyer buyer;
        private final BigDecimal amount;
        private final String status;

        Order(String orderNo, Buyer buyer, BigDecimal amount, String status) {
            this.orderNo = orderNo;
            this.buyer = buyer;
            this.amount = amount;
            this.status = status;
        }

        String getOrderNo() {
            return orderNo;
        }

        Buyer getBuyer() {
            return buyer;
        }

        BigDecimal getAmount() {
            return amount;
        }

        boolean isPaid() {
            return "PAID".equals(status);
        }
    }

    static class Buyer {
        private final String name;
        private final Address address;

        Buyer(String name, Address address) {
            this.name = name;
            this.address = address;
        }

        String getName() {
            return name;
        }

        Address getAddress() {
            return address;
        }
    }

    static class Address {
        private final String city;

        Address(String city) {
            this.city = city;
        }

        String getCity() {
            return city;
        }
    }
}
```

运行结果：

```text
张三
李四
未知买家
支付金额: 99.00
支付金额: 35.50
未支付或订单不存在
已支付订单: A001
异常信息: 订单不存在
```

这个 Demo 里包含了几个典型场景：

* `findOrder`：用 `Optional<Order>` 表达订单可能不存在
* `getBuyerName`：用 `map` 安全获取买家姓名
* `getPaidAmountText`：用 `filter` 判断订单状态
* `orElse`：为空或条件不满足时给默认文案
* `orElseThrow`：订单必须存在时抛出业务异常

### 实战 Demo：查询不到时再走备用查询

实际项目里，经常会出现多个查询入口。

比如先按邮箱查用户，查不到再按手机号查。

```java
import java.util.Arrays;
import java.util.List;
import java.util.Optional;

public class OptionalFallbackDemo {

    public static void main(String[] args) {
        UserService userService = new UserService();

        User user = userService.findByEmail("none@example.com")
                .or(() -> userService.findByPhone("13800000000"))
                .orElseThrow(() -> new IllegalArgumentException("用户不存在"));

        System.out.println(user.getName());
    }

    static class UserService {
        private final List<User> users = Arrays.asList(
                new User("张三", "zhangsan@example.com", "13800000000"),
                new User("李四", "lisi@example.com", "13900000000")
        );

        public Optional<User> findByEmail(String email) {
            return users.stream()
                    .filter(user -> user.getEmail().equals(email))
                    .findFirst();
        }

        public Optional<User> findByPhone(String phone) {
            return users.stream()
                    .filter(user -> user.getPhone().equals(phone))
                    .findFirst();
        }
    }

    static class User {
        private final String name;
        private final String email;
        private final String phone;

        User(String name, String email, String phone) {
            this.name = name;
            this.email = email;
            this.phone = phone;
        }

        String getName() {
            return name;
        }

        String getEmail() {
            return email;
        }

        String getPhone() {
            return phone;
        }
    }
}
```

输出：

```text
张三
```

这类写法比手动判断更紧凑：

```java
Optional<User> user = findByEmail(email);

if (user.isEmpty()) {
    user = findByPhone(phone);
}
```

### Optional 和 Stream 的配合

`Stream` 里的很多方法本身就会返回 `Optional`。

比如 `findFirst`：

```java
Optional<String> first = Arrays.asList("Java", "Kotlin", "Go")
        .stream()
        .filter(name -> name.startsWith("K"))
        .findFirst();

System.out.println(first.orElse("没有匹配结果"));
```

输出：

```text
Kotlin
```

再比如求最大值：

```java
Optional<Integer> max = Arrays.asList(10, 20, 30)
        .stream()
        .max(Integer::compareTo);

System.out.println(max.orElse(0));
```

输出：

```text
30
```

为什么这些方法返回 `Optional`？

因为结果可能不存在。

```java
Optional<String> first = List.<String>of()
        .stream()
        .findFirst();
```

空集合里没有第一个元素，所以返回空 `Optional`。

### OptionalInt、OptionalLong、OptionalDouble

除了 `Optional<T>`，Java 还提供了 3 个基本类型版本：

* `OptionalInt`
* `OptionalLong`
* `OptionalDouble`

它们用来避免基本类型装箱。

```java
import java.util.OptionalInt;

public class OptionalIntDemo {
    public static void main(String[] args) {
        OptionalInt maxAge = OptionalInt.of(30);

        System.out.println(maxAge.orElse(0));
    }
}
```

输出：

```text
30
```

`IntStream` 的一些操作也会返回 `OptionalInt`：

```java
OptionalInt max = Arrays.asList(10, 20, 30)
        .stream()
        .mapToInt(Integer::intValue)
        .max();

System.out.println(max.orElse(0));
```

输出：

```text
30
```

### 常见使用建议

### 适合作为方法返回值

`Optional` 最适合放在方法返回值上。

```java
public Optional<User> findById(Long id) {
    User user = queryUser(id);
    return Optional.ofNullable(user);
}
```

调用方看到返回值类型，就知道这个方法可能查不到数据。

```java
User user = userRepository.findById(id)
        .orElseThrow(() -> new IllegalArgumentException("用户不存在"));
```

### 不适合作为实体字段

实体字段通常不建议写成 `Optional`。

```java
public class User {
    private Optional<String> nickname;
}
```

更常见的写法是保持字段为普通类型：

```java
public class User {
    private String nickname;

    public Optional<String> getNickname() {
        return Optional.ofNullable(nickname);
    }
}
```

原因很简单：

```text
字段表示数据本身，Optional 更适合表达方法返回结果是否存在。
```

对于数据库实体、JSON 序列化、ORM 映射，这种写法也更自然。

### 不适合作为方法参数

方法参数也通常不建议写成 `Optional`。

```java
public void updateNickname(Long userId, Optional<String> nickname) {
}
```

调用这个方法时，反而多了一层包装：

```java
updateNickname(1L, Optional.of("小张"));
updateNickname(1L, Optional.empty());
```

更常见的写法是传普通参数：

```java
public void updateNickname(Long userId, String nickname) {
}
```

如果参数有明确的业务含义，可以拆成更清楚的方法：

```java
public void updateNickname(Long userId, String nickname) {
}

public void clearNickname(Long userId) {
}
```

### 不适合放进集合

集合里通常不建议放 `Optional`。

```java
List<Optional<User>> users;
```

更常见的是：

```java
List<User> users;
```

如果没有数据，用空集合表示：

```java
List<User> users = Collections.emptyList();
```

如果集合中某些元素可能不存在，一般在收集前处理掉空值：

```java
List<User> users = userIds.stream()
        .map(userRepository::findById)
        .flatMap(Optional::stream)
        .collect(Collectors.toList());
```

### get 的使用边界

`get()` 只有在已经确定有值时才安全。

```java
if (optional.isPresent()) {
    User user = optional.get();
}
```

但大多数场景可以改成更直接的写法。

有默认值：

```java
User user = optional.orElse(defaultUser);
```

空值时报错：

```java
User user = optional.orElseThrow(() -> new IllegalArgumentException("用户不存在"));
```

有值时执行逻辑：

```java
optional.ifPresent(user -> sendMessage(user));
```

转换字段：

```java
String name = optional.map(User::getName).orElse("匿名用户");
```

### 链式调用不宜过长

`Optional` 支持链式调用，但业务逻辑很复杂时，拆成普通变量会更清楚。

比如这类代码：

```java
String result = Optional.ofNullable(order)
        .map(Order::buyer)
        .filter(Buyer::isVip)
        .map(Buyer::address)
        .map(Address::city)
        .filter(city -> city.startsWith("上"))
        .map(city -> city + " VIP")
        .orElse("普通订单");
```

如果中间夹杂很多业务规则，拆开后通常更容易维护：

```java
Optional<Order> orderOptional = Optional.ofNullable(order);

Optional<Buyer> vipBuyer = orderOptional
        .map(Order::buyer)
        .filter(Buyer::isVip);

String result = vipBuyer
        .map(Buyer::address)
        .map(Address::city)
        .filter(city -> city.startsWith("上"))
        .map(city -> city + " VIP")
        .orElse("普通订单");
```

链式调用适合表达清晰的数据流。

逻辑复杂时，适当拆开，变量名本身就是说明。

### 常用方法汇总

| 方法 | 作用 | 常见场景 |
| --- | --- | --- |
| `Optional.of(value)` | 创建有值的 Optional，值不能为 null | 明确值不为空 |
| `Optional.ofNullable(value)` | 创建可能为空的 Optional | 包装外部返回值 |
| `Optional.empty()` | 创建空 Optional | 返回空结果 |
| `isPresent()` | 判断是否有值 | 简单分支判断 |
| `isEmpty()` | 判断是否为空 | Java 11+ |
| `ifPresent(...)` | 有值时执行逻辑 | 打印、通知、补充操作 |
| `ifPresentOrElse(...)` | 有值和无值分别处理 | Java 9+ |
| `orElse(...)` | 为空时返回默认值 | 默认字符串、默认数字 |
| `orElseGet(...)` | 为空时执行 Supplier 获取默认值 | 默认值创建成本较高 |
| `orElseThrow(...)` | 为空时抛异常 | 必须存在的数据 |
| `map(...)` | 转换内部值 | 取字段、格式化 |
| `flatMap(...)` | 转换返回 Optional 的值 | 避免 `Optional<Optional<T>>` |
| `filter(...)` | 条件过滤 | 状态判断、权限判断 |
| `or(...)` | 为空时切换到另一个 Optional | Java 9+，备用查询 |
| `stream()` | 转成 0 或 1 个元素的 Stream | Java 9+，集合流水线 |

### 总结

`Optional` 的重点不是把所有 `null` 都消灭掉，而是把“结果可能不存在”这件事表达清楚。

最常见的使用流程是：

```java
Optional.ofNullable(value)
        .map(...)
        .filter(...)
        .orElse(...);
```

或者：

```java
repository.findById(id)
        .orElseThrow(() -> new IllegalArgumentException("数据不存在"));
```

实际项目里，`Optional` 最适合放在方法返回值上，用来表示查询结果、匹配结果、计算结果可能为空。

只要控制好边界，不把它塞进实体字段、方法参数和集合元素里，代码会更容易看出空值处理逻辑。
