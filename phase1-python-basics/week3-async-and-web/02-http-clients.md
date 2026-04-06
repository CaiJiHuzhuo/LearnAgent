# HTTP 请求库

> requests、httpx、aiohttp：Python 中的 HTTP 客户端选择

## 概述

Python 有多个 HTTP 客户端库，各有适用场景：

| 库 | 类型 | 特点 | 推荐场景 |
|----|------|------|---------|
| `requests` | 同步 | 简单易用，生态成熟 | 同步脚本、简单爬虫 |
| `httpx` | 同步+异步 | 与 requests 相似 API，支持 async | **AI Agent 开发首选** |
| `aiohttp` | 异步 | 高性能，功能丰富 | 高并发异步服务 |

Java 对应：Java 的 `HttpClient`（Java 11+）、OkHttp、RestTemplate 等。

## 1. requests：经典同步 HTTP 客户端

```bash
pip install requests
```

### 基本使用

```python
import requests

# GET 请求
response = requests.get("https://api.github.com/users/python")
print(response.status_code)  # 200
print(response.headers["Content-Type"])  # application/json; charset=utf-8
data = response.json()  # 自动解析 JSON
print(data["login"])  # python

# 带查询参数
params = {"q": "python", "sort": "stars", "per_page": 5}
response = requests.get("https://api.github.com/search/repositories", params=params)
# 等价于：GET /search/repositories?q=python&sort=stars&per_page=5

# POST 请求（JSON 数据）
payload = {"username": "alice", "password": "secret"}
response = requests.post(
    "https://api.example.com/auth/login",
    json=payload  # 自动设置 Content-Type: application/json
)

# POST 请求（表单数据）
response = requests.post(
    "https://api.example.com/upload",
    data={"field": "value"},  # application/x-www-form-urlencoded
)

# 设置请求头
headers = {
    "Authorization": "Bearer your-token",
    "User-Agent": "MyApp/1.0",
    "Accept": "application/json",
}
response = requests.get("https://api.example.com/data", headers=headers)

# 超时设置（重要！避免永久等待）
response = requests.get("https://api.example.com", timeout=10.0)  # 10秒超时
response = requests.get("https://api.example.com", timeout=(3.05, 27))  # (连接超时, 读取超时)

# 处理错误
response = requests.get("https://api.example.com/not-found")
response.raise_for_status()  # 如果 4xx/5xx，抛出 HTTPError
```

### Session 对象（重用连接）

```python
import requests

# Session 复用 TCP 连接，提高性能（类比 Java 的 connection pool）
with requests.Session() as session:
    session.headers.update({
        "Authorization": "Bearer your-token",
        "Content-Type": "application/json",
    })
    
    # 所有请求共享 headers 和 cookies
    r1 = session.get("https://api.example.com/users")
    r2 = session.get("https://api.example.com/posts")
    r3 = session.post("https://api.example.com/comments", json={"text": "..."})
```

## 2. httpx：现代 HTTP 客户端（推荐）

httpx 是 requests 的现代替代品，API 几乎完全兼容，额外支持异步。

```bash
pip install httpx
```

### 同步使用（与 requests 几乎相同）

```python
import httpx

# 完全兼容 requests 的 API
with httpx.Client(timeout=10.0) as client:
    response = client.get("https://api.github.com/users/python")
    print(response.status_code)  # 200
    data = response.json()

# 直接使用（不推荐，每次创建新连接）
response = httpx.get("https://httpbin.org/get")
```

### 异步使用（AI Agent 场景的关键！）

```python
import asyncio
import httpx
from typing import Any

async def fetch_json(url: str, **kwargs: Any) -> dict:
    """异步获取 JSON 数据"""
    async with httpx.AsyncClient(timeout=30.0) as client:
        response = await client.get(url, **kwargs)
        response.raise_for_status()
        return response.json()


async def call_openai_api(
    messages: list[dict],
    model: str = "gpt-3.5-turbo",
    api_key: str = "your-api-key",
) -> str:
    """异步调用 OpenAI API"""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "https://api.openai.com/v1/chat/completions",
            headers={"Authorization": f"Bearer {api_key}"},
            json={
                "model": model,
                "messages": messages,
                "temperature": 0.7,
            },
            timeout=60.0,  # LLM 可能需要较长时间
        )
        response.raise_for_status()
        return response.json()["choices"][0]["message"]["content"]


async def main():
    # 并发调用多个 API
    results = await asyncio.gather(
        fetch_json("https://api.github.com/users/python"),
        fetch_json("https://api.github.com/repos/python/cpython"),
        return_exceptions=True,
    )
    for r in results:
        if isinstance(r, Exception):
            print(f"请求失败：{r}")
        else:
            print(f"成功：{r.get('login') or r.get('full_name')}")


asyncio.run(main())
```

### httpx 的高级功能

```python
import httpx

# 自定义传输层（代理、SSL 等）
proxies = {"http://": "http://proxy.example.com:8080"}
client = httpx.Client(proxies=proxies)

# 禁用 SSL 验证（不推荐在生产环境使用）
client = httpx.Client(verify=False)

# 自定义重试（需要 tenacity 库）
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
async def fetch_with_retry(url: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()

# 流式响应（用于 LLM 流式输出）
async def stream_llm_response(prompt: str) -> None:
    """处理 LLM 的流式输出"""
    async with httpx.AsyncClient() as client:
        async with client.stream(
            "POST",
            "https://api.openai.com/v1/chat/completions",
            json={
                "model": "gpt-3.5-turbo",
                "messages": [{"role": "user", "content": prompt}],
                "stream": True,  # 开启流式
            },
            headers={"Authorization": "Bearer your-key"},
        ) as response:
            async for line in response.aiter_lines():
                if line.startswith("data: "):
                    data = line[6:]
                    if data == "[DONE]":
                        break
                    import json
                    chunk = json.loads(data)
                    delta = chunk["choices"][0]["delta"].get("content", "")
                    if delta:
                        print(delta, end="", flush=True)
    print()  # 换行
```

