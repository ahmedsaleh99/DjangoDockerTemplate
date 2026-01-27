# DjangoDockerTemplate

A production-ready Django application template with Docker, PostgreSQL, Gunicorn, and Nginx. This template is designed for containerized development and deployment with automated SSL certificate management using Step CA as a private ACME server.

## Features

- 🐳 **Docker & Docker Compose** - Fully containerized application stack
- 🗄️ **PostgreSQL** - Production-grade relational database
- 🚀 **Gunicorn** - Python WSGI HTTP server for production
- 🌐 **nginx-proxy** - Automated reverse proxy configuration
- 🔒 **Step CA + acme-companion** - Private ACME server for automatic SSL/TLS certificates (no public domain required)
- 🐍 **Python 3.12** - Latest stable Python with virtual environment support
- 🎯 **Multi-environment** - Separate configurations for development, staging, and production
- 📁 **Static & Media Files** - Properly configured file handling
- 🔧 **Environment Variables** - Secure configuration management

## Quick Start

### Prerequisites

- Docker (version 20.10+)
- Docker Compose (version 1.29+)
- Python 3.11 (for local development)
- Git

### Development Setup
To Make a long story short clone the repo and let docker left the load.

```bash
# Clone the repository
git clone https://github.com/ahmedsaleh99/DjangoDockerTemplate.git
cd DjangoDockerTemplate

# Build and start containers
docker-compose up -d --build

# Run migrations
docker-compose exec web python manage.py migrate

# Create superuser
docker-compose exec web python manage.py createsuperuser

# Access the application
# Open http://localhost:8000 in your browser
```

### Stop the Application

```bash
# Stop containers
docker-compose down

# Stop and remove volumes (⚠️ this will delete your database)
docker-compose down -v
```

## From Scratch - Step by Step

This section guides you through creating a Django project from scratch before containerizing it.
Here we start with empty repo and move forward step by step toward the repo in its current state 

### Step 1: Create Virtual Environment

```bash
# Create a new directory for your project
mkdir DjangoDockerTemplate # Our empty repo
cd DjangoDockerTemplate

# Create virtual environment with Python 3.12
python3.12 -m venv venv

# Activate the virtual environment
# On Linux/Mac:
source venv/bin/activate

# On Windows:
# venv\Scripts\activate

# install Django
pip install django==5.2.10
```


### Step 2: Create Django Project

Use django-admin to create a new project named `hello_django`:

```bash
django-admin startproject hello_django
```
After creating the Django project, your directory structure should look like this:

```
DjangoDockerTemplate/
├── venv/                         # Virtual environment (do NOT commit to git)
├── hello_django/                 # Django project root
│   ├── manage.py                 # Django management script
│   └── hello_django/             # Django project package
│       ├── __init__.py
│       ├── asgi.py
│       ├── settings.py           # Project settings
│       ├── urls.py               # URL configuration
│       └── wsgi.py               # WSGI configuration
└── requirements.txt              # Python dependencies
```

### Step 3: Create requirements.txt

Create a `requirements.txt` file with Django 5:

```bash
touch hello_django/requirements.txt
```
Add the following packages tp `` we will need it later when we dockerize the app.

```txt
django==5.2.10
psycopg2-binary==2.9.11
gunicorn==24.1.1
```

### Step 4: Verify Installation

Test that Django is working correctly:

```bash
# Run development server
cd hello_django/
python manage.py runserver

# You should see:
# Starting development server at http://127.0.0.1:8000/
# Visit http://127.0.0.1:8000/ in your browser
```

Press `Ctrl+C` to stop the development server.

### Step 5: Create Dockerfile

Now let's dockerize the application by creating a Dockerfile in the `hello_django` directory:

```bash
# Navigate to the hello_django directory (if not already there)
cd hello_django
touch Dockerfile
```

Add the following content to the `Dockerfile` just created:

```dockerfile
FROM python:3.12.12-slim-trixie

ENV APP_HOME=/usr/src/app
RUN mkdir $APP_HOME
WORKDIR $APP_HOME


# set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# install system dependencies
RUN apt-get update \
    && apt-get install -y --no-install-recommends netcat-openbsd \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean \
    && apt-get autoremove -y

COPY ./requirements.txt .
RUN pip install --upgrade pip \
    && pip install -r requirements.txt

# copy project
COPY . .

```

