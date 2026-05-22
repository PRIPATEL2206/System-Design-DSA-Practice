# Django Real-World Project Structure — Senior Level

## 1. Production Project Layout

```
myproject/
├── config/                          # Project configuration (replaces myproject/)
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py                 # Shared settings
│   │   ├── development.py          # Local overrides
│   │   ├── staging.py
│   │   ├── production.py
│   │   └── test.py                 # Test-specific settings
│   ├── urls.py                     # Root URL configuration
│   ├── wsgi.py
│   ├── asgi.py
│   └── celery.py                   # Celery app definition
│
├── apps/                            # All Django apps
│   ├── __init__.py
│   ├── accounts/                   # User management
│   │   ├── __init__.py
│   │   ├── models.py
│   │   ├── admin.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   ├── services.py            # Business logic (NOT in views)
│   │   ├── selectors.py           # Complex read queries
│   │   ├── tasks.py               # Celery tasks
│   │   ├── signals.py
│   │   ├── permissions.py
│   │   ├── validators.py
│   │   ├── exceptions.py
│   │   └── tests/
│   │       ├── __init__.py
│   │       ├── conftest.py
│   │       ├── factories.py
│   │       ├── test_models.py
│   │       ├── test_views.py
│   │       └── test_services.py
│   │
│   ├── products/                   # Product catalog
│   │   ├── ... (same structure)
│   │   └── management/
│   │       └── commands/
│   │           └── import_products.py
│   │
│   ├── orders/                     # Order processing
│   │   └── ... (same structure)
│   │
│   └── common/                     # Shared utilities
│       ├── __init__.py
│       ├── models.py               # TimeStampedModel, SoftDeleteModel
│       ├── mixins.py               # Shared view/serializer mixins
│       ├── pagination.py           # Custom pagination classes
│       ├── permissions.py          # Shared permissions
│       ├── exceptions.py           # Custom exception handler
│       └── utils.py                # Helper functions
│
├── templates/                       # HTML templates (if any)
│   ├── base.html
│   └── emails/
│
├── static/                          # Static files (CSS, JS)
│
├── media/                           # User uploads (local dev only)
│
├── scripts/                         # One-off scripts, data migrations
│   ├── backfill_slugs.py
│   └── export_data.py
│
├── docs/                            # Project documentation
│   ├── api.md
│   └── architecture.md
│
├── docker/                          # Docker configs
│   ├── Dockerfile
│   ├── Dockerfile.celery
│   └── nginx.conf
│
├── .github/
│   └── workflows/
│       ├── test.yml
│       └── deploy.yml
│
├── docker-compose.yml
├── docker-compose.prod.yml
├── manage.py
├── requirements/
│   ├── base.txt
│   ├── development.txt
│   ├── production.txt
│   └── test.txt
├── pyproject.toml                   # ruff, pytest, black config
├── Makefile                         # common commands
├── .env.example
└── .gitignore
```

---

## 2. Service Layer Pattern (Keep Views Thin)

### Why Service Layer?
```python
# ❌ BAD: Fat view with business logic
class OrderCreateView(APIView):
    def post(self, request):
        product = Product.objects.get(id=request.data['product_id'])
        if product.stock < request.data['quantity']:
            return Response({'error': 'Out of stock'}, status=400)
        product.stock -= request.data['quantity']
        product.save()
        order = Order.objects.create(user=request.user, total=product.price * quantity)
        send_email(request.user.email, 'Order confirmed', ...)
        return Response(OrderSerializer(order).data, status=201)

# Problem: untestable, not reusable, hard to maintain
# What if CLI command also needs to create orders? Duplicate logic.

# ✓ GOOD: Thin view → Service handles logic
class OrderCreateView(APIView):
    def post(self, request):
        serializer = OrderCreateSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        
        order = OrderService.create_order(
            user=request.user,
            **serializer.validated_data
        )
        
        return Response(OrderReadSerializer(order).data, status=201)
```

