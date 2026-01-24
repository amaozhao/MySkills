# Pattern: Async Database Migrations

This pattern overrides the default Alembic configuration to support **SQLAlchemy 2.0 Async** and **Pydantic**.

> **Why do I need this?**
> The standard command `alembic init` generates a **synchronous** configuration. Since we use `asyncpg`, the default config will fail. You must **overwrite** the generated files with the code below.

## 1. Setup Workflow

Run these commands once to initialize the folder structure, then apply the pattern.

```bash
# 1. Initialize the generic directory structure
uv run alembic init alembic

# 2. DELETE the default env.py (It is incompatible with Async)
rm alembic/env.py

# 3. Create/Overwrite alembic/env.py with the code below
# 4. Update alembic.ini with the code below
```

## 2. The Async Environment (`alembic/env.py`)

**Action**: Copy this content into `alembic/env.py`.

It bridges the async app engine with Alembic's runner and ensures Models are detected.

```python
import asyncio
from logging.config import fileConfig
import sys
import os

from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config
from alembic import context

# ------------------------------------------------------------------------
# 1. Python Path Setup
# ------------------------------------------------------------------------
# CRITICAL: Add current working directory to path so 'app' module is found
sys.path.insert(0, os.getcwd())

# ------------------------------------------------------------------------
# 2. App Imports
# ------------------------------------------------------------------------
from app.core.config import settings
from app.models.base import Base

# !!! CRITICAL FOR AUTOGENERATE !!!
# You MUST import all your models here, otherwise Alembic won't see them.
# Example: from app.models import user, item
# (Add your imports here)

# ------------------------------------------------------------------------
# 3. Config Setup
# ------------------------------------------------------------------------
config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# Overwrite the sqlalchemy.url with Pydantic's dynamic URL
config.set_main_option("sqlalchemy.url", str(settings.DATABASE_URL))

target_metadata = Base.metadata

# ------------------------------------------------------------------------
# 4. Migration Runners (Boilerplate)
# ------------------------------------------------------------------------

def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode."""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection: Connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_migrations_online() -> None:
    """Run migrations in 'online' mode using Async Engine."""
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)

    await connectable.dispose()


if context.is_offline_mode():
    run_migrations_offline()
else:
    asyncio.run(run_migrations_online())
```

## 3. Configuration (`alembic.ini`)

**Action**: Replace the content of `alembic.ini` with this optimized version.
It adds the `post_write_hooks` to auto-format generated scripts.

```ini
[alembic]
script_location = alembic
file_template = %%(year)d-%%(month).2d-%%(day).2d_%%(rev)s_%%(slug)s
timezone = UTC
prepend_sys_path = .

# Note: The actual URL is set in env.py from Pydantic settings.
# This placeholder is just to prevent errors.
sqlalchemy.url = driver://user:pass@localhost/dbname

# --- AUTO-FORMATTING ---
# This tells Alembic to run 'ruff format' on every new migration file
[post_write_hooks]
hooks = ruff_format

[post_write_hooks.ruff_format]
type = console_scripts
entrypoint = uv run ruff format
options = REVISION_SCRIPT_FILENAME
```

## 4. Daily Usage (Auto-Generation)

Once configured, you don't edit the above files. You just run these commands:

### A. Auto-Generate Migration
After you modify your python models (e.g., added a column to `app/models/user.py`):

```bash
# This scans your code and creates a new script in alembic/versions/
uv run alembic revision --autogenerate -m "add_phone_number_to_users"
```

### B. Apply Migration
To update your local database:

```bash
uv run alembic upgrade head
```

### C. Revert Migration
To undo the last change:

```bash
uv run alembic downgrade -1
```