# 第三周练习题：用 FastAPI 搭建 AI Agent API 服务

> 综合运用第三周和前两周的所有知识，构建一个完整的 AI Agent 后端服务

## 练习题总览

| 题号 | 题目 | 难度 | 考察知识点 |
|------|------|------|-----------|
| 1 | 异步并发请求 | ⭐⭐ | asyncio、httpx |
| 2 | 文件处理服务 | ⭐⭐ | FastAPI、文件 I/O |
| 3 | 带认证的 API | ⭐⭐⭐ | FastAPI 依赖注入 |
| 4 | 流式输出 | ⭐⭐⭐ | FastAPI、生成器 |
| 5 | 🎯 综合实战：AI Agent 后端服务 | ⭐⭐⭐⭐⭐ | 综合所有知识点 |

---

## 练习一：异步并发请求

### 题目描述

实现一个异步函数，并发调用多个 API 并聚合结果：
1. 并发获取 5 个不同 GitHub 用户的信息
2. 计算所有用户的平均 follower 数量
3. 找出 follower 最多的用户
4. 要求：添加超时控制（每个请求 5 秒超时）、异常处理、重试逻辑

### 输入输出示例

```python
# 示例调用（在异步环境中运行）
users = ["python", "requests", "pallets", "encode", "tiangolo"]
# result = await fetch_github_users(users)  # 在 asyncio 环境中使用
# {
#   "users": [{"login": "python", "followers": 1000}, ...],
#   "average_followers": 856.4,
#   "top_user": {"login": "python", "followers": 1000},
#   "failed": []  # 获取失败的用户名
# }
```

### 参考解答

```python
import asyncio
from typing import Optional
import httpx
from pydantic import BaseModel

class GitHubUser(BaseModel):
    login: str
    followers: int
    public_repos: int

class AggregateResult(BaseModel):
    users: list[GitHubUser]
    average_followers: float
    top_user: Optional[GitHubUser]
    failed: list[str]

async def fetch_user(
    client: httpx.AsyncClient,
    username: str
) -> Optional[GitHubUser]:
    """获取单个 GitHub 用户（带超时和错误处理）"""
    try:
        response = await client.get(
            f"/users/{username}",
            timeout=5.0
        )
        response.raise_for_status()
        data = response.json()
        return GitHubUser(
            login=data["login"],
            followers=data["followers"],
            public_repos=data["public_repos"],
        )
    except (httpx.TimeoutException, httpx.HTTPError) as e:
        print(f"获取 {username} 失败：{e}")
        return None

async def fetch_github_users(usernames: list[str]) -> AggregateResult:
    """并发获取多个 GitHub 用户信息"""
    async with httpx.AsyncClient(base_url="https://api.github.com") as client:
        tasks = [fetch_user(client, username) for username in usernames]
        results = await asyncio.gather(*tasks, return_exceptions=True)
    
    users = []
    failed = []
    
    for username, result in zip(usernames, results):
        if isinstance(result, GitHubUser):
            users.append(result)
        else:
            failed.append(username)
    
    avg_followers = sum(u.followers for u in users) / len(users) if users else 0
    top_user = max(users, key=lambda u: u.followers) if users else None
    
    return AggregateResult(
        users=users,
        average_followers=avg_followers,
        top_user=top_user,
        failed=failed,
    )

# 测试
import asyncio

async def main():
    usernames = ["python", "requests", "encode", "tiangolo", "fastapi"]
    result = await fetch_github_users(usernames)
    print(f"成功获取：{len(result.users)} 个用户")
    print(f"平均 Followers：{result.average_followers:.0f}")
    if result.top_user:
        print(f"Followers 最多：{result.top_user.login} ({result.top_user.followers})")
    if result.failed:
        print(f"获取失败：{result.failed}")

asyncio.run(main())
```

---

## 练习二：文件处理服务

### 题目描述

用 FastAPI 实现文件处理 API：
1. `POST /upload` - 上传文件（支持 .txt、.json、.csv）
2. `GET /files` - 列出已上传的文件
3. `GET /files/{filename}` - 下载文件
4. `DELETE /files/{filename}` - 删除文件
5. `POST /files/{filename}/analyze` - 分析文件内容（统计行数、字符数等）

