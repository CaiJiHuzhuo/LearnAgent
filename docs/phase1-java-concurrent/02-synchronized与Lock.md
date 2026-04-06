# 02 - synchronized 与 Lock

## 目录

1. [synchronized 关键字](#一synchronized-关键字)
2. [synchronized 底层原理](#二synchronized-底层原理)
3. [锁升级过程](#三锁升级过程)
4. [Lock 接口与 ReentrantLock](#四lock-接口与-reentrantlock)
5. [ReadWriteLock](#五readwritelock)
6. [StampedLock](#六stampedlock)
7. [synchronized 与 Lock 对比](#七synchronized-与-lock-对比)
8. [大厂常见面试题](#八大厂常见面试题)

---

## 一、synchronized 关键字

### 1.1 三种使用方式

```java
public class SyncDemo {

    private int count = 0;
    private static int staticCount = 0;

    // 1. 修饰实例方法 —— 锁对象是 this（当前实例）
    public synchronized void instanceMethod() {
        count++;
    }

    // 2. 修饰静态方法 —— 锁对象是当前类的 Class 对象
    public static synchronized void staticMethod() {
        staticCount++;
    }

    // 3. 修饰代码块 —— 锁对象可以指定
    public void blockMethod() {
        synchronized (this) {
            count++;
        }

        // 锁类对象（等效于锁静态方法）
        synchronized (SyncDemo.class) {
            staticCount++;
        }
    }
}
```

### 1.2 synchronized 的可重入性

```java
public class ReentrantDemo {
    public synchronized void methodA() {
        System.out.println("methodA");
        methodB(); // 同一线程可再次获取同一把锁
    }

    public synchronized void methodB() {
        System.out.println("methodB");
    }

    public static void main(String[] args) {
        new ReentrantDemo().methodA(); // 不会死锁
    }
}
```

> synchronized 是**可重入锁**，同一线程可多次获取同一锁，通过计数器（`_recursions`）维护重入次数。

---

## 二、synchronized 底层原理

### 2.1 对象头（Object Header）

Java 对象在堆中的内存布局：

```
┌─────────────────────────────┐
│  Mark Word (8 bytes)        │  ← 存储锁信息、GC 年龄、哈希码
│  Klass Pointer (4/8 bytes)  │  ← 指向类元数据
│  Instance Data              │  ← 实例字段
│  Padding                    │  ← 对齐填充
└─────────────────────────────┘
```

**Mark Word 不同状态下的存储内容：**

| 锁状态 | 存储内容 |
|--------|----------|
| 无锁 | 对象 hashcode、GC年龄、01 |
| 偏向锁 | 线程ID、Epoch、GC年龄、1 01 |
| 轻量级锁 | 栈中锁记录指针、00 |
| 重量级锁 | Monitor 对象指针、10 |
| GC标记 | 空、11 |

### 2.2 Monitor（监视器锁）

每个 Java 对象都关联一个 `ObjectMonitor`（C++ 实现），包含：

```
ObjectMonitor {
    _owner       → 持有锁的线程
    _waitSet     → 调用 wait() 等待的线程集合
    _entryList   → 竞争锁失败的线程阻塞队列
    _recursions  → 重入计数
    _count       → 记录锁被持有/等待次数
}
```

`synchronized` 字节码层面：
- **进入同步块**：`monitorenter` 指令
- **退出同步块**：`monitorexit` 指令（包括异常退出，编译器自动生成两条 `monitorexit`）

```
// javap -c SyncDemo.class 部分输出
0: aload_0
1: dup
2: astore_1
3: monitorenter    ← 获取 Monitor
4: ...
8: aload_1
9: monitorexit     ← 正常释放
10: goto 18
13: aload_1
14: monitorexit    ← 异常释放
```

---

## 三、锁升级过程

JDK 6 引入锁优化，锁状态只能升级不能降级（JDK 15 后偏向锁被废弃）：

```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
```

### 3.1 偏向锁（Biased Locking）

> JDK 15 废弃，JDK 18 彻底移除

- 假设锁只被一个线程访问，在 Mark Word 记录线程 ID
- 该线程再次加锁只需检查线程 ID，无需 CAS 操作
- 有其他线程竞争时，偏向锁撤销（STW），升级为轻量级锁

### 3.2 轻量级锁（Lightweight Lock）

- 在线程栈帧中创建 **Lock Record**，通过 CAS 将 Mark Word 替换为锁记录地址
- 适合**交替执行**、**竞争不激烈**场景
- CAS 自旋失败（默认自适应自旋）后升级为重量级锁

### 3.3 重量级锁（Heavyweight Lock）

- 依赖操作系统 `mutex`，涉及**用户态/内核态切换**，开销大
- 未抢到锁的线程进入阻塞队列（`_entryList`），等待操作系统调度

### 3.4 锁升级图示

```
首次加锁
    │
    ▼ (匿名偏向)
偏向锁（CAS 写入线程ID）
    │ 有竞争/调用 hashCode()
    ▼
轻量级锁（CAS + 自旋）
    │ 竞争激烈，自旋失败
    ▼
重量级锁（OS Mutex）
```

---

## 四、Lock 接口与 ReentrantLock

### 4.1 Lock 接口核心方法

```java
public interface Lock {
    void lock();                          // 获取锁（阻塞）
    void lockInterruptibly() throws InterruptedException; // 可中断获取
    boolean tryLock();                    // 尝试获取，立即返回
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();                        // 释放锁
    Condition newCondition();             // 创建条件变量
}
```

### 4.2 ReentrantLock 基本使用

```java
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockDemo {
    private final ReentrantLock lock = new ReentrantLock();
    private int count = 0;

    public void increment() {
        lock.lock(); // 必须在 try 外获取锁
        try {
            count++;
        } finally {
            lock.unlock(); // 必须在 finally 中释放
        }
    }

    // 尝试获取锁（非阻塞）
    public boolean tryIncrement() {
        if (lock.tryLock()) {
            try {
                count++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }

    // 可中断等待锁
    public void interruptibleIncrement() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
}
```

### 4.3 公平锁 vs 非公平锁

```java
// 公平锁：按等待顺序获取锁
ReentrantLock fairLock = new ReentrantLock(true);
// 非公平锁（默认）：允许插队，吞吐量更高
ReentrantLock unfairLock = new ReentrantLock(false);
```

**非公平锁为什么性能更高？**  
线程唤醒需要从阻塞态切换到就绪态，有时间开销。非公平锁允许当前持有 CPU 的线程直接获取锁，减少了线程切换次数。

### 4.4 Condition 条件变量（生产者-消费者）

```java
import java.util.concurrent.locks.*;
import java.util.*;

public class BoundedBuffer<T> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    public BoundedBuffer(int capacity) {
        this.capacity = capacity;
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await(); // 等待"不满"信号
            }
            queue.offer(item);
            notEmpty.signal(); // 通知消费者
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await(); // 等待"不空"信号
            }
            T item = queue.poll();
            notFull.signal(); // 通知生产者
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 五、ReadWriteLock

`ReentrantReadWriteLock` 允许多个读线程同时持有读锁，但写锁是独占的。

```
读-读：不互斥（并发）
读-写：互斥
写-写：互斥
```

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;
import java.util.*;

public class Cache<K, V> {
    private final Map<K, V> map = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final ReentrantReadWriteLock.ReadLock  readLock  = rwLock.readLock();
    private final ReentrantReadWriteLock.WriteLock writeLock = rwLock.writeLock();

    public V get(K key) {
        readLock.lock();
        try {
            return map.get(key);
        } finally {
            readLock.unlock();
        }
    }

    public void put(K key, V value) {
        writeLock.lock();
        try {
            map.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

    // 锁降级：写锁 → 读锁（写完后继续读，不让其他写线程抢入）
    public V getOrLoad(K key) {
        readLock.lock();
        V value = null;
        try {
            value = map.get(key);
        } finally {
            readLock.unlock();
        }
        if (value != null) return value;

        writeLock.lock();
        try {
            // 双重检查
            value = map.get(key);
            if (value == null) {
                value = loadFromDB(key);
                map.put(key, value);
            }
            readLock.lock(); // 锁降级：获取读锁
        } finally {
            writeLock.unlock(); // 释放写锁（读锁仍持有）
        }
        try {
            return value;
        } finally {
            readLock.unlock();
        }
    }

    private V loadFromDB(K key) { return null; /* 模拟DB加载 */ }
}
```

---

## 六、StampedLock

JDK 8 新增，在读写锁基础上引入**乐观读**，进一步提升读多写少场景性能。

```java
import java.util.concurrent.locks.StampedLock;

public class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    // 写操作
    void move(double deltaX, double deltaY) {
        long stamp = sl.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);
        }
    }

    // 乐观读（不加锁，读完后验证）
    double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead(); // 乐观读戳
        double curX = x, curY = y;
        if (!sl.validate(stamp)) { // 检查是否有写操作
            stamp = sl.readLock();  // 降级为悲观读
            try {
                curX = x;
                curY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.sqrt(curX * curX + curY * curY);
    }
}
```

> ⚠️ `StampedLock` **不可重入**，且不支持 `Condition`，使用需谨慎。

---

## 七、synchronized 与 Lock 对比

| 特性 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 实现层面 | JVM 内置，字节码指令 | Java API（AQS） |
| 锁类型 | 非公平（默认） | 公平/非公平可选 |
| 可中断 | 不支持 | 支持 `lockInterruptibly()` |
| 超时获取 | 不支持 | 支持 `tryLock(timeout)` |
| 条件变量 | 一个（`wait/notify`） | 多个（`Condition`） |
| 锁释放 | 自动（代码块结束/异常） | 手动（必须在 finally 释放） |
| 性能 | JDK 6 后优化，无显著差异 | 高并发下略优 |
| 代码复杂度 | 简单 | 相对复杂 |

**选择建议：**
- 简单同步场景用 `synchronized`，JVM 可做更多优化（锁消除、锁粗化）
- 需要高级特性（公平锁、超时、可中断、多条件）时用 `ReentrantLock`

---

## 八、大厂常见面试题

### Q1：synchronized 的锁升级过程是什么？

**答：**

JDK 6 引入锁优化，`synchronized` 存在四种状态，**只能升级不能降级**：

1. **无锁**：对象刚创建，没有线程竞争。
2. **偏向锁**（JDK 15 废弃）：第一个获取锁的线程将自己的线程 ID 写入 Mark Word，之后该线程进入同步块无需 CAS，只需比较线程 ID。
3. **轻量级锁**：有线程竞争时偏向锁撤销，通过 CAS 自旋尝试获取锁，适合短时间竞争。自旋消耗 CPU，竞争激烈时升级。
4. **重量级锁**：自旋失败后升级，依赖操作系统 mutex，未获锁线程进入内核态阻塞，等待系统调度唤醒，开销大但不浪费 CPU。

---

### Q2：ReentrantLock 如何实现公平锁与非公平锁？

**答：**

`ReentrantLock` 内部有 `FairSync` 和 `NonfairSync` 两个实现，均继承自 `AbstractQueuedSynchronizer（AQS）`。

- **非公平锁（默认）**：`lock()` 时先直接尝试 CAS 抢锁，抢不到才进入 AQS 等待队列。
- **公平锁**：`lock()` 时先检查 AQS 等待队列是否有前驱节点（`hasQueuedPredecessors()`），有则排队，严格按 FIFO 顺序获取锁。

公平锁避免了饥饿问题，但吞吐量低；非公平锁允许插队，减少线程切换，吞吐量更高。

---

### Q3：synchronized 与 ReentrantLock 的本质区别是什么？

**答：**

- `synchronized` 是 **JVM 级别**的关键字，通过 `monitorenter/monitorexit` 字节码指令实现，JVM 负责加锁解锁及异常处理。
- `ReentrantLock` 是 **Java API 级别**，基于 `AbstractQueuedSynchronizer（AQS）` 实现，本质是通过 CAS 修改状态 + LockSupport 阻塞/唤醒线程。

`synchronized` 的锁对象是 Java 对象头中的 Monitor；`ReentrantLock` 的锁状态维护在 AQS 的 `state` 字段中。

---

### Q4：什么是锁消除和锁粗化？

**答：**

- **锁消除（Lock Elimination）**：JIT 编译器通过逃逸分析，发现某段代码加锁的对象不可能被多线程访问，则自动消除该锁。例如：方法内局部变量上的 `synchronized` 通常会被消除。
- **锁粗化（Lock Coarsening）**：JIT 检测到多个连续加锁/解锁操作针对同一对象，会将多个加锁范围合并为一个更大的锁区域，减少反复加解锁的开销。

---

### Q5：如何排查 synchronized 导致的死锁？

**答：**

**死锁四个必要条件：** 互斥、请求保持、不剥夺、循环等待。

**排查工具：**

```bash
# 方式1：jstack 查看线程栈
jstack <pid> | grep -A 10 "BLOCKED"

# 方式2：jconsole 可视化查看死锁
jconsole

# 方式3：Arthas 在线诊断
thread -b  # 直接输出死锁信息
```

**典型死锁代码：**

```java
// 线程1 持有 lockA，等待 lockB
// 线程2 持有 lockB，等待 lockA
Object lockA = new Object(), lockB = new Object();
new Thread(() -> {
    synchronized (lockA) {
        try { Thread.sleep(100); } catch (InterruptedException e) {}
        synchronized (lockB) { /* ... */ }
    }
}).start();
new Thread(() -> {
    synchronized (lockB) {
        synchronized (lockA) { /* ... */ }
    }
}).start();
```

**解决方案：** 固定加锁顺序、使用 `tryLock` 超时机制、使用 `ReentrantLock` 可中断锁。

---

*参考资料：《深入理解Java虚拟机》周志明、《Java并发编程实战》*
