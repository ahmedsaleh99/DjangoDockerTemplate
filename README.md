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
      - postgres_data:/var/lib/postgresql/data/
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
```

**Explanation:**
- **db service**: Runs PostgreSQL 18.1 with persistent data storage
- **postgres_data volume**: Ensures database data persists between container restarts
- **Environment variables**: Configure PostgreSQL credentials and Django database connection
- **SQL_HOST=db**: The hostname matches the service name in docker-compose.yml