### 参考解答

```python
import shutil
from pathlib import Path
from typing import Annotated

from fastapi import FastAPI, UploadFile, File, HTTPException
from fastapi.responses import FileResponse
from pydantic import BaseModel

app = FastAPI(title="文件处理服务")

UPLOAD_DIR = Path("uploads")
UPLOAD_DIR.mkdir(exist_ok=True)
ALLOWED_EXTENSIONS = {".txt", ".json", ".csv", ".md"}

class FileInfo(BaseModel):
    name: str
    size: int  # bytes
    extension: str

class FileAnalysis(BaseModel):
    name: str
    lines: int
    characters: int
    words: int
    size_bytes: int

@app.post("/upload", response_model=FileInfo)
async def upload_file(file: UploadFile = File(...)):
    """上传文件"""
    suffix = Path(file.filename).suffix.lower()
    if suffix not in ALLOWED_EXTENSIONS:
        raise HTTPException(400, f"不支持的文件类型：{suffix}，只允许 {ALLOWED_EXTENSIONS}")
    
    dest_path = UPLOAD_DIR / file.filename
    with dest_path.open("wb") as f:
        shutil.copyfileobj(file.file, f)
    
    return FileInfo(
        name=file.filename,
        size=dest_path.stat().st_size,
        extension=suffix,
    )

@app.get("/files", response_model=list[FileInfo])
async def list_files():
    """列出所有文件"""
    files = []
    for f in UPLOAD_DIR.iterdir():
        if f.is_file() and f.suffix.lower() in ALLOWED_EXTENSIONS:
            files.append(FileInfo(
                name=f.name,
                size=f.stat().st_size,
                extension=f.suffix,
            ))
    return sorted(files, key=lambda x: x.name)

@app.get("/files/{filename}")
async def download_file(filename: str):
    """下载文件"""
    file_path = UPLOAD_DIR / filename
    if not file_path.exists() or not file_path.is_file():
        raise HTTPException(404, f"文件不存在：{filename}")
    return FileResponse(file_path, filename=filename)

@app.delete("/files/{filename}", status_code=204)
async def delete_file(filename: str):
    """删除文件"""
    file_path = UPLOAD_DIR / filename
    if not file_path.exists():
        raise HTTPException(404, f"文件不存在：{filename}")
    file_path.unlink()

@app.post("/files/{filename}/analyze", response_model=FileAnalysis)
async def analyze_file(filename: str):
    """分析文件内容"""
    file_path = UPLOAD_DIR / filename
    if not file_path.exists():
        raise HTTPException(404, f"文件不存在：{filename}")
    
    content = file_path.read_text(encoding="utf-8", errors="ignore")
    lines = content.splitlines()
    words = content.split()
    
    return FileAnalysis(
        name=filename,
        lines=len(lines),
        characters=len(content),
        words=len(words),
        size_bytes=file_path.stat().st_size,
    )
```

---

## 练习三：带认证的 API

### 题目描述

为 FastAPI 应用添加 JWT 认证：
1. `POST /auth/register` - 注册用户
2. `POST /auth/login` - 登录，返回 JWT Token
3. `GET /profile` - 获取当前用户信息（需要 Token）
4. `PUT /profile` - 更新个人信息（需要 Token）

### 提示

使用以下库：
```bash
pip install "python-jose[cryptography]" "passlib[bcrypt]"
```

### 参考解答

