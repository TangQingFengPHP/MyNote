### 简介

`ActiveMQ` 是 Apache 提供的消息中间件。

它在 Java 生态里很常见，尤其是传统企业系统、老项目改造、JMS 规范相关系统中出现得比较多。

简单理解：

```text
生产者
  |
  v
ActiveMQ Broker
  |
  v
消费者
```

ActiveMQ 最常见的使用方式是 JMS。

JMS 全称是 `Java Message Service`，它是一套 Java 消息服务 API 规范。

ActiveMQ 和 JMS 的关系可以类比成：

```text
JMS 是规范
ActiveMQ 是实现
```

一句话概括：

```text
ActiveMQ 是 Java 生态里经典的 JMS 消息中间件，适合异步处理、系统解耦、任务队列和发布订阅场景。
```

### ActiveMQ 解决什么问题

一个业务动作经常会带出很多后续动作。

比如订单创建成功后：

```text
扣库存
发短信
发邮件
写日志
生成物流任务
同步积分
```

如果全部同步执行，接口会变慢。

如果短信服务异常，订单接口也可能被影响。

使用 ActiveMQ 后，可以拆成：

```text
订单服务
  |
  v
发送订单消息
  |
  v
ActiveMQ
  |
  +--> 库存服务消费消息
  +--> 短信服务消费消息
  +--> 邮件服务消费消息
  +--> 日志服务消费消息
```

常见作用：

* 异步处理
* 系统解耦
* 削峰填谷
* 任务分发
* 发布订阅
* 消息重试和失败兜底

### ActiveMQ Classic 和 Artemis

ActiveMQ 需要先区分两条线：

| 名称 | 说明 |
| --- | --- |
| `ActiveMQ Classic` | 传统 ActiveMQ 5.x / 6.x 线，老项目和 JMS 系统常见 |
| `ActiveMQ Artemis` | 新一代消息 Broker，性能和架构更现代 |

简单理解：

```text
ActiveMQ Classic：经典 JMS Broker
ActiveMQ Artemis：新一代多协议、高性能 Broker
```

如果是维护老系统，经常会遇到 ActiveMQ Classic。

如果是新系统选型，并且明确要用 Apache ActiveMQ 体系，Artemis 更值得优先评估。

本文示例以 ActiveMQ Classic + Spring Boot JMS 为主，因为它更贴近日常 Java 老系统和企业项目集成场景。

### ActiveMQ 和 RabbitMQ 的区别

| 对比项 | ActiveMQ | RabbitMQ |
| --- | --- | --- |
| 常见协议 | JMS、OpenWire、STOMP、AMQP、MQTT | AMQP 为主，也支持多协议 |
| Java 规范关系 | JMS 体系很强 | 不以 JMS 为核心 |
| 核心模型 | Queue / Topic | Exchange / Queue / Binding |
| 路由能力 | JMS 模型更直接 | Exchange 路由更灵活 |
| 常见场景 | 传统企业系统、JMS 项目 | 微服务消息、灵活路由、事件分发 |

ActiveMQ 更像：

```text
Queue / Topic 模型清晰，适合 JMS 项目。
```

RabbitMQ 更像：

```text
Exchange + Routing Key 更灵活，适合复杂路由。
```

### JMS 核心概念

| 概念 | 说明 |
| --- | --- |
| `ConnectionFactory` | 连接工厂 |
| `Connection` | 客户端到 Broker 的连接 |
| `Session` | 会话，负责生产、消费、确认、事务 |
| `Destination` | 目的地，Queue 或 Topic |
| `Queue` | 点对点队列 |
| `Topic` | 发布订阅主题 |
| `MessageProducer` | 消息生产者 |
| `MessageConsumer` | 消息消费者 |
| `Message` | 消息对象 |

整体流程：

```text
ConnectionFactory
  |
  v
Connection
  |
  v
Session
  |
  +--> MessageProducer -> Destination
  |
  +--> MessageConsumer <- Destination
```

