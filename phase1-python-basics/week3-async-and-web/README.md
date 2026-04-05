# 第三周：异步编程与 Web 开发

> 掌握 Python 异步编程，用 FastAPI 构建现代 Web 服务

## 🎯 本周学习目标

1. 深入理解 asyncio 异步编程（事件循环、async/await、Task）
2. 熟练使用 httpx/aiohttp 发起同步和异步 HTTP 请求
3. 使用 FastAPI 搭建 REST API 服务（路由、请求体、依赖注入）
4. 掌握 JSON 处理和文件 I/O 操作
5. 配置结构化日志（logging + loguru）
6. 了解 Jupyter Notebook 用于数据探索

## 📅 每日建议安排

| 天 | 上午（1-2h） | 下午（1-2h） | 晚上（1h） |
|----|------------|------------|----------|
| 周一 | [asyncio 基础](01-asyncio.md) | asyncio 实践练习 | 思考异步 vs 多线程 |
| 周二 | [HTTP 客户端](02-http-clients.md) | 异步 HTTP 请求实践 | 对比 requests 与 httpx |
| 周三 | [FastAPI 基础](03-fastapi.md) | 搭建第一个 API | 测试 API |
| 周四 | [JSON 与文件 I/O](04-json-and-file-io.md) | [日志配置](05-logging.md) | 完善 API |
| 周五 | [Jupyter Notebook](06-jupyter-notebook.md) | 综合复习 | 开始练习题 |
| 周末 | [完成练习题](exercises/week3-exercises.md) | 整理第一阶段笔记 | 准备阶段二 |

## 📖 知识点列表

| 序号 | 知识点 | 文件 | 难度 | 预计时间 |
|------|--------|------|------|---------|
| 01 | asyncio 异步编程 | [01-asyncio.md](01-asyncio.md) | ⭐⭐⭐⭐ | 3h |
| 02 | HTTP 请求库 | [02-http-clients.md](02-http-clients.md) | ⭐⭐ | 2h |
| 03 | FastAPI 框架 | [03-fastapi.md](03-fastapi.md) | ⭐⭐⭐ | 3h |
| 04 | JSON 与文件 I/O | [04-json-and-file-io.md](04-json-and-file-io.md) | ⭐⭐ | 1.5h |
| 05 | 日志库 | [05-logging.md](05-logging.md) | ⭐⭐ | 1.5h |
| 06 | Jupyter Notebook | [06-jupyter-notebook.md](06-jupyter-notebook.md) | ⭐ | 1h |
| - | 第三周练习题 | [exercises/week3-exercises.md](exercises/week3-exercises.md) | ⭐⭐⭐⭐ | 5h |

## 💡 本周重点

### 为什么 asyncio 对 AI Agent 至关重要？

AI Agent 的核心工作模式是：
```
用户提问 → 调用 LLM API（等待 2-10s）→ 调用工具 API（并行等待）→ 返回结果
```

使用 asyncio，可以在等待 LLM 的同时并发调用多个工具，显著降低延迟：

```python
import asyncio
import httpx

async def call_llm(prompt: str) -> str:
    async with httpx.AsyncClient() as client:
        # 等待 LLM 响应（通常 2-10s）
        response = await client.post("https://api.openai.com/v1/chat/completions", ...)
        return response.json()["choices"][0]["message"]["content"]

async def search_web(query: str) -> list[str]:
    # 搜索工具（并发执行）
    ...

async def agent_step(user_message: str) -> str:
    # 并发调用多个工具！
    llm_task = asyncio.create_task(call_llm(user_message))
    search_task = asyncio.create_task(search_web(user_message))
    
    llm_result, search_results = await asyncio.gather(llm_task, search_task)
    # ...
```

### FastAPI 是 AI Agent 开发的标配

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class ChatRequest(BaseModel):
    message: str
    session_id: str = "default"

@app.post("/chat")
async def chat(request: ChatRequest) -> dict:
    # 调用 LLM
    response = await call_llm(request.message)
    return {"response": response, "session_id": request.session_id}

# 运行：uvicorn main:app --reload
```

## ✅ 本周检查清单

- [ ] 能解释事件循环（Event Loop）的工作原理
- [ ] 能使用 `async/await` 编写异步函数
- [ ] 能用 `asyncio.gather()` 并发执行多个协程
- [ ] 能用 `asyncio.create_task()` 创建任务
- [ ] 能用 httpx 发起同步和异步 HTTP 请求
- [ ] 能用 FastAPI 创建 GET/POST/PUT/DELETE 接口
- [ ] 理解 FastAPI 的依赖注入
- [ ] 能正确处理 JSON 序列化和反序列化
- [ ] 能配置日志格式和日志级别
- [ ] 完成本周综合实战练习
