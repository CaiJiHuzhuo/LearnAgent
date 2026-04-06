# Python 基础语法

> 面向 Java 开发者的 Python 语法快速入门，重点讲解与 Java 的差异

## 概述

Python 和 Java 都是面向对象的语言，但 Python 的语法更简洁。有 Java 基础的开发者学 Python，最快的方法是找到**对应关系**，而不是从零开始。

## 1. 变量与数据类型

### 变量声明

```python
# Python：无需声明类型，直接赋值
name = "Alice"           # 字符串（str）
age = 25                 # 整数（int）
height = 1.75            # 浮点数（float）
is_active = True         # 布尔值（bool），注意首字母大写！
nothing = None           # 空值（类比 Java 的 null）

# Python 支持多重赋值
x, y, z = 1, 2, 3
a = b = c = 0

# 查看类型
print(type(name))        # <class 'str'>
print(type(age))         # <class 'int'>
print(isinstance(age, int))  # True
```

```java
// Java 对比：必须声明类型
String name = "Alice";
int age = 25;
double height = 1.75;
boolean isActive = true;
Object nothing = null;
```

### 基本数据类型对比

| Python 类型 | Java 类型 | 说明 |
|------------|----------|------|
| `int` | `int` / `long` / `BigInteger` | Python int 无大小限制！ |
| `float` | `double` | Python 默认 64 位浮点 |
| `str` | `String` | 不可变，支持 Unicode |
| `bool` | `boolean` | `True`/`False`（首字母大写） |
| `None` | `null` | 空值 |
| `bytes` | `byte[]` | 字节序列 |
| `complex` | 无直接对应 | 复数（`3+4j`） |

### 字符串操作

Python 的字符串操作比 Java 简洁得多：

```python
# 字符串定义（单引号、双引号等价）
s1 = 'Hello'
s2 = "World"

# 多行字符串（类比 Java 15 的文本块）
sql = """
SELECT *
FROM users
WHERE active = 1
"""

# 字符串拼接
greeting = s1 + ", " + s2  # "Hello, World"

# f-string 格式化（Python 3.6+，推荐使用）
name = "Alice"
age = 25
msg = f"姓名：{name}，年龄：{age}"
print(msg)  # 姓名：Alice，年龄：25

# f-string 支持表达式
print(f"明年 {age + 1} 岁")  # 明年 26 岁
print(f"{name!r}")            # 'Alice'（repr 格式）
print(f"{3.14159:.2f}")       # 3.14（保留两位小数）

# 字符串方法（与 Java String 类似）
text = "  Hello, Python!  "
print(text.strip())             # "Hello, Python!"（去除首尾空白）
print(text.lower())             # "  hello, python!  "
print(text.upper())             # "  HELLO, PYTHON!  "
print(text.replace("Python", "World"))  # "  Hello, World!  "
print("Python" in text)         # True（包含检查）
parts = "a,b,c".split(",")      # ['a', 'b', 'c']
joined = ",".join(parts)        # "a,b,c"
```

## 2. 数字运算

```python
# 基本运算（大部分与 Java 相同）
print(10 + 3)   # 13
print(10 - 3)   # 7
print(10 * 3)   # 30
print(10 / 3)   # 3.3333...（注意：Python 3 中 / 是真除法！）
print(10 // 3)  # 3（整除，类比 Java 的 10 / 3）
print(10 % 3)   # 1（取余）
print(10 ** 3)  # 1000（幂运算，Java 用 Math.pow）

# 常见陷阱：Python 3 的除法
# Java: 10 / 3 = 3（整数除法）
# Python: 10 / 3 = 3.3333（浮点除法）
# Python 整除用 //：10 // 3 = 3

# 数学库
import math
print(math.sqrt(16))    # 4.0
print(math.floor(3.7))  # 3
print(math.ceil(3.2))   # 4
print(abs(-5))          # 5（内置函数，不需要 Math.abs）
```

## 3. 控制流

