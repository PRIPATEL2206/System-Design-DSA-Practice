# Django Testing Strategies — Senior Level

## 1. Test Structure & Configuration

### pytest Setup (Preferred over unittest)
```python
# pytest.ini or pyproject.toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "myproject.settings.test"
python_files = ["tests.py", "test_*.py"]
addopts = "-v --tb=short --strict-markers -n auto"
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks integration tests",
]

# conftest.py (project root)
import pytest
from rest_framework.test import APIClient

@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def authenticated_client(api_client, user):
    api_client.force_authenticate(user=user)
    return api_client

@pytest.fixture
def user(db):
    from django.contrib.auth import get_user_model
    User = get_user_model()
    return User.objects.create_user(
        username='testuser',
        email='test@example.com',
        password='testpass123'
    )
```

### Test Directory Structure
```
apps/
├── orders/
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── conftest.py          # app-specific fixtures
│   │   ├── test_models.py       # model methods, constraints
│   │   ├── test_serializers.py  # validation, representation
│   │   ├── test_views.py        # API endpoints
│   │   ├── test_services.py     # business logic
│   │   └── test_tasks.py        # Celery tasks
│   ├── models.py
│   ├── views.py
│   └── services.py
```

---

## 2. Factory Pattern (Factory Boy)

```python
# tests/factories.py
import factory
from factory.django import DjangoModelFactory
from apps.orders.models import Order, OrderItem
from apps.products.models import Product, Category

class UserFactory(DjangoModelFactory):
    class Meta:
        model = 'auth.User'
    
    username = factory.Sequence(lambda n: f'user_{n}')
    email = factory.LazyAttribute(lambda obj: f'{obj.username}@example.com')
    password = factory.PostGenerationMethodCall('set_password', 'testpass123')
    is_active = True


class CategoryFactory(DjangoModelFactory):
    class Meta:
        model = Category
    
    name = factory.Sequence(lambda n: f'Category {n}')
    slug = factory.LazyAttribute(lambda obj: obj.name.lower().replace(' ', '-'))


class ProductFactory(DjangoModelFactory):
    class Meta:
        model = Product
    
    name = factory.Sequence(lambda n: f'Product {n}')
    price = factory.Faker('pydecimal', left_digits=3, right_digits=2, positive=True)
    stock = 100
    status = 'published'
    category = factory.SubFactory(CategoryFactory)


class OrderFactory(DjangoModelFactory):
    class Meta:
        model = Order
    
    user = factory.SubFactory(UserFactory)
    status = 'pending'
    total = factory.Faker('pydecimal', left_digits=4, right_digits=2, positive=True)


class OrderItemFactory(DjangoModelFactory):
    class Meta:
        model = OrderItem
    
    order = factory.SubFactory(OrderFactory)
    product = factory.SubFactory(ProductFactory)
    quantity = factory.Faker('random_int', min=1, max=5)
    price = factory.LazyAttribute(lambda obj: obj.product.price)


# Usage in tests:
def test_order_total():
    order = OrderFactory(total=0)
    OrderItemFactory(order=order, price=50, quantity=2)
    OrderItemFactory(order=order, price=30, quantity=1)
    
    order.recalculate_total()
    assert order.total == 130
```

---

## 3. Model Tests

```python
import pytest
from django.db import IntegrityError
from django.core.exceptions import ValidationError
from apps.products.models import Product
from .factories import ProductFactory, CategoryFactory

@pytest.mark.django_db
class TestProductModel:
    def test_create_product(self):
        product = ProductFactory(name='iPhone', price=999.99)
        assert product.name == 'iPhone'
        assert product.price == 999.99
        assert product.status == 'published'
    
    def test_slug_auto_generated(self):
        product = ProductFactory(name='iPhone 15 Pro Max')
        product.save()
        assert product.slug == 'iphone-15-pro-max'
    
    def test_price_cannot_be_negative(self):
        with pytest.raises(IntegrityError):
            ProductFactory(price=-10)
    
    def test_unique_name_per_category(self):
        category = CategoryFactory()
        ProductFactory(name='Widget', category=category)
        with pytest.raises(IntegrityError):
            ProductFactory(name='Widget', category=category)
    
    def test_soft_delete(self):
        product = ProductFactory()
        product.soft_delete()
        
        assert product.is_deleted is True
        assert product.deleted_at is not None
        assert Product.objects.count() == 0  # custom manager filters deleted
        assert Product.all_objects.count() == 1
    
    def test_str_representation(self):
        product = ProductFactory(name='Test Product')
        assert str(product) == 'Test Product'
    
    def test_custom_manager_published(self):
        ProductFactory(status='published')
        ProductFactory(status='draft')
        ProductFactory(status='published')
        
        assert Product.objects.published().count() == 2
```

