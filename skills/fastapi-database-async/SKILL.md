---
name: fastapi-database-async
description: Use when setting up SQLAlchemy 2.0 async database connections, connection pooling, or production-ready database configuration in FastAPI
---

# FastAPI Async Database Setup

## Overview
**Production-grade async SQLAlchemy 2.0 setup with connection pooling, health checks, and Pydantic v2 compatibility.**

## When to Use
- Setting up new FastAPI project with async database
- Configuring connection pool for production
- Using SQLAlchemy 2.0 with asyncpg
- Need `expire_on_commit=False` for Pydantic v2

## The Pattern

```python
# app/core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from app.core.config import settings

engine = create_async_engine(
    str(settings.DATABASE_URL),
    echo=settings.DEBUG,
    pool_size=20,
    max_overflow=10,
    pool_recycle=3600,
    pool_pre_ping=True,
    pool_use_lifo=True,
    future=True,
)

AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    autoflush=False,
    expire_on_commit=False,
)

async def get_db() -> AsyncGenerator[AsyncSession]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

## Base Model

```python
# app/models/base.py
from sqlalchemy.orm import DeclarativeBase, MappedAsDataclass, Mapped, mapped_column
from sqlalchemy import func, MetaData

POSTGRES_NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

class Base(MappedAsDataclass, DeclarativeBase):
    metadata = MetaData(naming_convention=POSTGRES_NAMING_CONVENTION)

class TimestampMixin(MappedAsDataclass):
    created_at: Mapped[datetime] = mapped_column(
        insert_default=func.now(),
    )
    updated_at: Mapped[datetime] = mapped_column(
        insert_default=func.now(),
        onupdate=func.now(),
    )
```

## Config with Auto-Driver

```python
# app/core/config.py
from typing import Annotated
from pydantic_settings import BaseSettings
from pydantic import Field, BeforeValidator

def ensure_async_driver(v: str | None) -> str:
    if isinstance(v, str) and v.startswith("postgresql://"):
        return v.replace("postgresql://", "postgresql+asyncpg://", 1)
    return v or ""

AsyncPostgresDsn = Annotated[str, BeforeValidator(ensure_async_driver)]

class Settings(BaseSettings):
    DATABASE_URL: AsyncPostgresDsn
    DEBUG: bool = False

settings = Settings()
```

## Key Settings

| Setting | Value | Purpose |
|---------|-------|---------|
| `pool_pre_ping=True` | Health check | Prevents stale connections |
| `pool_use_lifo=True` | LIFO | Reuses hot connections first |
| `expire_on_commit=False` | Pydantic v2 | Prevents detached object errors |

## The Bottom Line

**Async engine + session factory + proper config = reliable database layer.**
