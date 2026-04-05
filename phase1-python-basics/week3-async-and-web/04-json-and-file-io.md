# JSON 处理与文件 I/O

> Python 中的 JSON 序列化、文件读写、路径操作

## 概述

JSON 处理和文件 I/O 是日常开发不可缺少的操作。Python 内置了强大的 `json` 模块和文件操作 API，配合 Pydantic 可以实现更优雅的数据处理。

| 操作 | Java | Python |
|------|------|--------|
| JSON 序列化 | Jackson / Gson | `json` 模块 / Pydantic |
| 读文件 | `Files.readString()` | `open()` + `read()` |
| 写文件 | `Files.writeString()` | `open()` + `write()` |
| 路径操作 | `Paths.get()`, `File` | `pathlib.Path` |
| CSV | Apache Commons CSV | `csv` 模块 / pandas |
| 配置文件 | `@Value`, YAML | `python-dotenv`, `pydantic-settings` |

## 1. JSON 处理

### 内置 json 模块

```python
import json

# 字典 → JSON 字符串（序列化）
data = {
    "name": "Alice",
    "age": 25,
    "skills": ["Python", "FastAPI"],
    "address": {"city": "北京", "country": "CN"},
}

json_str = json.dumps(data)  # '{"name": "Alice", ...}'
json_pretty = json.dumps(data, indent=2, ensure_ascii=False)  # 格式化+中文不转义
print(json_pretty)

# JSON 字符串 → 字典（反序列化）
parsed = json.loads(json_str)
print(parsed["name"])  # Alice
print(type(parsed))    # <class 'dict'>

# 写入文件
with open("data.json", "w", encoding="utf-8") as f:
    json.dump(data, f, indent=2, ensure_ascii=False)

# 从文件读取
with open("data.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)
```

### 处理特殊类型

```python
import json
from datetime import datetime, date
from decimal import Decimal
from enum import Enum

# ❌ datetime 不能直接 JSON 序列化
data = {"created_at": datetime.now()}
# json.dumps(data)  # TypeError: Object of type datetime is not JSON serializable

# ✅ 方法1：自定义 JSONEncoder
class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, (datetime, date)):
            return obj.isoformat()
        if isinstance(obj, Decimal):
            return float(obj)
        if isinstance(obj, Enum):
            return obj.value
        return super().default(obj)

json_str = json.dumps(data, cls=CustomEncoder)

# ✅ 方法2：使用 Pydantic（推荐，自动处理）
from pydantic import BaseModel
from datetime import datetime

class Event(BaseModel):
    name: str
    created_at: datetime
    price: float

event = Event(name="会议", created_at=datetime.now(), price=99.9)
json_str = event.model_dump_json()  # 自动处理 datetime
print(json_str)  # {"name":"会议","created_at":"2024-01-01T10:00:00","price":99.9}
```

## 2. 文件 I/O

### 文本文件

```python
# 写文件
with open("hello.txt", "w", encoding="utf-8") as f:
    f.write("Hello, World!\n")
    f.write("Python 文件 I/O 示例\n")
    print("这会写到文件", file=f)  # print 可以指定输出到文件

# 读文件
# 方式1：读取全部内容
with open("hello.txt", "r", encoding="utf-8") as f:
    content = f.read()
    print(content)

# 方式2：逐行读取（推荐大文件使用）
with open("hello.txt", "r", encoding="utf-8") as f:
    for line in f:  # 文件对象是迭代器
        print(line.strip())  # strip() 去除换行符

# 方式3：读取所有行到列表
with open("hello.txt", "r", encoding="utf-8") as f:
    lines = f.readlines()  # 包含换行符
    lines = [line.strip() for line in f.readlines()]  # 去换行符

# 追加写入
with open("hello.txt", "a", encoding="utf-8") as f:
    f.write("追加的内容\n")

# 文件模式
# "r"  - 只读（默认）
# "w"  - 覆盖写入
# "a"  - 追加写入
# "r+" - 读写
# "rb" - 二进制读
# "wb" - 二进制写
```

### 二进制文件

