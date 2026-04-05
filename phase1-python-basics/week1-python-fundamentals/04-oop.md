# Python 面向对象编程

> 面向 Java 程序员的 Python OOP 指南：类、继承、魔术方法

## 概述

Python 和 Java 都支持面向对象编程，但 Python 的 OOP 更灵活、更简洁。最重要的差异是 Python 的**鸭子类型**（Duck Typing）和**魔术方法**（Magic Methods）。

## 1. 类的定义

```python
# Python 类定义
class Person:
    # 类变量（所有实例共享）
    species = "Homo sapiens"
    
    # 构造方法（类比 Java 的构造器）
    def __init__(self, name: str, age: int):
        # 实例变量（每个实例独有）
        self.name = name      # 类比 Java 的 this.name
        self.age = age
        self._email = None    # 约定：_ 前缀表示"受保护"（Python 无 private）
        self.__secret = "隐私"  # __ 前缀触发名称改编（name mangling）
    
    # 实例方法（第一个参数必须是 self，类比 Java 的 this）
    def greet(self) -> str:
        return f"你好，我是 {self.name}，{self.age} 岁"
    
    # 类方法（类比 Java 的静态工厂方法）
    @classmethod
    def from_birth_year(cls, name: str, birth_year: int) -> "Person":
        import datetime
        age = datetime.date.today().year - birth_year
        return cls(name, age)
    
    # 静态方法（类比 Java 的 static 方法，与类/实例无关）
    @staticmethod
    def is_adult(age: int) -> bool:
        return age >= 18
    
    # 属性（类比 Java 的 getter/setter，但更简洁）
    @property
    def email(self) -> str:
        return self._email
    
    @email.setter
    def email(self, value: str):
        if "@" not in value:
            raise ValueError(f"无效的邮箱地址：{value}")
        self._email = value


# Java 对比
# public class Person {
#     private static final String SPECIES = "Homo sapiens";
#     private String name;
#     private int age;
#     
#     public Person(String name, int age) {
#         this.name = name;
#         this.age = age;
#     }
#     
#     public String greet() {
#         return "你好，我是 " + name + "，" + age + " 岁";
#     }
#     
#     public static boolean isAdult(int age) {
#         return age >= 18;
#     }
# }
```

### 使用示例

```python
# 创建实例（不需要 new！）
alice = Person("Alice", 25)
print(alice.greet())          # 你好，我是 Alice，25 岁
print(Person.is_adult(25))    # True（静态方法调用）
print(alice.species)          # Homo sapiens（类变量）

# 使用类方法创建实例
bob = Person.from_birth_year("Bob", 1995)

# 属性访问（看起来像直接访问字段，实际上调用了 getter/setter）
alice.email = "alice@example.com"  # 调用 setter，会进行验证
print(alice.email)                  # 调用 getter

try:
    alice.email = "invalid-email"  # 触发 ValueError
except ValueError as e:
    print(e)  # 无效的邮箱地址：invalid-email
```

## 2. 继承

```python
# 基类
class Animal:
    def __init__(self, name: str):
        self.name = name
    
    def speak(self) -> str:
        raise NotImplementedError("子类必须实现 speak 方法")
    
    def __str__(self) -> str:
        return f"Animal({self.name})"


# 继承（类比 Java 的 extends）
class Dog(Animal):
    def __init__(self, name: str, breed: str):
        super().__init__(name)  # 调用父类构造方法，类比 super()
        self.breed = breed
    
    def speak(self) -> str:
        return f"{self.name} 汪汪！"
    
    def fetch(self) -> str:
        return f"{self.name} 去捡球了！"


class Cat(Animal):
    def speak(self) -> str:
        return f"{self.name} 喵喵！"


# 使用
dog = Dog("旺财", "金毛")
cat = Cat("咪咪")

print(dog.speak())   # 旺财 汪汪！
print(cat.speak())   # 咪咪 喵喵！

# isinstance 检查（类比 Java 的 instanceof）
print(isinstance(dog, Dog))     # True
print(isinstance(dog, Animal))  # True（多态）
print(isinstance(dog, Cat))     # False

# issubclass 检查
print(issubclass(Dog, Animal))  # True
```

### 多继承（Python 特有）

```python
# Python 支持多继承（Java 只能通过接口实现类似效果）
class Flyable:
    def fly(self) -> str:
        return "我在飞！"
    
    def move(self) -> str:
        return "通过飞行移动"

class Swimmable:
    def swim(self) -> str:
        return "我在游泳！"
    
    def move(self) -> str:
        return "通过游泳移动"

class Duck(Animal, Flyable, Swimmable):
    def speak(self) -> str:
        return "嘎嘎！"
    
    def move(self) -> str:
        return "既能飞也能游！"

# MRO（方法解析顺序）：Python 使用 C3 线性化算法
print(Duck.__mro__)
# (<class 'Duck'>, <class 'Animal'>, <class 'Flyable'>, <class 'Swimmable'>, <class 'object'>)

donald = Duck("唐老鸭")
print(donald.fly())    # 我在飞！
print(donald.swim())   # 我在游泳！
print(donald.move())   # 既能飞也能游！（子类覆盖）
```

