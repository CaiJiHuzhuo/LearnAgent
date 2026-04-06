# 04 - AQS 原理

## 目录

1. [AQS 概述](#一aqs-概述)
2. [AQS 核心数据结构](#二aqs-核心数据结构)
3. [独占锁原理](#三独占锁原理)
4. [共享锁原理](#四共享锁原理)
5. [条件变量（ConditionObject）](#五条件变量conditionobject)
6. [基于 AQS 的组件](#六基于-aqs-的组件)
7. [自定义同步器](#七自定义同步器)
8. [大厂常见面试题](#八大厂常见面试题)

---

## 一、AQS 概述

`AbstractQueuedSynchronizer`（抽象队列同步器）是 Java 并发包的核心基础框架，由 Doug Lea 编写，位于 `java.util.concurrent.locks` 包。

**基于 AQS 实现的组件：**

| 组件 | 说明 |
|------|------|
| `ReentrantLock` | 可重入独占锁 |
| `ReentrantReadWriteLock` | 读写锁 |
| `CountDownLatch` | 倒计时门闩 |
| `Semaphore` | 信号量 |
| `CyclicBarrier` | 循环屏障 |
| `ThreadPoolExecutor.Worker` | 线程池工作线程 |

AQS 的核心思想：**用 int 类型的 `state` 变量表示同步状态，用 CLH 变体的 FIFO 双向队列（同步队列）管理等待线程**。

---

## 二、AQS 核心数据结构

### 2.1 state 变量

```java
// AQS 核心字段
private volatile int state; // 同步状态

// 三个核心操作（均为原子操作）
protected final int getState()
protected final void setState(int newState)
protected final boolean compareAndSetState(int expect, int update)
```

`state` 的语义由子类定义：
- `ReentrantLock`：0 表示未锁，> 0 表示被锁（值为重入次数）
- `Semaphore`：表示可用许可数量
- `CountDownLatch`：表示剩余计数

### 2.2 CLH 同步队列（等待队列）

AQS 内部使用一个**带头哨兵节点的双向链表**作为等待队列：

```
     head (哨兵节点)
      │
      ▼
┌─────────┐  prev  ┌─────────┐  prev  ┌─────────┐
│  Node   │◄──────│  Node   │◄──────│  Node   │
│ SIGNAL  │ next► │ SIGNAL  │ next► │  ...    │
└─────────┘        └─────────┘        └─────────┘
                                               ▲
                                               │
                                              tail
```

### 2.3 Node 节点结构

```java
static final class Node {
    // 节点模式：共享 or 独占
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;

    // 等待状态
    static final int CANCELLED =  1; // 已取消（超时或中断）
    static final int SIGNAL    = -1; // 后继节点需要唤醒
    static final int CONDITION = -2; // 在条件队列中等待
    static final int PROPAGATE = -3; // 共享模式，传播唤醒

    volatile int waitStatus;  // 节点状态
    volatile Node prev;       // 前驱
    volatile Node next;       // 后继
    volatile Thread thread;   // 等待线程
    Node nextWaiter;          // 条件队列中的下一个节点
}
```

---

## 三、独占锁原理

### 3.1 加锁流程（acquire）

```java
// AQS.acquire() 源码简化
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&          // 1. 尝试直接获取（子类实现）
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) // 2. 入队并等待
        selfInterrupt();             // 3. 根据中断标记重新中断
}

// 将当前线程包装成 Node 加入队尾
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) { // CAS 设置尾节点
            pred.next = node;
            return node;
        }
    }
    enq(node); // CAS 失败则自旋入队
    return node;
}

// 线程在队列中自旋等待
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) { // 自旋
            final Node p = node.predecessor();
            // 前驱是头节点才尝试获取锁（保证顺序）
            if (p == head && tryAcquire(arg)) {
                setHead(node); // 获取成功，出队
                p.next = null;
                failed = false;
                return interrupted;
            }
            // 判断是否需要 park（前驱 waitStatus == SIGNAL 才 park）
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt()) // LockSupport.park()
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

**加锁流程图：**

```
acquire(1)
    │
    ▼
tryAcquire(1)  ──成功──► 返回，持有锁
    │
    │失败
    ▼
addWaiter(EXCLUSIVE)  ──► 将线程包装成 Node，CAS 加入队尾
    │
    ▼
acquireQueued()
    │
    ▼（自旋）
前驱是 head？ ──是──► tryAcquire() ──成功──► setHead()，持有锁
    │                               │
    │否或失败                       │
    ▼                               │
shouldPark? ──是──► LockSupport.park() 阻塞
    │
    ▼（被唤醒）
继续自旋
```

### 3.2 解锁流程（release）

```java
// AQS.release() 源码简化
public final boolean release(int arg) {
    if (tryRelease(arg)) {            // 子类实现（ReentrantLock 减少重入计数）
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);       // 唤醒后继节点
        return true;
    }
    return false;
}

// 唤醒后继节点
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    // 跳过 CANCELLED 节点，从尾部反向找有效节点
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread); // 唤醒
}
```

### 3.3 ReentrantLock 对 AQS 的实现

```java
// ReentrantLock.NonfairSync
static final class NonfairSync extends Sync {

    final void lock() {
        // 非公平：先直接 CAS 抢一次
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 再次尝试（不检查队列）
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        } else if (current == getExclusiveOwnerThread()) {
            // 可重入：直接累加
            int nextc = c + acquires;
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

---

## 四、共享锁原理

共享锁允许多个线程同时持有（如 `Semaphore`、`CountDownLatch`、读写锁的读锁）。

```java
// AQS.acquireShared() 简化
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)    // 返回值 >= 0 表示成功
        doAcquireShared(arg);
}

// 共享模式入队并自旋
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    // ...
    for (;;) {
        final Node p = node.predecessor();
        if (p == head) {
            int r = tryAcquireShared(arg);
            if (r >= 0) {
                setHeadAndPropagate(node, r); // 唤醒后继共享节点（传播）
                return;
            }
        }
        if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
            interrupted = true;
    }
}
```

**共享锁与独占锁的关键区别：** 共享锁获取成功后，会通过 `setHeadAndPropagate` 传播唤醒后续等待的共享节点，实现多个线程同时持有。

---

## 五、条件变量（ConditionObject）

AQS 内部类 `ConditionObject` 实现了 `Condition` 接口，每个 `Condition` 对象维护一个独立的**条件队列**（单向链表）。

```
条件队列（ConditionObject 维护）
  firstWaiter → Node → Node → Node → lastWaiter
  （节点 waitStatus = CONDITION）

同步队列（AQS 主队列）
  head → Node → Node → tail
```

### await() 流程

```java
// Condition.await() 简化
public final void await() throws InterruptedException {
    // 1. 将当前线程加入条件队列
    Node node = addConditionWaiter();
    // 2. 完全释放锁（包括重入）
    int savedState = fullyRelease(node);
    // 3. 阻塞，直到被 signal 迁移到同步队列
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if (checkInterruptWhileWaiting(node) != 0) break;
    }
    // 4. 重新竞争锁
    acquireQueued(node, savedState);
}
```

### signal() 流程

```java
// Condition.signal() 简化
public final void signal() {
    // 将条件队列头节点移到 AQS 同步队列尾部
    Node first = firstWaiter;
    if (first != null)
        doSignal(first); // transferForSignal() 完成迁移
}
```

---

## 六、基于 AQS 的组件

### 6.1 Semaphore（信号量）

```java
import java.util.concurrent.Semaphore;

// 控制同时访问资源的线程数量（如连接池）
Semaphore semaphore = new Semaphore(3); // 最多3个线程同时访问

ExecutorService pool = Executors.newFixedThreadPool(10);
for (int i = 0; i < 10; i++) {
    pool.submit(() -> {
        try {
            semaphore.acquire();        // 获取许可（state-1），不足则阻塞
            System.out.println(Thread.currentThread().getName() + " 获得许可");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            semaphore.release();        // 释放许可（state+1）
        }
    });
}
pool.shutdown();
```

### 6.2 CountDownLatch

```java
import java.util.concurrent.CountDownLatch;

// 等待多个线程全部完成（state 初始值 = N，countDown 一次 state-1，await 等待 state==0）
int N = 5;
CountDownLatch latch = new CountDownLatch(N);

for (int i = 0; i < N; i++) {
    final int id = i;
    new Thread(() -> {
        System.out.println("子线程 " + id + " 执行完毕");
        latch.countDown(); // state - 1
    }).start();
}

latch.await(); // 等待 state == 0
System.out.println("所有子线程执行完毕，主线程继续");
```

---

## 七、自定义同步器

通过继承 AQS 并重写以下方法，可以实现自定义同步语义：

```java
// 独占锁需重写
protected boolean tryAcquire(int arg)   // 尝试获取
protected boolean tryRelease(int arg)   // 尝试释放

// 共享锁需重写
protected int tryAcquireShared(int arg)   // >=0 成功，<0 失败
protected boolean tryReleaseShared(int arg)

// 是否被当前线程独占
protected boolean isHeldExclusively()
```

**示例：实现一个不可重入的互斥锁**

```java
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.locks.Condition;

public class Mutex {
    private static class Sync extends AbstractQueuedSynchronizer {

        // state=0 表示未锁，state=1 表示已锁
        @Override
        protected boolean tryAcquire(int acquires) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int releases) {
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        Condition newCondition() {
            return new ConditionObject();
        }
    }

    private final Sync sync = new Sync();

    public void lock()            { sync.acquire(1); }
    public boolean tryLock()      { return sync.tryAcquire(1); }
    public void unlock()          { sync.release(1); }
    public boolean isLocked()     { return sync.isHeldExclusively(); }
    public Condition newCondition() { return sync.newCondition(); }
}
```

---

## 八、大厂常见面试题

### Q1：AQS 的核心原理是什么？

**答：**

AQS（AbstractQueuedSynchronizer）是 Java 并发包中用于构建锁和同步器的基础框架，核心有两部分：

1. **`state` 变量**：volatile int，表示同步状态，语义由子类定义（如 ReentrantLock 中为重入计数，Semaphore 中为许可数）。状态修改通过 CAS 保证原子性。

2. **CLH 变体同步队列**：当线程获取同步状态失败时，被封装成 `Node` 加入 FIFO 双向队列，通过 `LockSupport.park()` 阻塞。持有锁的线程释放后，通过 `LockSupport.unpark()` 唤醒队首线程，被唤醒的线程重新尝试获取。

通过继承 AQS 并实现 `tryAcquire`/`tryRelease`（独占）或 `tryAcquireShared`/`tryReleaseShared`（共享）方法，可以快速实现各种同步工具。

---

### Q2：AQS 中独占锁和共享锁有什么区别？

**答：**

| 模式 | 代表组件 | 获取方法 | 特点 |
|------|----------|----------|------|
| 独占（Exclusive） | ReentrantLock | `acquire` | 同一时刻只有一个线程持有 |
| 共享（Shared） | Semaphore、CountDownLatch | `acquireShared` | 多个线程可同时持有 |

关键区别在于**释放后的传播**：
- 独占锁释放时只唤醒队列中下一个节点（一个线程）
- 共享锁释放时会通过 `setHeadAndPropagate` 传播，连续唤醒后续的共享节点，实现多个线程同时获取

---

### Q3：ReentrantLock 的公平锁与非公平锁在 AQS 层面如何实现？

**答：**

两者都继承 `AQS`，区别在于 `tryAcquire` 实现：

- **非公平锁（NonfairSync）**：`lock()` 时先直接 CAS 抢占（`compareAndSetState(0, 1)`），失败才调用 `acquire(1)`；在 `tryAcquire` 中也会先 CAS，不管队列中是否有等待线程。
- **公平锁（FairSync）**：`tryAcquire` 中先调用 `hasQueuedPredecessors()` 检查队列是否有前驱节点，有则放弃抢占，乖乖排队，保证 FIFO 顺序。

---

### Q4：CountDownLatch 和 CyclicBarrier 的区别？

**答：**

| 特性 | CountDownLatch | CyclicBarrier |
|------|---------------|---------------|
| 基础 | AQS 共享锁 | ReentrantLock + Condition |
| 计数方向 | 倒计数（N→0） | 累加（0→N） |
| 重置 | **不可重置**（一次性） | **可重置循环使用** |
| 等待方 | 一个或多个线程等待 N 个事件 | N 个线程互相等待到达屏障 |
| 回调 | 无 | 可指定 `barrierAction`（所有线程到达后执行） |
| 典型场景 | 主线程等待多个子任务完成 | 多个线程分阶段协作 |

---

### Q5：LockSupport.park() 与 Object.wait() 的区别？

**答：**

| 特性 | `LockSupport.park()` | `Object.wait()` |
|------|---------------------|-----------------|
| 使用前提 | 无需持有锁 | 必须在 `synchronized` 块内 |
| 唤醒方式 | `LockSupport.unpark(thread)` 指定线程 | `notify()`/`notifyAll()` 通知对象等待集 |
| 先唤醒后阻塞 | **支持**（unpark 在 park 前，park 立即返回） | 不支持（notify 在 wait 前无效） |
| 中断响应 | 中断后 park 立即返回，不抛异常，需手动检查中断标记 | 抛出 `InterruptedException` |
| 应用 | AQS、J.U.C 底层 | Object 监视器等待 |

AQS 选择 `LockSupport.park/unpark` 是因为它可以精准唤醒指定线程，且支持"先唤醒后阻塞"，避免了 wait/notify 的竞争问题。

---

*参考资料：《Java并发编程实战》、Doug Lea《The java.util.concurrent Synchronizer Framework》、OpenJDK AQS 源码*