### Queue 点对点模式

Queue 是点对点模式。

特点：

```text
一条消息只会被一个消费者处理。
```

示例：

```text
order.queue
  |
  +--> consumer-1
  +--> consumer-2
```

如果 `order.queue` 有 10 条消息，两个消费者会竞争消费。

每条消息最终只会被其中一个消费者处理。

适合：

* 订单处理
* 短信发送
* 邮件发送
* 报表生成
* 后台任务

### Topic 发布订阅模式

Topic 是发布订阅模式。

特点：

```text
一条消息可以被多个订阅者收到。
```

示例：

```text
order.topic
  |
  +--> sms-consumer
  +--> email-consumer
  +--> log-consumer
```

订单支付成功消息发送到 `order.topic` 后，短信、邮件、日志服务都能收到。

适合：

* 业务事件广播
* 通知消息
* 日志订阅
* 缓存刷新

### Topic 持久订阅和非持久订阅

Topic 需要注意订阅方式。

| 类型 | 离线期间消息是否保留 | 说明 |
| --- | --- | --- |
| 非持久订阅 | 不保留 | 消费者在线时才能收到 |
| 持久订阅 | 保留 | 消费者离线后再上线也能收到 |

非持久订阅适合临时通知。

持久订阅适合关键业务事件。

持久订阅通常需要设置唯一的 `clientId` 和订阅名。

### Docker 启动 ActiveMQ Classic

本地学习可以使用 Docker。

不同镜像维护方式不同，实际项目以团队镜像仓库或官方发布包为准。

示例：

```bash
docker run -d \
  --name activemq-classic \
  -p 61616:61616 \
  -p 8161:8161 \
  apache/activemq-classic:latest
```

端口说明：

| 端口 | 作用 |
| --- | --- |
| `61616` | JMS / OpenWire 连接端口 |
| `8161` | Web 管理控制台 |

管理后台：

```text
http://localhost:8161/admin
```

常见默认账号：

```text
admin / admin
```

如果镜像没有提供默认账号，需要查看镜像说明或自定义配置。

### 原生 JMS 依赖

Spring Boot 3 使用 `jakarta.jms` 包名。

如果使用 ActiveMQ Classic 6 或 Jakarta 体系，可以选择 Jakarta 版本客户端。

```xml
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-client-jakarta</artifactId>
    <version>${activemq.version}</version>
</dependency>
```

如果是老项目，可能仍然使用 `javax.jms` 和 ActiveMQ 5.x 客户端。

```xml
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-client</artifactId>
    <version>${activemq.version}</version>
</dependency>
```

包名差异：

```text
老项目：javax.jms.*
新项目：jakarta.jms.*
```

### 原生 JMS：发送 Queue 消息

```java
package com.example.activemq.raw;

import jakarta.jms.Connection;
import jakarta.jms.ConnectionFactory;
import jakarta.jms.DeliveryMode;
import jakarta.jms.Destination;
import jakarta.jms.MessageProducer;
import jakarta.jms.Session;
import jakarta.jms.TextMessage;
import org.apache.activemq.ActiveMQConnectionFactory;

public class RawQueueProducer {

    private static final String BROKER_URL = "tcp://localhost:61616";
    private static final String QUEUE_NAME = "order.queue";

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ActiveMQConnectionFactory(BROKER_URL);

        try (Connection connection = factory.createConnection();
             Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE)) {

            Destination destination = session.createQueue(QUEUE_NAME);
            MessageProducer producer = session.createProducer(destination);
            producer.setDeliveryMode(DeliveryMode.PERSISTENT);

            TextMessage message = session.createTextMessage("订单创建成功");
            message.setStringProperty("bizType", "order");
            message.setStringProperty("messageId", "msg-1001");

            producer.send(message);

            System.out.println("消息已发送：" + message.getText());
        }
    }
}
```

说明：

