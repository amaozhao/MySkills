---

name: fastapi-enterprise-accelerator

description: |

  The definitive guide for building enterprise-grade FastAPI systems. Features a clean **Layered Architecture**, "uv" package management, Pydantic v2, SQLAlchemy 2.0 Async, and complete DevOps infrastructure.

  Use when: Architecting new production systems, establishing team standards, or needing production-ready patterns for Auth, Testing, Migrations, and Docker.

user-invocable: true

---

# FastAPI Enterprise Accelerator

A production-ready blueprint for building scalable Python APIs. It enforces a **Standard Layered Architecture** with strict separation of concerns, combining the maintainability of Enterprise patterns with the development speed of Modern Python tooling.

## Key Technologies

- **Package Manager**: `uv` (Rust-based, lightning fast)

- **Framework**: FastAPI 0.128+ with GZip & CORS pre-configured

- **ORM**: SQLAlchemy 2.0 (Async + Type Mapped `MappedAsDataclass`)

- **Validation**: Pydantic v2 (Strict schemas, computed fields)

- **Security**: OAuth2 with JWT (Local + SaaS support)

- **Ops**: Alembic Migrations, Docker Multi-Stage Builds, Gunicorn

- **Testing**: `pytest-asyncio` (Session-scoped DB isolation)

## 1. Project Architecture

We follow a **Clean Layered Structure**. This separates technical concerns (HTTP, DB) from business logic, making the codebase easy to navigate and test.

**👉 See Reference:** `patterns/project_structure.md`

## 2. Core Components (The "Glue")

Essential files to bootstrap the application lifecycle and standard data structures.

### Application Entrypoint

The `main.py` assembling DB connection, middlewares, router aggregation, and lifespan hooks.

**👉 See Reference:** `patterns/entrypoint.py`

### Common Schemas

Standardized Pydantic models for **Pagination** (`PagedResponse`) and API metadata.

**👉 See Reference:** `patterns/common_schemas.py`

## 3. Implementation Patterns

Copy-paste these battle-tested patterns to build your features.

### Database Setup (Robust Async)

Production-grade SQLAlchemy 2.0 setup with connection pooling and auto-reconnection.

**👉 See Reference:** `patterns/database_setup.py`

### The Generic Repository

A Type-Safe `BaseRepository[T]` to handle CRUD, Bulk Inserts, and Pydantic-to-DB mapping.

**👉 See Reference:** `patterns/base_repository.py`

### Authentication & Security

Complete JWT implementation supporting both **Self-Hosted** and **SaaS** (Clerk/Supabase) strategies.

**👉 See Reference:** `patterns/auth_security.py`

### Global Error Handling

Catch business logic errors and return standardized, frontend-friendly JSON responses (400/404/500).

**👉 See Reference:** `patterns/error_handling.py`

## 4. Quality & Operations

Infrastructure to ensure code quality and stability.

### Testing Strategy

A modern `pytest` setup separating **Unit Tests** (Mocked) from **Integration Tests** (Real DB with Transaction Rollback).

**👉 See Reference:** `patterns/testing_strategy.py`

### Database Migrations

Async Alembic configuration with auto-formatting hooks for migration scripts.

**👉 See Reference:** `patterns/migrations.py`

### Deployment Infrastructure

Dockerfile (Multi-stage), Gunicorn config (Proxy headers), and Startup Scripts.

**👉 See Reference:** `patterns/infrastructure.yaml`

## 5. Quick Start Checklist

Follow this sequence to initialize a new enterprise project:

1.  **Initialize Project**:

    ```bash

    uv init my-app

    cd my-app

    uv add "fastapi[standard]" "sqlalchemy[asyncio]" "pydantic-settings" "alembic" "asyncpg" "python-jose[cryptography]" "passlib[bcrypt]" "gunicorn"

    uv add --dev pytest pytest-asyncio httpx

    ```

2.  **Create Directory Structure**:

    ```bash

    mkdir -p app/{api/endpoints,core,services,repositories,models,schemas} tests/{unit,integration} deploy scripts alembic

    ```

3.  **Install Core Patterns** (Copy content from references):

    *   `patterns/entrypoint.py` -> `app/main.py`

    *   `patterns/database_setup.py` -> `app/core/database.py` (and `app/models/base.py`)

    *   `patterns/error_handling.py` -> `app/core/exceptions.py`

    *   `patterns/common_schemas.py` -> `app/schemas/common.py`

4.  **Install Data Layer**:

    *   `patterns/base_repository.py` -> `app/repositories/base.py`

    *   `patterns/migrations.py` -> `alembic/env.py` (and `alembic.ini`)

5.  **Install Security**:

    *   `patterns/auth_security.py` -> `app/core/security.py` (and `app/api/deps.py`)

6.  **Install Ops**:

    *   `patterns/infrastructure.yaml` -> `deploy/Dockerfile` (and other deploy files)

    *   `patterns/testing_strategy.py` -> `tests/conftest.py`

7.  **Run**:

    ```bash

    uv run fastapi dev app/main.py

    ```

## 6. Critical Rules

### DOs

1.  **Layered Responsibility**:

    *   **Router**: Parsing Request -> Calling Service.

    *   **Service**: Business Logic -> Transaction Management (`commit`).

    *   **Repository**: SQL Query Construction.

2.  **Unit of Work**: Call `await db.commit()` **ONLY** in the Service layer.

3.  **Validation**: Use Pydantic for all inputs.

### DON'Ts

1.  **No SQL in Router**: Never write `select(...)` inside an API endpoint.

2.  **No Blocking Code**: Avoid `time.sleep` or sync IO in `async def`.

3.  **No Circular Imports**: Strictly follow `Router -> Service -> Repo -> Model`.