# FastAPI Pydantic & Validation — Senior Level

## 1. Pydantic v2 Model Patterns

### Basic Models with Validation
```python
from pydantic import BaseModel, Field, field_validator, model_validator
from pydantic import EmailStr, HttpUrl, constr, conint
from datetime import datetime, date
from uuid import UUID
from enum import Enum
from typing import Annotated

# Enums for choices
class OrderStatus(str, Enum):
    PENDING = "pending"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

# Annotated types (reusable validation)
PositiveFloat = Annotated[float, Field(gt=0)]
NonEmptyStr = Annotated[str, Field(min_length=1, max_length=255)]

class ProductCreate(BaseModel):
    name: NonEmptyStr
    description: str = ""
    price: PositiveFloat
    sale_price: PositiveFloat | None = None
    sku: constr(pattern=r'^[A-Z]{2}\d{6}$')  # e.g., AB123456
    category_id: UUID
    tags: list[str] = Field(default_factory=list, max_length=10)
    metadata: dict[str, str] = {}

    # Field-level validator
    @field_validator('tags')
    @classmethod
    def tags_must_be_lowercase(cls, v):
        return [tag.lower().strip() for tag in v]

    # Cross-field validation
    @model_validator(mode='after')
    def sale_price_less_than_price(self):
        if self.sale_price and self.sale_price >= self.price:
            raise ValueError('sale_price must be less than price')
        return self
```

### Nested Models
```python
class Address(BaseModel):
    street: str
    city: str
    state: str
    zip_code: constr(pattern=r'^\d{6}$')
    country: str = "IN"

class UserProfile(BaseModel):
    bio: str = ""
    avatar_url: HttpUrl | None = None
    social_links: dict[str, HttpUrl] = {}

class UserCreate(BaseModel):
    email: EmailStr
    password: constr(min_length=8)
    full_name: NonEmptyStr
    phone: constr(pattern=r'^\+91\d{10}$')
    address: Address
    profile: UserProfile = UserProfile()
    
    @field_validator('password')
    @classmethod
    def password_strength(cls, v):
        if not any(c.isupper() for c in v):
            raise ValueError('must contain uppercase letter')
        if not any(c.isdigit() for c in v):
            raise ValueError('must contain digit')
        return v

class UserResponse(BaseModel):
    id: UUID
    email: str
    full_name: str
    address: Address
    profile: UserProfile
    created_at: datetime
    
    model_config = {"from_attributes": True}
```

---

## 2. Request/Response Model Patterns

### Separate Input/Output Models
```python
# Base (shared fields)
class OrderBase(BaseModel):
    shipping_address: Address
    notes: str = ""

# Create (input — what client sends)
class OrderCreate(OrderBase):
    items: list[OrderItemCreate]

class OrderItemCreate(BaseModel):
    product_id: UUID
    quantity: conint(ge=1, le=100)

# Response (output — what client receives)
class OrderResponse(OrderBase):
    id: UUID
    user_id: UUID
    status: OrderStatus
    total: float
    items: list[OrderItemResponse]
    created_at: datetime
    shipped_at: datetime | None = None
    
    model_config = {"from_attributes": True}

class OrderItemResponse(BaseModel):
    product_id: UUID
    product_name: str
    quantity: int
    unit_price: float
    subtotal: float
    
    model_config = {"from_attributes": True}

# Update (partial — all fields optional)
class OrderUpdate(BaseModel):
    shipping_address: Address | None = None
    notes: str | None = None
    status: OrderStatus | None = None
```

### Paginated Response Pattern
```python
from typing import Generic, TypeVar
from pydantic import BaseModel, computed_field
import math

T = TypeVar('T')

class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    page_size: int
    
    @computed_field
    @property
    def total_pages(self) -> int:
        return math.ceil(self.total / self.page_size)
    
    @computed_field
    @property
    def has_next(self) -> bool:
        return self.page < self.total_pages
    
    @computed_field
    @property
    def has_prev(self) -> bool:
        return self.page > 1

# Usage in endpoint
@router.get("/products/", response_model=PaginatedResponse[ProductResponse])
async def list_products(page: int = 1, page_size: int = 20, db = Depends(get_db)):
    total = await db.scalar(select(func.count(Product.id)))
    products = await db.execute(
        select(Product).offset((page - 1) * page_size).limit(page_size)
    )
    return PaginatedResponse(
        items=products.scalars().all(),
        total=total,
        page=page,
        page_size=page_size,
    )
```

---

## 3. Advanced Validation

### Custom Validators
```python
from pydantic import field_validator, model_validator
from datetime import date

class EventCreate(BaseModel):
    name: str
    start_date: date
    end_date: date
    max_attendees: int = Field(ge=1)
    ticket_price: float = Field(ge=0)
    early_bird_price: float | None = None
    registration_deadline: date
    
    @field_validator('start_date')
    @classmethod
    def start_date_must_be_future(cls, v):
        if v <= date.today():
            raise ValueError('start_date must be in the future')
        return v
    
    @model_validator(mode='after')
    def validate_dates(self):
        if self.end_date < self.start_date:
            raise ValueError('end_date must be after start_date')
        if self.registration_deadline > self.start_date:
            raise ValueError('registration_deadline must be before start_date')
        if self.early_bird_price and self.early_bird_price >= self.ticket_price:
            raise ValueError('early_bird_price must be less than ticket_price')
        return self


# Validator with database check (using dependency)
class ProductCreate(BaseModel):
    name: str
    sku: str
    
    # Note: Can't do DB validation in Pydantic directly
    # Do it in the endpoint or service layer instead

# In endpoint:
@router.post("/products/")
async def create_product(product: ProductCreate, db = Depends(get_db)):
    # DB-level validation
    existing = await db.execute(select(Product).where(Product.sku == product.sku))
    if existing.scalar_one_or_none():
        raise HTTPException(400, detail="SKU already exists")
    ...
```

