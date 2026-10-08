# PHP Monolog 日志实战详解：级别、Handler、Processor 与 Symfony 配置

程序运行出错时，日志往往是还原现场最直接的线索。只用 `echo` 或 `var_dump()` 排查，开发时勉强够用，到了线上就很难按级别筛选、按请求追踪，也不适合把告警发到不同地方。

Monolog 是 PHP 生态里常用的日志库，实现了 PSR-3 日志接口。它能把日志写入文件、PHP 错误日志、Syslog 或其他服务，也能给日志补充请求 ID、用户 ID、内存用量等上下文。

这篇文章从一段可以运行的独立 PHP 示例开始，再讲日志级别、Handler、Processor、Formatter 和 Symfony 集成配置。

## Monolog 是什么？

Monolog 是负责记录和分发日志的 PHP 库。它实现 PSR-3 的 `Psr\Log\LoggerInterface`，因此业务代码可以依赖接口，而不必绑定某一个具体日志实现。

Monolog 的核心工作流可以概括成：

```text
业务代码调用 Logger
        ↓
日志记录经过 Processor 补充上下文
        ↓
Logger 按 Handler 栈传递记录
        ↓
Handler 按级别过滤并决定输出位置
        ↓
Formatter 把记录变成文本、JSON 等格式
```

几个核心概念：

- **Logger**：日志入口，通常带有一个 Channel 名称。
- **Handler**：决定日志写到哪里，以及哪些级别的日志会被接收。
- **Processor**：在输出前给日志补充或调整信息。
- **Formatter**：把日志记录转换成最终输出格式。
- **Channel**：标记日志来源，例如 `app`、`request`、`payment`。

Monolog 不是 Symfony EventDispatcher，也没有叫“事件监听器”的核心机制。Monolog 自己通过 Processor 和 Handler 管理日志记录；Symfony 的事件监听器可以调用 Monolog 写日志，但那是两个组件的配合。

## 安装 Monolog

独立 PHP 项目安装：

```bash
composer require monolog/monolog
```

Monolog 3.x 要求 PHP 8.1 或更高版本。Symfony 项目一般使用 MonologBundle 将 Monolog 接入服务容器和 YAML 配置：

```bash
composer require symfony/monolog-bundle
```

## 独立 PHP Demo：把日志写成 JSON 文件

创建 `app.php`：

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use Monolog\Formatter\JsonFormatter;
use Monolog\Handler\StreamHandler;
use Monolog\Level;
use Monolog\Logger;
use Monolog\Processor\PsrLogMessageProcessor;

$logger = new Logger('shop');

$handler = new StreamHandler(
    __DIR__ . '/var/log/shop.log',
    Level::Debug,
);
$handler->setFormatter(new JsonFormatter());

$logger->pushHandler($handler);
$logger->pushProcessor(new PsrLogMessageProcessor());

$logger->info('订单 {order_id} 创建成功', [
    'order_id' => 10086,
    'user_id' => 42,
]);

try {
    throw new RuntimeException('库存服务暂时不可用');
} catch (Throwable $exception) {
    $logger->error('订单处理失败', [
        'order_id' => 10086,
        'exception' => $exception,
    ]);
}
```

运行：

```bash
php app.php
```

日志会写入 `var/log/shop.log`。如果目录不存在，需要先创建：

```bash
mkdir -p var/log
```

输出为 JSON Lines，一行一条记录，大致如下：

```json
{"message":"订单 10086 创建成功","context":{"order_id":10086,"user_id":42},"level":200,"level_name":"INFO","channel":"shop","datetime":"2026-10-08T...","extra":{}}
```

`PsrLogMessageProcessor` 会把消息中的 `{order_id}` 替换成 `context` 对应的值。原始 `context` 仍会保留在结构化字段中，便于日志平台检索。

## 日志级别：先分清严重程度

PSR-3 定义了八个级别，从低到高依次是：

```text
debug < info < notice < warning < error < critical < alert < emergency
```

常见用法：

| 级别 | 常见场景 |
| --- | --- |
| `debug` | 调试细节，例如分支选择、SQL 参数摘要 |
| `info` | 正常业务节点，例如订单创建、用户登录 |
| `notice` | 值得记录但不算错误的特殊情况 |
| `warning` | 可恢复异常，例如第三方请求重试后成功 |
| `error` | 单次操作失败，需要排查 |
| `critical` | 关键模块不可用或持续失败 |
| `alert` | 需要尽快处理的高危故障 |
| `emergency` | 系统整体不可用 |

级别不仅是写日志时的标签，也常用作 Handler 的过滤门槛。Handler 设置为 `Level::Warning` 后，会接收 `warning` 及更严重的记录，忽略 `debug`、`info` 和 `notice`。

PSR-3 的 logger 方法也可以直接使用：

```php
$logger->debug('查询参数已解析', ['page' => 2]);
$logger->info('订单已创建', ['order_id' => 10086]);
$logger->warning('支付服务响应较慢', ['duration_ms' => 1800]);
$logger->error('支付失败', ['order_id' => 10086]);
```

## Handler：日志往哪里写？

Handler 是 Monolog 的输出端。一个 Logger 可以有多个 Handler，让同一条日志同时写文件、输出到 stderr，或者发送到告警渠道。

常见 Handler：

- `StreamHandler`：写入文件、`php://stdout`、`php://stderr` 等 PHP Stream。
- `RotatingFileHandler`：按日期切分文件，并保留指定天数。
- `ErrorLogHandler`：写入 PHP `error_log()`。
- `SyslogHandler`：写入系统 Syslog。
- `FingersCrossedHandler`：先暂存低级别记录，达到指定级别后再统一交给下层 Handler。
- `BufferHandler`：先缓存日志，满足条件或关闭时再批量处理。
- `NullHandler`：丢弃日志，常用于禁用某个输出分支。

