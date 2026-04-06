# 01 - Java集合框架

## 目录

1. [集合框架整体结构](#一集合框架整体结构)
2. [ArrayList](#二arraylist)
3. [LinkedList](#三linkedlist)
4. [HashMap](#四hashmap)
5. [ConcurrentHashMap](#五concurrenthashmap)
6. [TreeMap 与 LinkedHashMap](#六treemap-与-linkedhashmap)
7. [HashSet](#七hashset)
8. [ArrayList vs LinkedList 性能对比](#八arraylist-vs-linkedlist-性能对比)
9. [HashMap 线程不安全的表现](#九hashmap-线程不安全的表现)
10. [大厂常见面试题](#十大厂常见面试题)

---

## 一、集合框架整体结构

Java 集合框架分为两大体系：**Collection**（单列集合）和 **Map**（键值对集合）。

```
                          Iterable
                             │
                         Collection
                 ┌───────────┼───────────┐
                List        Set        Queue
              ┌──┴──┐    ┌──┴──┐    ┌───┴───┐
         ArrayList  │  HashSet  │  PriorityQueue
                    │       TreeSet  │
              LinkedList        LinkedHashSet
                                          Deque
                                           │
                                      ArrayDeque

                           Map
                 ┌──────────┼──────────┐
             HashMap    TreeMap    Hashtable
                │
         LinkedHashMap
         
         ConcurrentHashMap（线程安全）
```

### 1.1 核心接口说明

| 接口 | 特点 | 常用实现 |
|------|------|----------|
| `List` | 有序、可重复、支持索引访问 | ArrayList、LinkedList |
| `Set` | 无序（逻辑上）、不可重复 | HashSet、TreeSet、LinkedHashSet |
| `Queue` | 先进先出（FIFO）或优先级 | PriorityQueue、ArrayDeque |
| `Map` | 键值对、键不可重复 | HashMap、TreeMap、LinkedHashMap |

### 1.2 集合与数组的区别

| 维度 | 数组 | 集合 |
|------|------|------|
| 长度 | 固定，创建后不可变 | 动态扩容 |
| 元素类型 | 基本类型 + 引用类型 | 只能存引用类型（自动装箱） |
| 类型安全 | 编译期检查 | 泛型擦除，运行时不检查 |
| 功能 | 仅下标访问 | 丰富的 API（排序、查找、遍历等） |

---

## 二、ArrayList

### 2.1 底层原理

`ArrayList` 底层是 **`Object[]` 数组**，支持随机访问（`RandomAccess` 标记接口）。

```java
// JDK 8 源码核心字段
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    
    private static final int DEFAULT_CAPACITY = 10;       // 默认初始容量
    transient Object[] elementData;                        // 存储元素的数组
    private int size;                                      // 实际元素个数
}
```

### 2.2 扩容机制（1.5 倍）

```
初始状态（懒初始化）：
  elementData = {} （空数组，第一次 add 时才分配）

第一次 add：
  分配 DEFAULT_CAPACITY = 10 的数组

后续扩容流程：
  add(element)
      │
      ▼
  ensureCapacityInternal(size + 1)
      │
      ▼
  size + 1 > elementData.length ?
      │
  YES ▼
  grow(minCapacity)
      │
      ▼
  newCapacity = oldCapacity + (oldCapacity >> 1)   ← 1.5 倍
      │
      ▼
  elementData = Arrays.copyOf(elementData, newCapacity)
```

```java
// JDK 8 扩容源码
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // 1.5 倍
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

**关键点：**
- 默认初始容量 10，但采用懒初始化，构造时不分配
- 扩容为原来的 **1.5 倍**（`oldCapacity >> 1`）
- 扩容需要 `Arrays.copyOf`，涉及数组拷贝，**开销较大**
- 如果已知大小，建议用 `new ArrayList<>(expectedSize)` 指定初始容量

### 2.3 fail-fast 机制

ArrayList 的迭代器采用 **fail-fast** 策略：在迭代过程中，如果检测到集合被结构性修改（`modCount` 变化），立即抛出 `ConcurrentModificationException`。

```java
// 错误写法：迭代时直接删除
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
for (String s : list) {
    if ("b".equals(s)) {
        list.remove(s); // 抛出 ConcurrentModificationException
    }
}

// 正确写法1：使用迭代器的 remove
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if ("b".equals(it.next())) {
        it.remove(); // 安全删除
    }
}

// 正确写法2：JDK 8 removeIf
list.removeIf("b"::equals);
```

### 2.4 常用操作时间复杂度

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| `get(index)` | O(1) | 数组下标直接访问 |
| `add(element)` | 均摊 O(1) | 尾部追加，偶尔扩容 |
| `add(index, e)` | O(n) | 需要移动后续元素 |
| `remove(index)` | O(n) | 需要移动后续元素 |
| `contains(o)` | O(n) | 遍历查找 |

---

## 三、LinkedList

### 3.1 底层原理

`LinkedList` 底层是**双向链表**，同时实现了 `List` 和 `Deque` 接口，可用作列表、栈和队列。

```java
public class LinkedList<E> extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, java.io.Serializable {
    
    transient int size = 0;
    transient Node<E> first;  // 头节点
    transient Node<E> last;   // 尾节点
    
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
        
        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
}
```

```
双向链表结构：

  null ← [prev|A|next] ⇄ [prev|B|next] ⇄ [prev|C|next] → null
           ↑ first                            ↑ last
```

### 3.2 操作时间复杂度

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| `addFirst` / `addLast` | O(1) | 头尾插入 |
| `get(index)` | O(n) | 需要从头或尾遍历（会根据 index 判断从哪端开始） |
| `add(index, e)` | O(n) | 定位 O(n) + 插入 O(1) |
| `remove(index)` | O(n) | 定位 O(n) + 删除 O(1) |
| `removeFirst` / `removeLast` | O(1) | 头尾删除 |

---

## 四、HashMap

### 4.1 底层数据结构

JDK 8 的 HashMap 采用 **数组 + 链表 + 红黑树** 的结构。

```
HashMap 内部结构（JDK 8）：

  table[] 数组
  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │  ...
  └─┬─┴───┴─┬─┴───┴───┴─┬─┴───┴───┘
    │       │           │
    ▼       ▼           ▼
  [K1,V1] [K3,V3]    [K5,V5]          ← Node 节点
    │       │           │
    ▼       ▼           ▼
  [K2,V2] [K4,V4]    [K6,V6]
                        │
                        ▼
                      当链表长度 ≥ 8 且数组长度 ≥ 64 时
                      转为红黑树（TreeNode）
```

### 4.2 核心参数

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;  // 16，默认数组大小
static final int MAXIMUM_CAPACITY = 1 << 30;          // 最大容量
static final float DEFAULT_LOAD_FACTOR = 0.75f;       // 默认负载因子
static final int TREEIFY_THRESHOLD = 8;               // 链表转红黑树的阈值
static final int UNTREEIFY_THRESHOLD = 6;             // 红黑树退化为链表的阈值
static final int MIN_TREEIFY_CAPACITY = 64;           // 转树时数组最小长度
```

### 4.3 hash 计算（扰动函数）

```java
// JDK 8
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
// 高16位与低16位异或，让高位也参与下标计算，减少碰撞
// 数组下标 = (n - 1) & hash  （n 为数组长度，必须是2的幂）
```

**为什么数组长度必须是 2 的幂？**
- `(n - 1) & hash` 等效于 `hash % n`，但位运算效率更高
- 保证哈希值能均匀分布到每个桶

### 4.4 put 流程

```
put(key, value)
    │
    ▼
① 计算 hash = hash(key)
    │
    ▼
② table 为空？ ──YES──► resize() 初始化
    │ NO
    ▼
③ table[i = (n-1) & hash] 为空？
    │                │
    YES              NO
    ▼                ▼
  直接插入      ④ 判断首节点 key 是否相同
  新节点            │
                 ┌──┴──┐
                YES    NO
                 │      │
                 ▼      ▼
              覆盖旧值  ⑤ 判断节点类型
                        │
                    ┌───┴───┐
                  TreeNode  Node（链表）
                    │         │
                    ▼         ▼
                 红黑树插入   遍历链表
                             │
                          找到相同key？
                          ┌──┴──┐
                         YES    NO
                          │      │
                          ▼      ▼
                       覆盖旧值  尾插法插入
                                 │
                                 ▼
                          链表长度 ≥ 8？
                              │
                             YES
                              ▼
                         treeifyBin()
                         （数组长度 ≥ 64 才转树，否则扩容）
    │
    ▼
⑥ ++size > threshold？ ──YES──► resize() 扩容
```

### 4.5 扩容机制（resize）

扩容时数组大小翻倍（`newCap = oldCap << 1`），每个节点要么留在原位，要么迁移到 `原位 + oldCap` 的位置。

```
扩容迁移规则（JDK 8 优化）：

  oldCap = 16, newCap = 32
  
  对于原链表中的每个节点：
    if (hash & oldCap) == 0  →  放在原索引 j
    if (hash & oldCap) != 0  →  放在索引 j + oldCap
    
  ┌──────────────┐       ┌──────────────┐
  │ table[5]     │       │ table[5]     │  低位链表
  │  A → B → C   │ ───► │  A → C       │
  │              │       │ table[21]    │  高位链表
  └──────────────┘       │  B           │
                         └──────────────┘
```

### 4.6 JDK 7 vs JDK 8 区别

| 维度 | JDK 7 | JDK 8 |
|------|-------|-------|
| 数据结构 | 数组 + 链表 | 数组 + 链表 + 红黑树 |
| 插入方式 | **头插法** | **尾插法** |
| 扩容 | 先扩容再插入 | 先插入再扩容 |
| hash 计算 | 4 次位运算 + 5 次异或 | 1 次位运算 + 1 次异或 |
| 线程安全 | 头插法导致死循环（环形链表） | 尾插法不会死循环，但仍线程不安全 |

---

## 五、ConcurrentHashMap

### 5.1 JDK 7 实现：Segment 分段锁

```
ConcurrentHashMap（JDK 7）：

  Segment[] （默认 16 个，每个 Segment 继承 ReentrantLock）
  ┌─────────┬─────────┬─────────┬─────────┐
  │ Seg[0]  │ Seg[1]  │ Seg[2]  │  ...    │
  └────┬────┴────┬────┴────┬────┴─────────┘
       │         │         │
       ▼         ▼         ▼
  HashEntry[]  HashEntry[]  HashEntry[]
  ┌──┬──┐    ┌──┬──┐    ┌──┬──┐
  │  │  │    │  │  │    │  │  │
  └──┴──┘    └──┴──┘    └──┴──┘

  并发度 = Segment 数量（默认 16）
  不同 Segment 可同时操作，互不阻塞
```

### 5.2 JDK 8 实现：CAS + synchronized

```
ConcurrentHashMap（JDK 8）：

  Node[] table
  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │
  └─┬─┴───┴─┬─┴───┴───┴─┬─┴───┴───┘
    │       │           │
    ▼       ▼           ▼
  Node    Node        Node
    │                   │
    ▼                   ▼
  Node              TreeNode（红黑树）

  空桶插入：CAS（无锁）
  非空桶操作：synchronized(桶头节点)
  并发度 = 数组长度（远大于16）
```

### 5.3 JDK 8 put 流程

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // key 和 value 都不能为 null（与 HashMap 不同）
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();                        // 初始化数组
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<>(hash, key, value)))
                break;                                // CAS 插入空桶
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);               // 协助扩容
        else {
            synchronized (f) {                        // 锁住桶头节点
                // 链表或红黑树操作...
            }
        }
    }
    addCount(1L, binCount);                           // 更新计数
    return null;
}
```

### 5.4 size() 计算

JDK 8 的 `ConcurrentHashMap` 使用 **baseCount + CounterCell[]** 来统计元素个数，类似 `LongAdder` 的思路：

```
size() 计算原理：

  baseCount（基础计数器）
      │
      ├─ 无竞争时：CAS 更新 baseCount
      │
      └─ 有竞争时：分散到 CounterCell[] 数组
                   每个线程映射到不同的 Cell
                   
  size = baseCount + Σ counterCells[i].value
```

### 5.5 JDK 7 vs JDK 8 对比

| 维度 | JDK 7 | JDK 8 |
|------|-------|-------|
| 数据结构 | Segment[] + HashEntry[] + 链表 | Node[] + 链表/红黑树 |
| 锁粒度 | Segment 级别（分段锁） | 桶级别（更细） |
| 锁实现 | ReentrantLock | CAS + synchronized |
| 并发度 | 固定（默认 16） | 动态（= 数组长度） |
| 链表优化 | 无 | 长度 ≥ 8 转红黑树 |
| key/value 允许 null | 不允许 | 不允许 |

---

## 六、TreeMap 与 LinkedHashMap

### 6.1 TreeMap（红黑树）

`TreeMap` 基于**红黑树**实现，键按自然顺序或自定义 Comparator 排序，所有操作时间复杂度 O(log n)。

```java
// 自然排序
TreeMap<Integer, String> map = new TreeMap<>();
map.put(3, "C");
map.put(1, "A");
map.put(2, "B");
System.out.println(map); // {1=A, 2=B, 3=C}

// 自定义排序（降序）
TreeMap<Integer, String> desc = new TreeMap<>(Comparator.reverseOrder());
desc.put(3, "C");
desc.put(1, "A");
desc.put(2, "B");
System.out.println(desc); // {3=C, 2=B, 1=A}

// 范围查询
map.subMap(1, 3);   // {1=A, 2=B}（左闭右开）
map.headMap(2);      // {1=A}
map.tailMap(2);      // {2=B, 3=C}
map.firstKey();      // 1
map.lastKey();       // 3
```

### 6.2 LinkedHashMap（有序 HashMap）

`LinkedHashMap` 在 HashMap 的基础上维护了一条**双向链表**，支持两种顺序：

```java
// 插入顺序（默认）
LinkedHashMap<String, Integer> insertOrder = new LinkedHashMap<>();
insertOrder.put("B", 2);
insertOrder.put("A", 1);
insertOrder.put("C", 3);
// 遍历顺序：B → A → C

// 访问顺序（accessOrder = true）→ LRU 缓存基础
LinkedHashMap<String, Integer> accessOrder = new LinkedHashMap<>(16, 0.75f, true);
accessOrder.put("A", 1);
accessOrder.put("B", 2);
accessOrder.put("C", 3);
accessOrder.get("A");  // 访问 A，A 移到链表尾部
// 遍历顺序：B → C → A
```

### 6.3 基于 LinkedHashMap 实现 LRU 缓存

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true); // accessOrder = true
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity; // 超出容量时移除最久未访问的元素
    }

    public static void main(String[] args) {
        LRUCache<String, Integer> cache = new LRUCache<>(3);
        cache.put("A", 1);
        cache.put("B", 2);
        cache.put("C", 3);
        cache.get("A");       // 访问 A
        cache.put("D", 4);   // 容量超限，移除最久未访问的 B
        System.out.println(cache); // {C=3, A=1, D=4}
    }
}
```

---

## 七、HashSet

### 7.1 底层实现

`HashSet` 底层就是 **HashMap**，元素作为 key，value 统一用一个 `PRESENT` 占位对象。

```java
public class HashSet<E> extends AbstractSet<E>
        implements Set<E>, Cloneable, java.io.Serializable {
    
    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object(); // 所有 value 共用

    public HashSet() {
        map = new HashMap<>();
    }

    public boolean add(E e) {
        return map.put(e, PRESENT) == null; // put 返回 null 说明是新增
    }

    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }
}
```

**推论：**
- HashSet 判断重复依赖 `hashCode()` + `equals()`
- 自定义对象放入 HashSet 时，**必须同时重写** `hashCode()` 和 `equals()`

---

## 八、ArrayList vs LinkedList 性能对比

| 维度 | ArrayList | LinkedList |
|------|-----------|------------|
| 底层结构 | 动态数组 | 双向链表 |
| 随机访问 | O(1)，实现 RandomAccess | O(n)，需要遍历 |
| 头部插入 | O(n)，需要移动元素 | O(1) |
| 尾部插入 | 均摊 O(1) | O(1) |
| 中间插入 | O(n) | O(n)（定位 O(n) + 插入 O(1)） |
| 内存占用 | 连续内存，紧凑 | 每个节点额外存储 prev/next 指针 |
| CPU 缓存 | 友好（数组连续存储） | 不友好（节点分散在堆中） |
| 迭代性能 | 更快（缓存局部性好） | 较慢（指针跳跃） |

**实际结论：** 绝大多数场景优先选 `ArrayList`。即使在中间插入删除，由于 CPU 缓存局部性，ArrayList 往往也比 LinkedList 快。LinkedList 仅在频繁头部插入/删除的场景下有优势（此时应优先考虑 `ArrayDeque`）。

---

## 九、HashMap 线程不安全的表现

### 9.1 JDK 7：死循环（环形链表）

JDK 7 的 HashMap 扩容使用**头插法**迁移链表，多线程并发扩容时可能导致链表成环，后续 `get()` 陷入死循环。

```
线程1 和 线程2 同时扩容，原链表：A → B → null

线程1 暂停在 B.next = A 之前
线程2 完成扩容：B → A → null（头插法反转了）

线程1 恢复：
  A.next = B（线程1的局部变量）
  B.next = A（线程2已设置）
  
结果：A ⇄ B 形成环！
```

### 9.2 JDK 8：数据覆盖

JDK 8 改为尾插法，不会死循环，但仍有数据覆盖问题：

```java
// 两个线程同时 put，hash 到同一个空桶
// 线程1：判断 table[i] == null，准备插入，被挂起
// 线程2：判断 table[i] == null，直接插入
// 线程1 恢复：直接插入，覆盖线程2 的数据
```

### 9.3 解决方案

| 方案 | 说明 |
|------|------|
| `Collections.synchronizedMap()` | 每个方法加 synchronized，性能差 |
| `Hashtable` | 同上，过时不推荐 |
| **`ConcurrentHashMap`** | **推荐**，CAS + synchronized，高并发性能好 |

---

## 十、大厂常见面试题

### Q1：ArrayList 和 Array 有什么区别？

**答：**

ArrayList 是动态数组，长度可变，只能存引用类型；Array 是固定长度数组，可存基本类型和引用类型。ArrayList 提供丰富的 API（add/remove/contains 等），Array 只有下标访问。性能上 Array 更快（无装箱、无扩容开销），ArrayList 更灵活。

---

### Q2：HashMap 的 key 可以是 null 吗？ConcurrentHashMap 呢？

**答：**

- **HashMap**：允许一个 null key 和多个 null value。null key 的 hash 值固定为 0，存放在 `table[0]`。
- **ConcurrentHashMap**：key 和 value 都**不允许 null**。原因是在并发环境下，`get()` 返回 null 无法区分是 key 不存在还是 value 为 null，会产生歧义。

---

### Q3：HashMap 为什么用红黑树而不是 AVL 树？

**答：**

| 维度 | 红黑树 | AVL 树 |
|------|--------|--------|
| 平衡度 | 近似平衡 | 严格平衡 |
| 查找 | O(log n)，略慢 | O(log n)，略快 |
| 插入/删除 | 旋转少（最多 2-3 次） | 旋转多（可能 O(log n) 次） |
| 适用场景 | 插入删除多 | 查找多，修改少 |

HashMap 的场景是读写都频繁，红黑树在插入删除时性能更好，综合表现更优。

---

### Q4：ConcurrentHashMap 为什么不允许 null key/value？

**答：**

Doug Lea 的设计哲学：在并发场景下，`map.get(key)` 返回 null 有二义性——key 不存在 vs value 就是 null。单线程的 HashMap 可以用 `containsKey()` 二次确认，但并发场景下 `containsKey()` 和 `get()` 之间可能被其他线程修改，无法保证一致性。

---

### Q5：HashMap 的容量为什么必须是 2 的幂？

**答：**

1. **高效取模**：`(n - 1) & hash` 等效于 `hash % n`，位运算比取模快。
2. **均匀分布**：当 n 是 2 的幂时，`n - 1` 的二进制全是 1（如 15 = 0b1111），与 hash 做 & 运算能保证结果均匀分布在 `[0, n-1]`。
3. **扩容优化**：扩容时只需看 `hash & oldCap` 的一个 bit 即可判断节点是否需要迁移。

如果传入的容量不是 2 的幂，HashMap 会向上取最近的 2 的幂（`tableSizeFor` 方法）。

---

### Q6：如何选择 HashMap 的初始容量？

**答：**

公式：`initialCapacity = (int)(expectedSize / loadFactor) + 1`

例如预计存 100 个元素：`100 / 0.75 + 1 ≈ 134`，HashMap 会向上取到 256。

Guava 提供了工具方法 `Maps.newHashMapWithExpectedSize(100)` 自动计算。

---

*参考资料：JDK 8 源码、《Java编程思想》、美团技术博客 HashMap 源码分析*
