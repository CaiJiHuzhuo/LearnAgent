# asyncio 异步编程

> Python 异步编程核心：事件循环、async/await、Task、并发控制

## 概述

asyncio 是 Python 内置的异步 I/O 框架，特别适合 AI Agent 开发中大量涉及的网络 I/O 场景（调用 LLM API、数据库查询、工具调用等）。

| 概念 | Java 对应 | Python |
|------|---------|--------|
| 协程（Coroutine） | `CompletableFuture` | `async def` 函数 |
| 事件循环 | `EventLoop`（NIO） | `asyncio.get_event_loop()` |
| 等待 | `.get()`, `.join()` | `await` |
| 并发执行 | `CompletableFuture.allOf()` | `asyncio.gather()` |
| 任务 | `Future`/`CompletableFuture` | `asyncio.Task` |
| 超时 | `future.get(timeout, unit)` | `asyncio.wait_for(coro, timeout)` |

## 1. 同步 vs 异步：直观对比

```python
import time
import asyncio

# === 同步方式：顺序执行，总共 3s ===
def sync_fetch(name: str, delay: float) -> str:
    time.sleep(delay)  # 阻塞！线程停在这里
    return f"{name} 完成"

def sync_main():
    start = time.time()
    r1 = sync_fetch("任务A", 1.0)
    r2 = sync_fetch("任务B", 1.0)  # 需要等 A 完成
    r3 = sync_fetch("任务C", 1.0)  # 需要等 B 完成
    print(f"总耗时：{time.time()-start:.1f}s")  # 总耗时：3.0s
    print(r1, r2, r3)


# === 异步方式：并发执行，总共 ~1s ===
async def async_fetch(name: str, delay: float) -> str:
    await asyncio.sleep(delay)  # 非阻塞！挂起协程，让出控制权
    return f"{name} 完成"

async def async_main():
    start = time.time()
    # gather 并发执行三个协程
    r1, r2, r3 = await asyncio.gather(
        async_fetch("任务A", 1.0),
        async_fetch("任务B", 1.0),
        async_fetch("任务C", 1.0),
    )
    print(f"总耗时：{time.time()-start:.1f}s")  # 总耗时：~1.0s
    print(r1, r2, r3)


asyncio.run(async_main())
```

## 2. 基本概念

### 协程函数和协程对象

```python
import asyncio

# async def 定义的是协程函数（Coroutine Function）
async def say_hello(name: str) -> str:
    await asyncio.sleep(1)  # 模拟异步操作（如网络请求）
    return f"Hello, {name}!"

# 调用协程函数返回的是协程对象，不会立即执行！
coro = say_hello("Alice")  # 创建了协程对象，但还没执行
print(type(coro))  # <class 'coroutine'>

# 需要用 await 或 asyncio.run() 来执行
result = asyncio.run(say_hello("Alice"))  # 在脚本顶层使用
print(result)  # Hello, Alice!

# 在异步函数中用 await 执行另一个协程
async def main():
    result = await say_hello("Bob")  # 等待协程完成
    print(result)  # Hello, Bob!

asyncio.run(main())
```

### await 关键字

```python
import asyncio
import httpx

async def fetch_status(url: str) -> int:
    """获取 URL 的 HTTP 状态码"""
    async with httpx.AsyncClient() as client:
        # await 暂停当前协程，等待 HTTP 响应
        # 暂停期间，事件循环可以执行其他协程
        response = await client.get(url, timeout=10.0)
        return response.status_code

async def main():
    # await 只能在 async 函数内使用
    status = await fetch_status("https://httpbin.org/get")
    print(f"状态码：{status}")

asyncio.run(main())
```

## 3. asyncio.gather：并发执行

