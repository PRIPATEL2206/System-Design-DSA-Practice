# FastAPI & Django — Complete Implementation Examples

> Real-world API implementations you can reference in interviews and build upon.

---

## Part A: FastAPI Complete Examples

---

### Example 1: Full CRUD API with Database (SQLAlchemy + PostgreSQL)

```python
# project structure:
# app/
# ├── main.py
# ├── database.py
# ├── models.py
# ├── schemas.py
# ├── crud.py
# └── routers/
#     ├── hcp.py
#     └── auth.py

# ========== database.py ==========
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "postgresql://user:password@localhost:5432/hcp_db"

engine = create_engine(DATABASE_URL, pool_size=20, max_overflow=10, pool_pre_ping=True)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


# ========== models.py (SQLAlchemy ORM) ==========
from sqlalchemy import Column, Integer, String, DateTime, Boolean, Float, ForeignKey, Enum
from sqlalchemy.orm import relationship
from datetime import datetime
import enum

class RegionEnum(str, enum.Enum):
    NORTH = "north"
    SOUTH = "south"
    EAST = "east"
    WEST = "west"

class HCP(Base):
    __tablename__ = "hcp_records"
    
    id = Column(Integer, primary_key=True, index=True)
    hcp_id = Column(String(50), unique=True, index=True, nullable=False)
    name = Column(String(200), nullable=False)
    specialty = Column(String(100))
    region = Column(Enum(RegionEnum), index=True)
    email = Column(String(200), unique=True)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    interactions = relationship("Interaction", back_populates="hcp")

class Interaction(Base):
    __tablename__ = "interactions"
    
    id = Column(Integer, primary_key=True, index=True)
    hcp_record_id = Column(Integer, ForeignKey("hcp_records.id"))
    interaction_type = Column(String(50))  # visit, prescription, conference
    value = Column(Float, default=0)
    notes = Column(String(500))
    interaction_date = Column(DateTime, default=datetime.utcnow)
    
    hcp = relationship("HCP", back_populates="interactions")


# ========== schemas.py (Pydantic models) ==========
from pydantic import BaseModel, Field, EmailStr, validator
from typing import Optional, List
from datetime import datetime

class HCPCreate(BaseModel):
    name: str = Field(..., min_length=2, max_length=200)
    specialty: str = Field(..., max_length=100)
    region: str
    email: EmailStr
    
    @validator("region")
    def validate_region(cls, v):
        valid = ["north", "south", "east", "west"]
        if v.lower() not in valid:
            raise ValueError(f"Region must be one of: {valid}")
        return v.lower()

class HCPUpdate(BaseModel):
    name: Optional[str] = None
    specialty: Optional[str] = None
    region: Optional[str] = None
    email: Optional[EmailStr] = None
    is_active: Optional[bool] = None

class HCPResponse(BaseModel):
    id: int
    hcp_id: str
    name: str
    specialty: str
    region: str
    email: str
    is_active: bool
    created_at: datetime
    interaction_count: int = 0
    
    class Config:
        from_attributes = True

class InteractionCreate(BaseModel):
    interaction_type: str = Field(..., pattern="^(visit|prescription|conference|call)$")
    value: float = Field(ge=0)
    notes: Optional[str] = Field(None, max_length=500)

class InteractionResponse(BaseModel):
    id: int
    interaction_type: str
    value: float
    notes: Optional[str]
    interaction_date: datetime
    
    class Config:
        from_attributes = True

class PaginatedHCPResponse(BaseModel):
    items: List[HCPResponse]
    total: int
    page: int
    page_size: int
    pages: int


# ========== crud.py ==========
from sqlalchemy.orm import Session
from sqlalchemy import func
import uuid

def create_hcp(db: Session, hcp_data: HCPCreate) -> HCP:
    hcp = HCP(
        hcp_id=f"HCP-{uuid.uuid4().hex[:8].upper()}",
        name=hcp_data.name,
        specialty=hcp_data.specialty,
        region=hcp_data.region,
        email=hcp_data.email
    )
    db.add(hcp)
    db.commit()
    db.refresh(hcp)
    return hcp

def get_hcp(db: Session, hcp_id: str) -> Optional[HCP]:
    return db.query(HCP).filter(HCP.hcp_id == hcp_id).first()

def get_hcps(db: Session, page: int = 1, page_size: int = 20,
             region: str = None, specialty: str = None,
             search: str = None) -> tuple:
    query = db.query(HCP).filter(HCP.is_active == True)
    
    if region:
        query = query.filter(HCP.region == region)
    if specialty:
        query = query.filter(HCP.specialty.ilike(f"%{specialty}%"))
    if search:
        query = query.filter(HCP.name.ilike(f"%{search}%"))
    
    total = query.count()
    items = query.offset((page - 1) * page_size).limit(page_size).all()
    
    return items, total

def update_hcp(db: Session, hcp_id: str, hcp_data: HCPUpdate) -> Optional[HCP]:
    hcp = db.query(HCP).filter(HCP.hcp_id == hcp_id).first()
    if not hcp:
        return None
    
    update_data = hcp_data.dict(exclude_unset=True)
    for field, value in update_data.items():
        setattr(hcp, field, value)
    
    db.commit()
    db.refresh(hcp)
    return hcp

def delete_hcp(db: Session, hcp_id: str) -> bool:
    hcp = db.query(HCP).filter(HCP.hcp_id == hcp_id).first()
    if not hcp:
        return False
    hcp.is_active = False  # Soft delete
    db.commit()
    return True

def add_interaction(db: Session, hcp_id: str, data: InteractionCreate) -> Optional[Interaction]:
    hcp = db.query(HCP).filter(HCP.hcp_id == hcp_id).first()
    if not hcp:
        return None
    
    interaction = Interaction(
        hcp_record_id=hcp.id,
        interaction_type=data.interaction_type,
        value=data.value,
        notes=data.notes
    )
    db.add(interaction)
    db.commit()
    db.refresh(interaction)
    return interaction


# ========== main.py ==========
from fastapi import FastAPI, Depends, HTTPException, Query, status
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy.orm import Session
import math

app = FastAPI(
    title="HCP Management API",
    description="Healthcare Professional records management system",
    version="2.0.0"
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# --- HCP Endpoints ---
@app.post("/api/v1/hcp", response_model=HCPResponse, status_code=status.HTTP_201_CREATED,
          tags=["HCP"])
async def create_hcp_endpoint(hcp_data: HCPCreate, db: Session = Depends(get_db)):
    """Create a new HCP record."""
    existing = db.query(HCP).filter(HCP.email == hcp_data.email).first()
    if existing:
        raise HTTPException(status_code=409, detail="Email already registered")
    return create_hcp(db, hcp_data)

@app.get("/api/v1/hcp", response_model=PaginatedHCPResponse, tags=["HCP"])
async def list_hcps(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    region: Optional[str] = None,
    specialty: Optional[str] = None,
    search: Optional[str] = None,
    db: Session = Depends(get_db)
):
    """List HCP records with filtering and pagination."""
    items, total = get_hcps(db, page, page_size, region, specialty, search)
    return PaginatedHCPResponse(
        items=items,
        total=total,
        page=page,
        page_size=page_size,
        pages=math.ceil(total / page_size)
    )

@app.get("/api/v1/hcp/{hcp_id}", response_model=HCPResponse, tags=["HCP"])
async def get_hcp_endpoint(hcp_id: str, db: Session = Depends(get_db)):
    """Get a single HCP record by ID."""
    hcp = get_hcp(db, hcp_id)
    if not hcp:
        raise HTTPException(status_code=404, detail="HCP not found")
    return hcp

@app.patch("/api/v1/hcp/{hcp_id}", response_model=HCPResponse, tags=["HCP"])
async def update_hcp_endpoint(hcp_id: str, hcp_data: HCPUpdate, 
                              db: Session = Depends(get_db)):
    """Update an HCP record (partial update)."""
    hcp = update_hcp(db, hcp_id, hcp_data)
    if not hcp:
        raise HTTPException(status_code=404, detail="HCP not found")
    return hcp

@app.delete("/api/v1/hcp/{hcp_id}", status_code=status.HTTP_204_NO_CONTENT, tags=["HCP"])
async def delete_hcp_endpoint(hcp_id: str, db: Session = Depends(get_db)):
    """Soft-delete an HCP record."""
    if not delete_hcp(db, hcp_id):
        raise HTTPException(status_code=404, detail="HCP not found")

# --- Interactions ---
@app.post("/api/v1/hcp/{hcp_id}/interactions", response_model=InteractionResponse,
          status_code=status.HTTP_201_CREATED, tags=["Interactions"])
async def add_interaction_endpoint(hcp_id: str, data: InteractionCreate,
                                   db: Session = Depends(get_db)):
    """Add an interaction for an HCP."""
    interaction = add_interaction(db, hcp_id, data)
    if not interaction:
        raise HTTPException(status_code=404, detail="HCP not found")
    return interaction

@app.get("/api/v1/hcp/{hcp_id}/interactions", response_model=List[InteractionResponse],
         tags=["Interactions"])
async def list_interactions(hcp_id: str, limit: int = Query(50, le=200),
                           db: Session = Depends(get_db)):
    """List interactions for an HCP."""
    hcp = get_hcp(db, hcp_id)
    if not hcp:
        raise HTTPException(status_code=404, detail="HCP not found")
    return db.query(Interaction)\
        .filter(Interaction.hcp_record_id == hcp.id)\
        .order_by(Interaction.interaction_date.desc())\
        .limit(limit).all()
```

