---
name: python-error-handling
description: Use when writing error handling code in Python, especially with custom exception hierarchies, exception chaining, exception groups, or context managers
---

# Python Error Handling

## Overview
**Python error handling best practices: specific exceptions, custom hierarchies, exception chaining, notes for context, and context managers for cleanup.**

## When to Use
- Writing try/except blocks
- Creating custom exceptions
- Handling multiple exceptions
- Adding context to errors
- Resource cleanup

## Be Specific with Exceptions

```python
# Good: Specific exceptions
try:
    await connection.execute(query)
except OSError as err:
    logger.error(f"Database connection failed: {err}")
except ValueError as err:
    logger.error(f"Invalid data format: {err}")

# Bad: Too broad
except Exception:
    pass
```

## Custom Exception Hierarchy

```python
class AppError(Exception):
    """Base exception for application errors."""
    pass

class ValidationError(AppError):
    """Validation-related errors."""
    pass

class DatabaseError(AppError):
    """Database-related errors."""
    pass

class NotFoundError(DatabaseError):
    """Resource not found."""
    pass

# Usage
if not user:
    raise NotFoundError(f"User {user_id} not found")
```

## Exception Chaining

```python
try:
    await process_order(order_id)
except OrderError as exc:
    raise OrderProcessingError(f"Failed {order_id}") from exc
```

## Exception Groups (Python 3.11+)

```python
try:
    results = await batch_process(orders)
except ExceptionGroup as eg:
    print(f"Failed: {len(eg.exceptions)} errors")
    for e in eg.exceptions:
        print(f"  - {e}")
```

## Adding Context with Notes (Python 3.11+)

```python
try:
    validate_order(order)
except ValidationError as e:
    e.add_note(f"Order ID: {order.id}")
    e.add_note(f"User: {order.user_id}")
    raise
```

## Context Managers for Resources

```python
# Good: Automatic cleanup
async with await get_connection() as conn:
    await conn.execute(query)

# Or with contextlib
from contextlib import asynccontextmanager

@asynccontextmanager
async def managed_file(path):
    f = await open_file(path)
    try:
        yield f
    finally:
        await f.close()
```

## Logging Best Practices

```python
import logging
logger = logging.getLogger(__name__)

try:
    risky_operation()
except SpecificError as e:
    logger.error(f"Operation failed: {e}", exc_info=True)  # Stack trace
except AnotherError as e:
    logger.warning(f"Expected failure: {e}")  # No stack trace
```

## The Bottom Line

**Be specific, chain exceptions, add context, use context managers.**
