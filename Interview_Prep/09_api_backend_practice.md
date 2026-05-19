# API & Backend Practice Problems

> FastAPI, Django, REST API design, authentication, and backend architecture problems.

---

## Problem 1: Design and Implement a REST API for ETL Job Management

> Build a complete API for managing ETL pipeline jobs — create, monitor, retry, cancel.

### Requirements:
- CRUD for pipeline jobs
- Status tracking (queued, running, success, failed)
- Retry failed jobs with configurable attempts
- Pagination for job listing
- Authentication via JWT

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from fastapi import FastAPI, HTTPException, Depends, Query, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime
from enum import Enum
import uuid
import jwt

app = FastAPI(title="ETL Pipeline Manager")
security = HTTPBearer()

# --- Models ---
class JobStatus(str, Enum):
    QUEUED = "queued"
    RUNNING = "running"
    SUCCESS = "success"
    FAILED = "failed"
    CANCELLED = "cancelled"
    RETRYING = "retrying"

class JobCreate(BaseModel):
    pipeline_name: str
    source: str
    destination: str
    schedule: Optional[str] = None
    config: dict = Field(default_factory=dict)

class JobResponse(BaseModel):
    job_id: str
    pipeline_name: str
    status: JobStatus
    source: str
    destination: str
    created_at: datetime
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None
    records_processed: int = 0
    error_message: Optional[str] = None
    retry_count: int = 0
    max_retries: int = 3

class PaginatedResponse(BaseModel):
    items: List[JobResponse]
    total: int
    page: int
    page_size: int
    has_next: bool

# --- In-memory store (use DB in production) ---
jobs_db: dict = {}

# --- Auth ---
SECRET_KEY = "your-secret-key"

def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)):
    try:
        payload = jwt.decode(credentials.credentials, SECRET_KEY, algorithms=["HS256"])
        return payload
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

# --- Endpoints ---
@app.post("/api/v1/jobs", response_model=JobResponse, status_code=status.HTTP_201_CREATED)
async def create_job(job: JobCreate, user=Depends(verify_token)):
    job_id = str(uuid.uuid4())
    new_job = JobResponse(
        job_id=job_id,
        pipeline_name=job.pipeline_name,
        status=JobStatus.QUEUED,
        source=job.source,
        destination=job.destination,
        created_at=datetime.utcnow(),
    )
    jobs_db[job_id] = new_job
    return new_job

@app.get("/api/v1/jobs", response_model=PaginatedResponse)
async def list_jobs(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    status_filter: Optional[JobStatus] = None,
    pipeline_name: Optional[str] = None,
    user=Depends(verify_token)
):
    filtered = list(jobs_db.values())
    
    if status_filter:
        filtered = [j for j in filtered if j.status == status_filter]
    if pipeline_name:
        filtered = [j for j in filtered if j.pipeline_name == pipeline_name]
    
    total = len(filtered)
    start = (page - 1) * page_size
    end = start + page_size
    items = filtered[start:end]
    
    return PaginatedResponse(
        items=items,
        total=total,
        page=page,
        page_size=page_size,
        has_next=end < total
    )

@app.get("/api/v1/jobs/{job_id}", response_model=JobResponse)
async def get_job(job_id: str, user=Depends(verify_token)):
    if job_id not in jobs_db:
        raise HTTPException(status_code=404, detail=f"Job {job_id} not found")
    return jobs_db[job_id]

@app.post("/api/v1/jobs/{job_id}/retry", response_model=JobResponse)
async def retry_job(job_id: str, user=Depends(verify_token)):
    if job_id not in jobs_db:
        raise HTTPException(status_code=404, detail=f"Job {job_id} not found")
    
    job = jobs_db[job_id]
    if job.status != JobStatus.FAILED:
        raise HTTPException(status_code=400, detail="Only failed jobs can be retried")
    if job.retry_count >= job.max_retries:
        raise HTTPException(status_code=400, detail="Max retries exceeded")
    
    job.status = JobStatus.RETRYING
    job.retry_count += 1
    job.error_message = None
    return job

