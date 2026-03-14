---
name: fastapi-errors
description: FastAPI 统一异常处理 - BusinessException、异常处理器、错误响应格式
---

# FastAPI Error Handling

## 概述

集中式异常处理器返回一致的 JSON 响应，区分 4xx 客户端错误和 5xx 服务器错误，同时掩藏敏感信息。

## BusinessException

```python
# app/core/exceptions.py
from typing import Any, Dict, Optional
from fastapi import Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from sqlalchemy.exc import SQLAlchemyError, IntegrityError
import logging

logger = logging.getLogger(__name__)


class BusinessException(Exception):
    """业务逻辑异常"""
    def __init__(
        self,
        message: str,
        code: str = "BUSINESS_ERROR",
        status_code: int = status.HTTP_400_BAD_REQUEST,
        details: Optional[Dict[str, Any]] = None
    ):
        self.message = message
        self.code = code
        self.status_code = status_code
        self.details = details or {}
        super().__init__(self.message)
```

## 异常处理器

```python
# app/core/handlers.py
from app.core.exceptions import BusinessException


async def business_exception_handler(request: Request, exc: BusinessException):
    logger.info(f"Business Error: {exc.code} - {exc.message}")
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
                "details": exc.details,
                "path": request.url.path
            }
        }
    )


async def validation_exception_handler(request: Request, exc: RequestValidationError):
    errors = []
    for error in exc.errors():
        loc = ".".join(str(x) for x in error["loc"]) if error["loc"] else "unknown"
        errors.append({"field": loc, "message": error["msg"]})

    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={
            "error": {
                "code": "VALIDATION_ERROR",
                "message": "Invalid request parameters",
                "details": errors,
                "path": request.url.path
            }
        }
    )


async def integrity_exception_handler(request: Request, exc: IntegrityError):
    logger.warning(f"Integrity Error: {str(exc)}")
    return JSONResponse(
        status_code=status.HTTP_409_CONFLICT,
        content={
            "error": {
                "code": "CONFLICT_ERROR",
                "message": "Data conflict",
                "path": request.url.path
            }
        }
    )


async def sqlalchemy_exception_handler(request: Request, exc: SQLAlchemyError):
    logger.error(f"Database Error: {str(exc)}", exc_info=True)
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "error": {
                "code": "DATABASE_ERROR",
                "message": "A database error occurred",
                "path": request.url.path
            }
        }
    )


async def global_exception_handler(request: Request, exc: Exception):
    logger.error(f"Unhandled Exception: {str(exc)}", exc_info=True)
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "error": {
                "code": "INTERNAL_SERVER_ERROR",
                "message": "An unexpected error occurred",
                "path": request.url.path
            }
        }
    )
```

## 注册处理器

```python
# app/main.py
from fastapi import FastAPI
from fastapi.exceptions import RequestValidationError
from sqlalchemy.exc import IntegrityError, SQLAlchemyError
from app.core.exceptions import BusinessException
from app.core.handlers import (
    business_exception_handler,
    validation_exception_handler,
    integrity_exception_handler,
    sqlalchemy_exception_handler,
    global_exception_handler,
)

app = FastAPI()

app.add_exception_handler(BusinessException, business_exception_handler)
app.add_exception_handler(RequestValidationError, validation_exception_handler)
app.add_exception_handler(IntegrityError, integrity_exception_handler)
app.add_exception_handler(SQLAlchemyError, sqlalchemy_exception_handler)
app.add_exception_handler(Exception, global_exception_handler)
```

## 使用

```python
# 在 Service 中抛出异常
if not user:
    raise BusinessException(
        code="USER_NOT_FOUND",
        message="User not found",
        status_code=404
    )
```

## 响应格式

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User with id 123 does not exist",
    "details": {},
    "path": "/api/users/123"
  }
}
```

## 错误码

| 码 | 状态 | 含义 |
|----|------|------|
| VALIDATION_ERROR | 422 | Pydantic 验证失败 |
| CONFLICT_ERROR | 409 | 唯一约束冲突 |
| DATABASE_ERROR | 500 | 数据库连接错误 |
| INTERNAL_SERVER_ERROR | 500 | 未处理异常 |

---

## 相关文件

- [DI.md](./DI.md) - 依赖注入
- [database.md](./database.md) - 数据库
- [testing.md](./testing.md) - 测试
