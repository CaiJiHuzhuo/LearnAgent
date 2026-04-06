# 类型提示（Type Hints）与 Pydantic 数据校验

> 为 Python 代码添加类型信息，使用 Pydantic 构建健壮的数据模型

## 概述

Python 的类型提示（Type Hints，PEP 484）让动态语言也能享受静态类型检查的好处。Pydantic 则是 AI Agent 开发中不可缺少的数据校验库，被 FastAPI、LangChain 等广泛使用。

| 概念 | Java | Python |
|------|------|--------|
| 静态类型 | 语言强制 | 类型提示（可选，工具检查） |
| 类型检查工具 | javac、IDE | mypy、pyright、ruff |
| 数据校验 | Bean Validation（JSR 380） | **Pydantic** |
| 数据类 | Lombok `@Data` / Java `record` | Pydantic `BaseModel` |

## 1. 基础类型提示

```python
# 变量类型提示（Python 3.6+）
name: str = "Alice"
age: int = 25
height: float = 1.75
is_active: bool = True
data: bytes = b"binary data"

# 函数参数和返回值类型提示
def greet(name: str, greeting: str = "Hello") -> str:
    return f"{greeting}, {name}!"

# 无返回值
def print_info(info: str) -> None:
    print(info)
```

## 2. 复杂类型提示

### Python 3.9+ 内置泛型（推荐）

```python
# Python 3.9+ 可以直接用小写 list、dict、tuple、set
def process_names(names: list[str]) -> dict[str, int]:
    return {name: len(name) for name in names}

def get_coords() -> tuple[float, float]:
    return (39.9, 116.4)

def unique_tags(tags: list[str]) -> set[str]:
    return set(tags)
```

### typing 模块（兼容旧版本）

```python
from typing import (
    List, Dict, Tuple, Set,  # 容器类型（Python 3.9+ 可用小写）
    Optional,     # 可选类型
    Union,        # 联合类型
    Any,          # 任意类型（慎用！）
    Callable,     # 函数类型
    TypeVar,      # 泛型变量
    Generic,      # 泛型类
    Literal,      # 字面量类型
    Final,        # 不可变常量
    ClassVar,     # 类变量
    TypedDict,    # 带类型的字典
    Protocol,     # 协议（鸭子类型的正式表达）
    Annotated,    # 附加元数据的类型
)

# Optional[T] 等价于 Union[T, None]
def find_user(user_id: int) -> Optional[str]:  # 可能返回 None
    users = {1: "Alice", 2: "Bob"}
    return users.get(user_id)

# 更现代的写法（Python 3.10+）
def find_user(user_id: int) -> str | None:  # 用 | 表示 Union
    return None

# Callable[[参数类型], 返回类型]
from typing import Callable

def apply_twice(func: Callable[[int], int], value: int) -> int:
    return func(func(value))

# TypedDict：带类型的字典
from typing import TypedDict

class UserInfo(TypedDict):
    name: str
    age: int
    email: str

def create_user(info: UserInfo) -> str:
    return f"{info['name']} ({info['age']})"

# Literal：限制取值范围
from typing import Literal

def set_direction(direction: Literal["north", "south", "east", "west"]) -> None:
    print(f"前往 {direction}")

# Final：常量
from typing import Final

MAX_RETRY: Final = 3
```

## 3. 高级类型技巧

```python
from typing import TypeVar, Generic

# TypeVar：泛型类型变量
T = TypeVar("T")

def first(lst: list[T]) -> T | None:
    """获取列表第一个元素"""
    return lst[0] if lst else None

# 带约束的 TypeVar
Number = TypeVar("Number", int, float)

def add(a: Number, b: Number) -> Number:
    return a + b

# 泛型类
class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []
    
    def push(self, item: T) -> None:
        self._items.append(item)
    
    def pop(self) -> T:
        return self._items.pop()
    
    def peek(self) -> T | None:
        return self._items[-1] if self._items else None
    
    def __len__(self) -> int:
        return len(self._items)


stack: Stack[int] = Stack()
stack.push(1)
stack.push(2)
print(stack.pop())  # 2
```

## 4. Pydantic：数据校验的利器

Pydantic 是 Python 生态中最流行的数据校验库，FastAPI、LangChain 都大量使用它。

