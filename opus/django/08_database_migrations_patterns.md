# Django Database Patterns & Migrations — Senior Level

## 1. Zero-Downtime Migrations

### The Problem
```
Standard deployment:
  1. Deploy new code (expects new column)
  2. Run migration (adds new column)
  Gap: between step 1 and 2, code crashes because column doesn't exist

Correct order:
  1. Run migration FIRST (add column as nullable)
  2. Deploy new code (uses new column)
  3. Later: backfill data, add NOT NULL constraint
```

### Safe Migration Patterns

#### Adding a Column (3-Step Process)
```python
# Step 1: Add nullable column (backwards-compatible)
# Deploy this migration BEFORE new code
class Migration(migrations.Migration):
    operations = [
        migrations.AddField(
            model_name='order',
            name='tracking_number',
            field=models.CharField(max_length=100, null=True, blank=True),
        ),
    ]

# Step 2: Deploy code that WRITES to new column but doesn't REQUIRE it
# Old code still works (column is nullable)

# Step 3: Backfill data + add NOT NULL (after all old code is gone)
class Migration(migrations.Migration):
    operations = [
        migrations.RunPython(backfill_tracking_numbers, migrations.RunPython.noop),
        migrations.AlterField(
            model_name='order',
            name='tracking_number',
            field=models.CharField(max_length=100, default=''),
        ),
    ]

def backfill_tracking_numbers(apps, schema_editor):
    Order = apps.get_model('orders', 'Order')
    Order.objects.filter(tracking_number__isnull=True).update(tracking_number='')
```

#### Renaming a Column (Never Rename Directly!)
```python
# ❌ DANGEROUS: breaks old code immediately
# migrations.RenameField(model_name='order', old_name='total', new_name='total_amount')

# ✓ SAFE: 4 deploys

# Deploy 1: Add new column
migrations.AddField('order', 'total_amount', models.DecimalField(null=True, ...))

# Deploy 2: Write to BOTH columns (dual-write)
# Code: order.total = X; order.total_amount = X; order.save()

# Deploy 3: Backfill + switch reads to new column
# Migration: UPDATE orders SET total_amount = total WHERE total_amount IS NULL
# Code: reads from total_amount, writes to both

# Deploy 4: Remove old column (all code uses new name)
migrations.RemoveField('order', 'total')
```

#### Removing a Column
```python
# ❌ DANGEROUS: removing column while old code still references it
# migrations.RemoveField(model_name='order', name='legacy_field')

# ✓ SAFE: 2 deploys

# Deploy 1: Remove all code references to the field
# (but don't remove from model yet — keep field in DB)

# Deploy 2: Remove the field from model + migration
migrations.RemoveField(model_name='order', name='legacy_field')
```

---

## 2. Migration Best Practices

### Data Migrations
```python
from django.db import migrations

def populate_slugs(apps, schema_editor):
    """Generate slugs for existing products."""
    Product = apps.get_model('products', 'Product')
    from django.utils.text import slugify
    
    for product in Product.objects.filter(slug='').iterator(chunk_size=1000):
        product.slug = slugify(product.name)
        product.save(update_fields=['slug'])

def reverse_slugs(apps, schema_editor):
    """Reverse migration: clear slugs."""
    Product = apps.get_model('products', 'Product')
    Product.objects.update(slug='')

class Migration(migrations.Migration):
    dependencies = [('products', '0005_add_slug_field')]
    
    operations = [
        migrations.RunPython(populate_slugs, reverse_slugs),
    ]
```

### Large Table Migrations (Avoid Locking)
```python
# Problem: ALTER TABLE on 100M row table → locks table for minutes
# Solution: Use django-pg-zero-downtime-migrations or SeparateDatabaseAndState

from django.db import migrations

class Migration(migrations.Migration):
    # Tell Django: "I'll handle the DB change manually"
    operations = [
        migrations.SeparateDatabaseAndState(
            state_operations=[
                migrations.AddField(
                    model_name='order',
                    name='priority',
                    field=models.IntegerField(default=0),
                ),
            ],
            database_operations=[
                # Use concurrent index creation (doesn't lock table)
                migrations.RunSQL(
                    sql="ALTER TABLE orders_order ADD COLUMN priority INTEGER DEFAULT 0;",
                    reverse_sql="ALTER TABLE orders_order DROP COLUMN priority;",
                ),
            ],
        ),
    ]

# For indexes on large tables:
class Migration(migrations.Migration):
    atomic = False  # IMPORTANT: allow non-atomic operations
    
    operations = [
        migrations.RunSQL(
            sql="CREATE INDEX CONCURRENTLY idx_orders_priority ON orders_order (priority);",
            reverse_sql="DROP INDEX idx_orders_priority;",
        ),
    ]
```

