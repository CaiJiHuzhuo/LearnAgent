# 05 - JIT 编译与优化

## 目录

1. [解释执行与 JIT 编译](#一解释执行与-jit-编译)
2. [JIT 编译器类型](#二jit-编译器类型)
3. [热点代码检测](#三热点代码检测)
4. [JIT 优化技术](#四jit-优化技术)
5. [逃逸分析](#五逃逸分析)
6. [向量化与 SIMD](#六向量化与-simd)
7. [AOT 编译（GraalVM Native Image）](#七aot-编译graalvm-native-image)
8. [大厂常见面试题](#八大厂常见面试题)

---

## 一、解释执行与 JIT 编译

### 1.1 Java 执行方式演进

```
源代码（.java）
    │
    ▼ javac（前端编译）
字节码（.class）
    │
    ├─► 解释执行（Interpreter）：逐条翻译字节码，启动快，执行慢
    │
    └─► JIT 编译（Just-In-Time）：热点代码编译为本地机器码，执行快
```

### 1.2 混合模式（Mixed Mode）

HotSpot JVM 默认使用**混合模式**：

```
程序启动：解释执行（快速启动）
    │
    ▼（方法调用次数 / 回边跳转次数达到阈值）
JIT 编译（后台异步）：生成本地代码
    │
    ▼
替换解释执行，直接运行本地代码（On-Stack Replacement, OSR）
```

```bash
# 查看执行模式
java -version
# 输出包含：mixed mode（混合）/ interpreted（纯解释）/ compiled（纯编译）

# 强制解释执行（不进行 JIT 编译）
java -Xint -jar app.jar

# 强制编译执行（跳过解释器，全部 JIT）
java -Xcomp -jar app.jar
```

---

## 二、JIT 编译器类型

### 2.1 C1 编译器（Client Compiler）

- 简单快速的三段优化
- 适合客户端程序（快速启动，编译耗时短）
- 优化级别：L0（无优化）→ L1（简单优化）→ L2（有限内联）→ L3（完整 C1）

### 2.2 C2 编译器（Server Compiler）

- 深度优化，编译耗时长，生成高质量本地代码
- 适合服务端长时间运行的程序（启动后性能更好）
- JDK 64 位版本默认使用 Server 模式

### 2.3 分层编译（Tiered Compilation，JDK 7+ 默认）

```
Level 0: 解释执行（Interpreter）
Level 1: C1 编译（无 profiling）
Level 2: C1 编译（方法调用次数 & 回边次数 profiling）
Level 3: C1 编译（全 profiling）
Level 4: C2 编译（基于 profiling 的激进优化）
```

```
方法首次调用
    │ L0（解释）
    ▼ 调用次数增加
    L3（C1 全 profiling）← 收集类型信息
    │ profiling 充分
    ▼
    L4（C2 优化） ← 基于 profiling 做激进优化（方法内联、类型特化等）
```

```bash
# 关闭分层编译（仅用 C2）
-XX:-TieredCompilation
# 编译阈值（分层编译下调整）
-XX:CompileThreshold=10000  # 默认10000次
```

### 2.4 GraalVM Compiler（JDK 17+ 可选）

- 用 Java 编写的 JIT 编译器，可替换 C2
- 更丰富的优化（如更激进的内联）
- 同时支持 AOT（提前编译）

---

## 三、热点代码检测

### 3.1 计数器

HotSpot 使用两类计数器（在方法区/方法对象中）：

1. **方法调用计数器（Invocation Counter）**：记录方法被调用的次数
2. **回边计数器（Back Edge Counter）**：记录方法中循环体执行次数

```
方法调用计数器 >= CompileThreshold（默认 10000）→ 触发 JIT 编译
回边计数器 >= OnStackReplacePercentage → 触发 OSR（栈上替换）
```

### 3.2 计数器热度衰减

若方法调用不频繁，计数器会**周期性衰减**（不是精确计数，而是近似热点）：

```bash
# 关闭计数器衰减
-XX:-UseCounterDecay

# 设置方法调用计数器触发 JIT 的阈值
-XX:CompileThreshold=1000
```

### 3.3 查看 JIT 编译情况

```bash
# 打印 JIT 编译的方法
-XX:+PrintCompilation
# 输出格式：时间戳  编号  层级  方法名  编译大小
# 示例：
# 1234  1    3     java.lang.String::hashCode (67 bytes)
# 1235  2    4     java.util.HashMap::get (31 bytes)

# 输出到文件
-XX:+PrintCompilation -XX:+LogCompilation -XX:LogFile=/tmp/jit.log

# 使用 JITWatch 工具可视化分析 JIT 编译日志
```

---

## 四、JIT 优化技术

### 4.1 方法内联（Method Inlining）

最重要的 JIT 优化，将被调用方法的代码**直接嵌入调用处**，消除方法调用开销，同时为其他优化创造机会。

```java
// 内联前
int sum = 0;
for (int i = 0; i < n; i++) {
    sum += add(i, 1); // 方法调用
}
int add(int a, int b) { return a + b; }

// 内联后（JIT 自动完成）
int sum = 0;
for (int i = 0; i < n; i++) {
    sum += i + 1; // 直接展开，无方法调用
}
```

**内联限制：**
```bash
# 最大内联方法大小（字节码 bytes）
-XX:MaxInlineSize=35       # 默认35，超过不内联
-XX:FreqInlineSize=325     # 热点方法最大内联大小

# 影响内联的因素：方法过长、虚方法（多态）、递归
```

**代码设计建议：**
- 保持方法简短（< 35 字节码），利于内联
- 使用 `private`/`final` 方法（非虚方法，JIT 可直接内联）
- 减少不必要的多态（虚方法内联需要类型分析）

### 4.2 锁优化

#### 锁消除（Lock Elimination）

```java
// 代码中对 sb（局部变量）加锁，但 sb 不逃逸出方法
// JIT 发现后直接消除同步开销
public String buildString(String a, String b) {
    StringBuffer sb = new StringBuffer(); // StringBuffer 方法有 synchronized
    sb.append(a);  // JIT 发现 sb 不逃逸，消除所有 synchronized
    sb.append(b);
    return sb.toString();
}
// 实际运行等价于使用 StringBuilder
```

#### 锁粗化（Lock Coarsening）

```java
// JIT 将循环内多次加解锁合并为循环外一次
// 优化前
for (int i = 0; i < 100; i++) {
    synchronized (lock) { list.add(i); }
}
// 优化后（JIT 自动）
synchronized (lock) {
    for (int i = 0; i < 100; i++) { list.add(i); }
}
```

#### 偏向锁 / 轻量级锁

参见 [02-synchronized与Lock.md](../phase1-java-concurrent/02-synchronized与Lock.md)

### 4.3 公共子表达式消除（CSE）

```java
// 优化前
int result = a * b + c * d + a * b;  // a*b 计算了两次

// 优化后（JIT 自动）
int temp = a * b;
int result = temp + c * d + temp;    // 使用缓存结果
```

### 4.4 数组越界检查消除

```java
// JIT 分析循环范围后，自动消除数组越界检查
for (int i = 0; i < array.length; i++) {
    process(array[i]);  // 通常每次都要检查 i < array.length
    // JIT：i 在 [0, length) 内，消除运行时检查
}
```

### 4.5 循环展开（Loop Unrolling）

```java
// 优化前
for (int i = 0; i < 8; i++) {
    sum += array[i];
}

// 循环展开后（减少循环条件判断次数）
sum += array[0];
sum += array[1];
sum += array[2];
sum += array[3];
sum += array[4];
sum += array[5];
sum += array[6];
sum += array[7];
```

---

## 五、逃逸分析

### 5.1 什么是逃逸分析

**逃逸分析（Escape Analysis）**：JIT 分析对象的动态作用域，判断对象是否"逃逸"出方法或线程：

- **方法逃逸**：对象作为返回值或参数传出方法
- **线程逃逸**：对象被其他线程访问（赋值给实例/静态字段）
- **不逃逸**：对象只在方法内使用

### 5.2 基于逃逸分析的三种优化

#### 栈上分配（Stack Allocation）

```java
public void method() {
    // Point 不逃逸，JIT 可以在栈上分配（栈帧中）
    // 而不是在堆上分配（减少 GC 压力）
    Point p = new Point(1, 2);
    int dist = p.distance(3, 4);
    // p 用完自动随栈帧销毁，无需 GC
}
```

#### 标量替换（Scalar Replacement）

```java
// 对象不逃逸时，JIT 将其分解为各个字段（标量）
// 这些字段可以分配在栈上或寄存器中
class Point { int x, y; }

// 优化前
Point p = new Point();
p.x = 1;
p.y = 2;
int sum = p.x + p.y;

// 标量替换后（JIT 自动）
int x = 1;   // 不再创建 Point 对象
int y = 2;
int sum = x + y;
```

```bash
# 开启/关闭逃逸分析（JDK 8 默认开启）
-XX:+DoEscapeAnalysis
-XX:-DoEscapeAnalysis   # 关闭（用于对比测试）

# 开启标量替换（默认开启）
-XX:+EliminateAllocations

# 查看逃逸分析结果
-XX:+PrintEscapeAnalysis
```

#### 同步消除

见上一节锁消除。

### 5.3 逃逸分析的限制

- 逃逸分析本身有性能开销，JIT 在权衡后决定是否应用
- 对象规模大时，栈上分配可能不合算（需要大量复制）
- 分析不完全准确，保守估计可能错过优化机会

---

## 六、向量化与 SIMD

### 6.1 自动向量化

JIT（主要是 C2）可以将循环中的操作**自动向量化**，利用 CPU 的 SIMD 指令（如 SSE2、AVX2）同时处理多个数据。

```java
// JIT 可能将此循环向量化：
// 一次处理 4/8 个 int（AVX2 256位，每次 8×32位）
int[] a = new int[1024];
int[] b = new int[1024];
int[] c = new int[1024];
for (int i = 0; i < a.length; i++) {
    c[i] = a[i] + b[i]; // 可能被向量化为 8个并行加法
}
```

### 6.2 JDK 17 向量 API（孵化器）

```java
import jdk.incubator.vector.*;

static final VectorSpecies<Float> SPECIES = FloatVector.SPECIES_256;

// 显式使用 SIMD（AVX2）
void addWithVectorAPI(float[] a, float[] b, float[] c) {
    int bound = SPECIES.loopBound(a.length);
    int i = 0;
    for (; i < bound; i += SPECIES.length()) {
        FloatVector va = FloatVector.fromArray(SPECIES, a, i);
        FloatVector vb = FloatVector.fromArray(SPECIES, b, i);
        va.add(vb).intoArray(c, i);
    }
    // 处理剩余元素
    for (; i < a.length; i++) {
        c[i] = a[i] + b[i];
    }
}
```

---

## 七、AOT 编译（GraalVM Native Image）

### 7.1 AOT vs JIT

| 维度 | JIT | AOT（Native Image） |
|------|-----|-------------------|
| 编译时机 | 运行时（热点触发） | 构建时（提前编译） |
| 启动速度 | 慢（需要预热） | **极快**（毫秒级） |
| 峰值性能 | 高（运行时优化） | 较低（无法利用运行时信息） |
| 内存占用 | 较高 | **低**（无 JIT 编译器本身的内存） |
| 适用场景 | 长时间运行的服务 | Serverless、CLI 工具、微服务 |

### 7.2 GraalVM Native Image

```bash
# 安装 GraalVM 并安装 native-image
gu install native-image

# 编译为本地可执行文件
native-image -cp . HelloWorld

# Spring Boot 3 + Native
./mvnw spring-boot:build-image  # 需要 Docker
```

```java
// 反射、JNI、动态代理需要通过配置文件声明
// reflect-config.json
// [{"name":"com.example.MyClass","allDeclaredMethods":true}]
```

### 7.3 JDK 21 Project Leyden（提前优化）

JEP 483（Class Data Sharing 增强）和 Leyden 项目旨在：
- 将 AOT 优化集成进 JDK 本身（不依赖 GraalVM）
- 通过训练运行（training run）收集 profile，用于加速正式运行

---

## 八、大厂常见面试题

### Q1：JIT 编译器的作用是什么？分层编译是怎么工作的？

**答：**

JIT（Just-In-Time）编译器在程序运行时将热点字节码编译为**本地机器码**，避免解释执行的开销，大幅提升执行效率。

**分层编译（JDK 7+ 默认）** 共 5 个层级：
- **Level 0**：解释器执行
- **Level 1-3**：C1 编译（简单→有 profiling），编译快，质量中等，同时收集类型/分支统计信息
- **Level 4**：C2 编译，利用 profiling 数据做激进优化（内联、逃逸分析、向量化等）

流程：方法先在 L0 解释执行 → 计数器达到阈值进入 C1（L3 收集信息）→ profiling 充分后 C2 深度优化（L4）。

---

### Q2：什么是逃逸分析？它能带来哪些优化？

**答：**

逃逸分析判断对象是否逃逸出其创建的方法或线程，分析后可进行三种优化：

1. **栈上分配**：不逃逸的对象直接分配在栈帧中，随方法返回自动释放，减少堆分配和 GC 压力。
2. **标量替换**：不逃逸对象被分解为各个基本字段，字段直接分配在栈或寄存器中，完全消除对象分配。
3. **同步消除（锁消除）**：对只在当前线程中使用的对象（无线程逃逸），删除其上的同步锁（`synchronized`），因为不可能存在竞争。

**开启参数：** JDK 8+ 默认开启（`-XX:+DoEscapeAnalysis`），可通过 `-XX:-DoEscapeAnalysis` 关闭对比性能。

---

### Q3：为什么 Java 的 JIT 编译有时比 C++ 性能更好？

**答：**

C++ 是静态编译，编译时所有优化都基于静态信息。Java JIT 可以利用**运行时 profiling 信息**做更激进的动态优化，有时超过静态编译：

1. **内联更精准**：通过 profiling 知道某虚方法调用时的实际类型，直接内联（C++ 虚函数调用无法内联）
2. **类型特化**：如发现某字段总是 ArrayList，直接生成 ArrayList 专用代码，跳过多态调度
3. **分支预测优化**：根据真实运行频率优化分支，将热路径放在 CPU 指令缓存友好的位置
4. **去优化（Deoptimization）**：激进假设失败时退回解释执行，允许做更大胆的优化
5. **硬件感知**：运行时检测当前 CPU 指令集，利用 AVX2/AVX512 等新指令

---

### Q4：什么情况下方法不会被 JIT 内联？

**答：**

以下情况阻碍内联：
1. **方法过大**：字节码超过 `MaxInlineSize`（默认 35 bytes）。对于热点方法超过 `FreqInlineSize`（默认 325 bytes）
2. **虚方法（多态）**：需要运行时类型决议，JIT 会尝试去虚化（devirtualize），但如果接收者类型多样则无法内联
3. **Native 方法**：本地方法无字节码，无法内联
4. **递归**：深度递归可能导致内联栈溢出
5. **编译队列繁忙**：方法未被 JIT 编译（仍在解释执行）时不会内联

**实践建议：** 保持关键路径方法短小（< 35 字节码），尽量使用 `final`/`private` 减少虚方法开销。

---

### Q5：AOT 和 JIT 分别适合什么场景？

**答：**

| 场景 | 推荐 | 理由 |
|------|------|------|
| 长时间运行的微服务 | JIT | 预热后峰值性能高，JIT 持续优化 |
| Serverless / FaaS | AOT（GraalVM Native） | 极速冷启动（毫秒级），低内存占用 |
| CLI 工具 | AOT | 每次启动都是冷启动，JIT 预热无意义 |
| 大数据批处理 | JIT（分层） | 长时间运行充分预热，吞吐量优先 |
| 延迟极敏感（交易）| ZGC + JIT | 结合低延迟 GC，JIT 优化热点路径 |

Java 21 通过 Project Leyden 和 CRaC（Coordinated Restore at Checkpoint）技术，正在弥合 AOT 和 JIT 的差距，实现"提前训练 + 快速恢复"。

---

*参考资料：《深入理解Java虚拟机》周志明、HotSpot JIT 官方文档、Cliff Click（Java JIT 专家）系列博文、GraalVM 官方文档*
