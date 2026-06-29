### 简介

`RabbitMQ` 是一个消息代理，也常被叫作消息队列中间件。

它基于 AMQP 协议，常用于异步处理、系统解耦、流量削峰、任务分发和事件通知。

简单理解：

```text
生产者
  |
  v
RabbitMQ
  |
  v
消费者
```

更完整一点：

```text
Producer
  |
  v
Exchange
  |
  v
Queue
  |
  v
Consumer
```

一句话概括：

```text
RabbitMQ 负责接收消息、路由消息、保存消息，并把消息交给消费者处理。
```

### RabbitMQ 解决什么问题

一个下单接口可能包含很多动作：

```text
创建订单
扣库存
发短信
发邮件
生成物流单
写操作日志
```

如果全部同步执行，接口耗时会变长。

如果短信服务或邮件服务短暂不可用，下单流程也容易被拖住。

引入 RabbitMQ 后，可以拆成：

```text
订单服务
  |
  v
发送订单创建消息
  |
  v
RabbitMQ
  |
  +--> 短信服务消费消息
  +--> 邮件服务消费消息
  +--> 物流服务消费消息
  +--> 日志服务消费消息
```

常见收益：

* 异步处理：主流程更快返回
* 系统解耦：生产者不直接依赖消费者
* 削峰填谷：突发流量先进入队列
* 失败缓冲：消费者短暂异常时，消息可以暂存在队列
* 任务分发：多个消费者共同处理同一类任务

### 核心概念

| 概念 | 说明 |
| --- | --- |
| `Producer` | 生产者，发送消息的应用 |
| `Consumer` | 消费者，处理消息的应用 |
| `Broker` | RabbitMQ 服务端 |
| `Exchange` | 交换机，负责路由消息 |
| `Queue` | 队列，真正保存消息 |
| `Binding` | 交换机和队列之间的绑定关系 |
| `Routing Key` | 路由键，生产者发送消息时携带 |
| `Binding Key` | 绑定键，队列绑定交换机时使用 |
| `Virtual Host` | 虚拟主机，用于逻辑隔离 |
| `Connection` | 客户端到 RabbitMQ 的 TCP 连接 |
| `Channel` | 连接里的轻量通信通道 |

一个消息从发送到消费，大致是：

```text
Producer 发送消息到 Exchange
Exchange 根据 Routing Key 和 Binding 找到 Queue
Queue 保存消息
Consumer 从 Queue 获取消息
Consumer 处理完成后确认消息
RabbitMQ 删除已确认消息
```

### Exchange 类型

RabbitMQ 常见交换机有几种。

| 类型 | 路由规则 | 常见场景 |
| --- | --- | --- |
| `Direct` | routing key 精确匹配 | 订单状态、指定业务事件 |
| `Fanout` | 广播到所有绑定队列 | 通知、缓存刷新、发布订阅 |
| `Topic` | 通配符匹配 routing key | 业务事件总线、日志分类 |
| `Headers` | 根据消息头匹配 | 较少使用 |

### Direct Exchange

`Direct Exchange` 按 routing key 精确匹配。

```text
Exchange: order.exchange

Queue: order.created.queue
Binding Key: order.created

Queue: order.paid.queue
Binding Key: order.paid
```

发送：

```text
routing key = order.created
```

消息进入：

```text
order.created.queue
```

适合订单创建、订单支付、库存扣减这类明确事件。

### Fanout Exchange

`Fanout Exchange` 会把消息发送给所有绑定队列。

它忽略 routing key。

```text
notice.exchange
  |
  +--> sms.queue
  +--> email.queue
  +--> app.push.queue
```

适合广播通知。

比如用户注册成功后：

```text
短信服务收到消息
邮件服务收到消息
积分服务收到消息
```

### Topic Exchange

`Topic Exchange` 支持通配符。

规则：

| 通配符 | 含义 |
| --- | --- |
| `*` | 匹配一个单词 |
| `#` | 匹配零个或多个单词 |

示例：

```text
order.created
order.paid
order.cancelled
product.stock.changed
```

绑定：

```text
order.*        匹配 order.created、order.paid
order.#        匹配 order.created、order.paid、order.refund.created
*.created      匹配 order.created、user.created
#              匹配所有 routing key
```

Topic 很适合做业务事件分发。

### Docker 启动 RabbitMQ

本地学习可以用带管理页面的镜像。

```bash
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=admin123 \
  rabbitmq:management
```

端口说明：

