# 第二周练习题：带类型提示和 Pydantic 模型的 REST 客户端

> 综合运用第二周知识：类型提示、Pydantic、装饰器、生成器、异常处理

## 练习题总览

| 题号 | 题目 | 难度 | 考察知识点 |
|------|------|------|-----------|
| 1 | 类型注解改造 | ⭐ | 类型提示 |
| 2 | 缓存装饰器 | ⭐⭐ | 装饰器 |
| 3 | Pydantic 数据模型 | ⭐⭐ | Pydantic |
| 4 | 数据处理生成器 | ⭐⭐⭐ | 生成器、迭代器 |
| 5 | 🎯 综合实战：GitHub API 客户端 | ⭐⭐⭐⭐ | 综合所有知识点 |

---

## 练习一：类型注解改造

### 题目描述

将以下无类型注解的代码改造为带完整类型注解的版本，要求：
1. 所有函数参数和返回值都有类型注解
2. 所有变量都有适当的类型注解（关键变量）
3. 使用 `Optional`、`Union`、`List`、`Dict` 等类型

### 原始代码（无类型注解）

```python
def process_students(students):
    result = {}
    for student in students:
        name = student.get("name")
        scores = student.get("scores", [])
        if not scores:
            average = 0
        else:
            average = sum(scores) / len(scores)
        result[name] = {
            "average": average,
            "grade": get_grade(average),
            "passed": average >= 60
        }
    return result

def get_grade(score):
    if score >= 90:
        return "A"
    elif score >= 80:
        return "B"
    elif score >= 70:
        return "C"
    elif score >= 60:
        return "D"
    else:
        return "F"

def find_top_students(processed, n):
    sorted_students = sorted(
        processed.items(),
        key=lambda x: x[1]["average"],
        reverse=True
    )
    return [name for name, _ in sorted_students[:n]]
```

### 期望改造后的效果

```python
# 调用时 IDE 应该有完整的类型提示
students = [
    {"name": "Alice", "scores": [85, 90, 78]},
    {"name": "Bob", "scores": [60, 55, 70]},
]
result = process_students(students)
# result 的类型应该是 dict[str, dict[str, Any]]
```

### 参考解答

```python
from typing import TypedDict, Optional

class StudentInput(TypedDict):
    name: str
    scores: list[float]

class StudentResult(TypedDict):
    average: float
    grade: str
    passed: bool

def get_grade(score: float) -> str:
    if score >= 90:
        return "A"
    elif score >= 80:
        return "B"
    elif score >= 70:
        return "C"
    elif score >= 60:
        return "D"
    else:
        return "F"

def process_students(
    students: list[StudentInput]
) -> dict[str, StudentResult]:
    result: dict[str, StudentResult] = {}
    for student in students:
        name: str = student.get("name", "unknown")
        scores: list[float] = student.get("scores", [])
        average: float = sum(scores) / len(scores) if scores else 0.0
        result[name] = StudentResult(
            average=average,
            grade=get_grade(average),
            passed=average >= 60
        )
    return result

def find_top_students(
    processed: dict[str, StudentResult],
    n: int
) -> list[str]:
    sorted_students = sorted(
        processed.items(),
        key=lambda x: x[1]["average"],
        reverse=True
    )
    return [name for name, _ in sorted_students[:n]]
```

---

## 练习二：缓存装饰器

### 题目描述

实现一个自定义的 LRU（最近最少使用）缓存装饰器 `@lru_cache`，要求：
1. 支持 `maxsize` 参数控制最大缓存数量
2. 支持查看缓存信息（命中次数、未命中次数）
3. 支持清除缓存
4. 使用类型提示

### 输入输出示例

```python
@lru_cache(maxsize=3)
def expensive_compute(n: int) -> int:
    """模拟耗时计算"""
    import time
    time.sleep(0.1)
    return n ** 2

expensive_compute(5)   # 未命中缓存，计算，耗时 0.1s
expensive_compute(5)   # 命中缓存，立即返回
expensive_compute(10)  # 未命中
expensive_compute(15)  # 未命中
expensive_compute(20)  # 未命中，缓存满了，淘汰最久未使用的

print(expensive_compute.cache_info())
# CacheInfo(hits=1, misses=4, maxsize=3, currsize=3)

expensive_compute.cache_clear()
print(expensive_compute.cache_info())
# CacheInfo(hits=0, misses=0, maxsize=3, currsize=0)
```

### 提示

- 使用 `collections.OrderedDict` 实现 LRU 逻辑
- 或者使用 `functools.lru_cache`（标准库已有实现，但本题目的是自己实现）

### 参考解答

