# Python 环境搭建

> 本文介绍如何搭建 Python 开发环境，包括 pyenv、conda、venv 的使用，以及如何配置 IDE。

## 概述

Java 开发者通常使用 JDK + Maven/Gradle 管理环境和依赖。Python 的工具链略有不同：

| 概念 | Java | Python |
|------|------|--------|
| 运行时版本管理 | SDKMAN、jenv | **pyenv**、conda |
| 虚拟环境（隔离项目依赖） | 无直接对应（Maven/Gradle 各管各的） | **venv**、virtualenv、conda env |
| 包管理器 | Maven、Gradle | **pip**、poetry、uv |
| 项目依赖文件 | pom.xml、build.gradle | requirements.txt、pyproject.toml |

## 1. Python 版本管理：pyenv

### 什么是 pyenv？

类似于 Java 的 SDKMAN，pyenv 允许你在同一台机器上安装和切换多个 Python 版本。

### 安装 pyenv

**macOS / Linux：**

```bash
# 安装 pyenv（官方推荐方式）
curl https://pyenv.run | bash

# 配置 shell（添加到 ~/.bashrc 或 ~/.zshrc）
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"

# 重新加载 shell 配置
source ~/.bashrc  # 或 source ~/.zshrc
```

**Windows（推荐使用 pyenv-win）：**

```powershell
# 使用 PowerShell 安装
Invoke-WebRequest -UseBasicParsing -Uri "https://raw.githubusercontent.com/pyenv-win/pyenv-win/master/pyenv-win/install-pyenv-win.ps1" -OutFile "./install-pyenv-win.ps1"; &"./install-pyenv-win.ps1"
```

### 常用 pyenv 命令

```bash
# 查看可安装的 Python 版本
pyenv install --list | grep "3.11"

# 安装 Python 3.11.9（推荐用于 AI 开发）
pyenv install 3.11.9

# 设置全局默认版本
pyenv global 3.11.9

# 在当前目录设置局部版本（会生成 .python-version 文件）
pyenv local 3.11.9

# 查看已安装版本
pyenv versions

# 查看当前使用版本
python --version
```

## 2. 虚拟环境：venv

### 为什么需要虚拟环境？

类比：Java 的 Maven 会把每个项目的依赖下载到本地仓库（`~/.m2`），不同项目可以使用不同版本的同一个库。

Python 的 pip 默认安装到全局，如果项目 A 需要 `requests==2.28.0`，项目 B 需要 `requests==2.31.0`，就会冲突。**虚拟环境**解决了这个问题——每个项目有自己独立的 Python 环境。

### 创建和使用 venv

```bash
# 1. 在项目目录下创建虚拟环境（.venv 是惯例名称）
python -m venv .venv

# 2. 激活虚拟环境
# macOS / Linux
source .venv/bin/activate

# Windows（PowerShell）
.venv\Scripts\Activate.ps1

# Windows（CMD）
.venv\Scripts\activate.bat

# 3. 激活后，命令行前缀会变成 (.venv)
(.venv) $ python --version  # 使用虚拟环境内的 Python

# 4. 在虚拟环境中安装包
pip install requests

# 5. 导出依赖列表
pip freeze > requirements.txt

# 6. 从依赖文件安装（类似 mvn install）
pip install -r requirements.txt

# 7. 退出虚拟环境
deactivate
```

### 常见陷阱

```bash
# ❌ 错误：忘记激活虚拟环境，包装到了全局
pip install requests  # 如果没激活 .venv，装到全局了

# ✅ 正确：确认虚拟环境已激活
which python  # 应该显示 .venv/bin/python 的路径
```

## 3. 包管理：pip 基础

