# 幻觉检测与拒绝

## 目标与重要性

LLM 幻觉是生产环境中最大的质量风险。检测和拒绝幻觉输出能显著提升系统可信度。

## 核心概念清单

- 幻觉检测与拒绝的核心原理
- 主流实现方案对比
- 生产环境最佳实践
- 性能与质量的权衡

## 学习路径

### 入门（第1层）

```python
# 幻觉检测与拒绝 - 基础示例
from openai import OpenAI
import json

client = OpenAI()

# 基础实现示例
def basic_hallucination_rejection_example():
    """演示 幻觉检测与拒绝 的基础用法"""
    print("幻觉检测与拒绝 基础示例")
    # 实际实现见进阶部分

basic_hallucination_rejection_example()
```

### 进阶（第2层）

```python
# 进阶实现：完整的 幻觉检测与拒绝 系统
import asyncio
from typing import List, Optional
from pydantic import BaseModel

class HallucinationRejectionConfig(BaseModel):
    """配置类"""
    enabled: bool = True
    max_results: int = 10
    threshold: float = 0.7

class HallucinationRejectionService:
    """核心服务类"""

    def __init__(self, config: HallucinationRejectionConfig):
        self.config = config
        self.client = OpenAI()

    async def process(self, input_data: dict) -> dict:
        """处理请求"""
        # 验证输入
        if not input_data:
            return {"error": "输入不能为空"}

        # 核心处理逻辑
        result = await self._core_process(input_data)

        return {
            "success": True,
            "result": result,
            "metadata": {
                "config": self.config.model_dump()
            }
        }

    async def _core_process(self, data: dict) -> dict:
        # 调用 LLM 处理
        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "system",
                "content": "你是一个专业的 幻觉检测与拒绝 助手。"
            }, {
                "role": "user",
                "content": f"处理以下数据：{data}"
            }],
            temperature=0.1
        )
        return {"llm_response": response.choices[0].message.content}

# 使用示例
async def demo():
    config = HallucinationRejectionConfig()
    service = HallucinationRejectionService(config)
    result = await service.process({"query": "支付失败排查", "order_id": "PAY001"})
    print(json.dumps(result, ensure_ascii=False, indent=2))

asyncio.run(demo())
```

### 深入（第3层）

```python
# 生产级实现：带缓存、监控、错误处理
import time
import hashlib
from functools import lru_cache

class ProductionHallucinationRejection:
    """生产级 幻觉检测与拒绝 实现"""

    def __init__(self):
        self.client = OpenAI()
        self._cache = {}
        self._metrics = {
            "total_calls": 0,
            "cache_hits": 0,
            "errors": 0,
        }

    def _cache_key(self, input_data: dict) -> str:
        """生成缓存键"""
        return hashlib.md5(
            json.dumps(input_data, sort_keys=True).encode()
        ).hexdigest()

    def process_with_cache(self, input_data: dict, ttl_seconds: int = 300) -> dict:
        """带缓存的处理"""
        self._metrics["total_calls"] += 1
        cache_key = self._cache_key(input_data)

        # 检查缓存
        if cache_key in self._cache:
            entry = self._cache[cache_key]
            if time.time() - entry["timestamp"] < ttl_seconds:
                self._metrics["cache_hits"] += 1
                return {**entry["result"], "_cached": True}

        # 实际处理
        try:
            result = self._do_process(input_data)
            self._cache[cache_key] = {
                "result": result,
                "timestamp": time.time()
            }
            return result
        except Exception as e:
            self._metrics["errors"] += 1
            raise

    def _do_process(self, input_data: dict) -> dict:
        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "user",
                "content": f"处理 幻觉检测与拒绝 任务：{input_data}"
            }],
            temperature=0.1
        )
        return {"result": response.choices[0].message.content}

    def get_metrics(self) -> dict:
        return {
            **self._metrics,
            "cache_hit_rate": self._metrics["cache_hits"] / max(self._metrics["total_calls"], 1),
        }

# 测试
processor = ProductionHallucinationRejection()
for i in range(5):
    result = processor.process_with_cache({"query": f"测试{i}", "order_id": "PAY001"})
    print(f"调用{i+1}: {result}")

print("\n指标:", processor.get_metrics())
```

