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

