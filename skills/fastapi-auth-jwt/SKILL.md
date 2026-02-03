---
name: fastapi-auth-jwt
description: Use when building FastAPI APIs that need authentication, either self-hosted JWT or external providers like Clerk/Supabase
---

# FastAPI JWT Authentication

## Overview
**Flexible auth supporting Local JWT (self-issued) and SaaS providers (Clerk/Supabase/Auth0) with unified token validation.**

## When to Use
- Building APIs with Bearer token authentication
- Supporting multiple auth providers
- Need JWT verification without implementing login flows
- Integrating external auth (Clerk/Supabase)

## The Pattern

```python
# app/core/security.py
from datetime import datetime, timedelta
from jose import jwt
from passlib.context import CryptContext
from app.core.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)

def create_access_token(subject: str) -> str:
    expire = datetime.utcnow() + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    return jwt.encode(
        {"exp": expire, "sub": str(subject)},
        settings.SECRET_KEY,
        algorithm=settings.ALGORITHM
    )
```

## Token Validation

```python
# app/api/deps.py
from typing import Annotated
from fastapi import Depends, status
from fastapi.security import OAuth2PasswordBearer
from jose import jwt, JWTError
from app.core.exceptions import BusinessException
from app.schemas.token import TokenPayload

reusable_oauth2 = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")

async def get_current_user_token(
    token: Annotated[str, Depends(reusable_oauth2)]
) -> TokenPayload:
    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM],
            audience=settings.AUTH_AUDIENCE,
        )
        token_data = TokenPayload(**payload)
        if not token_data.sub:
            raise ValueError("Missing 'sub' claim")
        return token_data
    except (JWTError, ValueError):
        raise BusinessException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            code="INVALID_TOKEN",
            message="Could not validate credentials"
        )
```

## Provider Configuration

| Provider | Algorithm | SECRET_KEY |
|----------|-----------|------------|
| Local/Supabase | HS256 | Your JWT secret string |
| Clerk/Auth0 | RS256 | PEM Public Key |

## Usage

```python
# Protected endpoint
@router.get("/me")
async def get_me(current_user: CurrentUser):
    return current_user
```

## The Bottom Line

**Local = issue + verify; SaaS = verify only.**