@app.post("/api/v1/jobs/{job_id}/cancel", response_model=JobResponse)
async def cancel_job(job_id: str, user=Depends(verify_token)):
    if job_id not in jobs_db:
        raise HTTPException(status_code=404, detail=f"Job {job_id} not found")
    
    job = jobs_db[job_id]
    if job.status in [JobStatus.SUCCESS, JobStatus.CANCELLED]:
        raise HTTPException(status_code=400, detail=f"Cannot cancel job in {job.status} state")
    
    job.status = JobStatus.CANCELLED
    job.completed_at = datetime.utcnow()
    return job

@app.get("/api/v1/jobs/stats/summary")
async def job_stats(user=Depends(verify_token)):
    all_jobs = list(jobs_db.values())
    return {
        "total": len(all_jobs),
        "by_status": {
            status.value: len([j for j in all_jobs if j.status == status])
            for status in JobStatus
        },
        "avg_processing_time_seconds": None,  # Calculate from completed jobs
        "failure_rate": len([j for j in all_jobs if j.status == JobStatus.FAILED]) / max(len(all_jobs), 1)
    }

# --- Health check (no auth) ---
@app.get("/health")
async def health():
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}
```

**Interview tip:** Notice: proper HTTP status codes, pagination, filtering, state machine validation (can't retry a successful job), versioned API path (`/api/v1/`), health endpoint.
</details>

---

## Problem 2: Implement Rate-Limited API with Redis

> Build a FastAPI middleware that rate-limits requests per user with sliding window.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
import time
import redis
from typing import Tuple

app = FastAPI()
redis_client = redis.Redis(host="localhost", port=6379, db=0)

class RateLimiter:
    def __init__(self, requests_per_minute: int = 60, burst_size: int = 10):
        self.requests_per_minute = requests_per_minute
        self.burst_size = burst_size
        self.window_seconds = 60

    def is_allowed(self, user_id: str) -> Tuple[bool, dict]:
        key = f"rate_limit:{user_id}"
        now = time.time()
        window_start = now - self.window_seconds

        pipe = redis_client.pipeline()
        # Remove old entries outside window
        pipe.zremrangebyscore(key, 0, window_start)
        # Count current requests in window
        pipe.zcard(key)
        # Add current request
        pipe.zadd(key, {f"{now}": now})
        # Set expiry on key
        pipe.expire(key, self.window_seconds)
        
        results = pipe.execute()
        current_count = results[1]

        remaining = max(0, self.requests_per_minute - current_count - 1)
        headers = {
            "X-RateLimit-Limit": str(self.requests_per_minute),
            "X-RateLimit-Remaining": str(remaining),
            "X-RateLimit-Reset": str(int(now + self.window_seconds)),
        }

        if current_count >= self.requests_per_minute:
            return False, headers
        return True, headers

rate_limiter = RateLimiter(requests_per_minute=100)

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    # Extract user identity (from JWT, API key, or IP)
    user_id = request.headers.get("X-API-Key", request.client.host)
    
    allowed, headers = rate_limiter.is_allowed(user_id)
    
    if not allowed:
        return JSONResponse(
            status_code=429,
            content={"error": "Rate limit exceeded", "retry_after": 60},
            headers={**headers, "Retry-After": "60"}
        )
    
    response = await call_next(request)
    for key, value in headers.items():
        response.headers[key] = value
    return response

# Tiered rate limiting (different limits for different plans)
class TieredRateLimiter:
    TIERS = {
        "free": {"requests_per_minute": 20, "daily_limit": 1000},
        "pro": {"requests_per_minute": 100, "daily_limit": 50000},
        "enterprise": {"requests_per_minute": 1000, "daily_limit": None},
    }
    
    def get_limits(self, user_tier: str) -> dict:
        return self.TIERS.get(user_tier, self.TIERS["free"])
```

**Interview tip:** Mention: Why sorted set (ZSET) in Redis — allows sliding window with range removal. Headers (X-RateLimit-*) follow industry standard. 429 status code is correct for rate limiting.
</details>

---

## Problem 3: Implement WebSocket with Authentication & Rooms

