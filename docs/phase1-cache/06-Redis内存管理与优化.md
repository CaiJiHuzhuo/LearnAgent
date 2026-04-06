# 06 - Redis 内存管理与优化

> 本章深入讲解 Redis 的**内存分析**、**八种淘汰策略**、**大 Key 与热 Key 问题**以及**内存优化技巧**，是大厂面试和生产运维的高频考点。

## 目录

1. [Redis 内存使用分析](#一redis-内存使用分析)
2. [八种内存淘汰策略](#二八种内存淘汰策略)
3. [Redis 大 Key 问题](#三redis-大-key-问题)
4. [热 Key 问题](#四热-key-问题)
5. [Redis 内存优化技巧](#五redis-内存优化技巧)
6. [大厂常见面试题](#六大厂常见面试题)

---

## 一、Redis 内存使用分析

### 1.1 INFO memory 命令详解（核心考点 ⭐⭐）

```bash
127.0.0.1:6379> INFO memory

# Memory
used_memory:1073741824          # Redis 分配器分配的内存总量（字节），即数据占用
used_memory_human:1.00G         # 可读格式
used_memory_rss:1258291200      # 操作系统视角的 Redis 进程占用内存（含碎片）
used_memory_rss_human:1.17G
used_memory_peak:2147483648     # 内存使用峰值
used_memory_peak_human:2.00G
mem_fragmentation_ratio:1.17    # 内存碎片率 = used_memory_rss / used_memory
mem_allocator:jemalloc-5.1.0    # 内存分配器
```

**关键指标解读**：

| 指标 | 含义 | 关注点 |
|------|------|--------|
| `used_memory` | 数据实际占用内存 | 监控是否接近 `maxmemory` |
| `used_memory_rss` | OS 分配给 Redis 的物理内存 | 包含内存碎片 |
| `mem_fragmentation_ratio` | 内存碎片率 | > 1.5 需要关注，< 1 说明使用了 swap |
| `used_memory_peak` | 历史内存峰值 | 评估是否需要扩容 |

### 1.2 内存占用组成

```
  Redis 内存总占用
  ┌──────────────────────────────────────────────────┐
  │                                                  │
  │  ┌──────────────┐  数据内存（最大部分）             │
  │  │ Key-Value    │  所有键值对、过期字典等            │
  │  │ 数据存储      │                                 │
  │  └──────────────┘                                 │
  │                                                  │
  │  ┌──────────────┐  进程内存                        │
  │  │ Redis 自身    │  代码、栈、共享库等               │
  │  │ 运行开销      │  通常几十 MB                     │
  │  └──────────────┘                                 │
  │                                                  │
  │  ┌──────────────┐  缓冲内存                        │
  │  │ 客户端缓冲区  │  输入/输出缓冲区                  │
  │  │ 复制积压缓冲区│  主从复制使用                     │
  │  │ AOF 缓冲区   │  AOF 重写期间使用                 │
  │  └──────────────┘                                 │
  │                                                  │
  │  ┌──────────────┐  内存碎片                        │
  │  │ 碎片内存      │  分配器未使用的内存空间            │
  │  │              │  频繁修改/删除后产生               │
  │  └──────────────┘                                 │
  └──────────────────────────────────────────────────┘
```

### 1.3 内存碎片率分析

| `mem_fragmentation_ratio` 范围 | 含义 | 处理 |
|-------------------------------|------|------|
| **1.0 ~ 1.5** | 正常范围 | 无需处理 |
| **> 1.5** | 碎片较多 | 需要碎片整理 |
| **> 2.0** | 碎片严重 | 需紧急处理 |
| **< 1.0** | 使用了 swap | 性能严重下降，需扩容 |

---

## 二、八种内存淘汰策略

### 2.1 maxmemory 配置

```conf
# redis.conf
maxmemory 4gb                    # 设置最大内存限制
maxmemory-policy allkeys-lru     # 内存淘汰策略
```

当 Redis 使用内存达到 `maxmemory` 时，触发淘汰策略。

### 2.2 八种淘汰策略详解（核心考点 ⭐⭐⭐）

```
               ┌────────────────────────────────────┐
               │          内存淘汰策略                │
               └────────────┬───────────────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
     不淘汰            所有 key             设置了过期时间的 key
   ┌────────┐     ┌──────────────┐     ┌──────────────┐
   │noeviction│    │ allkeys-lru  │     │ volatile-lru │
   └────────┘    │ allkeys-lfu  │     │ volatile-lfu │
                 │ allkeys-random│    │volatile-random│
                 └──────────────┘     │ volatile-ttl  │
                                      └──────────────┘
```

| 策略 | 淘汰范围 | 淘汰规则 | 说明 |
|------|---------|---------|------|
| **noeviction** | 不淘汰 | 写入报错 OOM | 默认策略，只读不写 |
| **allkeys-lru** | 所有 key | 淘汰最近最少使用 | **最常用**，适合大部分场景 |
| **volatile-lru** | 有过期时间的 key | 淘汰最近最少使用 | 保护无过期 key |
| **allkeys-random** | 所有 key | 随机淘汰 | 各 key 访问频率相近时使用 |
| **volatile-random** | 有过期时间的 key | 随机淘汰 | 不推荐 |
| **volatile-ttl** | 有过期时间的 key | 淘汰 TTL 最短的 | 优先删除即将过期的 |
| **allkeys-lfu** | 所有 key | 淘汰最不经常使用 | Redis 4.0+，根据频率淘汰 |
| **volatile-lfu** | 有过期时间的 key | 淘汰最不经常使用 | Redis 4.0+ |

### 2.3 LRU vs LFU 算法原理（核心考点 ⭐⭐⭐）

#### LRU（Least Recently Used，最近最少使用）

淘汰**最近最长时间没有被访问**的数据。

```
  访问序列：A B C D E （内存只能存 4 个）

  [A]                    ← 访问 A
  [B, A]                 ← 访问 B
  [C, B, A]              ← 访问 C
  [D, C, B, A]           ← 访问 D（满了）
  [E, D, C, B] → 淘汰 A  ← 访问 E，A 最久未访问
```

**问题**：偶尔一次访问就能续命，无法区分"频繁访问"和"偶尔访问"。

#### LFU（Least Frequently Used，最不经常使用）

淘汰**访问频率最低**的数据。

```
  Key    访问次数    最近访问时间
  ─────────────────────────────
  A       100        1分钟前     ← 高频热点
  B       2          10秒前      ← 低频，偶尔访问
  C       50         30秒前      ← 中频

  内存不足时，淘汰 B（访问次数最少）
```

**LFU 优势**：能区分真正的热点数据和偶尔被访问的数据。

#### LRU vs LFU 对比

| 对比项 | LRU | LFU |
|--------|-----|-----|
| **判断依据** | 最近访问时间 | 访问频率 |
| **时间局部性** | 强 | 弱 |
| **频率敏感性** | 弱 | 强 |
| **问题** | 偶尔访问的冷数据也能续命 | 历史高频但已不再访问的 key 难以淘汰 |
| **适用场景** | 最近访问的数据最可能再次访问 | 访问模式稳定，热点明显 |
| **Redis 版本** | 全版本支持 | 4.0+ |

### 2.4 Redis 近似 LRU 实现（核心考点 ⭐⭐）

Redis 并没有实现严格的 LRU 算法（需要维护链表开销太大），而是采用 **随机采样近似 LRU**。

```
  标准 LRU：维护所有 key 的访问链表 → 内存开销大

  Redis 近似 LRU：
  1. 随机采样 N 个 key（默认 N=5，由 maxmemory-samples 控制）
  2. 从采样的 key 中淘汰 idle time 最大的
  3. 重复直到释放足够内存

  ┌────────────────────────────────────┐
  │  所有 key（如 100 万个）             │
  │                                    │
  │  随机采样 5 个：                     │
  │  [K1: 10s, K2: 300s, K3: 5s,      │
  │   K4: 600s, K5: 50s]              │
  │                                    │
  │  淘汰 K4（idle time 最大 = 600s）   │
  └────────────────────────────────────┘
```

```conf
# 采样数量，值越大越接近真实 LRU，但 CPU 消耗越高
maxmemory-samples 5    # 默认值，推荐 5~10
```

### 2.5 策略选型建议

| 场景 | 推荐策略 |
|------|---------|
| **通用缓存** | `allkeys-lru`（最常用） |
| **热点明显的缓存** | `allkeys-lfu`（区分冷热数据） |
| **有持久化 key + 缓存 key 混用** | `volatile-lru`（只淘汰有过期时间的） |
| **数据不允许丢失** | `noeviction`（写入报错，由应用决定处理） |

---

## 三、Redis 大 Key 问题

### 3.1 什么是大 Key（核心考点 ⭐⭐⭐）

| 数据类型 | 大 Key 判定标准 |
|---------|----------------|
| **String** | value 大小 > **10 KB** |
| **Hash** | 元素数量 > **5000** 或 value 总大小 > **10 MB** |
| **List** | 元素数量 > **5000** |
| **Set** | 元素数量 > **5000** |
| **ZSet** | 元素数量 > **5000** |

> 上述阈值因业务而异，阿里云 Redis 规范一般以 String > 10KB、集合类型 > 5000 元素为参考。

### 3.2 大 Key 的危害

```
                    大 Key 的危害
  ┌──────────────────────────────────────────────┐
  │                                              │
  │  1. 内存不均  →  Cluster 模式下某个分片内存     │
  │                 远高于其他分片                  │
  │                                              │
  │  2. 阻塞操作  →  DEL 大 Key 时会阻塞主线程     │
  │                 （同步删除耗时可达秒级）          │
  │                                              │
  │  3. 网络拥塞  →  读取大 Key 占用大量带宽         │
  │                 影响其他请求                    │
  │                                              │
  │  4. 主从同步延迟  →  大 Key 序列化/传输耗时      │
  │                    导致复制积压                 │
  │                                              │
  │  5. 过期删除慢  →  大 Key 过期时触发同步删除     │
  │                  造成短暂阻塞                   │
  └──────────────────────────────────────────────┘
```

### 3.3 大 Key 发现方法

#### 方法一：redis-cli --bigkeys

```bash
# 扫描所有 key，统计每种类型中最大的 key
redis-cli --bigkeys

# 输出示例：
# -------- summary -------
# Biggest string found 'user:avatar:10001' has 1048576 bytes (1MB)
# Biggest hash found 'product:detail:2001' has 12000 fields
# Biggest list found 'order:queue' has 50000 items
```

**优点**：使用 SCAN 扫描，不会阻塞  
**缺点**：只能找每种类型最大的 1 个 key

#### 方法二：MEMORY USAGE

```bash
# 查看单个 key 的内存占用（Redis 4.0+）
127.0.0.1:6379> MEMORY USAGE user:profile:10001
(integer) 10240   # 10 KB
```

#### 方法三：RDB 分析工具

```bash
# 使用 rdb-tools 离线分析 RDB 文件
pip install rdbtools
rdb --command memory dump.rdb --bytes 10240 -f bigkeys.csv
# 导出所有大于 10KB 的 key
```

#### 方法四：SCAN + 批量检测

```java
// 使用 SCAN 遍历 + MEMORY USAGE 检测
public List<String> findBigKeys(StringRedisTemplate redisTemplate, long threshold) {
    List<String> bigKeys = new ArrayList<>();
    ScanOptions options = ScanOptions.scanOptions().count(100).build();

    try (Cursor<byte[]> cursor = redisTemplate.getConnectionFactory()
            .getConnection().scan(options)) {
        while (cursor.hasNext()) {
            String key = new String(cursor.next());
            Long memUsage = redisTemplate.execute((RedisCallback<Long>) conn ->
                conn.memoryUsage(key.getBytes()));
            if (memUsage != null && memUsage > threshold) {
                bigKeys.add(key + " -> " + memUsage + " bytes");
            }
        }
    }
    return bigKeys;
}
```

### 3.4 大 Key 删除方案

#### 方案一：UNLINK 异步删除（推荐）

```bash
# UNLINK 命令（Redis 4.0+）：异步删除，不阻塞主线程
127.0.0.1:6379> UNLINK bigkey:user:profile
(integer) 1
```

`UNLINK` 与 `DEL` 的区别：

| 命令 | 执行方式 | 阻塞主线程 | 适用场景 |
|------|---------|-----------|---------|
| `DEL` | 同步删除 | 是（大 key 可能阻塞秒级） | 小 key |
| `UNLINK` | 异步删除（后台线程） | 否 | 大 key |

#### 方案二：分批删除

```java
// Hash 大 Key 分批删除
public void deleteBigHash(String key) {
    ScanOptions options = ScanOptions.scanOptions().count(100).build();
    try (Cursor<Map.Entry<Object, Object>> cursor =
            redisTemplate.opsForHash().scan(key, options)) {
        List<Object> fields = new ArrayList<>();
        while (cursor.hasNext()) {
            fields.add(cursor.next().getKey());
            if (fields.size() >= 100) {
                redisTemplate.opsForHash().delete(key, fields.toArray());
                fields.clear();
            }
        }
        if (!fields.isEmpty()) {
            redisTemplate.opsForHash().delete(key, fields.toArray());
        }
    }
    redisTemplate.delete(key); // 最后删除 key 本身
}

// List 大 Key 分批删除
public void deleteBigList(String key) {
    long length = redisTemplate.opsForList().size(key);
    int batchSize = 100;
    for (long i = 0; i < length; i += batchSize) {
        redisTemplate.opsForList().trim(key, batchSize, -1);
    }
    redisTemplate.delete(key);
}
```

#### 方案三：SCAN 渐进遍历删除

适用于 Set 和 ZSet：

```bash
# Set 分批删除
SSCAN big:set:key 0 COUNT 100
# 取出结果后 SREM 删除
SREM big:set:key member1 member2 ...

# ZSet 分批删除
ZREMRANGEBYRANK big:zset:key 0 99
# 每次删 100 个，直到为空
```

### 3.5 大 Key 预防方案

| 方案 | 说明 |
|------|------|
| **数据拆分** | 将大 Hash 拆分为多个小 Hash（如按 ID 分段） |
| **数据压缩** | 对 value 进行 GZIP/Snappy 压缩后再存储 |
| **序列化优化** | 使用 Protobuf/MessagePack 代替 JSON，减小体积 |
| **合理设计数据结构** | 避免将所有信息都塞进一个 key |
| **监控报警** | 定期扫描大 key，超过阈值报警 |

```java
// 大 Hash 拆分示例：将 user:profile 按字段组拆分
// ✗ 一个大 Hash 存所有信息
HSET user:10001 name "张三" age 25 avatar "<1MB 图片Base64>" ...

// ✓ 拆分为多个小 Hash
HSET user:basic:10001 name "张三" age 25 email "test@example.com"
HSET user:avatar:10001 data "<压缩后的图片>"
HSET user:settings:10001 theme "dark" lang "zh"
```

---

## 四、热 Key 问题

### 4.1 什么是热 Key

**热 Key** 是指在短时间内被**极高频率访问**的 Key，导致请求集中在某一个 Redis 实例上，出现性能瓶颈。

```
  正常分布：                          热 Key 分布：
  ┌────────┐ ┌────────┐ ┌────────┐    ┌────────┐ ┌────────┐ ┌────────┐
  │ Node-1 │ │ Node-2 │ │ Node-3 │    │ Node-1 │ │ Node-2 │ │ Node-3 │
  │  33%   │ │  33%   │ │  33%   │    │  10%   │ │  80%   │ │  10%   │
  │ 请求    │ │ 请求    │ │ 请求    │    │ 请求    │ │热Key!   │ │ 请求    │
  └────────┘ └────────┘ └────────┘    └────────┘ └────────┘ └────────┘
```

**常见场景**：
- 突发热点新闻/微博热搜
- 秒杀商品详情页
- 直播间礼物排行榜

### 4.2 发现热 Key

| 方法 | 说明 | 优缺点 |
|------|------|--------|
| `redis-cli --hotkeys` | Redis 4.0+ 使用 LFU 策略时可用 | 简单但需要 LFU 淘汰策略 |
| `MONITOR` 命令 | 实时监控所有命令 | 性能开销大，**生产慎用** |
| 客户端统计 | 在 Redis 客户端中统计 key 访问次数 | 侵入代码，但精准 |
| 代理层统计 | 在 Proxy（如 Twemproxy/Codis）中统计 | 无侵入，需要代理层支持 |
| 大数据分析 | 将访问日志上报到 Flink/Spark 分析 | 延迟高，适合离线分析 |

```bash
# 使用 --hotkeys 发现热 key（需要 maxmemory-policy 包含 lfu）
redis-cli --hotkeys

# 输出示例：
# hot key found with counter: 2560000  keyname: product:detail:8888
# hot key found with counter: 1280000  keyname: ranking:live:room:100
```

### 4.3 热 Key 解决方案

#### 方案一：本地缓存（最常用）

```java
// 使用 Caffeine 作为本地缓存
private final Cache<String, Object> localCache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(10, TimeUnit.SECONDS) // 短 TTL，减少不一致
    .build();

public Object getHotData(String key) {
    // L1 本地缓存
    Object value = localCache.getIfPresent(key);
    if (value != null) return value;

    // L2 Redis
    value = redisTemplate.opsForValue().get(key);
    if (value != null) {
        localCache.put(key, value); // 回填本地缓存
    }
    return value;
}
```

#### 方案二：读写分离

增加从节点，将热 Key 的读请求分散到多个从节点：

```
         ┌─────────┐
         │ Master  │ ← 写
         └────┬────┘
      ┌───────┼───────┐
      ▼       ▼       ▼
  ┌───────┐┌───────┐┌───────┐
  │ Slave1││ Slave2││ Slave3│ ← 读（分散热 Key 压力）
  └───────┘└───────┘└───────┘
```

#### 方案三：Key 分散（打散热 Key）

将一个热 Key 复制到多个 Key 上，读取时随机选择。

```java
// 将热 key 分散为多个副本
private static final int SHARD_COUNT = 10;

public void setHotKey(String key, Object value) {
    for (int i = 0; i < SHARD_COUNT; i++) {
        redisTemplate.opsForValue().set(key + ":" + i, value, 30, TimeUnit.MINUTES);
    }
}

public Object getHotKey(String key) {
    // 随机选择一个分片
    int shard = ThreadLocalRandom.current().nextInt(SHARD_COUNT);
    return redisTemplate.opsForValue().get(key + ":" + shard);
}
```

### 4.4 方案对比

| 方案 | 效果 | 复杂度 | 一致性 |
|------|------|--------|--------|
| 本地缓存 | 最好（完全避免 Redis 访问） | 低 | 弱（各实例缓存可能不同） |
| 读写分离 | 好（分散到多节点） | 中 | 强（主从同步） |
| Key 分散 | 好（分散到多分片） | 中 | 需自行维护一致性 |

---

## 五、Redis 内存优化技巧

### 5.1 合理选择数据结构

```java
// ✗ 用 String 存储对象的每个字段（N 个 key，开销大）
SET user:10001:name "张三"
SET user:10001:age "25"
SET user:10001:email "test@example.com"
// 每个 key 有额外的元数据开销（dictEntry、redisObject 等约 80+ 字节）

// ✓ 用 Hash 存储（1 个 key，内存更紧凑）
HSET user:10001 name "张三" age "25" email "test@example.com"
// 小数据量时使用 ziplist 编码，内存极省
```

### 5.2 使用 ziplist 编码优化（核心考点 ⭐⭐）

当数据量较小时，Redis 的 Hash、List、Set、ZSet 会使用 **ziplist（压缩列表）** 编码，内存效率极高。

```
  hashtable 编码：                    ziplist 编码：
  ┌──────┐  ┌──────┐  ┌──────┐     ┌──┬──┬──┬──┬──┬──┬──┬──┐
  │entry │→│entry │→│entry │     │  连续内存块，无指针开销   │
  │ +ptr │  │ +ptr │  │ +ptr │     └──┴──┴──┴──┴──┴──┴──┴──┘
  └──────┘  └──────┘  └──────┘
  每个 entry 有指针开销              紧凑存储，零额外开销
```

**触发 ziplist 编码的配置**：

```conf
# Hash：field 数 ≤ 128 且 value 长度 ≤ 64 字节时使用 ziplist
hash-max-ziplist-entries 128
hash-max-ziplist-value 64

# List：Redis 3.2+ 使用 quicklist（ziplist 的链表）
list-max-ziplist-size -2    # -2 表示每个 ziplist 节点最大 8KB

# ZSet：元素数 ≤ 128 且元素长度 ≤ 64 字节时使用 ziplist
zset-max-ziplist-entries 128
zset-max-ziplist-value 64

# Set：全为整数且数量 ≤ 512 时使用 intset
set-max-intset-entries 512
```

### 5.3 键值精简化

| 优化项 | 优化前 | 优化后 | 效果 |
|--------|--------|--------|------|
| Key 命名 | `user:profile:detail:10001` | `u:p:10001` | key 变短，节省内存 |
| Value 格式 | JSON `{"name":"张三","age":25}` | Protobuf 二进制 | 体积减少 50%+ |
| 字段命名 | `"username"` | `"un"` | Hash field 缩短 |
| 数值存储 | `"10001"` 字符串 | 整数编码 | 自动使用 int 编码 |

```java
// 使用 Protobuf 序列化（比 JSON 体积小 50%~80%）
byte[] data = user.toByteArray(); // Protobuf 序列化
redisTemplate.opsForValue().set(key, data);
```

### 5.4 合理设置过期时间

```java
// ✗ 所有 key 不设置过期时间（内存只增不减）
redisTemplate.opsForValue().set(key, value);

// ✓ 根据业务设置合理的 TTL
redisTemplate.opsForValue().set(key, value, 30, TimeUnit.MINUTES);

// ✓ 不同业务不同 TTL
// 用户 session：30 分钟
// 商品详情缓存：2 小时
// 热搜榜单：5 分钟
// 验证码：5 分钟
```

### 5.5 内存碎片整理

```bash
# Redis 4.0+ 手动触发碎片整理
127.0.0.1:6379> MEMORY PURGE

# Redis 4.0+ 开启自动碎片整理（activedefrag）
config set activedefrag yes

# 碎片整理相关配置
active-defrag-enabled yes           # 开启自动碎片整理
active-defrag-ignore-bytes 100mb    # 碎片大小超过 100MB 才整理
active-defrag-threshold-lower 10    # 碎片率超过 10% 才整理
active-defrag-threshold-upper 100   # 碎片率超过 100% 时全力整理
active-defrag-cycle-min 1           # 整理最小占用 CPU 百分比
active-defrag-cycle-max 25          # 整理最大占用 CPU 百分比
```

**注意**：碎片整理会消耗 CPU 资源，建议在低峰期开启或使用自动整理并控制 CPU 上限。

---

## 六、大厂常见面试题

### Q1：Redis 的内存淘汰策略有哪些？生产环境怎么选？

**答：**

Redis 提供 8 种淘汰策略，分为三类：

1. **不淘汰**：`noeviction`（默认），内存满时拒绝写入
2. **所有 key 范围**：`allkeys-lru`、`allkeys-lfu`、`allkeys-random`
3. **有过期时间的 key 范围**：`volatile-lru`、`volatile-lfu`、`volatile-random`、`volatile-ttl`

**生产选型**：
- **通用缓存场景**：`allkeys-lru`（最常用）
- **热点明显**：`allkeys-lfu`（Redis 4.0+，能区分冷热数据）
- **缓存和持久化 key 混用**：`volatile-lru`（保护没有过期时间的持久化 key）

---

### Q2：Redis 的 LRU 是严格的 LRU 吗？怎么实现的？

**答：**

不是严格 LRU，是**近似 LRU**。

严格 LRU 需要维护全局的访问链表，每次访问都要移动节点，内存和性能开销大。

Redis 的近似 LRU：
1. 每个 key 的 redisObject 中有一个 24 位的 `lru` 字段，记录最近访问的时间戳
2. 淘汰时随机采样 N 个 key（`maxmemory-samples`，默认 5）
3. 从采样的 key 中淘汰 `lru` 值最小（idle time 最大）的
4. 采样数越大越接近真实 LRU，但 CPU 消耗越高

Redis 官方测试，`maxmemory-samples=10` 时效果已非常接近真实 LRU。

---

### Q3：什么是大 Key？大 Key 有什么危害？怎么处理？

**答：**

**大 Key 判定**：String > 10KB，Hash/List/Set/ZSet > 5000 元素。

**危害**：
1. 阻塞主线程（DEL 大 key 可能阻塞秒级）
2. 网络带宽占用大
3. Cluster 模式下内存分布不均
4. 主从同步延迟

**发现**：`redis-cli --bigkeys`、`MEMORY USAGE`、RDB 分析工具

**删除**：
- Redis 4.0+：使用 `UNLINK` 异步删除
- 分批删除：使用 SCAN + 分批 SREM/HDEL/ZREMRANGEBYRANK
- 避免直接 `DEL`

**预防**：数据拆分、压缩、序列化优化、监控报警。

---

### Q4：热 Key 问题怎么发现和解决？

**答：**

**发现**：
- `redis-cli --hotkeys`（需要 LFU 策略）
- 客户端 SDK 统计
- 代理层统计（Twemproxy/Codis）

**解决**：
1. **本地缓存**（最有效）：使用 Caffeine/Guava Cache 作为 L1 缓存，短 TTL
2. **读写分离**：增加从节点分散读压力
3. **Key 分散**：将热 Key 复制到多个副本（如 `key:0` ~ `key:9`），随机读取

**核心思路**：减少单节点的访问压力，将流量分散到多个节点或本地。

---

### Q5：如何对 Redis 进行内存优化？

**答：**

1. **合理选择数据结构**：用 Hash 代替多个 String，利用 ziplist 编码
2. **利用小数据编码**：控制 Hash/ZSet/Set 的元素数量和大小在 ziplist 阈值内
3. **键值精简化**：缩短 Key 名、使用 Protobuf 等高效序列化
4. **设置合理 TTL**：避免 key 只增不删
5. **碎片整理**：开启 `activedefrag` 或定期执行 `MEMORY PURGE`
6. **避免大 Key**：数据拆分、压缩
7. **合理设置 maxmemory**：留 20%~30% 余量给碎片和缓冲

---

### Q6：`mem_fragmentation_ratio` 很高或很低分别代表什么？

**答：**

`mem_fragmentation_ratio = used_memory_rss / used_memory`

- **> 1.0 且 < 1.5**：正常，有少量碎片
- **> 1.5**：碎片较多，可能是频繁修改/删除导致。需要碎片整理（`MEMORY PURGE` 或 `activedefrag`）
- **> 2.0**：碎片严重，紧急处理
- **< 1.0**：说明 Redis 使用了 **swap**（虚拟内存），性能会严重下降。原因是物理内存不足，需要扩容或减少数据量

生产环境应监控此指标并设置告警阈值。

---

### Q7：LRU 和 LFU 有什么区别？各自适合什么场景？

**答：**

| 维度 | LRU | LFU |
|------|-----|-----|
| 依据 | 最近访问时间 | 访问频率 |
| 缺陷 | 偶尔被访问的冷数据也能续命 | 历史高频数据不再访问也难淘汰 |
| 适用场景 | 访问模式有时间局部性 | 热点稳定，需区分冷热 |

**选型建议**：
- 大部分场景用 `allkeys-lru`，简单有效
- 如果业务热点明确（如商品缓存有明显冷热差异），用 `allkeys-lfu`
- LFU 在 Redis 4.0+ 可用，通过 `lfu-log-factor` 和 `lfu-decay-time` 调节衰减速度
