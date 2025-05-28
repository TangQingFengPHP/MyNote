### 简介

`PHP` 设计模式是对软件开发中常见问题的可复用解决方案，通过标准化的结构提升代码的可维护性、扩展性和复用性。

### 创建型模式（对象创建）

关注对象的创建过程，解决 “如何灵活、安全地生成对象” 的问题。

#### 单例模式（Singleton）

意图：确保一个类仅有一个实例，并提供全局访问点。
适用场景：全局配置、数据库连接池、日志管理器（需共享状态）。

实现要点：

* 私有构造函数（禁止外部实例化）。

* 静态变量保存唯一实例。

* 静态方法返回实例（延迟初始化）。

```php
class DatabaseConnection {
    // 保存唯一实例
    private static ?self $instance = null;
    private string $connection;

    // 私有构造函数（防止外部 new）
    private function __construct(string $dsn) {
        $this->connection = "Connected to: $dsn";
    }

    // 静态方法获取实例（延迟初始化）
    public static function getInstance(string $dsn = 'mysql:host=localhost'): self {
        if (self::$instance === null) {
            self::$instance = new self($dsn);
        }
        return self::$instance;
    }

    // 禁止克隆（防止复制）
    private function __clone() {}

    // 禁止反序列化（防止外部重建实例）
    public function __wakeup() {
        throw new \Exception("Singleton cannot be deserialized");
    }

    public function getConnection(): string {
        return $this->connection;
    }
}

// 使用示例
$db1 = DatabaseConnection::getInstance();
$db2 = DatabaseConnection::getInstance('mysql:host=backup');
var_dump($db1 === $db2);  // 输出：bool(true)（始终返回同一个实例）
```

#### 工厂模式（Factory Method）

意图：定义创建对象的接口，让子类决定实例化哪个类。
适用场景：对象创建逻辑复杂（如依赖配置、外部服务），需解耦对象创建与使用。

实现要点：

* 抽象工厂类：声明创建对象的方法（`createProduct()`）。

* 具体工厂类：实现抽象方法，返回具体产品。

* 抽象产品类：定义产品的公共接口。

```php
// 抽象产品：日志记录器
interface Logger {
    public function log(string $message): void;
}

// 具体产品：文件日志
class FileLogger implements Logger {
    public function log(string $message): void {
        file_put_contents('app.log', $message . PHP_EOL, FILE_APPEND);
    }
}

// 具体产品：数据库日志
class DbLogger implements Logger {
    public function log(string $message): void {
        // 模拟写入数据库
        echo "Logged to DB: $message" . PHP_EOL;
    }
}

// 抽象工厂：日志工厂
abstract class LoggerFactory {
    abstract public function createLogger(): Logger;
}

// 具体工厂：文件日志工厂
class FileLoggerFactory extends LoggerFactory {
    public function createLogger(): Logger {
        return new FileLogger();
    }
}

// 具体工厂：数据库日志工厂
class DbLoggerFactory extends LoggerFactory {
    public function createLogger(): Logger {
        return new DbLogger();
    }
}

// 使用示例
$factory = new FileLoggerFactory();  // 切换工厂即可更换日志类型
$logger = $factory->createLogger();
$logger->log("User logged in");
```

优势：客户端只需知道工厂类型，无需关心具体产品的创建细节，便于扩展新的产品类型。

#### 建造者模式（Builder）

意图：将复杂对象的构建过程与表示分离，允许通过不同步骤构建不同表示。
适用场景：对象包含多个可选属性（如配置对象、复杂实体类）。

实现要点：

* 产品类：包含复杂属性的实体。

* 抽象建造者：定义构建步骤（`setPartA()/setPartB()`）。

* 具体建造者：实现构建步骤，返回产品。

* 指导者（可选）：定义构建顺序，调用建造者方法。

