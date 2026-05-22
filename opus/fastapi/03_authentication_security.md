# FastAPI Authentication & Security — Senior Level

## 1. JWT Authentication (Complete Implementation)

```python
# auth/jwt.py
from datetime import datetime, timedelta
from jose import JWTError, jwt
from passlib.context import CryptContext
from pydantic import BaseModel
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

# Config
SECRET_KEY = "your-secret-key"  # from env in production
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE = 30  # minutes
REFRESH_TOKEN_EXPIRE = 7  # days

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")


# Models
class TokenPayload(BaseModel):
    sub: str  # user_id
    exp: datetime
    type: str  # "access" or "refresh"

class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int


# Token generation
def create_access_token(user_id: str) -> str:
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE)
    payload = {"sub": user_id, "exp": expire, "type": "access"}
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def create_refresh_token(user_id: str) -> str:
    expire = datetime.utcnow() + timedelta(days=REFRESH_TOKEN_EXPIRE)
    payload = {"sub": user_id, "exp": expire, "type": "refresh"}
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def verify_token(token: str, token_type: str = "access") -> TokenPayload:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        if payload.get("type") != token_type:
            raise HTTPException(status_code=401, detail="Invalid token type")
        return TokenPayload(**payload)
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid or expired token")


# Password hashing
def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)


# Dependency: get current user from token
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
):
    payload = verify_token(token, "access")
    user = await db.get(User, payload.sub)
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    if not user.is_active:
        raise HTTPException(status_code=401, detail="User is deactivated")
    return user


# Optional auth (user may or may not be logged in)
async def get_current_user_optional(
    token: str | None = Depends(OAuth2PasswordBearer(tokenUrl="login", auto_error=False)),
    db: AsyncSession = Depends(get_db),
):
    if not token:
        return None
    try:
        payload = verify_token(token)
        return await db.get(User, payload.sub)
    except HTTPException:
        return None
```

### Auth Endpoints
```python
# auth/router.py
from fastapi import APIRouter, Depends, HTTPException
from fastapi.security import OAuth2PasswordRequestForm

router = APIRouter(prefix="/auth", tags=["Authentication"])

@router.post("/register", response_model=UserResponse, status_code=201)
async def register(data: UserCreate, db = Depends(get_db)):
    # Check duplicate email
    existing = await db.execute(select(User).where(User.email == data.email))
    if existing.scalar_one_or_none():
        raise HTTPException(400, "Email already registered")
    
    user = User(
        email=data.email,
        password_hash=hash_password(data.password),
        full_name=data.full_name,
    )
    db.add(user)
    await db.commit()
    return user


@router.post("/login", response_model=TokenResponse)
async def login(form: OAuth2PasswordRequestForm = Depends(), db = Depends(get_db)):
    user = await db.execute(select(User).where(User.email == form.username))
    user = user.scalar_one_or_none()
    
    if not user or not verify_password(form.password, user.password_hash):
        raise HTTPException(401, "Invalid email or password")
    
    return TokenResponse(
        access_token=create_access_token(str(user.id)),
        refresh_token=create_refresh_token(str(user.id)),
        expires_in=ACCESS_TOKEN_EXPIRE * 60,
    )


@router.post("/refresh", response_model=TokenResponse)
async def refresh_token(refresh_token: str, db = Depends(get_db)):
    payload = verify_token(refresh_token, "refresh")
    user = await db.get(User, payload.sub)
    if not user:
        raise HTTPException(401, "User not found")
    
    return TokenResponse(
        access_token=create_access_token(str(user.id)),
        refresh_token=create_refresh_token(str(user.id)),
        expires_in=ACCESS_TOKEN_EXPIRE * 60,
    )


@router.get("/me", response_model=UserResponse)
async def get_me(user = Depends(get_current_user)):
    return user
```

---

## 2. Role-Based Access Control (RBAC)

```python
from enum import Enum
from functools import wraps

class Role(str, Enum):
    USER = "user"
    ADMIN = "admin"
    MODERATOR = "moderator"

# Dependency factory for role checking
def require_roles(*roles: Role):
    async def role_checker(user = Depends(get_current_user)):
        if user.role not in roles:
            raise HTTPException(
                status_code=403,
                detail=f"Requires role: {', '.join(r.value for r in roles)}"
            )
        return user
    return role_checker

# Usage
@router.delete("/users/{user_id}")
async def delete_user(
    user_id: UUID,
    admin = Depends(require_roles(Role.ADMIN)),
    db = Depends(get_db),
):
    ...

@router.put("/posts/{post_id}/approve")
async def approve_post(
    post_id: UUID,
    user = Depends(require_roles(Role.ADMIN, Role.MODERATOR)),
):
    ...
```

---

## 3. API Key Authentication

