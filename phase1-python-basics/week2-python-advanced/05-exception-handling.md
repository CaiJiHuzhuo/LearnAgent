# Python 异常处理最佳实践

> 编写健壮的 Python 代码：正确处理异常，而不是掩盖它们

## 概述

Python 的异常处理哲学与 Java 有所不同。Java 有受检异常（Checked Exception）强制你处理，Python 则鼓励"请求宽恕而不是请求许可"（EAFP）。但这不意味着可以随意吞掉异常。

| 概念 | Java | Python |
|------|------|--------|
| 受检异常 | 必须声明或处理 | 无，所有异常都是"非受检" |
| 异常层次 | `Throwable → Exception → RuntimeException` | `BaseException → Exception` |
| 多异常捕获 | `catch (A \| B e)` | `except (A, B) as e:` |
| finally | ✅ | ✅ |
| else（无异常时） | ❌ | ✅ |
| try-with-resources | ✅ | `with` 语句 |

## 1. 异常层次结构

```python
# Python 异常继承树（简化版）
# BaseException
# ├── SystemExit           # sys.exit() 触发，通常不应捕获
# ├── KeyboardInterrupt    # Ctrl+C，通常不应捕获
# ├── GeneratorExit        # 生成器关闭时触发
# └── Exception            # 通常只捕获这个及其子类
#     ├── ArithmeticError
#     │   ├── ZeroDivisionError
#     │   └── OverflowError
#     ├── LookupError
#     │   ├── IndexError   # list[99] 越界
#     │   └── KeyError     # dict["不存在的键"]
#     ├── ValueError       # 值不合法（如 int("abc")）
#     ├── TypeError        # 类型不匹配（如 1 + "a"）
#     ├── AttributeError   # 访问不存在的属性
#     ├── FileNotFoundError
#     ├── ConnectionError
#     ├── TimeoutError
#     └── RuntimeError
#         └── RecursionError

# 查看异常层次
print(ZeroDivisionError.__mro__)
# (<class 'ZeroDivisionError'>, <class 'ArithmeticError'>, <class 'Exception'>, 
#  <class 'BaseException'>, <class 'object'>)
```

## 2. 基本异常处理语法

```python
# 完整的 try-except-else-finally 结构
try:
    result = int("42")          # 尝试执行
except ValueError as e:
    print(f"值错误：{e}")        # 处理特定异常
except (TypeError, AttributeError) as e:
    print(f"类型/属性错误：{e}") # 处理多种异常
except Exception as e:
    print(f"未知错误：{e}")      # 兜底（最好记录日志而不只是打印）
    raise                        # 重新抛出！不要静默吞掉未知异常
else:
    print(f"成功，结果：{result}")  # 无异常时执行（Java 没有这个！）
finally:
    print("总是执行（清理资源）")    # 无论如何都执行


# 重新抛出异常（保留原始 traceback）
try:
    risky_operation()
except ConnectionError as e:
    logger.error(f"连接失败：{e}")
    raise  # 重新抛出，保留完整堆栈信息

# 链式异常（包装异常）
try:
    data = json.loads(raw_data)
except json.JSONDecodeError as e:
    raise ValueError(f"无效的 JSON 数据：{raw_data!r}") from e
    # from e 保留了原始异常的信息（__cause__）
```

## 3. 自定义异常

```python
# 自定义异常层次结构（最佳实践）
class AppError(Exception):
    """应用基础异常"""
    pass

class ValidationError(AppError):
    """数据校验异常"""
    def __init__(self, field: str, message: str):
        self.field = field
        self.message = message
        super().__init__(f"字段 '{field}' 校验失败：{message}")

class NotFoundError(AppError):
    """资源未找到异常"""
    def __init__(self, resource_type: str, resource_id: str):
        self.resource_type = resource_type
        self.resource_id = resource_id
        super().__init__(f"{resource_type} '{resource_id}' 不存在")

class RateLimitError(AppError):
    """限流异常"""
    def __init__(self, limit: int, window: str):
        self.limit = limit
        self.window = window
        super().__init__(f"超出限流：{window} 内最多 {limit} 次请求")


# 使用自定义异常
def get_user(user_id: str) -> dict:
    users = {"U001": {"name": "Alice"}}
    if user_id not in users:
        raise NotFoundError("用户", user_id)
    return users[user_id]

try:
    user = get_user("U999")
except NotFoundError as e:
    print(f"错误：{e}")
    print(f"类型：{e.resource_type}，ID：{e.resource_id}")
```

## 4. EAFP vs LBYL

EAFP（Easier to Ask Forgiveness than Permission）是 Python 推荐的风格：

```python
# LBYL（Look Before You Leap）风格 - Java 更常见
def get_config_value(config: dict, key: str) -> str:
    if key in config:                  # 先检查
        value = config[key]
        if isinstance(value, str):     # 再检查类型
            if len(value) > 0:         # 再检查非空
                return value
    return "default"

# EAFP（Easier to Ask Forgiveness than Permission）风格 - Python 推荐
def get_config_value(config: dict, key: str) -> str:
    try:
        value = config[key]
        if not isinstance(value, str) or not value:
            raise ValueError("值为空或不是字符串")
        return value
    except (KeyError, ValueError):
        return "default"

# 更简洁的 EAFP 写法
def get_config_value(config: dict, key: str) -> str:
    return config.get(key) or "default"
```

**什么时候用哪种风格？**

