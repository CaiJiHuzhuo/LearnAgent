# 02 - RocketMQ 架构与原理

## 目录

1. [RocketMQ 整体架构](#一rocketmq-整体架构)
2. [消息存储机制](#二消息存储机制)
3. [消息类型](#三消息类型)
4. [消费模式](#四消费模式)
5. [RocketMQ 高可用](#五rocketmq-高可用)
6. [Spring Boot 整合 RocketMQ](#六spring-boot-整合-rocketmq)
7. [大厂常见面试题](#七大厂常见面试题)

---

## 一、RocketMQ 整体架构

### 1.1 四大核心组件

| 组件 | 说明 |
|------|------|
| **NameServer** | 注册中心，无状态，管理 Broker 路由信息 |
| **Broker** | 消息存储与转发，分 Master 和 Slave |
| **Producer** | 消息生产者，属于某个 Producer Group |
| **Consumer** | 消息消费者，属于某个 Consumer Group |

### 1.2 架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        RocketMQ 集群架构                              │
│                                                                      │
│  ┌────────────┐  ┌────────────┐                                      │
│  │ NameServer │  │ NameServer │   ← 无状态，集群部署，互不通信         │
│  └─────┬──────┘  └─────┬──────┘                                      │
│        │               │                                             │
│   注册/心跳         注册/心跳                                         │
│        │               │                                             │
│  ┌─────▼───────────────▼──────┐  ┌────────────────────────────┐     │
│  │     Broker-Master-A        │  │     Broker-Master-B        │     │
│  │  ┌────────────────────┐    │  │  ┌────────────────────┐    │     │
│  │  │ CommitLog          │    │  │  │ CommitLog          │    │     │
│  │  │ ConsumeQueue       │    │  │  │ ConsumeQueue       │    │     │
│  │  │ IndexFile          │    │  │  │ IndexFile          │    │     │
│  │  └────────────────────┘    │  │  └────────────────────┘    │     │
│  └────────┬───────────────────┘  └────────┬───────────────────┘     │
│           │ 主从同步                        │ 主从同步                │
│  ┌────────▼───────────────────┐  ┌────────▼───────────────────┐     │
│  │     Broker-Slave-A         │  │     Broker-Slave-B         │     │
│  └────────────────────────────┘  └────────────────────────────┘     │
│                                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐    │
│  │ Producer-1 │  │ Producer-2 │  │ Consumer-1 │  │ Consumer-2 │    │
│  │ (Group-A)  │  │ (Group-A)  │  │ (Group-X)  │  │ (Group-X)  │    │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.3 各组件详解

**NameServer：**
- 轻量级注册中心，**无状态**，集群中各节点互不通信
- 每个 Broker 启动时向**所有** NameServer 注册路由信息
- Producer/Consumer 从 NameServer 获取路由表
- Broker 每 30 秒发送心跳，NameServer 每 10 秒检查，120 秒未收到心跳则剔除

**Broker：**
- 负责消息的接收、存储和投递
- 分为 Master（brokerId=0）和 Slave（brokerId>0）
- Master 处理读写请求，Slave 处理读请求（当 Master 繁忙时）
- 消息存储基于 CommitLog + ConsumeQueue 机制

**Producer Group：**
- 同一类生产者的集合，发送同一类消息
- 事务消息回查时，Broker 可向同组任意 Producer 发起回查

**Consumer Group：**
- 同一类消费者的集合，消费同一类消息
- 组内消费者负载均衡，每条消息只被组内一个消费者消费

---

## 二、消息存储机制

### 2.1 存储架构

```
┌────────────────────────────────────────────────────────────────┐
│                    Broker 存储结构                               │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  CommitLog（物理存储）                     │   │
│  │  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐     │   │
│  │  │msg1 │msg2 │msg3 │msg4 │msg5 │msg6 │msg7 │ ... │     │   │
│  │  │TopA │TopB │TopA │TopA │TopB │TopC │TopA │     │     │   │
│  │  └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘     │   │
│  │  （所有 Topic 的消息顺序写入同一个文件，单文件 1GB）         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                     构建索引                                     │
│                          ▼                                      │
│  ┌──────────────────────────────────────────────────┐          │
│  │            ConsumeQueue（逻辑队列/索引）            │          │
│  │  TopicA-Queue0: [offset1, size1, tag1]           │          │
│  │                 [offset3, size3, tag3]           │          │
│  │                 [offset4, size4, tag4]           │          │
│  │  TopicA-Queue1: [offset7, size7, tag7]           │          │
│  │  TopicB-Queue0: [offset2, size2, tag2]           │          │
│  │                 [offset5, size5, tag5]           │          │
│  │  每条记录固定 20 字节（offset 8B + size 4B + tag 8B）│          │
│  └──────────────────────────────────────────────────┘          │
│                          │                                      │
│  ┌──────────────────────────────────────────────────┐          │
│  │            IndexFile（消息索引）                    │          │
│  │  按 Message Key / 时间戳 建立 Hash 索引             │          │
│  │  支持按 Key 或时间范围查询消息                       │          │
│  └──────────────────────────────────────────────────┘          │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 CommitLog

| 特性 | 说明 |
|------|------|
| 存储方式 | 所有 Topic 消息**混合存储**在同一个文件中 |
| 写入方式 | **顺序写**，磁盘顺序写性能接近内存 |
| 文件大小 | 默认单文件 **1GB**，写满后创建新文件 |
| 文件命名 | 以起始偏移量命名，如 `00000000000000000000`、`00000000001073741824` |

### 2.3 ConsumeQueue

| 特性 | 说明 |
|------|------|
| 作用 | 消息的**逻辑队列**，可理解为 CommitLog 的索引 |
| 结构 | 每个 Topic 的每个 Queue 对应一个 ConsumeQueue 文件 |
| 单条记录 | 固定 20 字节：`commitLogOffset(8B) + msgSize(4B) + tagHashCode(8B)` |
| 读取流程 | 先查 ConsumeQueue 获取 offset → 再去 CommitLog 读取完整消息 |

### 2.4 零拷贝：mmap + PageCache

```
传统 IO（4次拷贝）：
  磁盘 → 内核缓冲区 → 用户缓冲区 → Socket缓冲区 → 网卡

mmap 方式（3次拷贝）：
  磁盘 → 内核缓冲区（mmap映射到用户空间）→ Socket缓冲区 → 网卡

RocketMQ 的策略：
  - CommitLog 使用 mmap 方式读写（MappedByteBuffer）
  - 利用 PageCache 加速读取，热数据在内存中
  - 写入时先写 PageCache，异步/同步刷盘到磁盘
```

---

## 三、消息类型

### 3.1 普通消息

**三种发送方式：**

| 方式 | 说明 | 可靠性 | 性能 |
|------|------|--------|------|
| **同步发送** | 发送后等待 Broker 响应 | 高 | 一般 |
| **异步发送** | 发送后通过回调获取结果 | 较高 | 较高 |
| **单向发送** | 只管发送不管结果 | 低 | 最高 |

```java
// 同步发送
SendResult result = producer.send(new Message("TopicTest", "Hello".getBytes()));
System.out.println("发送结果：" + result.getSendStatus());

// 异步发送
producer.send(new Message("TopicTest", "Hello".getBytes()), new SendCallback() {
    @Override
    public void onSuccess(SendResult sendResult) {
        System.out.println("发送成功：" + sendResult.getMsgId());
    }
    @Override
    public void onException(Throwable e) {
        System.out.println("发送失败：" + e.getMessage());
    }
});

// 单向发送（日志采集等不要求可靠性的场景）
producer.sendOneway(new Message("TopicTest", "Hello".getBytes()));
```

### 3.2 顺序消息

> 保证消息按照发送顺序被消费。

**全局顺序 vs 分区顺序：**

| 类型 | 实现方式 | 性能 | 适用场景 |
|------|---------|------|---------|
| **全局顺序** | Topic 只设 1 个 Queue | 低（无法并行） | 极少使用 |
| **分区顺序** | 相同业务 Key 的消息路由到同一 Queue | 较高（Queue级并行） | 订单状态变更 |

```java
// 生产者：相同订单 ID 的消息发送到同一个 Queue
String orderId = "ORDER_001";
Message msg = new Message("OrderTopic", ("创建订单: " + orderId).getBytes());

producer.send(msg, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        String orderId = (String) arg;
        int index = Math.abs(orderId.hashCode()) % mqs.size();
        return mqs.get(index);
    }
}, orderId);

// 消费者：使用 MessageListenerOrderly 保证单 Queue 内顺序消费
consumer.registerMessageListener(new MessageListenerOrderly() {
    @Override
    public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs,
                                                ConsumeOrderlyContext context) {
        for (MessageExt msg : msgs) {
            System.out.println("顺序消费：" + new String(msg.getBody()));
        }
        return ConsumeOrderlyStatus.SUCCESS;
    }
});
```

### 3.3 延迟消息

> 消息发送后不立即被消费，而是等待指定时间后才对消费者可见。

**RocketMQ 18 个延迟级别：**

| 级别 | 延迟时间 | 级别 | 延迟时间 |
|------|---------|------|---------|
| 1 | 1s | 10 | 6min |
| 2 | 5s | 11 | 7min |
| 3 | 10s | 12 | 8min |
| 4 | 30s | 13 | 9min |
| 5 | 1min | 14 | 10min |
| 6 | 2min | 15 | 20min |
| 7 | 3min | 16 | 30min |
| 8 | 4min | 17 | 1h |
| 9 | 5min | 18 | 2h |

```java
// 发送延迟消息（延迟级别 3 = 10秒后消费）
Message msg = new Message("DelayTopic", "延迟消息内容".getBytes());
msg.setDelayTimeLevel(3); // 10秒延迟
producer.send(msg);

// 典型场景：订单超时取消
// 下单时发送延迟消息（30分钟），到时间后检查订单状态
Message orderTimeout = new Message("OrderTimeoutTopic",
    orderId.getBytes());
orderTimeout.setDelayTimeLevel(16); // 30分钟延迟
producer.send(orderTimeout);
```

### 3.4 事务消息

> 保证**本地事务**和**消息发送**的原子性，实现分布式事务的最终一致性。

**核心机制：Half Message + 二阶段提交 + 回查**

```
事务消息完整流程：

  Producer                   Broker                    Consumer
     │                         │                          │
     │  1. 发送 Half Message   │                          │
     │ ───────────────────────→│                          │
     │                         │ （Half消息对消费者不可见）  │
     │  2. 返回发送结果        │                          │
     │ ←───────────────────────│                          │
     │                         │                          │
     │  3. 执行本地事务        │                          │
     │  ┌──────────────┐       │                          │
     │  │ 扣减库存      │       │                          │
     │  │ 更新订单状态  │       │                          │
     │  └──────────────┘       │                          │
     │                         │                          │
     │  4a. 本地事务成功       │                          │
     │     → Commit           │                          │
     │ ───────────────────────→│  5. 投递消息给消费者      │
     │                         │ ────────────────────────→│
     │                         │                          │
     │  4b. 本地事务失败       │                          │
     │     → Rollback         │                          │
     │ ───────────────────────→│  （删除 Half 消息）       │
     │                         │                          │
     │  4c. 未知/超时          │                          │
     │                         │  6. 回查本地事务状态      │
     │ ←───────────────────────│                          │
     │  7. 返回 Commit/Rollback│                          │
     │ ───────────────────────→│                          │
```

```java
// 事务消息生产者
TransactionMQProducer producer = new TransactionMQProducer("tx-group");
producer.setTransactionListener(new TransactionListener() {
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            // 执行本地事务（如扣减库存）
            orderService.createOrder(msg);
            return LocalTransactionState.COMMIT_MESSAGE;
        } catch (Exception e) {
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        // Broker 回查本地事务状态
        String orderId = msg.getKeys();
        OrderStatus status = orderService.queryOrderStatus(orderId);
        switch (status) {
            case SUCCESS:
                return LocalTransactionState.COMMIT_MESSAGE;
            case FAIL:
                return LocalTransactionState.ROLLBACK_MESSAGE;
            default:
                return LocalTransactionState.UNKNOW;
        }
    }
});

// 发送事务消息
Message msg = new Message("TransTopic", orderJson.getBytes());
producer.sendMessageInTransaction(msg, null);
```

### 3.5 批量消息

```java
// 批量发送（同一 Topic，相同 waitStoreMsgOK，不能是延迟/事务消息）
List<Message> messages = new ArrayList<>();
messages.add(new Message("BatchTopic", "msg-1".getBytes()));
messages.add(new Message("BatchTopic", "msg-2".getBytes()));
messages.add(new Message("BatchTopic", "msg-3".getBytes()));
// 注意：批量消息总大小不超过 4MB
producer.send(messages);
```

---

## 四、消费模式

### 4.1 集群消费 vs 广播消费

| 维度 | 集群消费（Clustering） | 广播消费（Broadcasting） |
|------|---------------------|------------------------|
| **消费行为** | 组内竞争消费，每条消息被组内一个消费者消费 | 每条消息被组内所有消费者消费 |
| **Offset 管理** | Broker 端管理 | Consumer 端管理 |
| **失败重试** | 支持 | 不支持 |
| **适用场景** | 业务消费（订单处理） | 配置推送、缓存刷新 |

```
集群消费：
  msg1 ──→ Consumer-1 ✓
  msg2 ──→ Consumer-2 ✓
  msg3 ──→ Consumer-1 ✓  （消息被负载均衡分配）

广播消费：
  msg1 ──→ Consumer-1 ✓
  msg1 ──→ Consumer-2 ✓  （每个消费者都收到）
```

### 4.2 Push vs Pull（长轮询）

> RocketMQ 的 Push 模式实际是**长轮询（Long Polling）**实现。

```
长轮询流程：
  Consumer ──拉取请求──→ Broker
                          │
                     有消息？
                    ├── 是 ──→ 立即返回消息
                    └── 否 ──→ 挂起请求（最长 15秒）
                                │
                           有新消息到达？
                          ├── 是 ──→ 唤醒并返回
                          └── 否 ──→ 超时返回空
```

### 4.3 消费者负载均衡策略

| 策略 | 说明 |
|------|------|
| **AllocateMessageQueueAveragely** | 平均分配（默认），将 Queue 均匀分给消费者 |
| **AllocateMessageQueueAveragelyByCircle** | 环形分配，轮询方式分配 |
| **AllocateMessageQueueByConfig** | 配置分配，手动指定 |
| **AllocateMessageQueueConsistentHash** | 一致性哈希，减少 Rebalance 影响 |

---

## 五、RocketMQ 高可用

### 5.1 主从同步

| 模式 | 说明 | 可靠性 | 性能 |
|------|------|--------|------|
| **同步复制（SYNC_MASTER）** | Master 写入后等 Slave 同步完成再返回 | 高（不丢消息） | 较低 |
| **异步复制（ASYNC_MASTER）** | Master 写入即返回，异步同步到 Slave | 较高（可能丢少量） | 高 |

### 5.2 刷盘策略

| 模式 | 说明 | 可靠性 | 性能 |
|------|------|--------|------|
| **同步刷盘（SYNC_FLUSH）** | 消息写入磁盘后才返回成功 | 最高 | 低 |
| **异步刷盘（ASYNC_FLUSH）** | 消息写入 PageCache 即返回，异步刷盘 | 较高 | 高 |

**推荐配置（金融级可靠）：** 同步复制 + 同步刷盘

```
┌──────────┐    同步复制    ┌──────────┐
│  Master  │──────────────→│  Slave   │
│ 同步刷盘  │               │ 同步刷盘  │
│ 写磁盘后  │               │          │
│ 再返回ACK │               │          │
└──────────┘               └──────────┘

可靠性：消息写入Master磁盘 + 复制到Slave → 双重保障
```

### 5.3 Dledger 模式

> RocketMQ 4.5+ 引入 Dledger（基于 Raft 协议），支持**自动主从切换**。

```
Dledger 集群（3节点）：
  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │  Broker-1  │  │  Broker-2  │  │  Broker-3  │
  │  (Leader)  │  │ (Follower) │  │ (Follower) │
  └─────┬──────┘  └──────┬─────┘  └──────┬─────┘
        │                │               │
        └────── Raft 协议，自动选举 ───────┘

Leader 宕机 → Follower 自动选举为新 Leader → 无需人工干预
```

**传统主从 vs Dledger：**

| 维度 | 传统主从 | Dledger |
|------|---------|---------|
| 主从切换 | 手动 | 自动（Raft 选举） |
| 数据一致性 | 异步复制可能丢数据 | Raft 保证强一致 |
| 部署要求 | 至少 2 节点 | 至少 3 节点 |
| 性能 | 较高 | 略低（Raft 开销） |

---

## 六、Spring Boot 整合 RocketMQ

### 6.1 Maven 依赖

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.2.3</version>
</dependency>
```

### 6.2 配置

```yaml
rocketmq:
  name-server: 127.0.0.1:9876
  producer:
    group: my-producer-group
    send-message-timeout: 3000
    retry-times-when-send-failed: 3
```

### 6.3 生产者

```java
@Service
public class OrderProducer {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    // 同步发送
    public void sendSync(String topic, Object message) {
        SendResult result = rocketMQTemplate.syncSend(topic, message);
        log.info("发送结果：{}", result.getSendStatus());
    }

    // 异步发送
    public void sendAsync(String topic, Object message) {
        rocketMQTemplate.asyncSend(topic, message, new SendCallback() {
            @Override
            public void onSuccess(SendResult sendResult) {
                log.info("异步发送成功：{}", sendResult.getMsgId());
            }
            @Override
            public void onException(Throwable e) {
                log.error("异步发送失败", e);
            }
        });
    }

    // 发送延迟消息
    public void sendDelay(String topic, Object message, int delayLevel) {
        rocketMQTemplate.syncSend(topic,
            MessageBuilder.withPayload(message).build(),
            3000, delayLevel);
    }

    // 发送顺序消息
    public void sendOrderly(String topic, Object message, String hashKey) {
        rocketMQTemplate.syncSendOrderly(topic, message, hashKey);
    }
}
```

### 6.4 消费者

```java
@Component
@RocketMQMessageListener(
    topic = "order-topic",
    consumerGroup = "order-consumer-group",
    consumeMode = ConsumeMode.CONCURRENTLY,  // 并发消费
    messageModel = MessageModel.CLUSTERING     // 集群消费
)
public class OrderConsumer implements RocketMQListener<OrderDTO> {

    @Override
    public void onMessage(OrderDTO order) {
        log.info("收到订单消息：{}", order.getOrderId());
        // 处理业务逻辑
        orderService.processOrder(order);
    }
}

// 顺序消费
@Component
@RocketMQMessageListener(
    topic = "order-status-topic",
    consumerGroup = "order-status-group",
    consumeMode = ConsumeMode.ORDERLY  // 顺序消费
)
public class OrderStatusConsumer implements RocketMQListener<OrderStatusDTO> {

    @Override
    public void onMessage(OrderStatusDTO statusDTO) {
        log.info("顺序消费订单状态变更：{}", statusDTO);
        orderService.updateStatus(statusDTO);
    }
}
```

---

## 七、大厂常见面试题

### Q1：RocketMQ 的架构是怎样的？各组件的作用是什么？

**答：**

RocketMQ 由四大核心组件组成：
1. **NameServer**：轻量级注册中心，无状态，管理 Broker 路由信息。Broker 启动时注册，Producer/Consumer 启动时获取路由
2. **Broker**：消息存储和转发的核心。分 Master 和 Slave，Master 负责读写，Slave 负责读
3. **Producer**：消息生产者，从 NameServer 获取路由，与 Master Broker 建连发送消息
4. **Consumer**：消息消费者，从 NameServer 获取路由，从 Master/Slave Broker 拉取消息消费

### Q2：RocketMQ 的消息存储原理？为什么性能高？

**答：**

存储采用 **CommitLog + ConsumeQueue + IndexFile** 三层结构：
1. **CommitLog**：所有消息顺序写入同一个文件（1GB），**顺序写磁盘性能接近内存**
2. **ConsumeQueue**：每个 Queue 一个索引文件，单条记录 20 字节，消费时先查索引再读 CommitLog
3. **IndexFile**：按 Key/时间建立 Hash 索引，支持按 Key 查询消息

**高性能原因：**
- **顺序写**：CommitLog 顺序追加写，避免随机 IO
- **零拷贝**：使用 mmap（MappedByteBuffer）减少数据拷贝
- **PageCache**：利用操作系统 PageCache 加速读取
- **ConsumeQueue 轻量**：索引记录仅 20 字节，读取极快

### Q3：RocketMQ 事务消息的实现原理？

**答：**

事务消息通过 **Half Message + 二阶段提交 + 回查** 实现：
1. Producer 发送 **Half Message**（对消费者不可见）
2. Broker 返回发送成功
3. Producer 执行**本地事务**
4. 根据本地事务结果，发送 **Commit**（消息可见）或 **Rollback**（删除消息）
5. 若 Broker 未收到确认（网络异常等），定时**回查** Producer 的本地事务状态
6. 默认回查 15 次，间隔 60 秒

### Q4：RocketMQ 如何保证消息顺序？

**答：**

1. **全局顺序**：Topic 只创建 1 个 Queue，所有消息进同一个 Queue——但吞吐量极低
2. **分区顺序**（推荐）：
   - 生产端：使用 `MessageQueueSelector`，将相同业务 Key 的消息路由到同一个 Queue
   - 消费端：使用 `MessageListenerOrderly`，单线程消费每个 Queue
   - 本质：**保证同一 Queue 内的消息有序**

### Q5：RocketMQ 的延迟消息是如何实现的？

**答：**

1. 发送时设置 `delayTimeLevel`（1-18级，对应 1s 到 2h）
2. Broker 收到后先存入特殊 Topic（`SCHEDULE_TOPIC_XXXX`）
3. 定时服务扫描到期的消息，转投到目标 Topic
4. 消费者从目标 Topic 正常消费

**注意：** 开源版不支持任意时间延迟，只支持 18 个固定级别。阿里云商业版支持秒级精度任意延迟。

### Q6：RocketMQ 如何保证高可用？

**答：**

1. **NameServer 集群**：无状态，互不通信，任一节点宕机不影响整体
2. **Broker 主从**：Master 宕机后，Slave 可提供读服务（但无法自动切换写）
3. **Dledger 模式**：基于 Raft 协议自动主从切换，Leader 宕机秒级选举新 Leader
4. **消息持久化**：同步刷盘 + 同步复制保证消息不丢失
5. **发送重试**：Producer 发送失败自动重试其他 Broker
