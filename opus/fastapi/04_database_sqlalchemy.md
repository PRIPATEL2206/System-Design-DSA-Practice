# FastAPI + SQLAlchemy Async Database — Senior Level

## 1. Async SQLAlchemy 2.0 Setup

```python
# database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase
from config import settings

# Async engine (uses asyncpg for PostgreSQL)
engine = create_async_engine(
    settings.database_url,  # postgresql+asyncpg://user:pass@host/db
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,      # verify connection before use
    pool_recycle=300,         # recycle connections after 5 min
    echo=settings.debug,     # log SQL in debug mode
)

# Session factory
async_session = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,  # don't expire objects after commit
)

# Base model class
class Base(DeclarativeBase):
    pass

# Dependency
async def get_db():
    async with async_session() as session:
        try:
            yield session
        finally:
            await session.close()
```

---

## 2. Model Definition (SQLAlchemy 2.0 Style)

```python
# models.py
import uuid
from datetime import datetime
from sqlalchemy import String, Integer, ForeignKey, DateTime, Boolean, Numeric, Text, Index
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID, JSONB, ARRAY
from database import Base


class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)


class User(TimestampMixin, Base):
    __tablename__ = "users"
    
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    full_name: Mapped[str] = mapped_column(String(255))
    role: Mapped[str] = mapped_column(String(20), default="user")
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    
    # Relationships
    orders: Mapped[list["Order"]] = relationship(back_populates="user", lazy="selectin")
    
    def __repr__(self):
        return f"<User {self.email}>"


class Product(TimestampMixin, Base):
    __tablename__ = "products"
    
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(255), index=True)
    slug: Mapped[str] = mapped_column(String(255), unique=True)
    description: Mapped[str] = mapped_column(Text, default="")
    price: Mapped[float] = mapped_column(Numeric(10, 2))
    stock: Mapped[int] = mapped_column(Integer, default=0)
    status: Mapped[str] = mapped_column(String(20), default="draft")
    metadata: Mapped[dict] = mapped_column(JSONB, default=dict)
    tags: Mapped[list] = mapped_column(ARRAY(String), default=list)
    
    category_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("categories.id"))
    category: Mapped["Category"] = relationship(back_populates="products")
    
    __table_args__ = (
        Index("ix_product_status_created", "status", "created_at"),
        Index("ix_product_category_status", "category_id", "status"),
    )


class Order(TimestampMixin, Base):
    __tablename__ = "orders"
    
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"))
    status: Mapped[str] = mapped_column(String(20), default="pending")
    total: Mapped[float] = mapped_column(Numeric(10, 2), default=0)
    
    user: Mapped["User"] = relationship(back_populates="orders")
    items: Mapped[list["OrderItem"]] = relationship(back_populates="order", lazy="selectin")
```

---

## 3. CRUD Operations (Async)

```python
# repositories/product_repo.py
from sqlalchemy import select, func, update, delete
from sqlalchemy.ext.asyncio import AsyncSession
from models import Product
from uuid import UUID

class ProductRepository:
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def get_by_id(self, product_id: UUID) -> Product | None:
        return await self.db.get(Product, product_id)
    
    async def get_by_slug(self, slug: str) -> Product | None:
        result = await self.db.execute(
            select(Product).where(Product.slug == slug)
        )
        return result.scalar_one_or_none()
    
    async def list(
        self,
        skip: int = 0,
        limit: int = 20,
        status: str | None = None,
        category_id: UUID | None = None,
        search: str | None = None,
    ) -> tuple[list[Product], int]:
        query = select(Product)
        count_query = select(func.count(Product.id))
        
        # Filters
        if status:
            query = query.where(Product.status == status)
            count_query = count_query.where(Product.status == status)
        if category_id:
            query = query.where(Product.category_id == category_id)
            count_query = count_query.where(Product.category_id == category_id)
        if search:
            query = query.where(Product.name.ilike(f"%{search}%"))
            count_query = count_query.where(Product.name.ilike(f"%{search}%"))
        
        # Pagination
        query = query.offset(skip).limit(limit).order_by(Product.created_at.desc())
        
        # Execute both queries
        total = await self.db.scalar(count_query)
        result = await self.db.execute(query)
        products = result.scalars().all()
        
        return products, total
    
    async def create(self, data: dict) -> Product:
        product = Product(**data)
        self.db.add(product)
        await self.db.commit()
        await self.db.refresh(product)
        return product
    
    async def update(self, product_id: UUID, data: dict) -> Product | None:
        product = await self.get_by_id(product_id)
        if not product:
            return None
        
        for key, value in data.items():
            setattr(product, key, value)
        
        await self.db.commit()
        await self.db.refresh(product)
        return product
    
    async def bulk_update_status(self, product_ids: list[UUID], status: str) -> int:
        result = await self.db.execute(
            update(Product)
            .where(Product.id.in_(product_ids))
            .values(status=status)
        )
        await self.db.commit()
        return result.rowcount
    
    async def delete(self, product_id: UUID) -> bool:
        result = await self.db.execute(
            delete(Product).where(Product.id == product_id)
        )
        await self.db.commit()
        return result.rowcount > 0
```

