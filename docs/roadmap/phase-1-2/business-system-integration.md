# 业务系统集成

## 目标与重要性

AI Agent 的价值在于与现有业务系统深度集成。掌握如何将 LLM 能力安全地嵌入到 Java/Spring Boot 业务系统中，是落地 AI Agent 的关键工程能力。

## 核心概念清单

- REST API 集成模式
- 异步消息队列集成
- 数据库查询封装
- 事件驱动 Agent
- 双向集成（Java 调 Python Agent / Python 调 Java API）

## 学习路径

### 入门（第1层）

```python
# Python Agent 通过 REST API 调用 Java 业务服务
import httpx
import os
from typing import Optional

class PaymentSystemClient:
    """封装支付系统 API 调用"""

    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "X-Service-Name": "ai-agent"
        }

    async def get_order(self, order_id: str) -> dict:
        async with httpx.AsyncClient(timeout=10.0) as client:
            response = await client.get(
                f"{self.base_url}/api/v1/orders/{order_id}",
                headers=self.headers
            )
            response.raise_for_status()
            return response.json()

    async def create_refund(
        self,
        order_id: str,
        amount: float,
        reason: str,
        operator: str = "ai-agent"
    ) -> dict:
        async with httpx.AsyncClient(timeout=30.0) as client:
            response = await client.post(
                f"{self.base_url}/api/v1/refunds",
                json={
                    "order_id": order_id,
                    "amount": amount,
                    "reason": reason,
                    "operator": operator,
                    "source": "AI_AGENT"  # 标记来自 AI Agent
                },
                headers=self.headers
            )
            response.raise_for_status()
            return response.json()

# 使用
payment_client = PaymentSystemClient(
    base_url=os.getenv("PAYMENT_API_URL"),
    api_key=os.getenv("PAYMENT_API_KEY")
)
```

### 进阶（第2层）

```python
# 事件驱动：监听业务事件，触发 Agent 处理
import asyncio
from typing import Callable

# 方式1：通过 Kafka/RabbitMQ 消费消息
# pip install aiokafka
async def process_payment_events():
    from aiokafka import AIOKafkaConsumer
    import json

    consumer = AIOKafkaConsumer(
        'payment.events',
        bootstrap_servers='kafka:9092',
        group_id='ai-agent-group',
        value_deserializer=lambda m: json.loads(m.decode('utf-8'))
    )
    await consumer.start()

    try:
        async for msg in consumer:
            event = msg.value
            if event.get("event_type") == "PAYMENT_FAILED":
                # 触发 AI 分析
                await handle_payment_failure_event(event)

    finally:
        await consumer.stop()

async def handle_payment_failure_event(event: dict):
    order_id = event["order_id"]
    print(f"处理支付失败事件: {order_id}")

    # 调用 AI Agent 分析
    analysis = await analyze_payment_failure(order_id)

    # 根据分析结果执行动作
    if analysis.needs_notification:
        await send_user_notification(order_id, analysis.user_message)

    if analysis.auto_retryable:
        await schedule_retry(order_id)

# 方式2：HTTP Webhook
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

@app.post("/webhooks/payment-events")
async def payment_webhook(event: dict, background_tasks: BackgroundTasks):
    """接收支付系统的 Webhook 回调"""
    if event["event_type"] == "PAYMENT_FAILED":
        # 异步处理，不阻塞响应
        background_tasks.add_task(
            handle_payment_failure_event,
            event
        )
    return {"status": "accepted", "event_id": event.get("event_id")}
```

### 深入（第3层）

