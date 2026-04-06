# Java 开发工程师学习资料

> 面向大厂 Java 开发工程师面试备考，深度覆盖 **Java 并发**、**JVM**、**Spring**、**ORM**、**数据库** 五大核心模块。

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
├── phase1-spring/                     # Spring 全家桶
│   ├── 01-Spring-IoC与AOP.md
│   ├── 02-SpringMVC与Web开发.md
│   ├── 03-SpringBoot核心.md
│   ├── 04-SpringCloud微服务.md
│   ├── 05-Spring事务管理.md
│   └── 06-Spring常见面试题.md
├── phase1-orm/                        # ORM 框架
│   ├── 01-MyBatis核心原理.md
│   ├── 02-MyBatis-Plus实战.md
│   └── 03-ORM常见面试题.md
├── phase1-database/                   # 数据库（MySQL + Redis）
│   ├── 01-MySQL索引深入.md
│   ├── 02-MySQL事务与锁.md
│   ├── 03-SQL优化实战.md
│   ├── 04-分库分表.md
│   ├── 05-Redis基础与实战.md
│   └── 06-数据库常见面试题.md
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

## 模块三：Spring 全家桶

### 主要内容

Spring 是 Java 后端开发的核心框架，本模块从 IoC/AOP 到微服务全面覆盖：

| 文件 | 核心内容 |
|------|---------|
| [01-Spring IoC与AOP](./phase1-spring/01-Spring-IoC与AOP.md) | IoC 容器、Bean 生命周期、依赖注入、AOP 原理（JDK/CGLIB 代理）、三级缓存解决循环依赖 |
| [02-SpringMVC与Web开发](./phase1-spring/02-SpringMVC与Web开发.md) | DispatcherServlet 流程、常用注解、参数校验、全局异常处理、拦截器、RESTful API |
| [03-SpringBoot核心](./phase1-spring/03-SpringBoot核心.md) | 自动配置原理、Starter 机制、条件注解、配置管理、内嵌容器、Actuator 监控 |
| [04-SpringCloud微服务](./phase1-spring/04-SpringCloud微服务.md) | Nacos 注册中心、OpenFeign 远程调用、Gateway 网关、Sentinel 熔断限流、配置中心 |
| [05-Spring事务管理](./phase1-spring/05-Spring事务管理.md) | 声明式事务原理、七种传播行为、隔离级别、事务失效场景、分布式事务（Seata） |
| [06-Spring常见面试题](./phase1-spring/06-Spring常见面试题.md) | 20+ 高频面试题汇总（IoC/AOP、MVC、Boot、Cloud、事务） |

### 知识体系图

```
Spring
├── IoC / DI
│   ├── BeanFactory / ApplicationContext
│   ├── Bean 生命周期
│   ├── 依赖注入（构造器/Setter/字段）
│   └── 循环依赖（三级缓存）
│
├── AOP
│   ├── JDK 动态代理 / CGLIB
│   ├── 五种通知类型
│   └── 切入点表达式
│
├── Spring MVC
│   ├── DispatcherServlet 处理流程
│   ├── 参数解析与数据校验
│   └── 拦截器 vs 过滤器
│
├── Spring Boot
│   ├── 自动配置原理
│   ├── Starter 机制
│   └── Actuator 监控
│
├── Spring Cloud
│   ├── Nacos（注册中心 + 配置中心）
│   ├── OpenFeign + LoadBalancer
│   ├── Gateway 网关
│   └── Sentinel 熔断限流
│
└── 事务管理
    ├── 声明式事务（@Transactional）
    ├── 传播行为（REQUIRED/REQUIRES_NEW/NESTED）
    ├── 事务失效场景
    └── 分布式事务（Seata）
```

---

## 模块四：ORM 框架

### 主要内容

ORM 是数据持久层的核心，本模块覆盖 MyBatis 和 MyBatis-Plus：

| 文件 | 核心内容 |
|------|---------|
| [01-MyBatis核心原理](./phase1-orm/01-MyBatis核心原理.md) | 核心架构、SQL 执行流程、MapperProxy 动态代理、结果映射、动态 SQL、缓存机制、插件 |
| [02-MyBatis-Plus实战](./phase1-orm/02-MyBatis-Plus实战.md) | 通用 CRUD、LambdaQueryWrapper、分页插件、代码生成器、自动填充、逻辑删除、乐观锁 |
| [03-ORM常见面试题](./phase1-orm/03-ORM常见面试题.md) | 15+ 高频面试题汇总（MyBatis 原理、缓存、#{}/${}、MyBatis-Plus、N+1 问题） |

### 知识体系图

