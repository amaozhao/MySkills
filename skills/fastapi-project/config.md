---
name: fastapi-config
description: FastAPI 配置管理最佳实践 - 环境变量、多环境、secrets、pydantic-settings
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
        case_sensitive=True,  # 变量名大小写敏感
    )


settings = Settings()
```

### 环境变量文件

```bash
# .env - 开发环境
DEBUG=true
DATABASE_URL=sqlite+aiosqlite:///./dev.db
SECRET_KEY=dev-secret-key-change-in-production
ALGORITHM=HS256
REDIS_URL=redis://localhost:6379/0
CORS_ORIGINS=["http://localhost:3000","http://localhost:8080"]
```

```bash
# .env.prod - 生产环境
DEBUG=false
DATABASE_URL=postgresql+asyncpg://user:pass@host:5432/db
SECRET_KEY=<your-secure-random-secret-key>
ALGORITHM=HS256
REDIS_URL=redis://redis-host:6379/0
CORS_ORIGINS=["https://yourdomain.com"]
```

---

## 多环境配置

### 方式一：环境变量切换

```python
# app/core/config.py
import os

class Settings(BaseSettings):
    DEBUG: bool = False

    # 根据环境加载不同配置
    @property
    def ENVIRONMENT(self) -> str:
        return os.getenv("ENVIRONMENT", "development")

    @property
    def is_production(self) -> bool:
        return self.ENVIRONMENT == "production"


# 使用：ENVIRONMENT=production uvicorn app.main:app
```

### 方式二：多文件配置

```python
# app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
import os


def get_settings():
    """根据环境返回配置类"""
    env = os.getenv("ENVIRONMENT", "development")

    config_dict = {
        "env_file": f".env.{env}" if env != "development" else ".env",
    }

    class Settings(BaseSettings):
        model_config = SettingsConfigDict(**config_dict)

        # 字段定义...
        DEBUG: bool = False
        DATABASE_URL: str = ""

    return Settings()


settings = get_settings()
```

### 方式三：配置类继承

```python
# app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field


class BaseConfig(BaseSettings):
    """基础配置"""
    APP_NAME: str = "FastAPI"
    DEBUG: bool = False


class DevSettings(BaseConfig):
    """开发配置"""
    DEBUG: bool = True
    DATABASE_URL: str = "sqlite+aiosqlite:///./dev.db"


class ProdSettings(BaseConfig):
    """生产配置"""
    DEBUG: bool = False
    DATABASE_URL: str = Field(default="")


def get_settings():
    import os
    env = os.getenv("ENVIRONMENT", "development")

    settings_map = {
        "development": DevSettings,
        "production": ProdSettings,
    }

    return settings_map.get(env, DevSettings)()


settings = get_settings()
```

---

## 敏感信息管理

### Secrets 管理

```python
# app/core/config.py
from pydantic import Field
from pydantic_settings import BaseSettings
from typing import Optional


class Settings(BaseSettings):
    # 敏感信息不从 .env 读取，使用其他方式
    DATABASE_URL: str

    # API Keys - 从环境变量或 secret file 读取
    OPENAI_API_KEY: Optional[str] = None

    @property
    def is_production(self) -> bool:
        import os
        return os.getenv("ENVIRONMENT") == "production"

    def get_database_url(self) -> str:
        """构建数据库 URL"""
        if self.is_production:
            # 生产环境从 AWS Secrets / Vault 等获取
            return self._get_from_secrets("database_url")
        return self.DATABASE_URL

    def _get_from_secrets(self, key: str) -> str:
        """从 secrets manager 获取"""
        # 实现你的 secrets 获取逻辑
        import os
        return os.getenv(key.upper())


# 重要：.gitignore 添加 .env
# .env 只存本地开发配置
# 生产环境使用环境变量或 secrets manager
```

### 敏感字段隐藏

```python
# app/core/config.py
from pydantic import Field
from pydantic_settings import BaseSettings
import json


class Settings(BaseSettings):
    SECRET_KEY: str = "dev-secret"

    # 使用 field_serializer 隐藏敏感信息
    model_config = {
        "extra": "ignore",
    }

    def model_dump(self, **kwargs):
        """打印配置时隐藏敏感字段"""
        data = super().model_dump(**kwargs)
        sensitive_keys = ["SECRET_KEY", "DATABASE_PASSWORD"]
        for key in sensitive_keys:
            if key in data:
                data[key] = "***HIDDEN***"
        return data
```

---

## 配置验证

### 字段验证

```python
# app/core/config.py
from pydantic import Field, field_validator
from pydantic_settings import BaseSettings
import os