```java
// Java Spring Boot 端：将 Python Agent 作为微服务调用
@Service
@Slf4j
public class AiAgentIntegrationService {

    private final WebClient agentWebClient;
    private final PaymentOrderRepository orderRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public AiAgentIntegrationService(
        @Value("${ai.agent.url}") String agentUrl,
        WebClient.Builder webClientBuilder,
        PaymentOrderRepository orderRepository,
        KafkaTemplate<String, Object> kafkaTemplate
    ) {
        this.agentWebClient = webClientBuilder
            .baseUrl(agentUrl)
            .filter(ExchangeFilterFunction.ofRequestProcessor(request -> {
                // 注入 trace_id
                return Mono.just(ClientRequest.from(request)
                    .header("X-Trace-Id", MDC.get("traceId"))
                    .build());
            }))
            .build();
        this.orderRepository = orderRepository;
        this.kafkaTemplate = kafkaTemplate;
    }

    // 同步调用：分析支付失败
    public Mono<PaymentFailureAnalysis> analyzeFailure(String orderId) {
        return agentWebClient.post()
            .uri("/analyze/payment-failure")
            .bodyValue(Map.of("order_id", orderId))
            .retrieve()
            .bodyToMono(PaymentFailureAnalysis.class)
            .timeout(Duration.ofSeconds(30))
            .retryWhen(Retry.backoff(2, Duration.ofSeconds(1))
                .filter(e -> e instanceof WebClientResponseException.ServiceUnavailable))
            .doOnSuccess(analysis ->
                log.info("AI分析完成: orderId={}, category={}", orderId, analysis.category()))
            .doOnError(e ->
                log.error("AI分析失败: orderId={}", orderId, e));
    }

    // 异步处理：发布事件供 Agent 处理
    public void publishForAgentProcessing(PaymentOrder order) {
        var event = new PaymentFailureEvent(
            order.getOrderId(),
            order.getFailCode(),
            order.getUserId(),
            LocalDateTime.now()
        );
        kafkaTemplate.send("payment.events.for-agent", event);
        log.info("已发布支付失败事件给AI Agent: orderId={}", order.getOrderId());
    }
}

// Agent API 端点（暴露给 Java 调用）
@RestController
@RequestMapping("/api/v1/agent")
public class AgentController {

    private final PaymentFailureAgentService agentService;

    @PostMapping("/analyze")
    public ResponseEntity<AnalysisResult> analyze(
        @RequestBody AnalysisRequest request,
        @RequestHeader("X-Trace-Id") String traceId
    ) {
        MDC.put("traceId", traceId);
        try {
            AnalysisResult result = agentService.analyze(request.getOrderId());
            return ResponseEntity.ok(result);
        } finally {
            MDC.clear();
        }
    }
}
```

```python
# Python FastAPI Agent 服务端
from fastapi import FastAPI, Header, BackgroundTasks
from pydantic import BaseModel
import asyncio

app = FastAPI()

class AnalysisRequest(BaseModel):
    order_id: str
    context: dict = {}

class AnalysisResult(BaseModel):
    order_id: str
    root_cause: str
    category: str
    recommended_actions: list[str]
    confidence: float
    needs_manual_review: bool

@app.post("/analyze/payment-failure", response_model=AnalysisResult)
async def analyze_payment_failure(
    request: AnalysisRequest,
    x_trace_id: str = Header(default="")
):
    """支付失败分析端点"""
    # 传递 trace_id 到所有下游调用
    result = await run_payment_analysis_agent(
        order_id=request.order_id,
        trace_id=x_trace_id
    )
    return result

@app.get("/health")
async def health():
    return {"status": "ok", "service": "payment-analysis-agent"}
```

## 最小可执行练习

