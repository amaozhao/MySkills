---
name: fastapi-migrations
description: FastAPI Alembic 异步迁移配置 - auto-generate、离线迁移
paths: **/*.py
---

# FastAPI Alembic Migrations

## 概述

SQLAlchemy 2.0 兼容的 Alembic 异步迁移配置。

## 初始化

```bash
uv run alembic init alembic
rm alembic/env.py  # 默认是同步的！
```

## Async env.py

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
config.set_main_option("sqlalchemy.url", str(settings.DATABASE_URL))
target_metadata = Base.metadata

# 导入所有模型！
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
    pass
else:
    asyncio.run(run_migrations_online())
```

## alembic.ini

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

## 常用命令

```bash
# 生成迁移
uv run alembic revision --autogenerate -m "add_user_table"

# 应用迁移
uv run alembic upgrade head

# 回滚
uv run alembic downgrade -1

# 预览 SQL
uv run alembic upgrade --sql +head
```

## 常见问题

| 问题 | 解决 |
|------|------|
| ModuleNotFoundError: app | env.py 添加 sys.path.insert |
| 模型未检测到 | 在 env.py 显式导入所有模型 |
| 迁移失败 | 先用 --sql 预览 |
