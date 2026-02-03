---
name: solid-principles
description: Use when designing Python class hierarchies, interfaces, or module boundaries - specifically when code is hard to extend, modify, or when changes cause cascading bugs
---

# SOLID Principles

## Overview
**Five object-oriented design principles for maintainable, extensible code: SRP, OCP, LSP, ISP, DIP.**

## When to Use
- Designing new classes or interfaces
- Refactoring God Classes
- Adding new features breaks existing code
- Unit tests require too many mocks

## 1. SRP - Single Responsibility Principle

**A class should have only one reason to change.**

```python
# Bad: Two reasons to change (data + report)
class User:
    def save(self): ...
    def generate_report(self): ...

# Good: Separate concerns
class User: ...
class UserRepository: ...
class ReportGenerator: ...
```

## 2. OCP - Open/Closed Principle

**Software entities should be open for extension, closed for modification.**

```python
# Bad: Modify to add new type
def calculate_price(order: Order) -> float:
    if order.type == "standard": ...
    elif order.type == "premium": ...

# Good: Extend via inheritance
from abc import ABC, abstractmethod

class PricingStrategy(ABC):
    @abstractmethod
    def calculate(self, order: Order) -> float: ...

class StandardPricing(PricingStrategy): ...
class PremiumPricing(PricingStrategy): ...
```

## 3. LSP - Liskov Substitution Principle

**Subtypes must be substitutable for their base types.**

```python
# Bad: Subclass changes expected behavior
class Bird:
    def fly(self): ...

class Penguin(Bird):
    def fly(self): raise Exception("Can't fly!")  # Violates LSP

# Good: Design to contract
class FlyingBird:
    @abstractmethod
    def fly(self): ...

class Bird(ABC): ...

class Sparrow(FlyingBird): ...

class Penguin(Bird):  # Doesn't implement fly
```

## 4. ISP - Interface Segregation Principle

**Clients should not be forced to depend on methods they do not use.**

```python
# Bad: Fat interface
class Machine(ABC):
    @abstractmethod
    def print(self): ...
    @abstractmethod
    def scan(self): ...
    @abstractmethod
    def fax(self): ...

# Good: Segregated interfaces
class Printer(ABC):
    @abstractmethod
    def print(self): ...

class Scanner(ABC):
    @abstractmethod
    def scan(self): ...
```

## 5. DIP - Dependency Inversion Principle

**Depend on abstractions, not concretions.**

```python
# Bad: High-level depends on low-level
class MySQLDatabase:
    def connect(self): ...

class UserService:
    def __init__(self):
        self.db = MySQLDatabase()  # Tight coupling

# Good: Depend on abstraction
class Database(ABC):
    @abstractmethod
    def connect(self): ...

class UserService:
    def __init__(self, db: Database):  # Accept any DB
        self.db = db
```

## Quick Reference

| Principle | Focus | Bad Sign |
|-----------|-------|----------|
| SRP | One responsibility | God classes |
| OCP | Extension, not modification | Changing existing code for new features |
| LSP | Substitutability | Type checks in base methods |
| ISP | Fine-grained interfaces | Empty method implementations |
| DIP | Abstractions over concretions | `import` concrete classes |

## The Bottom Line

**SOLID = Easier changes, fewer bugs, better testing.**
