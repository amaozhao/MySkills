# CRUD Template

## Schema Template

```python
# app/schemas/{resource}.py
from datetime import datetime
from pydantic import BaseModel, EmailStr


class {Resource}Base(BaseModel):
    """基础 Schema"""
    pass


class {Resource}Create({Resource}Base):
    """创建 Schema"""
    pass


class {Resource}Update({Resource}Base):
    """更新 Schema"""
    pass


class {Resource}Response({Resource}Base):
    """响应 Schema"""
    id: int
    created_at: datetime
    updated_at: datetime

    model_config = {"from_attributes": True}
```

## Repository Template

```python
# app/repositories/{resource}.py
from sqlalchemy import select
from app.models.base import BaseRepository
from app.models.{resource} import {Resource}


class {Resource}Repository(BaseRepository[{Resource}]):
    async def get_by_email(self, email: str) -> {Resource} | None:
        stmt = select({Resource}).where({Resource}.email == email)
        return await self.db.execute(stmt).scalar_one_or_none()
```

## Service Template

```python
# app/services/{resource}.py
from app.repositories.{resource} import {Resource}Repository
from app.models.{resource} import {Resource}
from app.schemas.{resource} import {Resource}Create
from app.core.exceptions import BusinessException


class {Resource}Service:
    def __init__(self, db: AsyncSession):
        self.repo = {Resource}Repository({Resource}, db)

    async def create(self, data: {Resource}Create) -> {Resource}:
        existing = await self.repo.get_by_email(data.email)
        if existing:
            raise BusinessException(
                code="{RESOURCE}_ALREADY_EXISTS",
                message="Resource already exists"
            )

        obj = {Resource}(**data.model_dump())
        self.repo.create(obj)
        await self.repo.db.commit()
        return obj

    async def get(self, id: int) -> {Resource}:
        obj = await self.repo.get(id)
        if not obj:
            raise BusinessException(code="NOT_FOUND", message="Not found", status_code=404)
        return obj
```

## Router Template

```python
# app/api/endpoints/{resource}.py
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.api.deps import get_db
from app.schemas.{resource} import {Resource}Create, {Resource}Response
from app.services.{resource} import {Resource}Service

router = APIRouter()


@router.post("/", response_model={Resource}Response)
async def create_{resource}(
    data: {Resource}Create,
    db: AsyncSession = Depends(get_db)
):
    service = {Resource}Service(db)
    return await service.create(data)


@router.get("/{id}", response_model={Resource}Response)
async def get_{resource}(
    id: int,
    db: AsyncSession = Depends(get_db)
):
    service = {Resource}Service(db)
    return await service.get(id)
```