```
ORM 框架
├── MyBatis
│   ├── 核心组件（SqlSession/Executor/MappedStatement）
│   ├── MapperProxy 动态代理
│   ├── #{}（预编译）vs ${}（字符串替换）
│   ├── 动态 SQL（if/where/foreach）
│   ├── 结果映射（resultMap/association/collection）
│   └── 缓存（一级缓存/二级缓存）
│
└── MyBatis-Plus
    ├── BaseMapper / IService
    ├── LambdaQueryWrapper
    ├── 分页插件
    ├── 自动填充
    ├── 逻辑删除
    └── 乐观锁
```

---

## 模块五：数据库（MySQL + Redis）

### 主要内容

数据库是后端开发的根基，本模块覆盖 MySQL 核心与 Redis 实战：

| 文件 | 核心内容 |
|------|---------|
| [01-MySQL索引深入](./phase1-database/01-MySQL索引深入.md) | B+ 树原理、聚簇/非聚簇索引、回表、覆盖索引、索引下推、最左前缀、索引失效场景、EXPLAIN |
| [02-MySQL事务与锁](./phase1-database/02-MySQL事务与锁.md) | ACID、redo/undo log、隔离级别、MVCC 原理、ReadView、行锁（Record/Gap/Next-Key）、死锁 |
| [03-SQL优化实战](./phase1-database/03-SQL优化实战.md) | 慢查询定位、EXPLAIN 分析、SELECT/INSERT/JOIN/分页优化、深分页解决方案 |
| [04-分库分表](./phase1-database/04-分库分表.md) | 垂直/水平拆分、分片算法、ShardingSphere、分布式 ID（雪花算法）、跨片查询、数据迁移 |
| [05-Redis基础与实战](./phase1-database/05-Redis基础与实战.md) | 五大数据类型、持久化（RDB/AOF）、主从/哨兵/Cluster、缓存穿透/击穿/雪崩、分布式锁 |
| [06-数据库常见面试题](./phase1-database/06-数据库常见面试题.md) | 25+ 高频面试题汇总（MySQL 索引、事务锁、SQL 优化、分库分表、Redis） |

### 知识体系图

```
数据库
├── MySQL
│   ├── 索引
│   │   ├── B+ 树结构（为什么不用 B 树/红黑树）
│   │   ├── 聚簇索引 vs 非聚簇索引（回表）
│   │   ├── 覆盖索引 / 索引下推
│   │   ├── 最左前缀原则
│   │   └── 索引失效场景
│   │
│   ├── 事务与锁
│   │   ├── ACID（redo log / undo log）
│   │   ├── 隔离级别（RR 默认）
│   │   ├── MVCC（ReadView + 版本链）
│   │   ├── 行锁（Record/Gap/Next-Key Lock）
│   │   └── 死锁处理
│   │
│   ├── SQL 优化
│   │   ├── 慢查询日志
│   │   ├── EXPLAIN 执行计划
│   │   ├── JOIN 优化
│   │   └── 深分页优化
│   │
│   └── 分库分表
│       ├── 垂直/水平拆分
│       ├── 分片算法
│       ├── 分布式 ID（雪花算法）
│       └── ShardingSphere
│
└── Redis
    ├── 数据结构（String/Hash/List/Set/ZSet）
    ├── 持久化（RDB/AOF/混合）
    ├── 高可用（主从/哨兵/Cluster）
    ├── 缓存策略（穿透/击穿/雪崩）
    ├── 缓存一致性（Cache Aside）
    └── 分布式锁（Redisson）
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

### 第三周：Spring 全家桶

```
Day 1: 01-Spring-IoC与AOP.md       → 搞懂 Bean 生命周期和三级缓存
Day 2: 02-SpringMVC与Web开发.md    → 理解 DispatcherServlet 处理流程
Day 3: 03-SpringBoot核心.md         → 掌握自动配置原理和 Starter 机制
Day 4: 04-SpringCloud微服务.md      → 了解 Nacos/Gateway/Sentinel 核心组件
Day 5: 05-Spring事务管理.md         → 掌握传播行为和事务失效场景
Day 6-7: 06-Spring面试题汇总        → 查漏补缺，模拟面试
```

### 第四周：ORM + 数据库

```
Day 1: 01-MyBatis核心原理.md        → 理解 MapperProxy 动态代理和 SQL 执行流程
Day 2: 02-MyBatis-Plus实战.md       → 掌握 LambdaQueryWrapper 和常用功能
Day 3: 01-MySQL索引深入.md          → 搞透 B+ 树和索引失效场景
Day 4: 02-MySQL事务与锁.md          → 理解 MVCC 原理和 Next-Key Lock
Day 5: 03-SQL优化实战.md + 04-分库分表.md → EXPLAIN 实战和分库分表方案
Day 6: 05-Redis基础与实战.md        → 掌握缓存三大问题和分布式锁
Day 7: 面试题汇总                    → 数据库 + ORM 面试题查漏补缺
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

