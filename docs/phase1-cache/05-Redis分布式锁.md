# 05 - Redis 分布式锁

> 本章深入讲解 Redis 分布式锁的**实现原理**、**SETNX 手动实现**、**Redisson 框架**、**RedLock 算法**以及**生产环境最佳实践**，是大厂面试高频考点。

## 目录

1. [分布式锁概述](#一分布式锁概述)
2. [SETNX 实现分布式锁](#二setnx-实现分布式锁)
3. [Redisson 分布式锁](#三redisson-分布式锁)
4. [RedLock 算法](#四redlock-算法)
5. [分布式锁方案对比](#五分布式锁方案对比)
6. [生产环境最佳实践](#六生产环境最佳实践)
7. [大厂常见面试题](#七大厂常见面试题)

---

## 一、分布式锁概述

### 1.1 为什么需要分布式锁

在单机环境中，可以使用 `synchronized` 或 `ReentrantLock` 来保证线程安全。但在分布式系统中，**多个 JVM 进程之间无法共享锁**，需要引入外部中间件实现分布式锁。

```
   单机环境：                              分布式环境：
   ┌─────────────┐                   ┌──────────┐  ┌──────────┐
   │   JVM       │                   │  JVM-1   │  │  JVM-2   │
   │ ┌─────────┐ │                   │ Thread A │  │ Thread B │
   │ │ Thread A│ │                   └─────┬────┘  └─────┬────┘
   │ │ Thread B│ │ synchronized           │              │
   │ │ 共享锁   │ │ 即可解决               │  需要外部锁   │
   │ └─────────┘ │                   ┌────▼──────────────▼────┐
   └─────────────┘                   │     分布式锁（Redis）    │
                                     └────────────────────────┘
```

### 1.2 分布式锁需满足的条件

| 条件 | 说明 |
|------|------|
| **互斥性** | 任意时刻，只有一个客户端能持有锁 |
| **不会死锁** | 即使持锁客户端宕机，锁也能被释放（设置过期时间） |
| **加锁和解锁必须是同一客户端** | 防止误删他人的锁 |
| **容错性** | 只要大部分 Redis 节点存活，客户端就能正常加锁/解锁 |
| **可重入性**（可选） | 同一线程可以多次获取同一把锁 |

---

## 二、SETNX 实现分布式锁

### 2.1 基础实现：SET key value NX EX（核心考点 ⭐⭐⭐）

Redis 的 `SET` 命令支持 `NX`（Not eXists）和 `EX`（Expire）参数，**原子性地完成加锁和设置过期时间**。

```bash
# 加锁（原子操作）
SET lock:order:1001 "uuid-thread-1" NX EX 30
# NX：key 不存在时才设置（互斥）
# EX 30：设置过期时间 30 秒（防死锁）

# 返回 OK 表示加锁成功，返回 nil 表示 key 已存在（加锁失败）
```

> **为什么不用 SETNX + EXPIRE 两条命令？**
> 因为 `SETNX` 和 `EXPIRE` 不是原子操作，如果 `SETNX` 成功后宕机，`EXPIRE` 没有执行，锁永远不会过期，导致死锁。

### 2.2 释放锁：Lua 脚本保证原子性（核心考点 ⭐⭐⭐）

释放锁时必须**先判断锁是否属于自己，再删除**，两步操作必须原子执行。

```
   错误方式（非原子操作）：
   线程A: GET lock → 值是自己的 → 准备 DEL
                                     ↑ 此时锁过期，被线程B获取
   线程A:                          DEL lock → 误删了线程B的锁！
```

**正确方式：使用 Lua 脚本**

```lua
-- unlock.lua：判断并释放锁（原子操作）
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

### 2.3 完整 Java 实现

```java
public class RedisDistributedLock {

    private final StringRedisTemplate redisTemplate;
    private final String lockKey;
    private final String lockValue;  // 唯一标识（UUID + 线程ID）
    private final long expireTime;

    public RedisDistributedLock(StringRedisTemplate redisTemplate,
                                 String lockKey, long expireSeconds) {
        this.redisTemplate = redisTemplate;
        this.lockKey = lockKey;
        this.lockValue = UUID.randomUUID() + ":" + Thread.currentThread().getId();
        this.expireTime = expireSeconds;
    }

    /**
     * 尝试加锁
     */
    public boolean tryLock() {
        Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, expireTime, TimeUnit.SECONDS);
        return Boolean.TRUE.equals(result);
    }

    /**
     * 尝试加锁（带重试）
     */
    public boolean tryLock(long waitTime, TimeUnit unit) throws InterruptedException {
        long deadline = System.nanoTime() + unit.toNanos(waitTime);
        while (System.nanoTime() < deadline) {
            if (tryLock()) {
                return true;
            }
            Thread.sleep(50); // 自旋等待
        }
        return false;
    }

    /**
     * 释放锁（Lua 脚本保证原子性）
     */
    public boolean unlock() {
        String script =
            "if redis.call('GET', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('DEL', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";

        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(lockKey),
            lockValue
        );
        return Long.valueOf(1L).equals(result);
    }
}
```

**使用示例**：

```java
public void deductStock(Long productId) {
    RedisDistributedLock lock = new RedisDistributedLock(
        redisTemplate, "lock:stock:" + productId, 30);

    if (lock.tryLock()) {
        try {
            // 业务逻辑：扣减库存
            int stock = getStock(productId);
            if (stock > 0) {
                updateStock(productId, stock - 1);
            }
        } finally {
            lock.unlock(); // 必须在 finally 中释放
        }
    } else {
        throw new RuntimeException("获取锁失败，请稍后重试");
    }
}
```

### 2.4 SETNX 方式的问题

| 问题 | 说明 |
|------|------|
| **锁过期但业务未完成** | 过期时间难以精确预估，可能导致锁提前释放 |
| **不可重入** | 同一线程再次获取同一把锁会失败 |
| **无自动续期** | 需要手动实现续期机制 |
| **单点故障** | 单 Redis 实例宕机，锁失效 |

---

## 三、Redisson 分布式锁

### 3.1 Redisson 简介

Redisson 是 Redis 的 Java 客户端，提供了丰富的分布式工具，包括**分布式锁、信号量、闭锁**等，底层基于 Lua 脚本和 Netty 实现。

### 3.2 RLock 基本使用（核心考点 ⭐⭐⭐）

```java
@Autowired
private RedissonClient redissonClient;

public void deductStock(Long productId) {
    RLock lock = redissonClient.getLock("lock:stock:" + productId);
    try {
        // 尝试加锁：最多等待 5 秒，锁自动释放时间 30 秒
        boolean locked = lock.tryLock(5, 30, TimeUnit.SECONDS);
        if (locked) {
            try {
                // 业务逻辑
                int stock = getStock(productId);
                if (stock > 0) {
                    updateStock(productId, stock - 1);
                }
            } finally {
                lock.unlock();
            }
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}
```

```java
// 简化写法：lock() 方法会一直阻塞直到获取锁
RLock lock = redissonClient.getLock("lock:order");
lock.lock(); // 阻塞等待，自动启用 Watchdog
try {
    // 业务逻辑
} finally {
    // 确保是当前线程持有锁才释放
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

### 3.3 看门狗机制（Watchdog）（核心考点 ⭐⭐⭐）

当调用 `lock()` 方法（不指定 leaseTime）时，Redisson 会启动 **Watchdog 自动续期机制**，防止锁过期但业务未完成。

```
   ┌────────────────────────────────────────────────────┐
   │                  Watchdog 工作原理                    │
   │                                                    │
   │  lock() 调用                                       │
   │    │                                               │
   │    ├── 加锁成功，默认 lockWatchdogTimeout = 30 秒    │
   │    │                                               │
   │    ├── 启动 Watchdog 后台线程                        │
   │    │                                               │
   │    │   ┌──────────────────────────────────────┐    │
   │    │   │  每 10 秒（30 / 3）检查一次            │    │
   │    │   │                                      │    │
   │    │   │  锁还持有？                           │    │
   │    │   │    ├── 是 → 续期到 30 秒（重置 TTL）  │    │
   │    │   │    └── 否 → 停止续期                  │    │
   │    │   └──────────────────────────────────────┘    │
   │    │                                               │
   │    └── unlock() 调用 → 取消 Watchdog 定时任务       │
   └────────────────────────────────────────────────────┘
```

**核心参数**：

| 参数 | 默认值 | 说明 |
|------|-------|------|
| `lockWatchdogTimeout` | 30 秒 | 锁的默认过期时间 |
| 续期间隔 | 10 秒（timeout / 3） | 每隔 10 秒续期一次 |
| 续期动作 | 重置 TTL 为 30 秒 | 将锁的过期时间延长到 30 秒 |

> **注意**：如果调用 `tryLock(waitTime, leaseTime, unit)` 指定了 `leaseTime`，Watchdog **不会启动**，锁会在 `leaseTime` 后自动过期。

### 3.4 可重入锁实现原理（核心考点 ⭐⭐⭐）

Redisson 使用 **Redis Hash 结构** 实现可重入锁，field 存储客户端标识（UUID:线程ID），value 存储重入次数。

```
  Redis 中的锁结构：

  KEY: lock:order:1001
  TYPE: Hash
  ┌───────────────────────────────┐
  │ field                 │ value │
  ├───────────────────────┼───────┤
  │ uuid-xxx:thread-1     │   2   │  ← 重入了 2 次
  └───────────────────────┴───────┘
  TTL: 30000ms
```

**加锁 Lua 脚本**（简化版）：

```lua
-- 加锁 Lua 脚本
-- KEYS[1] = lock key
-- ARGV[1] = 过期时间（ms）
-- ARGV[2] = 客户端标识（uuid:threadId）

-- 1. 锁不存在，直接加锁
if redis.call('EXISTS', KEYS[1]) == 0 then
    redis.call('HINCRBY', KEYS[1], ARGV[2], 1)  -- 设置重入计数为 1
    redis.call('PEXPIRE', KEYS[1], ARGV[1])      -- 设置过期时间
    return nil
end

-- 2. 锁存在且是自己的，重入计数 +1
if redis.call('HEXISTS', KEYS[1], ARGV[2]) == 1 then
    redis.call('HINCRBY', KEYS[1], ARGV[2], 1)  -- 重入计数 +1
    redis.call('PEXPIRE', KEYS[1], ARGV[1])      -- 重置过期时间
    return nil
end

-- 3. 锁被其他客户端持有，返回剩余 TTL
return redis.call('PTTL', KEYS[1])
```

**解锁 Lua 脚本**（简化版）：

```lua
-- 解锁 Lua 脚本
-- 1. 锁不是自己的，直接返回
if redis.call('HEXISTS', KEYS[1], ARGV[2]) == 0 then
    return nil
end

-- 2. 重入计数 -1
local counter = redis.call('HINCRBY', KEYS[1], ARGV[2], -1)

-- 3. 计数归零，真正释放锁
if counter > 0 then
    redis.call('PEXPIRE', KEYS[1], ARGV[1])  -- 重置过期时间
    return 0
else
    redis.call('DEL', KEYS[1])  -- 删除锁
    redis.call('PUBLISH', KEYS[2], ARGV[3])  -- 通知等待的客户端
    return 1
end
```

### 3.5 读写锁（RReadWriteLock）

```java
RReadWriteLock rwLock = redissonClient.getReadWriteLock("rw:lock:config");

// 读锁（共享锁）：多个线程可同时获取
RLock readLock = rwLock.readLock();
readLock.lock();
try {
    // 读操作
    return getConfig();
} finally {
    readLock.unlock();
}

// 写锁（排他锁）：只允许一个线程获取，且阻塞所有读锁
RLock writeLock = rwLock.writeLock();
writeLock.lock();
try {
    // 写操作
    updateConfig(newConfig);
} finally {
    writeLock.unlock();
}
```

| 锁组合 | 是否互斥 |
|--------|---------|
| 读锁 + 读锁 | 不互斥（共享） |
| 读锁 + 写锁 | 互斥 |
| 写锁 + 写锁 | 互斥 |

### 3.6 联锁（RedissonMultiLock）

联锁要求 **所有锁都加锁成功才算成功**，任一失败则全部释放。适用于需要同时锁定多个资源的场景。

```java
RLock lock1 = redissonClient1.getLock("lock:account:A");
RLock lock2 = redissonClient2.getLock("lock:account:B");
RLock lock3 = redissonClient3.getLock("lock:account:C");

// 联锁：同时获取三把锁
RedissonMultiLock multiLock = new RedissonMultiLock(lock1, lock2, lock3);
multiLock.lock();
try {
    // 同时操作 A、B、C 三个账户
    transfer(accountA, accountB, accountC);
} finally {
    multiLock.unlock();
}
```

### 3.7 红锁（RedissonRedLock）

红锁基于 RedLock 算法，使用多个独立 Redis 实例，**过半数实例加锁成功即视为获取锁成功**。

```java
RLock lock1 = redissonClient1.getLock("lock:order");
RLock lock2 = redissonClient2.getLock("lock:order");
RLock lock3 = redissonClient3.getLock("lock:order");

// 红锁：至少 2/3 实例加锁成功
RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);
redLock.lock();
try {
    // 业务逻辑
} finally {
    redLock.unlock();
}
```

> **注意**：Redisson 已在较新版本中标记 `RedissonRedLock` 为废弃，推荐使用 `RedissonMultiLock` 配合多个独立实例实现类似效果。

---

## 四、RedLock 算法

### 4.1 算法流程（核心考点 ⭐⭐⭐）

RedLock 是 Redis 作者 Antirez 提出的分布式锁算法，基于 **N 个独立 Redis 实例**（非集群，而是彼此完全独立的实例）。

```
   客户端                 N 个独立 Redis 实例（如 N=5）
     │
     │  1. 获取当前时间 T1
     │
     ├──► Redis-1: SET lock val NX EX → 成功 ✓
     ├──► Redis-2: SET lock val NX EX → 成功 ✓
     ├──► Redis-3: SET lock val NX EX → 失败 ✗
     ├──► Redis-4: SET lock val NX EX → 成功 ✓
     ├──► Redis-5: SET lock val NX EX → 失败 ✗
     │
     │  2. 获取当前时间 T2
     │
     │  3. 计算加锁耗时 = T2 - T1
     │
     │  4. 判断是否加锁成功：
     │     a) 过半数实例成功（3/5 ✓）
     │     b) 加锁耗时 < 锁的有效期
     │
     │  5. 锁的实际有效时间 = 原TTL - 加锁耗时
     │
     │  失败时：向所有实例发送 DEL 释放锁
```

**算法步骤详解**：

1. 获取当前时间戳 T1（毫秒级）
2. 依次向 N 个 Redis 实例发送加锁请求，每个请求有较短的超时时间（如 5~50ms）
3. 获取当前时间戳 T2，计算加锁总耗时 `elapsed = T2 - T1`
4. 加锁成功条件：**过半实例加锁成功（≥ N/2 + 1）** 且 `elapsed < lockTTL`
5. 锁的实际有效时间 = `lockTTL - elapsed`
6. 如果加锁失败，向**所有实例**发送解锁命令

### 4.2 Martin Kleppmann vs Antirez 之争（核心考点 ⭐⭐）

2016 年，分布式系统专家 Martin Kleppmann 发表文章质疑 RedLock 的安全性，随后 Antirez 进行了反驳，这场争论至今仍是经典话题。

**Martin Kleppmann 的核心观点**：

| 问题 | 说明 |
|------|------|
| **时钟漂移** | RedLock 依赖各 Redis 实例的系统时钟基本同步，如果某实例时钟跳跃，锁可能提前过期 |
| **GC 暂停** | 客户端获取锁后发生长时间 GC 暂停，锁过期后被其他客户端获取，GC 恢复后两个客户端同时持锁 |
| **Fencing Token** | 建议使用递增的 fencing token，资源端检查 token 顺序，但 RedLock 无法提供此机制 |

```
   GC 暂停导致的问题：

   Client A: 获取锁 → GC 暂停 (30s+) → 锁已过期 → GC 恢复 → 认为仍持锁 → 写数据！
                                        ↑
   Client B:                    获取锁（成功） → 写数据！
                                                   ↑
                                            两个客户端同时写！
```

**Antirez 的反驳**：
- 时钟漂移可以通过合理配置 NTP 控制在很小范围内
- GC 暂停问题不是 RedLock 特有的，所有分布式锁都有此问题
- 实际生产中，时钟跳跃和长时间 GC 暂停是极端场景

### 4.3 时钟漂移问题

RedLock 假设所有 Redis 实例的系统时钟大致同步，如果某实例的时钟发生跳跃（如 NTP 校时）：

```
   正常情况：
   Redis-1: TTL 30s  →  29s  →  28s  →  ...  →  0（过期）
   Redis-2: TTL 30s  →  29s  →  28s  →  ...  →  0（过期）

   时钟跳跃：
   Redis-1: TTL 30s  →  29s  →  28s  →  ...  →  0（过期）
   Redis-2: TTL 30s  →  时钟向前跳 20s  →  TTL 变为 10s  → 很快过期！
```

如果 Redis-2 提前过期，另一个客户端可能在 Redis-2 上获取锁，导致两个客户端认为自己持有锁。

---

## 五、分布式锁方案对比

### 5.1 Redis vs Zookeeper

| 对比项 | Redis 分布式锁 | Zookeeper 分布式锁 |
|--------|--------------|-------------------|
| **实现方式** | SET NX EX / Redisson | 临时顺序节点 |
| **一致性保证** | AP（最终一致） | CP（强一致，ZAB 协议） |
| **性能** | 高（内存操作） | 中（需要磁盘持久化+节点同步） |
| **锁释放** | 依赖过期时间 / Watchdog | 客户端断连后临时节点自动删除 |
| **可重入** | Redisson 支持（Hash + 计数） | Curator 框架支持 |
| **公平性** | 默认非公平 | 天然公平（顺序节点排队） |
| **Watch 机制** | 无（需轮询或订阅） | 原生 Watch（监听前一节点删除） |
| **脑裂问题** | RedLock 可缓解 | ZAB 协议保证不会脑裂 |

### 5.2 选型建议

| 场景 | 推荐方案 |
|------|---------|
| **高并发、低延迟** | Redis（Redisson） |
| **强一致性要求** | Zookeeper（Curator） |
| **已有 Redis 基础设施** | Redis（Redisson） |
| **已有 ZK 基础设施** | Zookeeper（Curator） |
| **简单场景** | Redis SETNX + Lua |

---

## 六、生产环境最佳实践

### 6.1 锁粒度设计

```java
// ✗ 粗粒度锁：所有订单共用一把锁，性能差
RLock lock = redissonClient.getLock("lock:order");

// ✓ 细粒度锁：每个订单一把锁，互不影响
RLock lock = redissonClient.getLock("lock:order:" + orderId);

// ✓ 更细粒度：按用户 + 操作类型
RLock lock = redissonClient.getLock("lock:user:" + userId + ":pay");
```

### 6.2 锁超时时间设定

```java
// ✗ 不指定 leaseTime，依赖 Watchdog（适合大部分场景但不是万能的）
lock.lock();

// ✓ 根据业务评估合理的超时时间
// 普通数据库操作：10-30秒
lock.tryLock(5, 30, TimeUnit.SECONDS);

// 长时间任务：使用 Watchdog 自动续期
lock.lock(); // 不指定 leaseTime
```

### 6.3 异常处理与防死锁

```java
RLock lock = redissonClient.getLock("lock:business:" + bizId);
boolean locked = false;
try {
    locked = lock.tryLock(5, TimeUnit.SECONDS);
    if (!locked) {
        log.warn("获取锁失败，业务ID={}", bizId);
        throw new BizException("操作频繁，请稍后重试");
    }
    // 业务逻辑
    doBusiness(bizId);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw new BizException("操作被中断");
} finally {
    // 确保只释放自己持有的锁
    if (locked && lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

### 6.4 避免锁嵌套

```java
// ✗ 危险：嵌套获取不同的锁，可能导致死锁
RLock lockA = redissonClient.getLock("lock:A");
RLock lockB = redissonClient.getLock("lock:B");
lockA.lock();
lockB.lock(); // 如果另一个线程以 B→A 的顺序加锁，死锁！

// ✓ 使用 MultiLock 同时获取
RedissonMultiLock multiLock = new RedissonMultiLock(lockA, lockB);
multiLock.lock();
```

### 6.5 监控与报警

```java
// 记录锁的持有时间，超过阈值报警
long start = System.currentTimeMillis();
lock.lock();
try {
    doBusiness();
} finally {
    long elapsed = System.currentTimeMillis() - start;
    if (elapsed > 5000) {
        log.warn("锁持有时间过长：{}ms，key={}", elapsed, lock.getName());
        // 触发报警
    }
    lock.unlock();
}
```

---

## 七、大厂常见面试题

### Q1：Redis 分布式锁的实现原理？如何防止误删？

**答：**

使用 `SET key value NX EX` 原子命令加锁，`NX` 保证互斥性，`EX` 设置过期时间防止死锁。

**防止误删**：在 value 中存储客户端唯一标识（UUID + 线程ID），释放锁时通过 Lua 脚本原子地执行 "先比较 value 再删除"，确保只释放自己的锁。

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

---

### Q2：Redisson 的 Watchdog 机制是什么？什么时候会启动？

**答：**

Watchdog 是 Redisson 的**自动续期机制**。当调用 `lock()` 或 `tryLock(waitTime)` 但**不指定 leaseTime** 时自动启动。

工作方式：
1. 加锁时默认 TTL 为 30 秒（`lockWatchdogTimeout`）
2. 后台每 10 秒（30/3）检查一次锁是否仍被持有
3. 如果仍持有，将 TTL 续期到 30 秒
4. `unlock()` 后取消 Watchdog 定时任务

**不启动的场景**：调用 `tryLock(waitTime, leaseTime, unit)` 手动指定了 leaseTime，锁会在指定时间后自动过期，不续期。

---

### Q3：Redisson 可重入锁是怎么实现的？

**答：**

Redisson 使用 **Redis Hash 结构** 存储锁信息：
- Key：锁名称
- Field：客户端标识（UUID:线程ID）
- Value：重入计数

加锁时：锁不存在则创建（计数=1）；锁存在且是自己的则计数+1。  
解锁时：计数-1；计数归零则真正删除锁并发布事件通知等待者。

这种设计允许同一线程多次获取同一把锁而不会死锁，与 Java 的 `ReentrantLock` 语义一致。

---

### Q4：RedLock 算法的流程是什么？它有什么争议？

**答：**

**流程**：向 N 个独立 Redis 实例发起加锁请求，如果**过半数实例加锁成功**（≥ N/2+1）且**总耗时小于锁TTL**，则加锁成功。锁的实际有效时间 = TTL - 加锁耗时。失败时向所有实例发送解锁。

**争议**（Martin Kleppmann vs Antirez）：
1. **时钟漂移**：RedLock 依赖各实例时钟大致同步，NTP 校时可能导致时钟跳跃
2. **GC 暂停**：客户端获取锁后 GC 暂停可能导致锁过期，两个客户端同时持锁
3. **Fencing Token**：Martin 建议使用递增 token 保护资源，但 RedLock 不支持

**实际选型**：对一致性要求极高时考虑 Zookeeper；大部分 Redis 场景使用 Redisson 的 Watchdog 即可。

---

### Q5：Redis 分布式锁和 Zookeeper 分布式锁有什么区别？怎么选？

**答：**

| 维度 | Redis | Zookeeper |
|------|-------|-----------|
| 一致性 | AP（最终一致） | CP（强一致） |
| 性能 | 高 | 中 |
| 锁释放 | 依赖过期时间 | 临时节点自动删除 |
| 公平性 | 非公平（需自行实现） | 天然公平（顺序节点） |

**选型建议**：
- **高并发、低延迟场景** → Redis（Redisson），如秒杀、库存扣减
- **强一致性要求** → Zookeeper（Curator），如分布式协调、选举
- **已有基础设施** → 跟随现有技术栈，减少运维成本

---

### Q6：如果锁过期了但业务还没执行完，怎么办？

**答：**

三种解决方案：

1. **Watchdog 自动续期**（推荐）：使用 Redisson 的 `lock()` 方法，不指定 leaseTime，Watchdog 每 10 秒自动续期。

2. **合理设置过期时间**：根据业务 P99 耗时设置足够长的过期时间，如业务 P99 为 2 秒，过期时间设为 10~30 秒。

3. **Lua 脚本手动续期**：启动后台线程定时检查并续期，但实现复杂，不如直接用 Redisson。

**核心原则**：生产环境推荐使用 Redisson，Watchdog 机制已经解决了这个问题。

---

### Q7：分布式锁在生产中有哪些注意事项？

**答：**

1. **锁粒度要细**：按业务 ID 加锁，避免一把大锁导致串行化
2. **释放锁必须在 finally 中**：防止业务异常导致锁未释放
3. **检查是否是自己的锁**：释放前调用 `isHeldByCurrentThread()` 确认
4. **避免锁嵌套**：容易导致死锁，可用 MultiLock 代替
5. **监控锁持有时间**：超过预期阈值报警
6. **tryLock 优于 lock**：设置合理的等待时间，避免无限阻塞
7. **不要在锁内做 RPC/IO 操作**：会大幅增加锁持有时间