---

## 4. API/View Tests

```python
import pytest
from rest_framework import status
from .factories import ProductFactory, UserFactory, OrderFactory

@pytest.mark.django_db
class TestProductAPI:
    endpoint = '/api/v1/products/'
    
    def test_list_products_unauthenticated(self, api_client):
        ProductFactory.create_batch(5, status='published')
        ProductFactory.create_batch(3, status='draft')
        
        response = api_client.get(self.endpoint)
        
        assert response.status_code == status.HTTP_200_OK
        assert response.data['count'] == 5  # only published
    
    def test_create_product_authenticated(self, authenticated_client):
        category = CategoryFactory()
        data = {
            'name': 'New Product',
            'price': '99.99',
            'category': category.id,
            'status': 'draft'
        }
        
        response = authenticated_client.post(self.endpoint, data)
        
        assert response.status_code == status.HTTP_201_CREATED
        assert response.data['name'] == 'New Product'
    
    def test_create_product_unauthenticated_fails(self, api_client):
        response = api_client.post(self.endpoint, {'name': 'Test'})
        assert response.status_code == status.HTTP_401_UNAUTHORIZED
    
    def test_update_product_not_owner_fails(self, api_client):
        product = ProductFactory()
        other_user = UserFactory()
        api_client.force_authenticate(other_user)
        
        response = api_client.patch(
            f'{self.endpoint}{product.id}/',
            {'name': 'Hacked'}
        )
        assert response.status_code == status.HTTP_403_FORBIDDEN
    
    def test_filter_by_category(self, api_client):
        cat1 = CategoryFactory(name='Electronics')
        cat2 = CategoryFactory(name='Books')
        ProductFactory.create_batch(3, category=cat1, status='published')
        ProductFactory.create_batch(2, category=cat2, status='published')
        
        response = api_client.get(f'{self.endpoint}?category={cat1.id}')
        
        assert response.data['count'] == 3
    
    def test_search_products(self, api_client):
        ProductFactory(name='iPhone 15', status='published')
        ProductFactory(name='Samsung Galaxy', status='published')
        
        response = api_client.get(f'{self.endpoint}?search=iphone')
        
        assert response.data['count'] == 1
        assert response.data['results'][0]['name'] == 'iPhone 15'
    
    def test_pagination(self, api_client):
        ProductFactory.create_batch(25, status='published')
        
        response = api_client.get(f'{self.endpoint}?page_size=10')
        
        assert response.data['count'] == 25
        assert len(response.data['results']) == 10
        assert response.data['next'] is not None


@pytest.mark.django_db
class TestOrderAPI:
    endpoint = '/api/v1/orders/'
    
    def test_create_order_insufficient_stock(self, authenticated_client):
        product = ProductFactory(stock=2)
        data = {
            'items': [{'product': product.id, 'quantity': 5}]
        }
        
        response = authenticated_client.post(self.endpoint, data, format='json')
        
        assert response.status_code == status.HTTP_400_BAD_REQUEST
        assert 'stock' in str(response.data).lower()
    
    def test_create_order_decreases_stock(self, authenticated_client):
        product = ProductFactory(stock=10)
        data = {
            'items': [{'product': product.id, 'quantity': 3}]
        }
        
        response = authenticated_client.post(self.endpoint, data, format='json')
        
        assert response.status_code == status.HTTP_201_CREATED
        product.refresh_from_db()
        assert product.stock == 7
    
    def test_user_sees_only_own_orders(self, api_client):
        user1 = UserFactory()
        user2 = UserFactory()
        OrderFactory.create_batch(3, user=user1)
        OrderFactory.create_batch(2, user=user2)
        
        api_client.force_authenticate(user1)
        response = api_client.get(self.endpoint)
        
        assert response.data['count'] == 3
```

