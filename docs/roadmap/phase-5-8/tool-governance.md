# 工具治理

## 目标与重要性

随着工具数量增加，工具治理确保每个工具有明确的权限、监控、版本管理和使用审计。

## 核心概念清单

- 工具治理的核心原理与应用场景
- 主流框架和工具对比
- 生产环境关键设计决策
- 与其他组件的集成方式
- 常见陷阱和解决方案

## 学习路径

### 入门（第1层）

```python
# 工具治理 - 基础概念演示
from openai import OpenAI
from pydantic import BaseModel
from typing import Optional, List
import json, time, asyncio

client = OpenAI()

# 基础数据模型
class AgentConfig(BaseModel):
    name: str
    model: str = "gpt-4o-mini"
    temperature: float = 0.1
    max_iterations: int = 10
    enabled: bool = True

# 基础 工具治理 实现
class BasicToolGovernance:
    """基础 工具治理 实现"""

    def __init__(self, config: AgentConfig):
        self.config = config
        self.client = OpenAI()

    def process(self, query: str, context: dict = None) -> dict:
        """处理请求"""
        messages = [
            {"role": "system", "content": f"你是一个专业的{self.config.name}助手。"},
            {"role": "user", "content": query}
        ]

        if context:
            messages[0]["content"] += f"\n\n相关上下文：{json.dumps(context, ensure_ascii=False)}"

        response = self.client.chat.completions.create(
            model=self.config.model,
            messages=messages,
            temperature=self.config.temperature,
            max_tokens=500
        )

        return {
            "query": query,
            "response": response.choices[0].message.content,
            "tokens_used": response.usage.total_tokens,
        }

# 快速验证
config = AgentConfig(name="工具治理")
agent = BasicToolGovernance(config)
result = agent.process("介绍 工具治理 的主要功能")
print(result["response"][:200])
```

### 进阶（第2层）

```python
# 工具治理 进阶实现：完整功能版本
import asyncio
from dataclasses import dataclass, field
from typing import Callable, Dict, Any
import logging

logger = logging.getLogger(__name__)

@dataclass
class ProcessingResult:
    success: bool
    data: Dict[str, Any]
    metadata: Dict[str, Any] = field(default_factory=dict)
    errors: List[str] = field(default_factory=list)
    processing_time_ms: float = 0

class AdvancedToolGovernance:
    """进阶 工具治理 实现，支持异步、重试、监控"""

    def __init__(self, config: AgentConfig):
        self.config = config
        self.client = OpenAI()
        self._metrics = {
            "total_requests": 0,
            "successful": 0,
            "failed": 0,
            "total_latency_ms": 0
        }
        self._hooks: Dict[str, List[Callable]] = {
            "before_process": [],
            "after_process": [],
            "on_error": []
        }

    def add_hook(self, event: str, callback: Callable):
        """添加处理钩子，支持扩展"""
        if event in self._hooks:
            self._hooks[event].append(callback)

    async def process_async(self, query: str, context: dict = None) -> ProcessingResult:
        """异步处理，支持钩子"""
        start_time = time.time()
        self._metrics["total_requests"] += 1

        # 触发前置钩子
        for hook in self._hooks["before_process"]:
            await asyncio.get_event_loop().run_in_executor(None, hook, query, context)

        try:
            # 核心处理逻辑
            result = await self._async_core_process(query, context)

            processing_time = (time.time() - start_time) * 1000
            self._metrics["successful"] += 1
            self._metrics["total_latency_ms"] += processing_time

            final_result = ProcessingResult(
                success=True,
                data=result,
                metadata={"processing_time_ms": processing_time},
            )

            # 触发后置钩子
            for hook in self._hooks["after_process"]:
                await asyncio.get_event_loop().run_in_executor(None, hook, final_result)

            return final_result

        except Exception as e:
            self._metrics["failed"] += 1
            logger.error(f"{self.config.name} 处理失败: {e}")

            for hook in self._hooks["on_error"]:
                await asyncio.get_event_loop().run_in_executor(None, hook, e)

            return ProcessingResult(
                success=False,
                data={},
                errors=[str(e)],
                processing_time_ms=(time.time() - start_time) * 1000
            )

    async def _async_core_process(self, query: str, context: dict = None) -> dict:
        import asyncio
        loop = asyncio.get_event_loop()

        # 在线程池中运行同步 OpenAI 调用
        result = await loop.run_in_executor(
            None,
            lambda: self.client.chat.completions.create(
                model=self.config.model,
                messages=[
                    {"role": "system", "content": f"你是专业的{self.config.name}系统。"},
                    {"role": "user", "content": f"处理请求：{query}\n上下文：{context or '无'}"}
                ],
                temperature=self.config.temperature
            )
        )

        return {
            "response": result.choices[0].message.content,
            "model": result.model,
            "tokens": result.usage.total_tokens,
        }

    def get_metrics(self) -> dict:
        total = self._metrics["total_requests"]
        return {
            **self._metrics,
            "success_rate": self._metrics["successful"] / max(total, 1),
            "avg_latency_ms": self._metrics["total_latency_ms"] / max(self._metrics["successful"], 1),
        }

# 演示
async def advanced_demo():
    config = AgentConfig(name="工具治理")
    agent = AdvancedToolGovernance(config)

    # 添加监控钩子
    def log_request(query, context):
        logger.info(f"处理请求: {query[:50]}")

    agent.add_hook("before_process", log_request)

    # 并发处理多个请求
    queries = [
        "分析支付失败原因",
        "处理退款申请",
        "查询账户状态"
    ]

    tasks = [agent.process_async(q) for q in queries]
    results = await asyncio.gather(*tasks)

    for query, result in zip(queries, results):
        status = "成功" if result.success else "失败"
        print(f"{status}: {query} - {result.metadata.get('processing_time_ms', 0):.0f}ms")

    print("\n整体指标:", agent.get_metrics())

asyncio.run(advanced_demo())
```