```text
Session.AUTO_ACKNOWLEDGE：自动确认
DeliveryMode.PERSISTENT：持久化消息
createQueue：创建 Queue 目的地
```

### 原生 JMS：消费 Queue 消息

```java
package com.example.activemq.raw;

import jakarta.jms.Connection;
import jakarta.jms.ConnectionFactory;
import jakarta.jms.MessageConsumer;
import jakarta.jms.Session;
import jakarta.jms.TextMessage;
import org.apache.activemq.ActiveMQConnectionFactory;

public class RawQueueConsumer {

    private static final String BROKER_URL = "tcp://localhost:61616";
    private static final String QUEUE_NAME = "order.queue";

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ActiveMQConnectionFactory(BROKER_URL);

        Connection connection = factory.createConnection();
        Session session = connection.createSession(false, Session.CLIENT_ACKNOWLEDGE);
        MessageConsumer consumer = session.createConsumer(session.createQueue(QUEUE_NAME));

        connection.start();

        consumer.setMessageListener(message -> {
            try {
                if (message instanceof TextMessage textMessage) {
                    System.out.println("收到消息：" + textMessage.getText());
                }

                message.acknowledge();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        });

        System.out.println("消费者已启动");
        Thread.currentThread().join();
    }
}
```

`CLIENT_ACKNOWLEDGE` 表示客户端手动确认。

注意：

```text
JMS 的 acknowledge() 可能确认当前 Session 中已消费的一批消息。
```

如果需要更强的边界控制，可以考虑事务会话。

### JMS 确认模式

JMS 常见确认模式：

| 模式 | 说明 |
| --- | --- |
| `AUTO_ACKNOWLEDGE` | 监听器正常返回后自动确认 |
| `CLIENT_ACKNOWLEDGE` | 客户端调用 `acknowledge()` 确认 |
| `DUPS_OK_ACKNOWLEDGE` | 允许延迟确认，可能出现重复 |
| `SESSION_TRANSACTED` | 使用事务提交或回滚 |

`AUTO_ACKNOWLEDGE` 使用简单。

关键业务更常见的是事务或手动确认。

### JMS 事务

创建事务会话：

```java
Session session = connection.createSession(true, Session.SESSION_TRANSACTED);
```

处理成功：

```java
session.commit();
```

处理失败：

```java
session.rollback();
```

示例：

```java
MessageConsumer consumer = session.createConsumer(session.createQueue("order.queue"));

TextMessage message = (TextMessage) consumer.receive();

try {
    System.out.println("处理消息：" + message.getText());

    // 执行业务逻辑

    session.commit();
} catch (Exception e) {
    session.rollback();
}
```

事务回滚后，消息会重新投递。

### Spring Boot 依赖

Spring Boot 集成 ActiveMQ Classic：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
</dependency>
```

如果选择 Artemis：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-artemis</artifactId>
</dependency>
```

Classic 和 Artemis 不建议在同一个应用里混用。

### application.yml

ActiveMQ Classic 示例：

```yaml
spring:
  activemq:
    broker-url: tcp://localhost:61616
    user: admin
    password: admin
  jms:
    listener:
      acknowledge-mode: auto
```

如果需要连接池，可以配置连接池依赖和对应参数。

不同 Spring Boot 版本对连接池自动配置细节略有差异，实际项目以当前版本文档为准。

### 开启 JMS 注解

使用 `@JmsListener` 需要开启 JMS 注解支持。

```java
package com.example.activemq.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.jms.annotation.EnableJms;

@Configuration
@EnableJms
public class JmsConfig {
}
```

### 消息对象

```java
package com.example.activemq.order;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public record OrderCreatedMessage(
        Long orderId,
        String orderNo,
        Long userId,
        BigDecimal amount,
        LocalDateTime createdAt
) {
}
```

### JSON 消息转换器

业务系统里更常用 JSON，而不是 Java 原生对象序列化。

