---
name: fastapi-architecture
description: FastAPI 分层架构 - Router/Service/Repository 分离、Repository 模式
paths: **/*.py
---

# FastAPI Layered Architecture

## 概述

严格分层：Router 解析请求，Service 处理业务逻辑，Repository 构建 SQL，仅在 Service 层提交事务。

## 架构图

```
Request → Router → Service → Repository → DB
DB → Service [commit only here]
```

## 目录结构

```
app/
├── api/endpoints/    # Router - HTTP 层
├── services/         # Service - 业务逻辑
├── repositories/     # Repository - 数据访问
├── models/           # Model - 数据库模型
└── schemas/          # Schema - Pydantic 模型
```

## 层级职责

| 层级 | 必须做 | 禁止做 |
|------|--------|--------|
| **Router** | 解析请求、验证、调用 Service | SQL、业务逻辑、提交 |
| **Service** | 业务逻辑、调用 Repo、commit | 请求解析、SQL 构建 |
| **Repository** | 构建 SQL、返回模型实例 | 提交、业务逻辑 |

## 代码示例

### Router

```python
# app/api/endpoints/users.py
from fastapi import APIRouter, Depends
from app.schemas.user import UserCreate, UserResponse
from app.services.user import create_user

router = APIRouter()


@router.post("/users", response_model=UserResponse)
async def create_user_endpoint(data: UserCreate) -> UserResponse:
    return await create_user(data)
```

### Service

```python
# app/services/user.py
from app.repositories.user import UserRepository
from app.models.user import User
from app.schemas.user import UserCreate

async def create_user(data: UserCreate) -> User:
    repo = UserRepository(User)
    user = repo.create(data)
    await repo.db.commit()  # Service 处理提交
    return user
```

### Repository

```python
# app/repositories/user.py
from app.models.base import BaseRepository

class UserRepository(BaseRepository[User]):
    async def find_by_email(self, email: str) -> User | None:
        return await self.db.execute(
            select(User).where(User.email == email)
        ).scalar_one_or_none()
```

## Repository 模式

### Base Repository

```python
# app/repositories/base.py
from typing import Generic, TypeVar, Type, Any, Sequence, Optional
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel
from app.models.base import Base

T = TypeVar("T", bound=Base)


class BaseRepository(Generic[T]):
    def __init__(self, model: Type[T], db: AsyncSession):
        self.model = model
        self.db = db

    async def get(self, id: Any) -> Optional[T]:
        return await self.db.get(self.model, id)

    async def get_by(self, **kwargs) -> Optional[T]:
        stmt = select(self.model).filter_by(**kwargs)
        result = await self.db.execute(stmt)
        return result.scalars().first()

    async def list(self, skip: int = 0, limit: int = 100, **filters) -> Sequence[T]:
        stmt = select(self.model).filter_by(**filters).offset(skip).limit(limit)
        result = await self.db.execute(stmt)
        return result.scalars().all()

    def create(self, data: T) -> T:
        self.db.add(data)
        return data

    async def update(self, db_obj: T, obj_in: BaseModel | dict) -> T:
        if isinstance(obj_in, BaseModel):
            update_data = obj_in.model_dump(exclude_unset=True)
        else:
            update_data = obj_in
        for field, value in update_data.items():
            if hasattr(db_obj, field):
                setattr(db_obj, field, value)
        self.db.add(db_obj)
        return db_obj  # 不 commit，Service 处理
```

## 常见违规

| 违规 | 症状 | 修复 |
|------|------|------|
| SQL 在 Router | endpoint 中有 select() | 移到 Repository |
| Commit 在 Repo | repo 中有 await db.commit() | 移到 Service |
| 解析在 Service | service 中有 User.model_validate() | 移到 Router |
| 循环导入 | import 错误 | 遵循 Router→Service→Repo→Model |
