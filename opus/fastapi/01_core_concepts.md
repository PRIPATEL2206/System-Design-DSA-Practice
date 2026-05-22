# FastAPI Core Concepts — Senior Level

## 1. Path Operations & Routing

### Basic CRUD with Type Safety
```python
from fastapi import FastAPI, HTTPException, status, Query, Path
from pydantic import BaseModel, Field
from uuid import UUID
from datetime import datetime

app = FastAPI(title="Product API", version="1.0.0")

# Request/Response models
class ProductCreate(BaseModel):
    name: str = Field(..., min_length=3, max_length=255)
    price: float = Field(..., gt=0)
    category_id: UUID
    tags: list[str] = []

class ProductResponse(BaseModel):
    id: UUID
    name: str
    price: float
    category_id: UUID
    tags: list[str]
    created_at: datetime

    model_config = {"from_attributes": True}  # Pydantic v2: read from ORM objects


# Endpoints with full type annotations
@app.post("/products/", response_model=ProductResponse, status_code=status.HTTP_201_CREATED)
async def create_product(product: ProductCreate):
    # product is already validated by Pydantic
    db_product = await ProductService.create(product)
    return db_product


@app.get("/products/", response_model=list[ProductResponse])
async def list_products(
    skip: int = Query(0, ge=0),
    limit: int = Query(20, ge=1, le=100),
    category: UUID | None = Query(None),
    search: str | None = Query(None, min_length=2),
    sort_by: str = Query("created_at", pattern="^(name|price|created_at)$"),
):
    return await ProductService.list(skip=skip, limit=limit, category=category, search=search)


@app.get("/products/{product_id}", response_model=ProductResponse)
async def get_product(product_id: UUID = Path(...)):
    product = await ProductService.get_by_id(product_id)
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    return product


@app.put("/products/{product_id}", response_model=ProductResponse)
async def update_product(product_id: UUID, product: ProductCreate):
    updated = await ProductService.update(product_id, product)
    if not updated:
        raise HTTPException(status_code=404, detail="Product not found")
    return updated


@app.delete("/products/{product_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_product(product_id: UUID):
    deleted = await ProductService.delete(product_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="Product not found")
```

### Router Organization (Like Django urls.py)
```python
# routers/products.py
from fastapi import APIRouter

router = APIRouter(prefix="/products", tags=["Products"])

@router.get("/")
async def list_products():
    ...

@router.post("/")
async def create_product():
    ...


# routers/orders.py
from fastapi import APIRouter

router = APIRouter(prefix="/orders", tags=["Orders"])

@router.get("/")
async def list_orders():
    ...


# main.py
from fastapi import FastAPI
from routers import products, orders, auth

app = FastAPI()
app.include_router(products.router, prefix="/api/v1")
app.include_router(orders.router, prefix="/api/v1")
app.include_router(auth.router, prefix="/api/v1/auth")
```

---

## 2. Dependency Injection (FastAPI's Killer Feature)

### Basic Dependencies
```python
from fastapi import Depends, Header, HTTPException

# Simple dependency
async def get_db():
    """Provide database session, close after request."""
    db = SessionLocal()
    try:
        yield db  # yield = lifespan dependency (cleanup after)
    finally:
        await db.close()


# Auth dependency
async def get_current_user(token: str = Header(..., alias="Authorization")):
    """Extract and validate JWT token."""
    if not token.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Invalid token format")
    
    payload = decode_jwt(token.removeprefix("Bearer "))
    user = await UserService.get_by_id(payload["sub"])
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    return user


# Permission dependency
def require_role(required_role: str):
    """Factory: creates a dependency that checks user role."""
    async def role_checker(user = Depends(get_current_user)):
        if user.role != required_role:
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user
    return role_checker


# Usage in endpoints
@router.get("/admin/dashboard")
async def admin_dashboard(user = Depends(require_role("admin"))):
    return {"message": f"Welcome admin {user.name}"}


@router.get("/products/")
async def list_products(
    db = Depends(get_db),
    user = Depends(get_current_user),  # requires auth
):
    return await ProductService.list(db, user.organization_id)
```

