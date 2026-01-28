# Production Deployment Guide: Gunicorn and Nginx

## Introduction

This guide covers deploying a Django application to production using Docker, Gunicorn, and Nginx. In production, we use a multi-stage Docker build to optimize image size, run the application behind Gunicorn (a production-grade WSGI server), and serve it through Nginx as a reverse proxy to handle static files and improve performance.

The production setup differs from development in several key ways:
- **Gunicorn** replaces Django's development server for better performance and stability
- **Nginx** acts as a reverse proxy, handling static/media files and forwarding requests to Gunicorn
- **Multi-stage Docker builds** reduce final image size by separating build dependencies from runtime
- **Non-root user** improves security by running the application with minimal privileges
- **Static file collection** is automated through Django's `collectstatic` command

This guide will walk you through creating production Dockerfiles, configuring Nginx, setting up environment variables, and deploying your application.

---

## Step 14: Production Dockerfile

Create `hello_django/Dockerfile.prod` for production:

```dockerfile
# Pull official base image
FROM python:3.12.12-slim-trixie AS builder

# Set work directory
WORKDIR /usr/src/app

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# install system dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc

# Lint
RUN pip install --upgrade pip
RUN pip install flake8==7.0.0
COPY . .
RUN flake8 --ignore=E501,F401 .

# Install dependencies
COPY ./requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /usr/src/app/wheels -r requirements.txt

# Final stage
FROM python:3.12.12-slim-trixie

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

# install system dependencies
RUN apt-get update \
    && apt-get install -y --no-install-recommends netcat-openbsd \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean \
    && apt-get autoremove -y

COPY --from=builder /usr/src/app/wheels /wheels
COPY --from=builder /usr/src/app/requirements.txt .
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

**Key Features:**
- **Multi-stage build**: Builder stage installs dependencies, final stage only includes runtime requirements
- **Linting**: Runs flake8 during build to catch code issues early
- **Non-root user**: Creates and uses `app` user for improved security
- **Optimized layers**: Minimizes image size by cleaning up apt cache

---

## Step 15: Production Entrypoint Script

Create `hello_django/entrypoint.prod.sh`:

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
# python manage.py migrate --noinput
python manage.py collectstatic --no-input --clear
exec "$@"
```

Update permissions:
```bash
chmod +x hello_django/entrypoint.prod.sh
```

**What it does:**
- Waits for PostgreSQL to be ready before starting the application
- Collects static files automatically on container startup
- Migration command is commented out for manual control (can be uncommented for auto-migrations)

---

## Step 16: Production Environment Variables

Create `.env.prod` in the main directory:

```env
DEBUG=0
SECRET_KEY=your_production_secret_key_change_this
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_prod
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres
```

Create `.env.prod.db` for production database:

```env
POSTGRES_USER=hello_django
POSTGRES_PASSWORD=hello_django
POSTGRES_DB=hello_django_prod
```

**Important:**
- **DEBUG=0**: Disables debug mode for security
- **SECRET_KEY**: Use a strong, unique key for production (never commit this to version control)
- **DJANGO_ALLOWED_HOSTS**: Update with your actual domain names
- **Database credentials**: Use strong passwords in actual production

---

## Step 17: Production Docker Compose

Create `docker-compose.prod.yml`:

```yaml
services:
  web:
    build:
      context: ./hello_django
      dockerfile: Dockerfile.prod
    command: gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000
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
    image: postgres:18.1-trixie
    volumes:
      - postgres_data:/var/lib/postgresql
    env_file:
      - ./.env.prod.db
    
    healthcheck: # (optional) healthcheck
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 5s
      timeout: 5s
      retries: 10
    
  
  nginx:
    build: ./nginx
    volumes:
      - static_volume:/home/app/web/staticfiles
      - media_volume:/home/app/web/mediafiles
    ports:
      - 1337:80
    depends_on:
      - web

volumes:
  postgres_data:
  static_volume:
  media_volume:
```

**Configuration Details:**
- **web service**: Uses Gunicorn to serve the Django application
- **db service**: PostgreSQL with health checks
- **nginx service**: Reverse proxy accessible on port 1337
- **Named volumes**: Persist database data and static/media files

---

## Step 18: Nginx Configuration

Create `nginx/Dockerfile`:

```dockerfile
FROM nginx:1.27-alpine

RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d
```

Create `nginx/nginx.conf`:

```nginx
upstream hello_django {
    server web:8000;
}

server {
    listen 80;

    location / {
        proxy_pass http://hello_django;
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
}
```

