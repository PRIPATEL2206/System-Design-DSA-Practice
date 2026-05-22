# Django REST Framework — All CRUD Approaches (Complete Reference)

> Every possible way to build CRUD APIs in Django REST Framework.

---

## Table of Contents

1. [Approach 1: Function-Based Views (FBV)](#approach-1)
2. [Approach 2: Class-Based APIView](#approach-2)
3. [Approach 3: Generic Views (ListCreateAPIView, etc.)](#approach-3)
4. [Approach 4: ModelViewSet (Most Common)](#approach-4)
5. [Approach 5: ViewSet with Custom Actions](#approach-5)
6. [Approach 6: Mixins (Pick & Choose)](#approach-6)
7. [Approach 7: Nested Resources (Router)](#approach-7)
8. [Approach 8: Filtering, Searching, Ordering](#approach-8)
9. [Approach 9: Permissions & Authentication](#approach-9)
10. [Approach 10: Full Production Setup](#approach-10)

---

## Common Setup (Used Across All Approaches)

```python
# ========== models.py ==========
from django.db import models
from django.contrib.auth.models import User

class Product(models.Model):
    CATEGORY_CHOICES = [
        ('electronics', 'Electronics'),
        ('clothing', 'Clothing'),
        ('food', 'Food'),
        ('books', 'Books'),
    ]
    
    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    category = models.CharField(max_length=50, choices=CATEGORY_CHOICES)
    stock = models.PositiveIntegerField(default=0)
    is_active = models.BooleanField(default=True)
    created_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        ordering = ['-created_at']
    
    def __str__(self):
        return self.name

class Review(models.Model):
    product = models.ForeignKey(Product, on_delete=models.CASCADE, related_name='reviews')
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    rating = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
    comment = models.TextField(max_length=500)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ['product', 'user']


# ========== serializers.py ==========
from rest_framework import serializers
from .models import Product, Review

class ReviewSerializer(serializers.ModelSerializer):
    user_name = serializers.CharField(source='user.get_full_name', read_only=True)
    
    class Meta:
        model = Review
        fields = ['id', 'rating', 'comment', 'user_name', 'created_at']
        read_only_fields = ['id', 'created_at']

class ProductListSerializer(serializers.ModelSerializer):
    """Lightweight serializer for list view."""
    review_count = serializers.IntegerField(read_only=True)
    avg_rating = serializers.FloatField(read_only=True)
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'price', 'category', 'stock', 
                  'review_count', 'avg_rating', 'created_at']

class ProductDetailSerializer(serializers.ModelSerializer):
    """Full serializer for detail view."""
    reviews = ReviewSerializer(many=True, read_only=True)
    created_by_name = serializers.CharField(source='created_by.get_full_name', read_only=True)
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'description', 'price', 'category', 'stock',
                  'is_active', 'reviews', 'created_by_name', 'created_at', 'updated_at']

class ProductCreateSerializer(serializers.ModelSerializer):
    """Serializer for creating/updating products."""
    
    class Meta:
        model = Product
        fields = ['name', 'description', 'price', 'category', 'stock']
    
    def validate_price(self, value):
        if value <= 0:
            raise serializers.ValidationError("Price must be positive")
        return value
    
    def validate_name(self, value):
        # Check unique name (exclude current instance on update)
        qs = Product.objects.filter(name__iexact=value)
        if self.instance:
            qs = qs.exclude(pk=self.instance.pk)
        if qs.exists():
            raise serializers.ValidationError("Product with this name already exists")
        return value
```

---

## Approach 1: Function-Based Views (FBV) {#approach-1}

> Simplest approach — explicit and easy to understand.

```python
# ========== views.py ==========
from rest_framework.decorators import api_view, permission_classes
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated
from .models import Product
from .serializers import ProductCreateSerializer, ProductDetailSerializer, ProductListSerializer

@api_view(['GET', 'POST'])
@permission_classes([IsAuthenticated])
def product_list_create(request):
    """
    GET  /api/products/     → List all products
    POST /api/products/     → Create a product
    """
    if request.method == 'GET':
        products = Product.objects.filter(is_active=True)
        
        # Optional filtering
        category = request.query_params.get('category')
        if category:
            products = products.filter(category=category)
        
        # Pagination (manual)
        page = int(request.query_params.get('page', 1))
        page_size = int(request.query_params.get('page_size', 20))
        start = (page - 1) * page_size
        end = start + page_size
        
        serializer = ProductListSerializer(products[start:end], many=True)
        return Response({
            'results': serializer.data,
            'count': products.count(),
            'page': page
        })
    
    elif request.method == 'POST':
        serializer = ProductCreateSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(created_by=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


@api_view(['GET', 'PUT', 'PATCH', 'DELETE'])
@permission_classes([IsAuthenticated])
def product_detail(request, pk):
    """
    GET    /api/products/<pk>/  → Get single product
    PUT    /api/products/<pk>/  → Full update
    PATCH  /api/products/<pk>/  → Partial update
    DELETE /api/products/<pk>/  → Delete product
    """
    try:
        product = Product.objects.get(pk=pk)
    except Product.DoesNotExist:
        return Response({'error': 'Not found'}, status=status.HTTP_404_NOT_FOUND)
    
    if request.method == 'GET':
        serializer = ProductDetailSerializer(product)
        return Response(serializer.data)
    
    elif request.method == 'PUT':
        serializer = ProductCreateSerializer(product, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    elif request.method == 'PATCH':
        serializer = ProductCreateSerializer(product, data=request.data, partial=True)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    elif request.method == 'DELETE':
        product.is_active = False  # Soft delete
        product.save()
        return Response(status=status.HTTP_204_NO_CONTENT)


# ========== urls.py ==========
from django.urls import path
from . import views

urlpatterns = [
    path('products/', views.product_list_create),
    path('products/<int:pk>/', views.product_detail),
]
```

---

## Approach 2: Class-Based APIView {#approach-2}

> More organized — separates HTTP methods into class methods.

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated
from django.shortcuts import get_object_or_404

class ProductListCreateView(APIView):
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        """List all products."""
        products = Product.objects.filter(is_active=True)
        serializer = ProductListSerializer(products, many=True)
        return Response(serializer.data)
    
    def post(self, request):
        """Create a new product."""
        serializer = ProductCreateSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save(created_by=request.user)
        return Response(serializer.data, status=status.HTTP_201_CREATED)


class ProductDetailView(APIView):
    permission_classes = [IsAuthenticated]
    
    def get_object(self, pk):
        return get_object_or_404(Product, pk=pk)
    
    def get(self, request, pk):
        """Retrieve a product."""
        product = self.get_object(pk)
        serializer = ProductDetailSerializer(product)
        return Response(serializer.data)
    
    def put(self, request, pk):
        """Full update."""
        product = self.get_object(pk)
        serializer = ProductCreateSerializer(product, data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)
    
    def patch(self, request, pk):
        """Partial update."""
        product = self.get_object(pk)
        serializer = ProductCreateSerializer(product, data=request.data, partial=True)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)
    
    def delete(self, request, pk):
        """Delete (soft)."""
        product = self.get_object(pk)
        product.is_active = False
        product.save()
        return Response(status=status.HTTP_204_NO_CONTENT)


# urls.py
urlpatterns = [
    path('products/', ProductListCreateView.as_view()),
    path('products/<int:pk>/', ProductDetailView.as_view()),
]
```

---

## Approach 3: Generic Views (Built-in CRUD) {#approach-3}

> DRF provides pre-built generic views — less code, convention-based.

```python
from rest_framework import generics
from rest_framework.permissions import IsAuthenticated, IsAdminUser

# --- Option A: Separate views for list/create and detail ---
class ProductListCreateView(generics.ListCreateAPIView):
    queryset = Product.objects.filter(is_active=True)
    permission_classes = [IsAuthenticated]
    
    def get_serializer_class(self):
        if self.request.method == 'POST':
            return ProductCreateSerializer
        return ProductListSerializer
    
    def perform_create(self, serializer):
        serializer.save(created_by=self.request.user)


class ProductDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Product.objects.filter(is_active=True)
    permission_classes = [IsAuthenticated]
    
    def get_serializer_class(self):
        if self.request.method in ['PUT', 'PATCH']:
            return ProductCreateSerializer
        return ProductDetailSerializer
    
    def perform_destroy(self, instance):
        instance.is_active = False
        instance.save()


# --- Option B: Individual views for maximum control ---
class ProductListView(generics.ListAPIView):
    """GET /products/ — List only."""
    queryset = Product.objects.filter(is_active=True)
    serializer_class = ProductListSerializer

class ProductCreateView(generics.CreateAPIView):
    """POST /products/ — Create only."""
    serializer_class = ProductCreateSerializer
    permission_classes = [IsAuthenticated]
    
    def perform_create(self, serializer):
        serializer.save(created_by=self.request.user)

class ProductRetrieveView(generics.RetrieveAPIView):
    """GET /products/<pk>/ — Retrieve only."""
    queryset = Product.objects.all()
    serializer_class = ProductDetailSerializer

class ProductUpdateView(generics.UpdateAPIView):
    """PUT/PATCH /products/<pk>/ — Update only."""
    queryset = Product.objects.all()
    serializer_class = ProductCreateSerializer
    permission_classes = [IsAuthenticated]

class ProductDestroyView(generics.DestroyAPIView):
    """DELETE /products/<pk>/ — Delete only."""
    queryset = Product.objects.all()
    permission_classes = [IsAdminUser]


# urls.py
urlpatterns = [
    # Option A
    path('products/', ProductListCreateView.as_view()),
    path('products/<int:pk>/', ProductDetailView.as_view()),
    
    # Option B (individual)
    # path('products/', ProductListView.as_view()),
    # path('products/create/', ProductCreateView.as_view()),
    # path('products/<int:pk>/', ProductRetrieveView.as_view()),
    # path('products/<int:pk>/update/', ProductUpdateView.as_view()),
    # path('products/<int:pk>/delete/', ProductDestroyView.as_view()),
]
```

---

## Approach 4: ModelViewSet (Most Common in Production) {#approach-4}

> Single class handles ALL CRUD operations + automatic URL routing.

```python
from rest_framework import viewsets, status
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated

class ProductViewSet(viewsets.ModelViewSet):
    """
    Automatically provides:
    - list:     GET    /products/
    - create:   POST   /products/
    - retrieve: GET    /products/<pk>/
    - update:   PUT    /products/<pk>/
    - partial:  PATCH  /products/<pk>/
    - destroy:  DELETE /products/<pk>/
    """
    queryset = Product.objects.filter(is_active=True)
    permission_classes = [IsAuthenticated]
    
    def get_serializer_class(self):
        if self.action == 'list':
            return ProductListSerializer
        elif self.action in ['create', 'update', 'partial_update']:
            return ProductCreateSerializer
        return ProductDetailSerializer
    
    def perform_create(self, serializer):
        serializer.save(created_by=self.request.user)
    
    def perform_destroy(self, instance):
        # Soft delete instead of hard delete
        instance.is_active = False
        instance.save()
    
    def get_queryset(self):
        """Dynamic queryset based on user/filters."""
        qs = super().get_queryset()
        
        # Non-admin users only see their own products
        if not self.request.user.is_staff:
            qs = qs.filter(created_by=self.request.user)
        
        return qs


# urls.py (automatic URL generation)
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'products', ProductViewSet, basename='product')

urlpatterns = router.urls
# This generates:
# GET    /products/           → list
# POST   /products/           → create
# GET    /products/{pk}/      → retrieve
# PUT    /products/{pk}/      → update
# PATCH  /products/{pk}/      → partial_update
# DELETE /products/{pk}/      → destroy
```

---

## Approach 5: ViewSet with Custom Actions {#approach-5}

> Extend ModelViewSet with custom endpoints.

```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from django.db.models import Avg, Count, Sum

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.filter(is_active=True)
    serializer_class = ProductDetailSerializer
    
    # Custom action: GET /products/stats/
    @action(detail=False, methods=['get'])
    def stats(self, request):
        """Aggregate statistics."""
        stats = self.get_queryset().aggregate(
            total=Count('id'),
            avg_price=Avg('price'),
            total_stock=Sum('stock')
        )
        by_category = self.get_queryset().values('category').annotate(
            count=Count('id'),
            avg_price=Avg('price')
        )
        return Response({'overview': stats, 'by_category': list(by_category)})
    
    # Custom action: POST /products/{pk}/add_review/
    @action(detail=True, methods=['post'])
    def add_review(self, request, pk=None):
        """Add a review to a product."""
        product = self.get_object()
        serializer = ReviewSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save(product=product, user=request.user)
        return Response(serializer.data, status=status.HTTP_201_CREATED)
    
    # Custom action: GET /products/{pk}/reviews/
    @action(detail=True, methods=['get'])
    def reviews(self, request, pk=None):
        """List reviews for a product."""
        product = self.get_object()
        reviews = product.reviews.all()
        serializer = ReviewSerializer(reviews, many=True)
        return Response(serializer.data)
    
    # Custom action: POST /products/bulk_create/
    @action(detail=False, methods=['post'])
    def bulk_create(self, request):
        """Create multiple products at once."""
        if not isinstance(request.data, list):
            return Response({'error': 'Expected a list'}, status=400)
        if len(request.data) > 50:
            return Response({'error': 'Max 50 items'}, status=400)
        
        created = []
        errors = []
        for i, item_data in enumerate(request.data):
            serializer = ProductCreateSerializer(data=item_data)
            if serializer.is_valid():
                serializer.save(created_by=request.user)
                created.append(serializer.data)
            else:
                errors.append({'index': i, 'errors': serializer.errors})
        
        return Response({
            'created': len(created),
            'failed': len(errors),
            'errors': errors
        }, status=201 if created else 400)
    
    # Custom action: POST /products/{pk}/toggle_active/
    @action(detail=True, methods=['post'])
    def toggle_active(self, request, pk=None):
        """Toggle product active status."""
        product = self.get_object()
        product.is_active = not product.is_active
        product.save()
        return Response({'is_active': product.is_active})
    
    # Custom action: GET /products/export/
    @action(detail=False, methods=['get'])
    def export(self, request):
        """Export products as CSV."""
        import csv
        from django.http import HttpResponse
        
        response = HttpResponse(content_type='text/csv')
        response['Content-Disposition'] = 'attachment; filename="products.csv"'
        
        writer = csv.writer(response)
        writer.writerow(['ID', 'Name', 'Price', 'Category', 'Stock'])
        
        for product in self.get_queryset():
            writer.writerow([product.id, product.name, product.price, 
                           product.category, product.stock])
        
        return response
```

---

## Approach 6: Mixins (Pick & Choose Operations) {#approach-6}

> Control exactly which operations are available.

```python
from rest_framework import viewsets, mixins

# Only List + Retrieve (Read-only)
class ProductReadOnlyViewSet(
    mixins.ListModelMixin,
    mixins.RetrieveModelMixin,
    viewsets.GenericViewSet
):
    queryset = Product.objects.filter(is_active=True)
    serializer_class = ProductListSerializer

# List + Create (no update/delete)
class ProductListCreateViewSet(
    mixins.ListModelMixin,
    mixins.CreateModelMixin,
    viewsets.GenericViewSet
):
    queryset = Product.objects.filter(is_active=True)
    serializer_class = ProductCreateSerializer

# Everything except Delete
class ProductNoDeletViewSet(
    mixins.ListModelMixin,
    mixins.CreateModelMixin,
    mixins.RetrieveModelMixin,
    mixins.UpdateModelMixin,
    viewsets.GenericViewSet
):
    queryset = Product.objects.filter(is_active=True)
    serializer_class = ProductDetailSerializer

# Custom mixin
class BulkCreateMixin:
    """Add bulk_create action to any ViewSet."""
    @action(detail=False, methods=['post'], url_path='bulk')
    def bulk_create(self, request):
        serializer = self.get_serializer(data=request.data, many=True)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data, status=201)

class SoftDeleteMixin:
    """Override destroy to soft-delete."""
    def perform_destroy(self, instance):
        instance.is_active = False
        instance.save()

# Combine mixins
class ProductViewSet(BulkCreateMixin, SoftDeleteMixin, viewsets.ModelViewSet):
    queryset = Product.objects.filter(is_active=True)
    serializer_class = ProductDetailSerializer
```

---

## Approach 7: Nested Resources {#approach-7}

> `/products/{id}/reviews/` — resources within resources.

```python
# Install: pip install drf-nested-routers

from rest_framework_nested import routers

# --- Parent router ---
router = routers.DefaultRouter()
router.register(r'products', ProductViewSet, basename='product')

# --- Nested router (reviews under products) ---
products_router = routers.NestedDefaultRouter(router, r'products', lookup='product')
products_router.register(r'reviews', ReviewViewSet, basename='product-reviews')

# This generates:
# GET/POST       /products/
# GET/PUT/DELETE  /products/{pk}/
# GET/POST       /products/{product_pk}/reviews/
# GET/PUT/DELETE  /products/{product_pk}/reviews/{pk}/

# --- Nested ViewSet ---
class ReviewViewSet(viewsets.ModelViewSet):
    serializer_class = ReviewSerializer
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        return Review.objects.filter(product_id=self.kwargs['product_pk'])
    
    def perform_create(self, serializer):
        product = get_object_or_404(Product, pk=self.kwargs['product_pk'])
        serializer.save(product=product, user=self.request.user)


# --- Manual nested without library ---
class ProductReviewsView(generics.ListCreateAPIView):
    serializer_class = ReviewSerializer
    
    def get_queryset(self):
        return Review.objects.filter(product_id=self.kwargs['product_id'])
    
    def perform_create(self, serializer):
        product = get_object_or_404(Product, pk=self.kwargs['product_id'])
        serializer.save(product=product, user=self.request.user)

# urls.py
urlpatterns = [
    path('products/<int:product_id>/reviews/', ProductReviewsView.as_view()),
]
```

---

## Approach 8: Filtering, Searching, Ordering {#approach-8}

> Production APIs need filtering — DRF has built-in support.

```python
# Install: pip install django-filter

# settings.py
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}

# --- Custom FilterSet ---
import django_filters

class ProductFilter(django_filters.FilterSet):
    name = django_filters.CharFilter(lookup_expr='icontains')
    min_price = django_filters.NumberFilter(field_name='price', lookup_expr='gte')
    max_price = django_filters.NumberFilter(field_name='price', lookup_expr='lte')
    category = django_filters.ChoiceFilter(choices=Product.CATEGORY_CHOICES)
    in_stock = django_filters.BooleanFilter(method='filter_in_stock')
    created_after = django_filters.DateFilter(field_name='created_at', lookup_expr='gte')
    created_before = django_filters.DateFilter(field_name='created_at', lookup_expr='lte')
    
    class Meta:
        model = Product
        fields = ['category', 'is_active']
    
    def filter_in_stock(self, queryset, name, value):
        if value:
            return queryset.filter(stock__gt=0)
        return queryset.filter(stock=0)


# --- Custom Pagination ---
from rest_framework.pagination import PageNumberPagination, CursorPagination

class StandardPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100

class CursorPaginationForFeed(CursorPagination):
    page_size = 20
    ordering = '-created_at'
    cursor_query_param = 'cursor'


# --- ViewSet with all features ---
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.filter(is_active=True).annotate(
        review_count=Count('reviews'),
        avg_rating=Avg('reviews__rating')
    )
    serializer_class = ProductListSerializer
    pagination_class = StandardPagination
    filterset_class = ProductFilter
    search_fields = ['name', 'description', 'category']  # ?search=keyword
    ordering_fields = ['name', 'price', 'created_at', 'stock', 'review_count']
    ordering = ['-created_at']  # Default ordering

# Usage:
# GET /products/?category=electronics&min_price=100&max_price=500
# GET /products/?search=laptop&ordering=-price
# GET /products/?in_stock=true&page=2&page_size=10
# GET /products/?created_after=2025-01-01&ordering=name
```

---

## Approach 9: Permissions & Authentication {#approach-9}

> Control who can do what — essential for production APIs.

```python
from rest_framework import permissions
from rest_framework.authentication import TokenAuthentication
from rest_framework_simplejwt.authentication import JWTAuthentication

# --- Custom Permissions ---
class IsOwnerOrReadOnly(permissions.BasePermission):
    """Only the creator can edit/delete."""
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.created_by == request.user

class IsAdminOrCreateOnly(permissions.BasePermission):
    """Anyone authenticated can create, only admin can list/update/delete."""
    def has_permission(self, request, view):
        if view.action == 'create':
            return request.user.is_authenticated
        return request.user.is_staff

class RoleBasedPermission(permissions.BasePermission):
    """Role-based access."""
    ROLE_PERMISSIONS = {
        'viewer': ['list', 'retrieve'],
        'editor': ['list', 'retrieve', 'create', 'update', 'partial_update'],
        'admin': ['list', 'retrieve', 'create', 'update', 'partial_update', 'destroy'],
    }
    
    def has_permission(self, request, view):
        user_role = getattr(request.user, 'role', 'viewer')
        allowed_actions = self.ROLE_PERMISSIONS.get(user_role, [])
        return view.action in allowed_actions


# --- Throttling (Rate Limiting) ---
from rest_framework.throttling import UserRateThrottle, AnonRateThrottle

class BurstRateThrottle(UserRateThrottle):
    rate = '60/min'

class SustainedRateThrottle(UserRateThrottle):
    rate = '1000/day'


# --- ViewSet with permissions ---
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.filter(is_active=True)
    authentication_classes = [JWTAuthentication]
    throttle_classes = [BurstRateThrottle, SustainedRateThrottle]
    
    def get_permissions(self):
        """Different permissions per action."""
        if self.action in ['list', 'retrieve']:
            return [permissions.AllowAny()]
        elif self.action == 'create':
            return [permissions.IsAuthenticated()]
        elif self.action in ['update', 'partial_update']:
            return [permissions.IsAuthenticated(), IsOwnerOrReadOnly()]
        elif self.action == 'destroy':
            return [permissions.IsAdminUser()]
        return [permissions.IsAuthenticated()]
    
    def get_serializer_class(self):
        if self.action in ['create', 'update', 'partial_update']:
            return ProductCreateSerializer
        if self.action == 'list':
            return ProductListSerializer
        return ProductDetailSerializer


# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
    }
}
```

---

## Approach 10: Full Production Setup {#approach-10}

> Complete production-ready Django REST API with all bells and whistles.

```python
# ========== settings.py (relevant parts) ==========
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
    'DEFAULT_PAGINATION_CLASS': 'core.pagination.StandardPagination',
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
    },
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
    ],
    'EXCEPTION_HANDLER': 'core.exceptions.custom_exception_handler',
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
}


# ========== core/exceptions.py ==========
from rest_framework.views import exception_handler
from rest_framework.response import Response
import logging

logger = logging.getLogger('api')

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    if response is not None:
        custom_response = {
            'error': {
                'code': response.status_code,
                'message': str(exc.detail) if hasattr(exc, 'detail') else str(exc),
                'type': exc.__class__.__name__
            }
        }
        response.data = custom_response
    else:
        logger.error(f"Unhandled exception: {exc}", exc_info=True)
        response = Response({
            'error': {
                'code': 500,
                'message': 'Internal server error',
                'type': 'InternalError'
            }
        }, status=500)
    
    return response


# ========== core/pagination.py ==========
from rest_framework.pagination import PageNumberPagination
from rest_framework.response import Response

class StandardPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100
    
    def get_paginated_response(self, data):
        return Response({
            'results': data,
            'pagination': {
                'count': self.page.paginator.count,
                'page': self.page.number,
                'page_size': self.get_page_size(self.request),
                'total_pages': self.page.paginator.num_pages,
                'has_next': self.page.has_next(),
                'has_prev': self.page.has_previous(),
            }
        })


# ========== core/middleware.py ==========
import time
import uuid
import logging

logger = logging.getLogger('api.request')

class RequestIDMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        request.id = request.headers.get('X-Request-ID', str(uuid.uuid4())[:8])
        start = time.time()
        
        response = self.get_response(request)
        
        duration = time.time() - start
        logger.info(f"[{request.id}] {request.method} {request.path} "
                   f"→ {response.status_code} ({duration:.3f}s)")
        
        response['X-Request-ID'] = request.id
        response['X-Response-Time'] = f"{duration:.3f}s"
        return response


# ========== Full ViewSet with everything ==========
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from django.db.models import Count, Avg, Sum, Q
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page

class ProductViewSet(viewsets.ModelViewSet):
    filterset_class = ProductFilter
    search_fields = ['name', 'description']
    ordering_fields = ['name', 'price', 'created_at', 'stock']
    ordering = ['-created_at']
    
    def get_queryset(self):
        qs = Product.objects.filter(is_active=True).annotate(
            review_count=Count('reviews'),
            avg_rating=Avg('reviews__rating')
        )
        # Optimize: select related foreign keys
        qs = qs.select_related('created_by')
        # Optimize: prefetch many-to-many
        if self.action == 'retrieve':
            qs = qs.prefetch_related('reviews', 'reviews__user')
        return qs
    
    def get_serializer_class(self):
        actions_map = {
            'list': ProductListSerializer,
            'create': ProductCreateSerializer,
            'update': ProductCreateSerializer,
            'partial_update': ProductCreateSerializer,
        }
        return actions_map.get(self.action, ProductDetailSerializer)
    
    def get_permissions(self):
        if self.action in ['list', 'retrieve', 'stats']:
            return [permissions.AllowAny()]
        if self.action == 'destroy':
            return [permissions.IsAdminUser()]
        return [permissions.IsAuthenticated()]
    
    def perform_create(self, serializer):
        serializer.save(created_by=self.request.user)
    
    def perform_destroy(self, instance):
        instance.is_active = False
        instance.save()
    
    # Cached stats endpoint
    @method_decorator(cache_page(60 * 5))  # Cache 5 minutes
    @action(detail=False, methods=['get'])
    def stats(self, request):
        qs = self.get_queryset()
        return Response({
            'total': qs.count(),
            'by_category': list(
                qs.values('category').annotate(
                    count=Count('id'),
                    avg_price=Avg('price'),
                    total_stock=Sum('stock')
                )
            ),
            'top_rated': list(
                qs.filter(avg_rating__isnull=False)
                .order_by('-avg_rating')[:5]
                .values('id', 'name', 'avg_rating')
            )
        })
    
    @action(detail=False, methods=['post'])
    def bulk_update_prices(self, request):
        """Bulk price update by category."""
        category = request.data.get('category')
        multiplier = request.data.get('multiplier', 1.0)
        
        if not category:
            return Response({'error': 'category required'}, status=400)
        
        updated = Product.objects.filter(
            category=category, is_active=True
        ).update(price=models.F('price') * multiplier)
        
        return Response({'updated': updated})


# ========== urls.py (Complete) ==========
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)

