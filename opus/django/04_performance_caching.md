# Django Performance & Caching — Senior Level

## 1. Query Optimization

### Identify Slow Queries
```python
# Method 1: django-debug-toolbar (development)
# pip install django-debug-toolbar
INSTALLED_APPS = [..., 'debug_toolbar']
MIDDLEWARE = ['debug_toolbar.middleware.DebugToolbarMiddleware', ...]
INTERNAL_IPS = ['127.0.0.1']

# Method 2: Log slow queries (production)
LOGGING = {
    'loggers': {
        'django.db.backends': {
            'level': 'DEBUG',  # logs ALL SQL queries
            'handlers': ['console'],
        }
    }
}

# Method 3: django-silk (production profiling)
# pip install django-silk
MIDDLEWARE = ['silk.middleware.SilkyMiddleware', ...]

# Method 4: Explain queries programmatically
from django.db import connection

qs = Product.objects.filter(price__gt=100)
print(qs.query)  # see generated SQL
print(qs.explain())  # PostgreSQL EXPLAIN output
```

### Common Query Optimizations
```python
# ❌ BAD: Fetching all objects to count them
count = len(Product.objects.all())  # loads ALL objects into memory

# ✓ GOOD: Database-level count
count = Product.objects.count()  # SELECT COUNT(*) — no objects loaded

# ❌ BAD: Checking existence by fetching
if Product.objects.filter(name='iPhone').first():

# ✓ GOOD: Database-level existence check
if Product.objects.filter(name='iPhone').exists():  # SELECT 1 LIMIT 1

# ❌ BAD: Fetching objects just to get IDs
ids = [p.id for p in Product.objects.all()]

# ✓ GOOD: values_list with flat=True
ids = list(Product.objects.values_list('id', flat=True))

# ❌ BAD: Updating objects one by one
for product in Product.objects.filter(category='electronics'):
    product.on_sale = True
    product.save()  # N queries!

# ✓ GOOD: Bulk update (single query)
Product.objects.filter(category='electronics').update(on_sale=True)

# ❌ BAD: Creating objects one by one
for item in items:
    Product.objects.create(**item)  # N INSERT queries

# ✓ GOOD: Bulk create (single INSERT with multiple rows)
products = [Product(**item) for item in items]
Product.objects.bulk_create(products, batch_size=1000)

# ✓ GOOD: Bulk update with different values
products = Product.objects.filter(category='electronics')
for p in products:
    p.price = p.price * 0.9  # 10% discount
Product.objects.bulk_update(products, ['price'], batch_size=1000)
```

### Database Indexing Strategy
```python
class Product(models.Model):
    name = models.CharField(max_length=255)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    category = models.ForeignKey(Category, on_delete=models.CASCADE)
    status = models.CharField(max_length=20)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        indexes = [
            # Single column indexes for frequently filtered fields
            models.Index(fields=['status']),
            models.Index(fields=['price']),
            
            # Composite index: queries that filter by BOTH fields
            # ORDER MATTERS: put high-cardinality field first
            models.Index(fields=['category', 'status']),
            
            # Covering index for common queries
            models.Index(fields=['status', 'created_at', 'price']),
            
            # Partial index: only index rows that matter
            models.Index(
                fields=['created_at'],
                condition=models.Q(status='published'),
                name='published_products_idx'
            ),
            
            # GIN index for JSONField or full-text search (PostgreSQL)
            # models.Index(fields=['metadata'], opclasses=['jsonb_path_ops'])
        ]

# When to add an index:
# ✓ Columns in WHERE clauses (filter, get, exclude)
# ✓ Columns in ORDER BY
# ✓ Columns in JOIN (ForeignKey — auto-indexed)
# ✓ Columns with high cardinality (many unique values)
# ✗ Don't index columns that are rarely queried
# ✗ Don't index columns with low cardinality (boolean, status with 3 values)
#    Exception: partial index on rare values IS useful
```

---

## 2. Caching Strategies

### Cache Levels in Django
```
Level 1: Browser cache (Cache-Control headers)
Level 2: CDN cache (CloudFront, Cloudflare)
Level 3: Reverse proxy cache (Nginx, Varnish)
Level 4: Django view cache (full page/response)
Level 5: Django template fragment cache
Level 6: Django low-level cache (specific data)
Level 7: Database query cache (query result)
Level 8: ORM cache (django-cachalot)
```

### Redis Cache Setup
```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://redis-host:6379/0',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'SERIALIZER': 'django_redis.serializers.json.JSONSerializer',
            'SOCKET_CONNECT_TIMEOUT': 5,
            'SOCKET_TIMEOUT': 5,
            'CONNECTION_POOL_KWARGS': {'max_connections': 50},
        },
        'KEY_PREFIX': 'myapp',
        'TIMEOUT': 300,  # default TTL: 5 minutes
    },
    'sessions': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://redis-host:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
    }
}

# Use Redis for sessions (faster than DB sessions)
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'sessions'
```

