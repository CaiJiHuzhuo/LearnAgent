# 04 - Spring Cloud 微服务

> 本章讲解 Spring Cloud 微服务核心组件，覆盖服务注册发现、远程调用、网关、熔断限流、配置中心等关键模块。

## 目录

1. [微服务概述](#一微服务概述)
2. [服务注册与发现](#二服务注册与发现)
3. [远程调用](#三远程调用)
4. [服务网关](#四服务网关)
5. [熔断与限流](#五熔断与限流)
6. [配置中心](#六配置中心)
7. [链路追踪](#七链路追踪)
8. [大厂常见面试题](#八大厂常见面试题)

---

## 一、微服务概述

### 1.1 单体 vs 微服务

| 维度 | 单体架构 | 微服务架构 |
|------|---------|-----------|
| 部署 | 整体打包部署 | 独立部署 |
| 扩展 | 只能整体扩展 | 按服务独立扩展 |
| 技术栈 | 统一 | 可异构 |
| 团队 | 单一团队 | 独立小团队 |
| 复杂度 | 代码耦合 | 运维复杂 |

### 1.2 Spring Cloud 技术栈演进

| 组件类型 | Netflix（第一代） | Alibaba（主流） | 官方/其他 |
|---------|-----------------|----------------|----------|
| 注册中心 | Eureka | **Nacos** | Consul、Zookeeper |
| 远程调用 | Feign | OpenFeign | RestTemplate |
| 负载均衡 | Ribbon | **LoadBalancer** | - |
| 熔断降级 | Hystrix | **Sentinel** | Resilience4j |
| 网关 | Zuul | - | **Spring Cloud Gateway** |
| 配置中心 | Spring Cloud Config | **Nacos Config** | Apollo |

---

## 二、服务注册与发现

### 2.1 Nacos 注册中心

**核心概念：**

| 概念 | 说明 |
|------|------|
| **服务注册** | 服务启动时向 Nacos 注册自己的地址信息 |
| **服务发现** | 消费者从 Nacos 获取服务提供者地址列表 |
| **心跳检测** | 服务定期发送心跳，Nacos 检测服务健康状态 |
| **命名空间** | 环境隔离（dev/test/prod） |
| **分组** | 同一命名空间下的逻辑分组 |

### 2.2 Nacos vs Eureka

| 特性 | Nacos | Eureka |
|------|-------|--------|
| CAP 模型 | CP + AP 可切换 | AP |
| 健康检查 | 主动探测 + 心跳 | 仅心跳 |
| 配置管理 | ✅ 支持 | ❌ 不支持 |
| 服务下线 | 实时推送 | 30s 缓存 |
| 雪崩保护 | ✅ | ✅ |

### 2.3 配置示例

```yaml
spring:
  application:
    name: order-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: dev
        group: DEFAULT_GROUP
```

---

## 三、远程调用

### 3.1 OpenFeign 声明式调用

```java
// 定义 Feign 客户端
@FeignClient(name = "user-service", fallbackFactory = UserClientFallback.class)
public interface UserClient {

    @GetMapping("/api/users/{id}")
    Result<UserVO> getUser(@PathVariable("id") Long id);

    @PostMapping("/api/users")
    Result<Void> createUser(@RequestBody UserCreateDTO dto);
}

// 降级处理
@Component
public class UserClientFallback implements FallbackFactory<UserClient> {
    @Override
    public UserClient create(Throwable cause) {
        return new UserClient() {
            @Override
            public Result<UserVO> getUser(Long id) {
                return Result.fail(503, "用户服务不可用");
            }
            @Override
            public Result<Void> createUser(UserCreateDTO dto) {
                return Result.fail(503, "用户服务不可用");
            }
        };
    }
}
```

### 3.2 负载均衡

Spring Cloud LoadBalancer 支持的策略：

| 策略 | 说明 |
|------|------|
| **RoundRobinLoadBalancer** | 轮询（默认） |
| **RandomLoadBalancer** | 随机 |
| 自定义 | 实现 `ReactorServiceInstanceLoadBalancer` |

---

## 四、服务网关

### 4.1 Spring Cloud Gateway

**核心概念：**

| 概念 | 说明 |
|------|------|
| **Route（路由）** | 网关基本构建块：ID + URI + Predicate + Filter |
| **Predicate（断言）** | 匹配条件（路径、Header、参数等） |
| **Filter（过滤器）** | 请求/响应的处理逻辑 |

### 4.2 配置示例

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=0
            - AddRequestHeader=X-Request-Source, gateway

        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST
```

### 4.3 自定义全局过滤器

```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (token == null || !tokenService.verify(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        // 将用户信息传递给下游服务
        ServerHttpRequest request = exchange.getRequest().mutate()
                .header("X-User-Id", tokenService.getUserId(token))
                .build();
        return chain.filter(exchange.mutate().request(request).build());
    }

    @Override
    public int getOrder() {
        return -1; // 优先级最高
    }
}
```

---

## 五、熔断与限流

### 5.1 Sentinel 核心概念

| 概念 | 说明 |
|------|------|
| **资源** | 需要保护的代码/接口 |
| **流控规则** | QPS/线程数限流 |
| **降级规则** | 慢调用/异常比例/异常数触发降级 |
| **热点规则** | 热点参数限流 |
| **系统规则** | 系统级保护（CPU/Load） |

### 5.2 Sentinel vs Hystrix

| 特性 | Sentinel | Hystrix |
|------|----------|---------|
| 隔离策略 | 信号量/线程池 | 线程池为主 |
| 限流 | 基于 QPS/并发线程数 | 有限支持 |
| 控制台 | 功能丰富的可视化控制台 | Dashboard（简单） |
| 规则配置 | 动态规则，实时推送 | 需重启 |
| 维护状态 | 阿里持续维护 | Netflix 已停止维护 |

### 5.3 使用示例

```java
// 注解方式
@SentinelResource(value = "getUser",
    blockHandler = "getUserBlockHandler",
    fallback = "getUserFallback")
public UserVO getUser(Long id) {
    return userMapper.selectById(id);
}

// 限流时触发
public UserVO getUserBlockHandler(Long id, BlockException ex) {
    throw new BusinessException("请求过于频繁，请稍后再试");
}

// 业务异常时触发
public UserVO getUserFallback(Long id, Throwable ex) {
    return new UserVO(); // 返回默认值
}
```

---

## 六、配置中心

### 6.1 Nacos Config

**优势：**
- 配置与服务注册合二为一
- 支持配置动态刷新
- 支持多环境、多分组管理
- 配置版本历史与回滚

### 6.2 配置示例

```yaml
# bootstrap.yml
spring:
  application:
    name: order-service
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        namespace: dev
        shared-configs:
          - data-id: common.yaml
            group: DEFAULT_GROUP
            refresh: true
```

### 6.3 配置动态刷新

```java
@RestController
@RefreshScope  // 配合 Nacos 配置动态刷新
public class ConfigController {

    @Value("${app.switch.enable:false}")
    private boolean switchEnable;

    @GetMapping("/config")
    public String getConfig() {
        return "switch: " + switchEnable;
    }
}
```

---

## 七、链路追踪

### 7.1 分布式链路追踪

| 组件 | 说明 |
|------|------|
| **Micrometer Tracing** | Spring Boot 3.x 官方方案（原 Spring Cloud Sleuth） |
| **Zipkin** | 链路数据收集和展示 |
| **SkyWalking** | 国产 APM 工具，功能全面 |

### 7.2 核心概念

| 概念 | 说明 |
|------|------|
| **TraceId** | 整条调用链的唯一标识 |
| **SpanId** | 单次调用的唯一标识 |
| **Baggage** | 跨服务传递的上下文数据 |

---

## 八、大厂常见面试题

### Q1：微服务有哪些核心组件？各自的作用？

**答：** 注册中心（Nacos，服务注册发现）、远程调用（OpenFeign，声明式 HTTP 调用）、网关（Gateway，路由转发和统一鉴权）、熔断限流（Sentinel，保护服务防止雪崩）、配置中心（Nacos Config，集中配置管理和动态刷新）。

### Q2：Nacos 作为注册中心和 Eureka 的区别？

**答：** Nacos 支持 CP/AP 可切换，Eureka 只支持 AP；Nacos 集成配置管理，Eureka 不支持；Nacos 支持主动健康检查，Eureka 仅依赖心跳；Nacos 服务变更实时推送，Eureka 有 30 秒缓存延迟。

### Q3：Sentinel 和 Hystrix 的区别？

**答：** Sentinel 支持更丰富的限流规则（QPS/线程数/热点参数），有可视化控制台和动态规则配置；Hystrix 以线程池隔离为主，已停止维护。Sentinel 是目前的主流选择。

### Q4：Gateway 网关的作用？

**答：** 统一入口、路由转发、负载均衡、统一鉴权、限流降级、请求/响应改写、跨域处理、日志监控。核心由 Route（路由）、Predicate（断言）、Filter（过滤器）三部分组成。

### Q5：如何设计微服务的服务拆分？

**答：**
1. 按业务域拆分（DDD 领域驱动设计）
2. 高内聚、低耦合
3. 独立数据库（每个服务有自己的数据存储）
4. 服务粒度适中（避免过细导致运维复杂）
5. 考虑团队组织结构（康威定律）

---

*下一篇：[05-Spring 事务管理](./05-Spring事务管理.md)*