### Spring TOP 10

1. ⭐⭐⭐ Spring Bean 生命周期
2. ⭐⭐⭐ Spring AOP 实现原理（JDK/CGLIB 代理）
3. ⭐⭐⭐ Spring 循环依赖与三级缓存
4. ⭐⭐⭐ Spring Boot 自动配置原理
5. ⭐⭐⭐ @Transactional 事务失效场景
6. ⭐⭐ Spring MVC 请求处理流程
7. ⭐⭐ 事务传播行为（REQUIRED/REQUIRES_NEW/NESTED）
8. ⭐⭐ Nacos vs Eureka 注册中心对比
9. ⭐⭐ Sentinel 熔断限流原理
10. ⭐ Spring 中的设计模式

### 数据库 TOP 10

1. ⭐⭐⭐ MySQL B+ 树索引原理
2. ⭐⭐⭐ MVCC 实现原理（ReadView + 版本链）
3. ⭐⭐⭐ 索引失效场景
4. ⭐⭐⭐ Redis 缓存穿透/击穿/雪崩
5. ⭐⭐⭐ 缓存与数据库一致性方案
6. ⭐⭐ InnoDB 行锁（Record/Gap/Next-Key Lock）
7. ⭐⭐ SQL 优化与 EXPLAIN 分析
8. ⭐⭐ Redis 分布式锁实现
9. ⭐⭐ 分库分表方案与分片策略
10. ⭐ Redis Cluster 原理

---

## 参考资源

### 📚 经典书籍

| 书名 | 适用 | 推荐程度 |
|------|------|---------|
| 《深入理解Java虚拟机》第三版（周志明） | JVM 全面深入 | ⭐⭐⭐⭐⭐ |
| 《Java并发编程实战》（Brian Goetz） | 并发编程权威 | ⭐⭐⭐⭐⭐ |
| 《Java并发编程的艺术》（方腾飞） | 并发编程 + AQS 源码 | ⭐⭐⭐⭐ |
| 《Effective Java》第三版（Josh Bloch） | Java 最佳实践 | ⭐⭐⭐⭐⭐ |
| 《高性能MySQL》第四版 | MySQL 全面深入 | ⭐⭐⭐⭐⭐ |
| 《Redis 设计与实现》（黄健宏） | Redis 底层原理 | ⭐⭐⭐⭐ |
| 《Spring 实战》第六版 | Spring Boot/Cloud | ⭐⭐⭐⭐ |
| 《MyBatis 技术内幕》（徐郡明） | MyBatis 源码解析 | ⭐⭐⭐ |

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

### Spring

- [ ] Bean 生命周期核心步骤（能说出 8 步）
- [ ] AOP 代理方式（JDK vs CGLIB，Spring Boot 默认用哪个）
- [ ] 三级缓存解决循环依赖的流程
- [ ] Spring MVC 请求处理流程（DispatcherServlet 核心流程）
- [ ] Spring Boot 自动配置原理（能说出 4 步）
- [ ] @Transactional 七种传播行为（重点 REQUIRED/REQUIRES_NEW/NESTED）
- [ ] 事务失效 8 大场景
- [ ] Nacos 注册中心原理
- [ ] Sentinel 限流和熔断的区别
- [ ] Spring 中用到的设计模式（至少说出 5 种）

### ORM

- [ ] MyBatis 执行流程（MapperProxy 动态代理原理）
- [ ] #{} 和 ${} 的区别（安全性和使用场景）
- [ ] MyBatis 一级缓存和二级缓存的区别
- [ ] MyBatis-Plus LambdaQueryWrapper 用法
- [ ] 乐观锁实现原理（@Version）
- [ ] N+1 问题及解决方案

### 数据库

- [ ] B+ 树结构特点（和 B 树的 3 个区别）
- [ ] 聚簇索引 vs 非聚簇索引（什么是回表）
- [ ] 覆盖索引概念和应用
- [ ] 索引失效 6 大场景
- [ ] MVCC 原理（ReadView 可见性规则）
- [ ] InnoDB 三种行锁（Record/Gap/Next-Key）
- [ ] 深分页优化方案（延迟关联/游标分页）
- [ ] Redis 缓存穿透/击穿/雪崩的解决方案
- [ ] Redis 分布式锁实现（SETNX + Lua 脚本）
- [ ] 缓存与数据库一致性方案（Cache Aside 模式）

---

*持续更新中，后续将添加分布式系统、消息队列、设计模式等模块。*
