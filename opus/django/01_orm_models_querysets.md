# Django ORM, Models & QuerySets — Senior Level

## 1. Model Design Patterns

### Abstract Base Models (DRY)
```python
from django.db import models
import uuid

class TimeStampedModel(models.Model):
    """Every model inherits created/updated timestamps."""
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True  # No table created for this model


class SoftDeleteModel(models.Model):
    """Soft delete instead of hard delete."""
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        abstract = True

    def soft_delete(self):
        from django.utils import timezone
        self.is_deleted = True
        self.deleted_at = timezone.now()
        self.save(update_fields=['is_deleted', 'deleted_at'])


class Order(TimeStampedModel, SoftDeleteModel):
    """Inherits id, created_at, updated_at, is_deleted, deleted_at."""
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20, choices=[
        ('pending', 'Pending'),
        ('paid', 'Paid'),
        ('shipped', 'Shipped'),
        ('delivered', 'Delivered'),
    ])
```

### Multi-Table Inheritance vs Proxy Models
```python
# Multi-table inheritance — creates separate table with OneToOneField
# USE WHEN: child has additional fields
class Vehicle(models.Model):
    make = models.CharField(max_length=100)
    year = models.IntegerField()

class Car(Vehicle):  # creates car table with vehicle_ptr_id FK
    num_doors = models.IntegerField()

# Proxy model — same table, different behavior
# USE WHEN: only changing methods/managers, no new fields
class PremiumUser(User):
    class Meta:
        proxy = True

    def get_discount(self):
        return 0.20

    objects = PremiumUserManager()
```

### Model Field Best Practices
```python
class Product(TimeStampedModel):
    # Use choices as class variable for reusability
    class Status(models.TextChoices):
        DRAFT = 'draft', 'Draft'
        PUBLISHED = 'published', 'Published'
        ARCHIVED = 'archived', 'Archived'

    name = models.CharField(max_length=255, db_index=True)  # index frequently filtered fields
    slug = models.SlugField(unique=True)  # auto-indexed (unique=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)  # NEVER use FloatField for money
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.DRAFT)
    metadata = models.JSONField(default=dict, blank=True)  # PostgreSQL JSON field
    
    # Correct FK: always set related_name and on_delete explicitly
    category = models.ForeignKey(
        'Category',
        on_delete=models.PROTECT,  # prevent category deletion if products exist
        related_name='products'    # category.products.all()
    )
    tags = models.ManyToManyField('Tag', blank=True, related_name='products')

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['status', 'created_at']),  # composite index
            models.Index(fields=['price'], name='price_idx'),
        ]
        constraints = [
            models.CheckConstraint(
                check=models.Q(price__gte=0),
                name='price_non_negative'
            ),
            models.UniqueConstraint(
                fields=['name', 'category'],
                name='unique_product_per_category'
            ),
        ]
```

---

## 2. Custom Managers & QuerySets

### Custom Manager (Filters at the Manager Level)
```python
class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(
            status='published', is_deleted=False
        )


class ActiveOrderManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(is_deleted=False)

    def pending(self):
        return self.get_queryset().filter(status='pending')

    def total_revenue(self):
        from django.db.models import Sum
        return self.get_queryset().filter(
            status='paid'
        ).aggregate(total=Sum('total'))['total'] or 0


class Order(TimeStampedModel):
    # ...fields...
    
    objects = ActiveOrderManager()      # Order.objects.all() excludes deleted
    all_objects = models.Manager()      # Order.all_objects.all() includes deleted
    
    # Usage:
    # Order.objects.pending()
    # Order.objects.total_revenue()
```