### if 语句

```python
# Python if 语句（注意冒号和缩进）
x = 10

if x > 0:
    print("正数")
elif x < 0:
    print("负数")
else:
    print("零")

# 单行条件表达式（三元运算符）
# Java: String result = x > 0 ? "正数" : "非正数";
result = "正数" if x > 0 else "非正数"
print(result)

# Python 没有 switch，但可用 match（Python 3.10+）
command = "quit"
match command:
    case "quit":
        print("退出")
    case "go north" | "go south":
        print("移动")
    case _:  # default
        print("未知命令")
```

### 循环

```python
# for 循环遍历序列（更像 Java 的 for-each）
fruits = ["苹果", "香蕉", "橙子"]
for fruit in fruits:
    print(fruit)

# 带索引的遍历（类比 Java 的普通 for 循环）
for i, fruit in enumerate(fruits):
    print(f"{i}: {fruit}")  # 0: 苹果

# range 生成数字序列（类比 Java 的 for(int i=0; i<n; i++)）
for i in range(5):           # 0, 1, 2, 3, 4
    print(i)

for i in range(1, 6):        # 1, 2, 3, 4, 5
    print(i)

for i in range(0, 10, 2):    # 0, 2, 4, 6, 8（步长为 2）
    print(i)

# while 循环（与 Java 相同）
count = 0
while count < 5:
    print(count)
    count += 1
    # Python 没有 count++，用 count += 1

# break 和 continue（与 Java 相同）
for i in range(10):
    if i == 3:
        continue  # 跳过 3
    if i == 7:
        break     # 退出循环
    print(i)      # 输出 0, 1, 2, 4, 5, 6

# for...else（Python 特有！循环正常结束时执行 else）
for i in range(5):
    if i == 10:
        break
else:
    print("循环完成，没有 break")  # 会执行
```

### Java vs Python 控制流对比

| Java | Python |
|------|--------|
| `for (int i = 0; i < n; i++)` | `for i in range(n)` |
| `for (String s : list)` | `for s in list` |
| `for (int i = 0; i < list.size(); i++)` | `for i, s in enumerate(list)` |
| `x > 0 ? a : b` | `a if x > 0 else b` |
| `switch` | `match`（Python 3.10+）|
| `do { } while(condition)` | `while True: ... if not condition: break` |

## 4. 函数

### 函数定义

```python
# 基本函数定义（无需声明返回类型）
def greet(name):
    return f"Hello, {name}!"

print(greet("Alice"))  # Hello, Alice!

# 带默认参数（类比 Java 的方法重载）
def greet(name, greeting="你好"):
    return f"{greeting}, {name}!"

print(greet("Alice"))           # 你好, Alice!
print(greet("Alice", "Hi"))     # Hi, Alice!

# 关键字参数（按名称传参，顺序不重要）
def create_user(name, age, city):
    return f"{name}, {age}岁, {city}"

# 可以按任意顺序传关键字参数
print(create_user(age=25, city="北京", name="Alice"))
```

### 可变参数

```python
# *args：可变数量位置参数（类比 Java 的可变参数 String... args）
def sum_all(*args):
    total = 0
    for num in args:
        total += num
    return total

print(sum_all(1, 2, 3))       # 6
print(sum_all(1, 2, 3, 4, 5)) # 15

# **kwargs：可变数量关键字参数（类比 Java 的 Map<String, Object>）
def print_info(**kwargs):
    for key, value in kwargs.items():
        print(f"{key}: {value}")

print_info(name="Alice", age=25, city="北京")
# name: Alice
# age: 25
# city: 北京

# 组合使用
def mixed(a, b, *args, key="default", **kwargs):
    print(f"a={a}, b={b}, args={args}, key={key}, kwargs={kwargs}")

mixed(1, 2, 3, 4, key="custom", x=10, y=20)
```

### Lambda 函数

