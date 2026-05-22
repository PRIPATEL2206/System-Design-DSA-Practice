# FastAPI Advanced Patterns — Senior Level

## 1. Event-Driven Architecture

### In-Process Event Bus
```python
from typing import Callable, Dict, List
import asyncio


class EventBus:
    """Simple in-process event bus for decoupling."""
    
    def __init__(self):
        self._handlers: Dict[str, List[Callable]] = {}
    
    def subscribe(self, event_type: str, handler: Callable):
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)
    
    async def publish(self, event_type: str, data: dict):
        handlers = self._handlers.get(event_type, [])
        await asyncio.gather(
            *[handler(data) for handler in handlers],
            return_exceptions=True,
        )
    
    def on(self, event_type: str):
        """Decorator to register event handler."""
        def decorator(func):
            self.subscribe(event_type, func)
            return func
        return decorator


event_bus = EventBus()


# Register handlers
@event_bus.on("order.created")
async def send_order_confirmation(data: dict):
    await EmailService.send(
        data["user_email"],
        "Order Confirmed",
        f"Order #{data['order_id'][:8]} total: ₹{data['total']}",
    )


@event_bus.on("order.created")
async def update_inventory(data: dict):
    for item in data["items"]:
        await InventoryService.deduct(item["product_id"], item["quantity"])


@event_bus.on("order.created")
async def track_analytics(data: dict):
    await AnalyticsService.track("order_created", {
        "user_id": data["user_id"],
        "total": data["total"],
    })


# Publish event from service
class OrderService:
    async def create(self, user, order_data):
        order = await self.repo.create(order_data)
        
        # Fire event (all handlers run concurrently)
        await event_bus.publish("order.created", {
            "order_id": str(order.id),
            "user_id": str(user.id),
            "user_email": user.email,
            "total": float(order.total),
            "items": order_data.items,
        })
        
        return order
```

### External Event Bus (SQS/SNS/Redis Streams)
```python
import json
from datetime import datetime


class ExternalEventPublisher:
    """Publish events to SNS for cross-service communication."""
    
    def __init__(self, sns_client, topic_arn: str):
        self.sns = sns_client
        self.topic_arn = topic_arn
    
    async def publish(self, event_type: str, data: dict):
        message = {
            "event_type": event_type,
            "data": data,
            "timestamp": datetime.utcnow().isoformat(),
            "source": "order-service",
        }
        
        self.sns.publish(
            TopicArn=self.topic_arn,
            Message=json.dumps(message),
            MessageAttributes={
                "event_type": {
                    "DataType": "String",
                    "StringValue": event_type,
                },
            },
        )


# Usage: multiple services subscribe to SNS topic
# Order Service publishes → SNS → SQS queues
#   → Inventory Service (deduct stock)
#   → Notification Service (send email)
#   → Analytics Service (track event)
```

---

## 2. CQRS Pattern (Command Query Separation)

```python
"""
CQRS: separate read and write models/paths

Write path (Commands):
  POST/PUT/DELETE → validate → write to PostgreSQL → publish event

Read path (Queries):
  GET → read from Redis/Elasticsearch (optimized for reads)
  
Why?
  - Reads are 100x more frequent than writes
  - Read model can be denormalized (no JOINs = fast)
  - Scale reads independently (Redis cluster, read replicas)
"""

# Write side (Command)
class CreateOrderCommand:
    async def execute(self, user_id: str, items: list) -> Order:
        async with db.begin():
            order = Order(user_id=user_id)
            # ... complex write logic with transactions
            await db.commit()
        
        # Update read model
        await self.update_read_model(order)
        return order
    
    async def update_read_model(self, order: Order):
        """Denormalize order for fast reads."""
        order_view = {
            "id": str(order.id),
            "user_id": str(order.user_id),
            "status": order.status,
            "total": float(order.total),
            "item_count": len(order.items),
            "created_at": order.created_at.isoformat(),
            # Pre-join user name (avoids JOIN on read)
            "user_name": order.user.full_name,
        }
        await redis.hset(f"order:{order.id}", mapping=order_view)
        await redis.zadd(
            f"user_orders:{order.user_id}",
            {str(order.id): order.created_at.timestamp()},
        )


# Read side (Query) — fast, no JOINs
class OrderQueryService:
    async def get_order(self, order_id: str) -> dict:
        # Try Redis first (denormalized view)
        cached = await redis.hgetall(f"order:{order_id}")
        if cached:
            return cached
        
        # Fallback to DB
        order = await db.get(Order, order_id)
        return OrderResponse.model_validate(order).model_dump()
    
    async def get_user_orders(self, user_id: str, limit: int = 20) -> list:
        # Redis sorted set (ordered by created_at)
        order_ids = await redis.zrevrange(
            f"user_orders:{user_id}", 0, limit - 1
        )
        
        pipeline = redis.pipeline()
        for oid in order_ids:
            pipeline.hgetall(f"order:{oid}")
        
        return await pipeline.execute()
```

---

## 3. Circuit Breaker Pattern