---

### Example 2: JWT Authentication System (FastAPI)

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel, EmailStr
from datetime import datetime, timedelta
from typing import Optional
import jwt
from passlib.context import CryptContext

app = FastAPI()

# --- Config ---
SECRET_KEY = "your-256-bit-secret"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30
REFRESH_TOKEN_EXPIRE_DAYS = 7

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")

# --- Models ---
class UserCreate(BaseModel):
    email: EmailStr
    password: str
    full_name: str
    role: str = "viewer"  # viewer, editor, admin

class UserResponse(BaseModel):
    id: int
    email: str
    full_name: str
    role: str
    is_active: bool
    created_at: datetime

class Token(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int

# --- In-memory user store (use DB in production) ---
users_db = {}

# --- Auth utilities ---
def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_token(data: dict, expires_delta: timedelta) -> str:
    to_encode = data.copy()
    to_encode["exp"] = datetime.utcnow() + expires_delta
    to_encode["iat"] = datetime.utcnow()
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def create_access_token(user_id: int, role: str) -> str:
    return create_token(
        {"sub": str(user_id), "role": role, "type": "access"},
        timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    )

def create_refresh_token(user_id: int) -> str:
    return create_token(
        {"sub": str(user_id), "type": "refresh"},
        timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    )

async def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id = payload.get("sub")
        token_type = payload.get("type")
        
        if user_id is None or token_type != "access":
            raise credentials_exception
        
        user = users_db.get(int(user_id))
        if user is None or not user["is_active"]:
            raise credentials_exception
        
        return user
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise credentials_exception

def require_role(allowed_roles: list):
    """Role-based access control dependency."""
    async def role_checker(user=Depends(get_current_user)):
        if user["role"] not in allowed_roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Role '{user['role']}' not authorized. Required: {allowed_roles}"
            )
        return user
    return role_checker