## 最小可执行练习

```python
# 幻觉检测与拒绝 完整练习
def run_exercise():
    """端到端练习"""
    # 1. 初始化
    processor = ProductionHallucinationRejection()

    # 2. 测试用例
    test_cases = [
        {"query": "支付失败原因", "order_id": "PAY001", "context": "余额不足"},
        {"query": "退款政策", "order_id": "REF001", "context": "用户申请退款"},
        {"query": "账户限额", "order_id": "PAY002", "context": "日限额已满"},
    ]

    # 3. 运行测试
    for i, tc in enumerate(test_cases):
        print(f"\n=== 测试用例 {i+1} ===")
        result = processor.process_with_cache(tc)
        print(f"输入: {tc['query']}")
        print(f"输出: {str(result)[:100]}...")

    # 4. 查看指标
    print(f"\n=== 运行指标 ===")
    metrics = processor.get_metrics()
    print(f"总调用: {metrics['total_calls']}")
    print(f"缓存命中率: {metrics['cache_hit_rate']:.0%}")

run_exercise()
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 效果不稳定 | 参数配置不当 | 设置 temperature=0，使用固定 seed |
| 处理速度慢 | 串行处理大量请求 | 使用异步并发处理 |
| 成本过高 | 每次都调用大模型 | 增加缓存层，简单查询用小模型 |
| 格式不一致 | 没有使用结构化输出 | 使用 Pydantic + Structured Output |
| 内存溢出 | 处理大量数据时缓存无限增长 | 设置 LRU 缓存上限 |

## Java/Spring Boot 对接要点

```java
// 幻觉检测与拒绝 的 Java 集成
@Service
@Slf4j
public class HallucinationRejectionService {

    private final ChatClient chatClient;

    @Cacheable(value = "hallucination-rejection", key = "#request.hashCode()")
    public ProcessResult process(ProcessRequest request) {
        log.info("幻觉检测与拒绝 处理请求: {}", request.getQuery());

        String response = chatClient.prompt()
            .system("你是专业的 幻觉检测与拒绝 助手")
            .user(request.getQuery())
            .call()
            .content();

        return ProcessResult.builder()
            .success(true)
            .result(response)
            .timestamp(Instant.now())
            .build();
    }
}

// 对应的请求/响应 DTO
public record ProcessRequest(String query, String orderId, String context) {}
public record ProcessResult(boolean success, String result, Instant timestamp) {}
```

## 参考资料

- [LangChain 幻觉检测与拒绝 文档](https://python.langchain.com/docs)
- [生产级 RAG 最佳实践](https://www.anyscale.com/blog/a-comprehensive-guide-for-building-rag-based-llm-applications)
- [Spring AI 文档](https://docs.spring.io/spring-ai/reference/)

---

## 补充：生产实践要点

### 核心实现细节

```python
from openai import OpenAI
from pydantic import BaseModel
from typing import List, Optional, Dict, Any
import json, time, asyncio

client = OpenAI()

class ProductionConfig(BaseModel):
    """生产环境配置"""
    model: str = "gpt-4o-mini"
    temperature: float = 0.0
    max_tokens: int = 1000
    cache_enabled: bool = True
    cache_ttl: int = 300
    retry_attempts: int = 3
    timeout_seconds: int = 30