### Migration Commands Cheatsheet
```bash
# Create migration
python manage.py makemigrations

# Show migration SQL without running
python manage.py sqlmigrate orders 0005

# Run migrations
python manage.py migrate

# Show migration status
python manage.py showmigrations

# Fake a migration (mark as done without running)
python manage.py migrate orders 0005 --fake

# Reverse to specific migration
python manage.py migrate orders 0003

# Squash migrations (merge many into one)
python manage.py squashmigrations orders 0001 0010
```

---

## 3. Multi-Database Setup

### Read Replica Configuration
```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'primary.db.amazonaws.com',
        'PORT': '5432',
    },
    'replica': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'replica.db.amazonaws.com',
        'PORT': '5432',
    },
}

# Database router
class ReadReplicaRouter:
    def db_for_read(self, model, **hints):
        return 'replica'
    
    def db_for_write(self, model, **hints):
        return 'default'
    
    def allow_relation(self, obj1, obj2, **hints):
        return True
    
    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == 'default'

DATABASE_ROUTERS = ['myproject.routers.ReadReplicaRouter']

# Override in specific views when you need read-after-write consistency
class OrderCreateView(APIView):
    def post(self, request):
        order = Order.objects.create(...)
        # Read from primary (not replica) to get just-written data
        order = Order.objects.using('default').get(id=order.id)
        return Response(OrderSerializer(order).data)
```

### Separate Database for Analytics
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'app_db',
    },
    'analytics': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'analytics_db',
    },
}

class AnalyticsRouter:
    analytics_models = {'PageView', 'EventLog', 'UserActivity'}
    
    def db_for_read(self, model, **hints):
        if model.__name__ in self.analytics_models:
            return 'analytics'
        return 'default'
    
    def db_for_write(self, model, **hints):
        if model.__name__ in self.analytics_models:
            return 'analytics'
        return 'default'
```

---

## 4. Database Query Patterns

### Efficient Pagination (Keyset/Cursor)
```python
# ❌ SLOW: OFFSET-based (scans skipped rows)
Product.objects.all()[10000:10020]
# SQL: SELECT * FROM product LIMIT 20 OFFSET 10000 (scans 10020 rows!)

# ✓ FAST: Keyset pagination (instant regardless of page)
last_seen_id = request.GET.get('after')
if last_seen_id:
    products = Product.objects.filter(id__gt=last_seen_id).order_by('id')[:20]
else:
    products = Product.objects.order_by('id')[:20]
# SQL: SELECT * FROM product WHERE id > 5000 ORDER BY id LIMIT 20 (uses index)
```

### Efficient Existence Checks
```python
# ❌ Loads object into memory just to check existence
try:
    product = Product.objects.get(slug='iphone')
    exists = True
except Product.DoesNotExist:
    exists = False

# ✓ Database-level check (no object loading)
exists = Product.objects.filter(slug='iphone').exists()

# ✓ Get or None pattern (when you need the object IF it exists)
from django.shortcuts import get_object_or_404
product = get_object_or_404(Product, slug='iphone')
```

### Efficient Aggregation
```python
from django.db.models import Count, Sum, Avg, Q

# Count with conditions (single query, no Python loop)
stats = Product.objects.aggregate(
    total=Count('id'),
    published=Count('id', filter=Q(status='published')),
    draft=Count('id', filter=Q(status='draft')),
    avg_price=Avg('price', filter=Q(status='published')),
)

# Group by with annotation
category_stats = Category.objects.annotate(
    product_count=Count('products'),
    total_revenue=Sum('products__orderitem__price'),
).filter(product_count__gt=0).order_by('-total_revenue')
```

### Batch Processing Large Datasets
```python
# ❌ BAD: Loads ALL objects into memory
for order in Order.objects.all():
    process(order)

# ✓ GOOD: iterator() — loads one chunk at a time
for order in Order.objects.all().iterator(chunk_size=1000):
    process(order)

# ✓ GOOD: Manual chunking for updates
from django.db.models import F

def process_in_batches(queryset, batch_size=1000):
    total = queryset.count()
    for start in range(0, total, batch_size):
        batch = queryset[start:start + batch_size]
        for obj in batch:
            process(obj)
        print(f'Processed {min(start + batch_size, total)}/{total}')
```

---

## 5. PostgreSQL-Specific Features

### Full-Text Search
```python
from django.contrib.postgres.search import (
    SearchVector, SearchQuery, SearchRank, TrigramSimilarity
)

# Basic full-text search
results = Product.objects.annotate(
    search=SearchVector('name', 'description', config='english')
).filter(search=SearchQuery('wireless headphones', config='english'))

# Ranked search (relevance scoring)
vector = SearchVector('name', weight='A') + SearchVector('description', weight='B')
query = SearchQuery('wireless headphones')

