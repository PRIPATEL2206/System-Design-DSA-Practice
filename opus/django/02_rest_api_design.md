# Django REST Framework & API Design — Senior Level

## 1. Serializer Patterns

### Basic vs Model Serializer
```python
from rest_framework import serializers
from .models import Product, Category, Order

# ModelSerializer — auto-generates fields from model
class ProductSerializer(serializers.ModelSerializer):
    category_name = serializers.CharField(source='category.name', read_only=True)
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'price', 'status', 'category', 'category_name', 'created_at']
        read_only_fields = ['id', 'created_at']
        extra_kwargs = {
            'price': {'min_value': 0},
            'name': {'min_length': 3},
        }
```

### Nested Serializers (Read vs Write)
```python
class OrderItemSerializer(serializers.ModelSerializer):
    product_name = serializers.CharField(source='product.name', read_only=True)
    
    class Meta:
        model = OrderItem
        fields = ['id', 'product', 'product_name', 'quantity', 'price']


class OrderReadSerializer(serializers.ModelSerializer):
    """For GET requests — nested, readable."""
    items = OrderItemSerializer(many=True, read_only=True)
    user = UserMinimalSerializer(read_only=True)
    total_items = serializers.SerializerMethodField()
    
    class Meta:
        model = Order
        fields = ['id', 'user', 'items', 'total', 'status', 'total_items', 'created_at']
    
    def get_total_items(self, obj):
        return obj.items.count()


class OrderWriteSerializer(serializers.ModelSerializer):
    """For POST/PUT — flat, writable."""
    items = OrderItemSerializer(many=True)
    
    class Meta:
        model = Order
        fields = ['items', 'shipping_address']
    
    def create(self, validated_data):
        items_data = validated_data.pop('items')
        user = self.context['request'].user
        
        with transaction.atomic():
            order = Order.objects.create(user=user, **validated_data)
            total = 0
            for item_data in items_data:
                item = OrderItem.objects.create(order=order, **item_data)
                total += item.price * item.quantity
            order.total = total
            order.save(update_fields=['total'])
        
        return order
```

### Custom Validation
```python
class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = '__all__'
    
    # Field-level validation
    def validate_price(self, value):
        if value <= 0:
            raise serializers.ValidationError("Price must be positive.")
        return value
    
    # Object-level validation (cross-field)
    def validate(self, data):
        if data.get('sale_price') and data['sale_price'] >= data['price']:
            raise serializers.ValidationError({
                'sale_price': 'Sale price must be less than regular price.'
            })
        return data
    
    # Unique together validation
    class Meta:
        model = Product
        fields = '__all__'
        validators = [
            serializers.UniqueTogetherValidator(
                queryset=Product.objects.all(),
                fields=['name', 'category'],
                message='Product name must be unique within category.'
            )
        ]
```

---

## 2. ViewSets & Views

### ModelViewSet (Full CRUD)
```python
from rest_framework import viewsets, permissions, filters, status
from rest_framework.decorators import action
from rest_framework.response import Response
from django_filters.rest_framework import DjangoFilterBackend

class ProductViewSet(viewsets.ModelViewSet):
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['category', 'status']
    search_fields = ['name', 'description']
    ordering_fields = ['price', 'created_at']
    ordering = ['-created_at']
    
    def get_queryset(self):
        qs = Product.objects.select_related('category').prefetch_related('tags')
        if self.request.user.is_staff:
            return qs
        return qs.filter(status='published')
    
    def get_serializer_class(self):
        if self.action in ['list', 'retrieve']:
            return ProductReadSerializer
        return ProductWriteSerializer
    
    # Custom action: POST /products/{id}/publish/
    @action(detail=True, methods=['post'], permission_classes=[permissions.IsAdminUser])
    def publish(self, request, pk=None):
        product = self.get_object()
        product.status = 'published'
        product.save(update_fields=['status'])
        return Response({'status': 'published'})
    
    # Custom list action: GET /products/trending/
    @action(detail=False, methods=['get'])
    def trending(self, request):
        trending = self.get_queryset().filter(
            status='published'
        ).order_by('-view_count')[:10]
        serializer = self.get_serializer(trending, many=True)
        return Response(serializer.data)
```

### APIView (When You Need Full Control)
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class OrderCreateView(APIView):
    permission_classes = [permissions.IsAuthenticated]
    throttle_classes = [UserRateThrottle]
    
    def post(self, request):
        serializer = OrderWriteSerializer(
            data=request.data,
            context={'request': request}
        )
        serializer.is_valid(raise_exception=True)
        order = serializer.save()
        
        # Trigger async task
        send_order_confirmation.delay(order.id)
        
        return Response(
            OrderReadSerializer(order).data,
            status=status.HTTP_201_CREATED
        )
```

---

## 3. Authentication & Permissions

### JWT Authentication (SimpleJWT)
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=30),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,
}

# urls.py
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view()),
    path('api/token/refresh/', TokenRefreshView.as_view()),
]
```

### Custom Permissions
```python
from rest_framework.permissions import BasePermission

class IsOwnerOrAdmin(BasePermission):
    """Object-level permission: only owner or admin can modify."""
    
    def has_object_permission(self, request, view, obj):
        if request.method in ('GET', 'HEAD', 'OPTIONS'):
            return True
        return obj.user == request.user or request.user.is_staff


class IsVerifiedUser(BasePermission):
    """Only email-verified users can access."""
    message = 'Email verification required.'
    
    def has_permission(self, request, view):
        return (
            request.user.is_authenticated and 
            request.user.profile.is_email_verified
        )


# Usage in ViewSet
class OrderViewSet(viewsets.ModelViewSet):
    permission_classes = [IsVerifiedUser, IsOwnerOrAdmin]
```

