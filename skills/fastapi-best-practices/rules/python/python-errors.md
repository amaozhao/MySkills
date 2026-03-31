---
name: python-errors
description: Python 错误处理 - 异常层次、链式异常、ExceptionGroup
---

# Python Error Handling

## 概述

Python 错误处理最佳实践：具体异常、自定义层次、异常链、context manager。

## 使用具体异常

```python
# 好：具体异常
try:
    await connection.execute(query)
except OSError as err:
    logger.error(f"Database connection failed: {err}")
except ValueError as err:
    logger.error(f"Invalid data format: {err}")

# 不好：太宽泛
except Exception:
    pass
```

## 自定义异常层次

```python
class AppError(Exception):
    """应用错误基类。"""
    pass

class ValidationError(AppError):
    """验证相关错误。"""
    pass

class DatabaseError(AppError):
    """数据库相关错误。"""
    pass

class NotFoundError(DatabaseError):
    """资源未找到。"""
    pass

# 使用
if not user:
    raise NotFoundError(f"User {user_id} not found")
```

## 异常链

```python
try:
    await process_order(order_id)
except OrderError as exc:
    raise OrderProcessingError(f"Failed {order_id}") from exc
```

## Exception Group (Python 3.11+)

```python
try:
    results = await batch_process(orders)
except ExceptionGroup as eg:
    print(f"Failed: {len(eg.exceptions)} errors")
    for e in eg.exceptions:
        print(f"  - {e}")
```

## 添加上下文 (Python 3.11+)

```python
try:
    validate_order(order)
except ValidationError as e:
    e.add_note(f"Order ID: {order.id}")
    e.add_note(f"User: {order.user_id}")
    raise
```

## 资源清理

```python
# 好：自动清理
async with await get_connection() as conn:
    await conn.execute(query)

# 或使用 contextlib
from contextlib import asynccontextmanager

@asynccontextmanager
async def managed_file(path):
    f = await open_file(path)
    try:
        yield f
    finally:
        await f.close()
```

## 日志最佳实践

```python
import logging
logger = logging.getLogger(__name__)

try:
    risky_operation()
except SpecificError as e:
    logger.error(f"Operation failed: {e}", exc_info=True)  # 堆栈跟踪
except AnotherError as e:
    logger.warning(f"Expected failure: {e}")  # 无堆栈
```
