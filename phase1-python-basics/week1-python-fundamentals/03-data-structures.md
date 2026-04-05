# Python 数据结构

> 掌握 Python 的四种核心数据结构：list、dict、set、tuple，以及强大的推导式语法

## 概述

Python 内置了非常强大的数据结构，与 Java 集合框架的对应关系如下：

| Python | Java | 特点 |
|--------|------|------|
| `list` | `ArrayList<E>` | 有序、可重复、可变 |
| `dict` | `HashMap<K,V>` | 键值对、无序（3.7+ 保留插入顺序） |
| `set` | `HashSet<E>` | 无序、不重复、可变 |
| `tuple` | 无直接对应（类似不可变的 `List`） | 有序、可重复、**不可变** |
| `frozenset` | `Collections.unmodifiableSet()` | 不可变的 set |

## 1. list（列表）

### 基本操作

```python
# 创建列表
nums = [1, 2, 3, 4, 5]
names = ["Alice", "Bob", "Carol"]
mixed = [1, "hello", 3.14, True]  # Python list 可以混合类型（Java 的泛型不行）

# 访问元素（索引从 0 开始，负索引从末尾计数）
print(nums[0])   # 1（第一个）
print(nums[-1])  # 5（最后一个）
print(nums[-2])  # 4（倒数第二个）

# 切片（Python 特有，非常强大！）
print(nums[1:3])    # [2, 3]（索引 1 到 2）
print(nums[:3])     # [1, 2, 3]（前 3 个）
print(nums[2:])     # [3, 4, 5]（从索引 2 到末尾）
print(nums[::2])    # [1, 3, 5]（步长为 2）
print(nums[::-1])   # [5, 4, 3, 2, 1]（反转）

# 修改元素
nums[0] = 10
print(nums)  # [10, 2, 3, 4, 5]

# 添加元素
nums.append(6)      # 末尾添加：[10, 2, 3, 4, 5, 6]
nums.insert(1, 99)  # 指定位置插入：[10, 99, 2, 3, 4, 5, 6]
nums.extend([7, 8]) # 末尾添加多个：类比 addAll

# 删除元素
nums.remove(99)     # 按值删除第一个匹配项
popped = nums.pop() # 删除并返回最后一个元素
del nums[0]         # 按索引删除

# 常用操作
print(len(nums))    # 长度，类比 size()
print(3 in nums)    # 包含检查，类比 contains()
nums.sort()         # 原地排序
nums.reverse()      # 原地反转
sorted_nums = sorted(nums)  # 返回新排序列表，不修改原列表
print(nums.index(3))  # 查找元素索引，类比 indexOf()
print(nums.count(3))  # 统计出现次数
```

### 列表常用方法对比

```python
# Java vs Python 对比
# Java: List<Integer> list = new ArrayList<>(); list.add(1);
# Python:
nums = []
nums.append(1)

# Java: list.size()
# Python:
len(nums)

# Java: list.get(0)
# Python:
nums[0]

# Java: list.set(0, 10)
# Python:
nums[0] = 10

# Java: list.remove(0)（按索引）
# Python:
del nums[0]
nums.pop(0)

# Java: Collections.sort(list)
# Python:
nums.sort()
sorted_nums = sorted(nums)  # 不修改原列表
```

## 2. dict（字典）

### 基本操作

```python
# 创建字典
person = {"name": "Alice", "age": 25, "city": "北京"}

# 也可以用 dict()
person2 = dict(name="Bob", age=30, city="上海")

# 访问元素
print(person["name"])           # "Alice"（键不存在会抛 KeyError）
print(person.get("name"))       # "Alice"（推荐）
print(person.get("email", ""))  # ""（键不存在时返回默认值，不抛异常）

# 修改 / 添加元素
person["age"] = 26         # 修改
person["email"] = "alice@example.com"  # 新增

# 删除元素
del person["city"]
removed = person.pop("email", None)  # 删除并返回，键不存在时返回 None

# 检查键是否存在
print("name" in person)      # True
print("city" in person)      # False（已删除）

# 遍历字典
for key in person:           # 遍历键
    print(key)

for key, value in person.items():  # 遍历键值对
    print(f"{key}: {value}")

for key in person.keys():    # 遍历键
    print(key)

for value in person.values():  # 遍历值
    print(value)

# 字典合并（Python 3.9+）
dict1 = {"a": 1, "b": 2}
dict2 = {"b": 3, "c": 4}
merged = dict1 | dict2   # {"a": 1, "b": 3, "c": 4}（b 被覆盖）

# 较老的写法
merged = {**dict1, **dict2}  # 同上
```

