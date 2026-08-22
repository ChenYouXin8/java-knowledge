---
tags:
  - Kafka
  - 消息队列
  - 大数据
  - 面试
created: 2026-08-22
---

# Kafka 入门

> Kafka 是分布式流处理平台，以超高吞吐著称。适合日志收集、大数据处理、实时数据管道。

---

## 一、核心概念

### 1.1 架构模型

```
Producer → Topic（主题）
             ├── Partition 0  →→  Broker 1
             ├── Partition 1  →→  Broker 2
             └── Partition 2  →→  Broker 3
                                     ↑
                              Consumer Group
                              ├── Consumer A (分区 0)
                              ├── Consumer B (分区 1)
                              └── Consumer C (分区 2)
```

### 1.2 核心术语

| 术语 | 作用 | 对比 RabbitMQ |
|---|---|---|
| **Broker** | Kafka 服务器节点 | 等于一个 RabbitMQ 实例 |
| **Topic** | 消息主题 | 等于 Exchange + Queue |
| **Partition** | 分区，Topic 的物理分片 | 无对应，是 Kafka 并行单元 |
| **Offset** | 消息在分区中的偏移量 | 消息序号 |
| **Consumer Group** | 消费者组 | 一组消费者共同消费一个 Topic |
| **Replication** | 副本 | 集群高可用 |

---

## 二、Kafka vs RabbitMQ

| 维度 | Kafka | RabbitMQ |
|---|---|---|
| 设计目标 | 高吞吐、日志流 | 消息可靠、路由灵活 |
| 吞吐量 | 10万+ msg/s | 万级 msg/s |
| 延迟 | 毫秒级 | 微秒级 |
| 消息模型 | Pull（消费者主动拉） | Push（推） |
| 消息可靠性 | 有可能丢（取决于配置） | 更可靠 |
| 路由能力 | 弱（按 Topic+分区） | 强（4 种交换机） |
| 典型场景 | 日志收集、大数据、流处理 | 业务消息、订单、通知 |
| 学习成本 | 中高 | 中 |

> **选型口诀**：业务消息选 RabbitMQ，数据流选 Kafka。

---

## 三、分区与消费者

### 3.1 为什么分区

- 分区 = 并行度：多分区 → 多消费者并行消费
- 分区 = 扩展：不同分区可以分布在不同 Broker

### 3.2 分区分配规则

```
Topic 有 3 个分区，Consumer Group 有 3 个消费者：
  Partition 0 → Consumer A
  Partition 1 → Consumer B
  Partition 2 → Consumer C

消费者 > 分区数时，多余的消费者空闲
消费者 < 分区数时，一个消费者消费多个分区
```

### 3.3 消息语义

| 语义 | 说明 | 配置 |
|---|---|---|
| At Most Once | 最多一次，可能丢 | 自动提交 offset |
| At Least Once | 至少一次，可能重复 | 手动提交 offset |
| Exactly Once | 精确一次 | 事务 + 幂等 |

> 企业开发用 At Least Once + 消费端幂等保证。

---

## 四、Spring Boot 整合 Kafka

### 4.1 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      retries: 3                    # 重试次数
      acks: all                     # 所有副本确认
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: my-group
      auto-offset-reset: earliest
      enable-auto-commit: false      # 手动提交
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
```

### 4.2 生产者

```java
@Autowired
private KafkaTemplate<String, Object> kafkaTemplate;

public void sendOrder(Order order) {
    kafkaTemplate.send("order-topic", order.getId().toString(), order);
}
```

### 4.3 消费者

```java
@KafkaListener(topics = "order-topic", groupId = "my-group")
public void receive(ConsumerRecord<String, Order> record,
                    Acknowledgment ack) {
    try {
        Order order = record.value();
        processOrder(order);
        // 手动提交 offset
        ack.acknowledge();
    } catch (Exception e) {
        log.error("消费失败", e);
        // 不提交 offset，下次会重新消费
    }
}
```

---

## 五、面试高频题

**Q1: Kafka 为什么快？**
> 1. 顺序写磁盘（比随机写快 6000 倍）；2. 零拷贝（sendfile）；3. 批量发送 + 压缩；4. 分区并行。

**Q2: 怎么保证消息不丢？**
> 生产端 acks=all + 重试；Broker 端配置副本 min.insync.replicas=2；消费端手动提交 offset。

**Q3: Kafka 消息积压怎么处理？**
> 增加分区数 + 增加消费者数量（消费者数 ≤ 分区数才有意义）。临时方案：跳过积压消息，后续补偿。

**Q4: Consumer Group 的作用？**
> 同组内一个分区只能被一个消费者消费（实现负载均衡），不同组各自独立消费全量（实现广播）。

**Q5: Kafka 和 RabbitMQ 怎么选？**
> 业务消息（订单/通知/邮件）选 RabbitMQ，数据流（日志/监控/大数据）选 Kafka。

---

## 🔗 相关笔记

- [[RabbitMQ实战|RabbitMQ 实战]] — 业务消息队列对比
- [[../04-Spring生态/SpringBoot-Web开发|Spring Boot Web 开发]] — 异步处理
- [[../03-Database/Redis最小复习笔记|Redis 最小复习笔记]] — 消费幂等
- [[../07-面试与项目/Java学习情况分析与实习冲刺计划|学习计划]] — 第二层会用+懂流程