```java
package com.example.activemq.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jms.support.converter.MappingJackson2MessageConverter;
import org.springframework.jms.support.converter.MessageConverter;
import org.springframework.jms.support.converter.MessageType;

@Configuration
public class JmsMessageConfig {

    @Bean
    public MessageConverter jacksonJmsMessageConverter() {
        MappingJackson2MessageConverter converter = new MappingJackson2MessageConverter();
        converter.setTargetType(MessageType.TEXT);
        converter.setTypeIdPropertyName("_type");
        return converter;
    }
}
```

### Queue 实战：订单消息

常量：

```java
package com.example.activemq.config;

public class ActiveMqDestinations {

    public static final String ORDER_QUEUE = "order.queue";
    public static final String NOTICE_TOPIC = "notice.topic";

    private ActiveMqDestinations() {
    }
}
```

生产者：

```java
package com.example.activemq.order;

import com.example.activemq.config.ActiveMqDestinations;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.stereotype.Service;

@Service
public class OrderMessageProducer {

    private final JmsTemplate jmsTemplate;

    public OrderMessageProducer(JmsTemplate jmsTemplate) {
        this.jmsTemplate = jmsTemplate;
    }

    public void sendOrderCreated(OrderCreatedMessage message) {
        jmsTemplate.convertAndSend(ActiveMqDestinations.ORDER_QUEUE, message);
    }
}
```

消费者：

```java
package com.example.activemq.order;

import com.example.activemq.config.ActiveMqDestinations;
import org.springframework.jms.annotation.JmsListener;
import org.springframework.stereotype.Component;

@Component
public class OrderMessageConsumer {

    @JmsListener(destination = ActiveMqDestinations.ORDER_QUEUE)
    public void handle(OrderCreatedMessage message) {
        System.out.println("收到订单消息：" + message);

        // 执行业务逻辑：扣库存、发通知、写日志等
    }
}
```

Controller：

```java
package com.example.activemq.order;

import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderMessageProducer producer;

    public OrderController(OrderMessageProducer producer) {
        this.producer = producer;
    }

    @PostMapping
    public String createOrder() {
        OrderCreatedMessage message = new OrderCreatedMessage(
                1001L,
                "O202606300001",
                2001L,
                new BigDecimal("99.80"),
                LocalDateTime.now()
        );

        producer.sendOrderCreated(message);
        return "订单消息已发送";
    }
}
```

访问：

```text
POST http://localhost:8080/api/orders
```

### 同时支持 Queue 和 Topic

`JmsTemplate` 有一个关键属性：

```text
pubSubDomain
```

含义：

```text
false：Queue 模式
true：Topic 模式
```

如果一个项目同时使用 Queue 和 Topic，可以分别创建两个模板。

```java
package com.example.activemq.config;

import jakarta.jms.ConnectionFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jms.config.DefaultJmsListenerContainerFactory;
import org.springframework.jms.core.JmsTemplate;

@Configuration
public class JmsTemplateConfig {

    @Bean
    public JmsTemplate queueJmsTemplate(ConnectionFactory connectionFactory) {
        JmsTemplate template = new JmsTemplate(connectionFactory);
        template.setPubSubDomain(false);
        return template;
    }

    @Bean
    public JmsTemplate topicJmsTemplate(ConnectionFactory connectionFactory) {
        JmsTemplate template = new JmsTemplate(connectionFactory);
        template.setPubSubDomain(true);
        return template;
    }

    @Bean
    public DefaultJmsListenerContainerFactory topicListenerContainerFactory(
            ConnectionFactory connectionFactory
    ) {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setPubSubDomain(true);
        return factory;
    }
}
```

### Topic 实战：广播通知

生产者：

