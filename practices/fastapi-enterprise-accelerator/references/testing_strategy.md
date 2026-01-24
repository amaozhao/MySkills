、# Pattern: Testing Strategy

This pattern implements a split-layer testing strategy using `pytest` and `httpx`. It is updated for `pytest-asyncio` v0.23+ standards and ensures strict test isolation via database transaction rollbacks.

## 1. Configuration (`pyproject.toml`)

Modern `pytest-asyncio` configuration requires defining the loop scope to allow session-scoped DB fixtures.

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
# CRITICAL: Allows session-scoped fixtures (like db engine) to work with async tests
asyncio_default_fixture_loop_scope = "session"
testpaths = ["tests"]
python_files = "test_*.py"
addopts = "-v --disable-warnings"
markers = [
    "unit: pure logic tests (fast, no db)",
    "integration: end-to-end tests (slow, real db)",
]
```

## 2. Global Fixtures (`tests/conftest.py`)

This setup creates a **single DB connection** for the entire test session but wraps **each test function** in a transaction that rolls back at the end. This keeps tests clean and extremely fast.

```python
import pytest
from typing import AsyncGenerator
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

from app.main import app
from app.core.config import settings
from app.core.database import Base, get_db

# -------------------------------------------------------------------------
# 1. Database Infrastructure (Session Scope)
# -------------------------------------------------------------------------

# Use the configured DB URL. 
# INSTRUCTION: For CI/CD, ensure your pipeline sets a separate DATABASE_URL 
# or ensure this logic runs against a disposable container.
TEST_DATABASE_URL = str(settings.DATABASE_URL)

engine = create_async_engine(TEST_DATABASE_URL, echo=False)

# Session factory for tests
TestingSessionLocal = async_sessionmaker(
    bind=engine, 
    class_=AsyncSession, 
    expire_on_commit=False, 
    autoflush=False
)

@pytest.fixture(scope="session", autouse=True)
async def setup_database_schema():
    """
    Create tables once at start of test session, drop at end.
    This is much faster than recreating tables for every test.
    """
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
        await conn.run_sync(Base.metadata.create_all)
    
    yield
    
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    
    await engine.dispose()

# -------------------------------------------------------------------------
# 2. Per-Test Isolation (Function Scope)
# -------------------------------------------------------------------------

@pytest.fixture(scope="function")
async def db() -> AsyncGenerator[AsyncSession, None]:
    """
    Creates a nested transaction for each test.
    Rolls back at the end, leaving the DB clean (empty tables) for the next test.
    """
    connection = await engine.connect()
    transaction = await connection.begin()
    
    # Bind session to this specific connection/transaction
    session = TestingSessionLocal(bind=connection)
    
    yield session
    
    # Cleanup
    await session.close()
    await transaction.rollback() # The Magic: Undo all changes
    await connection.close()

# -------------------------------------------------------------------------
# 3. HTTP Client
# -------------------------------------------------------------------------

@pytest.fixture(scope="function")
async def client(db: AsyncSession) -> AsyncGenerator[AsyncClient, None]:
    """
    A test client that overrides the 'get_db' dependency 
    to use our transaction-bound test session.
    """
    app.dependency_overrides[get_db] = lambda: db
    
    async with AsyncClient(
        transport=ASGITransport(app=app), 
        base_url="http://test"
    ) as ac:
        yield ac
        
    app.dependency_overrides.clear()
```

## 3. Unit Tests (`tests/services/test_user.py`)

**Goal**: Test business logic in isolation.
**Technique**: Use `unittest.mock` or direct instantiation. Do NOT use the `db` fixture.

```python
import pytest
from unittest.mock import AsyncMock, MagicMock
from app.services.user import UserService
from app.schemas.user import UserCreate
from app.core.exceptions import BusinessException

@pytest.mark.asyncio
async def test_create_user_logic_success():
    # 1. Arrange
    mock_repo = AsyncMock()
    mock_repo.get_by_email.return_value = None  # Logic check: User must not exist
    
    # Mock the return value of repo.create
    created_user_mock = MagicMock(id=1, email="test@example.com", hashed_password="xxx")
    mock_repo.create.return_value = created_user_mock
    
    mock_db = AsyncMock()
    
    service = UserService(mock_db)
    service.repo = mock_repo # Inject mock
    
    # 2. Act
    user_in = UserCreate(email="test@example.com", password="password123")
    result = await service.create_user(user_in)
    
    # 3. Assert
    assert result.email == "test@example.com"
    mock_repo.create.assert_called_once()
    mock_db.commit.assert_called_once() # Verify Unit of Work commit

@pytest.mark.asyncio
async def test_create_user_logic_duplicate():
    # 1. Arrange
    mock_repo = AsyncMock()
    mock_repo.get_by_email.return_value = True # Simulate user exists
    
    service = UserService(AsyncMock())
    service.repo = mock_repo
    
    # 2. Act & Assert
    with pytest.raises(BusinessException) as exc:
        await service.create_user(UserCreate(email="duplicate@test.com", password="123"))
    
    assert exc.value.code == "USER_ALREADY_EXISTS"
    mock_repo.create.assert_not_called()
```

## 4. Integration Tests (`tests/api/endpoints/test_auth.py`)

**Goal**: Verify API-to-DB roundtrip.
**Technique**: Use `client` and `db`. URL paths must match the flattened structure.

```python
import pytest
from httpx import AsyncClient
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from app.models.user import User

@pytest.mark.asyncio
async def test_register_and_login_flow(client: AsyncClient, db: AsyncSession):
    """
    Test the full registration and login lifecycle against a real DB.
    """
    email = "integration@test.com"
    password = "strongpassword"

    # 1. Register (POST /api/auth/register)
    # Note: Path is flat, no '/v1'
    resp = await client.post("/api/auth/register", json={
        "email": email,
        "password": password
    })
    assert resp.status_code == 200
    data = resp.json()
    assert data["email"] == email
    assert "password" not in data # Ensure security
    
    # 2. Verify DB Side Effect (Direct SQL check)
    # This proves the API actually wrote to the DB
    stmt = select(User).where(User.email == email)
    result = await db.execute(stmt)
    user_in_db = result.scalars().first()
    assert user_in_db is not None
    
    # 3. Login (POST /api/auth/login)
    # OAuth2 form expects 'username' field for email
    login_resp = await client.post("/api/auth/login", data={
        "username": email, 
        "password": password
    })
    assert login_resp.status_code == 200
    token_data = login_resp.json()
    assert "access_token" in token_data
    assert token_data["token_type"] == "bearer"
```