```php
// 产品类：用户配置
class UserConfig {
    public string $theme;
    public bool $notifications;
    public int $timeout;

    public function __toString(): string {
        return "Theme: $this->theme, Notifications: " . 
               ($this->notifications ? 'On' : 'Off') . 
               ", Timeout: $this->timeout";
    }
}

// 抽象建造者
interface UserConfigBuilder {
    public function setTheme(string $theme): self;
    public function enableNotifications(bool $enable): self;
    public function setTimeout(int $seconds): self;
    public function build(): UserConfig;
}

// 具体建造者
class DefaultUserConfigBuilder implements UserConfigBuilder {
    private UserConfig $config;

    public function __construct() {
        $this->reset();
    }

    private function reset(): void {
        $this->config = new UserConfig();
        $this->config->theme = 'light';  // 默认主题
        $this->config->notifications = true;  // 默认开启通知
        $this->config->timeout = 300;  // 默认超时5分钟
    }

    public function setTheme(string $theme): self {
        $this->config->theme = $theme;
        return $this;
    }

    public function enableNotifications(bool $enable): self {
        $this->config->notifications = $enable;
        return $this;
    }

    public function setTimeout(int $seconds): self {
        $this->config->timeout = $seconds;
        return $this;
    }

    public function build(): UserConfig {
        $result = $this->config;
        $this->reset();  // 重置以便下次构建
        return $result;
    }
}

// 使用示例
$builder = new DefaultUserConfigBuilder();
$config = $builder
    ->setTheme('dark')
    ->enableNotifications(false)
    ->setTimeout(600)
    ->build();

echo $config;  // 输出：Theme: dark, Notifications: Off, Timeout: 600
```

优势：避免构造函数参数过多（如 `__construct($theme, $notifications, $timeout)`），支持链式调用，提升代码可读性。

### 结构型模式（类 / 对象组合）

关注类和对象的组合方式，解决 “如何灵活组织类结构” 的问题。

#### 适配器模式（Adapter）

意图：将不兼容的接口转换为客户端期望的接口，解决类 / 对象间的接口不匹配问题。
适用场景：复用第三方库（接口不兼容）、旧系统升级（兼容新旧接口）。

实现要点：

* 目标接口：客户端期望的接口（`Target`）。

* 适配者：需要被适配的类（`Adaptee`，通常是第三方或旧代码）。

* 适配器：包装适配者，实现目标接口。

```php
// 目标接口（客户端期望的日志格式）
interface ModernLogger {
    public function log(string $level, string $message): void;
}

// 适配者（旧日志类，接口不兼容）
class LegacyLogger {
    public function write(string $message): void {
        file_put_contents('legacy.log', $message . PHP_EOL, FILE_APPEND);
    }
}

// 适配器：将 LegacyLogger 转换为 ModernLogger 接口
class LegacyLoggerAdapter implements ModernLogger {
    private LegacyLogger $adaptee;

    public function __construct(LegacyLogger $adaptee) {
        $this->adaptee = $adaptee;
    }

    public function log(string $level, string $message): void {
        // 转换格式：将 level + message 拼接为旧日志格式
        $this->adaptee->write("[$level] $message");
    }
}

// 使用示例
$legacyLogger = new LegacyLogger();
$adapter = new LegacyLoggerAdapter($legacyLogger);
$adapter->log('ERROR', 'Database connection failed');  // 实际调用 LegacyLogger::write
```

优势：无需修改旧代码，通过适配器兼容新接口，符合 “开闭原则”。

#### 装饰器模式（Decorator）

意图：动态为对象添加额外功能，替代继承实现灵活扩展。
适用场景：需要为对象添加多个可选功能（如日志记录、权限校验、缓存）。

实现要点：

* 抽象组件：定义核心功能接口（`Component`）。

* 具体组件：实现核心功能（`ConcreteComponent`）。

* 装饰器：包装组件，添加额外功能（`Decorator`）。

