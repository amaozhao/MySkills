---
name: python-new-features
description: Python 3.12 新特性 - f-string、类型参数、推导式内联
---

# Python 3.12 New Features

## 概述

Python 3.12 主要语法改进：f-string 变化（PEP 701）、类型参数语法（PEP 695）、推导式内联（PEP 709）。

## PEP 701: F-string 改进

```python
# 引用复用（无需转义）
f"This is the playlist: {', '.join(songs)}"

# 允许反斜杠
f"Line1: {line1}\nLine2: {line2}"

# 表达式内注释
f"Total: {sum([  # 求和
    item.price for item in items
])}"

# 任意嵌套
f"Result: {f"nested: {f"value: {1+1}"}"}"
```

## PEP 695: 类型参数语法

```python
# 新类型别名语法
type Point = tuple[float, float]
type ApiResponse = dict[str, int | str]

# 泛型函数
def max[T](args: Iterable[T]) -> T:
    return max(args)

# 泛型类
class Box[T]:
    def __init__(self, content: T):
        self.content = content

    def get(self) -> T:
        return self.content
```

## PEP 709: 推导式内联

```python
# Python 3.12 中快 2 倍
# 追踪回溯中不再单独显示帧
# locals() 包含外部变量

# 列表推导式（内联）
numbers = [x * 2 for x in range(1000)]

# 字典推导式（内联）
squares = {x: x*x for x in range(10)}
```

## 改进的错误信息

```python
# NameError 建议
>>> sys.version_info
NameError: name 'sys' is not defined. Did you forget to import 'sys'?

# ImportError 建议
>>> from collections import chainmap
ImportError: cannot import name 'chainmap' from 'collections'. Did you mean: 'ChainMap'?
```

## 性能提升

| 模块 | 提升 |
|------|------|
| `asyncio` | ~75% |
| `tokenize` | ~64% |
| `hashlib` | HACL* 验证 |

---

## 相关文件

- [type-hints.md](./type-hints.md) - 类型提示
- [style.md](./style.md) - 代码风格
