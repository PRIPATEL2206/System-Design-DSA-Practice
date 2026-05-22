# Django Security Hardening — Senior Level

## 1. Django's Built-in Security (What It Does Automatically)

```
┌─────────────────────────────────────────────────────────────┐
│ Attack              │ Django's Protection        │ Enabled?  │
├─────────────────────┼────────────────────────────┼───────────┤
│ SQL Injection       │ ORM parameterization       │ Auto      │
│ XSS                 │ Template auto-escaping     │ Auto      │
│ CSRF                │ CsrfViewMiddleware + token │ Auto      │
│ Clickjacking        │ X-Frame-Options: DENY      │ Auto      │
│ Session Hijacking   │ Secure cookies, rotation   │ Config    │
│ Host Header Attack  │ ALLOWED_HOSTS validation   │ Config    │
│ HTTPS redirect      │ SecurityMiddleware         │ Config    │
│ Password Hashing    │ PBKDF2/Argon2 (salted)     │ Auto      │
└─────────────────────┴────────────────────────────┴───────────┘
```

---

## 2. Production Security Settings

```python
# settings/production.py

# CRITICAL: Never expose these
DEBUG = False
SECRET_KEY = os.environ['DJANGO_SECRET_KEY']  # from env, NEVER in code
ALLOWED_HOSTS = ['yourdomain.com', 'www.yourdomain.com']

# HTTPS enforcement
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# HSTS (HTTP Strict Transport Security)
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Cookie security
SESSION_COOKIE_SECURE = True      # only send over HTTPS
SESSION_COOKIE_HTTPONLY = True     # JS can't access session cookie
SESSION_COOKIE_SAMESITE = 'Lax'   # prevent CSRF via cross-site requests
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_HTTPONLY = True

# Content security
SECURE_CONTENT_TYPE_NOSNIFF = True    # X-Content-Type-Options: nosniff
SECURE_BROWSER_XSS_FILTER = True     # X-XSS-Protection: 1; mode=block
X_FRAME_OPTIONS = 'DENY'             # prevent clickjacking

# Session config
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'sessions'
SESSION_COOKIE_AGE = 3600  # 1 hour timeout (not default 2 weeks)

# Password validation
AUTH_PASSWORD_VALIDATORS = [
    {'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'},
    {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
     'OPTIONS': {'min_length': 12}},
    {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'},
    {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'},
]
```

---

## 3. SQL Injection Prevention

### How Django Prevents It
```python
# ✓ SAFE: ORM always parameterizes
User.objects.filter(email=user_input)
# Generates: SELECT * FROM auth_user WHERE email = %s (parameterized)

# ✓ SAFE: Raw SQL with params
User.objects.raw("SELECT * FROM auth_user WHERE email = %s", [user_input])

# ❌ VULNERABLE: String formatting in raw SQL
User.objects.raw(f"SELECT * FROM auth_user WHERE email = '{user_input}'")
# Attacker: user_input = "'; DROP TABLE auth_user; --"

# ❌ VULNERABLE: .extra() with unsanitized input (DEPRECATED)
User.objects.extra(where=[f"email = '{user_input}'"])
```

### Dangerous Patterns to Avoid
```python
# ❌ NEVER interpolate user input into SQL
from django.db import connection
cursor = connection.cursor()
cursor.execute(f"SELECT * FROM products WHERE name LIKE '%{search}%'")

# ✓ SAFE parameterized version
cursor.execute("SELECT * FROM products WHERE name LIKE %s", [f'%{search}%'])

# ❌ NEVER use user input in .order_by() directly
# Attacker can inject: "name; DROP TABLE products"
order_field = request.GET.get('order_by', 'name')
Product.objects.order_by(order_field)

# ✓ SAFE: Whitelist allowed fields
ALLOWED_ORDER_FIELDS = {'name', 'price', '-price', 'created_at', '-created_at'}
order_field = request.GET.get('order_by', 'name')
if order_field not in ALLOWED_ORDER_FIELDS:
    order_field = 'name'
Product.objects.order_by(order_field)
```

---

## 4. XSS (Cross-Site Scripting) Prevention

### Django's Auto-Escaping
```html
<!-- ✓ SAFE: Auto-escaped by default -->
<p>{{ user_input }}</p>
<!-- Input: <script>alert('xss')</script> -->
<!-- Output: &lt;script&gt;alert(&#x27;xss&#x27;)&lt;/script&gt; -->

<!-- ❌ VULNERABLE: |safe filter disables escaping -->
<p>{{ user_input|safe }}</p>
<!-- ONLY use |safe for content YOU generated, NEVER user input -->

<!-- ❌ VULNERABLE: {% autoescape off %} -->
{% autoescape off %}
  {{ user_input }}
{% endautoescape %}
```

### XSS in DRF (API Responses)
```python
# DRF is mostly immune (returns JSON, not HTML)
# But if you render HTML from API data:

# ❌ Frontend vulnerability (React/Vue)
# Never use dangerouslySetInnerHTML with API data
<div dangerouslySetInnerHTML={{__html: apiResponse.description}} />

# ✓ Sanitize on backend before storing
import bleach

class ProductSerializer(serializers.ModelSerializer):
    def validate_description(self, value):
        return bleach.clean(value, tags=['p', 'b', 'i', 'a'], strip=True)
```

