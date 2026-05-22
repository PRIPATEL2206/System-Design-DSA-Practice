# Django Middleware, Signals & Async — Senior Level

## 1. Custom Middleware

### How Middleware Works
```
Request Phase (top → bottom):
  SecurityMiddleware.process_request()
  SessionMiddleware.process_request()
  YourMiddleware.process_request()     ← your code here
  View executes
  
Response Phase (bottom → top):
  YourMiddleware.process_response()    ← your code here
  SessionMiddleware.process_response()
  SecurityMiddleware.process_response()
```

### Middleware Template (New-Style)
```python
import time
import logging
import uuid

logger = logging.getLogger(__name__)


class RequestTimingMiddleware:
    """Log request duration for every API call."""
    
    def __init__(self, get_response):
        self.get_response = get_response  # called once at startup
    
    def __call__(self, request):
        # --- REQUEST PHASE ---
        start_time = time.time()
        request.request_id = str(uuid.uuid4())[:8]
        
        response = self.get_response(request)  # call next middleware/view
        
        # --- RESPONSE PHASE ---
        duration = time.time() - start_time
        
        logger.info(
            f"[{request.request_id}] {request.method} {request.path} "
            f"→ {response.status_code} ({duration:.3f}s)"
        )
        
        # Add timing header
        response['X-Request-Duration'] = f'{duration:.3f}s'
        response['X-Request-ID'] = request.request_id
        
        # Alert if slow
        if duration > 2.0:
            logger.warning(f"SLOW REQUEST: {request.path} took {duration:.3f}s")
        
        return response


class MaintenanceModeMiddleware:
    """Return 503 for all requests when maintenance mode is active."""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        from django.core.cache import cache
        from django.http import JsonResponse
        
        if cache.get('maintenance_mode'):
            # Allow admin access
            if request.path.startswith('/admin/'):
                return self.get_response(request)
            
            return JsonResponse(
                {'error': 'Service under maintenance. Back shortly.'},
                status=503
            )
        
        return self.get_response(request)


class TenantMiddleware:
    """Multi-tenant: set tenant context from subdomain or header."""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Extract tenant from subdomain: tenant1.api.example.com
        host = request.get_host().split('.')[0]
        
        try:
            from apps.tenants.models import Tenant
            request.tenant = Tenant.objects.get(subdomain=host)
        except Tenant.DoesNotExist:
            from django.http import JsonResponse
            return JsonResponse({'error': 'Invalid tenant'}, status=404)
        
        response = self.get_response(request)
        return response
```

### Middleware with Exception Handling
```python
class ExceptionLoggingMiddleware:
    """Catch unhandled exceptions, log them, return clean JSON error."""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        return self.get_response(request)
    
    def process_exception(self, request, exception):
        """Called when view raises an unhandled exception."""
        import traceback
        from django.http import JsonResponse
        
        logger.error(
            f"Unhandled exception on {request.method} {request.path}: "
            f"{exception}\n{traceback.format_exc()}"
        )
        
        # Don't expose internals in production
        from django.conf import settings
        if settings.DEBUG:
            return JsonResponse({
                'error': str(exception),
                'traceback': traceback.format_exc()
            }, status=500)
        
        return JsonResponse(
            {'error': 'Internal server error'},
            status=500
        )
```

### Rate Limiting Middleware
```python
from django.core.cache import cache
from django.http import JsonResponse

class RateLimitMiddleware:
    """Simple IP-based rate limiting using Redis."""
    
    RATE_LIMIT = 100  # requests per window
    WINDOW = 60       # seconds
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        if request.path.startswith('/api/'):
            ip = self.get_client_ip(request)
            key = f'ratelimit:{ip}'
            
            current = cache.get(key, 0)
            if current >= self.RATE_LIMIT:
                return JsonResponse(
                    {'error': 'Rate limit exceeded. Try again later.'},
                    status=429,
                    headers={'Retry-After': str(self.WINDOW)}
                )
            
            cache.set(key, current + 1, timeout=self.WINDOW)
        
        return self.get_response(request)
    
    def get_client_ip(self, request):
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            return x_forwarded_for.split(',')[0].strip()
        return request.META.get('REMOTE_ADDR')
```

---

## 2. Django Signals