### 按级别分流

例如普通日志进入滚动文件，错误日志单独写文件：

```php
use Monolog\Handler\RotatingFileHandler;
use Monolog\Level;

$logger->pushHandler(new RotatingFileHandler(
    __DIR__ . '/var/log/app.log',
    14,
    Level::Debug,
));

$logger->pushHandler(new RotatingFileHandler(
    __DIR__ . '/var/log/error.log',
    30,
    Level::Error,
));
```

默认 `bubble` 为 `true`，所以 error 及以上级别的记录也会继续写入普通日志文件。若错误日志只希望进入专用文件，可将错误 Handler 的 `bubble` 设为 `false`。

这里的 `maxFiles` 表示最多保留多少个滚动文件。按日期切分适合简单应用；高流量线上环境通常还会结合系统 `logrotate` 或集中式日志平台管理文件轮转。

### Handler 栈和 bubbling

Monolog 的 `pushHandler()` 会把新 Handler 放到栈顶，因此最后添加的 Handler 最先收到记录。Handler 默认允许日志继续传给下一个 Handler，这个行为由 `bubble` 控制：

```php
$logger->pushHandler(new StreamHandler(
    __DIR__ . '/var/log/critical.log',
    Level::Critical,
    bubble: false,
));
```

`bubble: false` 表示当前 Handler 处理记录后停止向下传递。这个参数容易造成“某些日志怎么没写进另一个文件”的问题，配置多个 Handler 时需要一起检查级别和 bubbling。

## FingersCrossedHandler：只有出错时才输出上下文

生产环境常见需求是：成功请求不必把大量调试日志写盘；一旦出现错误，把错误发生前的上下文一并落盘。`FingersCrossedHandler` 正是为这种场景准备的。

```php
use Monolog\Handler\FingersCrossedHandler;
use Monolog\Handler\StreamHandler;
use Monolog\Level;

$nestedHandler = new StreamHandler(
    __DIR__ . '/var/log/prod.log',
    Level::Debug,
);

$logger->pushHandler(new FingersCrossedHandler(
    $nestedHandler,
    Level::Error,
));
```

当没有达到 `error` 时，记录暂存在内存里；出现 `error` 后，缓冲区里的记录连同错误记录一起交给下层 Handler。适合保留问题前因，同时减少正常请求的日志噪声。

默认配置未必限制缓冲记录数量。长时间运行的 Worker 如果持续产生日志又长时间没有触发刷新，需要关注内存占用，并合理设置 buffer 限制或刷新策略。

## Processor：给每条日志补充上下文

Processor 接收一条日志记录，可以在 Handler 输出前补充 `extra` 数据。Monolog 3 中，记录是 `LogRecord` 对象；修改后应返回新的记录对象。

```php
use Monolog\LogRecord;

$logger->pushProcessor(static function (LogRecord $record): LogRecord {
    return $record->with(extra: array_merge($record->extra, [
        'pid' => getmypid(),
        'memory_mb' => round(memory_get_usage(true) / 1024 / 1024, 2),
    ]));
});
```

内置 Processor 还包括：