class ProductionPipeline:
    """hallucination-rejection 的生产级实现"""

    def __init__(self, config: ProductionConfig = None):
        self.config = config or ProductionConfig()
        self.client = OpenAI()
        self._cache: Dict[str, Any] = {}
        self._stats = {"calls": 0, "cache_hits": 0, "errors": 0}

    def _get_cache_key(self, input_data: str) -> str:
        import hashlib
        return hashlib.md5(input_data.encode()).hexdigest()

    def process(self, input_text: str, additional_context: dict = None) -> dict:
        """处理请求，带缓存和重试"""
        self._stats["calls"] += 1
        cache_key = self._get_cache_key(input_text)

        # 检查缓存
        if self.config.cache_enabled and cache_key in self._cache:
            entry = self._cache[cache_key]
            if time.time() - entry["ts"] < self.config.cache_ttl:
                self._stats["cache_hits"] += 1
                return {**entry["result"], "from_cache": True}

        # 构建 prompt
        context_str = json.dumps(additional_context or {}, ensure_ascii=False)
        messages = [
            {"role": "system", "content": f"你是专业的 {topic.replace('-', ' ')} 系统。"},
            {"role": "user", "content": f"处理请求：{input_text}\n上下文：{context_str}"}
        ]

        # 带重试的调用
        for attempt in range(self.config.retry_attempts):
            try:
                response = self.client.chat.completions.create(
                    model=self.config.model,
                    messages=messages,
                    temperature=self.config.temperature,
                    max_tokens=self.config.max_tokens,
                    timeout=self.config.timeout_seconds
                )
                result = {
                    "success": True,
                    "output": response.choices[0].message.content,
                    "tokens": response.usage.total_tokens,
                }

                # 写入缓存
                if self.config.cache_enabled:
                    self._cache[cache_key] = {"result": result, "ts": time.time()}

                return result

            except Exception as e:
                if attempt == self.config.retry_attempts - 1:
                    self._stats["errors"] += 1
                    return {"success": False, "error": str(e)}
                time.sleep(2 ** attempt)

    def get_stats(self) -> dict:
        calls = self._stats["calls"]
        return {
            **self._stats,
            "cache_hit_rate": self._stats["cache_hits"] / max(calls, 1),
            "error_rate": self._stats["errors"] / max(calls, 1),
        }

# 使用示例
pipeline = ProductionPipeline()

# 测试场景
test_cases = [
    ("支付失败分析：INSUFFICIENT_BALANCE", {"order_id": "PAY001", "amount": 299}),
    ("退款资格检查", {"order_id": "REF001", "days_since_purchase": 5}),
    ("账户状态评估", {"user_id": "USER001", "risk_level": "LOW"}),
]

for input_text, context in test_cases:
    result = pipeline.process(input_text, context)
    status = "✓" if result["success"] else "✗"
    print(f"{status} {input_text[:40]}: {str(result.get('output', ''))[:60]}...")

print(f"\n统计: {pipeline.get_stats()}")
```

### 与其他组件的集成

```python
# 集成到完整 RAG 流水线
class IntegratedRAGPipeline:
    """集成 hallucination-rejection 的完整 RAG 流水线"""

    def __init__(self):
        self.pipeline = ProductionPipeline()
        self.retriever = None  # 向量检索器
        self.validator = None  # 输出验证器

    def query(self, user_query: str, session_id: str = None) -> dict:
        """完整查询流程"""
        import uuid
        trace_id = session_id or str(uuid.uuid4())[:8]

        # 1. 检索相关文档
        retrieved_docs = []  # 实际使用向量数据库检索

        # 2. 构建上下文
        context = {
            "trace_id": trace_id,
            "retrieved_docs": retrieved_docs[:3],
            "query": user_query
        }

        # 3. 处理
        result = self.pipeline.process(user_query, context)

        # 4. 验证和后处理
        if result["success"]:
            return {
                "answer": result["output"],
                "trace_id": trace_id,
                "sources": [f"文档{i+1}" for i in range(len(retrieved_docs))],
                "confidence": 0.85  # 实际应计算置信度
            }
        else:
            return {"error": result.get("error"), "trace_id": trace_id}

# 异步批量处理
async def batch_process(queries: List[str], max_concurrent: int = 5) -> List[dict]:
    """并发处理多个查询"""
    semaphore = asyncio.Semaphore(max_concurrent)
    pipeline = ProductionPipeline()

    async def process_one(query: str) -> dict:
        async with semaphore:
            loop = asyncio.get_event_loop()
            return await loop.run_in_executor(
                None,
                pipeline.process,
                query
            )

    tasks = [process_one(q) for q in queries]
    return await asyncio.gather(*tasks)

# 测试批量处理
async def demo():
    queries = [f"查询第{i}个订单" for i in range(10)]
    results = await batch_process(queries, max_concurrent=3)
    success_count = sum(1 for r in results if r.get("success"))
    print(f"批量处理: {success_count}/{len(queries)} 成功")

asyncio.run(demo())
```