### 深入（第3层）

```python
# 工具治理 生产级实现：分布式、高可用
from typing import Optional
import uuid

class DistributedToolGovernance:
    """生产级 工具治理：支持分布式部署和高可用"""

    def __init__(
        self,
        primary_config: AgentConfig,
        fallback_config: Optional[AgentConfig] = None,
        cache_ttl: int = 300
    ):
        self.primary = AdvancedToolGovernance(primary_config)
        self.fallback = AdvancedToolGovernance(fallback_config) if fallback_config else None
        self.cache: dict = {}
        self.cache_ttl = cache_ttl

    def _cache_key(self, query: str, context: dict = None) -> str:
        import hashlib
        key_data = f"{query}::{json.dumps(context or {}, sort_keys=True)}"
        return hashlib.md5(key_data.encode()).hexdigest()

    async def process(
        self,
        query: str,
        context: dict = None,
        trace_id: str = None,
        use_cache: bool = True
    ) -> ProcessingResult:
        """带降级、缓存的处理"""
        trace_id = trace_id or str(uuid.uuid4())[:8]

        # 检查缓存
        if use_cache:
            cache_key = self._cache_key(query, context)
            if cache_key in self.cache:
                entry = self.cache[cache_key]
                if time.time() - entry["ts"] < self.cache_ttl:
                    logger.info(f"[{trace_id}] 缓存命中: {query[:30]}")
                    return ProcessingResult(
                        success=True,
                        data=entry["data"],
                        metadata={"cached": True, "trace_id": trace_id}
                    )

        # 主处理
        try:
            result = await self.primary.process_async(query, context)
            if result.success and use_cache:
                cache_key = self._cache_key(query, context)
                self.cache[cache_key] = {"data": result.data, "ts": time.time()}
            return result

        except Exception as e:
            logger.warning(f"[{trace_id}] 主处理失败，尝试降级: {e}")

            # 降级到备用
            if self.fallback:
                try:
                    return await self.fallback.process_async(query, context)
                except Exception as fallback_e:
                    logger.error(f"[{trace_id}] 降级也失败: {fallback_e}")

            return ProcessingResult(
                success=False,
                data={},
                errors=[str(e)],
                metadata={"trace_id": trace_id}
            )

# 生产环境配置示例
primary = AgentConfig(name="工具治理", model="gpt-4o", temperature=0.0)
fallback = AgentConfig(name="工具治理-fallback", model="gpt-4o-mini", temperature=0.0)

distributed_agent = DistributedToolGovernance(
    primary_config=primary,
    fallback_config=fallback,
    cache_ttl=600  # 10分钟缓存
)

# 测试降级机制
async def test_resilience():
    result = await distributed_agent.process(
        "分析支付失败",
        context={"order_id": "PAY001", "fail_code": "INSUFFICIENT_BALANCE"},
        trace_id="test-001"
    )
    print(f"结果: {result.success}")
    print(f"缓存: {result.metadata.get('cached', False)}")

asyncio.run(test_resilience())
```

## 最小可执行练习

