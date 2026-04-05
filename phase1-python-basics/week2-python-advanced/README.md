# 第二周：Python 进阶特性

> 掌握 Python 高级特性，编写更优雅、更高效的代码

## 🎯 本周学习目标

1. 理解迭代器与生成器，用 `yield` 实现高效的数据流处理
2. 掌握装饰器，能编写自定义装饰器（计时、日志、重试等）
3. 熟练使用上下文管理器（`with` 语句）管理资源
4. 为代码添加类型提示，使用 Pydantic 进行数据校验
5. 掌握 Python 异常处理最佳实践
6. 熟悉 pip、poetry、uv 等包管理工具

## 📅 每日建议安排

| 天 | 上午（1-2h） | 下午（1-2h） | 晚上（1h） |
|----|------------|------------|----------|
| 周一 | [迭代器与生成器](01-iterators-generators.md) | 练习 yield | 思考生成器的应用场景 |
| 周二 | [装饰器基础](02-decorators.md) | 编写自定义装饰器 | 探索 functools |
| 周三 | [上下文管理器](03-context-managers.md) | [异常处理](05-exception-handling.md) | 练习资源管理 |
| 周四 | [类型提示](04-type-hints-pydantic.md) | Pydantic 模型 | 给已有代码加类型 |
| 周五 | [包管理工具](06-package-management.md) | 配置 poetry 项目 | 开始练习题 |
| 周末 | [完成练习题](exercises/week2-exercises.md) | 复习本周内容 | 预习第三周 |

## 📖 知识点列表

| 序号 | 知识点 | 文件 | 难度 | 预计时间 |
|------|--------|------|------|---------|
| 01 | 迭代器与生成器（yield） | [01-iterators-generators.md](01-iterators-generators.md) | ⭐⭐⭐ | 2h |
| 02 | 装饰器（decorator） | [02-decorators.md](02-decorators.md) | ⭐⭐⭐ | 2h |
| 03 | 上下文管理器（with 语句） | [03-context-managers.md](03-context-managers.md) | ⭐⭐ | 1.5h |
| 04 | 类型提示 + Pydantic | [04-type-hints-pydantic.md](04-type-hints-pydantic.md) | ⭐⭐⭐ | 3h |
| 05 | 异常处理最佳实践 | [05-exception-handling.md](05-exception-handling.md) | ⭐⭐ | 1.5h |
| 06 | 包管理工具（pip/poetry/uv） | [06-package-management.md](06-package-management.md) | ⭐⭐ | 2h |
| - | 第二周练习题 | [exercises/week2-exercises.md](exercises/week2-exercises.md) | ⭐⭐⭐ | 4h |

## 💡 本周重点

### 生成器：Python 的"懒加载"

```python
# 处理大文件时，不要一次性全部加载
def read_large_file(filepath):
    with open(filepath) as f:
        for line in f:          # 文件对象本身就是迭代器
            yield line.strip()  # 逐行生成，内存效率极高

# 相比：一次性读入内存（大文件会 OOM）
lines = open("huge_file.txt").readlines()  # ❌ 危险！
```

### 装饰器：Python 的"AOP"

```python
import functools
import time

def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} 耗时：{time.time()-start:.3f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "完成"

slow_function()  # slow_function 耗时：1.001s
```

### Pydantic：Python 的"Bean Validation"

```python
from pydantic import BaseModel, EmailStr, validator

class User(BaseModel):
    name: str
    age: int
    email: EmailStr
    
    @validator("age")
    def age_must_be_positive(cls, v):
        if v <= 0:
            raise ValueError("年龄必须大于 0")
        return v

user = User(name="Alice", age=25, email="alice@example.com")
# User(name='Alice', age=25, email='alice@example.com')

# 自动类型转换
user2 = User(name="Bob", age="30", email="bob@example.com")
# age 被自动转换为 int

# 校验失败
user3 = User(name="Carol", age=-1, email="not-email")
# ValidationError: 2 validation errors
```

## ✅ 本周检查清单

- [ ] 理解迭代器协议（`__iter__` 和 `__next__`）
- [ ] 能用 `yield` 编写生成器函数
- [ ] 理解装饰器的工作原理（函数包装）
- [ ] 能编写带参数的装饰器
- [ ] 能用 `contextlib.contextmanager` 创建上下文管理器
- [ ] 能为函数、类添加完整的类型提示
- [ ] 能用 Pydantic 定义数据模型并进行校验
- [ ] 掌握 Python 异常处理的 EAFP 风格
- [ ] 能用 poetry 或 uv 创建和管理 Python 项目
- [ ] 完成本周所有练习题