```python
import asyncio
import time

async def task(name: str, delay: float) -> dict:
    """模拟异步任务（如 API 调用）"""
    await asyncio.sleep(delay)
    return {"name": name, "duration": delay}

async def main():
    start = time.time()
    
    # gather 同时运行多个协程，等待所有完成
    results = await asyncio.gather(
        task("搜索 API", 1.0),
        task("天气 API", 0.5),
        task("新闻 API", 0.8),
    )
    
    print(f"总耗时：{time.time()-start:.2f}s")  # ~1.0s（最长任务的时间）
    for r in results:
        print(f"  {r['name']}: {r['duration']}s")


# gather 处理异常
async def main_with_error():
    async def might_fail(succeed: bool):
        await asyncio.sleep(0.1)
        if not succeed:
            raise ValueError("任务失败！")
        return "成功"
    
    # 默认行为：一个失败则全部取消
    try:
        results = await asyncio.gather(
            might_fail(True),
            might_fail(False),  # 这个会失败
            might_fail(True),
        )
    except ValueError as e:
        print(f"有任务失败：{e}")
    
    # return_exceptions=True：失败的任务返回异常对象，不影响其他
    results = await asyncio.gather(
        might_fail(True),
        might_fail(False),
        might_fail(True),
        return_exceptions=True,
    )
    for r in results:
        if isinstance(r, Exception):
            print(f"任务异常：{r}")
        else:
            print(f"任务成功：{r}")

asyncio.run(main())
```

## 4. Task：后台任务

```python
import asyncio

async def background_task(name: str, delay: float):
    """后台任务"""
    await asyncio.sleep(delay)
    print(f"后台任务 {name} 完成")
    return name

async def main():
    # create_task 立即调度任务，不等待
    task1 = asyncio.create_task(background_task("A", 2.0))
    task2 = asyncio.create_task(background_task("B", 1.0))
    
    print("任务已创建，继续执行其他代码...")
    await asyncio.sleep(0.5)  # 让事件循环有机会运行任务
    print("做了一些其他工作")
    
    # 等待任务完成
    result1 = await task1
    result2 = await task2
    print(f"任务完成：{result1}, {result2}")
    
    # 取消任务
    task3 = asyncio.create_task(background_task("C", 100.0))
    await asyncio.sleep(0.1)
    task3.cancel()  # 取消任务
    
    try:
        await task3
    except asyncio.CancelledError:
        print("任务 C 已取消")


asyncio.run(main())
```

## 5. 超时控制

```python
import asyncio

async def slow_operation():
    await asyncio.sleep(10)
    return "完成"

async def main():
    # asyncio.wait_for：设置超时
    try:
        result = await asyncio.wait_for(slow_operation(), timeout=2.0)
    except asyncio.TimeoutError:
        print("操作超时！")
    
    # asyncio.timeout（Python 3.11+，更现代的语法）
    try:
        async with asyncio.timeout(2.0):
            result = await slow_operation()
    except asyncio.TimeoutError:
        print("操作超时！")

asyncio.run(main())
```

## 6. 异步上下文管理器和迭代器

```python
import asyncio

# 异步上下文管理器
class AsyncDatabase:
    async def __aenter__(self):
        print("连接数据库")
        await asyncio.sleep(0.1)  # 模拟连接时间
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print("关闭数据库连接")
        await asyncio.sleep(0.05)
    
    async def query(self, sql: str) -> list:
        await asyncio.sleep(0.2)  # 模拟查询时间
        return [{"id": 1, "name": "Alice"}]


async def use_db():
    async with AsyncDatabase() as db:  # 使用 async with
        results = await db.query("SELECT * FROM users")
        print(results)


# 异步生成器
async def async_paginate(total: int, page_size: int = 10):
    """模拟异步分页"""
    for page in range(0, total, page_size):
        await asyncio.sleep(0.1)  # 模拟数据库查询
        yield list(range(page, min(page + page_size, total)))

async def use_pagination():
    async for page_data in async_paginate(35, page_size=10):  # 使用 async for
        print(f"获取到 {len(page_data)} 条数据：{page_data[:3]}...")


asyncio.run(use_db())
asyncio.run(use_pagination())
```

