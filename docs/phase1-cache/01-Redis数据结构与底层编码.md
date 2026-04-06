# 01 - Redis 数据结构与底层编码

> 本章深入讲解 Redis 的核心数据结构及其底层编码实现，覆盖 **五大基础类型**、**特殊数据类型**、**底层编码转换** 与 **多线程模型**，是大厂面试高频考点。

## 目录

1. [Redis 概述](#一redis-概述)
2. [String 类型与 SDS](#二string-类型与-sds)
3. [Hash 类型](#三hash-类型)
4. [List 类型](#四list-类型)
5. [Set 类型](#五set-类型)
6. [ZSet 有序集合](#六zset-有序集合)
7. [特殊数据类型](#七特殊数据类型)
8. [Redis 6.0 多线程模型](#八redis-60-多线程模型)
9. [底层编码转换条件总结](#九底层编码转换条件总结)
10. [大厂常见面试题](#十大厂常见面试题)

---

## 一、Redis 概述

### 1.1 Redis 是什么

Redis（Remote Dictionary Server）是一个开源的、基于内存的 **高性能 Key-Value 数据库**，支持多种数据结构，广泛用于缓存、消息队列、分布式锁等场景。

### 1.2 Redis 为什么快（核心考点 ⭐⭐⭐）

| 原因 | 说明 |
|------|------|
| **纯内存操作** | 所有数据存储在内存中，读写速度为纳秒级别 |
| **单线程模型** | 避免了线程切换和锁竞争的开销 |
| **IO 多路复用** | 基于 epoll/kqueue 实现，单线程处理大量并发连接 |
| **高效数据结构** | SDS、跳表、压缩列表等专为性能优化的结构 |
| **渐进式 Rehash** | Hash 扩容时不会一次性搬迁，避免阻塞 |

### 1.3 单线程模型

Redis 的 **命令执行** 始终是单线程的，这避免了：

- 多线程竞争和锁开销
- 上下文切换成本
- 死锁问题

```
                     ┌──────────────────────┐
                     │    Redis 主线程       │
                     │                      │
   Client A ────────►│  ┌──────────────┐   │
   Client B ────────►│  │ IO 多路复用器 │   │
   Client C ────────►│  │  (epoll)     │   │
                     │  └──────┬───────┘   │
                     │         │           │
                     │         ▼           │
                     │  ┌──────────────┐   │
                     │  │  事件分派器   │   │
                     │  └──────┬───────┘   │
                     │         │           │
                     │    ┌────┴────┐      │
                     │    ▼         ▼      │
                     │ 读事件    写事件     │
                     │ 处理器    处理器     │
                     └──────────────────────┘
```

### 1.4 IO 多路复用

IO 多路复用允许一个线程同时监听多个 Socket 连接，核心机制：

```
1. 将所有 Client Socket 注册到 epoll 实例
2. 调用 epoll_wait() 阻塞等待事件就绪
3. 事件就绪后，依次处理每个 Socket 的读/写操作
4. 全程无需为每个连接创建独立线程
```

```java
// Java 中类似的 NIO 多路复用
Selector selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select(); // 类似 epoll_wait
    Set<SelectionKey> keys = selector.selectedKeys();
    for (SelectionKey key : keys) {
        if (key.isAcceptable()) {
            // 处理新连接
        } else if (key.isReadable()) {
            // 处理读事件
        }
    }
}
```

---

## 二、String 类型与 SDS

### 2.1 SDS（Simple Dynamic String）结构（核心考点 ⭐⭐⭐）

Redis 没有直接使用 C 语言的原生字符串（`char[]`），而是自己构建了 **SDS（Simple Dynamic String）** 结构。

```c
struct sdshdr {
    int len;      // 已使用的长度
    int free;     // 剩余可用空间
    char buf[];   // 实际存储的字符数组（兼容 C 字符串，以 '\0' 结尾）
};
```

```
+------+------+---+---+---+---+---+---+----+
| len  | free | R | e | d | i | s | \0|    |
|  5   |  3   |   |   |   |   |   |   |    |
+------+------+---+---+---+---+---+---+----+
                ◄── buf (已用5 + 空闲3) ──►
```

### 2.2 SDS 相比 C 字符串的优势

| 特性 | C 字符串 | SDS |
|------|---------|-----|
| 获取长度 | O(n) 遍历 | O(1) 直接读 `len` |
| 缓冲区溢出 | 不检查，可能溢出 | 自动扩容，安全 |
| 内存分配 | 每次修改都需要分配 | **空间预分配 + 惰性释放** |
| 二进制安全 | 以 `\0` 判断结尾，不能存二进制 | 以 `len` 判断长度，可存任意二进制数据 |
| 兼容 C 函数 | 原生兼容 | 兼容部分 C 字符串函数 |

### 2.3 空间预分配策略

当 SDS 需要扩容时：

- 如果修改后 `len < 1MB`：分配与 `len` 等大的 `free` 空间（即总空间 = `2 * len + 1`）
- 如果修改后 `len >= 1MB`：分配 `1MB` 的 `free` 空间（即总空间 = `len + 1MB + 1`）

### 2.4 惰性空间释放

当 SDS 缩短时，不会立即释放多余空间，而是记录到 `free` 字段，等待下次使用，减少内存重分配次数。

### 2.5 String 的三种编码

| 编码 | 条件 | 说明 |
|------|------|------|
| `int` | 值是整数且可以用 `long` 表示 | 直接在指针位置存数值 |
| `embstr` | 字符串长度 ≤ 44 字节 | SDS 和 redisObject 一次分配，连续内存 |
| `raw` | 字符串长度 > 44 字节 | SDS 和 redisObject 分别分配 |

```bash
# Redis 命令演示
127.0.0.1:6379> SET count 100
OK
127.0.0.1:6379> OBJECT ENCODING count
"int"

127.0.0.1:6379> SET name "hello"
OK
127.0.0.1:6379> OBJECT ENCODING name
"embstr"

127.0.0.1:6379> SET longstr "this is a very long string that exceeds forty-four bytes limit"
OK
127.0.0.1:6379> OBJECT ENCODING longstr
"raw"
```

### 2.6 Java RedisTemplate 操作 String

```java
@Autowired
private StringRedisTemplate redisTemplate;

// 基本操作
redisTemplate.opsForValue().set("user:1:name", "张三");
redisTemplate.opsForValue().set("user:1:name", "张三", 30, TimeUnit.MINUTES);

String name = redisTemplate.opsForValue().get("user:1:name");

// 原子递增
redisTemplate.opsForValue().increment("article:1001:views"); // +1
redisTemplate.opsForValue().increment("stock:sku:1001", -1);  // -1（扣库存）

// SETNX（分布式锁基础）
Boolean success = redisTemplate.opsForValue()
    .setIfAbsent("lock:order:1001", "uuid-xxx", 10, TimeUnit.SECONDS);
```

---

## 三、Hash 类型

### 3.1 Hash 底层编码

Hash 类型有两种底层编码：**ziplist（压缩列表）** 和 **hashtable（哈希表）**。

```
                Hash 对象
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
    ziplist                hashtable
  (数据量小时)           (数据量大时)
```

### 3.2 ziplist（压缩列表）

ziplist 是一块 **连续内存**，将所有元素紧凑存储：

```
┌────────┬────────┬───────┬───────┬───────┬───────┬────────┐
│ zlbytes│ zltail │ zllen │ entry │ entry │ entry │ zlend  │
│ (4B)   │ (4B)   │ (2B)  │  key  │ value │ ...   │ (1B)   │
└────────┴────────┴───────┴───────┴───────┴───────┴────────┘
```

**优点**：内存紧凑、缓存友好（连续内存，CPU Cache Line 命中率高）  
**缺点**：查找复杂度 O(n)、连锁更新问题

### 3.3 hashtable（哈希表）

Redis 使用的哈希表支持 **渐进式 Rehash**：

```
dictht[0]（正在使用）         dictht[1]（Rehash 目标）
┌────┬─────────┐              ┌────┬─────────┐
│ 0  │ → entry │              │ 0  │         │
│ 1  │ → entry │   ─Rehash─►  │ 1  │ → entry │
│ 2  │         │              │ 2  │ → entry │
│ 3  │ → entry │              │ 3  │         │
└────┴─────────┘              │ 4  │ → entry │
                              │ 5  │         │
                              │ 6  │         │
                              │ 7  │ → entry │
                              └────┴─────────┘
```

**渐进式 Rehash 过程**：
1. 分配 `dictht[1]` 空间（通常为 `dictht[0]` 的 2 倍）
2. 维护一个 `rehashidx` 计数器，初始为 0
3. 每次 CRUD 操作时，顺带将 `dictht[0]` 的 `rehashidx` 索引上的所有键值对迁移到 `dictht[1]`
4. 迁移完成后，释放 `dictht[0]`，将 `dictht[1]` 设为 `dictht[0]`

### 3.4 编码转换条件

**ziplist → hashtable** 触发条件（满足任一）：

| 条件 | 默认阈值 | 配置项 |
|------|---------|--------|
| 单个 field 或 value 长度 > 阈值 | 64 字节 | `hash-max-ziplist-value` |
| field 数量 > 阈值 | 128 个 | `hash-max-ziplist-entries` |

### 3.5 Java RedisTemplate 操作 Hash

```java
// Hash 操作
HashOperations<String, String, String> hashOps = redisTemplate.opsForHash();

// 设置单个字段
hashOps.put("user:1001", "name", "张三");
hashOps.put("user:1001", "age", "25");

// 批量设置
Map<String, String> userMap = new HashMap<>();
userMap.put("name", "李四");
userMap.put("age", "30");
userMap.put("email", "lisi@example.com");
hashOps.putAll("user:1002", userMap);

// 获取单个字段
String name = hashOps.get("user:1001", "name");

// 获取所有字段
Map<String, String> allFields = hashOps.entries("user:1001");

// 字段递增
hashOps.increment("user:1001", "loginCount", 1);
```

---

## 四、List 类型

### 4.1 quicklist（Redis 3.2+）

Redis 3.2 之后，List 底层统一使用 **quicklist**，它结合了 **ziplist 和双向链表** 的优点：

```
quicklist（双向链表 + ziplist）
┌──────────┐    ┌──────────┐    ┌──────────┐
│ quicklist│    │ quicklist│    │ quicklist│
│  Node 1  │◄──►│  Node 2  │◄──►│  Node 3  │
│          │    │          │    │          │
│ ┌──────┐ │    │ ┌──────┐ │    │ ┌──────┐ │
│ │ziplist│ │    │ │ziplist│ │    │ │ziplist│ │
│ │entry1│ │    │ │entry4│ │    │ │entry7│ │
│ │entry2│ │    │ │entry5│ │    │ │entry8│ │
│ │entry3│ │    │ │entry6│ │    │ │entry9│ │
│ └──────┘ │    │ └──────┘ │    │ └──────┘ │
└──────────┘    └──────────┘    └──────────┘
```

### 4.2 为什么选择 quicklist

| 数据结构 | 优点 | 缺点 |
|---------|------|------|
| 双向链表 | 插入删除 O(1) | 内存碎片、每个节点需额外指针空间 |
| ziplist | 内存紧凑 | 插入删除可能触发连锁更新 O(n) |
| **quicklist** | **兼顾两者优点** | 复杂度略高 |

### 4.3 quicklist 配置

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `list-max-ziplist-size` | 每个 ziplist 节点的最大容量（正数=条目数，负数=字节限制） | -2（8KB） |
| `list-compress-depth` | 两端不压缩的节点数（0=不压缩） | 0 |

### 4.4 Java RedisTemplate 操作 List

```java
ListOperations<String, String> listOps = redisTemplate.opsForList();

// 左推入（模拟栈：LIFO）
listOps.leftPush("task:queue", "task-001");
listOps.leftPush("task:queue", "task-002");

// 右推入（模拟队列：FIFO）
listOps.rightPush("msg:queue", "msg-001");

// 弹出
String task = listOps.leftPop("task:queue");

// 阻塞弹出（消息队列场景）
String msg = listOps.leftPop("msg:queue", 30, TimeUnit.SECONDS);

// 范围查询
List<String> tasks = listOps.range("task:queue", 0, -1);

// 获取长度
Long size = listOps.size("task:queue");
```

---

## 五、Set 类型

### 5.1 Set 底层编码

| 编码 | 条件 | 说明 |
|------|------|------|
| `intset` | 所有元素都是整数，且元素数量 ≤ 512 | 有序整数数组，内存紧凑 |
| `hashtable` | 不满足 intset 条件 | value 为 NULL 的哈希表 |

### 5.2 intset（整数集合）

```c
typedef struct intset {
    uint32_t encoding;  // 编码方式：int16、int32、int64
    uint32_t length;    // 元素数量
    int8_t contents[];  // 实际存储数组（按升序排列）
};
```

**intset 的升级机制**：当插入的新元素类型比当前编码更大时（如 int16 → int32），会触发整个集合的升级，但 **不支持降级**。

```
intset (int16 编码)
┌──────────┬────────┬─────┬─────┬─────┐
│ encoding │ length │  1  │  5  │  10 │
│  int16   │   3    │     │     │     │
└──────────┴────────┴─────┴─────┴─────┘

插入 65535（超过 int16 范围） → 触发升级

intset (int32 编码)
┌──────────┬────────┬─────┬─────┬─────┬───────┐
│ encoding │ length │  1  │  5  │  10 │ 65535 │
│  int32   │   4    │     │     │     │       │
└──────────┴────────┴─────┴─────┴─────┴───────┘
```

### 5.3 编码转换条件

**intset → hashtable** 触发条件（满足任一）：

| 条件 | 默认阈值 | 配置项 |
|------|---------|--------|
| 存在非整数元素 | — | — |
| 元素数量 > 阈值 | 512 个 | `set-max-intset-entries` |

### 5.4 Java RedisTemplate 操作 Set

```java
SetOperations<String, String> setOps = redisTemplate.opsForSet();

// 添加元素
setOps.add("user:1001:tags", "Java", "Redis", "Spring");

// 判断是否存在
Boolean isMember = setOps.isMember("user:1001:tags", "Java");

// 集合运算（推荐好友、共同关注）
Set<String> intersection = setOps.intersect("user:1001:follow", "user:1002:follow"); // 共同关注
Set<String> diff = setOps.difference("user:1001:follow", "user:1002:follow");       // 可能认识

// 随机抽取（抽奖场景）
String lucky = setOps.randomMember("activity:participants");
List<String> luckyList = setOps.randomMembers("activity:participants", 3);
```

---

## 六、ZSet 有序集合

### 6.1 ZSet 底层编码

ZSet（Sorted Set）同时需要 **按 score 排序** 和 **按 member 查找**，因此有两种编码：

| 编码 | 条件 |
|------|------|
| `ziplist` | 元素数量 ≤ 128 **且** 所有元素长度 ≤ 64 字节 |
| `skiplist + hashtable` | 不满足 ziplist 条件 |

### 6.2 跳表（SkipList）原理详解（核心考点 ⭐⭐⭐）

跳表是一种 **基于有序链表的多层索引结构**，查找时间复杂度为 **O(log n)**。

```
Level 4:  Head ──────────────────────────────────────► 99 ──► NULL
              │                                         │
Level 3:  Head ──────────────► 35 ──────────────────► 99 ──► NULL
              │                 │                       │
Level 2:  Head ────► 12 ────► 35 ────► 55 ──────────► 99 ──► NULL
              │       │        │        │               │
Level 1:  Head ► 5 ► 12 ► 22 ► 35 ► 42 ► 55 ► 78 ► 99 ──► NULL
```

**查找过程**（以查找 55 为例）：
1. 从最高层开始，Head → 99，99 > 55，下降到 Level 3
2. Level 3：Head → 35，35 < 55，前进到 35 → 99，99 > 55，下降
3. Level 2：35 → 55，找到目标！

### 6.3 为什么 Redis 用跳表而不用红黑树（高频面试题 ⭐⭐⭐）

| 对比项 | 跳表 | 红黑树 |
|--------|------|--------|
| 实现复杂度 | 简单，易于理解和调试 | 复杂，旋转操作繁琐 |
| 范围查找 | 直接沿链表遍历，O(log n + k) | 中序遍历，实现复杂 |
| 插入/删除 | 只需修改前后指针 | 可能触发多次旋转 |
| 内存占用 | 平均每个节点 1.33 个指针（p=0.25） | 每个节点固定 2 个指针 + 颜色位 |
| 并发友好 | 局部修改，易于加锁 | 旋转涉及多节点，加锁复杂 |

### 6.4 Redis 跳表实现细节

```c
// 跳表节点
typedef struct zskiplistNode {
    sds ele;                         // member 值
    double score;                    // 分数
    struct zskiplistNode *backward;  // 后退指针（Level 1 用于反向遍历）
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 前进指针
        unsigned long span;            // 跨度（用于计算 rank）
    } level[];                       // 柔性数组，每层一个
} zskiplistNode;

// 跳表结构
typedef struct zskiplist {
    struct zskiplistNode *header, *tail; // 头尾指针
    unsigned long length;                // 节点数量
    int level;                          // 当前最高层数
} zskiplist;
```

**层数生成策略**：每个节点的层数通过随机函数生成，每层提升的概率为 **p = 0.25**，最大层数为 **32**。

### 6.5 ZSet 的 hashtable

ZSet 在使用 skiplist 编码时，**同时**维护一个 hashtable（`dict`），用于：
- 跳表：按 score 排序、范围查找（ZRANGEBYSCORE）
- 字典：O(1) 按 member 查找 score（ZSCORE）

```
ZSet 对象
├── skiplist（按 score 排序）
│   Level 2: Head → [A:10] ──────────► [C:30] → NULL
│   Level 1: Head → [A:10] → [B:20] → [C:30] → NULL
│
└── hashtable（按 member 查找）
    ┌────────┬───────┐
    │ member │ score │
    ├────────┼───────┤
    │   A    │  10   │
    │   B    │  20   │
    │   C    │  30   │
    └────────┴───────┘
```

### 6.6 Java RedisTemplate 操作 ZSet

```java
ZSetOperations<String, String> zSetOps = redisTemplate.opsForZSet();

// 添加元素（排行榜场景）
zSetOps.add("rank:game:1001", "player-A", 1500);
zSetOps.add("rank:game:1001", "player-B", 2300);
zSetOps.add("rank:game:1001", "player-C", 1800);

// 增加分数
zSetOps.incrementScore("rank:game:1001", "player-A", 200);

// 获取排名（从0开始，分数从低到高）
Long rank = zSetOps.rank("rank:game:1001", "player-B");

// Top N 排行榜（分数从高到低）
Set<ZSetOperations.TypedTuple<String>> topPlayers =
    zSetOps.reverseRangeWithScores("rank:game:1001", 0, 9);

for (ZSetOperations.TypedTuple<String> tuple : topPlayers) {
    System.out.println(tuple.getValue() + " : " + tuple.getScore());
}

// 按分数范围查询
Set<String> players = zSetOps.rangeByScore("rank:game:1001", 1000, 2000);
```

---

## 七、特殊数据类型

### 7.1 HyperLogLog（基数统计）

HyperLogLog 用于 **大数据量下的去重计数**，误差率约 0.81%，固定占用 **12 KB** 内存。

| 场景 | 命令 |
|------|------|
| 添加元素 | `PFADD key element [element ...]` |
| 获取基数 | `PFCOUNT key [key ...]` |
| 合并 | `PFMERGE destkey sourcekey [sourcekey ...]` |

**典型场景**：统计网站 UV（独立访客数）

```java
HyperLogLogOperations<String, String> hllOps = redisTemplate.opsForHyperLogLog();

// 记录 UV
hllOps.add("uv:page:home:20240101", "user-001", "user-002", "user-003");
hllOps.add("uv:page:home:20240101", "user-001"); // 重复不计

// 获取 UV 数
Long uv = hllOps.size("uv:page:home:20240101"); // ≈ 3

// 合并多天数据
hllOps.union("uv:page:home:week01", "uv:page:home:20240101", "uv:page:home:20240102");
```

### 7.2 Bitmap（位图）

Bitmap 本质是一个 String，按 **bit 位** 操作，适用于状态标记类场景。

| 命令 | 说明 |
|------|------|
| `SETBIT key offset value` | 设置指定位 |
| `GETBIT key offset` | 获取指定位 |
| `BITCOUNT key [start end]` | 统计为 1 的位数 |
| `BITOP AND/OR/XOR dest key1 key2` | 位运算 |

**典型场景**：用户签到、布隆过滤器

```java
// 用户签到（一个月最多 4 字节）
redisTemplate.opsForValue().setBit("sign:user:1001:202401", 0, true);  // 1月1日签到
redisTemplate.opsForValue().setBit("sign:user:1001:202401", 4, true);  // 1月5日签到

// 查询某天是否签到
Boolean signed = redisTemplate.opsForValue().getBit("sign:user:1001:202401", 4);

// 统计签到天数（使用 Redis 命令）
// BITCOUNT sign:user:1001:202401
```

### 7.3 GeoSpatial（地理空间）

基于 ZSet 实现，score 存储经纬度的 GeoHash 编码。

| 命令 | 说明 |
|------|------|
| `GEOADD key lng lat member` | 添加地理位置 |
| `GEODIST key member1 member2 [unit]` | 计算距离 |
| `GEORADIUS key lng lat radius unit` | 搜索附近 |
| `GEOSEARCH key FROMMEMBER member BYRADIUS radius unit` | 6.2+ 搜索 |

```java
GeoOperations<String, String> geoOps = redisTemplate.opsForGeo();

// 添加门店位置
geoOps.add("store:locations",
    new Point(116.40, 39.90), "北京店",
    new Point(121.47, 31.23), "上海店",
    new Point(113.26, 23.13), "广州店"
);

// 搜索附近 100km 的门店
GeoResults<RedisGeoCommands.GeoLocation<String>> results =
    geoOps.radius("store:locations",
        new Circle(new Point(116.40, 39.90),
            new Distance(100, Metrics.KILOMETERS)));
```

### 7.4 Stream（消息流，Redis 5.0+）

Stream 是 Redis 内置的 **消息队列**，支持消费者组、消息确认、持久化。

```
Stream: mystream
┌──────────────┬──────────────┬──────────────┐
│ 1697000001-0 │ 1697000002-0 │ 1697000003-0 │
│ name: "Tom"  │ name: "Jim"  │ name: "Amy"  │
│ age: 20      │ age: 25      │ age: 22      │
└──────────────┴──────────────┴──────────────┘
                     ▲
               Consumer Group
            ┌────────┴────────┐
        Consumer-1       Consumer-2
        (已读到002)      (已读到001)
```

```bash
# 添加消息
XADD mystream * name Tom age 20

# 创建消费者组
XGROUP CREATE mystream mygroup 0

# 消费者读取
XREADGROUP GROUP mygroup consumer-1 COUNT 1 BLOCK 2000 STREAMS mystream >

# 确认消息
XACK mystream mygroup 1697000001-0
```

---

## 八、Redis 6.0 多线程模型

### 8.1 多线程架构（核心考点 ⭐⭐⭐）

Redis 6.0 引入 **IO 多线程**，但 **命令执行仍然是单线程**。

```
                 ┌───────────────────────────────────┐
Client 请求 ──►  │        IO 读取（多线程并行）        │
                 │  Thread-1  Thread-2  Thread-3     │
                 └────────────────┬──────────────────┘
                                  │ 读取完毕
                                  ▼
                 ┌───────────────────────────────────┐
                 │     命令执行（主线程串行执行）       │
                 │          Main Thread              │
                 └────────────────┬──────────────────┘
                                  │ 执行完毕
                                  ▼
                 ┌───────────────────────────────────┐
Client 响应 ◄──  │        IO 写回（多线程并行）        │
                 │  Thread-1  Thread-2  Thread-3     │
                 └───────────────────────────────────┘
```

### 8.2 为什么只对 IO 做多线程

| 方面 | 说明 |
|------|------|
| 瓶颈所在 | Redis 的性能瓶颈通常在 **网络 IO**，而非 CPU 计算 |
| 线程安全 | 命令执行保持单线程，无需加锁，代码简单且安全 |
| 性能提升 | IO 多线程可提升 2 倍以上网络吞吐量 |

### 8.3 开启多线程配置

```conf
# redis.conf
io-threads 4              # IO 线程数（建议设置为 CPU 核心数的 3/4）
io-threads-do-reads yes   # 开启读的多线程（默认只开启写的多线程）
```

---

## 九、底层编码转换条件总结

| 数据类型 | 编码 A | 编码 B | 转换条件（A → B） |
|---------|--------|--------|------------------|
| **String** | `int` | `embstr`/`raw` | 值不再是整数 |
| **String** | `embstr` | `raw` | 长度超过 44 字节，或对 embstr 执行修改命令 |
| **Hash** | `ziplist` | `hashtable` | field 数 > 128 **或** 单个 value > 64 字节 |
| **List** | `quicklist` | — | 始终使用 quicklist（Redis 3.2+） |
| **Set** | `intset` | `hashtable` | 存在非整数元素 **或** 元素数 > 512 |
| **ZSet** | `ziplist` | `skiplist` + `hashtable` | 元素数 > 128 **或** 单个元素 > 64 字节 |

> ⚠️ **注意**：编码转换是 **单向** 的（从紧凑编码到通用编码），一旦转换不会自动回退。

---

## 十、大厂常见面试题

### Q1：Redis 为什么快？单线程为什么能扛住高并发？

**答：**

Redis 快的原因有五个方面：
1. **纯内存操作**：数据存储在内存中，避免了磁盘 IO
2. **单线程模型**：避免了多线程上下文切换和锁竞争开销
3. **IO 多路复用**：基于 epoll 实现，一个线程可以处理数万连接
4. **高效数据结构**：如 SDS、跳表、ziplist 都是针对性能优化设计的
5. **渐进式 Rehash**：避免一次性大量数据迁移导致阻塞

单线程并不意味着"慢"，因为 Redis 的操作大多是内存操作，单次操作在纳秒级别完成。瓶颈往往在网络 IO 而非 CPU，IO 多路复用解决了并发连接的问题。

---

### Q2：SDS 和 C 字符串有什么区别？为什么不用 C 原生字符串？

**答：**

SDS 相比 C 字符串有四大优势：
1. **O(1) 获取长度**：SDS 维护 `len` 字段，C 字符串需要 O(n) 遍历
2. **杜绝缓冲区溢出**：修改前检查空间是否充足，不足则自动扩容
3. **减少内存重分配**：通过空间预分配和惰性释放，减少扩容/缩容次数
4. **二进制安全**：以 `len` 而非 `\0` 判断结尾，可以存储图片、音视频等二进制数据

---

### Q3：Redis 的跳表是怎么实现的？为什么不用红黑树或 B+ 树？

**答：**

**跳表实现**：在有序链表上建立多层索引，每层以概率 p=0.25 向上提升，最高 32 层。查找从最高层开始，逐层下降缩小范围，时间复杂度 O(log n)。

**不用红黑树的原因**：
- 跳表实现简单，代码量约为红黑树的 1/3
- 范围查询性能更好，跳表直接沿链表遍历
- 跳表的修改操作只涉及局部节点，并发友好

**不用 B+ 树的原因**：
- B+ 树是为磁盘访问优化的（减少磁盘 IO），Redis 数据在内存中，不需要这种优化
- 跳表在内存中的操作效率与 B+ 树相当，且实现更简单

---

### Q4：Hash 类型什么时候用 ziplist，什么时候用 hashtable？

**答：**

Hash 默认使用 ziplist 编码（内存紧凑），当满足以下 **任一** 条件时转为 hashtable：
1. 某个 field 或 value 的长度超过 `hash-max-ziplist-value`（默认 64 字节）
2. field 的数量超过 `hash-max-ziplist-entries`（默认 128 个）

转换是 **不可逆** 的，一旦转为 hashtable 不会再回退为 ziplist。这种设计是因为 ziplist 适合少量数据，数据量大时查找效率下降（O(n)），而 hashtable 查找为 O(1)。

---

### Q5：Redis 6.0 的多线程和 Memcached 的多线程有什么区别？

**答：**

| 对比项 | Redis 6.0 | Memcached |
|--------|-----------|-----------|
| 多线程范围 | 仅 **网络 IO** 读写 | **所有操作**（IO + 命令执行） |
| 命令执行 | **单线程**串行执行 | 多线程并行执行 |
| 线程安全 | 无需加锁（命令串行） | 需要加锁保护共享数据 |
| 数据结构 | 丰富（String/Hash/List/Set/ZSet 等） | 仅 Key-Value |

Redis 这种 "IO 多线程 + 命令单线程" 的设计，既提升了网络吞吐量，又保持了代码的简洁性和线程安全性。

---

### Q6：HyperLogLog 的原理是什么？为什么 12KB 就能统计上亿数据？

**答：**

HyperLogLog 基于 **概率统计** 原理：
1. 将元素 Hash 后取二进制表示
2. 统计二进制中前导零的最大长度（类似"抛硬币连续正面"的概率）
3. 前导零越长，说明样本量越大

Redis 的 HyperLogLog 使用 **16384 个桶（寄存器）**，每个桶占 6 bit，总共 `16384 × 6 / 8 = 12288 字节 ≈ 12 KB`。通过调和平均数合并各桶的估算值，误差率约 0.81%。

适用于不需要精确值的大规模去重统计，如 UV 统计、独立 IP 数等。

---

*下一篇：[02-Redis 持久化机制](./02-Redis持久化机制.md)*
