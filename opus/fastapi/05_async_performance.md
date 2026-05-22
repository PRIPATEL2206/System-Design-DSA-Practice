# FastAPI Async & Performance — Senior Level

## 1. Async/Await Deep Dive

### How FastAPI Handles Concurrency
```
Request comes in
    ↓
If endpoint is `async def`:
    → Runs on the main event loop (uvicorn's asyncio loop)
    → MUST use `await` for I/O (otherwise blocks entire loop)

If endpoint is `def` (sync):
    → Runs in a thread pool (threadpool executor)
    → Safe for blocking libraries (pandas, scikit-learn)
    → Slightly slower (thread context switch overhead)

Key Rule:
  async def + await = concurrent I/O (thousands of connections)
  def = thread pool (limited by thread count, default ~40)
```

### Concurrent I/O with asyncio.gather
```python
import asyncio
import httpx
from fastapi import FastAPI

app = FastAPI()

async def fetch_user_profile(user_id: str) -> dict:
    async with httpx.AsyncClient() as client:
        resp = await client.get(f"http://user-service/users/{user_id}")
        return resp.json()

async def fetch_user_orders(user_id: str) -> list:
    async with httpx.AsyncClient() as client:
        resp = await client.get(f"http://order-service/users/{user_id}/orders")
        return resp.json()

async def fetch_user_recommendations(user_id: str) -> list:
    async with httpx.AsyncClient() as client:
        resp = await client.get(f"http://ml-service/recommendations/{user_id}")
        return resp.json()


@app.get("/users/{user_id}/dashboard")
async def user_dashboard(user_id: str):
    # BAD: sequential (3 seconds total if each takes 1s)
    # profile = await fetch_user_profile(user_id)
    # orders = await fetch_user_orders(user_id)
    # recs = await fetch_user_recommendations(user_id)

    # GOOD: concurrent (1 second total — all run in parallel)
    profile, orders, recs = await asyncio.gather(
        fetch_user_profile(user_id),
        fetch_user_orders(user_id),
        fetch_user_recommendations(user_id),
    )

    return {"profile": profile, "orders": orders, "recommendations": recs}
```

### asyncio.gather with Error Handling
```python
@app.get("/users/{user_id}/dashboard")
async def user_dashboard(user_id: str):
    results = await asyncio.gather(
        fetch_user_profile(user_id),
        fetch_user_orders(user_id),
        fetch_user_recommendations(user_id),
        return_exceptions=True,  # don't fail entire gather if one fails
    )

    profile = results[0] if not isinstance(results[0], Exception) else None
    orders = results[1] if not isinstance(results[1], Exception) else []
    recs = results[2] if not isinstance(results[2], Exception) else []

    return {"profile": profile, "orders": orders, "recommendations": recs}
```

---

## 2. Background Tasks (In-Process)

### FastAPI BackgroundTasks (Simple)
```python
from fastapi import BackgroundTasks

async def send_notification(user_id: str, message: str):
    # Runs after response is sent
    await notification_service.push(user_id, message)

async def update_analytics(event: str, data: dict):
    await analytics_service.track(event, data)

@app.post("/orders/", status_code=201)
async def create_order(
    order: OrderCreate,
    background_tasks: BackgroundTasks,
    user=Depends(get_current_user),
):
    new_order = await OrderService.create(user, order)

    # Fire-and-forget tasks (run after response)
    background_tasks.add_task(send_notification, str(user.id), "Order placed!")
    background_tasks.add_task(update_analytics, "order_created", {"order_id": str(new_order.id)})

    return new_order
```

### ARQ (Redis-based task queue — like Celery but async)
```python
# worker.py
from arq import create_pool
from arq.connections import RedisSettings

async def process_payment(ctx, order_id: str):
    """Heavy task — runs in separate worker process."""
    order = await OrderService.get(order_id)
    result = await PaymentGateway.charge(order.total, order.payment_method)
    await OrderService.update_status(order_id, "paid" if result.success else "failed")

async def generate_report(ctx, report_type: str, params: dict):
    """Long-running report generation."""
    data = await ReportService.generate(report_type, params)
    url = await S3Service.upload(f"reports/{report_type}_{datetime.now().isoformat()}.csv", data)
    await NotificationService.send(ctx["user_id"], f"Report ready: {url}")

class WorkerSettings:
    functions = [process_payment, generate_report]
    redis_settings = RedisSettings(host="redis", port=6379)
    max_jobs = 10
    job_timeout = 300


# Enqueue from endpoint
from arq import create_pool

@app.post("/orders/{order_id}/pay")
async def pay_order(order_id: str):
    redis = await create_pool(RedisSettings())
    await redis.enqueue_job("process_payment", order_id)
    return {"status": "payment_processing"}
```