```python
# Lambda：匿名函数（类比 Java 的 Lambda 表达式）
square = lambda x: x ** 2
print(square(5))  # 25

# 常用于排序等场景
students = [("Alice", 85), ("Bob", 92), ("Carol", 78)]

# 按成绩排序
students.sort(key=lambda s: s[1])
print(students)  # [('Carol', 78), ('Alice', 85), ('Bob', 92)]

# 降序
students.sort(key=lambda s: s[1], reverse=True)
```

### 函数是一等公民

```python
# Python 中函数可以作为参数传递（类比 Java 的函数式接口）
def apply(func, value):
    return func(value)

def double(x):
    return x * 2

print(apply(double, 5))      # 10
print(apply(lambda x: x ** 2, 5))  # 25

# 返回函数（闭包）
def make_multiplier(n):
    def multiplier(x):
        return x * n  # 引用了外部变量 n
    return multiplier

triple = make_multiplier(3)
print(triple(5))  # 15
```

## 5. 作用域（LEGB 规则）

```python
# Python 变量作用域：Local → Enclosing → Global → Built-in
x = 10  # 全局变量

def outer():
    y = 20  # outer 函数的局部变量
    
    def inner():
        z = 30  # inner 函数的局部变量
        print(x, y, z)  # 10 20 30（可以访问外层变量）
    
    inner()

outer()

# 修改全局变量需要 global 声明
count = 0

def increment():
    global count  # 声明要修改全局变量
    count += 1

increment()
print(count)  # 1

# 修改外层局部变量需要 nonlocal 声明
def counter():
    n = 0
    def add():
        nonlocal n
        n += 1
        return n
    return add

c = counter()
print(c())  # 1
print(c())  # 2
```

## 常见陷阱与注意事项

### 1. 缩进错误

```python
# ❌ 错误：混用空格和 Tab 会导致 IndentationError
# 下面的代码中第二行用了 Tab，第一行用了空格：
# if True:
#     print("使用4个空格")   <- 4个空格
#     print("使用Tab")       <- Tab键（看起来一样，但实际不同！）
# 运行时报错：IndentationError: inconsistent use of tabs and spaces

# ✅ 正确：统一使用 4 个空格（PEP 8 规范）
if True:
    print("使用4个空格")
    print("统一使用4个空格")
```

### 2. 整除 vs 浮点除法

```python
# ❌ 误以为 / 是整除
result = 7 / 2  # 3.5，不是 3！

# ✅ 需要整除用 //
result = 7 // 2  # 3
```

### 3. 可变默认参数陷阱

```python
# ❌ 错误：使用可变对象作为默认参数
def append_item(item, lst=[]):  # 默认参数只创建一次！
    lst.append(item)
    return lst

print(append_item(1))  # [1]
print(append_item(2))  # [1, 2]  <- 不是预期的 [2]！

# ✅ 正确：使用 None 作为默认值
def append_item(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst
```

### 4. 比较运算符

```python
# Python 的 == 比较值，is 比较身份（内存地址）
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)   # True（值相等）
print(a is b)   # False（不同对象）
print(a is c)   # True（同一对象）

# 注意：检查 None 用 is，不用 ==
x = None
if x is None:   # ✅ 正确
    print("x 是 None")
if x == None:   # ⚠️ 可以工作，但不是最佳实践
    print("x 是 None")
```

## 小结

| 概念 | Java | Python |
|------|------|--------|
| 变量声明 | `String name = "Alice"` | `name = "Alice"` |
| 字符串格式化 | `String.format()` | f-string: `f"Hello {name}"` |
| 整除 | `10 / 3 = 3` | `10 // 3 = 3` |
| 取幂 | `Math.pow(2, 10)` | `2 ** 10` |
| 空值 | `null` | `None` |
| 布尔值 | `true`/`false` | `True`/`False` |
| 三元运算 | `a ? b : c` | `b if a else c` |
| 可变参数 | `String... args` | `*args` |
| 命名参数 | 不支持（只能用 Builder 模式） | `func(key=value)` |