### Chainable Custom QuerySet (Best Practice)
```python
class ProductQuerySet(models.QuerySet):
    def published(self):
        return self.filter(status='published')

    def in_stock(self):
        return self.filter(stock__gt=0)

    def expensive(self, min_price=100):
        return self.filter(price__gte=min_price)

    def with_category(self):
        return self.select_related('category')

    def with_tags(self):
        return self.prefetch_related('tags')

    def annotated_with_review_count(self):
        from django.db.models import Count
        return self.annotate(review_count=Count('reviews'))


class ProductManager(models.Manager):
    def get_queryset(self):
        return ProductQuerySet(self.model, using=self._db)

    # Expose QuerySet methods on Manager
    def published(self):
        return self.get_queryset().published()


class Product(models.Model):
    objects = ProductManager()

    # Now you can chain:
    # Product.objects.published().in_stock().expensive(50).with_category()
```

---

## 3. QuerySet Optimization (N+1 Problem)

### The N+1 Problem
```python
# ❌ BAD: N+1 queries (1 query for orders + N queries for user)
orders = Order.objects.all()
for order in orders:
    print(order.user.username)  # hits DB for EACH order

# ✓ GOOD: select_related for ForeignKey/OneToOne (SQL JOIN)
orders = Order.objects.select_related('user').all()
for order in orders:
    print(order.user.username)  # no extra queries — joined in 1 SQL

# ✓ GOOD: prefetch_related for ManyToMany/reverse FK (2 queries)
products = Product.objects.prefetch_related('tags').all()
for product in products:
    print(product.tags.all())  # uses prefetched data, no extra queries
```

### select_related vs prefetch_related
```python
# select_related: SQL JOIN (for ForeignKey, OneToOneField)
# Single query, but wider result set
Order.objects.select_related('user', 'user__profile')

# prefetch_related: separate query + Python join (for M2M, reverse FK)
# 2+ queries, but handles large datasets better
Category.objects.prefetch_related('products', 'products__tags')

# Prefetch with custom queryset (filter prefetched objects)
from django.db.models import Prefetch

categories = Category.objects.prefetch_related(
    Prefetch(
        'products',
        queryset=Product.objects.filter(status='published').order_by('-price'),
        to_attr='published_products'  # access as category.published_products
    )
)
```

### Only / Defer (Load Specific Columns)
```python
# ❌ Loads ALL columns including large text fields
products = Product.objects.all()

# ✓ Only load needed columns (SELECT name, price FROM ...)
products = Product.objects.only('name', 'price')

# ✓ Load everything EXCEPT large fields
products = Product.objects.defer('description', 'metadata')
```

---

## 4. Advanced QuerySet Operations

### Aggregation & Annotation
```python
from django.db.models import Count, Sum, Avg, F, Q, Value, Case, When
from django.db.models.functions import Coalesce, TruncMonth

# Aggregate: returns a dict (single row)
stats = Order.objects.aggregate(
    total_revenue=Sum('total'),
    avg_order=Avg('total'),
    order_count=Count('id')
)
# {'total_revenue': Decimal('50000'), 'avg_order': Decimal('125'), 'order_count': 400}

# Annotate: adds computed column to each row
users = User.objects.annotate(
    order_count=Count('orders'),
    total_spent=Coalesce(Sum('orders__total'), Value(0))
).filter(order_count__gt=5).order_by('-total_spent')

# Conditional annotation (like SQL CASE WHEN)
products = Product.objects.annotate(
    price_tier=Case(
        When(price__lt=50, then=Value('budget')),
        When(price__lt=200, then=Value('mid')),
        default=Value('premium')
    )
)

# Monthly revenue report
monthly = Order.objects.filter(status='paid').annotate(
    month=TruncMonth('created_at')
).values('month').annotate(
    revenue=Sum('total'),
    orders=Count('id')
).order_by('month')
```

### F Expressions (Reference Other Fields)
```python
# Compare field to field (no Python evaluation — runs in SQL)
products_below_cost = Product.objects.filter(price__lt=F('cost'))

# Update without race condition
Product.objects.filter(id=1).update(stock=F('stock') - 1)

# This is ATOMIC — no race condition even with concurrent requests
# Equivalent SQL: UPDATE product SET stock = stock - 1 WHERE id = 1
```

