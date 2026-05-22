# FastAPI Project Structure — Senior Level

## 1. Production Project Layout

```
my_app/
├── app/
│   ├── __init__.py
│   ├── main.py                 # App factory, lifespan, middleware
│   ├── config.py               # Pydantic Settings
│   ├── database.py             # Engine, session factory, Base
│   │
│   ├── models/                 # SQLAlchemy models
│   │   ├── __init__.py         # Export all models
│   │   ├── user.py
│   │   ├── product.py
│   │   ├── order.py
│   │   └── mixins.py           # TimestampMixin, SoftDeleteMixin
│   │
│   ├── schemas/                # Pydantic schemas (request/response)
│   │   ├── __init__.py
│   │   ├── user.py             # UserCreate, UserResponse, UserUpdate
│   │   ├── product.py
│   │   ├── order.py
│   │   └── common.py           # PaginatedResponse, ErrorResponse
│   │
│   ├── api/                    # Route handlers (thin layer)
│   │   ├── __init__.py
│   │   ├── deps.py             # Shared dependencies
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── router.py       # Include all v1 routers
│   │       ├── auth.py
│   │       ├── users.py
│   │       ├── products.py
│   │       └── orders.py
│   │
│   ├── services/               # Business logic
│   │   ├── __init__.py
│   │   ├── user_service.py
│   │   ├── product_service.py
│   │   ├── order_service.py
│   │   └── payment_service.py
│   │
│   ├── repositories/           # Database access layer
│   │   ├── __init__.py
│   │   ├── base.py             # BaseRepository (generic CRUD)
│   │   ├── user_repo.py
│   │   ├── product_repo.py
│   │   └── order_repo.py
│   │
│   ├── core/                   # Cross-cutting concerns
│   │   ├── __init__.py
│   │   ├── auth.py             # JWT, password hashing
│   │   ├── security.py         # Rate limiting, CORS config
│   │   ├── exceptions.py       # Custom exception classes
│   │   └── middleware.py       # Timing, logging, request ID
│   │
│   ├── workers/                # Background task workers
│   │   ├── __init__.py
│   │   ├── settings.py         # ARQ WorkerSettings
│   │   ├── email_tasks.py
│   │   └── report_tasks.py
│   │
│   └── utils/                  # Pure utility functions
│       ├── __init__.py
│       ├── pagination.py
│       ├── slugify.py
│       └── s3.py
│
├── alembic/                    # Database migrations
│   ├── versions/
│   ├── env.py
│   └── alembic.ini
│
├── tests/
│   ├── conftest.py
│   ├── factories.py
│   ├── test_auth.py
│   ├── test_products.py
│   ├── test_orders.py
│   └── integration/
│       └── test_order_flow.py
│
├── Dockerfile
├── docker-compose.yml
├── gunicorn.conf.py
├── nginx.conf
├── requirements.txt
├── requirements-dev.txt
├── .env.example
├── .github/workflows/ci.yml
└── README.md
```

---