```python
import asyncio
import time
from enum import Enum
from dataclasses import dataclass


class CircuitState(Enum):
    CLOSED = "closed"      # normal operation
    OPEN = "open"          # failing, reject immediately
    HALF_OPEN = "half_open"  # testing if service recovered


@dataclass
class CircuitBreaker:
    """Prevent cascade failures when external service is down."""
    
    name: str
    failure_threshold: int = 5
    recovery_timeout: int = 30
    half_open_max_calls: int = 3
    
    def __post_init__(self):
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = 0
        self.half_open_calls = 0
    
    async def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                self.half_open_calls = 0
            else:
                raise CircuitOpenError(f"Circuit {self.name} is OPEN")
        
        if self.state == CircuitState.HALF_OPEN:
            if self.half_open_calls >= self.half_open_max_calls:
                raise CircuitOpenError(f"Circuit {self.name} is HALF_OPEN (max calls reached)")
            self.half_open_calls += 1
        
        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
    
    def _on_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.CLOSED
        self.failure_count = 0
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN


class CircuitOpenError(Exception):
    pass


# Usage
payment_circuit = CircuitBreaker(name="payment_gateway", failure_threshold=3, recovery_timeout=60)


@router.post("/orders/{order_id}/pay")
async def pay_order(order_id: str):
    try:
        result = await payment_circuit.call(
            PaymentGateway.charge,
            order_id=order_id,
        )
        return {"status": "paid", "transaction_id": result.id}
    except CircuitOpenError:
        # Graceful degradation
        await enqueue_task("retry_payment", {"order_id": order_id})
        return {"status": "payment_queued", "message": "Payment will be processed shortly"}
```

---

## 4. Retry Pattern with Exponential Backoff

```python
import asyncio
import random
from functools import wraps
from typing import Type


def retry(
    max_attempts: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 30.0,
    exceptions: tuple[Type[Exception], ...] = (Exception,),
    exponential: bool = True,
):
    """Retry decorator with exponential backoff and jitter."""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e
                    
                    if attempt == max_attempts - 1:
                        raise
                    
                    # Exponential backoff with jitter
                    if exponential:
                        delay = min(base_delay * (2 ** attempt), max_delay)
                    else:
                        delay = base_delay
                    
                    jitter = random.uniform(0, delay * 0.1)
                    await asyncio.sleep(delay + jitter)
            
            raise last_exception
        return wrapper
    return decorator


# Usage
@retry(max_attempts=3, base_delay=1.0, exceptions=(httpx.TimeoutException, httpx.ConnectError))
async def call_external_api(url: str, payload: dict) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.post(url, json=payload, timeout=10.0)
        response.raise_for_status()
        return response.json()


# With circuit breaker + retry together
@retry(max_attempts=2, exceptions=(httpx.TimeoutException,))
async def call_payment_gateway(amount: float, method: str):
    return await payment_circuit.call(
        _do_payment_call, amount, method
    )
```

---

## 5. Idempotency Pattern

```python
from fastapi import Header, HTTPException
import hashlib
import json


class IdempotencyService:
    """Prevent duplicate operations (double-submit, retries)."""
    
    def __init__(self, redis):
        self.redis = redis
        self.ttl = 86400  # 24 hours
    
    async def check_and_lock(self, key: str) -> dict | None:
        """Returns cached response if key exists, else locks it."""
        existing = await self.redis.get(f"idempotency:{key}")
        if existing:
            return json.loads(existing)
        
        # Lock with TTL (prevent race condition)
        locked = await self.redis.set(
            f"idempotency:{key}", 
            json.dumps({"status": "processing"}),
            ex=self.ttl,
            nx=True,  # only set if not exists
        )
        
        if not locked:
            # Another request is processing this key
            raise HTTPException(409, "Request is being processed")
        
        return None
    
    async def save_response(self, key: str, response: dict):
        await self.redis.set(
            f"idempotency:{key}",
            json.dumps(response),
            ex=self.ttl,
        )


# Dependency
async def idempotency_check(
    idempotency_key: str = Header(None, alias="Idempotency-Key"),
):
    if not idempotency_key:
        return None
    
    service = IdempotencyService(app.state.redis)
    cached = await service.check_and_lock(idempotency_key)
    
    if cached and cached.get("status") != "processing":
        # Return cached response directly
        from fastapi.responses import JSONResponse
        return JSONResponse(content=cached)
    
    return idempotency_key


# Usage
@router.post("/payments/")
async def create_payment(
    data: PaymentCreate,
    idempotency_key: str = Depends(idempotency_check),
):
    # If idempotency_check returned a Response, FastAPI returns it directly
    
    result = await PaymentService.create(data)
    
    # Cache the response
    if idempotency_key:
        service = IdempotencyService(app.state.redis)
        await service.save_response(idempotency_key, result.model_dump(mode="json"))
    
    return result
```

---

## 6. Rate Limiting (Advanced)

