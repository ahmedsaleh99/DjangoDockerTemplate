# DjangoDockerTemplate

A production-ready Django application template with Docker, PostgreSQL, Gunicorn, and Nginx. This template is designed for containerized development and deployment with automated SSL certificate management using Step CA as a private ACME server.

## Features

- 🐳 **Docker & Docker Compose** - Fully containerized application stack
- 🗄️ **PostgreSQL 18.1** - Production-grade relational database
- 🚀 **Gunicorn** - Python WSGI HTTP server for production
- 🌐 **nginx-proxy** - Automated reverse proxy configuration
- 🔒 **Step CA + acme-companion** - Private ACME server for automatic SSL/TLS certificates (no public domain required)
- 🐍 **Python 3.12** - Latest stable Python with virtual environment support
- 🎯 **Multi-environment** - Separate configurations for development, staging, and production
- 📁 **Static & Media Files** - Properly configured file handling with Nginx
- 🔧 **Environment Variables** - Secure configuration management

## Quick Start

### Prerequisites

- Docker (version 20.10+)
- Docker Compose (version 1.29+)
- Python 3.12 (for local development)
- Git

### Development Setup

Clone the repository and let Docker handle the setup:

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

# Access the application at http://localhost:8000
```

### Stop the Application

```bash
# Stop containers
docker-compose down

# Stop and remove volumes (⚠️ this will delete your database)
docker-compose down -v
```

## Documentation

For detailed step-by-step guides, see the following documentation:

- **[Development Guide](docs/development-guide.md)** - Build a Django project from scratch with Docker (Steps 1-13)
- **[Production Deployment](docs/production-deployment.md)** - Deploy with Gunicorn and Nginx (Steps 14-21)
- **[Auto-SSL Setup](docs/auto-ssl-setup.md)** - Configure automatic SSL with Step CA and nginx-proxy (Steps 22-24)

## Technology Stack

| Component | Version |
|-----------|---------|
| Python | 3.12 |
| Django | 5.0 |
| PostgreSQL | 18.1-trixie |
| Nginx | 1.27-alpine |
| Gunicorn | 21.2.0 |
| psycopg2-binary | 2.9.9 |
| Step CA | latest |
| nginx-proxy | 1.27-alpine |
| acme-companion | 2.6 |

## Project Structure

```
DjangoDockerTemplate/
├── hello_django/                 # Django application
│   ├── Dockerfile                # Development Dockerfile
│   ├── Dockerfile.prod           # Production Dockerfile
│   ├── entrypoint.sh             # Development entrypoint
│   ├── entrypoint.prod.sh        # Production entrypoint
│   ├── manage.py
│   ├── requirements.txt
│   └── hello_django/             # Django project
│       └── settings.py
├── nginx/                        # Nginx configuration
│   ├── Dockerfile                # Nginx Dockerfile
│   ├── nginx.tmpl                # nginx-proxy template
│   ├── custom.conf               # Custom proxy config
│   └── vhost.d/
│       └── default               # Static/media file config
├── docker-compose.yml            # Development environment
├── docker-compose.prod.yml       # Production environment
├── docker-compose.staging.yml    # Staging with auto-SSL
├── .env.dev                      # Development environment variables
├── .env.prod                     # Production environment variables
├── .env.prod.db                  # Production database variables
├── .env.staging                  # Staging environment variables
├── .env.staging.db               # Staging database variables
└── docs/                         # Documentation
    ├── development-guide.md
    ├── production-deployment.md
    └── auto-ssl-setup.md
```

## Environment Configurations

### Development (.env.dev)
- Django development server
- PostgreSQL with local volumes
- Debug mode enabled
- No SSL/HTTPS

### Staging (.env.staging)
- Gunicorn WSGI server
- PostgreSQL with persistent volumes
- Step CA for automatic SSL
- nginx-proxy for reverse proxy
- Local domain testing support

### Production (.env.prod)
- Gunicorn WSGI server (optimized workers)
- PostgreSQL with persistent volumes
- SSL/TLS with Step CA or Let's Encrypt
- nginx-proxy with acme-companion
- Static and media files served by Nginx

## Common Commands

### Development

```bash
# Start development environment
docker-compose up -d --build

# View logs
docker-compose logs -f web

# Run migrations
docker-compose exec web python manage.py migrate

# Create superuser
docker-compose exec web python manage.py createsuperuser

# Collect static files
docker-compose exec web python manage.py collectstatic --no-input

# Django shell
docker-compose exec web python manage.py shell

# Stop environment
docker-compose down
```

### Production

```bash
# Start production environment
docker-compose -f docker-compose.prod.yml up -d --build

# Run migrations
docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput

# Collect static files
docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input

# Create superuser
docker-compose -f docker-compose.prod.yml exec web python manage.py createsuperuser

# View logs
docker-compose -f docker-compose.prod.yml logs -f

# Stop environment
docker-compose -f docker-compose.prod.yml down
```

### Staging

```bash
# Start staging environment with auto-SSL
docker-compose -f docker-compose.staging.yml up -d --build

# Run migrations
docker-compose -f docker-compose.staging.yml exec web python manage.py migrate --noinput

# Collect static files
docker-compose -f docker-compose.staging.yml exec web python manage.py collectstatic --no-input

# View SSL certificate logs
docker-compose -f docker-compose.staging.yml logs -f acme-companion

# Stop environment
docker-compose -f docker-compose.staging.yml down
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- Inspired by [testdriven.io's Django Docker tutorial](https://testdriven.io/blog/dockerizing-django-with-postgres-gunicorn-and-nginx/)
- Uses [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) for automatic Nginx configuration
- Uses [acme-companion](https://github.com/nginx-proxy/acme-companion) for automatic SSL certificates
- Uses [Step CA](https://github.com/smallstep/certificates) as a private ACME server
