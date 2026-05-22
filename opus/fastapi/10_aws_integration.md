# FastAPI + AWS Integration — Senior Level

## 1. S3 (File Storage)

### Upload & Download with Pre-signed URLs
```python
import boto3
from botocore.config import Config
from fastapi import APIRouter, UploadFile, File, Depends, HTTPException
from uuid import uuid4
from app.config import settings

router = APIRouter(prefix="/files", tags=["Files"])

s3_client = boto3.client(
    "s3",
    region_name=settings.aws_region,
    config=Config(signature_version="s3v4"),
)


@router.post("/upload")
async def upload_file(
    file: UploadFile = File(...),
    user=Depends(get_current_user),
):
    """Direct upload to S3."""
    # Validate file
    allowed_types = {"image/jpeg", "image/png", "application/pdf"}
    if file.content_type not in allowed_types:
        raise HTTPException(400, "File type not allowed")
    
    if file.size > 10 * 1024 * 1024:  # 10MB limit
        raise HTTPException(400, "File too large (max 10MB)")
    
    # Generate unique key
    ext = file.filename.split(".")[-1]
    key = f"uploads/{user.id}/{uuid4()}.{ext}"
    
    # Upload
    s3_client.upload_fileobj(
        file.file,
        settings.aws_s3_bucket,
        key,
        ExtraArgs={
            "ContentType": file.content_type,
            "Metadata": {"uploaded_by": str(user.id)},
        },
    )
    
    return {"key": key, "url": f"https://{settings.aws_s3_bucket}.s3.amazonaws.com/{key}"}


@router.get("/presigned-upload")
async def get_presigned_upload_url(
    filename: str,
    content_type: str,
    user=Depends(get_current_user),
):
    """Generate pre-signed URL for client-side upload (better for large files)."""
    ext = filename.split(".")[-1]
    key = f"uploads/{user.id}/{uuid4()}.{ext}"
    
    url = s3_client.generate_presigned_url(
        "put_object",
        Params={
            "Bucket": settings.aws_s3_bucket,
            "Key": key,
            "ContentType": content_type,
        },
        ExpiresIn=300,  # 5 minutes
    )
    
    return {"upload_url": url, "key": key}


@router.get("/presigned-download/{file_key:path}")
async def get_presigned_download_url(file_key: str):
    """Generate pre-signed URL for download (private files)."""
    url = s3_client.generate_presigned_url(
        "get_object",
        Params={"Bucket": settings.aws_s3_bucket, "Key": file_key},
        ExpiresIn=3600,  # 1 hour
    )
    return {"download_url": url}
```

### Async S3 with aioboto3
```python
import aioboto3
from contextlib import asynccontextmanager

session = aioboto3.Session()


@asynccontextmanager
async def get_s3():
    async with session.client("s3", region_name=settings.aws_region) as client:
        yield client


async def upload_async(file_data: bytes, key: str, content_type: str):
    async with get_s3() as s3:
        await s3.put_object(
            Bucket=settings.aws_s3_bucket,
            Key=key,
            Body=file_data,
            ContentType=content_type,
        )


async def download_async(key: str) -> bytes:
    async with get_s3() as s3:
        response = await s3.get_object(Bucket=settings.aws_s3_bucket, Key=key)
        return await response["Body"].read()
```

---

## 2. SQS (Message Queue)

### Producer: Enqueue Tasks
```python
import boto3
import json
from app.config import settings

sqs = boto3.client("sqs", region_name=settings.aws_region)
QUEUE_URL = settings.sqs_queue_url


async def enqueue_task(task_type: str, payload: dict):
    """Send message to SQS queue."""
    sqs.send_message(
        QueueUrl=QUEUE_URL,
        MessageBody=json.dumps({
            "task_type": task_type,
            "payload": payload,
            "timestamp": datetime.utcnow().isoformat(),
        }),
        MessageGroupId=task_type,  # FIFO queue
        MessageDeduplicationId=str(uuid4()),
    )


# Usage in endpoint
@router.post("/orders/{order_id}/process")
async def process_order(order_id: str, user=Depends(get_current_user)):
    order = await OrderService.get(order_id)
    
    # Enqueue for async processing
    await enqueue_task("process_payment", {
        "order_id": order_id,
        "amount": float(order.total),
        "user_id": str(user.id),
    })
    
    return {"status": "processing", "order_id": order_id}
```