---

## 5. Service Layer Tests

```python
import pytest
from unittest.mock import patch, MagicMock
from apps.orders.services import OrderService
from .factories import UserFactory, ProductFactory

@pytest.mark.django_db
class TestOrderService:
    def setup_method(self):
        self.service = OrderService()
    
    def test_create_order_success(self):
        user = UserFactory()
        product = ProductFactory(stock=10, price=50)
        
        order = self.service.create_order(
            user=user,
            items=[{'product_id': product.id, 'quantity': 2}]
        )
        
        assert order.total == 100
        assert order.status == 'pending'
        product.refresh_from_db()
        assert product.stock == 8
    
    def test_create_order_atomic_rollback(self):
        """If payment fails, stock should NOT be decreased."""
        user = UserFactory()
        product = ProductFactory(stock=10, price=50)
        
        with patch.object(self.service, 'process_payment', side_effect=Exception('Payment failed')):
            with pytest.raises(Exception):
                self.service.create_order(
                    user=user,
                    items=[{'product_id': product.id, 'quantity': 2}]
                )
        
        product.refresh_from_db()
        assert product.stock == 10  # unchanged — transaction rolled back
    
    def test_concurrent_orders_no_oversell(self):
        """Two users ordering last item — one should fail."""
        product = ProductFactory(stock=1)
        user1 = UserFactory()
        user2 = UserFactory()
        
        # First order succeeds
        order1 = self.service.create_order(
            user=user1,
            items=[{'product_id': product.id, 'quantity': 1}]
        )
        assert order1 is not None
        
        # Second order fails (stock = 0)
        with pytest.raises(InsufficientStockError):
            self.service.create_order(
                user=user2,
                items=[{'product_id': product.id, 'quantity': 1}]
            )
```

---

## 6. Mocking External Services

```python
import pytest
from unittest.mock import patch, MagicMock
import responses  # pip install responses (for mocking HTTP)

@pytest.mark.django_db
class TestPaymentService:
    
    @patch('apps.payments.services.stripe.Charge.create')
    def test_charge_success(self, mock_stripe):
        mock_stripe.return_value = MagicMock(
            id='ch_123',
            status='succeeded',
            amount=5000
        )
        
        result = PaymentService().charge(amount=5000, token='tok_visa')
        
        assert result.status == 'succeeded'
        mock_stripe.assert_called_once_with(
            amount=5000,
            currency='inr',
            source='tok_visa'
        )
    
    @patch('apps.payments.services.stripe.Charge.create')
    def test_charge_failure_retry(self, mock_stripe):
        mock_stripe.side_effect = [
            stripe.error.APIConnectionError('timeout'),
            MagicMock(id='ch_123', status='succeeded')
        ]
        
        result = PaymentService().charge(amount=5000, token='tok_visa')
        
        assert result.status == 'succeeded'
        assert mock_stripe.call_count == 2

    # Mock HTTP requests with responses library
    @responses.activate
    def test_external_api_call(self):
        responses.add(
            responses.GET,
            'https://api.exchangerate.host/latest',
            json={'rates': {'INR': 83.5}},
            status=200
        )
        
        rate = CurrencyService().get_rate('USD', 'INR')
        assert rate == 83.5


# Mock Celery tasks (don't actually run async)
@pytest.fixture(autouse=True)
def celery_eager(settings):
    settings.CELERY_TASK_ALWAYS_EAGER = True
    settings.CELERY_TASK_EAGER_PROPAGATES = True
```

---

## 7. Performance Tests