---

## 4. Pagination

### Cursor Pagination (Best for Large Datasets)
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.CursorPagination',
    'PAGE_SIZE': 20,
}

# Custom pagination
from rest_framework.pagination import CursorPagination, PageNumberPagination

class ProductCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'  # must be a unique, sequential field
    cursor_query_param = 'cursor'

class StandardPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100

# In ViewSet
class ProductViewSet(viewsets.ModelViewSet):
    pagination_class = ProductCursorPagination
```

### When to Use Which Pagination
```
PageNumber:  Simple, allows jumping to page 5
             Problem: OFFSET is slow on large tables (OFFSET 10000 scans 10000 rows)
             Use for: small datasets (<100K rows), admin panels

Limit/Offset: Same as PageNumber but API-style (?limit=20&offset=40)
              Same OFFSET performance problem
              Use for: same as PageNumber

Cursor:      Encodes position in opaque cursor token
             Always O(1) regardless of dataset size (uses WHERE id < X)
             Can't jump to arbitrary pages
             Use for: infinite scroll, large datasets, real-time feeds
```

---

## 5. Throttling & Rate Limiting

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
        'login': '5/minute',
        'order_create': '10/minute',
    },
}

# Custom throttle for specific endpoints
from rest_framework.throttling import UserRateThrottle

class LoginRateThrottle(UserRateThrottle):
    scope = 'login'

class OrderCreateThrottle(UserRateThrottle):
    scope = 'order_create'

# Apply to view
class LoginView(APIView):
    throttle_classes = [LoginRateThrottle]
    
    def post(self, request):
        # login logic...
        pass
```

---

## 6. API Versioning

### URL Path Versioning (Most Common)
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
}

# urls.py
urlpatterns = [
    path('api/v1/', include('api.v1.urls')),
    path('api/v2/', include('api.v2.urls')),
]

# Or version-aware views
class ProductViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        if self.request.version == 'v2':
            return ProductV2Serializer
        return ProductV1Serializer
```

---

## 7. Error Handling

### Custom Exception Handler
```python
# exceptions.py
from rest_framework.views import exception_handler
from rest_framework.exceptions import APIException
from rest_framework import status

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    if response is not None:
        response.data = {
            'error': {
                'code': response.status_code,
                'message': get_error_message(exc),
                'details': response.data if isinstance(response.data, dict) else {'errors': response.data}
            }
        }
    
    return response

def get_error_message(exc):
    if hasattr(exc, 'detail'):
        if isinstance(exc.detail, str):
            return exc.detail
        return 'Validation failed'
    return str(exc)

# Custom exception classes
class InsufficientStockError(APIException):
    status_code = status.HTTP_409_CONFLICT
    default_detail = 'Insufficient stock for requested quantity.'
    default_code = 'insufficient_stock'

# settings.py
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'core.exceptions.custom_exception_handler',
}
```

---

## 8. Signals for Side Effects

### Using Signals with DRF
```python
# signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from .models import Order

@receiver(post_save, sender=Order)
def order_post_save(sender, instance, created, **kwargs):
    if created:
        send_order_notification.delay(instance.id)
    elif instance.status == 'shipped':
        send_shipping_notification.delay(instance.id)
```

---

## 9. URL Configuration

### Router Setup
```python
# urls.py
from rest_framework.routers import DefaultRouter
from .views import ProductViewSet, OrderViewSet, UserViewSet

router = DefaultRouter()
router.register(r'products', ProductViewSet, basename='product')
router.register(r'orders', OrderViewSet, basename='order')
router.register(r'users', UserViewSet, basename='user')

urlpatterns = [
    path('api/v1/', include(router.urls)),
    path('api/v1/auth/', include('authentication.urls')),
]

# Generated URLs:
# GET/POST     /api/v1/products/
# GET/PUT/DEL  /api/v1/products/{id}/
# POST         /api/v1/products/{id}/publish/   (custom action)
# GET          /api/v1/products/trending/       (custom list action)
```

---

## 10. Interview Questions

### Q: How do you handle different serializers for read vs write in DRF?
```
Override get_serializer_class() in ViewSet:
- list/retrieve actions → ReadSerializer (nested, verbose)
- create/update actions → WriteSerializer (flat, writable)

This keeps responses rich for frontend while keeping writes simple.
```

### Q: How do you optimize DRF for high traffic?
```
1. Proper select_related/prefetch_related in get_queryset()
2. Cursor pagination (not page number) for large datasets
3. Cache list endpoints with Redis (cache per user or per query params)
4. Use .only() to select specific fields
5. Throttle to prevent abuse
6. Async tasks (Celery) for heavy operations in create/update
7. Read replicas for GET endpoints
```

### Q: How do you handle API versioning when schema changes?
```
1. URL-based versioning: /api/v1/ and /api/v2/
2. Keep v1 working for 6-12 months (deprecation period)
3. Version-specific serializers with get_serializer_class()
4. Document breaking changes in changelog
5. Add Deprecation header to old version responses
6. Monitor old version usage → sunset when < 5% traffic
```
