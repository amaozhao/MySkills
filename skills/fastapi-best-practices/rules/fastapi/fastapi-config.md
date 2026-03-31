---
name: fastapi-config
description: FastAPI 配置管理最佳实践 - 环境变量、多环境、secrets、pydantic-settings
paths: **/*.py
---

# FastAPI 配置管理

## 概述

使用 pydantic-settings 实现类型安全的配置管理，支持多环境切换和敏感信息保护。

## 基础配置

### 项目配置

```python
# app/core/config.py
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict
from typing import Optional


class Settings(BaseSettings):
    """项目配置"""

    # 应用基础
    APP_NAME: str = "FastAPI App"
    APP_VERSION: str = "1.0.0"
    DEBUG: bool = False

    # 数据库
    DATABASE_URL: str = Field(default="sqlite+aiosqlite:///./db.sqlite")

    # 安全
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"

    # CORS
    CORS_ORIGINS: list[str] = ["http://localhost:3000"]

    # 环境配置
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=True,
    )


settings = Settings()
```

### 环境变量文件

```bash
# .env - 开发环境
DEBUG=true
DATABASE_URL=sqlite+aiosqlite:///./dev.db
SECRET_KEY=dev-secret-key-change-in-production
```

```bash
# .env.prod - 生产环境
DEBUG=false
DATABASE_URL=postgresql+asyncpg://user:pass@host:5432/db
SECRET_KEY=<your-secure-random-secret-key>
```

## 多环境配置

### 方式一：环境变量切换

```python
class Settings(BaseSettings):
    DEBUG: bool = False

    @property
    def ENVIRONMENT(self) -> str:
        return os.getenv("ENVIRONMENT", "development")

    @property
    def is_production(self) -> bool:
        return self.ENVIRONMENT == "production"
```

## 敏感信息管理

```python
class Settings(BaseSettings):
    # API Keys - 从环境变量或 secret file 读取
    OPENAI_API_KEY: Optional[str] = None

    def get_database_url(self) -> str:
        """构建数据库 URL"""
        if self.is_production:
            return self._get_from_secrets("database_url")
        return self.DATABASE_URL

# .gitignore 添加 .env
# 生产环境使用环境变量或 secrets manager
```

## 启动时验证

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用启动时验证配置"""
    if not settings.DATABASE_URL:
        raise RuntimeError("DATABASE_URL is required")

    if settings.is_production:
        if len(settings.SECRET_KEY) < 32:
            raise RuntimeError("SECRET_KEY must be at least 32 characters")
    yield
```
