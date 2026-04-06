# 03 - Kafka 架构与原理

> 本章深入讲解 Kafka 的架构设计、分区副本机制、生产消费流程及高吞吐原理。

## 目录

1. [Kafka 整体架构](#一kafka-整体架构)
2. [分区机制](#二分区机制)
3. [副本机制](#三副本机制)
4. [生产者](#四生产者)
5. [消费者](#五消费者)
6. [高吞吐原理](#六高吞吐原理)
7. [Exactly-Once 语义](#七exactly-once-语义)
8. [Spring Boot 整合 Kafka](#八spring-boot-整合-kafka)
9. [大厂常见面试题](#九大厂常见面试题)

---

## 一、Kafka 整体架构

### 1.1 核心组件

```
                    ┌──────────────────────────────────┐
                    │     Zookeeper / KRaft 集群        │
                    │  （元数据管理、Controller 选举）    │
                    └──────────────┬───────────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
   ┌────▼─────┐             ┌─────▼──────┐            ┌──────▼─────┐
   │ Broker 0 │             │  Broker 1  │            │  Broker 2  │
   │┌────────┐│             │┌──────────┐│            │┌──────────┐│
   ││Topic-A ││             ││ Topic-A  ││            ││ Topic-A  ││
   ││Part-0  ││             ││ Part-1   ││            ││ Part-2   ││
   ││(Leader)││             ││(Leader)  ││            ││(Leader)  ││
   │├────────┤│             │├──────────┤│            │├──────────┤│
   ││Topic-A ││             ││ Topic-A  ││            ││ Topic-A  ││
   ││Part-2  ││             ││ Part-0   ││            ││ Part-1   ││
   ││(Follow)││             ││(Follower)││            ││(Follower)││
   │└────────┘│             │└──────────┘│            │└──────────┘│
   └──────────┘             └────────────┘            └────────────┘
        ▲                         ▲                         ▲
        │                         │                         │
   ┌────┴────┐               ┌────┴────┐              ┌────┴────┐
   │Producer │               │Producer │              │Consumer │
   │ Group   │               │ Group   │              │ Group   │
   └─────────┘               └─────────┘              └─────────┘
```

| 组件 | 说明 |
|------|------|
| **Broker** | Kafka 服务节点，负责消息存储和转发 |
| **Topic** | 消息主题，逻辑分类 |
| **Partition** | 分区，Topic 的物理分片，有序且不可变的消息序列 |
| **Replica** | 副本，分区的冗余备份，分为 Leader 和 Follower |
| **Producer** | 消息生产者 |
| **Consumer** | 消息消费者 |
| **Consumer Group** | 消费者组，组内每个分区只能被一个消费者消费 |
| **Zookeeper/KRaft** | 元数据管理、Controller 选举（Kafka 3.x+ 推荐 KRaft） |

### 1.2 Zookeeper vs KRaft

| 对比项 | Zookeeper 模式 | KRaft 模式（Kafka 3.x+） |
|--------|---------------|--------------------------|
| 架构 | 外部依赖 ZK 集群 | Kafka 自身管理元数据 |
| 复杂度 | 需运维 ZK 集群 | 无外部依赖，运维简化 |
| 扩展性 | ZK 成为瓶颈（百万分区） | 支持更多分区 |
| 故障恢复 | Controller 切换依赖 ZK | 基于 Raft 协议，更快 |
| 状态 | 生产可用 | Kafka 3.3+ 生产可用 |

---

## 二、分区机制

### 2.1 分区的作用

1. **并行度**：多分区可被多个消费者并行消费
2. **有序性**：分区内消息严格有序（分区间无序）
3. **扩展性**：分区分布在不同 Broker，水平扩展
4. **负载均衡**：均匀分散到多个 Broker

### 2.2 分区策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| **轮询（RoundRobin）** | 消息均匀分配到各分区 | 无顺序要求 |
| **Key 哈希** | `hash(key) % partitions`，相同 Key 到同一分区 | 保证同 Key 有序 |
| **指定分区** | 手动指定 partition | 特殊路由需求 |
| **自定义分区器** | 实现 Partitioner 接口 | 复杂路由逻辑 |

```java
// 自定义分区器
public class OrderPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        // 按订单ID哈希，保证同一订单的消息有序
        String orderId = (String) key;
        return Math.abs(orderId.hashCode()) % numPartitions;
    }
}
```

### 2.3 分区数设计原则

```
期望吞吐量 / min(生产者单分区吞吐, 消费者单分区吞吐) = 分区数

示例：
- 期望吞吐量：100MB/s
- 单个生产者写入分区速度：50MB/s
- 单个消费者读取分区速度：30MB/s
- 分区数 >= 100 / 30 ≈ 4 个
```

> ⚠️ 分区数过多的问题：文件句柄增多、Leader 选举变慢、端到端延迟增大、Rebalance 时间增长。

---

## 三、副本机制

### 3.1 Leader / Follower 模型

```
Partition 0:
  ┌──────────────┐     同步     ┌──────────────┐
  │   Leader     │ ──────────→  │  Follower 1  │
  │  (Broker 0)  │              │  (Broker 1)  │
  │              │ ──────────→  │              │
  └──────┬───────┘              └──────────────┘
         │             同步     ┌──────────────┐
         └──────────────────→   │  Follower 2  │
                                │  (Broker 2)  │
                                └──────────────┘

- 所有读写请求都由 Leader 处理
- Follower 从 Leader 拉取数据同步（Pull 模式）
- Kafka 2.4+ 支持 Follower 读取（rack-aware）
```

### 3.2 ISR 机制

**ISR（In-Sync Replicas）**：与 Leader 保持同步的副本集合。

| 概念 | 说明 |
|------|------|
| **ISR** | 同步副本集合，包含 Leader + 已追上的 Follower |
| **OSR** | 非同步副本，落后于 Leader 超过 `replica.lag.time.max.ms` |
| **AR** | 所有副本 = ISR + OSR |

```
ISR 动态维护：
- Follower 追上 Leader → 加入 ISR
- Follower 落后超时 → 移出 ISR，进入 OSR
- replica.lag.time.max.ms = 30000（默认 30s）
```

### 3.3 LEO 和 HW

```
Partition 消息：  [msg0] [msg1] [msg2] [msg3] [msg4] [msg5]
                                        ↑              ↑
                                        HW             LEO (Leader)

Leader  LEO = 6, HW = 4
Follow1 LEO = 5
Follow2 LEO = 4  ← HW = min(所有 ISR 的 LEO) = 4
```

| 概念 | 说明 |
|------|------|
| **LEO（Log End Offset）** | 下一条待写入消息的位移 |
| **HW（High Watermark）** | 所有 ISR 副本都已同步到的位移，消费者只能读到 HW 之前的消息 |

**HW 的作用**：保证消费者读到的消息一定是已在所有 ISR 中同步的，避免 Leader 切换导致数据不一致。

### 3.4 Leader 选举

```
Leader 宕机 → Controller 检测到
    │
    ▼
从 ISR 列表中选择第一个存活的副本作为新 Leader
    │
    ▼
如果 ISR 为空？
    ├── unclean.leader.election.enable = false → 分区不可用（保证一致性）
    └── unclean.leader.election.enable = true  → 从 OSR 中选（可能丢数据）
```

---

## 四、生产者

### 4.1 发送流程

```
Producer
    │
    ▼
① 拦截器（ProducerInterceptor）
    │
    ▼
② 序列化器（Serializer）
    │
    ▼
③ 分区器（Partitioner）→ 确定目标分区
    │
    ▼
④ RecordAccumulator（消息累加器/缓冲区）
    │  按分区分组，批量聚合（batch.size = 16KB）
    │  等待时间（linger.ms = 0，可调大以增加批量）
    ▼
⑤ Sender 线程 → 从缓冲区拉取批量消息
    │
    ▼
⑥ 构建 Request → 发送到 Broker
    │
    ▼
⑦ 收到 Response → 回调 Callback
```

### 4.2 ACK 机制

| acks | 含义 | 可靠性 | 性能 |
|------|------|--------|------|
| **0** | 不等待确认，发送即忘 | 最低（可能丢消息） | 最高 |
| **1** | Leader 写入即确认 | 中等（Leader 宕机可能丢） | 中等 |
| **-1 (all)** | 所有 ISR 副本写入后确认 | 最高 | 最低 |

```java
// 生产者配置
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("acks", "all");                    // 最高可靠性
props.put("retries", 3);                     // 重试次数
props.put("batch.size", 16384);              // 批量大小 16KB
props.put("linger.ms", 5);                   // 等待聚合时间
props.put("buffer.memory", 33554432);        // 缓冲区 32MB
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
// 幂等生产者
props.put("enable.idempotence", true);
```

### 4.3 生产者幂等

```
开启：enable.idempotence = true

原理：
Producer 分配 PID（Producer ID）
每条消息带 <PID, Partition, SeqNumber>
Broker 端对比 SeqNumber：
  - SeqNumber = 期望值 → 写入
  - SeqNumber < 期望值 → 重复，丢弃
  - SeqNumber > 期望值 → 乱序，异常

局限：只能保证单分区、单会话内的幂等（重启后 PID 变化）
```

### 4.4 事务消息

```java
// 事务生产者
props.put("transactional.id", "tx-order-001");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions();

try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("topic-a", "key1", "value1"));
    producer.send(new ProducerRecord<>("topic-b", "key2", "value2"));
    // 提交偏移量（Consume-Transform-Produce 模式）
    producer.sendOffsetsToTransaction(offsets, consumerGroupId);
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

```
事务流程：
1. Producer 向 Transaction Coordinator 注册 transactional.id
2. beginTransaction() → 标记事务开始
3. send() → 消息写入目标分区（标记为未提交）
4. commitTransaction() → Coordinator 写入 COMMIT 标记到 __transaction_state
5. Coordinator 向相关分区写入 Transaction Marker
6. 消费者 isolation.level=read_committed 时，只读已提交消息
```

---

## 五、消费者

### 5.1 消费者组

```
Topic（3 个分区）          Consumer Group A
┌─────────────┐
│ Partition 0  │  ──────→  Consumer 1
│ Partition 1  │  ──────→  Consumer 2
│ Partition 2  │  ──────→  Consumer 3
└─────────────┘

规则：
- 同一消费者组内，一个分区只能被一个消费者消费
- 不同消费者组可以独立消费同一 Topic（发布/订阅模型）
- 消费者数量 > 分区数 → 多余消费者空闲
```

### 5.2 分区分配策略

| 策略 | 说明 | 特点 |
|------|------|------|
| **Range** | 按 Topic 维度，分区数/消费者数 | 可能不均匀 |
| **RoundRobin** | 所有分区轮询分配 | 较均匀 |
| **Sticky** | 尽可能保持原分配，减少迁移 | Rebalance 更平滑 |
| **CooperativeSticky** | 渐进式 Rebalance | 不停止全部消费（推荐） |

### 5.3 Offset 管理

```
Offset 存储在 Kafka 内部 Topic：__consumer_offsets（50 个分区）

┌─────────────────────────────────────┐
│        __consumer_offsets           │
│  Key: <group, topic, partition>     │
│  Value: offset                      │
└─────────────────────────────────────┘
```

| 提交方式 | 说明 | 风险 |
|----------|------|------|
| **自动提交** | `enable.auto.commit=true`，定时提交 | 可能重复消费或丢失 |
| **手动同步提交** | `consumer.commitSync()` | 阻塞，性能低 |
| **手动异步提交** | `consumer.commitAsync()` | 可能提交失败 |
| **指定 Offset 提交** | `commitSync(offsets)` | 精确控制，最灵活 |

```java
// 手动提交示例
Properties props = new Properties();
props.put("enable.auto.commit", "false");
props.put("isolation.level", "read_committed"); // 事务消息

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList("orders"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);
    }
    consumer.commitSync(); // 处理完后手动提交
}
```

### 5.4 Rebalance 机制

```
触发条件：
1. 消费者加入/离开消费者组
2. 消费者心跳超时（session.timeout.ms = 45s）
3. 订阅的 Topic 分区数变化

Rebalance 流程：
① Consumer 发送 JoinGroup 请求到 GroupCoordinator
② Coordinator 选择 Group Leader（第一个加入的消费者）
③ Leader 执行分区分配策略
④ Leader 将分配结果发送给 Coordinator
⑤ Coordinator 通过 SyncGroup 响应分发给所有消费者

问题：Rebalance 期间消费者停止消费（Stop The World）
```

**减少 Rebalance 的方法：**
- 合理设置 `session.timeout.ms` 和 `heartbeat.interval.ms`
- 增大 `max.poll.interval.ms`（处理慢时避免被踢出）
- 使用 CooperativeSticky 策略（渐进式 Rebalance）
- 使用静态成员（`group.instance.id`）避免短暂断开触发 Rebalance

---

## 六、高吞吐原理

### 6.1 Kafka 高吞吐的六大原因

| 技术 | 说明 |
|------|------|
| **顺序磁盘 IO** | 消息追加写入日志文件，顺序IO接近内存速度 |
| **零拷贝** | `sendfile()` 系统调用，数据不经过用户态 |
| **PageCache** | 利用操作系统页缓存，减少磁盘IO |
| **批量处理** | 生产端批量发送，消费端批量拉取 |
| **压缩** | 支持 GZIP/Snappy/LZ4/ZSTD 压缩 |
| **分区并行** | 多分区并行读写，水平扩展 |

### 6.2 零拷贝详解

```
传统IO（4次拷贝，4次上下文切换）：
磁盘 → 内核缓冲区 → 用户缓冲区 → Socket缓冲区 → 网卡

零拷贝 sendfile（2次拷贝，2次上下文切换）：
磁盘 → 内核缓冲区 ──────────────────→ 网卡
                   （DMA 直接传输，不经过用户态）
```

### 6.3 消息存储结构

```
Topic-Partition 目录结构：
topic-orders-0/
├── 00000000000000000000.log      # 日志段文件（消息数据）
├── 00000000000000000000.index    # 偏移量索引（稀疏索引）
├── 00000000000000000000.timeindex # 时间戳索引
├── 00000000000001048576.log      # 下一个日志段（约1GB一个段）
├── 00000000000001048576.index
└── 00000000000001048576.timeindex

查找消息流程：
1. 二分查找定位 .log 文件
2. 通过 .index 稀疏索引定位消息在 .log 中的物理位置
3. 顺序扫描到目标消息
```

---

## 七、Exactly-Once 语义

### 7.1 三种消息语义

| 语义 | 说明 | 实现方式 |
|------|------|---------|
| **At Most Once** | 最多一次（可能丢失） | 生产者 acks=0，消费者先提交再处理 |
| **At Least Once** | 至少一次（可能重复） | 生产者 acks=all + 重试，消费者先处理再提交 |
| **Exactly Once** | 精确一次（不丢不重） | 幂等 + 事务 |

### 7.2 实现 Exactly-Once

```
生产端：
  幂等生产者（enable.idempotence=true）
  + 事务（transactional.id）
  → 保证生产端 Exactly-Once

消费端：
  isolation.level=read_committed
  + 手动管理 Offset（与业务处理同一事务）
  → 保证消费端 Exactly-Once

Consume-Transform-Produce 模式：
  消费 → 处理 → 生产 + 提交 Offset 在同一事务中
  → 端到端 Exactly-Once
```

---

## 八、Spring Boot 整合 Kafka

### 8.1 依赖配置

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      acks: all
      retries: 3
      batch-size: 16384
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
    consumer:
      group-id: order-group
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    listener:
      ack-mode: manual_immediate
```

### 8.2 生产者

```java
@Service
public class OrderProducer {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendOrder(String orderId, String orderJson) {
        kafkaTemplate.send("topic-orders", orderId, orderJson)
            .addCallback(
                result -> log.info("发送成功: topic={}, partition={}, offset={}",
                    result.getRecordMetadata().topic(),
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset()),
                ex -> log.error("发送失败: {}", ex.getMessage())
            );
    }
}
```

### 8.3 消费者

```java
@Component
public class OrderConsumer {

    @KafkaListener(topics = "topic-orders", groupId = "order-group")
    public void consume(ConsumerRecord<String, String> record, Acknowledgment ack) {
        try {
            log.info("收到消息: key={}, value={}, partition={}, offset={}",
                record.key(), record.value(), record.partition(), record.offset());
            // 业务处理
            processOrder(record.value());
            // 手动确认
            ack.acknowledge();
        } catch (Exception e) {
            log.error("消费失败", e);
            // 不确认，等待重新投递
        }
    }
}
```

---

## 九、大厂常见面试题

### Q1：Kafka 为什么吞吐量这么高？

**答：** 六大原因：① 顺序磁盘 IO（追加写入，避免随机IO）；② 零拷贝（sendfile 系统调用）；③ PageCache（利用操作系统缓存）；④ 批量处理（生产和消费都是批量操作）；⑤ 压缩（减少网络传输和磁盘占用）；⑥ 分区并行（多分区水平扩展读写能力）。

### Q2：ISR 机制是什么？有什么作用？

**答：** ISR（In-Sync Replicas）是与 Leader 保持同步的副本集合。Follower 落后超过 `replica.lag.time.max.ms`（默认30s）会被移出 ISR。当 `acks=all` 时，Producer 需要等 ISR 中所有副本确认才返回成功。Leader 宕机时，优先从 ISR 中选新 Leader，保证数据不丢失。

### Q3：Kafka 消费者 Rebalance 是什么？如何避免不必要的 Rebalance？

**答：** Rebalance 是消费者组内分区重新分配的过程。触发条件：消费者加入/离开、心跳超时、分区数变化。避免方法：① 增大 `session.timeout.ms` 和 `max.poll.interval.ms`；② 使用 CooperativeSticky 分配策略；③ 设置 `group.instance.id`（静态成员）；④ 确保消费逻辑不超时。

### Q4：Kafka 如何保证消息不丢失？

**答：**
- **生产端**：`acks=all` + `retries>0` + `min.insync.replicas>=2`
- **Broker 端**：`replication.factor>=3`，至少2个ISR副本
- **消费端**：手动提交 Offset（`enable.auto.commit=false`），处理完消息后再提交

### Q5：Kafka 的 Exactly-Once 如何实现？

**答：** 生产端：开启幂等（`enable.idempotence=true`）+ 事务（`transactional.id`），保证单会话内消息不重复。消费端：`isolation.level=read_committed` 只读已提交消息 + 将 Offset 提交与业务处理放入同一事务。完整端到端 Exactly-Once 使用 Consume-Transform-Produce 模式。

### Q6：Kafka 和 RocketMQ 的区别？

**答：**

| 对比项 | Kafka | RocketMQ |
|--------|-------|----------|
| 语言 | Scala/Java | Java |
| 吞吐量 | 极高（百万级 TPS） | 高（十万级 TPS） |
| 延迟 | ms 级 | ms 级 |
| 事务消息 | 支持（0.11+） | 原生支持（Half Message） |
| 延迟消息 | 不原生支持 | 原生支持（18个级别） |
| 消息回溯 | 支持（按 Offset/时间） | 支持（按时间） |
| 适用场景 | 大数据/日志/流处理 | 电商/金融业务消息 |

---

*下一篇：[04-消息可靠性与幂等](./04-消息可靠性与幂等.md)*