> Extend your chat app with proper JWT auth, room management, and message persistence.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Query, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from typing import Dict, Set, List, Optional
from datetime import datetime
from pydantic import BaseModel
import json
import jwt
import asyncio

app = FastAPI()

# --- Models ---
class Message(BaseModel):
    id: str
    room_id: str
    sender_id: str
    content: str
    message_type: str = "text"  # text, image, system
    timestamp: datetime
    delivered: bool = False
    read_by: List[str] = []

# --- Connection Manager ---
class ConnectionManager:
    def __init__(self):
        self.connections: Dict[str, WebSocket] = {}
        self.rooms: Dict[str, Set[str]] = {}
        self.user_rooms: Dict[str, Set[str]] = {}
        self.message_store: Dict[str, List[Message]] = {}  # room_id -> messages
        self.offline_queue: Dict[str, List[Message]] = {}  # user_id -> pending messages

    async def connect(self, websocket: WebSocket, user_id: str) -> bool:
        await websocket.accept()
        self.connections[user_id] = websocket
        
        # Deliver queued messages
        if user_id in self.offline_queue:
            for msg in self.offline_queue[user_id]:
                await self.send_to_user(user_id, msg.dict())
            del self.offline_queue[user_id]
        
        # Notify rooms of user coming online
        for room_id in self.user_rooms.get(user_id, set()):
            await self.broadcast_to_room(room_id, {
                "type": "presence",
                "user_id": user_id,
                "status": "online"
            }, exclude=user_id)
        
        return True

    def disconnect(self, user_id: str):
        self.connections.pop(user_id, None)
        # Don't remove from rooms — they rejoin on reconnect

    async def send_to_user(self, user_id: str, data: dict):
        if user_id in self.connections:
            try:
                await self.connections[user_id].send_json(data)
                return True
            except Exception:
                self.disconnect(user_id)
                return False
        else:
            # Queue for offline delivery
            msg = Message(**data) if "content" in data else None
            if msg:
                self.offline_queue.setdefault(user_id, []).append(msg)
            return False

    async def broadcast_to_room(self, room_id: str, data: dict, exclude: str = None):
        members = self.rooms.get(room_id, set())
        tasks = []
        for user_id in members:
            if user_id != exclude:
                tasks.append(self.send_to_user(user_id, data))
        await asyncio.gather(*tasks)

    def join_room(self, user_id: str, room_id: str):
        self.rooms.setdefault(room_id, set()).add(user_id)
        self.user_rooms.setdefault(user_id, set()).add(room_id)

    def leave_room(self, user_id: str, room_id: str):
        self.rooms.get(room_id, set()).discard(user_id)
        self.user_rooms.get(user_id, set()).discard(room_id)

    def get_room_members(self, room_id: str) -> List[dict]:
        members = self.rooms.get(room_id, set())
        return [
            {"user_id": uid, "online": uid in self.connections}
            for uid in members
        ]

    def store_message(self, room_id: str, message: Message):
        self.message_store.setdefault(room_id, []).append(message)
        # Keep last 1000 messages per room in memory
        if len(self.message_store[room_id]) > 1000:
            self.message_store[room_id] = self.message_store[room_id][-1000:]

    def get_history(self, room_id: str, limit: int = 50, before: str = None) -> List[Message]:
        messages = self.message_store.get(room_id, [])
        if before:
            messages = [m for m in messages if m.id < before]
        return messages[-limit:]

manager = ConnectionManager()

# --- Auth ---
SECRET = "your-jwt-secret"

def verify_ws_token(token: str) -> Optional[dict]:
    try:
        return jwt.decode(token, SECRET, algorithms=["HS256"])
    except jwt.InvalidTokenError:
        return None

