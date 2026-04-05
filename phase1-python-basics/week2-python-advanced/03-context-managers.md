# 上下文管理器（Context Manager）

> 用 `with` 语句优雅地管理资源，类比 Java 的 try-with-resources

## 概述

上下文管理器是 Python 中管理资源（文件、网络连接、数据库连接、锁等）的标准方式，确保资源在使用后正确释放，无论是否发生异常。

| Java | Python |
|------|--------|
| `try-with-resources` (Java 7+) | `with` 语句 |
| `AutoCloseable` 接口 | 实现 `__enter__` 和 `__exit__` |
| `Closeable` 接口 | `contextlib.contextmanager` |

## 1. with 语句基础

```python
# 最常见的用法：文件操作
# ❌ 传统方式（可能忘记关闭）
f = open("data.txt", "w")
f.write("Hello, World!")
# 如果这里抛出异常，f.close() 就不会被调用
f.close()

# ✅ 使用 with 语句（自动关闭）
with open("data.txt", "w", encoding="utf-8") as f:
    f.write("Hello, World!")
    # 无论是否异常，离开 with 块时文件都会被关闭
# 离开 with 块后，f 已关闭：f.closed == True

# Java 对比：
# try (FileWriter fw = new FileWriter("data.txt")) {
#     fw.write("Hello, World!");
# }
```

```python
# 同时管理多个资源
with open("input.txt", "r") as source, open("output.txt", "w") as dest:
    for line in source:
        dest.write(line.upper())

# Python 3.10+ 的括号语法（更易读）
with (
    open("input.txt", "r") as source,
    open("output.txt", "w") as dest
):
    for line in source:
        dest.write(line.upper())
```

## 2. 实现上下文管理器：`__enter__` 和 `__exit__`

```python
import time

class Timer:
    """计时上下文管理器"""
    
    def __enter__(self):
        self.start = time.perf_counter()
        return self  # 返回值赋给 as 子句
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.elapsed = time.perf_counter() - self.start
        print(f"耗时：{self.elapsed:.4f}s")
        return False  # False 表示不抑制异常（异常会继续传播）
    
    # exc_type：异常类型（无异常时为 None）
    # exc_val：异常实例（无异常时为 None）
    # exc_tb：异常 traceback（无异常时为 None）


with Timer() as t:
    time.sleep(0.5)
    print("做了一些工作")
print(f"通过 t.elapsed 访问：{t.elapsed:.4f}s")
```

```python
class DatabaseConnection:
    """数据库连接上下文管理器"""
    
    def __init__(self, host: str, port: int = 5432):
        self.host = host
        self.port = port
        self.connection = None
    
    def __enter__(self):
        print(f"连接到 {self.host}:{self.port}")
        self.connection = self._connect()
        return self.connection
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            print(f"发生异常 {exc_type.__name__}，回滚事务")
            self.connection.rollback()
        else:
            print("提交事务")
            self.connection.commit()
        
        print("关闭数据库连接")
        self.connection.close()
        return False  # 不抑制异常
    
    def _connect(self):
        # 实际连接逻辑
        pass


with DatabaseConnection("localhost") as conn:
    conn.execute("INSERT INTO users VALUES (1, 'Alice')")
    # 正常情况：提交事务并关闭连接
    # 异常情况：回滚事务并关闭连接
```

### `__exit__` 返回值的含义

```python
class SuppressError:
    """抑制特定类型的异常"""
    
    def __init__(self, *exception_types):
        self.exception_types = exception_types
    
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        # 返回 True 表示抑制异常（不再传播）
        if exc_type and issubclass(exc_type, self.exception_types):
            print(f"已抑制异常：{exc_val}")
            return True  # 抑制异常！
        return False  # 不抑制


with SuppressError(FileNotFoundError):
    open("不存在的文件.txt")
    print("这行不会执行")
print("异常被抑制，程序继续运行")

# contextlib 中有内置的 suppress
from contextlib import suppress
with suppress(FileNotFoundError):
    open("不存在的文件.txt")
```

## 3. 使用 `contextlib.contextmanager`

用生成器函数创建上下文管理器（更简洁）：

```python
from contextlib import contextmanager
import time

@contextmanager
def timer():
    """用生成器实现的计时上下文管理器"""
    start = time.perf_counter()
    try:
        yield  # 暂停，执行 with 块内的代码
    finally:
        elapsed = time.perf_counter() - start
        print(f"耗时：{elapsed:.4f}s")


with timer():
    time.sleep(0.3)


@contextmanager
def managed_resource(name: str):
    """通用资源管理"""
    print(f"获取资源：{name}")
    resource = {"name": name, "data": []}
    try:
        yield resource  # 将资源传给 with 语句的 as 变量
    except Exception as e:
        print(f"处理异常：{e}")
        raise  # 重新抛出异常
    finally:
        print(f"释放资源：{name}")
        resource["data"].clear()


with managed_resource("数据库连接") as res:
    res["data"].append("查询结果")
    print(f"使用资源：{res}")
```

## 4. 实用上下文管理器示例

### 临时切换工作目录