### Built-in Signals
```python
from django.db.models.signals import pre_save, post_save, pre_delete, post_delete
from django.dispatch import receiver
from django.contrib.auth.signals import user_logged_in, user_login_failed

# Post-save: trigger after model is saved
@receiver(post_save, sender=Order)
def order_created_handler(sender, instance, created, **kwargs):
    if created:
        # Send confirmation email (async)
        send_order_email.delay(instance.id)
        # Update analytics
        cache.incr('total_orders_today')

# Pre-save: validate/modify before saving
@receiver(pre_save, sender=Product)
def product_pre_save(sender, instance, **kwargs):
    # Auto-generate slug from name
    if not instance.slug:
        from django.utils.text import slugify
        instance.slug = slugify(instance.name)

# Post-delete: cleanup related resources
@receiver(post_delete, sender=UserProfile)
def cleanup_user_files(sender, instance, **kwargs):
    # Delete profile image from S3
    if instance.avatar:
        instance.avatar.delete(save=False)

# Auth signals
@receiver(user_logged_in)
def log_successful_login(sender, request, user, **kwargs):
    LoginAudit.objects.create(
        user=user,
        ip_address=get_client_ip(request),
        user_agent=request.META.get('HTTP_USER_AGENT', '')
    )

@receiver(user_login_failed)
def log_failed_login(sender, credentials, request, **kwargs):
    logger.warning(f"Failed login for {credentials.get('username')} from {get_client_ip(request)}")
```

### Custom Signals
```python
# signals.py
from django.dispatch import Signal

# Define custom signals
order_completed = Signal()  # args: order, user
payment_failed = Signal()   # args: order, error

# Emit signal
class OrderService:
    def complete_order(self, order):
        order.status = 'completed'
        order.save()
        
        # Emit signal — any handler can listen
        order_completed.send(
            sender=self.__class__,
            order=order,
            user=order.user
        )

# Handle signal (in different module — loose coupling!)
@receiver(order_completed)
def update_loyalty_points(sender, order, user, **kwargs):
    user.profile.loyalty_points += int(order.total)
    user.profile.save(update_fields=['loyalty_points'])

@receiver(order_completed)
def notify_warehouse(sender, order, **kwargs):
    notify_warehouse_system.delay(order.id)
```

### Signals: When to Use vs When to Avoid
```python
# ✓ USE signals for:
# - Audit logging (who did what, when)
# - Cache invalidation on model changes
# - Sending notifications (async via Celery)
# - Loose coupling between apps

# ❌ AVOID signals for:
# - Business logic that MUST happen (use service layer instead)
# - Anything that should be transactional with the save
# - Complex chains of signals (debugging nightmare)
# - Performance-critical paths (signals add overhead)

# ❌ BAD: Critical logic in signal (invisible, hard to debug)
@receiver(post_save, sender=Order)
def deduct_inventory(sender, instance, created, **kwargs):
    if created:
        instance.product.stock -= instance.quantity
        instance.product.save()
    # Problem: if this fails, order is saved but stock isn't deducted
    # This should be in a service with transaction.atomic()

# ✓ GOOD: Use service layer for critical logic
class OrderService:
    @transaction.atomic
    def create_order(self, user, product, quantity):
        product = Product.objects.select_for_update().get(id=product.id)
        if product.stock < quantity:
            raise InsufficientStockError()
        product.stock -= quantity
        product.save()
        order = Order.objects.create(user=user, product=product, quantity=quantity)
        return order
```

---

## 3. Celery (Async Task Queue)

### Setup
```python
# celery.py (project root, same level as settings.py)
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

app = Celery('myproject')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()

# settings.py
CELERY_BROKER_URL = 'redis://redis:6379/0'
CELERY_RESULT_BACKEND = 'redis://redis:6379/1'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_TIME_LIMIT = 300  # hard kill after 5 min
CELERY_TASK_SOFT_TIME_LIMIT = 240  # raise SoftTimeLimitExceeded after 4 min
```

### Task Patterns
```python
# tasks.py
from celery import shared_task
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)

# Basic task
@shared_task
def send_order_email(order_id):
    order = Order.objects.select_related('user').get(id=order_id)
    send_email(
        to=order.user.email,
        subject=f'Order #{order.id} confirmed',
        body=render_order_email(order)
    )

# Task with retry (for transient failures)
@shared_task(
    bind=True,
    max_retries=3,
    default_retry_delay=60,  # wait 60s before retry
)
def charge_payment(self, order_id):
    try:
        order = Order.objects.get(id=order_id)
        result = payment_gateway.charge(order.total, order.payment_token)
        order.payment_status = 'charged'
        order.save()
    except PaymentGatewayTimeout as exc:
        # Retry with exponential backoff
        raise self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))
    except PaymentDeclined:
        order.payment_status = 'declined'
        order.save()
        notify_user_payment_failed.delay(order_id)

# Periodic tasks (cron-like)
from celery.schedules import crontab

# settings.py
CELERY_BEAT_SCHEDULE = {
    'cleanup-expired-sessions': {
        'task': 'accounts.tasks.cleanup_sessions',
        'schedule': crontab(hour=3, minute=0),  # daily at 3am
    },
    'generate-daily-report': {
        'task': 'analytics.tasks.daily_report',
        'schedule': crontab(hour=8, minute=0, day_of_week='mon-fri'),
    },
    'sync-inventory': {
        'task': 'products.tasks.sync_inventory',
        'schedule': 300.0,  # every 5 minutes
    },
}

# Task with rate limiting
@shared_task(rate_limit='10/m')  # max 10 calls per minute
def call_external_api(item_id):
    response = requests.get(f'https://api.example.com/items/{item_id}')
    return response.json()

# Chain tasks (run sequentially)
from celery import chain
chain(
    validate_order.s(order_id),
    charge_payment.s(),
    send_confirmation.s()
).delay()

# Group tasks (run in parallel)
from celery import group
group(
    send_email.s(user_id) for user_id in user_ids
).delay()
```

