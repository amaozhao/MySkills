---
name: fastapi-testing-strategy
description: Use when testing FastAPI APIs that need database isolation, mocking strategies, or split unit/integration test workflows
---

# FastAPI Testing Strategy

## Overview
**Split-layer testing: mocked unit tests for logic, transaction-rolled-back integration tests for API-DB roundtrips.**

## When to Use
- Writing tests for FastAPI endpoints
- Need clean database state between tests
- Separating business logic tests from API tests
- Using pytest-asyncio with async database

## Pytest Config

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "session"
testpaths = ["tests"]
python_files = "test_*.py"
markers = [
    "unit: pure logic tests",
    "integration: end-to-end tests",
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
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as ac:
        yield ac
    app.dependency_overrides.clear()
```

## Unit Test (Mocked)

```python
# tests/services/test_user.py
import pytest
from unittest.mock import AsyncMock, MagicMock
from app.services.user import UserService
from app.schemas.user import UserCreate
from app.core.exceptions import BusinessException

@pytest.mark.asyncio
async def test_create_user_success():
    mock_repo = AsyncMock()
    mock_repo.get_by_email.return_value = None
    mock_repo.create.return_value = MagicMock(id=1, email="test@test.com")

    service = UserService(AsyncMock())
    service.repo = mock_repo

    result = await service.create_user(UserCreate(email="test@test.com", password="pass"))

    assert result.email == "test@test.com"
    mock_repo.create.assert_called_once()
    mock_db.commit.assert_called_once()

@pytest.mark.asyncio
async def test_create_user_duplicate():
    mock_repo = AsyncMock()
    mock_repo.get_by_email.return_value = True

    service = UserService(AsyncMock())
    service.repo = mock_repo

    with pytest.raises(BusinessException) as exc:
        await service.create_user(UserCreate(email="exists@test.com", password="pass"))

    assert exc.value.code == "USER_ALREADY_EXISTS"
```

## Integration Test (Real DB)

```python
# tests/api/test_users.py
import pytest
from httpx import AsyncClient
from sqlalchemy import select
from app.models.user import User

@pytest.mark.asyncio
async def test_create_user_api(client: AsyncClient, db: AsyncSession):
    resp = await client.post("/api/users", json={"email": "test@test.com", "password": "pass"})
    assert resp.status_code == 200

    # Verify DB state
    result = await db.execute(select(User).where(User.email == "test@test.com"))
    assert result.scalars().first() is not None
```

## Test Layering

| Layer | Fixture | Isolation | Purpose |
|-------|---------|-----------|---------|
| Unit | `mock_repo` | Mocked DB | Test business logic |
| Integration | `client + db` | Transaction rollback | Test API-DB roundtrip |

## The Bottom Line

**Unit tests mock the repo; integration tests use transaction rollback.**
