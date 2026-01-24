# Pattern: Robust Database Setup

This pattern establishes a production-grade asynchronous database connection using SQLAlchemy 2.0. It includes robust connection pooling config, automatic driver correction, and modern ORM base classes.

## 1. Engine & Session (`app/core/database.py`)

This file handles the raw connection to the database. It is the heart of your data layer.

```python
from typing import AsyncGenerator, Any
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from app.core.config import settings

# 1. Async Engine Configuration
# We separate the engine creation to allow it to be imported by Alembic if needed.
engine = create_async_engine(
    # The URL is guaranteed to be correct by Pydantic settings
    str(settings.DATABASE_URL),
    
    # Logging: Only enable SQL echo in non-production environments
    echo=settings.DEBUG,
    
    # Connection Pool Settings (Optimized for AsyncPG)
    pool_size=20,              # Baseline number of open connections
    max_overflow=10,           # Allow burst up to 30 connections total
    pool_recycle=3600,         # Force recycle connections every hour
    pool_pre_ping=True,        # Health check connections before handing them out
    
    # Advanced Options
    pool_use_lifo=True,        # Use LIFO to reuse hot connections first
    future=True,
)

# 2. Async Session Factory
# This factory creates the ephemeral sessions used per request.
AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    autoflush=False,          # We prefer explicit flushes
    expire_on_commit=False    # Vital for Pydantic v2 compatibility
)

# 3. FastAPI Dependency
async def get_db() -> AsyncGenerator[AsyncSession, Any]:
    """
    Dependency to provide a database session for a request.
    Guarantees session closure to prevent connection leaks.
    """
    async with AsyncSessionLocal() as session:
        try:
            yield session
            # Note: We do NOT commit here. 
            # Commit is the responsibility of the Service layer (Unit of Work).
        except Exception:
            # Although the context manager handles cleanup, explicit rollback
            # aids in debugging and ensures clean state on error.
            await session.rollback()
            raise
        finally:
            await session.close()
```

## 2. Declarative Base (`app/models/base.py`)

This file defines the shared functionality for all Database Entities. We use `MappedAsDataclass` for modern, type-safe model definitions.

```python
from datetime import datetime
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, MappedAsDataclass
from sqlalchemy import func, MetaData

# Consistent Naming Convention for Constraints
# Essential for Alembic to generate stable migration scripts across environments.
POSTGRES_NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

class Base(MappedAsDataclass, DeclarativeBase):
    """
    The root model class.
    
    Features:
    - MappedAsDataclass: Generates __init__, __repr__ automatically.
    - DeclarativeBase: SQLAlchemy 2.0 style mapping.
    - MetaData: Applies naming conventions.
    """
    metadata = MetaData(naming_convention=POSTGRES_NAMING_CONVENTION)

class TimestampMixin(MappedAsDataclass):
    """
    A reusable Mixin to add audit timestamps to any table.
    
    Usage:
        class User(Base, TimestampMixin): ...
    """
    created_at: Mapped[datetime] = mapped_column(
        insert_default=func.now(),
        sort_order=9999  # Pushes these columns to the end of the table
    )
    updated_at: Mapped[datetime] = mapped_column(
        insert_default=func.now(),
        onupdate=func.now(),
        sort_order=9999
    )
```

## 3. Configuration (`app/core/config.py`)

This handles the environment variable loading. It includes a smart validator to automatically fix the database driver scheme (converting `postgresql://` to `postgresql+asyncpg://`).

```python
from typing import Annotated
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field, BeforeValidator

# --- Validators ---

def ensure_async_driver(v: str | None) -> str:
    """
    Automatically transforms standard Postgres URLs to AsyncPG URLs.
    Example: postgresql://user:pass@db/app -> postgresql+asyncpg://user:pass@db/app
    """
    if isinstance(v, str) and v.startswith("postgresql://"):
        return v.replace("postgresql://", "postgresql+asyncpg://", 1)
    return v or ""

# Custom Pydantic type for auto-correction
AsyncPostgresDsn = Annotated[str, BeforeValidator(ensure_async_driver)]

# --- Settings ---

class Settings(BaseSettings):
    PROJECT_NAME: str = "FastAPI Enterprise"
    DEBUG: bool = False
    
    # Database Connection String
    # Format: postgresql+asyncpg://user:password@host:port/dbname
    DATABASE_URL: AsyncPostgresDsn

    model_config = SettingsConfigDict(
        env_file=".env",
        env_ignore_empty=True,
        case_sensitive=True,
        extra="ignore"  # Allow other system env vars to coexist
    )

settings = Settings()
```