router = DefaultRouter()
router.register(r'products', ProductViewSet, basename='product')

urlpatterns = [
    # API v1
    path('api/v1/', include(router.urls)),
    
    # Auth
    path('api/v1/auth/login/', TokenObtainPairView.as_view(), name='token_obtain'),
    path('api/v1/auth/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/v1/auth/verify/', TokenVerifyView.as_view(), name='token_verify'),
    
    # Health
    path('health/', lambda r: JsonResponse({'status': 'ok'})),
]
```

---

## Quick Comparison Table

| Approach | Lines of Code | Flexibility | When to Use |
|----------|---------------|-------------|-------------|
| FBV (`@api_view`) | Many | Maximum | Simple APIs, learning |
| APIView class | Medium | High | Custom logic per method |
| Generic views | Few | Medium | Standard CRUD, less custom logic |
| ModelViewSet | Fewest | Medium | Standard CRUD with router |
| ViewSet + Actions | Medium | High | CRUD + custom endpoints |
| Mixins | Few | High | Need only some operations |
| Nested routers | Medium | High | Parent-child resources |

## Interview Answer: "Which approach do you use?"

> "For most projects, I use **ModelViewSet** with DRF routers because it gives you full CRUD with minimal code. I add **custom actions** for business-specific endpoints, **django-filter** for complex filtering, and **custom permissions** for role-based access. For special cases — like an endpoint that needs to aggregate data from multiple models — I drop down to **APIView** for full control."

---

## Django vs FastAPI — Same API, Side by Side

| Feature | Django REST Framework | FastAPI |
|---------|----------------------|---------|
| Route definition | `router.register()` or `path()` | `@app.get()` decorators |
| Validation | Serializers | Pydantic models |
| DB access | Django ORM (sync) | SQLAlchemy (sync/async) |
| Auth | Built-in + SimpleJWT | DIY with python-jose |
| Pagination | Built-in classes | Manual or fastapi-pagination |
| Filtering | django-filter | Query parameters |
| Admin panel | Built-in | None |
| Async | Django 4.1+ (limited) | Native async |
| API docs | Browsable API | Swagger + ReDoc auto-generated |
| Testing | `APITestCase` | `TestClient` |