# --- Auth Endpoints ---
@app.post("/api/v1/auth/register", response_model=UserResponse, status_code=201)
async def register(user_data: UserCreate):
    # Check if email exists
    for user in users_db.values():
        if user["email"] == user_data.email:
            raise HTTPException(409, "Email already registered")
    
    user_id = len(users_db) + 1
    users_db[user_id] = {
        "id": user_id,
        "email": user_data.email,
        "full_name": user_data.full_name,
        "password_hash": hash_password(user_data.password),
        "role": user_data.role,
        "is_active": True,
        "created_at": datetime.utcnow()
    }
    return users_db[user_id]

@app.post("/api/v1/auth/login", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    # Find user by email
    user = None
    for u in users_db.values():
        if u["email"] == form_data.username:
            user = u
            break
    
    if not user or not verify_password(form_data.password, user["password_hash"]):
        raise HTTPException(401, "Invalid email or password")
    
    if not user["is_active"]:
        raise HTTPException(403, "Account deactivated")
    
    return Token(
        access_token=create_access_token(user["id"], user["role"]),
        refresh_token=create_refresh_token(user["id"]),
        expires_in=ACCESS_TOKEN_EXPIRE_MINUTES * 60
    )

@app.post("/api/v1/auth/refresh", response_model=Token)
async def refresh_token(refresh_token: str):
    try:
        payload = jwt.decode(refresh_token, SECRET_KEY, algorithms=[ALGORITHM])
        if payload.get("type") != "refresh":
            raise HTTPException(401, "Invalid token type")
        
        user_id = int(payload["sub"])
        user = users_db.get(user_id)
        if not user:
            raise HTTPException(401, "User not found")
        
        return Token(
            access_token=create_access_token(user["id"], user["role"]),
            refresh_token=create_refresh_token(user["id"]),
            expires_in=ACCESS_TOKEN_EXPIRE_MINUTES * 60
        )
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, "Refresh token expired. Please login again.")
    except jwt.InvalidTokenError:
        raise HTTPException(401, "Invalid refresh token")

@app.get("/api/v1/auth/me", response_model=UserResponse)
async def get_me(user=Depends(get_current_user)):
    return user

# --- Protected routes with role-based access ---
@app.get("/api/v1/admin/users", dependencies=[Depends(require_role(["admin"]))])
async def list_users():
    return list(users_db.values())

@app.delete("/api/v1/admin/users/{user_id}", dependencies=[Depends(require_role(["admin"]))])
async def deactivate_user(user_id: int):
    if user_id not in users_db:
        raise HTTPException(404, "User not found")
    users_db[user_id]["is_active"] = False
    return {"deactivated": True}
```

---

### Example 3: File Upload + Background Processing (FastAPI)

```python
from fastapi import FastAPI, UploadFile, File, BackgroundTasks, HTTPException
from fastapi.responses import FileResponse
from typing import List
import uuid
import os
import pandas as pd
from datetime import datetime

app = FastAPI()
UPLOAD_DIR = "/tmp/uploads"
os.makedirs(UPLOAD_DIR, exist_ok=True)

# Track processing status
processing_jobs = {}

class ProcessingStatus:
    def __init__(self, filename: str):
        self.job_id = str(uuid.uuid4())
        self.filename = filename
        self.status = "uploaded"  # uploaded, validating, processing, completed, failed
        self.progress = 0
        self.total_rows = 0
        self.processed_rows = 0
        self.errors = []
        self.result_file = None
        self.started_at = datetime.utcnow()
        self.completed_at = None

def process_csv_file(job_id: str, filepath: str):
    """Background task: validate and process uploaded CSV."""
    job = processing_jobs[job_id]
    
    try:
        # Step 1: Validate
        job.status = "validating"
        df = pd.read_csv(filepath)
        job.total_rows = len(df)
        
        required_cols = ["hcp_id", "name", "region", "value"]
        missing = [col for col in required_cols if col not in df.columns]
        if missing:
            job.status = "failed"
            job.errors.append(f"Missing columns: {missing}")
            return
        
        # Step 2: Process
        job.status = "processing"
        
        # Clean data
        df["name"] = df["name"].str.strip().str.title()
        df["region"] = df["region"].str.lower()
        df = df.dropna(subset=["hcp_id"])
        
        # Validate regions
        valid_regions = {"north", "south", "east", "west"}
        invalid_mask = ~df["region"].isin(valid_regions)
        if invalid_mask.any():
            job.errors.append(f"{invalid_mask.sum()} rows with invalid region")
            df = df[~invalid_mask]
        
        # Remove duplicates
        dupes = df.duplicated(subset=["hcp_id"], keep="last")
        if dupes.any():
            job.errors.append(f"{dupes.sum()} duplicate records removed")
            df = df[~dupes]
        
        job.processed_rows = len(df)
        job.progress = 100
        
        # Step 3: Save result
        result_path = filepath.replace(".csv", "_processed.csv")
        df.to_csv(result_path, index=False)
        job.result_file = result_path
        
        job.status = "completed"
        job.completed_at = datetime.utcnow()
        
    except Exception as e:
        job.status = "failed"
        job.errors.append(str(e))