### Sliding Window with Redis
```python
import time
from fastapi import Request, HTTPException


class SlidingWindowRateLimiter:
    """Production rate limiter with multiple tiers."""
    
    def __init__(self, redis):
        self.redis = redis
    
    async def check(self, key: str, limit: int, window: int) -> tuple[bool, dict]:
        """Check if request is allowed. Returns (allowed, info)."""
        now = time.time()
        window_start = now - window
        
        pipe = self.redis.pipeline()
        pipe.zremrangebyscore(key, 0, window_start)  # remove expired
        pipe.zcard(key)                                # count current
        pipe.zadd(key, {f"{now}": now})               # add this request
        pipe.expire(key, window)                       # set TTL
        results = await pipe.execute()
        
        current_count = results[1]
        
        return current_count < limit, {
            "limit": limit,
            "remaining": max(0, limit - current_count - 1),
            "reset": int(now + window),
        }


class RateLimitMiddleware:
    """Multi-tier rate limiting middleware."""
    
    TIERS = {
        "free": {"requests": 100, "window": 3600},
        "pro": {"requests": 1000, "window": 3600},
        "enterprise": {"requests": 10000, "window": 3600},
    }
    
    async def __call__(self, request: Request, call_next):
        # Determine user tier
        user = getattr(request.state, "user", None)
        tier = user.plan if user else "free"
        key = f"rate:{user.id if user else request.client.host}"
        
        config = self.TIERS[tier]
        limiter = SlidingWindowRateLimiter(request.app.state.redis)
        
        allowed, info = await limiter.check(key, config["requests"], config["window"])
        
        if not allowed:
            raise HTTPException(
                status_code=429,
                detail="Rate limit exceeded",
                headers={
                    "X-RateLimit-Limit": str(info["limit"]),
                    "X-RateLimit-Remaining": "0",
                    "X-RateLimit-Reset": str(info["reset"]),
                    "Retry-After": str(config["window"]),
                },
            )
        
        response = await call_next(request)
        response.headers["X-RateLimit-Limit"] = str(info["limit"])
        response.headers["X-RateLimit-Remaining"] = str(info["remaining"])
        response.headers["X-RateLimit-Reset"] = str(info["reset"])
        
        return response
```

---

## 7. Distributed Locking

```python
import uuid
import asyncio


class DistributedLock:
    """Redis-based distributed lock (Redlock-like)."""
    
    def __init__(self, redis, name: str, timeout: int = 10):
        self.redis = redis
        self.name = f"lock:{name}"
        self.timeout = timeout
        self.token = str(uuid.uuid4())
    
    async def acquire(self) -> bool:
        return await self.redis.set(
            self.name, self.token, ex=self.timeout, nx=True
        )
    
    async def release(self):
        # Only release if we own the lock (Lua script for atomicity)
        script = """
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end
        """
        await self.redis.eval(script, 1, self.name, self.token)
    
    async def __aenter__(self):
        for _ in range(50):  # retry for 5 seconds
            if await self.acquire():
                return self
            await asyncio.sleep(0.1)
        raise TimeoutError(f"Could not acquire lock: {self.name}")
    
    async def __aexit__(self, *args):
        await self.release()


# Usage: prevent double-processing
@router.post("/orders/{order_id}/process")
async def process_order(order_id: str):
    async with DistributedLock(app.state.redis, f"order:{order_id}", timeout=30):
        # Only one server processes this order at a time
        order = await OrderService.get(order_id)
        if order.status != "pending":
            return {"status": order.status}  # already processed
        
        await OrderService.process(order)
        return {"status": "processed"}
```

---

## 8. Interview Questions

### Q: How do you handle distributed transactions across microservices?
```
Problem: Order Service + Payment Service + Inventory Service
  All must succeed or all must rollback.

Solutions (best to worst):
  1. Saga Pattern (preferred)
     - Each service does its step + publishes event
     - If one fails → compensating transactions undo previous steps
     - Choreography: services react to events
     - Orchestration: central coordinator manages flow
  
  2. Outbox Pattern
     - Write event to outbox table in same DB transaction
     - Background process polls outbox and publishes to message queue
     - Guarantees at-least-once delivery
  
  3. Two-Phase Commit (2PC)
     - Prepare → Commit/Rollback
     - Slow, locks resources, single point of failure
     - Avoid in microservices

Example Saga:
  Order Created → Payment Charged → Inventory Deducted → Order Confirmed
  
  If Inventory fails:
    → Compensate: Refund Payment
    → Compensate: Cancel Order
    → Notify User: "Out of stock"
```

### Q: How do you implement idempotency in APIs?
```
Why: Network retries, double-clicks, webhook replays

Implementation:
  1. Client sends Idempotency-Key header (UUID)
  2. Server checks Redis: if key exists → return cached response
  3. If not exists → process request, cache response with TTL
  4. Same key + same request = same response (no double charge)

Key decisions:
  - TTL: 24 hours (balance storage vs. retry window)
  - Scope: per-user + per-endpoint + idempotency key
  - Race condition: use Redis NX (set-if-not-exists)
  - Response: cache full response including status code

Where it matters:
  - Payment processing (double charge = very bad)
  - Order creation (duplicate orders)
  - Any non-idempotent operation (POST, not GET/PUT/DELETE)
```
