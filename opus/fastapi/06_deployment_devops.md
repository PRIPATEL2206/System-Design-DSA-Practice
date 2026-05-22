# FastAPI Deployment & DevOps — Senior Level

## 1. Production Dockerfile

```dockerfile
# Multi-stage build
FROM python:3.12-slim as builder

WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# --- Final stage ---
FROM python:3.12-slim

WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 \
    PATH="/install/bin:$PATH" \
    PYTHONPATH="/install/lib/python3.12/site-packages"

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 curl && rm -rf /var/lib/apt/lists/*

COPY --from=builder /install /install
COPY . .

RUN adduser --disabled-password --no-create-home appuser
USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Gunicorn + Uvicorn workers (production)
CMD ["gunicorn", "main:app", \
     "-w", "4", \
     "-k", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--timeout", "120", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

---

## 2. Uvicorn vs Gunicorn Configuration

### Development
```bash
# Hot reload for development
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Production
```python
# gunicorn.conf.py
import multiprocessing

bind = "0.0.0.0:8000"
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "uvicorn.workers.UvicornWorker"
timeout = 120
keepalive = 5
max_requests = 1000
max_requests_jitter = 100
accesslog = "-"
errorlog = "-"
loglevel = "info"
preload_app = True
```

```
Why Gunicorn + Uvicorn (not just Uvicorn)?

Uvicorn alone:  single process, if it crashes → all connections lost
Gunicorn:       process manager, restarts crashed workers, manages multiple processes

Production: gunicorn -k uvicorn.workers.UvicornWorker -w 4
  = 4 Uvicorn processes managed by Gunicorn
  = if one dies, Gunicorn spawns a replacement
  = max_requests prevents memory leaks (restart after 1000 requests)
```

---

## 3. docker-compose (Full Stack)

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
    ports:
      - "6379:6379"

  worker:
    build: .
    command: arq app.worker.WorkerSettings
    env_file: .env
    depends_on:
      - redis
      - db

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - api

volumes:
  postgres_data:
```

---

## 4. Nginx Configuration

```nginx
upstream fastapi {
    server api:8000;
}

server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # Gzip
    gzip on;
    gzip_types application/json text/plain;
    gzip_min_length 256;

    # Security
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header Strict-Transport-Security "max-age=31536000" always;

    location / {
        proxy_pass http://fastapi;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        proxy_read_timeout 120s;
        client_max_body_size 50M;
    }

    location /health {
        proxy_pass http://fastapi;
        access_log off;
    }
}
```

---

## 5. CI/CD (GitHub Actions)

```yaml
name: Deploy FastAPI

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test_db
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
      redis:
        image: redis:7
        ports: ['6379:6379']
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: ruff check . && ruff format --check .
      - run: pytest --cov=app -n auto
        env:
          DATABASE_URL: postgresql+asyncpg://postgres:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      - uses: aws-actions/amazon-ecr-login@v2
        id: ecr
      - run: |
          docker build -t ${{ steps.ecr.outputs.registry }}/fastapi-app:${{ github.sha }} .
          docker push ${{ steps.ecr.outputs.registry }}/fastapi-app:${{ github.sha }}
      - run: |
          aws ecs update-service --cluster production --service fastapi --force-new-deployment
```

---

## 6. Health Check Endpoint

```python
from fastapi import APIRouter
from sqlalchemy import text

router = APIRouter(tags=["Health"])

@router.get("/health")
async def health_check(db = Depends(get_db)):
    checks = {}
    
    # Database
    try:
        await db.execute(text("SELECT 1"))
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"error: {str(e)}"
    
    # Redis
    try:
        redis = app.state.redis
        await redis.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"error: {str(e)}"
    
    all_ok = all(v == "ok" for v in checks.values())
    return {"status": "healthy" if all_ok else "unhealthy", "checks": checks}
```

---

## 7. Interview Questions

### Q: How do you deploy FastAPI to production?
```
1. Docker container with Gunicorn + Uvicorn workers
2. Nginx as reverse proxy (SSL, gzip, rate limit)
3. ECS Fargate or Kubernetes for orchestration
4. ALB for load balancing + health checks
5. RDS for database, ElastiCache for Redis
6. CI/CD: test → build image → push ECR → deploy ECS

Key differences from Django:
  - No collectstatic step (API-only, no static files)
  - Use uvicorn.workers.UvicornWorker (not gthread)
  - Native WebSocket support (no Channels needed)
  - Faster cold start (smaller framework)
```

### Q: How many Uvicorn workers for a FastAPI app?
```
Formula: 2 * CPU_CORES + 1 (same as Gunicorn)

But since FastAPI is async:
  - Each worker handles MANY concurrent requests via event loop
  - 4 workers can handle 10,000+ concurrent connections
  - Unlike Django where 4 workers = 4 concurrent requests

4-core machine: 9 workers, each handling ~2000 concurrent connections
  = ~18,000 concurrent capacity (vs Django's 9)

Memory-limited: fewer workers (each ~50-100MB)
CPU-bound: more workers (offload to thread pool anyway)
```