```python
import pytest
from django.test.utils import override_settings

@pytest.mark.django_db
class TestQueryPerformance:
    
    def test_product_list_query_count(self, django_assert_num_queries, api_client):
        """Product list should use constant number of queries regardless of count."""
        ProductFactory.create_batch(50, status='published')
        
        # Should be exactly 2 queries: COUNT + SELECT (with prefetch)
        with django_assert_num_queries(2):
            api_client.get('/api/v1/products/')
    
    def test_order_detail_no_n_plus_one(self, django_assert_num_queries, authenticated_client):
        """Order detail with 10 items should not cause N+1."""
        order = OrderFactory(user=authenticated_client.handler._force_user)
        OrderItemFactory.create_batch(10, order=order)
        
        # Should be 3 queries: order + items + products (prefetched)
        with django_assert_num_queries(3):
            authenticated_client.get(f'/api/v1/orders/{order.id}/')

    @pytest.mark.slow
    def test_bulk_create_performance(self):
        """Creating 10K products should take < 5 seconds."""
        import time
        
        products = [Product(name=f'Product {i}', price=i) for i in range(10000)]
        
        start = time.time()
        Product.objects.bulk_create(products, batch_size=1000)
        duration = time.time() - start
        
        assert duration < 5.0
```

---

## 8. Fixtures & Conftest Patterns

```python
# conftest.py
import pytest
from rest_framework.test import APIClient
from .factories import UserFactory

@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def user(db):
    return UserFactory()

@pytest.fixture
def admin_user(db):
    return UserFactory(is_staff=True, is_superuser=True)

@pytest.fixture
def authenticated_client(api_client, user):
    api_client.force_authenticate(user=user)
    return api_client

@pytest.fixture
def admin_client(api_client, admin_user):
    api_client.force_authenticate(admin_user)
    return admin_client

# Reusable data fixtures
@pytest.fixture
def sample_products(db):
    from .factories import ProductFactory, CategoryFactory
    electronics = CategoryFactory(name='Electronics')
    books = CategoryFactory(name='Books')
    return {
        'electronics': ProductFactory.create_batch(5, category=electronics),
        'books': ProductFactory.create_batch(3, category=books),
    }

# Freeze time for date-dependent tests
@pytest.fixture
def frozen_time():
    from freezegun import freeze_time
    with freeze_time('2024-06-15 10:00:00') as frozen:
        yield frozen
```

---

## 9. Test Commands

```bash
# Run all tests
pytest

# Run specific file/class/test
pytest apps/orders/tests/test_views.py
pytest apps/orders/tests/test_views.py::TestOrderAPI
pytest apps/orders/tests/test_views.py::TestOrderAPI::test_create_order_success

# Run with coverage
pytest --cov=apps --cov-report=html

# Run fast (parallel)
pytest -n auto  # uses all CPU cores

# Skip slow tests
pytest -m "not slow"

# Run only integration tests
pytest -m integration

# Show slowest 10 tests
pytest --durations=10

# Stop on first failure
pytest -x
```

---

## 10. Interview Questions

### Q: What's the testing pyramid for Django?
```
         /  E2E  \          (few: Selenium, full stack)
        /─────────\
       / Integration\       (moderate: API tests, DB interaction)
      /──────────────\
     /   Unit Tests   \     (many: pure functions, services, validators)
    /──────────────────\

Unit (70%):        Fast, test one function/method, mock dependencies
Integration (20%): Test API endpoints with real DB, factory data
E2E (10%):         Full browser tests (Selenium/Playwright), slow

For DRF APIs: most tests are integration (API + DB), not pure unit.
```

### Q: How do you test async Celery tasks?
```
1. CELERY_TASK_ALWAYS_EAGER = True in test settings
   → Tasks run synchronously during tests (no broker needed)

2. For testing retry/failure behavior:
   → Mock the task function, assert .delay() was called
   → Use celery.contrib.pytest fixtures

3. For integration testing task chains:
   → Use in-memory broker (kombu memory transport)
```
