# FastAPI Testing Strategies — Senior Level

## 1. Test Setup (pytest + httpx)

### conftest.py
```python
# tests/conftest.py
import pytest
import asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from main import app
from database import Base, get_db
from config import settings

# Test database
TEST_DATABASE_URL = "postgresql+asyncpg://postgres:test@localhost:5432/test_db"

engine = create_async_engine(TEST_DATABASE_URL, echo=False)
TestSessionLocal = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


@pytest.fixture(scope="session")
def event_loop():
    """Create event loop for entire test session."""
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()


@pytest.fixture(scope="session", autouse=True)
async def create_tables():
    """Create all tables once before tests, drop after."""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


@pytest.fixture
async def db_session():
    """Provide transactional test database session (rolls back after each test)."""
    async with engine.connect() as conn:
        transaction = await conn.begin()
        session = AsyncSession(bind=conn, expire_on_commit=False)

        yield session

        await session.close()
        await transaction.rollback()


@pytest.fixture
async def client(db_session):
    """HTTP client with overridden DB dependency."""
    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac

    app.dependency_overrides.clear()


@pytest.fixture
async def authenticated_client(client, db_session):
    """Client with valid auth token."""
    from auth.jwt import create_access_token
    from models import User

    user = User(
        email="test@example.com",
        password_hash="hashed",
        full_name="Test User",
        role="user",
    )
    db_session.add(user)
    await db_session.flush()

    token = create_access_token(str(user.id))
    client.headers["Authorization"] = f"Bearer {token}"
    client._test_user = user

    yield client
```

---

## 2. Unit Tests (Endpoints)

### CRUD Endpoint Tests
```python
# tests/test_products.py
import pytest
from uuid import uuid4

pytestmark = pytest.mark.asyncio


class TestCreateProduct:
    async def test_create_product_success(self, authenticated_client):
        payload = {
            "name": "Test Product",
            "price": 99.99,
            "category_id": str(uuid4()),
            "tags": ["electronics", "gadgets"],
        }

        response = await authenticated_client.post("/api/v1/products/", json=payload)

        assert response.status_code == 201
        data = response.json()
        assert data["name"] == "Test Product"
        assert data["price"] == 99.99
        assert "id" in data
        assert "created_at" in data

    async def test_create_product_invalid_price(self, authenticated_client):
        payload = {"name": "Bad Product", "price": -10, "category_id": str(uuid4())}

        response = await authenticated_client.post("/api/v1/products/", json=payload)

        assert response.status_code == 422
        errors = response.json()["detail"]
        assert any("price" in str(e) for e in errors)

    async def test_create_product_unauthorized(self, client):
        payload = {"name": "Product", "price": 10, "category_id": str(uuid4())}

        response = await client.post("/api/v1/products/", json=payload)

        assert response.status_code == 401


class TestListProducts:
    async def test_list_products_pagination(self, authenticated_client, db_session):
        # Create 25 products
        from models import Product
        for i in range(25):
            db_session.add(Product(name=f"Product {i}", price=10.0, slug=f"product-{i}"))
        await db_session.flush()

        # Page 1
        response = await authenticated_client.get("/api/v1/products/?page=1&page_size=10")
        assert response.status_code == 200
        data = response.json()
        assert len(data["items"]) == 10
        assert data["total"] == 25
        assert data["has_next"] is True

        # Page 3
        response = await authenticated_client.get("/api/v1/products/?page=3&page_size=10")
        data = response.json()
        assert len(data["items"]) == 5
        assert data["has_next"] is False

    async def test_list_products_search(self, authenticated_client, db_session):
        from models import Product
        db_session.add(Product(name="iPhone 15", price=999, slug="iphone-15"))
        db_session.add(Product(name="Samsung Galaxy", price=899, slug="samsung"))
        await db_session.flush()

        response = await authenticated_client.get("/api/v1/products/?search=iphone")
        data = response.json()
        assert len(data["items"]) == 1
        assert data["items"][0]["name"] == "iPhone 15"


class TestUpdateProduct:
    async def test_partial_update(self, authenticated_client, db_session):
        from models import Product
        product = Product(name="Original", price=50.0, slug="original")
        db_session.add(product)
        await db_session.flush()

        response = await authenticated_client.patch(
            f"/api/v1/products/{product.id}",
            json={"price": 75.0},
        )

        assert response.status_code == 200
        assert response.json()["price"] == 75.0
        assert response.json()["name"] == "Original"  # unchanged
```

---

## 3. Testing Authentication & Authorization