```php
// 抽象组件：文件存储接口
interface FileStorage {
    public function save(string $path, string $content): void;
    public function read(string $path): string;
}

// 具体组件：基础文件存储
class BasicFileStorage implements FileStorage {
    public function save(string $path, string $content): void {
        file_put_contents($path, $content);
    }

    public function read(string $path): string {
        return file_get_contents($path) ?: '';
    }
}

// 装饰器：添加日志功能
class LoggedFileStorage implements FileStorage {
    private FileStorage $storage;

    public function __construct(FileStorage $storage) {
        $this->storage = $storage;
    }

    public function save(string $path, string $content): void {
        echo "Log: Saving file $path" . PHP_EOL;
        $this->storage->save($path, $content);
    }

    public function read(string $path): string {
        echo "Log: Reading file $path" . PHP_EOL;
        return $this->storage->read($path);
    }
}

// 装饰器：添加加密功能
class EncryptedFileStorage implements FileStorage {
    private FileStorage $storage;
    private string $key = 'secret_key';

    public function __construct(FileStorage $storage) {
        $this->storage = $storage;
    }

    public function save(string $path, string $content): void {
        $encrypted = openssl_encrypt($content, 'AES-256-CBC', $this->key);
        $this->storage->save($path, $encrypted);
    }

    public function read(string $path): string {
        $encrypted = $this->storage->read($path);
        return openssl_decrypt($encrypted, 'AES-256-CBC', $this->key) ?: '';
    }
}

// 使用示例（组合装饰器）
$storage = new BasicFileStorage();
$loggedStorage = new LoggedFileStorage($storage);
$secureStorage = new EncryptedFileStorage($loggedStorage);  // 先加密，再记录日志

$secureStorage->save('data.txt', '敏感信息');
echo $secureStorage->read('data.txt');  // 输出：敏感信息（解密后）
```

优势：通过组合而非继承添加功能，避免类爆炸（如 `EncryptedLoggedFileStorage` 无需单独定义）。

#### 代理模式（Proxy）

意图：为对象提供一个代理，控制对原对象的访问（如延迟加载、权限校验）。
适用场景：远程服务代理（`RPC`）、资源消耗大的对象（延迟加载）、权限控制。

实现要点：

* 主题接口：定义原对象和代理的公共接口（`Subject`）。

* 真实主题：实际提供功能的对象（`RealSubject`）。

* 代理：包装真实主题，添加控制逻辑（`Proxy`）。

```php
// 主题接口：用户信息
interface UserInfo {
    public function getProfile(): string;
}

// 真实主题：实际从数据库加载用户信息
class RealUserInfo implements UserInfo {
    private int $userId;

    public function __construct(int $userId) {
        $this->userId = $userId;
        // 模拟耗时的数据库查询（仅在需要时执行）
        sleep(1);  // 假设加载需1秒
    }

    public function getProfile(): string {
        return "User $this->userId: 姓名=张三, 邮箱=zhangsan@example.com";
    }
}

// 代理：延迟加载真实对象
class UserInfoProxy implements UserInfo {
    private ?RealUserInfo $realUser = null;
    private int $userId;

    public function __construct(int $userId) {
        $this->userId = $userId;
    }

    public function getProfile(): string {
        // 延迟加载：仅在需要时创建真实对象
        if ($this->realUser === null) {
            $this->realUser = new RealUserInfo($this->userId);
        }
        return $this->realUser->getProfile();
    }
}

// 使用示例
$proxy = new UserInfoProxy(1001);
echo "开始获取用户信息..." . PHP_EOL;
echo $proxy->getProfile();  // 此时才会触发真实对象的加载（延迟1秒）
```

优势：减少资源消耗（如避免启动时加载所有大对象），提升系统响应速度。

### 行为型模式（对象交互）

关注对象间的通信与协作，解决 “如何灵活设计对象行为” 的问题。

#### 观察者模式（Observer）

意图：定义对象间的一对多依赖关系，当一个对象状态变化时，所有依赖它的对象自动更新。
适用场景：事件通知（如用户注册后发送邮件 / 短信）、状态监控（如库存变化触发提醒）。

实现要点：

* 主题（`Subject`）：维护观察者列表，提供注册 / 移除接口，状态变化时通知所有观察者。

* 观察者（`Observer`）：定义更新接口，接收主题通知。

