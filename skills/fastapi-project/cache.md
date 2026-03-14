---
name: fastapi-cache
description: FastAPI 缓存最佳实践 - Redis 缓存装饰器、缓存策略、分布式锁
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


# 便捷访问
async def get_redis() -> redis.Redis:
    return await RedisClient.get_connection()
```

### 应用启动/关闭

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.core.cache import RedisClient


@asynccontextmanager
async def lifespan(app: FastAPI):
    # 启动时连接 Redis
    await RedisClient.get_connection()
    yield
    # 关闭时断开连接
    await RedisClient.close()


app = FastAPI(lifespan=lifespan)
```

---

## 缓存装饰器

### 基础缓存装饰器

```python
# app/cache/decorator.py
import json
import hashlib
from functools import wraps
from typing import Optional, Callable, Any
import redis.asyncio as redis


def cache(
    key_prefix: str,
    expire: int = 300,
    skip_cache: bool = False
):
    """缓存装饰器

    Args:
        key_prefix: 缓存键前缀
        expire: 过期时间（秒），默认 5 分钟
        skip_cache: 是否跳过缓存（用于测试）
    """
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, **kwargs) -> Any:
            # 获取 Redis 连接
            r = await get_redis()

            # 构建缓存键
            cache_key = _build_cache_key(key_prefix, func.__name__, args, kwargs)

            # 尝试获取缓存
            if not skip_cache:
                cached = await r.get(cache_key)
                if cached:
                    return json.loads(cached)

            # 执行原函数
            result = await func(*args, **kwargs)

            # 存入缓存
            if result is not None:
                await r.setex(
                    cache_key,
                    expire,
                    json.dumps(result, default=str)
                )

            return result

        return wrapper
    return decorator


def _build_cache_key(prefix: str, func_name: str, args: tuple, kwargs: dict) -> str:
    """构建缓存键"""
    # 用函数名和参数生成唯一键
    key_data = f"{func_name}:{args}:{kwargs}"
    key_hash = hashlib.md5(key_data.encode()).hexdigest()[:8]
    return f"{prefix}:{func_name}:{key_hash}"


# 使用示例
@cache(key_prefix="user", expire=600)
async def get_user_by_id(user_id: int) -> dict:
    """获取用户信息，缓存 10 分钟"""
    # 实际从数据库查询
    return {"id": user_id, "name": "User"}
```

### 带条件缓存

```python
# app/cache/conditional.py
from functools import wraps
from typing import Optional, Callable, Any


def cache_if(
    key_prefix: str,
    expire: int = 300,
    condition: Optional[Callable] = None
):
    """条件缓存装饰器

    Args:
        key_prefix: 缓存键前缀
        expire: 过期时间（秒）
        condition: 条件函数，返回 True 才缓存
    """
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, **kwargs) -> Any:
            r = await get_redis()
            cache_key = _build_cache_key(key_prefix, func.__name__, args, kwargs)

            # 尝试获取缓存
            cached = await r.get(cache_key)
            if cached:
                return json.loads(cached)

            # 执行函数
            result = await func(*args, **kwargs)

            # 检查条件
            should_cache = (
                result is not None and
                (condition is None or condition(result))
            )

            if should_cache:
                await r.setex(cache_key, expire, json.dumps(result, default=str))

            return result

        return wrapper
    return decorator


# 使用：只缓存非空结果
@cache_if(key_prefix="items", condition=lambda x: x and len(x) > 0)
async def search_items(query: str) -> list:
    """搜索项目，空结果不缓存"""
    ...
```

---

## 缓存管理

### 缓存清理

```python
# app/cache/manager.py
import redis.asyncio as redis
from typing import Optional
import json


class CacheManager:
    """缓存管理器"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def get(self, key: str) -> Optional[dict]:
        """获取缓存"""
        cached = await self.redis.get(key)
        if cached:
            return json.loads(cached)
        return None

    async def set(self, key: str, value: dict, expire: int = 300):
        """设置缓存"""
        await self.redis.setex(key, expire, json.dumps(value, default=str))

    async def delete(self, key: str):
        """删除缓存"""
        await self.redis.delete(key)

    async def delete_pattern(self, pattern: str):
        """删除匹配的所有缓存"""
        keys = []
        async for key in self.redis.scan_iter(match=pattern):
            keys.append(key)
        if keys:
            await self.redis.delete(*keys)

    async def clear_prefix(self, prefix: str):
        """清除指定前缀的所有缓存"""
        await self.delete_pattern(f"{prefix}:*")


# 使用
cache_manager = CacheManager(await get_redis())

# 清除用户相关缓存
await cache_manager.clear_prefix("user")

# 清除所有缓存
await cache_manager.delete_pattern("*")
```

