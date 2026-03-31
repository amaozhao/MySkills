---
name: fastapi-cache
description: FastAPI 缓存最佳实践 - Redis 缓存装饰器、缓存策略、分布式锁
paths: **/*.py
---

# FastAPI 缓存

## 概述

使用 Redis 实现应用级缓存，减少数据库查询压力，提升响应速度。

## 基础配置

### Redis 连接

```python
# app/core/cache.py
import redis.asyncio as redis
from typing import Optional
from app.core.config import settings


class RedisClient:
    """Redis 客户端单例"""

    _instance: Optional[redis.Redis] = None

    @classmethod
    async def get_connection(cls) -> redis.Redis:
        if cls._instance is None:
            cls._instance = redis.from_url(
                settings.REDIS_URL,
                encoding="utf-8",
                decode_responses=True,
            )
        return cls._instance

    @classmethod
    async def close(cls):
        if cls._instance:
            await cls._instance.close()
            cls._instance = None


async def get_redis() -> redis.Redis:
    return await RedisClient.get_connection()
```

## 缓存装饰器

```python
# app/cache/decorator.py
import json
import hashlib
from functools import wraps
from typing import Callable, Any


def cache(key_prefix: str, expire: int = 300, skip_cache: bool = False):
    """缓存装饰器"""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, **kwargs) -> Any:
            r = await get_redis()
            cache_key = _build_cache_key(key_prefix, func.__name__, args, kwargs)

            if not skip_cache:
                cached = await r.get(cache_key)
                if cached:
                    return json.loads(cached)

            result = await func(*args, **kwargs)

            if result is not None:
                await r.setex(cache_key, expire, json.dumps(result, default=str))

            return result
        return wrapper
    return decorator


def _build_cache_key(prefix: str, func_name: str, args: tuple, kwargs: dict) -> str:
    key_data = f"{func_name}:{args}:{kwargs}"
    key_hash = hashlib.md5(key_data.encode()).hexdigest()[:8]
    return f"{prefix}:{func_name}:{key_hash}"


# 使用示例
@cache(key_prefix="user", expire=600)
async def get_user_by_id(user_id: int) -> dict:
    return {"id": user_id, "name": "User"}
```

## 缓存策略

### Cache-Aside（推荐）

```python
async def get_user_cache_aside(user_id: int):
    r = await get_redis()
    cache_key = f"user:{user_id}"

    # 1. 查缓存
    cached = await r.get(cache_key)
    if cached:
        return json.loads(cached)

    # 2. 缓存没有，查数据库
    user = await db.get_user(user_id)

    # 3. 存入缓存
    if user:
        await r.setex(cache_key, 600, json.dumps(user, default=str))

    return user
```

## 分布式锁

```python
# app/cache/lock.py
import asyncio
import redis.asyncio as redis


class DistributedLock:
    """Redis 分布式锁"""

    async def acquire(self, lock_key: str, expire: int = 10, retry_times: int = 3) -> bool:
        for _ in range(retry_times):
            acquired = await self.redis.set(lock_key, "1", nx=True, ex=expire)
            if acquired:
                return True
            await asyncio.sleep(0.2)
        return False

    async def release(self, lock_key: str):
        await self.redis.delete(lock_key)


# 使用
async with distributed_lock(redis, f"task:{task_id}"):
    await process_task()
```

## 缓存时间选择

| 数据类型 | 推荐缓存时间 | 原因 |
|---------|-------------|------|
| 用户信息 | 5-15 分钟 | 变化不频繁 |
| 列表数据 | 1-5 分钟 | 可能频繁变化 |
| 配置数据 | 30-60 分钟 | 几乎不变 |
| 搜索结果 | 2-5 分钟 | 变化频繁 |
