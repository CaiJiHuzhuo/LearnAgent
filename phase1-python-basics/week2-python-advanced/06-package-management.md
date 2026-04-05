# Python 包管理工具

> pip、poetry、uv：从入门到现代包管理实践

## 概述

Python 的包管理生态比 Java 更分散，但近年来已有大幅改善。本文介绍三个核心工具：

| 工具 | 类比 Java | 特点 |
|------|---------|------|
| **pip** | Maven（只管安装） | 最基础，几乎所有 Python 环境内置 |
| **poetry** | Maven + 项目配置 | 现代化，完整的依赖管理和发包工具 |
| **uv** | Gradle（速度快） | 极速，Rust 编写，推荐新项目使用 |

## 1. pip：基础包管理

### 常用命令

```bash
# 安装包
pip install requests                  # 安装最新版本
pip install requests==2.31.0         # 安装指定版本
pip install "requests>=2.28,<3.0"    # 版本范围
pip install "fastapi[all]"           # 安装带可选依赖的包

# 升级
pip install --upgrade requests       # 升级到最新版本
pip install --upgrade pip            # 升级 pip 本身

# 卸载
pip uninstall requests               # 卸载（会询问确认）
pip uninstall -y requests            # 卸载（不询问）

# 查看已安装的包
pip list                             # 列出所有已安装包
pip list --outdated                  # 列出可升级的包
pip show requests                    # 查看包的详细信息

# 导出和恢复依赖
pip freeze > requirements.txt        # 导出当前环境的所有包
pip install -r requirements.txt      # 从文件安装

# 配置国内镜像源（永久配置）
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

# 单次使用镜像源
pip install requests -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### requirements.txt 文件

```
# requirements.txt 示例（类比 pom.xml 的 <dependencies>）
# 精确版本锁定（用于部署）
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
httpx==0.25.2
python-dotenv==1.0.0

# 版本范围（用于库开发）
# requests>=2.28.0,<3.0.0
# pydantic>=2.0.0
```

### pip 的局限性

- 不自动解析依赖冲突
- 没有锁文件（同一个 requirements.txt 在不同时间安装结果可能不同）
- 没有项目元数据（名称、版本等）
- 没有发包功能

## 2. poetry：现代化项目管理

Poetry 解决了 pip 的所有局限性，是目前最流行的 Python 项目管理工具。

### 安装 poetry

```bash
# 官方推荐安装方式（macOS/Linux/Windows）
curl -sSL https://install.python-poetry.org | python3 -

# 验证安装
poetry --version  # Poetry (version 1.7.x)
```

### 创建新项目

```bash
# 创建新项目（类比 mvn archetype:generate）
poetry new my-agent-project
# 目录结构：
# my-agent-project/
# ├── pyproject.toml    # 类比 pom.xml
# ├── README.md
# ├── my_agent_project/
# │   └── __init__.py
# └── tests/
#     └── __init__.py

# 在现有目录初始化（交互式）
cd existing-project
poetry init
```

### pyproject.toml 配置文件

```toml
# pyproject.toml（类比 pom.xml，但更简洁）
[tool.poetry]
name = "my-agent-project"
version = "0.1.0"
description = "AI Agent 学习项目"
authors = ["Alice <alice@example.com>"]
readme = "README.md"
python = "^3.11"       # 要求 Python 3.11+

