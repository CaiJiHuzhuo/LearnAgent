# 02 - gRPC与Protobuf

## 目录

1. [gRPC概述与特点](#一grpc概述与特点)
2. [Protobuf序列化原理](#二protobuf序列化原理)
3. [HTTP/2协议基础](#三http2协议基础)
4. [四种通信模式](#四四种通信模式)
5. [gRPC vs REST vs Dubbo对比](#五grpc-vs-rest-vs-dubbo对比)
6. [Spring Boot整合gRPC](#六spring-boot整合grpc)
7. [拦截器与中间件](#七拦截器与中间件)
8. [大厂常见面试题](#八大厂常见面试题)

---

## 一、gRPC概述与特点

> gRPC 是 Google 开源的高性能 RPC 框架，基于 HTTP/2 协议传输，使用 Protobuf 作为接口描述语言和序列化协议，支持多种编程语言。

### 1.1 核心特点

```
┌──────────────────────────────────────────────────────┐
│                     gRPC 架构                         │
│                                                      │
│  ┌─────────────┐   HTTP/2    ┌─────────────┐        │
│  │   Client    │ ←─────────→ │   Server    │        │
│  │  (Stub)     │   Protobuf  │  (Service)  │        │
│  └─────────────┘             └─────────────┘        │
│        │                           │                 │
│        ▼                           ▼                 │
│  ┌─────────────┐           ┌─────────────┐          │
│  │  Java/Go/   │           │  Java/Go/   │          │
│  │  Python/C++ │           │  Python/C++ │          │
│  └─────────────┘           └─────────────┘          │
└──────────────────────────────────────────────────────┘
```

| 特点 | 说明 |
|------|------|
| **高性能** | 基于 HTTP/2 + Protobuf，序列化体积小、传输效率高 |
| **跨语言** | 支持 Java、Go、Python、C++、Node.js 等 10+ 语言 |
| **强类型** | 通过 .proto 文件定义接口，自动生成代码，编译时类型检查 |
| **流式传输** | 支持客户端流、服务端流、双向流，天然适合实时通信 |
| **多路复用** | HTTP/2 单连接多流，减少连接开销 |
| **内置治理** | 支持拦截器、超时、取消、认证等 |

### 1.2 适用场景

```
微服务内部通信（高性能低延迟） ──→ ✅ gRPC 最佳
跨语言微服务调用              ──→ ✅ gRPC 非常合适
实时流数据（如监控、聊天）     ──→ ✅ gRPC 双向流
浏览器直接调用后端            ──→ ❌ 考虑 gRPC-Web 或 REST
公开 API（第三方集成）        ──→ ❌ 推荐 REST/GraphQL
```

---

## 二、Protobuf序列化原理

> Protocol Buffers（Protobuf）是 Google 开源的高效序列化协议，通过 .proto 文件定义数据结构，自动生成各语言代码。

### 2.1 .proto文件定义

```protobuf
// user.proto
syntax = "proto3";

package com.example.user;

option java_multiple_files = true;
option java_package = "com.example.user.proto";

// 消息定义
message User {
    int64 id = 1;           // 字段编号 1
    string name = 2;        // 字段编号 2
    string email = 3;       // 字段编号 3
    UserStatus status = 4;  // 枚举类型
    repeated string tags = 5; // 列表类型
}

// 枚举定义
enum UserStatus {
    UNKNOWN = 0;
    ACTIVE = 1;
    INACTIVE = 2;
}

// 服务定义
service UserService {
    rpc GetUser (GetUserRequest) returns (User);
    rpc ListUsers (ListUsersRequest) returns (stream User);
}

message GetUserRequest {
    int64 id = 1;
}

message ListUsersRequest {
    int32 page_size = 1;
    string page_token = 2;
}
```

### 2.2 编码原理

> Protobuf 使用 **Tag-Length-Value（TLV）** 编码方式，每个字段由字段编号和类型标识组成。

```
字段编码格式: (field_number << 3) | wire_type

Wire Type 类型:
┌────────────┬────────────┬──────────────────────────┐
│ Wire Type  │ 含义       │ 适用类型                   │
├────────────┼────────────┼──────────────────────────┤
│ 0          │ Varint     │ int32, int64, bool, enum  │
│ 1          │ 64-bit     │ fixed64, double           │
│ 2          │ Length     │ string, bytes, message    │
│ 5          │ 32-bit     │ fixed32, float            │
└────────────┴────────────┴──────────────────────────┘
```

**Varint 变长编码示例（数字 300 的编码）：**

```
300 的二进制: 100101100

Varint 编码过程:
  1. 拆分为 7-bit 组: 0000010 | 0101100
  2. 倒序排列（小端序）: 0101100 | 0000010
  3. 非最后一组高位设1:  10101100 | 00000010
  4. 最终编码: 0xAC 0x02（仅需 2 字节）

对比:
  int32 固定编码: 00 00 01 2C（4 字节）
  Varint 编码:    AC 02      （2 字节，节省 50%）
```

### 2.3 Protobuf vs JSON 对比

| 维度 | Protobuf | JSON |
|------|----------|------|
| **编码格式** | 二进制 | 文本 |
| **数据体积** | 小（约 JSON 的 1/3~1/5） | 大（含字段名、引号等冗余） |
| **序列化速度** | 快（约 JSON 的 5~10 倍） | 慢 |
| **可读性** | 差（二进制不可读） | 好（文本可直接查看） |
| **Schema** | 强制定义 .proto | 无 Schema（灵活但易出错） |
| **向后兼容** | 优秀（字段编号机制） | 一般（需额外约定） |
| **跨语言** | 自动生成代码 | 各语言自行解析 |
| **适合场景** | RPC 内部通信 | REST API、配置文件 |

**体积对比示例：**

```
User { id: 1, name: "张三", email: "zhangsan@example.com" }

JSON 编码 (约 62 字节):
{"id":1,"name":"张三","email":"zhangsan@example.com"}

Protobuf 编码 (约 30 字节):
08 01 12 06 E5 BC A0 E4 B8 89 1A 16 7A 68 61 6E
67 73 61 6E 40 65 78 61 6D 70 6C 65 2E 63 6F 6D

体积减少约 50%
```

### 2.4 向后兼容性

```
版本1: message User { int64 id = 1; string name = 2; }
版本2: message User { int64 id = 1; string name = 2; string email = 3; }

兼容规则:
  ✅ 新增字段（使用新编号）  → 旧代码忽略未知字段
  ✅ 删除字段（保留编号）    → 新代码使用默认值
  ❌ 修改字段编号           → 数据解析错误
  ❌ 修改字段类型           → 数据解析错误
```

---

## 三、HTTP/2协议基础

> gRPC 基于 HTTP/2 协议，充分利用了 HTTP/2 的多路复用、头部压缩、服务端推送等特性。

### 3.1 HTTP/1.1 vs HTTP/2

| 维度 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| **传输方式** | 文本协议 | 二进制协议 |
| **多路复用** | 不支持（一个连接一个请求） | 支持（一个连接多个流） |
| **头部压缩** | 无 | HPACK 压缩 |
| **服务端推送** | 不支持 | 支持 |
| **请求优先级** | 不支持 | 支持 |
| **连接管理** | 短连接或 Keep-Alive | 长连接 + 多路复用 |

### 3.2 多路复用

```
HTTP/1.1 (队头阻塞):
  连接1: 请求A ──────── 响应A ──────── 请求C ──── 响应C
  连接2: 请求B ──────── 响应B

HTTP/2 (多路复用):
  单连接:
    Stream 1: 请求A ─────── 响应A
    Stream 2: ── 请求B ──── 响应B
    Stream 3: ──── 请求C ──── 响应C
    （多个请求/响应交错传输，互不阻塞）
```

```
┌───────────────────────────────────────────┐
│            HTTP/2 单个 TCP 连接            │
│                                           │
│  Stream 1 ─→ │ HEADERS │ DATA │ DATA │    │
│  Stream 2 ─→ │ HEADERS │ DATA │          │
│  Stream 3 ─→ │ HEADERS │ DATA │ DATA │    │
│              ↑                            │
│         帧(Frame)在流中交错传输             │
└───────────────────────────────────────────┘

HTTP/2 核心概念:
  Connection: 一个 TCP 连接
  Stream:     一个双向数据流，对应一个请求/响应
  Frame:      最小传输单位（HEADERS帧、DATA帧等）
```

### 3.3 HPACK头部压缩

```
HTTP/1.1 每次都发送完整头部:
  GET /users HTTP/1.1
  Host: api.example.com
  Content-Type: application/json
  Authorization: Bearer eyJhbGciOi...

HTTP/2 HPACK 压缩:
  ┌────────────────────────┐
  │ 静态表（61个常用头部）   │ ← :method GET = 索引 2
  ├────────────────────────┤
  │ 动态表（本次连接累积）   │ ← Authorization = 索引 62
  └────────────────────────┘

  首次请求: 发送完整头部，构建动态表
  后续请求: 只发送变化部分，引用索引号
  压缩率:   通常可达 80% 以上
```

---

## 四、四种通信模式

### 4.1 模式总览

| 模式 | 客户端 | 服务端 | 适用场景 |
|------|--------|--------|---------|
| **Unary** | 发一条 | 回一条 | 普通请求/响应，如查询用户 |
| **Server Streaming** | 发一条 | 回多条（流） | 大数据分批返回，如查询列表 |
| **Client Streaming** | 发多条（流） | 回一条 | 批量上传，如文件上传 |
| **Bidirectional Streaming** | 发多条（流） | 回多条（流） | 实时交互，如聊天、监控 |

### 4.2 Proto定义

```protobuf
service UserService {
    // 1. Unary: 一请求一响应
    rpc GetUser (GetUserRequest) returns (User);

    // 2. Server Streaming: 服务端流
    rpc ListUsers (ListUsersRequest) returns (stream User);

    // 3. Client Streaming: 客户端流
    rpc UploadUsers (stream User) returns (UploadResponse);

    // 4. Bidirectional Streaming: 双向流
    rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}
```

### 4.3 Unary（一元调用）

```
Client                          Server
  │                               │
  │── GetUserRequest(id=1) ──────→│
  │                               │ 处理请求
  │←── User(id=1,name="张三") ────│
  │                               │
```

```java
// 服务端实现
@GrpcService
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {
    @Override
    public void getUser(GetUserRequest request,
                        StreamObserver<User> responseObserver) {
        User user = User.newBuilder()
                .setId(request.getId())
                .setName("张三")
                .setEmail("zhangsan@example.com")
                .build();
        responseObserver.onNext(user);
        responseObserver.onCompleted();
    }
}

// 客户端调用
User user = userStub.getUser(
    GetUserRequest.newBuilder().setId(1L).build()
);
```

### 4.4 Server Streaming（服务端流）

```
Client                          Server
  │                               │
  │── ListUsersRequest ──────────→│
  │                               │
  │←── User("张三") ──────────────│
  │←── User("李四") ──────────────│
  │←── User("王五") ──────────────│
  │←── [完成信号] ────────────────│
  │                               │
```

```java
// 服务端实现
@Override
public void listUsers(ListUsersRequest request,
                      StreamObserver<User> responseObserver) {
    List<User> users = queryUsersFromDB(request);
    for (User user : users) {
        responseObserver.onNext(user);  // 逐个推送
    }
    responseObserver.onCompleted();
}

// 客户端调用
Iterator<User> users = userStub.listUsers(
    ListUsersRequest.newBuilder().setPageSize(10).build()
);
while (users.hasNext()) {
    User user = users.next();
    System.out.println("收到用户: " + user.getName());
}
```

### 4.5 Client Streaming（客户端流）

```
Client                          Server
  │                               │
  │── User("张三") ──────────────→│
  │── User("李四") ──────────────→│
  │── User("王五") ──────────────→│
  │── [完成信号] ────────────────→│
  │                               │ 处理所有数据
  │←── UploadResponse(count=3) ──│
  │                               │
```

```java
// 客户端调用（异步 stub）
StreamObserver<UploadResponse> responseObserver = new StreamObserver<>() {
    @Override
    public void onNext(UploadResponse response) {
        System.out.println("上传成功: " + response.getCount() + " 条");
    }
    @Override
    public void onError(Throwable t) { }
    @Override
    public void onCompleted() { }
};

StreamObserver<User> requestObserver =
    asyncStub.uploadUsers(responseObserver);

// 逐条发送
requestObserver.onNext(User.newBuilder().setName("张三").build());
requestObserver.onNext(User.newBuilder().setName("李四").build());
requestObserver.onNext(User.newBuilder().setName("王五").build());
requestObserver.onCompleted();
```

### 4.6 Bidirectional Streaming（双向流）

```
Client                          Server
  │                               │
  │── ChatMessage("你好") ───────→│
  │←── ChatMessage("你好！") ─────│
  │── ChatMessage("天气如何？") ──→│
  │←── ChatMessage("晴天") ───────│
  │── [完成] ────────────────────→│
  │←── [完成] ────────────────────│
  │                               │

  双方可以独立发送和接收，不需要等待对方响应
```

```java
// 服务端实现
@Override
public StreamObserver<ChatMessage> chat(
        StreamObserver<ChatMessage> responseObserver) {
    return new StreamObserver<>() {
        @Override
        public void onNext(ChatMessage message) {
            // 收到客户端消息，处理后回复
            ChatMessage reply = ChatMessage.newBuilder()
                    .setContent("收到: " + message.getContent())
                    .build();
            responseObserver.onNext(reply);
        }
        @Override
        public void onError(Throwable t) { }
        @Override
        public void onCompleted() {
            responseObserver.onCompleted();
        }
    };
}
```

---

## 五、gRPC vs REST vs Dubbo对比

### 5.1 综合对比表

| 维度 | gRPC | REST | Dubbo |
|------|------|------|-------|
| **传输协议** | HTTP/2 | HTTP/1.1 或 HTTP/2 | TCP（自定义协议） |
| **序列化** | Protobuf（默认） | JSON（通常） | Hessian2（默认） |
| **IDL** | .proto 文件 | OpenAPI/Swagger | Java 接口 |
| **性能** | 高 | 较低 | 高 |
| **跨语言** | 10+ 语言 | 任意语言 | 主要 Java |
| **流式传输** | 四种模式 | 不原生支持 | 不原生支持 |
| **浏览器支持** | 需 gRPC-Web | 天然支持 | 不支持 |
| **生态** | 云原生（K8s、Istio） | 最广泛 | Java/Spring 生态 |
| **学习成本** | 中等 | 低 | 低（Java开发者） |
| **适合场景** | 微服务内部、跨语言 | 对外API、Web服务 | Java微服务内部 |

### 5.2 性能对比

```
延迟对比（单次调用，越短越好）:
Dubbo     ██░░░░░░░░░░░░░░░░░░  0.5ms
gRPC      ███░░░░░░░░░░░░░░░░░  0.8ms
REST/JSON ████████████░░░░░░░░  3.0ms

吞吐量对比（QPS，越长越好）:
Dubbo     ████████████████████  50000 QPS
gRPC      ██████████████████░░  45000 QPS
REST/JSON ████████░░░░░░░░░░░░  20000 QPS

传输体积对比（同一对象，越短越好）:
Protobuf  ███░░░░░░░░░░░░░░░░░  30 bytes
Hessian2  ██████░░░░░░░░░░░░░░  60 bytes
JSON      ████████████████░░░░  150 bytes
```

### 5.3 选型建议

```
场景判断:
  微服务内部通信（Java体系） ──→ Dubbo（生态成熟、功能丰富）
  微服务内部通信（多语言）    ──→ gRPC（跨语言、高性能）
  对外公开 API               ──→ REST（通用性最强）
  实时流数据                  ──→ gRPC（原生支持流式传输）
  云原生架构（K8s/Istio）     ──→ gRPC（Service Mesh 首选）
```

---

## 六、Spring Boot整合gRPC

### 6.1 项目结构

```
grpc-demo/
├── grpc-api/                  # Proto 文件和生成代码
│   └── src/main/proto/
│       └── user.proto
├── grpc-server/               # gRPC 服务端
│   └── src/main/java/
│       └── com/example/server/
│           └── UserGrpcService.java
└── grpc-client/               # gRPC 客户端
    └── src/main/java/
        └── com/example/client/
            └── UserController.java
```

### 6.2 依赖配置

```xml
<!-- grpc-server pom.xml -->
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-server-spring-boot-starter</artifactId>
    <version>2.15.0.RELEASE</version>
</dependency>

<!-- grpc-client pom.xml -->
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-client-spring-boot-starter</artifactId>
    <version>2.15.0.RELEASE</version>
</dependency>
```

### 6.3 服务端实现

```yaml
# application.yml
grpc:
  server:
    port: 9090
```

```java
@GrpcService
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {

    @Override
    public void getUser(GetUserRequest request,
                        StreamObserver<User> responseObserver) {
        User user = User.newBuilder()
                .setId(request.getId())
                .setName("张三")
                .setEmail("zhangsan@example.com")
                .setStatus(UserStatus.ACTIVE)
                .build();

        responseObserver.onNext(user);
        responseObserver.onCompleted();
    }

    @Override
    public void listUsers(ListUsersRequest request,
                          StreamObserver<User> responseObserver) {
        // 服务端流：逐条返回用户
        for (int i = 1; i <= request.getPageSize(); i++) {
            User user = User.newBuilder()
                    .setId(i)
                    .setName("用户" + i)
                    .build();
            responseObserver.onNext(user);
        }
        responseObserver.onCompleted();
    }
}
```

### 6.4 客户端调用

```yaml
# application.yml
grpc:
  client:
    user-service:
      address: static://localhost:9090
      negotiationType: plaintext
```

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @GrpcClient("user-service")
    private UserServiceGrpc.UserServiceBlockingStub userStub;

    @GetMapping("/{id}")
    public Map<String, Object> getUser(@PathVariable Long id) {
        User user = userStub.getUser(
            GetUserRequest.newBuilder().setId(id).build()
        );
        return Map.of(
            "id", user.getId(),
            "name", user.getName(),
            "email", user.getEmail()
        );
    }
}
```

---

## 七、拦截器与中间件

> gRPC 拦截器类似于 Servlet Filter 或 Spring Interceptor，可以在调用前后插入通用逻辑。

### 7.1 拦截器架构

```
Client 端:
  请求 → ClientInterceptor1 → ClientInterceptor2 → 网络 → Server

Server 端:
  网络 → ServerInterceptor1 → ServerInterceptor2 → 业务处理
```

### 7.2 服务端拦截器

```java
// 日志拦截器
public class LoggingServerInterceptor implements ServerInterceptor {
    private static final Logger log = LoggerFactory.getLogger(
        LoggingServerInterceptor.class);

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String methodName = call.getMethodDescriptor().getFullMethodName();
        long startTime = System.currentTimeMillis();
        log.info("gRPC 调用开始: {}", methodName);

        ServerCall.Listener<ReqT> listener = next.startCall(call, headers);

        return new ForwardingServerCallListener.SimpleForwardingServerCallListener
                <>(listener) {
            @Override
            public void onComplete() {
                long cost = System.currentTimeMillis() - startTime;
                log.info("gRPC 调用完成: {}, 耗时: {}ms", methodName, cost);
                super.onComplete();
            }
        };
    }
}

// 注册拦截器
@GrpcGlobalServerInterceptor
public LoggingServerInterceptor loggingInterceptor() {
    return new LoggingServerInterceptor();
}
```

### 7.3 客户端拦截器

```java
// 认证拦截器：自动携带 Token
public class AuthClientInterceptor implements ClientInterceptor {
    private static final Metadata.Key<String> AUTH_KEY =
        Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);

    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next) {

        return new ForwardingClientCall.SimpleForwardingClientCall<>(
                next.newCall(method, callOptions)) {
            @Override
            public void start(Listener<RespT> listener, Metadata headers) {
                headers.put(AUTH_KEY, "Bearer " + getToken());
                super.start(listener, headers);
            }
        };
    }
}
```

### 7.4 常见拦截器场景

| 场景 | 拦截器类型 | 说明 |
|------|-----------|------|
| **日志记录** | Server + Client | 记录调用方法、耗时、参数 |
| **认证鉴权** | Server | 校验 Token，拒绝未授权请求 |
| **Token传递** | Client | 自动携带认证 Token |
| **链路追踪** | Server + Client | 传递 TraceId，串联调用链 |
| **限流** | Server | 基于滑动窗口或令牌桶限流 |
| **异常处理** | Server | 统一异常包装和错误码映射 |
| **指标监控** | Server + Client | Prometheus 指标采集 |

---

## 八、大厂常见面试题

### Q1：gRPC和REST有什么区别？各自适合什么场景？

**答：**

核心区别：
1. **协议**：gRPC 基于 HTTP/2（二进制），REST 通常基于 HTTP/1.1（文本）
2. **序列化**：gRPC 用 Protobuf（高效紧凑），REST 用 JSON（可读性好）
3. **接口定义**：gRPC 用 .proto 强类型定义，REST 用 URL + HTTP Method
4. **流式传输**：gRPC 原生支持四种流式模式，REST 不原生支持

场景选择：
- **微服务内部调用**：gRPC（高性能、强类型、跨语言）
- **对外公开 API**：REST（通用性好、浏览器友好、调试方便）
- **实时通信**：gRPC（双向流式传输）

### Q2：Protobuf为什么比JSON快、体积小？

**答：**

1. **编码方式**：Protobuf 使用二进制编码 + Varint 变长整数，JSON 使用文本编码
2. **字段标识**：Protobuf 用字段编号（1~2字节），JSON 用完整字段名字符串
3. **无冗余字符**：Protobuf 没有花括号、引号、逗号等分隔符
4. **预编译**：Protobuf 通过 .proto 文件生成序列化/反序列化代码，无需运行时反射

### Q3：HTTP/2的多路复用是什么？解决了什么问题？

**答：**

- **问题**：HTTP/1.1 每个请求独占一个 TCP 连接，浏览器限制 6 个并发连接，请求多时排队等待（队头阻塞）
- **多路复用**：HTTP/2 在一个 TCP 连接上并行传输多个请求/响应，每个请求作为一个 Stream，数据拆分为 Frame 交错传输
- **效果**：减少 TCP 连接数量和握手开销，消除 HTTP 层队头阻塞，提高并发性能
- **注意**：TCP 层仍可能存在队头阻塞（丢包重传），这由 HTTP/3（QUIC）解决

### Q4：gRPC的四种通信模式分别是什么？

**答：**

1. **Unary（一元）**：客户端发一条请求，服务端回一条响应，最常见的模式
2. **Server Streaming（服务端流）**：客户端发一条请求，服务端返回多条响应流，适合大数据分批返回
3. **Client Streaming（客户端流）**：客户端发送多条请求流，服务端回一条响应，适合批量上传
4. **Bidirectional Streaming（双向流）**：双方同时发送数据流，独立读写互不阻塞，适合实时交互场景

### Q5：Protobuf如何实现向后兼容？

**答：**

1. **字段编号**：Protobuf 用编号标识字段，不依赖字段名和顺序
2. **新增字段**：使用新编号，旧代码遇到未知字段会自动忽略
3. **删除字段**：保留编号不复用（使用 `reserved`），新代码用默认值
4. **不可变更**：不能修改已有字段的编号或类型
5. **默认值**：`proto3` 中所有字段有默认值（0、""、false），未传字段使用默认值

### Q6：gRPC在微服务架构中如何实现服务发现和负载均衡？

**答：**

1. **客户端负载均衡**：gRPC 内置 `pick_first`（默认）和 `round_robin` 策略，由客户端直接选择后端节点
2. **Name Resolver**：通过 DNS、Consul、Nacos 等解析服务名为地址列表
3. **代理负载均衡**：通过 Envoy、Nginx 等代理层转发请求
4. **Service Mesh**：Istio + Envoy 透明代理，应用无感知的负载均衡、熔断、重试
5. 生产实践中，gRPC 通常与 Service Mesh 配合使用，实现完整的服务治理
