# FastAPI 框架入门

> 用 FastAPI 快速构建高性能 REST API，完整支持异步和类型提示

## 概述

FastAPI 是目前最流行的 Python Web 框架之一，特别适合 AI Agent 后端开发。

| 特性 | FastAPI | Java Spring Boot |
|------|---------|----------------|
| 性能 | 基于 Starlette/uvicorn，接近 Go | 优秀，但有 JVM 开销 |
| 类型安全 | 通过 Pydantic 和 Type Hints | 通过 Java 静态类型 |
| 自动文档 | 内置 Swagger UI 和 ReDoc ✅ | 需要 Springfox/SpringDoc |
| 异步支持 | 原生 async/await ✅ | WebFlux（响应式） |
| 依赖注入 | 内置 ✅ | Spring 容器 |
| 数据校验 | Pydantic 自动校验 ✅ | Bean Validation（JSR 380） |
| 学习曲线 | 较低 | 较高（Spring 生态庞大） |

## 1. 快速开始

### 安装

```bash
pip install "fastapi[all]"
# 包含：fastapi + uvicorn（ASGI 服务器）+ pydantic + python-multipart 等
```

### 最小示例

```python
# main.py
from fastapi import FastAPI

app = FastAPI(
    title="我的 API",
    description="AI Agent 学习项目 API",
    version="0.1.0",
)

@app.get("/")
async def root():
    return {"message": "Hello, FastAPI!"}

@app.get("/hello/{name}")
async def say_hello(name: str):
    return {"message": f"你好，{name}！"}
```

```bash
# 启动服务器
uvicorn main:app --reload  # --reload 开启热重载（开发环境）
# 访问 http://localhost:8000
# 查看文档 http://localhost:8000/docs（Swagger UI）
# 查看文档 http://localhost:8000/redoc（ReDoc）
```

## 2. 路由与 HTTP 方法

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

# 模拟数据库
fake_db: dict[int, dict] = {
    1: {"id": 1, "name": "Alice", "email": "alice@example.com"},
    2: {"id": 2, "name": "Bob", "email": "bob@example.com"},
}
next_id = 3


# Pydantic 模型（用于请求体和响应体）
class UserCreate(BaseModel):
    name: str
    email: str

class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[str] = None

class UserResponse(BaseModel):
    id: int
    name: str
    email: str


# GET /users - 获取所有用户（列表）
@app.get("/users", response_model=list[UserResponse])
async def get_users():
    return list(fake_db.values())


# GET /users/{user_id} - 获取单个用户
@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int):  # 路径参数，自动类型转换
    if user_id not in fake_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"用户 {user_id} 不存在",
        )
    return fake_db[user_id]