```php
// 主题接口
interface Subject {
    public function attach(Observer $observer): void;
    public function detach(Observer $observer): void;
    public function notify(): void;
}

// 观察者接口
interface Observer {
    public function update(string $event, mixed $data): void;
}

// 具体主题：用户服务（触发注册事件）
class UserService implements Subject {
    private array $observers = [];

    public function attach(Observer $observer): void {
        $this->observers[] = $observer;
    }

    public function detach(Observer $observer): void {
        $this->observers = array_filter($this->observers, fn($o) => $o !== $observer);
    }

    public function notify(): void {
        foreach ($this->observers as $observer) {
            $observer->update('user.register', $this->userData);
        }
    }

    private array $userData;

    public function registerUser(string $name, string $email): void {
        $this->userData = ['name' => $name, 'email' => $email];
        $this->notify();  // 注册后通知观察者
    }
}

// 具体观察者：邮件通知
class EmailNotifier implements Observer {
    public function update(string $event, mixed $data): void {
        if ($event === 'user.register') {
            $email = $data['email'];
            echo "发送注册邮件到：$email" . PHP_EOL;
        }
    }
}

// 具体观察者：短信通知
class SmsNotifier implements Observer {
    public function update(string $event, mixed $data): void {
        if ($event === 'user.register') {
            $phone = '13800138000';  // 假设从用户数据获取手机号
            echo "发送注册短信到：$phone" . PHP_EOL;
        }
    }
}

// 使用示例
$userService = new UserService();
$userService->attach(new EmailNotifier());
$userService->attach(new SmsNotifier());

$userService->registerUser('张三', 'zhangsan@example.com');
// 输出：
// 发送注册邮件到：zhangsan@example.com
// 发送注册短信到：13800138000
```

优势：解耦主题与观察者，支持动态添加 / 移除观察者（如新增微信通知仅需实现 `Observer` 接口）。

#### 策略模式（Strategy）

意图：定义一系列算法，将它们封装为独立对象，使算法可互换。
适用场景：同一功能有多种实现方式（如支付方式、排序算法），需动态切换。

实现要点：

* 策略接口：定义算法的公共方法（`Strategy`）。

* 具体策略：实现策略接口（`ConcreteStrategyA/ConcreteStrategyB`）。
上下文：持有策略对象，委托算法执行（`Context`）。

```php
// 策略接口：支付方式
interface PaymentStrategy {
    public function pay(float $amount): string;
}

// 具体策略：支付宝支付
class AlipayStrategy implements PaymentStrategy {
    public function pay(float $amount): string {
        return "通过支付宝支付 $amount 元";
    }
}

// 具体策略：微信支付
class WechatPayStrategy implements PaymentStrategy {
    public function pay(float $amount): string {
        return "通过微信支付 $amount 元";
    }
}

// 上下文：订单处理
class OrderContext {
    private PaymentStrategy $paymentStrategy;

    public function setPaymentStrategy(PaymentStrategy $strategy): void {
        $this->paymentStrategy = $strategy;
    }

    public function processPayment(float $amount): string {
        return $this->paymentStrategy->pay($amount);
    }
}

// 使用示例
$order = new OrderContext();

// 选择支付宝支付
$order->setPaymentStrategy(new AlipayStrategy());
echo $order->processPayment(99.9);  // 输出：通过支付宝支付 99.9 元

// 动态切换为微信支付
$order->setPaymentStrategy(new WechatPayStrategy());
echo $order->processPayment(199.9);  // 输出：通过微信支付 199.9 元
```

优势：算法独立于上下文，支持运行时动态切换，避免大量 `if-else` 判断。

#### 模板方法模式（Template Method）

意图：定义算法的骨架，将具体步骤延迟到子类实现。
适用场景：多个子类有公共步骤（如初始化→执行→清理），需统一流程。

实现要点：

* 抽象类：定义模板方法（包含固定步骤）和抽象方法（由子类实现）。

* 具体子类：实现抽象方法，完成具体步骤。

