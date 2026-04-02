---
name: fastapi-auth
description: FastAPI JWT 认证 - 本地 JWT / Clerk / Supabase / Auth0
paths: **/*.py
---

# FastAPI JWT Authentication

## 概述

灵活的认证支持：自托管 JWT（自己签发）和 SaaS 认证提供商（Clerk/Supabase/Auth0）。

## 密码工具

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

## Token 验证

```python
# app/api/deps.py
from typing import Annotated
from fastapi import Depends, status
from fastapi.security import OAuth2PasswordBearer
from jose import jwt, JWTError
from app.core.exceptions import BusinessException
from app.schemas.token import TokenPayload

reusable_oauth2 = OAuth2PasswordBearer(tokenUrl="/api/auth/login")


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

## 提供商配置

| 提供商 | 算法 | 密钥 |
|--------|------|------|
| 本地/Supabase | HS256 | JWT secret string |
| Clerk/Auth0 | RS256 | PEM Public Key |

## 使用

```python
# 保护端点
@router.get("/me")
async def get_me(current_user: CurrentUser):
    return current_user
```
