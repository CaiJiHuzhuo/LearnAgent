# 03 - Java 8 新特性

## 目录

1. [Lambda 表达式](#一lambda-表达式)
2. [四大函数式接口](#二四大函数式接口)
3. [Stream API](#三stream-api)
4. [Optional](#四optional)
5. [接口默认方法与静态方法](#五接口默认方法与静态方法)
6. [新日期时间 API](#六新日期时间-api)
7. [CompletableFuture](#七completablefuture)
8. [实战代码示例](#八实战代码示例)
9. [大厂常见面试题](#九大厂常见面试题)

---

## 一、Lambda 表达式

### 1.1 基本语法

Lambda 表达式是匿名函数的简洁写法，用于实现**函数式接口**（只有一个抽象方法的接口）。

```java
// 语法：(参数列表) -> { 方法体 }

// 完整形式
(int a, int b) -> { return a + b; }

// 省略参数类型（类型推断）
(a, b) -> { return a + b; }

// 单行表达式（省略 return 和大括号）
(a, b) -> a + b

// 单参数（省略小括号）
x -> x * 2

// 无参数
() -> System.out.println("Hello")
```

### 1.2 Lambda 与匿名内部类对比

```java
// 匿名内部类
Comparator<String> comp1 = new Comparator<String>() {
    @Override
    public int compare(String s1, String s2) {
        return s1.length() - s2.length();
    }
};

// Lambda 表达式
Comparator<String> comp2 = (s1, s2) -> s1.length() - s2.length();

// 方法引用
Comparator<Integer> comp3 = Integer::compareTo;
```

### 1.3 方法引用

方法引用是 Lambda 的进一步简化，当 Lambda 体只是调用一个现有方法时可以使用。

| 类型 | 语法 | Lambda 等价 |
|------|------|------------|
| 静态方法引用 | `类名::staticMethod` | `x -> 类名.staticMethod(x)` |
| 实例方法引用 | `对象::instanceMethod` | `x -> 对象.instanceMethod(x)` |
| 类的实例方法引用 | `类名::instanceMethod` | `(obj, x) -> obj.instanceMethod(x)` |
| 构造方法引用 | `类名::new` | `x -> new 类名(x)` |

```java
// 静态方法引用
Function<String, Integer> parser = Integer::parseInt;

// 实例方法引用
String str = "Hello";
Supplier<Integer> len = str::length;

// 类的实例方法引用
Function<String, String> upper = String::toUpperCase;

// 构造方法引用
Supplier<ArrayList<String>> listFactory = ArrayList::new;
Function<Integer, int[]> arrayFactory = int[]::new;
```

### 1.4 Lambda 底层实现

Lambda 并不是语法糖转为匿名内部类，而是使用 **`invokedynamic`** 指令在运行时动态生成实现类：

```
编译时：生成 invokedynamic 指令 + bootstrap 方法
  ↓
首次调用：LambdaMetafactory 生成实现类（ASM 字节码生成）
  ↓
后续调用：直接调用已生成的实现类

优势：
  - 避免大量匿名内部类文件（.class）
  - JVM 可以更好地优化（内联等）
  - 比匿名内部类更高效
```

---

## 二、四大函数式接口

Java 8 在 `java.util.function` 包下定义了大量函数式接口，最核心的四个：

| 接口 | 方法签名 | 用途 | 典型场景 |
|------|---------|------|---------|
| `Supplier<T>` | `T get()` | 无入参，返回结果 | 工厂方法、懒加载 |
| `Consumer<T>` | `void accept(T t)` | 接收参数，无返回 | 遍历处理、日志输出 |
| `Function<T,R>` | `R apply(T t)` | 接收参数，返回结果 | 数据转换、映射 |
| `Predicate<T>` | `boolean test(T t)` | 接收参数，返回布尔 | 过滤、条件判断 |

### 2.1 Supplier（供给型）

```java
// 无参数，返回结果
Supplier<String> supplier = () -> "Hello World";
System.out.println(supplier.get()); // Hello World

// 懒加载场景
Supplier<Connection> connSupplier = () -> DriverManager.getConnection(url);
// 只在真正需要时才创建连接
Connection conn = connSupplier.get();
```

### 2.2 Consumer（消费型）

```java
// 接收参数，执行操作
Consumer<String> printer = System.out::println;
printer.accept("Hello"); // Hello

// andThen 组合
Consumer<String> upperPrinter = s -> System.out.println(s.toUpperCase());
printer.andThen(upperPrinter).accept("hello");
// 输出：
// hello
// HELLO
```

### 2.3 Function（函数型）

```java
// 输入 T，输出 R
Function<String, Integer> strToInt = Integer::parseInt;
System.out.println(strToInt.apply("123")); // 123

// compose：先执行参数函数，再执行当前函数
Function<Integer, Integer> doubleIt = x -> x * 2;
Function<String, Integer> parseAndDouble = doubleIt.compose(strToInt);
System.out.println(parseAndDouble.apply("5")); // 10

// andThen：先执行当前函数，再执行参数函数
Function<String, String> parseAndFormat = strToInt.andThen(i -> "结果: " + i);
System.out.println(parseAndFormat.apply("42")); // 结果: 42
```

### 2.4 Predicate（断言型）

```java
// 返回 boolean
Predicate<String> isNotEmpty = s -> s != null && !s.isEmpty();
System.out.println(isNotEmpty.test("hello")); // true
System.out.println(isNotEmpty.test(""));      // false

// 组合操作
Predicate<Integer> isPositive = x -> x > 0;
Predicate<Integer> isEven = x -> x % 2 == 0;

Predicate<Integer> isPositiveEven = isPositive.and(isEven);
Predicate<Integer> isNegativeOrOdd = isPositive.negate().or(isEven.negate());

System.out.println(isPositiveEven.test(4));  // true
System.out.println(isPositiveEven.test(-2)); // false
```

### 2.5 扩展函数式接口

| 接口 | 说明 |
|------|------|
| `BiFunction<T,U,R>` | 两个参数，一个返回值 |
| `BiConsumer<T,U>` | 两个参数，无返回值 |
| `BiPredicate<T,U>` | 两个参数，返回布尔 |
| `UnaryOperator<T>` | 输入和输出类型相同（`Function<T,T>`） |
| `BinaryOperator<T>` | 两个同类型参数，返回同类型（`BiFunction<T,T,T>`） |

---

## 三、Stream API

### 3.1 什么是 Stream

Stream 是对集合数据进行函数式操作的抽象管道，**不存储数据**，只是对数据源的一系列操作描述。

```
数据源 → Stream → 中间操作(过滤/映射/排序) → 终端操作(收集/统计/遍历)
                   (惰性求值，不执行)          (触发计算)
```

### 3.2 创建 Stream

```java
// 1. 从集合
List<String> list = Arrays.asList("a", "b", "c");
Stream<String> s1 = list.stream();        // 顺序流
Stream<String> s2 = list.parallelStream(); // 并行流

// 2. 从数组
int[] arr = {1, 2, 3};
IntStream s3 = Arrays.stream(arr);

// 3. Stream.of
Stream<String> s4 = Stream.of("x", "y", "z");

// 4. 无限流
Stream<Integer> s5 = Stream.iterate(0, n -> n + 2); // 0, 2, 4, 6, ...
Stream<Double> s6 = Stream.generate(Math::random);   // 随机数流
```

### 3.3 中间操作（Intermediate Operations）

中间操作返回 Stream，可以链式调用，**惰性求值**。

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David", "Amy");

// filter：过滤
names.stream().filter(s -> s.startsWith("A")); // Alice, Amy

// map：映射转换
names.stream().map(String::toUpperCase); // ALICE, BOB, ...

// flatMap：扁平化映射
List<List<Integer>> nested = Arrays.asList(
    Arrays.asList(1, 2), Arrays.asList(3, 4));
nested.stream().flatMap(Collection::stream); // 1, 2, 3, 4

// sorted：排序
names.stream().sorted();                                // 自然排序
names.stream().sorted(Comparator.comparingInt(String::length)); // 按长度排

// distinct：去重
Stream.of(1, 2, 2, 3, 3).distinct(); // 1, 2, 3

// limit / skip
names.stream().limit(3);  // 前3个
names.stream().skip(2);   // 跳过前2个

// peek：调试（不修改元素）
names.stream().peek(System.out::println).collect(Collectors.toList());
```

### 3.4 终端操作（Terminal Operations）

终端操作触发实际计算，一个 Stream 只能有一个终端操作。

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

// forEach：遍历
names.stream().forEach(System.out::println);

// collect：收集为集合
List<String> filtered = names.stream()
    .filter(s -> s.length() > 3)
    .collect(Collectors.toList());

// toMap
Map<String, Integer> nameLength = names.stream()
    .collect(Collectors.toMap(Function.identity(), String::length));

// groupingBy：分组
Map<Integer, List<String>> grouped = names.stream()
    .collect(Collectors.groupingBy(String::length));
// {3=[Bob], 5=[Alice], 7=[Charlie]}

// joining：拼接
String joined = names.stream().collect(Collectors.joining(", "));
// "Alice, Bob, Charlie"

// reduce：聚合
Optional<String> longest = names.stream()
    .reduce((s1, s2) -> s1.length() >= s2.length() ? s1 : s2);

// 统计
long count = names.stream().count();
boolean anyMatch = names.stream().anyMatch(s -> s.startsWith("A"));
boolean allMatch = names.stream().allMatch(s -> s.length() > 2);
Optional<String> first = names.stream().findFirst();
```

### 3.5 并行流（Parallel Stream）

```java
// 方式一：parallelStream()
long count = list.parallelStream().filter(x -> x > 0).count();

// 方式二：stream().parallel()
long sum = IntStream.rangeClosed(1, 1_000_000).parallel().sum();
```

**并行流注意事项：**

| 要点 | 说明 |
|------|------|
| 底层实现 | ForkJoinPool.commonPool()，默认线程数 = CPU 核心数 - 1 |
| 线程安全 | 不要在 Lambda 中修改共享可变状态 |
| 数据量 | 数据量小时并行流反而更慢（线程调度开销） |
| 数据源 | ArrayList、数组适合并行（支持随机拆分），LinkedList 不适合 |
| 操作类型 | 无状态操作（filter/map）适合并行，有状态操作（sorted/distinct）可能较慢 |

---

## 四、Optional

### 4.1 为什么需要 Optional

Optional 用于优雅处理 **null**，避免 `NullPointerException`。它强制调用者思考空值情况。

```java
// 传统写法：大量 null 检查
String city = null;
if (user != null) {
    Address address = user.getAddress();
    if (address != null) {
        city = address.getCity();
    }
}
if (city == null) city = "Unknown";

// Optional 写法
String city = Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("Unknown");
```

### 4.2 创建 Optional

```java
// 1. of：值不能为 null，否则抛 NPE
Optional<String> opt1 = Optional.of("hello");

// 2. ofNullable：值可以为 null
Optional<String> opt2 = Optional.ofNullable(null); // 空 Optional

// 3. empty：创建空 Optional
Optional<String> opt3 = Optional.empty();
```

### 4.3 链式调用

```java
Optional<String> name = Optional.ofNullable(getName());

// map：转换值（如果存在）
Optional<Integer> length = name.map(String::length);

// flatMap：避免 Optional 嵌套
Optional<String> city = Optional.ofNullable(user)
    .flatMap(User::getAddress)   // getAddress 返回 Optional<Address>
    .flatMap(Address::getCity);  // getCity 返回 Optional<String>

// filter：条件过滤
Optional<String> longName = name.filter(s -> s.length() > 5);
```

### 4.4 获取值

```java
Optional<String> opt = Optional.ofNullable(getValue());

// orElse：为空时返回默认值（无论是否为空都会执行参数表达式）
String val1 = opt.orElse("default");

// orElseGet：为空时通过 Supplier 获取（惰性求值，推荐）
String val2 = opt.orElseGet(() -> computeDefault());

// orElseThrow：为空时抛异常
String val3 = opt.orElseThrow(() -> new RuntimeException("值不存在"));

// ifPresent：值存在时执行操作
opt.ifPresent(System.out::println);

// JDK 11：ifPresentOrElse
opt.ifPresentOrElse(
    val -> System.out.println("值: " + val),
    () -> System.out.println("空值")
);
```

### 4.5 最佳实践

```java
// ✅ 用于方法返回值，表明可能为空
public Optional<User> findUserById(Long id) {
    User user = userMapper.selectById(id);
    return Optional.ofNullable(user);
}

// ❌ 不要用于字段
class User {
    private Optional<String> name; // 不推荐！
}

// ❌ 不要用于方法参数
void process(Optional<String> name) { } // 不推荐！

// ❌ 不要 isPresent + get（和 null 检查没区别）
if (opt.isPresent()) {
    String val = opt.get(); // 不推荐！
}

// ✅ 使用链式调用
String result = opt.map(String::toUpperCase).orElse("DEFAULT");
```

---

## 五、接口默认方法与静态方法

Java 8 允许接口中定义**默认方法**（`default`）和**静态方法**（`static`），解决接口演进问题。

### 5.1 默认方法

```java
public interface Collection<E> {
    // 已有抽象方法
    boolean add(E e);
    
    // Java 8 新增默认方法，不会破坏已有实现
    default Stream<E> stream() {
        return StreamSupport.stream(spliterator(), false);
    }
    
    default boolean removeIf(Predicate<? super E> filter) {
        boolean removed = false;
        Iterator<E> each = iterator();
        while (each.hasNext()) {
            if (filter.test(each.next())) {
                each.remove();
                removed = true;
            }
        }
        return removed;
    }
}
```

### 5.2 冲突解决规则

当一个类实现多个接口，且这些接口有同名默认方法时：

```java
interface A {
    default String hello() { return "A"; }
}

interface B {
    default String hello() { return "B"; }
}

// 必须手动解决冲突
class C implements A, B {
    @Override
    public String hello() {
        return A.super.hello(); // 明确选择 A 的实现
    }
}
```

**优先级规则：**
1. **类中的方法优先级最高**（类或父类中定义的方法 > 接口默认方法）
2. **子接口优先**（更具体的接口 > 更通用的接口）
3. **都不满足 → 编译错误**，必须手动覆写

### 5.3 静态方法

```java
public interface Comparator<T> {
    // 静态工厂方法
    static <T extends Comparable<? super T>> Comparator<T> naturalOrder() {
        return (Comparator<T>) Comparators.NaturalOrderComparator.INSTANCE;
    }
    
    static <T extends Comparable<? super T>> Comparator<T> reverseOrder() {
        return Collections.reverseOrder();
    }
}

// 使用
Comparator<String> comp = Comparator.naturalOrder();
```

---

## 六、新日期时间 API

Java 8 引入了 `java.time` 包，替代饱受诟病的 `Date` 和 `Calendar`。新 API 的特点：**不可变**、**线程安全**、**设计清晰**。

### 6.1 核心类

| 类 | 说明 | 示例 |
|----|------|------|
| `LocalDate` | 日期（年月日） | 2024-01-15 |
| `LocalTime` | 时间（时分秒纳秒） | 14:30:00 |
| `LocalDateTime` | 日期 + 时间 | 2024-01-15T14:30:00 |
| `Instant` | 时间戳（UTC） | 1705312200 |
| `Duration` | 时间间隔（时分秒） | PT1H30M |
| `Period` | 日期间隔（年月日） | P1Y2M3D |
| `ZonedDateTime` | 带时区的日期时间 | 2024-01-15T14:30+08:00[Asia/Shanghai] |

### 6.2 常用操作

```java
// 创建
LocalDate date = LocalDate.now();                      // 当前日期
LocalDate birthday = LocalDate.of(1995, 6, 15);        // 指定日期
LocalDateTime dateTime = LocalDateTime.now();            // 当前日期时间
Instant timestamp = Instant.now();                       // 当前时间戳

// 解析
LocalDate parsed = LocalDate.parse("2024-01-15");
LocalDateTime parsedDt = LocalDateTime.parse("2024-01-15T14:30:00");

// 日期计算（返回新对象，不可变）
LocalDate tomorrow = date.plusDays(1);
LocalDate lastMonth = date.minusMonths(1);
LocalDate nextYear = date.plusYears(1);

// 日期比较
boolean isBefore = date.isBefore(tomorrow);  // true
boolean isAfter = date.isAfter(tomorrow);    // false

// 日期间隔
Period period = Period.between(birthday, date);
System.out.println(period.getYears() + " 年 " + period.getMonths() + " 月");

Duration duration = Duration.between(
    LocalTime.of(9, 0), LocalTime.of(17, 30));
System.out.println(duration.toHours() + " 小时 " + duration.toMinutesPart() + " 分");
```

### 6.3 DateTimeFormatter

```java
// 预定义格式
String s1 = dateTime.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);

// 自定义格式
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String s2 = dateTime.format(fmt);               // "2024-01-15 14:30:00"
LocalDateTime dt = LocalDateTime.parse(s2, fmt); // 解析

// 线程安全！对比 SimpleDateFormat（线程不安全）
// DateTimeFormatter 可以作为 static final 全局使用
public static final DateTimeFormatter FORMATTER =
    DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
```

### 6.4 与旧 API 互转

```java
// Date → LocalDateTime
Date date = new Date();
LocalDateTime ldt = date.toInstant()
    .atZone(ZoneId.systemDefault())
    .toLocalDateTime();

// LocalDateTime → Date
Date newDate = Date.from(
    ldt.atZone(ZoneId.systemDefault()).toInstant());

// Timestamp → LocalDateTime
Timestamp ts = Timestamp.valueOf(ldt);
LocalDateTime fromTs = ts.toLocalDateTime();
```

---

## 七、CompletableFuture

CompletableFuture 是 Java 8 引入的异步编程核心类，支持链式调用和组合操作。**详细内容请参考并发模块**，此处仅列出核心用法速查。

```java
// 异步执行（无返回值）
CompletableFuture<Void> f1 = CompletableFuture.runAsync(() -> {
    System.out.println("异步任务");
});

// 异步执行（有返回值）
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> {
    return "结果";
});

// 链式转换
CompletableFuture<Integer> f3 = f2.thenApply(String::length);

// 组合多个异步任务
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2);
CompletableFuture<Object> any = CompletableFuture.anyOf(f1, f2);

// 异常处理
f2.exceptionally(ex -> "默认值")
  .thenAccept(System.out::println);
```

---

## 八、实战代码示例

### 8.1 Stream 综合实战：电商订单分析

```java
@Data
@AllArgsConstructor
class Order {
    private String orderId;
    private String userId;
    private String product;
    private double amount;
    private LocalDateTime createTime;
    private String status; // PAID, SHIPPED, COMPLETED, CANCELLED
}

public class OrderAnalysis {
    public static void main(String[] args) {
        List<Order> orders = getOrders();
        
        // 1. 统计已完成订单的总金额
        double totalAmount = orders.stream()
            .filter(o -> "COMPLETED".equals(o.getStatus()))
            .mapToDouble(Order::getAmount)
            .sum();
        
        // 2. 按用户分组，统计每人的订单数
        Map<String, Long> orderCountByUser = orders.stream()
            .collect(Collectors.groupingBy(Order::getUserId, Collectors.counting()));
        
        // 3. 找出消费最高的前3名用户
        List<Map.Entry<String, Double>> topSpenders = orders.stream()
            .filter(o -> !"CANCELLED".equals(o.getStatus()))
            .collect(Collectors.groupingBy(
                Order::getUserId,
                Collectors.summingDouble(Order::getAmount)))
            .entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(3)
            .collect(Collectors.toList());
        
        // 4. 按月份统计订单金额
        Map<Integer, Double> monthlyAmount = orders.stream()
            .collect(Collectors.groupingBy(
                o -> o.getCreateTime().getMonthValue(),
                Collectors.summingDouble(Order::getAmount)));
        
        // 5. 生成订单汇总报告
        String report = orders.stream()
            .collect(Collectors.groupingBy(Order::getStatus, Collectors.counting()))
            .entrySet().stream()
            .map(e -> e.getKey() + ": " + e.getValue() + " 单")
            .collect(Collectors.joining(" | "));
        System.out.println("订单汇总: " + report);
    }
}
```

### 8.2 Optional 实战：多层嵌套安全取值

```java
@Data
class Company {
    private String name;
    private Department department;
}

@Data
class Department {
    private String name;
    private Optional<Manager> manager;
}

@Data
class Manager {
    private String name;
    private Optional<String> email;
}

// 安全获取公司经理的邮箱
public String getManagerEmail(Company company) {
    return Optional.ofNullable(company)
        .map(Company::getDepartment)
        .flatMap(Department::getManager)
        .flatMap(Manager::getEmail)
        .orElse("无邮箱信息");
}
```

### 8.3 函数式接口实战：策略模式简化

```java
// 传统策略模式需要定义多个策略类
// 用函数式接口，一行搞定

// 定义验证器
Map<String, Predicate<String>> validators = new HashMap<>();
validators.put("非空", s -> s != null && !s.isEmpty());
validators.put("长度", s -> s.length() >= 6 && s.length() <= 20);
validators.put("数字", s -> s.matches(".*\\d.*"));
validators.put("字母", s -> s.matches(".*[a-zA-Z].*"));

// 密码校验：所有规则都通过
public boolean validatePassword(String password) {
    return validators.values().stream()
        .allMatch(validator -> validator.test(password));
}

// 获取未通过的规则名
public List<String> getFailedRules(String password) {
    return validators.entrySet().stream()
        .filter(e -> !e.getValue().test(password))
        .map(Map.Entry::getKey)
        .collect(Collectors.toList());
}
```

---

## 九、大厂常见面试题

### Q1：Lambda 表达式和匿名内部类有什么区别？

**答：**

| 维度 | Lambda | 匿名内部类 |
|------|--------|-----------|
| 接口要求 | 只能用于函数式接口（一个抽象方法） | 可用于任何接口/抽象类 |
| `this` 指向 | 外层类实例 | 匿名内部类自身 |
| 编译结果 | `invokedynamic` + 运行时生成 | 生成独立的 `.class` 文件 |
| 性能 | 更好（JVM 可优化） | 较差（类加载开销） |
| 变量捕获 | 只能捕获 effectively final 变量 | 同左 |

---

### Q2：Stream 的惰性求值是什么意思？

**答：**

中间操作（filter/map/sorted 等）不会立即执行，只是记录操作；只有遇到**终端操作**（collect/forEach/count 等）时才触发实际计算。

好处：
1. **短路优化**：`findFirst` 找到第一个就停止
2. **融合操作**：多个中间操作合并为一次遍历
3. **按需计算**：无限流可以通过 `limit` 控制

```java
// 以下代码不会打印任何内容（没有终端操作）
Stream.of(1, 2, 3).filter(x -> {
    System.out.println("过滤: " + x);
    return x > 1;
}).map(x -> {
    System.out.println("映射: " + x);
    return x * 2;
});
// 加上 .collect(Collectors.toList()) 才会执行
```

---

### Q3：Stream 和 for 循环哪个性能好？

**答：**

- **小数据量（< 1000）**：for 循环通常更快（Stream 有创建管道的开销）
- **大数据量**：差异不大，Stream 可读性更好
- **并行场景**：parallelStream 在大数据量时有优势
- **特殊优化**：基本类型建议用 `IntStream`/`LongStream` 避免装箱

结论：优先选择可读性好的写法，性能瓶颈出现时再优化。Stream 的最大价值是**声明式编程**，提高代码可读性和可维护性。

---

### Q4：Optional 的 orElse 和 orElseGet 有什么区别？

**答：**

```java
// orElse：无论 Optional 是否为空，都会执行参数
Optional.of("hello").orElse(generateDefault()); // generateDefault() 被调用了！

// orElseGet：只有 Optional 为空时才执行 Supplier
Optional.of("hello").orElseGet(() -> generateDefault()); // generateDefault() 不会调用
```

当默认值的计算代价高（如数据库查询、远程调用）时，**必须用 `orElseGet`** 避免不必要的计算。

---

### Q5：新的日期时间 API 比 Date/Calendar 好在哪里？

**答：**

| 维度 | Date/Calendar | java.time |
|------|--------------|-----------|
| 可变性 | 可变（线程不安全） | **不可变**（线程安全） |
| 月份 | 从 0 开始（0=一月） | 从 1 开始（1=一月） |
| 格式化 | SimpleDateFormat（线程不安全） | DateTimeFormatter（线程安全） |
| 时区 | 设计混乱 | 清晰的 `ZonedDateTime`/`ZoneId` |
| API 设计 | 臃肿、命名不清 | 清晰、链式调用 |
| 日期计算 | 复杂 | `plusDays`/`minusMonths` 简洁 |

---

### Q6：parallelStream 使用什么线程池？如何自定义？

**答：**

默认使用 `ForkJoinPool.commonPool()`，线程数 = CPU 核心数 - 1。

自定义方式：

```java
// 方式一：全局设置（影响所有 parallelStream）
System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "16");

// 方式二：提交到自定义 ForkJoinPool（推荐）
ForkJoinPool customPool = new ForkJoinPool(8);
List<Integer> result = customPool.submit(() ->
    list.parallelStream()
        .filter(x -> x > 0)
        .collect(Collectors.toList())
).get();
customPool.shutdown();
```

---

*参考资料：《Java 8 实战》、Oracle Java 8 官方文档、Brian Goetz - State of the Lambda*
