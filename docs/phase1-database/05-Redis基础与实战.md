# 05 - Redis 基础与实战

> 本章讲解 Redis 的数据结构、持久化、高可用、缓存策略及实战应用。

## 目录

1. [Redis 概述](#一redis-概述)
2. [数据结构](#二数据结构)
3. [持久化](#三持久化)
4. [高可用架构](#四高可用架构)
5. [缓存策略](#五缓存策略)
6. [实战应用](#六实战应用)
7. [大厂常见面试题](#七大厂常见面试题)

---

## 一、Redis 概述

### 1.1 Redis 特点

| 特点 | 说明 |
|------|------|
| **高性能** | 内存存储，单线程模型（Redis 6.0+ 网络层多线程） |
| **丰富数据结构** | String/Hash/List/Set/ZSet/Stream 等 |
| **持久化** | RDB + AOF |
| **高可用** | 主从复制 + 哨兵 + Cluster |
| **原子操作** | 单线程保证命令原子性 |

### 1.2 Redis 为什么快？

1. **内存存储**：数据在内存中，读写极快
2. **单线程模型**：避免上下文切换和锁竞争
3. **IO 多路复用**：epoll 高效处理大量连接
4. **高效数据结构**：SDS、ziplist、skiplist 等

---

## 二、数据结构

### 2.1 五大基础数据类型

| 类型 | 底层实现 | 常用命令 | 应用场景 |
|------|---------|---------|---------|
| **String** | SDS | GET/SET/INCR/SETNX | 缓存、计数器、分布式锁 |
| **Hash** | ziplist/hashtable | HGET/HSET/HMGET | 对象缓存（用户信息） |
| **List** | quicklist | LPUSH/RPOP/LRANGE | 消息队列、最新列表 |
| **Set** | intset/hashtable | SADD/SMEMBERS/SINTER | 标签、共同关注、去重 |
| **ZSet** | ziplist/skiplist | ZADD/ZRANGE/ZRANGEBYSCORE | 排行榜、延迟队列 |

### 2.2 特殊数据类型

| 类型 | 说明 | 应用场景 |
|------|------|---------|
| **HyperLogLog** | 基数估算（误差 0.81%） | UV 统计 |
| **Bitmap** | 位图操作 | 签到、在线状态 |
| **GeoSpatial** | 地理位置 | 附近的人 |
| **Stream** | 消息流（Redis 5.0+） | 消息队列 |

### 2.3 ZSet 底层实现（跳表）

```
跳表结构（Skip List）：多层链表，通过随机层数加速查找

Level 3:  HEAD ────────────────────────────── 50
Level 2:  HEAD ───────── 20 ────────────────── 50
Level 1:  HEAD ── 10 ── 20 ── 30 ── 40 ── 50
Level 0:  HEAD ── 10 ── 20 ── 30 ── 40 ── 50  （底层是完整有序链表）

查找 30：从最高层开始，逐层定位，平均时间复杂度 O(log n)
```

---

## 三、持久化

### 3.1 RDB（Redis DataBase）

| 特性 | 说明 |
|------|------|
| 方式 | 内存快照，生成 dump.rdb 文件 |
| 触发 | `save`（阻塞）/`bgsave`（后台 fork 子进程） |
| 优点 | 文件紧凑，恢复快 |
| 缺点 | 可能丢失最后一次快照后的数据 |

### 3.2 AOF（Append Only File）

| 特性 | 说明 |
|------|------|
| 方式 | 记录每个写命令到 appendonly.aof |
| 刷盘策略 | `always`（每次写）/ `everysec`（每秒，推荐）/ `no`（OS 决定） |
| 优点 | 数据安全性高 |
| 缺点 | 文件大，恢复慢 |
| 重写 | `BGREWRITEAOF` 压缩 AOF 文件 |

### 3.3 混合持久化（Redis 4.0+）

```
开启：aof-use-rdb-preamble yes

AOF 重写时：RDB 快照 + 增量 AOF
恢复时：先加载 RDB 部分，再重放 AOF 增量

兼顾 RDB 的快速恢复和 AOF 的数据安全
```

---

## 四、高可用架构

### 4.1 主从复制

```
Master (读写) ──复制──→ Slave 1 (只读)
                      → Slave 2 (只读)
```

- 全量复制：`PSYNC ? -1` → RDB + 缓冲区命令
- 增量复制：断线重连后，基于 offset 传输增量数据

### 4.2 哨兵模式（Sentinel）

| 功能 | 说明 |
|------|------|
| **监控** | 持续检查主从节点是否正常 |
| **自动故障转移** | 主节点宕机时自动选举新主 |
| **通知** | 通知客户端主节点地址变更 |

```
故障转移流程：
1. 哨兵检测到主节点主观下线（SDOWN）
2. 多个哨兵确认 → 客观下线（ODOWN）
3. 哨兵选举 Leader
4. Leader 哨兵选择新主节点（优先级/offset/runid）
5. 新主节点执行 SLAVEOF NO ONE
6. 其他从节点指向新主节点
```

### 4.3 Cluster 集群

| 特性 | 说明 |
|------|------|
| 数据分片 | **16384 个哈希槽**，分配到不同节点 |
| 路由 | `CRC16(key) % 16384` 确定槽位 |
| 高可用 | 每个主节点有从节点，自动故障转移 |
| 扩缩容 | 迁移哈希槽实现 |

---

## 五、缓存策略

### 5.1 缓存三大问题（核心考点 ⭐⭐⭐）

| 问题 | 说明 | 解决方案 |
|------|------|---------|
| **缓存穿透** | 查询不存在的数据，缓存和 DB 都没有 | 布隆过滤器、缓存空值 |
| **缓存击穿** | 热点 Key 过期，大量请求直达 DB | 互斥锁、逻辑过期 |
| **缓存雪崩** | 大量 Key 同时过期 / Redis 宕机 | 随机过期时间、多级缓存、集群 |

### 5.2 缓存穿透解决方案

```
方案一：布隆过滤器
请求 → 布隆过滤器判断 → 不存在 → 直接返回
                      → 可能存在 → 查缓存 → 查 DB

方案二：缓存空值
请求 → 查缓存 → 查 DB → 不存在 → 缓存空值（短 TTL）
```

### 5.3 缓存击穿解决方案

```
方案一：互斥锁
请求 → 缓存未命中 → 获取分布式锁
  ├── 获取成功 → 查 DB → 写缓存 → 释放锁
  └── 获取失败 → 等待重试

方案二：逻辑过期
缓存不设 TTL，在 value 中存储逻辑过期时间
请求 → 读缓存 → 判断逻辑过期
  ├── 未过期 → 返回
  └── 已过期 → 获取锁 → 开线程异步更新 → 返回旧数据
```

### 5.4 缓存更新策略

| 策略 | 说明 | 一致性 |
|------|------|--------|
| **Cache Aside**（旁路缓存） | 读：先缓存后 DB；写：先更新 DB 再删缓存 | 最终一致 |
| **Read/Write Through** | 由缓存层代理读写 DB | 强一致 |
| **Write Behind** | 异步批量写 DB | 弱一致 |

**Cache Aside 模式（最常用）：**

```
写操作：先更新数据库，再删除缓存（而不是更新缓存）
读操作：先读缓存，未命中则读数据库并写缓存

为什么删除缓存而不是更新？
→ 避免并发写导致缓存与 DB 不一致（后更新的先写缓存）
```

---

## 六、实战应用

### 6.1 分布式锁

```java
// SETNX + 过期时间（原子操作）
Boolean locked = redisTemplate.opsForValue()
    .setIfAbsent("lock:order:" + orderId, requestId, 30, TimeUnit.SECONDS);

if (Boolean.TRUE.equals(locked)) {
    try {
        // 业务逻辑
    } finally {
        // 释放锁（Lua 脚本保证原子性）
        String script = "if redis.call('get',KEYS[1]) == ARGV[1] then " +
                        "return redis.call('del',KEYS[1]) else return 0 end";
        redisTemplate.execute(new DefaultRedisScript<>(script, Long.class),
                Collections.singletonList("lock:order:" + orderId), requestId);
    }
}
```

**Redisson 分布式锁（推荐）：**

```java
RLock lock = redissonClient.getLock("lock:order:" + orderId);
try {
    if (lock.tryLock(5, 30, TimeUnit.SECONDS)) {
        // 业务逻辑
    }
} finally {
    lock.unlock();
}
```

### 6.2 排行榜

```java
// 使用 ZSet 实现排行榜
redisTemplate.opsForZSet().add("rank:score", userId, score);

// 获取 TOP 10（分数从高到低）
Set<ZSetOperations.TypedTuple<String>> top10 =
    redisTemplate.opsForZSet().reverseRangeWithScores("rank:score", 0, 9);
```

---

## 七、大厂常见面试题

### Q1：Redis 为什么快？

**答：** 内存存储、单线程避免竞争、IO 多路复用（epoll）、高效数据结构（SDS/skiplist/ziplist）。

### Q2：Redis 的持久化方式？

**答：** RDB（快照，恢复快但可能丢数据）、AOF（命令追加，数据安全但文件大）、混合持久化（RDB + AOF，推荐）。

### Q3：缓存穿透、击穿、雪崩的区别和解决方案？

**答：**
- **穿透**：查不存在的数据 → 布隆过滤器/缓存空值
- **击穿**：热点 Key 过期 → 互斥锁/逻辑过期
- **雪崩**：大量 Key 同时过期 → 随机 TTL/多级缓存/集群

### Q4：如何保证缓存与数据库一致性？

**答：** Cache Aside 模式：写操作先更新 DB 再删除缓存；读操作先读缓存未命中则读 DB 写缓存。极端情况用延迟双删或订阅 binlog 更新缓存。

### Q5：Redis 的分布式锁怎么实现？

**答：** `SET key value NX EX`（原子加锁+过期），释放用 Lua 脚本保证原子性。生产推荐使用 Redisson，支持看门狗自动续期、可重入锁。集群环境可用 RedLock 算法。

### Q6：Redis Cluster 的原理？

**答：** 16384 个哈希槽分配到不同节点，`CRC16(key) % 16384` 确定数据所在槽位。每个主节点有从节点做故障转移。客户端收到 MOVED 重定向后更新槽位映射。

---

*下一篇：[06-数据库常见面试题](./06-数据库常见面试题.md)*