### 常用技巧

```python
# 统计字符频率（类比 Java 的 Map.getOrDefault）
text = "hello world"
freq = {}
for char in text:
    freq[char] = freq.get(char, 0) + 1
print(freq)

# 更简洁：使用 defaultdict
from collections import defaultdict
freq = defaultdict(int)
for char in text:
    freq[char] += 1

# 更更简洁：使用 Counter
from collections import Counter
freq = Counter(text)
print(freq.most_common(3))  # 出现最多的 3 个字符

# 嵌套字典
employees = {
    "E001": {"name": "Alice", "dept": "Engineering"},
    "E002": {"name": "Bob", "dept": "Marketing"},
}
print(employees["E001"]["name"])  # "Alice"
```

## 3. set（集合）

```python
# 创建集合（类比 Java 的 HashSet）
fruits = {"apple", "banana", "orange"}
empty_set = set()  # 注意：{} 创建的是空字典！

# 从列表创建集合（去重）
nums = [1, 2, 2, 3, 3, 3]
unique_nums = set(nums)  # {1, 2, 3}

# 基本操作
fruits.add("grape")       # 添加元素
fruits.remove("banana")   # 删除（不存在会抛异常）
fruits.discard("mango")   # 删除（不存在不抛异常）
print("apple" in fruits)  # True（包含检查）
print(len(fruits))        # 长度

# 集合运算（Java 没有这么方便的语法！）
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

print(a | b)   # 并集：{1, 2, 3, 4, 5, 6}（类比 addAll）
print(a & b)   # 交集：{3, 4}（类比 retainAll）
print(a - b)   # 差集：{1, 2}（在 a 但不在 b）
print(a ^ b)   # 对称差：{1, 2, 5, 6}（只在其中一个中的元素）

# 子集和超集检查
print({1, 2} <= {1, 2, 3})  # True（{1,2} 是 {1,2,3} 的子集）
print({1, 2, 3} >= {1, 2})  # True（{1,2,3} 是 {1,2} 的超集）

# 实用场景：去重
users = ["alice", "bob", "alice", "carol", "bob"]
unique_users = list(set(users))  # ['alice', 'carol', 'bob']（顺序不保证）
```

## 4. tuple（元组）

```python
# 创建元组（不可变列表）
point = (3, 4)
rgb = (255, 128, 0)
single = (42,)  # 注意：单元素元组需要尾随逗号
empty = ()

# 访问元素（与 list 相同）
print(point[0])   # 3
print(point[-1])  # 4

# 元组解包（非常强大！）
x, y = point
print(x, y)  # 3 4

# 交换变量（无需临时变量！）
a, b = 1, 2
a, b = b, a  # Java 需要: int temp = a; a = b; b = temp;
print(a, b)  # 2 1

# 函数返回多个值（实际上返回的是元组）
def min_max(nums):
    return min(nums), max(nums)

lo, hi = min_max([3, 1, 4, 1, 5, 9])
print(lo, hi)  # 1 9

# 元组作为字典键（list 不能作为键，因为不可 hash）
locations = {(40.7128, -74.0060): "New York", (51.5074, -0.1278): "London"}
print(locations[(40.7128, -74.0060)])  # "New York"

# tuple vs list：何时用哪个？
# - tuple：数据不变的序列（坐标、RGB 颜色、数据库记录）
# - list：需要修改的序列（购物车、待办事项）
```

## 5. 推导式（Comprehension）

推导式是 Python 最优雅的特性之一，Java 程序员可以类比 Stream API。

### 列表推导式

```python
# 基本格式：[表达式 for 变量 in 可迭代对象 if 条件]

# 生成平方数列表
squares = [x ** 2 for x in range(10)]
# 等价于：
squares = []
for x in range(10):
    squares.append(x ** 2)

# 带条件过滤（类比 Java Stream 的 filter）
even_squares = [x ** 2 for x in range(10) if x % 2 == 0]
# 等价于 Java:
# List<Integer> evenSquares = IntStream.range(0, 10)
#     .filter(x -> x % 2 == 0)
#     .map(x -> x * x)
#     .boxed().collect(Collectors.toList());

# 嵌套推导式（类比嵌套循环）
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [x for row in matrix for x in row]
# [1, 2, 3, 4, 5, 6, 7, 8, 9]

# 字符串处理
words = ["hello", "world", "python"]
upper_words = [w.upper() for w in words]
# ['HELLO', 'WORLD', 'PYTHON']
```

