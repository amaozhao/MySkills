---
name: fastapi-scaffold
description: FastAPI 项目脚手架生成器 - 初始化新的 FastAPI 项目，包含完整的目录结构、配置文件、最佳实践设置。创建新的 FastAPI 项目时使用。
disable-model-invocation: true
---

# FastAPI Project Scaffold

## 概述

初始化新的 FastAPI 项目，包含完整的目录结构、配置文件和最佳实践设置。

## 输入

项目名称和基本需求：
- 项目名称
- 是否需要数据库（PostgreSQL/SQLite）
- 是否需要认证
- 是否需要 WebSocket

## 生成流程

1. **创建目录结构**
2. **生成配置文件**（pyproject.toml, .env.example）
3. **生成核心模块**（config, database, security, exceptions）
4. **生成 main.py**（包含中间件和异常处理）
5. **生成基础模型和 Schema**

## 目录结构

```
{project_name}/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── deps.py
│   │   └── endpoints/
│   │       └── health.py
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py
│   │   ├── database.py
│   │   ├── security.py
│   │   └── exceptions.py
│   ├── models/
│   │   ├── __init__.py
│   │   └── base.py
│   ├── schemas/
│   │   └── base.py
│   ├── services/
│   │   └── __init__.py
│   ├── repositories/
│   │   └── __init__.py
│   └── cache/
│       └── __init__.py
├── tests/
│   ├── conftest.py
│   ├── unit/
│   └── integration/
├── alembic/
│   ├── env.py
│   └── versions/
├── pyproject.toml
├── .env.example
└── .gitignore
```

## 核心文件

### pyproject.toml

```toml
[project]
name = "{project_name}"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.109.0",
    "uvicorn[standard]>=0.27.0",
    "sqlalchemy>=2.0.0",
    "pydantic>=2.0",
    "pydantic-settings>=2.0",
    "python-jose[cryptography]",
    "passlib[bcrypt]",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
target-version = "py312"
line-length = 100
```

### app/core/config.py

```python
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    APP_NAME: str = "{project_name}"
    DEBUG: bool = False
    DATABASE_URL: str = Field(default="sqlite+aiosqlite:///./db.sqlite")
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

    model_config = SettingsConfigDict(
        env_file=".env",
        case_sensitive=True,
    )


settings = Settings()
```

### app/main.py

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.api.deps import get_db
from app.core.exceptions import BusinessException
from app.core.handlers import (
    business_exception_handler,
    validation_exception_handler,
    sqlalchemy_exception_handler,
    global_exception_handler,
)


def create_app() -> FastAPI:
    app = FastAPI(title=settings.APP_NAME)

    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    app.add_exception_handler(BusinessException, business_exception_handler)
    app.add_exception_handler(RequestValidationError, validation_exception_handler)
    app.add_exception_handler(SQLAlchemyError, sqlalchemy_exception_handler)
    app.add_exception_handler(Exception, global_exception_handler)

    return app


app = create_app()
```

## 使用方式

```
/fastapi-scaffold

输入：项目名 myapi，需要 PostgreSQL 和 JWT 认证
```

## 最佳实践集成

- SQLAlchemy 2.0 异步配置
- Pydantic v2 + pydantic-settings
- 统一异常处理
- Repository 模式
- 分层架构
