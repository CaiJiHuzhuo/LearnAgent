# Python 工程化基础

## 目标与重要性

Python 工程化是 AI Agent 开发的基础。良好的工程实践能让你的代码可维护、可测试、可部署。在 Agent 开发中，工程化尤为重要：错误处理、异步编程、配置管理直接影响 Agent 的稳定性。

## 核心概念清单

- 虚拟环境与依赖管理（venv、pip、poetry）
- 类型注解（Type Hints）与 Pydantic
- 异步编程（async/await、asyncio）
- 错误处理与重试机制
- 配置管理（.env、pydantic-settings）
- 日志系统（loguru、structlog）
- 项目结构与模块化

## 学习路径

### 入门（第1层）

掌握基础的 Python 工程化工具：

```bash
# 创建虚拟环境
python -m venv .venv
source .venv/bin/activate

# 使用 pip-tools 管理依赖
pip install pip-tools
# 创建 requirements.in
echo "openai>=1.0.0" > requirements.in
echo "pydantic>=2.0.0" >> requirements.in
pip-compile requirements.in
pip-sync requirements.txt
```

```python
# 基础类型注解
from typing import Optional, Union, List, Dict, Any
from pydantic import BaseModel, Field
from datetime import datetime

class PaymentOrder(BaseModel):
    order_id: str = Field(..., description="订单ID", pattern=r"^PAY\d+$")
    amount: float = Field(..., gt=0, description="金额，必须大于0")
    currency: str = Field(default="CNY", max_length=3)
    status: str
    created_at: datetime
    metadata: Dict[str, Any] = Field(default_factory=dict)

    class Config:
        # 允许从 ORM 对象创建
        from_attributes = True

# 使用
order = PaymentOrder(
    order_id="PAY202401001",
    amount=299.00,
    status="PENDING",
    created_at=datetime.now()
)
print(order.model_dump_json())
```

### 进阶（第2层）

异步编程和错误处理：

```python
import asyncio
import time
from functools import wraps
from typing import TypeVar, Callable, Any
import logging

logger = logging.getLogger(__name__)

T = TypeVar('T')

def retry_async(
    max_attempts: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    exceptions: tuple = (Exception,)
):
    """异步重试装饰器"""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, **kwargs):
            last_exception = None
            current_delay = delay

            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e
                    if attempt < max_attempts - 1:
                        logger.warning(
                            f"第{attempt+1}次尝试失败: {e}，"
                            f"{current_delay}秒后重试"
                        )
                        await asyncio.sleep(current_delay)
                        current_delay *= backoff
                    else:
                        logger.error(f"所有{max_attempts}次尝试均失败")

            raise last_exception
        return wrapper
    return decorator

# 使用示例
@retry_async(max_attempts=3, delay=1.0, exceptions=(ConnectionError, TimeoutError))
async def call_payment_api(order_id: str) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.get(
            f"https://api.payment.com/orders/{order_id}",
            timeout=aiohttp.ClientTimeout(total=10)
        ) as response:
            response.raise_for_status()
            return await response.json()
```

### 深入（第3层）

配置管理与结构化日志：

```python
# config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import SecretStr, Field

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False
    )

    # OpenAI 配置
    openai_api_key: SecretStr
    openai_model: str = "gpt-4o-mini"
    openai_max_tokens: int = 2000
    openai_temperature: float = 0.1

    # 数据库配置
    database_url: str = "postgresql://localhost/agent_db"

    # 日志配置
    log_level: str = "INFO"
    log_format: str = "json"

    # 业务配置
    max_refund_amount: float = 10000.0
    refund_time_limit_days: int = 7

settings = Settings()

# logger.py
from loguru import logger
import sys

def setup_logger():
    logger.remove()
    logger.add(
        sys.stdout,
        format="{time:YYYY-MM-DD HH:mm:ss.SSS} | {level} | {name}:{line} | {message}",
        level="INFO",
        serialize=True  # JSON 格式
    )
    logger.add(
        "logs/agent.log",
        rotation="100 MB",
        retention="30 days",
        compression="gz"
    )

# 使用结构化日志
logger.info(
    "LLM调用完成",
    extra={
        "trace_id": "abc123",
        "model": "gpt-4o-mini",
        "tokens_used": 450,
        "latency_ms": 1200
    }
)
```

## 最小可执行练习

创建一个带有完整工程化配置的项目骨架：

```
my_agent/
├── .env.example
├── requirements.in
├── requirements.txt
├── pyproject.toml
├── src/
│   └── my_agent/
│       ├── __init__.py
│       ├── config.py
│       ├── logger.py
│       ├── models.py
│       └── main.py
└── tests/
    └── test_models.py
```

