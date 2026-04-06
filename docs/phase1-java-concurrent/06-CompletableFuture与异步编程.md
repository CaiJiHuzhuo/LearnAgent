# 06 - CompletableFuture 与异步编程

## 目录

1. [Future 的局限性](#一future-的局限性)
2. [CompletableFuture 基础](#二completablefuture-基础)
3. [异步任务创建](#三异步任务创建)
4. [结果转换与处理](#四结果转换与处理)
5. [任务组合](#五任务组合)
6. [异常处理](#六异常处理)
7. [线程池配置最佳实践](#七线程池配置最佳实践)
8. [实战：异步并行查询](#八实战异步并行查询)
9. [大厂常见面试题](#九大厂常见面试题)

---

## 一、Future 的局限性

JDK 5 引入的 `Future` 接口虽然可以获取异步任务结果，但有明显局限：

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<String> future = pool.submit(() -> "Hello");

// 问题1：get() 是阻塞的，调用方线程被阻塞
String result = future.get();

// 问题2：不能链式回调（任务完成后自动触发下一步）
// 问题3：多个 Future 无法方便地组合（等待全部/任一完成）
// 问题4：没有异常处理回调机制
```

`CompletableFuture`（JDK 8）解决了以上所有问题，同时实现了 `Future` 和 `CompletionStage` 两个接口。

---

## 二、CompletableFuture 基础

### 2.1 类结构

```
CompletableFuture<T>
    implements Future<T>        // 可获取结果
    implements CompletionStage<T> // 可链式组合、回调
```

### 2.2 手动完成

```java
// 手动创建一个 CompletableFuture
CompletableFuture<String> cf = new CompletableFuture<>();

// 在另一线程完成
new Thread(() -> {
    try {
        Thread.sleep(1000);
        cf.complete("完成结果");           // 正常完成
        // cf.completeExceptionally(new RuntimeException("失败")); // 异常完成
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}).start();

// 获取结果（阻塞）
String result = cf.get();
System.out.println(result);

// 非阻塞获取（有默认值）
String r = cf.getNow("默认值");

// 超时获取
String r2 = cf.get(2, TimeUnit.SECONDS);
```

---

## 三、异步任务创建

### 3.1 supplyAsync / runAsync

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

// runAsync：无返回值
CompletableFuture<Void> run = CompletableFuture.runAsync(() ->
    System.out.println("异步执行，无返回值")
);

// supplyAsync：有返回值
CompletableFuture<String> supply = CompletableFuture.supplyAsync(() -> {
    // 默认使用 ForkJoinPool.commonPool()（不推荐生产使用）
    return "异步计算结果";
});

// ⚠️ 生产环境强烈建议指定自定义线程池
ExecutorService pool = Executors.newFixedThreadPool(8);
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
    return fetchDataFromDB();
}, pool);

// 记得关闭线程池
// pool.shutdown();
```

> ⚠️ **重要**：避免使用默认的 `ForkJoinPool.commonPool()`，它是整个 JVM 共用的，若被耗尽会影响其他功能（如并行流）。

---

## 四、结果转换与处理

### 4.1 thenApply：转换结果（有返回值）

```java
CompletableFuture<Integer> result = CompletableFuture
    .supplyAsync(() -> "Hello World", pool)
    .thenApply(s -> s.length())        // String → Integer
    .thenApply(len -> len * 2);        // Integer → Integer

System.out.println(result.get()); // 22
```

### 4.2 thenAccept：消费结果（无返回值）

```java
CompletableFuture.supplyAsync(() -> fetchUser(1L), pool)
    .thenAccept(user -> System.out.println("用户: " + user.getName()));
```

### 4.3 thenRun：不使用结果，只在完成后执行

```java
CompletableFuture.supplyAsync(() -> saveData(), pool)
    .thenRun(() -> System.out.println("保存完成，发送通知"));
```

### 4.4 thenCompose：扁平化链式异步（避免 CompletableFuture 嵌套）

```java
// 错误方式：thenApply 返回 CompletableFuture<CompletableFuture<User>>
CompletableFuture<CompletableFuture<User>> nested =
    CompletableFuture.supplyAsync(() -> getUserId(), pool)
        .thenApply(id -> CompletableFuture.supplyAsync(() -> fetchUser(id), pool));

// 正确方式：thenCompose 返回 CompletableFuture<User>（扁平）
CompletableFuture<User> flat =
    CompletableFuture.supplyAsync(() -> getUserId(), pool)
        .thenCompose(id -> CompletableFuture.supplyAsync(() -> fetchUser(id), pool));
```

### 4.5 Async 后缀方法

所有 `then*` 方法都有对应的 `then*Async` 版本，区别在于执行线程：

| 方法 | 执行线程 |
|------|---------|
| `thenApply(fn)` | 上一个任务的完成线程（或当前调用线程） |
| `thenApplyAsync(fn)` | 公共 ForkJoinPool 中的线程 |
| `thenApplyAsync(fn, pool)` | 指定线程池中的线程 |

```java
// 推荐：每步指定线程池，明确任务在哪里执行
CompletableFuture.supplyAsync(() -> query(), ioPool)     // IO线程池查询
    .thenApplyAsync(data -> transform(data), cpuPool)    // CPU线程池处理
    .thenAcceptAsync(result -> respond(result), ioPool); // IO线程池响应
```

---

## 五、任务组合

### 5.1 thenCombine：两个任务都完成后合并结果

```java
CompletableFuture<String> userFuture = CompletableFuture
    .supplyAsync(() -> fetchUser(1L), pool)
    .thenApply(User::getName);

CompletableFuture<Double> priceFuture = CompletableFuture
    .supplyAsync(() -> fetchPrice("SKU001"), pool);

// 两个任务并发执行，都完成后合并
CompletableFuture<String> combined = userFuture.thenCombine(priceFuture,
    (userName, price) -> userName + " 购买，价格: " + price);

System.out.println(combined.get());
```

### 5.2 allOf：等待所有任务完成

```java
List<Long> userIds = Arrays.asList(1L, 2L, 3L, 4L, 5L);

// 并行查询所有用户
List<CompletableFuture<User>> futures = userIds.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> fetchUser(id), pool))
    .collect(Collectors.toList());

// 等待全部完成
CompletableFuture<Void> allDone = CompletableFuture.allOf(
    futures.toArray(new CompletableFuture[0]));

// 收集所有结果
CompletableFuture<List<User>> allResults = allDone.thenApply(v ->
    futures.stream()
        .map(CompletableFuture::join) // 已完成，join 不会阻塞
        .collect(Collectors.toList())
);

List<User> users = allResults.get();
```

### 5.3 anyOf：任意一个完成即返回

```java
// 从多个数据源查询，哪个快用哪个（降级策略）
CompletableFuture<Object> fastest = CompletableFuture.anyOf(
    CompletableFuture.supplyAsync(() -> fetchFromCache(), pool),
    CompletableFuture.supplyAsync(() -> fetchFromDB(), pool),
    CompletableFuture.supplyAsync(() -> fetchFromRemote(), pool)
);

Object result = fastest.get();
```

### 5.4 applyToEither：两个中最先完成的

```java
// 主链路 vs 备用链路，取最快的
CompletableFuture<String> result = primaryFuture
    .applyToEither(backupFuture, s -> "结果: " + s);
```

---

## 六、异常处理

### 6.1 exceptionally：异常兜底

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> {
        if (Math.random() > 0.5) throw new RuntimeException("查询失败");
        return "正常结果";
    }, pool)
    .exceptionally(ex -> {
        System.err.println("异常: " + ex.getMessage());
        return "默认值"; // 返回降级结果
    });

System.out.println(cf.get()); // 不会抛异常
```

### 6.2 handle：无论成功失败都处理（类似 try-finally）

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> queryDB(), pool)
    .handle((result, ex) -> {
        if (ex != null) {
            log.error("查询失败", ex);
            return "降级数据";
        }
        return result;
    });
```

### 6.3 whenComplete：结果监听（不影响结果）

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> "result", pool)
    .whenComplete((result, ex) -> {
        // 只监听，不修改结果
        if (ex != null) {
            metrics.increment("error");
        } else {
            log.info("成功: {}", result);
        }
    });
// cf 的结果仍是 "result"
```

### 6.4 异常处理方法对比

| 方法 | 是否修改结果 | 异常时是否执行 | 正常时是否执行 |
|------|------------|--------------|--------------|
| `exceptionally` | 是（仅异常时） | ✅ | ❌ |
| `handle` | 是 | ✅ | ✅ |
| `whenComplete` | 否 | ✅ | ✅ |

---

## 七、线程池配置最佳实践

```java
import java.util.concurrent.*;

public class AsyncConfig {

    // IO 密集型任务线程池（数据库查询、HTTP调用）
    public static final ExecutorService IO_POOL = new ThreadPoolExecutor(
        10, 50,
        60L, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(200),
        new ThreadFactoryBuilder().setNameFormat("async-io-%d").build(),
        new ThreadPoolExecutor.CallerRunsPolicy()
    );

    // CPU 密集型任务线程池（计算、数据处理）
    public static final ExecutorService CPU_POOL = new ThreadPoolExecutor(
        Runtime.getRuntime().availableProcessors(),
        Runtime.getRuntime().availableProcessors(),
        0L, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<>(100),
        new ThreadFactoryBuilder().setNameFormat("async-cpu-%d").build(),
        new ThreadPoolExecutor.AbortPolicy()
    );

    // 关闭线程池（应用退出时）
    public static void shutdown() {
        IO_POOL.shutdown();
        CPU_POOL.shutdown();
    }
}
```

---

## 八、实战：异步并行查询

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.*;

// 电商商品详情页：并行获取商品信息、库存、价格、推荐
public class ProductDetailService {

    private final ExecutorService pool;

    public ProductDetailService(ExecutorService pool) {
        this.pool = pool;
    }

    public ProductDetailVO getProductDetail(Long productId) throws Exception {
        // 并行发起多个查询
        CompletableFuture<Product>       productFuture = CompletableFuture
            .supplyAsync(() -> getProduct(productId), pool);

        CompletableFuture<Integer>       stockFuture   = CompletableFuture
            .supplyAsync(() -> getStock(productId), pool);

        CompletableFuture<Double>        priceFuture   = CompletableFuture
            .supplyAsync(() -> getPrice(productId), pool);

        CompletableFuture<List<Product>> recommendFuture = CompletableFuture
            .supplyAsync(() -> getRecommend(productId), pool)
            .exceptionally(ex -> Collections.emptyList()); // 推荐失败不影响主流程

        // 等待核心数据（商品、库存、价格）
        CompletableFuture<Void> coreFutures = CompletableFuture.allOf(
            productFuture, stockFuture, priceFuture);

        return coreFutures.thenApply(v -> {
            ProductDetailVO vo = new ProductDetailVO();
            vo.setProduct(productFuture.join());
            vo.setStock(stockFuture.join());
            vo.setPrice(priceFuture.join());
            vo.setRecommendList(recommendFuture.getNow(Collections.emptyList()));
            return vo;
        }).get(3, TimeUnit.SECONDS); // 整体超时 3 秒
    }

    // 模拟各服务调用
    private Product getProduct(Long id) {
        try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        return new Product(id, "商品" + id);
    }
    private Integer getStock(Long id) {
        try { Thread.sleep(80); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        return 100;
    }
    private Double getPrice(Long id) {
        try { Thread.sleep(120); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        return 99.9;
    }
    private List<Product> getRecommend(Long id) {
        try { Thread.sleep(200); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        return Arrays.asList(new Product(2L, "推荐商品"));
    }

    // 简化 VO 类
    static class Product {
        Long id; String name;
        Product(Long id, String name) { this.id = id; this.name = name; }
    }
    static class ProductDetailVO {
        Product product; Integer stock; Double price; List<Product> recommendList;
        void setProduct(Product p) { this.product = p; }
        void setStock(Integer s) { this.stock = s; }
        void setPrice(Double p) { this.price = p; }
        void setRecommendList(List<Product> l) { this.recommendList = l; }
    }
}
```

---

## 九、大厂常见面试题

### Q1：CompletableFuture 与 Future 有什么区别？

**答：**

| 维度 | Future | CompletableFuture |
|------|--------|------------------|
| 结果获取 | `get()`（阻塞） | `get()`（阻塞）+ `join()`（不抛检查异常）+ `getNow(默认值)` |
| 回调 | 不支持 | 支持（`thenApply`/`thenAccept` 等） |
| 链式组合 | 不支持 | 支持（`thenCompose`、`thenCombine` 等） |
| 异常处理 | 仅 `ExecutionException` | `exceptionally`/`handle`/`whenComplete` |
| 多任务等待 | 需循环调用 `isDone()` | `allOf`/`anyOf` |
| 手动完成 | 不支持 | `complete`/`completeExceptionally` |

---

### Q2：thenApply 和 thenCompose 的区别？

**答：**

- `thenApply(Function<T, U>)`：同步转换，将结果从 T 映射到 U，返回 `CompletableFuture<U>`。
- `thenCompose(Function<T, CompletableFuture<U>>)`：异步转换，函数本身返回 `CompletableFuture<U>`，用于扁平化嵌套的异步调用，避免返回 `CompletableFuture<CompletableFuture<U>>`。

类比：`thenApply` 相当于 `Stream.map`，`thenCompose` 相当于 `Stream.flatMap`。

---

### Q3：CompletableFuture 默认使用哪个线程池？有什么问题？

**答：**

默认使用 `ForkJoinPool.commonPool()`（公共线程池）。问题：

1. **共用**：整个 JVM 共用，并行流（`parallelStream`）、其他框架也可能使用，可能相互影响。
2. **线程数受限**：默认线程数 = CPU 核数 - 1，IO 密集型任务下线程很快耗尽，任务排队等待。
3. **不可监控**：无法对其做线程池监控和动态调整。

**最佳实践：** 始终为 `supplyAsync`/`runAsync` 指定自定义线程池，IO 密集型和 CPU 密集型使用不同配置的线程池。

---

### Q4：如何处理 CompletableFuture 中的超时？

**答：**

JDK 9 新增了 `orTimeout` 和 `completeOnTimeout` 方法：

```java
// JDK 9+：超时后抛 TimeoutException
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> slowQuery(), pool)
    .orTimeout(2, TimeUnit.SECONDS);

// JDK 9+：超时后使用默认值完成
CompletableFuture<String> cf2 = CompletableFuture
    .supplyAsync(() -> slowQuery(), pool)
    .completeOnTimeout("默认值", 2, TimeUnit.SECONDS);

// JDK 8 兼容写法
CompletableFuture<String> cf3 = CompletableFuture
    .supplyAsync(() -> slowQuery(), pool);
try {
    String result = cf3.get(2, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    cf3.cancel(true);
    // 处理超时
}
```

---

### Q5：allOf 中某个任务异常会影响其他任务吗？

**答：**

`CompletableFuture.allOf()` 的行为：
- 所有子任务是**并发执行**的，某一个任务抛出异常**不会取消其他任务**。
- `allOf` 返回的 `CompletableFuture<Void>` 会在**任意一个**子任务异常时以该异常完成（`exceptionally`）。
- 调用 `allOf.get()` 时，若有子任务异常，会抛出 `ExecutionException`（封装了第一个异常）。

**最佳实践：** 对每个子 `CompletableFuture` 单独 `exceptionally` 处理，避免一个任务失败导致整体失败：

```java
List<CompletableFuture<String>> futures = tasks.stream()
    .map(task -> CompletableFuture
        .supplyAsync(() -> execute(task), pool)
        .exceptionally(ex -> "降级结果"))
    .collect(Collectors.toList());

CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
    .thenApply(v -> futures.stream().map(CompletableFuture::join).collect(toList()))
    .get();
```

---

*参考资料：《Java 8 实战》、JDK 源码、美团技术博客《CompletableFuture原理与实践》*