```python
# tests/test_auth.py
import pytest

pytestmark = pytest.mark.asyncio


class TestLogin:
    async def test_login_success(self, client, db_session):
        from models import User
        from auth.jwt import hash_password

        user = User(
            email="login@test.com",
            password_hash=hash_password("SecurePass123"),
            full_name="Login User",
        )
        db_session.add(user)
        await db_session.flush()

        response = await client.post(
            "/api/v1/auth/login",
            data={"username": "login@test.com", "password": "SecurePass123"},
        )

        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert "refresh_token" in data
        assert data["token_type"] == "bearer"

    async def test_login_wrong_password(self, client, db_session):
        from models import User
        from auth.jwt import hash_password

        user = User(
            email="wrong@test.com",
            password_hash=hash_password("CorrectPass"),
            full_name="User",
        )
        db_session.add(user)
        await db_session.flush()

        response = await client.post(
            "/api/v1/auth/login",
            data={"username": "wrong@test.com", "password": "WrongPass"},
        )

        assert response.status_code == 401

    async def test_login_nonexistent_user(self, client):
        response = await client.post(
            "/api/v1/auth/login",
            data={"username": "noone@test.com", "password": "pass"},
        )
        assert response.status_code == 401


class TestRBAC:
    async def test_admin_endpoint_as_user(self, authenticated_client):
        response = await authenticated_client.delete("/api/v1/admin/users/some-id")
        assert response.status_code == 403

    async def test_admin_endpoint_as_admin(self, client, db_session):
        from models import User
        from auth.jwt import create_access_token

        admin = User(email="admin@test.com", password_hash="x", full_name="Admin", role="admin")
        db_session.add(admin)
        await db_session.flush()

        token = create_access_token(str(admin.id))
        client.headers["Authorization"] = f"Bearer {token}"

        response = await client.get("/api/v1/admin/users/")
        assert response.status_code == 200
```

---

## 4. Testing with Mocks

### Mocking External Services
```python
# tests/test_orders.py
import pytest
from unittest.mock import AsyncMock, patch

pytestmark = pytest.mark.asyncio


class TestCreateOrder:
    @patch("services.payment.PaymentGateway.charge")
    async def test_create_order_payment_success(self, mock_charge, authenticated_client, db_session):
        # Setup mock
        mock_charge.return_value = AsyncMock(success=True, transaction_id="txn_123")

        # Create product in DB
        from models import Product
        product = Product(name="Widget", price=25.0, stock=10, slug="widget")
        db_session.add(product)
        await db_session.flush()

        # Create order
        response = await authenticated_client.post("/api/v1/orders/", json={
            "items": [{"product_id": str(product.id), "quantity": 2}],
            "payment_method": "card_xxx",
        })

        assert response.status_code == 201
        assert response.json()["total"] == 50.0
        mock_charge.assert_called_once_with(50.0, "card_xxx")

    @patch("services.payment.PaymentGateway.charge")
    async def test_create_order_payment_fails(self, mock_charge, authenticated_client, db_session):
        mock_charge.side_effect = Exception("Payment declined")

        from models import Product
        product = Product(name="Widget", price=25.0, stock=10, slug="widget2")
        db_session.add(product)
        await db_session.flush()

        response = await authenticated_client.post("/api/v1/orders/", json={
            "items": [{"product_id": str(product.id), "quantity": 1}],
            "payment_method": "card_bad",
        })

        assert response.status_code == 400
        assert "payment" in response.json()["detail"].lower()


class TestWithRedis:
    @pytest.fixture
    def mock_redis(self, monkeypatch):
        """Mock Redis for cache tests."""
        mock = AsyncMock()
        mock.get.return_value = None  # cache miss by default
        mock.set.return_value = True
        monkeypatch.setattr("app.state.redis", mock)
        return mock

    async def test_product_caching(self, authenticated_client, mock_redis, db_session):
        from models import Product
        product = Product(name="Cached", price=10, slug="cached")
        db_session.add(product)
        await db_session.flush()

        # First call = cache miss, hits DB
        response = await authenticated_client.get(f"/api/v1/products/{product.id}")
        assert response.status_code == 200
        mock_redis.set.assert_called_once()

        # Simulate cache hit
        import json
        mock_redis.get.return_value = json.dumps(response.json())

        response2 = await authenticated_client.get(f"/api/v1/products/{product.id}")
        assert response2.status_code == 200
```

---

## 5. Integration Tests

