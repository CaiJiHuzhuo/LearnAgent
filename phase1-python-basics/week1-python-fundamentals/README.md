# 第一周：Python 基础入门

> 有 Java 基础的开发者，一周内掌握 Python 核心语法

## 🎯 本周学习目标

1. 搭建可用的 Python 开发环境
2. 掌握 Python 基础语法，理解与 Java 的核心差异
3. 熟练使用 Python 内置数据结构（list、dict、set、tuple）
4. 能用 Python 编写面向对象程序
5. 系统总结 Java 与 Python 的核心差异

## 📅 每日建议安排

| 天 | 上午（1-2h） | 下午（1-2h） | 晚上（1h） |
|----|------------|------------|----------|
| 周一 | [环境搭建](01-environment-setup.md) | [基础语法 - 变量与类型](02-basic-syntax.md) | 练习基础语法 |
| 周二 | [基础语法 - 控制流与函数](02-basic-syntax.md) | [数据结构 - list 与 tuple](03-data-structures.md) | 练习列表操作 |
| 周三 | [数据结构 - dict 与 set](03-data-structures.md) | [数据结构 - 推导式](03-data-structures.md) | 练习推导式 |
| 周四 | [面向对象 - 类与继承](04-oop.md) | [面向对象 - 魔术方法](04-oop.md) | 练习 OOP |
| 周五 | [Java vs Python 总结](05-java-vs-python.md) | 复习本周内容 | 开始练习题 |
| 周末 | [完成练习题](exercises/week1-exercises.md) | 整理笔记 | 预习第二周 |

## 📖 知识点列表

| 序号 | 知识点 | 文件 | 难度 | 预计时间 |
|------|--------|------|------|---------|
| 01 | Python 环境搭建（pyenv、conda、venv） | [01-environment-setup.md](01-environment-setup.md) | ⭐ | 2h |
| 02 | 基础语法（变量、数据类型、控制流、函数） | [02-basic-syntax.md](02-basic-syntax.md) | ⭐⭐ | 3h |
| 03 | 数据结构（list、dict、set、tuple、推导式） | [03-data-structures.md](03-data-structures.md) | ⭐⭐ | 3h |
| 04 | 面向对象编程（类、继承、魔术方法） | [04-oop.md](04-oop.md) | ⭐⭐⭐ | 3h |
| 05 | Java 与 Python 核心差异总结 | [05-java-vs-python.md](05-java-vs-python.md) | ⭐⭐ | 1h |
| - | 第一周练习题 | [exercises/week1-exercises.md](exercises/week1-exercises.md) | ⭐⭐⭐ | 4h |

## 💡 本周重点

### 最重要的概念转变

作为 Java 程序员，学 Python 最大的"思维转变"是：

1. **无需声明类型**（但推荐用 Type Hints）
   ```python
   # Java: String name = "Alice";
   name = "Alice"  # Python 直接赋值
   ```

2. **缩进替代大括号**
   ```python
   # Java: if (x > 0) { ... }
   if x > 0:
       print("正数")  # 缩进表示代码块
   ```

3. **一切皆对象**（连函数都是对象）
   ```python
   def greet(name):
       return f"Hello, {name}!"
   
   # 函数可以作为变量传递
   say_hello = greet
   print(say_hello("Alice"))  # Hello, Alice!
   ```

4. **鸭子类型**（不检查类型，只检查行为）
   ```python
   # 不需要接口，只需要有对应的方法
   class Duck:
       def quack(self):
           return "嘎嘎！"
   
   class Person:
       def quack(self):
           return "我在模仿鸭子！"
   
   def make_it_quack(obj):
       print(obj.quack())  # 不关心类型，只关心有没有 quack() 方法
   ```

## ✅ 本周检查清单

- [ ] Python 环境搭建完成，能在终端运行 `python --version`
- [ ] 理解 Python 缩进规则，不再犯缩进错误
- [ ] 能用 f-string 格式化字符串
- [ ] 掌握 list/dict/set/tuple 的基本操作
- [ ] 能写列表推导式
- [ ] 理解 Python 类与 Java 类的区别
- [ ] 知道什么是 `__init__`、`__str__`、`__repr__`
- [ ] 完成本周所有练习题

## 🚀 快速开始

```bash
# 1. 安装 Python（推荐 3.11+）
# macOS/Linux 推荐使用 pyenv
curl https://pyenv.run | bash
pyenv install 3.11.9
pyenv global 3.11.9

# 2. 验证安装
python --version  # Python 3.11.9
python -m pip --version

# 3. 创建第一周学习目录
mkdir week1-practice
cd week1-practice
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# 4. 打开 Python REPL 开始探索
python
```
