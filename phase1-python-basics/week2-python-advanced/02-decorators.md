# 装饰器（Decorator）

> Python 装饰器：优雅地修改函数行为，类比 Java 的 AOP 与注解处理器

## 概述

装饰器是 Python 中实现"横切关注点"（Cross-Cutting Concerns）的核心机制，类比 Java 中的 AOP（面向切面编程）或注解处理器（Annotation Processor）。

常见用途：
- 性能计时（`@timer`）
- 日志记录（`@log`）
- 权限验证（`@login_required`）
- 缓存（`@cache`）
- 重试机制（`@retry`）
- 参数校验

## 1. 理解装饰器的本质

装饰器的本质是**高阶函数**（接受函数并返回函数的函数）：

```python
# 不使用装饰器语法的写法
def greet(name):
    return f"Hello, {name}!"

def add_exclamation(func):
    """包装函数，在返回值后加感叹号"""
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        return result + "!!!"
    return wrapper

# 手动应用"装饰器"
greet = add_exclamation(greet)
print(greet("Alice"))  # Hello, Alice!!!!

# 使用装饰器语法（等价，更简洁）
@add_exclamation
def greet(name):
    return f"Hello, {name}!"

print(greet("Alice"))  # Hello, Alice!!!!
```

```java
// Java 中类似的概念：代理模式 / AOP
// 需要用 Spring AOP 或手写代理类来实现
@Aspect
public class LoggingAspect {
    @Around("@annotation(Timed)")
    public Object timeMethod(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        System.out.println("耗时：" + (System.currentTimeMillis() - start) + "ms");
        return result;
    }
}
```

## 2. 编写正确的装饰器

使用 `functools.wraps` 保留被装饰函数的元信息：

```python
import functools
import time

def timer(func):
    """计时装饰器"""
    @functools.wraps(func)  # 保留 func 的 __name__、__doc__ 等信息
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"[{func.__name__}] 耗时：{elapsed:.4f}s")
        return result
    return wrapper


@timer
def slow_function(n: int) -> int:
    """计算 1 到 n 的和"""
    time.sleep(0.1)  # 模拟耗时操作
    return sum(range(n + 1))


result = slow_function(100)
print(result)                  # [slow_function] 耗时：0.1001s \n 5050
print(slow_function.__name__)  # slow_function（不是 wrapper！因为有 @wraps）
print(slow_function.__doc__)   # 计算 1 到 n 的和


def logger(func):
    """日志装饰器：记录函数调用和返回值"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        args_repr = [repr(a) for a in args]
        kwargs_repr = [f"{k}={v!r}" for k, v in kwargs.items()]
        signature = ", ".join(args_repr + kwargs_repr)
        print(f"调用 {func.__name__}({signature})")
        result = func(*args, **kwargs)
        print(f"{func.__name__} 返回 {result!r}")
        return result
    return wrapper


@logger
def add(a: int, b: int) -> int:
    return a + b

add(3, 4)
# 调用 add(3, 4)
# add 返回 7
```

## 3. 带参数的装饰器

需要在装饰器外再包一层函数：

```python
import functools

def retry(max_attempts: int = 3, delay: float = 1.0):
    """带参数的重试装饰器"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    print(f"第 {attempt} 次尝试失败：{e}")
                    if attempt < max_attempts:
                        time.sleep(delay)
            raise last_exception
        return wrapper
    return decorator


# 使用带参数的装饰器
@retry(max_attempts=3, delay=0.5)
def unstable_api_call():
    import random
    if random.random() < 0.7:  # 70% 概率失败
        raise ConnectionError("网络超时")
    return "成功！"


# 等价于：
# unstable_api_call = retry(max_attempts=3, delay=0.5)(unstable_api_call)

try:
    result = unstable_api_call()
    print(result)
except ConnectionError:
    print("所有重试均失败")
```

## 4. 叠加多个装饰器

```python
import functools

def bold(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return f"**{func(*args, **kwargs)}**"
    return wrapper

def italic(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return f"_{func(*args, **kwargs)}_"
    return wrapper


@bold
@italic
def greet(name):
    return f"Hello, {name}"

# 执行顺序：先 italic，再 bold（从下到上应用）
# 等价于：bold(italic(greet))
print(greet("Alice"))  # **_Hello, Alice_**
```

## 5. 类装饰器

```python
import functools
import time

class Timer:
    """用类实现的计时装饰器"""
    
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func = func
        self.call_count = 0
        self.total_time = 0
    
    def __call__(self, *args, **kwargs):
        start = time.perf_counter()
        result = self.func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        self.call_count += 1
        self.total_time += elapsed
        print(f"[{self.func.__name__}] 第 {self.call_count} 次调用，耗时 {elapsed:.4f}s")
        return result
    
    def stats(self):
        if self.call_count == 0:
            print("还未调用")
            return
        avg = self.total_time / self.call_count
        print(f"调用 {self.call_count} 次，总耗时 {self.total_time:.4f}s，平均 {avg:.4f}s")


@Timer
def compute(n):
    return sum(range(n))


compute(10000)
compute(100000)
compute.stats()  # 类装饰器可以有额外方法！
```

