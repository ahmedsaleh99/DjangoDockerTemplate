# Production Deployment Guide

This guide covers deploying Django to production with Docker, PostgreSQL, Gunicorn, and automated SSL using nginx-proxy and acme-companion with Step CA.

**Note:** This guide uses nginx-proxy for automatic Nginx configuration. For manual Nginx setup, refer to the traditional approach in the archived documentation.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Production Architecture](#production-architecture)
- [Production Dockerfile](#production-dockerfile)
- [Gunicorn Configuration](#gunicorn-configuration)
- [Nginx Configuration](#nginx-configuration)
- [Docker Compose for Production](#docker-compose-for-production)
- [Static and Media Files](#static-and-media-files)
- [Environment Variables](#environment-variables)
- [Deployment Steps](#deployment-steps)
- [Health Checks and Monitoring](#health-checks-and-monitoring)

## Prerequisites

- A Linux server (Ubuntu 20.04+ recommended) or local development environment
- Domain name or hostname (can be local like `myapp.local` when using Step CA)
- Docker and Docker Compose installed
- Basic knowledge of Linux command line
- SSH access to your server (for remote deployment)

**Note:** With Step CA, you don't need a public domain. See [SSL/HTTPS Setup](ssl-setup.md) for details on using private certificates.

## Production Architecture

```
Internet
    │
    ▼
┌─────────────────────────────────────┐
│   Nginx (Port 80/443)               │
│   - Reverse Proxy                   │
│   - Static/Media File Serving       │
│   - SSL Termination                 │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│   Gunicorn (Port 8000)              │
│   - WSGI HTTP Server                │
│   - Django Application              │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│   PostgreSQL (Port 5432)            │
│   - Production Database             │
└─────────────────────────────────────┘
```

## Production Dockerfile

Create `Dockerfile.prod`:

```dockerfile
###########
# BUILDER #
###########

# Pull official base image
FROM python:3.11.4-slim-buster as builder

# Set work directory
WORKDIR /usr/src/app

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Install system dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc

# Lint
RUN pip install --upgrade pip
RUN pip install flake8==6.0.0
COPY . /usr/src/app/
RUN flake8 --ignore=E501,F401 .

# Install Python dependencies
COPY ./requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /usr/src/app/wheels -r requirements.txt


#########
# FINAL #
#########

# Pull official base image
FROM python:3.11.4-slim-buster

# Create directory for the app user
RUN mkdir -p /home/app

# Create the app user
RUN addgroup --system app && adduser --system --group app

# Create the appropriate directories
ENV HOME=/home/app
ENV APP_HOME=/home/app/web
RUN mkdir $APP_HOME
RUN mkdir $APP_HOME/staticfiles
RUN mkdir $APP_HOME/mediafiles
WORKDIR $APP_HOME

# Install dependencies
RUN apt-get update && apt-get install -y --no-install-recommends netcat
COPY --from=builder /usr/src/app/wheels /wheels
COPY --from=builder /usr/src/app/requirements.txt .
RUN pip install --upgrade pip
RUN pip install --no-cache /wheels/*

# Copy entrypoint.prod.sh
COPY ./entrypoint.prod.sh .
RUN sed -i 's/\r$//g'  $APP_HOME/entrypoint.prod.sh
RUN chmod +x  $APP_HOME/entrypoint.prod.sh

# Copy project
COPY . $APP_HOME

# Chown all the files to the app user
RUN chown -R app:app $APP_HOME

# Change to the app user
USER app

# Run entrypoint.prod.sh
ENTRYPOINT ["/home/app/web/entrypoint.prod.sh"]
```

**Key differences from development:**
- Multi-stage build for smaller image size
- Non-root user for security
- Linting step to catch errors
- Optimized for production performance

## Production Entry Point

Create `entrypoint.prod.sh`:

```bash
#!/bin/sh

if [ "$DATABASE" = "postgres" ]
then
    echo "Waiting for postgres..."

    while ! nc -z $SQL_HOST $SQL_PORT; do
      sleep 0.1
    done

    echo "PostgreSQL started"
fi

exec "$@"
```

**Note**: We don't flush and migrate automatically in production for safety.

## Gunicorn Configuration

Update `requirements.txt` to include Gunicorn:

```txt
Django==4.2.7
psycopg2-binary==2.9.9
gunicorn==21.2.0
```

Gunicorn will be run with this command in docker-compose:

```bash
gunicorn config.wsgi:application --bind 0.0.0.0:8000
```

### Recommended Gunicorn Settings

For optimal performance, create `gunicorn_config.py`:

```python
# Gunicorn configuration file
import multiprocessing

max_requests = 1000
max_requests_jitter = 50

log_file = "-"

bind = "0.0.0.0:8000"

worker_class = "sync"
workers = multiprocessing.cpu_count() * 2 + 1
worker_connections = 1000
timeout = 30
keepalive = 2

errorlog = '-'
loglevel = 'info'
accesslog = '-'
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s"'
```

Then run with:

```bash
gunicorn config.wsgi:application --config gunicorn_config.py
```

## Nginx Configuration

### Nginx Dockerfile

Create `nginx/Dockerfile`:

```dockerfile
FROM nginx:1.25

RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d
```

### Nginx Configuration File

Create `nginx/nginx.conf`:

```nginx
upstream django {
    server web:8000;
}

server {
    listen 80;
    server_name your_domain.com www.your_domain.com;

    location / {
        proxy_pass http://django;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
        alias /home/app/web/staticfiles/;
    }

    location /media/ {
        alias /home/app/web/mediafiles/;
    }

    client_max_body_size 100M;
}
```

**Key configurations:**
- `upstream django`: Defines the Gunicorn backend
- `proxy_pass`: Forwards requests to Gunicorn
- `proxy_set_header`: Preserves client information
- `location /static/`: Serves static files directly
- `location /media/`: Serves media files directly
- `client_max_body_size`: Max upload size

## Docker Compose for Production

Create `docker-compose.prod.yml`:

```yaml
version: '3.8'

services:
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
      - ./.env.prod
    depends_on:
      - db
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    env_file:
      - ./.env.prod.db
  nginx:
    build: ./nginx
    volumes:
      - static_volume:/home/app/web/staticfiles
      - media_volume:/home/app/web/mediafiles
    ports:
      - 80:80
      - 443:443
    depends_on:
      - web

volumes:
  postgres_data:
  static_volume:
  media_volume:
```

**Key differences from development:**
- Uses production Dockerfile
- Runs Gunicorn instead of Django dev server
- Adds Nginx service
- Named volumes for static and media files
- Exposes ports through Nginx only

## Static and Media Files

### Update Django Settings

Add to `settings.py`:

```python
# Static files
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

# Media files
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'mediafiles'
```

### Collect Static Files

Before deploying, collect static files:

```bash
docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input
```

## Environment Variables

### Production Django Variables

Create `.env.prod`:

```env
DEBUG=0
SECRET_KEY=your-production-secret-key-here-must-be-different-from-dev
DJANGO_ALLOWED_HOSTS=your_domain.com www.your_domain.com
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=django_prod
SQL_USER=django_user
SQL_PASSWORD=strong_database_password_here
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres
```

### Production Database Variables

Create `.env.prod.db`:

```env
POSTGRES_USER=django_user
POSTGRES_PASSWORD=strong_database_password_here
POSTGRES_DB=django_prod
```

**Security Notes:**
- ⚠️ Never use DEBUG=1 in production
- ⚠️ Use a strong, unique SECRET_KEY
- ⚠️ Use strong database passwords
- ⚠️ Add `.env.prod*` to `.gitignore`
- ⚠️ Limit ALLOWED_HOSTS to your domain only

## Deployment Steps

### 1. Prepare Your Server

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Add user to docker group (optional)
sudo usermod -aG docker $USER
```

### 2. Clone Your Repository

```bash
git clone https://github.com/yourusername/your-repo.git
cd your-repo
```

### 3. Set Up Environment Variables

```bash
# Create production environment files
nano .env.prod
nano .env.prod.db

# Secure the files
chmod 600 .env.prod .env.prod.db
```

### 4. Build and Start Containers

```bash
docker-compose -f docker-compose.prod.yml up -d --build
```

### 5. Run Migrations

```bash
docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
```

### 6. Collect Static Files

```bash
docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input
```

### 7. Create Superuser

```bash
docker-compose -f docker-compose.prod.yml exec web python manage.py createsuperuser
```

### 8. Verify Deployment

Visit your domain in a browser: `http://your_domain.com`

## Database Backups

### Create Backup Script

Create `scripts/backup_db.sh`:

```bash
#!/bin/bash

# Configuration
BACKUP_DIR="/home/backups/postgres"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_$DATE.sql"

# Create backup directory if it doesn't exist
mkdir -p $BACKUP_DIR

# Create backup
docker-compose -f docker-compose.prod.yml exec -T db pg_dump -U django_user django_prod > $BACKUP_FILE

# Compress backup
gzip $BACKUP_FILE

# Delete backups older than 30 days
find $BACKUP_DIR -name "*.gz" -mtime +30 -delete

echo "Backup completed: $BACKUP_FILE.gz"
```

Make it executable:

```bash
chmod +x scripts/backup_db.sh
```

### Schedule Automatic Backups

Add to crontab:

```bash
crontab -e

# Add this line for daily backups at 2 AM
0 2 * * * /path/to/scripts/backup_db.sh >> /var/log/backup.log 2>&1
```

### Restore from Backup

```bash
# Stop the application
docker-compose -f docker-compose.prod.yml down

# Restore database
gunzip -c backup_20240101_020000.sql.gz | docker-compose -f docker-compose.prod.yml exec -T db psql -U django_user -d django_prod

# Start the application
docker-compose -f docker-compose.prod.yml up -d
```

## Health Checks and Monitoring

### Docker Health Checks

Add to `docker-compose.prod.yml`:

```yaml
services:
  web:
    # ... other configurations
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
  
  db:
    # ... other configurations
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U django_user -d django_prod"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### Django Health Check Endpoint

Create a simple health check view:

```python
# views.py
from django.http import JsonResponse
from django.db import connection

def health_check(request):
    try:
        # Check database connection
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        return JsonResponse({"status": "healthy"}, status=200)
    except Exception as e:
        return JsonResponse({"status": "unhealthy", "error": str(e)}, status=500)
```

```python
# urls.py
from django.urls import path
from . import views

urlpatterns = [
    # ... other URLs
    path('health/', views.health_check, name='health_check'),
]
```

### Monitoring with Logs

```bash
# View all logs
docker-compose -f docker-compose.prod.yml logs

# View specific service logs
docker-compose -f docker-compose.prod.yml logs web
docker-compose -f docker-compose.prod.yml logs nginx
docker-compose -f docker-compose.prod.yml logs db

# Follow logs in real-time
docker-compose -f docker-compose.prod.yml logs -f

# View last 100 lines
docker-compose -f docker-compose.prod.yml logs --tail=100
```

## Common Production Tasks

### Updating Your Application

```bash
# Pull latest code
git pull origin main

# Rebuild and restart
docker-compose -f docker-compose.prod.yml up -d --build

# Run migrations
docker-compose -f docker-compose.prod.yml exec web python manage.py migrate

# Collect static files
docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input
```

### Scaling Workers

Edit `docker-compose.prod.yml`:

```yaml
services:
  web:
    # ... other configurations
    scale: 3  # Run 3 instances
```

Or scale dynamically:

```bash
docker-compose -f docker-compose.prod.yml up -d --scale web=3
```

### Viewing Container Stats

```bash
docker stats
```

### Accessing Production Shell

```bash
# Django shell
docker-compose -f docker-compose.prod.yml exec web python manage.py shell

# Container bash
docker-compose -f docker-compose.prod.yml exec web /bin/bash

# Database shell
docker-compose -f docker-compose.prod.yml exec db psql -U django_user -d django_prod
```

## Performance Optimization

### 1. Enable Gzip Compression

Add to `nginx/nginx.conf`:

```nginx
gzip on;
gzip_disable "msie6";
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_types text/plain text/css text/xml text/javascript application/json application/javascript application/xml+rss application/rss+xml font/truetype font/opentype application/vnd.ms-fontobject image/svg+xml;
```

### 2. Browser Caching

Add to `nginx/nginx.conf`:

```nginx
location /static/ {
    alias /home/app/web/staticfiles/;
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

### 3. Database Connection Pooling

Add to `requirements.txt`:

```txt
psycopg2-binary==2.9.9
django-db-connection-pool==1.2.4
```

Update `settings.py`:

```python
DATABASES = {
    'default': {
        'ENGINE': 'dj_db_conn_pool.backends.postgresql',
        # ... other settings
        'POOL_OPTIONS': {
            'POOL_SIZE': 10,
            'MAX_OVERFLOW': 10
        }
    }
}
```

### 4. Redis Caching

Add Redis to `docker-compose.prod.yml`:

```yaml
services:
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  redis_data:
```

Update Django settings:

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://redis:6379/1',
    }
}
```

## Security Checklist

- [ ] DEBUG=0 in production
- [ ] Strong SECRET_KEY
- [ ] Secure database passwords
- [ ] ALLOWED_HOSTS properly configured
- [ ] HTTPS enabled (see SSL/HTTPS guide)
- [ ] Regular security updates
- [ ] Database backups configured
- [ ] Non-root user in containers
- [ ] Environment variables not in version control
- [ ] Rate limiting configured
- [ ] CSRF protection enabled
- [ ] Security headers configured

## Troubleshooting

### Application Won't Start

```bash
# Check logs
docker-compose -f docker-compose.prod.yml logs web

# Check if all containers are running
docker-compose -f docker-compose.prod.yml ps

# Restart services
docker-compose -f docker-compose.prod.yml restart
```

### Static Files Not Loading

```bash
# Collect static files again
docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input

# Check Nginx configuration
docker-compose -f docker-compose.prod.yml exec nginx nginx -t

# Check file permissions
docker-compose -f docker-compose.prod.yml exec web ls -la /home/app/web/staticfiles/
```

### Database Connection Errors

```bash
# Check database is running
docker-compose -f docker-compose.prod.yml logs db

# Verify environment variables
docker-compose -f docker-compose.prod.yml exec web env | grep SQL
```

### High Memory Usage

```bash
# Check container resource usage
docker stats

# Reduce Gunicorn workers
# Edit gunicorn_config.py and reduce workers count
```

## Next Steps

- [SSL/HTTPS Setup with Step CA](ssl-setup.md) - Automated SSL with nginx-proxy and acme-companion
- [Best Practices and Security](best-practices.md)
- [Troubleshooting Guide](troubleshooting.md)
