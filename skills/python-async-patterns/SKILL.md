---
name: python-async-patterns
description: Use when writing async Python code with asyncio, including TaskGroups, rate limiting, timeouts, context managers, or concurrent task execution
---

# Python Async Patterns

## Overview
**Python async patterns using asyncio: TaskGroups for error handling, semaphores for rate limiting, timeouts, and proper context lifecycle management.**

## When to Use
- Writing async/await code
- Running concurrent tasks
- Need rate limiting
- Handling timeouts
- Managing async resources

## Task Groups (Python 3.11+)

```python
async def process_all(orders: list[Order]) -> list[Result]:
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(process_order(o)) for o in orders]

    # Collect results, raise on first exception
    results = [t.result() for t in tasks]
    return [r for r in results if not isinstance(r, Exception)]
```

## Concurrent Execution

```python
async def fetch_all_prices(symbols: list[str]) -> dict[str, float]:
    tasks = [fetch_price(s) for s in symbols]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return dict(zip(symbols, results))
```

## Rate Limiting with Semaphore

```python
class RateLimiter:
    def __init__(self, max_concurrent: int, rate: float):
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.rate = rate

    async def __aenter__(self):
        async with self.semaphore:
            await asyncio.sleep(self.rate)
        return self

    async def __aexit__(self, *args):
        pass

# Usage
async with RateLimiter(10, 0.1):  # 10 concurrent, 0.1s between
    await fetch_price("BTC")
```

## Timeout Handling

```python
from asyncio import timeout, TimeoutError

async def fetch_with_timeout(symbol: str, timeout_sec: float = 5.0):
    try:
        async with timeout(timeout_sec):
            return await exchange.fetch_price(symbol)
    except TimeoutError:
        logger.warning(f"Timeout fetching {symbol}")
        return None
```

## Async Context Manager

```python
class Connection:
    async def __aenter__(self):
        self.conn = await connect()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.conn.close()

# Usage
async with Connection() as conn:
    await conn.execute(query)
```

## Contextlib asynccontextmanager

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def managed_resource():
    resource = await acquire()
    try:
        yield resource
    finally:
        await resource.release()

# Usage
async with managed_resource() as r:
    await r.execute()
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `time.sleep()` in async | Use `asyncio.sleep()` |
| Forgetting `await` | Check all IO operations |
| No timeout | Wrap with `asyncio.timeout()` |
| Too many concurrent | Use `asyncio.Semaphore` |

## The Bottom Line

**Use TaskGroup for error handling, Semaphore for rate limiting, timeout for safety.**