| 端口 | 作用 |
| --- | --- |
| `5672` | AMQP 客户端连接端口 |
| `15672` | 管理后台页面 |

管理后台：

```text
http://localhost:15672
```

账号：

```text
admin / admin123
```

### 原生 Java Client 依赖

非 Spring 项目可以直接使用 RabbitMQ Java Client。

```xml
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>${rabbitmq.client.version}</version>
</dependency>
```

版本建议交给 Maven 属性、BOM 或依赖管理统一控制。

### 原生 Demo：发送消息

这个 Demo 使用默认交换机。

默认交换机比较特殊，routing key 等于队列名时，消息会直接路由到对应队列。

```java
package com.example.rabbitmq.raw;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.nio.charset.StandardCharsets;
import java.util.Map;

public class RawProducer {

    private static final String QUEUE_NAME = "demo.hello.queue";

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("admin123");
        factory.setVirtualHost("/");

        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {

            Map<String, Object> arguments = Map.of("x-queue-type", "quorum");
            channel.queueDeclare(QUEUE_NAME, true, false, false, arguments);

            String message = "Hello RabbitMQ";

            channel.basicPublish(
                    "",
                    QUEUE_NAME,
                    null,
                    message.getBytes(StandardCharsets.UTF_8)
            );

            System.out.println("消息已发送：" + message);
        }
    }
}
```

这里的参数：

```java
channel.queueDeclare(QUEUE_NAME, true, false, false, arguments);
```

含义：

| 参数 | 说明 |
| --- | --- |
| `queue` | 队列名 |
| `durable` | 队列是否持久化 |
| `exclusive` | 是否仅当前连接可用 |
| `autoDelete` | 最后一个消费者断开后是否自动删除 |
| `arguments` | 队列额外参数 |

`x-queue-type=quorum` 表示声明 quorum queue。

### 原生 Demo：消费消息

```java
package com.example.rabbitmq.raw;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

import java.nio.charset.StandardCharsets;
import java.util.Map;

public class RawConsumer {

    private static final String QUEUE_NAME = "demo.hello.queue";

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("admin123");
        factory.setVirtualHost("/");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        Map<String, Object> arguments = Map.of("x-queue-type", "quorum");
        channel.queueDeclare(QUEUE_NAME, true, false, false, arguments);
        channel.basicQos(10);

        DeliverCallback callback = (consumerTag, delivery) -> {
            long deliveryTag = delivery.getEnvelope().getDeliveryTag();
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);

            try {
                System.out.println("收到消息：" + message);
                channel.basicAck(deliveryTag, false);
            } catch (Exception e) {
                channel.basicNack(deliveryTag, false, true);
            }
        };

        boolean autoAck = false;
        channel.basicConsume(QUEUE_NAME, autoAck, callback, consumerTag -> {
        });

        System.out.println("消费者已启动");
    }
}
```

这里使用手动确认：

```java
boolean autoAck = false;
```

处理成功：

```java
channel.basicAck(deliveryTag, false);
```

处理失败并重新入队：

```java
channel.basicNack(deliveryTag, false, true);
```

### Spring Boot 依赖

Spring Boot 项目通常使用 Spring AMQP。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

它会提供：

* `RabbitTemplate`
* `@RabbitListener`
* 连接工厂自动配置
* 消息转换
* 监听容器
* 队列、交换机、绑定声明能力

### application.yml

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: admin
    password: admin123
    virtual-host: /
    publisher-confirm-type: correlated
    publisher-returns: true
    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 10
```

说明：

| 配置 | 作用 |
| --- | --- |
| `publisher-confirm-type: correlated` | 开启发布确认 |
| `publisher-returns: true` | 路由失败时返回消息 |
| `acknowledge-mode: manual` | 消费端手动确认 |
| `prefetch: 10` | 单个消费者最多预取 10 条未确认消息 |

### 消息对象

示例使用订单创建事件。

```java
package com.example.rabbitmq.order;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public record OrderCreatedEvent(
        Long orderId,
        String orderNo,
        Long userId,
        BigDecimal amount,
        LocalDateTime createdAt
) {
}
```

### JSON 消息转换器

默认序列化方式不适合跨语言系统。

业务项目更常见的是 JSON。

```java
package com.example.rabbitmq.config;