---

## 3. Streaming Responses

### StreamingResponse (Large Files / Data)
```python
from fastapi.responses import StreamingResponse
import csv
import io

@app.get("/export/products")
async def export_products(db=Depends(get_db)):
    async def generate_csv():
        # Header
        yield "id,name,price,stock\n"

        # Stream rows (don't load all into memory)
        offset = 0
        batch_size = 1000
        while True:
            result = await db.execute(
                select(Product).offset(offset).limit(batch_size)
            )
            products = result.scalars().all()
            if not products:
                break

            for p in products:
                yield f"{p.id},{p.name},{p.price},{p.stock}\n"

            offset += batch_size

    return StreamingResponse(
        generate_csv(),
        media_type="text/csv",
        headers={"Content-Disposition": "attachment; filename=products.csv"},
    )
```

### Server-Sent Events (SSE)
```python
from fastapi.responses import StreamingResponse
import asyncio
import json

@app.get("/events/orders/{user_id}")
async def order_events(user_id: str):
    async def event_stream():
        pubsub = app.state.redis.pubsub()
        await pubsub.subscribe(f"orders:{user_id}")

        try:
            while True:
                message = await pubsub.get_message(
                    ignore_subscribe_messages=True, timeout=1.0
                )
                if message:
                    data = json.loads(message["data"])
                    yield f"event: order_update\ndata: {json.dumps(data)}\n\n"
                else:
                    # Heartbeat every 15 seconds
                    yield f": heartbeat\n\n"
                    await asyncio.sleep(15)
        finally:
            await pubsub.unsubscribe(f"orders:{user_id}")

    return StreamingResponse(
        event_stream(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # disable nginx buffering
        },
    )
```

---

## 4. Caching Strategies

### Redis Caching with TTL
```python
import json
from functools import wraps

class CacheService:
    def __init__(self, redis):
        self.redis = redis

    async def get(self, key: str):
        data = await self.redis.get(key)
        return json.loads(data) if data else None

    async def set(self, key: str, value, ttl: int = 300):
        await self.redis.set(key, json.dumps(value, default=str), ex=ttl)

    async def delete(self, key: str):
        await self.redis.delete(key)

    async def delete_pattern(self, pattern: str):
        keys = await self.redis.keys(pattern)
        if keys:
            await self.redis.delete(*keys)


# Cache decorator
def cached(key_prefix: str, ttl: int = 300):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Build cache key from arguments
            cache_key = f"{key_prefix}:{':'.join(str(v) for v in kwargs.values())}"
            
            redis = app.state.redis
            cached_data = await redis.get(cache_key)
            if cached_data:
                return json.loads(cached_data)

            result = await func(*args, **kwargs)
            await redis.set(cache_key, json.dumps(result, default=str), ex=ttl)
            return result
        return wrapper
    return decorator


# Usage
@cached(key_prefix="product", ttl=600)
async def get_product_cached(product_id: str) -> dict:
    product = await db.get(Product, product_id)
    return ProductResponse.model_validate(product).model_dump()


# Cache invalidation on write
@app.put("/products/{product_id}")
async def update_product(product_id: UUID, data: ProductUpdate, db=Depends(get_db)):
    product = await ProductService.update(db, product_id, data)
    
    # Invalidate cache
    await app.state.redis.delete(f"product:{product_id}")
    await app.state.redis.delete_pattern("products:list:*")
    
    return product
```

### HTTP Cache Headers
```python
from fastapi import Response

@app.get("/products/{product_id}")
async def get_product(product_id: UUID, response: Response):
    product = await ProductService.get(product_id)
    
    # Cache for 5 minutes in browser/CDN
    response.headers["Cache-Control"] = "public, max-age=300"
    response.headers["ETag"] = f'"{product.updated_at.timestamp()}"'
    
    return product

@app.get("/products/")
async def list_products(response: Response):
    # Private = don't cache in CDN, only browser
    response.headers["Cache-Control"] = "private, max-age=60"
    return await ProductService.list()
```

---

## 5. Connection Pooling & HTTP Client

