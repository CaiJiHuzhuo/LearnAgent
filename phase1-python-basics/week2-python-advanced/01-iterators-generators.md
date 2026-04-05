# 迭代器与生成器

> 理解 Python 的迭代器协议和生成器，实现高效的惰性数据处理

## 概述

生成器是 Python 中处理大量数据的利器，对于 AI Agent 开发中常见的流式输出（Streaming）场景尤为重要。Java 程序员可以类比 Java 8 的 `Stream<T>`，但生成器更轻量，使用更简单。

| 概念 | Java 对应 | Python |
|------|---------|--------|
| 迭代器 | `Iterator<T>` | 实现 `__iter__` 和 `__next__` 的对象 |
| 可迭代对象 | `Iterable<T>` | 实现 `__iter__` 的对象 |
| 生成器函数 | 无直接对应 | 包含 `yield` 的函数 |
| 生成器表达式 | `Stream.of(...).map(...)` | `(x for x in iterable)` |

## 1. 迭代器协议

```python
# Python 的迭代器协议：实现 __iter__ 和 __next__
class CountUp:
    """从 start 计数到 end 的迭代器"""
    
    def __init__(self, start: int, end: int):
        self.current = start
        self.end = end
    
    def __iter__(self):
        return self  # 迭代器返回自身
    
    def __next__(self) -> int:
        if self.current > self.end:
            raise StopIteration  # 通知循环结束
        value = self.current
        self.current += 1
        return value


# 使用迭代器
counter = CountUp(1, 5)
for n in counter:
    print(n)  # 1 2 3 4 5

# 手动迭代
counter2 = CountUp(1, 3)
print(next(counter2))  # 1
print(next(counter2))  # 2
print(next(counter2))  # 3
# print(next(counter2))  # StopIteration!

# 内置类型都是可迭代的
for char in "hello":     # str 是可迭代的
    print(char)
for item in [1, 2, 3]:  # list 是可迭代的
    print(item)
for key in {"a": 1}:    # dict 是可迭代的（遍历键）
    print(key)
```

## 2. 生成器函数

生成器函数用 `yield` 关键字替代 `return`，每次调用 `next()` 时从上次 `yield` 的地方继续执行：

```python
# 最简单的生成器
def count_up(start: int, end: int):
    """生成器版本的 CountUp，代码更简洁"""
    current = start
    while current <= end:
        yield current  # 暂停并返回当前值，下次继续
        current += 1


# 使用
for n in count_up(1, 5):
    print(n)  # 1 2 3 4 5

gen = count_up(1, 3)
print(next(gen))  # 1
print(next(gen))  # 2
print(next(gen))  # 3

# 生成器是惰性的：不会一次性生成所有值
def infinite_counter(start=0):
    """无限计数生成器（永不结束）"""
    n = start
    while True:
        yield n
        n += 1


counter = infinite_counter()
for n in counter:
    if n >= 5:
        break
    print(n)  # 0 1 2 3 4
```

### 生成器的实际应用

```python
import os

# 应用1：读取大文件（不会把整个文件加载进内存）
def read_large_file(filepath: str):
    """逐行读取大文件"""
    with open(filepath, encoding="utf-8") as f:
        for line in f:
            yield line.strip()

# 使用
for line in read_large_file("data.csv"):
    if line.startswith("#"):
        continue
    # 处理每一行...


# 应用2：处理数据管道
def parse_csv_line(line: str) -> list[str]:
    return line.split(",")

def filter_empty(lines):
    for line in lines:
        if line:  # 过滤空行
            yield line

def parse_lines(lines):
    for line in lines:
        yield parse_csv_line(line)

# 链式处理（惰性，不创建中间列表）
def process_file(filepath: str):
    lines = read_large_file(filepath)
    non_empty = filter_empty(lines)
    parsed = parse_lines(non_empty)
    return parsed


# 应用3：模拟流式输出（AI Agent 常用场景！）
import time

def stream_response(text: str, delay: float = 0.05):
    """模拟 LLM 流式输出"""
    for char in text:
        yield char
        time.sleep(delay)

# 模拟 ChatGPT 的流式输出
response = "你好！我是 AI 助手，很高兴见到你！"
for char in stream_response(response):
    print(char, end="", flush=True)
print()
```

## 3. yield from（委托生成器）

```python
# yield from 可以委托给另一个可迭代对象
def chain(*iterables):
    """连接多个可迭代对象（类似 itertools.chain）"""
    for it in iterables:
        yield from it  # 等价于：for item in it: yield item


list(chain([1, 2], [3, 4], [5, 6]))  # [1, 2, 3, 4, 5, 6]


# 递归生成器：遍历嵌套结构
def flatten(nested):
    """展平任意深度的嵌套列表"""
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)  # 递归委托
        else:
            yield item


nested = [1, [2, 3], [4, [5, 6]], 7]
print(list(flatten(nested)))  # [1, 2, 3, 4, 5, 6, 7]
```

