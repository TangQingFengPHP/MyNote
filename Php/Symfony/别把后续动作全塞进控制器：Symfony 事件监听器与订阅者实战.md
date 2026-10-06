# 别把后续动作全塞进控制器：Symfony 事件监听器与订阅者实战

用户注册成功后，系统可能还要发欢迎邮件、记录审计日志、赠送积分、通知其他模块。如果这些动作都堆在注册控制器里，控制器会越来越长，注册逻辑也会和邮件、积分、通知绑在一起。

Symfony EventDispatcher 可以把“某件事已经发生”广播出去，再由独立的监听器处理后续动作。注册服务只负责注册用户并派发事件，不需要知道后面有哪些监听器。

本文以 Symfony 7.4 为例，用“注册完成后记录日志并发送欢迎邮件”搭建一个完整 Demo，同时说明事件、监听器、订阅者、优先级、内核事件，以及 Symfony 事件和 Doctrine 生命周期事件的区别。

## 事件机制解决什么问题？

没有事件时，注册代码可能逐渐变成这样：

```php
$user = $userRepository->create($email, $password);
$mailer->sendWelcomeEmail($user);
$points->giveSignupPoints($user);
$auditLogger->recordSignup($user);
$adminNotifier->notifyNewUser($user);
```

注册服务因此知道了太多外围工作。增加短信通知或新积分规则时，还得回头修改注册流程。

改为事件后，注册服务只表达一件事：用户已经注册。

```text
RegisterUserService
        │
        │ dispatch(UserRegisteredEvent)
        ▼
   EventDispatcher
      ├── SendWelcomeEmailListener
      ├── GrantSignupPointsListener
      └── RecordSignupListener
```

发布事件的代码不需要依赖监听器。监听器之间也可以各自维护职责，这就是事件机制带来的解耦。

事件派发默认是**同步**的：`dispatch()` 会依次调用所有监听器，等它们执行完才继续往下走。如果监听器发邮件很慢，当前 HTTP 请求仍然会变慢。需要异步处理时，可以让监听器投递 Symfony Messenger 消息，由 Worker 后台执行。

## Event、Listener、Subscriber 分别是什么？

- **Event（事件）**：一个对象，描述发生了什么，并携带处理所需的数据。
- **Dispatcher（派发器）**：根据事件名称找到对应监听器并调用。
- **Listener（监听器）**：处理一个或少量事件，常用 `#[AsEventListener]` 注册。
- **Subscriber（订阅者）**：一个类集中声明自己关心的多个事件，实现 `EventSubscriberInterface`。

Symfony 事件名可以使用事件类的完整类名。调用 `dispatch($event)` 时，如果没有显式传入名称，派发器默认使用 `$event::class`。

自定义事件可以是普通 PHP 对象。若需要调用 `stopPropagation()` 阻止后续监听器执行，可以继承 `Symfony\Contracts\EventDispatcher\Event`。

## 准备项目

完整 Symfony 项目通常已经安装 EventDispatcher。若是精简项目，可安装：

```bash
composer require symfony/event-dispatcher
```

下面的 Demo 假设项目已安装 Doctrine ORM、Symfony Mailer 和 MakerBundle：

```bash
composer require symfony/orm-pack symfony/mailer
composer require --dev symfony/maker-bundle
```

Symfony Flex 的标准 `config/services.yaml` 通常开启自动装配和自动配置：

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true

    App\:
        resource: '../src/'
        exclude:
            - '../src/DependencyInjection/'
            - '../src/Entity/'
            - '../src/Kernel.php'
```

这项配置让 Symfony 自动发现 `src/` 下的服务，并识别事件监听器属性和订阅者接口。若项目使用了不同的服务配置，需要确认监听器类已注册为服务。

## 第一步：定义用户注册事件

创建 `src/Event/UserRegisteredEvent.php`：

```php
<?php

namespace App\Event;

use App\Entity\User;

final class UserRegisteredEvent
{
    public function __construct(
        private readonly User $user,
        private readonly \DateTimeImmutable $registeredAt,
    ) {
    }

    public function getUser(): User
    {
        return $this->user;
    }

