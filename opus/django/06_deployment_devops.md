# Django Deployment & DevOps — Senior Level

## 1. Docker Setup (Production-Ready)

### Dockerfile
```dockerfile
# Multi-stage build: smaller final image
FROM python:3.12-slim as builder

WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Install system deps for psycopg2, pillow, etc.
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# --- Final stage ---
FROM python:3.12-slim

WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/install/bin:$PATH" \
    PYTHONPATH="/install/lib/python3.12/site-packages"

# Runtime deps only
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 curl && \
    rm -rf /var/lib/apt/lists/*

COPY --from=builder /install /install
COPY . .

# Collect static files at build time
RUN python manage.py collectstatic --noinput

# Non-root user (security)
RUN adduser --disabled-password --no-create-home appuser
USER appuser

EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8000/health/ || exit 1

# Gunicorn entrypoint
CMD ["gunicorn", "myproject.wsgi:application", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", \
     "--worker-class", "gthread", \
     "--threads", "2", \
     "--timeout", "120", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

### docker-compose.yml (Full Stack)
```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - static_files:/app/staticfiles
      - media_files:/app/media

  db:
    image: postgres:16
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  celery_worker:
    build: .
    command: celery -A myproject worker -l info --concurrency=4
    env_file: .env
    depends_on:
      - redis
      - db

  celery_beat:
    build: .
    command: celery -A myproject beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler
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
      - static_files:/var/www/static
      - media_files:/var/www/media
      - ./certs:/etc/nginx/certs
    depends_on:
      - web

volumes:
  postgres_data:
  static_files:
  media_files:
```

---

## 2. Gunicorn Configuration

### gunicorn.conf.py
```python
import multiprocessing
import os

# Server socket
bind = '0.0.0.0:8000'
backlog = 2048

# Worker processes
workers = int(os.environ.get('GUNICORN_WORKERS', multiprocessing.cpu_count() * 2 + 1))
worker_class = 'gthread'  # use 'uvicorn.workers.UvicornWorker' for async
threads = 2
worker_connections = 1000
max_requests = 1000        # restart worker after 1000 requests (prevent memory leaks)
max_requests_jitter = 100  # randomize restart to avoid all workers restarting at once

# Timeout
timeout = 120              # kill worker if request takes > 120s
graceful_timeout = 30      # wait 30s for in-flight requests on shutdown
keepalive = 5

# Logging
accesslog = '-'            # stdout
errorlog = '-'             # stderr
loglevel = 'info'
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s" %(D)s'

# Preload app (share memory between workers, faster startup)
preload_app = True

# Hooks
def on_starting(server):
    pass

def post_fork(server, worker):
    server.log.info(f"Worker spawned (pid: {worker.pid})")

def worker_exit(server, worker):
    server.log.info(f"Worker exited (pid: {worker.pid})")
```

### Worker Types Decision
```
sync (default):   1 request per worker at a time
                  Best for: CPU-bound views (calculations, rendering)
                  Workers needed: 2 * CPU + 1

gthread:          Multiple threads per worker
                  Best for: I/O-bound views (DB queries, file ops)
                  Workers: CPU + 1, Threads: 2-4

gevent/eventlet:  Async via greenlets (1000s of concurrent connections)
                  Best for: many slow I/O operations, WebSocket-like
                  Workers: 1 per CPU, greenlets: 1000+

uvicorn:          ASGI (true async, Django 4.1+)
                  Best for: async views, Django Channels
                  Workers: CPU count
```

---

## 3. Nginx Configuration

```nginx
upstream django {
    server web:8000;
    # For multiple instances:
    # server web1:8000 weight=3;
    # server web2:8000 weight=2;
}

