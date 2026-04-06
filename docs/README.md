# Java 基础与进阶学习资料

> 面向大厂 Java 开发工程师面试备考，深度覆盖 **Java 并发** 与 **JVM** 两大核心模块。

---

## 目录结构

```
docs/
├── phase1-java-concurrent/            # Java 并发
│   ├── 01-线程基础与线程池.md
│   ├── 02-synchronized与Lock.md
│   ├── 03-volatile与CAS.md
│   ├── 04-AQS原理.md
│   ├── 05-并发工具类.md
│   ├── 06-CompletableFuture与异步编程.md
│   └── 07-并发常见面试题.md
├── phase1-jvm/                        # JVM
│   ├── 01-JVM内存模型.md
│   ├── 02-垃圾回收算法与收集器.md
│   ├── 03-类加载机制.md
│   ├── 04-JVM调优实战.md
│   ├── 05-JIT编译与优化.md
│   └── 06-JVM常见面试题.md
└── README.md                          # 本文件
```

---

## 模块一：Java 并发

### 主要内容

Java 并发是大厂面试必考核心，本模块从线程基础到 AQS 底层，覆盖以下知识体系：

| 文件 | 核心内容 |
|------|---------|
| [01-线程基础与线程池](./phase1-java-concurrent/01-线程基础与线程池.md) | 线程生命周期、创建方式、ThreadPoolExecutor 七大参数、线程池工作流程、线程数配置公式 |
| [02-synchronized与Lock](./phase1-java-concurrent/02-synchronized与Lock.md) | synchronized 三种用法、Monitor 原理、锁升级（偏向→轻量→重量）、ReentrantLock、Condition、ReadWriteLock |
| [03-volatile与CAS](./phase1-java-concurrent/03-volatile与CAS.md) | JMM 三大特性、Happens-Before 规则、volatile 内存屏障、CAS 原子指令、AtomicInteger/LongAdder |
| [04-AQS原理](./phase1-java-concurrent/04-AQS原理.md) | AQS state 变量、CLH 双向队列、独占/共享锁获取释放流程、ConditionObject、自定义同步器 |
| [05-并发工具类](./phase1-java-concurrent/05-并发工具类.md) | CountDownLatch、CyclicBarrier、Semaphore、Exchanger、BlockingQueue、ConcurrentHashMap JDK 8 原理 |
| [06-CompletableFuture与异步编程](./phase1-java-concurrent/06-CompletableFuture与异步编程.md) | Future 局限、CompletableFuture 全 API、任务组合（allOf/anyOf）、异常处理、线程池最佳实践、实战案例 |
| [07-并发常见面试题](./phase1-java-concurrent/07-并发常见面试题.md) | 20+ 高频面试题汇总（线程基础、synchronized、volatile、CAS、AQS、线程池、并发容器） |

### 知识体系图

```
Java 并发
├── 内存模型
│   ├── JMM（主内存/工作内存）
│   ├── Happens-Before
│   └── volatile（可见性 + 有序性）
│
├── 锁机制
│   ├── synchronized（Monitor + 锁升级）
│   ├── ReentrantLock（AQS + 公平/非公平）
│   ├── ReadWriteLock
│   └── StampedLock（乐观读）
│
├── 无锁并发
│   ├── CAS（CPU 原子指令）
│   ├── Atomic* 原子类
│   └── LongAdder（分段竞争）
│
├── AQS 框架
│   ├── state 变量
│   ├── CLH 等待队列
│   ├── 独占模式（ReentrantLock）
│   └── 共享模式（Semaphore/CountDownLatch）
│
├── 并发工具
│   ├── CountDownLatch
│   ├── CyclicBarrier
│   ├── Semaphore
│   └── BlockingQueue
│
├── 线程池
│   ├── ThreadPoolExecutor
│   ├── 线程数配置
│   └── 动态线程池
│
└── 异步编程
    ├── CompletableFuture
    └── 响应式编程
```

---

## 模块二：JVM

### 主要内容

JVM 是 Java 工程师的必备深度知识，本模块从内存模型到 GC 调优，全面覆盖：

