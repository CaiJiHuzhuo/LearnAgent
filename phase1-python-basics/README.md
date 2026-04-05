# 阶段一：Python 编程基础（第 1-3 周）

> 面向 Java 程序员的 Python 快速入门学习指南

## 🎯 阶段目标

完成本阶段学习后，你将能够：

- 熟练使用 Python 基础语法，理解 Python 与 Java 的核心差异
- 掌握 Python 高级特性：装饰器、生成器、上下文管理器
- 使用类型提示（Type Hints）和 Pydantic 编写健壮的 Python 代码
- 理解并使用 Python 异步编程（asyncio）
- 使用 FastAPI 搭建 REST API 服务
- 具备进入阶段二（LLM 基础）的 Python 编程能力

## 📋 前置要求

| 要求 | 说明 |
|------|------|
| Java 经验 | 有 1 年以上 Java 开发经验，了解 OOP 概念 |
| 开发工具 | 安装了 VS Code 或 IntelliJ IDEA |
| 操作系统 | Windows / macOS / Linux 均可 |
| 网络 | 能够访问 PyPI（pip 安装包），推荐配置国内镜像 |

## 🗺️ 学习路线图

```
Week 1: Python 基础           Week 2: Python 进阶           Week 3: 异步与 Web
┌─────────────────────┐      ┌─────────────────────┐      ┌─────────────────────┐
│ 环境搭建             │      │ 迭代器与生成器        │      │ asyncio 异步编程     │
│ 基础语法 (vs Java)   │ ───► │ 装饰器               │ ───► │ HTTP 客户端库        │
│ 数据结构             │      │ 上下文管理器          │      │ FastAPI 框架         │
│ 面向对象编程         │      │ 类型提示 + Pydantic   │      │ JSON 与文件 I/O      │
│ Java vs Python 总结  │      │ 异常处理              │      │ 日志库               │
└─────────────────────┘      │ 包管理工具            │      │ Jupyter Notebook     │
                             └─────────────────────┘      └─────────────────────┘
```

## 📅 各周概览

### 第一周：Python 基础（打好地基）

| 文件 | 内容 | 预计时间 |
|------|------|---------|
| [01-environment-setup.md](week1-python-fundamentals/01-environment-setup.md) | pyenv、conda、venv 环境搭建 | 2h |
| [02-basic-syntax.md](week1-python-fundamentals/02-basic-syntax.md) | 变量、数据类型、控制流、函数 | 3h |
| [03-data-structures.md](week1-python-fundamentals/03-data-structures.md) | list、dict、set、tuple、推导式 | 3h |
| [04-oop.md](week1-python-fundamentals/04-oop.md) | 类、继承、魔术方法 | 3h |
| [05-java-vs-python.md](week1-python-fundamentals/05-java-vs-python.md) | 核心差异总结 | 1h |
| [练习题](week1-python-fundamentals/exercises/week1-exercises.md) | 用 Python 重写 Java 小项目 | 4h |

### 第二周：Python 进阶（掌握核心特性）

| 文件 | 内容 | 预计时间 |
|------|------|---------|
| [01-iterators-generators.md](week2-python-advanced/01-iterators-generators.md) | 迭代器与生成器（yield） | 2h |
| [02-decorators.md](week2-python-advanced/02-decorators.md) | 装饰器（decorator） | 2h |
| [03-context-managers.md](week2-python-advanced/03-context-managers.md) | 上下文管理器（with 语句） | 1.5h |
| [04-type-hints-pydantic.md](week2-python-advanced/04-type-hints-pydantic.md) | 类型提示与 Pydantic | 3h |
| [05-exception-handling.md](week2-python-advanced/05-exception-handling.md) | 异常处理最佳实践 | 1.5h |
| [06-package-management.md](week2-python-advanced/06-package-management.md) | pip、poetry、uv | 2h |
| [练习题](week2-python-advanced/exercises/week2-exercises.md) | REST 客户端开发 | 4h |

### 第三周：异步与 Web（实战应用）

| 文件 | 内容 | 预计时间 |
|------|------|---------|
| [01-asyncio.md](week3-async-and-web/01-asyncio.md) | async/await、事件循环、Task | 3h |
| [02-http-clients.md](week3-async-and-web/02-http-clients.md) | requests、httpx、aiohttp | 2h |
| [03-fastapi.md](week3-async-and-web/03-fastapi.md) | FastAPI 路由、请求体、依赖注入 | 3h |
| [04-json-and-file-io.md](week3-async-and-web/04-json-and-file-io.md) | JSON 处理与文件 I/O | 1.5h |
| [05-logging.md](week3-async-and-web/05-logging.md) | logging、loguru | 1.5h |
| [06-jupyter-notebook.md](week3-async-and-web/06-jupyter-notebook.md) | Jupyter Notebook 使用 | 1h |
| [练习题](week3-async-and-web/exercises/week3-exercises.md) | FastAPI 服务开发 | 5h |

## ✅ 里程碑检查清单

### 第一周末检查

- [ ] 已安装 Python 3.11+ 并配置好 IDE
- [ ] 能够用 Python 编写基本的变量操作、循环和函数
- [ ] 理解 Python list/dict 与 Java ArrayList/HashMap 的区别
- [ ] 能用 Python 类实现简单的继承和方法覆写
- [ ] 完成第一周所有练习题

### 第二周末检查

- [ ] 理解并能使用生成器（yield）优化内存
- [ ] 能编写简单的装饰器（如计时、日志装饰器）
- [ ] 能用 `with` 语句管理文件和网络资源
- [ ] 能为 Python 函数添加类型提示
- [ ] 能用 Pydantic 定义数据模型并进行校验
- [ ] 掌握 poetry 或 uv 包管理工具
- [ ] 完成第二周所有练习题

### 第三周末检查

- [ ] 理解事件循环，能编写 async/await 异步代码
- [ ] 能用 httpx 或 aiohttp 发起异步 HTTP 请求
- [ ] 能用 FastAPI 搭建包含 CRUD 接口的 REST API
- [ ] 能正确处理 JSON 序列化和文件读写
- [ ] 能配置结构化日志输出
- [ ] 完成第三周综合实战练习

### 阶段一总结检查

- [ ] 完成所有 16 个知识点文件的学习
- [ ] 完成 3 周共 12 道以上练习题
- [ ] 能够独立用 Python 实现一个简单的 REST API 服务
- [ ] 理解 Python 与 Java 的核心编程范式差异
- [ ] **准备好进入阶段二：LLM 基础**

## 📚 推荐资源

详细资源列表请查看：[推荐学习资源](resources/recommended-resources.md)

**快速参考：**
- [Python 官方文档](https://docs.python.org/zh-cn/3/)
- [Real Python 教程](https://realpython.com/)
- [FastAPI 官方文档](https://fastapi.tiangolo.com/zh/)

## 💡 给 Java 程序员的建议

1. **不要用 Java 的思维写 Python**：Python 有自己的习惯写法（Pythonic），多参考官方风格指南 PEP 8
2. **多用 REPL**：Python 交互式解释器非常适合快速验证想法，善用它
3. **理解动态类型**：虽然 Python 是动态类型，但从第二周起就应使用类型提示（Type Hints），养成良好习惯
4. **不要纠结于性能**：Python 在 AI Agent 开发中主要承担"胶水层"角色，性能瓶颈通常在底层库（NumPy、PyTorch 等都是 C/C++ 实现）
5. **拥抱生态系统**：Python 的 AI 生态是无可替代的优势，学会找包、用包比什么都重要