## 7. 并发控制：Semaphore

```python
import asyncio
import httpx

# 使用 Semaphore 限制并发数量（类比 Java 的 Semaphore）
async def fetch_with_limit(
    session: httpx.AsyncClient,
    url: str,
    sem: asyncio.Semaphore
) -> dict:
    async with sem:  # 获取许可（超过并发限制则等待）
        response = await session.get(url)
        return {"url": url, "status": response.status_code}


async def main():
    urls = [f"https://httpbin.org/delay/1?n={i}" for i in range(20)]
    
    # 最多同时 5 个并发请求
    sem = asyncio.Semaphore(5)
    
    async with httpx.AsyncClient() as client:
        tasks = [fetch_with_limit(client, url, sem) for url in urls]
        results = await asyncio.gather(*tasks)
    
    print(f"完成 {len(results)} 个请求")
```

## 8. asyncio 事件循环

```python
import asyncio

# 最常用：asyncio.run() 创建事件循环并运行
async def main():
    print("主协程")

asyncio.run(main())  # 推荐方式

# Jupyter Notebook 中（已有事件循环）
# 直接 await，不需要 asyncio.run()
# result = await some_coroutine()

# 在同步代码中运行协程（偶尔需要）
loop = asyncio.new_event_loop()
loop.run_until_complete(main())
loop.close()
```

## 常见陷阱与注意事项

### 1. 忘记 await

```python
async def get_data():
    await asyncio.sleep(1)
    return "data"

async def wrong():
    result = get_data()  # ❌ 忘记 await！result 是协程对象，不是 "data"
    print(result)        # <coroutine object get_data at 0x...>

async def correct():
    result = await get_data()  # ✅
    print(result)               # data
```

### 2. 在异步函数中使用阻塞操作

```python
import asyncio
import time
import requests  # 同步 HTTP 库

# ❌ 在异步函数中使用同步阻塞操作
async def bad_fetch(url: str):
    response = requests.get(url)  # 阻塞事件循环！其他协程无法运行
    return response.text

# ✅ 方法1：使用异步 HTTP 库
import httpx
async def good_fetch(url: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.text

# ✅ 方法2：在线程池中运行阻塞操作
async def run_blocking():
    loop = asyncio.get_event_loop()
    # 在线程池中运行阻塞函数，不阻塞事件循环
    result = await loop.run_in_executor(None, requests.get, "https://example.com")
    return result.text
```

### 3. CPU 密集型任务

```python
# asyncio 只适合 I/O 密集型！
# CPU 密集型任务（数学计算等）应该用 multiprocessing

import asyncio
from concurrent.futures import ProcessPoolExecutor

def cpu_heavy(n: int) -> int:
    """CPU 密集型任务"""
    return sum(range(n))

async def main():
    loop = asyncio.get_event_loop()
    # 在进程池中运行 CPU 密集型任务
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy, 10**8)
    print(result)
```

## 小结

| 场景 | 方案 |
|------|------|
| 运行单个协程 | `asyncio.run(coro)` |
| 并发多个协程 | `await asyncio.gather(*coros)` |
| 创建后台任务 | `task = asyncio.create_task(coro)` |
| 设置超时 | `await asyncio.wait_for(coro, timeout)` |
| 限制并发数 | `sem = asyncio.Semaphore(n)` |
| CPU 密集型 | `run_in_executor(ProcessPoolExecutor(), func)` |
| 同步阻塞操作 | `run_in_executor(None, sync_func)` |

**asyncio 在 AI Agent 中的应用：**
- 并发调用多个 LLM API（比较哪个更快/更好）
- 并发调用多个工具（搜索 + 天气 + 新闻同时获取）
- 流式输出（Streaming Response）
- 长时间运行的 Agent 任务（带超时控制）
