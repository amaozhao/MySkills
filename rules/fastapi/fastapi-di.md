---
name: fastapi-dependency-injection
description: FastAPI 依赖注入最佳实践 - 依赖组织、Depends 用法、生命周期管理
paths: **/*.py
---

# FastAPI 依赖注入

## 概述

FastAPI 的依赖注入系统是其核心特性，通过 `Depends` 实现请求级别的资源管理、依赖复用和测试友好性。

## 基础用法

### 基本模式

```python
from fastapi import Depends

# 定义依赖
async def get_username():
    return "guest"

# 使用依赖
@app.get("/user")
async def read_user(username: str = Depends(get_username)):
    return {"username": username}
```

### 带参数的依赖

```python
from typing import Annotated

def create_token(token: str, secret: str = "default"):
    return {"token": token, "secret": secret}

@app.post("/token")
async def get_token(
    token_data: Annotated[dict, Depends(create_token)]
):
    return token_data
```

## 常用依赖模式

### 1. 数据库依赖

```python
# app/core/database.py
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.engine import AsyncSessionLocal

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

### 2. 认证依赖

```python
# app/api/deps.py
from typing import Annotated
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import jwt, JWTError

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/auth/login")

async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)]
) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    return await get_user_by_id(user_id)
```

### 3. 配置依赖

```python
# app/core/config.py
from functools import lru_cache

class Settings(BaseSettings):
    DATABASE_URL: str
    SECRET_KEY: str
    ALGORITHM: str = "HS256"

    class Config:
        env_file = ".env"

@lru_cache()
def get_settings() -> Settings:
    return Settings()
```

## 依赖组织

### 分层依赖顺序

```python
# 依赖顺序：底层 → 上层

# 1. 基础设施层（最先定义）
async def get_db():
    ...

# 2. 通用业务层（可复用）
async def get_current_user(
    db: AsyncSession = Depends(get_db)
) -> User:
    ...

# 3. 特定业务层（最后定义）
async def require_admin(
    current_user: User = Depends(get_current_user)
) -> User:
    if current_user.role != "admin":
        raise HTTPException(status_code=403)
    return current_user
```

## 生命周期

| 场景 | 生命周期 |
|------|---------|
| 数据库 session | Request 级 |
| Redis 连接池 | App 级 |
| 配置对象 | App 级 |
| 当前用户 | Request 级 |
| 请求 ID | Request 级 |

## 常见陷阱

### 1. 依赖顺序错误

```python
# ❌ 错误：current_user 依赖 db，但 db 在后面定义
async def get_current_user(
    user: User = Depends(get_user),  # get_user 还没定义！
    db: AsyncSession = Depends(get_db)
):
    return user

# ✅ 正确：被依赖的放在前面
async def get_db():
    ...

async def get_user(db: AsyncSession = Depends(get_db)):
    ...
```

### 2. 忘记 yield

```python
# ❌ 错误：没有 yield
async def get_db():
    session = create_session()
    return session  # 连接不会释放！

# ✅ 正确：使用 yield
async def get_db():
    async with AsyncSessionLocal() as session:
        yield session
```

### 3. 同步函数在异步依赖中

```python
# ❌ 错误：同步函数阻塞事件循环
async def get_user():
    user = db.query(User).first()  # 同步数据库调用！
    return user

# ✅ 正确：使用异步
async def get_user():
    result = await db.execute(select(User))
    return result.scalar_one_or_none()
```
