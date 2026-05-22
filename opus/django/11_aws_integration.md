# Django + AWS Integration — Senior Level

## 1. S3 (File Storage)

### Setup with django-storages
```python
# pip install django-storages boto3

# settings/production.py
AWS_ACCESS_KEY_ID = env('AWS_ACCESS_KEY_ID')  # or use IAM role (preferred)
AWS_SECRET_ACCESS_KEY = env('AWS_SECRET_ACCESS_KEY')
AWS_STORAGE_BUCKET_NAME = 'myapp-media-prod'
AWS_S3_REGION_NAME = 'ap-south-1'
AWS_S3_CUSTOM_DOMAIN = f'{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com'
AWS_S3_OBJECT_PARAMETERS = {
    'CacheControl': 'max-age=86400',  # 1 day browser cache
}
AWS_DEFAULT_ACL = None  # use bucket policy, not per-object ACL
AWS_S3_FILE_OVERWRITE = False  # don't overwrite same-name files
AWS_QUERYSTRING_AUTH = False  # public URLs (no signed query params)

# Media files → S3
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'

# Static files → S3 (or use WhiteNoise for simpler setup)
STATICFILES_STORAGE = 'storages.backends.s3boto3.S3StaticStorage'
STATIC_URL = f'https://{AWS_S3_CUSTOM_DOMAIN}/static/'
```

### Separate Storage Backends (Media vs Private)
```python
# storages.py
from storages.backends.s3boto3 import S3Boto3Storage

class PublicMediaStorage(S3Boto3Storage):
    location = 'media'
    file_overwrite = False

class PrivateMediaStorage(S3Boto3Storage):
    location = 'private'
    default_acl = 'private'
    file_overwrite = False
    custom_domain = False  # use signed URLs
    querystring_auth = True  # signed URLs with expiry
    querystring_expire = 3600  # 1 hour

# Usage in models
class UserProfile(models.Model):
    avatar = models.ImageField(
        upload_to='avatars/%Y/%m/',
        storage=PublicMediaStorage()
    )

class Contract(models.Model):
    document = models.FileField(
        upload_to='contracts/%Y/%m/',
        storage=PrivateMediaStorage()  # signed URLs, not public
    )
```

### Pre-Signed Upload URLs (Client Uploads Directly to S3)
```python
# views.py — generate pre-signed URL for frontend to upload directly
import boto3
import uuid
from rest_framework.views import APIView
from rest_framework.response import Response

class GenerateUploadURL(APIView):
    """Frontend gets a pre-signed URL, uploads directly to S3 (faster)."""
    
    def post(self, request):
        file_name = request.data.get('file_name')
        file_type = request.data.get('file_type')
        
        # Generate unique key
        ext = file_name.split('.')[-1]
        key = f"uploads/{request.user.id}/{uuid.uuid4()}.{ext}"
        
        s3_client = boto3.client('s3', region_name='ap-south-1')
        
        presigned_url = s3_client.generate_presigned_url(
            'put_object',
            Params={
                'Bucket': 'myapp-media-prod',
                'Key': key,
                'ContentType': file_type,
            },
            ExpiresIn=300  # URL valid for 5 minutes
        )
        
        return Response({
            'upload_url': presigned_url,
            'file_key': key,
            'file_url': f"https://myapp-media-prod.s3.amazonaws.com/{key}"
        })

# Frontend usage:
# 1. POST /api/upload-url/ → gets presigned_url
# 2. PUT presigned_url with file body → uploads to S3 directly
# 3. POST /api/products/ with file_url → save reference in DB
```

---

## 2. SES (Email)

### Django Email Backend with SES
```python
# pip install django-ses

# settings.py
EMAIL_BACKEND = 'django_ses.SESBackend'
AWS_SES_REGION_NAME = 'ap-south-1'
AWS_SES_REGION_ENDPOINT = 'email.ap-south-1.amazonaws.com'
DEFAULT_FROM_EMAIL = 'noreply@yourdomain.com'

# For high-volume: use SES with Celery
# tasks.py
from celery import shared_task
from django.core.mail import send_mail
from django.template.loader import render_to_string

@shared_task(bind=True, max_retries=3)
def send_order_email(self, order_id):
    try:
        order = Order.objects.select_related('user').get(id=order_id)
        
        html_content = render_to_string('emails/order_confirmed.html', {
            'order': order,
            'items': order.items.all(),
        })
        
        send_mail(
            subject=f'Order #{order.id} Confirmed',
            message='',  # plain text fallback
            html_message=html_content,
            from_email='orders@yourdomain.com',
            recipient_list=[order.user.email],
        )
    except Exception as exc:
        self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))


# Bulk email with SES (using boto3 directly for templates)
@shared_task
def send_bulk_promotion(user_ids, template_name):
    import boto3
    
    ses = boto3.client('ses', region_name='ap-south-1')
    users = User.objects.filter(id__in=user_ids).values_list('email', flat=True)
    
    # SES allows 50 destinations per call
    for chunk in chunked(list(users), 50):
        ses.send_bulk_templated_email(
            Source='marketing@yourdomain.com',
            Template=template_name,
            Destinations=[
                {'Destination': {'ToAddresses': [email]}}
                for email in chunk
            ],
            DefaultTemplateData='{"name": "Customer"}'
        )
```

