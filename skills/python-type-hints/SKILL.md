---
name: python-type-hints
description: Use when writing type-safe Python code, especially with Python 3.12+ features like PEP 695 type parameters, TypedDict, Final, Self, or LiteralString
---

# Python Type Hints

## Overview
**Modern Python type hints use Python 3.12+ syntax: `|` for unions, `type` for aliases, `ReadOnly`/`NotRequired` for TypedDict, `Final` for constants.**

## When to Use
- Writing new Python code
- Adding type hints to existing code
- Using TypedDict with optional/required fields
- Defining generic functions or classes
- Preventing accidental mutations

## Modern Syntax (Python 3.10+)

```python
# Use | instead of Union
def process(value: int | str) -> None: ...

# Use | None instead of Optional
name: str | None = None

# List and dict types
from collections.abc import Iterable
prices: list[float] = []
data: dict[str, int] = {}
```

## PEP 695: Type Aliases

```python
# Old way
from typing import TypeAlias
Point: TypeAlias = tuple[float, float]

# New way (Python 3.12+)
type Point = tuple[float, float]
type ApiResponse = dict[str, int | str]
```

## PEP 695: Generic Functions/Classes

```python
def first[T](items: Iterable[T]) -> T | None:
    return next(iter(items), None)

class Container[T]:
    def __init__(self, content: T):
        self.content = content

    def get(self) -> T:
        return self.content
```

## TypedDict with Required/NotRequired

```python
from typing import TypedDict, Required, NotRequired, ReadOnly

class OrderRequest(TypedDict, total=False):
    symbol: Required[str]      # Must be provided
    quantity: NotRequired[int] # Optional
    price: NotRequired[float]

class Config(TypedDict):
    debug: bool
    timeout: ReadOnly[int]     # Immutable after creation
```

## Final for Constants

```python
from typing import Final

MAX_RETRY: Final = 3
DEFAULT_TIMEOUT: Final[int] = 30

class Config:
    MAX_CONNECTIONS: Final = 100
```

## Self Type

```python
from typing import Self

class Node:
    def copy(self) -> Self:
        return Node(**self.__dict__)

    def create_child(self) -> Self:
        return self.__class__()
```

## LiteralString for Safe SQL

```python
from typing import LiteralString

def execute_query(sql: LiteralString) -> None:
    # Only literal strings pass type checking
    ...

execute_query("SELECT * FROM users")  # OK
# execute_query(user_input)  # TypeError!
```

## The Bottom Line

**Use modern type hints: `int | str`, `type Alias = ...`, `Final`, `Self`.**