### Shared HTTP Client (Don't Create Per Request)
```python
import httpx
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Create ONE client for the app's lifetime
    app.state.http_client = httpx.AsyncClient(
        timeout=httpx.Timeout(30.0, connect=5.0),
        limits=httpx.Limits(max_connections=100, max_keepalive_connections=20),
        headers={"User-Agent": "MyApp/1.0"},
    )
    yield
    await app.state.http_client.aclose()

app = FastAPI(lifespan=lifespan)


# Use shared client in endpoints
@app.get("/external/weather")
async def get_weather(city: str, request: Request):
    client = request.app.state.http_client
    resp = await client.get(f"https://api.weather.com/v1/{city}")
    return resp.json()
```

### Database Connection Pool Tuning
```python
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,           # steady-state connections
    max_overflow=10,        # burst capacity (total max = 30)
    pool_pre_ping=True,     # check connection health before use
    pool_recycle=300,       # replace connections older than 5 min
    pool_timeout=30,        # wait max 30s for available connection
    echo=False,             # never True in production
)

"""
Sizing guide:
  - pool_size = expected concurrent DB queries
  - max_overflow = burst headroom
  - Total connections = pool_size + max_overflow
  - PostgreSQL default max_connections = 100
  - If 4 Gunicorn workers: each gets its own pool
    → 4 workers × 20 pool_size = 80 connections
    → Leave headroom for migrations, monitoring
  
  With PgBouncer (transaction mode):
    → Set pool_size higher (PgBouncer multiplexes)
    → Set pool_pre_ping=False (PgBouncer handles health)
"""
```

---

## 6. Performance Profiling

### Request Timing Middleware
```python
import time
import logging
from fastapi import Request

logger = logging.getLogger(__name__)

@app.middleware("http")
async def performance_middleware(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    duration = time.perf_counter() - start

    # Log slow requests
    if duration > 1.0:
        logger.warning(
            f"SLOW REQUEST: {request.method} {request.url.path} "
            f"took {duration:.3f}s"
        )

    response.headers["X-Response-Time"] = f"{duration:.4f}s"
    return response
```

### Profiling Endpoint (Development Only)
```python
import cProfile
import pstats
import io

@app.get("/debug/profile/{endpoint_path:path}")
async def profile_endpoint(endpoint_path: str):
    """Profile an endpoint (dev only)."""
    if settings.environment != "development":
        raise HTTPException(403, "Not available in production")
    
    profiler = cProfile.Profile()
    profiler.enable()
    
    async with httpx.AsyncClient() as client:
        await client.get(f"http://localhost:8000/{endpoint_path}")
    
    profiler.disable()
    
    stream = io.StringIO()
    stats = pstats.Stats(profiler, stream=stream)
    stats.sort_stats("cumulative")
    stats.print_stats(20)
    
    return {"profile": stream.getvalue()}
```

---

## 7. Interview Questions

### Q: How does FastAPI handle concurrency vs Django?
```
FastAPI (ASGI + asyncio):
  - Single thread, event loop
  - 1 worker handles 1000s of concurrent I/O-bound requests
  - `await` yields control during I/O → other requests proceed
  - Like Node.js but with Python syntax
  - 4 workers = 4 event loops = tens of thousands concurrent

Django (WSGI, traditional):
  - 1 request per thread/process
  - 4 workers = 4 concurrent requests
  - async Django exists but ecosystem not fully async
  - Needs Channels for WebSocket

Practical impact:
  FastAPI: 4 workers, 10,000 concurrent WebSocket + 5,000 HTTP
  Django:  4 workers, 4 concurrent requests (need 100+ workers)
```

### Q: When would async FastAPI be SLOWER than sync?
```
1. CPU-bound work in async endpoint (blocks entire event loop)
   → Use `def` endpoint or run_in_executor()
   
2. Using sync database driver (psycopg2 instead of asyncpg)
   → Forces thread pool overhead with no concurrency benefit
   
3. Very low traffic (< 50 concurrent)
   → async overhead not worth it, sync Flask is simpler
   
4. Heavy computation (pandas, numpy, ML inference)
   → Use def endpoint (FastAPI auto-threads it)
   → Or use background workers (ARQ, Celery)

Rule: async wins when you have MANY concurrent I/O operations
      sync wins when you have FEW concurrent CPU operations
```

### Q: How do you prevent memory leaks in long-running FastAPI?
```
1. max_requests in Gunicorn (restart worker after N requests)
2. pool_recycle for SQLAlchemy (refresh stale connections)
3. Close HTTP clients properly (lifespan shutdown)
4. Use streaming for large responses (don't buffer in memory)
5. Weak references for caches (or TTL-based eviction)
6. Monitor with prometheus_client + Grafana
7. Profile with memory_profiler or tracemalloc in staging
```