After creating the Dockerfile, your directory structure should look like this:

```
DjangoDockerTemplate/
├── venv/                         # Virtual environment (do NOT commit to git)
└── hello_django/                 # Django project root
    ├── Dockerfile                # Docker configuration (NEW)
    ├── manage.py                 # Django management script
    ├── requirements.txt          # Python dependencies
    └── hello_django/             # Django project package
        ├── __init__.py
        ├── asgi.py
        ├── settings.py           # Project settings
        ├── urls.py               # URL configuration
        └── wsgi.py               # WSGI configuration
```

### Step 6: Create Docker Compose and Environment File

Now let's create a docker-compose.yml file in the main directory to orchestrate our containers.

First, navigate back to the main directory and create the `.env.dev` file for environment variables:

```bash
# Navigate back to main directory
cd ..

# Create .env.dev file
touch .env.dev
```

Add the following environment variables to `.env.dev`:

```env
DEBUG=1
SECRET_KEY=foo
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
```

Now create the `docker-compose.yml` file in the main directory:

```bash
touch docker-compose.yml
```

Add the following content to `docker-compose.yml`:

```yaml
# dev environment docker compose
services:
  web:
    build: ./hello_django
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./hello_django/:/usr/src/app/
    ports:
      - 8000:8000
    env_file:
      - ./.env.dev
```

After creating these files, your directory structure should look like this:

```
DjangoDockerTemplate/
├── venv/                         # Virtual environment (do NOT commit to git)
├── .env.dev                      # Development environment variables (NEW)
├── docker-compose.yml            # Docker Compose configuration (NEW)
└── hello_django/                 # Django project root
    ├── Dockerfile                # Docker configuration
    ├── manage.py                 # Django management script
    ├── requirements.txt          # Python dependencies
    └── hello_django/             # Django project package
        ...
```

### Step 7: Adjust Django Settings for Environment Variables

Now we need to configure Django's `settings.py` to read from environment variables. Open `hello_django/hello_django/settings.py` and make the following changes:

First, import the `os` module at the top of the file (if not already imported):

```python
import os
```

Then, replace the hardcoded values with environment variable readers:

```python
SECRET_KEY = os.environ.get("SECRET_KEY")

DEBUG = bool(os.environ.get("DEBUG", default=0))

# 'DJANGO_ALLOWED_HOSTS' should be a single string of hosts with a space between each.
# For example: 'DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]'
ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS").split(" ")
```

**Explanation:**
- `SECRET_KEY`: Reads from the environment variable instead of being hardcoded
- `DEBUG`: Converts the environment variable to a boolean (0 = False, any other value = True)
- `ALLOWED_HOSTS`: Splits the space-separated string from the environment variable into a list

These changes allow your Django application to use the values from the `.env.dev` file we created in Step 6.

### Step 8: Build and Run with Docker Compose

Now it's time to build and run your Dockerized Django application using Docker Compose.

```bash
# Make sure you're in the main directory (DjangoDockerTemplate)
# Build the Docker image and start the containers
docker-compose up -d --build
```

**What this command does:**
- `docker-compose up`: Starts the services defined in docker-compose.yml
- `-d`: Runs containers in detached mode (in the background)
- `--build`: Builds the Docker image before starting containers

After running this command, Docker will:
1. Build the Docker image from the Dockerfile
2. Install all dependencies from requirements.txt
3. Start the web service container
4. Map port 8000 from the container to your local machine

**Check if the website is up:**

Open your web browser and navigate to:
```
http://localhost:8000
```

You should see the Django default welcome page with the message "The install worked successfully! Congratulations!"

**Useful commands:**

```bash
# View running containers
docker-compose ps

# View logs
docker-compose logs

# View logs for the web service only
docker-compose logs web

# Follow logs in real-time
docker-compose logs -f web

# Stop the containers
docker-compose down

# Stop containers and remove volumes
docker-compose down -v
```