### Dependency Injection Patterns
```python
# Pattern 1: Pagination dependency (reusable across endpoints)
class PaginationParams:
    def __init__(
        self,
        page: int = Query(1, ge=1),
        page_size: int = Query(20, ge=1, le=100),
    ):
        self.skip = (page - 1) * page_size
        self.limit = page_size
        self.page = page


@router.get("/products/")
async def list_products(pagination: PaginationParams = Depends()):
    products = await db.execute(
        select(Product).offset(pagination.skip).limit(pagination.limit)
    )
    return products.scalars().all()


# Pattern 2: Service injection
class ProductService:
    def __init__(self, db: AsyncSession = Depends(get_db)):
        self.db = db
    
    async def get_all(self):
        result = await self.db.execute(select(Product))
        return result.scalars().all()
    
    async def create(self, data: ProductCreate):
        product = Product(**data.model_dump())
        self.db.add(product)
        await self.db.commit()
        return product


@router.get("/products/")
async def list_products(service: ProductService = Depends()):
    return await service.get_all()


# Pattern 3: Request-scoped cache
from functools import lru_cache

@lru_cache()
def get_settings():
    """Cached app settings (loaded once)."""
    return Settings()

@router.get("/info")
async def info(settings = Depends(get_settings)):
    return {"app_name": settings.app_name, "debug": settings.debug}
```

---

## 3. Middleware

```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
import time
import uuid

app = FastAPI()

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://yourdomain.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


# Custom timing middleware
@app.middleware("http")
async def timing_middleware(request: Request, call_next):
    request_id = str(uuid.uuid4())[:8]
    start_time = time.time()
    
    # Add request ID to state (accessible in endpoints)
    request.state.request_id = request_id
    
    response = await call_next(request)
    
    duration = time.time() - start_time
    response.headers["X-Request-ID"] = request_id
    response.headers["X-Process-Time"] = f"{duration:.4f}"
    
    # Log slow requests
    if duration > 2.0:
        logger.warning(f"Slow request: {request.method} {request.url.path} took {duration:.3f}s")
    
    return response


# Structured logging middleware
@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    logger.info("request_started", method=request.method, path=request.url.path)
    
    response = await call_next(request)
    
    logger.info(
        "request_completed",
        method=request.method,
        path=request.url.path,
        status=response.status_code,
    )
    return response
```

---

## 4. Exception Handling

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

app = FastAPI()

# Custom exception classes
class AppException(Exception):
    def __init__(self, status_code: int, detail: str, error_code: str = None):
        self.status_code = status_code
        self.detail = detail
        self.error_code = error_code

class NotFoundError(AppException):
    def __init__(self, resource: str, id: str):
        super().__init__(404, f"{resource} with id '{id}' not found", "NOT_FOUND")

class InsufficientStockError(AppException):
    def __init__(self, product_name: str, available: int):
        super().__init__(409, f"Insufficient stock for '{product_name}'. Available: {available}", "INSUFFICIENT_STOCK")


# Global exception handlers
@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "code": exc.error_code,
                "message": exc.detail,
            }
        }
    )

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=422,
        content={
            "error": {
                "code": "VALIDATION_ERROR",
                "message": "Request validation failed",
                "details": exc.errors(),
            }
        }
    )

# Usage in endpoints
@router.get("/products/{product_id}")
async def get_product(product_id: UUID):
    product = await ProductService.get_by_id(product_id)
    if not product:
        raise NotFoundError("Product", str(product_id))
    return product
```

---

## 5. Lifespan Events (Startup/Shutdown)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup and shutdown logic."""
    # --- STARTUP ---
    print("Starting up...")
    app.state.db_pool = await create_db_pool()
    app.state.redis = await aioredis.from_url("redis://localhost")
    app.state.http_client = httpx.AsyncClient()
    
    yield  # App runs here
    
    # --- SHUTDOWN ---
    print("Shutting down...")
    await app.state.db_pool.close()
    await app.state.redis.close()
    await app.state.http_client.aclose()


app = FastAPI(lifespan=lifespan)
```

