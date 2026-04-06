# 05 - Spring 事务管理

> 本章深入讲解 Spring 事务管理机制，覆盖声明式事务原理、传播行为、隔离级别、事务失效场景等核心考点。

## 目录

1. [事务基础](#一事务基础)
2. [Spring 事务管理方式](#二spring-事务管理方式)
3. [@Transactional 详解](#三transactional-详解)
4. [事务传播行为](#四事务传播行为)
5. [事务隔离级别](#五事务隔离级别)
6. [事务失效场景](#六事务失效场景)
7. [分布式事务](#七分布式事务)
8. [大厂常见面试题](#八大厂常见面试题)

---

## 一、事务基础

### 1.1 ACID 特性

| 特性 | 英文 | 说明 |
|------|------|------|
| **原子性** | Atomicity | 事务中的操作要么全部成功，要么全部回滚 |
| **一致性** | Consistency | 事务前后数据满足完整性约束 |
| **隔离性** | Isolation | 并发事务之间互不干扰 |
| **持久性** | Durability | 事务一旦提交，数据永久保存 |

---

## 二、Spring 事务管理方式

### 2.1 编程式 vs 声明式

| 方式 | 实现 | 优点 | 缺点 |
|------|------|------|------|
| **编程式** | TransactionTemplate / PlatformTransactionManager | 精细控制 | 代码侵入 |
| **声明式** | @Transactional 注解 | 无侵入，简洁 | 粒度是方法级别 |

### 2.2 编程式事务

```java
@Service
public class OrderService {

    @Autowired
    private TransactionTemplate transactionTemplate;

    public void createOrder(OrderDTO dto) {
        transactionTemplate.execute(status -> {
            try {
                orderMapper.insert(dto);
                stockService.deduct(dto.getProductId(), dto.getQuantity());
                return true;
            } catch (Exception e) {
                status.setRollbackOnly();
                throw e;
            }
        });
    }
}
```

### 2.3 声明式事务原理

```
1. @Transactional 标注的方法/类
2. Spring AOP 创建代理对象
3. 代理方法执行前：
   ├── 获取 PlatformTransactionManager
   ├── 根据传播行为决定是否开启新事务
   └── 关闭自动提交（setAutoCommit(false)）
4. 执行目标方法
5. 正常返回 → 提交事务（commit）
6. 抛出异常 → 回滚事务（rollback）
   └── 默认只回滚 RuntimeException 和 Error
```

---

## 三、@Transactional 详解

### 3.1 核心属性

| 属性 | 说明 | 默认值 |
|------|------|--------|
| `propagation` | 传播行为 | REQUIRED |
| `isolation` | 隔离级别 | DEFAULT（使用数据库默认） |
| `timeout` | 超时时间（秒） | -1（不超时） |
| `readOnly` | 是否只读事务 | false |
| `rollbackFor` | 触发回滚的异常类型 | RuntimeException, Error |
| `noRollbackFor` | 不触发回滚的异常类型 | - |

### 3.2 推荐用法

```java
// 写操作：明确指定 rollbackFor
@Transactional(rollbackFor = Exception.class)
public void createOrder(OrderDTO dto) {
    orderMapper.insert(dto);
    stockService.deduct(dto.getProductId(), dto.getQuantity());
}

// 只读操作：标记 readOnly 优化性能
@Transactional(readOnly = true)
public OrderVO getOrder(Long id) {
    return orderMapper.selectById(id);
}
```

---

## 四、事务传播行为

### 4.1 七种传播行为（核心考点 ⭐⭐⭐）

| 传播行为 | 说明 | 常用场景 |
|---------|------|---------|
| **REQUIRED** | 有事务则加入，无则新建（默认） | 大部分业务方法 |
| **REQUIRES_NEW** | 总是新建事务，挂起当前事务 | 独立日志记录、审计 |
| **NESTED** | 有事务则嵌套（保存点），无则新建 | 部分回滚场景 |
| **SUPPORTS** | 有事务则加入，无则非事务执行 | 查询方法 |
| **NOT_SUPPORTED** | 总是非事务执行，挂起当前事务 | 大量数据查询 |
| **MANDATORY** | 必须在事务中执行，否则抛异常 | 强制要求事务的方法 |
| **NEVER** | 必须在非事务中执行，否则抛异常 | 不允许事务的方法 |

### 4.2 核心传播行为对比

```
场景：A 方法（有事务）调用 B 方法

REQUIRED：   A 和 B 在同一个事务中，B 异常 → A 和 B 都回滚
REQUIRES_NEW：B 开启新事务，B 异常只回滚 B，A 可以 catch 后继续
NESTED：     B 创建保存点，B 异常回滚到保存点，A 可以 catch 后继续
```

### 4.3 示例：REQUIRES_NEW

```java
@Service
public class OrderService {

    @Autowired
    private LogService logService;

    @Transactional(rollbackFor = Exception.class)
    public void createOrder(OrderDTO dto) {
        orderMapper.insert(dto);
        try {
            // 日志记录独立事务，失败不影响订单
            logService.saveLog(dto);
        } catch (Exception e) {
            log.warn("日志记录失败", e);
        }
    }
}

@Service
public class LogService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveLog(OrderDTO dto) {
        logMapper.insert(new OperationLog(dto));
    }
}
```

---

## 五、事务隔离级别

### 5.1 四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 性能 |
|---------|------|----------|------|------|
| **READ_UNCOMMITTED** | ✅ | ✅ | ✅ | 最高 |
| **READ_COMMITTED** | ❌ | ✅ | ✅ | 高 |
| **REPEATABLE_READ** | ❌ | ❌ | ✅ | 中 |
| **SERIALIZABLE** | ❌ | ❌ | ❌ | 最低 |

- MySQL 默认：**REPEATABLE_READ**（通过 MVCC + Next-Key Lock 基本解决幻读）
- Oracle 默认：**READ_COMMITTED**
- Spring 默认：**DEFAULT**（使用数据库默认隔离级别）

### 5.2 并发问题说明

| 问题 | 说明 |
|------|------|
| **脏读** | 读到其他事务未提交的数据 |
| **不可重复读** | 同一事务中两次读取同一数据结果不同（其他事务修改了数据） |
| **幻读** | 同一事务中两次查询结果集不同（其他事务插入/删除了数据） |

---

## 六、事务失效场景

### 6.1 常见失效场景（核心考点 ⭐⭐⭐）

| 序号 | 失效场景 | 原因 |
|------|---------|------|
| 1 | **方法不是 public** | Spring AOP 只能代理 public 方法 |
| 2 | **自调用（this 调用）** | 绕过了代理对象，直接调用目标方法 |
| 3 | **异常被 catch** | 事务管理器感知不到异常，不会回滚 |
| 4 | **抛出 checked 异常** | 默认只回滚 RuntimeException 和 Error |
| 5 | **非 Spring 管理的类** | 没有被 Spring 代理 |
| 6 | **数据库不支持事务** | MyISAM 引擎不支持事务 |
| 7 | **propagation 设置错误** | 如 NOT_SUPPORTED 不使用事务 |
| 8 | **多线程调用** | 新线程不在同一个事务中 |

### 6.2 自调用问题详解

```java
@Service
public class OrderService {

    // ❌ 事务失效！this.createOrderInternal() 没走代理
    @Transactional(rollbackFor = Exception.class)
    public void createOrder(OrderDTO dto) {
        this.createOrderInternal(dto); // 自调用
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void createOrderInternal(OrderDTO dto) {
        orderMapper.insert(dto);
    }
}
```

**解决方案：**

```java
// 方案一：注入自身代理
@Service
public class OrderService {
    @Autowired
    private OrderService self; // 注入代理对象

    public void createOrder(OrderDTO dto) {
        self.createOrderInternal(dto); // 通过代理调用
    }
}

// 方案二：AopContext 获取代理
@Service
public class OrderService {
    public void createOrder(OrderDTO dto) {
        ((OrderService) AopContext.currentProxy()).createOrderInternal(dto);
    }
}

// 方案三：拆分到不同类（推荐）
```

### 6.3 异常处理注意事项

```java
// ❌ 事务不会回滚（异常被吞了）
@Transactional(rollbackFor = Exception.class)
public void createOrder(OrderDTO dto) {
    try {
        orderMapper.insert(dto);
        stockService.deduct(dto.getProductId());
    } catch (Exception e) {
        log.error("创建订单失败", e);
        // 异常被 catch，事务管理器不知道发生了异常
    }
}

// ✅ 正确做法：catch 后重新抛出或手动回滚
@Transactional(rollbackFor = Exception.class)
public void createOrder(OrderDTO dto) {
    try {
        orderMapper.insert(dto);
        stockService.deduct(dto.getProductId());
    } catch (Exception e) {
        log.error("创建订单失败", e);
        throw e; // 重新抛出
        // 或者 TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```

---

## 七、分布式事务

### 7.1 分布式事务方案

| 方案 | 原理 | 一致性 | 性能 | 适用场景 |
|------|------|--------|------|---------|
| **2PC/XA** | 两阶段提交 | 强一致 | 低 | 金融转账 |
| **TCC** | Try-Confirm-Cancel | 强一致 | 中 | 资金相关 |
| **Saga** | 长事务拆分 + 补偿 | 最终一致 | 高 | 长流程业务 |
| **本地消息表** | 消息 + 定时任务 | 最终一致 | 高 | 异步通知 |
| **RocketMQ 事务消息** | 半消息 + 回查 | 最终一致 | 高 | 消息驱动 |
| **Seata AT** | 自动补偿 | 最终一致 | 中 | 微服务通用 |

### 7.2 Seata AT 模式

```
1. TM（事务管理器）向 TC（事务协调器）开启全局事务
2. RM（资源管理器）执行本地事务，生成 undo_log
3. 本地事务提交，向 TC 报告分支事务状态
4. TM 决定全局提交或回滚
   ├── 提交 → 异步删除 undo_log
   └── 回滚 → 根据 undo_log 反向补偿
```

---

## 八、大厂常见面试题

### Q1：Spring 事务的传播行为有哪些？REQUIRED 和 REQUIRES_NEW 的区别？

**答：**
- Spring 有 7 种传播行为：REQUIRED、REQUIRES_NEW、NESTED、SUPPORTS、NOT_SUPPORTED、MANDATORY、NEVER
- **REQUIRED**（默认）：有事务加入，无则新建，所有操作在同一事务中
- **REQUIRES_NEW**：总是新建独立事务，当前事务被挂起，新事务独立提交/回滚
- 核心区别：REQUIRED 异常会导致所有操作回滚；REQUIRES_NEW 新事务异常不影响外部事务（如果 catch 了）

### Q2：@Transactional 事务失效的场景有哪些？

**答：**
1. 方法不是 public（AOP 限制）
2. 自调用（绕过代理）
3. 异常被 catch 吞掉
4. 抛出 checked 异常未指定 rollbackFor
5. 非 Spring 管理的类
6. 数据库引擎不支持事务（如 MyISAM）
7. 传播行为设为 NOT_SUPPORTED
8. 多线程中新线程不在同一事务中

### Q3：Spring 事务的实现原理？

**答：** Spring 声明式事务基于 AOP 实现。`@Transactional` 注解的方法会被 `TransactionInterceptor` 拦截，在方法执行前开启事务（关闭 autocommit），执行后根据是否有异常决定提交或回滚。底层通过 `PlatformTransactionManager` 操作数据库连接。

### Q4：如何解决分布式事务？

**答：** 根据业务场景选择：强一致性用 Seata AT/TCC 模式；最终一致性用本地消息表或 RocketMQ 事务消息；长流程用 Saga 模式。一般互联网业务追求最终一致性，尽量避免分布式事务（如合理的服务拆分）。

### Q5：NESTED 和 REQUIRES_NEW 的区别？

**答：**
- **REQUIRES_NEW**：新建独立事务，两个事务完全独立，外部事务回滚不影响内部
- **NESTED**：嵌套事务，在外部事务中创建保存点，内部回滚只到保存点，但外部事务回滚会导致内部也回滚
- NESTED 依赖数据库 Savepoint 支持，底层是同一个数据库连接

---

*下一篇：[06-Spring 常见面试题](./06-Spring常见面试题.md)*
