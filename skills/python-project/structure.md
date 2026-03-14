---
name: python-structure
description: Python 项目结构 - src-layout、pyproject.toml、导入问题
---

# Python Project Structure

## 概述

现代 Python 项目使用 src-layout 和 pyproject.toml 实现干净的导入和标准化工具配置。

## 推荐结构

```
project/
├── src/                      # src-layout（避免导入问题）
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
├── pyproject.toml            # 现代配置
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

| 布局 | 优点 | 缺点 |
|------|------|------|
| **src-layout** | 导入干净，无包名冲突 | 多一层 src/ |
| **flat** | 简单 | 从根目录运行会失败 |

```python
# src-layout（推荐）
from mypackage.core.config import settings

# flat（避免）
from core.config import settings  # 从根目录运行会失败！
```

---

## 相关文件

- [type-hints.md](./type-hints.md) - 类型提示
- [packaging.md](./packaging.md) - 打包发布
