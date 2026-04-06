# 02 - MySQL 事务与锁

> 本章深入讲解 MySQL 事务的 ACID、隔离级别、MVCC 原理、锁机制等核心知识点。

## 目录

1. [事务基础](#一事务基础)
2. [隔离级别](#二隔离级别)
3. [MVCC 原理](#三mvcc-原理)
4. [锁机制](#四锁机制)
5. [死锁](#五死锁)
6. [大厂常见面试题](#六大厂常见面试题)

---

## 一、事务基础

### 1.1 ACID 特性

| 特性 | 说明 | 实现机制 |
|------|------|---------|
| **原子性** (Atomicity) | 事务要么全部成功，要么全部回滚 | **undo log**（回滚日志） |
| **一致性** (Consistency) | 事务前后数据满足完整性约束 | 由其他三个特性共同保证 |
| **隔离性** (Isolation) | 并发事务之间互不干扰 | **MVCC + 锁** |
| **持久性** (Durability) | 事务提交后数据永久保存 | **redo log**（重做日志） |

### 1.2 redo log 与 undo log

| 日志 | 作用 | 写入时机 |
|------|------|---------|
| **redo log** | 保证持久性，记录数据页的物理修改 | 事务执行中写入（WAL） |
| **undo log** | 保证原子性，记录数据修改前的值 | 事务执行前写入 |
| **binlog** | 主从复制和数据恢复 | 事务提交时写入 |

**WAL（Write-Ahead Logging）**：先写日志再写磁盘，确保数据不丢失。

---

## 二、隔离级别

### 2.1 四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|---------|------|----------|------|---------|
| **READ UNCOMMITTED** | ✅ | ✅ | ✅ | 无隔离 |
| **READ COMMITTED** | ❌ | ✅ | ✅ | MVCC（每次读取生成新 ReadView） |
| **REPEATABLE READ** | ❌ | ❌ | ✅* | MVCC（第一次读取生成 ReadView）+ 间隙锁 |
| **SERIALIZABLE** | ❌ | ❌ | ❌ | 串行执行 |

> *MySQL InnoDB 在 RR 级别通过 MVCC + Next-Key Lock **基本解决**了幻读问题。

- MySQL 默认隔离级别：**REPEATABLE READ**
- Oracle 默认隔离级别：READ COMMITTED

### 2.2 并发问题说明

| 问题 | 说明 | 示例 |
|------|------|------|
| **脏读** | 读到其他事务未提交的数据 | A 修改未提交，B 读到了 |
| **不可重复读** | 同一事务两次读结果不同（数据被修改） | A 两次查询，中间 B 修改了 |
| **幻读** | 同一事务两次查询结果集不同（数据被插入/删除） | A 两次范围查询，中间 B 插入了新行 |

---

## 三、MVCC 原理

### 3.1 什么是 MVCC（核心考点 ⭐⭐⭐）

**MVCC（Multi-Version Concurrency Control，多版本并发控制）**：通过保存数据的多个版本，实现读写不冲突，提高并发性能。

### 3.2 实现机制

MVCC 依赖三个组件：

| 组件 | 说明 |
|------|------|
| **隐藏字段** | `trx_id`（事务 ID）、`roll_pointer`（回滚指针） |
| **undo log 版本链** | 同一行数据的多个版本通过 roll_pointer 串联 |
| **ReadView** | 读视图，判断哪些版本对当前事务可见 |

### 3.3 ReadView 结构

| 字段 | 说明 |
|------|------|
| `m_ids` | 生成 ReadView 时活跃（未提交）的事务 ID 列表 |
| `min_trx_id` | m_ids 中最小的事务 ID |
| `max_trx_id` | 生成 ReadView 时系统应该分配的下一个事务 ID |
| `creator_trx_id` | 创建 ReadView 的事务 ID |

### 3.4 可见性判断规则

```
对于某一行数据版本（trx_id）：

1. trx_id == creator_trx_id     → ✅ 可见（自己的修改）
2. trx_id < min_trx_id          → ✅ 可见（事务已提交）
3. trx_id >= max_trx_id         → ❌ 不可见（事务在 ReadView 后才开始）
4. min_trx_id <= trx_id < max_trx_id
   ├── trx_id 在 m_ids 中       → ❌ 不可见（事务未提交）
   └── trx_id 不在 m_ids 中     → ✅ 可见（事务已提交）

如果不可见，沿 undo log 版本链找上一个版本，继续判断
```

### 3.5 RC 与 RR 的 MVCC 区别

| 隔离级别 | ReadView 创建时机 |
|---------|-----------------|
| **READ COMMITTED** | 每次 SELECT 都创建新的 ReadView |
| **REPEATABLE READ** | 第一次 SELECT 创建，后续复用 |

这就是为什么 RR 能保证可重复读——同一事务中用同一个 ReadView，看到的数据一致。

---

## 四、锁机制

### 4.1 锁分类

```
MySQL 锁
├── 按粒度
│   ├── 全局锁（FTWRL）
│   ├── 表级锁（表锁、意向锁、MDL 锁）
│   └── 行级锁（Record Lock、Gap Lock、Next-Key Lock）
│
├── 按模式
│   ├── 共享锁（S Lock）：读锁
│   └── 排他锁（X Lock）：写锁
│
└── 按思想
    ├── 乐观锁（版本号/CAS）
    └── 悲观锁（SELECT ... FOR UPDATE）
```

### 4.2 InnoDB 行锁类型（核心考点 ⭐⭐⭐）

| 锁类型 | 说明 | 锁定范围 |
|--------|------|---------|
| **Record Lock** | 记录锁 | 锁定单条索引记录 |
| **Gap Lock** | 间隙锁 | 锁定索引记录之间的间隙（防止插入） |
| **Next-Key Lock** | 临键锁 | Record Lock + Gap Lock（左开右闭区间） |

### 4.3 加锁规则

```
InnoDB 在 RR 隔离级别下的加锁规则：

1. 主键等值查询
   ├── 记录存在 → Record Lock
   └── 记录不存在 → Gap Lock

2. 唯一索引等值查询
   ├── 记录存在 → Record Lock
   └── 记录不存在 → Gap Lock

3. 普通索引等值查询
   └── Next-Key Lock（向右遍历到不满足条件时退化为 Gap Lock）

4. 范围查询
   └── Next-Key Lock
```

### 4.4 意向锁

| 锁类型 | 说明 |
|--------|------|
| **意向共享锁（IS）** | 事务准备给某行加 S 锁前，先获取表的 IS 锁 |
| **意向排他锁（IX）** | 事务准备给某行加 X 锁前，先获取表的 IX 锁 |

意向锁的作用：快速判断表中是否有行锁，避免逐行检查。

### 4.5 乐观锁 vs 悲观锁

| 维度 | 乐观锁 | 悲观锁 |
|------|--------|--------|
| 实现 | 版本号/时间戳 | `SELECT ... FOR UPDATE` |
| 适用 | 读多写少 | 写多读少 |
| 性能 | 无锁开销，冲突时重试 | 加锁等待 |
| 死锁 | 不会 | 可能 |

---

## 五、死锁

### 5.1 死锁条件

1. **互斥**：资源只能被一个事务持有
2. **持有并等待**：持有锁的事务等待其他锁
3. **不可剥夺**：已持有的锁不能被强制释放
4. **循环等待**：多个事务形成等待环

### 5.2 死锁示例

```sql
-- 事务 A
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1; -- 锁住 id=1
UPDATE accounts SET balance = balance + 100 WHERE id = 2; -- 等待 id=2 的锁

-- 事务 B
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;  -- 锁住 id=2
UPDATE accounts SET balance = balance + 50 WHERE id = 1;  -- 等待 id=1 的锁
-- 死锁！
```

### 5.3 死锁处理

| 策略 | 说明 |
|------|------|
| **等待超时** | `innodb_lock_wait_timeout`（默认 50s） |
| **死锁检测** | `innodb_deadlock_detect=ON`（默认开启），发现死锁回滚代价小的事务 |

### 5.4 避免死锁

1. **固定加锁顺序**：按主键/唯一索引顺序加锁
2. **减少事务粒度**：事务越短，持锁时间越短
3. **避免大事务**
4. **合理使用索引**：减少锁的范围

---

## 六、大厂常见面试题

### Q1：MySQL 的隔离级别有哪些？默认是什么？

**答：** READ UNCOMMITTED、READ COMMITTED、REPEATABLE READ、SERIALIZABLE。MySQL 默认 **REPEATABLE READ**，通过 MVCC + Next-Key Lock 基本解决幻读。

### Q2：MVCC 的实现原理？

**答：** MVCC 通过隐藏字段（trx_id、roll_pointer）、undo log 版本链、ReadView 实现。ReadView 记录活跃事务列表，通过可见性规则判断数据版本是否可见。RC 每次 SELECT 创建新 ReadView，RR 只在第一次 SELECT 创建。

### Q3：InnoDB 行锁有哪些类型？

**答：**
- **Record Lock**：锁定单条记录
- **Gap Lock**：锁定索引间隙，防止插入
- **Next-Key Lock**：Record Lock + Gap Lock，锁定左开右闭区间
- RR 隔离级别默认使用 Next-Key Lock 防止幻读

### Q4：MySQL 如何解决幻读？

**答：**
- **快照读**（普通 SELECT）：通过 MVCC 的 ReadView 解决
- **当前读**（SELECT FOR UPDATE）：通过 Next-Key Lock（间隙锁 + 记录锁）阻止其他事务在间隙中插入

### Q5：什么是死锁？如何避免？

**答：** 两个或多个事务互相等待对方持有的锁。避免方法：固定加锁顺序、减少事务粒度、合理使用索引减少锁范围。InnoDB 默认开启死锁检测，发现死锁自动回滚代价较小的事务。

---

*下一篇：[03-SQL 优化实战](./03-SQL优化实战.md)*