```python
# 完整的集成测试
import asyncio
import httpx

async def integration_test():
    # 1. 启动 Agent 服务（假设已启动在 localhost:8000）
    async with httpx.AsyncClient(base_url="http://localhost:8000") as client:
        # 健康检查
        health = await client.get("/health")
        assert health.status_code == 200

        # 分析请求
        result = await client.post(
            "/analyze/payment-failure",
            json={"order_id": "PAY202401001"},
            headers={"X-Trace-Id": "test-trace-001"}
        )
        assert result.status_code == 200
        data = result.json()
        print(f"分析结果: {data}")
        assert "root_cause" in data
        assert "category" in data

asyncio.run(integration_test())
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| Python Agent 超时 | LLM 调用慢 | 设置合理超时，使用异步处理 |
| Java 调 Python 失败 | 服务发现配置错误 | 使用 Eureka/Consul 或 K8s Service |
| 数据格式不一致 | Python 和 Java 的序列化差异 | 统一使用 JSON，明确字段类型 |
| Trace ID 丢失 | 没有在 HTTP 头中传递 | 始终在 Header 中传递 X-Trace-Id |

## 参考资料

- [Spring WebFlux 文档](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html)
- [FastAPI 文档](https://fastapi.tiangolo.com/)
- [Kafka Spring 集成](https://spring.io/projects/spring-kafka)

---

## 补充：集成模式最佳实践

### 数据库查询封装

```python
# 将数据库查询封装为 Agent 工具
from typing import Optional
import asyncpg
import json

class PaymentDatabaseTool:
    """支付数据库查询工具（封装为 Agent 可调用的工具）"""

    def __init__(self, db_url: str):
        self.db_url = db_url
        self._pool = None

    async def get_pool(self):
        if not self._pool:
            self._pool = await asyncpg.create_pool(self.db_url)
        return self._pool

    async def query_order(self, order_id: str) -> dict:
        """查询订单信息（只读操作）"""
        pool = await self.get_pool()
        async with pool.acquire() as conn:
            row = await conn.fetchrow(
                "SELECT order_id, status, amount, fail_code, created_at "
                "FROM payment_orders WHERE order_id = $1",
                order_id
            )
            if not row:
                return {"error": f"订单不存在: {order_id}"}
            return dict(row)

    async def query_user_payment_history(
        self,
        user_id: str,
        limit: int = 10
    ) -> list:
        """查询用户支付历史"""
        pool = await self.get_pool()
        async with pool.acquire() as conn:
            rows = await conn.fetch(
                "SELECT order_id, status, amount, created_at "
                "FROM payment_orders WHERE user_id = $1 "
                "ORDER BY created_at DESC LIMIT $2",
                user_id, limit
            )
            return [dict(r) for r in rows]

    def as_tool_definition(self) -> dict:
        """返回 OpenAI Function Calling 格式的工具定义"""
        return {
            "type": "function",
            "function": {
                "name": "query_payment_order",
                "description": "从数据库查询支付订单的详细信息",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "order_id": {
                            "type": "string",
                            "description": "订单号，格式如 PAY202401001"
                        }
                    },
                    "required": ["order_id"]
                }
            }
        }

# 数据脱敏中间件
class DataMaskingMiddleware:
    """在数据返回给 LLM 前进行脱敏"""

    MASK_RULES = {
        "card_number": lambda v: f"****{str(v)[-4:]}" if v else None,
        "phone": lambda v: f"{str(v)[:3]}****{str(v)[-4:]}" if v else None,
        "email": lambda v: f"{str(v).split('@')[0][:3]}***@{str(v).split('@')[1]}" if v and '@' in str(v) else None,
        "id_number": lambda v: f"{str(v)[:3]}***{str(v)[-4:]}" if v else None,
    }

    def mask(self, data: dict) -> dict:
        """脱敏字典数据"""
        masked = {}
        for key, value in data.items():
            key_lower = key.lower()
            if any(sensitive in key_lower for sensitive in ["card", "phone", "email", "id_"]):
                for field, mask_fn in self.MASK_RULES.items():
                    if field.replace("_", "") in key_lower.replace("_", ""):
                        masked[key] = mask_fn(value)
                        break
                else:
                    masked[key] = "***"
            else:
                masked[key] = value
        return masked

# 测试
middleware = DataMaskingMiddleware()
order_data = {
    "order_id": "PAY001",
    "amount": 299.00,
    "card_number": "6222020212345678",
    "user_phone": "13800138000",
    "user_email": "user@example.com",
    "status": "FAILED"
}
masked = middleware.mask(order_data)
print(json.dumps(masked, ensure_ascii=False, indent=2))
```
