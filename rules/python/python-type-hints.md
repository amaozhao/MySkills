---
name: python-type-hints
description: Python 类型提示 - PEP 695、TypedDict、Final、Self
paths: **/*.py
---

# Python Type Hints

## 概述

现代 Python 类型提示使用 3.12+ 语法：`|` 联合类型、`type` 别名、`Final` 常量。

## 现代语法 (3.10+)

```python
# 使用 | 代替 Union
def process(value: int | str) -> None: ...

# 使用 | None 代替 Optional
name: str | None = None

# List 和 dict 类型
from collections.abc import Iterable
prices: list[float] = []
data: dict[str, int] = {}
```

## PEP 695: 类型别名

```python
# 旧写法
from typing import TypeAlias
Point: TypeAlias = tuple[float, float]

# 新写法 (Python 3.12+)
type Point = tuple[float, float]
type ApiResponse = dict[str, int | str]
```

## PEP 695: 泛型函数/类

```python
def first[T](items: Iterable[T]) -> T | None:
    return next(iter(items), None)

class Container[T]:
    def __init__(self, content: T):
        self.content = content

    def get(self) -> T:
        return self.content
```

## TypedDict 必选/可选

```python
from typing import TypedDict, Required, NotRequired, ReadOnly

class OrderRequest(TypedDict, total=False):
    symbol: Required[str]        # 必填
    quantity: NotRequired[int]   # 可选
    price: NotRequired[float]

class Config(TypedDict):
    debug: bool
    timeout: ReadOnly[int]       # 创建后不可变
```

## Final 常量

```python
from typing import Final

MAX_RETRY: Final = 3
DEFAULT_TIMEOUT: Final[int] = 30

class Config:
    MAX_CONNECTIONS: Final = 100
```

## Self 类型

```python
from typing import Self

class Node:
    def copy(self) -> Self:
        return Node(**self.__dict__)

    def create_child(self) -> Self:
        return self.__class__()
```

## LiteralString 安全 SQL

```python
from typing import LiteralString

def execute_query(sql: LiteralString) -> None:
    # 只有字面量字符串通过类型检查
    ...

execute_query("SELECT * FROM users")  # OK
# execute_query(user_input)  # TypeError!
```
