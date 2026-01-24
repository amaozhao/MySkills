# Pattern: Authentication & Security

This pattern implements a flexible security layer. It supports two strategies:
1.  **Local Strategy**: You issue and verify your own JWTs (Self-Hosted).
2.  **SaaS Strategy**: You verify tokens issued by providers like **Clerk, Supabase, or Auth0**.

## 1. Configuration (`app/core/config.py`)

Add these settings to support both local and external providers.

```python
from typing import List
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # ... existing DB settings ...

    # --- Security Settings ---
    # 1. Secret Key: Used for HS256 (Local or Supabase)
    #    For Clerk/Auth0 (RS256), this might be the Public Key (PEM format).
    SECRET_KEY: str 
    
    # 2. Algorithm: "HS256" for Local/Supabase, "RS256" for Clerk/Auth0
    ALGORITHM: str = "HS256"
    
    # 3. Token Expiry (Only for Local issuance)
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    
    # 4. Audience (Optional: Required by some providers like Auth0)
    AUTH_AUDIENCE: str | None = None
```

## 2. Shared Schemas (`app/schemas/token.py`)

Define the structure of the Token payload to ensure type safety.

```python
from pydantic import BaseModel

class Token(BaseModel):
    """Response schema for login endpoints."""
    access_token: str
    token_type: str

class TokenPayload(BaseModel):
    """
    The data inside the JWT.
    Standard claims: 'sub' (subject), 'exp' (expiration).
    SaaS providers often add 'email', 'role', etc.
    """
    sub: str | int | None = None
    exp: int | None = None
    email: str | None = None  # Common in Supabase/Clerk tokens
```

## 3. Core Logic (`app/core/security.py`)

This file handles the low-level cryptography.

```python
from datetime import datetime, timedelta
from typing import Any, Union
from jose import jwt
from passlib.context import CryptContext
from app.core.config import settings

# --- Password Hashing (For Local Strategy) ---
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)

# --- JWT Generation (For Local Strategy) ---
def create_access_token(subject: Union[str, Any]) -> str:
    expire = datetime.utcnow() + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode = {"exp": expire, "sub": str(subject)}
    
    encoded_jwt = jwt.encode(
        to_encode, 
        settings.SECRET_KEY, 
        algorithm=settings.ALGORITHM
    )
    return encoded_jwt
```

## 4. Auth Dependencies (`app/api/deps.py`)

This is where the magic happens. We create a dependency that can validate **either** local tokens **or** SaaS tokens.

### Strategy A: Local JWT (Self-Hosted)
*   **Algorithm**: HS256
*   **Key**: Your arbitrary string.

### Strategy B: SaaS (Supabase / Clerk)
*   **Supabase**: Uses `HS256`. Set `SECRET_KEY` to your Supabase "JWT Secret" (from Project Settings > API).
*   **Clerk / Auth0**: Uses `RS256`. Set `SECRET_KEY` to the **PEM Public Key** provided by Clerk (from API Keys > Advanced > PEM Public Key). Set `ALGORITHM="RS256"`.

```python
from typing import Annotated, Optional
from fastapi import Depends, status, Request
from fastapi.security import OAuth2PasswordBearer
from jose import jwt, JWTError
from sqlalchemy.ext.asyncio import AsyncSession

from app.core import config, security
from app.core.database import get_db
from app.core.exceptions import BusinessException
from app.schemas.token import TokenPayload
from app.services import user as user_service 
# assuming you might sync SaaS users to local DB, otherwise optional.

# The URL is only used for Swagger UI. 
# For SaaS, you might point this to the provider's login page, 
# or keep it relative if you proxy auth.
reusable_oauth2 = OAuth2PasswordBearer(
    tokenUrl=f"/api/v1/auth/login",
    auto_error=True
)

async def get_current_user_token(
    token: Annotated[str, Depends(reusable_oauth2)]
) -> TokenPayload:
    """
    Pure Token Validation Layer.
    Verifies signature, expiration, and audience (if configured).
    Works for both Local and SaaS (Clerk/Supabase).
    """
    try:
        # 1. Decode & Verify Signature
        # For RS256 (Clerk), config.settings.SECRET_KEY must be the PEM Public Key.
        # For HS256 (Local/Supabase), it's the raw secret string.
        payload = jwt.decode(
            token,
            config.settings.SECRET_KEY,
            algorithms=[config.settings.ALGORITHM],
            audience=config.settings.AUTH_AUDIENCE, # Optional: Validate 'aud' claim
            options={"verify_at_hash": False}       # Clerk sometimes adds 'at_hash'
        )
        token_data = TokenPayload(**payload)
        
        if token_data.sub is None:
            raise ValueError("Token missing 'sub' claim")
            
        return token_data

    except (JWTError, ValueError) as e:
        # Log the specific error for debugging SaaS integrations
        print(f"Auth Error: {e}") 
        raise BusinessException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            code="INVALID_TOKEN",
            message="Could not validate credentials"
        )

# --- High Level Dependency ---

async def get_current_user(
    token_data: Annotated[TokenPayload, Depends(get_current_user_token)],
    db: Annotated[AsyncSession, Depends(get_db)]
):
    """
    Retrieves the local User representation.
    
    Hybrid Approach:
    1. Validates the Token (SaaS or Local).
    2. Checks if a corresponding user exists in YOUR database.
       (Useful if you need to attach local data like 'orders' to a Clerk user).
    """
    # Using 'sub' as the primary key reference. 
    # For Clerk, 'sub' is a string "user_2x...", for local it might be "1".
    user = await user_service.get_by_id_or_sub(db, token_data.sub)
    
    if not user:
        # OPTIONAL: Just-In-Time (JIT) Provisioning
        # If using Clerk, you might want to auto-create the user locally here.
        # user = await user_service.create_from_saas(db, token_data)
        
        raise BusinessException(
            status_code=status.HTTP_404_NOT_FOUND,
            code="USER_NOT_FOUND",
            message="User not found"
        )
        
    if not user.is_active:
        raise BusinessException(
            status_code=status.HTTP_403_FORBIDDEN,
            code="INACTIVE_USER",
            message="User is inactive"
        )
        
    return user

# Type alias for Route injection
CurrentUser = Annotated[Any, Depends(get_current_user)]
```

## 5. Implementation Guide

### Case 1: Local Dev / Self-Hosted
*   **.env**:
    ```bash
    SECRET_KEY="your-random-string"
    ALGORITHM="HS256"
    ```
*   **Flow**: Endpoint `/auth/login` generates tokens using `security.create_access_token`.

### Case 2: Supabase
*   **.env**:
    ```bash
    SECRET_KEY="<YOUR_SUPABASE_JWT_SECRET>"
    ALGORITHM="HS256"
    ```
*   **Flow**: Frontend logs in via Supabase SDK -> sends JWT to FastAPI -> `get_current_user_token` validates it using Supabase's secret.

### Case 3: Clerk / Auth0
*   **.env**:
    ```bash
    # Get this from Clerk Dashboard -> API Keys -> Advanced -> PEM Public Key
    SECRET_KEY="-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhki...\n-----END PUBLIC KEY-----"
    ALGORITHM="RS256"
    ```
*   **Flow**: Frontend logs in via Clerk SDK -> sends JWT to FastAPI -> `get_current_user_token` uses the PEM key to verify the RS256 signature.