---

## 4. Django Channels (WebSocket)

### Setup for Real-Time
```python
# pip install channels channels-redis

# settings.py
INSTALLED_APPS = [..., 'channels']
ASGI_APPLICATION = 'myproject.asgi.application'
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            'hosts': [('redis', 6379)],
        },
    },
}

# asgi.py
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

application = ProtocolTypeRouter({
    'http': get_asgi_application(),
    'websocket': AuthMiddlewareStack(
        URLRouter([
            path('ws/notifications/', NotificationConsumer.as_asgi()),
            path('ws/orders/<str:order_id>/', OrderTrackingConsumer.as_asgi()),
        ])
    ),
})
```

### WebSocket Consumer
```python
# consumers.py
from channels.generic.websocket import AsyncJsonWebsocketConsumer

class OrderTrackingConsumer(AsyncJsonWebsocketConsumer):
    async def connect(self):
        self.order_id = self.scope['url_route']['kwargs']['order_id']
        self.group_name = f'order_{self.order_id}'
        
        # Join order tracking group
        await self.channel_layer.group_add(self.group_name, self.channel_name)
        await self.accept()
    
    async def disconnect(self, code):
        await self.channel_layer.group_discard(self.group_name, self.channel_name)
    
    async def receive_json(self, content):
        # Handle incoming message from client
        pass
    
    # Handler for messages sent to this group
    async def order_status_update(self, event):
        await self.send_json({
            'type': 'status_update',
            'order_id': event['order_id'],
            'status': event['status'],
            'timestamp': event['timestamp'],
        })


# Sending updates from anywhere in Django (e.g., from a Celery task)
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync

def notify_order_status(order_id, new_status):
    channel_layer = get_channel_layer()
    async_to_sync(channel_layer.group_send)(
        f'order_{order_id}',
        {
            'type': 'order_status_update',  # maps to method name
            'order_id': str(order_id),
            'status': new_status,
            'timestamp': str(timezone.now()),
        }
    )
```

---

## 5. Management Commands

```python
# management/commands/process_pending_orders.py
from django.core.management.base import BaseCommand
from django.db import transaction

class Command(BaseCommand):
    help = 'Process all pending orders older than 1 hour'

    def add_arguments(self, parser):
        parser.add_argument('--dry-run', action='store_true', help='Show what would be processed')
        parser.add_argument('--batch-size', type=int, default=100, help='Batch size')

    def handle(self, *args, **options):
        from apps.orders.models import Order
        from datetime import timedelta
        from django.utils import timezone

        cutoff = timezone.now() - timedelta(hours=1)
        pending = Order.objects.filter(status='pending', created_at__lt=cutoff)
        
        total = pending.count()
        self.stdout.write(f'Found {total} pending orders')
        
        if options['dry_run']:
            self.stdout.write(self.style.WARNING('DRY RUN — no changes made'))
            return
        
        processed = 0
        for batch in self.chunked_queryset(pending, options['batch_size']):
            with transaction.atomic():
                for order in batch:
                    order.status = 'cancelled'
                    order.save(update_fields=['status'])
                    processed += 1
            
            self.stdout.write(f'Processed {processed}/{total}')
        
        self.stdout.write(self.style.SUCCESS(f'Done. Cancelled {processed} orders.'))
    
    def chunked_queryset(self, queryset, chunk_size):
        start = 0
        while True:
            chunk = list(queryset[start:start + chunk_size])
            if not chunk:
                break
            yield chunk
            start += chunk_size

# Usage:
# python manage.py process_pending_orders --dry-run
# python manage.py process_pending_orders --batch-size 50
```

---

## 6. Interview Questions

### Q: When would you use signals vs Celery vs middleware?
```
Signals:   Reactive to model changes (post_save, pre_delete)
           Synchronous (blocks the request unless you call .delay())
           Use for: cache invalidation, audit logs, simple notifications

Celery:    Background processing (doesn't block the request)
           Retries, scheduling, rate limiting built-in
           Use for: emails, payments, reports, heavy computation

Middleware: Runs on EVERY request/response
            Use for: logging, auth, rate limiting, request/response transformation
            NOT for business logic specific to one endpoint
```

### Q: How do you handle long-running tasks in Django?
```
1. Celery task: serialize work to queue, return immediately
2. WebSocket: push result to client when done (Django Channels)
3. Polling: client checks GET /task/{id}/status every 2s
4. SSE: server-sent events for one-way updates
5. Webhook: call client's URL when done (for server-to-server)
```