    public function getRegisteredAt(): \DateTimeImmutable
    {
        return $this->registeredAt;
    }
}
```

事件类尽量表述已经发生的事实，比如 `UserRegisteredEvent`，而不是命令式的 `RegisterUserEvent`。通常用不可变属性保存事件发生时需要的数据，避免监听器收到事件后再依赖外部状态拼出事实。

## 第二步：在业务服务里派发事件

创建 `src/Service/RegisterUserService.php`：

```php
<?php

namespace App\Service;

use App\Entity\User;
use App\Event\UserRegisteredEvent;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;
use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;

final class RegisterUserService
{
    public function __construct(
        private EntityManagerInterface $entityManager,
        private UserPasswordHasherInterface $passwordHasher,
        private EventDispatcherInterface $eventDispatcher,
    ) {
    }

    public function register(string $email, string $plainPassword): User
    {
        $user = new User();
        $user->setEmail($email);
        $user->setPassword(
            $this->passwordHasher->hashPassword($user, $plainPassword)
        );

        $this->entityManager->persist($user);
        $this->entityManager->flush();

        $this->eventDispatcher->dispatch(
            new UserRegisteredEvent($user, new \DateTimeImmutable())
        );

        return $user;
    }
}
```

这里先 `flush()` 再派发事件，表示事件代表“用户已经写入数据库”。这样监听器执行失败时，不会让事件听起来像注册成功但数据库还没有用户。

这不代表数据库写入和邮件发送天然处于同一个事务。同步监听器抛异常时，接口可能报错，但已经执行的数据库提交不会自动回滚。需要保证数据库事务和消息发送一致时，可考虑 Transactional Outbox 等模式。

## 第三步：监听事件并发送邮件

创建 `src/EventListener/SendWelcomeEmailListener.php`：

```php
<?php

namespace App\EventListener;

use App\Event\UserRegisteredEvent;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\Email;

#[AsEventListener(event: UserRegisteredEvent::class)]
final class SendWelcomeEmailListener
{
    public function __construct(private MailerInterface $mailer)
    {
    }

    public function __invoke(UserRegisteredEvent $event): void
    {
        $user = $event->getUser();

        $email = (new Email())
            ->from('no-reply@example.com')
            ->to($user->getEmail())
            ->subject('欢迎注册')
            ->text(sprintf(
                '账号 %s 已于 %s 注册成功。',
                $user->getEmail(),
                $event->getRegisteredAt()->format('Y-m-d H:i:s'),
            ));

        $this->mailer->send($email);
    }
}
```

`#[AsEventListener]` 将这个类标记为事件监听器。没有指定 `method` 时，Symfony 会调用 `__invoke()`；显式指定方法名也可以：

```php
#[AsEventListener(event: UserRegisteredEvent::class, method: 'send')]
final class SendWelcomeEmailListener
{
    public function send(UserRegisteredEvent $event): void
    {
        // 处理事件
    }
}
```

属性方式从 Symfony 6.1 起可用。属性注册依赖服务自动配置；若没有启用 `autoconfigure`，可在 `services.yaml` 手工添加 `kernel.event_listener` 标签。

## 第四步：监听同一个事件做另一件事

再加一个日志监听器 `src/EventListener/RecordUserRegisteredListener.php`：

```php
<?php

namespace App\EventListener;

use App\Event\UserRegisteredEvent;
use Psr\Log\LoggerInterface;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener(event: UserRegisteredEvent::class)]
final class RecordUserRegisteredListener
{
    public function __construct(private LoggerInterface $logger)
    {
    }

    public function __invoke(UserRegisteredEvent $event): void
    {
        $this->logger->info('User registered', [
            'email' => $event->getUser()->getEmail(),
            'registered_at' => $event->getRegisteredAt()->format(DATE_ATOM),
        ]);
    }
}
```

注册服务派发一次 `UserRegisteredEvent`，所有注册到该事件的监听器都会收到它。增加日志、通知或积分处理时，通常只需新增监听器，不必继续往注册服务里塞代码。

## 另一种注册方式：Event Subscriber

一个类需要集中监听多个相关事件时，可以实现 `EventSubscriberInterface`：

```php
<?php

namespace App\EventSubscriber;

use App\Event\UserRegisteredEvent;
use Psr\Log\LoggerInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

final class UserAuditSubscriber implements EventSubscriberInterface
{
    public function __construct(private LoggerInterface $logger)
    {
    }

    public static function getSubscribedEvents(): array
    {
        return [
            UserRegisteredEvent::class => 'onUserRegistered',
        ];
    }

    public function onUserRegistered(UserRegisteredEvent $event): void
    {
        $this->logger->notice('注册审计事件', [
            'user_id' => $event->getUser()->getId(),
        ]);
    }
}
```

