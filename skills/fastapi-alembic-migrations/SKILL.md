---
name: fastapi-alembic-migrations
description: Use when setting up database migrations for SQLAlchemy 2.0 async projects, or when Alembic can't connect to async database
---

# FastAPI Alembic Migrations

## Overview
**SQLAlchemy 2.0 async-compatible Alembic configuration with auto-formatting and model detection.**

## When to Use
- Initializing migrations for async SQLAlchemy project
- `alembic revision` fails with async engine
- Need auto-formatting on migration files
- Setting up new project database schema

## Setup Commands

```bash
uv run alembic init alembic
rm alembic/env.py  # Default is synchronous!
# Then create new env.py below
```

## Async Environment

```python
# alembic/env.py
import asyncio
import sys
import os
from sqlalchemy.ext.asyncio import async_engine_from_config
from alembic import context

sys.path.insert(0, os.getcwd())

from app.core.config import settings
from app.models.base import Base

config = context.config
if config.config_file_name:
    from logging.config import fileConfig
    fileConfig(config.config_file_name)

config.set_main_option("sqlalchemy.url", str(settings.DATABASE_URL))
target_metadata = Base.metadata

# Import ALL models here for autogenerate!
# from app.models import user, item

def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()

async def run_migrations_online():
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()

if context.is_offline_mode():
    # Offline logic here
    pass
else:
    asyncio.run(run_migrations_online())
```

## Alembic.ini

```ini
[alembic]
script_location = alembic
file_template = %%(year)d-%%(month).2d-%%(day).2d_%%(rev)s_%%(slug)s
timezone = UTC

[post_write_hooks]
hooks = ruff_format

[post_write_hooks.ruff_format]
type = console_scripts
entrypoint = uv run ruff format
options = REVISION_SCRIPT_FILENAME
```

## Daily Commands

```bash
# Generate migration from model changes
uv run alembic revision --autogenerate -m "describe_change"

# Apply migrations
uv run alembic upgrade head

# Revert
uv run alembic downgrade -1
```

## Common Issues

| Problem | Solution |
|---------|----------|
| "Module not found: app" | Add `sys.path.insert()` in env.py |
| Models not detected | Import models explicitly in env.py |
| Migration fails | Run with `--sql` first to preview |

## The Bottom Line

**Import models, async engine, auto-format = working migrations.**