### Consumer: Process Messages (Worker)
```python
# worker/sqs_consumer.py
import boto3
import json
import asyncio
from app.config import settings

sqs = boto3.client("sqs", region_name=settings.aws_region)


async def process_message(message: dict):
    """Route message to appropriate handler."""
    body = json.loads(message["Body"])
    task_type = body["task_type"]
    payload = body["payload"]
    
    handlers = {
        "process_payment": handle_payment,
        "send_email": handle_email,
        "generate_report": handle_report,
    }
    
    handler = handlers.get(task_type)
    if handler:
        await handler(payload)


async def poll_queue():
    """Long-poll SQS and process messages."""
    while True:
        response = sqs.receive_message(
            QueueUrl=settings.sqs_queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20,  # Long polling (reduce API calls)
            VisibilityTimeout=60,
        )
        
        messages = response.get("Messages", [])
        
        for message in messages:
            try:
                await process_message(message)
                # Delete on success
                sqs.delete_message(
                    QueueUrl=settings.sqs_queue_url,
                    ReceiptHandle=message["ReceiptHandle"],
                )
            except Exception as e:
                # Message becomes visible again after VisibilityTimeout
                print(f"Error processing message: {e}")
        
        if not messages:
            await asyncio.sleep(1)


if __name__ == "__main__":
    asyncio.run(poll_queue())
```

---

## 3. Secrets Manager

```python
import boto3
import json
from functools import lru_cache


@lru_cache()
def get_secret(secret_name: str) -> dict:
    """Retrieve secret from AWS Secrets Manager (cached)."""
    client = boto3.client("secretsmanager", region_name="ap-south-1")
    
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])


# Usage in config
class Settings(BaseSettings):
    environment: str = "production"
    
    @property
    def database_url(self) -> str:
        if self.environment == "production":
            secret = get_secret("prod/database/credentials")
            return (
                f"postgresql+asyncpg://{secret['username']}:{secret['password']}"
                f"@{secret['host']}:{secret['port']}/{secret['dbname']}"
            )
        return "postgresql+asyncpg://postgres:local@localhost/dev_db"
    
    @property
    def jwt_secret(self) -> str:
        if self.environment == "production":
            secret = get_secret("prod/app/jwt")
            return secret["secret_key"]
        return "dev-secret-key"


# Rotate secrets without restart
class SecretCache:
    def __init__(self, ttl: int = 300):
        self._cache = {}
        self._ttl = ttl
        self._timestamps = {}
    
    async def get(self, name: str) -> dict:
        import time
        now = time.time()
        
        if name in self._cache and (now - self._timestamps[name]) < self._ttl:
            return self._cache[name]
        
        secret = get_secret(name)
        self._cache[name] = secret
        self._timestamps[name] = now
        return secret
```

---

## 4. Lambda Integration (API Gateway → FastAPI on Lambda)

### Mangum Adapter (FastAPI → Lambda)
```python
# lambda_handler.py
from mangum import Mangum
from app.main import app

handler = Mangum(app, lifespan="off")

# Deploy: zip the code + dependencies → upload to Lambda
# API Gateway routes all requests to this Lambda
# Mangum converts API Gateway event → ASGI → FastAPI
```

### Invoke Lambda from FastAPI
```python
import boto3
import json

lambda_client = boto3.client("lambda", region_name="ap-south-1")


async def invoke_lambda(function_name: str, payload: dict, async_invoke: bool = False):
    """Invoke another Lambda function."""
    response = lambda_client.invoke(
        FunctionName=function_name,
        InvocationType="Event" if async_invoke else "RequestResponse",
        Payload=json.dumps(payload),
    )
    
    if not async_invoke:
        result = json.loads(response["Payload"].read())
        return result
    
    return {"status": "invoked"}


# Usage: trigger ML inference
@router.post("/predict")
async def predict(data: PredictionInput):
    result = await invoke_lambda(
        "ml-inference-function",
        {"features": data.features, "model_version": "v2"},
    )
    return {"prediction": result["prediction"], "confidence": result["confidence"]}
```

---

## 5. DynamoDB (NoSQL)

```python
import boto3
from boto3.dynamodb.conditions import Key, Attr
from decimal import Decimal

dynamodb = boto3.resource("dynamodb", region_name="ap-south-1")
table = dynamodb.Table("user_sessions")


class SessionStore:
    """DynamoDB-backed session store for WebSocket state."""
    
    async def create_session(self, user_id: str, connection_id: str, metadata: dict):
        table.put_item(Item={
            "user_id": user_id,
            "connection_id": connection_id,
            "connected_at": datetime.utcnow().isoformat(),
            "metadata": metadata,
            "ttl": int((datetime.utcnow() + timedelta(hours=24)).timestamp()),
        })
    
    async def get_user_sessions(self, user_id: str) -> list:
        response = table.query(
            KeyConditionExpression=Key("user_id").eq(user_id)
        )
        return response["Items"]
    
    async def delete_session(self, user_id: str, connection_id: str):
        table.delete_item(Key={
            "user_id": user_id,
            "connection_id": connection_id,
        })


# API events (high-write, low-read → perfect for DynamoDB)
events_table = dynamodb.Table("api_events")


async def log_api_event(event_type: str, user_id: str, data: dict):
    events_table.put_item(Item={
        "event_id": str(uuid4()),
        "user_id": user_id,
        "event_type": event_type,
        "data": json.loads(json.dumps(data), parse_float=Decimal),
        "timestamp": datetime.utcnow().isoformat(),
        "ttl": int((datetime.utcnow() + timedelta(days=30)).timestamp()),
    })
```

