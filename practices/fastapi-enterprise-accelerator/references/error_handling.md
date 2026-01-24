# Pattern: Global Error Handling

This pattern establishes a centralized, consistent JSON error response format. It distinguishes between Client Errors (4xx) and Server Errors (5xx), ensures security by masking raw exceptions, and standardizes validation feedback.

## The Implementation (`app/core/exceptions.py`)

```python
from typing import Any, Dict, Optional
from fastapi import Request, status, FastAPI
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from sqlalchemy.exc import SQLAlchemyError, IntegrityError
import logging

# 1. Setup Logger
logger = logging.getLogger("app.core.exceptions")

# -------------------------------------------------------------------------
# 1. Custom Business Exception
# -------------------------------------------------------------------------
class BusinessException(Exception):
    """
    Base class for all logic errors.
    Usage:
        raise BusinessException(
            code="USER_NOT_FOUND",
            message="User with id 123 does not exist",
            status_code=404
        )
    """
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

# -------------------------------------------------------------------------
# 2. Exception Handlers
# -------------------------------------------------------------------------

async def business_exception_handler(request: Request, exc: BusinessException):
    """
    Catches manual BusinessExceptions.
    Returns: 4xx (Client Errors)
    """
    # Log business errors as INFO, not ERROR
    logger.info(f"Business Error: {exc.code} - {exc.message}")
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
                "details": exc.details,
                "request_id": request.state.request_id if hasattr(request.state, "request_id") else None,
                "path": request.url.path
            }
        }
    )

async def validation_exception_handler(request: Request, exc: RequestValidationError):
    """
    Overrides FastAPI's default 422 error.
    Simplifies the complex Pydantic error list into a cleaner format.
    """
    errors = []
    for error in exc.errors():
        # Clean up field name: "body.email" instead of ["body", "email"]
        loc_str = ".".join(str(x) for x in error["loc"]) if error["loc"] else "unknown"
        # Simplify message: "Field required" instead of verbose Pydantic msg
        msg = error["msg"]
        errors.append({"field": loc_str, "message": msg})

    logger.info(f"Validation Error on {request.url.path}: {errors}")
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={
            "error": {
                "code": "VALIDATION_ERROR",
                "message": "Invalid request parameters",
                "details": errors,
                "request_id": request.state.request_id if hasattr(request.state, "request_id") else None,
                "path": request.url.path
            }
        }
    )

async def integrity_exception_handler(request: Request, exc: IntegrityError):
    """
    Catches DB Integrity Errors (Unique Constraint, Foreign Key).
    Returns: 409 Conflict instead of 500.
    """
    # We log the raw error for debugging, but hide it from the user
    logger.warning(f"Integrity Error: {str(exc)}")
    
    return JSONResponse(
        status_code=status.HTTP_409_CONFLICT,
        content={
            "error": {
                "code": "CONFLICT_ERROR",
                "message": "Data conflict. A record with this unique value likely already exists.",
                "request_id": request.state.request_id if hasattr(request.state, "request_id") else None,
                "path": request.url.path
            }
        }
    )

async def sqlalchemy_exception_handler(request: Request, exc: SQLAlchemyError):
    """
    Catches other DB errors (Connection issues, Syntax errors).
    Returns: 500 (Internal Server Error)
    """
    logger.error(f"Database Error: {str(exc)}", exc_info=True)
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "error": {
                "code": "DATABASE_ERROR",
                "message": "A database system error occurred.",
                "request_id": request.state.request_id if hasattr(request.state, "request_id") else None,
                "path": request.url.path
            }
        }
    )

async def global_exception_handler(request: Request, exc: Exception):
    """
    The Safety Net. Catches everything else (IndexError, KeyError, etc).
    Returns: 500 (Internal Server Error)
    """
    logger.error(f"Unhandled Exception: {str(exc)}", exc_info=True)
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "error": {
                "code": "INTERNAL_SERVER_ERROR",
                "message": "An unexpected error occurred. Please try again later.",
                "request_id": request.state.request_id if hasattr(request.state, "request_id") else None,
                "path": request.url.path
            }
        }
    )

# -------------------------------------------------------------------------
# 3. Registration Helper
# -------------------------------------------------------------------------
def setup_exception_handlers(app: FastAPI):
    """
    Call this in main.py to activate handlers.
    Order matters: specific errors first, general errors last.
    """
    app.add_exception_handler(BusinessException, business_exception_handler)
    app.add_exception_handler(RequestValidationError, validation_exception_handler)
    app.add_exception_handler(IntegrityError, integrity_exception_handler) # Catch 409 before 500
    app.add_exception_handler(SQLAlchemyError, sqlalchemy_exception_handler)
    app.add_exception_handler(Exception, global_exception_handler)
```

## Example Response Format

### 1. Business Logic Error (400)
```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User with id 99 does not exist",
    "details": {},
    "request_id": "req_12345abc",
    "path": "/api/users/99"
  }
}
```

### 2. Validation Error (422)
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": [
      {
        "field": "body.email",
        "message": "value is not a valid email address"
      }
    ],
    "request_id": "req_12345abc",
    "path": "/api/users/"
  }
}
```

### 3. Unique Constraint Violation (409)
Instead of a generic 500 error, the user gets a clear signal that the data conflicts (e.g., email already taken).
```json
{
  "error": {
    "code": "CONFLICT_ERROR",
    "message": "Data conflict. A record with this unique value likely already exists.",
    "request_id": "req_12345abc",
    "path": "/api/users/"
  }
}
```