---

## 3. SQS (Message Queue with Celery)

### Celery with SQS Backend
```python
# pip install celery[sqs]

# settings.py
CELERY_BROKER_URL = 'sqs://'
CELERY_BROKER_TRANSPORT_OPTIONS = {
    'region': 'ap-south-1',
    'visibility_timeout': 3600,  # 1 hour
    'polling_interval': 1,
    'queue_name_prefix': 'myapp-',
    'predefined_queues': {
        'celery': {
            'url': 'https://sqs.ap-south-1.amazonaws.com/123456789/myapp-default',
        },
        'high-priority': {
            'url': 'https://sqs.ap-south-1.amazonaws.com/123456789/myapp-high',
        },
    },
}

# Route tasks to specific queues
CELERY_TASK_ROUTES = {
    'apps.orders.tasks.process_payment': {'queue': 'high-priority'},
    'apps.analytics.tasks.*': {'queue': 'celery'},
}
```

### Direct SQS Integration (Without Celery)
```python
# For simple producer/consumer patterns
import boto3
import json

class SQSPublisher:
    def __init__(self, queue_url):
        self.sqs = boto3.client('sqs', region_name='ap-south-1')
        self.queue_url = queue_url
    
    def publish(self, message, delay_seconds=0):
        self.sqs.send_message(
            QueueUrl=self.queue_url,
            MessageBody=json.dumps(message),
            DelaySeconds=delay_seconds,
            MessageAttributes={
                'EventType': {
                    'StringValue': message.get('event_type', 'unknown'),
                    'DataType': 'String'
                }
            }
        )

# Usage: publish order events for other services
publisher = SQSPublisher('https://sqs.ap-south-1.amazonaws.com/123/order-events')

def on_order_created(order):
    publisher.publish({
        'event_type': 'order.created',
        'order_id': str(order.id),
        'user_id': order.user_id,
        'total': str(order.total),
        'timestamp': order.created_at.isoformat(),
    })
```

---

## 4. Lambda Integration

### Django Calling Lambda
```python
import boto3
import json

class LambdaClient:
    def __init__(self):
        self.client = boto3.client('lambda', region_name='ap-south-1')
    
    def invoke_sync(self, function_name, payload):
        """Invoke Lambda and wait for response."""
        response = self.client.invoke(
            FunctionName=function_name,
            InvocationType='RequestResponse',
            Payload=json.dumps(payload)
        )
        return json.loads(response['Payload'].read())
    
    def invoke_async(self, function_name, payload):
        """Fire-and-forget Lambda invocation."""
        self.client.invoke(
            FunctionName=function_name,
            InvocationType='Event',
            Payload=json.dumps(payload)
        )

# Usage: offload image processing to Lambda
lambda_client = LambdaClient()

@shared_task
def process_uploaded_image(image_key):
    result = lambda_client.invoke_sync(
        'image-processor',
        {
            'bucket': 'myapp-media-prod',
            'key': image_key,
            'operations': ['resize', 'thumbnail', 'compress']
        }
    )
    # result = {'thumbnails': ['thumb_sm.jpg', 'thumb_lg.jpg']}
    return result
```

### Django on Lambda (Serverless)
```python
# Using Zappa or Mangum for serverless Django

# mangum handler (for AWS Lambda + API Gateway)
# pip install mangum
from mangum import Mangum
from config.wsgi import application

handler = Mangum(application, lifespan='off')

# SAM template / serverless.yml deploys this as Lambda function
# API Gateway routes all requests to this handler
# Django handles routing internally
```

---

## 5. CloudWatch (Logging & Monitoring)

