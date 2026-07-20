### 简介

Kafka 是一个分布式事件流平台，也经常被当作高吞吐消息中间件使用。

它适合处理大量事件数据，比如订单事件、支付事件、日志采集、用户行为、数据同步、实时计算输入流。

简单理解：

```text
Producer
  |
  v
Kafka Topic
  |
  v
Consumer
```

更完整一点：

```text
Producer
  |
  v
Topic
  |
  +--> Partition 0
  +--> Partition 1
  +--> Partition 2
        |
        v
Consumer Group
```

Kafka 的特点不是“消息发完就删”，而是把消息追加到日志里，消费者按 offset 读取。消息能保留一段时间，不同消费者组可以各自消费同一批数据。

一句话概括：

```text
Kafka 适合高吞吐事件流、异步解耦、削峰填谷、日志采集、数据管道和实时计算场景。
```

### Kafka 解决什么问题

一个订单创建成功后，经常会触发很多后续动作：

```text
扣库存
发短信
发邮件
同步搜索索引
写用户行为日志
更新推荐数据
同步数据仓库
```

如果全部同步执行，下单接口会越来越重。

引入 Kafka 后，可以把订单创建变成一个事件：

```text
订单服务
  |
  v
发送 order.created 事件
  |
  v
Kafka
  |
  +--> 库存服务消费
  +--> 通知服务消费
  +--> 搜索服务消费
  +--> 数据分析服务消费
```

常见收益：

* 异步处理：主流程更快返回
* 系统解耦：生产者只关心发事件
* 削峰填谷：高峰消息先写入 Kafka
* 数据复用：多个消费者组各自读取同一主题
* 持久化日志：消息按保留策略保存
* 实时流处理：Kafka Streams、Flink、Spark 等可以接入

### 核心概念

Kafka 的几个概念需要先分清。

| 概念 | 说明 |
| --- | --- |
| `Producer` | 生产者，负责发送消息 |
| `Consumer` | 消费者，负责拉取消息 |
| `Broker` | Kafka 服务节点 |
| `Cluster` | 多个 Broker 组成的集群 |
| `Topic` | 主题，消息的逻辑分类 |
| `Partition` | 分区，Topic 的物理分片 |
| `Offset` | 消息在分区内的位置 |
| `Consumer Group` | 消费者组，用于负载均衡和消费进度隔离 |
| `Replica` | 副本，用于高可用 |
| `Leader` | 分区主副本，负责读写 |

一个 Topic 可以有多个分区：

```text
order.created
  |
  +--> partition-0: offset 0, 1, 2, 3
  +--> partition-1: offset 0, 1, 2, 3
  +--> partition-2: offset 0, 1, 2, 3
```

Kafka 只保证同一个分区内消息有序。

不同分区之间没有全局顺序。

### Topic、Partition、Offset 怎么理解

`Topic` 类似消息分类。

比如：

```text
order.created
order.paid
user.registered
product.changed
```

`Partition` 是 Topic 的分片。

分区带来两个能力：

| 能力 | 说明 |
| --- | --- |
| 并行写入 | 不同分区可以分散到不同 Broker |
| 并行消费 | 同一个消费者组里多个消费者可以分摊分区 |

`Offset` 是消息在某个分区里的位置。

```text
partition-0
offset: 0  1  2  3  4
msg:    A  B  C  D  E
```

消费者并不是“从队列里拿走消息”，而是记录已经消费到哪个 offset。

所以同一个 Topic 可以被多个消费者组独立消费：

```text
order.created
  |
  +--> inventory-group  消费库存逻辑
  +--> notice-group     消费通知逻辑
  +--> search-group     消费搜索同步逻辑
```

每个消费者组都有自己的 offset。

### 消费者组和分区分配

一个消费者组里，同一个分区在同一时刻只会分配给一个消费者。

假设 Topic 有 3 个分区：

```text
order.created
  partition-0
  partition-1
  partition-2
```

消费者组有 2 个消费者：

```text
inventory-group
  consumer-1 -> partition-0, partition-1
  consumer-2 -> partition-2
```

如果消费者组有 4 个消费者：