```python
import os
from contextlib import contextmanager

@contextmanager
def working_directory(path: str):
    """临时切换工作目录"""
    original = os.getcwd()
    try:
        os.chdir(path)
        yield
    finally:
        os.chdir(original)


with working_directory("/tmp"):
    print(os.getcwd())  # /tmp
    # 在 /tmp 下操作文件

print(os.getcwd())  # 恢复原始目录
```

### 临时修改环境变量

```python
import os
from contextlib import contextmanager

@contextmanager
def env_override(**kwargs):
    """临时修改环境变量"""
    original = {k: os.environ.get(k) for k in kwargs}
    try:
        os.environ.update({k: str(v) for k, v in kwargs.items()})
        yield
    finally:
        for key, value in original.items():
            if value is None:
                os.environ.pop(key, None)
            else:
                os.environ[key] = value


with env_override(DATABASE_URL="sqlite:///:memory:", DEBUG="true"):
    print(os.environ["DATABASE_URL"])  # sqlite:///:memory:

print(os.environ.get("DATABASE_URL"))  # 恢复原值
```

### 线程锁上下文管理器

```python
import threading

# threading.Lock 已经实现了上下文管理器协议
lock = threading.Lock()

# ❌ 手动管理锁（容易出错）
lock.acquire()
try:
    # 临界区代码
    pass
finally:
    lock.release()

# ✅ 使用 with 语句
with lock:
    # 临界区代码（自动获取和释放锁）
    pass
```

### HTTP 会话管理

```python
import httpx

# httpx 的 Client 是上下文管理器
with httpx.Client(timeout=10.0) as client:
    response = client.get("https://api.example.com/data")
    data = response.json()
# 离开 with 块，连接自动关闭

# 异步版本（第三周会详细介绍）
import asyncio

async def fetch_data():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
        return response.json()
```

## 5. contextlib 工具箱

```python
from contextlib import (
    contextmanager,
    asynccontextmanager,  # 异步上下文管理器
    suppress,             # 抑制异常
    redirect_stdout,      # 重定向标准输出
    ExitStack,            # 动态管理多个上下文管理器
)
import io

# 重定向输出（用于测试）
output = io.StringIO()
with redirect_stdout(output):
    print("这些输出会被捕获")
    print("不会显示在终端")
print(f"捕获到的输出：{output.getvalue()!r}")


# ExitStack：动态数量的上下文管理器
files_to_process = ["file1.txt", "file2.txt", "file3.txt"]

with ExitStack() as stack:
    files = [
        stack.enter_context(open(fname, "w"))
        for fname in files_to_process
    ]
    for i, f in enumerate(files):
        f.write(f"这是文件 {i+1} 的内容")
# 所有文件都会被关闭
```

## 常见陷阱与注意事项

### 1. `__exit__` 的返回值

```python
class BadContextManager:
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        # ❌ 隐式返回 None，等同于 False，不抑制异常
        # 但是如果你想要不抑制异常，应该显式 return False
        pass

class GoodContextManager:
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            # 处理异常
            print(f"处理异常：{exc_val}")
        return False  # 明确不抑制异常
```

### 2. 避免在 `__exit__` 中抑制所有异常

```python
class DangerousContextManager:
    def __exit__(self, exc_type, exc_val, exc_tb):
        return True  # ❌ 抑制所有异常！连 KeyboardInterrupt 都抑制！

# 应该只抑制预期的异常类型
class SafeContextManager:
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None and issubclass(exc_type, (ValueError, KeyError)):
            return True  # ✅ 只抑制特定异常
        return False
```

### 3. 生成器上下文管理器中的异常处理

```python
from contextlib import contextmanager

@contextmanager
def managed():
    try:
        yield  # 异常会在这里抛出
    except ValueError:
        print("处理 ValueError")
        # 如果不 re-raise，异常被抑制
    finally:
        print("清理资源")

# ❌ 以下写法会导致异常被意外抑制
@contextmanager
def buggy():
    try:
        yield
    except Exception:  # 太宽泛！
        pass  # 所有异常都被吞掉了

# ✅ 只处理预期的异常，其他的重新抛出
@contextmanager
def correct():
    try:
        yield
    except ValueError as e:
        print(f"处理 ValueError：{e}")
        # 不 raise，ValueError 被抑制
    except Exception:
        raise  # 其他异常重新抛出
    finally:
        print("总是执行清理")
```

## 小结

| 方法 | 语法 | 适用场景 |
|------|------|---------|
| 类实现 | `__enter__`/`__exit__` | 需要保持状态的上下文管理器 |
| 生成器实现 | `@contextmanager` | 简单的上下文管理器 |
| 内置 | `suppress`, `redirect_stdout` | 常见操作的简便方式 |
| 动态管理 | `ExitStack` | 动态数量的上下文管理器 |

**适用场景**：
- 文件操作：`with open(...) as f:`
- 网络连接：`with httpx.Client() as client:`
- 数据库事务：`with db.transaction():`
- 线程锁：`with lock:`
- 临时修改（目录、环境变量）：`@contextmanager`
- 计时/性能监控：`with Timer():`
