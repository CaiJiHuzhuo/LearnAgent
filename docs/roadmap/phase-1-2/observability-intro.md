# 可观测性入门

## 目标与重要性

AI Agent 是一个不透明的系统：很难知道 LLM 为什么给出某个答案、某次失败是哪个环节的问题。可观测性让你能够调试问题、优化性能、控制成本。

## 核心概念清单

- Trace（链路追踪）：完整的请求调用链
- Metrics（指标）：Token 消耗、延迟、成功率
- Logs（日志）：结构化日志，支持过滤和搜索
- LLM 专用追踪工具（LangSmith、Langfuse）

## 学习路径

### 入门（第1层）

```python
import time
import uuid
import logging
from dataclasses import dataclass, field
from typing import Optional

logger = logging.getLogger(__name__)

@dataclass
class LLMCallTrace:
    trace_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    span_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    parent_span_id: Optional[str] = None

    model: str = ""
    input_tokens: int = 0
    output_tokens: int = 0
    total_tokens: int = 0
    latency_ms: float = 0
    success: bool = True
    error: Optional[str] = None

    user_id: Optional[str] = None
    session_id: Optional[str] = None

    def to_log_dict(self) -> dict:
        return {
            "trace_id": self.trace_id,
            "model": self.model,
            "tokens": self.total_tokens,
            "latency_ms": round(self.latency_ms, 2),
            "success": self.success,
        }

class ObservableLLMClient:
    def __init__(self):
        from openai import OpenAI
        self.client = OpenAI()
        self.traces: list[LLMCallTrace] = []

    def chat(
        self,
        messages: list,
        model: str = "gpt-4o-mini",
        trace_id: Optional[str] = None,
        **kwargs
    ) -> tuple[str, LLMCallTrace]:
        trace = LLMCallTrace(
            trace_id=trace_id or str(uuid.uuid4()),
            model=model
        )
        start_time = time.time()

        try:
            response = self.client.chat.completions.create(
                model=model,
                messages=messages,
                **kwargs
            )
            trace.latency_ms = (time.time() - start_time) * 1000
            trace.input_tokens = response.usage.prompt_tokens
            trace.output_tokens = response.usage.completion_tokens
            trace.total_tokens = response.usage.total_tokens
            trace.success = True

            content = response.choices[0].message.content
        except Exception as e:
            trace.latency_ms = (time.time() - start_time) * 1000
            trace.success = False
            trace.error = str(e)
            raise
        finally:
            self.traces.append(trace)
            logger.info("LLM调用", extra=trace.to_log_dict())

        return content, trace

    def get_metrics(self) -> dict:
        if not self.traces:
            return {}
        successful = [t for t in self.traces if t.success]
        return {
            "total_calls": len(self.traces),
            "success_rate": len(successful) / len(self.traces),
            "total_tokens": sum(t.total_tokens for t in successful),
            "avg_latency_ms": sum(t.latency_ms for t in successful) / max(len(successful), 1),
            "p95_latency_ms": sorted(t.latency_ms for t in successful)[int(len(successful) * 0.95)] if successful else 0,
        }
```

### 进阶（第2层）

```python
# 使用 Langfuse 进行 LLM 观测
# pip install langfuse

from langfuse import Langfuse
from langfuse.openai import openai  # 包装的 OpenAI 客户端

langfuse = Langfuse(
    public_key="pk-lf-...",
    secret_key="sk-lf-...",
    host="https://cloud.langfuse.com"
)

def traced_agent_call(user_query: str, session_id: str) -> str:
    """使用 Langfuse 追踪的 Agent 调用"""
    trace = langfuse.trace(
        name="payment-analysis",
        user_id="user_001",
        session_id=session_id,
        metadata={"environment": "production"}
    )

    # 追踪检索步骤
    retrieval_span = trace.span(name="knowledge-retrieval")
    retrieved_docs = search_knowledge_base(user_query)
    retrieval_span.end(output={"doc_count": len(retrieved_docs)})

    # 追踪 LLM 调用
    generation = trace.generation(
        name="analyze-failure",
        model="gpt-4o-mini",
        input={"query": user_query, "docs": retrieved_docs}
    )

    response = openai.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": user_query}],
        langfuse_observation_id=generation.id
    )
    result = response.choices[0].message.content

    generation.end(output=result)
    trace.update(output=result)

    return result

# 成本告警
class CostMonitor:
    def __init__(self, daily_budget_cny: float = 100.0):
        self.daily_budget_cny = daily_budget_cny
        self.daily_cost_cny = 0.0

    def record_usage(self, input_tokens: int, output_tokens: int, model: str):
        COSTS = {
            "gpt-4o": (0.005, 0.015),
            "gpt-4o-mini": (0.00015, 0.0006),
        }
        input_price, output_price = COSTS.get(model, (0.005, 0.015))
        cost_usd = (input_tokens * input_price + output_tokens * output_price) / 1000
        cost_cny = cost_usd * 7.2
        self.daily_cost_cny += cost_cny

        if self.daily_cost_cny > self.daily_budget_cny * 0.8:
            logger.warning(f"日成本告警: 已用 ¥{self.daily_cost_cny:.2f} / 预算 ¥{self.daily_budget_cny:.2f}")

        if self.daily_cost_cny > self.daily_budget_cny:
            raise Exception(f"超过日成本预算 ¥{self.daily_budget_cny:.2f}，已暂停服务")
```

