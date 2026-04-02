---
name: fastapi-middleware
description: FastAPI 中间件最佳实践 - CORS、限流、请求日志、健康检查
paths: **/*.py
---

# FastAPI 中间件

## 概述

中间件是在请求到达路由之前或响应返回之后执行的函数。FastAPI 支持 ASGI 中间件。

## CORS 中间件

```python
# app/main.py
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## 请求日志中间件

```python
# app/middleware/logging.py
import time
from starlette.middleware.base import BaseHTTPMiddleware

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start_time = time.time()

        logger.info(f"Request: {request.method} {request.url.path}")

        response = await call_next(request)

        process_time = time.time() - start_time
        logger.info(f"Response: {response.status_code} took {process_time:.3f}s")

        response.headers["X-Process-Time"] = str(process_time)
        return response
```

## 限流中间件

```python
# app/middleware/rate_limit.py
from collections import defaultdict

class RateLimitMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, requests_per_minute: int = 60):
        super().__init__(app)
        self.requests_per_minute = requests_per_minute
        self.requests = defaultdict(list)

    async def dispatch(self, request: Request, call_next):
        client_ip = request.client.host if request.client else "unknown"
        current_time = time.time()

        self.requests[client_ip] = [
            t for t in self.requests[client_ip]
            if current_time - t < 60
        ]

        if len(self.requests[client_ip]) >= self.requests_per_minute:
            return JSONResponse(status_code=429, content={"error": "Too Many Requests"})

        self.requests[client_ip].append(current_time)
        return await call_next(request)
```

## 健康检查

```python
# app/api/health.py
from fastapi import APIRouter

router = APIRouter()


@router.get("/health")
async def health_check():
    return {"status": "ok"}


@router.get("/health/live")
async def liveness():
    """K8s 存活探针"""
    return {"status": "ok"}


@router.get("/health/ready")
async def readiness():
    """K8s 就绪探针"""
    return {"status": "ready"}
```

## 中间件顺序

```python
# 正确顺序（从外到内）
app.add_middleware(TrustedHostMiddleware)    # 1. 主机检查
app.add_middleware(GZipMiddleware)           # 2. 压缩
app.add_middleware(CORSMiddleware)           # 3. CORS
app.add_middleware(RateLimitMiddleware)      # 4. 限流
app.add_middleware(RequestLoggingMiddleware) # 5. 日志
```