```java
package com.example.activemq.notice;

import com.example.activemq.config.ActiveMqDestinations;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.stereotype.Service;

@Service
public class NoticeProducer {

    private final JmsTemplate topicJmsTemplate;

    public NoticeProducer(@Qualifier("topicJmsTemplate") JmsTemplate topicJmsTemplate) {
        this.topicJmsTemplate = topicJmsTemplate;
    }

    public void publish(String content) {
        topicJmsTemplate.convertAndSend(ActiveMqDestinations.NOTICE_TOPIC, content);
    }
}
```

短信消费者：

```java
package com.example.activemq.notice;

import com.example.activemq.config.ActiveMqDestinations;
import org.springframework.jms.annotation.JmsListener;
import org.springframework.stereotype.Component;

@Component
public class SmsNoticeConsumer {

    @JmsListener(
            destination = ActiveMqDestinations.NOTICE_TOPIC,
            containerFactory = "topicListenerContainerFactory"
    )
    public void handle(String message) {
        System.out.println("短信服务收到通知：" + message);
    }
}
```

邮件消费者：

```java
package com.example.activemq.notice;

import com.example.activemq.config.ActiveMqDestinations;
import org.springframework.jms.annotation.JmsListener;
import org.springframework.stereotype.Component;

@Component
public class EmailNoticeConsumer {

    @JmsListener(
            destination = ActiveMqDestinations.NOTICE_TOPIC,
            containerFactory = "topicListenerContainerFactory"
    )
    public void handle(String message) {
        System.out.println("邮件服务收到通知：" + message);
    }
}
```

一条 Topic 消息会被两个消费者都收到。

### Topic 持久订阅 Demo

非持久订阅只接收在线期间的 Topic 消息。

持久订阅需要设置 `clientId` 和订阅名。

```java
@Bean
public DefaultJmsListenerContainerFactory durableTopicListenerContainerFactory(
        ConnectionFactory connectionFactory
) {
    DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setPubSubDomain(true);
    factory.setSubscriptionDurable(true);
    factory.setClientId("notice-service-01");
    return factory;
}
```

监听：

```java
@JmsListener(
        destination = ActiveMqDestinations.NOTICE_TOPIC,
        subscription = "notice-service-subscription",
        containerFactory = "durableTopicListenerContainerFactory"
)
public void handleDurableTopic(String message) {
    System.out.println("持久订阅收到通知：" + message);
}
```

持久订阅注意点：

```text
同一个 broker 上 clientId 需要唯一
同一个 clientId + subscription 代表同一个持久订阅
```

### 消息选择器

消息选择器可以按消息属性过滤。

发送：

```java
jmsTemplate.convertAndSend("order.queue", message, jmsMessage -> {
    jmsMessage.setStringProperty("bizType", "order");
    jmsMessage.setStringProperty("level", "important");
    return jmsMessage;
});
```

消费：

```java
@JmsListener(
        destination = "order.queue",
        selector = "bizType = 'order' AND level = 'important'"
)
public void handleImportantOrder(OrderCreatedMessage message) {
    System.out.println("收到重要订单消息：" + message);
}
```

选择器适合按少量稳定属性过滤。

如果路由规则很复杂，通常更适合拆分队列或 Topic。

### 延迟消息

ActiveMQ Classic 支持调度消息。

常见属性：

| 属性 | 作用 |
| --- | --- |
| `AMQ_SCHEDULED_DELAY` | 延迟多久投递 |
| `AMQ_SCHEDULED_PERIOD` | 重复投递周期 |
| `AMQ_SCHEDULED_REPEAT` | 重复次数 |
| `AMQ_SCHEDULED_CRON` | Cron 表达式 |

发送延迟消息：

```java
import org.apache.activemq.ScheduledMessage;

jmsTemplate.convertAndSend("order.timeout.queue", message, jmsMessage -> {
    jmsMessage.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_DELAY, 30 * 60 * 1000L);
    return jmsMessage;
});
```

表示 30 分钟后投递。

Broker 需要开启 scheduler 支持。

### 死信队列

ActiveMQ Classic 默认有死信队列。

常见默认名称：