| 场景 | 推荐风格 | 原因 |
|------|---------|------|
| 文件是否存在 | EAFP | 文件可能在检查后被删除（竞态条件） |
| 字典键是否存在 | EAFP（`.get()`）| 更简洁 |
| 类型检查 | LBYL（`isinstance`）| 在处理前检查类型更清晰 |
| 用户输入校验 | LBYL | 提前校验，给出明确错误信息 |

## 5. 上下文相关的异常处理

```python
import logging

logger = logging.getLogger(__name__)

# API 调用中的异常处理示例
async def call_llm_api(prompt: str, max_retries: int = 3) -> str:
    """调用 LLM API，带重试逻辑"""
    import asyncio
    
    for attempt in range(1, max_retries + 1):
        try:
            response = await _make_api_request(prompt)
            return response
        except TimeoutError:
            if attempt == max_retries:
                raise  # 最后一次重试仍然超时，抛出异常
            wait = 2 ** attempt  # 指数退避：2, 4, 8 秒
            logger.warning(f"第 {attempt} 次超时，{wait}s 后重试")
            await asyncio.sleep(wait)
        except (ConnectionError, OSError) as e:
            logger.error(f"网络错误（不重试）：{e}")
            raise
        except Exception as e:
            logger.exception(f"未预期的错误：{e}")  # 记录完整堆栈
            raise
```

## 6. 异常处理反模式

```python
# ❌ 反模式1：吞掉所有异常（最危险！）
try:
    process_data()
except:              # 连 KeyboardInterrupt 也捕获了！
    pass             # 静默失败，问题被掩盖

# ❌ 反模式2：捕获 Exception 但不处理
try:
    process_data()
except Exception:
    pass  # 同样危险，问题被掩盖

# ❌ 反模式3：过宽泛的捕获
try:
    result = int(user_input)
    data = json.loads(result)
    write_to_db(data)
except Exception as e:
    print("出错了")  # 不知道是哪里出错，也不知道怎么处理

# ✅ 正确：针对性捕获，各处理各的
try:
    result = int(user_input)
except ValueError:
    return error_response("输入必须是数字")

try:
    data = json.loads(result)
except json.JSONDecodeError:
    return error_response("解析 JSON 失败")

try:
    write_to_db(data)
except DatabaseError as e:
    logger.error(f"数据库写入失败：{e}")
    raise
```

## 7. 异常处理的最佳实践

```python
import logging
from typing import Optional

logger = logging.getLogger(__name__)

# 最佳实践1：捕获具体异常，不要用裸 except
# ❌
try:
    value = d[key]
except:
    value = default

# ✅
try:
    value = d[key]
except KeyError:
    value = default

# 最佳实践2：记录异常但不吞掉
try:
    result = risky_operation()
except RiskyError as e:
    logger.exception("操作失败")  # 记录完整堆栈
    raise  # 重新抛出，让调用者决定如何处理

# 最佳实践3：使用异常链保留根因
try:
    raw = read_file("config.json")
except FileNotFoundError as e:
    raise ConfigError("找不到配置文件") from e  # 保留原始异常

# 最佳实践4：在合适的层次处理异常
# - 底层（数据层）：只抛出，不处理
# - 中间层（业务层）：可以捕获并转换为业务异常
# - 顶层（API 层）：捕获所有异常，返回适当的错误响应

# 最佳实践5：不要用异常控制流程
# ❌ 用异常做正常流程控制（很慢，语义不清）
def find_index(lst, value):
    try:
        return lst.index(value)
    except ValueError:
        return -1

# ✅ 先检查再操作
def find_index(lst, value):
    return lst.index(value) if value in lst else -1

# 最佳实践6：清理资源用 finally 或 with
file = None
try:
    file = open("data.txt")
    process(file)
finally:
    if file:
        file.close()

# 更好：使用 with 语句
with open("data.txt") as file:
    process(file)
```

## 常见陷阱

### 1. 捕获基础异常类太宽泛

```python
# ❌ 捕获 BaseException 会连 KeyboardInterrupt 也捕获
try:
    main_loop()
except BaseException:  # 别这样！
    cleanup()

# ✅ 只在顶层捕获 Exception，单独处理 KeyboardInterrupt
try:
    main_loop()
except KeyboardInterrupt:
    print("用户中断")
    cleanup()
except Exception as e:
    logger.error(f"意外错误：{e}")
    cleanup()
```

### 2. 异常信息丢失

```python
# ❌ 重新抛出新异常时，原始异常信息丢失
try:
    connect_to_db()
except ConnectionError:
    raise RuntimeError("初始化失败")  # 原始 ConnectionError 信息丢失

# ✅ 使用 from 保留原始异常
try:
    connect_to_db()
except ConnectionError as e:
    raise RuntimeError("初始化失败") from e  # __cause__ 保留原始异常
```

## 小结

| 原则 | 说明 |
|------|------|
| 捕获具体异常 | 不用裸 `except` 或 `except Exception` 兜底 |
| 不要静默失败 | 至少要记录日志 |
| 用异常链 | `raise X from Y` 保留根因 |
| 合适层次处理 | 不要在每个函数都捕获所有异常 |
| 优先用 with | 资源管理用 `with`，不用 try/finally |
| EAFP 风格 | Python 鼓励先做后捕获，而不是先检查 |
| 自定义异常 | 业务逻辑用自定义异常，清晰语义化 |
