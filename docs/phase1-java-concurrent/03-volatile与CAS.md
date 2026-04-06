# 03 - volatile 与 CAS

## 目录

1. [Java 内存模型（JMM）](#一java-内存模型jmm)
2. [volatile 关键字](#二volatile-关键字)
3. [volatile 底层实现](#三volatile-底层实现)
4. [CAS（Compare And Swap）](#四cascas原理)
5. [原子类（Atomic*）](#五原子类atomic)
6. [CAS 的问题与解决](#六cas-的问题与解决)
7. [大厂常见面试题](#七大厂常见面试题)

---

## 一、Java 内存模型（JMM）

### 1.1 JMM 基本概念

JMM（Java Memory Model）定义了多线程程序中共享变量的访问规则，屏蔽了不同 CPU 架构的内存差异。

```
主内存（Main Memory）
        ↑↓ read/write
┌──────────────────────────────┐
│  工作内存（Work Memory）      │
│  ┌────────┐  ┌────────┐      │
│  │ 线程1   │  │ 线程2   │     │
│  │ 本地副本│  │ 本地副本│     │
│  └────────┘  └────────┘      │
└──────────────────────────────┘
```

每个线程都有自己的**工作内存**（寄存器、L1/L2 缓存的抽象），对共享变量的操作必须先从主内存拷贝到工作内存，修改后再同步回主内存。

### 1.2 三大特性

| 特性 | 说明 |
|------|------|
| **原子性** | 操作不可被中断，要么全做要么全不做 |
| **可见性** | 一个线程修改后，其他线程能立即看到 |
| **有序性** | 程序执行顺序符合代码书写顺序（禁止指令重排序） |

### 1.3 Happens-Before 规则

JMM 通过 **Happens-Before** 关系保证可见性和有序性，常见规则：

1. **程序顺序规则**：同一线程内，前面的操作 happens-before 后面的操作
2. **volatile 规则**：volatile 写 happens-before 后续的 volatile 读
3. **锁规则**：解锁 happens-before 后续加锁
4. **线程启动规则**：`Thread.start()` happens-before 线程内所有操作
5. **线程终止规则**：线程所有操作 happens-before `Thread.join()` 返回
6. **传递性**：A hb B，B hb C → A hb C

---

## 二、volatile 关键字

### 2.1 volatile 的两个核心语义

1. **可见性**：对 volatile 变量的写操作，立即刷新到主内存；读操作，从主内存重新加载。
2. **禁止指令重排序**：通过内存屏障（Memory Barrier）防止编译器和 CPU 对 volatile 操作前后的指令进行重排。

### 2.2 可见性示例

```java
// 没有 volatile 时，flag 修改对其他线程不可见
// 线程可能永远循环（从工作内存读取旧值）
public class VisibilityDemo {
    private volatile boolean flag = true; // 加 volatile 才能及时可见

    public void stop() {
        flag = false;
    }

    public void loop() {
        while (flag) {
            // 不断轮询
        }
        System.out.println("线程已停止");
    }

    public static void main(String[] args) throws InterruptedException {
        VisibilityDemo demo = new VisibilityDemo();
        Thread t = new Thread(demo::loop);
        t.start();
        Thread.sleep(100);
        demo.stop(); // 修改 flag，loop 线程能及时看到
    }
}
```

### 2.3 有序性——双重检查锁（DCL）单例

```java
public class Singleton {
    // 必须加 volatile！
    private static volatile Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {               // 第一次检查（无锁读）
            synchronized (Singleton.class) {
                if (instance == null) {        // 第二次检查（加锁内）
                    instance = new Singleton();
                    // 对象创建分三步：
                    // 1. 分配内存空间
                    // 2. 初始化对象
                    // 3. 将引用指向内存地址
                    // 无 volatile 时 2、3 可能重排序，导致其他线程获得未初始化的对象
                }
            }
        }
        return instance;
    }
}
```

### 2.4 volatile 不能保证原子性

```java
public class AtomicityDemo {
    private volatile int count = 0;

    // 这仍然不是原子操作！i++ 等价于：
    // 1. 读取 count（volatile read）
    // 2. +1 运算
    // 3. 写回 count（volatile write）
    public void increment() {
        count++; // 线程不安全！
    }
}

// 正确做法：使用 AtomicInteger
import java.util.concurrent.atomic.AtomicInteger;
public class SafeCounter {
    private AtomicInteger count = new AtomicInteger(0);
    public void increment() {
        count.incrementAndGet(); // 原子操作
    }
}
```

---

## 三、volatile 底层实现

### 3.1 内存屏障

volatile 通过在读写操作前后插入内存屏障来实现语义：

| 屏障类型 | 作用 |
|----------|------|
| `LoadLoad` | 前面的 Load 先于后面的 Load |
| `StoreStore` | 前面的 Store 先于后面的 Store |
| `LoadStore` | 前面的 Load 先于后面的 Store |
| `StoreLoad` | 前面的 Store 先于后面的 Load（最强屏障） |

```
volatile 写：
    StoreStore Barrier
    volatile write
    StoreLoad Barrier

volatile 读：
    volatile read
    LoadLoad Barrier
    LoadStore Barrier
```

### 3.2 汇编层面

在 x86 架构上，volatile 写会生成 **`lock` 前缀指令**：

```assembly
lock addl $0x0,(%esp)   ; 空操作，但触发缓存一致性协议
```

`lock` 前缀的作用：
1. 将当前 CPU 缓存行写回主内存
2. 使其他 CPU 缓存中该内存地址的数据失效（MESI 协议）

---

## 四、CAS（CAS原理）

### 4.1 CAS 基本概念

CAS（Compare And Swap / Compare And Set）是一种**乐观锁**机制，操作包含三个参数：

- **V（内存地址）**：要更新的变量
- **E（期望值）**：期望当前值
- **N（新值）**：要更新的新值

**操作语义：** 若 V 的当前值 == E，则将 V 更新为 N，并返回 true；否则什么都不做，返回 false。

CAS 是**原子操作**，由 CPU 指令 `cmpxchg` 保证。

```java
// 概念伪代码（实际由 Unsafe.compareAndSwapInt 执行）
boolean compareAndSwap(long address, int expected, int newValue) {
    // 以下操作原子执行（CPU 层面）
    int current = *address;
    if (current == expected) {
        *address = newValue;
        return true;
    }
    return false;
}
```

### 4.2 Unsafe 类

Java 通过 `sun.misc.Unsafe` 暴露底层 CAS 操作：

```java
import sun.misc.Unsafe;
import java.lang.reflect.Field;

// AtomicInteger 内部实现
public class AtomicInteger {
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset; // value 字段的内存偏移

    static {
        try {
            valueOffset = unsafe.objectFieldOffset(
                AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value; // 必须是 volatile

    // CAS 更新
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

    // 自旋 CAS 实现 getAndIncrement
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
}
```

---

## 五、原子类（Atomic*）

### 5.1 原子类分类

| 分类 | 类 |
|------|-----|
| 基本类型 | `AtomicInteger`、`AtomicLong`、`AtomicBoolean` |
| 引用类型 | `AtomicReference`、`AtomicStampedReference`、`AtomicMarkableReference` |
| 数组类型 | `AtomicIntegerArray`、`AtomicLongArray`、`AtomicReferenceArray` |
| 字段更新 | `AtomicIntegerFieldUpdater`、`AtomicLongFieldUpdater`、`AtomicReferenceFieldUpdater` |
| 累加器 | `LongAdder`、`DoubleAdder`、`LongAccumulator`（高并发推荐） |

### 5.2 AtomicInteger 常用方法

```java
import java.util.concurrent.atomic.AtomicInteger;

AtomicInteger ai = new AtomicInteger(0);

int old = ai.getAndIncrement();  // i++，返回旧值
int nev = ai.incrementAndGet();  // ++i，返回新值
ai.getAndAdd(5);                 // 加5，返回旧值
ai.compareAndSet(0, 10);         // CAS：若当前值==0，设为10
ai.updateAndGet(x -> x * 2);    // 函数式更新
ai.accumulateAndGet(3, Integer::sum); // 累加
```

### 5.3 LongAdder vs AtomicLong

高并发场景下，`LongAdder` 性能远优于 `AtomicLong`：

```
AtomicLong：多线程竞争同一 value 字段，CAS 重试激烈

LongAdder：分段累加
  ┌──────────┐
  │  base    │  ← 无竞争时直接累加到 base
  ├──────────┤
  │ cells[0] │  ← 线程0 竞争时累加到对应 Cell
  │ cells[1] │
  │ cells[2] │
  │ ...      │
  └──────────┘
  最终结果 = base + Σcells[i]
```

```java
import java.util.concurrent.atomic.LongAdder;

LongAdder adder = new LongAdder();
adder.increment();
adder.add(100);
long sum = adder.sum(); // 汇总结果（非精确实时值）
```

### 5.4 AtomicStampedReference（解决 ABA 问题）

```java
import java.util.concurrent.atomic.AtomicStampedReference;

AtomicStampedReference<Integer> ref =
    new AtomicStampedReference<>(100, 0); // 初始值100，版本号0

int[] stampHolder = new int[1];
Integer currentValue = ref.get(stampHolder);
int currentStamp = stampHolder[0];

// CAS 更新，同时检查版本号
boolean success = ref.compareAndSet(
    currentValue,    // 期望值
    200,             // 新值
    currentStamp,    // 期望版本号
    currentStamp + 1 // 新版本号
);
```

---

## 六、CAS 的问题与解决

### 6.1 ABA 问题

**问题：** 线程1 读取值 A，线程2 将 A→B→A，线程1 CAS 时发现仍是 A，认为没有变化，实际已被修改过。

**解决：** 使用 `AtomicStampedReference` 加版本号，或 `AtomicMarkableReference` 加标记位。

```java
// 典型 ABA 场景（链表头节点操作）
// 线程1：读取 head=A，准备 CAS(A→C)
// 线程2：将 A 移除（A→B），又将 A 加回（B→A）
// 线程1：CAS 成功，但链表结构已变，可能引发内存问题
```

### 6.2 自旋开销问题

**问题：** CAS 失败后持续自旋，消耗 CPU。

**解决：**
- 限制自旋次数
- 使用 `LongAdder` 分散竞争
- 竞争激烈时退化为悲观锁（synchronized）

### 6.3 只能保证单个变量的原子性

**问题：** CAS 只能保证一个变量的原子操作，多个变量无法保证。

**解决：**
- 使用 `AtomicReference` 将多个变量封装为一个对象
- 使用 `synchronized` 或 `Lock`

```java
// 将 x 和 y 封装成一个 Point 对象用 AtomicReference 保护
AtomicReference<Point> pointRef = new AtomicReference<>(new Point(0, 0));
Point oldPoint = pointRef.get();
pointRef.compareAndSet(oldPoint, new Point(oldPoint.x + 1, oldPoint.y + 1));
```

---

## 七、大厂常见面试题

### Q1：volatile 能保证线程安全吗？

**答：**

**不能完全保证线程安全。** volatile 只提供了**可见性**和**禁止指令重排序**，但不能保证**原子性**。

典型反例：`count++` 是三步操作（读-加-写），即使 count 是 volatile，多线程执行仍然会出现竞态条件，导致结果不准确。

**能用 volatile 的场景：**
- 写操作不依赖当前值（如 `flag = false`）
- 只有一个线程写，多个线程读（如状态标志位）
- 双重检查锁（DCL）中防止对象半初始化

---

### Q2：CAS 的 ABA 问题是什么？如何解决？

**答：**

**ABA 问题：** 线程1 准备 CAS 更新变量 V（期望值 A），此时线程2 先将 V 从 A 改为 B，再改回 A。线程1 执行 CAS 时，V 的当前值仍是 A，认为没有变化，CAS 成功，但实际上 V 已经被修改过，可能导致逻辑错误（如链表节点复用引发内存问题）。

**解决方案：**
1. `AtomicStampedReference`：每次 CAS 时携带版本号（stamp），A→B→A 后版本号从 0→1→2，虽值相同但版本号不同，CAS 失败。
2. `AtomicMarkableReference`：用布尔标记代替版本号，只区分"是否被修改过"。

---

### Q3：volatile 是如何实现内存可见性的？

**答：**

在 JVM 层面，volatile 写操作会在前后插入 `StoreStore` 和 `StoreLoad` 内存屏障，volatile 读操作会在后面插入 `LoadLoad` 和 `LoadStore` 屏障，防止指令重排序。

在硬件层面（x86），volatile 写会生成 `lock` 前缀指令，该指令触发 MESI 缓存一致性协议：
1. 将当前 CPU 缓存行刷新到主内存
2. 其他 CPU 的缓存中，该内存地址对应的缓存行标记为**失效（Invalid）**
3. 其他 CPU 下次读取该变量时，必须从主内存重新加载

---

### Q4：LongAdder 为什么比 AtomicLong 性能高？

**答：**

- `AtomicLong` 使用一个 `value` 字段，多线程都对该字段进行 CAS，竞争激烈时大量线程自旋重试，白白消耗 CPU。
- `LongAdder` 内部维护一个 `base` 字段和一个 `Cell[]` 数组（每个 Cell 有独立的 `value`）。无竞争时直接 CAS 更新 `base`；有竞争时，不同线程 hash 到不同的 Cell 上，各自累加，大幅减少冲突。最终求和时将 `base + Σcells[i]`。

这是一种**分散热点**的思想，以空间换时间，高并发吞吐量通常比 AtomicLong 高 4~8 倍。但 `LongAdder.sum()` 返回的不是精确的实时值，适合统计计数，不适合需要精确值的 CAS 场景。

---

### Q5：双重检查锁（DCL）中 volatile 的必要性？

**答：**

对象创建（`new Singleton()`）在字节码层面分为三步：
1. 分配内存空间
2. 初始化对象（执行构造方法）
3. 将引用赋值给变量

JIT 编译和 CPU 可能将步骤 2 和 3 **重排序**为：1→3→2。  

在没有 volatile 的情况下：
- 线程A 执行到步骤3（引用已赋值但对象未初始化），被挂起
- 线程B 在第一次检查 `instance != null`，直接返回**未初始化的 instance**，使用时会出现空指针或数据异常

加上 `volatile` 后，步骤3 有 `StoreStore` 屏障，禁止了 2 和 3 的重排序，保证其他线程读到的 instance 一定是完整初始化过的对象。

---

*参考资料：《深入理解Java虚拟机》周志明、JSR-133 Java Memory Model、Doug Lea 并发编程文章*
