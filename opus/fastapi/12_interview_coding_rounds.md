# FastAPI Interview Coding Rounds — Senior Level

## 1. URL Shortener (System Design + Code)

### Requirements
```
- Shorten URL: POST /shorten → returns short code
- Redirect: GET /{code} → 301 redirect to original URL
- Analytics: GET /{code}/stats → click count, last accessed
- Rate limited: 10 shortens/minute per user
```

### Implementation
```python
from fastapi import FastAPI, HTTPException, Depends, Request
from fastapi.responses import RedirectResponse
from pydantic import BaseModel, HttpUrl
from sqlalchemy import select, update
from datetime import datetime
import hashlib
import string

app = FastAPI()

ALPHABET = string.ascii_letters + string.digits  # base62


def encode_base62(num: int) -> str:
    if num == 0:
        return ALPHABET[0]
    result = []
    while num:
        result.append(ALPHABET[num % 62])
        num //= 62
    return "".join(reversed(result))


class ShortenRequest(BaseModel):
    url: HttpUrl
    custom_code: str | None = None


class URLMapping(Base):
    __tablename__ = "url_mappings"
    
    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    code: Mapped[str] = mapped_column(String(10), unique=True, index=True)
    original_url: Mapped[str] = mapped_column(Text)
    user_id: Mapped[str] = mapped_column(String(36), nullable=True)
    clicks: Mapped[int] = mapped_column(Integer, default=0)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    last_accessed: Mapped[datetime] = mapped_column(DateTime, nullable=True)


@app.post("/shorten", status_code=201)
async def shorten_url(
    data: ShortenRequest,
    db=Depends(get_db),
    user=Depends(get_current_user_optional),
):
    if data.custom_code:
        # Check if custom code is taken
        existing = await db.execute(
            select(URLMapping).where(URLMapping.code == data.custom_code)
        )
        if existing.scalar_one_or_none():
            raise HTTPException(409, "Code already taken")
        code = data.custom_code
    else:
        # Generate unique code
        mapping = URLMapping(original_url=str(data.url), user_id=str(user.id) if user else None)
        db.add(mapping)
        await db.flush()  # get auto-increment ID
        code = encode_base62(mapping.id)
        mapping.code = code
    
    await db.commit()
    
    return {"short_url": f"https://short.io/{code}", "code": code}


@app.get("/{code}")
async def redirect(code: str, db=Depends(get_db)):
    # Try cache first
    redis = app.state.redis
    cached_url = await redis.get(f"url:{code}")
    
    if cached_url:
        # Increment click async (fire-and-forget)
        await redis.incr(f"clicks:{code}")
        return RedirectResponse(url=cached_url, status_code=301)
    
    # DB lookup
    result = await db.execute(select(URLMapping).where(URLMapping.code == code))
    mapping = result.scalar_one_or_none()
    
    if not mapping:
        raise HTTPException(404, "Short URL not found")
    
    # Cache for future lookups
    await redis.set(f"url:{code}", mapping.original_url, ex=3600)
    
    # Update stats
    mapping.clicks += 1
    mapping.last_accessed = datetime.utcnow()
    await db.commit()
    
    return RedirectResponse(url=mapping.original_url, status_code=301)


@app.get("/{code}/stats")
async def url_stats(code: str, db=Depends(get_db)):
    result = await db.execute(select(URLMapping).where(URLMapping.code == code))
    mapping = result.scalar_one_or_none()
    
    if not mapping:
        raise HTTPException(404, "Not found")
    
    return {
        "code": mapping.code,
        "original_url": mapping.original_url,
        "clicks": mapping.clicks,
        "created_at": mapping.created_at,
        "last_accessed": mapping.last_accessed,
    }
```

---

## 2. Rate Limiter Service (Common Interview Problem)

