# 01 - Dubbo核心原理

## 目录

1. [Dubbo整体架构](#一dubbo整体架构)
2. [SPI机制](#二spi机制)
3. [服务注册与发现流程](#三服务注册与发现流程)
4. [通信模型](#四通信模型)
5. [负载均衡策略](#五负载均衡策略)
6. [容错策略](#六容错策略)
7. [序列化协议](#七序列化协议)
8. [服务调用完整链路](#八服务调用完整链路)
9. [Spring Boot整合Dubbo](#九spring-boot整合dubbo)
10. [大厂常见面试题](#十大厂常见面试题)

---

## 一、Dubbo整体架构

> Dubbo 是阿里巴巴开源的高性能 RPC 框架，现已成为 Apache 顶级项目。它提供了服务注册与发现、负载均衡、容错、监控等微服务核心能力。

### 1.1 核心角色

```
                    ┌───────────────┐
                    │   Registry    │
                    │   注册中心     │
                    └───┬───────┬───┘
                 ②注册 │       │ ③订阅
                  服务  │       │ 服务
                        ↓       ↓
┌──────────────┐              ┌───────────────┐
│   Provider   │←─── ④调用 ───│   Consumer    │
│   服务提供者  │              │   服务消费者   │
└──────┬───────┘              └───────┬───────┘
       │                              │
       │ ①启动                        │ ⑤监控
       ↓                              ↓
┌─────────────────────────────────────────────┐
│                  Monitor                     │
│                 监控中心                       │
└─────────────────────────────────────────────┘
```

| 角色 | 说明 |
|------|------|
| **Provider（服务提供者）** | 暴露服务的应用，启动时向注册中心注册自己提供的服务 |
| **Consumer（服务消费者）** | 调用远程服务的应用，启动时向注册中心订阅所需的服务 |
| **Registry（注册中心）** | 服务注册与发现的目录服务，如 Zookeeper、Nacos |
| **Monitor（监控中心）** | 统计服务调用次数、调用耗时等指标 |
| **Container（服务容器）** | 负责启动、加载、运行服务提供者 |

### 1.2 调用流程简述

1. **Provider** 启动时向 **Registry** 注册自己的地址和服务信息
2. **Consumer** 启动时向 **Registry** 订阅需要的服务
3. **Registry** 将 Provider 地址列表推送给 Consumer
4. **Consumer** 根据负载均衡策略选择一台 Provider 发起调用
5. Provider 和 Consumer 的调用统计信息定期上报到 **Monitor**

---

## 二、SPI机制

> SPI（Service Provider Interface）是 Dubbo 实现高度可扩展性的核心机制。Dubbo 几乎所有的核心组件（协议、负载均衡、序列化等）都通过 SPI 扩展。

### 2.1 Java SPI vs Dubbo SPI

| 维度 | Java SPI | Dubbo SPI |
|------|----------|-----------|
| **配置文件** | `META-INF/services/接口全限定名` | `META-INF/dubbo/接口全限定名` |
| **加载方式** | 一次性加载所有实现类 | 按需加载，支持懒加载 |
| **是否支持 key** | 不支持，只能遍历 | 支持 key=value 形式，按名称获取 |
| **IoC/AOP** | 不支持 | 支持依赖注入和包装器（Wrapper） |
| **自适应扩展** | 不支持 | 支持 `@Adaptive` 注解，运行时动态决定 |

### 2.2 Dubbo SPI配置示例

```
# 文件路径: META-INF/dubbo/org.apache.dubbo.rpc.Protocol
dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
http=org.apache.dubbo.rpc.protocol.http.HttpProtocol
grpc=org.apache.dubbo.rpc.protocol.grpc.GrpcProtocol
```

### 2.3 自适应扩展（@Adaptive）

> 自适应扩展允许在运行时根据 URL 参数动态选择具体的扩展实现，是 Dubbo 实现灵活配置的关键。

```java
// 自适应扩展接口定义
@SPI("dubbo")
public interface Protocol {
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
}

// 使用方式：根据URL中的protocol参数自动选择实现
// url = dubbo://192.168.1.1:20880/com.example.UserService
// 会自动使用 DubboProtocol
ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);
Protocol protocol = loader.getAdaptiveExtension();
```

### 2.4 Dubbo SPI加载流程

```
ExtensionLoader.getExtension("dubbo")
        │
        ▼
 ┌──────────────────┐
 │ 1. 从缓存获取实例 │
 └────────┬─────────┘
          │ 缓存未命中
          ▼
 ┌──────────────────────────┐
 │ 2. 加载配置文件           │
 │   META-INF/dubbo/        │
 │   META-INF/dubbo/internal│
 │   META-INF/services/     │
 └────────┬─────────────────┘
          ▼
 ┌──────────────────────────┐
 │ 3. 根据 key 找到实现类    │
 └────────┬─────────────────┘
          ▼
 ┌──────────────────────────┐
 │ 4. 反射创建实例           │
 └────────┬─────────────────┘
          ▼
 ┌──────────────────────────┐
 │ 5. 依赖注入（IoC）        │
 └────────┬─────────────────┘
          ▼
 ┌──────────────────────────┐
 │ 6. Wrapper 包装（AOP）    │
 └────────┬─────────────────┘
          ▼
      返回实例
```

---

## 三、服务注册与发现流程

### 3.1 注册流程

```
Provider 启动
    │
    ▼
┌──────────────────────────────────┐
│ 1. 解析 @DubboService 注解       │
│    获取接口名、版本号、分组等信息  │
└──────────┬───────────────────────┘
           ▼
┌──────────────────────────────────┐
│ 2. 启动 Netty Server 监听端口    │
│    默认端口: 20880               │
└──────────┬───────────────────────┘
           ▼
┌──────────────────────────────────┐
│ 3. 向注册中心注册服务节点         │
│    创建: /dubbo/com.example      │
│    .UserService/providers/       │
│    dubbo://192.168.1.1:20880     │
└──────────────────────────────────┘
```

### 3.2 发现流程

```
Consumer 启动
    │
    ▼
┌──────────────────────────────────┐
│ 1. 解析 @DubboReference 注解     │
│    确定需要订阅的服务             │
└──────────┬───────────────────────┘
           ▼
┌──────────────────────────────────┐
│ 2. 向注册中心订阅服务地址列表     │
│    监听: /dubbo/com.example      │
│    .UserService/providers/       │
└──────────┬───────────────────────┘
           ▼
┌──────────────────────────────────┐
│ 3. 注册中心推送 Provider 列表     │
│    [192.168.1.1:20880,           │
│     192.168.1.2:20880]           │
└──────────┬───────────────────────┘
           ▼
┌──────────────────────────────────┐
│ 4. Consumer 本地缓存地址列表      │
│    后续调用直接从缓存获取         │
└──────────────────────────────────┘
```

> **注意：** 注册中心宕机后，Consumer 仍可通过本地缓存正常调用 Provider，体现了 Dubbo 的高可用设计。

---

## 四、通信模型

### 4.1 Netty通信架构

> Dubbo 默认使用 Netty 作为网络通信框架，采用 NIO 非阻塞模型，支持高并发网络通信。

```
┌─────────────────────────────────────────────────┐
│                  Consumer 端                     │
│  ┌───────────┐    ┌──────────┐   ┌───────────┐  │
│  │ 业务线程   │───→│ 编码器    │──→│  Netty    │  │
│  │           │    │ Encoder  │   │  Client   │──┼──→ 网络
│  └───────────┘    └──────────┘   └───────────┘  │
└─────────────────────────────────────────────────┘
                                       │
                                       ↓ TCP
┌─────────────────────────────────────────────────┐
│                  Provider 端                     │
│  ┌───────────┐    ┌──────────┐   ┌───────────┐  │
│  │ 业务线程池 │←───│ 解码器    │←──│  Netty    │  │
│  │           │    │ Decoder  │   │  Server   │←─┼── 网络
│  └───────────┘    └──────────┘   └───────────┘  │
└─────────────────────────────────────────────────┘
```

### 4.2 Dubbo协议格式

> Dubbo 自定义了高效的二进制协议，协议头固定 16 字节。

```
 0      1      2             4             8            12           16
 +------+------+-------------+-------------+------------+------------+
 | Magic High  | Magic Low   |             |            |            |
 | 0xda        | 0xbb        | Flag        | Status     | Request ID |
 +------+------+-------------+-------------+------------+------------+
 |                         Data Length (4 bytes)                      |
 +-------------------------------------------------------------------+
 |                         Body (变长)                                |
 +-------------------------------------------------------------------+

 Magic:     魔数 0xdabb，用于识别 Dubbo 协议数据包
 Flag:      标志位（请求/响应、单向/双向、序列化方式等）
 Status:    响应状态（OK=20, Timeout=30 等）
 Request ID: 请求唯一标识，用于匹配请求与响应
 Data Length: 消息体长度
```

### 4.3 请求-响应匹配机制

```
Consumer 端:
  请求1 (ID=1) ──→ ┌────────┐
  请求2 (ID=2) ──→ │ Netty  │ ──→ Provider
  请求3 (ID=3) ──→ │ Client │
                    └────────┘
                        ↑
  响应2 (ID=2) ←── ─────┘  ← 响应可能乱序到达
  响应1 (ID=1) ←── ─────┘
  响应3 (ID=3) ←── ─────┘

  通过 Request ID 匹配：
  - 发送请求时创建 DefaultFuture，存入 Map<Long, DefaultFuture>
  - 收到响应时根据 ID 找到对应 Future，唤醒等待线程
```

```java
// Dubbo 请求响应匹配核心逻辑（简化版）
public class DefaultFuture extends CompletableFuture<Object> {
    private static final Map<Long, DefaultFuture> FUTURES = new ConcurrentHashMap<>();

    public DefaultFuture(Request request, int timeout) {
        FUTURES.put(request.getId(), this);
    }

    // 收到响应时调用
    public static void received(Response response) {
        DefaultFuture future = FUTURES.remove(response.getId());
        if (future != null) {
            future.complete(response.getResult());
        }
    }
}
```

---

## 五、负载均衡策略

> Dubbo 内置了多种负载均衡策略，Consumer 在每次调用时根据策略选择一个 Provider。

### 5.1 策略对比

| 策略 | 名称 | 原理 | 适用场景 |
|------|------|------|---------|
| **Random** | 加权随机 | 按权重随机选择，权重越高被选概率越大 | 默认策略，通用场景 |
| **RoundRobin** | 加权轮询 | 按权重轮流分配请求 | 需要均匀分布的场景 |
| **LeastActive** | 最少活跃 | 优先选择活跃调用数最少的 Provider | 处理能力不均匀的场景 |
| **ConsistentHash** | 一致性哈希 | 相同参数的请求总是发到同一台 Provider | 有状态服务，需要会话粘滞 |
| **ShortestResponse** | 最短响应 | 选择平均响应时间最短的 Provider | 对延迟敏感的场景 |

### 5.2 加权随机算法

```java
// 简化版加权随机负载均衡
public class RandomLoadBalance {
    public <T> Invoker<T> select(List<Invoker<T>> invokers) {
        int totalWeight = 0;
        boolean sameWeight = true;
        int[] weights = new int[invokers.size()];

        for (int i = 0; i < invokers.size(); i++) {
            weights[i] = getWeight(invokers.get(i));
            totalWeight += weights[i];
            if (i > 0 && weights[i] != weights[0]) {
                sameWeight = false;
            }
        }

        // 权重不同，按权重随机
        if (!sameWeight && totalWeight > 0) {
            int offset = ThreadLocalRandom.current().nextInt(totalWeight);
            for (int i = 0; i < invokers.size(); i++) {
                offset -= weights[i];
                if (offset < 0) {
                    return invokers.get(i);
                }
            }
        }
        // 权重相同，随机选择
        return invokers.get(ThreadLocalRandom.current().nextInt(invokers.size()));
    }
}
```

### 5.3 一致性哈希算法

```
              ┌──────────────────────┐
              │    Hash 环 (0~2^32)   │
              │                      │
              │     Node-A(hash1)    │
              │    ╱                 │
              │   ○ ─── ○ Node-B    │
              │  ╱       ╲          │
              │ ○         ○         │
              │  ╲       ╱          │
              │   ○ ─── ○ Node-C   │
              │                     │
              └─────────────────────┘

  请求路由: hash(请求参数) → 顺时针找到最近的节点
  虚拟节点: 每个物理节点映射 160 个虚拟节点，保证均匀分布
```

---

## 六、容错策略

> 当服务调用失败时，Dubbo 提供多种容错策略来保障系统的稳定性。

### 6.1 策略对比

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| **Failover** | 失败自动切换到其他节点重试（默认重试2次） | 读操作、幂等操作 |
| **Failfast** | 失败立即报错，不重试 | 写操作（如下单、扣款） |
| **Failsafe** | 失败时忽略异常，返回空结果 | 日志记录、监控上报 |
| **Failback** | 失败后记录请求，定时自动重发 | 消息通知、数据同步 |
| **Forking** | 并行调用多个节点，只要一个成功即返回 | 实时性要求极高的读操作 |
| **Broadcast** | 广播调用所有节点，任一失败即报错 | 通知所有节点更新缓存 |

### 6.2 容错流程

```
Consumer 发起调用
        │
        ▼
┌───────────────────┐
│ 负载均衡选择节点    │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐    成功    ┌─────────┐
│ 调用 Provider      │─────────→│ 返回结果 │
└────────┬──────────┘           └─────────┘
         │ 失败
         ▼
┌───────────────────────────────────────────────┐
│              根据容错策略处理                    │
├───────────────────────────────────────────────┤
│ Failover:  重试其他节点（最多重试N次）           │
│ Failfast:  直接抛出异常                         │
│ Failsafe:  忽略异常，返回空结果                  │
│ Failback:  记录失败请求，后台定时重发             │
│ Forking:   并行调用多个节点                      │
│ Broadcast: 逐个调用所有节点                      │
└───────────────────────────────────────────────┘
```

```java
// 配置容错策略
@DubboReference(cluster = "failover", retries = 2)
private UserService userService;

// Failover 核心逻辑（简化版）
public Result doInvoke(Invocation invocation, List<Invoker> invokers) {
    int retries = getRetries(invocation);
    List<Invoker> invoked = new ArrayList<>();
    for (int i = 0; i <= retries; i++) {
        Invoker invoker = loadBalance.select(invokers, invocation);
        invoked.add(invoker);
        try {
            return invoker.invoke(invocation);
        } catch (RpcException e) {
            // 记录失败，继续重试
            logger.warn("Failover, retry " + i);
        }
    }
    throw new RpcException("Failed after " + retries + " retries");
}
```

---

## 七、序列化协议

### 7.1 Dubbo支持的序列化协议

| 协议 | 类型 | 速度 | 体积 | 跨语言 | 说明 |
|------|------|------|------|--------|------|
| **Hessian2** | 二进制 | 快 | 较小 | 支持 | Dubbo 默认序列化协议 |
| **Protobuf** | 二进制 | 非常快 | 最小 | 支持 | Google 出品，需要预定义 Schema |
| **Kryo** | 二进制 | 非常快 | 小 | 不支持 | 仅 Java，适合内部 RPC |
| **JSON** | 文本 | 慢 | 大 | 支持 | 可读性好，但性能较差 |
| **FST** | 二进制 | 快 | 小 | 不支持 | Java 快速序列化，替代 Java 原生 |
| **Avro** | 二进制 | 快 | 小 | 支持 | 动态 Schema，大数据生态常用 |

### 7.2 序列化性能对比

```
序列化速度（越短越好）:
Protobuf  ████░░░░░░░░░░░░░░░░  100μs
Kryo      █████░░░░░░░░░░░░░░░  120μs
FST       ██████░░░░░░░░░░░░░░  150μs
Hessian2  ████████░░░░░░░░░░░░  200μs
JSON      ████████████████░░░░  400μs
Java原生   ████████████████████  500μs

序列化后体积（越短越好）:
Protobuf  ███░░░░░░░░░░░░░░░░░  50 bytes
Kryo      █████░░░░░░░░░░░░░░░  80 bytes
Hessian2  ███████░░░░░░░░░░░░░  120 bytes
JSON      ████████████████░░░░  300 bytes
Java原生   ████████████████████  450 bytes
```

### 7.3 选型建议

```
内部 Java 微服务 RPC？ ──→ Hessian2（默认）或 Kryo（更快）
跨语言调用？           ──→ Protobuf（高性能）或 JSON（简单）
高性能 + 跨语言？       ──→ Protobuf（最优选择）
调试友好？             ──→ JSON（可读性最佳）
```

---

## 八、服务调用完整链路

> 一次完整的 Dubbo RPC 调用涉及代理生成、路由过滤、负载均衡、网络通信、序列化等多个环节。

### 8.1 Consumer端调用链路

```
userService.getUser(1)
        │
        ▼
 ┌──────────────────┐
 │ 1. Proxy 代理     │ ← 动态代理（Javassist/JDK Proxy）
 └────────┬─────────┘
          ▼
 ┌──────────────────┐
 │ 2. Filter 链      │ ← ConsumerContextFilter → FutureFilter → MonitorFilter
 └────────┬─────────┘
          ▼
 ┌──────────────────┐
 │ 3. Router 路由    │ ← 条件路由、标签路由、脚本路由
 └────────┬─────────┘
          ▼
 ┌──────────────────┐
 │ 4. Cluster 容错   │ ← Failover / Failfast / ...
 └────────┬─────────┘
          ▼
 ┌──────────────────┐
 │ 5. LoadBalance    │ ← Random / RoundRobin / ...
 │    负载均衡       │
 └────────┬─────────┘
          ▼
 ┌──────────────────┐
 │ 6. Serialize      │ ← Hessian2 序列化请求
 │    序列化         │
 └────────┬─────────┘
          ▼
 ┌──────────────────┐
 │ 7. Netty Client   │ ← 发送 TCP 请求
 │    网络传输       │
 └──────────────────┘
```

### 8.2 Provider端处理链路

```
 ┌──────────────────┐
 │ 1. Netty Server   │ ← 接收 TCP 请求
 │    网络接收       │
 └────────┬─────────┘
          ▼
 ┌──────────────────┐
 │ 2. Deserialize    │ ← Hessian2 反序列化
 │    反序列化       │
 └────────┬─────────┘
          ▼
 ┌──────────────────┐
 │ 3. 分发到线程池    │ ← 默认使用 fixed 线程池（200线程）
 └────────┬─────────┘
          ▼
 ┌──────────────────┐
 │ 4. Filter 链      │ ← ContextFilter → ExceptionFilter → TimeoutFilter
 └────────┬─────────┘
          ▼
 ┌──────────────────┐
 │ 5. Invoker 调用   │ ← 反射调用真实的服务实现
 └────────┬─────────┘
          ▼
 ┌──────────────────┐
 │ 6. 返回结果       │ ← 序列化响应并通过 Netty 返回
 └──────────────────┘
```

---

## 九、Spring Boot整合Dubbo

### 9.1 项目结构

```
dubbo-demo/
├── dubbo-api/               # 公共接口模块
│   └── src/main/java/
│       └── com/example/api/
│           └── UserService.java
├── dubbo-provider/          # 服务提供者
│   └── src/main/java/
│       └── com/example/provider/
│           └── UserServiceImpl.java
└── dubbo-consumer/          # 服务消费者
    └── src/main/java/
        └── com/example/consumer/
            └── UserController.java
```

### 9.2 接口定义

```java
// dubbo-api: 公共接口
public interface UserService {
    User getUser(Long id);
    List<User> listUsers();
}
```

### 9.3 Provider实现

```xml
<!-- pom.xml 依赖 -->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>3.2.0</version>
</dependency>
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-registry-nacos</artifactId>
    <version>3.2.0</version>
</dependency>
```

```yaml
# application.yml
dubbo:
  application:
    name: dubbo-provider
  registry:
    address: nacos://localhost:8848
  protocol:
    name: dubbo
    port: 20880
  provider:
    timeout: 3000
    retries: 0
```

```java
@DubboService(version = "1.0.0")
public class UserServiceImpl implements UserService {
    @Override
    public User getUser(Long id) {
        return new User(id, "张三", "zhangsan@example.com");
    }

    @Override
    public List<User> listUsers() {
        return Arrays.asList(
            new User(1L, "张三", "zhangsan@example.com"),
            new User(2L, "李四", "lisi@example.com")
        );
    }
}
```

### 9.4 Consumer调用

```yaml
# application.yml
dubbo:
  application:
    name: dubbo-consumer
  registry:
    address: nacos://localhost:8848
  consumer:
    timeout: 5000
    retries: 2
    check: false
```

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @DubboReference(version = "1.0.0", loadbalance = "random",
                    cluster = "failover", retries = 2)
    private UserService userService;

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }

    @GetMapping("/list")
    public List<User> listUsers() {
        return userService.listUsers();
    }
}
```

---

## 十、大厂常见面试题

### Q1：Dubbo的整体架构是怎样的？各角色的职责是什么？

**答：**

Dubbo 架构包含五个核心角色：
1. **Provider**：暴露服务，启动时注册到注册中心
2. **Consumer**：消费服务，启动时从注册中心订阅服务
3. **Registry**：注册中心，存储 Provider 地址并推送给 Consumer
4. **Monitor**：监控中心，统计调用次数和耗时
5. **Container**：服务容器，负责启动和运行 Provider

调用过程：Provider 注册 → Consumer 订阅 → Registry 推送地址 → Consumer 直连调用 Provider。

### Q2：Dubbo SPI和Java SPI有什么区别？

**答：**

核心区别：
1. **按需加载**：Java SPI 一次性加载所有实现，Dubbo SPI 按 key 按需加载
2. **扩展能力**：Dubbo SPI 支持 IoC 依赖注入和 AOP 包装器
3. **自适应扩展**：Dubbo SPI 通过 `@Adaptive` 注解实现运行时根据 URL 参数动态选择实现
4. **配置格式**：Dubbo SPI 使用 `key=value` 格式，可以精确获取指定扩展

### Q3：Dubbo的负载均衡策略有哪些？默认是哪个？

**答：**

五种策略：Random（加权随机，**默认**）、RoundRobin（加权轮询）、LeastActive（最少活跃调用数）、ConsistentHash（一致性哈希）、ShortestResponse（最短响应时间）。
- Random 适合通用场景，按权重分配流量
- LeastActive 自动感知 Provider 处理能力，慢节点自动接收更少请求
- ConsistentHash 用于有状态服务，保证相同请求路由到相同节点

### Q4：Dubbo的容错策略有哪些？生产中如何选择？

**答：**

六种策略的选择原则：
- **读操作（幂等）**：使用 Failover，失败自动切换重试
- **写操作（非幂等）**：使用 Failfast，失败立即报错，避免重复写入
- **非核心操作**：使用 Failsafe，如日志记录，失败不影响主流程
- **异步消息场景**：使用 Failback，失败后自动定时重发
- **极致实时性**：使用 Forking，并行调用多个节点，任一成功即返回

### Q5：一次Dubbo调用经历了哪些步骤？

**答：**

Consumer 端：代理拦截 → Filter 链 → Router 路由 → Cluster 容错 → LoadBalance 负载均衡 → Serialize 序列化 → Netty 发送。
Provider 端：Netty 接收 → 反序列化 → 线程池分发 → Filter 链 → Invoker 反射调用 → 序列化响应 → Netty 返回。

### Q6：Dubbo的序列化协议如何选择？

**答：**

- **默认**：Hessian2，平衡了性能和跨语言能力
- **高性能 Java 内部调用**：Kryo 或 FST，序列化速度更快
- **跨语言 + 高性能**：Protobuf，体积最小、速度最快
- **调试便利**：JSON，可读性好，但性能最差
- 生产建议：内部服务用 Hessian2 或 Protobuf，对外接口用 JSON