```python
# tests/test_models.py
import pytest
from pydantic import ValidationError
from src.my_agent.models import PaymentOrder
from datetime import datetime

def test_valid_order():
    order = PaymentOrder(
        order_id="PAY202401001",
        amount=299.00,
        status="PENDING",
        created_at=datetime.now()
    )
    assert order.amount == 299.00

def test_invalid_order_id():
    with pytest.raises(ValidationError):
        PaymentOrder(
            order_id="INVALID",  # 不符合正则
            amount=299.00,
            status="PENDING",
            created_at=datetime.now()
        )

def test_negative_amount():
    with pytest.raises(ValidationError):
        PaymentOrder(
            order_id="PAY202401001",
            amount=-100.0,  # 不能为负
            status="PENDING",
            created_at=datetime.now()
        )
```

## 常见坑与排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| `ImportError` 找不到模块 | 没有激活虚拟环境 | `source .venv/bin/activate` |
| Pydantic v1 vs v2 API 不兼容 | 版本混用 | 统一使用 v2，注意 `model_dump()` 而非 `dict()` |
| 异步函数在同步上下文中调用 | 事件循环冲突 | 使用 `asyncio.run()` 或 `nest_asyncio` |
| 环境变量未加载 | `.env` 文件路径错误 | 确保从项目根目录运行 |
| 日志中文乱码 | 编码设置问题 | 确保 `encoding='utf-8'` |

## Java/Spring Boot 对接要点

当 Python Agent 需要与 Java Spring Boot 服务交互时：

```java
// 使用 Spring WebClient 调用 Python Agent API
@Service
public class AgentClientService {

    private final WebClient webClient;

    public AgentClientService(WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("http://python-agent-service:8000")
            .defaultHeader("Content-Type", "application/json")
            .build();
    }

    public Mono<AnalysisResult> analyzePaymentFailure(String orderId) {
        return webClient.post()
            .uri("/analyze")
            .bodyValue(Map.of("order_id", orderId))
            .retrieve()
            .bodyToMono(AnalysisResult.class)
            .timeout(Duration.ofSeconds(30))
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)));
    }
}
```

## 参考资料

- [Pydantic 官方文档](https://docs.pydantic.dev/latest/)
- [Python asyncio 文档](https://docs.python.org/3/library/asyncio.html)
- [loguru 文档](https://loguru.readthedocs.io/)
- [pydantic-settings 配置管理](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)

---

## 补充：Agent 专用工程模式

### 依赖注入与测试友好性

```python
from abc import ABC, abstractmethod
from typing import Protocol

# 定义接口（Protocol）
class LLMProvider(Protocol):
    def complete(self, messages: list, **kwargs) -> str: ...

class OrderRepository(Protocol):
    def get_order(self, order_id: str) -> dict: ...

# 实现
class OpenAIProvider:
    def complete(self, messages: list, **kwargs) -> str:
        from openai import OpenAI
        client = OpenAI()
        response = client.chat.completions.create(
            model=kwargs.get("model", "gpt-4o-mini"),
            messages=messages,
            temperature=kwargs.get("temperature", 0.1)
        )
        return response.choices[0].message.content

class MockLLMProvider:
    """测试用的 Mock LLM"""
    def __init__(self, fixed_response: str = "测试响应"):
        self.fixed_response = fixed_response
        self.call_count = 0
        self.last_messages = None

    def complete(self, messages: list, **kwargs) -> str:
        self.call_count += 1
        self.last_messages = messages
        return self.fixed_response

# Agent 使用依赖注入
class PaymentAnalysisAgent:
    def __init__(self, llm: LLMProvider, order_repo: OrderRepository):
        self.llm = llm
        self.order_repo = order_repo

    def analyze(self, order_id: str) -> str:
        order = self.order_repo.get_order(order_id)
        messages = [{"role": "user", "content": f"分析订单: {order}"}]
        return self.llm.complete(messages)

# 测试
mock_llm = MockLLMProvider("余额不足是根因")
mock_repo = type("MockRepo", (), {"get_order": lambda self, id: {"status": "FAILED"}})()
agent = PaymentAnalysisAgent(mock_llm, mock_repo)

result = agent.analyze("PAY001")
assert result == "余额不足是根因"
assert mock_llm.call_count == 1
print("测试通过:", result)
```

### 项目模板最佳实践

```toml
# pyproject.toml
[tool.poetry]
name = "payment-agent"
version = "0.1.0"
description = "支付系统 AI Agent"
python = "^3.10"

[tool.poetry.dependencies]
openai = "^1.0.0"
pydantic = "^2.0.0"
pydantic-settings = "^2.0.0"
loguru = "^0.7.0"
fastapi = "^0.104.0"
uvicorn = "^0.24.0"

[tool.poetry.dev-dependencies]
pytest = "^7.0.0"
pytest-asyncio = "^0.21.0"
pytest-cov = "^4.0.0"
black = "^23.0.0"
ruff = "^0.1.0"

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
line-length = 100
select = ["E", "F", "I"]
```