### Content Security Policy (CSP)
```python
# Install: pip install django-csp
# settings.py
MIDDLEWARE = [
    'csp.middleware.CSPMiddleware',
    ...
]

CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC = ("'self'", 'cdn.jsdelivr.net')
CSP_STYLE_SRC = ("'self'", "'unsafe-inline'", 'fonts.googleapis.com')
CSP_IMG_SRC = ("'self'", 'data:', 's3.amazonaws.com')
CSP_FONT_SRC = ("'self'", 'fonts.gstatic.com')
CSP_CONNECT_SRC = ("'self'", 'api.yourdomain.com')
```

---

## 5. CSRF Protection

### How CSRF Works in Django
```python
# Django's CSRF protection:
# 1. Server sends CSRF token in cookie + form
# 2. On POST/PUT/DELETE, client must include token
# 3. Server compares cookie token vs request token
# 4. If mismatch → 403 Forbidden

# For templates (auto-included):
<form method="post">
    {% csrf_token %}
    ...
</form>

# For AJAX/DRF (get token from cookie):
# JavaScript:
function getCookie(name) {
    let value = document.cookie.match('(^|;)\\s*' + name + '\\s*=\\s*([^;]+)');
    return value ? value.pop() : '';
}

fetch('/api/orders/', {
    method: 'POST',
    headers: {
        'X-CSRFToken': getCookie('csrftoken'),
        'Content-Type': 'application/json',
    },
    body: JSON.stringify(data)
});
```

### CSRF with DRF + JWT
```python
# If using JWT authentication (token in Authorization header):
# CSRF is NOT needed because:
# - CSRF exploits browser auto-sending cookies
# - JWT is manually added to headers — attacker can't access it
# - So DRF exempts token auth from CSRF by default

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        # No SessionAuthentication = no CSRF needed for API
    ],
}
```

---

## 6. Authentication Security

### Secure Custom User Model
```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    email = models.EmailField(unique=True)
    is_email_verified = models.BooleanField(default=False)
    failed_login_attempts = models.IntegerField(default=0)
    locked_until = models.DateTimeField(null=True, blank=True)
    
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']

    def lock_account(self):
        from django.utils import timezone
        from datetime import timedelta
        self.locked_until = timezone.now() + timedelta(minutes=30)
        self.save(update_fields=['locked_until'])

    def is_locked(self):
        from django.utils import timezone
        if self.locked_until and self.locked_until > timezone.now():
            return True
        return False
```

### Rate-Limited Login with Account Locking
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from django.contrib.auth import authenticate

class LoginView(APIView):
    throttle_classes = [LoginRateThrottle]  # 5 attempts/minute
    
    def post(self, request):
        email = request.data.get('email')
        password = request.data.get('password')
        
        try:
            user = User.objects.get(email=email)
        except User.DoesNotExist:
            return Response(
                {'error': 'Invalid credentials'},
                status=status.HTTP_401_UNAUTHORIZED
            )
        
        if user.is_locked():
            return Response(
                {'error': 'Account locked. Try again in 30 minutes.'},
                status=status.HTTP_423_LOCKED
            )
        
        auth_user = authenticate(email=email, password=password)
        
        if auth_user is None:
            user.failed_login_attempts += 1
            if user.failed_login_attempts >= 5:
                user.lock_account()
            user.save(update_fields=['failed_login_attempts'])
            return Response(
                {'error': 'Invalid credentials'},
                status=status.HTTP_401_UNAUTHORIZED
            )
        
        user.failed_login_attempts = 0
        user.save(update_fields=['failed_login_attempts'])
        
        token = generate_jwt_token(user)
        return Response({'token': token})
```

---

## 7. Object-Level Permissions & Data Isolation

```python
# Multi-tenant data isolation
class TenantMiddleware:
    """Ensure users only see their organization's data."""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        if request.user.is_authenticated:
            request.tenant = request.user.organization
        return self.get_response(request)


class TenantModelMixin:
    """Mixin for ViewSets that filters by tenant."""
    
    def get_queryset(self):
        qs = super().get_queryset()
        if not self.request.user.is_superuser:
            qs = qs.filter(organization=self.request.tenant)
        return qs
    
    def perform_create(self, serializer):
        serializer.save(organization=self.request.tenant)


class OrderViewSet(TenantModelMixin, viewsets.ModelViewSet):
    queryset = Order.objects.all()
    # Users can ONLY see their organization's orders
    # No matter what ID they pass in the URL
```

---

## 8. File Upload Security

```python
import os
import magic
from django.core.exceptions import ValidationError

ALLOWED_EXTENSIONS = {'.jpg', '.jpeg', '.png', '.pdf', '.docx'}
ALLOWED_MIME_TYPES = {'image/jpeg', 'image/png', 'application/pdf'}
MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB

