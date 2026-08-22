---
tags:
  - RabbitMQ
  - 消息队列
  - 异步
  - 面试
created: 2026-08-22
---

# RabbitMQ 实战

> RabbitMQ 是企业最常用的消息队列，基于 AMQP 协议。解决异步处理、应用解耦、流量削峰。

---

## 一、核心概念

### 1.1 消息流转模型

```
Producer → Exchange（交换机）→ Queue（队列）→ Consumer
           ↑ 路由键 binding
```

### 1.2 四大核心组件

| 组件 | 作用 | 类比 |
|---|---|---|
| Producer | 消息生产者 | 寄件人 |
| Exchange | 交换机，路由消息 | 快递分拣中心 |
| Queue | 队列，存储消息 | 收件人信箱 |
| Consumer | 消息消费者 | 收件人 |
| Binding | 绑定关系（Exchange→Queue） | 分拣规则 |
| Routing Key | 路由键 | 邮编 |

### 1.3 为什么要用消息队列

| 场景 | 不用 MQ | 用 MQ |
|---|---|---|
| 用户注册 | 同步写 DB + 发邮件 + 发短信，耗时 3s | 写 DB 立即返回，发邮件/短信异步处理 |
| 秒杀 | 瞬间 10 万请求打 DB，DB 炸 | 请求先进队列，按速率消费，DB 压力可控 |
| 订单系统 | 订单服务直接调库存/支付/物流，耦合 | 各服务只管发消息/收消息，解耦 |

---

## 二、四种交换机类型

### 2.1 速查表

| 类型 | 路由规则 | 典型场景 |
|---|---|---|
| **Fanout** | 广播到所有绑定队列 | 通知/广播 |
| **Direct** | 精确匹配 Routing Key | 按类型分发 |
| **Topic** | 模式匹配（* 和 #） | 按主题订阅 |
| **Headers** | 按 Header 属性匹配 | 少用 |

### 2.2 Fanout（广播）

```
Producer → Exchange(fanout)
               ├──→ Queue A → Consumer 1
               ├──→ Queue B → Consumer 2
               └──→ Queue C → Consumer 3
```
不关心 Routing Key，绑定的队列都能收到。

### 2.3 Direct（直连）

```
Producer → Exchange(direct)
               routingKey=info  →→ Queue(info)
               routingKey=error →→ Queue(error)
```
精确匹配 Routing Key。

### 2.4 Topic（主题）

```
Producer → Exchange(topic)
               routingKey=order.insert →→ Queue(order.*)
               routingKey=user.update  →→ Queue(*.update)
```
- `*` 匹配一个词
- `#` 匹配零或多个词

---

## 三、Spring Boot 整合 RabbitMQ

### 3.1 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    listener:
      simple:
        acknowledge-mode: manual    # 手动 ACK
        prefetch: 1                 # 预取数量
```

### 3.2 声明交换机和队列

```java
@Configuration
public class RabbitConfig {
    // 交换机
    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order.exchange");
    }

    // 队列
    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable("order.queue")
            .withArgument("x-dead-letter-exchange", "dlx.exchange")
            .withArgument("x-dead-letter-routing-key", "dlx.key")
            .build();
    }

    // 绑定
    @Bean
    public Binding binding() {
        return BindingBuilder.bind(orderQueue())
            .to(orderExchange())
            .with("order.create");
    }
}
```

### 3.3 生产者发送消息

```java
@Autowired
private RabbitTemplate rabbitTemplate;

public void sendOrder(Order order) {
    rabbitTemplate.convertAndSend(
        "order.exchange",       // 交换机
        "order.create",         // routing key
        order                   // 消息体
    );
}
```

### 3.4 消费者接收消息

```java
@RabbitListener(queues = "order.queue")
public void receive(Order order, Channel channel, Message message) throws IOException {
    try {
        // 处理业务逻辑
        processOrder(order);
        // 手动确认
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    } catch (Exception e) {
        // 处理失败，拒绝并重新入队
        channel.basicNack(
            message.getMessageProperties().getDeliveryTag(),
            false,   // 不批量
            true     // requeue：重新入队
        );
    }
}
```

---

## 四、可靠性投递

### 4.1 三个环节保证

```
Producer → Exchange（确认机制）
    ↓ 路由
Exchange → Queue（返回机制）
    ↓ 存储
Queue → Consumer（ACK 机制）
```

### 4.2 生产者确认

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated    # 交换机确认
    publisher-returns: true                # 路由失败返回
```

```java
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
    if (!ack) {
        log.error("消息未到达交换机: {}", cause);
        // 重发
    }
});

rabbitTemplate.setReturnsCallback(returned -> {
    log.error("路由失败: {}", returned.getMessage());
});
});
```

### 4.3 消费者手动 ACK

```java
// 成功
channel.basicAck(tag, false);
// 失败重试
channel.basicNack(tag, false, true);
// 失败丢弃
channel.basicReject(tag, false);
```

---

## 五、死信队列（DLX）

### 5.1 消息变成死信的条件

- 消息被消费者拒绝（basicNack/basicReject 且 requeue=false）
- 消息 TTL 过期
- 队列达到最大长度

### 5.2 应用场景：延迟队列

```
消息 → 正常队列(TTL=30s)
            ↓ TTL 过期
         死信交换机 → 死信队列 → 消费者
```

```java
// 正常队列配置 TTL + 死信
@Bean
public Queue normalQueue() {
    return QueueBuilder.durable("normal.queue")
        .withArgument("x-message-ttl", 30000)         // 30秒过期
        .withArgument("x-dead-letter-exchange", "dlx.exchange")
        .withArgument("x-dead-letter-routing-key", "dlx.key")
        .build();
}
```

---

## 六、面试高频题

**Q1: RabbitMQ 四种交换机类型？**
> Fanout（广播）、Direct（精确匹配）、Topic（模式匹配）、Headers（头匹配，少用）。

**Q2: 怎么保证消息不丢失？**
> 1. 生产者开启确认模式（ConfirmCallback）；2. 队列持久化 + 消息持久化；3. 消费者手动 ACK。

**Q3: 死信队列是什么？消息什么时候变死信？**
> 被拒绝(requeue=false)、TTL 过期、队列满。死信会被路由到绑定的死信交换机。常用于延迟队列。

**Q4: RabbitMQ 和 Kafka 怎么选？**
> RabbitMQ：消息可靠、路由灵活、延迟低，适合业务消息（订单/通知）。Kafka：高吞吐、日志流处理，适合大数据场景。

**Q5: 消息积压怎么处理？**
> 1. 增加消费者数量；2. 消费者批量消费；3. 临时扩容队列。根因是生产速率 > 消费速率。

---

## 🔗 相关笔记

- [[Kafka入门|Kafka 入门]] — 高吞吐消息队列对比
- [[../04-Spring生态/SpringBoot-Web开发|Spring Boot Web 开发]] — 业务层异步处理
- [[../03-Database/Redis最小复习笔记|Redis 最小复习笔记]] — 分布式锁配合消息消费
- [[../07-面试与项目/Java学习情况分析与实习冲刺计划|学习计划]] — 第二层会用+懂流程