| 文件 | 核心内容 |
|------|---------|
| [01-JVM内存模型](./phase1-jvm/01-JVM内存模型.md) | 运行时数据区（程序计数器/虚拟机栈/堆/方法区）、堆结构（Eden/Survivor/Old）、对象内存布局、对象创建过程 |
| [02-垃圾回收算法与收集器](./phase1-jvm/02-垃圾回收算法与收集器.md) | GC Roots 可达性分析、四种引用类型、标记-清除/复制/整理算法、Serial/CMS/G1/ZGC 收集器对比 |
| [03-类加载机制](./phase1-jvm/03-类加载机制.md) | 类生命周期、加载/验证/准备/解析/初始化五阶段、类加载器层次、双亲委派模型、打破双亲委派（Tomcat/SPI/OSGi） |
| [04-JVM调优实战](./phase1-jvm/04-JVM调优实战.md) | JVM 调优思路、常用参数大全、GC 日志分析、jstat/jmap/jstack/Arthas 工具使用、内存泄漏/CPU 飙升/Full GC 排查实战 |
| [05-JIT编译与优化](./phase1-jvm/05-JIT编译与优化.md) | 解释执行 vs JIT、C1/C2 分层编译、热点代码检测、方法内联/逃逸分析/锁消除/向量化优化、AOT/GraalVM Native Image |
| [06-JVM常见面试题](./phase1-jvm/06-JVM常见面试题.md) | 20+ 高频面试题汇总（内存结构、GC、类加载、JIT、调优诊断） |

### 知识体系图

```
JVM
├── 内存结构
│   ├── 线程私有：程序计数器/虚拟机栈/本地方法栈
│   └── 线程共享：堆（Eden/S0/S1/Old）/元空间
│
├── 垃圾回收
│   ├── 对象存活判断（可达性分析 + GC Roots）
│   ├── GC 算法（标记-清除/复制/整理）
│   ├── 分代收集（新生代/老年代）
│   └── 收集器（Serial/ParNew/CMS/G1/ZGC）
│
├── 类加载
│   ├── 加载→验证→准备→解析→初始化
│   ├── ClassLoader 层次
│   ├── 双亲委派模型
│   └── 打破双亲委派（Tomcat/SPI）
│
├── JIT 编译
│   ├── 分层编译（C1/C2）
│   ├── 方法内联
│   ├── 逃逸分析（栈上分配/标量替换/锁消除）
│   └── AOT（GraalVM Native Image）
│
└── 调优实战
    ├── 常用 JVM 参数
    ├── GC 日志分析
    ├── 内存泄漏排查
    ├── CPU 飙升排查
    └── Full GC 排查
```

---

## 建议学习顺序

### 第一周：并发基础

```
Day 1: 01-线程基础与线程池.md        → 搞清楚线程池七大参数和工作流程
Day 2: 02-synchronized与Lock.md    → 理解锁升级机制和 ReentrantLock
Day 3: 03-volatile与CAS.md         → 搞透 JMM、volatile、CAS 三者关系
Day 4: 04-AQS原理.md                → 读懂 AQS 源码，理解 acquire/release 流程
Day 5-6: 05-并发工具类.md           → 熟练使用并理解实现原理
Day 7: 06-CompletableFuture.md      → 掌握异步编程范式，实战联系
```

### 第二周：JVM 深度

```
Day 1: 01-JVM内存模型.md            → 画出 JVM 内存分区图，能手写对象布局
Day 2: 02-垃圾回收算法与收集器.md   → 理解 G1 Region 模型和分阶段回收
Day 3: 03-类加载机制.md              → 理解双亲委派并能说明打破场景
Day 4: 04-JVM调优实战.md            → 动手实践 jstat/jmap/Arthas 工具
Day 5: 05-JIT编译与优化.md          → 理解逃逸分析对性能的影响
Day 6-7: 07/06 面试题汇总           → 查漏补缺，重点题反复练习
```

### 学习方法

1. **先理论后实践**：每个知识点先读文档理解原理，再用 JDK 工具验证
2. **画图辅助记忆**：JVM 内存结构、锁升级、AQS 队列等适合画图记忆
3. **代码敲一遍**：文档中的代码示例自己敲一遍，观察运行结果
4. **面试题配合复习**：每读完一个模块，对照面试题检验掌握程度
5. **结合项目经验**：思考在自己项目中用过哪些并发工具，出现过哪些 JVM 问题

---

## 大厂面试高频考点

### Java 并发 TOP 10

1. ⭐⭐⭐ 线程池七大参数和工作流程
2. ⭐⭐⭐ synchronized 锁升级（偏向→轻量→重量）
3. ⭐⭐⭐ volatile 的作用和底层原理（内存屏障）
4. ⭐⭐⭐ AQS 核心原理（state + CLH 队列）
5. ⭐⭐⭐ ConcurrentHashMap JDK 8 的实现原理
6. ⭐⭐ CAS 的原理和三大问题（ABA/自旋/单变量）
7. ⭐⭐ ReentrantLock 公平/非公平锁实现
8. ⭐⭐ CountDownLatch vs CyclicBarrier
9. ⭐⭐ CompletableFuture 的常用 API 和线程池配置
10. ⭐ ThreadLocal 原理和内存泄漏

### JVM TOP 10