server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    # SSL
    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_min_length 256;

    # Static files (served by Nginx, not Django)
    location /static/ {
        alias /var/www/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    location /media/ {
        alias /var/www/media/;
        expires 7d;
    }

    # Django app
    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_read_timeout 120s;
        proxy_send_timeout 120s;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        
        # File upload limit
        client_max_body_size 50M;
    }

    # Health check (no logging)
    location /health/ {
        proxy_pass http://django;
        access_log off;
    }

    # Block hidden files
    location ~ /\. {
        deny all;
    }
}
```

---

## 4. CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: myapp
  ECS_CLUSTER: production
  ECS_SERVICE: django-web

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test_db
          POSTGRES_PASSWORD: test_pass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports: ['5432:5432']
      redis:
        image: redis:7
        ports: ['6379:6379']
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      
      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt
      
      - name: Run linting
        run: |
          ruff check .
          ruff format --check .
      
      - name: Run tests
        env:
          DATABASE_URL: postgres://postgres:test_pass@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379/0
        run: pytest --cov=apps --cov-report=xml -n auto
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
      
      - name: Run database migrations
        run: |
          aws ecs run-task \
            --cluster $ECS_CLUSTER \
            --task-definition django-migrate \
            --overrides '{"containerOverrides":[{"name":"web","command":["python","manage.py","migrate","--noinput"]}]}'
      
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster $ECS_CLUSTER \
            --service $ECS_SERVICE \
            --force-new-deployment
      
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster $ECS_CLUSTER \
            --services $ECS_SERVICE
```

---

## 5. AWS Deployment Architecture

```
                        ┌──────────────┐
                        │  Route 53    │ (DNS)
                        └──────┬───────┘
                               │
                        ┌──────▼───────┐
                        │ CloudFront   │ (CDN: static + media from S3)
                        └──────┬───────┘
                               │
                        ┌──────▼───────┐
                        │     ALB      │ (Application Load Balancer)
                        │  (SSL term.) │
                        └──┬────────┬──┘
                           │        │
              ┌────────────▼─┐   ┌──▼────────────┐
              │ ECS Task 1   │   │ ECS Task 2     │
              │ (Gunicorn)   │   │ (Gunicorn)     │
              └──────┬───────┘   └──────┬─────────┘
                     │                  │
        ┌────────────┼──────────────────┼────────────────┐
        │            │                  │                 │
   ┌────▼────┐  ┌───▼────┐  ┌─────────▼──────┐  ┌─────▼─────┐
   │ RDS     │  │ Redis  │  │ S3             │  │ SQS/Redis │
   │ (Postgres│  │ (Elasti│  │ (static+media) │  │ (Celery   │
   │  + read  │  │  Cache)│  │                │  │  broker)  │
   │  replica)│  │        │  │                │  │           │
   └──────────┘  └────────┘  └────────────────┘  └─────┬─────┘
                                                        │
                                                 ┌──────▼──────┐
                                                 │ ECS Task    │
                                                 │ (Celery     │
                                                 │  workers)   │
                                                 └─────────────┘
```

### ECS Task Definition
```json
{
  "family": "django-web",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "123456789.dkr.ecr.ap-south-1.amazonaws.com/myapp:latest",
      "portMappings": [{"containerPort": 8000}],
      "environment": [
        {"name": "DJANGO_SETTINGS_MODULE", "value": "myproject.settings.production"}
      ],
      "secrets": [
        {"name": "SECRET_KEY", "valueFrom": "arn:aws:secretsmanager:ap-south-1:123:secret:prod/django"},
        {"name": "DATABASE_URL", "valueFrom": "arn:aws:secretsmanager:ap-south-1:123:secret:prod/db"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/django-web",
          "awslogs-region": "ap-south-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8000/health/ || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3
      }
    }
  ]
}
```

---

## 6. Environment Configuration

### settings/ Structure
```
myproject/
├── settings/
│   ├── __init__.py        # empty
│   ├── base.py            # shared settings
│   ├── development.py     # local dev overrides
│   ├── staging.py         # staging environment
│   └── production.py      # production settings
```