```text
inventory-group
  consumer-1 -> partition-0
  consumer-2 -> partition-1
  consumer-3 -> partition-2
  consumer-4 -> 空闲
```

消费者数量超过分区数，多出来的消费者不会消费这个 Topic。

所以分区数基本决定了同一个消费者组的最大并行度。

### 顺序性怎么保证

Kafka 的顺序保证范围是分区。

如果同一个订单的消息需要按顺序处理，发送时可以把 `orderId` 作为 key。

```text
key = orderId
```

Kafka 默认会根据 key 选择分区。相同 key 通常进入同一个分区，同一个分区内保持顺序。

示例：

```java
kafkaTemplate.send("order.event", orderId.toString(), message);
```

这样同一个订单的创建、支付、发货事件，会落到同一个分区里。

需要注意：

```text
Kafka 不保证整个 Topic 的全局顺序，只保证单个分区内有序。
```

如果一个 Topic 只有一个分区，可以获得全局顺序，但吞吐和并行能力会下降。

### 本地启动 Kafka

Kafka 新版本可以使用 KRaft 模式，不再依赖 ZooKeeper。

本地开发可以用 Docker Compose 起一个单节点 Kafka。

```yaml
services:
  kafka:
    image: apache/kafka:latest
    container_name: kafka-demo
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
```

启动：

```bash
docker compose up -d
```

创建 Topic：

```bash
docker exec -it kafka-demo /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic order.created \
  --partitions 3 \
  --replication-factor 1
```

查看 Topic：

```bash
docker exec -it kafka-demo /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic order.created
```

### 原生 Java 客户端依赖

非 Spring 项目可以直接使用 `kafka-clients`。

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>${kafka.version}</version>
</dependency>
```

原生客户端适合：

* 非 Spring 项目
* 需要精细控制 Producer / Consumer 配置
* 学习 Kafka 底层消费和提交机制

Spring Boot 项目更常用 Spring Kafka。

### 原生 Producer Demo

```java
package com.example.kafka.raw;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;
import java.util.concurrent.ExecutionException;

public class RawKafkaProducerDemo {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {
            for (int i = 1; i <= 5; i++) {
                String orderId = "1000" + i;
                String value = """
                        {"orderId":"%s","status":"CREATED"}
                        """.formatted(orderId);

                ProducerRecord<String, String> record = new ProducerRecord<>(
                        "order.created",
                        orderId,
                        value
                );

                producer.send(record, (metadata, exception) -> {
                    if (exception == null) {
                        System.out.printf(
                                "发送成功 topic=%s partition=%d offset=%d%n",
                                metadata.topic(),
                                metadata.partition(),
                                metadata.offset()
                        );
                    } else {
                        exception.printStackTrace();
                    }
                }).get();
            }
        }
    }
}
```

几个关键点：

| 配置 | 说明 |
| --- | --- |
| `acks=all` | 等待所有同步副本确认后返回 |
| `enable.idempotence=true` | 开启幂等生产，减少重试导致的重复写入 |
| `key=orderId` | 相同订单进入同一分区，方便保持顺序 |

### 原生 Consumer Demo

```java
package com.example.kafka.raw;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.List;
import java.util.Properties;

public class RawKafkaConsumerDemo {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "inventory-service");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(List.of("order.created"));

            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf(
                            "收到消息 partition=%d offset=%d key=%s value=%s%n",
                            record.partition(),
                            record.offset(),
                            record.key(),
                            record.value()
                    );
                }

                if (!records.isEmpty()) {
                    consumer.commitSync();
                }
            }
        }
    }
}
```

这里关闭了自动提交 offset。

业务处理完成后再 `commitSync()`，语义更清楚。

如果消费过程中抛异常，又提前提交了 offset，就可能出现消息没有处理成功但消费进度已经前移的问题。

### Spring Boot 集成 Kafka

Spring Boot 项目常用 Spring Kafka。

Maven 依赖：

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

如果项目使用 Spring Boot 依赖管理，不需要手动写 Spring Kafka 版本。

`application.yml`：

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      acks: all
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      properties:
        enable.idempotence: true
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.example.kafka"
    listener:
      ack-mode: manual
```

