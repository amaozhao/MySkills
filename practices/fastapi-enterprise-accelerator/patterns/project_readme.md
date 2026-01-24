# Reference 11: patterns/project_readme.md

# [Project Name]

An enterprise-grade FastAPI application built with Layered Architecture.

## ⚡ Tech Stack

*   **Framework**: FastAPI
*   **Package Manager**: uv
*   **Database**: PostgreSQL + Async SQLAlchemy 2.0
*   **Migrations**: Alembic
*   **Testing**: Pytest + AsyncIO

## 🚀 Quick Start

### 1. Prerequisites
*   Python 3.12+
*   [uv](https://github.com/astral-sh/uv) (Recommended)
*   Docker & Docker Compose

### 2. Local Development

```bash
# 1. Install dependencies
uv sync

# 2. Start Infrastructure (DB)
docker-compose up -d db

# 3. Apply Migrations
bash scripts/prestart.sh
# OR manually: uv run alembic upgrade head

# 4. Run Development Server
uv run fastapi dev app/main.py