## 2. App Factory Pattern

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import settings
from app.database import engine
from app.api.v1.router import api_router
from app.core.middleware import TimingMiddleware, RequestIDMiddleware
from app.core.exceptions import register_exception_handlers


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup/shutdown lifecycle."""
    import aioredis
    
    # Startup
    app.state.redis = await aioredis.from_url(str(settings.redis_url))
    
    yield
    
    # Shutdown
    await app.state.redis.close()
    await engine.dispose()


def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.app_name,
        version="1.0.0",
        docs_url="/docs" if settings.debug else None,
        redoc_url="/redoc" if settings.debug else None,
        lifespan=lifespan,
    )

    # Middleware (order matters: last added = first executed)
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    app.add_middleware(TimingMiddleware)
    app.add_middleware(RequestIDMiddleware)

    # Routes
    app.include_router(api_router, prefix="/api/v1")

    # Exception handlers
    register_exception_handlers(app)

    return app


app = create_app()
```

---

## 3. Service Layer Pattern

### Why: Separate business logic from HTTP handling
```
Endpoint (thin) → Service (business logic) → Repository (DB access)

Benefits:
  - Endpoints only handle HTTP concerns (validation, status codes)
  - Services are testable without HTTP
  - Repositories are swappable (PostgreSQL today, DynamoDB tomorrow)
  - Same service works for REST, WebSocket, CLI, or worker
```

### Base Repository
```python
# app/repositories/base.py
from typing import TypeVar, Generic, Type
from uuid import UUID
from sqlalchemy import select, func, delete
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import Base

ModelType = TypeVar("ModelType", bound=Base)


class BaseRepository(Generic[ModelType]):
    def __init__(self, model: Type[ModelType], db: AsyncSession):
        self.model = model
        self.db = db

    async def get(self, id: UUID) -> ModelType | None:
        return await self.db.get(self.model, id)

    async def get_multi(self, skip: int = 0, limit: int = 20) -> list[ModelType]:
        result = await self.db.execute(
            select(self.model).offset(skip).limit(limit)
        )
        return result.scalars().all()

    async def count(self) -> int:
        result = await self.db.execute(select(func.count(self.model.id)))
        return result.scalar()

    async def create(self, obj: ModelType) -> ModelType:
        self.db.add(obj)
        await self.db.commit()
        await self.db.refresh(obj)
        return obj

    async def update(self, obj: ModelType, data: dict) -> ModelType:
        for key, value in data.items():
            setattr(obj, key, value)
        await self.db.commit()
        await self.db.refresh(obj)
        return obj

    async def delete(self, id: UUID) -> bool:
        result = await self.db.execute(
            delete(self.model).where(self.model.id == id)
        )
        await self.db.commit()
        return result.rowcount > 0
```

### Concrete Repository
```python
# app/repositories/product_repo.py
from sqlalchemy import select
from app.models.product import Product
from app.repositories.base import BaseRepository


class ProductRepository(BaseRepository[Product]):
    def __init__(self, db):
        super().__init__(Product, db)

    async def get_by_slug(self, slug: str) -> Product | None:
        result = await self.db.execute(
            select(Product).where(Product.slug == slug)
        )
        return result.scalar_one_or_none()

    async def search(self, query: str, limit: int = 20) -> list[Product]:
        result = await self.db.execute(
            select(Product)
            .where(Product.name.ilike(f"%{query}%"))
            .limit(limit)
        )
        return result.scalars().all()

    async def get_by_category(self, category_id, skip=0, limit=20):
        result = await self.db.execute(
            select(Product)
            .where(Product.category_id == category_id, Product.status == "active")
            .offset(skip).limit(limit)
        )
        return result.scalars().all()
```

### Service Layer
```python
# app/services/product_service.py
from uuid import UUID
from sqlalchemy.ext.asyncio import AsyncSession
from app.repositories.product_repo import ProductRepository
from app.schemas.product import ProductCreate, ProductUpdate
from app.core.exceptions import NotFoundError, ConflictError
from app.utils.slugify import slugify


class ProductService:
    def __init__(self, db: AsyncSession):
        self.repo = ProductRepository(db)

    async def get(self, product_id: UUID):
        product = await self.repo.get(product_id)
        if not product:
            raise NotFoundError("Product", str(product_id))
        return product

    async def list(self, skip: int = 0, limit: int = 20):
        products = await self.repo.get_multi(skip=skip, limit=limit)
        total = await self.repo.count()
        return products, total

    async def create(self, data: ProductCreate):
        # Business logic: generate slug, check uniqueness
        slug = slugify(data.name)
        existing = await self.repo.get_by_slug(slug)
        if existing:
            raise ConflictError(f"Product with slug '{slug}' already exists")

        from app.models.product import Product
        product = Product(**data.model_dump(), slug=slug)
        return await self.repo.create(product)

    async def update(self, product_id: UUID, data: ProductUpdate):
        product = await self.get(product_id)
        update_data = data.model_dump(exclude_unset=True)

        if "name" in update_data:
            update_data["slug"] = slugify(update_data["name"])

        return await self.repo.update(product, update_data)

    async def delete(self, product_id: UUID):
        await self.get(product_id)  # raises NotFoundError if missing
        return await self.repo.delete(product_id)
```

### Thin Endpoint
```python
# app/api/v1/products.py
from fastapi import APIRouter, Depends, status
from uuid import UUID
from app.database import get_db
from app.services.product_service import ProductService
from app.schemas.product import ProductCreate, ProductUpdate, ProductResponse
from app.schemas.common import PaginatedResponse
from app.api.deps import get_current_user

router = APIRouter(prefix="/products", tags=["Products"])


def get_product_service(db=Depends(get_db)) -> ProductService:
    return ProductService(db)


@router.get("/", response_model=PaginatedResponse[ProductResponse])
async def list_products(
    page: int = 1,
    page_size: int = 20,
    service: ProductService = Depends(get_product_service),
):
    products, total = await service.list(skip=(page - 1) * page_size, limit=page_size)
    return PaginatedResponse(items=products, total=total, page=page, page_size=page_size)


@router.post("/", response_model=ProductResponse, status_code=status.HTTP_201_CREATED)
async def create_product(
    data: ProductCreate,
    service: ProductService = Depends(get_product_service),
    user=Depends(get_current_user),
):
    return await service.create(data)


@router.get("/{product_id}", response_model=ProductResponse)
async def get_product(
    product_id: UUID,
    service: ProductService = Depends(get_product_service),
):
    return await service.get(product_id)


@router.patch("/{product_id}", response_model=ProductResponse)
async def update_product(
    product_id: UUID,
    data: ProductUpdate,
    service: ProductService = Depends(get_product_service),
    user=Depends(get_current_user),
):
    return await service.update(product_id, data)


@router.delete("/{product_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_product(
    product_id: UUID,
    service: ProductService = Depends(get_product_service),
    user=Depends(get_current_user),
):
    await service.delete(product_id)
```

---

## 4. Configuration Management

```python
# app/config.py
from pydantic_settings import BaseSettings
from pydantic import Field, PostgresDsn, RedisDsn
from functools import lru_cache


class Settings(BaseSettings):
    # App
    app_name: str = "FastAPI App"
    debug: bool = False
    environment: str = "production"  # development | staging | production
    
    # Server
    host: str = "0.0.0.0"
    port: int = 8000
    workers: int = 4
    
    # Database
    database_url: PostgresDsn
    db_pool_size: int = Field(20, ge=1)
    db_max_overflow: int = 10
    
    # Redis
    redis_url: RedisDsn
    
    # Auth
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 7
    
    # CORS
    cors_origins: list[str] = ["http://localhost:3000"]
    
    # AWS
    aws_region: str = "ap-south-1"
    aws_s3_bucket: str = ""
    
    # External
    payment_api_key: str = ""
    sentry_dsn: str = ""

    model_config = {
        "env_file": ".env",
        "env_file_encoding": "utf-8",
        "case_sensitive": False,
    }


@lru_cache()
def get_settings() -> Settings:
    return Settings()


settings = get_settings()
```

---

## 5. Dependency Injection Patterns

```python
# app/api/deps.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.core.auth import verify_token
from app.models.user import User

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")


async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    payload = verify_token(token)
    user = await db.get(User, payload.sub)
    if not user or not user.is_active:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    return user


async def get_current_active_admin(user: User = Depends(get_current_user)) -> User:
    if user.role != "admin":
        raise HTTPException(status_code=403, detail="Admin access required")
    return user


class RateLimitDep:
    """Configurable rate limit dependency."""
    def __init__(self, requests: int, window: int):
        self.requests = requests
        self.window = window

    async def __call__(self, request):
        # Check rate limit via Redis
        ...


# Router-level dependencies
from fastapi import APIRouter

admin_router = APIRouter(
    prefix="/admin",
    tags=["Admin"],
    dependencies=[Depends(get_current_active_admin)],
)
```

---

## 6. Interview Questions

### Q: How do you structure a large FastAPI project?
```
Layered architecture:
  API Layer (routers) → Service Layer → Repository Layer → Database

Each layer has a single responsibility:
  - API: HTTP concerns (validation, status codes, response formatting)
  - Service: Business logic (rules, orchestration, external calls)
  - Repository: Data access (queries, transactions, caching)

Why not just put everything in endpoints?
  - Endpoints become 200+ lines
  - Can't reuse logic (WebSocket needs same logic as REST)
  - Hard to test (need HTTP client for business logic tests)
  - Tight coupling to framework (can't move to gRPC easily)

When is this overkill?
  - Prototypes, hackathons, < 10 endpoints
  - Just use endpoints + service functions (skip repository)
```

### Q: How does FastAPI DI compare to Django's approach?
```
Django:
  - No built-in DI
  - Uses middleware, decorators, and import-time singletons
  - @login_required decorator for auth
  - request.user populated by middleware

FastAPI:
  - First-class DI with Depends()
  - Dependencies form a DAG (resolved per-request)
  - Cacheable (same dep used twice = same instance)
  - Overridable in tests (app.dependency_overrides)
  - Type-safe (IDE understands the chain)

FastAPI DI is more explicit and testable.
Django's approach is more implicit but simpler for small apps.
```
