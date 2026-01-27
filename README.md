# DjangoDockerTemplate

A production-ready Django application template with Docker, PostgreSQL, Gunicorn, and Nginx. This template is designed for containerized development and deployment with automated SSL certificate management using Step CA as a private ACME server.

## Features

- 🐳 **Docker & Docker Compose** - Fully containerized application stack
- 🗄️ **PostgreSQL** - Production-grade relational database
- 🚀 **Gunicorn** - Python WSGI HTTP server for production
- 🌐 **nginx-proxy** - Automated reverse proxy configuration
- 🔒 **Step CA + acme-companion** - Private ACME server for automatic SSL/TLS certificates (no public domain required)
- 🐍 **Python 3.11** - Latest stable Python with virtual environment support
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

```bash
# Clone the repository
git clone https://github.com/ahmedsaleh99/DjangoDockerTemplate.git
cd DjangoDockerTemplate

# Create Python virtual environment
python3.11 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

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

### Step 1: Create Virtual Environment

```bash
# Create a new directory for your project
mkdir django_docker_project
cd django_docker_project

# Create virtual environment with Python 3.11
python3.11 -m venv venv

# Activate the virtual environment
# On Linux/Mac:
source venv/bin/activate

# On Windows:
# venv\Scripts\activate
```

### Step 2: Create requirements.txt

Create a `requirements.txt` file with Django 5:

```txt
Django==5.0
psycopg2-binary==2.9.9
gunicorn==21.2.0
```

Install the requirements:

```bash
pip install -r requirements.txt
```

### Step 3: Create Django Project

Use django-admin to create a new project named `hello_django`:

```bash
django-admin startproject hello_django .
```

**Note:** The `.` at the end creates the project in the current directory.

### Step 4: Project Structure

After creating the Django project, your directory structure should look like this:

```
django_docker_project/
├── venv/                    # Virtual environment (excluded from git)
├── hello_django/            # Django project directory
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py         # Project settings
│   ├── urls.py             # URL configuration
│   └── wsgi.py             # WSGI configuration
├── manage.py                # Django management script
└── requirements.txt         # Python dependencies
```

### Step 5: Verify Installation

Test that Django is working correctly:

```bash
# Run development server
python manage.py runserver

# You should see:
# Starting development server at http://127.0.0.1:8000/
# Visit http://127.0.0.1:8000/ in your browser
```

Press `Ctrl+C` to stop the development server.