### 抽象类（类比 Java 的 abstract class）

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        """计算面积"""
        pass
    
    @abstractmethod
    def perimeter(self) -> float:
        """计算周长"""
        pass
    
    def describe(self) -> str:
        return f"面积：{self.area():.2f}，周长：{self.perimeter():.2f}"


class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius
    
    def area(self) -> float:
        import math
        return math.pi * self.radius ** 2
    
    def perimeter(self) -> float:
        import math
        return 2 * math.pi * self.radius


class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height
    
    def area(self) -> float:
        return self.width * self.height
    
    def perimeter(self) -> float:
        return 2 * (self.width + self.height)


# 不能直接实例化抽象类
# shape = Shape()  # TypeError!

circle = Circle(5)
rect = Rectangle(4, 6)
print(circle.describe())  # 面积：78.54，周长：31.42
print(rect.describe())    # 面积：24.00，周长：20.00
```

## 3. 魔术方法（Magic Methods / Dunder Methods）

魔术方法（双下划线方法）是 Python OOP 的核心特色，让你的类与 Python 内置语法无缝集成。

### 常用魔术方法

```python
class Vector:
    """二维向量类，展示常用魔术方法"""
    
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y
    
    # __str__：用于 print()，类比 Java 的 toString()
    def __str__(self) -> str:
        return f"Vector({self.x}, {self.y})"
    
    # __repr__：用于调试，应该返回可重建对象的字符串
    def __repr__(self) -> str:
        return f"Vector(x={self.x!r}, y={self.y!r})"
    
    # __add__：支持 + 运算符（类比 Java 的运算符重载，但 Java 不支持！）
    def __add__(self, other: "Vector") -> "Vector":
        return Vector(self.x + other.x, self.y + other.y)
    
    # __sub__：支持 - 运算符
    def __sub__(self, other: "Vector") -> "Vector":
        return Vector(self.x - other.x, self.y - other.y)
    
    # __mul__：支持 * 运算符（向量与标量相乘）
    def __mul__(self, scalar: float) -> "Vector":
        return Vector(self.x * scalar, self.y * scalar)
    
    # __eq__：支持 == 比较，类比 Java 的 equals()
    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Vector):
            return NotImplemented
        return self.x == other.x and self.y == other.y
    
    # __hash__：支持在 set 和 dict 中使用，类比 Java 的 hashCode()
    # 注意：定义了 __eq__ 就必须定义 __hash__
    def __hash__(self) -> int:
        return hash((self.x, self.y))
    
    # __len__：支持 len()
    def __len__(self) -> int:
        import math
        return int(math.sqrt(self.x ** 2 + self.y ** 2))
    
    # __abs__：支持 abs()
    def __abs__(self) -> float:
        import math
        return math.sqrt(self.x ** 2 + self.y ** 2)
    
    # __bool__：支持 if 判断
    def __bool__(self) -> bool:
        return self.x != 0 or self.y != 0


# 使用
v1 = Vector(1, 2)
v2 = Vector(3, 4)

print(v1)              # Vector(1, 2)（调用 __str__）
print(repr(v1))        # Vector(x=1, y=2)（调用 __repr__）
print(v1 + v2)         # Vector(4, 6)（调用 __add__）
print(v1 - v2)         # Vector(-2, -2)（调用 __sub__）
print(v1 * 3)          # Vector(3, 6)（调用 __mul__）
print(v1 == Vector(1, 2))  # True（调用 __eq__）
print(abs(v2))         # 5.0（调用 __abs__）

# 可以放入 set（因为实现了 __hash__）
vectors = {v1, v2}
print(len(vectors))    # 2
```

### 容器协议魔术方法

```python
class NumberList:
    """自定义容器，实现容器协议"""
    
    def __init__(self, *args):
        self._data = list(args)
    
    # 支持 len()
    def __len__(self) -> int:
        return len(self._data)
    
    # 支持 obj[index]（类比 Java 的 get(index)）
    def __getitem__(self, index):
        return self._data[index]
    
    # 支持 obj[index] = value（类比 Java 的 set(index, value)）
    def __setitem__(self, index, value):
        self._data[index] = value
    
    # 支持 del obj[index]
    def __delitem__(self, index):
        del self._data[index]
    
    # 支持 item in obj（类比 Java 的 contains()）
    def __contains__(self, item) -> bool:
        return item in self._data
    
    # 支持 for x in obj（迭代）
    def __iter__(self):
        return iter(self._data)


nums = NumberList(1, 2, 3, 4, 5)
print(len(nums))       # 5
print(nums[2])         # 3
print(3 in nums)       # True
for n in nums:         # 可以迭代
    print(n)
```

### 上下文管理器魔术方法

```python
class DatabaseConnection:
    """模拟数据库连接，展示上下文管理器"""
    
    def __init__(self, host: str):
        self.host = host
        self.connected = False
    
    # 进入 with 块时调用
    def __enter__(self):
        print(f"连接到 {self.host}")
        self.connected = True
        return self  # 返回 as 子句中的对象
    
    # 离开 with 块时调用（无论是否发生异常）
    def __exit__(self, exc_type, exc_val, exc_tb):
        print(f"断开 {self.host} 连接")
        self.connected = False
        return False  # 不抑制异常