```python
# 工具治理 端到端练习
import asyncio

async def complete_exercise():
    """完整的 工具治理 练习流程"""
    print("=== 工具治理 练习 ===\n")

    # 1. 创建配置
    config = AgentConfig(
        name="工具治理",
        model="gpt-4o-mini",
        temperature=0.1,
        max_iterations=5
    )
    print(f"1. 创建配置: {config.name}")

    # 2. 初始化服务
    service = AdvancedToolGovernance(config)
    print(f"2. 初始化服务成功")

    # 3. 处理业务场景
    scenarios = [
        ({"query": "支付失败原因分析", "order_id": "PAY001"}, "场景1: 支付分析"),
        ({"query": "退款资格检查", "order_id": "REF001", "amount": 299}, "场景2: 退款检查"),
        ({"query": "账户风险评估", "user_id": "USER001"}, "场景3: 风险评估"),
    ]

    for scenario_data, scenario_name in scenarios:
        query = scenario_data.pop("query")
        result = await service.process_async(query, scenario_data)
        status = "✓" if result.success else "✗"
        print(f"\n3. {scenario_name}")
        print(f"   {status} 状态: {'成功' if result.success else '失败'}")
        if result.success:
            print(f"   响应: {result.data.get('response', '')[:80]}...")
            print(f"   耗时: {result.metadata.get('processing_time_ms', 0):.0f}ms")

    # 4. 查看统计
    print(f"\n4. 运行统计:")
    metrics = service.get_metrics()
    print(f"   总请求: {metrics['total_requests']}")
    print(f"   成功率: {metrics['success_rate']:.0%}")
    print(f"   平均延迟: {metrics['avg_latency_ms']:.0f}ms")

asyncio.run(complete_exercise())
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 处理结果不稳定 | LLM 的随机性 | 设置 temperature=0，使用 structured output |
| 并发时资源竞争 | 共享状态没有锁保护 | 使用 asyncio.Lock 或线程安全数据结构 |
| 内存泄漏 | 缓存无限增长 | 设置 TTL 和最大缓存大小 |
| 降级不生效 | 异常类型不匹配 | 捕获 Exception 基类，增加日志 |
| 指标数据不准确 | 异步场景下的竞态 | 使用原子操作或专用指标库 |

## Java/Spring Boot 对接要点

```java
// 工具治理 的 Spring Boot 实现
@Service
@Slf4j
public class ToolGovernanceService {

    private final ChatClient chatClient;
    private final MeterRegistry meterRegistry;

    private final Counter requestCounter;
    private final Timer requestTimer;

    public ToolGovernanceService(ChatClient chatClient, MeterRegistry meterRegistry) {
        this.chatClient = chatClient;
        this.meterRegistry = meterRegistry;
        this.requestCounter = Counter.builder("tool-governance.requests")
            .register(meterRegistry);
        this.requestTimer = Timer.builder("tool-governance.latency")
            .register(meterRegistry);
    }

    public ProcessResult process(String query, Map<String, Object> context) {
        requestCounter.increment();
        Timer.Sample sample = Timer.start(meterRegistry);

        try {
            String contextJson = objectMapper.writeValueAsString(context);
            String response = chatClient.prompt()
                .system("你是专业的 工具治理 助手")
                .user(String.format("处理请求：%s\n上下文：%s", query, contextJson))
                .call()
                .content();

            return ProcessResult.success(response);

        } catch (Exception e) {
            log.error("工具治理 处理失败: {}", e.getMessage(), e);
            return ProcessResult.failure(e.getMessage());
        } finally {
            sample.stop(requestTimer);
        }
    }
}

// REST 控制器
@RestController
@RequestMapping("/api/v1/tool-governance")
public class ToolGovernanceController {

    private final ToolGovernanceService service;

    @PostMapping("/process")
    public ResponseEntity<ProcessResult> process(@RequestBody ProcessRequest request) {
        ProcessResult result = service.process(request.query(), request.context());
        return result.isSuccess()
            ? ResponseEntity.ok(result)
            : ResponseEntity.internalServerError().body(result);
    }
}
```

## 参考资料

- [LangChain 框架文档](https://python.langchain.com/docs)
- [LangGraph 工作流编排](https://langchain-ai.github.io/langgraph/)
- [Spring AI 文档](https://docs.spring.io/spring-ai/reference/)
- [生产级 LLM 应用最佳实践](https://www.anyscale.com/blog/a-comprehensive-guide-for-building-rag-based-llm-applications)
- [AI Agent 安全指南](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