`getSubscribedEvents()` 返回“事件名 => 方法名”的映射。订阅多个事件时可以集中写在同一个数组里，也可以给处理方法设置优先级：

```php
public static function getSubscribedEvents(): array
{
    return [
        UserRegisteredEvent::class => [
            ['validateRegistration', 20],
            ['recordRegistration', 0],
        ],
    ];
}
```

### Listener 和 Subscriber 怎么选？

- 一个类只负责监听一个事件：`#[AsEventListener]` 清楚直接。
- 一个类围绕同一职责处理多个事件：Subscriber 便于集中查看订阅关系。
- 两者底层都由 EventDispatcher 调用，不存在“Subscriber 一定更快”或“Listener 功能更强”的区别。

项目里优先保持职责清晰，避免一个 Subscriber 最后变成装着所有业务逻辑的大类。

## 优先级：谁先执行？

多个监听器监听同一事件时，可以设置优先级。数字越大，越早执行；默认优先级是 `0`。

```php
#[AsEventListener(event: UserRegisteredEvent::class, priority: 20)]
final class ValidateRegisteredUserListener
{
    public function __invoke(UserRegisteredEvent $event): void
    {
        // 优先级较高，先执行
    }
}
```

Subscriber 也可以设置优先级：

```php
return [
    UserRegisteredEvent::class => ['onUserRegistered', 20],
];
```

高优先级先运行。相同优先级下，执行顺序取决于注册顺序，不建议把业务正确性建立在未明确约定的同优先级顺序上。

## 停止事件传播

普通事件对象没有 `stopPropagation()`。需要在某个监听器决定“不再通知后面的监听器”时，让事件继承 `Event`：

```php
<?php

namespace App\Event;

use Symfony\Contracts\EventDispatcher\Event;

final class PaymentApprovedEvent extends Event
{
    public function __construct(private readonly string $orderNo)
    {
    }

    public function getOrderNo(): string
    {
        return $this->orderNo;
    }
}
```

监听器中调用：

```php
$event->stopPropagation();
```

之后尚未执行的低优先级监听器会被跳过。这个能力会改变其他监听器是否收到事件，使用前应确认确实需要中止整个事件流程。它不是异常处理机制，也不等于撤销已经完成的操作。

## Symfony 内置的 Kernel 事件

Symfony 在处理 HTTP 请求时会派发一系列内核事件，常见用途包括给请求补充信息、统一修改响应头、处理未捕获异常。

| 事件 | 发生时机 | 常见用途 |
| --- | --- | --- |
| `kernel.request` | 请求处理早期 | 设置 Locale、拦截请求、补充 Request 属性 |
| `kernel.controller` | 控制器确定后、执行前 | 检查或包装控制器 |
| `kernel.controller_arguments` | 控制器参数解析后 | 检查解析好的参数 |
| `kernel.view` | 控制器没有返回 Response 时 | 把控制器结果转换成 Response |
| `kernel.response` | 响应即将返回时 | 添加响应头、Cookie、修改响应内容 |
| `kernel.exception` | 请求处理中抛出异常时 | 记录异常或自定义错误响应 |
| `kernel.terminate` | 响应发送后 | 执行不影响响应耗时的收尾工作 |

例如统一给响应加一个追踪头：

```php
<?php

namespace App\EventListener;

use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

#[AsEventListener(event: KernelEvents::RESPONSE, method: 'onKernelResponse')]
final class ResponseHeaderListener
{
    public function onKernelResponse(ResponseEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return;
        }

        $event->getResponse()->headers->set('X-App-Name', 'SymfonyDemo');
    }
}
```

Kernel 事件可能在主请求和子请求中触发。只处理真正的浏览器主请求时，使用 `isMainRequest()` 过滤，避免子请求（例如片段渲染）重复执行逻辑。

## Symfony EventDispatcher 和 Doctrine 事件不是一回事

Symfony 项目里还经常见到 Doctrine 的实体生命周期事件，例如 `prePersist`、`postUpdate`。它们和 Symfony EventDispatcher 的自定义业务事件属于两套机制：