results = Product.objects.annotate(
    rank=SearchRank(vector, query)
).filter(rank__gte=0.1).order_by('-rank')

# Trigram similarity (fuzzy matching — handles typos)
results = Product.objects.annotate(
    similarity=TrigramSimilarity('name', 'iphon')  # typo for "iphone"
).filter(similarity__gt=0.3).order_by('-similarity')

# GIN index for search performance
class Product(models.Model):
    class Meta:
        indexes = [
            GinIndex(
                SearchVector('name', 'description', config='english'),
                name='search_vector_idx'
            ),
        ]
```

### JSONField Queries (PostgreSQL)
```python
class Product(models.Model):
    metadata = models.JSONField(default=dict)
    # metadata = {"color": "red", "specs": {"weight": 150, "battery": "5000mAh"}}

# Query nested JSON
Product.objects.filter(metadata__color='red')
Product.objects.filter(metadata__specs__weight__gte=100)

# Contains
Product.objects.filter(metadata__contains={'color': 'red'})

# Has key
Product.objects.filter(metadata__has_key='specs')
Product.objects.filter(metadata__has_keys=['color', 'size'])
```

### Array Fields
```python
from django.contrib.postgres.fields import ArrayField

class Product(models.Model):
    tags = ArrayField(models.CharField(max_length=50), default=list)

# Query arrays
Product.objects.filter(tags__contains=['electronics'])
Product.objects.filter(tags__overlap=['sale', 'new'])
Product.objects.filter(tags__len=3)
```

---

## 6. Connection Management

```python
# settings.py
DATABASES = {
    'default': {
        ...
        'CONN_MAX_AGE': 600,  # reuse connections for 10 min (0 = close after each request)
        'CONN_HEALTH_CHECKS': True,  # Django 4.1+: verify connection before reuse
        'OPTIONS': {
            'connect_timeout': 5,
            'options': '-c statement_timeout=30000',  # kill queries after 30s
        },
    }
}

# For Celery workers (long-lived processes):
# Close stale connections before each task
from django import db

@shared_task
def my_task():
    db.close_old_connections()
    # ... task logic ...
```

---

## 7. Database Monitoring Queries

```sql
-- Find slow queries (PostgreSQL)
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active' AND now() - pg_stat_activity.query_start > interval '5 seconds';

-- Table sizes
SELECT relname, pg_size_pretty(pg_total_relation_size(relid))
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Index usage (find unused indexes)
SELECT schemaname, relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%pkey%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Cache hit ratio (should be > 99%)
SELECT 
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as ratio
FROM pg_statio_user_tables;

-- Lock monitoring
SELECT relation::regclass, mode, granted
FROM pg_locks WHERE NOT granted;
```

---

## 8. Interview Questions

### Q: How do you do zero-downtime migrations?
```
1. Never remove/rename columns in same deploy as code change
2. Add new columns as NULLABLE first
3. Deploy code that uses new column (optional path)
4. Backfill data
5. Add NOT NULL constraint
6. Remove old column (after old code fully replaced)

For large tables:
- CREATE INDEX CONCURRENTLY (doesn't lock)
- Add columns with DEFAULT (Postgres 11+ is instant)
- Backfill in batches (not one UPDATE of 100M rows)
```

### Q: How do you handle database scaling in Django?
```
1. Read replicas: route reads to replica, writes to primary
   - DATABASE_ROUTERS in settings
   - use .using('replica') or router auto-routes

2. Connection pooling: PgBouncer in front of PostgreSQL
   - Reduces "too many connections" errors
   - Django → PgBouncer (6432) → PostgreSQL (5432)

3. Caching: Redis for hot queries (reduce DB load by 80%)

4. Partitioning: split large tables by date/tenant
   - PostgreSQL table partitioning for time-series data
   - Django-postgres-extra for partition support

5. Sharding: split data across multiple databases (last resort)
   - Route by tenant_id or user_id
   - Custom DATABASE_ROUTERS
```

### Q: What's the difference between select_for_update and F() expressions?
```
F() expressions: atomic column updates without locking
  Product.objects.filter(id=1).update(stock=F('stock') - 1)
  → Single UPDATE query, atomic, no race condition for THIS operation
  → But can't prevent stock going negative

select_for_update: row-level lock with validation
  with transaction.atomic():
      product = Product.objects.select_for_update().get(id=1)
      if product.stock >= quantity:  # can validate!
          product.stock -= quantity
          product.save()
  → Lock held until transaction commits
  → Other requests WAIT (serialized access)
  → Use when you need to CHECK before UPDATE

Rule: Use F() for simple increments/decrements
      Use select_for_update for check-then-act patterns
```