## 6. 使用 functools 内置装饰器

```python
import functools
import time

# @functools.cache（Python 3.9+）：无限缓存
# @functools.lru_cache：有限缓存（LRU 策略）

@functools.lru_cache(maxsize=128)
def fibonacci(n: int) -> int:
    """斐波那契数列（带缓存）"""
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

print(fibonacci(50))  # 极快，因为有缓存

# 查看缓存信息
print(fibonacci.cache_info())
# CacheInfo(hits=48, misses=51, maxsize=128, currsize=51)

# 清除缓存
fibonacci.cache_clear()


# @functools.singledispatch：单分派（类比 Java 的方法重载）
@functools.singledispatch
def process(arg):
    raise NotImplementedError(f"不支持的类型：{type(arg)}")

@process.register(int)
def _(arg):
    return f"处理整数：{arg * 2}"

@process.register(str)
def _(arg):
    return f"处理字符串：{arg.upper()}"

@process.register(list)
def _(arg):
    return f"处理列表：{len(arg)} 个元素"

print(process(42))        # 处理整数：84
print(process("hello"))   # 处理字符串：HELLO
print(process([1, 2, 3])) # 处理列表：3 个元素
```

## 7. 实际项目中的装饰器示例

### 权限验证装饰器

```python
from functools import wraps

# 模拟用户上下文
current_user = {"name": "Alice", "roles": ["user"]}

def require_role(role: str):
    """权限验证装饰器"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            if role not in current_user.get("roles", []):
                raise PermissionError(f"需要 '{role}' 权限，当前用户没有该权限")
            return func(*args, **kwargs)
        return wrapper
    return decorator


@require_role("admin")
def delete_user(user_id: int):
    print(f"删除用户 {user_id}")

try:
    delete_user(123)  # 当前用户没有 admin 权限
except PermissionError as e:
    print(e)  # 需要 'admin' 权限，当前用户没有该权限


### API 调用频率限制装饰器

import time
from collections import defaultdict

def rate_limit(calls_per_second: float = 1.0):
    """限制函数调用频率"""
    min_interval = 1.0 / calls_per_second
    last_called = {}
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            now = time.time()
            key = func.__name__
            if key in last_called:
                elapsed = now - last_called[key]
                if elapsed < min_interval:
                    wait = min_interval - elapsed
                    time.sleep(wait)
            last_called[key] = time.time()
            return func(*args, **kwargs)
        return wrapper
    return decorator


@rate_limit(calls_per_second=2.0)  # 每秒最多 2 次
def call_api(endpoint: str) -> dict:
    print(f"调用 {endpoint}")
    return {"status": "ok"}
```

## 常见陷阱与注意事项

### 1. 忘记 `@functools.wraps`

```python
# ❌ 没有 @wraps，函数元信息丢失
def bad_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@bad_decorator
def my_func():
    """这是文档字符串"""
    pass

print(my_func.__name__)  # wrapper（而不是 my_func！）
print(my_func.__doc__)   # None

# ✅ 使用 @wraps 保留元信息
def good_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

### 2. 装饰器应用顺序

```python
@decorator_a  # 最后应用（在外层）
@decorator_b  # 先应用（在内层）
def func():
    pass

# 等价于：func = decorator_a(decorator_b(func))
# 调用时：decorator_a 的逻辑先执行
```

### 3. 装饰了类方法时的 `self`

```python
class MyClass:
    @timer  # ❌ timer 没有处理 self，可能导致问题
    def method(self):
        pass

# ✅ 使用 *args, **kwargs 自动处理 self
def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):  # self 会作为第一个 arg 传入
        ...
        return func(*args, **kwargs)
    return wrapper
```

## 小结

| 装饰器类型 | 语法 | 适用场景 |
|-----------|------|---------|
| 简单装饰器 | `@decorator` | 无参数的横切关注点 |
| 带参数装饰器 | `@decorator(args)` | 可配置的横切关注点 |
| 类装饰器 | `@ClassName` | 需要保持状态的装饰器 |
| 叠加装饰器 | 多个 `@` | 组合多个功能 |

**与 Java 对比**：
| Java | Python |
|------|--------|
| Spring AOP `@Aspect` | 自定义装饰器 |
| `@Cacheable` | `@functools.lru_cache` |
| `@Transactional` | `@transaction`（自定义或使用 ORM） |
| `@Scheduled` | `@scheduler`（第三方库） |
| 方法重载（Overloading） | `@functools.singledispatch` |