### Service Implementation
```python
# apps/orders/services.py
from django.db import transaction
from apps.products.models import Product
from apps.orders.models import Order, OrderItem
from apps.common.exceptions import InsufficientStockError
from .tasks import send_order_confirmation

class OrderService:
    
    @staticmethod
    @transaction.atomic
    def create_order(user, items):
        """
        Create an order with stock validation and payment.
        
        Args:
            user: User placing the order
            items: list of {'product_id': uuid, 'quantity': int}
        
        Returns:
            Order instance
        
        Raises:
            InsufficientStockError: if any product is out of stock
        """
        order = Order.objects.create(user=user, status='pending', total=0)
        total = 0
        
        for item in items:
            product = Product.objects.select_for_update().get(id=item['product_id'])
            
            if product.stock < item['quantity']:
                raise InsufficientStockError(
                    f'{product.name}: requested {item["quantity"]}, available {product.stock}'
                )
            
            product.stock -= item['quantity']
            product.save(update_fields=['stock'])
            
            OrderItem.objects.create(
                order=order,
                product=product,
                quantity=item['quantity'],
                price=product.price
            )
            total += product.price * item['quantity']
        
        order.total = total
        order.save(update_fields=['total'])
        
        # Async side effects (not part of transaction)
        transaction.on_commit(lambda: send_order_confirmation.delay(order.id))
        
        return order
    
    @staticmethod
    def cancel_order(order):
        if order.status not in ('pending', 'paid'):
            raise ValueError(f'Cannot cancel order in {order.status} status')
        
        with transaction.atomic():
            for item in order.items.select_related('product'):
                Product.objects.filter(id=item.product_id).update(
                    stock=F('stock') + item.quantity
                )
            order.status = 'cancelled'
            order.save(update_fields=['status'])
        
        return order
```

### Selectors (Complex Read Queries)
```python
# apps/orders/selectors.py
from django.db.models import Sum, Count, Q, F
from django.utils import timezone
from datetime import timedelta

class OrderSelector:
    
    @staticmethod
    def get_user_orders(user, status=None):
        qs = Order.objects.filter(user=user).select_related(
            'user'
        ).prefetch_related(
            'items__product'
        ).order_by('-created_at')
        
        if status:
            qs = qs.filter(status=status)
        return qs
    
    @staticmethod
    def get_revenue_stats(days=30):
        cutoff = timezone.now() - timedelta(days=days)
        return Order.objects.filter(
            status='paid',
            created_at__gte=cutoff
        ).aggregate(
            total_revenue=Sum('total'),
            order_count=Count('id'),
            avg_order_value=Sum('total') / Count('id'),
        )
    
    @staticmethod
    def get_top_customers(limit=10):
        return User.objects.annotate(
            total_spent=Sum('orders__total', filter=Q(orders__status='paid')),
            order_count=Count('orders', filter=Q(orders__status='paid'))
        ).filter(
            total_spent__isnull=False
        ).order_by('-total_spent')[:limit]
```

---

## 3. Settings Organization

```python
# config/settings/base.py
import os
from pathlib import Path
import environ

env = environ.Env()
BASE_DIR = Path(__file__).resolve().parent.parent.parent
environ.Env.read_env(os.path.join(BASE_DIR, '.env'))

SECRET_KEY = env('SECRET_KEY')
DEBUG = env.bool('DEBUG', default=False)

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # Third party
    'rest_framework',
    'django_filters',
    'corsheaders',
    # Local apps
    'apps.common',
    'apps.accounts',
    'apps.products',
    'apps.orders',
]

AUTH_USER_MODEL = 'accounts.User'

# DRF
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_PAGINATION_CLASS': 'apps.common.pagination.StandardPagination',
    'PAGE_SIZE': 20,
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
    'EXCEPTION_HANDLER': 'apps.common.exceptions.custom_exception_handler',
}


# config/settings/production.py
from .base import *

DEBUG = False
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')

DATABASES = {
    'default': env.db('DATABASE_URL'),
    'replica': env.db('DATABASE_REPLICA_URL'),
}
DATABASES['default']['CONN_MAX_AGE'] = 600

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': env('REDIS_URL'),
    }
}

# Security
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

# Static/Media on S3
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
STATICFILES_STORAGE = 'storages.backends.s3boto3.S3StaticStorage'
AWS_STORAGE_BUCKET_NAME = env('AWS_S3_BUCKET')
```

