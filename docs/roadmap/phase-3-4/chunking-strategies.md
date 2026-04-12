# 分块策略

## 目标与重要性

文档分块是 RAG 系统的基础工程，直接影响检索质量。分块太小丢失上下文，分块太大引入噪声。

## 核心概念清单

- 分块策略的核心原理
- 主流实现方案对比
- 生产环境最佳实践
- 性能与质量的权衡

## 学习路径

### 入门（第1层）

```python
# 分块策略 - 基础示例
from openai import OpenAI
import json

client = OpenAI()

# 基础实现示例
def basic_chunking_strategies_example():
    """演示 分块策略 的基础用法"""
    print("分块策略 基础示例")
    # 实际实现见进阶部分

basic_chunking_strategies_example()
```

### 进阶（第2层）

```python
# 进阶实现：完整的 分块策略 系统
import asyncio
from typing import List, Optional
from pydantic import BaseModel

class ChunkingStrategiesConfig(BaseModel):
    """配置类"""
    enabled: bool = True
    max_results: int = 10
    threshold: float = 0.7

class ChunkingStrategiesService:
    """核心服务类"""

    def __init__(self, config: ChunkingStrategiesConfig):
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
                "content": "你是一个专业的 分块策略 助手。"
            }, {
                "role": "user",
                "content": f"处理以下数据：{data}"
            }],
            temperature=0.1
        )
        return {"llm_response": response.choices[0].message.content}

# 使用示例
async def demo():
    config = ChunkingStrategiesConfig()
    service = ChunkingStrategiesService(config)
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

class ProductionChunkingStrategies:
    """生产级 分块策略 实现"""

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
                "content": f"处理 分块策略 任务：{input_data}"
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
processor = ProductionChunkingStrategies()
for i in range(5):
    result = processor.process_with_cache({"query": f"测试{i}", "order_id": "PAY001"})
    print(f"调用{i+1}: {result}")

print("\n指标:", processor.get_metrics())
```

## 最小可执行练习

```python
# 分块策略 完整练习
def run_exercise():
    """端到端练习"""
    # 1. 初始化
    processor = ProductionChunkingStrategies()

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
// 分块策略 的 Java 集成
@Service
@Slf4j
public class ChunkingStrategiesService {

    private final ChatClient chatClient;

    @Cacheable(value = "chunking-strategies", key = "#request.hashCode()")
    public ProcessResult process(ProcessRequest request) {
        log.info("分块策略 处理请求: {}", request.getQuery());

        String response = chatClient.prompt()
            .system("你是专业的 分块策略 助手")
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

- [LangChain 分块策略 文档](https://python.langchain.com/docs)
- [生产级 RAG 最佳实践](https://www.anyscale.com/blog/a-comprehensive-guide-for-building-rag-based-llm-applications)
- [Spring AI 文档](https://docs.spring.io/spring-ai/reference/)

---

## 补充：分块策略详解

### 主流分块方法对比

```python
from langchain.text_splitter import (
    RecursiveCharacterTextSplitter,
    CharacterTextSplitter,
    TokenTextSplitter,
    MarkdownHeaderTextSplitter,
)
from typing import List

# 方法1：固定大小分块
def fixed_size_chunking(text: str, chunk_size: int = 500, overlap: int = 50) -> List[str]:
    """简单固定大小分块"""
    chunks = []
    for i in range(0, len(text), chunk_size - overlap):
        chunks.append(text[i:i + chunk_size])
    return chunks

# 方法2：递归字符分块（推荐）
def recursive_chunking(text: str, chunk_size: int = 500) -> List[str]:
    """递归按分隔符分块，优先在段落边界分割"""
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=50,
        separators=["\n\n", "\n", "。", "！", "？", "，", " ", ""],
        length_function=len
    )
    return splitter.split_text(text)

# 方法3：语义分块（最精准但最慢）
from openai import OpenAI
import numpy as np

def semantic_chunking(text: str, threshold: float = 0.8) -> List[str]:
    """基于语义相似度的分块：在语义突变处分割"""
    client = OpenAI()

    # 先按句子分割
    sentences = [s.strip() for s in text.split("。") if s.strip()]

    if len(sentences) < 2:
        return [text]

    # 获取每个句子的 Embedding
    embeddings = client.embeddings.create(
        model="text-embedding-3-small",
        input=sentences
    ).data
    emb_vectors = [e.embedding for e in embeddings]

    # 计算相邻句子的相似度
    chunks = []
    current_chunk = [sentences[0]]

    for i in range(1, len(sentences)):
        v1 = np.array(emb_vectors[i-1])
        v2 = np.array(emb_vectors[i])
        similarity = np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))

        if similarity < threshold:
            # 语义突变 → 新建块
            chunks.append("。".join(current_chunk))
            current_chunk = [sentences[i]]
        else:
            current_chunk.append(sentences[i])

    if current_chunk:
        chunks.append("。".join(current_chunk))

    return chunks

# 方法4：Markdown 结构分块
def markdown_chunking(markdown_text: str) -> List[dict]:
    """按 Markdown 标题结构分块，保留层级信息"""
    splitter = MarkdownHeaderTextSplitter(
        headers_to_split_on=[
            ("#", "H1"),
            ("##", "H2"),
            ("###", "H3"),
        ]
    )
    docs = splitter.split_text(markdown_text)
    return [
        {
            "content": doc.page_content,
            "metadata": doc.metadata
        }
        for doc in docs
    ]

# 分块质量评估
def evaluate_chunking(chunks: List[str], min_size: int = 100, max_size: int = 1000) -> dict:
    """评估分块质量"""
    sizes = [len(c) for c in chunks]
    too_small = sum(1 for s in sizes if s < min_size)
    too_large = sum(1 for s in sizes if s > max_size)

    return {
        "total_chunks": len(chunks),
        "avg_size": sum(sizes) / len(sizes) if sizes else 0,
        "min_size": min(sizes) if sizes else 0,
        "max_size": max(sizes) if sizes else 0,
        "too_small_pct": too_small / len(chunks) * 100 if chunks else 0,
        "too_large_pct": too_large / len(chunks) * 100 if chunks else 0,
    }

# 实际测试
sample_text = """退款政策说明

一、退款申请条件
用户在购买商品后，满足以下条件可申请退款：
1. 购买时间在7天以内
2. 商品未使用或未激活
3. 存在质量问题或描述不符

二、退款流程
提交退款申请后，系统将在1-3个工作日内审核。
审核通过后，退款将在3-7个工作日内到账。
退款金额将原路退回至支付账户。

三、特殊情况处理
对于虚拟商品，一经激活不予退款。
促销商品退款按实际支付金额处理。
如有疑问，请联系在线客服。"""

for method_name, method_fn in [
    ("固定大小", lambda t: fixed_size_chunking(t, 200)),
    ("递归分割", lambda t: recursive_chunking(t, 200)),
]:
    chunks = method_fn(sample_text)
    stats = evaluate_chunking(chunks)
    print(f"\n{method_name}分块:")
    print(f"  块数: {stats['total_chunks']}, 平均大小: {stats['avg_size']:.0f}字符")
    print(f"  过小: {stats['too_small_pct']:.0f}%, 过大: {stats['too_large_pct']:.0f}%")
```