```python
"""
Design a rate limiter that supports:
  - Per-user limits
  - Per-endpoint limits
  - Sliding window algorithm
  - Returns remaining quota in headers
"""

from fastapi import FastAPI, Request, HTTPException, Depends
import time
import json

app = FastAPI()


class RateLimiterConfig:
    RULES = {
        "POST /api/orders": {"limit": 10, "window": 60},
        "POST /api/auth/login": {"limit": 5, "window": 300},
        "GET /api/products": {"limit": 100, "window": 60},
        "DEFAULT": {"limit": 60, "window": 60},
    }


class TokenBucketLimiter:
    """Token bucket implementation using Redis."""
    
    def __init__(self, redis):
        self.redis = redis
    
    async def is_allowed(self, key: str, max_tokens: int, refill_rate: float) -> tuple[bool, int]:
        """
        Token bucket: starts full, drains with requests, refills over time.
        
        max_tokens: bucket capacity
        refill_rate: tokens added per second
        """
        now = time.time()
        bucket_key = f"bucket:{key}"
        
        # Lua script for atomic token bucket
        lua_script = """
        local key = KEYS[1]
        local max_tokens = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        
        local bucket = redis.call('hmget', key, 'tokens', 'last_refill')
        local tokens = tonumber(bucket[1]) or max_tokens
        local last_refill = tonumber(bucket[2]) or now
        
        -- Refill tokens based on elapsed time
        local elapsed = now - last_refill
        tokens = math.min(max_tokens, tokens + (elapsed * refill_rate))
        
        local allowed = 0
        if tokens >= 1 then
            tokens = tokens - 1
            allowed = 1
        end
        
        redis.call('hmset', key, 'tokens', tokens, 'last_refill', now)
        redis.call('expire', key, math.ceil(max_tokens / refill_rate) + 1)
        
        return {allowed, math.floor(tokens)}
        """
        
        result = await self.redis.eval(lua_script, 1, bucket_key, max_tokens, refill_rate, now)
        return bool(result[0]), int(result[1])


class RateLimitMiddleware:
    async def __call__(self, request: Request, call_next):
        # Build rate limit key
        user = getattr(request.state, "user", None)
        identifier = str(user.id) if user else request.client.host
        endpoint = f"{request.method} {request.url.path}"
        
        # Get rule for this endpoint
        rule = RateLimiterConfig.RULES.get(endpoint, RateLimiterConfig.RULES["DEFAULT"])
        key = f"{identifier}:{endpoint}"
        
        limiter = TokenBucketLimiter(request.app.state.redis)
        refill_rate = rule["limit"] / rule["window"]
        
        allowed, remaining = await limiter.is_allowed(key, rule["limit"], refill_rate)
        
        if not allowed:
            raise HTTPException(
                429,
                "Rate limit exceeded",
                headers={
                    "X-RateLimit-Limit": str(rule["limit"]),
                    "X-RateLimit-Remaining": "0",
                    "Retry-After": str(rule["window"]),
                },
            )
        
        response = await call_next(request)
        response.headers["X-RateLimit-Limit"] = str(rule["limit"])
        response.headers["X-RateLimit-Remaining"] = str(remaining)
        return response
```

---

## 3. Task Queue with Priority (Like Celery)

