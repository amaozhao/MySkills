---
name: fastapi-repository-pattern
description: Use when building FastAPI APIs that need type-safe CRUD operations, clean database access patterns, or want to avoid SQL scattered across services
---

# FastAPI Repository Pattern

## Overview
**Type-safe generic repository that encapsulates CRUD operations, keeping SQL out of services and enabling testability.**

## When to Use
- Need consistent CRUD across multiple models
- Want type hints for all database operations
- Building APIs with SQLAlchemy 2.0 async
- Need to isolate database logic for testing

## The Pattern

```python
from typing import Generic, TypeVar, Type, Any, Sequence, Optional
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel
from app.models.base import Base

T = TypeVar("T", bound=Base)

class BaseRepository(Generic[T]):
    """Type-safe generic repository for CRUD operations."""

    def __init__(self, model: Type[T], db: AsyncSession):
        self.model = model
        self.db = db

    async def get(self, id: Any) -> Optional[T]:
        """Fetch by primary key."""
        return await self.db.get(self.model, id)

    async def get_by(self, **kwargs) -> Optional[T]:
        """Fetch by exact filter."""
        stmt = select(self.model).filter_by(**kwargs)
        result = await self.db.execute(stmt)
        return result.scalars().first()

    async def list(
        self,
        skip: int = 0,
        limit: int = 100,
        order_by: Any = None,
        **filters
    ) -> Sequence[T]:
        """Fetch paginated list."""
        stmt = select(self.model)
        if filters:
            stmt = stmt.filter_by(**filters)
        if order_by is not None:
            stmt = stmt.order_by(order_by)
        stmt = stmt.offset(skip).limit(limit)
        result = await self.db.execute(stmt)
        return result.scalars().all()

    async def count(self, **filters) -> int:
        """Count records matching filter."""
        stmt = select(func.count()).select_from(self.model)
        if filters:
            stmt = stmt.filter_by(**filters)
        result = await self.db.execute(stmt)
        return result.scalar() or 0

    def create(self, data: T) -> T:
        """Schedule new record for insertion."""
        self.db.add(data)
        return data

    def create_multi(self, data_list: Sequence[T]) -> Sequence[T]:
        """Schedule multiple records (bulk insert)."""
        self.db.add_all(data_list)
        return data_list

    async def update(
        self,
        db_obj: T,
        obj_in: BaseModel | dict[str, Any]
    ) -> T:
        """Update DB object from Pydantic schema or dict."""
        if isinstance(obj_in, BaseModel):
            update_data = obj_in.model_dump(exclude_unset=True)
        else:
            update_data = obj_in

        for field, value in update_data.items():
            if hasattr(db_obj, field):
                setattr(db_obj, field, value)

        self.db.add(db_obj)
        return db_obj  # No commit - service handles it

    async def delete(self, id: Any) -> bool:
        """Delete by ID (triggers ORM events)."""
        obj = await self.get(id)
        if obj:
            await self.db.delete(obj)
            return True
        return False

    async def exists(self, **filters) -> bool:
        """Check if record exists (optimized)."""
        stmt = select(1).select_from(self.model).filter_by(**filters).limit(1)
        result = await self.db.execute(stmt)
        return result.scalar() is not None
```

## Usage Example

```python
# Define model
class User(Base):
    id: Mapped[int] = Column(primary_key=True)
    email: Mapped[str] = Column(unique=True)

# Define repository
class UserRepository(BaseRepository[User]):
    async def find_by_email(self, email: str) -> User | None:
        return await self.get_by(email=email)

# Use in service
async def get_user_by_email(email: str) -> User | None:
    repo = UserRepository(User, db)
    return await repo.find_by_email(email)
```

## The Bottom Line

**Repository isolates data access; Service isolates business logic.**