### 字典推导式

```python
# 基本格式：{键表达式: 值表达式 for 变量 in 可迭代对象 if 条件}

# 创建平方字典
squares = {x: x ** 2 for x in range(6)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16, 5: 25}

# 字典键值互换
original = {"a": 1, "b": 2, "c": 3}
inverted = {v: k for k, v in original.items()}
# {1: 'a', 2: 'b', 3: 'c'}

# 过滤字典
scores = {"Alice": 85, "Bob": 60, "Carol": 92, "Dave": 45}
passing = {name: score for name, score in scores.items() if score >= 60}
# {"Alice": 85, "Bob": 60, "Carol": 92}
```

### 集合推导式

```python
# 基本格式：{表达式 for 变量 in 可迭代对象 if 条件}

# 提取唯一字符
unique_chars = {c for c in "hello world" if c != " "}
# {'h', 'e', 'l', 'o', 'w', 'r', 'd'}
```

### 生成器表达式

```python
# 类似列表推导式，但用 () 而不是 []
# 生成器是惰性的（不一次性生成所有元素），节省内存

# 计算大量数字的平方和（不把所有平方都存在内存里）
total = sum(x ** 2 for x in range(1000000))

# 找第一个满足条件的元素
first_even = next(x for x in range(100) if x % 2 == 0)
# 0

# 列表推导式 vs 生成器表达式
# 列表推导式：立即生成所有元素，占用内存
squares_list = [x ** 2 for x in range(1000000)]  # 占用大量内存

# 生成器表达式：按需生成，内存效率高
squares_gen = (x ** 2 for x in range(1000000))   # 几乎不占内存
```

## 常见陷阱与注意事项

### 1. 浅拷贝陷阱

```python
# ❌ 错误：列表赋值是引用，不是拷贝
a = [1, 2, 3]
b = a          # b 和 a 指向同一个列表！
b.append(4)
print(a)       # [1, 2, 3, 4]  <- a 也变了！

# ✅ 方法1：切片创建浅拷贝
b = a[:]

# ✅ 方法2：copy() 方法
b = a.copy()

# ✅ 方法3：list() 构造函数
b = list(a)

# 注意：对于嵌套列表，以上都是浅拷贝
# 完全独立的拷贝需要深拷贝
import copy
nested = [[1, 2], [3, 4]]
deep = copy.deepcopy(nested)
```

### 2. 字典的 `get` vs `[]`

```python
d = {"key": "value"}

# ❌ 容易报错
print(d["missing_key"])  # KeyError!

# ✅ 使用 get 更安全
print(d.get("missing_key"))           # None
print(d.get("missing_key", "default"))  # "default"
```

### 3. 集合无序性

```python
s = {3, 1, 4, 1, 5, 9}
print(s)  # {1, 3, 4, 5, 9}（顺序不保证！）
# 不能用索引访问集合元素：s[0] 会报 TypeError
```

### 4. 推导式可读性

```python
# ❌ 过于复杂的推导式（难以阅读）
result = [f(x) for x in range(100) if g(x) and h(x) for y in some_list(x) if y > 0]

# ✅ 改用普通循环，可读性更好
result = []
for x in range(100):
    if g(x) and h(x):
        for y in some_list(x):
            if y > 0:
                result.append(f(x))
```

## 小结

| 数据结构 | 可变 | 有序 | 重复 | Java 对应 | 创建语法 |
|---------|------|------|------|---------|---------|
| `list` | ✅ | ✅ | ✅ | `ArrayList` | `[1, 2, 3]` |
| `dict` | ✅ | ✅(3.7+) | 键唯一 | `HashMap` | `{"k": "v"}` |
| `set` | ✅ | ❌ | ❌ | `HashSet` | `{1, 2, 3}` |
| `tuple` | ❌ | ✅ | ✅ | 不可变List | `(1, 2, 3)` |
| `frozenset` | ❌ | ❌ | ❌ | `unmodifiableSet` | `frozenset([1, 2])` |

**推导式三兄弟**：
- 列表推导式：`[expr for x in iterable if condition]`
- 字典推导式：`{k: v for k, v in iterable if condition}`
- 集合推导式：`{expr for x in iterable if condition}`
- 生成器表达式：`(expr for x in iterable if condition)` → 惰性求值，节省内存
