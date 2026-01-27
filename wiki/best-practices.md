# Best Practices and Security Guide

Security best practices, optimization tips, and recommendations for Django Docker deployments.

## Table of Contents

- [Security Best Practices](#security-best-practices)
- [Docker Best Practices](#docker-best-practices)
- [Django Security](#django-security)
- [Database Security](#database-security)
- [Nginx Security](#nginx-security)
- [Performance Optimization](#performance-optimization)
- [Monitoring and Logging](#monitoring-and-logging)
- [Backup Strategies](#backup-strategies)
- [CI/CD Best Practices](#cicd-best-practices)
- [Development Workflow](#development-workflow)

## Security Best Practices

### Environment Variables

**❌ Don't**:
- Hardcode secrets in code
- Commit `.env` files to version control
- Use the same secrets in dev and production
- Store secrets in Dockerfiles

**✅ Do**:
```bash
# Add to .gitignore
.env
.env.dev
.env.prod
.env.prod.db

# Use strong, unique secrets
python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'

# Use environment-specific files
.env.dev          # Development
.env.prod         # Production
.env.staging      # Staging
```

### Secret Management

**Production Secret Management**:

1. **Docker Secrets** (Docker Swarm):
```yaml
services:
  web:
    secrets:
      - db_password
      - secret_key
secrets:
  db_password:
    external: true
  secret_key:
    external: true
```

2. **Environment Variables from Host**:
```bash
# Don't store in docker-compose
export DJANGO_SECRET_KEY="your-secret"
docker-compose up
```

3. **External Secret Managers**:
- AWS Secrets Manager
- HashiCorp Vault
- Azure Key Vault
- Google Cloud Secret Manager

### HTTPS/SSL

**Required Settings**:
```python
# settings.py (production only)
if not DEBUG:
    # Force HTTPS
    SECURE_SSL_REDIRECT = True
    
    # Secure cookies
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
    
    # HSTS
    SECURE_HSTS_SECONDS = 31536000  # 1 year
    SECURE_HSTS_INCLUDE_SUBDOMAINS = True
    SECURE_HSTS_PRELOAD = True
    
    # Proxy headers
    SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

### Security Headers

**Nginx Configuration**:
```nginx
# Security Headers
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

**Django Middleware**:
```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    # ... other middleware
]

# Security settings
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'SAMEORIGIN'
```

### Input Validation

**Always Validate**:
```python
# Use Django forms
from django import forms

class UserForm(forms.Form):
    email = forms.EmailField(max_length=254)
    age = forms.IntegerField(min_value=0, max_value=150)
    
    def clean_email(self):
        email = self.cleaned_data['email']
        # Custom validation
        if not email.endswith('@example.com'):
            raise forms.ValidationError('Only example.com emails allowed')
        return email

# Use model validation
from django.core.validators import MinValueValidator, MaxValueValidator

class Product(models.Model):
    price = models.DecimalField(
        max_digits=10,
        decimal_places=2,
        validators=[MinValueValidator(0.01)]
    )
```

### SQL Injection Prevention

**❌ Don't**:
```python
# Never use raw SQL with string formatting
cursor.execute("SELECT * FROM users WHERE username = '%s'" % username)

# Never trust user input in raw queries
User.objects.raw(f"SELECT * FROM users WHERE id = {user_id}")
```

**✅ Do**:
```python
# Use Django ORM
User.objects.filter(username=username)

# Or parameterized queries
cursor.execute("SELECT * FROM users WHERE username = %s", [username])

# For raw queries, use parameters
User.objects.raw("SELECT * FROM users WHERE id = %s", [user_id])
```

### XSS Prevention

**Template Auto-escaping**:
```django
{# Django auto-escapes by default #}
{{ user_input }}

{# Mark safe only when necessary #}
{{ trusted_html|safe }}

{# Or in Python #}
from django.utils.html import escape
safe_text = escape(user_input)
```

### CSRF Protection

**Always Use CSRF Tokens**:
```django
{# In forms #}
<form method="post">
    {% csrf_token %}
    <!-- form fields -->
</form>
```

```python
# For AJAX requests
from django.views.decorators.csrf import csrf_exempt, csrf_protect

# Don't disable CSRF unless absolutely necessary
@csrf_exempt  # ❌ Avoid this
def api_view(request):
    pass

# Instead, use proper CSRF handling
@csrf_protect  # ✅ Better
def api_view(request):
    pass
```

## Docker Best Practices

### Multi-stage Builds

**Reduce Image Size**:
```dockerfile
# ✅ Multi-stage build
FROM python:3.11 as builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-slim
COPY --from=builder /root/.local /root/.local
COPY . /app
ENV PATH=/root/.local/bin:$PATH
```

### Non-root User

**Security Best Practice**:
```dockerfile
# ✅ Run as non-root user
FROM python:3.11-slim

# Create user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Set up app directory
WORKDIR /home/appuser/app
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

CMD ["gunicorn", "config.wsgi:application"]
```

### Layer Optimization

**Optimize Build Cache**:
```dockerfile
# ✅ Order from least to most frequently changed
FROM python:3.11-slim

# Install system dependencies (rarely changes)
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies (changes occasionally)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code (changes frequently)
COPY . .
```

### Health Checks

**Monitor Container Health**:
```dockerfile
# Dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health/ || exit 1
```

```yaml
# docker-compose.yml
services:
  web:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### Resource Limits

**Prevent Resource Exhaustion**:
```yaml
# docker-compose.yml
services:
  web:
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
```

### Network Isolation

**Use Custom Networks**:
```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    networks:
      - backend
  
  db:
    networks:
      - backend
  
  nginx:
    networks:
      - frontend
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access
```

## Django Security

### Debug Mode

**❌ Never in Production**:
```python
# settings.py
DEBUG = False  # ALWAYS False in production

# Use environment variable
DEBUG = int(os.environ.get('DEBUG', 0))
```

### Allowed Hosts

**Restrict Access**:
```python
# ❌ Don't
ALLOWED_HOSTS = ['*']

# ✅ Do
ALLOWED_HOSTS = [
    'example.com',
    'www.example.com',
    '192.168.1.100',
]

# Or from environment
ALLOWED_HOSTS = os.environ.get('DJANGO_ALLOWED_HOSTS', '').split(' ')
```

### Secret Key

**Strong and Unique**:
```python
# Generate new key for each environment
# python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'

SECRET_KEY = os.environ.get('SECRET_KEY')

# Validate it exists
if not SECRET_KEY:
    raise ValueError('SECRET_KEY environment variable is not set')
```

### Password Validation

**Strong Passwords**:
```python
# settings.py
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {
            'min_length': 12,  # Increase from default 8
        }
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]
```

### Admin Security

**Protect Admin Panel**:
```python
# urls.py - Change admin URL
from django.contrib import admin

urlpatterns = [
    path('secret-admin-url/', admin.site.urls),  # Not 'admin/'
]

# Require HTTPS for admin
admin.site.site_header = "My Admin"
admin.site.login_template = 'admin/login.html'

# Use django-admin-honeypot to catch attacks
# pip install django-admin-honeypot
```

### Rate Limiting

**Prevent Brute Force**:
```python
# Use django-ratelimit
# pip install django-ratelimit

from django_ratelimit.decorators import ratelimit

@ratelimit(key='ip', rate='5/m', method='POST')
def login_view(request):
    # Login logic
    pass
```

## Database Security

### Connection Security

**Use SSL/TLS**:
```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'OPTIONS': {
            'sslmode': 'require',
            'sslrootcert': '/path/to/root.crt',
        },
    }
}
```

### Least Privilege

**Minimal Permissions**:
```sql
-- Create limited user
CREATE USER django_app WITH PASSWORD 'strong_password';

-- Grant only necessary permissions
GRANT CONNECT ON DATABASE mydb TO django_app;
GRANT USAGE ON SCHEMA public TO django_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO django_app;

-- Don't grant
-- DROP, CREATE, ALTER permissions to application user
```

### Backup Encryption

**Encrypt Backups**:
```bash
# Encrypted backup
docker-compose exec db pg_dump -U django_user django_db | \
  gpg --symmetric --cipher-algo AES256 > backup_$(date +%Y%m%d).sql.gpg

# Restore
gpg --decrypt backup_20240101.sql.gpg | \
  docker-compose exec -T db psql -U django_user -d django_db
```

## Nginx Security

### Hide Version

**Don't Expose Nginx Version**:
```nginx
# nginx.conf
server_tokens off;
```

### Request Limits

**Prevent DoS**:
```nginx
# nginx.conf
http {
    # Limit request rate
    limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;
    limit_req zone=one burst=20 nodelay;
    
    # Limit connections
    limit_conn_zone $binary_remote_addr zone=addr:10m;
    limit_conn addr 10;
    
    # Timeouts
    client_body_timeout 12;
    client_header_timeout 12;
    keepalive_timeout 15;
    send_timeout 10;
    
    # Buffer sizes
    client_body_buffer_size 10K;
    client_header_buffer_size 1k;
    client_max_body_size 8m;
    large_client_header_buffers 2 1k;
}
```

### Block Bad Bots

**Filter Traffic**:
```nginx
# Block user agents
if ($http_user_agent ~* (bot|crawler|spider)) {
    return 403;
}

# Block specific IPs
deny 192.168.1.1;
allow all;
```

## Performance Optimization

### Database Optimization

**Query Optimization**:
```python
# ✅ Use select_related for foreign keys
posts = Post.objects.select_related('author').all()

# ✅ Use prefetch_related for many-to-many
posts = Post.objects.prefetch_related('tags').all()

# ✅ Only select needed fields
posts = Post.objects.only('title', 'created_at')

# ✅ Use iterator for large querysets
for user in User.objects.iterator():
    process_user(user)

# ❌ Avoid N+1 queries
for post in Post.objects.all():
    print(post.author.name)  # N+1 query

# ✅ Better
posts = Post.objects.select_related('author')
for post in posts:
    print(post.author.name)
```

**Database Indexes**:
```python
class Post(models.Model):
    title = models.CharField(max_length=200, db_index=True)
    slug = models.SlugField(unique=True, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['created_at', 'title']),
            models.Index(fields=['-created_at']),
        ]
```

### Caching Strategy

**Multiple Cache Levels**:
```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://redis:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'KEY_PREFIX': 'myapp',
        'TIMEOUT': 300,
    }
}

# Per-site cache
MIDDLEWARE = [
    'django.middleware.cache.UpdateCacheMiddleware',
    # ... other middleware
    'django.middleware.cache.FetchFromCacheMiddleware',
]

# Per-view cache
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # 15 minutes
def my_view(request):
    pass

# Template fragment cache
{% load cache %}
{% cache 500 sidebar %}
    <!-- expensive template code -->
{% endcache %}

# Low-level cache
from django.core.cache import cache

def get_user_data(user_id):
    key = f'user_data_{user_id}'
    data = cache.get(key)
    
    if data is None:
        data = expensive_operation(user_id)
        cache.set(key, data, 300)  # 5 minutes
    
    return data
```

### Static Files Optimization

**CDN and Compression**:
```python
# settings.py
# Use WhiteNoise for static files
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',
    # ...
]

STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

# Or use S3/CloudFront
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
STATICFILES_STORAGE = 'storages.backends.s3boto3.S3StaticStorage'

AWS_STORAGE_BUCKET_NAME = 'your-bucket'
AWS_S3_CUSTOM_DOMAIN = f'{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com'
AWS_S3_OBJECT_PARAMETERS = {
    'CacheControl': 'max-age=86400',
}
```

### Gunicorn Tuning

**Worker Configuration**:
```python
# gunicorn_config.py
import multiprocessing

# Workers
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = 'sync'  # or 'gevent' for async

# Connections
worker_connections = 1000
max_requests = 1000
max_requests_jitter = 50

# Timeouts
timeout = 30
graceful_timeout = 30
keepalive = 2

# Logging
accesslog = '-'
errorlog = '-'
loglevel = 'info'

# Process naming
proc_name = 'django_app'

# Server mechanics
daemon = False
pidfile = None
user = None
group = None
tmp_upload_dir = None
```

## Monitoring and Logging

### Application Monitoring

**Setup Logging**:
```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/django/debug.log',
            'maxBytes': 1024 * 1024 * 15,  # 15MB
            'backupCount': 10,
            'formatter': 'verbose',
        },
    },
    'root': {
        'handlers': ['console', 'file'],
        'level': 'INFO',
    },
    'loggers': {
        'django': {
            'handlers': ['console', 'file'],
            'level': 'INFO',
            'propagate': False,
        },
        'django.db.backends': {
            'handlers': ['console'],
            'level': 'DEBUG' if DEBUG else 'INFO',
        },
    },
}
```

### Error Tracking

**Use Sentry**:
```python
# pip install sentry-sdk

