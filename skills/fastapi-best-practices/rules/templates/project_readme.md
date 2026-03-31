# Pattern: Project Readme

This is a template for your project's `README.md`. It provides onboarding instructions tailored to the **FastAPI Enterprise Accelerator** stack (uv, Alembic, Docker, Pytest).

## The Implementation (`README.md`)

```markdown
# FastAPI Enterprise Project

An enterprise-grade, async Python API built with Layered Architecture.

## ⚡ Tech Stack

*   **Framework**: FastAPI (0.128+)
*   **Language**: Python 3.12+
*   **Package Manager**: [uv](https://github.com/astral-sh/uv)
*   **Database**: PostgreSQL 15+ with Async SQLAlchemy 2.0
*   **Migrations**: Alembic
*   **Testing**: Pytest + AsyncIO

## 🚀 Quick Start

### 1. Prerequisites
*   **Python 3.12+** installed.
*   **uv** installed (Recommended way to manage python packages).
    *   Mac/Linux: `curl -LsSf https://astral.sh/uv/install.sh | sh`
    *   Windows: `powershell -c "irm https://astral.sh/uv/install.ps1 | iex"`
*   **Docker & Docker Compose** (For running the database).

### 2. Setup Environment

1.  **Clone the repository**:
    ```bash
    git clone <your-repo-url>
    cd <project-folder>
    ```

2.  **Install dependencies**:
    ```bash
    uv sync
    ```

3.  **Start Infrastructure (PostgreSQL)**:
    ```bash
    # Starts the DB container in background
    docker-compose up -d db
    ```

4.  **Configure Environment**:
    ```bash
    cp .env.example .env
    # Edit .env if needed (Default DB URL usually works with docker-compose)
    ```

5.  **Run Migrations**:
    Initialize the database schema.
    ```bash
    bash scripts/prestart.sh
    # OR manually: uv run alembic upgrade head
    ```

### 3. Run Application

Start the development server with hot-reload:

```bash
uv run fastapi dev app/main.py
```

*   **API**: http://localhost:8000
*   **Docs**: http://localhost:8000/docs

---

## 🛠 Common Commands

### Database Migrations (Alembic)
We use Alembic to manage database schema changes.

*   **Create a new migration** (Run this after changing `app/models/`):
    ```bash
    uv run alembic revision --autogenerate -m "describe_your_change"
    ```
    *Check the generated file in `alembic/versions/` to verify it.*

*   **Apply migrations**:
    ```bash
    uv run alembic upgrade head
    ```

*   **Revert last migration**:
    ```bash
    uv run alembic downgrade -1
    ```

### Testing (Pytest)
We strictly separate Unit tests (Fast, Mocked) and Integration tests (Slow, Real DB).

*   **Run All Tests**:
    ```bash
    uv run pytest
    ```

*   **Run Unit Tests Only**:
    ```bash
    uv run pytest -m unit
    ```

*   **Run Integration Tests Only**:
    ```bash
    uv run pytest -m integration
    ```

### Code Quality

*   **Format Code**:
    ```bash
    uv run ruff format .
    ```

*   **Lint Code**:
    ```bash
    uv run ruff check .
    ```

---

## 🐳 Deployment

The project includes a multi-stage `Dockerfile` optimized for production.

### Build & Run

```bash
# 1. Build Image
docker build -t my-app -f deploy/Dockerfile .

# 2. Run Container
# Ensure you pass the correct DATABASE_URL env var
docker run -p 8000:8000 --env-file .env my-app
```

### Production Checklist
1.  Set `DEBUG=False` in `.env`.
2.  Set a strong `SECRET_KEY`.
3.  Ensure `DATABASE_URL` points to your production RDS/CloudSQL instance.
4.  Run behind a Load Balancer (Nginx/ALB) handling SSL termination.
```