### View-Level Caching
```python
from django.views.decorators.cache import cache_page
from django.utils.decorators import method_decorator

# Function-based view: cache entire response for 15 minutes
@cache_page(60 * 15)
def product_list(request):
    products = Product.objects.published()
    return render(request, 'products/list.html', {'products': products})

# Class-based view
@method_decorator(cache_page(60 * 15), name='dispatch')
class ProductListView(ListView):
    model = Product

# Cache with vary headers (different cache per user/language)
from django.views.decorators.vary import vary_on_cookie, vary_on_headers

@vary_on_cookie  # different cache per session cookie
@cache_page(60 * 15)
def dashboard(request):
    ...

# DRF: per-view caching
from rest_framework.decorators import api_view
from django.views.decorators.cache import cache_page

class ProductViewSet(viewsets.ReadOnlyModelViewSet):
    @method_decorator(cache_page(60 * 5))
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```

### Low-Level Cache (Most Flexible)
```python
from django.core.cache import cache

# Basic get/set
def get_product_detail(product_id):
    cache_key = f'product:{product_id}'
    
    product_data = cache.get(cache_key)
    if product_data is None:
        product = Product.objects.select_related('category').get(id=product_id)
        product_data = ProductSerializer(product).data
        cache.set(cache_key, product_data, timeout=300)  # 5 min TTL
    
    return product_data

# Cache invalidation on update
def update_product(product_id, data):
    product = Product.objects.get(id=product_id)
    # ... update product ...
    product.save()
    
    # Invalidate cache
    cache.delete(f'product:{product_id}')
    cache.delete('product_list')  # also invalidate list cache

# Atomic cache operations (prevent race conditions)
def increment_view_count(product_id):
    cache_key = f'views:{product_id}'
    try:
        cache.incr(cache_key)
    except ValueError:
        cache.set(cache_key, 1, timeout=86400)

# Cache with version (easy bulk invalidation)
cache.set('product_list', data, version=2)
cache.incr_version('product_list')  # invalidates old version

# get_or_set (atomic get + set if missing)
data = cache.get_or_set(
    'expensive_query_result',
    lambda: expensive_computation(),
    timeout=600
)
```

### Cache Patterns

#### Pattern 1: Cache-Aside (Most Common)
```python
def get_user_profile(user_id):
    key = f'user_profile:{user_id}'
    data = cache.get(key)
    if data is None:                          # Cache MISS
        data = UserProfile.objects.get(user_id=user_id)
        cache.set(key, data, timeout=300)     # Populate cache
    return data                               # Cache HIT
```

#### Pattern 2: Write-Through (Update Cache on Write)
```python
def update_user_profile(user_id, new_data):
    profile = UserProfile.objects.get(user_id=user_id)
    for key, value in new_data.items():
        setattr(profile, key, value)
    profile.save()
    
    # Immediately update cache (not just delete)
    cache.set(f'user_profile:{user_id}', profile, timeout=300)
```

#### Pattern 3: Cache Stampede Prevention
```python
import time
import random

def get_popular_products():
    key = 'popular_products'
    data = cache.get(key)
    
    if data is None:
        # Use lock to prevent 1000 concurrent requests all hitting DB
        lock_key = f'{key}:lock'
        if cache.add(lock_key, 1, timeout=10):  # acquired lock
            try:
                data = compute_popular_products()
                cache.set(key, data, timeout=300)
            finally:
                cache.delete(lock_key)
        else:
            # Another request is computing — wait briefly or return stale
            time.sleep(0.1)
            data = cache.get(key)
    
    return data
```

---

## 3. Database Connection Pooling

```python
# Django default: opens new connection per request (slow)
# Fix: use connection pooling

# Method 1: django-db-connection-pool
# pip install django-db-connection-pool
DATABASES = {
    'default': {
        'ENGINE': 'dj_db_conn_pool.backends.postgresql',
        'NAME': 'mydb',
        'POOL_OPTIONS': {
            'POOL_SIZE': 10,       # connections always open
            'MAX_OVERFLOW': 20,    # extra connections when busy
            'RECYCLE': 300,        # recycle connections after 5 min
        }
    }
}

# Method 2: PgBouncer (external connection pooler)
# Django connects to PgBouncer (port 6432)
# PgBouncer pools connections to PostgreSQL (port 5432)
DATABASES = {
    'default': {
        'HOST': 'pgbouncer-host',
        'PORT': '6432',  # PgBouncer port, not Postgres
        ...
    }
}
# PgBouncer config: pool_mode = transaction (recommended)
```

---

## 4. Async Performance (Django 4.1+)

```python
# Async views — handle many concurrent I/O operations
import httpx
from django.http import JsonResponse

async def fetch_multiple_apis(request):
    async with httpx.AsyncClient() as client:
        # Fetch 3 APIs concurrently (not sequentially!)
        import asyncio
        results = await asyncio.gather(
            client.get('https://api.service-a.com/data'),
            client.get('https://api.service-b.com/data'),
            client.get('https://api.service-c.com/data'),
        )
    
    return JsonResponse({
        'service_a': results[0].json(),
        'service_b': results[1].json(),
        'service_c': results[2].json(),
    })

# Async ORM (Django 4.1+)
from django.http import JsonResponse

async def async_product_list(request):
    products = []
    async for product in Product.objects.filter(status='published'):
        products.append({'name': product.name, 'price': str(product.price)})
    return JsonResponse({'products': products})
```