# --- WebSocket Endpoint ---
@app.websocket("/ws/chat")
async def chat_endpoint(websocket: WebSocket, token: str = Query(...)):
    user = verify_ws_token(token)
    if not user:
        await websocket.close(code=4001, reason="Invalid token")
        return
    
    user_id = user["user_id"]
    await manager.connect(websocket, user_id)
    
    try:
        while True:
            raw = await websocket.receive_text()
            data = json.loads(raw)
            
            match data.get("type"):
                case "join_room":
                    room_id = data["room_id"]
                    manager.join_room(user_id, room_id)
                    
                    # Send room history
                    history = manager.get_history(room_id)
                    await manager.send_to_user(user_id, {
                        "type": "room_history",
                        "room_id": room_id,
                        "messages": [m.dict() for m in history],
                        "members": manager.get_room_members(room_id)
                    })
                    
                    # Notify room
                    await manager.broadcast_to_room(room_id, {
                        "type": "user_joined",
                        "user_id": user_id,
                        "room_id": room_id,
                        "timestamp": datetime.utcnow().isoformat()
                    }, exclude=user_id)
                
                case "message":
                    import uuid
                    msg = Message(
                        id=str(uuid.uuid4()),
                        room_id=data["room_id"],
                        sender_id=user_id,
                        content=data["content"],
                        message_type=data.get("message_type", "text"),
                        timestamp=datetime.utcnow()
                    )
                    manager.store_message(data["room_id"], msg)
                    
                    await manager.broadcast_to_room(data["room_id"], {
                        "type": "message",
                        **msg.dict(),
                        "timestamp": msg.timestamp.isoformat()
                    })
                
                case "typing":
                    await manager.broadcast_to_room(data["room_id"], {
                        "type": "typing",
                        "user_id": user_id,
                        "room_id": data["room_id"]
                    }, exclude=user_id)
                
                case "read_receipt":
                    await manager.broadcast_to_room(data["room_id"], {
                        "type": "read_receipt",
                        "user_id": user_id,
                        "message_id": data["message_id"],
                        "room_id": data["room_id"]
                    }, exclude=user_id)
                
                case "leave_room":
                    manager.leave_room(user_id, data["room_id"])
                    await manager.broadcast_to_room(data["room_id"], {
                        "type": "user_left",
                        "user_id": user_id,
                        "room_id": data["room_id"]
                    })
    
    except WebSocketDisconnect:
        manager.disconnect(user_id)
        for room_id in manager.user_rooms.get(user_id, set()):
            await manager.broadcast_to_room(room_id, {
                "type": "presence",
                "user_id": user_id,
                "status": "offline"
            })