这份配置表示：

| 配置 | 含义 |
| --- | --- |
| `bootstrap-servers` | Kafka 地址 |
| `producer.acks=all` | 生产者等待同步副本确认 |
| `enable.idempotence=true` | 开启生产者幂等 |
| `JsonSerializer` | 发送 JSON 对象 |
| `JsonDeserializer` | 接收 JSON 对象 |
| `enable-auto-commit=false` | 关闭自动提交 offset |
| `ack-mode=manual` | 消费成功后手动提交 |

### Topic 配置

Spring Boot 可以用 `NewTopic` 在启动时创建 Topic。

```java
package com.example.kafka.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class KafkaTopicConfig {

    public static final String ORDER_CREATED_TOPIC = "order.created";

    public static final String ORDER_CREATED_DLT = "order.created.DLT";

    @Bean
    public NewTopic orderCreatedTopic() {
        return new NewTopic(ORDER_CREATED_TOPIC, 3, (short) 1);
    }

    @Bean
    public NewTopic orderCreatedDlt() {
        return new NewTopic(ORDER_CREATED_DLT, 3, (short) 1);
    }
}
```

生产环境通常由运维平台、IaC 脚本或 Kafka 管理流程创建 Topic。

应用启动自动建 Topic 更适合本地开发和测试环境。

### 消息对象

```java
package com.example.kafka.order;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public record OrderCreatedEvent(
        Long orderId,
        Long userId,
        BigDecimal amount,
        LocalDateTime createdAt
) {
}
```

事件对象最好表达已经发生的事实。

比如 `OrderCreatedEvent` 表示订单已经创建，而不是让消费者去猜这是命令、通知还是状态同步。

### Spring Kafka Producer

```java
package com.example.kafka.order;

import com.example.kafka.config.KafkaTopicConfig;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;

    public OrderEventProducer(KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendOrderCreated(OrderCreatedEvent event) {
        String key = event.orderId().toString();

        kafkaTemplate.send(KafkaTopicConfig.ORDER_CREATED_TOPIC, key, event)
                .whenComplete((result, exception) -> {
                    if (exception == null) {
                        var metadata = result.getRecordMetadata();
                        System.out.printf(
                                "订单事件发送成功，topic=%s partition=%d offset=%d%n",
                                metadata.topic(),
                                metadata.partition(),
                                metadata.offset()
                        );
                    } else {
                        exception.printStackTrace();
                    }
                });
    }
}
```

发送时使用 `orderId` 作为 key。

这样同一个订单的事件会进入同一个分区，消费者处理顺序更容易控制。

### Controller 测试接口

```java
package com.example.kafka.order;

import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderEventProducer orderEventProducer;

    public OrderController(OrderEventProducer orderEventProducer) {
        this.orderEventProducer = orderEventProducer;
    }

    @PostMapping("/mock")
    public String mockCreateOrder() {
        OrderCreatedEvent event = new OrderCreatedEvent(
                System.currentTimeMillis(),
                10001L,
                new BigDecimal("99.80"),
                LocalDateTime.now()
        );

        orderEventProducer.sendOrderCreated(event);
        return "ok";
    }
}
```

### Spring Kafka Consumer

```java
package com.example.kafka.order;

import com.example.kafka.config.KafkaTopicConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Component;

@Component
public class OrderCreatedConsumer {

    @KafkaListener(
            topics = KafkaTopicConfig.ORDER_CREATED_TOPIC,
            groupId = "inventory-service"
    )
    public void onMessage(ConsumerRecord<String, OrderCreatedEvent> record,
                          Acknowledgment acknowledgment) {
        try {
            OrderCreatedEvent event = record.value();

            System.out.printf(
                    "库存服务收到订单事件，partition=%d offset=%d orderId=%d%n",
                    record.partition(),
                    record.offset(),
                    event.orderId()
            );

            deductStock(event);

            acknowledgment.acknowledge();
        } catch (RuntimeException e) {
            throw e;
        }
    }

    private void deductStock(OrderCreatedEvent event) {
        System.out.println("扣减库存，orderId=" + event.orderId());
    }
}
```

手动提交 offset 的基本原则：

