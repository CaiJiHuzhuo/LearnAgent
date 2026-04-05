# Python 日志库

> logging 与 loguru：Python 项目的日志最佳实践

## 概述

良好的日志是 AI Agent 可观测性的基础。Python 内置 `logging` 模块功能完善，`loguru` 则提供了更简洁的 API。

| 对比 | Java | Python |
|------|------|--------|
| 内置日志 | `java.util.logging` | `logging`（标准库） |
| 流行框架 | SLF4J + Logback/Log4j | `logging` / `loguru` |
| 日志级别 | TRACE/DEBUG/INFO/WARN/ERROR | DEBUG/INFO/WARNING/ERROR/CRITICAL |
| 结构化日志 | Logstash / structlog | loguru + structlog |

## 1. 标准库 logging

### 基础配置

```python
import logging

# 最简单的配置（适合脚本）
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)

logger = logging.getLogger(__name__)  # 推荐：用模块名命名

logger.debug("调试信息")
logger.info("普通信息")
logger.warning("警告信息")
logger.error("错误信息")
logger.critical("严重错误")

# 带异常信息
try:
    1 / 0
except ZeroDivisionError:
    logger.exception("除零错误")  # 自动附加 traceback
    # 等价于：logger.error("除零错误", exc_info=True)
```

### 使用 f-string vs % 格式化

```python
# ❌ 不推荐：直接用 f-string（即使不打印日志也会格式化字符串）
logger.debug(f"用户 {user_id} 的数据：{expensive_function()}")

# ✅ 推荐：用 % 格式化（懒求值，日志级别不够时不格式化）
logger.debug("用户 %s 的数据：%s", user_id, data)

# 或者：用 is_enabled 检查
if logger.isEnabledFor(logging.DEBUG):
    logger.debug(f"用户 {user_id} 的数据：{expensive_function()}")
```

### 完整的生产级配置

```python
import logging
import logging.config
from pathlib import Path

def setup_logging(
    log_level: str = "INFO",
    log_file: str | None = None,
) -> None:
    """配置日志（生产级）"""
    
    handlers = {
        "console": {
            "class": "logging.StreamHandler",
            "level": log_level,
            "formatter": "standard",
            "stream": "ext://sys.stdout",
        }
    }
    
    if log_file:
        Path(log_file).parent.mkdir(parents=True, exist_ok=True)
        handlers["file"] = {
            "class": "logging.handlers.RotatingFileHandler",
            "level": log_level,
            "formatter": "detailed",
            "filename": log_file,
            "maxBytes": 10 * 1024 * 1024,  # 10MB
            "backupCount": 5,               # 保留 5 个备份
            "encoding": "utf-8",
        }
    
    config = {
        "version": 1,
        "disable_existing_loggers": False,
        "formatters": {
            "standard": {
                "format": "%(asctime)s [%(levelname)-8s] %(name)s: %(message)s",
                "datefmt": "%Y-%m-%d %H:%M:%S",
            },
            "detailed": {
                "format": (
                    "%(asctime)s [%(levelname)-8s] %(name)s "
                    "[%(filename)s:%(lineno)d] %(funcName)s(): %(message)s"
                ),
            },
        },
        "handlers": handlers,
        "root": {
            "level": log_level,
            "handlers": list(handlers.keys()),
        },
        "loggers": {
            # 关闭第三方库的调试日志
            "httpx": {"level": "WARNING"},
            "asyncio": {"level": "WARNING"},
        },
    }
    
    logging.config.dictConfig(config)


# 使用
setup_logging(log_level="DEBUG", log_file="logs/app.log")
logger = logging.getLogger(__name__)
logger.info("日志系统初始化完成")
```

## 2. loguru：简洁的日志库（推荐）

loguru 是一个更现代的日志库，无需复杂配置即可使用。

```bash
pip install loguru
```

### 基础使用

```python
from loguru import logger

# 直接使用，无需配置！
logger.debug("调试信息")
logger.info("普通信息")
logger.warning("警告信息")
logger.error("错误信息")
logger.critical("严重错误")

# 带异常
try:
    1 / 0
except ZeroDivisionError:
    logger.exception("除零错误")  # 自动显示完整 traceback

# f-string 完全安全（懒求值）
user_id = 123
logger.debug(f"用户 {user_id} 登录")  # loguru 自动优化，不影响性能
```

### 配置 loguru

```python
import sys
from loguru import logger

# 移除默认处理器
logger.remove()

# 添加控制台处理器
logger.add(
    sys.stdout,
    level="DEBUG",
    format="<green>{time:YYYY-MM-DD HH:mm:ss}</green> | "
           "<level>{level: <8}</level> | "
           "<cyan>{name}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> - "
           "<level>{message}</level>",
    colorize=True,
)

# 添加文件处理器（自动轮转）
logger.add(
    "logs/app_{time:YYYY-MM-DD}.log",  # 按日期滚动
    level="INFO",
    format="{time:YYYY-MM-DD HH:mm:ss} | {level: <8} | {name}:{function}:{line} - {message}",
    rotation="00:00",     # 每天凌晨轮转
    retention="30 days",  # 保留 30 天
    compression="zip",    # 压缩旧日志
    encoding="utf-8",
)

# 也可以按文件大小轮转
logger.add(
    "logs/app.log",
    rotation="10 MB",   # 超过 10MB 轮转
    retention=5,        # 保留 5 个文件
)
```

### loguru 的高级功能

