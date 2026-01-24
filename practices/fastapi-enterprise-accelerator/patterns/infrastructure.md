# Pattern: Deployment Infrastructure

This pattern provides a complete, production-ready deployment pipeline. It fixes common omissions like **`.dockerignore`** (crucial for build context size) and adds **Proxy Headers** support in Gunicorn (crucial for security behind Load Balancers).

## 1. Production Dockerfile (`deploy/Dockerfile`)

Optimized for security (non-root) and caching. It uses a multi-stage build to keep the final image size minimal.

```dockerfile
# ==========================================
# Stage 1: Builder
# ==========================================
FROM python:3.12-slim-bookworm as builder

# Install uv (The fastest Python package manager)
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

# Compile bytecode for faster startup
ENV UV_COMPILE_BYTECODE=1 
ENV UV_LINK_MODE=copy

WORKDIR /app

# Install dependencies (Cached layer)
# We assume uv.lock exists. If not, run 'uv pip compile' locally first.
# This layer will be cached unless pyproject.toml/uv.lock changes.
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project --no-dev

# ==========================================
# Stage 2: Runner
# ==========================================
FROM python:3.12-slim-bookworm

# Security: Create a non-root user
RUN groupadd -r app && useradd -r -g app app

WORKDIR /app

# Copy Virtual Environment from builder
# This isolates the runtime environment completely
COPY --from=builder /app/.venv /app/.venv

# Set Path to use the venv by default
ENV PATH="/app/.venv/bin:$PATH"
ENV PYTHONPATH="/app"
# Ensure logs are streamed to stdout immediately (no buffering)
ENV PYTHONUNBUFFERED=1

# Copy Application Code
# Note: reliance on .dockerignore to filter out garbage is critical here
COPY . .

# Grant permissions to the non-root user
RUN chown -R app:app /app

# Switch user
USER app

# Expose port (Documentation only, runtime port is defined by Gunicorn)
EXPOSE 8000

# Entrypoint
# We point to the specific config file location in the deploy folder
CMD ["bash", "-c", "bash ./scripts/prestart.sh && gunicorn app.main:app -c deploy/gunicorn_conf.py"]
```

## 2. Docker Ignore (`.dockerignore`)

**Missing in most tutorials**, but vital. Create this file in the project root to prevent local virtualenvs, git history, and secrets from being copied into the container build context.

```text
# Git
.git
.gitignore

# Python Artifacts
__pycache__
*.pyc
*.pyo
*.pyd
.Python
env/
venv/
.venv/
pip-log.txt

# Testing & Quality
.pytest_cache
.coverage
htmlcov/
.mypy_cache/
.ruff_cache/
tests/

# System
.DS_Store

# Docker (Don't copy the Dockerfile into itself)
deploy/Dockerfile
docker-compose.yml

# Environment Secrets (CRITICAL)
.env
.env.*
```

## 3. Startup Script (`scripts/prestart.sh`)

Ensures the DB is actually accessible before the app crashes on startup. Includes robust error handling.

```bash
#! /usr/bin/env bash

set -e

echo "INFO: Running pre-start script..."

# 1. Wait for DB connection
# This python script attempts to connect using the app's settings.
echo "INFO: Checking database connection..."
python << END
import sys
import asyncio
from sqlalchemy.ext.asyncio import create_async_engine
from app.core.config import settings

async def check():
    try:
        # Create a temp engine just to check connectivity
        engine = create_async_engine(str(settings.DATABASE_URL))
        async with engine.connect() as conn:
            pass
        print("SUCCESS: Database is reachable.")
    except Exception as e:
        print(f"ERROR: Connection failed. {e}")
        sys.exit(1)
    
asyncio.run(check())
END

# 2. Run Migrations
# This ensures the schema is always up-to-date with the code
echo "INFO: Applying Alembic migrations..."
alembic upgrade head

# 3. (Optional) Seed Data
# echo "INFO: Seeding initial data..."
# python -m scripts.seed_db

echo "INFO: Pre-start complete. Starting server."
```

## 4. Gunicorn Config (`deploy/gunicorn_conf.py`)

Enhanced to handle **Load Balancers** (AWS ALB, Nginx, Cloudflare) correctly by trusting forwarded headers.

```python
import multiprocessing
import os

# Binding
# Allow dynamic port via env var (Default behavior for Cloud Run / Heroku)
port = os.getenv("PORT", "8000")
bind = f"0.0.0.0:{port}"

# Workers
# Async workers (Uvicorn) allow high concurrency per worker.
# We don't need many workers, usually 1 per core is sufficient.
workers_per_core = 1
cores = multiprocessing.cpu_count()
default_web_concurrency = workers_per_core * cores
web_concurrency = os.getenv("WEB_CONCURRENCY", None) or default_web_concurrency
workers = int(web_concurrency)

# Worker Class
# This is CRITICAL for FastAPI
worker_class = "uvicorn.workers.UvicornWorker"

# Logging
loglevel = "info"
accesslog = "-"  # stdout
errorlog = "-"   # stderr

# Proxy Handling (CRITICAL for Production)
# Trust the "X-Forwarded-For" headers from the load balancer.
# Without this, FastAPI will think all requests come from the LB IP (e.g. 10.0.0.1)
forwarded_allow_ips = "*"
proxy_allow_ips = "*"

# Timeouts
keepalive = 120
timeout = 120
```

## 5. Local Orchestration (`docker-compose.yml`)

This file is optimized for local development, enabling hot-reloading.

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: deploy/Dockerfile
    container_name: fastapi_app
    # Override CMD for Hot-Reload during development
    # Note: We mount the volume below so changes are reflected instantly
    command: bash -c "bash ./scripts/prestart.sh && uv run uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload"
    volumes:
      - ./app:/app/app        # Sync code changes
      - ./tests:/app/tests    # Sync tests (so you can run tests in container)
      - ./alembic:/app/alembic # Sync migrations
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://postgres:postgres@db:5432/app_db
      - DEBUG=True
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app_network

  db:
    image: postgres:15-alpine
    container_name: fastapi_db
    restart: always
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=app_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      # Native Postgres health check
      test: ["CMD-SHELL", "pg_isready -U postgres -d app_db"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app_network

volumes:
  postgres_data:

networks:
  app_network:
    driver: bridge
```