# 03 - Redis 高可用架构

> 本章深入讲解 Redis 三大高可用方案：**主从复制**、**哨兵模式（Sentinel）** 和 **Cluster 集群**，覆盖复制机制、故障转移、数据分片等核心考点。

## 目录

1. [高可用概述](#一高可用概述)
2. [主从复制](#二主从复制)
3. [哨兵模式（Sentinel）](#三哨兵模式sentinel)
4. [Cluster 集群](#四cluster-集群)
5. [三种高可用方案对比](#五三种高可用方案对比)
6. [大厂常见面试题](#六大厂常见面试题)

---

## 一、高可用概述

### 1.1 为什么需要高可用

单节点 Redis 存在以下风险：

| 风险 | 说明 |
|------|------|
| **单点故障** | 节点宕机导致服务不可用 |
| **容量瓶颈** | 单机内存有限，无法存储海量数据 |
| **性能瓶颈** | 单机 QPS 有上限（约 10 万/秒） |

### 1.2 Redis 高可用演进

```
单机 Redis
    │
    ▼ 数据备份 + 读写分离
主从复制（Master-Slave）
    │
    ▼ 自动故障转移
哨兵模式（Sentinel）
    │
    ▼ 数据分片 + 水平扩展
Cluster 集群
```

---

## 二、主从复制

### 2.1 主从复制概述

主从复制是 Redis 高可用的基石：**主节点（Master）** 负责写操作，**从节点（Slave/Replica）** 复制主节点的数据，提供读服务。

```
         ┌─────────────┐
         │   Master    │
         │  (读 + 写)  │
         └──────┬──────┘
                │ 复制
        ┌───────┼───────┐
        ▼       ▼       ▼
   ┌────────┐┌────────┐┌────────┐
   │ Slave1 ││ Slave2 ││ Slave3 │
   │ (只读) ││ (只读) ││ (只读) │
   └────────┘└────────┘└────────┘
```

### 2.2 配置主从复制

```conf
# 从节点 redis.conf
replicaof 192.168.1.100 6379    # 指定主节点地址（Redis 5.0+ 推荐用 replicaof）
# slaveof 192.168.1.100 6379    # 旧版配置

masterauth your_password         # 主节点密码（如有）
replica-read-only yes            # 从节点只读（默认 yes）
```

```bash
# 运行时动态配置
127.0.0.1:6380> REPLICAOF 192.168.1.100 6379
# 取消主从关系
127.0.0.1:6380> REPLICAOF NO ONE
```

### 2.3 全量复制流程（核心考点 ⭐⭐⭐）

当从节点 **首次连接** 主节点或 **无法增量复制** 时，触发全量复制。

```
  Slave（从节点）                          Master（主节点）
      │                                        │
      │  1. PSYNC ? -1（首次请求全量复制）        │
      │ ──────────────────────────────────────► │
      │                                        │
      │  2. +FULLRESYNC <runid> <offset>        │
      │ ◄────────────────────────────────────── │
      │                                        │
      │                          3. 执行 BGSAVE │
      │                             生成 RDB    │
      │                                        │
      │                          4. 期间的写命令 │
      │                             写入 repl   │
      │                             buffer      │
      │                                        │
      │  5. 发送 RDB 文件                       │
      │ ◄────────────────────────────────────── │
      │                                        │
      │  6. 从节点清空旧数据                     │
      │     加载 RDB 文件                       │
      │                                        │
      │  7. 发送 repl buffer 中的增量命令        │
      │ ◄────────────────────────────────────── │
      │                                        │
      │  8. 从节点执行增量命令                    │
      │     达到数据一致                         │
```

**关键参数**：
- `PSYNC ? -1`：`?` 表示不知道主节点 runid，`-1` 表示没有 offset，请求全量复制
- `+FULLRESYNC`：主节点同意全量复制，返回自己的 `runid` 和当前 `offset`

### 2.4 增量复制流程（核心考点 ⭐⭐⭐）

当从节点 **短暂断线重连** 后，可以通过增量复制获取断线期间的数据。

```
Slave（从节点）                            Master（主节点）
    │                                          │
    │  1. PSYNC <runid> <offset>                │
    │ ──────────────────────────────────────►   │
    │                                          │
    │  2. Master 检查：                         │
    │     - runid 是否匹配？                    │
    │     - offset 是否在 repl_backlog 范围内？  │
    │                                          │
    │  ┌── 匹配 ──┐                            │
    │  │          │                            │
    │  │  3. +CONTINUE                         │
    │  │ ◄─────────────────────────────────── │
    │  │                                      │
    │  │  4. 发送 offset 之后的增量命令          │
    │  │ ◄─────────────────────────────────── │
    │  │                                      │
    │  └── 不匹配 ──► 触发全量复制               │
```

### 2.5 repl_backlog_buffer（复制积压缓冲区）

```
repl_backlog_buffer（环形缓冲区，默认 1MB）
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ . │ . │ D │ E │ F │ G │ H │ . │ . │ . │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
              ▲                   ▲
          slave_offset       master_offset

增量复制范围：slave_offset → master_offset 之间的数据

如果 slave_offset 的数据已被覆盖 → 只能全量复制
```

**关键配置**：
```conf
repl-backlog-size 1mb          # 积压缓冲区大小（建议调大，如 256mb）
repl-backlog-ttl 3600          # 主节点无从节点时，缓冲区保留时间
```

**缓冲区大小估算公式**：
```
repl-backlog-size = 主节点每秒写入量(MB) × 平均断线重连时间(秒) × 2（安全系数）
```

### 2.6 主从复制数据一致性问题

| 问题 | 说明 | 应对策略 |
|------|------|---------|
| **异步复制延迟** | 主节点写入后，从节点不一定立即同步 | `min-replicas-to-write`、`min-replicas-max-lag` |
| **读到旧数据** | 从节点可能读到尚未同步的旧版本数据 | 对一致性要求高的读操作走主节点 |
| **从节点数据丢失** | 主从切换时未同步的数据可能丢失 | 配合 Sentinel 的 `min-replicas` 配置 |

```conf
# 至少 1 个从节点延迟不超过 10 秒时，主节点才接受写入
min-replicas-to-write 1
min-replicas-max-lag 10
```

---

## 三、哨兵模式（Sentinel）

### 3.1 哨兵概述

主从复制无法 **自动故障转移**，需要人工介入。Sentinel（哨兵）提供了自动化的高可用方案。

### 3.2 三大核心功能

| 功能 | 说明 |
|------|------|
| **监控（Monitoring）** | 持续检查主从节点是否正常运行 |
| **自动故障转移（Failover）** | 主节点故障时，自动将从节点提升为新的主节点 |
| **通知（Notification）** | 故障转移后，通知客户端新的主节点地址 |

### 3.3 Sentinel 部署架构

```
┌────────────────────────────────────────────┐
│              Sentinel 集群                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │Sentinel-1│ │Sentinel-2│ │Sentinel-3│   │
│  └─────┬────┘ └─────┬────┘ └─────┬────┘   │
└────────┼────────────┼────────────┼─────────┘
         │            │            │
     监控 │        监控 │        监控 │
         │            │            │
         ▼            ▼            ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐
    │ Master  │──│ Slave-1 │  │ Slave-2 │
    │ (6379)  │  │ (6380)  │  │ (6381)  │
    └─────────┘  └─────────┘  └─────────┘
```

### 3.4 主观下线（SDOWN）vs 客观下线（ODOWN）（核心考点 ⭐⭐⭐）

| 类型 | 判断方 | 条件 | 说明 |
|------|--------|------|------|
| **SDOWN**（主观下线） | 单个 Sentinel | 发送 PING 后超过 `down-after-milliseconds` 未收到有效回复 | 可能是网络抖动导致误判 |
| **ODOWN**（客观下线） | Sentinel 集群 | 超过 `quorum` 个 Sentinel 都认为主节点 SDOWN | 确认主节点真的下线，触发故障转移 |

```
Sentinel-1 ──PING──► Master ──(超时)── SDOWN
Sentinel-2 ──PING──► Master ──(超时)── SDOWN
Sentinel-3 ──PING──► Master ──(超时)── SDOWN

3 个 Sentinel 都判定 SDOWN
且数量 ≥ quorum（假设 quorum=2）
    │
    ▼
Master 被标记为 ODOWN（客观下线）
    │
    ▼
触发故障转移流程
```

### 3.5 Leader 选举（Raft 算法思想）

当 Master 被判定为 ODOWN 后，需要选举一个 Sentinel 作为 Leader 来执行故障转移。

**选举过程**：
1. 每个发现 ODOWN 的 Sentinel 向其他 Sentinel 发送 `SENTINEL is-master-down-by-addr` 请求投票
2. 每个 Sentinel 在每个 **配置纪元（epoch）** 中只有 **一票**，投给第一个请求的 Sentinel
3. 获得 **超过半数**（N/2 + 1）投票的 Sentinel 成为 Leader
4. 如果没有 Sentinel 获得多数票，等待一段时间后重新选举

```
Sentinel-1: "我发现 Master 下线了，请投票给我"
    │
    ├──► Sentinel-2: "你是第一个请求的，投你" ✓
    ├──► Sentinel-3: "你是第一个请求的，投你" ✓
    │
    ▼
Sentinel-1 获得 3/3 票（≥ 2），成为 Leader
    │
    ▼
执行故障转移
```

### 3.6 新主节点选举策略

Leader Sentinel 按以下 **优先级** 从从节点中选择新的 Master：

| 优先级 | 选举规则 | 说明 |
|--------|---------|------|
| 1 | 过滤掉不健康的从节点 | 断线时间过长、处于 SDOWN 状态的节点排除 |
| 2 | `replica-priority` 最小 | 手动设置的优先级（值越小优先级越高，0 表示永不参选） |
| 3 | `replication offset` 最大 | 复制偏移量最大，说明数据最完整 |
| 4 | `runid` 最小 | 最后的兜底规则，选择 runid 字典序最小的节点 |

### 3.7 故障转移完整流程

```
1. 发现故障
   │
   ├── Sentinel 对 Master 发送 PING
   ├── 超时 → Sentinel 标记 SDOWN
   └── quorum 个 Sentinel 都 SDOWN → 标记 ODOWN

2. Leader 选举
   │
   ├── Sentinel 互相投票
   └── 获得多数票的 Sentinel 成为 Leader

3. 选择新 Master
   │
   ├── 过滤不健康从节点
   ├── 按 priority → offset → runid 排序
   └── 选出最优从节点

4. 执行故障转移
   │
   ├── 向新 Master 发送 REPLICAOF NO ONE（解除从属关系）
   ├── 向其他 Slave 发送 REPLICAOF <new-master>（切换主节点）
   ├── 将旧 Master 标记为新 Master 的从节点
   └── 通知客户端新的 Master 地址

5. 后续
   │
   └── 旧 Master 恢复后自动成为新 Master 的从节点
```

### 3.8 哨兵集群发现机制（pub/sub）

Sentinel 之间的 **自动发现** 通过 Redis 的 **发布/订阅** 机制实现：

1. 每个 Sentinel 每 2 秒向主节点的 `__sentinel__:hello` 频道发布自己的信息
2. 其他 Sentinel 订阅该频道，获取其他 Sentinel 的地址
3. 新的 Sentinel 加入后自动被集群发现

```
Sentinel-1 ──PUBLISH──► __sentinel__:hello ──SUBSCRIBE──► Sentinel-2
                              │
                              └──SUBSCRIBE──► Sentinel-3

消息内容：Sentinel IP、端口、runid、监控的 Master 信息
```

### 3.9 Sentinel 配置

```conf
# sentinel.conf
port 26379
sentinel monitor mymaster 192.168.1.100 6379 2    # 监控主节点，quorum=2
sentinel down-after-milliseconds mymaster 5000     # 5秒无响应判定 SDOWN
sentinel failover-timeout mymaster 60000           # 故障转移超时时间 60秒
sentinel parallel-syncs mymaster 1                 # 故障转移后同时同步的从节点数
sentinel auth-pass mymaster your_password          # 主节点密码
```

### 3.10 Java 客户端连接 Sentinel

```java
@Configuration
public class RedisSentinelConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisSentinelConfiguration sentinelConfig = new RedisSentinelConfiguration()
            .master("mymaster")
            .sentinel("192.168.1.201", 26379)
            .sentinel("192.168.1.202", 26379)
            .sentinel("192.168.1.203", 26379);
        sentinelConfig.setPassword("your_password");

        return new LettuceConnectionFactory(sentinelConfig);
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

---

## 四、Cluster 集群

### 4.1 Cluster 概述

Redis Cluster 是 Redis 官方的 **分布式方案**，提供数据分片、自动故障转移和水平扩展能力。

### 4.2 16384 哈希槽分片（核心考点 ⭐⭐⭐）

Redis Cluster 将所有数据划分为 **16384 个哈希槽（slot）**，每个节点负责一部分槽。

```
Cluster 哈希槽分配：

Node A: 槽 0 ~ 5460        (5461 个槽)
Node B: 槽 5461 ~ 10922    (5462 个槽)
Node C: 槽 10923 ~ 16383   (5461 个槽)

┌─────────────────────────────────────────────────┐
│                16384 个哈希槽                     │
│ ┌───────────────┬───────────────┬──────────────┐ │
│ │  Node A       │  Node B       │  Node C      │ │
│ │  0 ~ 5460     │  5461 ~ 10922 │ 10923~16383  │ │
│ └───────────────┴───────────────┴──────────────┘ │
└─────────────────────────────────────────────────┘
```

### 4.3 Key 路由算法

```
slot = CRC16(key) % 16384
```

**示例**：
```
CRC16("user:1001") = 28832
28832 % 16384 = 12448 → 路由到 Node C（10923~16383）

CRC16("order:2001") = 7856
7856 % 16384 = 7856  → 路由到 Node B（5461~10922）
```

**Hash Tag**：通过 `{}` 指定参与计算的部分，确保相关 Key 在同一个槽：
```bash
SET {user:1001}.name "张三"      # CRC16("user:1001")
SET {user:1001}.age "25"         # CRC16("user:1001") → 同一个槽
SET {user:1001}.email "a@b.com"  # CRC16("user:1001") → 同一个槽
```

### 4.4 为什么是 16384 个槽

| 考虑因素 | 说明 |
|---------|------|
| **Gossip 消息大小** | 每个节点发送心跳需要携带槽位信息（bitmap），16384 位 = 2KB，可以接受 |
| **集群规模** | Redis 官方建议集群不超过 1000 个节点，16384 个槽足够分配 |
| **如果用 65536** | 心跳消息需要 8KB，带宽消耗翻倍 |

### 4.5 Gossip 协议

Cluster 节点间通过 **Gossip 协议** 通信，交换集群状态信息。

| 消息类型 | 说明 |
|---------|------|
| **PING** | 每个节点每秒随机向几个节点发送 PING，携带自身状态和部分其他节点状态 |
| **PONG** | 收到 PING/MEET 后回复 PONG，携带自身最新状态 |
| **MEET** | 邀请新节点加入集群（`CLUSTER MEET ip port`） |
| **FAIL** | 当某节点被判定为故障，向全集群广播 FAIL 消息 |

```
Node A                    Node B                    Node C
  │                         │                         │
  │────── PING ──────────►│                         │
  │                         │                         │
  │◄────── PONG ──────────│                         │
  │                         │                         │
  │                         │────── PING ──────────►│
  │                         │                         │
  │                         │◄────── PONG ──────────│
  │                         │                         │
  │ (通过交换信息，每个节点逐渐了解整个集群的状态)      │
```

### 4.6 故障检测与自动转移

#### 4.6.1 故障检测

```
1. Node A 向 Node B 发送 PING
2. 超过 cluster-node-timeout 未收到 PONG
3. Node A 将 Node B 标记为 PFAIL（疑似下线，类似 SDOWN）

4. Node A 通过 Gossip 告知其他节点 "B 可能下线了"
5. 如果集群中 **超过半数** 的主节点都将 B 标记为 PFAIL
6. B 被标记为 FAIL（确认下线，类似 ODOWN）
7. 广播 FAIL 消息到整个集群
```

#### 4.6.2 自动故障转移

```
Master-B 被标记为 FAIL
       │
       ▼
Master-B 的 Slave 发起选举
       │
       ▼
其他 Master 节点投票
（每个 configEpoch 只投一票）
       │
       ▼
获得多数票的 Slave 成为新 Master
       │
       ├── 执行 REPLICAOF NO ONE
       ├── 接管 Master-B 的所有槽
       ├── 广播 PONG 通知全集群更新配置
       └── 其他 Slave 改为复制新 Master
```

### 4.7 扩缩容（槽迁移）

#### 4.7.1 扩容：添加新节点

```
原集群（3节点）：
Node A: 0~5460 | Node B: 5461~10922 | Node C: 10923~16383

添加 Node D，重新分配槽：

Node A: 0~4095     | Node B: 5461~9556
Node C: 10923~14847 | Node D: 4096~5460, 9557~10922, 14848~16383

每个节点迁移约 1365 个槽到 Node D
```

#### 4.7.2 槽迁移流程

```
源节点 (Node A)                    目标节点 (Node D)
      │                                  │
      │  1. CLUSTER SETSLOT <slot>       │
      │     IMPORTING <node-A-id>        │
      │                            ◄─────│
      │                                  │
      │  2. CLUSTER SETSLOT <slot>       │
      │     MIGRATING <node-D-id>        │
      │─────►                            │
      │                                  │
      │  3. CLUSTER GETKEYSINSLOT        │
      │     <slot> <count>               │
      │─────► 获取该槽的所有 key          │
      │                                  │
      │  4. MIGRATE <host> <port>        │
      │     <key> 0 <timeout>            │
      │─────────────────────────────────►│
      │     逐个迁移 key                  │
      │                                  │
      │  5. CLUSTER SETSLOT <slot>       │
      │     NODE <node-D-id>             │
      │     通知所有节点更新槽映射          │
```

#### 4.7.3 缩容

```
1. 将待移除节点的所有槽迁移到其他节点
2. 确认该节点不再持有任何槽
3. 从集群中移除该节点：CLUSTER FORGET <node-id>
```

### 4.8 客户端路由：MOVED vs ASK 重定向（核心考点 ⭐⭐⭐）

| 重定向类型 | 触发场景 | 客户端行为 |
|-----------|---------|-----------|
| **MOVED** | 槽已经 **永久迁移** 到其他节点 | 更新本地槽映射表，后续请求直接发往新节点 |
| **ASK** | 槽正在 **迁移中**，部分 Key 已迁移 | 仅本次请求重定向到目标节点（先发 ASKING 命令），不更新本地映射 |

```
客户端                        Node A                    Node D
  │                             │                         │
  │  GET key1                   │                         │
  │────────────────────────────►│                         │
  │                             │ (slot 已永久迁移)         │
  │  -MOVED 12448 Node-D:6379  │                         │
  │◄────────────────────────────│                         │
  │                             │                         │
  │  更新本地映射                │                         │
  │  GET key1                   │                         │
  │─────────────────────────────────────────────────────►│
  │                             │                         │
  │  "value1"                   │                         │
  │◄─────────────────────────────────────────────────────│
```

```
客户端                        Node A                    Node D
  │                             │                         │
  │  GET key2                   │                         │
  │────────────────────────────►│                         │
  │                             │ (slot 正在迁移中)        │
  │  -ASK 12448 Node-D:6379    │                         │
  │◄────────────────────────────│                         │
  │                             │                         │
  │  ASKING + GET key2          │                         │
  │─────────────────────────────────────────────────────►│
  │                             │                         │
  │  "value2"                   │                         │
  │◄─────────────────────────────────────────────────────│
  │                             │                         │
  │  (不更新本地映射，下次仍发到 Node A)                     │
```

### 4.9 Cluster 的限制

| 限制 | 说明 | 解决方案 |
|------|------|---------|
| **多 Key 操作需在同一槽** | `MGET`、`MSET`、Pipeline 中的 Key 必须在同一个槽 | 使用 Hash Tag `{tag}` |
| **事务限制** | `MULTI/EXEC` 中的 Key 必须在同一个槽 | 使用 Hash Tag |
| **Lua 脚本限制** | 脚本中涉及的所有 Key 必须在同一个槽 | 使用 Hash Tag |
| **数据库只有 db0** | Cluster 模式只支持 db0 | 无法使用 SELECT 切换数据库 |
| **批量操作性能** | 跨节点的批量操作需要多次网络往返 | 按槽分组后并行执行 |

### 4.10 Java 客户端连接 Cluster

```java
@Configuration
public class RedisClusterConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisClusterConfiguration clusterConfig = new RedisClusterConfiguration(
            Arrays.asList(
                "192.168.1.101:6379",
                "192.168.1.102:6379",
                "192.168.1.103:6379",
                "192.168.1.104:6379",
                "192.168.1.105:6379",
                "192.168.1.106:6379"
            )
        );
        clusterConfig.setMaxRedirects(3);   // 最大重定向次数
        clusterConfig.setPassword("your_password");

        LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
            .readFrom(ReadFrom.REPLICA_PREFERRED)  // 优先从从节点读
            .build();

        return new LettuceConnectionFactory(clusterConfig, clientConfig);
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

```yaml
# Spring Boot application.yml 简化配置
spring:
  redis:
    cluster:
      nodes:
        - 192.168.1.101:6379
        - 192.168.1.102:6379
        - 192.168.1.103:6379
      max-redirects: 3
    password: your_password
    lettuce:
      pool:
        max-active: 200
        max-idle: 50
        min-idle: 10
```

---

## 五、三种高可用方案对比

| 对比项 | 主从复制 | 哨兵模式 | Cluster 集群 |
|--------|---------|---------|-------------|
| **数据分片** | ✗ 不支持 | ✗ 不支持 | ✓ 16384 槽分片 |
| **自动故障转移** | ✗ 需要人工 | ✓ Sentinel 自动 | ✓ 集群自动 |
| **读写分离** | ✓ 支持 | ✓ 支持 | ✓ 支持 |
| **水平扩展** | ✗ 不支持 | ✗ 不支持 | ✓ 动态扩缩容 |
| **写能力** | 单主 | 单主 | **多主** |
| **存储容量** | 受单机限制 | 受单机限制 | 所有节点容量之和 |
| **部署复杂度** | 低 | 中 | 高 |
| **适用场景** | 读多写少的简单场景 | 中小规模需要自动故障转移 | 大规模、高并发、大数据量 |
| **典型节点数** | 1 主 + N 从 | 3+ Sentinel + 1 主 + N 从 | 6+（3 主 + 3 从） |

### 5.1 选型建议

```
数据量 < 10GB，QPS < 10万
    └── 哨兵模式（Sentinel）

数据量 > 10GB 或 QPS > 10万
    └── Cluster 集群

只需要数据备份和读写分离
    └── 主从复制

数据量 > 100GB，QPS > 50万
    └── Cluster 集群 + 客户端分片
```

---

## 六、大厂常见面试题

### Q1：Redis 主从复制的原理？全量复制和增量复制的区别？

**答：**

**主从复制**：从节点连接主节点后，主节点将数据同步给从节点。分为全量复制和增量复制。

**全量复制**（首次连接或无法增量同步时）：
1. 从节点发送 `PSYNC ? -1`
2. 主节点执行 BGSAVE 生成 RDB 文件
3. 主节点将 RDB 发送给从节点，同时将期间的写命令写入 repl buffer
4. 从节点加载 RDB 后，再执行 repl buffer 中的增量命令

**增量复制**（短暂断线重连）：
1. 从节点发送 `PSYNC <runid> <offset>`
2. 主节点检查 offset 是否在 repl_backlog_buffer 范围内
3. 如果在范围内，发送 offset 之后的增量命令
4. 如果不在范围内（数据已被覆盖），触发全量复制

---

### Q2：哨兵模式是如何实现自动故障转移的？

**答：**

故障转移分为四个阶段：

1. **故障发现**：每个 Sentinel 每秒向主节点发送 PING，超时标记 SDOWN。当 quorum 个 Sentinel 都标记 SDOWN 后，主节点被标记为 ODOWN。

2. **Leader 选举**：基于 Raft 算法思想，Sentinel 互相投票，获得多数票的成为 Leader。

3. **选择新 Master**：Leader 按 `priority 最低 → offset 最大 → runid 最小` 的顺序选择最优从节点。

4. **执行切换**：向新 Master 发送 `REPLICAOF NO ONE`，向其他 Slave 发送 `REPLICAOF <new-master>`，通知客户端更新地址。

---

### Q3：Redis Cluster 的哈希槽是怎么分配的？为什么是 16384 个槽？

**答：**

**哈希槽分配**：
- Cluster 将数据空间划分为 16384 个槽
- 每个主节点负责一部分槽
- Key 通过 `CRC16(key) % 16384` 计算所属槽，路由到对应节点

**为什么是 16384**：
- 考虑到 Gossip 通信开销：每个节点的心跳消息需要携带所有槽的 bitmap（16384 位 = 2KB），如果用 65536 个槽则需要 8KB，带宽消耗增加
- Redis 官方建议集群最多 1000 个节点，16384 个槽完全足够
- 是节省带宽和保证足够槽数量之间的平衡

---

### Q4：MOVED 和 ASK 重定向的区别是什么？

**答：**

| 对比项 | MOVED | ASK |
|--------|-------|-----|
| 含义 | 槽已经 **永久迁移** | 槽 **正在迁移中** |
| 触发场景 | 请求的 Key 所在的槽不属于当前节点 | 请求的 Key 已经迁移到目标节点，但槽还未完全迁移 |
| 客户端行为 | **更新本地槽映射**，后续请求直接发到新节点 | **不更新映射**，仅本次临时重定向 |
| 重定向前 | 直接发送命令到目标节点 | 需要先发送 `ASKING` 命令，再发送实际命令 |

---

### Q5：Redis Cluster 中如何保证多 Key 操作在同一个槽？

**答：**

使用 **Hash Tag** 机制：在 Key 中用 `{}` 包裹一个 tag，Redis 只对 `{}` 内的内容计算 CRC16。

```bash
# 以下三个 Key 都会路由到 CRC16("user:1001") 对应的槽
SET {user:1001}.name "张三"
SET {user:1001}.age "25"
SET {user:1001}.email "test@example.com"

# 这样就可以对这三个 Key 执行 MGET、事务、Lua 脚本等多 Key 操作
MGET {user:1001}.name {user:1001}.age {user:1001}.email
```

**注意**：过度使用 Hash Tag 可能导致 **数据倾斜**（大量 Key 集中在同一个槽），需要合理设计 tag 粒度。

---

### Q6：Gossip 协议是什么？在 Redis Cluster 中起什么作用？

**答：**

Gossip（八卦）协议是一种去中心化的通信协议，节点之间通过互相交换信息，最终使所有节点达到一致状态。

在 Redis Cluster 中的作用：
1. **集群状态同步**：每个节点通过 PING/PONG 消息交换自身和部分其他节点的状态
2. **故障检测**：通过 Gossip 传播 PFAIL 标记，当半数以上主节点都标记某节点 PFAIL 时，判定为 FAIL
3. **新节点发现**：通过 MEET 消息邀请新节点加入，新节点的信息通过 Gossip 逐步传播到全集群
4. **配置更新**：槽映射变更、新 Master 上任等信息通过 Gossip 传播

**优点**：去中心化，无单点故障，最终一致性  
**缺点**：信息传播有延迟，集群规模越大收敛越慢

---

### Q7：如何进行 Redis Cluster 的扩缩容？需要停机吗？

**答：**

**不需要停机**，Redis Cluster 支持在线扩缩容。

**扩容流程**：
1. 启动新节点，通过 `CLUSTER MEET` 加入集群
2. 使用 `redis-cli --cluster reshard` 将部分槽从现有节点迁移到新节点
3. 迁移过程中，客户端通过 ASK 重定向访问正在迁移的 Key
4. 迁移完成后，客户端通过 MOVED 重定向更新映射

**缩容流程**：
1. 将待移除节点的所有槽迁移到其他节点
2. 确认无槽后，通过 `CLUSTER FORGET` 从集群中移除
3. 如果该节点有从节点，从节点会自动切换到其他主节点

整个过程对客户端 **基本透明**，可能出现少量重定向导致的延迟增加。

---

*上一篇：[02-Redis 持久化机制](./02-Redis持久化机制.md)*