```python
from datetime import datetime, timedelta
from typing import Annotated, Optional

from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from pydantic import BaseModel, EmailStr

# 配置
SECRET_KEY = "your-secret-key-change-in-production"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

app = FastAPI(title="认证 API")

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

# 模型
class UserCreate(BaseModel):
    username: str
    email: EmailStr
    password: str

class UserResponse(BaseModel):
    username: str
    email: str

class Token(BaseModel):
    access_token: str
    token_type: str = "bearer"

# 模拟数据库
fake_users_db: dict[str, dict] = {}

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode["exp"] = expire
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: Annotated[str, Depends(oauth2_scheme)]) -> dict:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="无效的认证凭证",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    
    user = fake_users_db.get(username)
    if user is None:
        raise credentials_exception
    return user

@app.post("/auth/register", response_model=UserResponse, status_code=201)
async def register(user_data: UserCreate):
    if user_data.username in fake_users_db:
        raise HTTPException(400, "用户名已存在")
    
    fake_users_db[user_data.username] = {
        "username": user_data.username,
        "email": user_data.email,
        "hashed_password": hash_password(user_data.password),
    }
    return UserResponse(username=user_data.username, email=user_data.email)

@app.post("/auth/login", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = fake_users_db.get(form_data.username)
    if not user or not verify_password(form_data.password, user["hashed_password"]):
        raise HTTPException(status_code=401, detail="用户名或密码错误")
    
    token = create_token(
        data={"sub": user["username"]},
        expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),
    )
    return Token(access_token=token)

@app.get("/profile", response_model=UserResponse)
async def get_profile(current_user: Annotated[dict, Depends(get_current_user)]):
    return UserResponse(username=current_user["username"], email=current_user["email"])
```

---

## 练习四：流式输出

### 题目描述

实现一个模拟 LLM 流式输出的 API：
1. `POST /chat/stream` - 模拟流式 AI 回复
2. 使用 Server-Sent Events（SSE）格式
3. 每次生成一个词语后立即发送
4. 客户端代码演示如何消费流式响应

### 参考解答

```python
import asyncio
import json
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from pydantic import BaseModel

app = FastAPI()

class ChatRequest(BaseModel):
    message: str
    delay: float = 0.1  # 模拟每个 token 的延迟

async def generate_stream(message: str, delay: float = 0.1):
    """模拟 LLM 流式生成"""
    # 模拟 AI 回复
    response_parts = [
        "好的，",
        "让我来",
        "回答你的问题。",
        f"\n\n你问的是：'{message}'",
        "\n\n这是一个很好的问题！",
        "\nPython 的异步编程",
        "使用 async/await 语法，",
        "基于事件循环工作，",
        "非常适合 I/O 密集型场景。",
        "\n\n希望这个回答对你有帮助！",
    ]
    
    for part in response_parts:
        # SSE 格式：每行以 "data: " 开头，以 "\n\n" 结尾
        chunk = {
            "type": "content",
            "content": part,
        }
        yield f"data: {json.dumps(chunk, ensure_ascii=False)}\n\n"
        await asyncio.sleep(delay)
    
    # 发送结束标志
    yield f"data: {json.dumps({'type': 'done'})}\n\n"

@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    """流式聊天接口"""
    return StreamingResponse(
        generate_stream(request.message, request.delay),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",
        },
    )

# 测试客户端（Python）
async def consume_stream():
    """消费流式响应的客户端代码"""
    import httpx
    
    async with httpx.AsyncClient() as client:
        async with client.stream(
            "POST",
            "http://localhost:8000/chat/stream",
            json={"message": "什么是 Python 异步编程？"},
        ) as response:
            async for line in response.aiter_lines():
                if line.startswith("data: "):
                    data = json.loads(line[6:])
                    if data["type"] == "content":
                        print(data["content"], end="", flush=True)
                    elif data["type"] == "done":
                        print("\n[流式输出完成]")
                        break

# 运行客户端测试
if __name__ == "__main__":
    asyncio.run(consume_stream())
```

---

## 练习五：🎯 综合实战 - AI Agent 后端服务

### 题目描述

构建一个完整的 AI Agent 后端服务，综合运用三周所有知识：

**功能需求：**

1. **用户认证**（JWT）
2. **会话管理**：创建、查询、删除对话会话
3. **聊天 API**：发送消息，支持流式和非流式模式
4. **工具调用**：实现两个 Tool（天气查询、计算器），Agent 可以决定是否调用
5. **配置管理**：使用 `pydantic-settings` 管理配置
6. **日志**：使用 `loguru` 记录请求和错误
7. **文档**：FastAPI 自动生成 Swagger 文档

**项目结构：**