If you see the Django welcome page at http://localhost:8000, congratulations! Your Django application is now successfully running in a Docker container.

### Step 9: Add PostgreSQL Database

Now let's add PostgreSQL as our database. We'll use the official PostgreSQL image and configure Django to connect to it.

First, update the `docker-compose.yml` file to add the PostgreSQL service. Open `docker-compose.yml` and modify it:

```yaml
# dev environment docker compose
services:
  web:
    build: ./hello_django
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./hello_django/:/usr/src/app/
    ports:
      - 8000:8000
    env_file:
      - ./.env.dev
  
  db:
    image: postgres:18.1-trixie
    volumes:
      - postgres_data:/var/lib/postgresql/
    environment:
      - POSTGRES_USER=hello_django
      - POSTGRES_PASSWORD=hello_django
      - POSTGRES_DB=hello_django_dev

volumes:
  postgres_data:
```

Next, update the `.env.dev` file to add the database connection variables:

```env
DEBUG=1
SECRET_KEY=foo
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_dev
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres
```

**Explanation:**
- **db service**: Runs PostgreSQL 18.1 with persistent data storage
- **postgres_data volume**: Ensures database data persists between container restarts
- **Environment variables**: Configure PostgreSQL credentials and Django database connection
- **SQL_HOST=db**: The hostname matches the service name in docker-compose.yml

### Step 10: Update Django Database Configuration

Update the `DATABASES` dict in `hello_django/hello_django/settings.py`:

```python
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
```

Here, the database is configured based on the environment variables that we just defined. Take note of the default values.

### Step 11: Build and Run Migrations

Build the new image and spin up the two containers:

```bash
docker-compose up -d --build
```

Run the migrations:

```bash
docker-compose exec web python manage.py migrate --noinput
```

### Step 12: Create Entrypoint Script

Next, add an `entrypoint.sh` file to the `hello_django` directory to verify that Postgres is healthy before applying the migrations and running the Django development server:

Create `hello_django/entrypoint.sh`:

```bash
#!/bin/sh
# if database is postgres wait until it become up and running
if [ "$DATABASE" = "postgres" ]
then
    echo "Waiting for postgres..."

    while ! nc -z $SQL_HOST $SQL_PORT; do
      sleep 0.1
    done

    echo "PostgreSQL started"
fi

python manage.py flush --no-input # always delete old database 
python manage.py migrate # always do migration

exec "$@"
```

Update the file permissions locally:

```bash
chmod +x hello_django/entrypoint.sh
```

### Step 13: Update Dockerfile to Use Entrypoint

Then, update the Dockerfile to copy over the entrypoint.sh file and run it as the Docker entrypoint command. 

Update `hello_django/Dockerfile`:

```dockerfile
# Pull official base image
FROM python:3.12.12-slim-trixie

# Set work directory
WORKDIR /usr/src/app

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Install dependencies
COPY ./requirements.txt .
RUN pip install --upgrade pip
RUN pip install -r requirements.txt

# Copy entrypoint.sh
COPY ./entrypoint.sh .
# remove \r from the shell script to be valid for linux
# in case you used a windows machine for devlopment
RUN sed -i 's/\r$//g' /usr/src/app/entrypoint.sh
RUN chmod +x /usr/src/app/entrypoint.sh

# Copy project
COPY . .

# Run entrypoint.sh
ENTRYPOINT ["/usr/src/app/entrypoint.sh"]
```

**Explanation:**
- The `sed` command handles Windows line ending issues (CRLF to LF conversion)
- The `chmod` command ensures the script is executable
- The `ENTRYPOINT` directive runs the entrypoint script before starting the Django development server

---

## Production Deployment
The production environment is almost the same except we used multistage builds for docker image to reduce image size.
### Step 14: Production Dockerfile

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

### Step 15: Production Entrypoint Script

Create `app/entrypoint.prod.sh`:

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
chmod +x app/entrypoint.prod.sh
```

### Step 16: Production Environment Variables

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

### Step 17: Production Docker Compose

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

### Step 18: Nginx Configuration

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

### Step 19: Update Django Settings for Production

Update `hello_django/hello_django/settings.py` to handle static and media files:

```python
# Static files (CSS, JavaScript, Images)
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

