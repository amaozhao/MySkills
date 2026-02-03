---
name: python-project-structure
description: Use when starting a new Python project, setting up pyproject.toml, or organizing code with src-layout pattern
---

# Python Project Structure

## Overview
**Modern Python projects use src-layout and pyproject.toml for clean imports and standardized tooling.**

## When to Use
- Starting a new Python project
- Restructuring existing code
- Setting up pyproject.toml
- Fixing import issues

## Recommended Structure

```
project/
├── src/                      # src-layout (prevents import issues)
│   └── mypackage/
│       ├── __init__.py
│       ├── core/
│       ├── models/
│       ├── services/
│       └── api/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── conftest.py
├── pyproject.toml            # Modern configuration
├── .env
├── .gitignore
└── README.md
```

## pyproject.toml

```toml
[project]
name = "myproject"
version = "1.0.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.109.0",
    "sqlalchemy>=2.0.0",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
target-version = "py312"
line-length = 100

[tool.mypy]
python_version = "3.12"
warn_return_any = true
```

## src-layout vs Flat-layout

| Layout | Pros | Cons |
|--------|------|------|
| **src-layout** | Clean imports, no package name conflicts | Extra `src/` level |
| **flat** | Simple structure | `__init__.py` required in root |

```python
# src-layout (recommended)
from mypackage.core.config import settings

# Flat-layout (avoid)
from core.config import settings  # Fails if running from root!
```

## The Bottom Line

**Use src-layout + pyproject.toml for modern Python projects.**
