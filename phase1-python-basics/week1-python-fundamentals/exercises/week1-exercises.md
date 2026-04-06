# 第一周练习题：用 Python 重写 Java 小项目

> 通过将熟悉的 Java 代码用 Python 重写，快速掌握 Python 语法和惯用写法

## 练习题总览

| 题号 | 题目 | 难度 | 考察知识点 |
|------|------|------|-----------|
| 1 | 学生成绩管理 | ⭐ | 变量、列表、字典、循环 |
| 2 | 字符串处理工具 | ⭐⭐ | 字符串操作、列表推导式 |
| 3 | 简单计算器 | ⭐⭐ | 函数、异常处理、字典 |
| 4 | 图书馆管理系统 | ⭐⭐⭐ | 类、继承、列表操作 |
| 5 | 🎯 综合实战：员工管理系统 | ⭐⭐⭐⭐ | 综合所有知识点 |

---

## 练习一：学生成绩管理

### 题目描述

用 Python 实现一个简单的学生成绩管理程序，完成以下功能：
1. 存储若干学生的姓名和成绩（用字典）
2. 计算平均分
3. 找出最高分和最低分学生
4. 按成绩从高到低排序并输出

### 输入输出示例

```python
# 输入数据
students = {
    "Alice": 85,
    "Bob": 92,
    "Carol": 78,
    "Dave": 95,
    "Eve": 67
}

# 期望输出
# 平均分：83.40
# 最高分：Dave (95分)
# 最低分：Eve (67分)
# 成绩排名：
# 1. Dave: 95
# 2. Bob: 92
# 3. Alice: 85
# 4. Carol: 78
# 5. Eve: 67
```

### 提示

- 使用 `sum()` 和 `len()` 计算平均分
- 使用 `max()` 和 `min()` 配合 `key` 参数找最高/最低分
- 使用 `sorted()` 排序，注意 `reverse=True` 参数

### 参考解答

```python
def analyze_grades(students: dict[str, int]) -> None:
    """分析学生成绩并输出报告"""
    
    if not students:
        print("没有学生数据")
        return
    
    # 计算平均分
    avg = sum(students.values()) / len(students)
    print(f"平均分：{avg:.2f}")
    
    # 最高分和最低分
    best = max(students, key=students.get)
    worst = min(students, key=students.get)
    print(f"最高分：{best} ({students[best]}分)")
    print(f"最低分：{worst} ({students[worst]}分)")
    
    # 按成绩排序
    print("成绩排名：")
    sorted_students = sorted(students.items(), key=lambda x: x[1], reverse=True)
    for rank, (name, score) in enumerate(sorted_students, 1):
        print(f"{rank}. {name}: {score}")


# 测试
students = {"Alice": 85, "Bob": 92, "Carol": 78, "Dave": 95, "Eve": 67}
analyze_grades(students)
```

---

## 练习二：字符串处理工具

### 题目描述

实现以下字符串处理函数：
1. `is_palindrome(s)`：判断字符串是否是回文（忽略大小写和空格）
2. `word_count(text)`：统计文本中每个单词出现的次数，返回字典
3. `camel_to_snake(name)`：将驼峰命名转换为下划线命名（如 `camelCase` → `camel_case`）

### 输入输出示例

```python
# 回文检测
is_palindrome("racecar")      # True
is_palindrome("A man a plan a canal Panama")  # True
is_palindrome("hello")        # False

# 单词计数
word_count("the cat sat on the mat")
# {"the": 2, "cat": 1, "sat": 1, "on": 1, "mat": 1}

# 驼峰转下划线
camel_to_snake("camelCase")        # "camel_case"
camel_to_snake("myVariableName")   # "my_variable_name"
camel_to_snake("getHTTPResponse")  # "get_h_t_t_p_response"
```

### 提示

- 回文：清理字符串后用切片 `[::-1]` 反转
- 单词计数：用 `split()` 分词，用字典或 `Counter` 统计
- 驼峰转下划线：遍历字符，遇到大写字母前加 `_`

### 参考解答