# Media files
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'mediafiles'
```

### Step 20: Build and Run Production

Build and run the production stack:

```bash
docker-compose -f docker-compose.prod.yml up -d --build
```

Here are two usefull  manual migrations and collect static files:

```bash
docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input --clear
```

Create superuser for the first time on production:

```bash
docker-compose -f docker-compose.prod.yml exec web python manage.py createsuperuser
```

Access the application at `http://localhost:1337`

### Step 21: Project Structure

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

### Production Commands

**Stop production containers:**
```bash
docker-compose -f docker-compose.prod.yml down
```

**Stop and remove volumes:**
```bash
docker-compose -f docker-compose.prod.yml down -v
```

**View logs:**
```bash
docker-compose -f docker-compose.prod.yml logs -f
```

**Rebuild after changes:**
```bash
docker-compose -f docker-compose.prod.yml up -d --build
```

---

## Advanced: Automatic SSL with Step CA, nginx-proxy, and acme-companion

For automatic Nginx reverse proxy configuration and SSL certificate management using Step CA (private ACME server), nginx-proxy, and acme-companion.

### Local Domain Setup for Testing

Since Step CA works with domains (not public domains required), you'll need to map your local IP address to a staging domain name for testing purposes.

#### 1. Get Your Local IP Address

**On Windows:**
```bash
ipconfig
```
Look for your IPv4 address (e.g., `192.168.1.100`)

**On Linux/Mac:**
```bash
hostname -I
# or
ifconfig
# or
ip addr show
```

#### 2. Map Local Domain to Hosts File

**On Windows:**

1. Open Notepad as Administrator
2. Open file: `C:\Windows\System32\drivers\etc\hosts`
3. Add your local domain mapping:

```
192.168.1.100    staging.yourdomain.local
192.168.1.100    www.staging.yourdomain.local
```

**On Linux/Mac:**

1. Edit hosts file with sudo:

```bash
sudo nano /etc/hosts
```

2. Add your local domain mapping:

```
192.168.1.100    staging.yourdomain.local
192.168.1.100    www.staging.yourdomain.local
```

3. Save and exit (Ctrl+X, then Y, then Enter)

#### 3. Verify Domain Resolution

**Test that your domain resolves correctly:**

```bash
ping staging.yourdomain.local
```

You should see responses from your local IP address (192.168.1.100 in this example).

**Test in browser:**

Open `http://staging.yourdomain.local` in your browser - you should be able to reach your local machine (even if nothing is running yet).

**Notes:**
- Replace `192.168.1.100` with your actual local IP address
- Replace `staging.yourdomain.local` with your desired staging domain name
- You can use any domain suffix like `.local`, `.test`, or `.dev` for local testing
- This eliminates the need for a public domain name when using Step CA
- All devices that need to access the staging site will need this hosts file entry

### Django Configuration for HTTPS Proxy

First, to run the Django app behind an HTTPS proxy you'll need to add the `SECURE_PROXY_SSL_HEADER` setting to `settings.py`:

```python
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
```

In this tuple, when `X-Forwarded-Proto` is set to `https` the request is secure.

You'll also need to update `CSRF_TRUSTED_ORIGINS` inside `settings.py`:

```python
CSRF_TRUSTED_ORIGINS = os.environ.get("CSRF_TRUSTED_ORIGINS").split(" ")
```

Add `CSRF_TRUSTED_ORIGINS` to your `.env.staging` and `.env.prod` files:

```bash
CSRF_TRUSTED_ORIGINS=https://yourdomain.com https://www.yourdomain.com
```

### Step 22: Understanding the Architecture

This setup provides:
- **nginx-proxy**: Automated Nginx reverse proxy that auto-configures based on container labels
- **docker-gen**: Watches Docker containers and regenerates Nginx configs
- **acme-companion**: Automatic SSL certificate provisioning and renewal via ACME protocol
- **Step CA**: Private ACME server (no public domain required)

