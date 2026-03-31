---
name: python-testing
description: Python 单元测试 - pytest、fixtures、mock、async 测试
---

# Python Unit Testing

## 概述

Python 单元测试最佳实践：pytest 配置、fixtures、mock、异步测试。

## pytest 配置

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = "test_*.py"
python_classes = "Test*"
python_functions = "test_*"
addopts = "-v --tb=short"
asyncio_mode = "auto"
```

## fixtures

```python
# tests/conftest.py
import pytest
from unittest.mock import MagicMock

@pytest.fixture
def mock_db():
    return MagicMock()

@pytest.fixture
def sample_user():
    return {"id": 1, "name": "Test User", "email": "test@example.com"}

@pytest.fixture
async def async_client():
    # 异步 fixture 示例
    client = await create_test_client()
    yield client
    await client.close()
```

## 单元测试

```python
# tests/test_user_service.py
import pytest
from unittest.mock import AsyncMock, MagicMock

def test_user_creation():
    """测试用户创建"""
    user_data = {"name": "John", "email": "john@example.com"}

    # 测试逻辑
    user = User(**user_data)

    assert user.name == "John"
    assert user.email == "john@example.com"

def test_user_validation():
    """测试用户验证"""
    with pytest.raises(ValueError):
        User(name="", email="invalid")

def test_user_mock():
    """使用 mock"""
    mock_repo = MagicMock()
    mock_repo.get_by_email.return_value = None

    result = mock_repo.get_by_email("test@example.com")

    assert result is None
    mock_repo.get_by_email.assert_called_once_with("test@example.com")
```

## 异步测试

```python
# tests/test_async.py
import pytest

@pytest.mark.asyncio
async def test_async_fetch():
    """异步函数测试"""
    result = await async_fetch_data()
    assert result is not None

@pytest.mark.asyncio
async def test_async_context_manager():
    """异步上下文管理器测试"""
    async with await get_resource() as resource:
        assert resource.is_ready()
```

## Mock 进阶

```python
from unittest.mock import patch, AsyncMock

# Mock 函数
@patch("module.function_name")
def test_with_mock(mock_fn):
    mock_fn.return_value = "mocked"
    result = module.other_function()
    assert result == "mocked"

# Mock 异步函数
@patch("module.async_function", new_callable=AsyncMock)
async def test_async_mock(mock_fn):
    mock_fn.return_value = "async mocked"
    result = await module.async_other_function()
    assert result == "async mocked"
```

## 测试组织

```
tests/
├── conftest.py           # 共享 fixtures
├── unit/
│   ├── test_user.py
│   └── test_order.py
└── integration/
    └── test_api.py
```