### Q Objects (Complex Lookups)
```python
from django.db.models import Q

# OR condition
expensive_or_new = Product.objects.filter(
    Q(price__gt=1000) | Q(created_at__gte='2024-01-01')
)

# NOT condition
not_draft = Product.objects.filter(~Q(status='draft'))

# Complex: (expensive AND published) OR (cheap AND in_stock)
results = Product.objects.filter(
    (Q(price__gt=1000) & Q(status='published')) |
    (Q(price__lt=50) & Q(stock__gt=0))
)

# Dynamic Q building (useful for search/filter APIs)
def build_filter(params):
    q = Q()
    if params.get('min_price'):
        q &= Q(price__gte=params['min_price'])
    if params.get('category'):
        q &= Q(category__slug=params['category'])
    if params.get('search'):
        q &= Q(name__icontains=params['search']) | Q(description__icontains=params['search'])
    return Product.objects.filter(q)
```

### Subqueries
```python
from django.db.models import Subquery, OuterRef

# Get latest order for each user
latest_order = Order.objects.filter(
    user=OuterRef('pk')
).order_by('-created_at')

users_with_latest_order = User.objects.annotate(
    latest_order_total=Subquery(latest_order.values('total')[:1]),
    latest_order_date=Subquery(latest_order.values('created_at')[:1])
)

# Exists subquery (efficient "has related objects" check)
from django.db.models import Exists

has_orders = Order.objects.filter(user=OuterRef('pk'))
active_buyers = User.objects.annotate(
    is_buyer=Exists(has_orders)
).filter(is_buyer=True)
```

---

## 5. Raw SQL & Database Functions

### When to Use Raw SQL
```python
# Use raw SQL ONLY when ORM can't express the query efficiently
# (complex window functions, CTEs, database-specific features)

# Method 1: raw() — returns model instances
users = User.objects.raw('''
    SELECT u.*, COUNT(o.id) as order_count
    FROM auth_user u
    LEFT JOIN orders_order o ON o.user_id = u.id
    GROUP BY u.id
    HAVING COUNT(o.id) > 5
    ORDER BY order_count DESC
''')

# Method 2: cursor (pure SQL, no model mapping)
from django.db import connection

def get_revenue_by_month():
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT DATE_TRUNC('month', created_at) as month,
                   SUM(total) as revenue
            FROM orders_order
            WHERE status = %s
            GROUP BY 1
            ORDER BY 1
        """, ['paid'])
        columns = [col[0] for col in cursor.description]
        return [dict(zip(columns, row)) for row in cursor.fetchall()]

# IMPORTANT: Always use parameterized queries (%s) — NEVER string format
# ❌ cursor.execute(f"SELECT * FROM users WHERE name = '{name}'")  # SQL INJECTION!
# ✓ cursor.execute("SELECT * FROM users WHERE name = %s", [name])
```

### Window Functions (PostgreSQL)
```python
from django.db.models import Window, F
from django.db.models.functions import Rank, DenseRank, RowNumber

# Rank products by price within each category
products = Product.objects.annotate(
    price_rank=Window(
        expression=Rank(),
        partition_by=F('category'),
        order_by=F('price').desc()
    )
)

# Running total
orders = Order.objects.annotate(
    running_total=Window(
        expression=Sum('total'),
        order_by=F('created_at').asc(),
        frame=RowRange(start=None, end=0)  # unbounded preceding to current
    )
)
```

---

## 6. Transactions & Concurrency