# POST /users - 创建用户
@app.post("/users", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(user: UserCreate):  # 请求体自动解析和校验
    global next_id
    new_user = {"id": next_id, **user.model_dump()}
    fake_db[next_id] = new_user
    next_id += 1
    return new_user


# PUT /users/{user_id} - 全量更新
@app.put("/users/{user_id}", response_model=UserResponse)
async def update_user(user_id: int, user: UserCreate):
    if user_id not in fake_db:
        raise HTTPException(status_code=404, detail=f"用户 {user_id} 不存在")
    fake_db[user_id] = {"id": user_id, **user.model_dump()}
    return fake_db[user_id]


# PATCH /users/{user_id} - 部分更新
@app.patch("/users/{user_id}", response_model=UserResponse)
async def partial_update_user(user_id: int, user: UserUpdate):
    if user_id not in fake_db:
        raise HTTPException(status_code=404, detail=f"用户 {user_id} 不存在")
    
    existing = fake_db[user_id]
    update_data = user.model_dump(exclude_unset=True)  # 只取设置了的字段
    existing.update(update_data)
    return existing


# DELETE /users/{user_id} - 删除用户
@app.delete("/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: int):
    if user_id not in fake_db:
        raise HTTPException(status_code=404, detail=f"用户 {user_id} 不存在")
    del fake_db[user_id]
    # 204 不返回内容
```

## 3. 请求参数

```python
from fastapi import FastAPI, Query, Path, Body
from typing import Optional

app = FastAPI()


# 查询参数（URL 中的 ?key=value）
@app.get("/items")
async def get_items(
    skip: int = 0,              # 默认值 0
    limit: int = 10,            # 默认值 10
    q: Optional[str] = None,    # 可选参数
    active: bool = True,        # 布尔参数（?active=true 或 ?active=1）
):
    return {"skip": skip, "limit": limit, "q": q, "active": active}

# 访问：GET /items?skip=0&limit=5&q=python&active=false


# 带校验的查询参数
@app.get("/search")
async def search(
    q: str = Query(min_length=3, max_length=50, description="搜索关键词"),
    page: int = Query(default=1, ge=1, description="页码"),
    size: int = Query(default=10, ge=1, le=100, description="每页数量"),
):
    return {"query": q, "page": page, "size": size}


# 带校验的路径参数
@app.get("/products/{product_id}")
async def get_product(
    product_id: int = Path(ge=1, description="商品 ID"),
):
    return {"product_id": product_id}


# 多种参数混合
class Item(BaseModel):
    name: str
    price: float

@app.post("/shops/{shop_id}/items")
async def create_item(
    shop_id: int,           # 路径参数
    item: Item,             # 请求体
    discount: float = 0.0,  # 查询参数
):
    return {
        "shop_id": shop_id,
        "item": item,
        "final_price": item.price * (1 - discount),
    }
```

## 4. 响应模型和状态码

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional

app = FastAPI()


class UserPublic(BaseModel):
    """公开的用户信息（不包含密码）"""
    id: int
    username: str
    email: str

class UserPrivate(BaseModel):
    """包含敏感信息的用户数据（内部使用）"""
    id: int
    username: str
    email: str
    password_hash: str  # 不应该返回给客户端


# response_model：过滤响应字段（只返回指定字段）
@app.get("/me", response_model=UserPublic)
async def get_current_user():
    # 返回的是 UserPrivate，但 FastAPI 会根据 response_model 过滤
    return UserPrivate(
        id=1,
        username="alice",
        email="alice@example.com",
        password_hash="$2b$12$...",  # 不会出现在响应中！
    )


# 多种响应类型
from fastapi import HTTPException
from pydantic import BaseModel

class Message(BaseModel):
    message: str

@app.get(
    "/items/{item_id}",
    responses={
        200: {"model": Item, "description": "成功返回商品"},
        404: {"model": Message, "description": "商品不存在"},
    }
)
async def get_item(item_id: int):
    if item_id > 100:
        raise HTTPException(status_code=404, detail="商品不存在")
    return Item(name=f"商品{item_id}", price=99.9)
```

## 5. 依赖注入（Dependency Injection）

FastAPI 的依赖注入比 Spring 简单得多：

```python
from fastapi import FastAPI, Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from typing import Annotated

app = FastAPI()


# 简单依赖：数据库连接
class Database:
    def __init__(self):
        self.connected = True
    
    def query(self, sql: str) -> list:
        # 模拟数据库查询
        return [{"id": 1, "data": "test"}]

def get_db() -> Database:
    """依赖函数：提供数据库连接"""
    db = Database()
    try:
        yield db  # 类似 context manager
    finally:
        # 清理资源（如关闭连接）
        pass

# 使用依赖
@app.get("/data")
async def get_data(db: Annotated[Database, Depends(get_db)]):
    return db.query("SELECT * FROM data")


# 认证依赖
security = HTTPBearer()

def verify_token(credentials: HTTPAuthorizationCredentials = Security(security)) -> dict:
    """验证 Bearer Token"""
    token = credentials.credentials
    # 简单示例：实际应该验证 JWT
    if token != "valid-token":
        raise HTTPException(status_code=401, detail="无效的 Token")
    return {"user_id": 1, "username": "alice"}

@app.get("/protected")
async def protected_route(
    current_user: Annotated[dict, Depends(verify_token)]
):
    return {"message": f"你好，{current_user['username']}！"}


# 多级依赖
def get_pagination(skip: int = 0, limit: int = 10) -> dict:
    return {"skip": skip, "limit": limit}

Pagination = Annotated[dict, Depends(get_pagination)]

@app.get("/users")
async def list_users(
    pagination: Pagination,
    db: Annotated[Database, Depends(get_db)],
):
    return {"pagination": pagination, "data": db.query("SELECT * FROM users")}
```

## 6. 中间件和异常处理

```python
import time
import logging
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

app = FastAPI()
logger = logging.getLogger(__name__)


# 自定义中间件（类比 Spring 的过滤器）
@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()
    
    # 前置处理
    logger.info(f"开始处理：{request.method} {request.url}")
    
    response = await call_next(request)  # 调用下一个处理器
    
    # 后置处理
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    logger.info(f"处理完成：{response.status_code}，耗时 {process_time:.3f}s")
    
    return response


# 全局异常处理（类比 Spring 的 @ControllerAdvice）
class AppException(Exception):
    def __init__(self, code: int, message: str):
        self.code = code
        self.message = message

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.code,
        content={"code": exc.code, "message": exc.message},
    )

@app.exception_handler(RequestValidationError)
async def validation_error_handler(request: Request, exc: RequestValidationError):
    """请求数据校验失败的处理"""
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={
            "code": 422,
            "message": "请求数据校验失败",
            "errors": exc.errors(),
        }
    )
```

## 7. 路由分组（APIRouter）

```python
# users/router.py
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel

router = APIRouter(prefix="/users", tags=["用户管理"])

class User(BaseModel):
    id: int
    name: str

@router.get("/", response_model=list[User])
async def list_users():
    return [User(id=1, name="Alice"), User(id=2, name="Bob")]

@router.get("/{user_id}", response_model=User)
async def get_user(user_id: int):
    return User(id=user_id, name="Alice")


# main.py
from fastapi import FastAPI
# from users.router import router as users_router

app = FastAPI()
# app.include_router(users_router)

# 也可以在注册时覆盖前缀
# app.include_router(users_router, prefix="/api/v1")
```

## 8. 完整示例：AI 聊天 API

```python
"""
AI 聊天 API 示例（综合 FastAPI 核心功能）
"""
import asyncio
from datetime import datetime
from typing import Optional
from fastapi import FastAPI, HTTPException, Depends
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
import uvicorn

app = FastAPI(title="AI Chat API", version="1.0.0")

# 数据模型
class Message(BaseModel):
    role: str  # "user" | "assistant"
    content: str
    timestamp: datetime = None
    
    def __init__(self, **data):
        if "timestamp" not in data:
            data["timestamp"] = datetime.now()
        super().__init__(**data)

class ChatRequest(BaseModel):
    session_id: str = "default"
    message: str
    stream: bool = False

class ChatResponse(BaseModel):
    session_id: str
    reply: str
    created_at: datetime

# 内存中的会话存储（生产环境用 Redis）
sessions: dict[str, list[Message]] = {}

# 依赖：获取会话历史
def get_session(session_id: str) -> list[Message]:
    if session_id not in sessions:
        sessions[session_id] = []
    return sessions[session_id]

# 模拟 LLM 回复
async def generate_reply(messages: list[Message]) -> str:
    await asyncio.sleep(0.5)  # 模拟 LLM 延迟
    last_msg = messages[-1].content
    return f"[AI 回复] 收到你的消息：'{last_msg}'，我正在学习中..."

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    history = get_session(request.session_id)
    
    # 添加用户消息
    user_msg = Message(role="user", content=request.message)
    history.append(user_msg)
    
    # 调用 LLM（实际项目中调用 OpenAI 等）
    reply = await generate_reply(history)
    
    # 添加 AI 回复
    ai_msg = Message(role="assistant", content=reply)
    history.append(ai_msg)
    
    return ChatResponse(
        session_id=request.session_id,
        reply=reply,
        created_at=datetime.now(),
    )

@app.get("/sessions/{session_id}/history")
async def get_history(session_id: str) -> list[Message]:
    if session_id not in sessions:
        raise HTTPException(status_code=404, detail="会话不存在")
    return sessions[session_id]

@app.delete("/sessions/{session_id}")
async def clear_session(session_id: str):
    sessions.pop(session_id, None)
    return {"message": f"会话 {session_id} 已清除"}

@app.get("/health")
async def health_check():
    return {"status": "ok", "sessions": len(sessions)}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## 常见陷阱与注意事项

### 1. 路径参数顺序

```python
# ❌ 错误：/users/me 会被 /users/{user_id} 捕获，user_id="me"
@app.get("/users/{user_id}")
async def get_user(user_id: int): ...

@app.get("/users/me")  # 永远不会被匹配到！
async def get_current_user(): ...

# ✅ 正确：精确路径放在前面
@app.get("/users/me")
async def get_current_user(): ...

@app.get("/users/{user_id}")
async def get_user(user_id: int): ...
```

### 2. 异步端点中的阻塞调用

```python
# ❌ 在 async 函数中使用同步阻塞操作
@app.get("/data")
async def get_data():
    import requests
    response = requests.get("https://api.example.com")  # 阻塞！
    return response.json()

# ✅ 使用异步 HTTP 客户端
@app.get("/data")
async def get_data():
    import httpx
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com")
        return response.json()
```

## 小结

FastAPI 与 Spring Boot 核心概念对比：

| Spring Boot | FastAPI |
|-------------|---------|
| `@RestController` | `FastAPI()` + 路由装饰器 |
| `@GetMapping("/path")` | `@app.get("/path")` |
| `@PostMapping` | `@app.post(...)` |
| `@RequestBody` | 函数参数中的 Pydantic 模型 |
| `@PathVariable` | 路径参数（`{param}`） |
| `@RequestParam` | 函数参数（有默认值） |
| `@Valid` | Pydantic 自动校验 |
| `ResponseEntity` | `response_model=` 参数 |
| `@Autowired` | `Depends(...)` |
| `@ControllerAdvice` | `@app.exception_handler(...)` |
| `Filter` | `@app.middleware("http")` |
| `@Component` | Python 函数/类 |
