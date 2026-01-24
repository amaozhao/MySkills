---
name: fastapi-enterprise-accelerator
description: |
  The definitive guide for building enterprise-grade FastAPI systems. Features a clean **Layered Architecture**, "uv" package management, Pydantic v2, SQLAlchemy 2.0 Async, and complete DevOps infrastructure.

  Use when: Architecting new production systems, establishing team standards, or needing production-ready references for Auth, Testing, Migrations, and Docker.
user-invocable: true
---

# FastAPI Enterprise Accelerator

A production-ready blueprint for building scalable Python APIs. It enforces a **Standard Layered Architecture** with strict separation of concerns, combining the maintainability of Enterprise references with the development speed of Modern Python tooling.

## Key Technologies

- **Package Manager**: `uv` (Rust-based, lightning fast)
- **Framework**: FastAPI 0.128+ with GZip & CORS pre-configured
- **ORM**: SQLAlchemy 2.0 (Async + Type Mapped `MappedAsDataclass`)
- **Validation**: Pydantic v2 (Strict schemas, computed fields)
- **Security**: OAuth2 with JWT (Local + SaaS support)
- **Ops**: Alembic Migrations, Docker Multi-Stage Builds, Gunicorn
- **Testing**: `pytest-asyncio` (Session-scoped DB isolation)

## 1. Project Documentation

Start here. These references establish the structure and onboarding guide for your project.

### Project Architecture
The folder structure explaining the **Layered Architecture** (API, Services, Repositories).
**👉 See Reference:** `references/project_structure.md`

### Project Readme
The standard `README.md` template containing runbooks for install, test, and deploy.
**👉 See Reference:** `references/project_readme.md`

## 2. Core Components (The "Glue")

Essential files to bootstrap the application lifecycle and standard data structures.

### Application Entrypoint
The `main.py` assembling DB connection, middlewares, router aggregation, and lifespan hooks.
**👉 See Reference:** `references/entrypoint.py`

### Common Schemas
Standardized Pydantic models for **Pagination** (`PagedResponse`), Errors, and Metadata.
**👉 See Reference:** `references/common_schemas.py`

## 3. Implementation references

Copy-paste these battle-tested references to build your features.

### Database Setup (Robust Async)
Production-grade SQLAlchemy 2.0 setup with connection pooling and auto-reconnection.
**👉 See Reference:** `references/database_setup.py`

### The Generic Repository
A Type-Safe `BaseRepository[T]` to handle CRUD, Bulk Inserts, and Pydantic-to-DB mapping.
**👉 See Reference:** `references/base_repository.py`

### Authentication & Security
Complete JWT implementation supporting both **Self-Hosted** and **SaaS** (Clerk/Supabase) strategies.
**👉 See Reference:** `references/auth_security.py`

### Global Error Handling
Catch business logic errors and return standardized, frontend-friendly JSON responses (400/404/500).
**👉 See Reference:** `references/error_handling.py`

## 4. Quality & Operations

Infrastructure to ensure code quality and stability.

### Testing Strategy
A modern `pytest` setup separating **Unit Tests** (Mocked) from **Integration Tests** (Real DB with Transaction Rollback).
**👉 See Reference:** `references/testing_strategy.py`

### Database Migrations
Async Alembic configuration with auto-formatting hooks (`ruff`) for migration scripts.
**👉 See Reference:** `references/migrations.py`

### Deployment Infrastructure
Dockerfile (Multi-stage), Gunicorn config (Proxy headers), and Startup Scripts.
**👉 See Reference:** `references/infrastructure.yaml`

## 5. Quick Start Checklist

Follow this sequence to initialize a new enterprise project:

1.  **Initialize Project**:
    ```bash
    uv init my-app
    cd my-app
    uv add "fastapi[standard]" "sqlalchemy[asyncio]" "pydantic-settings" "alembic" "asyncpg" "python-jose[cryptography]" "passlib[bcrypt]" "gunicorn"
    uv add --dev pytest pytest-asyncio httpx ruff
    ```

2.  **Create Directory Structure**:
    ```bash
    mkdir -p app/{api/endpoints,core,services,repositories,models,schemas} tests/{unit,integration} deploy scripts alembic
    ```

3.  **Install Core references** (Copy content from references):
    *   `references/project_readme.md` -> `README.md`
    *   `references/entrypoint.py` -> `app/main.py`
    *   `references/database_setup.py` -> `app/core/database.py` (and `app/models/base.py`)
    *   `references/error_handling.py` -> `app/core/exceptions.py`
    *   `references/common_schemas.py` -> `app/schemas/common.py`

4.  **Install Data Layer**:
    *   `references/base_repository.py` -> `app/repositories/base.py`
    *   `references/migrations.py` -> `alembic/env.py` (and `alembic.ini`)

5.  **Install Security**:
    *   `references/auth_security.py` -> `app/core/security.py` (and `app/api/deps.py`)

6.  **Install Ops**:
    *   `references/infrastructure.yaml` -> `deploy/Dockerfile` (and other deploy files)
    *   `references/testing_strategy.py` -> `tests/conftest.py`

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
