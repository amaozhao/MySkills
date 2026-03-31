---
name: python-style
description: Python 代码风格 - PEP 8、docstring、pythonic 模式
---

# Python Code Style

## 概述

PEP 8 规范、docstring 约定和 pythonic 模式。

## PEP 8 基础

```python
# 命名
class MyClass: ...              # 类用 CamelCase
def my_function(): ...         # 函数用 snake_case
MY_CONSTANT = 42               # 常量用 UPPER_SNAKE_CASE
_instance = SomeClass()        # 私有用 _prefix

# 导入（每行一个）
import sys
import os
from typing import Optional

# 行长度：88 字符（Black 默认）
# 换行：对齐开始括号
result = some_function(
    arg1, arg2, arg3,
    arg4, arg5, arg6,
)
```

## Docstring 约定

```python
class Calculator:
    """执行算术运算。

    这个类提供基本的数学函数。

    Example:
        >>> calc = Calculator()
        >>> calc.add(2, 3)
        5
    """

    def add(self, a: int, b: int) -> int:
        """两个数相加。

        Args:
            a: 第一个数。
            b: 第二个数。

        Returns:
            a + b 的和。

        Raises:
            ValueError: 如果输入不是整数。
        """
        return a + b
```

## Pythonic 模式

```python
# 循环带索引
for i, item in enumerate(items):
    ...

# 交换
a, b = b, a

# 列表推导式（推荐）
squares = [x*x for x in range(10)]

# 生成器（大序列）
def get_lines(filename):
    with open(filename) as f:
        for line in f:
            yield line

# 上下文管理器
with open("file.txt") as f:
    data = f.read()

# 三元运算符
value = "positive" if x > 0 else "non-positive"

# 检查空（不用 len）
if items:
    ...

# 合并字典 (3.9+)
d1 = {"a": 1}
d2 = {"b": 2}
merged = d1 | d2
```

## 反模式

| 不好 | 好 |
|------|-----|
| `for i in range(len(items))` | `for item in items` |
| `if len(items) == 0` | `if not items` |
| 循环中 `list.append` | 列表推导式 |
| `try/except: pass` | 记录或处理具体异常 |
| 可变默认参数 | `def func(arg=None)` |
