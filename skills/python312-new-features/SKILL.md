---
name: python312-new-features
description: Use when writing Python 3.12+ code and want to leverage new syntax features like PEP 701 f-strings, PEP 695 type parameters, or PEP 709 comprehension inlining
---

# Python 3.12 New Features

## Overview
**Python 3.12 brings major syntax improvements: f-string changes (PEP 701), type parameter syntax (PEP 695), and faster comprehensions (PEP 709).**

## When to Use
- Using Python 3.12+ in pyproject.toml
- Writing complex f-strings with quotes/nesting
- Creating generic functions or type aliases
- Optimizing list/dict/set comprehensions

## PEP 701: F-string Improvements

```python
# Quote reuse (no more escaping)
f"This is the playlist: {', '.join(songs)}"

# Backslashes allowed
f"Line1: {line1}\nLine2: {line2}"

# Comments inside expressions
f"Total: {sum([  # Sum all items
    item.price for item in items
])}"

# Arbitrary nesting
f"Result: {f"nested: {f"value: {1+1}"}"}"
```

## PEP 695: Type Parameter Syntax

```python
# New type alias syntax
type Point = tuple[float, float]
type ApiResponse = dict[str, int | str]

# Generic functions
def max[T](args: Iterable[T]) -> T:
    return max(args)

# Generic classes
class Box[T]:
    def __init__(self, content: T):
        self.content = content

    def get(self) -> T:
        return self.content
```

## PEP 709: Comprehension Inlining

```python
# Up to 2x faster in Python 3.12
# No separate frame in tracebacks
# locals() includes outer variables

# List comprehension (inlined)
numbers = [x * 2 for x in range(1000)]

# Dict comprehension (inlined)
squares = {x: x*x for x in range(10)}
```

## Improved Error Messages

```python
# NameError with suggestion
>>> sys.version_info
NameError: name 'sys' is not defined. Did you forget to import 'sys'?

# ImportError with suggestion
>>> from collections import chainmap
ImportError: cannot import name 'chainmap' from 'collections'. Did you mean: 'ChainMap'?
```

## Performance Improvements

| Module | Improvement |
|--------|-------------|
| `asyncio` | ~75% faster |
| `tokenize` | ~64% faster |
| `hashlib` | HACL* verified |

## The Bottom Line

**Python 3.12 = Better f-strings, new type syntax, faster comprehensions.**