```python
# 读写图片等二进制文件
with open("image.jpg", "rb") as f:
    image_data = f.read()

with open("image_copy.jpg", "wb") as f:
    f.write(image_data)

# 使用 pathlib 更简洁
from pathlib import Path

image_path = Path("image.jpg")
image_data = image_path.read_bytes()  # 读取二进制
Path("image_copy.jpg").write_bytes(image_data)  # 写入二进制
```

## 3. pathlib：现代路径操作

`pathlib` 是 Python 3.4+ 内置的路径操作库，远比字符串拼接路径好用。

```python
from pathlib import Path

# 创建路径对象
current_dir = Path(".")           # 当前目录
home_dir = Path.home()           # 用户主目录（/home/user 或 C:\Users\user）
abs_path = Path("/usr/local")    # 绝对路径

# 路径拼接（/ 运算符！）
data_dir = home_dir / "projects" / "my-agent" / "data"
config_file = data_dir / "config.json"

print(config_file)              # /home/user/projects/my-agent/data/config.json
print(config_file.parent)       # /home/user/projects/my-agent/data
print(config_file.name)         # config.json
print(config_file.stem)         # config
print(config_file.suffix)       # .json
print(config_file.absolute())   # 绝对路径

# 路径检查
config_file.exists()            # 是否存在
config_file.is_file()           # 是否是文件
config_file.is_dir()            # 是否是目录

# 创建目录
data_dir.mkdir(parents=True, exist_ok=True)  # 递归创建，已存在不报错

# 文件读写（Path 对象自带 read/write 方法）
text_file = Path("example.txt")
text_file.write_text("Hello!", encoding="utf-8")  # 写文本
content = text_file.read_text(encoding="utf-8")   # 读文本

binary_file = Path("data.bin")
binary_file.write_bytes(b"\x00\x01\x02")           # 写二进制
data = binary_file.read_bytes()                    # 读二进制

# 遍历目录
project_dir = Path(".")
for f in project_dir.iterdir():  # 列出直接子项
    print(f.name, "目录" if f.is_dir() else "文件")

# glob 匹配文件
for py_file in project_dir.glob("**/*.py"):  # 递归查找所有 .py 文件
    print(py_file)

for md_file in project_dir.glob("*.md"):     # 当前目录的 .md 文件
    print(md_file)

# 删除文件/目录
text_file.unlink()              # 删除文件
empty_dir = Path("empty")
empty_dir.mkdir(exist_ok=True)
empty_dir.rmdir()               # 删除空目录

import shutil
shutil.rmtree("non-empty-dir")  # 删除非空目录

# 获取文件信息
stat = text_file.stat()
print(f"文件大小：{stat.st_size} 字节")
print(f"修改时间：{stat.st_mtime}")
```

## 4. CSV 文件处理

```python
import csv
from pathlib import Path

# 写 CSV
data = [
    ["姓名", "年龄", "城市"],
    ["Alice", 25, "北京"],
    ["Bob", 30, "上海"],
    ["Carol", 28, "广州"],
]

with open("users.csv", "w", newline="", encoding="utf-8-sig") as f:
    # utf-8-sig 确保 Excel 打开中文不乱码
    writer = csv.writer(f)
    writer.writerows(data)

# 读 CSV
with open("users.csv", "r", encoding="utf-8-sig") as f:
    reader = csv.DictReader(f)  # DictReader 把每行转为字典
    for row in reader:
        print(row)  # {'姓名': 'Alice', '年龄': '25', '城市': '北京'}
        # 注意：所有值都是字符串，需要手动转换类型

# 使用 dataclasses 和 csv
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
    city: str

with open("users.csv", "r", encoding="utf-8-sig") as f:
    reader = csv.DictReader(f)
    users = [User(name=r["姓名"], age=int(r["年龄"]), city=r["城市"]) for r in reader]

print(users[0])  # User(name='Alice', age=25, city='北京')
```

## 5. 配置文件管理

### .env 文件（python-dotenv）

```bash
pip install python-dotenv
```

