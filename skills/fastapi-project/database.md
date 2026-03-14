---
name: fastapi-database
description: FastAPI 异步数据库配置 - SQLAlchemy 2.0、连接池、Pydantic v2 兼容
---

# FastAPI Async Database

## 概述

生产级异步 SQLAlchemy 2.0 配置，包含连接池优化和 Pydantic v2 兼容性。

## 核心配置

### 异步 Engine

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

### 配置自动转换 Driver

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

## 连接池配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `pool_size=20` | 5 | 保持的连接数 |
| `max_overflow=10` | 10 | 额外连接数 |
| `pool_recycle=3600` | - | 连接回收时间（1小时） |
| `pool_pre_ping=True` | False | 使用前检查连接 |
| `pool_use_lifo=True` | False | LIFO 复用热连接 |

## Pydantic v2 兼容性

关键设置：`expire_on_commit=False`

```python
AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,  # 关键！
)
```

这防止 Pydantic v2 访问已分离对象时报错。

---

## 相关文件

- [config.md](./config.md) - 配置管理
- [migrations.md](./migrations.md) - 数据库迁移
- [DI.md](./DI.md) - 依赖注入
- [architecture.md](./architecture.md) - 分层架构
