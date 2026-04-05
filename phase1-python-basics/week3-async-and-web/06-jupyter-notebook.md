# Jupyter Notebook 使用指南

> 在 AI Agent 开发中使用 Jupyter Notebook 进行探索性开发和可视化

## 概述

Jupyter Notebook 是 AI/ML 领域最常用的交互式开发工具，特别适合：
- 探索性数据分析（EDA）
- 原型验证（快速测试想法）
- 可视化展示（分享研究结果）
- 学习和实验 LLM/Agent

对 Java 开发者来说，Jupyter Notebook 没有直接对应的工具，最接近的是在 IDE 中的 REPL/Scratch File，但 Notebook 能持久化代码和输出。

## 1. 安装与启动

```bash
# 安装
pip install jupyterlab  # 推荐（JupyterLab，更现代的界面）
pip install notebook    # 传统 Jupyter Notebook

# 启动 JupyterLab
jupyter lab

# 启动传统 Notebook
jupyter notebook

# 指定端口和主机
jupyter lab --port=8888 --ip=0.0.0.0

# VS Code 插件（推荐！不需要开浏览器）
# 在 VS Code 中安装 "Jupyter" 扩展即可
```

## 2. 核心概念

### 单元格类型

| 类型 | 说明 | 快捷键 |
|------|------|--------|
| Code | Python 代码 | `Esc + Y` |
| Markdown | 富文本 | `Esc + M` |
| Raw | 原始文本 | `Esc + R` |

### 常用快捷键

| 快捷键 | 作用 |
|--------|------|
| `Shift + Enter` | 运行当前单元格，跳到下一个 |
| `Ctrl + Enter` | 运行当前单元格，留在当前 |
| `Alt + Enter` | 运行当前单元格，在下方插入新单元格 |
| `A` (命令模式) | 在上方插入单元格 |
| `B` (命令模式) | 在下方插入单元格 |
| `DD` (命令模式) | 删除当前单元格 |
| `Ctrl + /` | 注释/取消注释 |
| `Ctrl + Z` | 撤销 |
| `Esc` | 进入命令模式 |
| `Enter` | 进入编辑模式 |

## 3. 基本使用

```python
# 单元格1：导入库
import asyncio
import json
from pathlib import Path
from datetime import datetime

# Jupyter 中可以直接显示对象，不需要 print
data = {"name": "Alice", "age": 25}
data  # 最后一行会自动显示
```

```python
# 单元格2：数据探索
users = [
    {"name": "Alice", "score": 85},
    {"name": "Bob", "score": 92},
    {"name": "Carol", "score": 78},
]

# 显示格式化的数据
for user in users:
    print(f"{user['name']}: {user['score']}")

# 使用 pandas 更方便（如果安装了）
try:
    import pandas as pd
    df = pd.DataFrame(users)
    df  # 在 Notebook 中显示为漂亮的表格
except ImportError:
    print("未安装 pandas")
```

```python
# 单元格3：魔法命令（Jupyter 特有）
# %timeit：测量代码执行时间
%timeit sum(range(1000000))

# %%timeit：测量整个单元格
%%timeit
total = 0
for i in range(1000000):
    total += i

# %time：单次计时
%time sorted([3, 1, 4, 1, 5, 9, 2, 6])

# %who：显示所有变量
%who

# %whos：显示所有变量（详细）
%whos

# %%bash：在单元格中运行 Shell 命令
%%bash
echo "当前目录：$(pwd)"
ls -la

# %load_ext：加载扩展
%load_ext autoreload
%autoreload 2  # 自动重新加载修改的模块（开发时很有用！）
```

## 4. 在 Notebook 中使用 asyncio

Jupyter 的事件循环已经在运行，可以直接 `await`：

```python
# 在 JupyterLab/Notebook 中，可以直接使用 await
import asyncio
import httpx

# 不需要 asyncio.run()！
async def fetch(url: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json()

# 直接 await（在 Jupyter 中有效）
result = await fetch("https://api.github.com/users/python")
print(result["login"])  # python

# 或者
result = await asyncio.gather(
    fetch("https://api.github.com/users/python"),
    fetch("https://api.github.com/users/requests"),
)
result
```

## 5. 可视化（简单示例）

```python
# 安装：pip install matplotlib
import matplotlib
matplotlib.use('Agg')  # 非交互模式
import matplotlib.pyplot as plt

# 折线图
scores = [85, 92, 78, 95, 67, 88]
names = ["Alice", "Bob", "Carol", "Dave", "Eve", "Frank"]

plt.figure(figsize=(10, 6))
plt.bar(names, scores, color="steelblue")
plt.title("学生成绩")
plt.xlabel("学生")
plt.ylabel("分数")
plt.ylim(0, 100)
plt.axhline(y=60, color="red", linestyle="--", label="及格线")
plt.legend()
plt.tight_layout()
plt.savefig("scores.png")  # 保存图片
plt.show()
```

```python
# 使用 plotly（交互式图表，更推荐）
# pip install plotly
import plotly.graph_objects as go

fig = go.Figure(data=[
    go.Bar(x=names, y=scores, marker_color="steelblue")
])
fig.update_layout(
    title="学生成绩分布",
    xaxis_title="学生",
    yaxis_title="分数",
    yaxis_range=[0, 100],
)
fig.show()  # 交互式图表（可以缩放、悬停查看数值）
```