import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMessageConfig {

    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

### Direct 实战：订单创建消息

配置交换机、队列和绑定。

```java
package com.example.rabbitmq.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.ExchangeBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.QueueBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OrderRabbitConfig {

    public static final String ORDER_EXCHANGE = "order.exchange";
    public static final String ORDER_CREATED_QUEUE = "order.created.queue";
    public static final String ORDER_CREATED_ROUTING_KEY = "order.created";

    @Bean
    public DirectExchange orderExchange() {
        return ExchangeBuilder
                .directExchange(ORDER_EXCHANGE)
                .durable(true)
                .build();
    }

    @Bean
    public Queue orderCreatedQueue() {
        return QueueBuilder
                .durable(ORDER_CREATED_QUEUE)
                .quorum()
                .build();
    }

    @Bean
    public Binding orderCreatedBinding(
            Queue orderCreatedQueue,
            DirectExchange orderExchange
    ) {
        return BindingBuilder
                .bind(orderCreatedQueue)
                .to(orderExchange)
                .with(ORDER_CREATED_ROUTING_KEY);
    }
}
```

生产者：

```java
package com.example.rabbitmq.order;

import com.example.rabbitmq.config.OrderRabbitConfig;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Service;

@Service
public class OrderEventProducer {

    private final RabbitTemplate rabbitTemplate;

    public OrderEventProducer(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public void sendOrderCreated(OrderCreatedEvent event) {
        rabbitTemplate.convertAndSend(
                OrderRabbitConfig.ORDER_EXCHANGE,
                OrderRabbitConfig.ORDER_CREATED_ROUTING_KEY,
                event
        );
    }
}
```

Controller：

```java
package com.example.rabbitmq.order;

import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderEventProducer producer;

    public OrderController(OrderEventProducer producer) {
        this.producer = producer;
    }

    @PostMapping
    public String createOrder() {
        OrderCreatedEvent event = new OrderCreatedEvent(
                1001L,
                "O202606290001",
                2001L,
                new BigDecimal("99.80"),
                LocalDateTime.now()
        );

        producer.sendOrderCreated(event);
        return "订单创建消息已发送";
    }
}
```

消费者：

```java
package com.example.rabbitmq.order;

import com.example.rabbitmq.config.OrderRabbitConfig;
import com.rabbitmq.client.Channel;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
public class OrderCreatedConsumer {

    @RabbitListener(queues = OrderRabbitConfig.ORDER_CREATED_QUEUE)
    public void handle(
            OrderCreatedEvent event,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag
    ) throws IOException {
        try {
            System.out.println("收到订单创建消息：" + event);

            // 执行业务逻辑：发短信、写日志、生成任务等

            channel.basicAck(deliveryTag, false);
        } catch (Exception e) {
            channel.basicNack(deliveryTag, false, false);
        }
    }
}
```

这里失败时使用：

```java
channel.basicNack(deliveryTag, false, false);
```

第三个参数 `false` 表示不重新入队。

如果配置了死信交换机，消息会进入死信队列。

### Fanout 实战：广播通知

一个用户注册事件，同时通知多个服务。

```text
user.registered.exchange
  |
  +--> sms.notice.queue
  +--> email.notice.queue
```

配置：

```java
package com.example.rabbitmq.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.ExchangeBuilder;
import org.springframework.amqp.core.FanoutExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.QueueBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class NoticeRabbitConfig {

    public static final String USER_REGISTERED_EXCHANGE = "user.registered.exchange";
    public static final String SMS_NOTICE_QUEUE = "sms.notice.queue";
    public static final String EMAIL_NOTICE_QUEUE = "email.notice.queue";

    @Bean
    public FanoutExchange userRegisteredExchange() {
        return ExchangeBuilder
                .fanoutExchange(USER_REGISTERED_EXCHANGE)
                .durable(true)
                .build();
    }

    @Bean
    public Queue smsNoticeQueue() {
        return QueueBuilder.durable(SMS_NOTICE_QUEUE).quorum().build();
    }

    @Bean
    public Queue emailNoticeQueue() {
        return QueueBuilder.durable(EMAIL_NOTICE_QUEUE).quorum().build();
    }

    @Bean
    public Binding smsNoticeBinding(Queue smsNoticeQueue, FanoutExchange userRegisteredExchange) {
        return BindingBuilder.bind(smsNoticeQueue).to(userRegisteredExchange);
    }

    @Bean
    public Binding emailNoticeBinding(Queue emailNoticeQueue, FanoutExchange userRegisteredExchange) {
        return BindingBuilder.bind(emailNoticeQueue).to(userRegisteredExchange);
    }
}
```

发送：

```java
rabbitTemplate.convertAndSend(
        NoticeRabbitConfig.USER_REGISTERED_EXCHANGE,
        "",
        new UserRegisteredEvent(3001L, "zhangsan@example.com", "13800000000")
);
```

事件对象：

```java
public record UserRegisteredEvent(
        Long userId,
        String email,
        String phone
) {
}
```

短信消费者：

```java
@RabbitListener(queues = NoticeRabbitConfig.SMS_NOTICE_QUEUE)
public void sendSms(UserRegisteredEvent event) {
    System.out.println("发送短信：" + event.phone());
}
```

邮件消费者：

```java
@RabbitListener(queues = NoticeRabbitConfig.EMAIL_NOTICE_QUEUE)
public void sendEmail(UserRegisteredEvent event) {
    System.out.println("发送邮件：" + event.email());
}
```

Fanout 会让两个队列都收到同一条消息。

### Topic 实战：业务事件分发

Topic 适合做事件总线。

```text
business.event.exchange
  |
  +--> order.event.queue    binding: order.#
  +--> all.event.queue      binding: #
```

配置：

```java
package com.example.rabbitmq.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.ExchangeBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.QueueBuilder;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class BusinessEventRabbitConfig {

    public static final String BUSINESS_EVENT_EXCHANGE = "business.event.exchange";
    public static final String ORDER_EVENT_QUEUE = "order.event.queue";
    public static final String ALL_EVENT_QUEUE = "all.event.queue";

    @Bean
    public TopicExchange businessEventExchange() {
        return ExchangeBuilder
                .topicExchange(BUSINESS_EVENT_EXCHANGE)
                .durable(true)
                .build();
    }

    @Bean
    public Queue orderEventQueue() {
        return QueueBuilder.durable(ORDER_EVENT_QUEUE).quorum().build();
    }

    @Bean
    public Queue allEventQueue() {
        return QueueBuilder.durable(ALL_EVENT_QUEUE).quorum().build();
    }

    @Bean
    public Binding orderEventBinding(Queue orderEventQueue, TopicExchange businessEventExchange) {
        return BindingBuilder
                .bind(orderEventQueue)
                .to(businessEventExchange)
                .with("order.#");
    }

    @Bean
    public Binding allEventBinding(Queue allEventQueue, TopicExchange businessEventExchange) {
        return BindingBuilder
                .bind(allEventQueue)
                .to(businessEventExchange)
                .with("#");
    }
}
```

发送：

```java
rabbitTemplate.convertAndSend(
        BusinessEventRabbitConfig.BUSINESS_EVENT_EXCHANGE,
        "order.paid",
        new OrderPaidEvent(1001L, "O202606290001")
);
```

`order.event.queue` 和 `all.event.queue` 都会收到这条消息。

### 发布确认

消费端确认和发布端确认不是一回事。

```text
发布确认：RabbitMQ 告诉生产者，消息是否到达 Broker
消费确认：消费者告诉 RabbitMQ，消息是否处理完成
```

开启配置：

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated
    publisher-returns: true
```

配置 `RabbitTemplate` 回调：

```java
package com.example.rabbitmq.config;

import jakarta.annotation.PostConstruct;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.rabbit.support.ReturnedMessage;
import org.springframework.stereotype.Component;

@Component
public class RabbitTemplateCallbackConfig {

    private final RabbitTemplate rabbitTemplate;

    public RabbitTemplateCallbackConfig(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    @PostConstruct
    public void init() {
        rabbitTemplate.setConfirmCallback(this::confirm);
        rabbitTemplate.setReturnsCallback(this::returned);
    }

    private void confirm(
            CorrelationData correlationData,
            boolean ack,
            String cause
    ) {
        if (ack) {
            System.out.println("消息到达 Broker：" + correlationData);
        } else {
            System.out.println("消息未到达 Broker：" + cause);
        }
    }

    private void returned(ReturnedMessage returnedMessage) {
        System.out.println("消息路由失败：" + returnedMessage);
    }
}
```

发送时带 `CorrelationData`：

```java
import org.springframework.amqp.rabbit.connection.CorrelationData;

CorrelationData correlationData = new CorrelationData("msg-1001");

rabbitTemplate.convertAndSend(
        OrderRabbitConfig.ORDER_EXCHANGE,
        OrderRabbitConfig.ORDER_CREATED_ROUTING_KEY,
        event,
        correlationData
);
```

发布确认只能说明消息到达了 Broker。

它不能说明消费者已经处理完成。

### 消费确认

手动 ACK 常见方法：

| 方法 | 含义 |
| --- | --- |
| `basicAck(tag, false)` | 确认当前消息 |
| `basicNack(tag, false, true)` | 拒绝当前消息，并重新入队 |
| `basicNack(tag, false, false)` | 拒绝当前消息，不重新入队 |
| `basicReject(tag, true)` | 拒绝当前消息，并重新入队 |
| `basicReject(tag, false)` | 拒绝当前消息，不重新入队 |

常见写法：

```java
try {
    handleBusiness(message);
    channel.basicAck(deliveryTag, false);
} catch (RetryableException e) {
    channel.basicNack(deliveryTag, false, true);
} catch (Exception e) {
    channel.basicNack(deliveryTag, false, false);
}
```

注意：

```text
basicNack(..., true) 会重新入队
如果业务一直失败，可能形成重复投递循环
```

所以更常见的是结合重试、死信队列和幂等处理。

### basicNack 和 basicReject 的区别

`basicNack` 和 `basicReject` 都可以拒绝消息。

核心区别是：

```text
basicReject 只能拒绝一条消息
basicNack 可以拒绝一条消息，也可以批量拒绝多条消息
```

`basicReject` 的参数：

```java
channel.basicReject(deliveryTag, true);
channel.basicReject(deliveryTag, false);
```

含义：

| 参数 | 说明 |
| --- | --- |
| `deliveryTag` | 消息投递标识 |
| `requeue` | 是否重新入队 |

`basicNack` 的参数：

```java
channel.basicNack(deliveryTag, false, true);
channel.basicNack(deliveryTag, false, false);
```

含义：

| 参数 | 说明 |
| --- | --- |
| `deliveryTag` | 消息投递标识 |
| `multiple` | 是否批量处理 |
| `requeue` | 是否重新入队 |

第二个参数 `multiple` 是关键。

```java
channel.basicNack(deliveryTag, false, false);
```

表示：

```text
只拒绝当前这一条消息，并且不重新入队。
```

```java
channel.basicNack(deliveryTag, true, false);
```

表示：

```text
拒绝当前 deliveryTag 及之前所有未确认消息，并且不重新入队。
```

对比：

| 方法 | 能否批量 | 是否能重新入队 | 常见用途 |
| --- | --- | --- | --- |
| `basicReject` | 否 | 是 | 拒绝单条消息 |
| `basicNack` | 是 | 是 | 拒绝单条或批量消息 |

业务代码里更常见的是：

```java
channel.basicAck(deliveryTag, false);
channel.basicNack(deliveryTag, false, false);
```

含义：

```text
处理成功：确认当前消息
处理失败：拒绝当前消息，不重新入队，让消息进入死信链路
```

### prefetch

`prefetch` 控制消费者一次最多拿多少条未确认消息。

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 10
```

如果消费很慢，prefetch 太大可能导致某个消费者堆积很多未确认消息。

如果消费很快，prefetch 太小可能影响吞吐。

常见思路：

```text
单条消息处理耗时长：prefetch 设置小一点
单条消息处理很快：prefetch 可以适当增大
```

### 死信队列

消息可能变成死信。

常见原因：

* 消息被拒绝，并且不重新入队
* 消息过期
* 队列长度超过限制
* quorum queue 投递次数超过限制

配置死信交换机：

```java
package com.example.rabbitmq.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.ExchangeBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.QueueBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DeadLetterRabbitConfig {

    public static final String ORDER_EXCHANGE = "order.dlx.demo.exchange";
    public static final String ORDER_QUEUE = "order.dlx.demo.queue";
    public static final String ORDER_ROUTING_KEY = "order.created";

    public static final String ORDER_DLX = "order.dead.exchange";
    public static final String ORDER_DLQ = "order.dead.queue";
    public static final String ORDER_DEAD_ROUTING_KEY = "order.dead";

    @Bean
    public DirectExchange orderDlxDemoExchange() {
        return ExchangeBuilder.directExchange(ORDER_EXCHANGE).durable(true).build();
    }

    @Bean
    public DirectExchange orderDeadExchange() {
        return ExchangeBuilder.directExchange(ORDER_DLX).durable(true).build();
    }

    @Bean
    public Queue orderDlxDemoQueue() {
        return QueueBuilder
                .durable(ORDER_QUEUE)
                .quorum()
                .deadLetterExchange(ORDER_DLX)
                .deadLetterRoutingKey(ORDER_DEAD_ROUTING_KEY)
                .ttl(60_000)
                .build();
    }

    @Bean
    public Queue orderDeadQueue() {
        return QueueBuilder
                .durable(ORDER_DLQ)
                .quorum()
                .build();
    }

    @Bean
    public Binding orderDlxDemoBinding() {
        return BindingBuilder
                .bind(orderDlxDemoQueue())
                .to(orderDlxDemoExchange())
                .with(ORDER_ROUTING_KEY);
    }

    @Bean
    public Binding orderDeadBinding() {
        return BindingBuilder
                .bind(orderDeadQueue())
                .to(orderDeadExchange())
                .with(ORDER_DEAD_ROUTING_KEY);
    }
}
```

这 6 个常量可以分成两组。

正常业务链路：

```java
public static final String ORDER_EXCHANGE = "order.dlx.demo.exchange";
public static final String ORDER_QUEUE = "order.dlx.demo.queue";
public static final String ORDER_ROUTING_KEY = "order.created";
```

含义：

```text
正常交换机：order.dlx.demo.exchange
正常队列：order.dlx.demo.queue
正常路由键：order.created
```

正常消息流转：

```text
生产者
  |
  | exchange = order.dlx.demo.exchange
  | routing key = order.created
  v
order.dlx.demo.exchange
  |
  v
order.dlx.demo.queue
  |
  v
正常消费者
```

死信链路：

```java
public static final String ORDER_DLX = "order.dead.exchange";
public static final String ORDER_DLQ = "order.dead.queue";
public static final String ORDER_DEAD_ROUTING_KEY = "order.dead";
```

含义：

```text
死信交换机：order.dead.exchange
死信队列：order.dead.queue
死信路由键：order.dead
```

死信消息流转：

```text
正常队列里的消息变成死信
  |
  v
order.dead.exchange
  |
  | routing key = order.dead
  v
order.dead.queue
```

关键配置在正常队列上：

```java
.deadLetterExchange(ORDER_DLX)
.deadLetterRoutingKey(ORDER_DEAD_ROUTING_KEY)
```

它的意思是：

```text
order.dlx.demo.queue 里的消息如果变成死信，
就转发到 order.dead.exchange，
并使用 routing key = order.dead。
```

再通过死信绑定：

```text
order.dead.exchange + order.dead -> order.dead.queue
```

死信最终进入 `order.dead.queue`。

### 死信配置逐段说明

正常交换机：

```java
@Bean
public DirectExchange orderDlxDemoExchange() {
    return ExchangeBuilder.directExchange(ORDER_EXCHANGE).durable(true).build();
}
```

创建 Direct 类型的正常业务交换机。

```text
名字：order.dlx.demo.exchange
持久化：true
```

死信交换机：

```java
@Bean
public DirectExchange orderDeadExchange() {
    return ExchangeBuilder.directExchange(ORDER_DLX).durable(true).build();
}
```

创建 Direct 类型的死信交换机。

```text
名字：order.dead.exchange
持久化：true
```

正常队列：

```java
@Bean
public Queue orderDlxDemoQueue() {
    return QueueBuilder
            .durable(ORDER_QUEUE)
            .quorum()
            .deadLetterExchange(ORDER_DLX)
            .deadLetterRoutingKey(ORDER_DEAD_ROUTING_KEY)
            .ttl(60_000)
            .build();
}
```

逐项说明：

| 配置 | 说明 |
| --- | --- |
| `durable(ORDER_QUEUE)` | 创建持久化正常队列 |
| `quorum()` | 使用 quorum queue |
| `deadLetterExchange(ORDER_DLX)` | 设置死信交换机 |
| `deadLetterRoutingKey(ORDER_DEAD_ROUTING_KEY)` | 设置死信 routing key |
| `ttl(60_000)` | 消息 60 秒后过期 |

这段配置的整体含义：

```text
order.dlx.demo.queue 是正常队列。
如果消息被拒绝且不重新入队，或者 60 秒过期，
就转发到 order.dead.exchange，
routing key 使用 order.dead。
```

死信队列：

```java
@Bean
public Queue orderDeadQueue() {
    return QueueBuilder
            .durable(ORDER_DLQ)
            .quorum()
            .build();
}
```

创建真正保存死信消息的队列。

```text
名字：order.dead.queue
类型：quorum queue
```

正常绑定：

```java
@Bean
public Binding orderDlxDemoBinding() {
    return BindingBuilder
            .bind(orderDlxDemoQueue())
            .to(orderDlxDemoExchange())
            .with(ORDER_ROUTING_KEY);
}
```

绑定正常消息链路：

```text
order.dlx.demo.exchange
  |
  | routing key = order.created
  v
order.dlx.demo.queue
```

死信绑定：

```java
@Bean
public Binding orderDeadBinding() {
    return BindingBuilder
            .bind(orderDeadQueue())
            .to(orderDeadExchange())
            .with(ORDER_DEAD_ROUTING_KEY);
}
```

绑定死信消息链路：

```text
order.dead.exchange
  |
  | routing key = order.dead
  v
order.dead.queue
```

完整正常流程：

```text
生产者
  |
  v
正常交换机
  |
  v
正常队列
  |
  v
消费者处理成功
  |
  v
basicAck
  |
  v
消息删除
```

完整死信流程：

```text
生产者
  |
  v
正常交换机
  |
  v
正常队列
  |
  v
消费者处理失败
  |
  v
basicNack(deliveryTag, false, false)
  |
  v
消息不重新入队
  |
  v
转发到死信交换机
  |
  v
进入死信队列
```

消费死信：

```java
@RabbitListener(queues = DeadLetterRabbitConfig.ORDER_DLQ)
public void handleDeadLetter(OrderCreatedEvent event) {
    System.out.println("收到死信消息：" + event);
}
```

死信队列常用于：

* 保存异常消息
* 后台人工处理
* 延迟处理
* 重试次数耗尽后的兜底

### 延迟消息

RabbitMQ 常见延迟方案有两类：

```text
TTL + 死信交换机
Delayed Message Exchange 插件
```

`TTL + DLX` 的思路：

```text
消息先进入延迟队列
延迟队列不设置消费者
消息过期后变成死信
死信进入真正业务队列
消费者处理业务队列
```

适合订单超时取消这类场景。

如果需要更灵活的 per-message 延迟，常见做法是启用 RabbitMQ delayed message exchange 插件。

具体方案需要结合 RabbitMQ 版本、插件启用情况、运维规范来定。

### 幂等处理

RabbitMQ 不能保证业务只处理一次。

实际系统里可能出现重复投递：

* 消费者处理完成但 ACK 失败
* 网络抖动导致重新投递
* 生产者重试发送
* 服务重启后重新消费未确认消息

消费者需要做幂等。

常见做法：

```text
消息带唯一 messageId
消费前查处理记录
没有处理过才执行业务
处理成功后保存记录
重复消息直接 ACK
```

消息对象：

```java
public record ReliableMessage<T>(
        String messageId,
        T payload
) {
}
```

消费者伪代码：

```java
@RabbitListener(queues = OrderRabbitConfig.ORDER_CREATED_QUEUE)
public void handle(
        ReliableMessage<OrderCreatedEvent> message,
        Channel channel,
        @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag
) throws IOException {
    try {
        if (messageLogRepository.exists(message.messageId())) {
            channel.basicAck(deliveryTag, false);
            return;
        }

        handleBusiness(message.payload());
        messageLogRepository.save(message.messageId());

        channel.basicAck(deliveryTag, false);
    } catch (Exception e) {
        channel.basicNack(deliveryTag, false, false);
    }
}
```

幂等位置可以放在：

* 数据库唯一键
* Redis `SETNX`
* 消息消费日志表
* 业务状态机

### 可靠消息常见链路

更完整的可靠链路通常包括：

```text
生产者本地业务事务
  |
  v
保存消息记录
  |
  v
发送 RabbitMQ
  |
  v
发布确认更新消息状态
  |
  v
消费者手动 ACK
  |
  v
消费者幂等处理
  |
  v
异常消息进入死信队列
```

如果业务对一致性要求很高，可以使用本地消息表或 outbox 模式。

简单示意：

```text
同一个数据库事务：
  1. 创建订单
  2. 写入待发送消息表

后台任务：
  1. 扫描待发送消息
  2. 发送 RabbitMQ
  3. 根据发布确认更新状态
```

### 管理后台常看指标

RabbitMQ 管理后台常看这些内容：

| 指标 | 含义 |
| --- | --- |
| `Ready` | 队列里等待消费的消息数 |
| `Unacked` | 已投递但未确认的消息数 |
| `Total` | Ready + Unacked |
| `Publish rate` | 发布速率 |
| `Deliver rate` | 投递速率 |
| `Ack rate` | 确认速率 |
| `Consumers` | 消费者数量 |
| `Memory` | 内存占用 |
| `Disk` | 磁盘占用 |

排查思路：

```text
Ready 持续增加：消费者处理能力不足或消费者异常
Unacked 很高：消费者处理慢、ACK 失败或 prefetch 过大
Publish 高于 Ack：生产速度超过消费速度
Consumers 为 0：消费者应用未启动或监听失败
```

### RabbitMQ 数据存储在哪里

RabbitMQ 的数据存储在 Broker 节点上。

主要包括：

* vhost、用户、权限等元数据
* exchange、queue、binding 等拓扑信息
* 队列里的消息数据

Docker 容器里常见数据目录是：

```text
/var/lib/rabbitmq
```

如果没有挂载 volume，容器删除后数据也会删除。

本地开发可以挂载数据卷：

```bash
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=admin123 \
  -v rabbitmq-data:/var/lib/rabbitmq \
  rabbitmq:management
```

这样 RabbitMQ 数据会保存到 Docker volume：

```text
rabbitmq-data
```

消息是否能在 RabbitMQ 重启后保留，还要看几个条件：

```text
exchange 是否 durable
queue 是否 durable
message 是否 persistent
```

常见配置：

```java
ExchangeBuilder.directExchange("order.exchange").durable(true).build();
QueueBuilder.durable("order.queue").build();
```

Spring AMQP 常见发送方式通常会配合持久化队列发送持久化消息。

生产项目仍然适合通过重启 RabbitMQ、查看管理后台和消费验证来确认消息持久化行为。

简单总结：

```text
RabbitMQ 数据由 Broker 管理。
持久化元数据和持久化消息会落到节点数据目录。
非持久化消息主要在内存中，服务重启后可能丢失。
```

### 常见使用建议

### 队列和交换机持久化

关键业务队列和交换机通常需要持久化。

```java
ExchangeBuilder.directExchange("order.exchange").durable(true).build();
QueueBuilder.durable("order.queue").build();
```

消息本身也需要持久化投递。

Spring AMQP 默认消息持久化策略通常已经适合多数持久化队列场景，但生产项目仍应结合消息转换器和发送属性做验证。

### 区分发布确认和消费确认

```text
发布确认：消息到没到 RabbitMQ
消费确认：消费者处理完没有
```

两者解决的问题不同，不能互相替代。

### 避免无限重新入队

消费失败后一直 `basicNack(..., true)`，可能导致同一条消息反复投递。

更常见做法：

```text
可重试异常：有限次数重试
不可恢复异常：进入死信队列
```

### 控制 prefetch 和并发

消费者吞吐不够时，可以增加消费者实例或并发数。

但数据库、Redis、第三方接口也有容量上限。

队列堆积时，需要同时看：

* 消费者数量
* 单条消息处理耗时
* prefetch
* 下游数据库连接池
* 下游接口限流

### 消费者需要幂等

消息系统里“至少一次”投递更常见。

消费者应该能承受重复消息。

比如订单支付成功消息，可以根据订单状态判断：

```text
未处理 -> 执行业务
已处理 -> 直接 ACK
```

### 常用 API 汇总

| API / 注解 | 作用 |
| --- | --- |
| `RabbitTemplate.convertAndSend(...)` | 发送消息 |
| `@RabbitListener` | 监听队列 |
| `QueueBuilder.durable(...)` | 创建持久化队列 |
| `ExchangeBuilder.directExchange(...)` | 创建 Direct 交换机 |
| `ExchangeBuilder.fanoutExchange(...)` | 创建 Fanout 交换机 |
| `ExchangeBuilder.topicExchange(...)` | 创建 Topic 交换机 |
| `BindingBuilder.bind(...).to(...)` | 绑定队列和交换机 |
| `Jackson2JsonMessageConverter` | JSON 消息转换 |
| `channel.basicAck(...)` | 确认消息 |
| `channel.basicNack(...)` | 拒绝消息 |
| `channel.basicQos(...)` | 设置预取数量 |
| `CorrelationData` | 发布确认关联数据 |

### 总结

`RabbitMQ` 的核心是消息路由和异步消费。

最重要的几个概念：

```text
Producer
Exchange
Routing Key
Binding
Queue
Consumer
ACK
DLX
```

普通业务可以从 Direct、Fanout、Topic 三种交换机开始。

真正进入工程实践后，需要重点关注：

* 发布确认
* 消费确认
* 消息持久化
* 死信队列
* 消费幂等
* prefetch
* 监控和告警

RabbitMQ 不是简单地“把消息发出去就结束”。

可靠消息处理更像一条链路：生产者确认、Broker 路由、队列保存、消费者确认、异常兜底、重复消费处理，每个环节都需要有明确设计。
