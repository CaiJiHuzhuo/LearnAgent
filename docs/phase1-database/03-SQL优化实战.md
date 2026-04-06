# 03 - SQL 优化实战

> 本章讲解 SQL 优化的方法论、慢查询分析、常见优化技巧及实战案例。

## 目录

1. [SQL 优化方法论](#一sql-优化方法论)
2. [慢查询日志](#二慢查询日志)
3. [EXPLAIN 分析](#三explain-分析)
4. [常见优化技巧](#四常见优化技巧)
5. [JOIN 优化](#五join-优化)
6. [分页优化](#六分页优化)
7. [大厂常见面试题](#七大厂常见面试题)

---

## 一、SQL 优化方法论

### 1.1 优化流程

```
1. 定位慢 SQL（慢查询日志 / APM 工具）
2. EXPLAIN 分析执行计划
3. 分析是否走索引、扫描行数、排序方式
4. 优化方案：
   ├── 加索引 / 优化索引
   ├── 改写 SQL
   ├── 优化表结构
   └── 业务层优化（缓存/异步）
5. 验证优化效果
```

### 1.2 优化原则

1. **减少数据访问量**：只查需要的列和行
2. **减少交互次数**：批量操作，减少网络往返
3. **减少服务器 CPU 开销**：避免排序、临时表
4. **利用更多资源**：读写分离、分库分表

---

## 二、慢查询日志

### 2.1 开启慢查询

```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 动态开启
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过 1 秒记录
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未使用索引的查询
```

### 2.2 分析工具

| 工具 | 说明 |
|------|------|
| **mysqldumpslow** | MySQL 自带，简单分析 |
| **pt-query-digest** | Percona Toolkit，功能强大 |
| **SkyWalking / Arthas** | APM 工具，实时监控慢 SQL |

---

## 三、EXPLAIN 分析

### 3.1 核心字段说明

```sql
EXPLAIN SELECT * FROM t_order WHERE user_id = 1001 ORDER BY create_time DESC LIMIT 10;
```

| 字段 | 说明 |
|------|------|
| **id** | 查询序号，id 越大优先级越高 |
| **select_type** | 查询类型（SIMPLE/PRIMARY/SUBQUERY） |
| **table** | 查询的表 |
| **type** | 访问类型（const > eq_ref > ref > range > index > ALL） |
| **possible_keys** | 可能使用的索引 |
| **key** | 实际使用的索引 |
| **key_len** | 索引使用长度 |
| **rows** | 预估扫描行数 |
| **filtered** | 过滤比例 |
| **Extra** | 额外信息 |

### 3.2 type 优化目标

```
目标：至少达到 range 级别，最好达到 ref 或 const

const    → 主键/唯一索引精确匹配（最佳）
eq_ref   → JOIN 使用主键/唯一索引（很好）
ref      → 非唯一索引等值查询（好）
range    → 索引范围查询（可接受）
index    → 全索引扫描（需优化）
ALL      → 全表扫描（必须优化）
```

---

## 四、常见优化技巧

### 4.1 SELECT 优化

```sql
-- ❌ 避免 SELECT *
SELECT * FROM t_user WHERE id = 1;

-- ✅ 只查需要的列
SELECT id, username, email FROM t_user WHERE id = 1;

-- ❌ 避免在 WHERE 中对列使用函数
SELECT * FROM t_order WHERE YEAR(create_time) = 2024;

-- ✅ 改为范围查询
SELECT * FROM t_order
WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';
```

### 4.2 INSERT 优化

```sql
-- ❌ 逐条插入
INSERT INTO t_user (name, age) VALUES ('A', 20);
INSERT INTO t_user (name, age) VALUES ('B', 21);

-- ✅ 批量插入
INSERT INTO t_user (name, age) VALUES ('A', 20), ('B', 21), ('C', 22);

-- ✅ 大批量数据使用 LOAD DATA
LOAD DATA INFILE '/data/users.csv' INTO TABLE t_user
FIELDS TERMINATED BY ',' LINES TERMINATED BY '\n';
```

### 4.3 UPDATE 优化

```sql
-- ❌ 避免无索引的 UPDATE（锁全表）
UPDATE t_user SET status = 0 WHERE name = 'test'; -- name 无索引

-- ✅ 确保 WHERE 条件走索引
UPDATE t_user SET status = 0 WHERE id = 1001;
```

### 4.4 COUNT 优化

```sql
-- COUNT(*) vs COUNT(1) vs COUNT(列)
-- COUNT(*) = COUNT(1) > COUNT(主键) > COUNT(普通列)
-- COUNT(*) 和 COUNT(1) 性能相同，MySQL 已优化
-- COUNT(列) 会排除 NULL 值

-- 大表 COUNT 优化方案：
-- 1. 维护计数器（Redis/数据库计数表）
-- 2. 近似值：SHOW TABLE STATUS（rows 字段）
-- 3. 缓存计数
```

### 4.5 ORDER BY 优化

```sql
-- 确保 ORDER BY 使用索引，避免 filesort
-- 组合索引 idx_user_create (user_id, create_time)

-- ✅ 走索引排序
SELECT * FROM t_order WHERE user_id = 1001 ORDER BY create_time DESC;

-- ❌ filesort（排序字段不在索引中）
SELECT * FROM t_order WHERE user_id = 1001 ORDER BY amount DESC;
```

---

## 五、JOIN 优化

### 5.1 JOIN 算法

| 算法 | 说明 | 条件 |
|------|------|------|
| **Nested Loop Join** | 嵌套循环，逐行匹配 | 被驱动表有索引 |
| **Block Nested Loop** | 块嵌套循环（Join Buffer） | 被驱动表无索引（MySQL 8.0.18 前） |
| **Hash Join** | 哈希连接 | MySQL 8.0.18+，无索引时自动使用 |

### 5.2 JOIN 优化原则

1. **小表驱动大表**：让数据量小的表做驱动表
2. **被驱动表加索引**：关联字段必须有索引
3. **避免过多 JOIN**：阿里规约建议不超过 3 张表
4. **减少 JOIN 中返回的行数**：先过滤再 JOIN

```sql
-- ❌ 不好：大表驱动小表
SELECT * FROM t_order o
JOIN t_user u ON o.user_id = u.id
WHERE u.status = 1;

-- ✅ 更好：先过滤再 JOIN
SELECT * FROM t_order o
JOIN (SELECT id FROM t_user WHERE status = 1) u ON o.user_id = u.id;
```

---

## 六、分页优化

### 6.1 深分页问题

```sql
-- 深分页：LIMIT 1000000, 10 需要扫描 1000010 行再丢弃前 100万行
SELECT * FROM t_order ORDER BY id LIMIT 1000000, 10;
```

### 6.2 优化方案

```sql
-- 方案一：延迟关联（推荐）
SELECT o.* FROM t_order o
JOIN (SELECT id FROM t_order ORDER BY id LIMIT 1000000, 10) tmp
ON o.id = tmp.id;
-- 子查询只扫描索引，不回表，然后用主键关联获取完整数据

-- 方案二：游标分页（使用上次最大 ID）
SELECT * FROM t_order WHERE id > 上次最大ID ORDER BY id LIMIT 10;
-- 最高效，但要求 ID 连续或有序

-- 方案三：使用搜索引擎（ES）处理复杂搜索分页
```

---

## 七、大厂常见面试题

### Q1：如何定位和优化慢 SQL？

**答：**
1. 开启慢查询日志定位慢 SQL
2. EXPLAIN 分析执行计划（关注 type、key、rows、Extra）
3. 优化方案：添加/优化索引、改写 SQL（避免函数/子查询）、分页优化
4. 验证优化效果

### Q2：EXPLAIN 中哪些字段最重要？

**答：**
- **type**：至少 range，目标 ref/const
- **key**：实际用的索引，NULL 表示没走索引
- **rows**：扫描行数，越小越好
- **Extra**：`Using index`（好）、`Using filesort`/`Using temporary`（需优化）

### Q3：大表分页如何优化？

**答：**
1. **延迟关联**：子查询先走索引找 ID，再用 ID 关联获取完整数据
2. **游标分页**：`WHERE id > lastId LIMIT n`，最高效
3. **搜索引擎**：复杂搜索条件用 ES 处理

### Q4：COUNT(*) 和 COUNT(1) 的区别？

**答：** 在 MySQL 中 `COUNT(*)` 和 `COUNT(1)` 性能完全相同，MySQL 优化器已统一处理。`COUNT(列)` 会排除 NULL 值。大表统计推荐维护计数器。

### Q5：JOIN 查询如何优化？

**答：** 小表驱动大表、被驱动表关联字段加索引、控制 JOIN 表数量（≤3）、先过滤再 JOIN 减少数据量。

---

*下一篇：[04-分库分表](./04-分库分表.md)*