@app.post("/api/v1/upload/csv")
async def upload_csv(file: UploadFile = File(...), background_tasks: BackgroundTasks = None):
    """Upload a CSV file for processing."""
    if not file.filename.endswith(".csv"):
        raise HTTPException(400, "Only CSV files accepted")
    
    if file.size > 100 * 1024 * 1024:  # 100MB limit
        raise HTTPException(400, "File too large. Max 100MB.")
    
    # Save file
    file_id = str(uuid.uuid4())
    filepath = os.path.join(UPLOAD_DIR, f"{file_id}.csv")
    
    content = await file.read()
    with open(filepath, "wb") as f:
        f.write(content)
    
    # Create processing job
    job = ProcessingStatus(file.filename)
    processing_jobs[job.job_id] = job
    
    # Start background processing
    background_tasks.add_task(process_csv_file, job.job_id, filepath)
    
    return {
        "job_id": job.job_id,
        "filename": file.filename,
        "size_bytes": len(content),
        "status": "uploaded",
        "status_url": f"/api/v1/upload/status/{job.job_id}"
    }

@app.get("/api/v1/upload/status/{job_id}")
async def get_upload_status(job_id: str):
    """Check processing status."""
    if job_id not in processing_jobs:
        raise HTTPException(404, "Job not found")
    
    job = processing_jobs[job_id]
    return {
        "job_id": job.job_id,
        "filename": job.filename,
        "status": job.status,
        "progress": job.progress,
        "total_rows": job.total_rows,
        "processed_rows": job.processed_rows,
        "errors": job.errors,
        "download_url": f"/api/v1/upload/download/{job_id}" if job.result_file else None
    }

@app.get("/api/v1/upload/download/{job_id}")
async def download_result(job_id: str):
    """Download processed file."""
    if job_id not in processing_jobs:
        raise HTTPException(404, "Job not found")
    
    job = processing_jobs[job_id]
    if not job.result_file or job.status != "completed":
        raise HTTPException(400, "File not ready")
    
    return FileResponse(
        job.result_file,
        media_type="text/csv",
        filename=f"processed_{job.filename}"
    )
```

---

### Example 4: FastAPI with Middleware, Error Handling, and Logging

```python
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
import time
import logging
import uuid
from typing import Callable

# --- Logging setup ---
logging.basicConfig(level=logging.INFO, format="%(asctime)s | %(levelname)s | %(message)s")
logger = logging.getLogger("api")

app = FastAPI()

# --- Custom exception classes ---
class AppException(Exception):
    def __init__(self, status_code: int, detail: str, error_code: str = None):
        self.status_code = status_code
        self.detail = detail
        self.error_code = error_code or "UNKNOWN_ERROR"

class NotFoundException(AppException):
    def __init__(self, resource: str, resource_id: str):
        super().__init__(404, f"{resource} with id '{resource_id}' not found", "NOT_FOUND")

class ConflictException(AppException):
    def __init__(self, detail: str):
        super().__init__(409, detail, "CONFLICT")

class RateLimitException(AppException):
    def __init__(self, retry_after: int = 60):
        super().__init__(429, "Rate limit exceeded", "RATE_LIMIT_EXCEEDED")
        self.retry_after = retry_after

# --- Exception handlers ---
@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "code": exc.error_code,
                "message": exc.detail,
                "request_id": request.state.request_id
            }
        }
    )

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    errors = []
    for error in exc.errors():
        errors.append({
            "field": ".".join(str(loc) for loc in error["loc"][1:]),
            "message": error["msg"],
            "type": error["type"]
        })
    return JSONResponse(
        status_code=422,
        content={
            "error": {
                "code": "VALIDATION_ERROR",
                "message": "Request validation failed",
                "details": errors,
                "request_id": request.state.request_id
            }
        }
    )

@app.exception_handler(Exception)
async def generic_exception_handler(request: Request, exc: Exception):
    logger.error(f"Unhandled exception: {exc}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={
            "error": {
                "code": "INTERNAL_ERROR",
                "message": "An unexpected error occurred",
                "request_id": request.state.request_id
            }
        }
    )

# --- Middleware ---
@app.middleware("http")
async def request_middleware(request: Request, call_next: Callable):
    # Generate request ID
    request_id = str(uuid.uuid4())[:8]
    request.state.request_id = request_id
    
    # Log request
    start_time = time.time()
    logger.info(f"[{request_id}] {request.method} {request.url.path}")
    
    # Process request
    response = await call_next(request)
    
    # Log response
    duration = time.time() - start_time
    logger.info(f"[{request_id}] {response.status_code} ({duration:.3f}s)")
    
    # Add response headers
    response.headers["X-Request-ID"] = request_id
    response.headers["X-Response-Time"] = f"{duration:.3f}s"
    
    return response

# --- Usage ---
@app.get("/api/v1/hcp/{hcp_id}")
async def get_hcp(hcp_id: str):
    # This will be caught by our custom exception handler
    raise NotFoundException("HCP", hcp_id)