```python
"""
Implement a simple task queue with:
  - Priority levels (high, medium, low)
  - Retry on failure
  - Task status tracking
  - Concurrent worker execution
"""

from fastapi import FastAPI, BackgroundTasks
from enum import Enum
from datetime import datetime
from uuid import uuid4
import asyncio
import json

app = FastAPI()


class Priority(str, Enum):
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"


class TaskStatus(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    RETRYING = "retrying"


class TaskQueue:
    PRIORITY_SCORES = {"high": 3, "medium": 2, "low": 1}
    
    def __init__(self, redis):
        self.redis = redis
    
    async def enqueue(self, task_name: str, payload: dict, priority: Priority = Priority.MEDIUM, max_retries: int = 3) -> str:
        task_id = str(uuid4())
        task = {
            "id": task_id,
            "name": task_name,
            "payload": payload,
            "priority": priority,
            "max_retries": max_retries,
            "attempt": 0,
            "status": TaskStatus.PENDING,
            "created_at": datetime.utcnow().isoformat(),
        }
        
        # Store task metadata
        await self.redis.hset(f"task:{task_id}", mapping={k: json.dumps(v) if isinstance(v, (dict, list)) else str(v) for k, v in task.items()})
        
        # Add to priority queue (sorted set, score = priority)
        score = self.PRIORITY_SCORES[priority] * 1e10 + (1e10 - datetime.utcnow().timestamp())
        await self.redis.zadd("task_queue", {task_id: score})
        
        return task_id
    
    async def dequeue(self) -> dict | None:
        # Pop highest priority task
        result = await self.redis.zpopmax("task_queue")
        if not result:
            return None
        
        task_id = result[0][0]
        task_data = await self.redis.hgetall(f"task:{task_id}")
        return task_data
    
    async def update_status(self, task_id: str, status: TaskStatus, result=None, error=None):
        updates = {"status": status, "updated_at": datetime.utcnow().isoformat()}
        if result:
            updates["result"] = json.dumps(result)
        if error:
            updates["error"] = str(error)
        
        await self.redis.hset(f"task:{task_id}", mapping=updates)
    
    async def get_status(self, task_id: str) -> dict:
        return await self.redis.hgetall(f"task:{task_id}")


# Worker
async def task_worker(queue: TaskQueue):
    """Background worker that processes tasks."""
    HANDLERS = {
        "send_email": handle_send_email,
        "generate_report": handle_generate_report,
        "process_payment": handle_process_payment,
    }
    
    while True:
        task = await queue.dequeue()
        if not task:
            await asyncio.sleep(1)
            continue
        
        task_id = task["id"]
        handler = HANDLERS.get(task["name"])
        
        if not handler:
            await queue.update_status(task_id, TaskStatus.FAILED, error="Unknown task")
            continue
        
        await queue.update_status(task_id, TaskStatus.RUNNING)
        
        try:
            payload = json.loads(task.get("payload", "{}"))
            result = await handler(payload)
            await queue.update_status(task_id, TaskStatus.COMPLETED, result=result)
        except Exception as e:
            attempt = int(task.get("attempt", 0)) + 1
            max_retries = int(task.get("max_retries", 3))
            
            if attempt < max_retries:
                await queue.update_status(task_id, TaskStatus.RETRYING)
                await self.redis.hset(f"task:{task_id}", "attempt", str(attempt))
                # Re-enqueue with delay
                await asyncio.sleep(2 ** attempt)
                await self.redis.zadd("task_queue", {task_id: 1})
            else:
                await queue.update_status(task_id, TaskStatus.FAILED, error=str(e))


# API endpoints
@app.post("/tasks/")
async def create_task(task_name: str, payload: dict, priority: Priority = Priority.MEDIUM):
    queue = TaskQueue(app.state.redis)
    task_id = await queue.enqueue(task_name, payload, priority)
    return {"task_id": task_id, "status": "pending"}


@app.get("/tasks/{task_id}")
async def get_task_status(task_id: str):
    queue = TaskQueue(app.state.redis)
    status = await queue.get_status(task_id)
    if not status:
        raise HTTPException(404, "Task not found")
    return status
```

---

## 4. Real-Time Leaderboard

```python
"""
Implement a gaming leaderboard with:
  - Add/update score
  - Get top N players
  - Get player rank
  - Get nearby players (±5 positions)
  Using Redis Sorted Sets for O(log N) operations.
"""

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

LEADERBOARD_KEY = "leaderboard:global"


class ScoreUpdate(BaseModel):
    user_id: str
    score: int
    username: str


@app.post("/leaderboard/score")
async def update_score(data: ScoreUpdate):
    redis = app.state.redis
    
    # Update score (Redis ZADD — O(log N))
    await redis.zadd(LEADERBOARD_KEY, {data.user_id: data.score})
    
    # Store username mapping
    await redis.hset("usernames", data.user_id, data.username)
    
    # Get new rank
    rank = await redis.zrevrank(LEADERBOARD_KEY, data.user_id)
    
    return {"user_id": data.user_id, "score": data.score, "rank": rank + 1}


@app.get("/leaderboard/top/{n}")
async def get_top_players(n: int = 10):
    redis = app.state.redis
    
    # Get top N with scores (O(log N + M))
    results = await redis.zrevrange(LEADERBOARD_KEY, 0, n - 1, withscores=True)
    
    leaderboard = []
    for rank, (user_id, score) in enumerate(results, 1):
        username = await redis.hget("usernames", user_id) or "Unknown"
        leaderboard.append({
            "rank": rank,
            "user_id": user_id,
            "username": username,
            "score": int(score),
        })
    
    return {"leaderboard": leaderboard}


@app.get("/leaderboard/rank/{user_id}")
async def get_player_rank(user_id: str):
    redis = app.state.redis
    
    rank = await redis.zrevrank(LEADERBOARD_KEY, user_id)
    if rank is None:
        raise HTTPException(404, "Player not found")
    
    score = await redis.zscore(LEADERBOARD_KEY, user_id)
    total_players = await redis.zcard(LEADERBOARD_KEY)
    
    return {
        "user_id": user_id,
        "rank": rank + 1,
        "score": int(score),
        "total_players": total_players,
        "percentile": round((1 - rank / total_players) * 100, 1),
    }


@app.get("/leaderboard/nearby/{user_id}")
async def get_nearby_players(user_id: str, range_size: int = 5):
    redis = app.state.redis
    
    rank = await redis.zrevrank(LEADERBOARD_KEY, user_id)
    if rank is None:
        raise HTTPException(404, "Player not found")
    
    start = max(0, rank - range_size)
    end = rank + range_size
    
    results = await redis.zrevrange(LEADERBOARD_KEY, start, end, withscores=True)
    
    nearby = []
    for i, (uid, score) in enumerate(results):
        nearby.append({
            "rank": start + i + 1,
            "user_id": uid,
            "score": int(score),
            "is_current_user": uid == user_id,
        })
    
    return {"nearby": nearby, "current_rank": rank + 1}
```

