# Development Setup Guide

This guide walks you through setting up a Django development environment with Docker, PostgreSQL, and all necessary tools.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Initial Setup](#initial-setup)
- [Project Structure](#project-structure)
- [Docker Configuration](#docker-configuration)
- [Django Configuration](#django-configuration)
- [Database Setup](#database-setup)
- [Running the Application](#running-the-application)
- [Common Development Tasks](#common-development-tasks)

## Prerequisites

Before you begin, ensure you have the following installed:

- **Python 3.11** (for local virtual environment)
- [Docker](https://docs.docker.com/get-docker/) (version 20.10+)
- [Docker Compose](https://docs.docker.com/compose/install/) (version 1.29+)
- Git
- A code editor (VS Code, PyCharm, etc.)

Verify your installations:

```bash
python3.11 --version
docker --version
docker-compose --version
git --version
```

## Initial Setup

### 1. Create Virtual Environment

Before setting up Docker, it's recommended to create a Python virtual environment for local development and testing.

**Using venv with Python 3.11:**

```bash
# Check Python version
python3.11 --version

# Create virtual environment
python3.11 -m venv venv

# Activate virtual environment
# On Linux/Mac:
source venv/bin/activate

# On Windows:
venv\Scripts\activate

# Upgrade pip
pip install --upgrade pip
```

Once activated, your terminal prompt should show `(venv)` indicating the virtual environment is active.

**Note**: The virtual environment is optional when using Docker, but useful for:
- Running Django management commands locally
- Using IDE features (autocomplete, linting)
- Testing code without rebuilding Docker containers
- Installing development tools

### 2. Create Django Project

First, let's create a new Django project structure:

```bash
mkdir django_docker_app
cd django_docker_app
```

### 3. Create Project Files

Create the following directory structure:

```
django_docker_app/
├── app/
│   ├── __init__.py
│   └── manage.py
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── .env.dev
```

### 4. Requirements File

Create `requirements.txt`:

```txt
Django==4.2.7
psycopg2-binary==2.9.9
gunicorn==21.2.0
```

**Note**: For production projects, consider using a two-file approach:
- `requirements.in` - Minimal dependencies without versions
- `requirements.txt` - Fully pinned dependencies (use `pip-compile` from `pip-tools`)

This ensures reproducible builds while making it easy to update dependencies.

### 5. Dockerfile for Development

Create `Dockerfile`:

```dockerfile
# Pull official base image
FROM python:3.11.4-slim-buster

# Set work directory
WORKDIR /usr/src/app

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Install system dependencies
RUN apt-get update && apt-get install -y netcat

# Install dependencies
RUN pip install --upgrade pip
COPY ./requirements.txt .
RUN pip install -r requirements.txt

# Copy entrypoint.sh
COPY ./entrypoint.sh .
RUN sed -i 's/\r$//g' /usr/src/app/entrypoint.sh
RUN chmod +x /usr/src/app/entrypoint.sh

# Copy project
COPY . .

# Run entrypoint.sh
ENTRYPOINT ["/usr/src/app/entrypoint.sh"]
```

### 6. Entry Point Script

Create `entrypoint.sh`:

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

python manage.py flush --no-input
python manage.py migrate

exec "$@"
```

This script:
- Waits for PostgreSQL to be ready
- Runs database migrations
- Executes the command passed to the container

### 7. Docker Compose Configuration

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./app:/usr/src/app/
    ports:
      - 8000:8000
    env_file:
      - ./.env.dev
    depends_on:
      - db
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      - POSTGRES_USER=django_user
      - POSTGRES_PASSWORD=django_password
      - POSTGRES_DB=django_dev

volumes:
  postgres_data:
```

### 8. Environment Variables

Create `.env.dev`:

```env
DEBUG=1
SECRET_KEY=your-secret-key-here-change-in-production
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=django_dev
SQL_USER=django_user
SQL_PASSWORD=django_password
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres
```

**Important**: Generate a secure SECRET_KEY:

```bash
python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'
```

## Django Configuration

### 1. Create Django Project

If you haven't created a Django project yet:

```bash
docker-compose run web django-admin startproject config .
```

### 2. Update Django Settings

Edit `app/config/settings.py`:

```python
import os
from pathlib import Path

# Build paths inside the project
BASE_DIR = Path(__file__).resolve().parent.parent

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = os.environ.get("SECRET_KEY")

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = int(os.environ.get("DEBUG", default=0))

ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS").split(" ")

# Application definition
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'config.urls'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

WSGI_APPLICATION = 'config.wsgi.application'

# Database
DATABASES = {
    "default": {
        "ENGINE": os.environ.get("SQL_ENGINE", "django.db.backends.sqlite3"),
        "NAME": os.environ.get("SQL_DATABASE", BASE_DIR / "db.sqlite3"),
        "USER": os.environ.get("SQL_USER", "user"),
        "PASSWORD": os.environ.get("SQL_PASSWORD", "password"),
        "HOST": os.environ.get("SQL_HOST", "localhost"),
        "PORT": os.environ.get("SQL_PORT", "5432"),
    }
}

# Password validation
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]

# Internationalization
LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'UTC'
USE_I18N = True
USE_TZ = True

# Static files (CSS, JavaScript, Images)
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

# Default primary key field type
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
```

## Database Setup

### 1. Build and Start Containers

```bash
docker-compose up -d --build
```

This command:
- Builds the Docker images
- Creates and starts the containers
- Runs in detached mode (background)

### 2. Verify Containers are Running

```bash
docker-compose ps
```

You should see two containers running:
- `web` (Django application)
- `db` (PostgreSQL database)

### 3. Run Migrations

```bash
docker-compose exec web python manage.py migrate
```

### 4. Create Superuser

```bash
docker-compose exec web python manage.py createsuperuser
```

## Running the Application

### Start the Application

```bash
docker-compose up
```

Or in detached mode:

```bash
docker-compose up -d
```

### Access the Application

- **Main site**: http://localhost:8000
- **Admin panel**: http://localhost:8000/admin

### View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f web
docker-compose logs -f db
```

### Stop the Application

```bash
docker-compose down
```

To remove volumes as well (⚠️ this will delete your database):

```bash
docker-compose down -v
```

## Common Development Tasks

### Running Django Management Commands

```bash
# Run any Django management command
docker-compose exec web python manage.py <command>

# Examples:
docker-compose exec web python manage.py makemigrations
docker-compose exec web python manage.py migrate
docker-compose exec web python manage.py createsuperuser
docker-compose exec web python manage.py collectstatic
docker-compose exec web python manage.py shell
```

### Creating a New Django App

```bash
docker-compose exec web python manage.py startapp myapp
```

### Running Tests

```bash
docker-compose exec web python manage.py test
```

### Accessing the Django Shell

```bash
docker-compose exec web python manage.py shell
```

### Accessing the Database Shell

```bash
docker-compose exec db psql --username=django_user --dbname=django_dev
```

### Installing New Python Packages

1. Add the package to `requirements.txt`
2. Rebuild the container:

```bash
docker-compose up -d --build
```

### Viewing Database Data

```bash
# Connect to PostgreSQL
docker-compose exec db psql -U django_user -d django_dev

# Useful PostgreSQL commands:
# \dt - List all tables
# \d table_name - Describe table structure
# SELECT * FROM table_name; - Query data
# \q - Quit
```

### Rebuilding Containers

If you make changes to Dockerfile or requirements:

```bash
docker-compose down
docker-compose up -d --build
```

### Debugging

#### Enable Django Debug Toolbar

1. Add to `requirements.txt`:
```txt
django-debug-toolbar==4.2.0
```

2. Update `settings.py`:
```python
INSTALLED_APPS = [
    # ...
    'debug_toolbar',
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    # ...
]

INTERNAL_IPS = [
    '127.0.0.1',
]

import socket
hostname, _, ips = socket.gethostbyname_ex(socket.gethostname())
INTERNAL_IPS += [ip[: ip.rfind(".")] + ".1" for ip in ips]
```

3. Update `urls.py`:
```python
from django.conf import settings
from django.urls import include, path

urlpatterns = [
    # ...
]

if settings.DEBUG:
    import debug_toolbar
    urlpatterns = [
        path('__debug__/', include(debug_toolbar.urls)),
    ] + urlpatterns
```

#### View Container Logs

```bash
docker-compose logs -f web
```

#### Execute Commands in Running Container

```bash
docker-compose exec web /bin/bash
```

## Environment Variables Explained

| Variable | Description | Example |
|----------|-------------|---------|
| `DEBUG` | Enable Django debug mode | `1` for development, `0` for production |
| `SECRET_KEY` | Django secret key | Random 50-character string |
| `DJANGO_ALLOWED_HOSTS` | Allowed hosts for Django | `localhost 127.0.0.1` |
| `SQL_ENGINE` | Database backend | `django.db.backends.postgresql` |
| `SQL_DATABASE` | Database name | `django_dev` |
| `SQL_USER` | Database user | `django_user` |
| `SQL_PASSWORD` | Database password | `django_password` |
| `SQL_HOST` | Database host | `db` (service name) |
| `SQL_PORT` | Database port | `5432` |
| `DATABASE` | Database type | `postgres` |

## Best Practices for Development

1. **Always use environment variables** for sensitive data
2. **Don't commit `.env` files** to version control (add to `.gitignore`)
3. **Use volumes** for code to enable hot-reloading
4. **Keep requirements.txt updated** when adding new packages
5. **Use meaningful commit messages**
6. **Run migrations** after model changes
7. **Write tests** for new features
8. **Use linting tools** (flake8, black, pylint)

## Troubleshooting

### Container Won't Start

```bash
# Check logs
docker-compose logs web

# Rebuild from scratch
docker-compose down -v
docker-compose up -d --build
```

### Database Connection Issues

Ensure PostgreSQL is ready before Django starts. The `entrypoint.sh` script handles this, but verify:

```bash
docker-compose logs db
```

### Permission Issues

If you encounter permission errors with volumes:

```bash
# Linux/Mac
sudo chown -R $USER:$USER .
```

### Port Already in Use

If port 8000 is already in use, change it in `docker-compose.yml`:

```yaml
ports:
  - 8001:8000  # Use port 8001 instead
```

## Next Steps

- [Staging Environment](staging-environment.md) - Set up staging for testing
- [Production Deployment Guide](production-deployment.md)
- [SSL/HTTPS Setup](ssl-setup.md)
- [Best Practices](best-practices.md)