```
</details>

---

## Problem 4: Implement a Background Task Queue with FastAPI

> Build an async task queue for long-running operations (like ETL job triggers, report generation).

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel
from typing import Dict, Optional, Callable
from datetime import datetime
from enum import Enum
import asyncio
import uuid
import traceback

app = FastAPI()

class TaskStatus(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"

class TaskInfo(BaseModel):
    task_id: str
    task_type: str
    status: TaskStatus
    created_at: datetime
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None
    result: Optional[dict] = None
    error: Optional[str] = None
    progress: float = 0.0

# Task registry
task_store: Dict[str, TaskInfo] = {}
task_handlers: Dict[str, Callable] = {}

def register_task(task_type: str):
    """Decorator to register task handlers."""
    def decorator(func):
        task_handlers[task_type] = func
        return func
    return decorator

# --- Task definitions ---
@register_task("generate_report")
async def generate_report(task_id: str, params: dict):
    """Simulate long-running report generation."""
    total_steps = 10
    for i in range(total_steps):
        await asyncio.sleep(1)  # Simulate work
        task_store[task_id].progress = (i + 1) / total_steps * 100
    
    return {"report_url": f"/reports/{task_id}.pdf", "rows_processed": 125000}

@register_task("trigger_etl")
async def trigger_etl(task_id: str, params: dict):
    """Trigger an ETL pipeline run."""
    pipeline = params.get("pipeline_name", "default")
    await asyncio.sleep(5)  # Simulate ETL execution
    return {"pipeline": pipeline, "records_processed": 1000000, "status": "success"}

@register_task("export_data")
async def export_data(task_id: str, params: dict):
    """Export data to S3."""
    format_type = params.get("format", "parquet")
    await asyncio.sleep(3)
    return {"s3_path": f"s3://exports/{task_id}.{format_type}", "size_mb": 256}

# --- Task executor ---
async def execute_task(task_id: str, task_type: str, params: dict):
    task = task_store[task_id]
    task.status = TaskStatus.RUNNING
    task.started_at = datetime.utcnow()
    
    try:
        handler = task_handlers[task_type]
        result = await handler(task_id, params)
        task.status = TaskStatus.COMPLETED
        task.result = result
    except Exception as e:
        task.status = TaskStatus.FAILED
        task.error = traceback.format_exc()
    finally:
        task.completed_at = datetime.utcnow()
        task.progress = 100.0

# --- API Endpoints ---
class TaskRequest(BaseModel):
    task_type: str
    params: dict = {}

@app.post("/api/v1/tasks", status_code=202)
async def submit_task(request: TaskRequest, background_tasks: BackgroundTasks):
    if request.task_type not in task_handlers:
        from fastapi import HTTPException
        raise HTTPException(400, f"Unknown task type: {request.task_type}")
    
    task_id = str(uuid.uuid4())
    task_info = TaskInfo(
        task_id=task_id,
        task_type=request.task_type,
        status=TaskStatus.PENDING,
        created_at=datetime.utcnow()
    )
    task_store[task_id] = task_info
    
    background_tasks.add_task(execute_task, task_id, request.task_type, request.params)
    
    return {
        "task_id": task_id,
        "status": "accepted",
        "poll_url": f"/api/v1/tasks/{task_id}"
    }

@app.get("/api/v1/tasks/{task_id}")
async def get_task_status(task_id: str):
    if task_id not in task_store:
        from fastapi import HTTPException
        raise HTTPException(404, "Task not found")
    return task_store[task_id]

@app.get("/api/v1/tasks")
async def list_tasks(status_filter: Optional[TaskStatus] = None, limit: int = 20):
    tasks = list(task_store.values())
    if status_filter:
        tasks = [t for t in tasks if t.status == status_filter]
    return sorted(tasks, key=lambda t: t.created_at, reverse=True)[:limit]

# Usage:
# POST /api/v1/tasks {"task_type": "generate_report", "params": {"date_range": "2025-05"}}
# → 202 {"task_id": "abc-123", "poll_url": "/api/v1/tasks/abc-123"}
# GET /api/v1/tasks/abc-123
# → {"status": "running", "progress": 40.0}
# → {"status": "completed", "result": {"report_url": "..."}}
```

**Interview tip:** 202 Accepted (not 200/201) is correct for async operations. Always provide a poll URL. Discuss alternatives: webhooks for completion notification, SSE for progress streaming.
</details>

---

## Problem 5: Implement API Versioning and Backward Compatibility

> Design an API that supports multiple versions simultaneously.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from fastapi import FastAPI, Request, APIRouter, Header
from fastapi.responses import JSONResponse
from typing import Optional
from pydantic import BaseModel
from datetime import datetime

app = FastAPI()

# --- Approach 1: URL-based versioning (recommended) ---
v1_router = APIRouter(prefix="/api/v1")
v2_router = APIRouter(prefix="/api/v2")

# V1 response format
class JobResponseV1(BaseModel):
    id: str
    name: str
    status: str
    created: str  # string timestamp

# V2 response format (improved)
class JobResponseV2(BaseModel):
    job_id: str           # renamed from 'id'
    pipeline_name: str    # renamed from 'name'
    status: str
    created_at: datetime  # proper datetime
    metadata: dict = {}   # new field

# Shared data layer
def get_job_from_db(job_id: str) -> dict:
    return {
        "job_id": job_id,
        "pipeline_name": "hcp_daily_etl",
        "status": "running",
        "created_at": datetime.utcnow(),
        "metadata": {"dpu_count": 6, "source": "aurora"}
    }

@v1_router.get("/jobs/{job_id}", response_model=JobResponseV1)
async def get_job_v1(job_id: str):
    raw = get_job_from_db(job_id)
    # Transform to V1 format
    return JobResponseV1(
        id=raw["job_id"],
        name=raw["pipeline_name"],
        status=raw["status"],
        created=raw["created_at"].isoformat()
    )