**Benefits:**
- Zero manual Nginx configuration
- Automatic SSL certificate issuance and renewal
- Works without public domains (ideal for internal/staging environments)
- Multiple applications on same host with automatic routing

### Step 23: Staging Environment with Auto-SSL

The repository includes `docker-compose.staging.yml` with Step CA integration.

**Key services:**
- **web**: Django app with Gunicorn
- **db**: PostgreSQL database
- **nginx-proxy**: Nginx reverse proxy (built from `./nginx/Dockerfile`)
- **docker-gen**: Config generator using `./nginx/nginx.tmpl`
- **acme-companion**: SSL certificate manager (nginxproxy/acme-companion:2.6)
- **step-ca**: Private CA server (smallstep/step-ca:latest)

**Create `.env.staging` file:**
```bash
DOMAIN=yourdomain.stage

DEBUG=0
SECRET_KEY=change_me
DJANGO_ALLOWED_HOSTS=${DOMAIN}
CSRF_TRUSTED_ORIGINS=https://${DOMAIN}

SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_prod
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres

LETSENCRYPT_HOST=${DOMAIN}
VIRTUAL_HOST=${DOMAIN}
VIRTUAL_PORT=8000
HTTPS_METHOD=redirect
```

**Create `.env.staging.db` file:**
```bash
POSTGRES_USER=hello_django
POSTGRES_PASSWORD=hello_django
POSTGRES_DB=hello_django_staging
```

**Environment Variables Explained:**
- `VIRTUAL_HOST`: Domain for nginx-proxy to route (can be multiple comma-separated)
- `VIRTUAL_PORT`: Internal port where app listens (8000 for Gunicorn)
- `LETSENCRYPT_HOST`: Domain for SSL certificate

**Build and run staging:**
```bash
docker-compose -f docker-compose.staging.yml up -d --build
```

**Run migrations:**
```bash
docker-compose -f docker-compose.staging.yml exec web python manage.py migrate --noinput
```

**Collect static files:**
```bash
docker-compose -f docker-compose.staging.yml exec web python manage.py collectstatic --no-input --clear
```

**Create superuser:**
```bash
docker-compose -f docker-compose.staging.yml exec web python manage.py createsuperuser
```

**Access:**
- HTTP: `http://yourdomain.com` (redirects to HTTPS)
- HTTPS: `https://yourdomain.com`
- Step CA UI: `https://localhost:9000`

### Nginx Configuration for nginx-proxy

Next, let's update the Nginx configuration in the "nginx" folder.

First, add directory called `vhost.d`. Then, add a file called `default` inside that directory to serve static and media files:

```nginx
location /static/ {
  alias /home/app/web/staticfiles/;
  add_header Access-Control-Allow-Origin *;
}

location /media/ {
  alias /home/app/web/mediafiles/;
  add_header Access-Control-Allow-Origin *;
}
```

Requests that match any of these patterns will be served from static or media folders. They won't be proxied to other containers. The web and nginx-proxy containers share the volumes in which the static and media files are located:

```
static_volume:/home/app/web/staticfiles
media_volume:/home/app/web/mediafiles
```

Add a `custom.conf` file to the "nginx" folder to hold custom proxy-wide configuration:

```nginx
client_max_body_size 10M;
```

Update `nginx/Dockerfile`:

```dockerfile
FROM nginxproxy/nginx-proxy
COPY vhost.d/default /etc/nginx/vhost.d/default
COPY custom.conf /etc/nginx/conf.d/custom.conf
```

Remove `nginx.conf`.

Your "nginx" directory should now look like this:

```
└── nginx
    ├── Dockerfile
    ├── custom.conf
    └── vhost.d
        └── default
```

### Step 24: Production with Auto-SSL

The `docker-compose.prod.yml` has been updated with the same nginx-proxy + acme-companion architecture.

**Key differences from staging:**
- Uses `.env.prod` and `.env.prod.db` instead of `.env.staging*`
- Can use public CA or self-hosted Step CA
- Production-grade environment variables

