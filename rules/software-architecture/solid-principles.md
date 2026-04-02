---
name: solid-principles
description: 设计 Python 类层次结构、接口或模块边界 - 当代码难以扩展、修改，或修改导致级联 bug 时使用
paths: **/*.py
---

# SOLID 原则

## 概述

**五大面向对象设计原则：SRP、OCP、LSP、ISP、DIP，助你写出可维护、可扩展的代码。**

## 适用场景

- 设计新的类或接口
- 重构上帝类（God Class）
- 添加新功能会破坏现有代码
- 单元测试需要大量 mock

---

## 1. SRP - 单一职责原则

**一个类只有一个变化原因。**

```python
# 错误：两个变化原因（数据 + 报表）
class User:
    def save(self): ...
    def generate_report(self): ...

# 正确：分离关注点
class User: ...
class UserRepository: ...
class ReportGenerator: ...
```

---

## 2. OCP - 开闭原则

**软件实体应该对扩展开放，对修改关闭。**

```python
# 错误：修改以添加新类型
def calculate_price(order: Order) -> float:
    if order.type == "standard": ...
    elif order.type == "premium": ...

# 正确：通过继承扩展
from abc import ABC, abstractmethod

class PricingStrategy(ABC):
    @abstractmethod
    def calculate(self, order: Order) -> float: ...

class StandardPricing(PricingStrategy): ...
class PremiumPricing(PricingStrategy): ...
```

---

## 3. LSP - 里氏替换原则

**子类型必须可以替换其基类型。**

```python
# 错误：子类改变了预期行为
class Bird:
    def fly(self): ...

class Penguin(Bird):
    def fly(self): raise Exception("不能飞！")  # 违反 LSP

# 正确：按契约设计
class FlyingBird:
    @abstractmethod
    def fly(self): ...

class Bird(ABC): ...

class Sparrow(FlyingBird): ...

class Penguin(Bird):  # 不实现 fly
```

---

## 4. ISP - 接口隔离原则

**客户端不应该依赖它不使用的方法。**

```python
# 错误：臃肿接口
class Machine(ABC):
    @abstractmethod
    def print(self): ...
    @abstractmethod
    def scan(self): ...
    @abstractmethod
    def fax(self): ...

# 正确：隔离接口
class Printer(ABC):
    @abstractmethod
    def print(self): ...

class Scanner(ABC):
    @abstractmethod
    def scan(self): ...
```

---

## 5. DIP - 依赖倒置原则

**依赖抽象，而不是具体实现。**

```python
# 错误：高层依赖低层
class MySQLDatabase:
    def connect(self): ...

class UserService:
    def __init__(self):
        self.db = MySQLDatabase()  # 紧耦合

# 正确：依赖抽象
class Database(ABC):
    @abstractmethod
    def connect(self): ...

class UserService:
    def __init__(self, db: Database):  # 接受任何数据库
        self.db = db
```

---

## 快速参考

| 原则 | 关注点 | 坏味道 |
|------|--------|--------|
| SRP | 单一职责 | 上帝类 |
| OCP | 扩展而非修改 | 为新功能改现有代码 |
| LSP | 可替换性 | 基方法中有类型检查 |
| ISP | 细粒度接口 | 空方法实现 |
| DIP | 抽象优于具体 | `import` 具体类 |

---

## 核心要点

**SOLID = 更容易修改、更少 bug、更好测试。**

---

## 相关文件

- [python-project/principles.md](../python-project/principles.md) - Python 设计原则（DRY、KISS、YAGNI）
