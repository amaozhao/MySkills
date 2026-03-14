---
name: fastapi-middleware
description: FastAPI 中间件最佳实践 - CORS、限流、请求日志、健康检查
---

# FastAPI 中间件

## 概述

中间件是在请求到达路由之前或响应返回之后执行的函数。FastAPI 支持 ASGI 中间件，用于处理跨域、限流、日志、安全等横切关注点。

## CORS 中间件

### 基础配置

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# CORS 配置
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],  # 允许的源
    allow_credentials=True,                    # 允许凭证
    allow_methods=["*"],                       # 允许的方法
    allow_headers=["*"],                       # 允许的头
)
```

### 生产环境配置

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.core.config import settings

app = FastAPI()

# 从配置读取 CORS origins
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.parsed_cors_origins,  # ["https://yourdomain.com"]
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    # 预检请求缓存时间
    max_age=600,
)
```

### 动态允许域

```python
# app/middleware/cors.py
from starlette.middleware.cors import CORSMiddleware
from starlette.requests import Request
from starlette.responses import Response


class DynamicCORSMiddleware(CORSMiddleware):
    """动态 CORS 中间件，根据请求动态允许源"""

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await super().__call__(scope, receive, send)

        # 从数据库或配置获取允许的源
        allowed_origins = get_allowed_origins_from_db()

        # 临时添加当前请求的 origin
        # 这里可以实现更复杂的逻辑
        return await super().__call__(scope, receive, send)
```

---

## 请求日志中间件

### 基础日志

```python
# app/middleware/logging.py
import time
import logging
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

logger = logging.getLogger(__name__)


class RequestLoggingMiddleware(BaseHTTPMiddleware):
    """请求日志中间件"""

    async def dispatch(self, request: Request, call_next):
        start_time = time.time()

        # 请求日志
        logger.info(
            f"Request: {request.method} {request.url.path} "
            f"from {request.client.host if request.client else 'unknown'}"
        )

        response = await call_next(request)

        # 响应日志
        process_time = time.time() - start_time
        logger.info(
            f"Response: {response.status_code} "
            f"took {process_time:.3f}s"
        )

        # 添加响应头
        response.headers["X-Process-Time"] = str(process_time)

        return response
```

### 结构化日志

```python
# app/middleware/logging.py
import time
import logging
import json
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response

logger = logging.getLogger("api")


class StructuredLoggingMiddleware(BaseHTTPMiddleware):
    """结构化日志中间件"""

    async def dispatch(self, request: Request, call_next) -> Response:
        start_time = time.time()
        request_id = request.headers.get("X-Request-ID", "")

        log_data = {
            "request_id": request_id,
            "method": request.method,
            "path": request.url.path,
            "client_ip": request.client.host if request.client else None,
        }

        try:
            response = await call_next(request)
            log_data["status_code"] = response.status_code
        except Exception as e:
            log_data["error"] = str(e)
            raise
        finally:
            log_data["duration_ms"] = int((time.time() - start_time) * 1000)
            logger.info(json.dumps(log_data))

        return response
```

---

## 限流中间件

### 简单内存限流

```python
# app/middleware/rate_limit.py
import time
from collections import defaultdict
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse


class RateLimitMiddleware(BaseHTTPMiddleware):
    """简单内存限流中间件"""

    def __init__(self, app, requests_per_minute: int = 60):
        super().__init__(app)
        self.requests_per_minute = requests_per_minute
        self.requests = defaultdict(list)

    async def dispatch(self, request: Request, call_next):
        client_ip = request.client.host if request.client else "unknown"
        current_time = time.time()

        # 清理超过1分钟的请求记录
        self.requests[client_ip] = [
            t for t in self.requests[client_ip]
            if current_time - t < 60
        ]

        # 检查是否超限
        if len(self.requests[client_ip]) >= self.requests_per_minute:
            return JSONResponse(
                status_code=429,
                content={"error": "Too Many Requests"}
            )

        # 记录请求
        self.requests[client_ip].append(current_time)

        response = await call_next(request)
        return response
```

### Redis 分布式限流

```python
# app/middleware/rate_limit.py
import time
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse
import redis.asyncio as redis


class RedisRateLimitMiddleware(BaseHTTPMiddleware):
    """Redis 分布式限流"""

    def __init__(
        self,
        app,
        redis_url: str = "redis://localhost:6379/0",
        requests_per_minute: int = 60,
        key_prefix: str = "rate_limit"
    ):
        super().__init__(app)
        self.redis_url = redis_url
        self.requests_per_minute = requests_per_minute
        self.key_prefix = key_prefix
        self._redis = None

    async def dispatch(self, request: Request, call_next):
        if self._redis is None:
            self._redis = await redis.from_url(self.redis_url)

        client_ip = request.client.host if request.client else "unknown"
        key = f"{self.key_prefix}:{client_ip}"

        # 使用 Redis 原子操作
        current = await self.redis.incr(key)

        # 首次设置过期时间
        if current == 1:
            await self.redis.expire(key, 60)

        if current > self.requests_per_minute:
            return JSONResponse(
                status_code=429,
                content={
                    "error": "Too Many Requests",
                    "retry_after": 60
                },
                headers={"Retry-After": "60"}
            )

        response = await call_next(request)
        return response
```

