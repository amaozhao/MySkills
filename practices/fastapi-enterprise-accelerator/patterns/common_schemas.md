# Pattern: Common Schemas

This pattern defines shared Pydantic models used across the entire application. It establishes a **Standardized Response Format** for pagination, simple messages, and errors.

## The Implementation (`app/schemas/common.py`)

```python
from typing import Generic, TypeVar, List, Any, Optional
from math import ceil
from pydantic import BaseModel, ConfigDict, computed_field

# Generic Type for Data
T = TypeVar("T")

# -------------------------------------------------------------------------
# 1. Pagination Schemas
# -------------------------------------------------------------------------

class MetaData(BaseModel):
    """
    Metadata for paginated responses.
    Automatically calculates helper fields like 'total_pages'.
    """
    total: int
    page: int
    size: int
    
    @computed_field
    def total_pages(self) -> int:
        if self.size == 0:
            return 0
        return ceil(self.total / self.size)

    @computed_field
    def has_next(self) -> bool:
        return self.page < self.total_pages

    @computed_field
    def has_prev(self) -> bool:
        return self.page > 1

class PagedResponse(BaseModel, Generic[T]):
    """
    A generic wrapper for lists.
    Ensures all list endpoints return data in the same shape.
    
    Usage in Router:
        response_model=PagedResponse[UserResponse]
    """
    data: List[T]
    meta: MetaData

    # Config to allow creating from ORM objects if needed
    model_config = ConfigDict(from_attributes=True)

    @classmethod
    def create(
        cls, 
        items: List[Any], 
        total: int, 
        page: int, 
        size: int
    ) -> "PagedResponse[T]":
        """Helper factory method to create a response easily."""
        return cls(
            data=items,
            meta=MetaData(total=total, page=page, size=size)
        )

# -------------------------------------------------------------------------
# 2. Standard Response Schemas
# -------------------------------------------------------------------------

class IDModel(BaseModel):
    """
    Simple response returning just an ID.
    Useful for 'Create' endpoints where you don't need to return the full object.
    """
    id: int

class Message(BaseModel):
    """
    Simple response for actions.
    Example: {"message": "Item deleted successfully"}
    """
    message: str

# -------------------------------------------------------------------------
# 3. Error Schemas (For OpenAPI / Swagger UI)
# -------------------------------------------------------------------------
# Use these in your Router decorators to explicitly document error responses.
# Example: @router.get(..., responses={400: {"model": ErrorResponse}})

class ErrorDetail(BaseModel):
    code: str
    message: str
    path: Optional[str] = None
    request_id: Optional[str] = None
    details: Optional[Any] = None  # For list of validation errors

class ErrorResponse(BaseModel):
    """
    Strictly matches the JSON structure returned by 
    'app/core/exceptions.py'
    """
    error: ErrorDetail
```