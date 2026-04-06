# 05 - Java基础常见面试题

## 目录

1. [集合框架面试题](#一集合框架面试题)
2. [IO/NIO 面试题](#二ionio-面试题)
3. [Java 8 新特性面试题](#三java-8-新特性面试题)
4. [设计模式面试题](#四设计模式面试题)
5. [综合面试题](#五综合面试题)

---

## 一、集合框架面试题

### Q1：ArrayList 和 Vector 有什么区别？

**答：**

| 维度 | ArrayList | Vector |
|------|-----------|--------|
| 线程安全 | 非线程安全 | 线程安全（方法加 synchronized） |
| 扩容策略 | 1.5 倍 | 2 倍 |
| 性能 | 快 | 慢（同步开销） |
| 迭代器 | fail-fast | fail-fast |
| 推荐程度 | **推荐** | **不推荐**（过时） |

需要线程安全的 List，推荐使用 `Collections.synchronizedList()` 或 `CopyOnWriteArrayList`。

---

### Q2：HashMap 在多线程环境下会出现什么问题？如何解决？

**答：**

**JDK 7 的问题：**
- **死循环（环形链表）**：多线程并发扩容时，头插法反转链表可能形成环，`get()` 陷入死循环，导致 CPU 100%。

**JDK 8 的问题：**
- **数据覆盖**：两个线程同时 put 到同一个空桶，后执行的线程覆盖先执行的数据。
- **size 不准确**：`size++` 非原子操作，可能丢失计数。

**解决方案：**

```java
// 方案1：ConcurrentHashMap（推荐）
Map<String, String> map = new ConcurrentHashMap<>();

// 方案2：Collections.synchronizedMap（整体加锁，性能差）
Map<String, String> syncMap = Collections.synchronizedMap(new HashMap<>());

// 方案3：Hashtable（过时，不推荐）
Map<String, String> table = new Hashtable<>();
```

---

### Q3：HashMap 的 put 过程中，是先扩容还是先插入？

**答：**

**JDK 7**：先判断是否需要扩容，再插入。即先 `resize()`，再 `addEntry()`。

**JDK 8**：先插入，再判断是否需要扩容。即先完成节点插入（链表尾插或红黑树插入），然后检查 `++size > threshold`，如果超过阈值则 `resize()`。

此外，链表长度达到 8 时会尝试转红黑树，但如果数组长度不足 64，会优先扩容而不是转树。

---

### Q4：HashMap 和 Hashtable 的区别？

**答：**

| 维度 | HashMap | Hashtable |
|------|---------|-----------|
| 线程安全 | 非线程安全 | 线程安全（synchronized） |
| null 键值 | 允许 1 个 null key + 多个 null value | **不允许** null key/value |
| 初始容量 | 16 | 11 |
| 扩容 | 2 倍 | 2 倍 + 1 |
| 迭代器 | fail-fast（Iterator） | fail-safe（Enumerator） |
| 继承 | AbstractMap | Dictionary（已废弃） |
| 推荐 | ✅ 单线程场景 | ❌ 过时，用 ConcurrentHashMap |

---

### Q5：ConcurrentHashMap 的 size() 是精确的吗？

**答：**

`size()` 返回的是**近似值**，不是精确值。原因：

1. JDK 8 的 ConcurrentHashMap 使用 `baseCount + CounterCell[]` 来统计，类似 `LongAdder` 的分段计数思想。
2. `size()` 方法调用 `sumCount()`，将 `baseCount` 和所有 `CounterCell` 的值累加。
3. 在高并发下，累加过程中可能有其他线程正在修改计数，因此结果是近似的。
4. 如果需要映射数量，推荐用 `mappingCount()`（返回 long，避免 int 溢出）。

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
int size = map.size();            // 近似值，返回 int
long count = map.mappingCount();  // 近似值，返回 long（推荐）
```

---

### Q6：TreeMap 和 HashMap 的区别？什么时候用 TreeMap？

**答：**

| 维度 | HashMap | TreeMap |
|------|---------|---------|
| 底层结构 | 数组 + 链表 + 红黑树 | 红黑树 |
| 是否有序 | 无序 | 按键排序（自然顺序或 Comparator） |
| 时间复杂度 | O(1)（均摊） | O(log n) |
| null 键 | 允许 1 个 | 不允许（需要比较） |
| 实现接口 | Map | NavigableMap、SortedMap |

**使用 TreeMap 的场景：**
- 需要按键排序输出（如排行榜、时间序列）
- 需要范围查询（`subMap`、`headMap`、`tailMap`）
- 需要获取最大/最小键（`firstKey`、`lastKey`）

---

### Q7：如何用 LinkedHashMap 实现 LRU 缓存？

**答：**

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;

    public LRUCache(int maxSize) {
        // accessOrder = true 开启访问顺序模式
        super(maxSize, 0.75f, true);
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // 当元素数量超过最大容量时，移除最久未访问的元素（链表头部）
        return size() > maxSize;
    }
}
```

**原理：**
- `accessOrder = true` 使得每次 `get()`/`put()` 操作都会将被访问的元素移到双向链表的尾部
- 链表头部始终是最久未被访问的元素
- `removeEldestEntry` 在每次 `put()` 后被调用，当超过容量时自动删除头部元素

---

## 二、IO/NIO 面试题

### Q8：Java IO 流有哪些分类？

**答：**

按照**数据类型**和**流向**可以分为四类：

```
              输入                  输出
  字节流   InputStream          OutputStream
  字符流   Reader               Writer
```

```
常用实现类体系：

  InputStream
    ├── FileInputStream          文件读取
    ├── ByteArrayInputStream     字节数组
    ├── BufferedInputStream      缓冲（装饰器）
    ├── DataInputStream          基本类型
    └── ObjectInputStream        对象反序列化
    
  OutputStream
    ├── FileOutputStream
    ├── ByteArrayOutputStream
    ├── BufferedOutputStream
    ├── DataOutputStream
    └── ObjectOutputStream
    
  Reader
    ├── FileReader
    ├── BufferedReader            (readLine)
    ├── InputStreamReader         (字节→字符，适配器模式)
    └── StringReader
    
  Writer
    ├── FileWriter
    ├── BufferedWriter
    ├── OutputStreamWriter
    └── StringWriter
```

**字节流 vs 字符流的选择：**
- 处理二进制数据（图片、视频、压缩包） → 字节流
- 处理文本数据 → 字符流（自动处理编码）

---

### Q9：NIO 中 Buffer 的 flip() 方法做了什么？

**答：**

`flip()` 将 Buffer 从**写模式**切换到**读模式**：

```java
// flip() 源码
public Buffer flip() {
    limit = position;    // 将 limit 设置为当前写到的位置
    position = 0;        // 将 position 重置为 0
    mark = -1;           // 清除 mark
    return this;
}
```

```
写模式（写入 "Hi" 后）：
  ┌───┬───┬───┬───┬───┐
  │ H │ i │   │   │   │
  └───┴───┴───┴───┴───┘
  0   1   2   3   4
              ↑           ↑
           position    capacity/limit

flip() 后变为读模式：
  ┌───┬───┬───┬───┬───┐
  │ H │ i │   │   │   │
  └───┴───┴───┴───┴───┘
  0   1   2   3   4
  ↑       ↑           ↑
position limit     capacity
```

这样后续的 `get()` 操作就能从头开始读取已写入的数据。

---

### Q10：什么是 Reactor 模式？和 NIO 是什么关系？

**答：**

Reactor 模式是一种基于事件驱动的设计模式，用于处理并发 IO 请求。NIO 的 Selector 就是 Reactor 模式的实现基础。

```
单 Reactor 单线程模型：

  ┌─────────────────────────────┐
  │         Reactor              │
  │  Selector.select()           │
  │      │                       │
  │   ┌──┴──┐                    │
  │   ▼     ▼                    │
  │ Accept  Read/Write           │
  │   │       │                  │
  │   ▼       ▼                  │
  │ 注册    业务处理              │
  │ 新连接    + 响应              │
  └─────────────────────────────┘
  适用场景：客户端少，业务处理快


主从 Reactor 多线程模型（Netty 采用）：

  ┌───────────────────┐    ┌───────────────────┐
  │  Main Reactor      │    │  Sub Reactor(s)    │
  │  (Boss Group)      │    │  (Worker Group)    │
  │                    │    │                    │
  │  只负责 Accept     │───►│  负责 Read/Write   │
  │  新连接            │    │  业务处理          │
  └───────────────────┘    └───────────────────┘
  
  Boss 线程：接受连接，注册到 Worker
  Worker 线程池：处理 IO 读写 + 业务逻辑
```

---

### Q11：Kafka 和 RocketMQ 分别用了哪种零拷贝？为什么？

**答：**

| 消息队列 | 零拷贝方式 | 实现 | 原因 |
|---------|-----------|------|------|
| **Kafka** | sendfile | `FileChannel.transferTo()` | 消息直接从磁盘文件发送到网络，不需要经过应用层处理 |
| **RocketMQ** | mmap | `MappedByteBuffer` | 需要在应用层做消息过滤（tag 过滤），必须读入用户空间 |

**总结：**
- 如果只需要传输文件（不在应用层处理数据） → `sendfile`
- 如果需要在应用层读取/修改数据 → `mmap`

---

## 三、Java 8 新特性面试题

### Q12：Stream 的 map 和 flatMap 有什么区别？

**答：**

| 方法 | 签名 | 作用 |
|------|------|------|
| `map` | `Stream<R> map(Function<T, R>)` | 一对一映射 |
| `flatMap` | `Stream<R> flatMap(Function<T, Stream<R>>)` | 一对多映射 + 扁平化 |

```java
// map：每个元素映射为一个结果
List<String> words = Arrays.asList("Hello", "World");
List<Integer> lengths = words.stream()
    .map(String::length)
    .collect(Collectors.toList()); // [5, 5]

// flatMap：每个元素映射为一个 Stream，然后合并
List<String> chars = words.stream()
    .map(w -> w.split(""))       // Stream<String[]>
    .flatMap(Arrays::stream)      // Stream<String>（扁平化）
    .distinct()
    .collect(Collectors.toList()); // [H, e, l, o, W, r, d]
```

类比：`map` 是 `1 → 1` 的变换，`flatMap` 是 `1 → N` 的变换 + 扁平化。

---

### Q13：Stream 中 reduce 操作怎么用？

**答：**

`reduce` 用于将 Stream 中的元素**聚合**为单个结果。

```java
// 形式1：有初始值
int sum = Stream.of(1, 2, 3, 4, 5)
    .reduce(0, Integer::sum);  // 15

// 形式2：无初始值（返回 Optional）
Optional<Integer> max = Stream.of(1, 2, 3)
    .reduce(Integer::max);  // Optional[3]

// 形式3：并行流使用的三参数版本
// reduce(identity, accumulator, combiner)
int result = Arrays.asList(1, 2, 3, 4)
    .parallelStream()
    .reduce(0,                  // 初始值
            Integer::sum,       // 累加器
            Integer::sum);      // 合并器（并行时合并各子流结果）
```

**执行过程：**
```
reduce(0, Integer::sum) 对 [1, 2, 3, 4, 5]：
  0 + 1 = 1
  1 + 2 = 3
  3 + 3 = 6
  6 + 4 = 10
  10 + 5 = 15
```

---

### Q14：Collectors.toMap 遇到 key 重复会怎样？怎么解决？

**答：**

默认情况下，key 重复会抛出 `IllegalStateException`。

```java
// 抛异常
List<User> users = Arrays.asList(
    new User(1, "Alice"), new User(1, "Bob"));
Map<Integer, String> map = users.stream()
    .collect(Collectors.toMap(User::getId, User::getName));
// IllegalStateException: Duplicate key 1

// 解决：提供合并函数
Map<Integer, String> map = users.stream()
    .collect(Collectors.toMap(
        User::getId,
        User::getName,
        (old, newVal) -> newVal  // key 冲突时保留新值
    ));

// 指定 Map 实现
Map<Integer, String> linked = users.stream()
    .collect(Collectors.toMap(
        User::getId,
        User::getName,
        (old, newVal) -> newVal,
        LinkedHashMap::new       // 保持插入顺序
    ));
```

---

### Q15：函数式接口和普通接口有什么区别？

**答：**

**函数式接口**是只有**一个抽象方法**的接口（可以有多个默认方法和静态方法），可以用 Lambda 表达式或方法引用来实例化。

```java
// 标记注解（可选，但推荐加上）
@FunctionalInterface
public interface Converter<F, T> {
    T convert(F from);
    
    // 以下不影响函数式接口判定
    default Converter<F, T> andThen(Function<T, T> after) { ... }  // 默认方法
    static <T> Converter<T, T> identity() { ... }                  // 静态方法
    boolean equals(Object obj);  // Object 类的 public 方法不算
}

// Lambda 实例化
Converter<String, Integer> converter = Integer::parseInt;
```

**注意：** `@FunctionalInterface` 注解不是必须的，但它能让编译器帮你检查接口是否符合函数式接口的定义。

---

### Q16：Java 8 的 CompletableFuture 和 Future 有什么区别？

**答：**

| 维度 | Future | CompletableFuture |
|------|--------|-------------------|
| 获取结果 | `get()` 阻塞等待 | 支持回调（`thenApply`/`thenAccept`） |
| 异常处理 | 只能 try-catch | `exceptionally()`/`handle()` |
| 组合操作 | 不支持 | `thenCompose`/`thenCombine`/`allOf`/`anyOf` |
| 手动完成 | 不支持 | `complete()`/`completeExceptionally()` |
| 链式调用 | ❌ | ✅ |

```java
// Future：阻塞等待
Future<String> future = executor.submit(() -> queryDB());
String result = future.get(); // 阻塞！

// CompletableFuture：非阻塞回调
CompletableFuture.supplyAsync(() -> queryDB())
    .thenApply(data -> process(data))
    .thenAccept(result -> System.out.println(result))
    .exceptionally(ex -> { log.error("失败", ex); return null; });
```

---

## 四、设计模式面试题

### Q17：代理模式和装饰器模式有什么区别？

**答：**

虽然结构相似（都持有目标对象的引用），但意图不同：

| 维度 | 代理模式 | 装饰器模式 |
|------|---------|-----------|
| **意图** | 控制访问 | 增强功能 |
| **创建时机** | 代理自己创建或注入目标对象 | 客户端主动组合装饰器 |
| **使用者感知** | 客户端不知道代理存在 | 客户端知道并主动使用装饰器 |
| **层数** | 通常一层代理 | 可以多层嵌套装饰 |
| **JDK 示例** | `java.lang.reflect.Proxy` | `BufferedInputStream` 装饰 `FileInputStream` |
| **Spring 示例** | AOP 代理 | — |

---

### Q18：单例模式如何被破坏？如何防止？

**答：**

**两种破坏方式：**

1. **反射攻击：**

```java
Singleton instance1 = Singleton.getInstance();

// 通过反射获取私有构造方法
Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
constructor.setAccessible(true);
Singleton instance2 = constructor.newInstance(); // 新实例！

System.out.println(instance1 == instance2); // false
```

**防御：** 在构造方法中检查
```java
private Singleton() {
    if (INSTANCE != null) {
        throw new RuntimeException("不允许反射创建实例");
    }
}
```

2. **反序列化攻击：**

```java
// 序列化后再反序列化，会生成新对象
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("singleton.ser"));
oos.writeObject(instance1);

ObjectInputStream ois = new ObjectInputStream(new FileInputStream("singleton.ser"));
Singleton instance3 = (Singleton) ois.readObject(); // 新实例！
```

**防御：** 添加 `readResolve` 方法
```java
private Object readResolve() {
    return INSTANCE; // 反序列化时返回已有实例
}
```

**终极方案：** 使用**枚举单例**，天然防止反射和反序列化破坏。

---

### Q19：工厂模式在实际项目中怎么用？

**答：**

**场景1：Spring 的 BeanFactory**

```java
// Spring 容器本身就是一个大工厂
ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");
UserService userService = ctx.getBean(UserService.class);
```

**场景2：多数据源/多渠道策略**

```java
// 支付渠道工厂 + 策略模式
@Component
public class PaymentFactory {
    @Autowired
    private Map<String, PaymentStrategy> strategyMap;
    // Spring 自动注入所有 PaymentStrategy 实现，key 为 Bean 名
    
    public PaymentStrategy getStrategy(String channel) {
        PaymentStrategy strategy = strategyMap.get(channel);
        if (strategy == null) {
            throw new IllegalArgumentException("不支持的支付渠道: " + channel);
        }
        return strategy;
    }
}

@Service("alipay")
public class AlipayStrategy implements PaymentStrategy { ... }

@Service("wechat")
public class WechatPayStrategy implements PaymentStrategy { ... }
```

**场景3：SLF4J 日志工厂**

```java
// LoggerFactory 根据 classpath 中的绑定自动选择日志实现
Logger logger = LoggerFactory.getLogger(MyClass.class);
```

---

### Q20：模板方法和策略模式有什么区别？

**答：**

| 维度 | 模板方法 | 策略模式 |
|------|---------|---------|
| 代码复用 | 通过**继承**复用算法骨架 | 通过**组合**替换算法 |
| 变化点 | 某些步骤延迟到子类 | 整个算法可替换 |
| 扩展方式 | 新建子类覆写抽象方法 | 新建策略类（或 Lambda） |
| 控制反转 | 父类控制流程，子类实现细节 | 客户端选择策略 |
| 粒度 | 算法中的部分步骤 | 整个算法 |

```java
// 模板方法：固定流程，变化的是某些步骤
abstract class DataExporter {
    final void export() {     // 骨架固定
        fetchData();          // 固定步骤
        format();             // 子类实现
        save();               // 固定步骤
    }
    abstract void format();
}

// 策略模式：整个算法可替换
interface SortStrategy {
    void sort(int[] arr);
}
// 客户端选择：quickSort / mergeSort / heapSort
```

---

## 五、综合面试题

### Q21：Java 集合框架中用了哪些设计模式？

**答：**

| 设计模式 | 应用场景 |
|---------|---------|
| **迭代器模式** | `Iterator` 接口，所有集合都实现了 `Iterable` |
| **适配器模式** | `Arrays.asList()` 将数组适配为 List |
| **装饰器模式** | `Collections.unmodifiableList()` 将 List 包装为不可修改 |
| **工厂方法** | `Collections.emptyList()` / `List.of()` |
| **模板方法** | `AbstractList` 中 `indexOf()` 使用 `get()`（子类实现） |
| **策略模式** | `Comparator` 作为排序策略传入 `Collections.sort()` |

---

### Q22：如何在遍历 List 时安全地删除元素？

**答：**

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c", "b", "d"));

// ❌ 错误1：增强 for 循环直接删除（ConcurrentModificationException）
for (String s : list) {
    if ("b".equals(s)) list.remove(s);
}

// ❌ 错误2：普通 for 循环正序删除（会跳过元素）
for (int i = 0; i < list.size(); i++) {
    if ("b".equals(list.get(i))) list.remove(i);
    // 删除 index=1 后，原 index=2 变成 index=1，被跳过
}

// ✅ 方法1：迭代器 remove
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if ("b".equals(it.next())) it.remove();
}

// ✅ 方法2：JDK 8 removeIf
list.removeIf("b"::equals);

// ✅ 方法3：普通 for 循环倒序删除
for (int i = list.size() - 1; i >= 0; i--) {
    if ("b".equals(list.get(i))) list.remove(i);
}

// ✅ 方法4：Stream 过滤生成新 List
List<String> filtered = list.stream()
    .filter(s -> !"b".equals(s))
    .collect(Collectors.toList());
```

---

### Q23：equals 和 hashCode 为什么必须同时重写？

**答：**

Java 规范约定：
1. 两个对象 `equals()` 相等，则 `hashCode()` 必须相同
2. `hashCode()` 相同，`equals()` 不一定相等（哈希冲突）

如果只重写 `equals()` 不重写 `hashCode()`，会导致 HashSet/HashMap 行为异常：

```java
class User {
    String name;
    int age;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User user = (User) o;
        return age == user.age && Objects.equals(name, user.name);
    }
    
    // 如果不重写 hashCode：
    // new User("张三", 25) 和 new User("张三", 25)
    // equals 返回 true，但 hashCode 不同（默认用对象内存地址）
    // 放入 HashSet 时会被当作两个不同的元素！
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age); // 必须同时重写
    }
}
```

---

### Q24：Comparable 和 Comparator 有什么区别？

**答：**

| 维度 | Comparable | Comparator |
|------|-----------|------------|
| 所在包 | `java.lang` | `java.util` |
| 方法 | `compareTo(T o)` | `compare(T o1, T o2)` |
| 修改类 | 需要修改类本身 | 不需要修改类 |
| 用途 | 定义**自然排序**（唯一） | 定义**自定义排序**（可多个） |
| 使用方式 | 类实现接口 | 外部传入 |

```java
// Comparable：类自身定义排序规则
public class Student implements Comparable<Student> {
    private String name;
    private int score;
    
    @Override
    public int compareTo(Student other) {
        return Integer.compare(this.score, other.score); // 按分数升序
    }
}

Collections.sort(students); // 使用自然排序

// Comparator：外部定义排序规则，不修改类
Collections.sort(students, Comparator.comparing(Student::getName)); // 按姓名
Collections.sort(students, Comparator.comparingInt(Student::getScore).reversed()); // 按分数降序
```

---

### Q25：Java 序列化和反序列化中有哪些坑？

**答：**

1. **serialVersionUID**：不指定时 JVM 自动生成，类结构变化后旧数据反序列化失败

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L; // 建议手动指定
    private String name;
}
```

2. **transient 字段**：不参与序列化，反序列化后为默认值

```java
private transient String password; // 不会被序列化
```

3. **单例被破坏**：反序列化会创建新对象，需要 `readResolve()` 防御

4. **安全风险**：反序列化可执行任意代码（反序列化漏洞），推荐使用 JSON（Jackson/Gson）替代 Java 原生序列化

5. **父类字段**：父类未实现 `Serializable` 时，其字段不会被序列化，反序列化时调用父类无参构造

---

### Q26：什么是 try-with-resources？和 finally 有什么区别？

**答：**

`try-with-resources`（JDK 7+）自动关闭实现了 `AutoCloseable` 接口的资源，比 `try-finally` 更简洁、更安全。

```java
// JDK 7 之前：try-finally 手动关闭
InputStream is = null;
try {
    is = new FileInputStream("data.txt");
    // 读取数据...
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (is != null) {
        try { is.close(); } catch (IOException e) { e.printStackTrace(); }
    }
}

// JDK 7+：try-with-resources 自动关闭
try (InputStream is = new FileInputStream("data.txt");
     BufferedReader reader = new BufferedReader(new InputStreamReader(is))) {
    String line = reader.readLine();
} catch (IOException e) {
    e.printStackTrace();
}
// 自动调用 close()，多个资源按声明的逆序关闭
```

**关键区别：**
- `try-with-resources` 会抑制 `close()` 中抛出的异常（Suppressed Exception），保留原始异常
- `try-finally` 中如果 finally 也抛异常，会覆盖 try 中的异常，导致原始异常丢失

---

*参考资料：《Effective Java》、《Java核心技术》、阿里巴巴 Java 开发手册、各大厂面试真题汇总*