```text
ActiveMQ.DLQ
```

消息可能进入死信队列的常见原因：

* 消费者重复失败
* 消息过期
* 消息被回滚多次
* 达到最大重投递次数

ActiveMQ 常见处理链路：

```text
消费者处理失败
  |
  v
Session rollback 或监听器抛异常
  |
  v
Broker 重新投递
  |
  v
超过最大重投递次数
  |
  v
进入 DLQ
```

死信队列适合：

* 保留异常消息
* 后台补偿处理
* 人工排查
* 告警通知

### Redelivery 重投递

ActiveMQ 客户端可以设置重投递策略。

```java
import org.apache.activemq.ActiveMQConnectionFactory;
import org.apache.activemq.RedeliveryPolicy;

ActiveMQConnectionFactory factory = new ActiveMQConnectionFactory("tcp://localhost:61616");

RedeliveryPolicy policy = new RedeliveryPolicy();
policy.setMaximumRedeliveries(3);
policy.setInitialRedeliveryDelay(1000);
policy.setRedeliveryDelay(2000);
policy.setUseExponentialBackOff(true);
policy.setBackOffMultiplier(2.0);

factory.setRedeliveryPolicy(policy);
```

含义：

| 配置 | 说明 |
| --- | --- |
| `maximumRedeliveries` | 最大重投递次数 |
| `initialRedeliveryDelay` | 首次重投递延迟 |
| `redeliveryDelay` | 重投递间隔 |
| `useExponentialBackOff` | 是否指数退避 |
| `backOffMultiplier` | 退避倍数 |

重投递次数耗尽后，消息通常会进入 DLQ。

### Spring 事务消费

Spring JMS 可以使用事务监听容器。

```java
@Bean
public DefaultJmsListenerContainerFactory txQueueListenerContainerFactory(
        ConnectionFactory connectionFactory
) {
    DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setSessionTransacted(true);
    return factory;
}
```

监听：

```java
@JmsListener(
        destination = ActiveMqDestinations.ORDER_QUEUE,
        containerFactory = "txQueueListenerContainerFactory"
)
public void handleWithTx(OrderCreatedMessage message) {
    System.out.println("事务消费消息：" + message);

    // 方法正常结束：提交 Session
    // 方法抛出异常：回滚 Session，消息等待重投递
}
```

事务消费适合：

* 消息处理成功才确认
* 失败后触发重投递
* 和本地数据库事务配合

### 幂等处理

消息中间件通常更接近“至少一次”投递。

因此，消费者需要能处理重复消息。

常见做法：

```text
消息带唯一 messageId
消费前检查处理记录
未处理则执行业务
处理成功后保存记录
重复消息直接跳过
```

示例：

```java
@JmsListener(destination = ActiveMqDestinations.ORDER_QUEUE)
public void handle(OrderCreatedMessage message) {
    String messageId = message.orderNo();

    if (messageLogRepository.exists(messageId)) {
        return;
    }

    processOrder(message);
    messageLogRepository.save(messageId);
}
```

可以用来做幂等的方式：

* 数据库唯一索引
* 消息消费日志表
* Redis `SETNX`
* 业务状态机

### ActiveMQ 数据存储在哪里

ActiveMQ 数据存储在 Broker 节点上。

主要包括：

* 队列和主题元数据
* 持久化消息
* 订阅关系
* 事务和调度消息信息

ActiveMQ Classic 常见持久化存储是：

```text
KahaDB
```

Artemis 常见持久化方式是：

```text
Journal
```

Docker 部署时，需要挂载数据目录。

Classic 常见目录：

```text
/opt/apache-activemq/data
```

不同镜像目录可能不同，需要以镜像说明为准。

示例：

```bash
docker run -d \
  --name activemq-classic \
  -p 61616:61616 \
  -p 8161:8161 \
  -v activemq-data:/opt/apache-activemq/data \
  apache/activemq-classic:latest
```