### 深入（第3层）

```python
# 构建自定义仪表盘数据
from collections import defaultdict, deque
from datetime import datetime, timedelta

class AgentMetricsDashboard:
    def __init__(self, window_minutes: int = 60):
        self.window_minutes = window_minutes
        self.call_times: deque = deque()
        self.token_usage: deque = deque()
        self.latencies: deque = deque()
        self.errors: deque = deque()

    def _cleanup_old_data(self):
        cutoff = time.time() - self.window_minutes * 60
        for queue in [self.call_times, self.token_usage, self.latencies, self.errors]:
            while queue and queue[0][0] < cutoff:
                queue.popleft()

    def record(self, tokens: int, latency_ms: float, success: bool):
        now = time.time()
        self._cleanup_old_data()
        self.call_times.append((now, 1))
        self.token_usage.append((now, tokens))
        self.latencies.append((now, latency_ms))
        if not success:
            self.errors.append((now, 1))

    def get_snapshot(self) -> dict:
        self._cleanup_old_data()
        n = len(self.call_times)
        if n == 0:
            return {"status": "no_data"}

        latency_values = [v for _, v in self.latencies]
        latency_values.sort()

        return {
            "window_minutes": self.window_minutes,
            "total_calls": n,
            "calls_per_minute": n / self.window_minutes,
            "total_tokens": sum(v for _, v in self.token_usage),
            "error_rate": len(self.errors) / n,
            "avg_latency_ms": sum(latency_values) / n,
            "p50_latency_ms": latency_values[n // 2],
            "p95_latency_ms": latency_values[int(n * 0.95)],
            "p99_latency_ms": latency_values[int(n * 0.99)] if n >= 100 else latency_values[-1],
        }
```

## 最小可执行练习

```python
# 为你的 Agent 添加基础观测
from openai import OpenAI
import uuid, time, logging

logging.basicConfig(level=logging.INFO, format='%(asctime)s %(levelname)s %(message)s')
logger = logging.getLogger(__name__)

client = OpenAI()
dashboard = AgentMetricsDashboard()

def my_agent(query: str) -> str:
    trace_id = str(uuid.uuid4())[:8]
    logger.info(f"[{trace_id}] 开始处理: {query[:50]}")
    start = time.time()

    try:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": query}],
            max_tokens=200
        )
        latency = (time.time() - start) * 1000
        tokens = response.usage.total_tokens
        result = response.choices[0].message.content

        dashboard.record(tokens, latency, True)
        logger.info(f"[{trace_id}] 完成: tokens={tokens}, latency={latency:.0f}ms")
        return result

    except Exception as e:
        latency = (time.time() - start) * 1000
        dashboard.record(0, latency, False)
        logger.error(f"[{trace_id}] 失败: {e}")
        raise

# 测试
for q in ["支付失败怎么办", "如何退款", "账户余额不足"]:
    my_agent(q)

print(dashboard.get_snapshot())
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 日志量太大 | 记录了所有 LLM 内容 | 只记录摘要，完整内容只在 debug 级别 |
| Trace ID 丢失 | 没有传递到所有下游调用 | 使用 contextvars 传递 trace_id |
| 成本估算不准 | 模型价格变化 | 从 response.usage 读取真实 token 数 |

## Java/Spring Boot 对接要点

```java
@Aspect
@Component
public class LlmCallAspect {

    private final MeterRegistry meterRegistry;
    private final Counter callCounter;
    private final Timer callTimer;

    @Around("@annotation(TracedLlmCall)")
    public Object traceCall(ProceedingJoinPoint pjp) throws Throwable {
        String traceId = MDC.get("traceId");
        Timer.Sample sample = Timer.start(meterRegistry);

        try {
            Object result = pjp.proceed();
            callCounter.increment(1, Tags.of("status", "success"));
            return result;
        } catch (Exception e) {
            callCounter.increment(1, Tags.of("status", "error"));
            throw e;
        } finally {
            sample.stop(callTimer);
        }
    }
}
```

## 参考资料

- [Langfuse 文档](https://langfuse.com/docs)
- [OpenTelemetry for LLM](https://opentelemetry.io/)
- [LLM 可观测性最佳实践](https://www.databricks.com/blog/llm-observability)