**Nginx Configuration Explained:**
- **upstream**: Defines the Gunicorn backend server
- **location /**: Proxies all requests to Gunicorn
- **location /static/**: Serves static files directly (CSS, JS, images)
- **location /media/**: Serves user-uploaded media files
- **Proxy headers**: Preserves client information for Django

---

## Step 19: Update Django Settings for Production

Update `hello_django/hello_django/settings.py` to handle static and media files:

```python
# Static files (CSS, JavaScript, Images)
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

# Media files
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'mediafiles'
```

**Settings Explanation:**
- **STATIC_URL**: URL prefix for static files
- **STATIC_ROOT**: Directory where `collectstatic` gathers all static files
- **MEDIA_URL**: URL prefix for user-uploaded files
- **MEDIA_ROOT**: Directory for storing uploaded media files

---

## Step 20: Build and Run Production

Build and run the production stack:

```bash
docker-compose -f docker-compose.prod.yml up -d --build
```

Here are two useful manual migrations and collect static files:

```bash
docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input --clear
```

Create superuser for the first time on production:

```bash
docker-compose -f docker-compose.prod.yml exec web python manage.py createsuperuser
```

Access the application at `http://localhost:1337`

---

## Step 21: Final Project Structure

```
DjangoDockerTemplate/
├── hello_django/              # Renamed from app/ to hello_django/
│   ├── hello_django/
│   │   ├── __init__.py
│   │   ├── asgi.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   ├── manage.py
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── Dockerfile.prod
│   ├── entrypoint.sh
│   └── entrypoint.prod.sh
├── nginx/
│   ├── Dockerfile
│   ├── nginx.tmpl              # Template for nginx-proxy auto-config
│   ├── custom.conf             # Custom proxy-wide configuration
│   └── vhost.d/
│       └── default             # Static/media files configuration
├── .env.dev
├── .env.staging                # Staging environment variables
├── .env.staging.db            # Staging database variables
├── .env.prod
├── .env.prod.db
├── docker-compose.yml         # Development
├── docker-compose.staging.yml # Staging with Step CA + nginx-proxy
├── docker-compose.prod.yml    # Production with Step CA + nginx-proxy
├── .gitignore
└── README.md
```

---

## Production Commands Reference

### Starting and Stopping

**Build and start production containers:**
```bash
docker-compose -f docker-compose.prod.yml up -d --build
```

**Start without rebuilding:**
```bash
docker-compose -f docker-compose.prod.yml up -d
```

**Stop production containers:**
```bash
docker-compose -f docker-compose.prod.yml down
```

**Stop and remove volumes (WARNING: deletes database data):**
```bash
docker-compose -f docker-compose.prod.yml down -v
```

### Monitoring and Debugging

**View logs:**
```bash
docker-compose -f docker-compose.prod.yml logs -f
```

**View logs for specific service:**
```bash
docker-compose -f docker-compose.prod.yml logs -f web
docker-compose -f docker-compose.prod.yml logs -f nginx
docker-compose -f docker-compose.prod.yml logs -f db
```

**Check container status:**
```bash
docker-compose -f docker-compose.prod.yml ps
```

### Database Management

**Run migrations:**
```bash
docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
```

**Create superuser:**
```bash
docker-compose -f docker-compose.prod.yml exec web python manage.py createsuperuser
```

**Access Django shell:**
```bash
docker-compose -f docker-compose.prod.yml exec web python manage.py shell
```

**Access database shell:**
```bash
docker-compose -f docker-compose.prod.yml exec db psql -U hello_django -d hello_django_prod
```

### Static and Media Files

**Collect static files:**
```bash
docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input --clear
```

**View static files:**
```bash
docker-compose -f docker-compose.prod.yml exec web ls -la /home/app/web/staticfiles
```

### Rebuilding and Updates

**Rebuild after code changes:**
```bash
docker-compose -f docker-compose.prod.yml up -d --build
```

**Rebuild specific service:**
```bash
docker-compose -f docker-compose.prod.yml up -d --build web
```

**Pull latest images:**
```bash
docker-compose -f docker-compose.prod.yml pull
```

### Backup and Restore

**Backup database:**
```bash
docker-compose -f docker-compose.prod.yml exec db pg_dump -U hello_django hello_django_prod > backup.sql
```

**Restore database:**
```bash
docker-compose -f docker-compose.prod.yml exec -T db psql -U hello_django hello_django_prod < backup.sql
```

---

## Production Checklist

Before deploying to production, ensure:

- [ ] `DEBUG=0` in `.env.prod`
- [ ] Strong `SECRET_KEY` generated and set
- [ ] `DJANGO_ALLOWED_HOSTS` configured with actual domain names
- [ ] Strong database passwords set
- [ ] Static files directory is writable
- [ ] Media files directory is writable
- [ ] Database migrations are up to date
- [ ] SSL/TLS certificates configured (if using HTTPS)
- [ ] Firewall rules configured
- [ ] Regular backup strategy in place
- [ ] Monitoring and logging configured
- [ ] Environment files (`.env.prod`, `.env.prod.db`) are not committed to version control

---

## Troubleshooting

### Static Files Not Loading

1. Ensure `collectstatic` has been run:
   ```bash
   docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input --clear
   ```

2. Check Nginx configuration for correct static file paths

3. Verify volume mounts in `docker-compose.prod.yml`

### Database Connection Issues

1. Check database is running:
   ```bash
   docker-compose -f docker-compose.prod.yml ps db
   ```

2. Verify environment variables in `.env.prod` and `.env.prod.db` match

3. Check database logs:
   ```bash
   docker-compose -f docker-compose.prod.yml logs db
   ```

### Application Not Accessible

1. Check all containers are running:
   ```bash
   docker-compose -f docker-compose.prod.yml ps
   ```

2. Verify port mapping (default: http://localhost:1337)

3. Check Nginx logs:
   ```bash
   docker-compose -f docker-compose.prod.yml logs nginx
   ```

4. Check web application logs:
   ```bash
   docker-compose -f docker-compose.prod.yml logs web
   ```

---

## Next Steps

- Set up SSL/TLS certificates for HTTPS
- Configure automated backups
- Set up monitoring and alerting
- Implement CI/CD pipeline
- Review the main README for advanced topics like Step CA and nginx-proxy integration

For development setup, refer to the Development Guide in the `docs/` directory.
