---
name: fastapi-error-handling
description: Use when building FastAPI APIs that need consistent error responses, custom business exceptions, or want to distinguish client errors from server errors
---

# FastAPI Error Handling

## Overview
**Centralized exception handlers that return consistent JSON responses, distinguishing 4xx client errors from 5xx server errors while masking sensitive details.**

## When to Use
- Building APIs with multiple error types
- Need consistent error format across endpoints
- Want custom business logic exceptions
- Need to handle validation, database, and general errors

## Core Pattern

```python
from typing import Any, Dict, Optional
from fastapi import Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from sqlalchemy.exc import SQLAlchemyError, IntegrityError
import logging

logger = logging.getLogger(__name__)

class BusinessException(Exception):
    """Base class for business logic errors."""
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

# Handlers
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

    logger.info(f"Validation Error: {errors}")
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
                "message": "Data conflict. A record with this value already exists.",
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
                "message": "A database system error occurred.",
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
                "message": "An unexpected error occurred.",
                "path": request.url.path
            }
        }
    )

def setup_exception_handlers(app):
    """Register handlers in main.py."""
    app.add_exception_handler(BusinessException, business_exception_handler)
    app.add_exception_handler(RequestValidationError, validation_exception_handler)
    app.add_exception_handler(IntegrityError, integrity_exception_handler)
    app.add_exception_handler(SQLAlchemyError, sqlalchemy_exception_handler)
    app.add_exception_handler(Exception, global_exception_handler)
```

## Usage

```python
# In service layer
if not user:
    raise BusinessException(
        code="USER_NOT_FOUND",
        message="User with id 123 does not exist",
        status_code=404
    )

# In router
@router.get("/users/{user_id}")
async def get_user(user_id: int):
    user = await get_user_service(user_id)
    return user
```

## Response Format

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

## Error Codes

| Code | Status | Meaning |
|------|--------|---------|
| `VALIDATION_ERROR` | 422 | Pydantic validation failed |
| `CONFLICT_ERROR` | 409 | Unique constraint violation |
| `DATABASE_ERROR` | 500 | SQLAlchemy connection error |
| `INTERNAL_SERVER_ERROR` | 500 | Unhandled exception |

## The Bottom Line

**Business exceptions for your code; generic handlers for everything else.**

- Use `BusinessException` for domain errors
- Register handlers specific to general
- Log details server-side; mask in responses