# settings.py
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

sentry_sdk.init(
    dsn=os.environ.get('SENTRY_DSN'),
    integrations=[DjangoIntegration()],
    traces_sample_rate=0.1,
    send_default_pii=False,
    environment='production',
)
```

### Health Monitoring

**Health Check Endpoint**:
```python
# views.py
from django.http import JsonResponse
from django.db import connection

def health_check(request):
    health = {
        'status': 'healthy',
        'checks': {}
    }
    
    # Database check
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        health['checks']['database'] = 'ok'
    except Exception as e:
        health['status'] = 'unhealthy'
        health['checks']['database'] = str(e)
    
    # Cache check
    try:
        from django.core.cache import cache
        cache.set('health_check', 'ok', 10)
        result = cache.get('health_check')
        health['checks']['cache'] = 'ok' if result == 'ok' else 'failed'
    except Exception as e:
        health['checks']['cache'] = str(e)
    
    status_code = 200 if health['status'] == 'healthy' else 500
    return JsonResponse(health, status=status_code)
```

## Backup Strategies

### Automated Backups

**Backup Script**:
```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# Database backup
docker-compose exec -T db pg_dump -U $DB_USER $DB_NAME | \
  gzip > "$BACKUP_DIR/db_$DATE.sql.gz"

# Media files backup
tar -czf "$BACKUP_DIR/media_$DATE.tar.gz" /path/to/mediafiles/