```python
import functools
from collections import OrderedDict
from dataclasses import dataclass
from typing import Callable, TypeVar, Any

F = TypeVar("F", bound=Callable[..., Any])

@dataclass
class CacheInfo:
    hits: int
    misses: int
    maxsize: int
    currsize: int
    
    def __repr__(self) -> str:
        return (f"CacheInfo(hits={self.hits}, misses={self.misses}, "
                f"maxsize={self.maxsize}, currsize={self.currsize})")


def lru_cache(maxsize: int = 128) -> Callable[[F], F]:
    """自定义 LRU 缓存装饰器"""
    def decorator(func: F) -> F:
        cache: OrderedDict[tuple, Any] = OrderedDict()
        hits = 0
        misses = 0
        
        @functools.wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            nonlocal hits, misses
            # 创建缓存键（args + sorted kwargs）
            key = args + tuple(sorted(kwargs.items()))
            
            if key in cache:
                hits += 1
                cache.move_to_end(key)  # 移到末尾（最近使用）
                return cache[key]
            
            misses += 1
            result = func(*args, **kwargs)
            cache[key] = result
            
            if len(cache) > maxsize:
                cache.popitem(last=False)  # 删除最久未使用的（头部）
            
            return result
        
        def cache_info() -> CacheInfo:
            return CacheInfo(hits=hits, misses=misses, 
                           maxsize=maxsize, currsize=len(cache))
        
        def cache_clear() -> None:
            nonlocal hits, misses
            cache.clear()
            hits = 0
            misses = 0
        
        wrapper.cache_info = cache_info  # type: ignore
        wrapper.cache_clear = cache_clear  # type: ignore
        
        return wrapper  # type: ignore
    
    return decorator


# 测试
import time

@lru_cache(maxsize=3)
def expensive_compute(n: int) -> int:
    time.sleep(0.01)
    return n ** 2

expensive_compute(5)
expensive_compute(5)   # 命中
expensive_compute(10)
expensive_compute(15)
expensive_compute(20)  # 淘汰 5

print(expensive_compute.cache_info())  # hits=1, misses=4
expensive_compute.cache_clear()
print(expensive_compute.cache_info())  # hits=0, misses=0
```

---

## 练习三：Pydantic 数据模型

### 题目描述

使用 Pydantic 为一个博客系统定义数据模型：
1. `Author`：作者（id、名称、邮箱、角色）
2. `Tag`：标签（id、名称）
3. `Post`：文章（id、标题、内容、作者、标签列表、状态、创建时间）

要求：
- 邮箱格式校验
- 标题不能为空，长度 5-200 字符
- 内容不能为空，最少 10 字符
- 角色只能是 "admin"、"editor"、"viewer" 之一
- 文章状态只能是 "draft"、"published"、"archived"
- 支持从 JSON 创建和导出到 JSON

### 参考解答

```python
from pydantic import BaseModel, EmailStr, Field, field_validator
from typing import Literal, Optional
from datetime import datetime

class Author(BaseModel):
    id: int
    name: str = Field(min_length=1, max_length=50)
    email: EmailStr
    role: Literal["admin", "editor", "viewer"] = "viewer"
    
    def can_publish(self) -> bool:
        return self.role in ("admin", "editor")


class Tag(BaseModel):
    id: int
    name: str = Field(min_length=1, max_length=30)
    
    @field_validator("name")
    @classmethod
    def name_lowercase(cls, v: str) -> str:
        return v.lower().strip()


class Post(BaseModel):
    id: int
    title: str = Field(min_length=5, max_length=200)
    content: str = Field(min_length=10)
    author: Author
    tags: list[Tag] = []
    status: Literal["draft", "published", "archived"] = "draft"
    created_at: datetime = Field(default_factory=datetime.now)
    updated_at: Optional[datetime] = None
    
    @field_validator("title")
    @classmethod
    def title_stripped(cls, v: str) -> str:
        return v.strip()
    
    def publish(self) -> None:
        if self.status != "draft":
            raise ValueError(f"只有草稿可以发布，当前状态：{self.status}")
        self.status = "published"
        self.updated_at = datetime.now()
    
    def word_count(self) -> int:
        return len(self.content.split())


# 测试
author = Author(id=1, name="Alice", email="alice@example.com", role="editor")

tags = [Tag(id=1, name="Python"), Tag(id=2, name="FastAPI")]

post = Post(
    id=1,
    title="Python 快速入门",
    content="Python 是一种优雅的编程语言，适合初学者和专家。" * 3,
    author=author,
    tags=tags,
)

print(post.model_dump_json(indent=2))
print(f"字数：{post.word_count()}")

# 从 JSON 创建
json_str = post.model_dump_json()
post2 = Post.model_validate_json(json_str)
print(post2.title)  # Python 快速入门
```

---

## 练习四：数据处理生成器

### 题目描述