## 6. Notebook 与 AI Agent 开发

```python
# 在 Notebook 中测试 LLM API（原型验证）
import os
from openai import AsyncOpenAI  # pip install openai

# 加载 API key
# %env OPENAI_API_KEY=sk-...  # 或从 .env 文件加载

client = AsyncOpenAI(api_key=os.getenv("OPENAI_API_KEY"))

async def chat(message: str) -> str:
    response = await client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[{"role": "user", "content": message}],
    )
    return response.choices[0].message.content

# 直接测试（Jupyter 支持直接 await）
result = await chat("用一句话解释 Python 的装饰器")
print(result)
```

```python
# 测试 RAG 管道
from langchain.text_splitter import RecursiveCharacterTextSplitter  # pip install langchain

text = """
Python 是一种解释型、高级、通用目的的编程语言。
它由 Guido van Rossum 设计，于 1991 年首次发布。
Python 的设计哲学强调代码的可读性，并且语法允许程序员比 C++ 或 Java 更少的代码行数来
表达概念。
"""

# 文本分块
splitter = RecursiveCharacterTextSplitter(chunk_size=100, chunk_overlap=20)
chunks = splitter.split_text(text)
print(f"分成了 {len(chunks)} 个块：")
for i, chunk in enumerate(chunks):
    print(f"\n块 {i+1}:")
    print(chunk)
```

## 7. Jupyter 使用技巧

### 重新加载模块（开发时）

```python
# 如果你修改了 .py 文件，需要重新加载
%load_ext autoreload
%autoreload 2  # 每次执行单元格前自动重新加载修改的模块

# 或手动重新加载
import importlib
import my_module
importlib.reload(my_module)
```

### 调试技巧

```python
# 使用 pdb 调试
import pdb

def buggy_function(x):
    pdb.set_trace()  # 在这里暂停，进入调试模式
    result = x * 2
    return result

buggy_function(5)

# VS Code 的 Jupyter 支持直接在 Notebook 中设置断点
```

### 显示进度条

```python
# pip install tqdm
from tqdm.notebook import tqdm  # notebook 版本有进度条动画
import time

for i in tqdm(range(100)):
    time.sleep(0.05)  # 模拟耗时操作
```

### 导出 Notebook

```bash
# 导出为 Python 脚本
jupyter nbconvert --to script notebook.ipynb

# 导出为 HTML（分享结果）
jupyter nbconvert --to html notebook.ipynb

# 导出为 PDF
jupyter nbconvert --to pdf notebook.ipynb
```

## 8. nbformat：程序化操作 Notebook

```python
# 读取 Notebook 文件（.ipynb 是 JSON 格式）
import json
from pathlib import Path

notebook_path = Path("my_notebook.ipynb")
if notebook_path.exists():
    notebook = json.loads(notebook_path.read_text())
    
    print(f"Notebook 格式版本：{notebook['nbformat']}")
    print(f"单元格数量：{len(notebook['cells'])}")
    
    for cell in notebook['cells']:
        print(f"\n[{cell['cell_type']}]")
        print("".join(cell['source']))
```

## 常见陷阱与注意事项

### 1. 执行顺序问题

```python
# ❌ 可能的问题：单元格乱序执行导致变量状态不一致
# 单元格2 依赖单元格1 的变量，但如果先执行 3 再执行 1 和 2，可能出错

# ✅ 建议：
# - 始终"Restart Kernel and Run All"来验证 Notebook 是否能完整运行
# - 不要依赖手动执行顺序
```

### 2. 大量内存占用

```python
# ❌ 在 Notebook 中加载大数据集后忘记清理
large_df = pd.read_csv("huge_file.csv")  # 占用 GB 内存

# ✅ 用完后及时释放
del large_df
import gc
gc.collect()
```

### 3. Notebook 版本控制

```bash
# .ipynb 文件包含输出，提交 Git 时可能很大
# ✅ 提交前清理输出
jupyter nbconvert --clear-output --inplace notebook.ipynb

# 或使用 nbstripout 自动清理
pip install nbstripout
nbstripout --install  # 自动在 git commit 时清理输出
```

## 小结

| 功能 | 命令/方法 |
|------|---------|
| 安装 | `pip install jupyterlab` |
| 启动 | `jupyter lab` |
| 运行单元格 | `Shift + Enter` |
| 魔法命令 | `%timeit`, `%%bash`, `%who` 等 |
| asyncio 支持 | 直接 `await`（无需 `asyncio.run()`） |
| 进度条 | `from tqdm.notebook import tqdm` |
| 重新加载模块 | `%autoreload 2` |
| 导出脚本 | `jupyter nbconvert --to script notebook.ipynb` |

**在 AI Agent 开发中的主要用途**：
- 快速测试 LLM API 调用
- 探索和调试 RAG 管道
- 可视化 Agent 对话历史
- 数据集探索和标注
- 分享实验结果（`.ipynb` 可以直接在 GitHub 渲染）
