# 04 - JVM 调优实战

## 目录

1. [JVM 调优思路](#一jvm-调优思路)
2. [常用 JVM 参数](#二常用-jvm-参数)
3. [GC 日志分析](#三gc-日志分析)
4. [常用诊断工具](#四常用诊断工具)
5. [内存泄漏排查实战](#五内存泄漏排查实战)
6. [CPU 飙升排查实战](#六cpu-飙升排查实战)
7. [频繁 Full GC 排查实战](#七频繁-full-gc-排查实战)
8. [线上调优最佳实践](#八线上调优最佳实践)
9. [大厂常见面试题](#九大厂常见面试题)

---

## 一、JVM 调优思路

调优的本质是**在有限的资源内最大化应用吞吐量，同时控制 GC 停顿时间**。

### 1.1 调优前的准备

```
① 明确调优目标：
   - 吞吐量优先（批处理）→ Parallel GC
   - 低延迟优先（Web服务）→ G1 / ZGC
   - 内存占用优先（嵌入式）→ Serial GC

② 建立基线监控：
   - GC 频率和停顿时间（jstat / GC日志）
   - 堆内存使用趋势（监控系统）
   - CPU 使用率和线程状态（jstack / arthas）

③ 分析问题根因（不要凭经验猜测）
④ 针对性调整一个参数，观察效果
⑤ 回归验证（压测）
```

### 1.2 调优层次

```
代码层  → 减少对象创建（对象池/复用）、减少大对象、优化数据结构
      ↓
JVM层  → 调整堆大小、新老代比例、GC 收集器、GC 参数
      ↓
OS层   → 大页内存（HugePage）、NUMA 感知、CPU 亲和性
```

---

## 二、常用 JVM 参数

### 2.1 堆内存参数

```bash
# 堆大小（建议 Xms=Xmx，避免动态扩容 GC）
-Xms4g -Xmx4g

# 新生代大小（或使用比例）
-Xmn2g
-XX:NewRatio=2        # 老年代:新生代 = 2:1
-XX:SurvivorRatio=8   # Eden:S0:S1 = 8:1:1

# 栈大小
-Xss512k

# 元空间（JDK 8+）
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
```

### 2.2 GC 收集器参数

```bash
# G1（推荐）
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45
-XX:G1HeapRegionSize=16m
-XX:G1ReservePercent=10    # 预留10%应对晋升失败

# ZGC（JDK 15+，超低延迟）
-XX:+UseZGC
-XX:+ZGenerational         # JDK 21+ 分代ZGC

# Parallel（吞吐优先）
-XX:+UseParallelGC
-XX:GCTimeRatio=99         # GC时间不超过1%
-XX:MaxGCPauseMillis=100
```

### 2.3 GC 日志参数

```bash
# JDK 9+（统一日志）
-Xlog:gc*=info:file=/logs/gc.log:time,uptime,pid,level,tags:filecount=10,filesize=100m

# JDK 8
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintGCCause
-Xloggc:/logs/gc.log
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=10
-XX:GCLogFileSize=100m
```

### 2.4 故障诊断参数

```bash
# OOM 时自动 Heap Dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/heapdump.hprof

# 打印 OOM 时的 GC 信息
-XX:+PrintHeapAtGC

# 退出时打印 JVM 标志
-XX:+PrintFlagsFinal

# 禁用显式 GC（System.gc()）
-XX:+DisableExplicitGC

# 大页内存（Linux）
-XX:+UseLargePages
```

---

## 三、GC 日志分析

### 3.1 JDK 8 GC 日志示例

```
# Minor GC
2024-01-15T10:00:01.234+0800: 1234.567: [GC (Allocation Failure)
    [PSYoungGen: 524288K->65536K(611328K)]
    ↑已使用→GC后            ↑新生代容量
    1048576K->589824K(2097152K), 0.0234567 secs]
    ↑堆总用→GC后              ↑堆总容量    ↑耗时
    [Times: user=0.09 sys=0.01, real=0.02 secs]

# Full GC
2024-01-15T10:01:05.678+0800: [Full GC (Ergonomics)
    [PSYoungGen: 65536K->0K(611328K)]
    [ParOldGen: 1048576K->786432K(1485824K)]
    1114112K->786432K(2097152K),
    [Metaspace: 32768K->32768K(1081344K)],
    1.2345678 secs]
```

### 3.2 G1 GC 日志示例（JDK 9+）

```
[2.345s][info][gc,start    ] GC(1) Pause Young (Normal) (G1 Evacuation Pause)
[2.356s][info][gc,heap     ] GC(1) Eden regions: 200->0(200)
[2.356s][info][gc,heap     ] GC(1) Survivor regions: 25->30(40)
[2.356s][info][gc,heap     ] GC(1) Old regions: 100->100
[2.356s][info][gc          ] GC(1) Pause Young (Normal) (G1 Evacuation Pause) 1234M->567M(4096M) 11.234ms
```

### 3.3 关键指标解读

| 指标 | 含义 | 告警阈值 |
|------|------|---------|
| YGC 频率 | 每分钟 Minor GC 次数 | > 60次/分 需关注 |
| YGC 停顿 | 每次 Minor GC 时间 | > 100ms 需调优 |
| Full GC 频率 | 每天 Full GC 次数 | > 1次/天 需调优 |
| Full GC 停顿 | Full GC 时间 | > 1s 需调优 |
| 老年代占用率 | Old Gen 占最大堆比例 | > 80% 告警 |

### 3.4 GC 分析工具

```bash
# GCViewer - 图形化 GC 日志分析
# GCEasy - 在线 GC 日志分析（https://gceasy.io）
# Garbagecat - 命令行 GC 日志分析

# jstat 实时监控
jstat -gcutil <pid> 1000    # 每秒输出一次
# 输出示例：
#  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
#  0.00  10.00  80.00  40.00  95.00  90.00     100    5.000     1    2.000    7.000
```

---

## 四、常用诊断工具

### 4.1 JDK 内置工具

```bash
# jps - 查看 JVM 进程
jps -lv

# jstack - 线程快照（死锁、CPU 飙升排查）
jstack -l <pid>
jstack -l <pid> > /tmp/thread_dump.txt

# jmap - 堆内存快照
jmap -heap <pid>                    # 堆内存概览
jmap -histo <pid> | head -30        # 堆对象统计（按数量排序）
jmap -dump:format=b,file=/tmp/heap.hprof <pid>  # Heap Dump

# jstat - GC 统计
jstat -gcutil <pid> 1000 20   # 每隔1秒打印，共20次
jstat -gccause <pid>          # 包含 GC cause

# jinfo - JVM 参数
jinfo -flags <pid>            # 查看所有 JVM 参数
jinfo -flag MaxHeapSize <pid> # 查看单个参数

# jcmd - 功能强大的多合一工具
jcmd <pid> help               # 查看所有命令
jcmd <pid> VM.flags           # 查看 JVM 参数
jcmd <pid> GC.run             # 触发 GC
jcmd <pid> Thread.print       # 线程快照
jcmd <pid> GC.heap_dump /tmp/heap.hprof
```

### 4.2 Arthas（推荐，阿里开源）

```bash
# 下载并启动 Arthas
curl -O https://alibaba.github.io/arthas/arthas-boot.jar
java -jar arthas-boot.jar <pid>

# 常用命令
dashboard          # 实时查看进程内存/线程/GC 概览
thread             # 线程状态统计
thread -b          # 查找死锁线程
thread -n 5        # 最消耗CPU的5个线程
jvm                # JVM 详细信息

memory             # 内存区域使用情况
heapdump /tmp/h.hprof  # Heap Dump
gc                 # GC 统计

trace MyClass method     # 方法调用链路追踪（含耗时）
watch MyClass method "{params, returnObj, throwExp}" # 观察方法参数和返回值
monitor -c 5 MyClass method  # 每5秒统计方法调用情况
stack MyClass method         # 方法调用堆栈
jad MyClass                  # 反编译（查看运行中的字节码）
retransform /tmp/MyClass.class  # 热更新字节码

# 在线诊断 ClassLoader
classloader --tree        # 类加载器树
classloader -c <hash>     # 查看指定 ClassLoader 加载的类
```

### 4.3 MAT（Memory Analyzer Tool）分析 Heap Dump

```
1. 打开 heapdump.hprof
2. 查看 Dominator Tree（支配树）：找出占用内存最多的对象
3. 查看 Leak Suspects Report：MAT 自动分析内存泄漏嫌疑
4. OQL 查询：select * from java.util.ArrayList where size > 10000
5. Incoming References：找出引用某对象的所有引用链
```

---

## 五、内存泄漏排查实战

### 5.1 现象

- 堆内存使用率持续增长，Full GC 后无法回到正常水位
- 最终触发 OOM（`java.lang.OutOfMemoryError: Java heap space`）

### 5.2 排查步骤

```bash
# Step 1: 监控老年代使用率
jstat -gcutil <pid> 5000
# 关注 O 列（老年代），若持续增长而 GC 后不回收 → 内存泄漏

# Step 2: 获取堆对象统计
jmap -histo:live <pid> | head -50
# 找出数量/大小异常的类（如 char[], byte[], HashMap$Entry 数量远超预期）

# Step 3: Heap Dump
jmap -dump:live,format=b,file=/tmp/heap.hprof <pid>
# 或用 Arthas: heapdump /tmp/heap.hprof

# Step 4: MAT 分析
# 打开 Heap Dump → Dominator Tree → 找最大对象 → 查 Incoming References
```

### 5.3 常见内存泄漏场景

```java
// 场景1：静态集合无限增长
public class StaticLeak {
    private static final List<Object> CACHE = new ArrayList<>();

    public void addData(Object obj) {
        CACHE.add(obj); // 永远不清空，持续增长
    }
}

// 场景2：监听器/回调未注销
eventBus.register(listener); // 注册后忘记 unregister
// listener 持有对象的引用，无法被 GC

// 场景3：ThreadLocal 未清理
private static ThreadLocal<LargeObject> local = new ThreadLocal<>();
// 线程池中线程不会死亡，local 中的对象永久存在
// 修复：finally 块中调用 local.remove()

// 场景4：连接/流未关闭
try {
    Connection conn = dataSource.getConnection();
    // ... 使用但未关闭
    // 忘记 conn.close()
}
// 修复：使用 try-with-resources
try (Connection conn = dataSource.getConnection()) {
    // ...
}

// 场景5：缓存无过期策略
Map<String, Object> cache = new HashMap<>();
// 修复：使用 LRU 缓存（LinkedHashMap 或 Caffeine）或设置过期时间
```

---

## 六、CPU 飙升排查实战

### 6.1 现象

- CPU 使用率异常高（>80%）
- 应用响应变慢

### 6.2 排查步骤

```bash
# Step 1: 找出 CPU 最高的 Java 进程
top -o %CPU

# Step 2: 找出进程中 CPU 最高的线程（Linux）
top -H -p <pid>
# 记录 CPU 最高的线程 ID（十进制）

# Step 3: 将线程 ID 转为十六进制
printf '%x\n' <tid>  # 如 tid=12345 → 3039

# Step 4: jstack 定位线程
jstack <pid> | grep -A 30 "nid=0x3039"
# 查看该线程的调用栈

# Step 5: Arthas（更方便）
thread -n 5  # 直接显示最消耗CPU的5个线程及其堆栈
```

### 6.3 常见 CPU 高的原因

| 原因 | 表现 | 解决 |
|------|------|------|
| 死循环 | 线程栈一直在同一代码段 | 修复循环逻辑 |
| GC 频繁 | GC 线程 CPU 高，YGC/FGC 频繁 | 调整堆大小/GC策略 |
| 正则表达式回溯 | 线程卡在 `Pattern.matcher` | 优化正则或设置超时 |
| 加解密/哈希计算 | 定位到加密方法 | 缓存/减少调用 |
| 序列化/反序列化 | 定位到 JSON 解析 | 减少序列化、使用更快的库 |

---

## 七、频繁 Full GC 排查实战

### 7.1 现象

- Full GC 每隔几分钟甚至更频繁发生
- 应用出现规律性卡顿

### 7.2 排查步骤

```bash
# Step 1: 确认 Full GC 频率和原因
jstat -gccause <pid> 5000
# LGCC（上次 GC 原因）和 GCC（当前 GC 原因）

# Step 2: 常见原因及对应措施
```

### 7.3 常见 Full GC 原因

| 原因 | GC Cause | 解决方案 |
|------|---------|---------|
| 老年代空间不足 | Allocation Failure | 调大堆/优化对象存活时间 |
| 大对象直接进老年代 | Humongous Allocation | 减小大对象/调大 Region |
| 元空间耗尽 | Metadata GC Threshold | 调大 MaxMetaspaceSize |
| `System.gc()` | System.gc() | 加 `-XX:+DisableExplicitGC` |
| CMS 并发模式失败 | Concurrent Mode Failure | 降低 CMSInitiatingOccupancyFraction |
| 晋升失败 | Promotion Failed | 增大 Survivor / 增大老年代 |

---

## 八、线上调优最佳实践

### 8.1 生产环境 JVM 配置模板（G1 + 4C8G）

```bash
java \
  -server \
  -Xms6g -Xmx6g \
  -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:InitiatingHeapOccupancyPercent=45 \
  -XX:G1HeapRegionSize=16m \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/logs/heapdump.hprof \
  -Xlog:gc*=info:file=/logs/gc.log:time,uptime,pid,level,tags:filecount=10,filesize=100m \
  -XX:+DisableExplicitGC \
  -XX:+PrintFlagsFinal \
  -jar app.jar
```

### 8.2 调优原则

1. **不要过早调优**：先上监控，有问题再调
2. **每次只改一个参数**：对比效果，避免多参数干扰
3. **充分压测验证**：线下压测 + 线上灰度
4. **保留 GC 日志**：用于事后分析
5. **配置 OOM 自动 Heap Dump**：方便排查
6. **监控 + 告警**：老年代 > 80% / Full GC 频率过高时告警

---

## 九、大厂常见面试题

### Q1：如何查找内存泄漏？

**答：**

1. **发现问题**：监控老年代使用率持续增长，Full GC 后不降回正常水位，最终触发 OOM。
2. **初步定位**：`jstat -gcutil <pid> 5000` 观察老年代增长曲线；`jmap -histo:live <pid>` 查看对象数量分布，找出异常增长的类型。
3. **Heap Dump**：`jmap -dump:live,format=b,file=heap.hprof <pid>`，通过 MAT 分析：
   - Dominator Tree 找最大内存占用对象
   - Leak Suspects Report 自动分析嫌疑
   - Incoming References 找出 GC Roots 引用链
4. **修复**：针对具体泄漏点修复（静态集合、未关闭资源、ThreadLocal 未 remove 等）。

---

### Q2：jstack 能解决什么问题？

**答：**

`jstack` 生成 JVM 线程快照（Thread Dump），主要用于：

1. **死锁检测**：`jstack <pid>` 输出尾部会显示 `Found X deadlock(s)`
2. **CPU 飙升定位**：结合 `top -H` 找到高 CPU 线程 ID，用 `jstack` 查看其调用栈，定位死循环/阻塞代码
3. **线程阻塞分析**：查找大量 `BLOCKED` 或 `WAITING` 状态的线程，找到竞争点
4. **应用假死排查**：所有线程都在等待某个锁或资源

```bash
# 建议连续采集3次，对比线程状态变化
jstack -l <pid> > /tmp/td1.txt
sleep 5
jstack -l <pid> > /tmp/td2.txt
diff /tmp/td1.txt /tmp/td2.txt
```

---

### Q3：-Xms 和 -Xmx 建议设置相同吗？

**答：**

**建议设置为相同**：

- 若 `-Xms < -Xmx`，JVM 会从 Xms 大小的堆开始，内存不足时动态扩容到 Xmx。扩容操作会触发 Full GC（整理内存），产生额外的 STW 停顿。
- 设置相同（如 `-Xms4g -Xmx4g`）可以避免动态扩容的 GC，启动时一次性申请固定内存。
- **注意**：`-Xmx` 不要超过物理内存的 75%~80%，给 OS、元空间、堆外内存（Direct Memory）留有余量。

---

### Q4：什么情况下会触发 Full GC？如何避免？

**答：**

**触发 Full GC 的场景：**
1. 老年代空间不足（晋升对象过多）
2. 元空间溢出
3. `System.gc()` 主动调用
4. CMS Concurrent Mode Failure（并发回收期间老年代填满）
5. Minor GC 后 Survivor 不足，需要老年代担保但空间不够

**避免 Full GC 策略：**
- 合理调大老年代，减少晋升频率（调整 `-Xmn` 或 `NewRatio`）
- 减少大对象创建（避免 Humongous Allocation）
- 及时释放不再使用的对象引用
- 禁用 `System.gc()`（`-XX:+DisableExplicitGC`）
- 使用 G1/ZGC 替代 CMS，避免 Concurrent Mode Failure
- 监控 Metaspace，设置合理上限

---

### Q5：如何在不重启的情况下修改 JVM 参数？

**答：**

大部分 JVM 参数无法在运行时修改，但以下方法可以：

1. **jinfo 修改部分参数（可管理标志）：**
   ```bash
   jinfo -flag +PrintGCDetails <pid>    # 开启 GC 详情日志
   jinfo -flag -DisableExplicitGC <pid> # 关闭禁用显式 GC
   jinfo -flag MaxHeapFreeRatio=70 <pid>
   ```
   只有标记为 `manageable` 的参数（`java -XX:+PrintFlagsFinal | grep manageable`）才能通过此方式修改。

2. **Arthas 动态修改：**
   ```bash
   # 观察方法调用（不修改字节码，不需要重启）
   watch、trace、retransform 等
   ```

3. **代码中通过 MXBean 修改：**
   ```java
   HotSpotDiagnosticMXBean mxBean = ManagementFactory
       .getPlatformMXBean(HotSpotDiagnosticMXBean.class);
   mxBean.setVMOption("PrintGCDetails", "true");
   ```

---

*参考资料：《深入理解Java虚拟机》、Arthas 官方文档、美团技术博客 JVM 调优系列*
