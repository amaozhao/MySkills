# Pattern: Application Entrypoint

This pattern assembles the database, middleware, routers, and exception handlers into a running application. It uses the modern **Lifespan** context manager for safe startup/shutdown sequences.

## The Implementation (`app/main.py`)

```python
from contextlib import asynccontextmanager
import logging

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.responses import RedirectResponse

from app.core.config import settings
from app.core.database import engine
from app.core.exceptions import setup_exception_handlers
# Note: Ensure you have created app/api/router.py to aggregate your module routers
from app.api.router import api_router  

# 1. Setup Logging
# In production, this should ideally be configured via a logging config file
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("uvicorn.startup")

# 2. Lifespan Context (The Modern Startup/Shutdown Pattern)
@asynccontextmanager
async def lifespan(app: FastAPI):
    # --- Startup Hook ---
    logger.info("Startup: Connecting to Database...")
    try:
        # Fail Fast: Check DB connection immediately upon startup.
        # If DB is down, the pod should crash/restart rather than serving 500s.
        async with engine.begin() as conn:
            await conn.run_sync(lambda x: x)
        logger.info("Startup: Database connection successful.")
    except Exception as e:
        logger.error(f"Startup: Database connection FAILED. {e}")
        raise e  # Stop the app startup
    
    yield
    
    # --- Shutdown Hook ---
    logger.info("Shutdown: Closing Database Connections...")
    await engine.dispose()

# 3. Initialize FastAPI
app = FastAPI(
    title=settings.PROJECT_NAME,
    version="1.0.0",
    lifespan=lifespan,
    # Security: Hide Swagger UI in Production to prevent schema reconnaissance
    docs_url="/docs" if settings.DEBUG else None,
    redoc_url=None,
    openapi_url="/openapi.json" if settings.DEBUG else None,
)

# 4. Global Middlewares
# Order matters! CORS usually comes first.

# 4.1 CORS (Cross-Origin Resource Sharing)
# Critical for allowing frontend apps (React/Vue) to communicate with the API.
if settings.BACKEND_CORS_ORIGINS:
    app.add_middleware(
        CORSMiddleware,
        allow_origins=[str(origin) for origin in settings.BACKEND_CORS_ORIGINS],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

# 4.2 GZip Compression
# Reduces payload size for large lists, improving network performance.
app.add_middleware(GZipMiddleware, minimum_size=1000)

# 5. Exception Handlers
# Register the standardized JSON error responses (400, 404, 500)
setup_exception_handlers(app)

# 6. Mount Routes
# All API endpoints are prefixed with /api (e.g., /api/auth/login)
app.include_router(api_router, prefix="/api")

# 7. System Endpoints

@app.get("/", include_in_schema=False)
async def root():
    """
    Redirect root access to documentation in Dev, 
    or show a generic message in Prod.
    """
    if settings.DEBUG:
        return RedirectResponse(url="/docs")
    return {"message": "API is running. Access denied to docs."}

@app.get("/health", tags=["System"])
async def health_check():
    """
    Kubernetes Liveness/Readiness Probe.
    Load Balancers (AWS ALB / Nginx) use this to check if the container is alive.
    """
    return {"status": "ok", "version": "1.0.0"}
```