实现一个数据处理管道，使用生成器处理 CSV 数据：
1. `read_csv(filepath)`: 生成器，逐行读取并解析 CSV
2. `filter_rows(rows, condition)`: 生成器，过滤满足条件的行
3. `transform_row(rows, func)`: 生成器，对每行数据进行转换
4. `batch(rows, size)`: 生成器，将数据分批处理

### 输入输出示例

```python
# 测试数据（模拟 CSV 内容）
CSV_DATA = """name,age,salary,department
Alice,25,8000,Engineering
Bob,30,12000,Engineering
Carol,28,9500,Marketing
Dave,35,15000,Management
Eve,22,6000,Marketing
Frank,40,20000,Management
"""

# 使用管道处理
pipeline = batch(
    transform_row(
        filter_rows(
            parse_rows(CSV_DATA.strip().split("\n")),
            condition=lambda r: int(r["age"]) >= 25
        ),
        func=lambda r: {**r, "salary": int(r["salary"]), "age": int(r["age"])}
    ),
    size=2
)

for batch_data in pipeline:
    print(batch_data)
# [{'name': 'Alice', 'age': 25, ...}, {'name': 'Bob', 'age': 30, ...}]
# [{'name': 'Carol', 'age': 28, ...}, {'name': 'Dave', 'age': 35, ...}]
# [{'name': 'Frank', 'age': 40, ...}]
```

### 参考解答

```python
from typing import Callable, Iterator, TypeVar

T = TypeVar("T")

def parse_rows(lines: list[str]) -> Iterator[dict[str, str]]:
    """解析 CSV 行，生成字典"""
    header = None
    for line in lines:
        if not line.strip():
            continue
        fields = line.strip().split(",")
        if header is None:
            header = fields
            continue
        yield dict(zip(header, fields))


def filter_rows(
    rows: Iterator[dict],
    condition: Callable[[dict], bool]
) -> Iterator[dict]:
    """过滤满足条件的行"""
    for row in rows:
        if condition(row):
            yield row


def transform_row(
    rows: Iterator[dict],
    func: Callable[[dict], dict]
) -> Iterator[dict]:
    """对每行数据进行转换"""
    for row in rows:
        yield func(row)


def batch(
    items: Iterator[T],
    size: int
) -> Iterator[list[T]]:
    """将数据分批，每批 size 个"""
    current_batch: list[T] = []
    for item in items:
        current_batch.append(item)
        if len(current_batch) == size:
            yield current_batch
            current_batch = []
    if current_batch:  # 不要忘记最后一批
        yield current_batch


# 测试
CSV_DATA = """name,age,salary,department
Alice,25,8000,Engineering
Bob,30,12000,Engineering
Carol,28,9500,Marketing
Dave,35,15000,Management
Eve,22,6000,Marketing
Frank,40,20000,Management"""

pipeline = batch(
    transform_row(
        filter_rows(
            parse_rows(CSV_DATA.strip().split("\n")),
            condition=lambda r: int(r["age"]) >= 25
        ),
        func=lambda r: {**r, "salary": int(r["salary"]), "age": int(r["age"])}
    ),
    size=2
)

for batch_data in pipeline:
    print(batch_data)
```

---

## 练习五：🎯 综合实战 - GitHub API 客户端

### 题目描述

使用 `requests`（或 `httpx`）构建一个 GitHub API 客户端，综合运用本周所有知识：

**功能需求：**
1. 使用 Pydantic 定义 GitHub 数据模型（Repo、Issue、User）
2. 使用装饰器实现：
   - `@retry(max_retries=3)`：失败重试
   - `@rate_limit(calls_per_minute=60)`：限流
3. 用生成器实现分页获取（GitHub API 有分页）
4. 完整的类型提示
5. 自定义异常类

### 数据模型

```python
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class GitHubUser(BaseModel):
    login: str
    id: int
    avatar_url: str
    html_url: str

class GitHubRepo(BaseModel):
    id: int
    name: str
    full_name: str
    description: Optional[str] = None
    html_url: str
    stargazers_count: int
    language: Optional[str] = None
    owner: GitHubUser

class GitHubIssue(BaseModel):
    id: int
    number: int
    title: str
    state: str  # "open" | "closed"
    body: Optional[str] = None
    user: GitHubUser
    created_at: datetime
    updated_at: datetime
```

### 参考解答

