# DevOps Phase 13 — Real-world DevOps Projects
## Spring Boot Deployment · Flask Deployment · Kubernetes Production · Blue-Green · Canary

---

> **Who this is for:** Beginners ready to build real production-grade deployments from scratch. Each project is a complete, hands-on walkthrough.

---

## Table of Contents

1. [Project 1: Deploy a Spring Boot Application (Full CI/CD)](#project-1-deploy-a-spring-boot-application)
2. [Project 2: Deploy a Flask Application with Docker & Nginx](#project-2-deploy-a-flask-application)
3. [Project 3: Kubernetes Production Deployment](#project-3-kubernetes-production-deployment)
4. [Project 4: Blue-Green Deployment](#project-4-blue-green-deployment)
5. [Project 5: Canary Deployment with Argo Rollouts](#project-5-canary-deployment)
6. [Project 6: Complete GitOps Pipeline with ArgoCD](#project-6-gitops-pipeline-with-argocd)
7. [Interview Mastery](#interview-mastery)

---

## Project 1: Deploy a Spring Boot Application

### Overview

We will build a complete CI/CD pipeline that:
1. Builds a Spring Boot application
2. Runs tests
3. Builds a Docker image
4. Pushes to a container registry
5. Deploys to Kubernetes
6. Exposes via Ingress with HTTPS

```mermaid
flowchart LR
    A[Developer\nPushes Code] --> B[GitHub Actions\nCI Pipeline]
    B --> C[Build JAR\nRun Tests]
    C --> D[Build Docker\nImage]
    D --> E[Push to\nECR/DockerHub]
    E --> F[Deploy to\nKubernetes]
    F --> G[Ingress\nHTTPS]
    G --> H[Users]
```

### Step 1: The Spring Boot Application

```java
// src/main/java/com/example/demo/DemoApplication.java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

```java
// src/main/java/com/example/demo/HealthController.java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.Map;

@RestController
public class HealthController {

    @GetMapping("/health")
    public Map<String, String> health() {
        return Map.of("status", "UP", "version", System.getenv().getOrDefault("APP_VERSION", "unknown"));
    }

    @GetMapping("/api/greeting")
    public Map<String, String> greeting() {
        return Map.of("message", "Hello from Spring Boot!");
    }
}
```

```yaml
# src/main/resources/application.yml
server:
  port: 8080
spring:
  application:
    name: demo-app
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
```

```xml
<!-- pom.xml (key dependencies) -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Step 2: Multi-Stage Dockerfile

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app
COPY pom.xml .
COPY mvnw .
COPY .mvn .mvn

# Download dependencies (cached layer)
RUN ./mvnw dependency:go-offline -B

# Copy source and build
COPY src ./src
RUN ./mvnw package -DskipTests -B

# Stage 2: Production
FROM eclipse-temurin:17-jre-alpine

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY --from=builder /app/target/*.jar app.jar

RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1

ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

### Step 3: GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven

    - name: Run tests
      run: ./mvnw test -B

    - name: Upload test results
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: test-results
        path: target/surefire-reports/

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
    - uses: actions/checkout@v4

    - name: Log in to registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,prefix=
          type=raw,value=latest

    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Scan image for vulnerabilities
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'HIGH,CRITICAL'

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
    - uses: actions/checkout@v4

    - name: Configure kubectl
      uses: azure/setup-kubectl@v3

    - name: Set Kubernetes context
      uses: azure/k8s-set-context@v3
      with:
        method: kubeconfig
        kubeconfig: ${{ secrets.KUBE_CONFIG }}

    - name: Deploy to Kubernetes
      run: |
        export IMAGE_TAG=${{ github.sha }}
        envsubst < k8s/deployment.yml | kubectl apply -f -
        kubectl apply -f k8s/service.yml
        kubectl apply -f k8s/ingress.yml

    - name: Wait for rollout
      run: |
        kubectl rollout status deployment/demo-app -n production --timeout=300s

    - name: Verify deployment
      run: |
        kubectl get pods -n production -l app=demo-app
        kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- \
          curl -s http://demo-app.production.svc.cluster.local:8080/health
```

### Step 4: Kubernetes Manifests

```yaml
# k8s/deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: production
  labels:
    app: demo-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: demo-app
        version: "${IMAGE_TAG}"
    spec:
      serviceAccountName: demo-app
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: demo-app
      containers:
      - name: demo-app
        image: ghcr.io/myorg/demo-app:${IMAGE_TAG}
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: APP_VERSION
          value: "${IMAGE_TAG}"
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 10
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 10"]

---
# k8s/service.yml
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: production
spec:
  selector:
    app: demo-app
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
  type: ClusterIP

---
# k8s/ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-app
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: demo-app-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-app
            port:
              number: 8080

---
# k8s/hpa.yml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: demo-app
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: demo-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## Project 2: Deploy a Flask Application

### Overview

Build and deploy a Python Flask API with:
- Docker multi-stage build
- Nginx reverse proxy
- PostgreSQL database
- Redis caching
- Docker Compose for local dev
- Kubernetes for production

```mermaid
flowchart LR
    A[Users] --> B[Nginx\n:443 SSL]
    B --> C[Gunicorn\nFlask App\n:5000]
    C --> D[(PostgreSQL\n:5432)]
    C --> E[(Redis\n:6379)]
```

### Step 1: Flask Application

```python
# app/__init__.py
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
import redis
import os

db = SQLAlchemy()
migrate = Migrate()
cache = None

def create_app():
    app = Flask(__name__)

    app.config['SQLALCHEMY_DATABASE_URI'] = os.environ.get(
        'DATABASE_URL', 'postgresql://user:pass@localhost:5432/mydb'
    )
    app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

    db.init_app(app)
    migrate.init_app(app, db)

    global cache
    cache = redis.Redis(
        host=os.environ.get('REDIS_HOST', 'localhost'),
        port=6379,
        decode_responses=True
    )

    from app.routes import api_bp
    app.register_blueprint(api_bp)

    return app
```

```python
# app/routes.py
from flask import Blueprint, jsonify, request
from app import db, cache
from app.models import User
import json

api_bp = Blueprint('api', __name__)

@api_bp.route('/health')
def health():
    try:
        db.session.execute(db.text('SELECT 1'))
        cache.ping()
        return jsonify({'status': 'healthy', 'database': 'connected', 'cache': 'connected'})
    except Exception as e:
        return jsonify({'status': 'unhealthy', 'error': str(e)}), 503

@api_bp.route('/api/users', methods=['GET'])
def get_users():
    cached = cache.get('users:all')
    if cached:
        return jsonify(json.loads(cached))

    users = User.query.all()
    result = [{'id': u.id, 'name': u.name, 'email': u.email} for u in users]

    cache.setex('users:all', 300, json.dumps(result))
    return jsonify(result)

@api_bp.route('/api/users', methods=['POST'])
def create_user():
    data = request.get_json()
    user = User(name=data['name'], email=data['email'])
    db.session.add(user)
    db.session.commit()

    cache.delete('users:all')
    return jsonify({'id': user.id, 'name': user.name}), 201
```

```python
# app/models.py
from app import db
from datetime import datetime

class User(db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
```

```
# requirements.txt
flask==3.0.0
flask-sqlalchemy==3.1.1
flask-migrate==4.0.5
gunicorn==21.2.0
psycopg2-binary==2.9.9
redis==5.0.1
```

### Step 2: Production Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim-bookworm AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.11-slim-bookworm

RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

COPY --from=builder /install /usr/local
COPY app/ ./app/
COPY migrations/ ./migrations/
COPY wsgi.py .

RUN chown -R appuser:appuser /app
USER appuser

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')" || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "--threads", "2", \
     "--timeout", "120", "--access-logfile", "-", "--error-logfile", "-", "wsgi:app"]
```

```python
# wsgi.py
from app import create_app
app = create_app()
```

### Step 3: Nginx Configuration

```nginx
# nginx/nginx.conf
upstream flask_app {
    server app:5000;
}

server {
    listen 80;
    server_name api.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    client_max_body_size 10M;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Strict-Transport-Security "max-age=31536000" always;

    # Gzip compression
    gzip on;
    gzip_types application/json text/plain;
    gzip_min_length 1000;

    location / {
        proxy_pass http://flask_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 30s;
        proxy_read_timeout 60s;
    }

    location /health {
        proxy_pass http://flask_app;
        access_log off;
    }

    location /static/ {
        alias /app/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

### Step 4: Docker Compose (Local Development)

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    environment:
      - DATABASE_URL=postgresql://myuser:mypass@db:5432/mydb
      - REDIS_HOST=redis
      - FLASK_ENV=production
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - app
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
    restart: unless-stopped

volumes:
  postgres_data:
```

### Step 5: Deploy Commands

```bash
# Local development
docker compose up --build -d

# Run database migrations
docker compose exec app flask db upgrade

# Check logs
docker compose logs -f app

# Production deployment (single server with Docker Compose)
scp docker-compose.prod.yml server:/app/
ssh server "cd /app && docker compose -f docker-compose.prod.yml pull && docker compose -f docker-compose.prod.yml up -d"

# Kubernetes deployment (production)
kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/secrets.yml
kubectl apply -f k8s/postgres.yml
kubectl apply -f k8s/redis.yml
kubectl apply -f k8s/flask-app.yml
kubectl apply -f k8s/ingress.yml
```

---

## Project 3: Kubernetes Production Deployment

### Overview

Deploy a full microservices application to production Kubernetes with:
- Multiple services
- Service mesh (Istio)
- Observability (Prometheus + Grafana)
- GitOps (ArgoCD)
- Secrets management (External Secrets)

```mermaid
flowchart TB
    subgraph Cluster ["Production Kubernetes Cluster"]
        subgraph Ingress ["Ingress Layer"]
            IG[Nginx Ingress Controller]
        end
        
        subgraph Apps ["Application Layer"]
            FE[Frontend\n3 replicas]
            API[API Service\n5 replicas]
            WORKER[Worker\n2 replicas]
        end
        
        subgraph Data ["Data Layer"]
            PG[(PostgreSQL\nStatefulSet)]
            REDIS[(Redis\nCluster)]
            KAFKA[Kafka\nCluster]
        end
        
        subgraph Platform ["Platform Services"]
            PROM[Prometheus]
            GRAF[Grafana]
            ARGO[ArgoCD]
            CERT[cert-manager]
        end
    end
    
    Users --> IG
    IG --> FE
    IG --> API
    API --> PG
    API --> REDIS
    API --> KAFKA
    KAFKA --> WORKER
    PROM --> API
    PROM --> FE
```

### Step 1: Cluster Setup

```bash
# Create EKS cluster with eksctl
eksctl create cluster \
  --name production \
  --region us-east-1 \
  --version 1.28 \
  --nodegroup-name workers \
  --node-type t3.large \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 10 \
  --managed \
  --with-oidc \
  --ssh-access \
  --ssh-public-key my-key

# Or with Terraform (recommended for production):
# See Phase 8 IaC for full Terraform EKS setup

# Verify cluster
kubectl get nodes
kubectl cluster-info
```

### Step 2: Platform Services Installation

```bash
# Install Nginx Ingress Controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.metrics.enabled=true

# Install cert-manager (automatic TLS certificates)
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true

# Install Prometheus + Grafana (monitoring stack)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=secure-password \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi

# Install ArgoCD (GitOps)
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  --set server.ingress.enabled=true \
  --set server.ingress.hosts[0]=argocd.example.com
```

### Step 3: Namespace and RBAC Setup

```yaml
# k8s/base/namespace.yml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    istio-injection: enabled

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services: "20"
    persistentvolumeclaims: "20"

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Step 4: Application Deployment

```yaml
# k8s/apps/api/deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: production
  labels:
    app: api-service
    tier: backend
spec:
  replicas: 5
  selector:
    matchLabels:
      app: api-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 2
  template:
    metadata:
      labels:
        app: api-service
        tier: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: api-service
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: api-service
      containers:
      - name: api
        image: myregistry.io/api-service:v1.5.0
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: database-url
        - name: REDIS_URL
          value: "redis://redis-cluster:6379"
        - name: KAFKA_BROKERS
          value: "kafka-0.kafka:9092,kafka-1.kafka:9092,kafka-2.kafka:9092"
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: api-config
              key: log-level
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 15
          failureThreshold: 3
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
      volumes:
      - name: tmp
        emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  selector:
    app: api-service
  ports:
  - port: 8080
    targetPort: 8080

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-service
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 5
  maxReplicas: 30
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 25
        periodSeconds: 60
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "500"

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-service
  namespace: production
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: api-service
```

### Step 5: Database (StatefulSet)

```yaml
# k8s/apps/postgres/statefulset.yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: production
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "$(POSTGRES_USER)"]
          periodSeconds: 10
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "$(POSTGRES_USER)"]
          periodSeconds: 30
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 100Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
  clusterIP: None   # Headless for StatefulSet
```

### Step 6: Monitoring — ServiceMonitor + Alerts

```yaml
# k8s/monitoring/servicemonitor.yml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-service
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: api-service
  namespaceSelector:
    matchNames:
    - production
  endpoints:
  - port: http
    path: /metrics
    interval: 15s

---
# k8s/monitoring/alerts.yml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-alerts
  namespace: monitoring
spec:
  groups:
  - name: api-service
    rules:
    - alert: HighErrorRate
      expr: |
        sum(rate(http_requests_total{job="api-service",status=~"5.."}[5m]))
        /
        sum(rate(http_requests_total{job="api-service"}[5m])) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate on API service"
        description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"

    - alert: HighLatency
      expr: |
        histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="api-service"}[5m])) by (le))
        > 2
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "P99 latency above 2 seconds"

    - alert: PodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total{namespace="production"}[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash-looping"
```

---

## Project 4: Blue-Green Deployment

### Overview

Blue-green deployment maintains two identical production environments. At any time, only one (Blue or Green) serves live traffic. Deployment = switch traffic from one to the other.

```mermaid
flowchart TB
    LB[Load Balancer / Ingress]
    
    subgraph Blue ["Blue Environment (LIVE)"]
        B1[Pod v1.0]
        B2[Pod v1.0]
        B3[Pod v1.0]
    end
    
    subgraph Green ["Green Environment (IDLE)"]
        G1[Pod v1.1]
        G2[Pod v1.1]
        G3[Pod v1.1]
    end
    
    LB -->|100% traffic| Blue
    LB -.->|0% traffic\nTest internally| Green
    
    style Blue fill:#4CAF50,color:#fff
    style Green fill:#9E9E9E,color:#fff
```

**After switch:**
```mermaid
flowchart TB
    LB[Load Balancer / Ingress]
    
    subgraph Blue ["Blue Environment (IDLE / Rollback)"]
        B1[Pod v1.0]
        B2[Pod v1.0]
        B3[Pod v1.0]
    end
    
    subgraph Green ["Green Environment (LIVE)"]
        G1[Pod v1.1]
        G2[Pod v1.1]
        G3[Pod v1.1]
    end
    
    LB -.->|0%| Blue
    LB -->|100% traffic| Green
    
    style Blue fill:#9E9E9E,color:#fff
    style Green fill:#4CAF50,color:#fff
```

### Kubernetes Blue-Green Implementation

```yaml
# blue-green/blue-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  namespace: production
  labels:
    app: myapp
    slot: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      slot: blue
  template:
    metadata:
      labels:
        app: myapp
        slot: blue
        version: v1.0.0
    spec:
      containers:
      - name: myapp
        image: myregistry.io/myapp:v1.0.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5

---
# blue-green/green-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  namespace: production
  labels:
    app: myapp
    slot: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      slot: green
  template:
    metadata:
      labels:
        app: myapp
        slot: green
        version: v1.1.0
    spec:
      containers:
      - name: myapp
        image: myregistry.io/myapp:v1.1.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5

---
# Service points to ACTIVE slot (blue or green)
# blue-green/service.yml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    app: myapp
    slot: blue      # ← Switch this to "green" for cutover
  ports:
  - port: 8080
    targetPort: 8080
```

### Blue-Green Deployment Script

```bash
#!/bin/bash
# scripts/blue-green-deploy.sh

set -euo pipefail

NEW_VERSION=$1
NAMESPACE="production"
APP="myapp"

# Determine current active slot
CURRENT_SLOT=$(kubectl get svc $APP -n $NAMESPACE -o jsonpath='{.spec.selector.slot}')

if [ "$CURRENT_SLOT" == "blue" ]; then
    NEW_SLOT="green"
else
    NEW_SLOT="blue"
fi

echo "Current active: $CURRENT_SLOT"
echo "Deploying to: $NEW_SLOT"
echo "New version: $NEW_VERSION"

# Step 1: Deploy new version to inactive slot
echo ">>> Deploying $NEW_VERSION to $NEW_SLOT..."
kubectl set image deployment/${APP}-${NEW_SLOT} \
    ${APP}=myregistry.io/${APP}:${NEW_VERSION} \
    -n $NAMESPACE

# Step 2: Wait for rollout to complete
echo ">>> Waiting for rollout..."
kubectl rollout status deployment/${APP}-${NEW_SLOT} -n $NAMESPACE --timeout=300s

# Step 3: Run smoke tests against new slot
echo ">>> Running smoke tests on $NEW_SLOT..."
NEW_POD=$(kubectl get pod -n $NAMESPACE -l "app=$APP,slot=$NEW_SLOT" -o jsonpath='{.items[0].metadata.name}')
kubectl exec $NEW_POD -n $NAMESPACE -- wget -qO- http://localhost:8080/health

# Step 4: Switch traffic
echo ">>> Switching traffic from $CURRENT_SLOT to $NEW_SLOT..."
kubectl patch svc $APP -n $NAMESPACE \
    -p "{\"spec\":{\"selector\":{\"app\":\"$APP\",\"slot\":\"$NEW_SLOT\"}}}"

echo ">>> Traffic switched! $NEW_SLOT is now live."
echo ">>> Previous version ($CURRENT_SLOT) kept for rollback."
echo ""
echo "To rollback, run:"
echo "  kubectl patch svc $APP -n $NAMESPACE -p '{\"spec\":{\"selector\":{\"slot\":\"$CURRENT_SLOT\"}}}'"
```

### AWS ALB Blue-Green (Terraform)

```hcl
# Two target groups — one for each environment
resource "aws_lb_target_group" "blue" {
  name     = "myapp-blue"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 15
  }
}

resource "aws_lb_target_group" "green" {
  name     = "myapp-green"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 15
  }
}

# Listener with weighted routing
resource "aws_lb_listener_rule" "app" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  action {
    type = "forward"
    forward {
      target_group {
        arn    = aws_lb_target_group.blue.arn
        weight = 100   # All traffic to blue
      }
      target_group {
        arn    = aws_lb_target_group.green.arn
        weight = 0     # No traffic to green
      }
    }
  }

  condition {
    path_pattern {
      values = ["/*"]
    }
  }
}

# To switch: change weights to blue=0, green=100
# To canary: use blue=90, green=10
```

---

## Project 5: Canary Deployment

### Overview

Canary deployment gradually shifts traffic from the old version to the new version. If errors are detected, traffic is automatically rolled back.

```mermaid
flowchart TB
    subgraph Phase1 ["Phase 1: 5% canary"]
        A1[95% → v1.0] 
        A2[5% → v1.1]
    end
    
    subgraph Phase2 ["Phase 2: 25% canary"]
        B1[75% → v1.0]
        B2[25% → v1.1]
    end
    
    subgraph Phase3 ["Phase 3: 50% canary"]
        C1[50% → v1.0]
        C2[50% → v1.1]
    end
    
    subgraph Phase4 ["Phase 4: Full promotion"]
        D1[0% → v1.0]
        D2[100% → v1.1]
    end
    
    Phase1 -->|Metrics OK\n5 min| Phase2
    Phase2 -->|Metrics OK\n5 min| Phase3
    Phase3 -->|Metrics OK\n10 min| Phase4
```

### Argo Rollouts — Canary with Analysis

```bash
# Install Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install kubectl plugin
brew install argoproj/tap/kubectl-argo-rollouts
# or
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

```yaml
# canary/rollout.yml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-service
  namespace: production
spec:
  replicas: 10
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      containers:
      - name: api
        image: myregistry.io/api-service:v1.1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          periodSeconds: 5
  strategy:
    canary:
      # Traffic splitting steps
      steps:
      - setWeight: 5
      - pause: {duration: 5m}        # Wait 5 min at 5%
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 25
      - pause: {duration: 5m}        # Wait 5 min at 25%
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 50
      - pause: {duration: 10m}       # Wait 10 min at 50%
      - analysis:
          templates:
          - templateName: success-rate
          - templateName: latency-check
      - setWeight: 100               # Full promotion
      
      # Automatic rollback on failure
      abortScaleDownDelaySeconds: 30
      
      # Anti-affinity between canary and stable
      antiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution: {}
      
      # Canary service (for testing canary directly)
      canaryService: api-service-canary
      stableService: api-service-stable
      
      trafficRouting:
        nginx:
          stableIngress: api-service-ingress
          additionalIngressAnnotations:
            canary-by-header: X-Canary

---
# Services for stable and canary
apiVersion: v1
kind: Service
metadata:
  name: api-service-stable
  namespace: production
spec:
  selector:
    app: api-service
  ports:
  - port: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: api-service-canary
  namespace: production
spec:
  selector:
    app: api-service
  ports:
  - port: 8080
```

### Analysis Templates (Automated Rollback Decisions)

```yaml
# canary/analysis-success-rate.yml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  args:
  - name: service-name
    value: api-service-canary
  metrics:
  - name: success-rate
    # Query Prometheus for error rate of canary pods
    provider:
      prometheus:
        address: http://prometheus.monitoring:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[5m]))
          /
          sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
    # Success rate must be >= 99%
    successCondition: result[0] >= 0.99
    failureCondition: result[0] < 0.95
    failureLimit: 3
    interval: 60s
    count: 5

---
# canary/analysis-latency.yml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency-check
  namespace: production
spec:
  metrics:
  - name: p99-latency
    provider:
      prometheus:
        address: http://prometheus.monitoring:9090
        query: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket{service="api-service-canary"}[5m]))
            by (le)
          )
    # P99 must be under 500ms
    successCondition: result[0] < 0.5
    failureCondition: result[0] >= 1.0
    failureLimit: 3
    interval: 60s
    count: 5
```

### Canary Operations

```bash
# Watch rollout progress
kubectl argo rollouts get rollout api-service -n production --watch

# Manually promote (skip waiting)
kubectl argo rollouts promote api-service -n production

# Abort and rollback
kubectl argo rollouts abort api-service -n production

# Retry after abort
kubectl argo rollouts retry rollout api-service -n production

# View rollout history
kubectl argo rollouts history api-service -n production

# Dashboard UI
kubectl argo rollouts dashboard
```

**Rollout visualization:**
```
NAME                           KIND        STATUS     AGE
⟳ api-service                  Rollout     ✔ Healthy  5d
├──# revision:3                                       
│  └──⧉ api-service-6b5d8f4c   ReplicaSet  ✔ Healthy  2h
│     ├──□ api-service-6b5d8f4c-abc12  Pod  ✔ Running  2h
│     ├──□ api-service-6b5d8f4c-def34  Pod  ✔ Running  2h
│     └──□ api-service-6b5d8f4c-ghi56  Pod  ✔ Running  2h
├──# revision:2                                       
│  └──⧉ api-service-7c6e9f5d   ReplicaSet  • ScaledDown 1d
└──# revision:1
   └──⧉ api-service-8d7f0g6e   ReplicaSet  • ScaledDown 3d
```

---

## Project 6: GitOps Pipeline with ArgoCD

### Overview

GitOps means: **Git is the single source of truth for your infrastructure and applications.** ArgoCD watches a git repository and automatically synchronizes the cluster to match.

```mermaid
flowchart LR
    A[Developer] -->|Push code| B[App Repo\nGitHub]
    B -->|CI builds image| C[Container\nRegistry]
    C -->|Update image tag| D[Config Repo\nGitHub]
    D -->|ArgoCD watches| E[ArgoCD]
    E -->|Syncs| F[Kubernetes\nCluster]
    F -->|Drift detected| E
```

### ArgoCD Application Definition

```yaml
# argocd/application.yml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-apps
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-config.git
    targetRevision: main
    path: environments/production
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Delete resources removed from git
      selfHeal: true     # Revert manual cluster changes
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

---
# ArgoCD ApplicationSet — deploy to multiple environments from one template
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: api-service
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: staging
        namespace: staging
        revision: develop
      - cluster: production
        namespace: production
        revision: main
  template:
    metadata:
      name: 'api-service-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/k8s-config.git
        targetRevision: '{{revision}}'
        path: 'apps/api-service/overlays/{{cluster}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          selfHeal: true
```

### Repository Structure (Kustomize)

```
k8s-config/
├── apps/
│   ├── api-service/
│   │   ├── base/
│   │   │   ├── kustomization.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── hpa.yaml
│   │   └── overlays/
│   │       ├── staging/
│   │       │   ├── kustomization.yaml
│   │       │   └── patch-replicas.yaml
│   │       └── production/
│   │           ├── kustomization.yaml
│   │           ├── patch-replicas.yaml
│   │           └── patch-resources.yaml
│   └── frontend/
│       ├── base/
│       └── overlays/
└── environments/
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

```yaml
# apps/api-service/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
- hpa.yaml

# apps/api-service/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
patches:
- path: patch-replicas.yaml
- path: patch-resources.yaml
images:
- name: myregistry.io/api-service
  newTag: v1.5.0    # ← CI updates this via PR

# apps/api-service/overlays/production/patch-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 5
```

### CI Pipeline That Updates GitOps Repo

```yaml
# .github/workflows/ci.yml (in the APPLICATION repo)
name: Build and Update Config

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.tag.outputs.tag }}
    steps:
    - uses: actions/checkout@v4

    - name: Generate tag
      id: tag
      run: echo "tag=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

    - name: Build and push image
      run: |
        docker build -t myregistry.io/api-service:${{ steps.tag.outputs.tag }} .
        docker push myregistry.io/api-service:${{ steps.tag.outputs.tag }}

  update-config:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - name: Checkout config repo
      uses: actions/checkout@v4
      with:
        repository: myorg/k8s-config
        token: ${{ secrets.CONFIG_REPO_TOKEN }}

    - name: Update image tag
      run: |
        cd apps/api-service/overlays/production
        kustomize edit set image myregistry.io/api-service:${{ needs.build.outputs.image-tag }}

    - name: Commit and push
      run: |
        git config user.name "CI Bot"
        git config user.email "ci@example.com"
        git add .
        git commit -m "deploy: api-service ${{ needs.build.outputs.image-tag }}"
        git push
    # ArgoCD detects the change and syncs automatically!
```

### ArgoCD Operations

```bash
# Login to ArgoCD
argocd login argocd.example.com --username admin --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)

# List applications
argocd app list

# Sync an application manually
argocd app sync production-apps

# View application status
argocd app get production-apps

# View diff between git and cluster
argocd app diff production-apps

# Rollback to previous revision
argocd app rollback production-apps 3

# View application history
argocd app history production-apps
```

---

## Interview Mastery

---

### Beginner Questions

---

**Q: What is CI/CD and why is it important?**

**Perfect Answer:**
> "CI/CD stands for Continuous Integration and Continuous Deployment. CI means every code change is automatically built and tested when pushed — catching bugs within minutes instead of days. CD means the code, once tested, is automatically deployed to production without manual intervention.
>
> It's important because it removes human error from repetitive processes, provides fast feedback to developers (you know within 10 minutes if your code broke something), and enables teams to deploy multiple times per day instead of once a month.
>
> A typical pipeline I'd build: push to GitHub → run unit tests → build Docker image → scan for vulnerabilities → deploy to staging → run integration tests → deploy to production with canary. If any step fails, the pipeline stops and the developer is notified immediately."

---

**Q: What is the difference between blue-green and canary deployments?**

**Perfect Answer:**
> "Blue-green deployment maintains two identical environments. All traffic goes to one (blue). You deploy to the other (green), test it, then switch ALL traffic at once. Rollback is instant — just switch back. Downside: you need 2x the infrastructure.
>
> Canary deployment gradually shifts traffic: first 5% goes to the new version, then 25%, then 50%, then 100%. At each step, you check metrics (error rate, latency). If something is wrong, you roll back only the canary pods — 95% of users were never affected.
>
> I prefer canary for large-scale production systems because:
> - You can catch issues that only appear under real traffic at scale
> - The blast radius of a bad deploy is limited (only 5% of users affected initially)
> - Combined with automated analysis (Argo Rollouts), it can auto-rollback without human intervention
>
> Blue-green is better when you need atomic switches (database schema changes that aren't backwards-compatible) or when traffic splitting isn't practical."

---

**Q: What is GitOps? How does it differ from traditional CI/CD?**

**Perfect Answer:**
> "GitOps means Git is the single source of truth for both application code AND infrastructure state. Instead of CI/CD pipelines pushing changes to production, a controller (ArgoCD, Flux) pulls the desired state from Git and reconciles the cluster to match.
>
> Traditional CI/CD: Developer pushes code → Pipeline builds → Pipeline deploys to cluster (push model)
>
> GitOps: Developer pushes code → CI builds image → CI updates config repo → ArgoCD detects change → ArgoCD syncs cluster (pull model)
>
> Key benefits:
> - **Audit trail:** Every change to production is a git commit — you know who changed what, when, and why
> - **Self-healing:** If someone manually changes the cluster (kubectl edit), ArgoCD reverts it to match git
> - **Rollback:** Rolling back = git revert + ArgoCD syncs
> - **Security:** CI pipeline doesn't need cluster credentials — only ArgoCD needs access
> - **Consistency:** Staging and production are guaranteed to match their respective git branches"

---

### Intermediate Questions

---

**Q: Walk me through how you'd set up a complete CI/CD pipeline for a microservices application.**

**Perfect Answer:**
> "I'd design a multi-stage pipeline with clear quality gates:
>
> **Stage 1: Code Quality (2 minutes)**
> - Lint and format check
> - SAST scanning (Semgrep)
> - Secret detection (Gitleaks)
> - Unit tests with coverage check (fail if < 80%)
>
> **Stage 2: Build (3 minutes)**
> - Docker multi-stage build
> - Push to private container registry (ECR/GHCR)
> - Tag with git SHA (immutable, traceable)
>
> **Stage 3: Security (2 minutes, parallel)**
> - Container image scan (Trivy) — fail on CRITICAL
> - Dependency scan (Snyk)
> - IaC scan if Terraform/K8s manifests changed (Checkov)
>
> **Stage 4: Integration Tests (5 minutes)**
> - Deploy to ephemeral test environment
> - Run API tests against real dependencies
> - Tear down environment
>
> **Stage 5: Deploy to Staging (auto)**
> - GitOps: update staging overlay in config repo
> - ArgoCD syncs to staging cluster
> - Run smoke tests
>
> **Stage 6: Deploy to Production (auto with canary)**
> - Update production overlay
> - Argo Rollouts: 5% → 25% → 50% → 100% with analysis
> - Automatic rollback if error rate > 1% or latency P99 > 500ms
>
> **Infrastructure:** GitHub Actions for CI, ArgoCD for CD, Argo Rollouts for progressive delivery. Config and app code in separate repos. Environments managed via Kustomize overlays."

---

**Q: How would you implement zero-downtime deployments in Kubernetes?**

**Perfect Answer:**
> "Zero-downtime requires careful coordination across the deployment, service, and pod lifecycle:
>
> **1. Rolling update strategy:**
> ```yaml
> strategy:
>   type: RollingUpdate
>   rollingUpdate:
>     maxUnavailable: 0  # Never reduce below desired count
>     maxSurge: 1        # Add 1 extra pod during rollout
> ```
>
> **2. Readiness probes:**
> New pods only receive traffic after passing readiness checks. Set `initialDelaySeconds` high enough for the app to fully start (JVM warmup, cache loading, etc.).
>
> **3. Graceful shutdown:**
> When a pod is terminated, it's removed from Service endpoints. But in-flight requests need to finish. I use a `preStop` hook with sleep to give the load balancer time to drain:
> ```yaml
> lifecycle:
>   preStop:
>     exec:
>       command: ['sh', '-c', 'sleep 15']
> ```
> Combined with `terminationGracePeriodSeconds: 30` — the app gets 15 seconds of drain time.
>
> **4. Pod Disruption Budget:**
> ```yaml
> minAvailable: 2
> ```
> Prevents Kubernetes from evicting too many pods during node maintenance.
>
> **5. Connection draining:**
> The application must handle SIGTERM gracefully — stop accepting new requests, finish in-flight ones, then exit.
>
> **6. Health check accuracy:**
> Readiness probe should check actual dependencies (DB connected, cache warmed) not just 'process is running'. This prevents routing traffic to a pod that's technically alive but can't serve requests."

---

### Advanced Questions

---

**Q: Design a deployment strategy for a globally distributed application with strict latency requirements (< 100ms P99).**

**Perfect Answer:**
> "Sub-100ms P99 globally requires computation at the edge — you can't achieve this with a centralized deployment due to speed-of-light latency across continents.
>
> **Architecture:**
>
> ```
> Multi-Region Active-Active:
> ┌─────────┐  ┌─────────┐  ┌─────────┐
> │ US-East │  │ EU-West │  │ AP-SE   │
> │ Cluster │  │ Cluster │  │ Cluster │
> └─────────┘  └─────────┘  └─────────┘
>      ↑             ↑            ↑
>      └─────── GeoDNS ───────────┘
>              (latency-based routing)
> ```
>
> **Deployment strategy: Regional canary**
>
> 1. Deploy to one region first (US-East canary)
> 2. Monitor for 30 minutes: error rate, latency, correctness
> 3. If healthy → deploy to EU-West
> 4. Monitor 30 minutes
> 5. If healthy → deploy to remaining regions
> 6. If any region shows problems → halt and investigate
>
> **Implementation with Argo Rollouts + ApplicationSets:**
> - Each region has its own ArgoCD Application
> - A controller manages the promotion order
> - Each region uses local canary (5% → 25% → 100%) before the next region starts
>
> **Latency considerations:**
> - Database: Each region has a local read replica. Writes go to primary region.
> - Cache: Regional Redis clusters (not cross-region — too slow)
> - Compute: Region-local pods only. No cross-region service calls in the hot path.
> - CDN: Static assets and cacheable API responses at edge PoPs
>
> **Rollback:**
> - Per-region rollback: if US-East has issues, roll back US-East only
> - Global rollback: revert all regions to previous version via git revert
> - DNS-level: remove unhealthy region from rotation (< 30 second failover)
>
> **Monitoring per region:**
> - Latency P99 per region (alert if any region > 80ms — approaching budget)
> - Cross-region replication lag (alert if > 100ms)
> - Regional error rates independently tracked"

---

**Q: Your canary deployment detected a 2% error rate increase on the new version, but only for users in a specific country. How do you investigate and what do you do?**

**Perfect Answer:**
> "This is a nuanced failure — not a global issue, so simple rollback might be overkill while a blanket promotion would hurt those users. Here's my approach:
>
> **Immediate action (don't promote further):**
> - Pause the canary rollout (`kubectl argo rollouts pause api-service`)
> - The 5% of users hitting the canary includes these affected country's users
>
> **Investigation (15-30 minutes):**
>
> 1. **Identify the pattern:**
>    - Which country? Check geo distribution in access logs
>    - Are errors 4xx or 5xx? (client-side vs server-side issue)
>    - Which endpoints are failing?
>
> 2. **Common causes for geo-specific failures:**
>    - Character encoding issues (non-ASCII names, addresses)
>    - Date/time format differences
>    - Currency or locale-specific logic change
>    - Third-party API unavailable in that region (payment gateway, geo service)
>    - CDN or DNS propagation issue specific to region
>
> 3. **Compare old vs new code:**
>    ```bash
>    git diff v1.0.0..v1.1.0 -- src/
>    # Look for changes involving: locale, i18n, dates, addresses, currency
>    ```
>
> 4. **Reproduce:**
>    - Send test requests to canary with that country's typical data
>    - Check canary pod logs with country-specific filters
>
> **Decision tree:**
> - **If root cause identified and fixable quickly (< 1 hour):**
>   - Fix the bug, build new image, deploy as new canary
>   - Verify fix resolves the country-specific issue
>   - Continue promotion
>
> - **If root cause unclear or fix is complex:**
>   - Abort canary rollout (rollback to stable)
>   - Open incident ticket with findings
>   - Fix in next sprint, add regression test for that country's data patterns
>
> **Prevention for future:**
> - Add locale-diverse test data to integration tests
> - Include geo-based analysis in canary AnalysisTemplates (error rate per region, not just global)
> - Synthetic monitoring from multiple countries"

---

### Scenario-Based Questions

---

**Q: You just deployed a new version and monitoring shows a gradual increase in memory usage on new pods. What's your plan?**

**Perfect Answer:**
> "Gradual memory increase = memory leak. It won't crash immediately but will eventually OOM-kill. My response:
>
> **Immediate (assess urgency):**
> - How fast is it growing? If 1MB/minute and limit is 512MB → I have ~8 hours
> - If growing fast → rollback immediately, investigate later
> - If growing slowly → I have time to diagnose while the canary is still running
>
> **If using canary deployment:**
> - Pause promotion (don't send more traffic to leaking pods)
> - Keep the canary running for observation but don't promote
>
> **Diagnosis:**
> ```bash
> # Watch memory growth in real-time
> kubectl top pods -l app=api-service --sort-by=memory -w
>
> # Get detailed memory info
> kubectl exec <pod> -- cat /proc/meminfo
>
> # If the app has a profiler endpoint:
> curl http://<pod-ip>:8080/debug/pprof/heap > heap.prof
>
> # Check: is it the app or a sidecar?
> kubectl exec <pod> -c api -- ps aux --sort=-%mem
> ```
>
> **Common causes in containerized apps:**
> - Connection pool not releasing connections (DB, HTTP client)
> - Unbounded in-memory cache (no eviction policy)
> - Event listeners registered but never removed
> - Goroutine/thread leak (creating workers without cleanup)
> - Large response bodies buffered in memory
>
> **Resolution:**
> 1. Rollback the canary to stable version
> 2. Compare the code diff: what changed that touches connections, caches, or long-lived objects?
> 3. Reproduce locally with load test + memory profiler
> 4. Fix the leak, add a test that monitors memory under sustained load
> 5. Redeploy as canary with close memory monitoring
>
> **Prevention:**
> - Add memory growth rate as a canary analysis metric
> - Set memory limits so OOM kills happen before the node is affected
> - Run periodic load tests that check for memory stability over time"

---

**Q: Your ArgoCD shows 'OutOfSync' for production, but no one made any changes to the git repo. What happened?**

**Perfect Answer:**
> "OutOfSync means the cluster state differs from what's in git. If no one changed git, someone (or something) changed the cluster directly. My investigation:
>
> **Step 1: What's different?**
> ```bash
> argocd app diff production-apps
> # Shows the exact diff between git (desired) and cluster (actual)
> ```
>
> **Step 2: Identify what changed and who:**
> ```bash
> # Check Kubernetes audit logs
> kubectl get events -n production --sort-by=.lastTimestamp
>
> # If you have audit logging enabled:
> # Look for non-ArgoCD actors modifying resources
> ```
>
> **Common causes:**
>
> 1. **HPA scaled replicas:** HPA changes replica count, which differs from the static count in git. 
>    Fix: Don't specify `replicas` in git when using HPA, or use ArgoCD's `ignoreDifferences`:
>    ```yaml
>    spec:
>      ignoreDifferences:
>      - group: apps
>        kind: Deployment
>        jsonPointers:
>        - /spec/replicas
>    ```
>
> 2. **Mutating webhooks:** Admission controllers (Istio sidecar injection, Vault agent injection) modify pod specs after ArgoCD applies them.
>    Fix: Add `ignoreDifferences` for injected fields.
>
> 3. **Manual kubectl change:** Someone ran `kubectl edit` or `kubectl scale`.
>    Fix: If selfHeal is enabled, ArgoCD will revert it automatically. Educate team that manual changes will be overwritten.
>
> 4. **CRD defaults:** Kubernetes adds default values to resources that weren't specified in git.
>    Fix: Explicitly set all fields, or use `ignoreDifferences`.
>
> 5. **Secrets updated externally:** External Secrets Operator updated a Secret that ArgoCD also manages.
>    Fix: Exclude externally-managed secrets from ArgoCD tracking.
>
> **Best practice:** Enable `selfHeal: true` in ArgoCD sync policy so drift is automatically corrected. Combined with `prune: true`, the cluster always matches git — any manual change is reverted within 3 minutes."

---

### FAANG-Style Question

---

**Q: Design the deployment system for a company with 200 microservices, 50 engineers, and 100+ deploys per day.**

**Perfect Answer:**
> "At this scale, deployment must be fully self-service with strong guardrails. Engineers shouldn't need DevOps approval for every deploy.
>
> **Architecture:**
>
> ```
> Developer Experience:
> PR merged → CI builds → image pushed → config repo updated → ArgoCD syncs
> (entire flow: < 15 minutes from merge to production)
> ```
>
> **Platform components:**
>
> 1. **Standardized CI templates (GitHub Actions reusable workflows):**
>    - Every service uses the same pipeline skeleton
>    - Teams customize via `deploy.yaml` config file in their repo
>    - Enforces: testing, scanning, image signing
>    - No custom pipelines per team (reduces maintenance from 200 to 1)
>
> 2. **GitOps with ArgoCD ApplicationSets:**
>    - One ArgoCD ApplicationSet generates apps from a registry
>    - Each service has a directory in the config repo
>    - Adding a new service = adding a directory + registering in the set
>    - Auto-discovery: new services automatically get ArgoCD applications
>
> 3. **Progressive delivery (Argo Rollouts by default):**
>    - Every service gets canary deployment automatically
>    - Standard analysis templates: error rate, latency, saturation
>    - Teams can customize thresholds via annotations
>    - Automated rollback without human intervention
>
> 4. **Self-service developer portal (Backstage):**
>    - Create new service from template (scaffolding)
>    - View deployment status across environments
>    - Trigger manual rollbacks
>    - View dependencies and service mesh topology
>
> 5. **Environments:**
>    - PR environments: ephemeral namespace per PR (auto-deleted on merge)
>    - Staging: mirrors production config, auto-deployed on main merge
>    - Production: canary deployment, auto-promoted if analysis passes
>
> **Guardrails (prevent bad deploys without blocking engineers):**
> - Image signing required (Cosign) — unsigned images rejected by admission controller
> - Security scan must pass (CRITICAL = block, HIGH = allow with ticket)
> - Readiness/liveness probes mandatory (enforced by OPA)
> - Resource limits required (enforced by admission)
> - Canary analysis gates (automated, not manual approval)
>
> **Scale considerations:**
> - ArgoCD: multiple shards (one controller per 50 apps)
> - Config repo: monorepo with path-based ownership (CODEOWNERS)
> - 100 deploys/day = ~12 per hour = canary analysis must complete in < 30 min
> - Notification system: Slack webhook per team's channel on deploy start/success/failure
>
> **Metrics I'd track:**
> - Deployment frequency per team
> - Lead time (commit to production)
> - Change failure rate (% of deploys that cause incidents)
> - MTTR (time to recover from failed deploy)
> - These are the DORA metrics — track them to measure platform effectiveness."

---

## Summary

```
Real-world DevOps Projects — Key Takeaways:

Spring Boot / Flask Deployment:
  Multi-stage Dockerfile → CI pipeline → K8s manifests → Ingress
  Pattern: Build, scan, push, deploy, verify.

Kubernetes Production:
  Platform services (monitoring, ingress, cert-manager)
  Security (PSS, network policies, RBAC)
  Observability (ServiceMonitor, PrometheusRules)

Blue-Green Deployment:
  Two identical environments, instant switch.
  Best for: atomic changes, database migrations.
  Rollback: switch traffic back (instant).

Canary Deployment:
  Gradual traffic shift with automated analysis.
  Best for: large-scale, metrics-driven safety.
  Rollback: automatic on metric breach.

GitOps (ArgoCD):
  Git = source of truth. Pull model.
  Self-healing cluster. Full audit trail.
  Kustomize overlays for environments.

Key principle: Automate EVERYTHING.
  If a human must approve/click/type for a standard deploy,
  the system is not mature enough.
```

---

[⬇️ Download This File](#)
