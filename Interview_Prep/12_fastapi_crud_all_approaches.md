# FastAPI — All CRUD Approaches (Complete Reference)

> Every possible way to build CRUD APIs in FastAPI, from basic to production-grade.

---

## Table of Contents

1. [Approach 1: Basic Functions (No Database)](#approach-1)
2. [Approach 2: SQLAlchemy ORM (Sync)](#approach-2)
3. [Approach 3: SQLAlchemy Async (AsyncIO)](#approach-3)
4. [Approach 4: Tortoise ORM (Django-like)](#approach-4)
5. [Approach 5: SQLModel (Pydantic + SQLAlchemy)](#approach-5)
6. [Approach 6: Raw SQL with databases library](#approach-6)
7. [Approach 7: MongoDB (Motor - Async)](#approach-7)
8. [Approach 8: Repository Pattern (Clean Architecture)](#approach-8)
9. [Approach 9: Generic CRUD Class (Reusable)](#approach-9)
10. [Approach 10: Full Production API with all features](#approach-10)

---

## Approach 1: Basic In-Memory CRUD (No Database) {#approach-1}

> Simplest approach — great for prototyping and understanding routing.

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime
import uuid

app = FastAPI()

# --- Schema ---
class ItemCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    description: Optional[str] = None
    price: float = Field(..., gt=0)
    category: str

class ItemUpdate(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    price: Optional[float] = Field(None, gt=0)
    category: Optional[str] = None

class ItemResponse(BaseModel):
    id: str
    name: str
    description: Optional[str]
    price: float
    category: str
    created_at: datetime

# --- In-memory store ---
items_db: dict = {}

# --- CRUD Endpoints ---

# CREATE
@app.post("/items", response_model=ItemResponse, status_code=status.HTTP_201_CREATED)
def create_item(item: ItemCreate):
    item_id = str(uuid.uuid4())
    new_item = ItemResponse(
        id=item_id,
        **item.dict(),
        created_at=datetime.utcnow()
    )
    items_db[item_id] = new_item
    return new_item

# READ ALL (with filtering)
@app.get("/items", response_model=List[ItemResponse])
def list_items(category: Optional[str] = None, min_price: Optional[float] = None,
               skip: int = 0, limit: int = 20):
    results = list(items_db.values())
    if category:
        results = [i for i in results if i.category == category]
    if min_price:
        results = [i for i in results if i.price >= min_price]
    return results[skip:skip + limit]

# READ ONE
@app.get("/items/{item_id}", response_model=ItemResponse)
def get_item(item_id: str):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    return items_db[item_id]

# UPDATE (PATCH - partial update)
@app.patch("/items/{item_id}", response_model=ItemResponse)
def update_item(item_id: str, item_data: ItemUpdate):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    
    existing = items_db[item_id]
    update_data = item_data.dict(exclude_unset=True)
    updated = existing.copy(update=update_data)
    items_db[item_id] = updated
    return updated

# UPDATE (PUT - full replace)
@app.put("/items/{item_id}", response_model=ItemResponse)
def replace_item(item_id: str, item: ItemCreate):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    
    updated = ItemResponse(
        id=item_id,
        **item.dict(),
        created_at=items_db[item_id].created_at
    )
    items_db[item_id] = updated
    return updated

# DELETE
@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_item(item_id: str):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    del items_db[item_id]
```

---

## Approach 2: SQLAlchemy ORM (Sync — Most Common) {#approach-2}

> Standard production approach — synchronous SQLAlchemy with PostgreSQL.

```python
from fastapi import FastAPI, Depends, HTTPException, Query, status
from sqlalchemy import create_engine, Column, Integer, String, Float, DateTime, Boolean
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime

# --- Database Setup ---
DATABASE_URL = "postgresql://user:pass@localhost:5432/mydb"
engine = create_engine(DATABASE_URL, pool_size=10, max_overflow=20)
SessionLocal = sessionmaker(bind=engine)
Base = declarative_base()

# --- ORM Model ---
class Product(Base):
    __tablename__ = "products"
    
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(200), nullable=False)
    description = Column(String(1000))
    price = Column(Float, nullable=False)
    category = Column(String(100), index=True)
    stock = Column(Integer, default=0)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

Base.metadata.create_all(bind=engine)

# --- Pydantic Schemas ---
class ProductCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=200)
    description: Optional[str] = None
    price: float = Field(..., gt=0)
    category: str
    stock: int = Field(default=0, ge=0)

class ProductUpdate(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    price: Optional[float] = Field(None, gt=0)
    category: Optional[str] = None
    stock: Optional[int] = Field(None, ge=0)
    is_active: Optional[bool] = None

class ProductResponse(BaseModel):
    id: int
    name: str
    description: Optional[str]
    price: float
    category: str
    stock: int
    is_active: bool
    created_at: datetime
    updated_at: datetime
    
    class Config:
        from_attributes = True

class PaginatedResponse(BaseModel):
    items: List[ProductResponse]
    total: int
    page: int
    page_size: int
    has_next: bool
    has_prev: bool

# --- Dependency ---
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# --- App ---
app = FastAPI()

# CREATE
@app.post("/products", response_model=ProductResponse, status_code=201)
def create_product(data: ProductCreate, db: Session = Depends(get_db)):
    product = Product(**data.dict())
    db.add(product)
    db.commit()
    db.refresh(product)
    return product

# READ ALL (paginated + filtered + sorted)
@app.get("/products", response_model=PaginatedResponse)
def list_products(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    category: Optional[str] = None,
    min_price: Optional[float] = None,
    max_price: Optional[float] = None,
    search: Optional[str] = None,
    sort_by: str = Query("created_at", regex="^(name|price|created_at|stock)$"),
    order: str = Query("desc", regex="^(asc|desc)$"),
    db: Session = Depends(get_db)
):
    query = db.query(Product).filter(Product.is_active == True)
    
    # Filters
    if category:
        query = query.filter(Product.category == category)
    if min_price is not None:
        query = query.filter(Product.price >= min_price)
    if max_price is not None:
        query = query.filter(Product.price <= max_price)
    if search:
        query = query.filter(Product.name.ilike(f"%{search}%"))
    
    # Sort
    sort_column = getattr(Product, sort_by)
    query = query.order_by(sort_column.desc() if order == "desc" else sort_column.asc())
    
    # Paginate
    total = query.count()
    items = query.offset((page - 1) * page_size).limit(page_size).all()
    
    return PaginatedResponse(
        items=items,
        total=total,
        page=page,
        page_size=page_size,
        has_next=(page * page_size) < total,
        has_prev=page > 1
    )

# READ ONE
@app.get("/products/{product_id}", response_model=ProductResponse)
def get_product(product_id: int, db: Session = Depends(get_db)):
    product = db.query(Product).filter(Product.id == product_id).first()
    if not product:
        raise HTTPException(404, "Product not found")
    return product

# UPDATE (PATCH)
@app.patch("/products/{product_id}", response_model=ProductResponse)
def update_product(product_id: int, data: ProductUpdate, db: Session = Depends(get_db)):
    product = db.query(Product).filter(Product.id == product_id).first()
    if not product:
        raise HTTPException(404, "Product not found")
    
    update_data = data.dict(exclude_unset=True)
    for key, value in update_data.items():
        setattr(product, key, value)
    
    db.commit()
    db.refresh(product)
    return product

# UPDATE (PUT - full replace)
@app.put("/products/{product_id}", response_model=ProductResponse)
def replace_product(product_id: int, data: ProductCreate, db: Session = Depends(get_db)):
    product = db.query(Product).filter(Product.id == product_id).first()
    if not product:
        raise HTTPException(404, "Product not found")
    
    for key, value in data.dict().items():
        setattr(product, key, value)
    
    db.commit()
    db.refresh(product)
    return product

# DELETE (soft)
@app.delete("/products/{product_id}", status_code=204)
def delete_product(product_id: int, db: Session = Depends(get_db)):
    product = db.query(Product).filter(Product.id == product_id).first()
    if not product:
        raise HTTPException(404, "Product not found")
    product.is_active = False
    db.commit()

# DELETE (hard)
@app.delete("/products/{product_id}/permanent", status_code=204)
def hard_delete_product(product_id: int, db: Session = Depends(get_db)):
    product = db.query(Product).filter(Product.id == product_id).first()
    if not product:
        raise HTTPException(404, "Product not found")
    db.delete(product)
    db.commit()

# BULK CREATE
@app.post("/products/bulk", response_model=List[ProductResponse], status_code=201)
def bulk_create(items: List[ProductCreate], db: Session = Depends(get_db)):
    if len(items) > 100:
        raise HTTPException(400, "Max 100 items per batch")
    products = [Product(**item.dict()) for item in items]
    db.bulk_save_objects(products)
    db.commit()
    return db.query(Product).order_by(Product.id.desc()).limit(len(items)).all()

# BULK UPDATE
@app.patch("/products/bulk")
def bulk_update(updates: List[dict], db: Session = Depends(get_db)):
    """Expects: [{"id": 1, "price": 99.99}, {"id": 2, "stock": 50}]"""
    updated_count = 0
    for update in updates:
        product_id = update.pop("id", None)
        if product_id:
            db.query(Product).filter(Product.id == product_id).update(update)
            updated_count += 1
    db.commit()
    return {"updated": updated_count}
```

---

## Approach 3: SQLAlchemy Async (Best Performance) {#approach-3}

> Fully async database operations — best for high-concurrency APIs.

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import select, func, update, delete
from typing import Optional, List
from datetime import datetime

# --- Async Database Setup ---
DATABASE_URL = "postgresql+asyncpg://user:pass@localhost:5432/mydb"
engine = create_async_engine(DATABASE_URL, pool_size=20)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

# --- Model (SQLAlchemy 2.0 style) ---
class Product(Base):
    __tablename__ = "products"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    price: Mapped[float]
    category: Mapped[str] = mapped_column(String(100), index=True)
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

# --- Dependency ---
async def get_db():
    async with async_session() as session:
        yield session

app = FastAPI()

# CREATE
@app.post("/products", status_code=201)
async def create_product(data: ProductCreate, db: AsyncSession = Depends(get_db)):
    product = Product(**data.dict())
    db.add(product)
    await db.commit()
    await db.refresh(product)
    return product

# READ ALL
@app.get("/products")
async def list_products(
    category: Optional[str] = None,
    skip: int = 0,
    limit: int = 20,
    db: AsyncSession = Depends(get_db)
):
    query = select(Product).where(Product.is_active == True)
    
    if category:
        query = query.where(Product.category == category)
    
    query = query.offset(skip).limit(limit).order_by(Product.created_at.desc())
    
    result = await db.execute(query)
    products = result.scalars().all()
    
    # Get total count
    count_query = select(func.count()).select_from(Product).where(Product.is_active == True)
    total = (await db.execute(count_query)).scalar()
    
    return {"items": products, "total": total}

# READ ONE
@app.get("/products/{product_id}")
async def get_product(product_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Product).where(Product.id == product_id))
    product = result.scalar_one_or_none()
    if not product:
        raise HTTPException(404, "Not found")
    return product

# UPDATE
@app.patch("/products/{product_id}")
async def update_product(product_id: int, data: ProductUpdate, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Product).where(Product.id == product_id))
    product = result.scalar_one_or_none()
    if not product:
        raise HTTPException(404, "Not found")
    
    for key, value in data.dict(exclude_unset=True).items():
        setattr(product, key, value)
    
    await db.commit()
    await db.refresh(product)
    return product

# BULK UPDATE (efficient — single query)
@app.patch("/products/bulk-update")
async def bulk_update(category: str, price_multiplier: float, db: AsyncSession = Depends(get_db)):
    stmt = (
        update(Product)
        .where(Product.category == category)
        .values(price=Product.price * price_multiplier)
    )
    result = await db.execute(stmt)
    await db.commit()
    return {"updated_rows": result.rowcount}

# DELETE
@app.delete("/products/{product_id}", status_code=204)
async def delete_product(product_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Product).where(Product.id == product_id))
    product = result.scalar_one_or_none()
    if not product:
        raise HTTPException(404, "Not found")
    await db.delete(product)
    await db.commit()
```

---

## Approach 4: Tortoise ORM (Django-like syntax) {#approach-4}

> If you prefer Django-style ORM but want to stay in FastAPI.

```python
from fastapi import FastAPI, HTTPException
from tortoise import fields, models
from tortoise.contrib.fastapi import register_tortoise
from tortoise.contrib.pydantic import pydantic_model_creator

app = FastAPI()

# --- Model ---
class Product(models.Model):
    id = fields.IntField(pk=True)
    name = fields.CharField(max_length=200)
    description = fields.TextField(null=True)
    price = fields.FloatField()
    category = fields.CharField(max_length=100)
    is_active = fields.BooleanField(default=True)
    created_at = fields.DatetimeField(auto_now_add=True)
    updated_at = fields.DatetimeField(auto_now=True)
    
    class Meta:
        table = "products"
        ordering = ["-created_at"]

# Auto-generate Pydantic schemas from model
ProductPydantic = pydantic_model_creator(Product, name="Product")
ProductCreatePydantic = pydantic_model_creator(Product, name="ProductCreate", exclude_readonly=True)

# --- CRUD ---
@app.post("/products", status_code=201)
async def create_product(data: ProductCreatePydantic):
    product = await Product.create(**data.dict())
    return await ProductPydantic.from_tortoise_orm(product)

@app.get("/products")
async def list_products(category: str = None, limit: int = 20, offset: int = 0):
    query = Product.filter(is_active=True)
    if category:
        query = query.filter(category=category)
    products = await ProductPydantic.from_queryset(query.offset(offset).limit(limit))
    total = await query.count()
    return {"items": products, "total": total}

@app.get("/products/{product_id}")
async def get_product(product_id: int):
    product = await Product.get_or_none(id=product_id)
    if not product:
        raise HTTPException(404, "Not found")
    return await ProductPydantic.from_tortoise_orm(product)

@app.patch("/products/{product_id}")
async def update_product(product_id: int, data: ProductCreatePydantic):
    product = await Product.get_or_none(id=product_id)
    if not product:
        raise HTTPException(404, "Not found")
    await product.update_from_dict(data.dict(exclude_unset=True)).save()
    return await ProductPydantic.from_tortoise_orm(product)

@app.delete("/products/{product_id}", status_code=204)
async def delete_product(product_id: int):
    deleted = await Product.filter(id=product_id).delete()
    if not deleted:
        raise HTTPException(404, "Not found")

# --- Register Tortoise ---
register_tortoise(
    app,
    db_url="postgres://user:pass@localhost:5432/mydb",
    modules={"models": ["__main__"]},
    generate_schemas=True,
    add_exception_handlers=True,
)
```

---

## Approach 5: SQLModel (Pydantic + SQLAlchemy Hybrid) {#approach-5}

> Created by FastAPI's author — single model for DB + API validation.

```python
from fastapi import FastAPI, Depends, HTTPException, Query
from sqlmodel import SQLModel, Field, Session, create_engine, select
from typing import Optional, List
from datetime import datetime

# --- Single model for both DB and API ---
class ProductBase(SQLModel):
    name: str = Field(max_length=200)
    description: Optional[str] = None
    price: float = Field(gt=0)
    category: str = Field(max_length=100)
    stock: int = Field(default=0, ge=0)

class Product(ProductBase, table=True):
    """Database model"""
    id: Optional[int] = Field(default=None, primary_key=True)
    is_active: bool = Field(default=True)
    created_at: datetime = Field(default_factory=datetime.utcnow)

class ProductCreate(ProductBase):
    """Request body for creation"""
    pass

class ProductUpdate(SQLModel):
    """Request body for update (all optional)"""
    name: Optional[str] = None
    description: Optional[str] = None
    price: Optional[float] = Field(None, gt=0)
    category: Optional[str] = None
    stock: Optional[int] = Field(None, ge=0)

class ProductResponse(ProductBase):
    """API response"""
    id: int
    is_active: bool
    created_at: datetime

# --- Engine ---
engine = create_engine("postgresql://user:pass@localhost:5432/mydb")
SQLModel.metadata.create_all(engine)

def get_session():
    with Session(engine) as session:
        yield session

app = FastAPI()

# CREATE
@app.post("/products", response_model=ProductResponse, status_code=201)
def create_product(data: ProductCreate, session: Session = Depends(get_session)):
    product = Product.from_orm(data)
    session.add(product)
    session.commit()
    session.refresh(product)
    return product

# READ ALL
@app.get("/products", response_model=List[ProductResponse])
def list_products(
    category: Optional[str] = None,
    offset: int = 0,
    limit: int = Query(default=20, le=100),
    session: Session = Depends(get_session)
):
    statement = select(Product).where(Product.is_active == True)
    if category:
        statement = statement.where(Product.category == category)
    statement = statement.offset(offset).limit(limit)
    return session.exec(statement).all()

# READ ONE
@app.get("/products/{product_id}", response_model=ProductResponse)
def get_product(product_id: int, session: Session = Depends(get_session)):
    product = session.get(Product, product_id)
    if not product:
        raise HTTPException(404, "Not found")
    return product

# UPDATE
@app.patch("/products/{product_id}", response_model=ProductResponse)
def update_product(product_id: int, data: ProductUpdate, session: Session = Depends(get_session)):
    product = session.get(Product, product_id)
    if not product:
        raise HTTPException(404, "Not found")
    for key, value in data.dict(exclude_unset=True).items():
        setattr(product, key, value)
    session.add(product)
    session.commit()
    session.refresh(product)
    return product

# DELETE
@app.delete("/products/{product_id}", status_code=204)
def delete_product(product_id: int, session: Session = Depends(get_session)):
    product = session.get(Product, product_id)
    if not product:
        raise HTTPException(404, "Not found")
    session.delete(product)
    session.commit()
```

---

## Approach 6: Raw SQL with `databases` Library {#approach-6}

> Maximum control, no ORM overhead — good for complex queries.

```python
from fastapi import FastAPI, HTTPException
from databases import Database
from pydantic import BaseModel
from typing import Optional, List
from datetime import datetime

DATABASE_URL = "postgresql://user:pass@localhost:5432/mydb"
database = Database(DATABASE_URL)

app = FastAPI()

@app.on_event("startup")
async def startup():
    await database.connect()

@app.on_event("shutdown")
async def shutdown():
    await database.disconnect()

class ProductCreate(BaseModel):
    name: str
    price: float
    category: str

# CREATE
@app.post("/products", status_code=201)
async def create_product(data: ProductCreate):
    query = """
        INSERT INTO products (name, price, category, created_at)
        VALUES (:name, :price, :category, NOW())
        RETURNING id, name, price, category, created_at
    """
    row = await database.fetch_one(query, values=data.dict())
    return dict(row)

# READ ALL
@app.get("/products")
async def list_products(category: Optional[str] = None, limit: int = 20, offset: int = 0):
    if category:
        query = """
            SELECT * FROM products 
            WHERE is_active = true AND category = :category
            ORDER BY created_at DESC 
            LIMIT :limit OFFSET :offset
        """
        rows = await database.fetch_all(query, {"category": category, "limit": limit, "offset": offset})
    else:
        query = """
            SELECT * FROM products WHERE is_active = true
            ORDER BY created_at DESC LIMIT :limit OFFSET :offset
        """
        rows = await database.fetch_all(query, {"limit": limit, "offset": offset})
    
    count_query = "SELECT COUNT(*) as total FROM products WHERE is_active = true"
    total = await database.fetch_val(count_query)
    
    return {"items": [dict(r) for r in rows], "total": total}

# READ ONE
@app.get("/products/{product_id}")
async def get_product(product_id: int):
    query = "SELECT * FROM products WHERE id = :id"
    row = await database.fetch_one(query, {"id": product_id})
    if not row:
        raise HTTPException(404, "Not found")
    return dict(row)

# UPDATE
@app.patch("/products/{product_id}")
async def update_product(product_id: int, data: dict):
    # Dynamic SET clause from provided fields
    set_clauses = ", ".join(f"{k} = :{k}" for k in data.keys())
    query = f"""
        UPDATE products SET {set_clauses}, updated_at = NOW()
        WHERE id = :id RETURNING *
    """
    row = await database.fetch_one(query, {**data, "id": product_id})
    if not row:
        raise HTTPException(404, "Not found")
    return dict(row)

# DELETE
@app.delete("/products/{product_id}", status_code=204)
async def delete_product(product_id: int):
    query = "DELETE FROM products WHERE id = :id"
    result = await database.execute(query, {"id": product_id})

# COMPLEX QUERY EXAMPLE
@app.get("/products/analytics/by-category")
async def analytics():
    query = """
        SELECT category, 
               COUNT(*) as count,
               AVG(price) as avg_price,
               SUM(stock) as total_stock
        FROM products
        WHERE is_active = true
        GROUP BY category
        ORDER BY count DESC
    """
    rows = await database.fetch_all(query)
    return [dict(r) for r in rows]
```

---

## Approach 7: MongoDB (Motor — Async) {#approach-7}

> NoSQL approach — flexible schema, great for document-based data.

```python
from fastapi import FastAPI, HTTPException
from motor.motor_asyncio import AsyncIOMotorClient
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime
from bson import ObjectId

app = FastAPI()

# --- MongoDB Connection ---
client = AsyncIOMotorClient("mongodb://localhost:27017")
db = client.mydb
collection = db.products

# --- Custom ObjectId handling ---
class PyObjectId(str):
    @classmethod
    def __get_validators__(cls):
        yield cls.validate
    
    @classmethod
    def validate(cls, v):
        if not ObjectId.is_valid(v):
            raise ValueError("Invalid ObjectId")
        return str(v)

class ProductCreate(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    category: str
    tags: List[str] = []
    metadata: dict = {}

class ProductResponse(BaseModel):
    id: str = Field(alias="_id")
    name: str
    price: float
    category: str
    tags: List[str] = []
    created_at: datetime
    
    class Config:
        populate_by_name = True

def product_serializer(product) -> dict:
    product["_id"] = str(product["_id"])
    return product

# CREATE
@app.post("/products", status_code=201)
async def create_product(data: ProductCreate):
    doc = {**data.dict(), "created_at": datetime.utcnow(), "is_active": True}
    result = await collection.insert_one(doc)
    created = await collection.find_one({"_id": result.inserted_id})
    return product_serializer(created)

# READ ALL (with MongoDB query operators)
@app.get("/products")
async def list_products(
    category: Optional[str] = None,
    min_price: Optional[float] = None,
    max_price: Optional[float] = None,
    tags: Optional[str] = None,  # comma-separated
    skip: int = 0,
    limit: int = 20
):
    filter_query = {"is_active": True}
    
    if category:
        filter_query["category"] = category
    if min_price or max_price:
        filter_query["price"] = {}
        if min_price:
            filter_query["price"]["$gte"] = min_price
        if max_price:
            filter_query["price"]["$lte"] = max_price
    if tags:
        filter_query["tags"] = {"$all": tags.split(",")}
    
    cursor = collection.find(filter_query).skip(skip).limit(limit).sort("created_at", -1)
    products = [product_serializer(doc) async for doc in cursor]
    total = await collection.count_documents(filter_query)
    
    return {"items": products, "total": total}

# READ ONE
@app.get("/products/{product_id}")
async def get_product(product_id: str):
    product = await collection.find_one({"_id": ObjectId(product_id)})
    if not product:
        raise HTTPException(404, "Not found")
    return product_serializer(product)

# UPDATE
@app.patch("/products/{product_id}")
async def update_product(product_id: str, data: dict):
    result = await collection.update_one(
        {"_id": ObjectId(product_id)},
        {"$set": {**data, "updated_at": datetime.utcnow()}}
    )
    if result.matched_count == 0:
        raise HTTPException(404, "Not found")
    product = await collection.find_one({"_id": ObjectId(product_id)})
    return product_serializer(product)

# ADD TO ARRAY (MongoDB-specific)
@app.post("/products/{product_id}/tags")
async def add_tag(product_id: str, tag: str):
    result = await collection.update_one(
        {"_id": ObjectId(product_id)},
        {"$addToSet": {"tags": tag}}
    )
    if result.matched_count == 0:
        raise HTTPException(404, "Not found")
    return {"added": tag}

# DELETE
@app.delete("/products/{product_id}", status_code=204)
async def delete_product(product_id: str):
    result = await collection.delete_one({"_id": ObjectId(product_id)})
    if result.deleted_count == 0:
        raise HTTPException(404, "Not found")

# AGGREGATION (MongoDB pipeline)
@app.get("/products/analytics/summary")
async def product_analytics():
    pipeline = [
        {"$match": {"is_active": True}},
        {"$group": {
            "_id": "$category",
            "count": {"$sum": 1},
            "avg_price": {"$avg": "$price"},
            "max_price": {"$max": "$price"},
            "all_tags": {"$addToSet": "$tags"}
        }},
        {"$sort": {"count": -1}}
    ]
    cursor = collection.aggregate(pipeline)
    return [doc async for doc in cursor]
```

---

## Approach 8: Repository Pattern (Clean Architecture) {#approach-8}

> Enterprise-grade separation of concerns — testable and maintainable.

```python
from abc import ABC, abstractmethod
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import Optional, List, Generic, TypeVar, Type
from pydantic import BaseModel

T = TypeVar("T")

# --- Abstract Repository (Interface) ---
class BaseRepository(ABC, Generic[T]):
    @abstractmethod
    def get(self, id: int) -> Optional[T]: ...
    
    @abstractmethod
    def get_all(self, skip: int = 0, limit: int = 20, **filters) -> List[T]: ...
    
    @abstractmethod
    def create(self, data: dict) -> T: ...
    
    @abstractmethod
    def update(self, id: int, data: dict) -> Optional[T]: ...
    
    @abstractmethod
    def delete(self, id: int) -> bool: ...
    
    @abstractmethod
    def count(self, **filters) -> int: ...

# --- Concrete Repository (SQLAlchemy) ---
class SQLAlchemyRepository(BaseRepository[T]):
    def __init__(self, db: Session, model: Type[T]):
        self.db = db
        self.model = model
    
    def get(self, id: int) -> Optional[T]:
        return self.db.query(self.model).filter(self.model.id == id).first()
    
    def get_all(self, skip: int = 0, limit: int = 20, **filters) -> List[T]:
        query = self.db.query(self.model)
        for key, value in filters.items():
            if value is not None and hasattr(self.model, key):
                query = query.filter(getattr(self.model, key) == value)
        return query.offset(skip).limit(limit).all()
    
    def create(self, data: dict) -> T:
        instance = self.model(**data)
        self.db.add(instance)
        self.db.commit()
        self.db.refresh(instance)
        return instance
    
    def update(self, id: int, data: dict) -> Optional[T]:
        instance = self.get(id)
        if not instance:
            return None
        for key, value in data.items():
            setattr(instance, key, value)
        self.db.commit()
        self.db.refresh(instance)
        return instance
    
    def delete(self, id: int) -> bool:
        instance = self.get(id)
        if not instance:
            return False
        self.db.delete(instance)
        self.db.commit()
        return True
    
    def count(self, **filters) -> int:
        query = self.db.query(self.model)
        for key, value in filters.items():
            if value is not None and hasattr(self.model, key):
                query = query.filter(getattr(self.model, key) == value)
        return query.count()

# --- Service Layer (Business Logic) ---
class ProductService:
    def __init__(self, repo: BaseRepository):
        self.repo = repo
    
    def create_product(self, data: ProductCreate) -> Product:
        # Business rule: prevent duplicate names
        existing = self.repo.get_all(name=data.name)
        if existing:
            raise ValueError("Product with this name already exists")
        return self.repo.create(data.dict())
    
    def get_product(self, product_id: int) -> Product:
        product = self.repo.get(product_id)
        if not product:
            raise ValueError("Product not found")
        return product
    
    def update_product(self, product_id: int, data: ProductUpdate) -> Product:
        update_data = data.dict(exclude_unset=True)
        # Business rule: price can't increase more than 50%
        if "price" in update_data:
            current = self.repo.get(product_id)
            if current and update_data["price"] > current.price * 1.5:
                raise ValueError("Price increase cannot exceed 50%")
        
        result = self.repo.update(product_id, update_data)
        if not result:
            raise ValueError("Product not found")
        return result
    
    def list_products(self, page: int, page_size: int, **filters):
        skip = (page - 1) * page_size
        items = self.repo.get_all(skip=skip, limit=page_size, **filters)
        total = self.repo.count(**filters)
        return {"items": items, "total": total, "page": page}

# --- Dependency Injection ---
def get_product_service(db: Session = Depends(get_db)) -> ProductService:
    repo = SQLAlchemyRepository(db, Product)
    return ProductService(repo)

# --- Routes (thin controllers) ---
app = FastAPI()

@app.post("/products", status_code=201)
def create_product(data: ProductCreate, service: ProductService = Depends(get_product_service)):
    try:
        return service.create_product(data)
    except ValueError as e:
        raise HTTPException(400, str(e))

@app.get("/products/{product_id}")
def get_product(product_id: int, service: ProductService = Depends(get_product_service)):
    try:
        return service.get_product(product_id)
    except ValueError as e:
        raise HTTPException(404, str(e))

@app.patch("/products/{product_id}")
def update_product(product_id: int, data: ProductUpdate, 
                   service: ProductService = Depends(get_product_service)):
    try:
        return service.update_product(product_id, data)
    except ValueError as e:
        raise HTTPException(400, str(e))
```

---

## Approach 9: Generic CRUD Class (Reusable for Any Model) {#approach-9}

> Write once, use for every model — DRY CRUD.

```python
from fastapi import APIRouter, Depends, HTTPException, Query
from sqlalchemy.orm import Session
from typing import TypeVar, Generic, Type, Optional, List
from pydantic import BaseModel

ModelType = TypeVar("ModelType")
CreateSchemaType = TypeVar("CreateSchemaType", bound=BaseModel)
UpdateSchemaType = TypeVar("UpdateSchemaType", bound=BaseModel)
ResponseSchemaType = TypeVar("ResponseSchemaType", bound=BaseModel)

class CRUDRouter(Generic[ModelType, CreateSchemaType, UpdateSchemaType, ResponseSchemaType]):
    def __init__(
        self,
        model: Type[ModelType],
        create_schema: Type[CreateSchemaType],
        update_schema: Type[UpdateSchemaType],
        response_schema: Type[ResponseSchemaType],
        prefix: str,
        tags: List[str] = None
    ):
        self.model = model
        self.create_schema = create_schema
        self.update_schema = update_schema
        self.response_schema = response_schema
        self.router = APIRouter(prefix=prefix, tags=tags or [prefix.strip("/")])
        
        self._register_routes()
    
    def _register_routes(self):
        model = self.model
        create_schema = self.create_schema
        update_schema = self.update_schema
        response_schema = self.response_schema
        
        @self.router.post("/", response_model=response_schema, status_code=201)
        def create(data: create_schema, db: Session = Depends(get_db)):
            instance = model(**data.dict())
            db.add(instance)
            db.commit()
            db.refresh(instance)
            return instance
        
        @self.router.get("/", response_model=List[response_schema])
        def list_all(skip: int = 0, limit: int = Query(20, le=100), 
                     db: Session = Depends(get_db)):
            return db.query(model).offset(skip).limit(limit).all()
        
        @self.router.get("/{item_id}", response_model=response_schema)
        def get_one(item_id: int, db: Session = Depends(get_db)):
            instance = db.query(model).filter(model.id == item_id).first()
            if not instance:
                raise HTTPException(404, "Not found")
            return instance
        
        @self.router.patch("/{item_id}", response_model=response_schema)
        def update(item_id: int, data: update_schema, db: Session = Depends(get_db)):
            instance = db.query(model).filter(model.id == item_id).first()
            if not instance:
                raise HTTPException(404, "Not found")
            for key, value in data.dict(exclude_unset=True).items():
                setattr(instance, key, value)
            db.commit()
            db.refresh(instance)
            return instance
        
        @self.router.delete("/{item_id}", status_code=204)
        def delete(item_id: int, db: Session = Depends(get_db)):
            instance = db.query(model).filter(model.id == item_id).first()
            if not instance:
                raise HTTPException(404, "Not found")
            db.delete(instance)
            db.commit()

# --- Usage: Create CRUD for any model in 3 lines ---
app = FastAPI()

product_crud = CRUDRouter(Product, ProductCreate, ProductUpdate, ProductResponse, "/products")
category_crud = CRUDRouter(Category, CategoryCreate, CategoryUpdate, CategoryResponse, "/categories")
user_crud = CRUDRouter(User, UserCreate, UserUpdate, UserResponse, "/users")

app.include_router(product_crud.router)
app.include_router(category_crud.router)
app.include_router(user_crud.router)
```

---

## Approach 10: Production-Ready (All Features Combined) {#approach-10}

> Complete setup with: auth, validation, caching, logging, error handling, testing.

```python
# See file: 11_fastapi_django_examples.md → Example 1 + Example 2 + Example 4
# This combines:
# - SQLAlchemy ORM (Approach 2)
# - JWT Auth (from file 11)
# - Custom error handling
# - Request logging middleware
# - Redis caching
# - Background tasks
# - API versioning
# - Health checks
# - Rate limiting

# Key additions for production:
from fastapi import FastAPI
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.gzip import GZipMiddleware

app = FastAPI(
    title="HCP API",
    version="2.0.0",
    docs_url="/api/docs",
    redoc_url="/api/redoc"
)

# Security middlewares
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["*.example.com", "localhost"])
app.add_middleware(GZipMiddleware, minimum_size=1000)

# Health + readiness
@app.get("/health/live")
async def liveness():
    return {"status": "alive"}

@app.get("/health/ready")
async def readiness():
    try:
        db = SessionLocal()
        db.execute("SELECT 1")
        db.close()
        return {"status": "ready", "database": "connected"}
    except Exception as e:
        return JSONResponse(503, {"status": "not ready", "error": str(e)})
```

---

## Quick Reference — Which Approach to Use?

| Situation | Approach |
|-----------|----------|
| Prototyping / learning | Approach 1 (In-memory) |
| Standard production API | Approach 2 (SQLAlchemy Sync) |
| High-concurrency API | Approach 3 (SQLAlchemy Async) |
| Prefer Django-like ORM | Approach 4 (Tortoise) |
| Want single model for DB+API | Approach 5 (SQLModel) |
| Need complex custom SQL | Approach 6 (Raw SQL) |
| Document/flexible schema | Approach 7 (MongoDB) |
| Enterprise / large team | Approach 8 (Repository Pattern) |
| Many similar models | Approach 9 (Generic CRUD) |
| Production deployment | Approach 10 (Full setup) |