[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.104.1"
pydantic = "^2.5.0"
httpx = "^0.25.0"
langchain = "^0.1.0"

[tool.poetry.group.dev.dependencies]
# 开发依赖（类比 Maven 的 <scope>test</scope>）
pytest = "^7.4.0"
mypy = "^1.7.0"
ruff = "^0.1.0"        # 代码格式化和 lint

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

### 常用 poetry 命令

```bash
# 安装所有依赖（类比 mvn install）
poetry install

# 只安装生产依赖（不含 dev 依赖）
poetry install --only main

# 添加依赖（类比 mvn dependency:add）
poetry add requests
poetry add "httpx>=0.25.0"
poetry add --group dev pytest  # 添加开发依赖

# 更新依赖
poetry update                  # 更新所有依赖
poetry update requests         # 更新指定包

# 删除依赖
poetry remove requests

# 在 poetry 管理的环境中运行命令
poetry run python main.py
poetry run pytest

# 激活虚拟环境
poetry shell                   # 进入虚拟环境 shell
# 退出：exit 或 Ctrl+D

# 查看依赖树（类比 mvn dependency:tree）
poetry show --tree

# 显示虚拟环境路径
poetry env info

# 导出 requirements.txt（兼容旧工具）
poetry export -f requirements.txt --output requirements.txt
```

### poetry.lock 文件

```
# poetry.lock 是自动生成的锁文件，记录所有依赖的精确版本
# 类比：Maven 的 effective pom + 所有 jar 的精确 SHA

# 提交 poetry.lock 到 Git：应用类项目 → YES
# 库开发 → 通常 NO（允许用户使用兼容范围的版本）

# 从 lock 文件精确复现环境（类比 mvn install 用 lock 文件）
poetry install --frozen  # 不更新 lock，用精确版本
```

## 3. uv：下一代极速包管理器

uv 是 Astral 公司用 Rust 编写的 Python 包管理器，速度比 pip 快 10-100 倍，正在迅速成为行业新标准。

### 安装 uv

```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# 或通过 pip
pip install uv

# 验证
uv --version  # uv 0.x.x
```

### uv 的主要用法

```bash
# === 作为 pip 的替代品 ===

# 安装包（极速！）
uv pip install requests
uv pip install -r requirements.txt

# 创建和管理虚拟环境
uv venv .venv          # 比 python -m venv 快很多
source .venv/bin/activate

# === 作为 poetry 的替代品（uv 项目管理）===

# 创建新项目
uv init my-project
cd my-project

# 添加依赖
uv add fastapi
uv add --dev pytest

# 安装所有依赖
uv sync

# 运行命令
uv run python main.py
uv run pytest

# 查看依赖树
uv tree

# === Python 版本管理（类似 pyenv）===
uv python install 3.11.9
uv python install 3.12.3
uv python list

# 在项目中指定 Python 版本
uv python pin 3.11
```

### pyproject.toml（uv 风格）

```toml
[project]
name = "my-agent-project"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.104.0",
    "pydantic>=2.5.0",
    "httpx>=0.25.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "mypy>=1.7.0",
    "ruff>=0.1.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

## 4. 推荐的项目结构

```
my-ai-agent/
├── pyproject.toml          # 项目配置（uv 或 poetry）
├── uv.lock                 # 锁文件（uv 生成）
├── .python-version         # Python 版本（pyenv 生成）
├── README.md
├── .env                    # 本地环境变量（不提交 Git！）
├── .env.example            # 环境变量模板（提交 Git）
├── .gitignore
├── src/
│   └── my_agent/
│       ├── __init__.py
│       ├── main.py         # 应用入口
│       ├── config.py       # 配置管理
│       ├── models.py       # Pydantic 模型
│       └── tools/          # Agent 工具
│           ├── __init__.py
│           └── web_search.py
└── tests/
    ├── __init__.py
    └── test_main.py
```

### .gitignore 常用配置

```gitignore
# Python
__pycache__/
*.py[cod]
*.so
.Python
.venv/
venv/
env/

# 环境变量（不提交！）
.env
.env.local

# 构建产物
dist/
build/
*.egg-info/

# IDE
.idea/
.vscode/
*.swp

# macOS
.DS_Store

# uv
.uv/
```

## 5. 常用包速查

```bash
# AI Agent 开发常用包

# HTTP 客户端
pip install httpx       # 现代异步 HTTP 客户端（推荐）
pip install requests    # 同步 HTTP 客户端（经典）
pip install aiohttp     # 另一个异步 HTTP 客户端

# Web 框架
pip install "fastapi[all]"  # 现代异步 Web 框架
pip install uvicorn     # ASGI 服务器（FastAPI 的运行时）

# 数据校验
pip install pydantic    # 数据校验和序列化

# LLM 相关
pip install openai              # OpenAI API 客户端
pip install anthropic           # Anthropic Claude API
pip install langchain           # LangChain 框架
pip install langchain-openai    # LangChain + OpenAI 集成

# 配置管理
pip install python-dotenv  # 加载 .env 文件

# 日志
pip install loguru      # 更好用的日志库

# 开发工具
pip install pytest      # 测试框架
pip install mypy        # 静态类型检查
pip install ruff        # 极速 lint 和格式化（Rust 编写）
```

## 常见陷阱与注意事项

### 1. 区分虚拟环境是否激活

```bash
# ❌ 在全局环境安装（污染系统 Python）
pip install requests  # 如果没有激活 venv

# ✅ 确认虚拟环境已激活
which python  # 应该显示 .venv/bin/python
# 或检查命令行前缀是否有 (.venv)
```

### 2. requirements.txt 的精确版本问题

```bash
# ❌ 不锁定版本（不同时间安装结果不同）
# requirements.txt
# requests
# fastapi

# ✅ 精确版本（推荐用于部署）
# requirements.txt
# requests==2.31.0
# fastapi==0.104.1

# 最佳：用 poetry 或 uv 的 lock 文件
```

### 3. 版本冲突的处理

```bash
# 遇到依赖冲突时
# 1. 查看冲突原因
pip install package-a package-b  # 可能报冲突

# 2. 使用 pipdeptree 查看依赖树
pip install pipdeptree
pipdeptree

# 3. 使用 poetry 或 uv（自动解析冲突）
poetry add package-a package-b  # 如果有冲突会告诉你
```

## 小结

| 场景 | 推荐工具 |
|------|---------|
| 快速学习、小脚本 | **pip + venv**（简单直接） |
| 新建正式项目 | **uv**（极速，现代） |
| 已有 poetry 项目 | **poetry**（继续使用） |
| 需要发布 Python 库 | **poetry** 或 **flit** |
| 数据科学项目 | **conda** 或 **uv** |

**2024 年推荐工作流（新项目）**：
```bash
# 1. 安装 uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. 创建项目
uv init my-project && cd my-project

# 3. 添加依赖
uv add fastapi pydantic httpx
uv add --dev pytest ruff mypy

# 4. 运行
uv run python main.py
```