```

---

## Part B: Django Complete Examples

---

### Example 5: Django REST Framework — Full API

```python
# project structure:
# myproject/
# ├── manage.py
# ├── myproject/
# │   ├── settings.py
# │   ├── urls.py
# │   └── wsgi.py
# └── hcp/
#     ├── models.py
#     ├── serializers.py
#     ├── views.py
#     ├── urls.py
#     ├── permissions.py
#     └── filters.py

# ========== hcp/models.py ==========
from django.db import models
from django.contrib.auth.models import User

class HCPRecord(models.Model):
    REGION_CHOICES = [
        ('north', 'North'),
        ('south', 'South'),
        ('east', 'East'),
        ('west', 'West'),
    ]
    
    hcp_id = models.CharField(max_length=50, unique=True, db_index=True)
    name = models.CharField(max_length=200)
    specialty = models.CharField(max_length=100)
    region = models.CharField(max_length=20, choices=REGION_CHOICES, db_index=True)
    email = models.EmailField(unique=True)
    is_active = models.BooleanField(default=True)
    created_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['region', 'specialty']),
            models.Index(fields=['created_at']),
        ]
    
    def __str__(self):
        return f"{self.hcp_id} - {self.name}"

class Interaction(models.Model):
    TYPE_CHOICES = [
        ('visit', 'Visit'),
        ('prescription', 'Prescription'),
        ('conference', 'Conference'),
        ('call', 'Phone Call'),
    ]
    
    hcp = models.ForeignKey(HCPRecord, on_delete=models.CASCADE, related_name='interactions')
    interaction_type = models.CharField(max_length=20, choices=TYPE_CHOICES)
    value = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    notes = models.TextField(blank=True, max_length=500)
    interaction_date = models.DateTimeField(auto_now_add=True)
    recorded_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    
    class Meta:
        ordering = ['-interaction_date']


# ========== hcp/serializers.py ==========
from rest_framework import serializers
from .models import HCPRecord, Interaction

class InteractionSerializer(serializers.ModelSerializer):
    recorded_by_name = serializers.CharField(source='recorded_by.get_full_name', read_only=True)
    
    class Meta:
        model = Interaction
        fields = ['id', 'interaction_type', 'value', 'notes', 'interaction_date', 'recorded_by_name']
        read_only_fields = ['id', 'interaction_date', 'recorded_by_name']

class HCPListSerializer(serializers.ModelSerializer):
    interaction_count = serializers.SerializerMethodField()
    total_value = serializers.SerializerMethodField()
    
    class Meta:
        model = HCPRecord
        fields = ['hcp_id', 'name', 'specialty', 'region', 'is_active', 
                  'interaction_count', 'total_value', 'created_at']
    
    def get_interaction_count(self, obj):
        return obj.interactions.count()
    
    def get_total_value(self, obj):
        return obj.interactions.aggregate(total=models.Sum('value'))['total'] or 0

class HCPDetailSerializer(serializers.ModelSerializer):
    interactions = InteractionSerializer(many=True, read_only=True)
    created_by_name = serializers.CharField(source='created_by.get_full_name', read_only=True)
    
    class Meta:
        model = HCPRecord
        fields = ['hcp_id', 'name', 'specialty', 'region', 'email', 'is_active',
                  'interactions', 'created_by_name', 'created_at', 'updated_at']
        read_only_fields = ['hcp_id', 'created_at', 'updated_at']

class HCPCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = HCPRecord
        fields = ['name', 'specialty', 'region', 'email']
    
    def validate_email(self, value):
        if HCPRecord.objects.filter(email=value).exists():
            raise serializers.ValidationError("Email already registered")
        return value
    
    def create(self, validated_data):
        import uuid
        validated_data['hcp_id'] = f"HCP-{uuid.uuid4().hex[:8].upper()}"
        validated_data['created_by'] = self.context['request'].user
        return super().create(validated_data)


# ========== hcp/filters.py ==========
import django_filters
from .models import HCPRecord

class HCPFilter(django_filters.FilterSet):
    name = django_filters.CharFilter(lookup_expr='icontains')
    specialty = django_filters.CharFilter(lookup_expr='icontains')
    region = django_filters.ChoiceFilter(choices=HCPRecord.REGION_CHOICES)
    created_after = django_filters.DateFilter(field_name='created_at', lookup_expr='gte')
    created_before = django_filters.DateFilter(field_name='created_at', lookup_expr='lte')
    min_interactions = django_filters.NumberFilter(method='filter_min_interactions')
    
    class Meta:
        model = HCPRecord
        fields = ['name', 'specialty', 'region', 'is_active']
    
    def filter_min_interactions(self, queryset, name, value):
        from django.db.models import Count
        return queryset.annotate(num_interactions=Count('interactions'))\
            .filter(num_interactions__gte=value)


# ========== hcp/permissions.py ==========
from rest_framework import permissions

class IsAdminOrReadOnly(permissions.BasePermission):
    def has_permission(self, request, view):
        if request.method in permissions.SAFE_METHODS:
            return True
        return request.user and request.user.is_staff

