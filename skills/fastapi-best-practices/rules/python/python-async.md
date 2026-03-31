---
name: python-async
description: Python 异步编程 - asyncio、TaskGroups、Semaphore、timeout
---

# Python Async Patterns

## 概述

Python asyncio 异步模式：TaskGroups 错误处理、Semaphore 限流、timeout 超时控制。

## Task Groups (Python 3.11+)

```python
async def process_all(orders: list[Order]) -> list[Result]:
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(process_order(o)) for o in orders]

    # 收集结果，一个异常则全部失败
    results = [t.result() for t in tasks]
    return [r for r in results if not isinstance(r, Exception)]
```

## 并发执行

```python
async def fetch_all_prices(symbols: list[str]) -> dict[str, float]:
    tasks = [fetch_price(s) for s in symbols]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return dict(zip(symbols, results))
```

## Semaphore 限流

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

# 使用
async with RateLimiter(10, 0.1):  # 10并发，0.1秒间隔
    await fetch_price("BTC")
```

## Timeout 处理

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

## 异步上下文管理器

```python
class Connection:
    async def __aenter__(self):
        self.conn = await connect()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.conn.close()

# 使用
async with Connection() as conn:
    await conn.execute(query)
```

## asynccontextmanager

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def managed_resource():
    resource = await acquire()
    try:
        yield resource
    finally:
        await resource.release()

# 使用
async with managed_resource() as r:
    await r.execute()
```

## 常见错误

| 错误 | 修复 |
|------|------|
| async 中使用 time.sleep() | 使用 asyncio.sleep() |
| 忘记 await | 检查所有 IO 操作 |
| 无超时控制 | 使用 asyncio.timeout() |
| 并发过高 | 使用 asyncio.Semaphore |