---

## 5. Event Booking System (Concurrency Problem)

```python
"""
Problem: 100 seats, 1000 users trying to book simultaneously.
Must prevent overbooking (race condition).
"""

from fastapi import FastAPI, HTTPException, Depends
from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel
from uuid import UUID

app = FastAPI()


class BookingRequest(BaseModel):
    event_id: UUID
    seats: int = 1


# Solution 1: Pessimistic Locking (SELECT FOR UPDATE)
@app.post("/bookings/pessimistic")
async def book_pessimistic(data: BookingRequest, db: AsyncSession = Depends(get_db), user=Depends(get_current_user)):
    async with db.begin():
        # Lock the event row — other transactions wait
        result = await db.execute(
            select(Event)
            .where(Event.id == data.event_id)
            .with_for_update()
        )
        event = result.scalar_one_or_none()
        
        if not event:
            raise HTTPException(404, "Event not found")
        
        if event.available_seats < data.seats:
            raise HTTPException(409, f"Only {event.available_seats} seats available")
        
        # Deduct seats (safe — we hold the lock)
        event.available_seats -= data.seats
        
        booking = Booking(
            event_id=data.event_id,
            user_id=user.id,
            seats=data.seats,
            status="confirmed",
        )
        db.add(booking)
    
    return {"booking_id": str(booking.id), "seats": data.seats, "status": "confirmed"}


# Solution 2: Optimistic Locking (CAS — Compare And Swap)
@app.post("/bookings/optimistic")
async def book_optimistic(data: BookingRequest, db: AsyncSession = Depends(get_db), user=Depends(get_current_user)):
    max_retries = 3
    
    for attempt in range(max_retries):
        event = await db.get(Event, data.event_id)
        if not event:
            raise HTTPException(404, "Event not found")
        
        if event.available_seats < data.seats:
            raise HTTPException(409, "Not enough seats")
        
        # Atomic update with version check
        result = await db.execute(
            update(Event)
            .where(
                Event.id == data.event_id,
                Event.available_seats == event.available_seats,  # CAS condition
            )
            .values(available_seats=Event.available_seats - data.seats)
        )
        
        if result.rowcount == 1:
            # Success — no one else modified it
            booking = Booking(
                event_id=data.event_id, user_id=user.id,
                seats=data.seats, status="confirmed",
            )
            db.add(booking)
            await db.commit()
            return {"booking_id": str(booking.id), "status": "confirmed"}
        
        # Someone else modified — retry
        await db.rollback()
    
    raise HTTPException(409, "Could not complete booking — try again")


# Solution 3: Redis Atomic Decrement (fastest)
@app.post("/bookings/redis")
async def book_redis(data: BookingRequest, user=Depends(get_current_user)):
    redis = app.state.redis
    key = f"event_seats:{data.event_id}"
    
    # Lua script for atomic check-and-decrement
    lua_script = """
    local available = tonumber(redis.call('get', KEYS[1]) or 0)
    local requested = tonumber(ARGV[1])
    if available >= requested then
        redis.call('decrby', KEYS[1], requested)
        return 1
    else
        return 0
    end
    """
    
    result = await redis.eval(lua_script, 1, key, data.seats)
    
    if result == 0:
        available = await redis.get(key)
        raise HTTPException(409, f"Only {available} seats available")
    
    # Seats reserved in Redis — now persist to DB async
    booking_id = str(uuid4())
    await enqueue_task("persist_booking", {
        "booking_id": booking_id,
        "event_id": str(data.event_id),
        "user_id": str(user.id),
        "seats": data.seats,
    })
    
    return {"booking_id": booking_id, "status": "confirmed"}
```

---

## 6. Notification System (Fan-out)

