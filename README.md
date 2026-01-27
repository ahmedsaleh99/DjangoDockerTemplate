# Django Docker Template

A production-ready Django application template with Docker, PostgreSQL, Gunicorn, and Nginx. This template provides a complete setup for both development and production environments.

## Features

- 🐳 **Docker & Docker Compose** - Containerized development and production environments
- 🗄️ **PostgreSQL** - Production-grade database
- 🚀 **Gunicorn** - Production WSGI HTTP server
- 🌐 **Nginx** - Reverse proxy and static file serving
- 🔒 **SSL/HTTPS** - Automatic SSL certificates using Step CA (private ACME server) with nginx-proxy and acme-companion
- 📁 **Media & Static Files** - Properly configured file handling
- 🔧 **Environment Variables** - Secure configuration management

## Quick Start

### Development Setup

```bash
# Clone the repository
git clone https://github.com/ahmedsaleh99/DjangoDockerTemplate.git
cd DjangoDockerTemplate

# Build and run the development environment
docker-compose up -d --build

# Run migrations
docker-compose exec web python manage.py migrate

# Create superuser
docker-compose exec web python manage.py createsuperuser

# Access the application at http://localhost:8000
```

### Production Deployment

See the [Production Deployment Guide](wiki/production-deployment.md) for detailed instructions on deploying to production with Gunicorn and Nginx.

## Documentation

- 📖 [Development Setup](wiki/development-setup.md) - Complete development environment guide
- 🚀 [Production Deployment](wiki/production-deployment.md) - Production setup with Gunicorn and Nginx
- 🔒 [SSL/HTTPS Setup](wiki/ssl-setup.md) - Step CA with nginx-proxy and acme-companion for automatic SSL
- 🛠️ [Troubleshooting](wiki/troubleshooting.md) - Common issues and solutions
- 💡 [Best Practices](wiki/best-practices.md) - Security and optimization tips

## Architecture

```
┌─────────────────────────────────────────────┐
│              Nginx (Reverse Proxy)          │
│         (Port 80/443 - SSL/HTTPS)           │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│           Gunicorn (WSGI Server)            │
│              Django Application              │
│               (Port 8000)                    │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│          PostgreSQL Database                 │
│               (Port 5432)                    │
└─────────────────────────────────────────────┘
```

## Project Structure

```
.
├── docker-compose.yml          # Development configuration
├── docker-compose.prod.yml     # Production configuration
├── Dockerfile                  # Development Dockerfile
├── Dockerfile.prod            # Production Dockerfile
├── .env.dev                   # Development environment variables
├── .env.prod                  # Production environment variables
├── .env.prod.db              # Production database environment variables
├── nginx/
│   ├── Dockerfile            # Nginx Dockerfile
│   └── nginx.conf           # Nginx configuration
└── wiki/                     # Documentation
    ├── development-setup.md
    ├── production-deployment.md
    ├── ssl-setup.md
    ├── troubleshooting.md
    └── best-practices.md
```

## Technologies

- **Django** - High-level Python web framework
- **PostgreSQL** - Advanced open source database
- **Gunicorn** - Python WSGI HTTP Server for UNIX
- **Nginx** - High-performance web server and reverse proxy
- **Docker** - Container platform
- **Docker Compose** - Multi-container orchestration
- **Step CA** - Private ACME server for internal SSL/TLS certificates
- **nginx-proxy** - Automated Nginx reverse proxy configuration
- **acme-companion** - Automatic SSL certificate provisioning and renewal

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

This template is based on the excellent tutorials from [TestDriven.io](https://testdriven.io/):
- [Dockerizing Django with Postgres, Gunicorn, and Nginx](https://testdriven.io/blog/dockerizing-django-with-postgres-gunicorn-and-nginx/)

Additional tools integrated:
- [Step CA](https://smallstep.com/docs/step-ca/) - Private ACME server for internal certificates
- [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) - Automated Nginx reverse proxy
- [acme-companion](https://github.com/nginx-proxy/acme-companion) - Automatic SSL certificate management

## Support

If you find this template helpful, please ⭐ star the repository!