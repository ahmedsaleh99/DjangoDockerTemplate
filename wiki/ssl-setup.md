# SSL/HTTPS Setup with Step CA, nginx-proxy, and acme-companion

This guide covers setting up SSL/HTTPS for your Django application using Step CA as a private ACME server, with nginx-proxy for automatic reverse proxy configuration and acme-companion for automatic certificate management.

## Table of Contents

- [Overview](#overview)
- [Why Step CA Instead of Let's Encrypt](#why-step-ca-instead-of-lets-encrypt)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Step CA Setup](#step-ca-setup)
- [nginx-proxy and acme-companion Setup](#nginx-proxy-and-acme-companion-setup)
- [Django Application Configuration](#django-application-configuration)
- [Complete Docker Compose Example](#complete-docker-compose-example)
- [Certificate Management](#certificate-management)
- [Troubleshooting](#troubleshooting)
- [Security Best Practices](#security-best-practices)

## Overview

This setup uses:

- **Step CA** - A private ACME server that issues certificates without requiring a public domain
- **nginx-proxy** - Automatically configures Nginx reverse proxy for your containers
- **acme-companion** - Automatically provisions and renews SSL certificates

**Benefits:**
- ✅ Works without a public domain name
- ✅ Perfect for internal/private deployments
- ✅ Fully automated certificate management
- ✅ Zero manual Nginx configuration needed
- ✅ Automatic certificate renewal
- ✅ Compatible with ACME protocol

## Why Step CA Instead of Let's Encrypt

| Feature | Let's Encrypt | Step CA |
|---------|---------------|---------|
| **Public Domain Required** | ✅ Yes | ❌ No |
| **Internal Networks** | ❌ No | ✅ Yes |
| **Custom Domain Names** | ❌ No | ✅ Yes |
| **Rate Limits** | ✅ Yes (50/week) | ❌ No limits |
| **Certificate Lifetime** | 90 days | Configurable |
| **DNS Challenge** | Required for wildcards | Not needed |
| **Internet Access** | Required | Not required |
| **Private CA** | No | ✅ Yes |

**Use Step CA when:**
- Deploying to private/internal networks
- Working with custom domain names (e.g., `myapp.local`)
- Don't have a public domain
- Want full control over your PKI
- Need certificates for development/staging environments

## Architecture

```
┌─────────────────────────────────────────────┐
│         nginx-proxy Container               │
│   - Automatic Nginx Configuration           │
│   - Reverse Proxy (Port 80/443)            │
│   - SSL Termination                         │
└──────────────┬──────────────────────────────┘
               │
               │  ┌──────────────────────────┐
               │  │  acme-companion          │
               │  │  - Certificate Request   │
               │  │  - Auto Renewal          │
               └──┤  - ACME Protocol         │
                  └────────┬─────────────────┘
                           │
                  ┌────────▼─────────────────┐
                  │      Step CA             │
                  │  - Private ACME Server   │
                  │  - Certificate Authority │
                  └──────────────────────────┘

┌─────────────────────────────────────────────┐
│   Django Application Container              │
│   - Gunicorn (Port 8000)                    │
│   - Environment Variables for proxy         │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│          PostgreSQL Database                 │
└─────────────────────────────────────────────┘
```

## Prerequisites

- Docker and Docker Compose installed
- Basic understanding of Docker networking
- A hostname/domain for your application (can be local, e.g., `myapp.local`)

## Step CA Setup

### 1. Create Step CA Container

Step CA will act as your private Certificate Authority.

Create a directory for Step CA data:

```bash
mkdir -p step-ca
```

### 2. Initialize Step CA

First, we'll initialize Step CA with a simple setup:

```yaml
# docker-compose.step-ca.yml
version: '3.8'

services:
  step-ca:
    image: smallstep/step-ca:latest
    container_name: step-ca
    environment:
      - DOCKER_STEPCA_INIT_NAME=My Private CA
      - DOCKER_STEPCA_INIT_DNS_NAMES=step-ca,localhost
      - DOCKER_STEPCA_INIT_PROVISIONER_NAME=acme
      - DOCKER_STEPCA_INIT_ACME=true
    volumes:
      - step-ca-data:/home/step
    networks:
      - nginx-proxy
    ports:
      - "9000:9000"
    restart: unless-stopped

volumes:
  step-ca-data:

networks:
  nginx-proxy:
    name: nginx-proxy
    driver: bridge
```

### 3. Start Step CA

```bash
docker-compose -f docker-compose.step-ca.yml up -d
```

### 4. Get Step CA Root Certificate

The root certificate will be needed to trust your private CA:

```bash
# Get the root certificate
docker exec step-ca step ca root > step-ca-root.crt

# View certificate details
openssl x509 -in step-ca-root.crt -text -noout
```

### 5. Configure ACME Provisioner

Step CA should already have ACME enabled from initialization. Verify it:

```bash
docker exec step-ca step ca provisioner list
```

You should see an ACME provisioner. If not, add one:

```bash
docker exec step-ca step ca provisioner add acme --type ACME
```

## nginx-proxy and acme-companion Setup

### 1. Create nginx-proxy Configuration

The nginx-proxy container automatically configures Nginx based on environment variables in other containers.

Create `docker-compose.proxy.yml`:

```yaml
version: '3.8'

services:
  nginx-proxy:
    image: nginxproxy/nginx-proxy:latest
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - nginx-certs:/etc/nginx/certs:ro
      - nginx-vhost:/etc/nginx/vhost.d
      - nginx-html:/usr/share/nginx/html
      - nginx-dhparam:/etc/nginx/dhparam
    labels:
      - "com.github.nginx-proxy.nginx-proxy"
    networks:
      - nginx-proxy
    restart: unless-stopped

  acme-companion:
    image: nginxproxy/acme-companion:latest
    container_name: acme-companion
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - nginx-certs:/etc/nginx/certs:rw
      - nginx-vhost:/etc/nginx/vhost.d
      - nginx-html:/usr/share/nginx/html
      - acme-state:/etc/acme.sh
    environment:
      - DEFAULT_EMAIL=admin@example.com
      - NGINX_PROXY_CONTAINER=nginx-proxy
      # Configure for Step CA
      - ACME_CA_URI=https://step-ca:9000/acme/acme/directory
      - CA_BUNDLE=/etc/nginx/certs/step-ca-root.crt
    depends_on:
      - nginx-proxy
    networks:
      - nginx-proxy
    restart: unless-stopped

volumes:
  nginx-certs:
  nginx-vhost:
  nginx-html:
  nginx-dhparam:
  acme-state:

networks:
  nginx-proxy:
    external: true
```

### 2. Add Root Certificate to acme-companion

The acme-companion needs to trust your Step CA:

```bash
# Copy root certificate to nginx-certs volume
docker run --rm -v nginx-certs:/certs -v $(pwd):/src alpine cp /src/step-ca-root.crt /certs/
```

### 3. Start nginx-proxy and acme-companion

```bash
docker-compose -f docker-compose.proxy.yml up -d
```

## Django Application Configuration

### 1. Update Django Docker Compose

Your Django application needs specific environment variables for nginx-proxy to detect and configure it:

```yaml
# docker-compose.yml
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
    environment:
      # nginx-proxy configuration
      - VIRTUAL_HOST=myapp.local
      - VIRTUAL_PORT=8000
      # acme-companion configuration
      - LETSENCRYPT_HOST=myapp.local
      - LETSENCRYPT_EMAIL=admin@example.com
    depends_on:
      - db
    networks:
      - nginx-proxy
      - backend
    restart: unless-stopped

  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    env_file:
      - ./.env.prod.db
    networks:
      - backend
    restart: unless-stopped

volumes:
  postgres_data:
  static_volume:
  media_volume:

networks:
  nginx-proxy:
    external: true
  backend:
    driver: bridge
```

### 2. Environment Variables Explained

| Variable | Purpose | Example |
|----------|---------|---------|
| `VIRTUAL_HOST` | Domain name for your app | `myapp.local` |
| `VIRTUAL_PORT` | Internal port (Gunicorn) | `8000` |
| `LETSENCRYPT_HOST` | Domain for certificate | `myapp.local` |
| `LETSENCRYPT_EMAIL` | Admin email | `admin@example.com` |

### 3. Multiple Domains

For multiple domains, use comma-separated values:

```yaml
environment:
  - VIRTUAL_HOST=myapp.local,www.myapp.local,app.myapp.local
  - LETSENCRYPT_HOST=myapp.local,www.myapp.local,app.myapp.local
```

### 4. Update Django Settings

```python
# settings.py

# HTTPS Settings
if not DEBUG:
    SECURE_SSL_REDIRECT = True
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
    SECURE_BROWSER_XSS_FILTER = True
    SECURE_CONTENT_TYPE_NOSNIFF = True
    
    # Important: nginx-proxy handles SSL, Django sees HTTP
    SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
    
    # Trust nginx-proxy headers
    USE_X_FORWARDED_HOST = True
    USE_X_FORWARDED_PORT = True

# Allowed hosts from environment
ALLOWED_HOSTS = os.environ.get('DJANGO_ALLOWED_HOSTS', '').split(' ')

# Add your domain
ALLOWED_HOSTS += ['myapp.local', 'www.myapp.local']
```

## Complete Docker Compose Example

For a complete setup, you can use a single docker-compose file or separate files:

### Option 1: Single File (Recommended for Development)

```yaml
# docker-compose.full.yml
version: '3.8'

services:
  # Step CA
  step-ca:
    image: smallstep/step-ca:latest
    environment:
      - DOCKER_STEPCA_INIT_NAME=My Private CA
      - DOCKER_STEPCA_INIT_DNS_NAMES=step-ca,localhost
      - DOCKER_STEPCA_INIT_ACME=true
    volumes:
      - step-ca-data:/home/step
    networks:
      - nginx-proxy
    ports:
      - "9000:9000"
    restart: unless-stopped

  # nginx-proxy
  nginx-proxy:
    image: nginxproxy/nginx-proxy:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - nginx-certs:/etc/nginx/certs:ro
      - nginx-vhost:/etc/nginx/vhost.d
      - nginx-html:/usr/share/nginx/html
    networks:
      - nginx-proxy
    restart: unless-stopped

  # acme-companion
  acme-companion:
    image: nginxproxy/acme-companion:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - nginx-certs:/etc/nginx/certs:rw
      - nginx-vhost:/etc/nginx/vhost.d
      - nginx-html:/usr/share/nginx/html
      - acme-state:/etc/acme.sh
    environment:
      - DEFAULT_EMAIL=admin@example.com
      - NGINX_PROXY_CONTAINER=nginx-proxy
      - ACME_CA_URI=https://step-ca:9000/acme/acme/directory
      - CA_BUNDLE=/etc/nginx/certs/step-ca-root.crt
    depends_on:
      - nginx-proxy
    networks:
      - nginx-proxy
    restart: unless-stopped

  # Django Application
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
    environment:
      - VIRTUAL_HOST=myapp.local
      - VIRTUAL_PORT=8000
      - LETSENCRYPT_HOST=myapp.local
      - LETSENCRYPT_EMAIL=admin@example.com
    depends_on:
      - db
      - step-ca
    networks:
      - nginx-proxy
      - backend
    restart: unless-stopped

  # Database
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    env_file:
      - ./.env.prod.db
    networks:
      - backend
    restart: unless-stopped

volumes:
  step-ca-data:
  nginx-certs:
  nginx-vhost:
  nginx-html:
  acme-state:
  postgres_data:
  static_volume:
  media_volume:

networks:
  nginx-proxy:
    driver: bridge
  backend:
    driver: bridge
```

### Option 2: Separate Files (Recommended for Production)

Keep infrastructure (Step CA, nginx-proxy) separate from application:

```bash
# Start infrastructure
docker-compose -f docker-compose.step-ca.yml up -d
docker-compose -f docker-compose.proxy.yml up -d

# Start application
docker-compose -f docker-compose.yml up -d
```

## Deployment Steps

### 1. Set Up DNS/Hosts File

For local development, add to `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows):

```
127.0.0.1   myapp.local
```

For production, configure your DNS to point to your server's IP.

### 2. Initialize Step CA

```bash
# Create network
docker network create nginx-proxy

# Start Step CA
docker-compose -f docker-compose.step-ca.yml up -d

# Wait for initialization
sleep 10

# Get root certificate
docker exec step-ca step ca root > step-ca-root.crt
```

### 3. Configure Trust (Optional for Browsers)

To avoid browser warnings, add the root certificate to your system's trust store:

**Linux:**
```bash
sudo cp step-ca-root.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

**macOS:**
```bash
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain step-ca-root.crt
```

**Windows:**
```powershell
certutil -addstore -f "ROOT" step-ca-root.crt
```

### 4. Copy Root Certificate to nginx-certs

```bash
docker run --rm -v nginx-certs:/certs -v $(pwd):/src alpine cp /src/step-ca-root.crt /certs/
```

### 5. Start nginx-proxy and acme-companion

```bash
docker-compose -f docker-compose.proxy.yml up -d
```

### 6. Start Django Application

```bash
# Run migrations
docker-compose -f docker-compose.yml run --rm web python manage.py migrate

# Collect static files
docker-compose -f docker-compose.yml run --rm web python manage.py collectstatic --no-input

# Start application
docker-compose -f docker-compose.yml up -d
```

### 7. Verify Certificate

```bash
# Wait for certificate issuance (30-60 seconds)
sleep 60

# Check certificate
docker exec nginx-proxy ls -la /etc/nginx/certs/

# Verify HTTPS
curl -v https://myapp.local
```

## Certificate Management

### Viewing Certificates

```bash
# List certificates
docker exec nginx-proxy ls -la /etc/nginx/certs/

# View certificate details
docker exec nginx-proxy openssl x509 -in /etc/nginx/certs/myapp.local.crt -text -noout
```

### Certificate Renewal

acme-companion automatically renews certificates. Check renewal logs:

```bash
# View acme-companion logs
docker logs acme-companion -f

# Force renewal (for testing)
docker restart acme-companion
```

### Certificate Lifetime

Step CA certificates have a default lifetime. To check:

```bash
docker exec step-ca step ca provisioner list
```

To change certificate lifetime:

```bash
# Set to 365 days
docker exec step-ca step ca provisioner update acme --x509-default-dur 8760h
```

## Troubleshooting

### Certificate Not Issued

**Problem:** Certificate not appearing in `/etc/nginx/certs/`

**Solutions:**

1. Check acme-companion logs:
```bash
docker logs acme-companion
```

2. Verify Step CA is accessible:
```bash
docker exec acme-companion curl -k https://step-ca:9000/health
```

3. Check environment variables:
```bash
docker exec web env | grep -E "VIRTUAL|LETSENCRYPT"
```

4. Verify root certificate is present:
```bash
docker exec acme-companion ls -la /etc/nginx/certs/step-ca-root.crt
```

### Connection Refused

**Problem:** Cannot access application via HTTPS

**Solutions:**

1. Check nginx-proxy is running:
```bash
docker ps | grep nginx-proxy
```

2. Verify port mappings:
```bash
docker port nginx-proxy
```

3. Check nginx configuration:
```bash
docker exec nginx-proxy cat /etc/nginx/conf.d/default.conf
```

### Certificate Not Trusted

**Problem:** Browser shows certificate warning

**Solutions:**

1. Install root certificate in system trust store (see deployment steps above)

2. For development, you can accept the self-signed certificate warning

3. Verify certificate chain:
```bash
openssl s_client -connect myapp.local:443 -CAfile step-ca-root.crt
```

### ACME Challenge Failed

**Problem:** acme-companion cannot complete ACME challenge

**Solutions:**

1. Check Step CA ACME endpoint:
```bash
docker exec step-ca curl -k https://localhost:9000/acme/acme/directory
```

2. Verify network connectivity:
```bash
docker exec acme-companion ping step-ca
```

3. Check CA bundle configuration:
```bash
docker exec acme-companion env | grep CA_BUNDLE
```

### nginx-proxy Not Detecting Container

**Problem:** nginx-proxy doesn't configure proxy for Django app

**Solutions:**

1. Ensure containers are on same network:
```bash
docker network inspect nginx-proxy
```

2. Check environment variables are set:
```bash
docker inspect web | grep -A 10 Env
```

3. Restart nginx-proxy:
```bash
docker restart nginx-proxy
```

## Advanced Configuration

### Custom Nginx Configuration

Create custom configuration for your application:

```bash
# Create vhost config directory
mkdir -p nginx-vhost.d
```

Create `nginx-vhost.d/myapp.local`:

```nginx
# Custom configuration for myapp.local
client_max_body_size 100M;

# Custom headers
add_header X-Custom-Header "My Value" always;

# Location-specific settings
location /static/ {
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

Mount in nginx-proxy:

```yaml
nginx-proxy:
  volumes:
    - ./nginx-vhost.d:/etc/nginx/vhost.d:ro
```

### Static File Serving via nginx-proxy

To serve static files directly through nginx-proxy:

```yaml
web:
  volumes:
    - static_volume:/app/staticfiles:ro
    - media_volume:/app/mediafiles:ro
  environment:
    - VIRTUAL_HOST=myapp.local
    - VIRTUAL_PORT=8000
    - LETSENCRYPT_HOST=myapp.local
```

Then create `nginx-vhost.d/myapp.local_location`:

```nginx
location /static/ {
    alias /app/staticfiles/;
}

location /media/ {
    alias /app/mediafiles/;
}
```

### Multiple Applications

Run multiple Django applications behind the same nginx-proxy:

```yaml
services:
  app1:
    environment:
      - VIRTUAL_HOST=app1.local
      - LETSENCRYPT_HOST=app1.local

  app2:
    environment:
      - VIRTUAL_HOST=app2.local
      - LETSENCRYPT_HOST=app2.local
```

Each will get its own certificate automatically.

### Wildcard Certificates

For wildcard certificates (e.g., `*.myapp.local`):

```yaml
web:
  environment:
    - VIRTUAL_HOST=*.myapp.local
    - LETSENCRYPT_HOST=*.myapp.local
```

**Note:** Step CA supports wildcards without DNS challenge, unlike Let's Encrypt.

## Security Best Practices

### 1. Secure Step CA

```bash
# Use strong passwords for Step CA
# Set password during initialization or update it
docker exec -it step-ca step ca provisioner update acme --password-file /path/to/password
```

### 2. Network Isolation

```yaml
# Keep Step CA in separate network from public
networks:
  nginx-proxy:
    driver: bridge
  step-ca-internal:
    driver: bridge
    internal: true
```

### 3. Restrict Access to Step CA

```yaml
step-ca:
  # Don't expose port 9000 publicly
  # Only expose to nginx-proxy network
  expose:
    - 9000
  # Remove public port mapping
  # ports:
  #   - "9000:9000"
```

### 4. Regular Updates

```bash
# Update containers regularly
docker-compose pull
docker-compose up -d
```

### 5. Monitor Logs

```bash
# Set up log aggregation
docker-compose logs -f > application.log &

# Or use log driver
services:
  web:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### 6. Backup Certificates and CA Data

```bash
# Backup Step CA data
docker run --rm -v step-ca-data:/data -v $(pwd):/backup alpine tar czf /backup/step-ca-backup.tar.gz -C /data .

# Backup certificates
docker run --rm -v nginx-certs:/data -v $(pwd):/backup alpine tar czf /backup/certs-backup.tar.gz -C /data .
```

## Comparison with Let's Encrypt Setup

| Aspect | Let's Encrypt | Step CA + nginx-proxy |
|--------|---------------|----------------------|
| Setup Complexity | Medium | Medium |
| Public Domain Required | Yes | No |
| Works Offline | No | Yes |
| Auto Configuration | Manual Nginx | Automatic |
| Certificate Renewal | Manual/Cron | Automatic |
| Internal Networks | No | Yes |
| Rate Limits | Yes | No |
| Browser Trust | Automatic | Manual (one-time) |

## Next Steps

- [Best Practices and Security](best-practices.md)
- [Troubleshooting Guide](troubleshooting.md)
- [Production Deployment](production-deployment.md)

## Resources

- [Step CA Documentation](https://smallstep.com/docs/step-ca/)
- [nginx-proxy Documentation](https://github.com/nginx-proxy/nginx-proxy)
- [acme-companion Documentation](https://github.com/nginx-proxy/acme-companion)
- [ACME Protocol](https://datatracker.ietf.org/doc/html/rfc8555)