---

## 6. ECS Fargate Deployment

### Task Definition
```json
{
  "family": "fastapi-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456.dkr.ecr.ap-south-1.amazonaws.com/fastapi-app:latest",
      "portMappings": [{"containerPort": 8000, "protocol": "tcp"}],
      "environment": [
        {"name": "ENVIRONMENT", "value": "production"}
      ],
      "secrets": [
        {"name": "DATABASE_URL", "valueFrom": "arn:aws:secretsmanager:ap-south-1:123456:secret:prod/db"},
        {"name": "JWT_SECRET", "valueFrom": "arn:aws:secretsmanager:ap-south-1:123456:secret:prod/jwt"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/fastapi-app",
          "awslogs-region": "ap-south-1",
          "awslogs-stream-prefix": "api"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3
      }
    }
  ]
}
```

### Service with ALB
```
Architecture:
  Internet → ALB (port 443) → Target Group → ECS Tasks (port 8000)

ALB configuration:
  - HTTPS listener (ACM certificate)
  - Health check: GET /health (interval 30s)
  - Stickiness: disabled (stateless API)
  - Deregistration delay: 30s (allow inflight requests to complete)

Auto-scaling:
  - Target: CPU 70% average
  - Min: 2 tasks, Max: 10 tasks
  - Scale-in cooldown: 300s
  - Scale-out cooldown: 60s
```

---

## 7. CloudWatch + Structured Logging

```python
import structlog
import json
from datetime import datetime

# Configure structured logging (JSON format for CloudWatch)
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.add_log_level,
        structlog.processors.JSONRenderer(),
    ]
)
logger = structlog.get_logger()


# Middleware for request logging
@app.middleware("http")
async def cloudwatch_logging(request: Request, call_next):
    start = time.perf_counter()
    
    response = await call_next(request)
    
    duration = time.perf_counter() - start
    
    logger.info(
        "http_request",
        method=request.method,
        path=request.url.path,
        status_code=response.status_code,
        duration_ms=round(duration * 1000, 2),
        user_agent=request.headers.get("user-agent", ""),
        request_id=request.state.request_id,
    )
    
    return response


# Custom metrics to CloudWatch
import boto3

cloudwatch = boto3.client("cloudwatch", region_name="ap-south-1")


async def put_metric(name: str, value: float, unit: str = "Count"):
    cloudwatch.put_metric_data(
        Namespace="FastAPI/Production",
        MetricData=[{
            "MetricName": name,
            "Value": value,
            "Unit": unit,
            "Timestamp": datetime.utcnow(),
            "Dimensions": [
                {"Name": "Service", "Value": "api"},
                {"Name": "Environment", "Value": settings.environment},
            ],
        }],
    )


# Usage
@router.post("/orders/")
async def create_order(order: OrderCreate, user=Depends(get_current_user)):
    result = await OrderService.create(user, order)
    await put_metric("OrderCreated", 1)
    await put_metric("OrderValue", float(result.total), "None")
    return result
```

---

## 8. Interview Questions

### Q: How do you deploy FastAPI on AWS?
```
Option 1: ECS Fargate (recommended for most cases)
  Docker → ECR → ECS Fargate → ALB → Route53
  + No server management
  + Auto-scaling
  + Secrets Manager integration
  - Higher cost for low traffic

Option 2: Lambda + API Gateway (serverless)
  Docker/Zip → Lambda → API Gateway
  + Pay per request (cheap for low traffic)
  + Zero maintenance
  - Cold starts (1-3s)
  - 15 min timeout
  - No WebSocket (use API Gateway WebSocket separately)

Option 3: EC2 + Docker (full control)
  Docker → EC2 → ALB
  + Cheapest for steady high traffic
  + Full control
  - Must manage servers, patching, scaling

For FastAPI specifically:
  ECS Fargate for production APIs (most teams)
  Lambda for low-traffic internal tools
  EC2 for cost-sensitive high-traffic APIs
```

### Q: How do you handle file uploads in a distributed FastAPI app?
```
Never store files on local disk (containers are ephemeral).

Pattern 1: Direct upload to S3
  Client → FastAPI → S3
  Simple, works for small files (<10MB)

Pattern 2: Pre-signed URL (better)
  Client → FastAPI (get pre-signed URL) → Client → S3 (direct upload)
  + Doesn't pass through your server
  + No memory/bandwidth overhead on API
  + Works for large files (multi-GB)
  + Set content-type, size limits in pre-signed policy

Pattern 3: Multipart upload (very large files)
  Client → S3 (CreateMultipartUpload) → upload parts → CompleteMultipart
  + Handles 5TB files
  + Resume interrupted uploads
  + Parallel part uploads

Always:
  - Validate file type (check magic bytes, not just extension)
  - Set max size
  - Scan for malware (S3 event → Lambda → ClamAV)
  - Serve via CloudFront (not direct S3 URL)
```