```python
# settings/base.py (shared)
import os
import environ

env = environ.Env()
BASE_DIR = Path(__file__).resolve().parent.parent.parent

SECRET_KEY = env('SECRET_KEY')
INSTALLED_APPS = [...]
MIDDLEWARE = [...]

# settings/production.py
from .base import *

DEBUG = False
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')

DATABASES = {'default': env.db('DATABASE_URL')}
DATABASES['default']['CONN_MAX_AGE'] = 600  # persistent connections

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': env('REDIS_URL'),
    }
}

# Logging to CloudWatch
LOGGING = {
    'version': 1,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'json',
        },
    },
    'formatters': {
        'json': {
            'class': 'pythonjsonlogger.jsonlogger.JsonFormatter',
            'format': '%(asctime)s %(levelname)s %(name)s %(message)s',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'INFO',
    },
}
```

---

## 7. Zero-Downtime Deployment

```
Step 1: Build new image
Step 2: Run migrations (must be backwards-compatible!)
Step 3: Deploy new code (rolling update)
Step 4: Health checks pass → old containers drain
Step 5: All traffic on new version

Key rules:
- Migrations must be backwards-compatible (old code + new DB works)
- Add columns as nullable first, populate, then add NOT NULL
- Never rename/delete columns in same deploy as code change
- Use feature flags for big changes (turn on after deploy is stable)
```

### Rolling Update ECS Config
```json
{
  "deploymentConfiguration": {
    "maximumPercent": 200,
    "minimumHealthyPercent": 100,
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true
    }
  }
}
```

---

## 8. Monitoring & Alerting

### Structured Logging
```python
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
)

logger = structlog.get_logger()

# In views/services:
logger.info("order_created", order_id=order.id, user_id=user.id, total=str(order.total))
# Output: {"event":"order_created","order_id":"uuid","user_id":5,"total":"99.99","level":"info","timestamp":"2024-01-15T10:30:00Z"}
```

### CloudWatch Alarms
```python
# Important metrics to monitor:
# 1. Request latency (p50, p95, p99)
# 2. Error rate (5xx responses / total requests)
# 3. CPU/Memory utilization per container
# 4. Database connection count
# 5. Redis memory usage
# 6. Celery queue depth
# 7. Response time per endpoint

# Alert thresholds:
# p99 latency > 2s → WARNING
# Error rate > 1% → CRITICAL
# CPU > 80% for 5 min → scale up
# DB connections > 80% of max → CRITICAL
# Celery queue > 1000 tasks → WARNING
```

---

## 9. Interview Questions

### Q: Walk me through a Django deployment pipeline.
```
1. Developer pushes to main
2. CI runs: lint → test → security scan → build Docker image
3. Push image to ECR (container registry)
4. Run database migrations (backwards-compatible)
5. ECS rolling update: spin up new tasks → health check → drain old tasks
6. CloudWatch monitors error rate post-deploy
7. If error rate spikes → automatic rollback via circuit breaker

Key: zero downtime. Users never see errors during deploy.
```

### Q: How do you scale a Django application?
```
Horizontal scaling (add more instances):
  - ECS/K8s auto-scaling based on CPU/request count
  - Multiple Gunicorn workers per container
  - Load balancer distributes traffic

Vertical scaling (bigger instances):
  - More CPU/RAM per container
  - More Gunicorn workers (2*CPU+1)

Database scaling:
  - Read replicas for GET requests
  - Connection pooling (PgBouncer)
  - Caching (Redis) to reduce DB load

Async offloading:
  - Celery for background tasks
  - Move heavy work off the request cycle
```

### Q: How many Gunicorn workers should you run?
```
Formula: 2 * CPU_CORES + 1

Why: at any given time, some workers are waiting on I/O (DB, network).
Having 2x CPU means while one worker waits, another uses the CPU.
The +1 accounts for the master process overhead.

4-core machine → 9 workers
With threads (gthread): 4 workers × 2 threads = 8 concurrent requests

For memory-constrained containers: fewer workers, more threads.
For CPU-bound apps: more workers, fewer threads.
```
