---
name: fastapi-dependency-injection
description: FastAPI 依赖注入最佳实践 - 依赖组织、Depends 用法、生命周期管理
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

---

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

# 使用
@app.get("/users")
async def list_users(
    db: AsyncSession = Depends(get_db)
):
    ...
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

# 使用
@app.get("/info")
async def get_info(
    settings: Annotated[Settings, Depends(get_settings)]
):
    return {"debug": settings.DEBUG}
```

### 4. 可选依赖

```python
from typing import Optional

async def get_cache():
    # 实际项目中连接 Redis
    return {"get": lambda k: None, "set": lambda k, v: None}

@app.get("/items")
async def list_items(
    cache: Optional[dict] = Depends(get_cache)
):
    if cache:
        # 使用缓存
        pass
    return [{"id": 1}]
```

---

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

### 依赖链

```python
# app/api/users.py
@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),           # 链1
    current_user: User = Depends(get_current_user) # 链2
):
    # db 和 current_user 都会被自动注入
    ...
```

---

## 依赖覆盖（测试）

### 覆盖数据库

```python
# tests/conftest.py
from unittest.mock import AsyncMock

@pytest.fixture
def mock_db():
    return AsyncMock(spec=AsyncSession)

@pytest.fixture
def client(mock_db):
    from app.main import app
    from app.api.deps import get_db

    app.dependency_overrides[get_db] = lambda: mock_db

    # 使用 TestClient 或 AsyncClient
    yield TestClient(app)

    app.dependency_overrides.clear()
```

### 覆盖认证

```python
# 测试受保护端点
async def test_get_me(client, mock_db):
    # 覆盖 get_current_user 返回测试用户
    async def override_get_current_user():
        return User(id=1, username="test", role="user")

    app.dependency_overrides[get_current_user] = override_get_current_user

    response = client.get("/api/me")
    assert response.status_code == 200
```

---

## 生命周期

### Request 级（默认）

```python
# 每个请求创建新实例
async def get_db():
    session = create_session()
    yield session
    await session.close()

# 每次请求：get_db() → 请求处理 → close()
```

### App 级（单例）

```python
# 应用启动时创建，整个应用生命周期复用
settings = Settings()  # 模块级单例

redis = Redis.from_url("redis://localhost")

# 整个应用生命周期：只创建一次
```

### 选择原则

| 场景 | 生命周期 |
|------|---------|
| 数据库 session | Request 级 |
| Redis 连接池 | App 级 |
| 配置对象 | App 级 |
| 当前用户 | Request 级 |
| 请求 ID | Request 级 |

---

## 常见陷阱

### 1. 依赖顺序错误

```python
# ❌ 错误：current_user 依赖 db，但 db 在后面定义
async def get_current_user(
    user: User = Depends(get_user),  # get_user 还没定义！
    db: AsyncSession = Depends(get_db)
):
    return user

async def get_user(...):
    ...

async def get_db():
    ...

# ✅ 正确：被依赖的放在前面
async def get_db():
    ...

async def get_user(db: AsyncSession = Depends(get_db)):
    ...

async def get_current_user(
    user: User = Depends(get_user),
    db: AsyncSession = Depends(get_db)  # 冗余但明确
):
    return user
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

---

## 最佳实践

### 1. 依赖分层

```python
# app/api/deps.py - 集中管理通用依赖

from app.core.database import get_db
from app.core.security import get_current_user
from app.core.config import get_settings

__all__ = ["get_db", "get_current_user", "get_settings"]
```

### 2. 类型注解

```python
from typing import Annotated

# ✅ 推荐：使用 Annotated
@app.get("/users")
async def list_users(
    db: Annotated[AsyncSession, Depends(get_db)]
):
    ...

# ❌ 避免：直接使用 Depends
async def list_users(
    db = Depends(get_db)
):
    ...
```

### 3. 依赖隔离

```python
# tests/api/test_users.py
from app.main import app
from app.api import deps

@pytest.fixture(autouse=True)
def reset_overrides():
    yield
    deps.app.dependency_overrides.clear()
```

---

## 完整示例

```python
# app/main.py
from fastapi import FastAPI, Depends
from typing import AsyncGenerator

app = FastAPI()

# 1. 基础设施依赖
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session

# 2. 业务依赖
async def get_current_user(
    db: AsyncSession = Depends(get_db)
) -> User:
    ...

# 3. 端点使用
@app.get("/items")
async def list_items(
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    return await db.execute(select(Item))
```

---

## 相关文件

- [config.md](./config.md) - 配置管理
- [architecture.md](./architecture.md) - 分层架构
- [database.md](./database.md) - 数据库配置
- [auth.md](./auth.md) - 认证实现