## 3. aiohttp：高性能异步客户端

```bash
pip install aiohttp
```

```python
import asyncio
import aiohttp
from typing import Any

async def fetch(session: aiohttp.ClientSession, url: str) -> Any:
    """使用 aiohttp 异步获取数据"""
    async with session.get(url) as response:
        return await response.json()


async def main():
    # aiohttp 需要手动管理 session
    timeout = aiohttp.ClientTimeout(total=30)
    
    async with aiohttp.ClientSession(timeout=timeout) as session:
        # 并发请求
        urls = [
            "https://api.github.com/users/python",
            "https://api.github.com/users/requests",
        ]
        tasks = [fetch(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        
        for r in results:
            print(r.get("login"))

asyncio.run(main())
```

## 4. 异常处理

```python
import httpx

async def robust_fetch(url: str) -> dict | None:
    """带完整错误处理的 HTTP 请求"""
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(url, timeout=10.0)
            response.raise_for_status()
            return response.json()
    
    except httpx.TimeoutException:
        print(f"请求超时：{url}")
        return None
    
    except httpx.HTTPStatusError as e:
        print(f"HTTP 错误 {e.response.status_code}：{url}")
        if e.response.status_code == 429:
            print("触发限流，请等待后重试")
        elif e.response.status_code == 401:
            print("认证失败，请检查 API Key")
        return None
    
    except httpx.RequestError as e:
        print(f"网络错误：{e}")
        return None
    
    except Exception as e:
        print(f"未知错误：{e}")
        raise
```

## 5. 实际项目中的 HTTP 客户端封装

```python
import asyncio
from typing import Any, Optional
import httpx
from pydantic import BaseModel


class APIClientConfig(BaseModel):
    base_url: str
    api_key: str
    timeout: float = 30.0
    max_retries: int = 3


class LLMAPIClient:
    """封装 LLM API 调用的客户端"""
    
    def __init__(self, config: APIClientConfig):
        self.config = config
        self._client: Optional[httpx.AsyncClient] = None
    
    async def __aenter__(self) -> "LLMAPIClient":
        self._client = httpx.AsyncClient(
            base_url=self.config.base_url,
            headers={"Authorization": f"Bearer {self.config.api_key}"},
            timeout=self.config.timeout,
        )
        return self
    
    async def __aexit__(self, *args: Any) -> None:
        if self._client:
            await self._client.aclose()
    
    async def chat(
        self,
        messages: list[dict[str, str]],
        model: str = "gpt-3.5-turbo",
    ) -> str:
        """发送聊天请求"""
        if not self._client:
            raise RuntimeError("Client not initialized. Use async with.")
        
        for attempt in range(self.config.max_retries):
            try:
                response = await self._client.post(
                    "/v1/chat/completions",
                    json={"model": model, "messages": messages},
                )
                response.raise_for_status()
                return response.json()["choices"][0]["message"]["content"]
            
            except httpx.HTTPStatusError as e:
                if e.response.status_code == 429:
                    wait = 2 ** attempt
                    await asyncio.sleep(wait)
                    continue
                raise
        
        raise Exception(f"请求失败，已重试 {self.config.max_retries} 次")


# 使用
async def main():
    config = APIClientConfig(
        base_url="https://api.openai.com",
        api_key="your-api-key",
    )
    
    async with LLMAPIClient(config) as client:
        response = await client.chat([
            {"role": "user", "content": "介绍一下 Python 的异步编程"}
        ])
        print(response)

asyncio.run(main())
```

## 常见陷阱与注意事项

### 1. 不要重复创建 AsyncClient

```python
# ❌ 每次请求都创建新 client（性能差，连接不能复用）
async def bad_approach():
    for url in urls:
        async with httpx.AsyncClient() as client:
            await client.get(url)

# ✅ 复用 client（连接池）
async def good_approach():
    async with httpx.AsyncClient() as client:
        for url in urls:
            await client.get(url)

# 或在应用启动时创建，关闭时销毁
```

### 2. 设置合理的超时

```python
# ❌ 没有超时（可能永久等待）
response = httpx.get("https://api.example.com")

# ✅ 设置超时
response = httpx.get("https://api.example.com", timeout=10.0)

# 更精细的控制
timeout = httpx.Timeout(
    connect=5.0,   # 连接超时
    read=30.0,     # 读取超时（LLM 需要更长）
    write=5.0,     # 写入超时
    pool=5.0,      # 从连接池获取连接的超时
)
response = httpx.get("https://api.example.com", timeout=timeout)
```

## 小结

| 需求 | 推荐 |
|------|------|
| 简单同步脚本 | `requests` |
| AI Agent（主要是异步） | `httpx`（异步模式） |
| 高性能异步服务 | `httpx` 或 `aiohttp` |
| 流式 LLM 输出 | `httpx`（`async with client.stream(...)`） |
| 调用 OpenAI 等官方 SDK | 用官方 SDK（`openai`、`anthropic`），内部已使用 httpx |

**对 Java 开发者的类比**：
- `requests.Session` ≈ OkHttp 的 `OkHttpClient`（连接池）
- `httpx.AsyncClient` ≈ Java 11 的 `HttpClient`（异步）
- `aiohttp.ClientSession` ≈ Netty 的异步 HTTP 客户端