### Settings Validation (pydantic-settings)
```python
from pydantic_settings import BaseSettings
from pydantic import Field, PostgresDsn, RedisDsn
from functools import lru_cache

class Settings(BaseSettings):
    # App
    app_name: str = "MyApp"
    debug: bool = False
    environment: str = "production"
    
    # Database
    database_url: PostgresDsn
    db_pool_size: int = Field(10, ge=1, le=100)
    db_max_overflow: int = 20
    
    # Redis
    redis_url: RedisDsn
    
    # Auth
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    jwt_expiry_minutes: int = 30
    
    # AWS
    aws_region: str = "ap-south-1"
    aws_s3_bucket: str
    
    # External APIs
    payment_api_key: str
    payment_api_url: str = "https://api.razorpay.com"
    
    model_config = {
        "env_file": ".env",
        "env_file_encoding": "utf-8",
        "case_sensitive": False,
    }

@lru_cache()
def get_settings() -> Settings:
    return Settings()

# Usage as dependency
@router.get("/config")
async def get_config(settings: Settings = Depends(get_settings)):
    return {"app": settings.app_name, "env": settings.environment}
```

---

## 4. Serialization Tricks

### model_dump() Options
```python
user = UserResponse(id=uuid4(), email="test@example.com", ...)

# Include only specific fields
user.model_dump(include={"id", "email"})
# {"id": "...", "email": "test@example.com"}

# Exclude fields
user.model_dump(exclude={"password_hash", "internal_notes"})

# Exclude None values (compact JSON)
user.model_dump(exclude_none=True)

# Exclude default values
user.model_dump(exclude_defaults=True)

# JSON-compatible (UUID → str, datetime → ISO string)
user.model_dump(mode="json")
```

### Custom JSON Encoders
```python
from pydantic import BaseModel, ConfigDict
from decimal import Decimal
from datetime import datetime

class OrderResponse(BaseModel):
    id: UUID
    total: Decimal
    created_at: datetime
    
    model_config = ConfigDict(
        from_attributes=True,
        json_encoders={
            Decimal: lambda v: float(v),
            datetime: lambda v: v.strftime("%Y-%m-%d %H:%M:%S"),
        }
    )
```

---

## 5. Query Parameter Models (Pydantic for Query Params)

```python
from fastapi import Depends, Query
from pydantic import BaseModel, Field
from enum import Enum

class SortOrder(str, Enum):
    ASC = "asc"
    DESC = "desc"

class ProductFilter(BaseModel):
    """Query parameters for product listing."""
    search: str | None = Field(None, min_length=2)
    category_id: UUID | None = None
    min_price: float | None = Field(None, ge=0)
    max_price: float | None = Field(None, ge=0)
    in_stock: bool | None = None
    sort_by: str = Field("created_at", pattern="^(name|price|created_at)$")
    sort_order: SortOrder = SortOrder.DESC
    page: int = Field(1, ge=1)
    page_size: int = Field(20, ge=1, le=100)
    
    @model_validator(mode='after')
    def validate_price_range(self):
        if self.min_price and self.max_price and self.min_price > self.max_price:
            raise ValueError('min_price cannot exceed max_price')
        return self


@router.get("/products/")
async def list_products(filters: ProductFilter = Depends()):
    query = select(Product)
    
    if filters.search:
        query = query.where(Product.name.ilike(f"%{filters.search}%"))
    if filters.category_id:
        query = query.where(Product.category_id == filters.category_id)
    if filters.min_price:
        query = query.where(Product.price >= filters.min_price)
    if filters.max_price:
        query = query.where(Product.price <= filters.max_price)
    if filters.in_stock is not None:
        query = query.where(Product.stock > 0 if filters.in_stock else Product.stock == 0)
    
    # Sort
    sort_col = getattr(Product, filters.sort_by)
    if filters.sort_order == SortOrder.DESC:
        query = query.order_by(sort_col.desc())
    else:
        query = query.order_by(sort_col.asc())
    
    # Paginate
    query = query.offset((filters.page - 1) * filters.page_size).limit(filters.page_size)
    
    result = await db.execute(query)
    return result.scalars().all()
```

---

## 6. Interview Questions

### Q: Why Pydantic over marshmallow/attrs?
```
Pydantic v2:
  - Written in Rust (pydantic-core) — 5-50x faster than marshmallow
  - Native Python type hints (no separate schema definition)
  - IDE support (autocomplete, type checking)
  - FastAPI integration (auto-validation, auto-docs)
  - JSON Schema generation (OpenAPI compatible)
  - Settings management (env vars, .env files)

Marshmallow:
  - Separate schema class (duplicates model definition)
  - Slower (pure Python)
  - More flexible for complex serialization
  - Better for Django integration (DRF-marshmallow)
```

### Q: How do you handle partial updates (PATCH) with Pydantic?
```python
class ProductUpdate(BaseModel):
    name: str | None = None
    price: float | None = None
    description: str | None = None

@router.patch("/products/{id}")
async def update_product(id: UUID, data: ProductUpdate, db = Depends(get_db)):
    product = await db.get(Product, id)
    
    # Only update fields that were explicitly sent
    update_data = data.model_dump(exclude_unset=True)
    
    for field, value in update_data.items():
        setattr(product, field, value)
    
    await db.commit()
    return product

# exclude_unset=True: only includes fields the client actually sent
# vs exclude_none=True: would exclude intentional null values
```