# 使用 with 语句（类比 Java 的 try-with-resources）
with DatabaseConnection("localhost:5432") as db:
    print(f"连接状态：{db.connected}")
    # 执行数据库操作...
# 离开 with 块后自动断开连接
print(f"连接状态：{db.connected}")  # False
```

## 4. 数据类（dataclass）

Python 3.7+ 的 `@dataclass` 装饰器可以自动生成常见方法（类比 Lombok 的 `@Data`）：

```python
from dataclasses import dataclass, field
from typing import List

# @dataclass 自动生成 __init__、__repr__、__eq__
@dataclass
class Student:
    name: str
    age: int
    grades: List[float] = field(default_factory=list)
    
    # 可以添加自定义方法
    def average_grade(self) -> float:
        if not self.grades:
            return 0.0
        return sum(self.grades) / len(self.grades)


# Java 对比（需要 Lombok）：
# @Data
# public class Student {
#     private String name;
#     private int age;
#     private List<Double> grades = new ArrayList<>();
# }

s1 = Student("Alice", 20, [85.0, 92.5, 78.0])
s2 = Student("Alice", 20, [85.0, 92.5, 78.0])
print(s1)                   # Student(name='Alice', age=20, grades=[85.0, 92.5, 78.0])
print(s1 == s2)             # True（自动生成的 __eq__）
print(s1.average_grade())   # 85.17

# 不可变 dataclass（类比 Java 的 record）
@dataclass(frozen=True)
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
# p.x = 3.0  # FrozenInstanceError!
```

## 5. 鸭子类型与协议

```python
# Python 不需要实现接口，只需要有对应的方法（鸭子类型）

class Duck:
    def quack(self):
        return "嘎嘎！"
    
    def walk(self):
        return "蹒跚而行"

class Person:
    def quack(self):
        return "我在模仿鸭子！"
    
    def walk(self):
        return "像人一样走路"

def make_it_quack(duck_like):
    # 不检查类型，只检查行为
    print(duck_like.quack())

make_it_quack(Duck())    # 嘎嘎！
make_it_quack(Person())  # 我在模仿鸭子！

# Python 3.8+ 的 Protocol：显式定义鸭子类型（类比 Java 接口）
from typing import Protocol

class Quackable(Protocol):
    def quack(self) -> str: ...

def make_noise(obj: Quackable) -> None:
    print(obj.quack())

# Duck 和 Person 无需显式 implements Quackable
make_noise(Duck())    # ✅ 有 quack() 方法，满足协议
make_noise(Person())  # ✅ 有 quack() 方法，满足协议
```

## 常见陷阱与注意事项

### 1. 忘记 `self`

```python
class Counter:
    count = 0
    
    def increment():  # ❌ 缺少 self！
        count += 1    # 这会修改的是局部变量，不是实例变量

    def increment(self):  # ✅ 正确
        self.count += 1
```

### 2. 类变量 vs 实例变量

```python
class MyClass:
    class_var = []  # ⚠️ 类变量，所有实例共享！
    
    def __init__(self):
        self.instance_var = []  # ✅ 实例变量，每个实例独有

a = MyClass()
b = MyClass()
a.class_var.append(1)
print(b.class_var)    # [1]  <- b 的类变量也变了！
a.instance_var.append(1)
print(b.instance_var) # []   <- b 的实例变量没变
```

### 3. `__eq__` 与 `__hash__`

```python
# 规则：如果实现了 __eq__，必须同时实现 __hash__
# 否则对象无法放入 set 或作为 dict 键

class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    def __eq__(self, other):
        return self.x == other.x and self.y == other.y
    
    # 如果不定义 __hash__，Python 会自动将 __hash__ 设为 None
    def __hash__(self):
        return hash((self.x, self.y))  # 用 tuple 的 hash
```

## 小结

| 概念 | Java | Python |
|------|------|--------|
| 构造方法 | `public ClassName(...)` | `def __init__(self, ...)` |
| `this` 关键字 | `this.field` | `self.field` |
| 方法 | `public void method()` | `def method(self):` |
| 静态方法 | `static void method()` | `@staticmethod def method():` |
| 工厂方法 | 静态方法 | `@classmethod def method(cls):` |
| 属性封装 | `getField()`/`setField()` | `@property` / `@field.setter` |
| 继承 | `class B extends A` | `class B(A):` |
| 调用父类 | `super.method()` | `super().method()` |
| 接口 | `interface` / `implements` | `Protocol` / 鸭子类型 |
| 抽象类 | `abstract class` | `class X(ABC):` |
| `toString` | `@Override toString()` | `def __str__(self):` |
| `equals` | `@Override equals()` | `def __eq__(self, other):` |
| `hashCode` | `@Override hashCode()` | `def __hash__(self):` |
| 数据类 | Lombok `@Data` / Java 14 `record` | `@dataclass` |
