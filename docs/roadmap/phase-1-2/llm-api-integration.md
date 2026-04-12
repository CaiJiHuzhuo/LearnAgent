# LLM API 集成

## 目标与重要性

能够稳定、高效地集成 LLM API 是 Agent 开发的基础工程能力。需要掌握错误处理、重试机制、流式输出、多模型支持等工程细节。

## 核心概念清单

- OpenAI API 基础用法
- 异步调用与流式输出
- 错误处理（速率限制、超时、网络错误）
- 多模型适配（OpenAI、Azure、国内模型）
- API Key 安全管理
- 请求/响应拦截与日志

## 学习路径

### 入门（第1层）

```python
from openai import OpenAI, AsyncOpenAI
import os

# 初始化客户端
client = OpenAI(
    api_key=os.getenv("OPENAI_API_KEY"),
    timeout=30.0,           # 总超时
    max_retries=3,          # 自动重试次数
)

# 基础调用
def simple_chat(user_input: str) -> str:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "你是支付系统客服。"},
            {"role": "user", "content": user_input}
        ],
        temperature=0.1,
        max_tokens=500
    )
    return response.choices[0].message.content

# 流式输出
def stream_chat(user_input: str) -> None:
    stream = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": user_input}],
        stream=True
    )
    for chunk in stream:
        if chunk.choices[0].delta.content:
            print(chunk.choices[0].delta.content, end="", flush=True)
    print()  # 换行
```

### 进阶（第2层）

```python
import asyncio
import time
from openai import AsyncOpenAI, RateLimitError, APITimeoutError, APIConnectionError

async_client = AsyncOpenAI()

# 并发调用多个 LLM 请求
async def batch_analyze(orders: list[str]) -> list[str]:
    """并发分析多个订单（注意速率限制）"""
    semaphore = asyncio.Semaphore(5)  # 最多5个并发

    async def analyze_one(order_id: str) -> str:
        async with semaphore:
            response = await async_client.chat.completions.create(
                model="gpt-4o-mini",
                messages=[{
                    "role": "user",
                    "content": f"简要分析订单 {order_id} 的支付风险"
                }],
                max_tokens=100
            )
            return response.choices[0].message.content

    tasks = [analyze_one(order_id) for order_id in orders]
    return await asyncio.gather(*tasks)

# 完善的错误处理
import logging
logger = logging.getLogger(__name__)

async def robust_chat(
    messages: list,
    model: str = "gpt-4o-mini",
    max_retries: int = 3
) -> str:
    """带完整错误处理的 LLM 调用"""
    for attempt in range(max_retries):
        try:
            response = await async_client.chat.completions.create(
                model=model,
                messages=messages,
                timeout=30.0
            )
            return response.choices[0].message.content

        except RateLimitError as e:
            wait_time = 2 ** attempt * 10  # 指数退避
            logger.warning(f"速率限制，等待 {wait_time}s 后重试 (attempt {attempt+1})")
            await asyncio.sleep(wait_time)

        except APITimeoutError:
            logger.warning(f"请求超时 (attempt {attempt+1})")
            if attempt == max_retries - 1:
                raise

        except APIConnectionError as e:
            logger.error(f"网络连接错误: {e}")
            raise

    raise Exception("所有重试均失败")
```

### 深入（第3层）