```python
"""
Send notification to user across multiple channels:
  - Push notification (Firebase)
  - Email (SES)
  - In-app (WebSocket/SSE)
  - SMS (for critical alerts)
"""

from fastapi import FastAPI
from enum import Enum
from abc import ABC, abstractmethod


class Channel(str, Enum):
    PUSH = "push"
    EMAIL = "email"
    IN_APP = "in_app"
    SMS = "sms"


class NotificationBase(ABC):
    @abstractmethod
    async def send(self, user_id: str, title: str, body: str, data: dict = None): ...


class PushNotification(NotificationBase):
    async def send(self, user_id: str, title: str, body: str, data: dict = None):
        tokens = await get_user_fcm_tokens(user_id)
        for token in tokens:
            await firebase.send(token, title=title, body=body, data=data)


class EmailNotification(NotificationBase):
    async def send(self, user_id: str, title: str, body: str, data: dict = None):
        user = await get_user(user_id)
        await ses_client.send_email(
            to=user.email, subject=title, body=body,
        )


class InAppNotification(NotificationBase):
    async def send(self, user_id: str, title: str, body: str, data: dict = None):
        # Save to DB + push via WebSocket
        notification = await save_notification(user_id, title, body, data)
        await ws_manager.send_to_user(user_id, {
            "type": "notification",
            "id": str(notification.id),
            "title": title,
            "body": body,
        })


class SMSNotification(NotificationBase):
    async def send(self, user_id: str, title: str, body: str, data: dict = None):
        user = await get_user(user_id)
        await sns_client.publish(PhoneNumber=user.phone, Message=f"{title}: {body}")


class NotificationService:
    CHANNELS = {
        Channel.PUSH: PushNotification(),
        Channel.EMAIL: EmailNotification(),
        Channel.IN_APP: InAppNotification(),
        Channel.SMS: SMSNotification(),
    }
    
    # User preferences define which channels to use
    NOTIFICATION_RULES = {
        "order.shipped": [Channel.PUSH, Channel.EMAIL, Channel.IN_APP],
        "order.delivered": [Channel.PUSH, Channel.IN_APP],
        "payment.failed": [Channel.PUSH, Channel.EMAIL, Channel.SMS],
        "promo.offer": [Channel.EMAIL, Channel.IN_APP],
    }
    
    async def notify(self, user_id: str, event_type: str, title: str, body: str, data: dict = None):
        channels = self.NOTIFICATION_RULES.get(event_type, [Channel.IN_APP])
        
        # Check user preferences (user may disable some channels)
        preferences = await get_user_notification_preferences(user_id)
        active_channels = [ch for ch in channels if ch in preferences.enabled_channels]
        
        # Fan-out to all channels concurrently
        import asyncio
        results = await asyncio.gather(
            *[self.CHANNELS[ch].send(user_id, title, body, data) for ch in active_channels],
            return_exceptions=True,
        )
        
        # Log failures (don't fail the whole operation)
        for ch, result in zip(active_channels, results):
            if isinstance(result, Exception):
                logger.error(f"Notification failed: {ch} for {user_id}: {result}")


# Usage
notification_service = NotificationService()

@app.post("/orders/{order_id}/ship")
async def ship_order(order_id: str):
    order = await OrderService.mark_shipped(order_id)
    
    await notification_service.notify(
        user_id=str(order.user_id),
        event_type="order.shipped",
        title="Order Shipped!",
        body=f"Your order #{order_id[:8]} is on the way.",
        data={"order_id": order_id, "tracking_url": order.tracking_url},
    )
    
    return order
```

---

## 7. Interview Tips for FastAPI Coding Rounds

```
1. Start with the data model (Pydantic schemas + SQLAlchemy models)
   → Shows you think about the domain before jumping to code

2. Use dependency injection for everything
   → DB sessions, auth, rate limiting, services
   → Demonstrates testability thinking

3. Handle errors explicitly
   → Custom exception classes, not bare HTTPException everywhere
   → Show you think about failure modes

4. Think about concurrency
   → Race conditions (use locking or atomic operations)
   → asyncio.gather for parallel I/O
   → Background tasks for non-blocking operations

5. Consider caching strategy
   → Redis for read-heavy data
   → Cache invalidation on writes
   → TTL-based expiry

6. Security checklist
   → Input validation (Pydantic handles it)
   → Auth on all mutation endpoints
   → Rate limiting on sensitive endpoints
   → No SQL injection (use SQLAlchemy, never raw strings)

7. Production readiness signals
   → Health check endpoint
   → Structured logging
   → Graceful shutdown (lifespan)
   → Configuration from environment variables
```