- `PsrLogMessageProcessor`：替换消息中的 `{key}` 占位符。
- `UidProcessor`：为记录附加进程内唯一标识。
- `WebProcessor`：补充请求 URI、方法和客户端 IP 等 Web 信息。
- `MemoryUsageProcessor`、`MemoryPeakUsageProcessor`：补充内存使用数据。
- `IntrospectionProcessor`：补充产生日志的文件、行号、类和方法。

Processor 可以注册到 Logger，也可以只注册到某个 Handler。请求信息 Processor 适合 Web 应用；命令行和常驻 Worker 中则要注意上下文是否会残留，尤其不能把上一个请求的用户信息带到下一条日志。

### context 和 extra 的区别

应用调用时传入的数据通常放在 `context`：

```php
$logger->info('订单创建完成', [
    'order_id' => 10086,
    'user_id' => 42,
]);
```

Processor 自动补充的数据通常放在 `extra`，比如请求 ID、PID、内存用量。两者最终都会进入 Formatter 输出，但来源和维护者不同。

记录异常时，PSR-3 约定把 Throwable 放在 `exception` 键中：

```php
try {
    $paymentService->charge($order);
} catch (Throwable $exception) {
    $logger->error('扣款失败', [
        'order_id' => $order->getId(),
        'exception' => $exception,
    ]);
}
```

这样 Formatter 可以把异常类型、消息和堆栈信息写完整，而不只是记录一段错误文字。

## Formatter：日志用什么格式保存？

Formatter 负责把 LogRecord 变成 Handler 最终输出的格式。常见选择：

- `LineFormatter`：人类阅读方便，常用于普通文本日志。
- `JsonFormatter`：结构化 JSON，适合 Loki、ELK 等日志平台采集。
- `HtmlFormatter`：生成 HTML 格式，适合邮件等场景。
- `NormalizerFormatter`：把对象、异常等复杂数据规范化为可序列化结构。

应用日志接入日志平台时，通常优先使用 JSON Lines：每行一个 JSON 对象，日志平台能按 `order_id`、`request_id`、`level_name` 等字段检索，不必再解析整段自由文本。

## Symfony 项目中的 Monolog

Symfony 项目安装 MonologBundle：

```bash
composer require symfony/monolog-bundle
```

标准环境下可在服务中注入 PSR-3 接口：

```php
<?php

namespace App\Service;

use Psr\Log\LoggerInterface;

final class OrderService
{
    public function __construct(private LoggerInterface $logger)
    {
    }

    public function createOrder(int $orderId): void
    {
        $this->logger->info('订单创建成功', [
            'order_id' => $orderId,
        ]);
    }
}
```

Symfony 默认配置通常会把 `debug` 及以上日志写到 `var/log/dev.log`，生产环境写到 `var/log/prod.log`。可以在 `config/packages/monolog.yaml` 调整 Handler：

```yaml
monolog:
    handlers:
        main:
            type: fingers_crossed
            action_level: error
            handler: nested
            excluded_http_codes: [404]
            channels: ['!event']

        nested:
            type: stream
            path: '%kernel.logs_dir%/%kernel.environment%.log'
            level: debug
            formatter: monolog.formatter.json
```

这个配置使用 `fingers_crossed`：普通日志先留在缓冲区，发生 `error` 后才把上下文写到文件；404 错误不触发缓冲写出；`event` 通道从该 Handler 排除；输出格式为 JSON。

生产环境对 404 的过滤、缓冲行为和 JSON 格式需要根据日志采集方式调整。文件路径及写权限也必须允许 PHP 运行用户访问。

### 按 Channel 分开记录支付日志

支付日志常常需要单独保留，避免和一般业务日志混在一起。配置 Channel 和专属 Handler：

```yaml
monolog:
    channels: ['payment']

    handlers:
        payment:
            type: rotating_file
            path: '%kernel.logs_dir%/payment.log'
            max_files: 30
            level: info
            channels: ['payment']

        main:
            type: fingers_crossed
            action_level: error
            handler: nested
            channels: ['!event', '!payment']

        nested:
            type: stream
            path: '%kernel.logs_dir%/%kernel.environment%.log'
            level: debug
```

在服务中注入 `monolog.logger.payment`：

```php
use Psr\Log\LoggerInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final class PaymentService
{
    public function __construct(
        #[Autowire(service: 'monolog.logger.payment')]
        private LoggerInterface $paymentLogger,
    ) {
    }

    public function pay(int $orderId): void
    {
        $this->paymentLogger->info('支付请求已提交', [
            'order_id' => $orderId,
        ]);
    }
}
```