**Update `.env.prod` with SSL variables:**
```bash
DOMAIN=yourdomain.prod # Change to your production domain

DEBUG=0
SECRET_KEY=change_me # use docker secrets instead for better security
DJANGO_ALLOWED_HOSTS=${DOMAIN}
CSRF_TRUSTED_ORIGINS=https://${DOMAIN}

SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_prod
SQL_USER=hello_django
SQL_PASSWORD=hello_django # use docker secrets instead for better security
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres

LETSENCRYPT_HOST=${DOMAIN}
VIRTUAL_HOST=${DOMAIN}
VIRTUAL_PORT=8000
HTTPS_METHOD=redirect
```

**For self-hosted Step CA:**
- Keep `step-ca` service in docker-compose
- Set `ACME_CA_URI=https://step-ca:9000/acme/acme/directory`
- Set `CA_BUNDLE=/home/step/certs/root_ca.crt`

**For public CA (e.g., Let's Encrypt):**
- Remove `step-ca` service from docker-compose
- Remove `ACME_CA_URI` from acme-companion environment
- Remove `CA_BUNDLE` from acme-companion environment
- Update `DEFAULT_EMAIL` in acme-companion

**Build and run production with auto-SSL:**
```bash
docker-compose -f docker-compose.prod.yml up -d --build
```

**First-time setup:**
```bash
# Run migrations
docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput

# Collect static files
docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input --clear

# Create superuser
docker-compose -f docker-compose.prod.yml exec web python manage.py createsuperuser
```

**Verify SSL certificate:**
```bash
docker-compose -f docker-compose.prod.yml logs acme-companion
```

**Certificate renewal:**
Certificates auto-renew via acme-companion. No manual intervention needed.

### Nginx Template Configuration

The [`nginx/nginx.tmpl`](https://raw.githubusercontent.com/nginx-proxy/nginx-proxy/main/nginx.tmpl) file is used by docker-gen to generate Nginx configurations dynamically. It:
- Auto-detects containers with `VIRTUAL_HOST` labels
- Configures upstream servers
- Sets up SSL with certificates from acme-companion
- Handles HTTP to HTTPS redirects
- Supports multiple apps on same host

**No manual editing required** - all configuration happens via environment variables in your Django container.

### Troubleshooting Auto-SSL

**Check nginx-proxy logs:**
```bash
docker logs nginx-proxy
```

**Check acme-companion logs:**
```bash
docker logs nginx-proxy-acme
```

**Check Step CA logs:**
```bash
docker logs step-ca
```

**View generated Nginx config:**
```bash
docker exec nginx-proxy cat /etc/nginx/conf.d/default.conf
```

**Certificate not issuing:**
- Verify `VIRTUAL_HOST` matches `LETSENCRYPT_HOST`
- Check acme-companion can reach Step CA (network connectivity)
- Ensure Step CA is initialized (check logs)
- Verify CA bundle path if using self-hosted CA


---

## License

MIT

## Acknowledgments

This template is based on best practices from:
- [TestDriven.io - Dockerizing Django with Postgres, Gunicorn, and Nginx](https://testdriven.io/blog/dockerizing-django-with-postgres-gunicorn-and-nginx/)
- [TestDriven.io - Django with Let's Encrypt](https://testdriven.io/blog/django-lets-encrypt/)
- [TestDriven.io - Docker Best Practices for Python Developers](https://testdriven.io/blog/docker-best-practices/)
- [SmallStep - Run your own private CA & ACME server using step-ca](https://smallstep.com/blog/private-acme-server/)
- [nginx-proxy - Automated Nginx Reverse Proxy for Docker](https://github.com/nginx-proxy/nginx-proxy)
- [acme-companion - LetsEncrypt companion for nginx-proxy](https://github.com/nginx-proxy/acme-companion)


Adapted to use:
- Python 3.12
- Django 5.0
- PostgreSQL 18.1
- Nginx 1.27-alpine
- Gunicorn 21.2.0
- Step CA for private ACME server
- nginx-proxy for automatic reverse proxy configuration
- acme-companion for automatic SSL certificate management
- Python 3.12
- Django 5.0
- PostgreSQL 18.1
- Step CA for private ACME server
- nginx-proxy and acme-companion for automated SSL management