---

## 6. Response Models & Serialization

```python
from pydantic import BaseModel, Field, computed_field
from datetime import datetime
from uuid import UUID

# Different models for different operations
class ProductBase(BaseModel):
    name: str = Field(..., min_length=3)
    price: float = Field(..., gt=0)
    description: str = ""

class ProductCreate(ProductBase):
    category_id: UUID

class ProductUpdate(BaseModel):
    """All fields optional for partial update."""
    name: str | None = None
    price: float | None = Field(None, gt=0)
    description: str | None = None

class ProductResponse(ProductBase):
    id: UUID
    category_id: UUID
    created_at: datetime
    
    @computed_field
    @property
    def price_display(self) -> str:
        return f"₹{self.price:,.2f}"
    
    model_config = {"from_attributes": True}

# Paginated response wrapper
class PaginatedResponse(BaseModel):
    items: list[ProductResponse]
    total: int
    page: int
    page_size: int
    total_pages: int

    @computed_field
    @property
    def has_next(self) -> bool:
        return self.page < self.total_pages


# Exclude fields from response
@router.get("/products/", response_model=list[ProductResponse])
async def list_products():
    ...

# Exclude specific fields
@router.get("/products/minimal")
async def list_minimal(db = Depends(get_db)):
    products = await ProductService.list(db)
    return [p.model_dump(include={"id", "name", "price"}) for p in products]
```

---

## 7. Background Tasks

```python
from fastapi import BackgroundTasks

# Simple background task (runs after response is sent)
async def send_email_task(email: str, subject: str, body: str):
    await email_service.send(email, subject, body)

async def log_analytics(event: str, user_id: str, metadata: dict):
    await analytics_service.track(event, user_id, metadata)


@router.post("/orders/", status_code=201)
async def create_order(
    order: OrderCreate,
    background_tasks: BackgroundTasks,
    user = Depends(get_current_user),
):
    new_order = await OrderService.create(user, order)
    
    # These run AFTER response is sent (non-blocking)
    background_tasks.add_task(send_email_task, user.email, "Order Confirmed", f"Order #{new_order.id}")
    background_tasks.add_task(log_analytics, "order_created", str(user.id), {"order_id": str(new_order.id)})
    
    return new_order
```

---

## 8. Interview Questions

### Q: What's the difference between FastAPI and Flask?
```
FastAPI:
  - Async native (async/await, ASGI)
  - Auto-validation via Pydantic + type hints
  - Auto-generated OpenAPI docs (Swagger)
  - Dependency injection built-in
  - ~10x faster than Flask

Flask:
  - Synchronous (WSGI, needs async extensions)
  - Manual validation (WTForms, marshmallow)
  - No auto-docs (needs flask-apispec)
  - No DI (manual wiring)
  - Simpler mental model, huge ecosystem

Choose FastAPI for: APIs, microservices, ML serving, async workloads
Choose Flask for: simple apps, legacy projects, SSR web apps
```

### Q: How does FastAPI's dependency injection work?
```
1. Dependencies are regular functions (sync or async)
2. FastAPI resolves the dependency tree at request time
3. `Depends()` declares that a parameter should be injected
4. Dependencies can depend on other dependencies (tree)
5. `yield` dependencies get cleanup after response (like context managers)
6. Dependencies are cached per-request (same dep called twice = same instance)

Benefits:
- Testable (override deps in tests)
- Composable (build complex auth/db/cache chains)
- Type-safe (IDE autocomplete works)
```

### Q: When should you use `async def` vs `def` in FastAPI?
```
async def: when doing I/O that has async support
  - Database queries (async SQLAlchemy, asyncpg)
  - HTTP calls (httpx, aiohttp)
  - Redis operations (aioredis)
  - File I/O (aiofiles)

def (sync): when doing CPU-bound work or using sync libraries
  - Heavy computation
  - Libraries without async support (pandas, scikit-learn)
  - FastAPI automatically runs sync functions in a thread pool

Rule: if you're calling `await` inside, use `async def`.
      If not, use `def` (FastAPI handles threading for you).
```