## 4. 生成器表达式

```python
# 生成器表达式：用 () 而不是 []
# 不会立即生成所有元素，节省内存

# 计算大量数字的平方和（不占用大量内存）
total = sum(x**2 for x in range(1_000_000))  # 不创建中间列表

# 与列表推导式对比
squares_list = [x**2 for x in range(10)]       # 立即生成列表，占内存
squares_gen = (x**2 for x in range(10))        # 惰性生成器，几乎不占内存

# 生成器只能遍历一次！
gen = (x**2 for x in range(5))
print(list(gen))  # [0, 1, 4, 9, 16]
print(list(gen))  # []  <- 已经耗尽！

# 链式生成器表达式
data = [1, -2, 3, -4, 5, -6]
positive_squares = (x**2 for x in data if x > 0)
print(list(positive_squares))  # [1, 9, 25]
```

## 5. itertools 工具库

Python 内置的 `itertools` 提供了强大的迭代器工具：

```python
import itertools

# itertools.count：无限计数
counter = itertools.count(start=10, step=2)
print(list(itertools.islice(counter, 5)))  # [10, 12, 14, 16, 18]

# itertools.cycle：循环迭代
cycler = itertools.cycle(["A", "B", "C"])
print([next(cycler) for _ in range(7)])   # ['A', 'B', 'C', 'A', 'B', 'C', 'A']

# itertools.chain：连接多个迭代器
chained = itertools.chain([1, 2], [3, 4], [5, 6])
print(list(chained))  # [1, 2, 3, 4, 5, 6]

# itertools.islice：切片迭代器（不加载全部数据）
gen = (x**2 for x in itertools.count())
first_10 = list(itertools.islice(gen, 10))  # 取前 10 个
print(first_10)  # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# itertools.groupby：按键分组
data = [("A", 1), ("A", 2), ("B", 3), ("B", 4), ("C", 5)]
for key, group in itertools.groupby(data, key=lambda x: x[0]):
    print(f"{key}: {list(group)}")
# A: [('A', 1), ('A', 2)]
# B: [('B', 3), ('B', 4)]
# C: [('C', 5)]

# itertools.product：笛卡尔积（类比嵌套循环）
for a, b in itertools.product([1, 2], ["x", "y"]):
    print(a, b)
# 1 x / 1 y / 2 x / 2 y

# itertools.combinations / permutations：组合/排列
print(list(itertools.combinations([1, 2, 3], 2)))
# [(1, 2), (1, 3), (2, 3)]
print(list(itertools.permutations([1, 2, 3], 2)))
# [(1, 2), (1, 3), (2, 1), (2, 3), (3, 1), (3, 2)]
```

## 6. 带发送值的生成器（高级）

```python
# 生成器可以通过 send() 接收外部值
def accumulator():
    """累加生成器"""
    total = 0
    while True:
        value = yield total  # yield 返回 total，接收 send 的值
        if value is None:
            break
        total += value


acc = accumulator()
next(acc)          # 启动生成器（必须先 next 或 send(None)）
print(acc.send(10))  # 10
print(acc.send(20))  # 30
print(acc.send(5))   # 35
```

## 常见陷阱与注意事项

### 1. 生成器只能遍历一次

```python
gen = (x for x in range(5))
for x in gen:
    print(x)  # 0 1 2 3 4

for x in gen:
    print(x)  # 什么都不输出！生成器已耗尽
```

### 2. 生成器函数 vs 返回列表

```python
# ❌ 处理大量数据时返回列表会占用大量内存
def get_all_users():
    return [load_user(id) for id in range(1_000_000)]  # 百万对象！

# ✅ 使用生成器按需加载
def get_all_users():
    for id in range(1_000_000):
        yield load_user(id)
```

### 3. 注意 StopIteration 的传播

```python
# Python 3.7+ 中，生成器内部的 StopIteration 会变成 RuntimeError
def gen():
    try:
        yield next(iter([]))  # iter([]) 是空迭代器，next() 抛 StopIteration
    except StopIteration:
        return  # ✅ 正确处理方式

# 而不是让它自然传播（Python 3.7+ 会报 RuntimeError）
```

## 小结

| 特性 | 说明 | 适用场景 |
|------|------|---------|
| 迭代器协议 | `__iter__` + `__next__` | 自定义可迭代对象 |
| 生成器函数 | `yield` 暂停执行 | 惰性序列、数据管道 |
| 生成器表达式 | `(expr for x in ...)` | 内存高效的推导式 |
| `yield from` | 委托给子迭代器 | 递归生成器、代码重用 |
| `itertools` | 内置迭代器工具库 | 组合、过滤、分组等 |

**AI Agent 开发中的应用：**
- 流式输出（Streaming）：`yield` 逐块产生 LLM 输出
- 大文件处理：逐行读取，不占内存
- 数据管道：多个生成器串联处理数据