```php
// 抽象类：数据导入流程
abstract class DataImporter {
    // 模板方法（定义固定流程）
    public final function import(string $filePath): void {
        $this->validateFile($filePath);
        $data = $this->readFile($filePath);
        $this->processData($data);
        $this->cleanup();
    }

    // 固定步骤（通用实现）
    protected function validateFile(string $filePath): void {
        if (!file_exists($filePath)) {
            throw new \Exception("文件 $filePath 不存在");
        }
    }

    // 抽象步骤（由子类实现）
    abstract protected function readFile(string $filePath): array;
    abstract protected function processData(array $data): void;

    // 可选步骤（钩子方法）
    protected function cleanup(): void {
        // 默认无操作，子类可重写
    }
}

// 具体子类：CSV 数据导入
class CsvImporter extends DataImporter {
    protected function readFile(string $filePath): array {
        $handle = fopen($filePath, 'r');
        $data = [];
        while (($row = fgetcsv($handle)) !== false) {
            $data[] = $row;
        }
        fclose($handle);
        return $data;
    }

    protected function processData(array $data): void {
        echo "处理 CSV 数据：" . count($data) . " 行" . PHP_EOL;
    }
}

// 具体子类：Excel 数据导入
class ExcelImporter extends DataImporter {
    protected function readFile(string $filePath): array {
        // 假设使用 PhpSpreadsheet 读取 Excel
        $spreadsheet = \PhpOffice\PhpSpreadsheet\IOFactory::load($filePath);
        $worksheet = $spreadsheet->getActiveSheet();
        return $worksheet->toArray();
    }

    protected function processData(array $data): void {
        echo "处理 Excel 数据：" . count($data) . " 行" . PHP_EOL;
    }

    // 重写钩子方法（清理临时文件）
    protected function cleanup(): void {
        echo "清理 Excel 临时文件" . PHP_EOL;
    }
}

// 使用示例
$csvImporter = new CsvImporter();
$csvImporter->import('data.csv');  // 输出：处理 CSV 数据：100 行

$excelImporter = new ExcelImporter();
$excelImporter->import('data.xlsx');  // 输出：处理 Excel 数据：200 行；清理 Excel 临时文件
```

优势：统一流程控制，子类仅需关注具体步骤，避免代码重复。

#### 责任链模式（Chain of Responsibility）

意图：将请求的发送者与接收者解耦，使多个对象都有机会处理请求（形成链式传递）。
适用场景：多级审批（如请假流程）、中间件（如请求过滤）。

实现要点：

* 处理者接口：定义处理请求和传递请求的方法（`Handler`）。

* 具体处理者：实现处理逻辑，决定是否传递请求（`ConcreteHandlerA/ConcreteHandlerB`）。

```php
// 处理者接口
interface RequestHandler {
    public function setNext(RequestHandler $handler): RequestHandler;
    public function handle(string $request): string;
}

// 抽象处理者（公共逻辑）
abstract class AbstractRequestHandler implements RequestHandler {
    private ?RequestHandler $nextHandler = null;

    public function setNext(RequestHandler $handler): RequestHandler {
        $this->nextHandler = $handler;
        return $handler;
    }

    public function handle(string $request): string {
        if ($this->nextHandler !== null) {
            return $this->nextHandler->handle($request);
        }
        return $request;
    }
}

// 具体处理者：敏感词过滤
class SensitiveWordFilter extends AbstractRequestHandler {
    public function handle(string $request): string {
        $filtered = str_replace(['非法', '违禁'], '***', $request);
        return parent::handle($filtered);  // 传递给下一个处理者
    }
}

// 具体处理者：HTML 转义
class HtmlEscapeFilter extends AbstractRequestHandler {
    public function handle(string $request): string {
        $escaped = htmlspecialchars($request, ENT_QUOTES);
        return parent::handle($escaped);
    }
}

// 具体处理者：日志记录
class LoggingFilter extends AbstractRequestHandler {
    public function handle(string $request): string {
        echo "记录请求：$request" . PHP_EOL;
        return parent::handle($request);
    }
}

// 使用示例
$chain = new SensitiveWordFilter();
$chain->setNext(new HtmlEscapeFilter())->setNext(new LoggingFilter());

$input = '用户输入：<script>非法内容</script>';
$result = $chain->handle($input);
echo "最终处理结果：$result";
// 输出：
// 记录请求：用户输入：<script>***内容</script>
// 最终处理结果：用户输入：&lt;script&gt;***内容&lt;/script&gt;
```

优势：请求处理逻辑解耦，支持动态调整处理顺序或添加新处理者。