```python
def is_palindrome(s: str) -> bool:
    """判断是否为回文（忽略大小写和非字母字符）"""
    # 只保留字母并转小写
    cleaned = "".join(c.lower() for c in s if c.isalnum())
    return cleaned == cleaned[::-1]


def word_count(text: str) -> dict[str, int]:
    """统计单词出现频率"""
    from collections import Counter
    words = text.lower().split()
    return dict(Counter(words))


def camel_to_snake(name: str) -> str:
    """驼峰命名转下划线命名"""
    result = []
    for i, char in enumerate(name):
        if char.isupper() and i > 0:
            result.append("_")
        result.append(char.lower())
    return "".join(result)


# 或用正则表达式（更简洁）
import re

def camel_to_snake_regex(name: str) -> str:
    # 在大写字母前插入下划线
    s = re.sub("([A-Z]+)([A-Z][a-z])", r"\1_\2", name)
    s = re.sub("([a-z])([A-Z])", r"\1_\2", s)
    return s.lower()


# 测试
print(is_palindrome("racecar"))   # True
print(is_palindrome("A man a plan a canal Panama"))  # True
print(word_count("the cat sat on the mat"))
print(camel_to_snake("myVariableName"))  # my_variable_name
```

---

## 练习三：简单计算器

### 题目描述

实现一个支持以下运算的计算器：
- 基本运算：`+`、`-`、`*`、`/`
- 扩展运算：`//`（整除）、`%`（取余）、`**`（幂运算）
- 要求：除以零时给出友好提示，操作符不合法时给出提示

### 输入输出示例

```python
calculate(10, "+", 5)   # 15
calculate(10, "/", 0)   # "错误：除数不能为零"
calculate(10, "?", 5)   # "错误：不支持的运算符 '?'"
calculate(2, "**", 10)  # 1024
```

### 提示

- 用字典存储运算符到函数的映射（比 if-elif 链更 Pythonic）
- 使用 `operator` 模块中的函数
- 考虑用 `try/except` 处理异常

### 参考解答

```python
import operator
from typing import Union

def calculate(a: float, op: str, b: float) -> Union[float, str]:
    """执行四则运算及扩展运算"""
    
    operations = {
        "+": operator.add,
        "-": operator.sub,
        "*": operator.mul,
        "/": operator.truediv,
        "//": operator.floordiv,
        "%": operator.mod,
        "**": operator.pow,
    }
    
    if op not in operations:
        return f"错误：不支持的运算符 '{op}'"
    
    if op in ("/", "//", "%") and b == 0:
        return "错误：除数不能为零"
    
    try:
        return operations[op](a, b)
    except Exception as e:
        return f"错误：{e}"


# 测试
print(calculate(10, "+", 5))    # 15
print(calculate(10, "/", 3))    # 3.3333...
print(calculate(10, "//", 3))   # 3
print(calculate(10, "/", 0))    # 错误：除数不能为零
print(calculate(2, "**", 10))   # 1024
print(calculate(10, "?", 5))    # 错误：不支持的运算符 '?'
```

---

## 练习四：图书馆管理系统

### 题目描述

用 Python 类实现一个简单的图书馆管理系统：

1. `Book` 类：属性有 `title`、`author`、`isbn`、`available`（是否可借）
2. `Library` 类：管理书籍列表，实现以下方法：
   - `add_book(book)`: 添加书籍
   - `borrow_book(isbn)`: 借书（若不存在或已被借出则提示）
   - `return_book(isbn)`: 还书
   - `search_by_author(author)`: 按作者搜索
   - `available_books()`: 返回所有可借书籍

### 输入输出示例

```python
library = Library()
library.add_book(Book("Python 编程", "Guido", "978-001"))
library.add_book(Book("流畅的Python", "Ramalho", "978-002"))
library.add_book(Book("Python Cookbook", "Beazley", "978-003"))

library.borrow_book("978-001")   # "已借出：Python 编程"
library.borrow_book("978-001")   # "该书已被借出：Python 编程"
library.return_book("978-001")   # "已归还：Python 编程"

books = library.search_by_author("Ramalho")
# [Book(title='流畅的Python', author='Ramalho', ...)]
```

### 参考解答