### 安装

```bash
pip install pydantic pydantic[email]
# 或（推荐）
pip install "pydantic[email]"
```

### 基础模型

```python
from pydantic import BaseModel, Field, EmailStr, validator
from typing import Optional
from datetime import datetime

class User(BaseModel):
    # 必填字段
    name: str
    age: int
    email: EmailStr  # 自动验证邮件格式
    
    # 可选字段（有默认值）
    is_active: bool = True
    bio: Optional[str] = None
    
    # 带约束的字段（使用 Field）
    score: float = Field(default=0.0, ge=0.0, le=100.0)  # 0.0 <= score <= 100.0
    username: str = Field(min_length=3, max_length=20)
    
    created_at: datetime = Field(default_factory=datetime.now)


# 创建实例（自动校验）
user = User(
    name="Alice",
    age=25,
    email="alice@example.com",
    username="alice123",
)
print(user)
# name='Alice' age=25 email='alice@example.com' is_active=True bio=None score=0.0 username='alice123' ...

# 自动类型转换
user2 = User(
    name="Bob",
    age="30",          # 字符串自动转换为 int
    email="bob@example.com",
    username="bob",
    score="85.5",      # 字符串自动转换为 float
)
print(user2.age)       # 30（int 类型）
print(user2.score)     # 85.5（float 类型）
```

### 校验失败

```python
from pydantic import ValidationError

try:
    bad_user = User(
        name="Alice",
        age=-5,               # 负年龄（如果有约束会报错）
        email="not-an-email", # 无效邮箱
        username="ab",        # 太短
        score=200,            # 超出范围
    )
except ValidationError as e:
    print(e)
    # 3 validation errors for User
    # email
    #   value is not a valid email address ...
    # username  
    #   String should have at least 3 characters ...
    # score
    #   Input should be less than or equal to 100 ...
    
    # 获取详细错误信息
    for error in e.errors():
        print(f"字段 {error['loc']}: {error['msg']}")
```

### 自定义校验器

```python
from pydantic import BaseModel, field_validator, model_validator
from typing import Self

class Registration(BaseModel):
    username: str
    password: str
    confirm_password: str
    age: int
    
    @field_validator("username")
    @classmethod
    def username_alphanumeric(cls, v: str) -> str:
        """用户名只能包含字母和数字"""
        if not v.isalnum():
            raise ValueError("用户名只能包含字母和数字")
        return v.lower()  # 转换为小写
    
    @field_validator("age")
    @classmethod
    def age_must_be_adult(cls, v: int) -> int:
        """必须年满 18 岁"""
        if v < 18:
            raise ValueError(f"必须年满 18 岁，当前年龄：{v}")
        return v
    
    @model_validator(mode="after")
    def passwords_match(self) -> Self:
        """密码和确认密码必须一致"""
        if self.password != self.confirm_password:
            raise ValueError("密码和确认密码不一致")
        return self


# 成功
reg = Registration(
    username="Alice123",
    password="secret123",
    confirm_password="secret123",
    age=25,
)
print(reg.username)  # alice123（已转小写）

# 失败
try:
    bad = Registration(
        username="alice!",      # 包含特殊字符
        password="secret",
        confirm_password="other",  # 不匹配
        age=16,                # 未成年
    )
except ValidationError as e:
    print(e)
```

### 嵌套模型

```python
from pydantic import BaseModel
from typing import List

class Address(BaseModel):
    street: str
    city: str
    country: str = "CN"

class OrderItem(BaseModel):
    product_id: int
    quantity: int
    price: float

class Order(BaseModel):
    order_id: str
    customer_name: str
    shipping_address: Address  # 嵌套模型
    items: List[OrderItem]     # 模型列表
    
    @property
    def total(self) -> float:
        return sum(item.price * item.quantity for item in self.items)


# 创建嵌套模型
order = Order(
    order_id="ORD-001",
    customer_name="Alice",
    shipping_address={"street": "中关村大街1号", "city": "北京"},  # 可以用字典！
    items=[
        {"product_id": 1, "quantity": 2, "price": 99.9},
        {"product_id": 2, "quantity": 1, "price": 199.0},
    ]
)
print(order.total)  # 398.8
print(order.shipping_address.city)  # 北京
```

