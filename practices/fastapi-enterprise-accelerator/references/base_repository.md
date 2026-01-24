# Pattern: Generic Repository (Enhanced)

This pattern implements a Type-Safe Generic Repository (`BaseRepository[T]`) that handles the heavy lifting of CRUD operations.

**Enhancements in this version:**
*   Includes `update()` logic to map Pydantic models to DB entities.
*   Includes `create_multi()` for bulk inserts.
*   Fixes potential bugs with Primary Key assumptions in delete.

## The Implementation (`app/repositories/base.py`)

```python
from typing import Generic, TypeVar, Type, Any, Sequence, Optional, Union, Dict
from sqlalchemy import select, func, inspect
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel
from app.models.base import Base

# T: The SQLAlchemy Model
T = TypeVar("T", bound=Base)

class BaseRepository(Generic[T]):
    """
    A robust generic repository providing standard CRUD operations.
    """

    def __init__(self, model: Type[T], db: AsyncSession):
        self.model = model
        self.db = db

    async def get(self, id: Any) -> Optional[T]:
        """Fetch a single record by primary key."""
        return await self.db.get(self.model, id)

    async def get_by(self, **kwargs) -> Optional[T]:
        """Fetch a single record by exact filter (e.g. email='x')."""
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
        """Fetch a paginated list of records."""
        stmt = select(self.model)
        if filters:
            stmt = stmt.filter_by(**filters)
        if order_by is not None:
            stmt = stmt.order_by(order_by)
            
        stmt = stmt.offset(skip).limit(limit)
        result = await self.db.execute(stmt)
        return result.scalars().all()

    async def count(self, **filters) -> int:
        """Count total records matching filter."""
        stmt = select(func.count()).select_from(self.model)
        if filters:
            stmt = stmt.filter_by(**filters)
        result = await self.db.execute(stmt)
        return result.scalar() or 0

    def create(self, data: T) -> T:
        """Schedule a new record for insertion."""
        self.db.add(data)
        return data

    def create_multi(self, data_list: Sequence[T]) -> Sequence[T]:
        """Schedule multiple records for insertion (Bulk)."""
        self.db.add_all(data_list)
        return data_list

    async def update(
        self, 
        db_obj: T, 
        obj_in: Union[BaseModel, Dict[str, Any]]
    ) -> T:
        """
        Update a database object with data from a Pydantic Schema or Dict.
        """
        # Convert Pydantic to dict if needed
        if isinstance(obj_in, BaseModel):
            update_data = obj_in.model_dump(exclude_unset=True)
        else:
            update_data = obj_in

        # Set attributes dynamically
        for field, value in update_data.items():
            if hasattr(db_obj, field):
                setattr(db_obj, field, value)
        
        # Explicitly add to session (in case it was detached)
        self.db.add(db_obj)
        # Note: No commit() here. Service layer handles transaction.
        return db_obj

    async def delete(self, id: Any) -> bool:
        """
        Delete a record by ID.
        Fetches the object first to ensure ORM events (cascades) trigger.
        """
        obj = await self.get(id)
        if obj:
            await self.db.delete(obj)
            return True
        return False

    async def exists(self, **filters) -> bool:
        """Check if record exists (Optimized)."""
        stmt = select(1).select_from(self.model).filter_by(**filters).limit(1)
        result = await self.db.execute(stmt)
        return result.scalar() is not None
```

## Why the changes matter

1.  **`update` Method**:
    *   **Scenario**: User wants to change their nickname but not their email.
    *   **Old Way**: You had to verify which fields are not None manually.
    *   **New Way**: `repo.update(user_obj, user_update_schema)`. The code `exclude_unset=True` ensures that fields the user *didn't* send (None) are not mistakenly overwritten with NULL in the database.

2.  **`delete` Method**:
    *   **Old Way**: `delete(Model).where(id==id)` is a raw SQL delete. It's fast but bypasses SQLAlchemy's logic. If you had an `on_delete` Python-side cascade, it wouldn't run.
    *   **New Way**: `db.delete(obj)` loads the object into the Session. This ensures SQLAlchemy knows the object is being deleted, keeping the Session state clean.

3.  **`create_multi`**:
    *   Essential for endpoints like "Import CSV" or "Initialize Default Data".