class IsOwnerOrAdmin(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.user.is_staff:
            return True
        return obj.created_by == request.user


# ========== hcp/views.py ==========
from rest_framework import viewsets, status, filters
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from django_filters.rest_framework import DjangoFilterBackend
from django.db.models import Sum, Count, Avg
from .models import HCPRecord, Interaction
from .serializers import (HCPListSerializer, HCPDetailSerializer, 
                          HCPCreateSerializer, InteractionSerializer)
from .filters import HCPFilter
from .permissions import IsAdminOrReadOnly

class HCPViewSet(viewsets.ModelViewSet):
    queryset = HCPRecord.objects.filter(is_active=True)
    permission_classes = [IsAuthenticated, IsAdminOrReadOnly]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_class = HCPFilter
    search_fields = ['name', 'hcp_id', 'specialty']
    ordering_fields = ['name', 'created_at', 'region']
    ordering = ['-created_at']
    
    def get_serializer_class(self):
        if self.action == 'list':
            return HCPListSerializer
        elif self.action == 'create':
            return HCPCreateSerializer
        return HCPDetailSerializer
    
    def destroy(self, request, *args, **kwargs):
        """Soft delete instead of hard delete."""
        instance = self.get_object()
        instance.is_active = False
        instance.save()
        return Response(status=status.HTTP_204_NO_CONTENT)
    
    @action(detail=True, methods=['post'])
    def add_interaction(self, request, pk=None):
        """Add interaction to an HCP. POST /api/hcp/{id}/add_interaction/"""
        hcp = self.get_object()
        serializer = InteractionSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save(hcp=hcp, recorded_by=request.user)
        return Response(serializer.data, status=status.HTTP_201_CREATED)
    
    @action(detail=True, methods=['get'])
    def interactions(self, request, pk=None):
        """List interactions for an HCP. GET /api/hcp/{id}/interactions/"""
        hcp = self.get_object()
        interactions = hcp.interactions.all()[:50]
        serializer = InteractionSerializer(interactions, many=True)
        return Response(serializer.data)
    
    @action(detail=False, methods=['get'])
    def stats(self, request):
        """Get aggregate statistics. GET /api/hcp/stats/"""
        qs = self.get_queryset()
        stats = qs.aggregate(
            total_hcps=Count('id'),
            total_interactions=Count('interactions'),
            total_value=Sum('interactions__value'),
            avg_value=Avg('interactions__value'),
        )
        
        region_stats = qs.values('region').annotate(
            count=Count('id'),
            total_value=Sum('interactions__value')
        ).order_by('region')
        
        specialty_stats = qs.values('specialty').annotate(
            count=Count('id')
        ).order_by('-count')[:10]
        
        return Response({
            "overview": stats,
            "by_region": list(region_stats),
            "top_specialties": list(specialty_stats)
        })
    
    @action(detail=False, methods=['post'])
    def bulk_create(self, request):
        """Bulk create HCP records. POST /api/hcp/bulk_create/"""
        records = request.data
        if not isinstance(records, list):
            return Response({"error": "Expected a list"}, status=400)
        if len(records) > 100:
            return Response({"error": "Max 100 records per batch"}, status=400)
        
        created = []
        errors = []
        for i, record_data in enumerate(records):
            serializer = HCPCreateSerializer(data=record_data, context={'request': request})
            if serializer.is_valid():
                serializer.save()
                created.append(serializer.data)
            else:
                errors.append({"index": i, "errors": serializer.errors})
        
        return Response({
            "created": len(created),
            "failed": len(errors),
            "errors": errors
        }, status=status.HTTP_201_CREATED if created else status.HTTP_400_BAD_REQUEST)


# ========== hcp/urls.py ==========
from rest_framework.routers import DefaultRouter
from .views import HCPViewSet

router = DefaultRouter()
router.register(r'hcp', HCPViewSet, basename='hcp')
urlpatterns = router.urls


# ========== myproject/urls.py ==========
from django.urls import path, include
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path('api/v1/', include('hcp.urls')),
    path('api/v1/auth/token/', TokenObtainPairView.as_view()),
    path('api/v1/auth/token/refresh/', TokenRefreshView.as_view()),
]
```

---

### Example 6: Django — Custom Management Command for ETL

```python
# hcp/management/commands/run_etl.py
from django.core.management.base import BaseCommand, CommandError
from django.db import transaction
from hcp.models import HCPRecord, Interaction
import csv
import os
from datetime import datetime