```bash
# 安装最新版本
pip install fastapi

# 安装指定版本
pip install fastapi==0.104.1

# 安装带额外功能的包
pip install "fastapi[all]"

# 升级包
pip install --upgrade fastapi

# 卸载包
pip uninstall fastapi

# 查看已安装的包
pip list

# 查看包信息
pip show fastapi

# 配置国内镜像（推荐，加速下载）
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

## 4. conda 环境（数据科学场景）

conda 是 Anaconda 生态的包管理和环境管理工具，特别适合数据科学场景（可以安装非 Python 包，如 CUDA 相关工具）。

### 安装 Miniconda（推荐轻量版）

前往 [Miniconda 官网](https://docs.conda.io/en/latest/miniconda.html) 下载对应系统的安装包。

### 常用 conda 命令

```bash
# 创建 conda 环境
conda create -n myenv python=3.11

# 激活环境
conda activate myenv

# 安装包
conda install numpy pandas

# 也可以混用 pip
pip install fastapi

# 查看所有环境
conda env list

# 导出环境
conda env export > environment.yml

# 从文件恢复环境
conda env create -f environment.yml

# 退出环境
conda deactivate
```

### pyenv vs conda 如何选择？

| 场景 | 推荐工具 |
|------|---------|
| 纯 Python 开发（Web、Agent） | **pyenv + venv** |
| 数据科学、机器学习 | **conda** |
| 需要非 Python 二进制依赖（CUDA、MKL） | **conda** |
| CI/CD 流水线 | **pyenv + venv**（更轻量） |

## 5. IDE 配置

### VS Code（推荐）

1. 安装 [Python 扩展](https://marketplace.visualstudio.com/items?itemName=ms-python.python)
2. 安装 [Pylance 扩展](https://marketplace.visualstudio.com/items?itemName=ms-python.vscode-pylance)（智能提示）
3. 按 `Ctrl+Shift+P`，输入 `Python: Select Interpreter`，选择虚拟环境的 Python

```json
// .vscode/settings.json（项目配置）
{
    "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
    "editor.formatOnSave": true,
    "[python]": {
        "editor.defaultFormatter": "charliermarsh.ruff"
    }
}
```

### PyCharm

1. 打开 `File > Settings > Project > Python Interpreter`
2. 点击齿轮图标 > `Add Interpreter`
3. 选择 `Existing Environment`，指向 `.venv/bin/python`

## 6. 验证安装

运行以下脚本验证环境是否正确：

```python
# 保存为 check_env.py，运行 python check_env.py
import sys
import platform

print(f"Python 版本: {sys.version}")
print(f"Python 路径: {sys.executable}")
print(f"操作系统: {platform.system()} {platform.release()}")

# 检查 Python 版本是否满足要求
major, minor = sys.version_info[:2]
if major == 3 and minor >= 11:
    print("✅ Python 版本符合要求（3.11+）")
else:
    print(f"❌ Python 版本过低！当前 {major}.{minor}，需要 3.11+")
```

## 常见陷阱与注意事项

1. **Windows 路径问题**：Windows 上 Python 可执行文件叫 `python`，但 Linux/macOS 可能是 `python3`。建议统一使用 `python3`（如果系统同时装了 Python 2）。

2. **不要用 sudo pip install**：这会装到系统 Python，可能破坏系统工具。始终使用虚拟环境！

3. **`.venv` 不要提交到 Git**：虚拟环境不应提交到版本控制，在 `.gitignore` 中添加 `.venv/`。

4. **国内网络问题**：如果安装包很慢，配置清华镜像源：
   ```bash
   pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
   ```

## 小结

| 工具 | 用途 | 类比 Java |
|------|------|----------|
| pyenv | 管理多个 Python 版本 | SDKMAN |
| venv | 为项目创建隔离的 Python 环境 | 项目级依赖隔离 |
| pip | 安装 Python 包 | Maven/Gradle |
| conda | 综合的包和环境管理（数据科学） | SDKMAN + Maven |

**推荐的日常工作流**：
1. 新项目：`python -m venv .venv && source .venv/bin/activate`
2. 安装依赖：`pip install -r requirements.txt`
3. 开发完后导出：`pip freeze > requirements.txt`