```
agent_api/
├── main.py            # FastAPI 应用入口
├── config.py          # 配置管理（pydantic-settings）
├── auth.py            # 认证逻辑（JWT）
├── models.py          # Pydantic 数据模型
├── session.py         # 会话管理
├── tools.py           # Agent 工具（天气、计算器）
├── agent.py           # Agent 逻辑（模拟 LLM 调用和工具选择）
└── routers/
    ├── auth.py        # 认证路由
    ├── chat.py        # 聊天路由
    └── tools.py       # 工具路由
```

### 核心代码参考

**config.py：**
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_name: str = "AI Agent API"
    debug: bool = False
    secret_key: str = "change-me-in-production"
    
    # 模拟 LLM 配置
    llm_delay: float = 0.5  # 模拟 LLM 响应延迟
    
    class Config:
        env_file = ".env"

settings = Settings()
```

**models.py：**
```python
from pydantic import BaseModel
from datetime import datetime
from typing import Optional, Literal

class Message(BaseModel):
    role: Literal["user", "assistant", "system", "tool"]
    content: str
    timestamp: datetime = None
    tool_name: Optional[str] = None  # 工具调用时的工具名称
    
    def __init__(self, **data):
        if "timestamp" not in data:
            data["timestamp"] = datetime.now()
        super().__init__(**data)

class Session(BaseModel):
    session_id: str
    user_id: str
    messages: list[Message] = []
    created_at: datetime
    updated_at: datetime

class ChatRequest(BaseModel):
    message: str
    session_id: Optional[str] = None
    stream: bool = False

class ChatResponse(BaseModel):
    session_id: str
    reply: str
    tools_used: list[str] = []
    created_at: datetime
```

**tools.py（工具实现）：**
```python
import asyncio
from typing import Any

class Tool:
    """工具基类"""
    name: str
    description: str
    
    async def run(self, **kwargs: Any) -> str:
        raise NotImplementedError

class Calculator(Tool):
    name = "calculator"
    description = "执行数学计算。参数：expression（字符串，如 '2 + 3 * 4'）"
    
    async def run(self, expression: str) -> str:
        try:
            # 安全的数学表达式求值
            allowed_chars = set("0123456789+-*/.() ")
            if not all(c in allowed_chars for c in expression):
                return "错误：不支持的字符"
            result = eval(expression)  # 生产环境应使用 ast.literal_eval 或 sympy
            return f"计算结果：{expression} = {result}"
        except Exception as e:
            return f"计算错误：{e}"

class WeatherTool(Tool):
    name = "weather"
    description = "获取城市天气（模拟）。参数：city（城市名称）"
    
    async def run(self, city: str) -> str:
        await asyncio.sleep(0.3)  # 模拟 API 延迟
        # 模拟天气数据
        weather_data = {
            "北京": "晴，25°C，湿度 45%",
            "上海": "多云，22°C，湿度 68%",
            "广州": "小雨，28°C，湿度 85%",
        }
        return weather_data.get(city, f"{city}：暂无天气数据")

TOOLS = {tool.name: tool for tool in [Calculator(), WeatherTool()]}
```

**agent.py（Agent 逻辑）：**
```python
import asyncio
import re
from .models import Message
from .tools import TOOLS

async def run_agent(
    user_message: str,
    history: list[Message],
) -> tuple[str, list[str]]:
    """
    简化的 Agent 逻辑：
    1. 检查用户消息是否需要调用工具
    2. 如果需要，调用工具并获取结果
    3. 生成最终回复
    
    Returns: (reply, tools_used)
    """
    tools_used = []
    tool_results = []
    
    # 简单的工具触发检测（生产环境用 LLM 决定）
    if any(keyword in user_message for keyword in ["计算", "算一下", "+", "-", "*", "/"]):
        # 提取数学表达式
        numbers = re.findall(r"[\d+\-*/().\s]+", user_message)
        if numbers:
            expr = numbers[0].strip()
            result = await TOOLS["calculator"].run(expression=expr)
            tool_results.append(result)
            tools_used.append("calculator")
    
    if any(keyword in user_message for keyword in ["天气", "weather", "温度"]):
        cities = ["北京", "上海", "广州"]
        for city in cities:
            if city in user_message:
                result = await TOOLS["weather"].run(city=city)
                tool_results.append(result)
                tools_used.append("weather")
                break
    
    # 生成回复（实际项目中调用 LLM）
    await asyncio.sleep(0.5)  # 模拟 LLM 延迟
    
    if tool_results:
        reply = (
            f"我已经为你查询了相关信息：\n\n"
            + "\n".join(tool_results)
            + f"\n\n根据以上信息，{user_message} 的回答是：以上数据供参考。"
        )
    else:
        reply = f"收到你的问题：'{user_message}'。这是一个模拟的 AI 回复，实际项目中会调用真实的 LLM API。"
    
    return reply, tools_used
