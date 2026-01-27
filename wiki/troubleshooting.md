# Troubleshooting Guide

Common issues and solutions for Django Docker deployments.

## Table of Contents

- [Docker Issues](#docker-issues)
- [Database Issues](#database-issues)
- [Static Files Issues](#static-files-issues)
- [Nginx Issues](#nginx-issues)
- [SSL/HTTPS Issues](#sslhttps-issues)
- [Performance Issues](#performance-issues)
- [Migration Issues](#migration-issues)
- [Connection Issues](#connection-issues)
- [Permission Issues](#permission-issues)

## Docker Issues

### Container Won't Start

**Symptoms**: Container exits immediately or fails to start

**Diagnosis**:
```bash
# Check container status
docker-compose ps

# View logs
docker-compose logs web
docker-compose logs db

# Check specific container
docker logs <container_id>
```

**Common Solutions**:

1. **Port already in use**:
```bash
# Find process using port
sudo lsof -i :8000
# Or
sudo netstat -tulpn | grep 8000

# Kill process
sudo kill -9 <PID>

# Or change port in docker-compose.yml
ports:
  - 8001:8000
```

2. **Insufficient resources**:
```bash
# Check Docker resources
docker system df

# Clean up
docker system prune -a
docker volume prune
```

3. **Invalid configuration**:
```bash
# Validate docker-compose file
docker-compose config

# Rebuild from scratch
docker-compose down -v
docker-compose up -d --build
```

### Build Failures

**Symptoms**: Docker build fails

**Solutions**:

1. **Check Dockerfile syntax**:
```bash
# Validate Dockerfile
docker build --no-cache -t test .
```

2. **Network issues during build**:
```bash
# Use different DNS
docker build --dns 8.8.8.8 -t test .

# Or configure Docker daemon
# /etc/docker/daemon.json
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```

3. **Dependency conflicts**:
```bash
# Clear pip cache
docker-compose build --no-cache

# Update requirements
pip freeze > requirements.txt
```

### Container Keeps Restarting

**Symptoms**: Container starts then immediately restarts

**Diagnosis**:
```bash
# Check restart count
docker ps -a

# View last 100 lines of logs
docker-compose logs --tail=100 web

# Check exit code
docker inspect <container_id> | grep -A 5 "State"
```

**Solutions**:

1. **Application error**:
```bash
# Run container interactively
docker-compose run --rm web /bin/bash

# Test command manually
python manage.py runserver
```

2. **Missing environment variables**:
```bash
# Check environment variables
docker-compose exec web env

# Verify .env file is loaded
docker-compose config
```

### Volume Permission Issues

**Symptoms**: Permission denied errors

**Solutions**:
```bash
# Fix ownership (Linux)
sudo chown -R $USER:$USER .

# Or change file permissions
chmod -R 755 .

# Check volume mounts
docker-compose exec web ls -la /usr/src/app
```

## Database Issues

### Connection Refused

**Symptoms**: `django.db.utils.OperationalError: could not connect to server`

**Solutions**:

1. **Database not ready**:
```bash
# Check if database is running
docker-compose logs db

# Wait for database in entrypoint.sh
while ! nc -z $SQL_HOST $SQL_PORT; do
  sleep 0.1
done
```

2. **Wrong credentials**:
```bash
# Verify environment variables
docker-compose exec web env | grep SQL

# Check database container
docker-compose exec db psql -U $SQL_USER -d $SQL_DATABASE
```

3. **Wrong host**:
```env
# In .env, use service name as host
SQL_HOST=db  # Not localhost or 127.0.0.1
```

### Database Migrations Fail

**Symptoms**: Migration errors or conflicts

**Solutions**:

1. **Reset migrations** (⚠️ data loss):
```bash
# Delete migration files (except __init__.py)
find . -path "*/migrations/*.py" -not -name "__init__.py" -delete
find . -path "*/migrations/*.pyc" -delete

# Recreate migrations
docker-compose exec web python manage.py makemigrations
docker-compose exec web python manage.py migrate
```

2. **Fake migrations** (if database already has tables):
```bash
docker-compose exec web python manage.py migrate --fake-initial
```

3. **Specific app migration**:
```bash
# Migrate specific app
docker-compose exec web python manage.py migrate app_name

# Show migration status
docker-compose exec web python manage.py showmigrations
```

### Database Locked

**Symptoms**: `database is locked` error (SQLite)

**Solution**: Use PostgreSQL in production (not SQLite)

### Can't Connect to PostgreSQL

**Symptoms**: Connection timeout or refused

**Solutions**:
```bash
# Check if PostgreSQL is listening
docker-compose exec db netstat -tulpn | grep 5432

# Check PostgreSQL logs
docker-compose logs db

# Test connection from web container
docker-compose exec web nc -zv db 5432

# Reset database container
docker-compose down db
docker-compose up -d db
```

## Static Files Issues

### Static Files Not Loading

**Symptoms**: CSS/JS not loading, 404 errors

**Solutions**:

1. **Collect static files**:
```bash
docker-compose exec web python manage.py collectstatic --no-input
```

2. **Check Nginx configuration**:
```bash
# Test Nginx config
docker-compose exec nginx nginx -t

# Check file permissions
docker-compose exec nginx ls -la /home/app/web/staticfiles/
```

3. **Verify Django settings**:
```python
# settings.py
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

STATICFILES_DIRS = [
    BASE_DIR / 'static',
]
```

4. **Check volume mounts**:
```yaml
# docker-compose.yml
volumes:
  - static_volume:/home/app/web/staticfiles
```

### Static Files Show 403 Forbidden

**Symptoms**: 403 error when accessing static files

**Solutions**:
```bash
# Fix permissions
docker-compose exec web chmod -R 755 /home/app/web/staticfiles/

# Check Nginx user permissions
docker-compose exec nginx whoami
docker-compose exec nginx ls -la /home/app/web/staticfiles/
```

### Static Files Not Updating

**Symptoms**: Old CSS/JS still served after changes

**Solutions**:
```bash
# Clear browser cache
# Or use hard refresh (Ctrl+Shift+R)

# Collect static files again
docker-compose exec web python manage.py collectstatic --clear --no-input

# Restart Nginx
docker-compose restart nginx

# Add cache busting
# settings.py
STATIC_URL = '/static/'
if not DEBUG:
    STATICFILES_STORAGE = 'django.contrib.staticfiles.storage.ManifestStaticFilesStorage'
```

## Nginx Issues

### 502 Bad Gateway

**Symptoms**: Nginx shows 502 error

**Solutions**:

1. **Check if Django is running**:
```bash
docker-compose ps
docker-compose logs web
```

2. **Verify upstream configuration**:
```nginx
# nginx.conf
upstream django {
    server web:8000;  # Use service name, not localhost
}
```

3. **Check proxy settings**:
```nginx
location / {
    proxy_pass http://django;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

4. **Increase timeouts**:
```nginx
# nginx.conf
proxy_connect_timeout 600;
proxy_send_timeout 600;
proxy_read_timeout 600;
send_timeout 600;
```

### 413 Request Entity Too Large

**Symptoms**: Large file upload fails

**Solution**:
```nginx
# nginx.conf
client_max_body_size 100M;
```

### Nginx Configuration Test Fails

**Symptoms**: `nginx -t` shows errors

**Solutions**:
```bash
# Test configuration
docker-compose exec nginx nginx -t

# Check syntax
docker-compose exec nginx cat /etc/nginx/conf.d/nginx.conf

# View error details
docker-compose logs nginx
```

### Nginx Won't Start

**Symptoms**: Nginx container exits

**Solutions**:
```bash
# Check logs
docker-compose logs nginx

# Test configuration
docker-compose run --rm nginx nginx -t

# Check file paths
docker-compose exec nginx ls -la /etc/nginx/conf.d/

# Rebuild Nginx
docker-compose up -d --build nginx
```

## nginx-proxy and Step CA Issues

### nginx-proxy Not Detecting Container

**Symptoms**: nginx-proxy doesn't create configuration for Django container

**Solutions**:
```bash
# Check containers are on same network
docker network inspect nginx-proxy

# Verify environment variables
docker inspect web | grep -E "VIRTUAL_HOST|VIRTUAL_PORT"

# Check nginx-proxy logs
docker logs nginx-proxy

# Restart nginx-proxy to trigger detection
docker restart nginx-proxy

# Verify nginx configuration was created
docker exec nginx-proxy cat /etc/nginx/conf.d/default.conf
```

### Step CA Connection Issues

**Symptoms**: acme-companion can't connect to Step CA

**Solutions**:
```bash
# Verify Step CA is running
docker ps | grep step-ca

# Check Step CA health
docker exec step-ca step ca health

# Test connection from acme-companion
docker exec acme-companion curl -k https://step-ca:9000/health

# Verify ACME endpoint
docker exec step-ca curl -k https://localhost:9000/acme/acme/directory

# Check network connectivity
docker exec acme-companion ping step-ca
```

### Certificate Not Issued by acme-companion

**Symptoms**: No certificate generated for domain

**Solutions**:
```bash
# Check acme-companion logs
docker logs acme-companion -f

# Verify environment variables on web container
docker exec web env | grep -E "LETSENCRYPT_HOST|LETSENCRYPT_EMAIL"

# Check root certificate is accessible
docker exec acme-companion ls -la /etc/nginx/certs/step-ca-root.crt

# Verify ACME_CA_URI is set correctly
docker exec acme-companion env | grep ACME_CA_URI

# Restart acme-companion to retry
docker restart acme-companion
```

### Root Certificate Trust Issues

**Symptoms**: Browser shows certificate warning despite certificate being issued

**Solutions**:
```bash
# Verify root certificate is correct
docker exec step-ca step ca root

# Export and install root certificate
docker exec step-ca step ca root > step-ca-root.crt

# Install on Linux
sudo cp step-ca-root.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# Install on macOS
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain step-ca-root.crt

# Install on Windows
certutil -addstore -f "ROOT" step-ca-root.crt
```

## SSL/HTTPS Issues

### Mixed Content Warnings

**Symptoms**: Browser shows "Mixed Content" errors

**Solutions**:

1. **Update Django settings**:
```python
# settings.py
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

2. **Update URLs in templates**:
```html
<!-- Use protocol-relative URLs -->
<script src="//cdn.example.com/script.js"></script>

<!-- Or explicitly HTTPS -->
<script src="https://cdn.example.com/script.js"></script>
```

3. **Check external resources**:
```bash
# Find HTTP URLs in templates
grep -r "http://" templates/
```

### Certificate Not Found

**Symptoms**: nginx-proxy can't find SSL certificate

**Solutions**:
```bash
# Check certificate path (with nginx-proxy and acme-companion)
docker exec nginx-proxy ls -la /etc/nginx/certs/

# Check for your domain certificate
docker exec nginx-proxy ls -la /etc/nginx/certs/your_domain.com.*

# View acme-companion logs
docker logs acme-companion

# Verify Step CA is accessible
docker exec acme-companion curl -k https://step-ca:9000/health

# Check root certificate is present
docker exec nginx-proxy ls -la /etc/nginx/certs/step-ca-root.crt

# Restart acme-companion to trigger certificate request
docker restart acme-companion
```

### Certificate Expired

**Symptoms**: Browser shows expired certificate warning

**Solutions**:
```bash
# Check certificate expiry (with nginx-proxy)
docker exec nginx-proxy openssl x509 -in /etc/nginx/certs/your_domain.com.crt -noout -dates

# acme-companion automatically renews certificates
# Check renewal logs
docker logs acme-companion

# Force renewal by restarting acme-companion
docker restart acme-companion

# Wait for renewal (30-60 seconds)
sleep 60

# nginx-proxy automatically reloads when certificates change
```

## Performance Issues

### Slow Response Times

**Symptoms**: Application responds slowly

**Diagnosis**:
```bash
# Check container resources
docker stats

# Check Django debug toolbar
# Install and enable django-debug-toolbar

# Profile specific view
# Use django-silk or django-extensions
```

**Solutions**:

1. **Increase Gunicorn workers**:
```python
# gunicorn_config.py
import multiprocessing
workers = multiprocessing.cpu_count() * 2 + 1
```

2. **Enable caching**:
```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://redis:6379/1',
    }
}
```

3. **Optimize database queries**:
```python
# Use select_related and prefetch_related
queryset = Model.objects.select_related('related_model').all()
```

4. **Add database indexes**:
```python
class MyModel(models.Model):
    field = models.CharField(max_length=100, db_index=True)
```

### High Memory Usage

**Symptoms**: Container using too much memory

**Solutions**:
```bash
# Check memory usage
docker stats

# Limit container memory
# docker-compose.yml
services:
  web:
    mem_limit: 512m
    memswap_limit: 1g

# Reduce Gunicorn workers
# gunicorn_config.py
workers = 2
```

### High CPU Usage

**Symptoms**: Container consuming high CPU

**Solutions**:
```bash
# Check CPU usage
docker stats

# Profile application
docker-compose exec web python -m cProfile manage.py runserver

# Check for infinite loops or heavy processing
# Use django-debug-toolbar
```

## Migration Issues

### Migration Conflicts

**Symptoms**: `Conflicting migrations detected`

**Solutions**:
```bash
# Show migration tree
docker-compose exec web python manage.py showmigrations

# Merge migrations
docker-compose exec web python manage.py makemigrations --merge

# Or manually edit migration files
```

### Can't Reverse Migration

**Symptoms**: Migration fails to reverse

**Solutions**:
```bash
# Check migration dependencies
docker-compose exec web python manage.py showmigrations app_name

# Fake migration
docker-compose exec web python manage.py migrate app_name 0001 --fake

# Manually reverse in database (⚠️ careful)
docker-compose exec db psql -U django_user -d django_prod
```

## Connection Issues

### Can't Access Application from Browser

**Symptoms**: Browser can't connect

**Solutions**:

1. **Check if containers are running**:
```bash
docker-compose ps
```

2. **Check port mappings**:
```bash
docker-compose port web 8000
```

3. **Check firewall**:
```bash
# Linux
sudo ufw status
sudo ufw allow 8000

# Or iptables
sudo iptables -L -n
```

4. **Check if bound to correct interface**:
```python
# Use 0.0.0.0, not 127.0.0.1
python manage.py runserver 0.0.0.0:8000
```

### Connection Timeout

**Symptoms**: Request times out

**Solutions**:
```bash
# Increase timeouts
# gunicorn_config.py
timeout = 120

# nginx.conf
proxy_connect_timeout 120;
proxy_read_timeout 120;
proxy_send_timeout 120;
```

## Permission Issues

### Permission Denied in Container

**Symptoms**: Can't write files, permission errors

**Solutions**:

1. **Run as non-root user** (production):
```dockerfile
# Dockerfile.prod
RUN addgroup --system app && adduser --system --group app
USER app
```

2. **Fix volume permissions** (development):
```bash
sudo chown -R $USER:$USER .
```

3. **Check container user**:
```bash
docker-compose exec web whoami
docker-compose exec web id
```

### Can't Write to Media/Static Directories

**Symptoms**: Permission denied when uploading files

**Solutions**:
```bash
# Create directories with correct permissions
docker-compose exec web mkdir -p /home/app/web/mediafiles
docker-compose exec web chmod 755 /home/app/web/mediafiles

# Or in Dockerfile
RUN mkdir -p $APP_HOME/mediafiles && chmod 755 $APP_HOME/mediafiles
```

## Logging and Debugging

### Enable Detailed Logging

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
        },
        'file': {
            'class': 'logging.FileHandler',
            'filename': 'debug.log',
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
    },
}
```

### View Container Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f web

# Last 100 lines
docker-compose logs --tail=100 web

# Since timestamp
docker-compose logs --since 2024-01-01T00:00:00 web
```

### Debug in Container

```bash
# Access shell
docker-compose exec web /bin/bash

# Run Django shell
docker-compose exec web python manage.py shell

# Run dbshell
docker-compose exec web python manage.py dbshell

# Install debugging tools
pip install ipdb
# Then use: import ipdb; ipdb.set_trace()
```

## General Debugging Workflow

1. **Check container status**:
```bash
docker-compose ps
```

2. **View logs**:
```bash
docker-compose logs -f
```

3. **Test configuration**:
```bash
docker-compose config
docker-compose exec nginx nginx -t
```

4. **Verify environment variables**:
```bash
docker-compose exec web env
```

5. **Access container**:
```bash
docker-compose exec web /bin/bash
```

6. **Test manually**:
```bash
docker-compose exec web python manage.py check
docker-compose exec web python manage.py test
```

7. **Rebuild if needed**:
```bash
docker-compose down
docker-compose up -d --build
```

## Getting Help

If you can't solve the issue:

1. **Check logs thoroughly**
2. **Search GitHub issues**
3. **Stack Overflow** with relevant tags (docker, django, nginx)
4. **Django Documentation**: https://docs.djangoproject.com/
5. **Docker Documentation**: https://docs.docker.com/
6. **PostgreSQL Documentation**: https://www.postgresql.org/docs/

## Common Error Messages

| Error | Likely Cause | Solution |
|-------|--------------|----------|
| `Connection refused` | Service not running | Check `docker-compose ps`, view logs |
| `Permission denied` | File/directory permissions | Fix with `chown` or `chmod` |
| `Port already in use` | Port conflict | Kill process or change port |
| `Command not found` | Missing dependency | Add to requirements.txt, rebuild |
| `No such file or directory` | Wrong path or missing file | Check paths, volume mounts |
| `502 Bad Gateway` | Upstream not responding | Check Django/Gunicorn is running |
| `Database locked` | Using SQLite | Switch to PostgreSQL |
| `Migration conflicts` | Conflicting migrations | Merge migrations |

## Next Steps

- [Best Practices](best-practices.md)
- [Development Setup](development-setup.md)
- [Production Deployment](production-deployment.md)