```text
业务处理成功后提交
业务处理失败时交给重试或死信处理
```

### 批量消费

吞吐要求高时，可以开启批量消费。

配置：

```yaml
spring:
  kafka:
    listener:
      type: batch
      ack-mode: manual
    consumer:
      max-poll-records: 100
```

消费者：

```java
package com.example.kafka.order;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class OrderBatchConsumer {

    @KafkaListener(topics = "order.created", groupId = "report-service")
    public void onBatch(List<ConsumerRecord<String, OrderCreatedEvent>> records,
                        Acknowledgment acknowledgment) {
        for (ConsumerRecord<String, OrderCreatedEvent> record : records) {
            System.out.printf(
                    "报表服务收到订单事件，partition=%d offset=%d%n",
                    record.partition(),
                    record.offset()
            );
        }

        acknowledgment.acknowledge();
    }
}
```

批量消费适合日志、报表、统计、同步类任务。

如果每条消息都要调用外部接口，批量拉取后仍然要控制并发和失败处理。

### 重试和死信 Topic

消费失败后，常见处理方式是：

```text
先重试几次
仍然失败则投递到死信 Topic
```

Spring Kafka 可以配置 `DefaultErrorHandler` 和 `DeadLetterPublishingRecoverer`。

```java
package com.example.kafka.config;

import org.apache.kafka.common.TopicPartition;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
import org.springframework.kafka.listener.DefaultErrorHandler;
import org.springframework.util.backoff.FixedBackOff;

@Configuration
public class KafkaErrorHandlerConfig {

    @Bean
    public DefaultErrorHandler defaultErrorHandler(KafkaTemplate<Object, Object> kafkaTemplate) {
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
                kafkaTemplate,
                (record, exception) -> new TopicPartition(record.topic() + ".DLT", record.partition())
        );

        return new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3L));
    }
}
```

这段配置表示：

| 配置 | 含义 |
| --- | --- |
| `FixedBackOff(1000L, 3L)` | 间隔 1 秒，重试 3 次 |
| `record.topic() + ".DLT"` | 失败后进入原 Topic 对应的死信 Topic |
| `record.partition()` | 尽量保持和原消息相同分区 |

死信消费者可以单独处理：

```java
package com.example.kafka.order;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class OrderCreatedDltConsumer {

    @KafkaListener(topics = "order.created.DLT", groupId = "order-dlt-service")
    public void onDltMessage(ConsumerRecord<String, OrderCreatedEvent> record) {
        System.out.printf(
                "收到死信消息，topic=%s partition=%d offset=%d key=%s%n",
                record.topic(),
                record.partition(),
                record.offset(),
                record.key()
        );
    }
}
```

死信消息通常要落库、告警、人工处理或后台补偿。

### Offset 提交和重复消费

Kafka 消费很难完全避免重复。

常见情况：

```text
业务处理成功
offset 还没提交
消费者宕机
重启后再次拉到同一条消息
```

所以消费者业务要尽量做幂等。

常见幂等手段：

| 场景 | 做法 |
| --- | --- |
| 订单状态更新 | 根据状态机判断是否已经处理 |
| 数据库写入 | 使用业务唯一键 |
| 扣库存 | 使用事件 ID 防重表 |
| 调用外部服务 | 保存请求流水和处理状态 |

一条消息可以带上事件 ID：

```java
public record OrderCreatedEvent(
        String eventId,
        Long orderId,
        Long userId,
        BigDecimal amount,
        LocalDateTime createdAt
) {
}
```

消费前先检查 `eventId` 是否已经处理过，处理成功后记录状态。

### Kafka 和 RabbitMQ 怎么选

Kafka 和 RabbitMQ 都能做消息中间件，但定位不一样。

| 对比项 | Kafka | RabbitMQ |
| --- | --- | --- |
| 核心模型 | 分区日志 | Exchange + Queue |
| 吞吐 | 很高 | 高 |
| 消息保留 | 按时间或大小保留 | 消费确认后通常删除 |
| 多消费者复用 | 多个消费者组各自消费 | 通常通过多个队列实现 |
| 顺序保证 | 分区内有序 | 单队列更直观 |
| 路由能力 | 相对简单 | Exchange 路由更灵活 |
| 典型场景 | 日志、事件流、数据管道、实时计算 | 业务异步、复杂路由、任务分发 |