class Command(BaseCommand):
    help = 'Run ETL pipeline to import HCP records from CSV'
    
    def add_arguments(self, parser):
        parser.add_argument('source_file', type=str, help='Path to CSV file')
        parser.add_argument('--dry-run', action='store_true', help='Validate without importing')
        parser.add_argument('--batch-size', type=int, default=1000, help='Batch size for bulk create')
        parser.add_argument('--skip-duplicates', action='store_true', help='Skip duplicate records')
    
    def handle(self, *args, **options):
        source_file = options['source_file']
        dry_run = options['dry_run']
        batch_size = options['batch_size']
        
        if not os.path.exists(source_file):
            raise CommandError(f"File not found: {source_file}")
        
        self.stdout.write(f"Processing: {source_file}")
        
        # Phase 1: Validate
        records, errors = self.validate_file(source_file)
        self.stdout.write(f"Valid records: {len(records)}, Errors: {len(errors)}")
        
        if errors:
            for err in errors[:10]:
                self.stderr.write(self.style.ERROR(f"  Row {err['row']}: {err['message']}"))
        
        if dry_run:
            self.stdout.write(self.style.SUCCESS("Dry run complete. No records imported."))
            return
        
        # Phase 2: Import
        created, updated, skipped = self.import_records(records, batch_size, 
                                                        options['skip_duplicates'])
        
        self.stdout.write(self.style.SUCCESS(
            f"ETL Complete: {created} created, {updated} updated, {skipped} skipped"
        ))
    
    def validate_file(self, filepath):
        records = []
        errors = []
        
        with open(filepath, 'r') as f:
            reader = csv.DictReader(f)
            
            required_cols = {'name', 'specialty', 'region', 'email'}
            if not required_cols.issubset(set(reader.fieldnames or [])):
                missing = required_cols - set(reader.fieldnames or [])
                errors.append({'row': 0, 'message': f'Missing columns: {missing}'})
                return records, errors
            
            for i, row in enumerate(reader, start=2):
                row_errors = []
                
                if not row.get('name', '').strip():
                    row_errors.append("Missing name")
                if row.get('region', '').lower() not in ['north', 'south', 'east', 'west']:
                    row_errors.append(f"Invalid region: {row.get('region')}")
                if not row.get('email', '').strip():
                    row_errors.append("Missing email")
                
                if row_errors:
                    errors.append({'row': i, 'message': '; '.join(row_errors)})
                else:
                    records.append({
                        'name': row['name'].strip().title(),
                        'specialty': row['specialty'].strip(),
                        'region': row['region'].strip().lower(),
                        'email': row['email'].strip().lower(),
                    })
        
        return records, errors
    
    @transaction.atomic
    def import_records(self, records, batch_size, skip_duplicates):
        created = 0
        updated = 0
        skipped = 0
        
        for i in range(0, len(records), batch_size):
            batch = records[i:i + batch_size]
            
            for record in batch:
                existing = HCPRecord.objects.filter(email=record['email']).first()
                
                if existing:
                    if skip_duplicates:
                        skipped += 1
                    else:
                        # Update existing
                        for key, value in record.items():
                            setattr(existing, key, value)
                        existing.save()
                        updated += 1
                else:
                    import uuid
                    HCPRecord.objects.create(
                        hcp_id=f"HCP-{uuid.uuid4().hex[:8].upper()}",
                        **record
                    )
                    created += 1
            
            self.stdout.write(f"  Processed {min(i + batch_size, len(records))}/{len(records)}")
        
        return created, updated, skipped

# Usage:
# python manage.py run_etl data/hcp_records.csv --dry-run
# python manage.py run_etl data/hcp_records.csv --batch-size 500 --skip-duplicates
```

---

### Example 7: Django — Signals, Middleware, and Caching

```python
# ========== hcp/signals.py ==========
from django.db.models.signals import post_save, pre_save
from django.dispatch import receiver
from django.core.cache import cache
from .models import HCPRecord, Interaction
import logging

logger = logging.getLogger(__name__)

@receiver(pre_save, sender=HCPRecord)
def normalize_hcp_data(sender, instance, **kwargs):
    """Normalize data before saving."""
    instance.name = instance.name.strip().title()
    instance.region = instance.region.lower()
    instance.email = instance.email.lower()

@receiver(post_save, sender=HCPRecord)
def invalidate_hcp_cache(sender, instance, **kwargs):
    """Invalidate cache when HCP record changes."""
    cache.delete(f"hcp:{instance.hcp_id}")
    cache.delete(f"hcp_list:region:{instance.region}")
    cache.delete("hcp_stats")

@receiver(post_save, sender=Interaction)
def update_interaction_stats(sender, instance, created, **kwargs):
    """Update cached stats when new interaction is added."""
    if created:
        cache.delete(f"hcp:{instance.hcp.hcp_id}")
        cache.delete("hcp_stats")
        logger.info(f"New interaction recorded for {instance.hcp.hcp_id}")


# ========== hcp/middleware.py ==========
import time
import logging
import uuid

logger = logging.getLogger('api.request')

class RequestLoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        request.id = str(uuid.uuid4())[:8]
        request.start_time = time.time()
        
        response = self.get_response(request)
        
        duration = time.time() - request.start_time
        logger.info(
            f"[{request.id}] {request.method} {request.path} "
            f"→ {response.status_code} ({duration:.3f}s)"
        )
        
        response['X-Request-ID'] = request.id
        response['X-Response-Time'] = f"{duration:.3f}s"
        return response

class APIVersionMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Extract API version from URL or header
        if '/api/v1/' in request.path:
            request.api_version = 'v1'
        elif '/api/v2/' in request.path:
            request.api_version = 'v2'
        else:
            request.api_version = request.headers.get('X-API-Version', 'v1')
        
        response = self.get_response(request)
        response['X-API-Version'] = request.api_version
        return response


# ========== hcp/caching.py ==========
from django.core.cache import cache
from functools import wraps

def cached_view(timeout=300, key_func=None):
    """Decorator for caching DRF view responses."""
    def decorator(view_func):
        @wraps(view_func)
        def wrapper(self, request, *args, **kwargs):
            # Build cache key
            if key_func:
                cache_key = key_func(request, *args, **kwargs)
            else:
                cache_key = f"view:{request.path}:{request.query_params.urlencode()}"
            
            # Try cache
            cached_response = cache.get(cache_key)
            if cached_response is not None:
                return Response(cached_response)
            
            # Execute view
            response = view_func(self, request, *args, **kwargs)
            
            # Cache successful responses
            if response.status_code == 200:
                cache.set(cache_key, response.data, timeout)
            
            return response
        return wrapper
    return decorator