```python
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class Book:
    title: str
    author: str
    isbn: str
    available: bool = True
    
    def __str__(self) -> str:
        status = "可借" if self.available else "已借出"
        return f"《{self.title}》by {self.author} [{status}]"


class Library:
    def __init__(self):
        self._books: dict[str, Book] = {}  # ISBN -> Book
    
    def add_book(self, book: Book) -> None:
        self._books[book.isbn] = book
        print(f"已添加：{book.title}")
    
    def borrow_book(self, isbn: str) -> str:
        book = self._books.get(isbn)
        if not book:
            return f"未找到 ISBN 为 {isbn} 的书籍"
        if not book.available:
            return f"该书已被借出：{book.title}"
        book.available = False
        return f"已借出：{book.title}"
    
    def return_book(self, isbn: str) -> str:
        book = self._books.get(isbn)
        if not book:
            return f"未找到 ISBN 为 {isbn} 的书籍"
        if book.available:
            return f"该书未被借出：{book.title}"
        book.available = True
        return f"已归还：{book.title}"
    
    def search_by_author(self, author: str) -> list[Book]:
        return [b for b in self._books.values() if author.lower() in b.author.lower()]
    
    def available_books(self) -> list[Book]:
        return [b for b in self._books.values() if b.available]
    
    def __len__(self) -> int:
        return len(self._books)
    
    def __str__(self) -> str:
        return f"图书馆（共 {len(self)} 本书，{len(self.available_books())} 本可借）"


# 测试
library = Library()
library.add_book(Book("Python 编程", "Guido", "978-001"))
library.add_book(Book("流畅的Python", "Ramalho", "978-002"))
library.add_book(Book("Python Cookbook", "Beazley", "978-003"))

print(library.borrow_book("978-001"))  # 已借出：Python 编程
print(library.borrow_book("978-001"))  # 该书已被借出：Python 编程
print(library.return_book("978-001"))  # 已归还：Python 编程
print(library)                          # 图书馆（共 3 本书，3 本可借）

books = library.search_by_author("Ramalho")
for book in books:
    print(book)
```

---

## 练习五：🎯 综合实战 - 员工管理系统

### 题目描述

实现一个完整的员工管理系统，综合运用本周所有知识点：

**需求：**
1. `Employee` 基类：姓名、工号、部门、基本工资
2. `Developer` 子类：额外属性技术栈（列表），`bonus` 按绩效级别（A/B/C）计算奖金
3. `Manager` 子类：额外属性管辖人数，奖金 = 基本工资 × 20%
4. `Department` 类：管理员工列表，支持：
   - 添加/删除员工
   - 计算部门总工资
   - 按工资排序
   - 查找指定技能的开发者
   - 生成部门报告

**约束：**
- 工号不能重复
- 工资必须大于 0
- 使用类型提示
- 实现 `__str__`、`__repr__`

### 输入输出示例

```python
dev1 = Developer("Alice", "D001", "工程部", 15000, ["Python", "FastAPI"], "A")
dev2 = Developer("Bob", "D002", "工程部", 12000, ["Java", "Spring"], "B")
mgr = Manager("Carol", "M001", "工程部", 20000, 5)

dept = Department("工程部")
dept.add_employee(dev1)
dept.add_employee(dev2)
dept.add_employee(mgr)

print(dept.total_salary())  # 47000 + 奖金...
print(dept.find_by_skill("Python"))  # [dev1]
dept.print_report()
# ======= 工程部报告 =======
# 员工数量: 3
# 总薪资支出: ¥XX,XXX
# ...
```

### 提示

- `Developer` 的奖金：A 级 = 工资 × 30%，B 级 = 工资 × 20%，C 级 = 工资 × 10%
- 使用 `@property` 实现薪资验证
- 使用列表推导式实现搜索功能
- 使用 `sorted()` 和 `key` 参数实现排序

### 参考解答

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Literal


class Employee:
    """员工基类"""
    
    BONUS_RATES: dict[str, float] = {}  # 子类覆盖
    
    def __init__(self, name: str, emp_id: str, department: str, base_salary: float):
        self.name = name
        self.emp_id = emp_id
        self.department = department
        self.base_salary = base_salary  # 调用 setter
    
    @property
    def base_salary(self) -> float:
        return self._base_salary
    
    @base_salary.setter
    def base_salary(self, value: float) -> None:
        if value <= 0:
            raise ValueError(f"工资必须大于 0，当前值：{value}")
        self._base_salary = value
    
    def bonus(self) -> float:
        return 0.0
    
    def total_salary(self) -> float:
        return self.base_salary + self.bonus()
    
    def __str__(self) -> str:
        return f"{self.name}({self.emp_id}) - {self.department} - ¥{self.total_salary():,.0f}"
    
    def __repr__(self) -> str:
        return f"{self.__class__.__name__}(name={self.name!r}, emp_id={self.emp_id!r})"