class Settings(BaseSettings):
    DEBUG: bool = False
    DATABASE_URL: str = ""
    REDIS_URL: str = "redis://localhost:6379/0"
    CORS_ORIGINS: str = "http://localhost:3000"

    @field_validator("DATABASE_URL")
    @classmethod
    def validate_database_url(cls, v: str) -> str:
        if not v:
            raise ValueError("DATABASE_URL is required")
        # 验证 URL 格式
        if not v.startswith(("sqlite://", "postgresql://", "mysql://")):
            raise ValueError("Invalid DATABASE_URL scheme")
        return v

    @field_validator("CORS_ORIGINS", mode="before")
    @classmethod
    def parse_cors_origins(cls, v):
        """支持 str 和 list 两种格式"""
        if isinstance(v, str):
            import json
            try:
                return json.loads(v)
            except json.JSONDecodeError:
                return [origin.strip() for origin in v.split(",")]
        return v

    @property
    def parsed_cors_origins(self) -> list[str]:
        """解析 CORS origins"""
        if isinstance(self.CORS_ORIGINS, str):
            import json
            try:
                return json.loads(self.CORS_ORIGINS)
            except:
                return [o.strip() for o in self.CORS_ORIGINS.split(",")]
        return self.CORS_ORIGINS or []
```

### 启动时验证

```python
# app/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager
from app.core.config import settings


@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用启动时验证配置"""
    # 验证必填配置
    if not settings.DATABASE_URL:
        raise RuntimeError("DATABASE_URL is required")

    if settings.is_production:
        # 生产环境额外检查
        if len(settings.SECRET_KEY) < 32:
            raise RuntimeError("SECRET_KEY must be at least 32 characters")

    yield
    # 清理


app = FastAPI(lifespan=lifespan)
```

---

## 动态配置

### 功能开关

```python
# app/core/config.py
from pydantic import Field
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    # 功能开关
    ENABLE_CACHE: bool = True
    ENABLE_WEBHOOK: bool = False
    ENABLE_ADMIN_PANEL: bool = False

    # 限制设置
    MAX_UPLOAD_SIZE: int = 10 * 1024 * 1024  # 10MB
    RATE_LIMIT_PER_MINUTE: int = 60


settings = Settings()


# 使用
@app.get("/items")
async def list_items():
    if not settings.ENABLE_CACHE:
        # 跳过缓存逻辑
        pass
```

### 条件配置

```python
# app/core/config.py
from pydantic import Field
from pydantic_settings import BaseSettings
from typing import Optional


class Settings(BaseSettings):
    # 日志配置
    LOG_LEVEL: str = "INFO"

    # 根据环境调整
    @property
    def database_pool_size(self) -> int:
        """根据环境返回不同的连接池大小"""
        if self.DEBUG:
            return 5
        return 20

    @property
    def log_config(self) -> dict:
        """日志配置"""
        return {
            "version": 1,
            "disable_existing_loggers": False,
            "level": self.LOG_LEVEL,
        }


settings = Settings()
```

---

## 在 FastAPI 中使用

### 依赖注入

```python
# app/api/info.py
from fastapi import APIRouter, Depends
from typing import Annotated
from app.core.config import Settings, get_settings

router = APIRouter()


@app.get("/config")
async def get_config(
    settings: Annotated[Settings, Depends(get_settings)]
):
    """返回非敏感配置信息"""
    return {
        "app_name": settings.APP_NAME,
        "debug": settings.DEBUG,
        "version": settings.APP_VERSION,
    }


@app.get("/health")
async def health_check(
    settings: Annotated[Settings, Depends(get_settings)]
):
    """健康检查"""
    return {
        "status": "ok",
        "debug": settings.DEBUG,
    }
```

---

## 最佳实践

### 1. 集中管理配置

```python
# app/core/config.py
# 所有配置集中在这里

from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # 应用
    APP_NAME: str
    DEBUG: bool = False

    # 数据库
    DATABASE_URL: str

    # 安全
    SECRET_KEY: str
    ALGORITHM: str = "HS256"

    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"

    # CORS
    CORS_ORIGINS: list[str] = []

    # 功能开关
    ENABLE_CACHE: bool = True


settings = Settings()
```

### 2. 类型安全

```python
# ✅ 推荐：完整类型注解
from typing import Optional
from pydantic import Field

class Settings(BaseSettings):
    DEBUG: bool = False
    DATABASE_URL: str = ""
    SECRET_KEY: str = ""
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    CORS_ORIGINS: list[str] = Field(default_factory=list)

# ❌ 避免：使用 Any 或不带类型
class BadSettings(BaseSettings):
    DEBUG = False  # 没有类型！
```

### 3. 环境隔离

```bash
# 开发环境
.env
.env.local

# 生产环境
# 不提交 .env.prod 到版本控制
# 使用环境变量或 secrets manager
```

---

## 配置检查清单

- [ ] 所有敏感信息使用环境变量或 secrets manager
- [ ] 生产环境 SECRET_KEY 至少 32 位
- [ ] CORS 配置明确列出允许的域名
- [ ] 数据库 URL 使用加密连接（生产环境）
- [ ] 启动时验证必填配置
- [ ] 配置类包含完整类型注解

---

## 相关文件

- [DI.md](./DI.md) - 依赖注入
- [database.md](./database.md) - 数据库配置
- [middleware.md](./middleware.md) - 中间件配置
- [cache.md](./cache.md) - Redis 缓存配置