@v2_router.get("/jobs/{job_id}", response_model=JobResponseV2)
async def get_job_v2(job_id: str):
    raw = get_job_from_db(job_id)
    return JobResponseV2(**raw)

app.include_router(v1_router)
app.include_router(v2_router)

# --- Approach 2: Header-based versioning ---
@app.get("/api/jobs/{job_id}")
async def get_job_versioned(job_id: str, api_version: str = Header(default="2")):
    raw = get_job_from_db(job_id)
    
    if api_version == "1":
        return {
            "id": raw["job_id"],
            "name": raw["pipeline_name"],
            "status": raw["status"],
            "created": raw["created_at"].isoformat()
        }
    else:  # v2 (default)
        return raw

# --- Deprecation middleware ---
DEPRECATED_ENDPOINTS = {
    "/api/v1/jobs": "2025-09-01"  # Sunset date
}

@app.middleware("http")
async def deprecation_warning(request: Request, call_next):
    response = await call_next(request)
    
    path = request.url.path
    for deprecated_path, sunset_date in DEPRECATED_ENDPOINTS.items():
        if path.startswith(deprecated_path):
            response.headers["Deprecation"] = "true"
            response.headers["Sunset"] = sunset_date
            response.headers["Link"] = '</api/v2/jobs>; rel="successor-version"'
    
    return response
```

**Interview tip:** URL-based versioning is most common and explicit. Always provide deprecation headers. Keep internal data model separate from API response models (allows independent evolution).
</details>

---

## Problem 6: Implement a Webhook System

> Build a webhook delivery system for notifying external systems when ETL jobs complete.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel, HttpUrl
from typing import Dict, List, Optional
from datetime import datetime
import httpx
import hashlib
import hmac
import json
import uuid
import asyncio

app = FastAPI()

# --- Models ---
class WebhookRegistration(BaseModel):
    url: HttpUrl
    events: List[str]  # ["job.completed", "job.failed"]
    secret: Optional[str] = None  # For signature verification

class WebhookDelivery(BaseModel):
    delivery_id: str
    webhook_id: str
    event: str
    payload: dict
    status_code: Optional[int] = None
    delivered_at: Optional[datetime] = None
    attempts: int = 0
    next_retry: Optional[datetime] = None

# --- Storage ---
webhooks: Dict[str, dict] = {}
deliveries: List[WebhookDelivery] = []

# --- Webhook signature ---
def sign_payload(payload: dict, secret: str) -> str:
    payload_bytes = json.dumps(payload, sort_keys=True).encode()
    return hmac.new(secret.encode(), payload_bytes, hashlib.sha256).hexdigest()

# --- Delivery engine ---
async def deliver_webhook(webhook_id: str, event: str, payload: dict):
    webhook = webhooks.get(webhook_id)
    if not webhook:
        return
    
    delivery_id = str(uuid.uuid4())
    delivery = WebhookDelivery(
        delivery_id=delivery_id,
        webhook_id=webhook_id,
        event=event,
        payload=payload
    )
    deliveries.append(delivery)
    
    headers = {
        "Content-Type": "application/json",
        "X-Webhook-ID": webhook_id,
        "X-Delivery-ID": delivery_id,
        "X-Event": event,
    }
    
    if webhook.get("secret"):
        headers["X-Signature-256"] = f"sha256={sign_payload(payload, webhook['secret'])}"
    
    # Retry with exponential backoff
    max_attempts = 5
    for attempt in range(max_attempts):
        delivery.attempts = attempt + 1
        try:
            async with httpx.AsyncClient(timeout=10) as client:
                response = await client.post(
                    str(webhook["url"]),
                    json=payload,
                    headers=headers
                )
                delivery.status_code = response.status_code
                
                if 200 <= response.status_code < 300:
                    delivery.delivered_at = datetime.utcnow()
                    return  # Success
                
                if response.status_code >= 400 and response.status_code < 500:
                    return  # Client error, don't retry
                    
        except Exception as e:
            delivery.status_code = 0
        
        # Exponential backoff: 1s, 2s, 4s, 8s, 16s
        if attempt < max_attempts - 1:
            await asyncio.sleep(2 ** attempt)

# --- Dispatch events ---
async def dispatch_event(event: str, payload: dict):
    """Dispatch event to all registered webhooks."""
    tasks = []
    for webhook_id, webhook in webhooks.items():
        if event in webhook.get("events", []) or "*" in webhook.get("events", []):
            tasks.append(deliver_webhook(webhook_id, event, payload))
    
    if tasks:
        await asyncio.gather(*tasks)

# --- API ---
@app.post("/api/v1/webhooks", status_code=201)
async def register_webhook(registration: WebhookRegistration):
    webhook_id = str(uuid.uuid4())
    webhooks[webhook_id] = {
        "id": webhook_id,
        "url": str(registration.url),
        "events": registration.events,
        "secret": registration.secret,
        "created_at": datetime.utcnow().isoformat()
    }
    return {"webhook_id": webhook_id, "url": str(registration.url)}

@app.delete("/api/v1/webhooks/{webhook_id}")
async def delete_webhook(webhook_id: str):
    if webhook_id not in webhooks:
        raise HTTPException(404, "Webhook not found")
    del webhooks[webhook_id]
    return {"deleted": True}

@app.get("/api/v1/webhooks/{webhook_id}/deliveries")
async def list_deliveries(webhook_id: str, limit: int = 20):
    return [d for d in deliveries if d.webhook_id == webhook_id][-limit:]

# --- Trigger (called internally when jobs complete) ---
@app.post("/internal/events")
async def trigger_event(event: str, payload: dict, background_tasks: BackgroundTasks):
    background_tasks.add_task(dispatch_event, event, payload)
    return {"dispatched": True}

# Example: When ETL job completes
# dispatch_event("job.completed", {
#     "job_id": "abc-123",
#     "pipeline": "hcp_daily",
#     "records_processed": 125000000,
#     "duration_seconds": 1800,
#     "completed_at": "2025-05-19T10:30:00Z"
# })
```