如果没有挂载数据卷，容器删除后数据也会删除。

消息重启后是否保留，取决于：

```text
消息是否持久化
目的地和 Broker 持久化配置
存储目录是否保留
```

### 管理后台常看指标

ActiveMQ 管理后台常见关注点：

| 指标 | 说明 |
| --- | --- |
| `Pending Messages` | 等待消费的消息 |
| `Consumers` | 消费者数量 |
| `Enqueued` | 入队消息总数 |
| `Dequeued` | 出队消息总数 |
| `Dispatch Queue` | 已分派但未确认消息 |
| `Expired` | 过期消息 |
| `DLQ` | 死信消息 |

排查思路：

```text
Pending Messages 持续增加：消费能力不足或消费者异常
Consumers 为 0：消费者未启动或监听配置错误
DLQ 增加：业务处理失败、重投递耗尽或消息过期
Enqueued 高于 Dequeued：生产速度超过消费速度
```

### 常见使用建议

### Queue 和 Topic 分清楚

Queue 是任务分发。

```text
一条消息只给一个消费者
```

Topic 是广播事件。

```text
一条消息给多个订阅者
```

如果业务是“发短信任务”，通常选 Queue。

如果业务是“订单已支付事件”，多个系统都要收到，通常选 Topic。

### JSON 优先于 Java 原生序列化

跨系统消息更适合 JSON。

原因：

* 可读性更好
* 跨语言更容易
* 版本演进更可控
* 排查消息更方便

### 关键消息设置持久化

发送端可以设置：

```java
jmsTemplate.setDeliveryPersistent(true);
```

原生 JMS 可以设置：

```java
producer.setDeliveryMode(DeliveryMode.PERSISTENT);
```

还需要确保 Broker 持久化存储配置正常。

### 消费者需要幂等

消息可能重复投递。

消费者处理逻辑需要支持重复执行。

例如：

```text
订单已处理 -> 直接跳过
订单未处理 -> 执行业务并记录处理状态
```

### 失败消息进入 DLQ

失败消息不适合无限重试。

更常见的做法：

```text
有限次数重投递
超过次数进入 DLQ
后台补偿或人工处理
```

### 常用 API 汇总

| API / 注解 | 作用 |
| --- | --- |
| `JmsTemplate.convertAndSend(...)` | 发送消息 |
| `@JmsListener` | 监听消息 |
| `ConnectionFactory` | 创建连接 |
| `Connection` | JMS 连接 |
| `Session` | JMS 会话 |
| `Queue` | 点对点目的地 |
| `Topic` | 发布订阅目的地 |
| `MessageProducer` | 消息生产者 |
| `MessageConsumer` | 消息消费者 |
| `TextMessage` | 文本消息 |
| `MappingJackson2MessageConverter` | JSON 消息转换 |
| `Session.CLIENT_ACKNOWLEDGE` | 客户端确认 |
| `session.commit()` | 提交事务会话 |
| `session.rollback()` | 回滚事务会话 |
| `RedeliveryPolicy` | 重投递策略 |

### 总结

`ActiveMQ` 是 Java 生态里经典的 JMS 消息中间件。

它的核心模型很清晰：

```text
Queue：点对点，一条消息一个消费者
Topic：发布订阅，一条消息多个订阅者
```

日常 Spring Boot 项目里，常用组合是：

```text
spring-boot-starter-activemq
JmsTemplate
@JmsListener
MappingJackson2MessageConverter
```

真正落到工程实践，需要重点关注：

* Queue 和 Topic 模型选择
* JSON 消息格式
* 消息持久化
* 事务或确认机制
* 重投递策略
* DLQ 死信队列
* 消费幂等
* 管理后台监控

ActiveMQ 比 RabbitMQ 更贴近 JMS 规范。

维护传统 Java 企业系统、对接 JMS 老系统、使用 Queue / Topic 这种经典模型时，ActiveMQ 依然有明确价值。