1. ⭐⭐⭐ JVM 内存分区（堆/栈/方法区）
2. ⭐⭐⭐ GC 算法和主流收集器（G1 为重点）
3. ⭐⭐⭐ Full GC 触发条件和避免方法
4. ⭐⭐⭐ 双亲委派模型及打破场景（Tomcat/SPI）
5. ⭐⭐⭐ 内存泄漏/OOM 排查步骤
6. ⭐⭐ 类加载五个阶段
7. ⭐⭐ G1 收集器原理（Region + 可预测停顿）
8. ⭐⭐ 逃逸分析的三种优化
9. ⭐⭐ jstack/jmap/Arthas 工具实战
10. ⭐ ZGC/Shenandoah 低延迟收集器

---

## 参考资源

### 📚 经典书籍

| 书名 | 适用 | 推荐程度 |
|------|------|---------|
| 《深入理解Java虚拟机》第三版（周志明） | JVM 全面深入 | ⭐⭐⭐⭐⭐ |
| 《Java并发编程实战》（Brian Goetz） | 并发编程权威 | ⭐⭐⭐⭐⭐ |
| 《Java并发编程的艺术》（方腾飞） | 并发编程 + AQS 源码 | ⭐⭐⭐⭐ |
| 《Effective Java》第三版（Josh Bloch） | Java 最佳实践 | ⭐⭐⭐⭐⭐ |

### 🌐 推荐网站

| 网站 | 内容 | 链接 |
|------|------|------|
| Oracle JDK 文档 | JDK API 官方文档 | https://docs.oracle.com/en/java/javase/ |
| OpenJDK | JVM 源码 | https://github.com/openjdk/jdk |
| 美团技术博客 | Java 并发/JVM 深度文章 | https://tech.meituan.com |
| InfoQ | 技术深度文章 | https://www.infoq.cn |
| GCEasy | 在线 GC 日志分析 | https://gceasy.io |
| Arthas 官方文档 | JVM 诊断工具 | https://arthas.aliyun.com |
| JVM Internals | JVM 原理图文解析 | http://blog.jamesdbloom.com |
| 并发编程网 | ifeve.com 并发专项 | http://ifeve.com |

### 🎥 视频资源

- **尚硅谷 JVM 教程**（宋红康）：B站搜索，内容全面
- **黑马程序员 Java 并发**：B站搜索，案例丰富
- **极客时间《深入拆解Java虚拟机》**（郑雨迪）：付费，深度好

### 🔧 工具推荐

| 工具 | 用途 | 获取方式 |
|------|------|---------|
| **Arthas** | 线上诊断、热更新 | `curl -O https://alibaba.github.io/arthas/arthas-boot.jar` |
| **MAT** | Heap Dump 分析 | Eclipse Memory Analyzer |
| **GCViewer** | GC 日志可视化 | GitHub: chewiebug/GCViewer |
| **JProfiler** | 商业 JVM 性能分析 | ej-technologies.com |
| **VisualVM** | JVM 监控（免费） | JDK 自带或单独下载 |
| **JITWatch** | JIT 编译日志分析 | GitHub: AdoptOpenJDK/jitwatch |

---

## 学习进度追踪

使用以下 Checklist 追踪学习进度：

### Java 并发

- [ ] 线程生命周期、创建方式（会画状态图）
- [ ] ThreadPoolExecutor 七大参数（能背出来）
- [ ] synchronized 锁升级（能画升级路径图）
- [ ] volatile 内存屏障（能解释 DCL 为何需要 volatile）
- [ ] AQS 独占锁加/解锁流程（能描述 Node 入队出队）
- [ ] CAS ABA 问题及 AtomicStampedReference 解决
- [ ] CountDownLatch vs CyclicBarrier（使用场景对比）
- [ ] ConcurrentHashMap JDK 8 put 流程
- [ ] CompletableFuture thenApply vs thenCompose
- [ ] ThreadLocal 内存泄漏原因及预防

### JVM

- [ ] JVM 内存分区（画图，不看文档）
- [ ] 对象内存布局（Mark Word 各状态含义）
- [ ] 可达性分析 + GC Roots 包含哪些
- [ ] 标记-清除/复制/整理算法优缺点
- [ ] G1 Region 模型和 Mixed GC 流程
- [ ] Full GC 5 个触发条件
- [ ] 类加载 5 个阶段（准备阶段赋零值还是代码值）
- [ ] 双亲委派模型（Tomcat 为什么打破）
- [ ] 逃逸分析三种优化（栈上分配/标量替换/锁消除）
- [ ] 线上 CPU 100% 排查四步法（能背出命令）
- [ ] 内存泄漏排查步骤（jmap + MAT）

---

*持续更新中，后续将添加 MySQL、Redis、Spring、分布式系统等模块。*