---

## 4. Makefile (Developer UX)

```makefile
.PHONY: help run test migrate shell

help:
	@echo "Available commands:"
	@echo "  make run         - Start development server"
	@echo "  make test        - Run all tests"
	@echo "  make migrate     - Run migrations"
	@echo "  make shell       - Django shell"
	@echo "  make docker-up   - Start all containers"
	@echo "  make lint        - Run linting"

run:
	python manage.py runserver

test:
	pytest --cov=apps -n auto

test-fast:
	pytest -x --no-header -q

migrate:
	python manage.py makemigrations
	python manage.py migrate

shell:
	python manage.py shell_plus

lint:
	ruff check apps/
	ruff format --check apps/

format:
	ruff format apps/

docker-up:
	docker compose up -d

docker-down:
	docker compose down

docker-logs:
	docker compose logs -f web

celery:
	celery -A config worker -l info

superuser:
	python manage.py createsuperuser
```

---

## 5. Requirements Organization

```
# requirements/base.txt
Django==5.0
djangorestframework==3.14
django-filter==23.5
django-cors-headers==4.3
django-environ==0.11
django-redis==5.4
django-storages[boto3]==1.14
celery[redis]==5.3
psycopg2-binary==2.9
gunicorn==21.2
whitenoise==6.6
sentry-sdk==1.40

# requirements/development.txt
-r base.txt
django-debug-toolbar==4.3
django-extensions==3.2
ipython==8.20
factory-boy==3.3

# requirements/test.txt
-r base.txt
pytest==8.0
pytest-django==4.8
pytest-cov==4.1
pytest-xdist==3.5
factory-boy==3.3
responses==0.25
freezegun==1.4

# requirements/production.txt
-r base.txt
boto3==1.34
django-health-check==3.18
```

---

## 6. URL Namespacing

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/v1/auth/', include('apps.accounts.urls', namespace='accounts')),
    path('api/v1/products/', include('apps.products.urls', namespace='products')),
    path('api/v1/orders/', include('apps.orders.urls', namespace='orders')),
    path('health/', include('health_check.urls')),
]

# apps/orders/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import OrderViewSet

app_name = 'orders'

router = DefaultRouter()
router.register('', OrderViewSet, basename='order')

urlpatterns = [
    path('', include(router.urls)),
]
```

---

## 7. Interview Questions

### Q: How do you structure a Django project for a team of 5-10 developers?
```
1. Apps by domain (accounts, orders, products), NOT by layer
2. Service layer separates business logic from views
3. Selectors for complex read queries
4. Shared utilities in common/ app
5. Settings split by environment (base/dev/prod/test)
6. Requirements split by environment
7. Makefile for common commands (DX)
8. Each app has its own tests/ directory with factories

Why: parallel development (each dev owns an app), clear boundaries,
easy to test in isolation, can later extract into microservice.
```

### Q: Where does business logic go in Django?
```
❌ In views (untestable, not reusable)
❌ In models (models become 1000+ lines, God Object anti-pattern)
❌ In serializers (serialization ≠ business logic)

✓ In services.py:
  - Write operations (create, update, cancel)
  - Multi-model transactions
  - External API calls
  - Complex validation

✓ In selectors.py:
  - Complex read queries
  - Aggregations, reports
  - Reusable query compositions

Views are THIN: validate input → call service → serialize output
```