```python
# .env 文件
# OPENAI_API_KEY=sk-...
# DATABASE_URL=postgresql://user:pass@localhost/mydb
# DEBUG=true
# MAX_WORKERS=4

import os
from dotenv import load_dotenv
from pathlib import Path

# 加载 .env 文件
load_dotenv()

# 读取环境变量
api_key = os.getenv("OPENAI_API_KEY")
debug = os.getenv("DEBUG", "false").lower() == "true"
max_workers = int(os.getenv("MAX_WORKERS", "2"))
```

### pydantic-settings（推荐）

```bash
pip install pydantic-settings
```

```python
from pydantic_settings import BaseSettings
from pydantic import Field

class Settings(BaseSettings):
    """应用配置（自动从环境变量和 .env 加载）"""
    
    # LLM 配置
    openai_api_key: str = Field(description="OpenAI API Key")
    openai_model: str = "gpt-3.5-turbo"
    
    # 应用配置
    debug: bool = False
    max_workers: int = 4
    
    # 数据库配置
    database_url: str = "sqlite:///./app.db"
    
    class Config:
        env_file = ".env"           # 从 .env 加载
        env_file_encoding = "utf-8"
        case_sensitive = False      # 不区分大小写


# 使用
settings = Settings()
print(settings.openai_model)  # gpt-3.5-turbo（从 .env 或环境变量）
print(settings.debug)         # False
```

## 6. 临时文件

```python
import tempfile

# 创建临时文件
with tempfile.NamedTemporaryFile(mode="w", suffix=".txt", delete=False) as f:
    f.write("临时内容")
    temp_path = Path(f.name)
    print(f"临时文件：{temp_path}")
# 出了 with 块，文件仍存在（delete=False）

# 清理
temp_path.unlink()

# 临时目录
with tempfile.TemporaryDirectory() as tmpdir:
    tmp_path = Path(tmpdir)
    (tmp_path / "data.txt").write_text("测试")
    print(f"临时目录：{tmp_path}")
# 出了 with 块，目录自动删除
```

## 常见陷阱与注意事项

### 1. 文件编码问题

```python
# ❌ 不指定编码（不同操作系统默认编码不同）
with open("chinese.txt", "w") as f:
    f.write("你好，世界！")  # Windows 可能用 GBK，Linux 用 UTF-8

# ✅ 显式指定 UTF-8
with open("chinese.txt", "w", encoding="utf-8") as f:
    f.write("你好，世界！")
```

### 2. Windows 路径斜杠问题

```python
# ❌ Windows 路径硬编码（不跨平台）
path = "C:\\Users\\alice\\data\\file.txt"
path = r"C:\Users\alice\data\file.txt"

# ✅ 使用 pathlib（自动处理分隔符）
path = Path.home() / "data" / "file.txt"
print(path)  # Windows: C:\Users\alice\data\file.txt
             # Linux/Mac: /home/alice/data/file.txt
```

### 3. 大文件处理

```python
# ❌ 一次性读取大文件（可能 OOM）
with open("huge_file.txt") as f:
    content = f.read()  # 几GB 的文件会耗尽内存！

# ✅ 逐行读取（内存友好）
with open("huge_file.txt", encoding="utf-8") as f:
    for line in f:  # 迭代器，逐行处理
        process(line)

# ✅ 分块读取二进制文件
CHUNK_SIZE = 8 * 1024 * 1024  # 8MB 分块
with open("huge_binary.bin", "rb") as f:
    while chunk := f.read(CHUNK_SIZE):  # 海象运算符（Python 3.8+）
        process_chunk(chunk)
```

## 小结

| 操作 | 代码 |
|------|------|
| 读 JSON 文件 | `json.load(open(path))` |
| 写 JSON 文件 | `json.dump(data, open(path, "w"), ensure_ascii=False)` |
| 读文本文件 | `Path(p).read_text(encoding="utf-8")` |
| 写文本文件 | `Path(p).write_text(text, encoding="utf-8")` |
| 路径拼接 | `Path("/base") / "sub" / "file.txt"` |
| 创建目录 | `Path(p).mkdir(parents=True, exist_ok=True)` |
| 遍历目录 | `Path(p).glob("**/*.py")` |
| 加载 .env | `from dotenv import load_dotenv; load_dotenv()` |
| Pydantic 配置 | `class Settings(BaseSettings): ...` |
