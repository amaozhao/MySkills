---
name: python-code-style
description: Use when writing Python code, writing docstrings, or reviewing code for PEP 8 compliance and pythonic patterns
---

# Python Code Style

## Overview
**PEP 8 guidelines, docstring conventions, and pythonic patterns for readable, maintainable Python code.**

## When to Use
- Writing new Python code
- Reviewing code style
- Writing docstrings
- Converting code to be more Pythonic

## PEP 8 Basics

```python
# Naming
class MyClass: ...              # CamelCase for classes
def my_function(): ...         # snake_case for functions
MY_CONSTANT = 42               # UPPER_SNAKE_CASE for constants
_instance = SomeClass()        # _prefix for private

# Imports (on separate lines)
# Bad: import sys, os
# Good:
import sys
import os
from typing import Optional

# Line length: 88 characters (Black default)
# Line breaks: align with opening delimiter
result = some_function(
    arg1, arg2, arg3,
    arg4, arg5, arg6,
)

# Spaces around operators
# Bad: x=1+2
# Good: x = 1 + 2

# Blank lines
class MyClass:
    """Docstring."""            # 2 blank lines after docstring


    def method(self):           # 2 blank lines between methods
        ...
```

## Docstring Conventions

```python
"""Module-level docstring.

One blank line after, then imports.
"""

class Calculator:
    """Perform arithmetic operations.

    This class provides basic math functions.

    Example:
        >>> calc = Calculator()
        >>> calc.add(2, 3)
        5
    """

    def add(self, a: int, b: int) -> int:
        """Add two numbers.

        Args:
            a: First number.
            b: Second number.

        Returns:
            Sum of a and b.

        Raises:
            ValueError: If inputs are not integers.
        """
        if not isinstance(a, int) or not isinstance(b, int):
            raise ValueError("Inputs must be integers")
        return a + b
```

## Docstring Formats

| Format | Description |
|--------|-------------|
| Google | `Args:`, `Returns:`, `Raises:` |
| NumPy | `Parameters`, `Returns`, `Raises` |
| Sphinx | `:param:`, `:return:`, `:raises:` |

Choose one and be consistent.

## Pythonic Patterns

```python
# Loop with index
for i, item in enumerate(items):
    ...

# Multiple assignments
a, b = b, a  # Swap

# List comprehension (preferred)
squares = [x*x for x in range(10)]

# Generator for large sequences
def get_lines(filename):
    with open(filename) as f:
        for line in f:
            yield line

# Context managers
with open("file.txt") as f:
    data = f.read()

# Multiple context managers
with open("input.txt") as infile, open("output.txt", "w") as outfile:
    outfile.write(infile.read())

# Ternary operator
value = "positive" if x > 0 else "non-positive"

# Check membership (not len)
if items:  # Not if len(items) > 0
    ...

# Merge dictionaries (3.9+)
d1 = {"a": 1}
d2 = {"b": 2}
merged = d1 | d2
```

## Anti-Patterns

| Bad | Good |
|-----|------|
| `for i in range(len(items))` | `for item in items` |
| `if len(items) == 0` | `if not items` |
| `list.append` in loop | List comprehension |
| `try/except: pass` | Log or handle specific exception |
| Mutable default args | `def func(arg=None): ...; if arg is None: arg = []` |

## The Bottom Line

**PEP 8 + consistent docstrings + pythonic patterns = readable code.**