```python
from fastapi import Security
from fastapi.security import APIKeyHeader, APIKeyQuery

# API key from header or query param
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)
api_key_query = APIKeyQuery(name="api_key", auto_error=False)

async def get_api_key(
    header_key: str | None = Security(api_key_header),
    query_key: str | None = Security(api_key_query),
    db = Depends(get_db),
):
    api_key = header_key or query_key
    if not api_key:
        raise HTTPException(401, "API key required")
    
    # Validate against database
    key_record = await db.execute(
        select(APIKey).where(APIKey.key == api_key, APIKey.is_active == True)
    )
    key_record = key_record.scalar_one_or_none()
    
    if not key_record:
        raise HTTPException(401, "Invalid API key")
    
    # Track usage
    key_record.last_used_at = datetime.utcnow()
    key_record.request_count += 1
    await db.commit()
    
    return key_record

# Usage
@router.get("/external/products")
async def external_products(api_key = Depends(get_api_key)):
    return await ProductService.list_for_partner(api_key.partner_id)
```

---

## 4. Rate Limiting

```python
from fastapi import Request
import time

class RateLimiter:
    """Token bucket rate limiter using Redis."""
    
    def __init__(self, requests: int = 100, window: int = 60):
        self.requests = requests
        self.window = window
    
    async def __call__(self, request: Request):
        import aioredis
        redis = request.app.state.redis
        
        # Key by user or IP
        if hasattr(request.state, 'user'):
            key = f"rate:{request.state.user.id}"
        else:
            key = f"rate:{request.client.host}"
        
        # Sliding window counter
        now = time.time()
        pipe = redis.pipeline()
        pipe.zremrangebyscore(key, 0, now - self.window)
        pipe.zcard(key)
        pipe.zadd(key, {str(now): now})
        pipe.expire(key, self.window)
        results = await pipe.execute()
        
        current_count = results[1]
        
        if current_count >= self.requests:
            raise HTTPException(
                status_code=429,
                detail="Rate limit exceeded",
                headers={"Retry-After": str(self.window)}
            )

# Apply as dependency
rate_limit = RateLimiter(requests=100, window=60)

@router.post("/orders/", dependencies=[Depends(rate_limit)])
async def create_order(...):
    ...

# Per-endpoint rate limits
strict_rate_limit = RateLimiter(requests=5, window=60)

@router.post("/auth/login", dependencies=[Depends(strict_rate_limit)])
async def login(...):
    ...
```

---

## 5. CORS & Security Headers

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from starlette.middleware.httpsredirect import HTTPSRedirectMiddleware

app = FastAPI()

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://yourdomain.com", "https://app.yourdomain.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    expose_headers=["X-Request-ID", "X-RateLimit-Remaining"],
)

# HTTPS redirect (production)
if settings.environment == "production":
    app.add_middleware(HTTPSRedirectMiddleware)

# Security headers middleware
@app.middleware("http")
async def security_headers(request: Request, call_next):
    response = await call_next(request)
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    response.headers["Content-Security-Policy"] = "default-src 'self'"
    return response
```

---

## 6. OAuth2 (Google/GitHub Login)

```python
from authlib.integrations.starlette_client import OAuth

oauth = OAuth()
oauth.register(
    name='google',
    client_id=settings.google_client_id,
    client_secret=settings.google_client_secret,
    server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
    client_kwargs={'scope': 'openid email profile'},
)

@router.get("/auth/google")
async def google_login(request: Request):
    redirect_uri = request.url_for('google_callback')
    return await oauth.google.authorize_redirect(request, redirect_uri)

@router.get("/auth/google/callback")
async def google_callback(request: Request, db = Depends(get_db)):
    token = await oauth.google.authorize_access_token(request)
    user_info = token.get('userinfo')
    
    # Find or create user
    user = await db.execute(select(User).where(User.email == user_info['email']))
    user = user.scalar_one_or_none()
    
    if not user:
        user = User(
            email=user_info['email'],
            full_name=user_info['name'],
            avatar_url=user_info.get('picture'),
            oauth_provider='google',
        )
        db.add(user)
        await db.commit()
    
    # Generate JWT
    access_token = create_access_token(str(user.id))
    return RedirectResponse(f"https://yourapp.com/auth?token={access_token}")
```

---

## 7. Interview Questions

### Q: How do you secure a FastAPI application?
```
1. Authentication: JWT with short-lived access + long-lived refresh tokens
2. Authorization: Role-based with dependency injection
3. Input validation: Pydantic auto-validates everything
4. Rate limiting: Redis-based sliding window per user/IP
5. CORS: Whitelist specific origins (never allow_all in prod)
6. HTTPS: HTTPSRedirectMiddleware + HSTS header
7. Security headers: X-Frame-Options, CSP, nosniff
8. Secrets: Environment variables + Secrets Manager (never in code)
9. SQL injection: SQLAlchemy parameterized queries (never raw strings)
10. Dependency updates: pip-audit regularly
```

### Q: JWT vs Session cookies — when to use which?
```
JWT:
  + Stateless (no server-side session store)
  + Works across domains (mobile apps, SPAs)
  + Scalable (any server can verify)
  - Can't be revoked until expiry (use short TTL + refresh)
  - Larger request size (token in every header)
  
  Best for: APIs, mobile apps, microservices, SPAs

Session cookies:
  + Can be revoked instantly (delete from store)
  + Smaller request size (just session ID)
  + HttpOnly + Secure = XSS-proof
  - Requires shared session store (Redis) for scaling
  - CSRF vulnerability (need CSRF tokens)
  
  Best for: traditional web apps, server-rendered pages

For FastAPI APIs: use JWT (stateless, scalable, frontend-friendly)
```