```python
from loguru import logger
from contextlib import contextmanager

# 绑定上下文（类比 MDC/SLF4J 的 MDC）
request_logger = logger.bind(request_id="REQ-001", user="alice")
request_logger.info("开始处理请求")

# 作用域绑定
with logger.contextualize(session_id="sess-123"):
    logger.info("会话开始")  # 所有日志都会带 session_id
    # 模拟请求处理
    logger.info("处理中...")
    logger.info("会话结束")

# 捕获异常并记录
@logger.catch  # 装饰器：自动捕获并记录异常
def risky_function():
    x = 1 / 0  # 不会崩溃，而是记录日志

risky_function()  # 记录异常但不抛出

# 计时
with logger.catch():
    import time
    time.sleep(0.1)

# 自定义日志级别
logger.level("SUCCESS", no=25, color="<green>", icon="✅")
logger.log("SUCCESS", "任务完成！")
```

## 3. 结构化日志

AI Agent 应用建议使用 JSON 格式日志，便于日志分析系统（ELK、Grafana）处理：

```python
import json
import logging
from datetime import datetime

class JSONFormatter(logging.Formatter):
    """将日志格式化为 JSON（便于 ELK 等系统处理）"""
    
    def format(self, record: logging.LogRecord) -> str:
        log_data = {
            "timestamp": datetime.fromtimestamp(record.created).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
        }
        
        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)
        
        # 合并额外字段
        for key, value in record.__dict__.items():
            if key not in ("name", "msg", "args", "levelname", "created", 
                          "module", "funcName", "lineno", "exc_info", "exc_text"):
                if not key.startswith("_"):
                    log_data[key] = value
        
        return json.dumps(log_data, ensure_ascii=False)


# loguru 的 JSON 序列化（更简单）
from loguru import logger
import sys

logger.remove()
logger.add(
    sys.stdout,
    format="{message}",  # 只输出消息本身
    serialize=True,       # 自动序列化为 JSON！
)

logger.bind(user_id=123, action="login").info("用户登录")
# 输出：{"text": "用户登录", "record": {"level": "INFO", "user_id": 123, "action": "login", ...}}
```

## 4. FastAPI 集成日志

```python
import logging
import time
from fastapi import FastAPI, Request

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
logger = logging.getLogger("myapp")

app = FastAPI()

@app.middleware("http")
async def log_requests(request: Request, call_next):
    """记录所有 HTTP 请求"""
    start = time.time()
    
    logger.info(
        "请求开始",
        extra={
            "method": request.method,
            "url": str(request.url),
            "client": request.client.host if request.client else "unknown",
        }
    )
    
    response = await call_next(request)
    
    logger.info(
        "请求完成",
        extra={
            "method": request.method,
            "url": str(request.url),
            "status_code": response.status_code,
            "duration": f"{time.time()-start:.3f}s",
        }
    )
    
    return response

# 使用 loguru 更简洁
from loguru import logger as log

@app.middleware("http")
async def log_requests_loguru(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    log.info(
        "{method} {url} - {status} ({duration:.3f}s)",
        method=request.method,
        url=request.url.path,
        status=response.status_code,
        duration=time.time() - start,
    )
    return response
```

## 常见陷阱与注意事项

### 1. 不要在库中配置根日志记录器

```python
# ❌ 在库代码中配置 basicConfig（会影响使用该库的应用）
import logging
logging.basicConfig(level=logging.DEBUG)  # 不要在库中这样做！

# ✅ 库代码只获取命名 logger，不配置
logger = logging.getLogger(__name__)  # 让应用来配置如何处理日志
```

### 2. 日志级别的使用规范

```python
# DEBUG：详细的开发调试信息（生产环境关闭）
logger.debug("进入 process_data 函数，参数：%s", data)

# INFO：正常运行的关键节点
logger.info("用户 %s 登录成功", user_id)
logger.info("处理了 %d 条记录", count)

# WARNING：需要关注但不是错误的情况
logger.warning("API 响应时间过长：%.2fs", response_time)
logger.warning("缓存命中率低：%.1f%%", hit_rate)

# ERROR：可恢复的错误
logger.error("数据库查询失败，将使用缓存数据：%s", e)

# CRITICAL：严重错误，可能导致程序崩溃
logger.critical("数据库连接池耗尽，无法处理请求")
```

### 3. 敏感信息脱敏

```python
# ❌ 记录敏感信息
logger.info("用户登录：username=%s, password=%s", username, password)
logger.debug("API 调用：key=%s, data=%s", api_key, user_data)

# ✅ 脱敏处理
def mask_sensitive(value: str, visible: int = 4) -> str:
    """脱敏：显示前几位，其余用 * 替代"""
    if len(value) <= visible:
        return "*" * len(value)
    return value[:visible] + "*" * (len(value) - visible)

logger.info("API 调用：key=%s", mask_sensitive(api_key))
```

## 小结

| 场景 | 推荐方案 |
|------|---------|
| 简单脚本 | `logging.basicConfig()` + `getLogger` |
| 生产应用 | `logging.config.dictConfig()` |
| 新项目 | **loguru**（更简洁，功能更强） |
| 需要 JSON 日志 | loguru `serialize=True` 或自定义 Formatter |
| FastAPI 应用 | loguru + middleware 记录请求日志 |

**最佳实践**：
1. 每个模块用 `logging.getLogger(__name__)`（或 loguru 的全局 `logger`）
2. 不要用 `print` 替代日志（生产环境不好控制）
3. 日志级别按语义使用，不滥用 `DEBUG`
4. 不记录密码、Token 等敏感信息
5. 生产环境使用 `INFO` 或 `WARNING` 级别