### 缓存版本管理

```python
# app/cache/versioned.py
from functools import wraps
import redis.asyncio as redis
import json


class VersionedCache:
    """带版本的缓存"""

    def __init__(self, redis_client: redis.Redis, version: str = "v1"):
        self.redis = redis_client
        self.version = version

    async def get(self, key: str) -> Optional[dict]:
        full_key = f"{self.version}:{key}"
        cached = await self.redis.get(full_key)
        return json.loads(cached) if cached else None

    async def set(self, key: str, value: dict, expire: int = 300):
        full_key = f"{self.version}:{key}"
        await self.redis.setex(full_key, expire, json.dumps(value, default=str))

    async def invalidate_prefix(self, prefix: str):
        """清除版本下所有缓存"""
        pattern = f"{self.version}:{prefix}:*"
        keys = []
        async for key in self.redis.scan_iter(match=pattern):
            keys.append(key)
        if keys:
            await self.redis.delete(*keys)

    async def bump_version(self):
        """升级版本（清除旧版本缓存）"""
        self.version = f"v{int(self.version[1:]) + 1}"
```

---

## 分布式锁

### 基础分布式锁

```python
# app/cache/lock.py
import asyncio
import redis.asyncio as redis
from typing import Optional


class DistributedLock:
    """Redis 分布式锁"""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def acquire(
        self,
        lock_key: str,
        expire: int = 10,
        retry_times: int = 3,
        retry_delay: float = 0.2
    ) -> bool:
        """获取锁

        Args:
            lock_key: 锁键
            expire: 锁过期时间（秒）
            retry_times: 重试次数
            retry_delay: 重试延迟（秒）
        """
        for _ in range(retry_times):
            # 使用 SET NX EX 原子操作
            acquired = await self.redis.set(
                lock_key,
                "1",
                nx=True,  # 只有不存在时才设置
                ex=expire  # 过期时间
            )
            if acquired:
                return True
            await asyncio.sleep(retry_delay)
        return False

    async def release(self, lock_key: str):
        """释放锁"""
        await self.redis.delete(lock_key)

    async def __aenter__(self):
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.release(self.lock_key)


# 使用
async def process_something(lock_key: str):
    lock = DistributedLock(await get_redis())

    if await lock.acquire(lock_key):
        try:
            # 执行需要锁的操作
            await do_something()
        finally:
            await lock.release(lock_key)
    else:
        # 获取锁失败
        raise Exception("Could not acquire lock")
```

### 上下文管理器方式

```python
# app/cache/lock.py
import asyncio
import redis.asyncio as redis
from contextlib import asynccontextmanager


@asynccontextmanager
async def distributed_lock(
    redis_client: redis.Redis,
    lock_key: str,
    expire: int = 10,
    timeout: float = 5.0
):
    """分布式锁上下文管理器

    Usage:
        async with distributed_lock(redis, "task:123"):
            await process_task()
    """
    lock_acquired = False

    try:
        # 尝试获取锁
        start_time = asyncio.get_event_loop().time()
        while True:
            acquired = await redis_client.set(
                lock_key,
                "1",
                nx=True,
                ex=expire
            )

            if acquired:
                lock_acquired = True
                break

            # 检查超时
            if asyncio.get_event_loop().time() - start_time >= timeout:
                raise TimeoutError(f"Could not acquire lock: {lock_key}")

            await asyncio.sleep(0.1)

        yield

    finally:
        if lock_acquired:
            await redis_client.delete(lock_key)


# 使用
async def process_task(task_id: int):
    redis_client = await get_redis()

    async with distributed_lock(redis_client, f"task:{task_id}"):
        # 任务处理逻辑
        await do_task(task_id)
```

---

## 缓存策略

### 缓存穿透防护