### Structured Logging to CloudWatch
```python
# settings/production.py
import watchtower

LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'json': {
            'class': 'pythonjsonlogger.jsonlogger.JsonFormatter',
            'format': '%(asctime)s %(levelname)s %(name)s %(message)s %(pathname)s %(lineno)d',
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'json',
        },
        'cloudwatch': {
            'class': 'watchtower.CloudWatchLogHandler',
            'log_group_name': 'myapp-django',
            'log_stream_name': '{machine_name}/{programname}/{logger_name}',
            'formatter': 'json',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['console', 'cloudwatch'],
            'level': 'INFO',
        },
        'apps': {
            'handlers': ['console', 'cloudwatch'],
            'level': 'INFO',
        },
    },
}
```

### Custom CloudWatch Metrics
```python
import boto3

cloudwatch = boto3.client('cloudwatch', region_name='ap-south-1')

def publish_metric(metric_name, value, unit='Count'):
    cloudwatch.put_metric_data(
        Namespace='MyApp',
        MetricData=[{
            'MetricName': metric_name,
            'Value': value,
            'Unit': unit,
            'Dimensions': [
                {'Name': 'Environment', 'Value': 'production'},
                {'Name': 'Service', 'Value': 'django-web'},
            ]
        }]
    )

# In services:
def create_order(user, items):
    order = OrderService.create_order(user, items)
    publish_metric('OrderCreated', 1)
    publish_metric('OrderTotal', float(order.total), unit='None')
    return order
```

---

## 6. Secrets Manager

```python
# settings/production.py
import boto3
import json

def get_secrets():
    client = boto3.client('secretsmanager', region_name='ap-south-1')
    response = client.get_secret_value(SecretId='prod/myapp/django')
    return json.loads(response['SecretString'])

secrets = get_secrets()

SECRET_KEY = secrets['DJANGO_SECRET_KEY']
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': secrets['DB_NAME'],
        'USER': secrets['DB_USER'],
        'PASSWORD': secrets['DB_PASSWORD'],
        'HOST': secrets['DB_HOST'],
        'PORT': '5432',
    }
}
```

---

## 7. ElastiCache (Redis on AWS)

```python
# settings/production.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': env('ELASTICACHE_URL'),
        # Example: 'redis://myapp-redis.abc123.0001.aps1.cache.amazonaws.com:6379/0'
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'CONNECTION_POOL_KWARGS': {
                'max_connections': 50,
                'retry_on_timeout': True,
            },
            'SOCKET_CONNECT_TIMEOUT': 5,
            'SOCKET_TIMEOUT': 5,
        }
    }
}

# Session storage on ElastiCache
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'

# Celery broker (can also use ElastiCache)
CELERY_BROKER_URL = env('ELASTICACHE_URL')
```

---

## 8. Full AWS Architecture for Django

```
┌─────────────────────────────────────────────────────────────────────┐
│                      AWS Production Setup                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   Route53 (DNS) → CloudFront (CDN) → ALB                            │
│                         ↕                 ↕                          │
│                    S3 (static)      ECS Fargate (Django)             │
│                                         ↕                            │
│                              ┌───────────┼───────────┐               │
│                              ↕           ↕           ↕               │
│                         RDS (PG)    ElastiCache   Secrets Mgr        │
│                         + replica    (Redis)                          │
│                                         ↕                            │
│                                    ECS (Celery workers)              │
│                                         ↕                            │
│                              ┌──────────┼──────────┐                 │
│                              ↕          ↕          ↕                 │
│                           SES       CloudWatch    SQS                │
│                         (email)     (logs/metrics) (events)          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. Interview Questions

### Q: How do you deploy Django on AWS?
```
Option 1: ECS Fargate (containerized, recommended)
  - Docker image → ECR → ECS with ALB
  - Auto-scaling based on CPU/memory
  - RDS for DB, ElastiCache for Redis, S3 for files

Option 2: Elastic Beanstalk (simpler, less control)
  - Upload zip → EB handles everything
  - Good for small teams, quick setup

Option 3: Lambda + API Gateway (serverless)
  - Using Zappa or Mangum
  - Zero cost at zero traffic, but cold starts
  - Not suitable for WebSocket or long-running tasks

My recommendation: ECS Fargate for production —
  full control, no server management, scales horizontally.
```

### Q: How do you handle file uploads at scale?
```
1. Pre-signed URL pattern (Django generates URL, frontend uploads directly to S3)
   - Faster: file goes straight to S3, not through Django
   - Scales: doesn't consume Django worker time
   - Secure: URL expires in 5 minutes

2. Post-upload processing:
   - S3 event → Lambda → resize/compress → save to another S3 path
   - Or: S3 event → SQS → Celery worker → process

3. Never store uploads on ECS disk (containers are ephemeral)
```