**Interview tip:** Key concepts: HMAC signature for security (receiver verifies payload authenticity), exponential backoff for reliability, delivery log for debugging, idempotent receivers (webhook might be delivered more than once).
</details>

---

## Problem 7: Implement a CQRS Pattern (Command Query Responsibility Segregation)

> Separate read and write models for your HCP data system.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Dict, List, Optional
from datetime import datetime
from abc import ABC, abstractmethod
import uuid

app = FastAPI()

# --- Events ---
class Event(BaseModel):
    event_id: str = ""
    event_type: str
    aggregate_id: str
    data: dict
    timestamp: datetime = None
    
    def __init__(self, **data):
        super().__init__(**data)
        if not self.event_id:
            self.event_id = str(uuid.uuid4())
        if not self.timestamp:
            self.timestamp = datetime.utcnow()

# --- Event Store ---
event_store: List[Event] = []

# --- Commands (Write Side) ---
class CreateHCPCommand(BaseModel):
    name: str
    specialty: str
    region: str

class UpdateHCPCommand(BaseModel):
    name: Optional[str] = None
    specialty: Optional[str] = None
    region: Optional[str] = None

# --- Command Handlers ---
@app.post("/api/v1/commands/create-hcp", status_code=201)
async def handle_create_hcp(cmd: CreateHCPCommand):
    hcp_id = str(uuid.uuid4())
    
    event = Event(
        event_type="HCPCreated",
        aggregate_id=hcp_id,
        data={"name": cmd.name, "specialty": cmd.specialty, "region": cmd.region}
    )
    event_store.append(event)
    
    # Update read model
    _update_read_model(event)
    
    return {"hcp_id": hcp_id, "event_id": event.event_id}

@app.post("/api/v1/commands/update-hcp/{hcp_id}")
async def handle_update_hcp(hcp_id: str, cmd: UpdateHCPCommand):
    changes = {k: v for k, v in cmd.dict().items() if v is not None}
    if not changes:
        raise HTTPException(400, "No changes provided")
    
    event = Event(
        event_type="HCPUpdated",
        aggregate_id=hcp_id,
        data=changes
    )
    event_store.append(event)
    _update_read_model(event)
    
    return {"hcp_id": hcp_id, "event_id": event.event_id}