```python
# app/cache/strategies.py
import json
from functools import wraps
import redis.asyncio as redis


def cache_with_fallback(
    key_prefix: str,
    expire: int = 300,
    null_expire: int = 60
):
    """缓存穿透防护：缓存空值

    Args:
        key_prefix: 缓存键前缀
        expire: 正常值过期时间
        null_expire: 空值过期时间（较短）
    """
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            r = await get_redis()
            cache_key = _build_cache_key(key_prefix, func.__name__, args, kwargs)

            # 尝试获取
            cached = await r.get(cache_key)
            if cached:
                return json.loads(cached)

            # 执行函数
            result = await func(*args, **kwargs)

            # 缓存结果
            if result:
                await r.setex(cache_key, expire, json.dumps(result, default=str))
            else:
                # 空值也缓存，但时间较短
                await r.setex(cache_key, null_expire, json.dumps(result, default=str))

            return result

        return wrapper
    return decorator


# 使用
@cache_with_fallback(key_prefix="user", expire=600, null_expire=30)
async def get_user_by_email(email: str) -> Optional[dict]:
    """用户不存在时也缓存，防止缓存穿透"""
    ...
```

### 缓存更新策略

```python
# app/cache/strategies.py
from enum import Enum


class CacheStrategy(Enum):
    CACHE_ASIDE = "cache_aside"      # 旁路缓存
    WRITE_THROUGH = "write_through"  # 写穿透
    WRITE_BACK = "write_back"        # 写回


# CACHE_ASIDE（推荐）
async def get_user_cache_aside(user_id: int) -> Optional[dict]:
    """旁路缓存：先查缓存，缓存没有再查数据库"""
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


async def update_user_cache_aside(user_id: int, data: dict):
    """更新时删除缓存（让下次读取时重新加载）"""
    # 1. 更新数据库
    await db.update_user(user_id, data)

    # 2. 删除缓存（不要更新缓存，可能导致不一致）
    r = await get_redis()
    await r.delete(f"user:{user_id}")
```

---

## 在 FastAPI 中使用

### 依赖注入方式

```python
# app/api/items.py
from fastapi import APIRouter, Depends
from typing import Optional

router = APIRouter()


@router.get("/items/{item_id}")
async def get_item(
    item_id: int,
    cache: CacheManager = Depends(get_cache_manager)
):
    """使用缓存管理器"""
    cache_key = f"item:{item_id}"

    # 尝试从缓存获取
    cached = await cache.get(cache_key)
    if cached:
        return cached

    # 缓存没有，从数据库获取
    item = await db.get_item(item_id)

    if item:
        await cache.set(cache_key, item.dict(), expire=300)

    return item


@router.put("/items/{item_id}")
async def update_item(
    item_id: int,
    data: ItemUpdate,
    cache: CacheManager = Depends(get_cache_manager)
):
    """更新时清除缓存"""
    item = await db.update_item(item_id, data)

    # 清除缓存
    await cache.delete(f"item:{item_id}")

    return item
```

---

## 最佳实践

### 1. 缓存键命名规范

```python
# ✅ 推荐：使用冒号分隔
user:123
user:123:profile
item:search:python

# ❌ 避免：使用特殊字符或过长的键
user/123/profile
this_is_a_very_long_key_name_that_is_not_readable
```

### 2. 缓存时间选择

| 数据类型 | 推荐缓存时间 | 原因 |
|---------|-------------|------|
| 用户信息 | 5-15 分钟 | 变化不频繁 |
| 列表数据 | 1-5 分钟 | 可能频繁变化 |
| 配置数据 | 30-60 分钟 | 几乎不变 |
| 搜索结果 | 2-5 分钟 | 变化频繁 |

### 3. 缓存监控

```python
# app/cache/monitor.py
import redis.asyncio as redis
from dataclasses import dataclass


@dataclass
class CacheStats:
    hits: int = 0
    misses: int = 0

    @property
    def hit_rate(self) -> float:
        total = self.hits + self.misses
        return self.hits / total if total > 0 else 0


stats = CacheStats()


async def track_cache_hit():
    stats.hits += 1


async def track_cache_miss():
    stats.misses += 1


# 使用
@cache(key_prefix="user")
async def get_user(user_id: int):
    # 记录缓存命中/未命中
    ...
```

### 4. 注意事项

- 不要缓存敏感信息（密码、Token 等）
- 设置合理的过期时间
- 监控缓存命中率
- 做好缓存穿透/击穿/雪崩防护

---

## 相关文件

- [config.md](./config.md) - 配置管理
- [DI.md](./DI.md) - 依赖注入
- [database.md](./database.md) - 数据库配置
- [middleware.md](./middleware.md) - 中间件（限流）