```python
from __future__ import annotations

import time
import functools
import logging
from typing import Iterator, Optional, Any
from datetime import datetime

import requests
from pydantic import BaseModel

logger = logging.getLogger(__name__)


# === 自定义异常 ===

class GitHubAPIError(Exception):
    """GitHub API 基础异常"""
    def __init__(self, message: str, status_code: Optional[int] = None):
        super().__init__(message)
        self.status_code = status_code

class RateLimitError(GitHubAPIError):
    """GitHub API 限流异常"""
    pass

class NotFoundError(GitHubAPIError):
    """资源未找到"""
    pass


# === Pydantic 数据模型 ===

class GitHubUser(BaseModel):
    login: str
    id: int
    avatar_url: str
    html_url: str

class GitHubRepo(BaseModel):
    id: int
    name: str
    full_name: str
    description: Optional[str] = None
    html_url: str
    stargazers_count: int
    language: Optional[str] = None
    owner: GitHubUser

class GitHubIssue(BaseModel):
    id: int
    number: int
    title: str
    state: str
    body: Optional[str] = None
    user: GitHubUser
    created_at: datetime
    updated_at: datetime


# === 装饰器 ===

def retry(max_retries: int = 3, delay: float = 1.0):
    """重试装饰器，遇到网络错误时自动重试"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except (requests.ConnectionError, requests.Timeout) as e:
                    if attempt == max_retries:
                        raise GitHubAPIError(f"重试 {max_retries} 次后仍失败") from e
                    wait = delay * attempt
                    logger.warning(f"第 {attempt} 次失败，{wait}s 后重试：{e}")
                    time.sleep(wait)
        return wrapper
    return decorator


# === GitHub API 客户端 ===

class GitHubClient:
    BASE_URL = "https://api.github.com"
    
    def __init__(self, token: Optional[str] = None):
        self.session = requests.Session()
        self.session.headers.update({
            "Accept": "application/vnd.github.v3+json",
            "User-Agent": "Python-Learning-Client/1.0",
        })
        if token:
            self.session.headers["Authorization"] = f"token {token}"
    
    @retry(max_retries=3)
    def _get(self, endpoint: str, params: Optional[dict] = None) -> Any:
        """发送 GET 请求"""
        url = f"{self.BASE_URL}{endpoint}"
        response = self.session.get(url, params=params, timeout=10)
        
        if response.status_code == 404:
            raise NotFoundError(f"资源不存在：{endpoint}", status_code=404)
        elif response.status_code == 403:
            raise RateLimitError("GitHub API 限流", status_code=403)
        elif response.status_code >= 400:
            raise GitHubAPIError(
                f"API 错误 {response.status_code}：{response.text}",
                status_code=response.status_code
            )
        
        return response.json()
    
    def get_repo(self, owner: str, repo: str) -> GitHubRepo:
        """获取仓库信息"""
        data = self._get(f"/repos/{owner}/{repo}")
        return GitHubRepo.model_validate(data)
    
    def iter_issues(
        self,
        owner: str,
        repo: str,
        state: str = "open",
        per_page: int = 30,
    ) -> Iterator[GitHubIssue]:
        """分页获取 Issues（生成器）"""
        page = 1
        while True:
            data = self._get(
                f"/repos/{owner}/{repo}/issues",
                params={"state": state, "page": page, "per_page": per_page}
            )
            if not data:  # 没有更多数据
                break
            for issue_data in data:
                if "pull_request" not in issue_data:  # 过滤 PR
                    yield GitHubIssue.model_validate(issue_data)
            if len(data) < per_page:  # 最后一页
                break
            page += 1
    
    def get_user_repos(self, username: str) -> list[GitHubRepo]:
        """获取用户的所有公开仓库"""
        data = self._get(f"/users/{username}/repos")
        return [GitHubRepo.model_validate(r) for r in data]


# 使用示例（需要 GitHub Token 来提高限流限制）
if __name__ == "__main__":
    client = GitHubClient()  # 不带 token 也可以用，但有限流
    
    try:
        # 获取仓库信息
        repo = client.get_repo("python", "cpython")
        print(f"仓库：{repo.full_name}")
        print(f"Stars：{repo.stargazers_count}")
        print(f"语言：{repo.language}")
        
        # 分页获取 Issues（使用生成器，不一次性加载全部）
        print("\n最新 5 个 Issues：")
        for i, issue in enumerate(client.iter_issues("python", "cpython")):
            if i >= 5:
                break
            print(f"  #{issue.number}: {issue.title}")
    
    except NotFoundError as e:
        print(f"资源不存在：{e}")
    except RateLimitError as e:
        print(f"API 限流，请稍后再试")
    except GitHubAPIError as e:
        print(f"API 错误：{e}")
```

---

## 本周总结

通过这 5 道练习，你应该掌握了：

- ✅ 为现有代码添加完整类型注解
- ✅ 实现带参数的装饰器（缓存、重试）
- ✅ 使用 Pydantic 定义和校验复杂数据模型
- ✅ 用生成器构建数据处理管道
- ✅ 综合运用所有技术构建实际的 API 客户端

**自测检验**：如果你能在 4 小时内独立完成练习五，说明第二周内容已经掌握，可以进入第三周！