| 类型 | 触发方 | 例子 | 适合场景 |
| --- | --- | --- | --- |
| Symfony EventDispatcher 事件 | 应用代码或 HttpKernel | `UserRegisteredEvent`、`kernel.response` | 业务流程解耦、请求/响应扩展 |
| Doctrine ORM 生命周期事件 | Doctrine ORM | `prePersist`、`postUpdate` | 实体持久化过程中的局部处理 |

仅给单个实体设置默认值，Doctrine Lifecycle Callback 通常更简单；需要注入服务并仅关注一个实体时，可用 Entity Listener；跨实体的 ORM 生命周期工作适合 Doctrine Lifecycle Listener。邮件通知等业务后续动作通常应由业务服务派发明确的领域事件，而不是全部塞进 `postPersist`。

尤其要避免在 Doctrine `flush()` 期间的生命周期回调里随意再次 `flush()` 或启动复杂业务流程，这容易造成变更追踪和事务边界问题。

## 需要异步执行时：把事件处理放进 Messenger

EventDispatcher 本身是同步机制。邮件网关慢、第三方接口超时，都会拖住当前请求。一个常见升级方向是由事件监听器投递 Messenger 消息，再由 Worker 后台发送邮件：

```php
use Symfony\Component\Messenger\MessageBusInterface;

final class QueueWelcomeEmailListener
{
    public function __construct(private MessageBusInterface $bus)
    {
    }

    public function __invoke(UserRegisteredEvent $event): void
    {
        $this->bus->dispatch(new SendWelcomeEmailMessage(
            $event->getUser()->getId()
        ));
    }
}
```

随后为 `SendWelcomeEmailMessage` 配置 Messenger transport，由 Worker 消费。消息通常只携带用户 ID 等必要数据，后台处理时再从数据库加载最新用户资料。

异步后需要考虑失败重试、幂等和消息最终一致性。事件派发和消息入队之间若必须保证不丢，可采用 Outbox 等事务消息模式。

## 查看监听器是否注册成功

常用调试命令：

```bash
# 查看某个事件的监听器及优先级
php bin/console debug:event-dispatcher 'App\Event\UserRegisteredEvent'

# 查看内核事件监听器
php bin/console debug:event-dispatcher kernel.response

# 检查服务容器配置
php bin/console lint:container
```

如果找不到监听器，先检查：类是否在 `src/` 服务资源范围内，`autoconfigure` 是否开启，属性引用的事件名称是否与派发时一致。

## 常见误区

1. **把事件当作异步队列。** `dispatch()` 默认同步运行所有监听器。
2. **在控制器里到处直接派发框架事件。** 业务事件最好由业务服务在明确的业务节点派发。
3. **让监听器承担核心业务决策。** 事件适合解耦后续动作，关键状态变更应由清晰的应用服务和事务负责。
4. **事件数据只传一个数据库 ID，却不说明语义。** 事件应准确描述发生的事实；若需要历史状态或当时数据，应把必要值作为事件字段保存。
5. **忽略事件的同步异常。** 监听器抛出的异常可能沿调用链返回，影响原请求。
6. **混淆 Doctrine Listener 和 Symfony Listener。** 两者触发时机、注册标签和参数对象完全不同。

## 小结

Symfony 事件监听器适合把“发生某件事之后要做的工作”从主流程中拆出来。事件对象说明发生了什么，EventDispatcher 负责找到监听器，Listener 或 Subscriber 执行后续逻辑。

一个事件监听器类只关注一项后续动作；一组相关事件可以放进 Subscriber；高优先级先执行；需要后台处理时接入 Messenger；涉及 Doctrine 持久化阶段的逻辑则要和 ORM 生命周期事件区分开。

## 参考资料

- [Symfony 7.4 Events and Event Listeners](https://symfony.com/doc/7.4/event_dispatcher.html)
- [Symfony 7.4 HttpKernel Component 与内核事件](https://symfony.com/doc/7.4/components/http_kernel.html)
- [Symfony 7.4 Doctrine Events](https://symfony.com/doc/7.4/doctrine/events.html)
- [Symfony 7.4 Messenger](https://symfony.com/doc/7.4/messenger.html)
- [Symfony EventDispatcher Component](https://symfony.com/doc/7.4/components/event_dispatcher.html)
