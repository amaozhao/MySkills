---
name: fastapi-testing
description: FastAPI 测试策略 - 单元测试(mock)、集成测试(transaction rollback)
---

# FastAPI Testing Strategy

## 概述

分层测试：单元测试 mock 仓库，集成测试使用事务回滚保证数据库隔离。

## Pytest 配置

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "session"
testpaths = ["tests"]
python_files = "test_*.py"
markers = [
    "unit: 纯逻辑测试",
    "integration: 端到端测试",
]
```

## Fixtures

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from app.main import app
from app.core.database import get_db, Base
from app.core.config import settings

TEST_DATABASE_URL = str(settings.DATABASE_URL)
engine = create_async_engine(TEST_DATABASE_URL, echo=False)

TestingSessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,
    autoflush=False,
)


@pytest.fixture(scope="session", autouse=True)
async def setup_db():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()


@pytest.fixture(scope="function")
async def db():
    connection = await engine.connect()
    transaction = await connection.begin()
    session = TestingSessionLocal(bind=connection)
    yield session
    await session.close()
    await transaction.rollback()
    await connection.close()


@pytest.fixture(scope="function")
async def client(db: AsyncSession):
    app.dependency_overrides[get_db] = lambda: db
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()
```

## 单元测试（Mock）

```python
# tests/services/test_user.py
import pytest
from unittest.mock import AsyncMock, MagicMock
from app.services.user import UserService
from app.schemas.user import UserCreate
from app.core.exceptions import BusinessException


@pytest.mark.asyncio
async def test_create_user_success():
    # Mock Repository
    mock_repo = AsyncMock()
    mock_repo.get_by_email.return_value = None
    mock_repo.create.return_value = MagicMock(id=1, email="test@test.com")

    # 创建 Service 并注入 mock
    service = UserService(AsyncMock())
    service.repo = mock_repo

    # 执行
    result = await service.create_user(UserCreate(email="test@test.com", password="pass"))

    # 断言
    assert result.email == "test@test.com"
    mock_repo.create.assert_called_once()
    mock_db.commit.assert_called_once()


@pytest.mark.asyncio
async def test_create_user_duplicate():
    mock_repo = AsyncMock()
    mock_repo.get_by_email.return_value = True  # 已存在

    service = UserService(AsyncMock())
    service.repo = mock_repo

    with pytest.raises(BusinessException) as exc:
        await service.create_user(UserCreate(email="exists@test.com", password="pass"))

    assert exc.value.code == "USER_ALREADY_EXISTS"
```

## 集成测试（真实 DB）

```python
# tests/api/test_users.py
import pytest
from httpx import AsyncClient
from sqlalchemy import select
from app.models.user import User


@pytest.mark.asyncio
async def test_create_user_api(client: AsyncClient, db: AsyncSession):
    # 调用 API
    resp = await client.post("/api/users", json={
        "email": "test@test.com",
        "password": "pass"
    })
    assert resp.status_code == 200

    # 验证数据库状态
    result = await db.execute(select(User).where(User.email == "test@test.com"))
    assert result.scalars().first() is not None
```

## 测试分层

| 层级 | Fixture | 隔离方式 | 用途 |
|------|---------|---------|------|
| 单元 | mock_repo | Mock | 测试业务逻辑 |
| 集成 | client + db | 事务回滚 | 测试 API-DB 往返 |

---

## 相关文件

- [architecture.md](./architecture.md) - 分层架构
- [errors.md](./errors.md) - 异常处理
- [DI.md](./DI.md) - 依赖注入