def validate_file(file):
    # Check file size
    if file.size > MAX_FILE_SIZE:
        raise ValidationError(f'File too large. Max {MAX_FILE_SIZE // 1024 // 1024}MB.')
    
    # Check extension
    ext = os.path.splitext(file.name)[1].lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise ValidationError(f'File type {ext} not allowed.')
    
    # Check MIME type (don't trust extension alone)
    mime = magic.from_buffer(file.read(1024), mime=True)
    file.seek(0)
    if mime not in ALLOWED_MIME_TYPES:
        raise ValidationError(f'Invalid file content type: {mime}')
    
    return file


class DocumentSerializer(serializers.ModelSerializer):
    file = serializers.FileField(validators=[validate_file])
    
    class Meta:
        model = Document
        fields = ['id', 'file', 'uploaded_at']
    
    def create(self, validated_data):
        # Generate safe filename (prevent path traversal)
        import uuid
        file = validated_data['file']
        ext = os.path.splitext(file.name)[1]
        file.name = f"{uuid.uuid4()}{ext}"
        return super().create(validated_data)
```

---

## 9. Secrets Management

```python
# ❌ NEVER: secrets in settings.py or code
SECRET_KEY = 'hardcoded-secret'
DATABASE_PASSWORD = 'my-password'

# ✓ Method 1: Environment variables
import os
SECRET_KEY = os.environ['DJANGO_SECRET_KEY']
DATABASES = {
    'default': {
        'PASSWORD': os.environ['DB_PASSWORD'],
    }
}

# ✓ Method 2: AWS Secrets Manager (production)
import boto3
import json

def get_secret(secret_name):
    client = boto3.client('secretsmanager', region_name='ap-south-1')
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

secrets = get_secret('prod/django/secrets')
SECRET_KEY = secrets['SECRET_KEY']
DATABASE_PASSWORD = secrets['DB_PASSWORD']

# ✓ Method 3: django-environ (.env file for local dev)
import environ
env = environ.Env()
environ.Env.read_env('.env')

SECRET_KEY = env('SECRET_KEY')
DEBUG = env.bool('DEBUG', default=False)
DATABASE_URL = env.db()  # parses DATABASE_URL env var
```

---

## 10. Security Checklist for Production

```
□ DEBUG = False
□ SECRET_KEY from environment (not in code/repo)
□ ALLOWED_HOSTS set to exact domains
□ HTTPS enforced (SECURE_SSL_REDIRECT = True)
□ HSTS enabled (SECURE_HSTS_SECONDS > 0)
□ Cookies: Secure, HttpOnly, SameSite=Lax
□ CSRF enabled for session-based auth
□ CSP headers configured
□ File uploads validated (size, type, MIME)
□ No .extra() or raw SQL with user input
□ Rate limiting on login/signup/password-reset
□ Account lockout after failed attempts
□ django-axes or similar for brute-force detection
□ Sensitive data encrypted at rest (DB, S3)
□ Logging: log auth failures, never log passwords/tokens
□ Dependencies: pip-audit / safety check for vulnerabilities
□ Admin URL changed from /admin/ to /secret-admin-panel/
□ CORS properly configured (not *)
```

---

## 11. CORS Configuration

```python
# pip install django-cors-headers
INSTALLED_APPS = [..., 'corsheaders']
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',  # MUST be before CommonMiddleware
    'django.middleware.common.CommonMiddleware',
    ...
]

# ❌ NEVER in production:
CORS_ALLOW_ALL_ORIGINS = True

# ✓ Whitelist specific origins:
CORS_ALLOWED_ORIGINS = [
    'https://yourfrontend.com',
    'https://app.yourfrontend.com',
]
CORS_ALLOW_CREDENTIALS = True  # if using cookies
CORS_ALLOW_HEADERS = list(default_headers) + ['X-Custom-Header']
```

---

## 12. Interview Questions

### Q: How does Django prevent SQL injection?
```
Django's ORM uses parameterized queries for ALL database operations.
User input is NEVER interpolated into SQL strings — it's always passed
as parameters that the database driver handles safely.

The only risk is: using .raw() or cursor.execute() with f-strings/format().
Rule: ALWAYS use %s placeholders with a params list.
```

### Q: What's the difference between authentication and authorization?
```
Authentication: WHO are you? (login, JWT, session)
Authorization: WHAT can you do? (permissions, roles)

Django separates these:
- Authentication: authenticate() → returns user or None
- Authorization: has_perm(), permission_classes, object-level checks

Senior pattern: RBAC (Role-Based Access Control)
  User → Role (admin, editor, viewer) → Permissions
```

### Q: How do you handle secrets in a Django deployment?
```
Never in code. Options by environment:
- Local dev: .env file (git-ignored) + django-environ
- CI/CD: GitHub Secrets / GitLab CI variables
- Production: AWS Secrets Manager / Parameter Store
- Kubernetes: K8s Secrets mounted as env vars

Rotation: Secrets Manager supports auto-rotation.
Key rule: If you can see the secret in git history, it's compromised.
```
