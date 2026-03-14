---
name: python-principles
description: Python 设计原则 - DRY、KISS、YAGNI、组合
---

# Python Design Principles

## 概述

核心设计原则：DRY（不要重复自己）、KISS（保持简单）、YAGNI（你不会需要它）、组合优于继承。

## DRY - Don't Repeat Yourself

**每条知识应该有单一、明确的表示。**

```python
# 不好：重复的验证逻辑
def create_user(name, email):
    if not name: raise ValueError("Name required")
    ...

def update_user(name, email):
    if not name: raise ValueError("Name required")
    ...

# 好：单一真相来源
class UserValidator:
    @staticmethod
    def validate(data):
        if not data.get("name"):
            raise ValueError("Name required")

def create_user(name, email):
    UserValidator.validate({"name": name})
    ...
```

## KISS - Keep It Simple

**选择简单的解决方案。**

```python
# 复杂：过度设计
class StringProcessor:
    def __init__(self, transformer_factory, validator_chain, cache_strategy):
        ...

# 简单：做一件事
def normalize_whitespace(text: str) -> str:
    return " " .join(text.split())
```

## YAGNI - You Aren't Gonna Need It

**不要实现尚未需要的功能。**

```python
# 不好："以防万一"的抽象
class IUserRepository(ABC):
    @abstractmethod
    def get_by_id(self): ...
    @abstractmethod
    def get_by_email(self): ...
    @abstractmethod
    def get_by_phone(self): ...  # 还没用到！

# 好：需要时再实现
class UserRepository:
    async def get_by_id(self, id: int):
        ...

# 之后需要时再添加
class UserRepository:
    async def get_by_id(self, id: int): ...
    async def get_by_email(self, email: str): ...  # 现在需要了
```

## Composition over Inheritance

**优先使用对象组合而非类继承。**

```python
# 继承（紧耦合）
class Bird(Animal):
    def __init__(self):
        self.fly_behavior = FlyBehavior()

# 组合（灵活）
class Bird:
    def __init__(self, fly_strategy: FlyStrategy):
        self.fly_strategy = fly_strategy

    def fly(self):
        self.fly_strategy.fly()

class FlyWithWings: ...
class FlyNoWay: ...

# 使用
sparrow = Bird(FlyWithWings())
penguin = Bird(FlyNoWay())
```

## 何时打破原则

| 原则 | 例外 | 原因 |
|------|------|------|
| DRY | 测试中复制粘贴 | 测试需要隔离 |
| DRY | 生成代码 | 生成文件是独立的 |
| KISS | 性能关键代码 | 有时复杂更快 |
| YAGNI | 基础架构 | 有些模式之后很难添加 |
| 组合 | 简单继承 | 一两层继承是可以的 |

---

## 相关文件

- [style.md](./style.md) - 代码风格
- [errors.md](./errors.md) - 错误处理
- [solid-principles](../solid-principles/SKILL.md) - 面向对象设计原则（SRP, OCP, DIP 等）