### 序列化与反序列化

```python
from pydantic import BaseModel
import json

class Config(BaseModel):
    host: str = "localhost"
    port: int = 8080
    debug: bool = False
    tags: list[str] = []

# 从字典创建（反序列化）
config = Config(host="0.0.0.0", port=5000)

# 转换为字典
config_dict = config.model_dump()
print(config_dict)  # {'host': '0.0.0.0', 'port': 5000, 'debug': False, 'tags': []}

# 转换为 JSON 字符串
config_json = config.model_dump_json()
print(config_json)  # {"host":"0.0.0.0","port":5000,"debug":false,"tags":[]}

# 从 JSON 字符串创建
config2 = Config.model_validate_json('{"host": "example.com", "port": 443}')
print(config2.host)  # example.com

# 从文件加载配置
with open("config.json") as f:
    config3 = Config.model_validate(json.load(f))
```

## 5. 类型检查工具：mypy

```bash
# 安装 mypy
pip install mypy

# 检查单个文件
mypy my_module.py

# 检查整个项目
mypy .

# 常用选项
mypy --strict my_module.py  # 最严格的检查
mypy --ignore-missing-imports my_module.py  # 忽略缺少类型信息的第三方库
```

```python
# 示例：mypy 能发现的错误
def add(a: int, b: int) -> int:
    return a + b

result = add("hello", "world")  # mypy: error: Argument 1 to "add" has incompatible type "str"; expected "int"
```

## 常见陷阱与注意事项

### 1. 类型提示不是运行时约束

```python
def greet(name: str) -> str:
    return f"Hello, {name}!"

# 运行时不会报错！类型提示只是提示
greet(42)    # 不报错，输出 "Hello, 42!"
greet(None)  # 不报错，输出 "Hello, None!"

# 只有用 mypy/pyright 检查时才会报错
```

### 2. 使用 `from __future__ import annotations` 避免前向引用问题

```python
# 在文件顶部加这行，让类型提示变成字符串（延迟求值）
from __future__ import annotations

class Tree:
    def __init__(self, value: int, children: list[Tree] | None = None):
        # 如果没有 from __future__ import annotations，Python 3.9 以下会报错
        # 因为 Tree 在定义时还不存在
        self.value = value
        self.children = children or []
```

### 3. Pydantic v1 vs v2 的差异

```python
# Pydantic v2（推荐，2023年发布，性能提升 5-50 倍）
from pydantic import BaseModel, field_validator

class User(BaseModel):
    name: str
    
    @field_validator("name")  # v2 使用 @field_validator
    @classmethod
    def validate_name(cls, v):
        return v.strip()
    
    # v2 使用 model_dump() 代替 v1 的 dict()
    # v2 使用 model_validate() 代替 v1 的 parse_obj()

# Pydantic v1（旧版，部分项目仍在使用）
# @validator 代替 @field_validator
# .dict() 代替 .model_dump()
# .parse_obj() 代替 .model_validate()
```

## 小结

**类型提示速查表**：

| 类型 | 示例 |
|------|------|
| 基础类型 | `int`, `str`, `float`, `bool`, `bytes` |
| 可选 | `str \| None` 或 `Optional[str]` |
| 联合 | `int \| str` 或 `Union[int, str]` |
| 列表 | `list[str]` |
| 字典 | `dict[str, int]` |
| 元组 | `tuple[int, str, float]` |
| 集合 | `set[str]` |
| 函数 | `Callable[[int, str], bool]` |
| 任意 | `Any`（慎用） |
| 字面量 | `Literal["a", "b", "c"]` |

**Pydantic 速查表**：

| 功能 | 代码 |
|------|------|
| 定义模型 | `class Model(BaseModel): field: type` |
| 带约束 | `field: float = Field(ge=0, le=100)` |
| 可选字段 | `field: str \| None = None` |
| 字段校验 | `@field_validator("field") @classmethod def validate(cls, v):` |
| 模型校验 | `@model_validator(mode="after") def validate(self):` |
| 转字典 | `model.model_dump()` |
| 转 JSON | `model.model_dump_json()` |
| 从字典创建 | `Model.model_validate(dict)` |
| 从 JSON 创建 | `Model.model_validate_json(json_str)` |