---

## 5. Static Files & Media Optimization

```python
# settings.py (production)
# Serve static via CDN, not Django
STATIC_URL = 'https://cdn.yourdomain.com/static/'
STATICFILES_STORAGE = 'storages.backends.s3boto3.S3StaticStorage'

# Media files on S3
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
AWS_STORAGE_BUCKET_NAME = 'my-media-bucket'
AWS_S3_CUSTOM_DOMAIN = f'{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com'
AWS_S3_OBJECT_PARAMETERS = {'CacheControl': 'max-age=86400'}

# WhiteNoise for static (simpler alternative to S3 for static files)
# pip install whitenoise
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # right after SecurityMiddleware
    ...
]
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

---

## 6. Response Optimization

### Compression & ETags
```python
# settings.py
MIDDLEWARE = [
    'django.middleware.gzip.GZipMiddleware',  # compress responses > 200 bytes
    ...
]

# Conditional GET (304 Not Modified)
from django.views.decorators.http import condition
from django.utils.http import http_date

def latest_product_etag(request):
    return str(Product.objects.latest('updated_at').updated_at.timestamp())

@condition(etag_func=latest_product_etag)
def product_list(request):
    # Only runs if ETag changed — otherwise returns 304
    ...
```

### Streaming Large Responses
```python
from django.http import StreamingHttpResponse
import csv

def export_orders_csv(request):
    """Stream 1M rows without loading all into memory."""
    orders = Order.objects.all().iterator(chunk_size=1000)
    
    def generate():
        yield 'id,user,total,status,created_at\n'
        for order in orders:
            yield f'{order.id},{order.user_id},{order.total},{order.status},{order.created_at}\n'
    
    response = StreamingHttpResponse(generate(), content_type='text/csv')
    response['Content-Disposition'] = 'attachment; filename="orders.csv"'
    return response
```

---

## 7. Monitoring & Profiling

### Application Performance Monitoring
```python
# Sentry (error tracking + performance)
# pip install sentry-sdk
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

sentry_sdk.init(
    dsn=os.environ['SENTRY_DSN'],
    integrations=[DjangoIntegration()],
    traces_sample_rate=0.1,  # 10% of requests traced
    profiles_sample_rate=0.1,
)

# Custom performance span
from sentry_sdk import start_span

def process_order(order):
    with start_span(op='process_order', description=f'Order #{order.id}'):
        validate_inventory(order)
        charge_payment(order)
        send_confirmation(order)
```

### Health Check Endpoint
```python
from django.http import JsonResponse
from django.db import connection
from django.core.cache import cache
import time

def health_check(request):
    checks = {}
    
    # Database check
    try:
        start = time.time()
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        checks['database'] = {'status': 'ok', 'latency_ms': round((time.time() - start) * 1000)}
    except Exception as e:
        checks['database'] = {'status': 'error', 'message': str(e)}
    
    # Redis check
    try:
        start = time.time()
        cache.set('health_check', 'ok', timeout=10)
        assert cache.get('health_check') == 'ok'
        checks['redis'] = {'status': 'ok', 'latency_ms': round((time.time() - start) * 1000)}
    except Exception as e:
        checks['redis'] = {'status': 'error', 'message': str(e)}
    
    all_ok = all(c['status'] == 'ok' for c in checks.values())
    status_code = 200 if all_ok else 503
    
    return JsonResponse({'status': 'healthy' if all_ok else 'unhealthy', 'checks': checks}, status=status_code)
```

---

## 8. Interview Questions

### Q: How do you optimize a slow Django view?
```
1. Profile first: django-silk or django-debug-toolbar
2. Check query count: N+1 → select_related/prefetch_related
3. Check query speed: add database indexes
4. Add caching: Redis for expensive queries/computations
5. Pagination: never load unbounded querysets
6. defer()/only(): skip large columns you don't need
7. Async: if view calls external APIs, make it async
8. Background tasks: move heavy work to Celery
```

### Q: How do you invalidate cache correctly?
```
Strategy 1: Time-based (TTL) — simplest, slight staleness OK
Strategy 2: Event-based — invalidate on model save (signal or override save())
Strategy 3: Version-based — increment version key, old cache becomes orphan

The hardest problem: cache invalidation for list queries
  - Solution A: invalidate ALL list caches when any item changes
  - Solution B: short TTL (30s) for lists, longer TTL (5min) for detail views
  - Solution C: cache per-query-params key, invalidate matching keys via pattern delete
```

### Q: Database vs Redis — when to use which for caching?
```
Redis:
  - Session data (fast, TTL support)
  - Rate limiting counters (INCR is atomic)
  - Leaderboards (sorted sets)
  - Pub/sub (real-time notifications)
  - Short-lived data (< 1 hour TTL)

Database:
  - Data that must survive Redis restart
  - Complex queries on cached data
  - Data > 512MB per key

Rule: If it's OK to lose it, use Redis. If losing it breaks something, use DB.
```
