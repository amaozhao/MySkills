---
name: python-design-principles
description: Use when making design decisions in Python code - specifically when choosing between patterns, deciding on abstraction levels, or evaluating code complexity
---

# Python Design Principles

## Overview
**Core design principles for Python: DRY (Don't Repeat Yourself), KISS (Keep It Simple, Stupid), YAGNI (You Aren't Gonna Need It), and Composition over Inheritance.**

## When to Use
- Deciding between code duplication and abstraction
- Choosing the simplest solution
- Adding "just in case" features
- Class inheritance vs composition

## DRY - Don't Repeat Yourself

**Every piece of knowledge should have a single, unambiguous representation.**

```python
# Bad: Repeated validation logic
def create_user(name, email):
    if not name: raise ValueError("Name required")
    ...

def update_user(name, email):
    if not name: raise ValueError("Name required")
    ...

# Good: Single source of truth
class UserValidator:
    @staticmethod
    def validate(data):
        if not data.get("name"):
            raise ValueError("Name required")

def create_user(name, email):
    UserValidator.validate({"name": name})
    ...

def update_user(name, email):
    UserValidator.validate({"name": name})
    ...
```

## KISS - Keep It Simple, Stupid

**Favor simple solutions over complex ones.**

```python
# Complex: Over-engineered
class StringProcessor:
    def __init__(self, transformer_factory, validator_chain, cache_strategy):
        ...

# Simple: Do one thing
def normalize_whitespace(text: str) -> str:
    return " ".join(text.split())
```

## YAGNI - You Aren't Gonna Need It

**Don't implement functionality until it is actually needed.**

```python
# Bad: "Just in case" abstraction
class IUserRepository(ABC):
    @abstractmethod
    def get_by_id(self): ...
    @abstractmethod
    def get_by_email(self): ...
    @abstractmethod
    def get_by_phone(self): ...  # Not used yet!

# Good: Implement when needed
class UserRepository:
    async def get_by_id(self, id: int):
        ...

# Add later when needed
class UserRepository:
    async def get_by_id(self, id: int): ...

    async def get_by_email(self, email: str): ...  # Now needed
```

## Composition over Inheritance

**Favor object composition over class inheritance.**

```python
# Inheritance (tight coupling)
class Bird(Animal):
    def __init__(self):
        self.fly_behavior = FlyBehavior()

class FlyingAnimal(Animal):
    ...

# Composition (flexible)
class Bird:
    def __init__(self, fly_strategy: FlyStrategy):
        self.fly_strategy = fly_strategy

    def fly(self):
        self.fly_strategy.fly()

class FlyWithWings: ...
class FlyNoWay: ...

# Usage
sparrow = Bird(FlyWithWings())
penguin = Bird(FlyNoWay())
```

## When to Break These Principles

| Principle | Exception | Reason |
|-----------|-----------|--------|
| DRY | Copy-paste in tests | Tests need isolation |
| DRY | Generated code | Generated files are separate |
| KISS | Performance-critical code | Sometimes complex is faster |
| YAGNI | Architecture foundations | Some patterns are too hard to add later |
| Composition | Simple hierarchies | Inheritance is fine for 1-2 levels |

## The Bottom Line

**DRY for business logic, KISS for solutions, YAGNI for features, Composition for flexibility.**
