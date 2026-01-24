# Enterprise Project Structure (Layered & Clean)

This directory structure follows the **Standard Layered Architecture**. It prioritizes clean naming conventions (context is derived from directory names) and ensures the testing suite strictly mirrors the application structure for predictability.

## The Complete Directory Tree

```text
root/
├── .github/                    # CI/CD Configurations
│   └── workflows/
│       ├── tests.yml           # Run pytest on PRs
│       └── deploy.yml          # Build & Deploy pipeline
│
├── alembic/                    # Database Migrations
│   ├── versions/               # Migration scripts (e.g., 1234_add_users.py)
│   ├── env.py                  # Alembic environment config
│   └── script.py.mako          # Migration template
│
├── deploy/                     # Deployment Infrastructure
│   ├── Dockerfile              # Production build
│   ├── docker-compose.yml      # Local dev environment
│   ├── gunicorn_conf.py        # Gunicorn configuration
│   └── nginx.conf              # Nginx reverse proxy config (optional)
│
├── scripts/                    # Maintenance & Operations
│   ├── __init__.py
│   ├── prestart.sh             # Startup hook (wait-for-db + migrate)
│   ├── lint.sh                 # Code style (ruff check)
│   ├── format.sh               # Code formatting (ruff format)
│   └── seed_db.py              # Initial data population
│
├── tests/                      # Testing Suite (Mirrors app/ structure)
│   ├── __init__.py
│   ├── conftest.py             # Global Fixtures (AsyncClient, DB Session)
│   ├── api/
│   │   ├── __init__.py
│   │   └── endpoints/
│   │       ├── __init__.py
│   │       ├── test_auth.py    # Tests for app/api/endpoints/auth.py
│   │       └── test_users.py   # Tests for app/api/endpoints/users.py
│   ├── core/
│   │   ├── __init__.py
│   │   └── test_security.py    # Tests for app/core/security.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── test_auth.py        # Tests for app/services/auth.py
│   │   └── test_user.py        # Tests for app/services/user.py
│   └── repositories/
│       ├── __init__.py
│       └── test_user.py        # Tests for app/repositories/user.py
│
├── .env.example                # Template for environment variables
├── .gitignore                  # Git ignore rules
├── alembic.ini                 # Alembic configuration
├── pyproject.toml              # Dependencies & Tool Config (uv, pytest, ruff)
├── uv.lock                     # Lock file
├── README.md                   # Project documentation
│
└── app/                        # Application Source Code
    ├── __init__.py
    ├── main.py                 # App Entrypoint & Lifespan
    │
    ├── api/                    # PRESENTATION LAYER
    │   ├── __init__.py
    │   ├── router.py           # Main router aggregator
    │   ├── deps.py             # Global dependencies (e.g., get_current_user)
    │   └── endpoints/          # Route Handlers
    │       ├── __init__.py
    │       ├── auth.py         # Login/Register endpoints
    │       ├── users.py        # User CRUD endpoints
    │       └── items.py        # Item endpoints
    │
    ├── core/                   # INFRASTRUCTURE
    │   ├── __init__.py
    │   ├── config.py           # Pydantic Settings
    │   ├── database.py         # DB Engine & Session
    │   ├── security.py         # JWT & Password logic
    │   ├── exceptions.py       # Global Exception handlers
    │   └── logging.py          # Logger configuration
    │
    ├── models/                 # DATABASE LAYER (SQLAlchemy)
    │   ├── __init__.py
    │   ├── base.py             # Declarative Base
    │   ├── user.py             # 'users' table
    │   └── item.py             # 'items' table
    │
    ├── schemas/                # VALIDATION LAYER (Pydantic)
    │   ├── __init__.py
    │   ├── common.py           # Shared schemas (Page, Error)
    │   ├── user.py             # UserCreate, UserUpdate, UserResponse
    │   ├── item.py             # Item schemas
    │   └── token.py            # Token schemas
    │
    ├── services/               # BUSINESS LOGIC LAYER
    │   ├── __init__.py
    │   ├── auth.py             # Authentication business logic
    │   └── user.py             # User management logic
    │
    └── repositories/           # DATA ACCESS LAYER
        ├── __init__.py
        ├── base.py             # Generic BaseRepository[T]
        └── user.py             # UserRepository (specific queries)
```

## Naming Conventions & Rules

### 1. Clean Naming (No Redundancy)
We avoid repeating the parent directory name in the file name. The directory provides the context.
*   **Bad**: `app/services/user_service.py` (Redundant "service")
*   **Good**: `app/services/user.py` (Clean)
*   **Usage**: `from app.services import user as user_service` (Alias on import if needed for clarity).

### 2. Mirrored Testing Structure
The `tests/` directory creates a 1:1 mirror of the `app/` directory.
*   Source: `app/services/user.py`
*   Test: `tests/services/test_user.py`

This makes it instantly obvious where to find (or add) tests for any given file.

### 3. Layer Responsibilities

*   **`api/endpoints/*.py`**:
    *   **Goal**: Handle HTTP I/O.
    *   **Allowed**: Pydantic validation, calling Services.
    *   **Forbidden**: SQL queries, Business logic branching.

*   **`services/*.py`**:
    *   **Goal**: Business Logic & Transaction Management.
    *   **Allowed**: Calling Repositories, hashing passwords, raising `BusinessException`, managing `await db.commit()`.

*   **`repositories/*.py`**:
    *   **Goal**: Database Abstraction.
    *   **Allowed**: Pure SQL/SQLAlchemy execution.
    *   **Forbidden**: Business logic, HTTP exceptions.

*   **`models/*.py`**:
    *   **Goal**: Database Schema Definition.
    *   **Allowed**: Columns, Relationships, Table args.