---

## 4. Transactions & Concurrency

```python
from sqlalchemy import select, update
from sqlalchemy.orm import selectinload

class OrderService:
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def create_order(self, user_id: UUID, items: list[dict]) -> Order:
        """Create order with atomic stock deduction."""
        async with self.db.begin():  # auto-commit or rollback
            order = Order(user_id=user_id, status="pending", total=0)
            self.db.add(order)
            await self.db.flush()  # get order.id without committing
            
            total = 0
            for item in items:
                # Lock product row (SELECT ... FOR UPDATE)
                result = await self.db.execute(
                    select(Product)
                    .where(Product.id == item["product_id"])
                    .with_for_update()
                )
                product = result.scalar_one()
                
                if product.stock < item["quantity"]:
                    raise ValueError(f"Insufficient stock: {product.name}")
                
                # Deduct stock
                product.stock -= item["quantity"]
                
                # Create order item
                order_item = OrderItem(
                    order_id=order.id,
                    product_id=product.id,
                    quantity=item["quantity"],
                    unit_price=float(product.price),
                )
                self.db.add(order_item)
                total += float(product.price) * item["quantity"]
            
            order.total = total
            # Transaction commits at end of `async with self.db.begin()`
        
        await self.db.refresh(order, ["items"])
        return order


# Optimistic locking alternative
class ProductService:
    async def update_stock_optimistic(self, product_id: UUID, quantity: int) -> bool:
        """Update stock without locking — retry on conflict."""
        for attempt in range(3):
            product = await self.db.get(Product, product_id)
            if product.stock < quantity:
                raise ValueError("Out of stock")
            
            result = await self.db.execute(
                update(Product)
                .where(Product.id == product_id, Product.stock == product.stock)
                .values(stock=Product.stock - quantity)
            )
            
            if result.rowcount == 1:
                await self.db.commit()
                return True
            
            await self.db.rollback()
        
        raise ValueError("Concurrent modification — try again")
```

---

## 5. Alembic Migrations

```bash
# Setup
pip install alembic
alembic init alembic
```

```python
# alembic/env.py (async version)
from logging.config import fileConfig
from sqlalchemy.ext.asyncio import create_async_engine
from alembic import context
from database import Base
from config import settings

config = context.config
config.set_main_option("sqlalchemy.url", settings.database_url)

target_metadata = Base.metadata

def run_migrations_online():
    connectable = create_async_engine(settings.database_url)
    
    async def do_migrations():
        async with connectable.connect() as connection:
            await connection.run_sync(do_run_migrations)
    
    import asyncio
    asyncio.run(do_migrations())

def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()
```

```bash
# Common commands
alembic revision --autogenerate -m "add products table"
alembic upgrade head
alembic downgrade -1
alembic history
alembic current
```

---

## 6. Eager Loading (Prevent N+1)

```python
from sqlalchemy.orm import selectinload, joinedload

# selectinload: separate query (best for collections)
query = select(Order).options(
    selectinload(Order.items).selectinload(OrderItem.product)
)

# joinedload: SQL JOIN (best for single objects)
query = select(Order).options(
    joinedload(Order.user)
)

# Combined
query = (
    select(Order)
    .options(
        joinedload(Order.user),                    # FK: JOIN
        selectinload(Order.items)                  # collection: separate query
            .joinedload(OrderItem.product),        # FK within collection
    )
    .where(Order.user_id == user_id)
)
```

---

## 7. Interview Questions

### Q: SQLAlchemy 1.x vs 2.0 — what changed?
```
SQLAlchemy 2.0:
  - Native async support (AsyncSession, create_async_engine)
  - Type-annotated models (Mapped[str] instead of Column(String))
  - select() is now the primary query API (not session.query())
  - Better type checker support (mypy/pyright)
  - Cleaner session lifecycle

Migration:
  Old: session.query(User).filter(User.email == x).first()
  New: await session.execute(select(User).where(User.email == x))
       result.scalar_one_or_none()
```

### Q: How do you handle database connections in async FastAPI?
```
1. create_async_engine with pool settings (pool_size, max_overflow)
2. async_sessionmaker as factory
3. Dependency injection: yield session, close in finally
4. NEVER share sessions between requests
5. Use pool_pre_ping=True to detect stale connections
6. Set pool_recycle to prevent connection timeout from DB

For high traffic: PgBouncer in front of PostgreSQL
  FastAPI → PgBouncer (6432) → PostgreSQL (5432)
```