---

## 健康检查

### 基础健康检查

```python
# app/api/health.py
from fastapi import APIRouter
from pydantic import BaseModel

router = APIRouter()


class HealthResponse(BaseModel):
    status: str
    version: str


@router.get("/health", response_model=HealthResponse)
async def health_check():
    """基础健康检查"""
    return HealthResponse(
        status="ok",
        version="1.0.0"
    )
```

### 完整健康检查

```python
# app/api/health.py
from fastapi import APIRouter, Depends
from pydantic import BaseModel
from typing import Optional
from datetime import datetime

router = APIRouter()


class HealthResponse(BaseModel):
    status: str
    timestamp: str
    version: str
    checks: dict


class DatabaseHealth(BaseModel):
    status: str
    latency_ms: Optional[int] = None


class RedisHealth(BaseModel):
    status: str
    latency_ms: Optional[int] = None


async def check_database() -> DatabaseHealth:
    """检查数据库连接"""
    import time
    try:
        start = time.time()
        # 实际项目中执行 SELECT 1
        # await db.execute(text("SELECT 1"))
        latency = int((time.time() - start) * 1000)
        return DatabaseHealth(status="ok", latency_ms=latency)
    except Exception as e:
        return DatabaseHealth(status="error")


async def check_redis() -> RedisHealth:
    """检查 Redis 连接"""
    import time
    try:
        start = time.time()
        # 实际项目中 ping Redis
        # await redis.ping()
        latency = int((time.time() - start) * 1000)
        return RedisHealth(status="ok", latency_ms=latency)
    except Exception:
        return RedisHealth(status="error")


@router.get("/health")
async def health_check():
    """完整健康检查"""
    db_health = await check_database()
    redis_health = await check_redis()

    # 整体状态
    all_ok = (
        db_health.status == "ok" and
        redis_health.status == "ok"
    )

    return HealthResponse(
        status="ok" if all_ok else "degraded",
        timestamp=datetime.utcnow().isoformat(),
        version="1.0.0",
        checks={
            "database": db_health.model_dump(),
            "redis": redis_health.model_dump(),
        }
    )


@router.get("/health/live")
async def liveness():
    """K8s 存活探针"""
    return {"status": "ok"}


@router.get("/health/ready")
async def readiness():
    """K8s 就绪探针"""
    db_health = await check_database()
    if db_health.status != "ok":
        return {"status": "not_ready", "reason": "database unavailable"}
    return {"status": "ready"}
```

---

## 中间件注册

### 统一注册

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.middleware.logging import RequestLoggingMiddleware
from app.middleware.rate_limit import RateLimitMiddleware
from app.core.config import settings


def create_app() -> FastAPI:
    app = FastAPI(title="FastAPI App")

    # 中间件顺序：从下往上执行
    # 1. 最后添加的最先执行

    # 限流（最先处理请求）
    app.add_middleware(
        RateLimitMiddleware,
        requests_per_minute=settings.RATE_LIMIT_PER_MINUTE,
    )

    # 日志
    app.add_middleware(RequestLoggingMiddleware)

    # CORS（最后处理）
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.parsed_cors_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    return app


app = create_app()
```

---

## 常见中间件组合

### API 项目常用组合

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from starlette.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI()

# 1. GZip 压缩
app.add_middleware(GZipMiddleware, minimum_size=1000)

# 2. 可信主机（防止主机头攻击）
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["yourdomain.com", "*.yourdomain.com"]
)

# 3. CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://yourdomain.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["Authorization"],
)

# 4. 自定义中间件
app.add_middleware(RequestLoggingMiddleware)
app.add_middleware(RateLimitMiddleware, requests_per_minute=100)
```

---

## 最佳实践

### 1. 中间件顺序

```python
# 正确顺序（从外到内）
app.add_middleware(TrustedHostMiddleware)    # 1. 主机检查
app.add_middleware(GZipMiddleware)          # 2. 压缩
app.add_middleware(CORSMiddleware)          # 3. CORS
app.add_middleware(RateLimitMiddleware)     # 4. 限流
app.add_middleware(RequestLoggingMiddleware) # 5. 日志
```

### 2. 避免在中间件中进行耗时操作

```python
# ❌ 错误：耗时操作
async def dispatch(self, request, call_next):
    result = await slow_database_query()  # 不要在这里做！
    return await call_next(request)

# ✅ 正确：快速返回
async def dispatch(self, request, call_next):
    # 只做快速检查
    if is_rate_limited(request):
        return error_response()
    return await call_next(request)
```

### 3. 合理使用依赖注入

```python
# 中间件中获取依赖
from fastapi import Request
from app.core.config import get_settings


class MyMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # 方式1：直接从 app 获取
        settings = request.app.state.settings

        # 方式2：使用 Depends（在路由中）
        # 中间件不适合用 Depends

        return await call_next(request)
```

---

## 相关文件

- [config.md](./config.md) - 配置管理
- [DI.md](./DI.md) - 依赖注入
- [cache.md](./cache.md) - Redis 缓存
- [auth.md](./auth.md) - 认证中间件