```python
# 多模型统一接口封装
from abc import ABC, abstractmethod
from typing import Optional
import httpx

class BaseLLMClient(ABC):
    @abstractmethod
    async def chat(self, messages: list, **kwargs) -> str:
        pass

    @abstractmethod
    async def stream_chat(self, messages: list, **kwargs):
        pass

class OpenAIClient(BaseLLMClient):
    def __init__(self, api_key: str, model: str = "gpt-4o-mini"):
        self.client = AsyncOpenAI(api_key=api_key)
        self.model = model

    async def chat(self, messages: list, **kwargs) -> str:
        response = await self.client.chat.completions.create(
            model=kwargs.get("model", self.model),
            messages=messages,
            temperature=kwargs.get("temperature", 0.1),
            max_tokens=kwargs.get("max_tokens", 500)
        )
        return response.choices[0].message.content

    async def stream_chat(self, messages: list, **kwargs):
        stream = await self.client.chat.completions.create(
            model=kwargs.get("model", self.model),
            messages=messages,
            stream=True
        )
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content

# 国内模型（通义千问、文心等）通常兼容 OpenAI 接口
class QwenClient(BaseLLMClient):
    def __init__(self, api_key: str):
        self.client = AsyncOpenAI(
            api_key=api_key,
            base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
        )
        self.model = "qwen-plus"

    async def chat(self, messages: list, **kwargs) -> str:
        response = await self.client.chat.completions.create(
            model=kwargs.get("model", self.model),
            messages=messages,
        )
        return response.choices[0].message.content

    async def stream_chat(self, messages: list, **kwargs):
        stream = await self.client.chat.completions.create(
            model=kwargs.get("model", self.model),
            messages=messages,
            stream=True
        )
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content

# 请求拦截器：记录所有 LLM 调用
class LLMCallLogger:
    def __init__(self):
        self.calls = []

    def log_call(self, messages: list, response: str, metadata: dict = None):
        self.calls.append({
            "timestamp": time.time(),
            "input_messages": messages,
            "output": response,
            "metadata": metadata or {}
        })

    def get_call_history(self) -> list:
        return self.calls
```

## 最小可执行练习

```python
# 构建一个带监控的 LLM 客户端
import time
import json
from dataclasses import dataclass, field
from openai import OpenAI

@dataclass
class LLMCallMetrics:
    total_calls: int = 0
    successful_calls: int = 0
    failed_calls: int = 0
    total_tokens: int = 0
    total_latency_ms: float = 0

    @property
    def avg_latency_ms(self) -> float:
        return self.total_latency_ms / max(self.total_calls, 1)

    @property
    def success_rate(self) -> float:
        return self.successful_calls / max(self.total_calls, 1)

class MonitoredLLMClient:
    def __init__(self):
        self.client = OpenAI()
        self.metrics = LLMCallMetrics()

    def chat(self, messages: list, model: str = "gpt-4o-mini", **kwargs) -> str:
        start_time = time.time()
        self.metrics.total_calls += 1

        try:
            response = self.client.chat.completions.create(
                model=model,
                messages=messages,
                **kwargs
            )
            latency = (time.time() - start_time) * 1000
            self.metrics.successful_calls += 1
            self.metrics.total_tokens += response.usage.total_tokens
            self.metrics.total_latency_ms += latency
            return response.choices[0].message.content

        except Exception as e:
            self.metrics.failed_calls += 1
            raise

    def print_metrics(self):
        print(f"总调用次数: {self.metrics.total_calls}")
        print(f"成功率: {self.metrics.success_rate:.1%}")
        print(f"平均延迟: {self.metrics.avg_latency_ms:.0f}ms")
        print(f"总消耗 Token: {self.metrics.total_tokens}")

# 使用
llm = MonitoredLLMClient()
for i in range(5):
    llm.chat([{"role": "user", "content": f"简短回答：{i+1}+{i+1}=?"}])
llm.print_metrics()
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| `RateLimitError` | 请求太频繁 | 实现指数退避重试 |
| `APITimeoutError` | 网络慢或模型慢 | 增大 timeout，实现重试 |
| API Key 泄露 | 硬编码在代码中 | 使用环境变量，不要提交到 git |
| 响应解析失败 | choices 为空 | 检查 finish_reason |
| 流式输出不完整 | 连接中断 | 实现流重连机制 |

## Java/Spring Boot 对接要点

```java
// 使用 Spring AI 或 langchain4j
@Configuration
public class LlmConfig {

    @Bean
    public OpenAiChatModel chatModel() {
        return OpenAiChatModel.builder()
            .apiKey(System.getenv("OPENAI_API_KEY"))
            .modelName("gpt-4o-mini")
            .temperature(0.1)
            .maxTokens(500)
            .build();
    }
}

@Service
public class ChatService {

    private final OpenAiChatModel chatModel;

    // 流式输出（SSE）
    public Flux<String> streamChat(String userInput) {
        return chatModel.stream(new Prompt(
            List.of(new UserMessage(userInput))
        )).map(response ->
            response.getResult().getOutput().getContent()
        );
    }
}
```

## 参考资料

- [OpenAI Python SDK](https://github.com/openai/openai-python)
- [Spring AI 文档](https://docs.spring.io/spring-ai/reference/)
- [LangChain4j](https://docs.langchain4j.dev/)