Channel 是日志来源标记，Handler 的 `channels` 配置决定哪些来源进入该输出。专用 Channel 适合支付、审计或第三方接口等需要单独保留和检索的场景。

## Monolog 与 Symfony 事件监听器配合

如果需要记录每个 HTTP 请求的耗时，可以在 Symfony 的 `kernel.response` 事件中注入 `LoggerInterface`。这不是 Monolog 自己的事件机制，而是 Symfony EventDispatcher 调用监听器，监听器再通过 PSR-3 Logger 写日志：

```php
<?php

namespace App\EventListener;

use Psr\Log\LoggerInterface;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

#[AsEventListener(event: KernelEvents::RESPONSE)]
final class RequestLogListener
{
    public function __construct(private LoggerInterface $logger)
    {
    }

    public function __invoke(ResponseEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return;
        }

        $request = $event->getRequest();
        $response = $event->getResponse();

        $this->logger->info('HTTP 请求完成', [
            'method' => $request->getMethod(),
            'path' => $request->getPathInfo(),
            'status_code' => $response->getStatusCode(),
        ]);
    }
}
```

生产环境记录客户端 IP 前，应确认反向代理信任配置正确；日志中也应避免写入密码、Cookie、Authorization Token 等敏感信息。

## 日志内容的几个实用习惯

### 写结构化上下文，少拼长字符串

推荐：

```php
$logger->warning('第三方支付请求失败', [
    'provider' => 'example-pay',
    'order_id' => $orderId,
    'http_status' => $statusCode,
]);
```

这样日志平台可以直接按字段筛选。只把所有信息拼进一句话，后续检索和聚合会困难很多。

### 为一次请求建立关联 ID

为每个请求生成或接收一个 `request_id`，并让同一请求中的关键日志都携带它。排查时即可从入口日志一路追到数据库、消息队列和第三方调用。

### 不要记录秘密和完整凭据

密码、Session ID、Cookie、Bearer Token、支付卡数据都不应直接写入日志。必要时记录脱敏后的尾号、内部业务 ID 或失败类别。

### 控制日志量

`debug` 适合开发排查，生产环境不一定要长期全量开启。查询循环、轮询和重试逻辑里尤其要避免每次都写大量重复日志。

## 常见问题排查

### 日志文件没有生成

检查 Handler 的文件路径、运行环境、日志级别、目录是否存在，以及 PHP 进程用户是否有写权限。Symfony 项目可先查看 `var/log/` 并用 `php bin/console debug:config monolog` 检查最终配置。

### 某些级别没有写出来

检查 Handler 的 `level` 门槛，以及多个 Handler 之间的 `bubble` 设置。比如 Handler 配成 `error` 时，`info` 自然不会进入该文件。

### 日志格式字段不符合预期

确认 Handler 关联的 Formatter；`context` 是调用方传入数据，`extra` 通常由 Processor 补充。JSON Formatter 输出的字段结构与普通 LineFormatter 不同。

### Worker 运行久了内存上涨

检查是否使用了 `BufferHandler` 或 `FingersCrossedHandler`，是否有大量记录滞留缓冲区，以及 Processor 是否把大型对象塞进 `context` 或 `extra`。长生命周期进程还要避免保留请求级状态。

## 小结

Monolog 把日志记录拆成几个清晰环节：Logger 接收调用，Processor 补充上下文，Handler 决定输出目标和级别，Formatter 负责输出格式。Handler 可以堆叠分流；Channel 可以区分日志来源；Symfony MonologBundle 则把这些能力带入服务容器和 YAML 配置。

写日志时优先使用 PSR-3 `LoggerInterface`，携带可检索的结构化上下文，异常放到 `exception` 字段，并避免记录秘密。这样日志才能真正用于定位问题，而不只是堆在磁盘上的文本。

## 参考资料

- [Monolog 官方仓库与安装说明](https://github.com/Seldaek/monolog)
- [Monolog 3 使用说明](https://github.com/Seldaek/monolog/blob/main/doc/01-usage.md)
- [Monolog Handlers、Formatters 与 Processors](https://github.com/Seldaek/monolog/blob/main/doc/02-handlers-formatters-processors.md)
- [Monolog 记录结构](https://github.com/Seldaek/monolog/blob/main/doc/message-structure.md)
- [Symfony 7.4 Logging / MonologBundle](https://symfony.com/doc/7.4/logging.html)