# Clean old backups
find $BACKUP_DIR -name "*.gz" -mtime +$RETENTION_DAYS -delete

# Upload to S3 (optional)
aws s3 cp "$BACKUP_DIR/db_$DATE.sql.gz" s3://bucket/backups/
```

### Test Restores

**Regularly Test**:
```bash
# Test database restore
gunzip -c backup.sql.gz | docker-compose exec -T db psql -U django_user -d test_db

# Verify data
docker-compose exec db psql -U django_user -d test_db -c "SELECT COUNT(*) FROM django_table;"
```

## CI/CD Best Practices

### GitHub Actions Example

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
    
    - name: Run tests
      run: |
        python manage.py test
    
    - name: Lint
      run: |
        flake8 .
    
    - name: Security check
      run: |
        bandit -r .
```

## Development Workflow

### Git Workflow

```bash
# Feature branch
git checkout -b feature/new-feature

# Make changes
git add .
git commit -m "feat: add new feature"

# Push
git push origin feature/new-feature

# Create pull request
# After review and merge to main, deploy
```

### Code Quality

```bash
# Use pre-commit hooks
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
      - id: black
  - repo: https://github.com/PyCQA/flake8
    rev: 6.0.0
    hooks:
      - id: flake8
```

## Security Checklist

- [ ] DEBUG = False in production
- [ ] Strong, unique SECRET_KEY
- [ ] ALLOWED_HOSTS properly configured
- [ ] HTTPS enabled with valid certificate
- [ ] Security headers configured
- [ ] CSRF protection enabled
- [ ] SQL injection prevention (use ORM)
- [ ] XSS protection (template escaping)
- [ ] Strong password validation
- [ ] Rate limiting enabled
- [ ] Admin URL changed
- [ ] Non-root user in containers
- [ ] Environment variables secured
- [ ] Regular security updates
- [ ] Database backups configured
- [ ] Error tracking enabled
- [ ] Logging configured
- [ ] Firewall configured
- [ ] SSL/TLS for database connections

## Next Steps

- [Development Setup](development-setup.md)
- [Production Deployment](production-deployment.md)
- [SSL/HTTPS Setup](ssl-setup.md)
- [Troubleshooting](troubleshooting.md)