### Atomic Transactions
```python
from django.db import transaction

# Method 1: Decorator
@transaction.atomic
def transfer_funds(from_account, to_account, amount):
    from_account.balance = F('balance') - amount
    from_account.save(update_fields=['balance'])
    
    to_account.balance = F('balance') + amount
    to_account.save(update_fields=['balance'])
    
    Transaction.objects.create(
        from_account=from_account,
        to_account=to_account,
        amount=amount
    )
    # If ANY line raises an exception, ALL changes are rolled back

# Method 2: Context manager (more control)
def create_order(user, items):
    try:
        with transaction.atomic():
            order = Order.objects.create(user=user, total=0)
            total = 0
            for item in items:
                OrderItem.objects.create(order=order, **item)
                total += item['price'] * item['quantity']
                # Decrease stock atomically
                updated = Product.objects.filter(
                    id=item['product_id'], stock__gte=item['quantity']
                ).update(stock=F('stock') - item['quantity'])
                if not updated:
                    raise ValueError(f"Insufficient stock for {item['product_id']}")
            order.total = total
            order.save(update_fields=['total'])
        return order
    except ValueError as e:
        return None  # transaction already rolled back
```

### select_for_update (Row-Level Locking)
```python
from django.db import transaction

@transaction.atomic
def purchase_item(user, product_id, quantity):
    # Lock the row — other transactions wait until this one finishes
    product = Product.objects.select_for_update().get(id=product_id)
    
    if product.stock < quantity:
        raise ValueError("Out of stock")
    
    product.stock -= quantity
    product.save(update_fields=['stock'])
    
    Order.objects.create(user=user, product=product, quantity=quantity)

# select_for_update options:
# .select_for_update()                → exclusive lock (wait for release)
# .select_for_update(nowait=True)     → raise DatabaseError immediately if locked
# .select_for_update(skip_locked=True) → skip locked rows (useful for task queues)
```

### Optimistic Locking (Version-Based)
```python
class Product(models.Model):
    stock = models.IntegerField()
    version = models.IntegerField(default=0)

def purchase_optimistic(product_id, quantity):
    product = Product.objects.get(id=product_id)
    
    # Try to update only if version hasn't changed
    updated = Product.objects.filter(
        id=product_id,
        version=product.version  # optimistic check
    ).update(
        stock=F('stock') - quantity,
        version=F('version') + 1
    )
    
    if not updated:
        raise ConflictError("Product was modified by another request. Retry.")
    
    # No lock held — higher throughput than select_for_update
    # But caller must handle retry logic
```

---

## 7. Interview Questions & Answers

### Q: What's the difference between select_related and prefetch_related?
```
select_related: SQL JOIN in a single query. For ForeignKey/OneToOne.
  SELECT * FROM order JOIN user ON order.user_id = user.id

prefetch_related: Separate query + Python-side join. For M2M/reverse FK.
  SELECT * FROM order;
  SELECT * FROM user WHERE id IN (1, 2, 3, ...);

Use select_related when: following FK "forward" (order → user)
Use prefetch_related when: following FK "backward" (user → orders) or M2M
```

### Q: How do you prevent race conditions in Django?
```
1. F() expressions — atomic field updates (UPDATE stock = stock - 1)
2. select_for_update() — row-level locking (pessimistic)
3. Version field + conditional update (optimistic)
4. transaction.atomic() — all-or-nothing writes
5. unique_together / UniqueConstraint — DB-level uniqueness
```

### Q: How do you handle N+1 queries in production?
```
1. Use django-debug-toolbar in development (shows query count)
2. select_related() for FK/OneToOne joins
3. prefetch_related() for M2M and reverse FK
4. only()/defer() to limit columns
5. In DRF: override get_queryset() in ViewSet with proper select/prefetch
6. Monitor with django-silk or New Relic APM in production
```

### Q: What's the difference between .update() and .save()?
```
.save():    Loads object into Python, modifies, writes back ALL fields
            Triggers pre_save/post_save signals
            Runs model validation (full_clean)
            Slower but safer

.update():  Direct SQL UPDATE, never loads object into Python
            Does NOT trigger signals
            Does NOT run validation
            Faster, atomic, but skips business logic

Use .update() for: bulk operations, atomic counter updates
Use .save() for: single objects that need validation/signals
```