class Developer(Employee):
    """开发人员"""
    
    BONUS_RATES = {"A": 0.30, "B": 0.20, "C": 0.10}
    
    def __init__(
        self,
        name: str,
        emp_id: str,
        department: str,
        base_salary: float,
        skills: list[str],
        performance: Literal["A", "B", "C"] = "B",
    ):
        super().__init__(name, emp_id, department, base_salary)
        self.skills = skills
        self.performance = performance
    
    def bonus(self) -> float:
        rate = self.BONUS_RATES.get(self.performance, 0)
        return self.base_salary * rate
    
    def has_skill(self, skill: str) -> bool:
        return skill.lower() in [s.lower() for s in self.skills]


class Manager(Employee):
    """管理人员"""
    
    BONUS_RATE = 0.20
    
    def __init__(
        self,
        name: str,
        emp_id: str,
        department: str,
        base_salary: float,
        direct_reports: int,
    ):
        super().__init__(name, emp_id, department, base_salary)
        self.direct_reports = direct_reports
    
    def bonus(self) -> float:
        return self.base_salary * self.BONUS_RATE


class Department:
    """部门类"""
    
    def __init__(self, name: str):
        self.name = name
        self._employees: dict[str, Employee] = {}  # emp_id -> Employee
    
    def add_employee(self, employee: Employee) -> None:
        if employee.emp_id in self._employees:
            raise ValueError(f"工号 {employee.emp_id} 已存在")
        self._employees[employee.emp_id] = employee
        print(f"已添加员工：{employee.name}")
    
    def remove_employee(self, emp_id: str) -> None:
        if emp_id not in self._employees:
            raise KeyError(f"未找到工号 {emp_id}")
        name = self._employees.pop(emp_id).name
        print(f"已移除员工：{name}")
    
    def total_salary(self) -> float:
        return sum(emp.total_salary() for emp in self._employees.values())
    
    def sorted_by_salary(self, reverse: bool = True) -> list[Employee]:
        return sorted(self._employees.values(), key=lambda e: e.total_salary(), reverse=reverse)
    
    def find_by_skill(self, skill: str) -> list[Developer]:
        return [
            emp for emp in self._employees.values()
            if isinstance(emp, Developer) and emp.has_skill(skill)
        ]
    
    def print_report(self) -> None:
        print(f"\n{'='*30}")
        print(f"  {self.name} 部门报告")
        print(f"{'='*30}")
        print(f"员工数量：{len(self._employees)}")
        print(f"总薪资支出：¥{self.total_salary():,.0f}")
        print("\n按薪资排序：")
        for i, emp in enumerate(self.sorted_by_salary(), 1):
            print(f"  {i}. {emp}")
        print(f"{'='*30}\n")
    
    def __len__(self) -> int:
        return len(self._employees)


# ===== 测试 =====
if __name__ == "__main__":
    dev1 = Developer("Alice", "D001", "工程部", 15000, ["Python", "FastAPI", "LangChain"], "A")
    dev2 = Developer("Bob", "D002", "工程部", 12000, ["Java", "Spring", "Python"], "B")
    dev3 = Developer("Dave", "D003", "工程部", 10000, ["JavaScript", "React"], "C")
    mgr = Manager("Carol", "M001", "工程部", 20000, 3)
    
    dept = Department("工程部")
    for emp in [dev1, dev2, dev3, mgr]:
        dept.add_employee(emp)
    
    print("\nPython 开发者：")
    for dev in dept.find_by_skill("Python"):
        print(f"  - {dev.name}: {dev.skills}")
    
    dept.print_report()
    
    # 测试验证
    try:
        bad_emp = Developer("Eve", "D004", "工程部", -1000, [], "A")
    except ValueError as e:
        print(f"验证错误（预期）：{e}")
```

---

## 本周总结

通过这 5 道练习题，你应该已经掌握了：

- ✅ Python 字典、列表的常用操作
- ✅ 字符串处理和推导式
- ✅ 函数定义和异常处理
- ✅ 类、继承、属性（`@property`）
- ✅ `@dataclass` 装饰器
- ✅ 类型提示（Type Hints）

**如果你能独立完成练习五（不看答案），说明你已经具备了进入第二周的基础！**