# --- Read Model (Query Side) ---
# Optimized for reads — denormalized, pre-computed
read_model: Dict[str, dict] = {}  # hcp_id -> current state
region_index: Dict[str, List[str]] = {}  # region -> [hcp_ids]
specialty_index: Dict[str, List[str]] = {}  # specialty -> [hcp_ids]

def _update_read_model(event: Event):
    """Project event into read model."""
    hcp_id = event.aggregate_id
    
    if event.event_type == "HCPCreated":
        read_model[hcp_id] = {
            "hcp_id": hcp_id,
            **event.data,
            "created_at": event.timestamp.isoformat(),
            "updated_at": event.timestamp.isoformat(),
            "version": 1
        }
        # Update indexes
        region_index.setdefault(event.data["region"], []).append(hcp_id)
        specialty_index.setdefault(event.data["specialty"], []).append(hcp_id)
    
    elif event.event_type == "HCPUpdated":
        if hcp_id in read_model:
            old = read_model[hcp_id]
            # Update indexes if region/specialty changed
            if "region" in event.data:
                region_index.get(old["region"], []).remove(hcp_id) if hcp_id in region_index.get(old["region"], []) else None
                region_index.setdefault(event.data["region"], []).append(hcp_id)
            
            read_model[hcp_id].update(event.data)
            read_model[hcp_id]["updated_at"] = event.timestamp.isoformat()
            read_model[hcp_id]["version"] += 1

# --- Query Handlers ---
@app.get("/api/v1/queries/hcp/{hcp_id}")
async def query_hcp(hcp_id: str):
    if hcp_id not in read_model:
        raise HTTPException(404, "HCP not found")
    return read_model[hcp_id]

@app.get("/api/v1/queries/hcp/by-region/{region}")
async def query_by_region(region: str):
    hcp_ids = region_index.get(region, [])
    return [read_model[hcp_id] for hcp_id in hcp_ids]

@app.get("/api/v1/queries/hcp/by-specialty/{specialty}")
async def query_by_specialty(specialty: str):
    hcp_ids = specialty_index.get(specialty, [])
    return [read_model[hcp_id] for hcp_id in hcp_ids]

# --- Rebuild read model from events (event replay) ---
@app.post("/api/v1/admin/rebuild-read-model")
async def rebuild_read_model():
    read_model.clear()
    region_index.clear()
    specialty_index.clear()
    
    for event in event_store:
        _update_read_model(event)
    
    return {"rebuilt": True, "total_events": len(event_store), "total_records": len(read_model)}

# --- Event history for an aggregate ---
@app.get("/api/v1/queries/hcp/{hcp_id}/history")
async def query_hcp_history(hcp_id: str):
    events = [e for e in event_store if e.aggregate_id == hcp_id]
    return [{"event_type": e.event_type, "data": e.data, "timestamp": e.timestamp} for e in events]
```

**Interview tip:** CQRS separates read/write concerns. Write model ensures correctness (events are immutable). Read model is optimized for specific query patterns. Can rebuild read model anytime by replaying events. Used in: healthcare records, financial systems, audit-heavy domains.
</details>

---

## Summary — API & Backend Problems

| # | Problem | Key Concepts | Difficulty |
|---|---------|-------------|-----------|
| 1 | ETL Job Management API | REST design, pagination, state machine, JWT | Medium |
| 2 | Rate Limiter with Redis | Sliding window, middleware, headers | Medium |
| 3 | WebSocket Chat (Full) | Auth, rooms, persistence, offline queue | Hard |
| 4 | Background Task Queue | Async processing, polling, progress | Medium |
| 5 | API Versioning | URL/Header versioning, deprecation, backward compat | Medium |
| 6 | Webhook System | HMAC signatures, retry, delivery guarantee | Hard |
| 7 | CQRS Pattern | Event sourcing, read/write separation, indexing | Hard |

**Practice order:** 1 → 4 → 2 → 5 → 6 → 3 → 7