### Testing Full Flows
```python
# tests/integration/test_order_flow.py
import pytest

pytestmark = pytest.mark.asyncio


class TestOrderFlow:
    """End-to-end order lifecycle test."""

    async def test_complete_order_lifecycle(self, authenticated_client, db_session):
        from models import Product, Category

        # 1. Create category
        cat = Category(name="Electronics", slug="electronics")
        db_session.add(cat)
        await db_session.flush()

        # 2. Create product with stock
        product = Product(
            name="Laptop", price=999.99, stock=5,
            slug="laptop", category_id=cat.id,
        )
        db_session.add(product)
        await db_session.flush()

        # 3. Place order
        order_response = await authenticated_client.post("/api/v1/orders/", json={
            "items": [{"product_id": str(product.id), "quantity": 2}],
            "shipping_address": {
                "street": "123 Main St",
                "city": "Mumbai",
                "state": "MH",
                "zip_code": "400001",
            },
        })
        assert order_response.status_code == 201
        order_id = order_response.json()["id"]

        # 4. Verify stock deducted
        product_response = await authenticated_client.get(f"/api/v1/products/{product.id}")
        assert product_response.json()["stock"] == 3  # 5 - 2

        # 5. Get order status
        status_response = await authenticated_client.get(f"/api/v1/orders/{order_id}")
        assert status_response.json()["status"] == "pending"
        assert status_response.json()["total"] == 1999.98  # 999.99 * 2

    async def test_order_insufficient_stock(self, authenticated_client, db_session):
        from models import Product

        product = Product(name="Rare Item", price=50, stock=1, slug="rare")
        db_session.add(product)
        await db_session.flush()

        response = await authenticated_client.post("/api/v1/orders/", json={
            "items": [{"product_id": str(product.id), "quantity": 5}],
            "shipping_address": {
                "street": "x", "city": "x", "state": "x", "zip_code": "000000",
            },
        })

        assert response.status_code == 409
        assert "stock" in response.json()["detail"].lower()
```

---

## 6. Factory Pattern (Test Data)

```python
# tests/factories.py
import factory
from factory import fuzzy
from models import User, Product, Order
from uuid import uuid4
from datetime import datetime


class UserFactory(factory.Factory):
    class Meta:
        model = User

    id = factory.LazyFunction(uuid4)
    email = factory.Sequence(lambda n: f"user{n}@test.com")
    password_hash = "hashed_password"
    full_name = factory.Faker("name")
    role = "user"
    is_active = True
    created_at = factory.LazyFunction(datetime.utcnow)


class ProductFactory(factory.Factory):
    class Meta:
        model = Product

    id = factory.LazyFunction(uuid4)
    name = factory.Faker("catch_phrase")
    slug = factory.Sequence(lambda n: f"product-{n}")
    price = fuzzy.FuzzyDecimal(10.0, 1000.0, precision=2)
    stock = fuzzy.FuzzyInteger(0, 100)
    status = "active"
    created_at = factory.LazyFunction(datetime.utcnow)


class OrderFactory(factory.Factory):
    class Meta:
        model = Order

    id = factory.LazyFunction(uuid4)
    status = "pending"
    total = fuzzy.FuzzyDecimal(10.0, 5000.0, precision=2)
    created_at = factory.LazyFunction(datetime.utcnow)


# Usage in tests
async def test_list_user_orders(authenticated_client, db_session):
    user = authenticated_client._test_user

    orders = [OrderFactory(user_id=user.id) for _ in range(5)]
    db_session.add_all(orders)
    await db_session.flush()

    response = await authenticated_client.get("/api/v1/orders/")
    assert len(response.json()["items"]) == 5
```

---

## 7. Performance Testing

```python
# tests/performance/test_load.py
import pytest
import asyncio
import time

pytestmark = pytest.mark.asyncio


class TestPerformance:
    async def test_concurrent_requests(self, client):
        """Ensure endpoint handles concurrent load."""
        async def make_request():
            return await client.get("/api/v1/products/?page=1&page_size=10")

        start = time.perf_counter()
        responses = await asyncio.gather(*[make_request() for _ in range(100)])
        duration = time.perf_counter() - start

        assert all(r.status_code == 200 for r in responses)
        assert duration < 5.0  # 100 concurrent in under 5s

    async def test_response_time(self, authenticated_client, db_session):
        """Single request should respond quickly."""
        start = time.perf_counter()
        response = await authenticated_client.get("/api/v1/products/")
        duration = time.perf_counter() - start

        assert response.status_code == 200
        assert duration < 0.5  # under 500ms
```

---

## 8. Interview Questions

### Q: How do you test async FastAPI endpoints?
```
1. pytest + pytest-asyncio for async test functions
2. httpx.AsyncClient with ASGITransport (not requests/TestClient for async)
3. Override dependencies with app.dependency_overrides
4. Transactional fixtures: begin transaction → run test → rollback
5. No real HTTP server needed (ASGI transport calls app directly)

Key difference from Django:
  Django: TestCase with self.client.get(...)
  FastAPI: async with AsyncClient(transport=ASGITransport(app=app)) as client
```

### Q: How do you isolate tests that share a database?
```
Strategy: Transactional isolation

1. Each test runs inside a database transaction
2. At test end, transaction is ROLLED BACK (never committed)
3. Database is always clean for next test
4. Tests can run in parallel (each has its own transaction)

Alternative: Truncate tables between tests (slower but simpler)

For CI: Use service containers (GitHub Actions services:)
  - postgres:16 on port 5432
  - redis:7 on port 6379
  - Run tests with test-specific DATABASE_URL
```
