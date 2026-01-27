# Staging Environment Setup

This guide covers setting up a staging environment for your Django application using Docker. The staging environment mirrors production but is used for testing before deploying to production.

## Table of Contents

- [Overview](#overview)
- [Why Use a Staging Environment](#why-use-a-staging-environment)
- [Staging vs Development vs Production](#staging-vs-development-vs-production)
- [Staging Environment Setup](#staging-environment-setup)
- [Docker Compose Configuration](#docker-compose-configuration)
- [Environment Variables](#environment-variables)
- [Deployment Process](#deployment-process)
- [Testing in Staging](#testing-in-staging)
- [Common Staging Workflows](#common-staging-workflows)
- [Troubleshooting](#troubleshooting)

## Overview

The staging environment is an intermediate environment between development and production that:
- Mirrors production configuration as closely as possible
- Uses production-like settings (Gunicorn, nginx-proxy, SSL certificates)
- Provides a safe testing ground before production deployment
- Allows for integration testing, user acceptance testing, and performance testing

## Why Use a Staging Environment

**Benefits:**
- ✅ Test production-like configurations without risking production
- ✅ Catch deployment issues before they reach production
- ✅ Perform integration testing with production-like data
- ✅ Validate SSL/certificate configurations
- ✅ Test database migrations with production-like data volumes
- ✅ Conduct user acceptance testing (UAT)
- ✅ Performance testing and optimization
- ✅ Train team members on new features

## Staging vs Development vs Production

| Aspect | Development | Staging | Production |
|--------|-------------|---------|------------|
| **Purpose** | Local development | Pre-production testing | Live application |
| **Configuration** | Django dev server | Production-like setup | Production setup |
| **Debug Mode** | `DEBUG=1` | `DEBUG=0` | `DEBUG=0` |
| **Server** | Django runserver | Gunicorn + nginx-proxy | Gunicorn + nginx-proxy |
| **SSL/HTTPS** | Optional | Yes (Step CA) | Yes (Step CA) |
| **Database** | SQLite or PostgreSQL | PostgreSQL | PostgreSQL |
| **Data** | Sample/test data | Production-like data | Real data |
| **Error Reporting** | Console output | Logging + Error tracking | Logging + Error tracking |
| **Performance** | Not critical | Should be monitored | Critical |
| **Availability** | Intermittent | High | Very High |
| **Access** | Developers only | Team + Stakeholders | Public/Customers |

## Staging Environment Setup

### 1. Create Staging Configuration Files

Create staging-specific Docker Compose and environment files:

```bash
# Create staging directory structure
mkdir -p staging
cd staging

# Copy production configurations as base
cp docker-compose.prod.yml docker-compose.staging.yml
cp .env.prod .env.staging
cp .env.prod.db .env.staging.db
```

### 2. Staging Dockerfile

You can use the same production Dockerfile or create a staging-specific one:

**Option 1: Use Production Dockerfile** (Recommended)
```yaml
# docker-compose.staging.yml
services:
  web:
    build:
      context: ./app
      dockerfile: Dockerfile.prod  # Same as production
```

**Option 2: Create Staging-Specific Dockerfile**
```dockerfile
# Dockerfile.staging
# Identical to Dockerfile.prod but with staging-specific adjustments if needed

FROM python:3.11.4-slim-buster as builder

WORKDIR /usr/src/app

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Install system dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc

# Install Python dependencies
RUN pip install --upgrade pip
COPY ./requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /usr/src/app/wheels -r requirements.txt

# Final stage
FROM python:3.11.4-slim-buster

RUN mkdir -p /home/app
RUN addgroup --system app && adduser --system --group app

ENV HOME=/home/app
ENV APP_HOME=/home/app/web
RUN mkdir $APP_HOME
RUN mkdir $APP_HOME/staticfiles
RUN mkdir $APP_HOME/mediafiles
WORKDIR $APP_HOME

RUN apt-get update && apt-get install -y --no-install-recommends netcat
COPY --from=builder /usr/src/app/wheels /wheels
COPY --from=builder /usr/src/app/requirements.txt .
RUN pip install --upgrade pip
RUN pip install --no-cache /wheels/*

COPY ./entrypoint.prod.sh .
RUN sed -i 's/\r$//g'  $APP_HOME/entrypoint.prod.sh
RUN chmod +x  $APP_HOME/entrypoint.prod.sh

COPY . $APP_HOME
RUN chown -R app:app $APP_HOME

USER app

ENTRYPOINT ["/home/app/web/entrypoint.prod.sh"]
```

## Docker Compose Configuration

### Full Staging Docker Compose

Create `docker-compose.staging.yml`:

```yaml
version: '3.8'

services:
  # Django Application
  web:
    build:
      context: ./app
      dockerfile: Dockerfile.prod
    command: gunicorn config.wsgi:application --bind 0.0.0.0:8000
    volumes:
      - static_volume:/home/app/web/staticfiles
      - media_volume:/home/app/web/mediafiles
    expose:
      - 8000
    env_file:
      - ./.env.staging
    environment:
      # nginx-proxy configuration
      - VIRTUAL_HOST=staging.myapp.local
      - VIRTUAL_PORT=8000
      # acme-companion configuration
      - LETSENCRYPT_HOST=staging.myapp.local
      - LETSENCRYPT_EMAIL=your-email@yourdomain.com
    depends_on:
      - db
    networks:
      - nginx-proxy
      - backend
    restart: unless-stopped
    labels:
      - "com.example.environment=staging"

  # Database
  db:
    image: postgres:15
    volumes:
      - postgres_data_staging:/var/lib/postgresql/data/
    env_file:
      - ./.env.staging.db
    networks:
      - backend
    restart: unless-stopped
    labels:
      - "com.example.environment=staging"

volumes:
  postgres_data_staging:
    name: staging_postgres_data
  static_volume:
    name: staging_static_volume
  media_volume:
    name: staging_media_volume

networks:
  nginx-proxy:
    external: true
  backend:
    driver: bridge
```

**Key Staging-Specific Settings:**
- Different `VIRTUAL_HOST` (e.g., `staging.myapp.local` instead of `myapp.local`)
- Separate named volumes with staging prefix
- Labels to identify staging containers
- Same production-like configuration (Gunicorn, nginx-proxy)

## Environment Variables

### Staging Django Environment (.env.staging)

```env
# Staging Environment Variables

# Debug should be off in staging (production-like)
DEBUG=0

# Use a different secret key than production
SECRET_KEY=your-staging-secret-key-different-from-production

# Allowed hosts - staging domain
DJANGO_ALLOWED_HOSTS=staging.myapp.local staging.yourdomain.com

# Database configuration
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=django_staging
SQL_USER=django_staging_user
SQL_PASSWORD=staging_database_password
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres

# Environment identifier
ENVIRONMENT=staging

# Email configuration (use test email service or real SMTP)
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp.mailtrap.io  # Or your staging email service
EMAIL_PORT=2525
EMAIL_USE_TLS=True
EMAIL_HOST_USER=your-mailtrap-user
EMAIL_HOST_PASSWORD=your-mailtrap-password

# Sentry/Error tracking (use staging project)
SENTRY_DSN=your-staging-sentry-dsn
SENTRY_ENVIRONMENT=staging

# AWS/S3 (if using cloud storage, use staging bucket)
AWS_STORAGE_BUCKET_NAME=your-staging-bucket
AWS_S3_REGION_NAME=us-east-1

# Redis (if using)
REDIS_URL=redis://redis:6379/2  # Different DB than production

# Celery (if using)
CELERY_BROKER_URL=redis://redis:6379/2
```

### Staging Database Environment (.env.staging.db)

```env
# PostgreSQL configuration for staging
POSTGRES_USER=django_staging_user
POSTGRES_PASSWORD=staging_database_password
POSTGRES_DB=django_staging
```

## Deployment Process

### Initial Staging Deployment

```bash
# 1. Set up DNS or hosts file
# Add to /etc/hosts (or equivalent)
echo "127.0.0.1 staging.myapp.local" | sudo tee -a /etc/hosts

# 2. Ensure nginx-proxy and Step CA are running
docker network create nginx-proxy
docker-compose -f docker-compose.step-ca.yml up -d
docker-compose -f docker-compose.proxy.yml up -d

# 3. Build and start staging environment
docker-compose -f docker-compose.staging.yml build
docker-compose -f docker-compose.staging.yml up -d

# 4. Run migrations
docker-compose -f docker-compose.staging.yml exec web python manage.py migrate

# 5. Collect static files
docker-compose -f docker-compose.staging.yml exec web python manage.py collectstatic --no-input

# 6. Create superuser
docker-compose -f docker-compose.staging.yml exec web python manage.py createsuperuser

# 7. Load test data (optional)
docker-compose -f docker-compose.staging.yml exec web python manage.py loaddata staging_fixtures.json
```

### Deploying Updates to Staging

```bash
# 1. Pull latest code
git pull origin develop  # or your staging branch

# 2. Rebuild and restart
docker-compose -f docker-compose.staging.yml build
docker-compose -f docker-compose.staging.yml up -d

# 3. Run migrations
docker-compose -f docker-compose.staging.yml exec web python manage.py migrate

# 4. Collect static files
docker-compose -f docker-compose.staging.yml exec web python manage.py collectstatic --no-input

# 5. Restart services
docker-compose -f docker-compose.staging.yml restart web
```

### Automated Deployment Script

Create `deploy-staging.sh`:

```bash
#!/bin/bash

set -e  # Exit on error

echo "🚀 Starting staging deployment..."

# Pull latest code
echo "📥 Pulling latest code..."
git pull origin develop

# Build images
echo "🔨 Building Docker images..."
docker-compose -f docker-compose.staging.yml build

# Stop old containers
echo "🛑 Stopping old containers..."
docker-compose -f docker-compose.staging.yml down

# Start new containers
echo "▶️  Starting new containers..."
docker-compose -f docker-compose.staging.yml up -d

# Wait for database
echo "⏳ Waiting for database..."
sleep 10

# Run migrations
echo "🔄 Running migrations..."
docker-compose -f docker-compose.staging.yml exec -T web python manage.py migrate --no-input

# Collect static files
echo "📦 Collecting static files..."
docker-compose -f docker-compose.staging.yml exec -T web python manage.py collectstatic --no-input

# Health check
echo "🏥 Running health check..."
sleep 5
curl -f https://staging.myapp.local/health/ || echo "⚠️  Health check failed"

echo "✅ Staging deployment complete!"
echo "🌐 Access at: https://staging.myapp.local"
```

Make it executable:
```bash
chmod +x deploy-staging.sh
```

## Testing in Staging

### 1. Smoke Tests

```bash
# Check if application is responding
curl -I https://staging.myapp.local

# Check health endpoint
curl https://staging.myapp.local/health/

# Check admin is accessible
curl -I https://staging.myapp.local/admin/
```

### 2. Database Tests

```bash
# Check database connectivity
docker-compose -f docker-compose.staging.yml exec web python manage.py dbshell

# Run Django checks
docker-compose -f docker-compose.staging.yml exec web python manage.py check
```

### 3. Run Test Suite

```bash
# Run all tests
docker-compose -f docker-compose.staging.yml exec web python manage.py test

# Run specific tests
docker-compose -f docker-compose.staging.yml exec web python manage.py test app.tests.test_views
```

### 4. Load Testing

```bash
# Use Apache Bench
ab -n 1000 -c 10 https://staging.myapp.local/

# Or use wrk
wrk -t12 -c400 -d30s https://staging.myapp.local/
```

### 5. Manual Testing Checklist

- [ ] User registration and login
- [ ] Password reset flow
- [ ] Main application features
- [ ] File uploads
- [ ] Email sending
- [ ] API endpoints
- [ ] Admin interface
- [ ] Static files loading
- [ ] Media files access
- [ ] SSL certificate validity
- [ ] Error pages (404, 500)

## Common Staging Workflows

### Syncing Production Data to Staging

**⚠️ Important:** Sanitize production data before using in staging!

```bash
# 1. Backup production database
docker-compose -f docker-compose.prod.yml exec db pg_dump -U django_user django_prod > prod_backup.sql

# 2. Sanitize sensitive data (create sanitize.py)
python sanitize_data.py prod_backup.sql staging_data.sql

# 3. Restore to staging
cat staging_data.sql | docker-compose -f docker-compose.staging.yml exec -T db psql -U django_staging_user -d django_staging

# 4. Run migrations (in case staging is ahead)
docker-compose -f docker-compose.staging.yml exec web python manage.py migrate
```

### Testing Migrations

```bash
# 1. Create migration
docker-compose -f docker-compose.staging.yml exec web python manage.py makemigrations

# 2. Test migration forward
docker-compose -f docker-compose.staging.yml exec web python manage.py migrate

# 3. Test migration backward (if possible)
docker-compose -f docker-compose.staging.yml exec web python manage.py migrate app_name 0003_previous_migration

# 4. Test forward again
docker-compose -f docker-compose.staging.yml exec web python manage.py migrate
```

### Feature Branch Testing

```bash
# 1. Create feature branch
git checkout -b feature/new-feature

# 2. Make changes and commit
git add .
git commit -m "Add new feature"

# 3. Deploy to staging
./deploy-staging.sh

# 4. Test the feature

# 5. If approved, merge to main
git checkout main
git merge feature/new-feature
```

## CI/CD Integration

### GitHub Actions Example

```yaml
# .github/workflows/staging.yml
name: Deploy to Staging

on:
  push:
    branches:
      - develop

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Deploy to staging server
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.STAGING_HOST }}
        username: ${{ secrets.STAGING_USER }}
        key: ${{ secrets.STAGING_SSH_KEY }}
        script: |
          cd /path/to/app
          ./deploy-staging.sh
    
    - name: Run tests
      run: |
        ssh user@staging-server "cd /path/to/app && docker-compose -f docker-compose.staging.yml exec -T web python manage.py test"
    
    - name: Notify team
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        text: 'Staging deployment completed'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

## Django Settings for Staging

Update `settings.py` to handle staging environment:

```python
# settings.py
import os

# Environment detection
ENVIRONMENT = os.environ.get('ENVIRONMENT', 'development')

# Debug
DEBUG = int(os.environ.get('DEBUG', 0))

# Security settings (same as production in staging)
if ENVIRONMENT in ['staging', 'production']:
    SECURE_SSL_REDIRECT = True
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
    SECURE_BROWSER_XSS_FILTER = True
    SECURE_CONTENT_TYPE_NOSNIFF = True
    SECURE_HSTS_SECONDS = 31536000
    SECURE_HSTS_INCLUDE_SUBDOMAINS = True
    SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# Logging - more verbose in staging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
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
            'filename': f'/var/log/django/{ENVIRONMENT}.log',
            'maxBytes': 1024 * 1024 * 15,  # 15MB
            'backupCount': 10,
            'formatter': 'verbose',
        },
    },
    'root': {
        'handlers': ['console', 'file'],
        'level': 'DEBUG' if ENVIRONMENT == 'staging' else 'INFO',
    },
}

# Email - use different backend for staging
if ENVIRONMENT == 'staging':
    EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
    # Configure with test email service
elif ENVIRONMENT == 'development':
    EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'

# Sentry - different DSN for staging
if ENVIRONMENT in ['staging', 'production']:
    import sentry_sdk
    from sentry_sdk.integrations.django import DjangoIntegration
    
    sentry_sdk.init(
        dsn=os.environ.get('SENTRY_DSN'),
        integrations=[DjangoIntegration()],
        environment=ENVIRONMENT,
        traces_sample_rate=0.1 if ENVIRONMENT == 'staging' else 0.01,
    )
```

## Monitoring Staging

### View Logs

```bash
# All services
docker-compose -f docker-compose.staging.yml logs -f

# Web service only
docker-compose -f docker-compose.staging.yml logs -f web

# Last 100 lines
docker-compose -f docker-compose.staging.yml logs --tail=100 web
```

### Monitor Resources

```bash
# Container stats
docker stats

# Specific staging containers
docker stats $(docker-compose -f docker-compose.staging.yml ps -q)
```

### Database Monitoring

```bash
# Connect to database
docker-compose -f docker-compose.staging.yml exec db psql -U django_staging_user -d django_staging

# Check database size
docker-compose -f docker-compose.staging.yml exec db psql -U django_staging_user -d django_staging -c "SELECT pg_size_pretty(pg_database_size('django_staging'));"

# Check active connections
docker-compose -f docker-compose.staging.yml exec db psql -U django_staging_user -d django_staging -c "SELECT count(*) FROM pg_stat_activity;"
```

## Troubleshooting

### Staging Container Won't Start

```bash
# Check logs
docker-compose -f docker-compose.staging.yml logs web

# Check environment variables
docker-compose -f docker-compose.staging.yml config

# Rebuild from scratch
docker-compose -f docker-compose.staging.yml down -v
docker-compose -f docker-compose.staging.yml build --no-cache
docker-compose -f docker-compose.staging.yml up -d
```

### SSL Certificate Issues

```bash
# Check certificate
docker exec nginx-proxy ls -la /etc/nginx/certs/ | grep staging

# Check acme-companion logs
docker logs acme-companion | grep staging

# Restart acme-companion to retry
docker restart acme-companion
```

### Database Connection Issues

```bash
# Check database is running
docker-compose -f docker-compose.staging.yml ps db

# Test connection from web container
docker-compose -f docker-compose.staging.yml exec web nc -zv db 5432

# Check environment variables
docker-compose -f docker-compose.staging.yml exec web env | grep SQL
```

### Staging Shows Production Data

**Problem**: Staging is showing production data or vice versa

**Solution**: Verify environment separation
```bash
# Check which environment is running
docker-compose -f docker-compose.staging.yml exec web python manage.py shell
>>> import os
>>> os.environ.get('ENVIRONMENT')
'staging'

# Verify database name
docker-compose -f docker-compose.staging.yml exec web python manage.py dbshell
\l  # Should show django_staging, not django_prod
```

## Best Practices

### 1. Keep Staging Close to Production

- Use the same Dockerfile
- Use the same Gunicorn configuration
- Use the same nginx-proxy setup
- Use PostgreSQL (not SQLite)
- Enable SSL/HTTPS

### 2. Use Different Secrets

- Never use production secrets in staging
- Use different `SECRET_KEY`
- Use different database passwords
- Use different API keys (when possible)

### 3. Data Management

- Regularly refresh staging data from production (sanitized)
- Keep staging data realistic but not sensitive
- Test data migrations before production

### 4. Access Control

- Limit access to team members and stakeholders
- Use basic auth if needed for extra protection
- Don't expose staging publicly unless necessary

### 5. Monitoring

- Set up error tracking (separate Sentry project)
- Monitor performance
- Track deployment frequency
- Log all deployments

### 6. Automation

- Automate deployments from develop branch
- Run automated tests after deployment
- Send notifications on deployment
- Create rollback procedures

## Next Steps

- [Production Deployment](production-deployment.md) - Deploy to production
- [SSL/HTTPS Setup](ssl-setup.md) - Configure SSL certificates
- [Best Practices](best-practices.md) - Security and optimization
- [Troubleshooting](troubleshooting.md) - Common issues and solutions