```

**main.py（FastAPI 应用）：**
```python
import uuid
from datetime import datetime
from typing import Annotated, Optional
from fastapi import FastAPI, HTTPException, Depends
from fastapi.responses import StreamingResponse
from loguru import logger
import json
import asyncio

from .config import settings
from .models import Message, Session, ChatRequest, ChatResponse
from .agent import run_agent

app = FastAPI(title=settings.app_name, debug=settings.debug)

# 内存会话存储（生产环境用 Redis）
sessions: dict[str, Session] = {}

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    """发送消息（非流式）"""
    # 获取或创建会话
    session_id = request.session_id or str(uuid.uuid4())
    if session_id not in sessions:
        sessions[session_id] = Session(
            session_id=session_id,
            user_id="anonymous",
            created_at=datetime.now(),
            updated_at=datetime.now(),
        )
    
    session = sessions[session_id]
    
    # 添加用户消息
    session.messages.append(Message(role="user", content=request.message))
    
    # 运行 Agent
    logger.info(f"处理消息：session={session_id}, message='{request.message[:50]}'")
    reply, tools_used = await run_agent(request.message, session.messages)
    
    # 添加 AI 回复
    session.messages.append(Message(role="assistant", content=reply))
    session.updated_at = datetime.now()
    
    return ChatResponse(
        session_id=session_id,
        reply=reply,
        tools_used=tools_used,
        created_at=datetime.now(),
    )

@app.get("/sessions/{session_id}/history")
async def get_history(session_id: str):
    """获取会话历史"""
    if session_id not in sessions:
        raise HTTPException(404, "会话不存在")
    return sessions[session_id]

@app.delete("/sessions/{session_id}", status_code=204)
async def delete_session(session_id: str):
    """删除会话"""
    sessions.pop(session_id, None)

@app.get("/health")
async def health():
    return {"status": "ok", "sessions": len(sessions)}
```

### 运行和测试

```bash
# 安装依赖
pip install "fastapi[all]" pydantic-settings loguru

# 启动服务
uvicorn agent_api.main:app --reload

# 测试（使用 curl）
# 发送聊天消息
curl -X POST "http://localhost:8000/chat" \
  -H "Content-Type: application/json" \
  -d '{"message": "帮我计算 123 * 456"}'

# 查询天气
curl -X POST "http://localhost:8000/chat" \
  -H "Content-Type: application/json" \
  -d '{"message": "北京今天天气怎么样？"}'

# 访问 Swagger 文档
# http://localhost:8000/docs
```

---

## 阶段一总结

🎉 **恭喜完成阶段一！**

通过三周的学习和 15+ 道练习题，你已经掌握了：

### 第一周：Python 基础
- ✅ Python 开发环境搭建
- ✅ 基础语法（与 Java 的对比）
- ✅ 数据结构（list、dict、set、tuple、推导式）
- ✅ 面向对象编程（类、继承、魔术方法）
- ✅ Python vs Java 核心差异

### 第二周：Python 进阶
- ✅ 迭代器与生成器（yield）
- ✅ 装饰器（AOP 风格编程）
- ✅ 上下文管理器（资源管理）
- ✅ 类型提示 + Pydantic 数据校验
- ✅ 异常处理最佳实践
- ✅ 包管理（pip/poetry/uv）

### 第三周：异步与 Web
- ✅ asyncio 异步编程
- ✅ HTTP 客户端（requests/httpx）
- ✅ FastAPI 框架
- ✅ JSON 与文件 I/O
- ✅ 日志库（logging/loguru）
- ✅ Jupyter Notebook

**你现在已经准备好进入阶段二：LLM 基础！** 🚀