# Usage in views.py:
# @action(detail=False, methods=['get'])
# @cached_view(timeout=600)
# def stats(self, request):
#     ...
```

---

## Part C: FastAPI vs Django — When to Use What

| Criteria | FastAPI | Django (DRF) |
|----------|---------|--------------|
| **Performance** | Async, very fast | Sync by default, slower |
| **Learning curve** | Easy if you know Python | Steeper, more conventions |
| **Best for** | Microservices, ML APIs, real-time | Full web apps, admin panels, CRUD |
| **ORM** | SQLAlchemy (flexible) | Django ORM (batteries included) |
| **Admin panel** | None built-in | Excellent built-in admin |
| **Auth** | DIY or libraries | Built-in auth system |
| **Validation** | Pydantic (automatic) | Serializers (explicit) |
| **WebSocket** | Native support | Channels (add-on) |
| **Auto docs** | Swagger + ReDoc (built-in) | DRF browsable API |
| **Ecosystem** | Growing | Massive, mature |
| **Testing** | TestClient (simple) | Django TestCase (comprehensive) |
| **Deployment** | Uvicorn/Gunicorn | Gunicorn/uWSGI |

### Use FastAPI when:
- Building microservices or ML model APIs
- Need async/WebSocket support
- Performance is critical
- Building a standalone API (no server-side rendering)
- Your team knows Python but not Django

### Use Django when:
- Building a full web application
- Need admin panel out of the box
- Complex auth/permissions (multi-tenant, role-based)
- Team already knows Django
- Need ORM migrations, forms, templates
- Building internal portals (like your TCS project)

---

## Part D: Common Interview Questions About APIs

### Q: "How would you handle API versioning?"
```
Three approaches:
1. URL path: /api/v1/users, /api/v2/users (most common, explicit)
2. Header: Accept: application/vnd.api.v2+json (cleaner URLs)
3. Query param: /api/users?version=2 (simplest but least RESTful)

I prefer URL-based because:
- Explicit and visible in logs
- Easy to route at load balancer level
- Clients can bookmark specific versions
- Can run v1 and v2 simultaneously on different services
```

### Q: "How do you handle pagination?"
```
Three patterns:
1. Offset-based: ?page=3&page_size=20
   - Simple, supports jumping to page N
   - Breaks if data changes between pages (missed/duplicate items)
   
2. Cursor-based: ?cursor=eyJpZCI6MTAwfQ==
   - Stable results even with inserts/deletes
   - Can't jump to arbitrary page
   - Best for infinite scroll / real-time feeds

3. Keyset: ?after_id=100&limit=20
   - Similar to cursor but with visible key
   - Fast (uses index seek, not offset)

I'd use:
- Offset for admin dashboards (users expect page numbers)
- Cursor for feeds, timelines, mobile apps
- Always return: total_count, has_next, next_cursor/page
```

### Q: "How do you design error responses?"
```json
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Human-readable summary",
        "details": [
            {"field": "email", "message": "Invalid email format"},
            {"field": "region", "message": "Must be one of: north, south, east, west"}
        ],
        "request_id": "abc-123",
        "documentation_url": "https://api.example.com/docs/errors#VALIDATION_ERROR"
    }
}

Rules:
- Always use proper HTTP status codes (don't return 200 with error body)
- Include machine-readable error code (for client-side handling)
- Include human-readable message (for developers)
- Include request_id (for debugging in logs)
- Never expose internal stack traces in production
```

### Q: "How do you secure an API?"
```
Layers:
1. Transport: HTTPS only (TLS 1.3)
2. Authentication: JWT tokens (short-lived access + refresh)
3. Authorization: RBAC or ABAC at endpoint level
4. Input validation: Pydantic/serializers validate all input
5. Rate limiting: Per-user, per-endpoint limits
6. CORS: Whitelist allowed origins
7. Headers: X-Content-Type-Options, X-Frame-Options, CSP
8. Logging: Audit log for sensitive operations
9. Dependencies: Keep packages updated (Dependabot)
10. Secrets: Environment variables, never in code
```

### Q: "How do you test APIs?"
```python
# FastAPI testing
from fastapi.testclient import TestClient
import pytest

@pytest.fixture
def client():
    return TestClient(app)

def test_create_hcp(client):
    response = client.post("/api/v1/hcp", json={
        "name": "Dr. Test",
        "specialty": "Cardiology",
        "region": "north",
        "email": "test@example.com"
    })
    assert response.status_code == 201
    assert response.json()["hcp_id"].startswith("HCP-")

def test_create_hcp_invalid_email(client):
    response = client.post("/api/v1/hcp", json={
        "name": "Dr. Test",
        "specialty": "Cardiology",
        "region": "north",
        "email": "invalid"
    })
    assert response.status_code == 422

def test_get_hcp_not_found(client):
    response = client.get("/api/v1/hcp/NONEXISTENT")
    assert response.status_code == 404

# Django testing
from rest_framework.test import APITestCase
from rest_framework import status

class HCPAPITest(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user('testuser', password='pass')
        self.client.force_authenticate(user=self.user)
    
    def test_create_hcp(self):
        data = {"name": "Dr. Test", "specialty": "Cardiology", 
                "region": "north", "email": "test@example.com"}
        response = self.client.post('/api/v1/hcp/', data)
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
    
    def test_list_hcps_pagination(self):
        # Create 25 records
        for i in range(25):
            HCPRecord.objects.create(hcp_id=f"HCP-{i}", name=f"Dr. {i}",
                                    specialty="General", region="north",
                                    email=f"dr{i}@test.com")
        
        response = self.client.get('/api/v1/hcp/?page=1&page_size=10')
        self.assertEqual(response.status_code, 200)
        self.assertEqual(len(response.data['results']), 10)
        self.assertEqual(response.data['count'], 25)
```