简单选型：

```text
事件流、高吞吐、日志采集、实时计算：Kafka 更合适
业务任务队列、复杂路由、延迟重试模型：RabbitMQ 更直观
```

### 常见参数

生产者常见参数：

| 参数 | 说明 |
| --- | --- |
| `acks` | 消息确认级别，常见值 `0`、`1`、`all` |
| `enable.idempotence` | 幂等生产者 |
| `retries` | 发送失败重试次数 |
| `batch.size` | 批量发送大小 |
| `linger.ms` | 等待更多消息组成批次的时间 |
| `compression.type` | 压缩类型，如 `gzip`、`snappy`、`lz4`、`zstd` |

消费者常见参数：

| 参数 | 说明 |
| --- | --- |
| `group.id` | 消费者组 ID |
| `enable.auto.commit` | 是否自动提交 offset |
| `auto.offset.reset` | 没有初始 offset 时从哪里开始 |
| `max.poll.records` | 单次 poll 最大消息数 |
| `max.poll.interval.ms` | 两次 poll 之间最大间隔 |
| `session.timeout.ms` | 消费者会话超时 |

Topic 常见参数：

| 参数 | 说明 |
| --- | --- |
| `partitions` | 分区数 |
| `replication.factor` | 副本数 |
| `retention.ms` | 保留时间 |
| `retention.bytes` | 保留大小 |
| `cleanup.policy` | 清理策略，常见 `delete`、`compact` |

### 常见问题

#### 消息为什么重复消费

Kafka 通常按“至少一次”语义使用。

消费者处理成功但 offset 提交失败，重启后会再次消费。

处理方式是消费者业务做幂等。

#### 消息为什么没有顺序

Kafka 只保证分区内有序。

如果相同业务对象需要有序，发送时使用相同 key，比如 `orderId`。

#### 新消费者为什么读不到历史消息

常见原因是 `auto.offset.reset=latest`。

新消费者组没有历史 offset 时，`latest` 会从最新位置开始读。

如果需要从头读，配置：

```yaml
spring:
  kafka:
    consumer:
      auto-offset-reset: earliest
```

#### 消费者数量增加后吞吐没提升

同一个消费者组里，消费者并行度受分区数限制。

如果 Topic 只有 3 个分区，消费者组里最多 3 个消费者能同时消费这个 Topic。

#### 消息积压怎么排查

常见方向：

| 方向 | 说明 |
| --- | --- |
| 消费速度 | 业务处理是否太慢 |
| 分区数 | 分区是否限制了并行度 |
| 批量消费 | 是否可以提高 `max.poll.records` |
| 外部依赖 | 数据库、HTTP 调用是否拖慢 |
| 重试死循环 | 是否一直消费失败又重试 |

可以通过 consumer lag 观察积压情况。

### 实践建议

| 场景 | 建议 |
| --- | --- |
| Spring Boot 项目 | 使用 Spring Kafka |
| 订单类事件 | key 使用 `orderId` |
| 消费成功语义 | 业务成功后再提交 offset |
| 重复消费 | 消费者做幂等 |
| 消费失败 | 配置重试和死信 Topic |
| 高吞吐 | 使用批量、压缩、合理分区 |
| 有序处理 | 控制 key 和分区 |
| Topic 创建 | 生产环境走统一管理流程 |
| 消息结构 | 使用清晰的事件对象和版本字段 |

### 小结

Kafka 的核心是分区日志。

Producer 把事件写入 Topic，Topic 拆成多个 Partition，Consumer Group 按 offset 消费，每个消费者组都有独立进度。

日常 Java 项目里，原生 `kafka-clients` 适合理解底层和做精细控制，Spring Boot 项目更常用 Spring Kafka，通过 `KafkaTemplate` 和 `@KafkaListener` 完成消息收发。

真正落地时，重点不只在“消息能发能收”，还包括分区设计、顺序性、offset 提交、幂等消费、重试、死信 Topic 和积压监控。
