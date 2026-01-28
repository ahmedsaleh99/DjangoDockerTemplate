# Auto-SSL Setup Guide: Step CA, nginx-proxy, and acme-companion

## Introduction

This guide provides comprehensive instructions for setting up automatic SSL certificate management using Step CA (a private ACME server), nginx-proxy (automated reverse proxy), and acme-companion (SSL certificate automation). This solution eliminates the need for manual SSL certificate configuration and renewal, providing a production-ready setup for both staging and production environments.

**Key Benefits:**
- **Zero manual Nginx configuration** - nginx-proxy auto-configures based on container labels
- **Automatic SSL certificate issuance and renewal** - acme-companion handles certificates via ACME protocol
- **Works without public domains** - Step CA provides private ACME server (ideal for internal/staging environments)
- **Multiple applications on same host** - automatic routing and SSL for all containerized apps
- **Production-ready** - can use public CAs like Let's Encrypt or self-hosted Step CA

**Architecture Components:**
- **nginx-proxy**: Automated Nginx reverse proxy that auto-configures based on container labels
- **docker-gen**: Watches Docker containers and regenerates Nginx configs dynamically
- **acme-companion**: Automatic SSL certificate provisioning and renewal via ACME protocol
- **Step CA**: Private ACME server (no public domain required)

---

## Local Domain Setup for Testing

Since Step CA works with domains (no public domains required), you'll need to map your local IP address to a staging domain name for testing purposes.

### 1. Get Your Local IP Address

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

### 2. Map Local Domain to Hosts File

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

### 3. Verify Domain Resolution

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

---

## Django Configuration for HTTPS Proxy

### SECURE_PROXY_SSL_HEADER Setting

To run the Django app behind an HTTPS proxy, add the `SECURE_PROXY_SSL_HEADER` setting to `settings.py`:

```python
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
```

In this tuple, when `X-Forwarded-Proto` is set to `https`, the request is considered secure.

### CSRF_TRUSTED_ORIGINS Configuration

Update `CSRF_TRUSTED_ORIGINS` inside `settings.py`:

```python
CSRF_TRUSTED_ORIGINS = os.environ.get("CSRF_TRUSTED_ORIGINS").split(" ")
```

Add `CSRF_TRUSTED_ORIGINS` to your `.env.staging` and `.env.prod` files:

```bash
CSRF_TRUSTED_ORIGINS=https://yourdomain.com https://www.yourdomain.com
```

---

## Step 22: Understanding nginx-proxy + acme-companion Architecture

This setup provides a complete automated SSL solution with the following components working together:

### Component Overview

**nginx-proxy**
- Automated Nginx reverse proxy that auto-configures based on container labels
- Watches for containers with `VIRTUAL_HOST` environment variable
- Generates and reloads Nginx configurations automatically
- Routes traffic to appropriate containers based on hostname

**docker-gen**
- Watches Docker containers for changes
- Regenerates Nginx configs when containers start/stop
- Uses `nginx.tmpl` template file for config generation
- Triggers Nginx reload automatically

**acme-companion**
- Automatic SSL certificate provisioning via ACME protocol
- Monitors containers with `LETSENCRYPT_HOST` environment variable
- Handles certificate issuance and renewal automatically
- Works with both public CAs (Let's Encrypt) and private CAs (Step CA)

**Step CA**
- Private ACME server for internal/staging environments
- No public domain required
- Full control over certificate policies
- Compatible with ACME protocol

### Benefits

- **Zero manual Nginx configuration** - all routing configured via container labels
- **Automatic SSL certificate issuance and renewal** - no manual certificate management
- **Works without public domains** - ideal for internal/staging environments
- **Multiple applications on same host** - automatic routing based on hostnames
- **Production-ready** - scales from staging to production environments

---

## Step 23: Staging Environment with Step CA Auto-SSL

The repository includes `docker-compose.staging.yml` with complete Step CA integration for staging environments.

### Service Architecture

The staging setup includes the following services:

- **web**: Django application running with Gunicorn
- **db**: PostgreSQL database
- **nginx-proxy**: Nginx reverse proxy (built from `./nginx/Dockerfile`)
- **docker-gen**: Configuration generator using `./nginx/nginx.tmpl`
- **acme-companion**: SSL certificate manager (nginxproxy/acme-companion:2.6)
- **step-ca**: Private CA server (smallstep/step-ca:latest)

### Environment Configuration

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

### Environment Variables Explained

- **`VIRTUAL_HOST`**: Domain for nginx-proxy to route traffic (can be multiple comma-separated domains)
- **`VIRTUAL_PORT`**: Internal port where the application listens (8000 for Gunicorn)
- **`LETSENCRYPT_HOST`**: Domain(s) for SSL certificate issuance
- **`HTTPS_METHOD`**: Set to `redirect` to force HTTP → HTTPS redirection
- **`DOMAIN`**: Your staging domain (e.g., staging.yourdomain.local)

### Staging Docker Compose Configuration

The `docker-compose.staging.yml` file orchestrates all services for the staging environment with automatic SSL:

**`docker-compose.staging.yml`:**

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
      - ./.env.staging
    depends_on:
      - db
  
  db:
    image: postgres:18.1-trixie
    volumes:
      - postgres_data:/var/lib/postgresql
    env_file:
      - ./.env.staging.db
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 5s
      timeout: 5s
      retries: 10

  nginx-proxy:
    container_name: nginx-proxy
    build: ./nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes_from:
      - web
    volumes:
      - vhost:/etc/nginx/vhost.d
      - conf:/etc/nginx/conf.d
      - html:/usr/share/nginx/html
      - certs:/etc/nginx/certs:ro
    depends_on:
      - web

  docker-gen:
    image: nginxproxy/docker-gen
    container_name: nginx-proxy-gen
    command: -notify-sighup nginx-proxy -watch -wait 5s:30s /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf
    volumes_from:
      - nginx-proxy
    volumes:
      - ./nginx/nginx.tmpl:/etc/docker-gen/templates/nginx.tmpl:ro
      - /var/run/docker.sock:/tmp/docker.sock:ro
    depends_on:
      - nginx-proxy

  acme-companion:
    image: nginxproxy/acme-companion:2.6
    container_name: nginx-proxy-acme
    environment:
      - NGINX_PROXY_CONTAINER=nginx-proxy-gen
      - DEFAULT_EMAIL=test@test.com
      - ACME_CA_URI=https://step-ca:9000/acme/acme/directory
      - CA_BUNDLE=/home/step/certs/root_ca.crt
    volumes_from:
      - nginx-proxy
    volumes:
      - certs:/etc/nginx/certs:rw
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - acme:/etc/acme.sh
      - step-data:/home/step:ro
    depends_on:
      - nginx-proxy
      - step-ca

  step-ca:
    image: smallstep/step-ca:latest
    container_name: step-ca
    restart: always
    environment:
      - DOCKER_STEPCA_INIT_NAME=Smallstep
      - DOCKER_STEPCA_INIT_DNS_NAMES=localhost,step-ca
      - DOCKER_STEPCA_INIT_REMOTE_MANAGEMENT=true
      - DOCKER_STEPCA_INIT_ACME=true
    volumes:
      - step-data:/home/step
    ports:
      - "9000:9000"

volumes:
  step-data:
  postgres_data:
  static_volume:
  media_volume:
  certs:
  html:
  vhost:
  conf:
  acme:
```

### Services Explained

#### `web` Service
**Role:** Django application with Gunicorn WSGI server
- **`build`**: Builds from production Dockerfile for optimized image
- **`command`**: Runs Gunicorn with 4 workers binding to port 8000
- **`volumes`**: Shares static and media files with nginx-proxy
- **`expose`**: Makes port 8000 available only to linked containers (not to host)
- **`env_file`**: Loads environment variables from .env.staging
- **`depends_on`**: Ensures database starts before web application

#### `db` Service
**Role:** PostgreSQL database with persistent storage
- **`image`**: Official PostgreSQL 18.1-trixie image
- **`volumes`**: Persists database data across container restarts
- **`env_file`**: Loads database credentials from .env.staging.db
- **`healthcheck`**: Monitors database readiness for dependent services

#### `nginx-proxy` Service
**Role:** Automated reverse proxy that auto-configures based on container environment variables
- **`build`**: Custom Nginx image with vhost.d and custom.conf
- **`restart`**: Always restarts on failure for high availability
- **`ports`**: Exposes ports 80 (HTTP) and 443 (HTTPS) to host
- **`volumes_from`**: Inherits volumes from web container (static/media)
- **`volumes`**: Stores SSL certs, vhost configs, and HTML content
- **`certs:ro`**: Read-only access to certificates (written by acme-companion)

#### `docker-gen` Service
**Role:** Watches Docker containers and regenerates Nginx configuration dynamically
- **`image`**: Official nginxproxy/docker-gen image
- **`command`**: 
  - `-notify-sighup nginx-proxy`: Sends reload signal to Nginx when config changes
  - `-watch`: Continuously monitors Docker events
  - `-wait 5s:30s`: Waits 5s minimum, 30s maximum before regenerating config
  - Generates `default.conf` from `nginx.tmpl` template
- **`volumes`**: 
  - Shares volumes with nginx-proxy for config updates
  - Mounts nginx.tmpl template for config generation
  - Mounts Docker socket to watch container changes

#### `acme-companion` Service
**Role:** Automatic SSL certificate issuance and renewal via ACME protocol
- **`image`**: nginxproxy/acme-companion:2.6 for certificate automation
- **`environment`**:
  - `NGINX_PROXY_CONTAINER`: Points to docker-gen container for coordination
  - `DEFAULT_EMAIL`: Email for certificate expiration notifications
  - `ACME_CA_URI`: Step CA ACME server URL (private CA)
  - `CA_BUNDLE`: Path to Step CA root certificate for trust
- **`volumes`**:
  - `certs:rw`: Read-write access to store issued certificates
  - Docker socket: Monitors containers for LETSENCRYPT_* variables
  - `acme`: Stores acme.sh client data and account keys
  - `step-data:ro`: Read-only access to Step CA certificates

#### `step-ca` Service
**Role:** Private ACME Certificate Authority (eliminates need for public domain)
- **`image`**: Official Smallstep Step CA image
- **`restart`**: Always restarts to ensure CA availability
- **`environment`**:
  - `DOCKER_STEPCA_INIT_NAME`: CA name (Smallstep)
  - `DOCKER_STEPCA_INIT_DNS_NAMES`: DNS names for CA access
  - `DOCKER_STEPCA_INIT_REMOTE_MANAGEMENT`: Enables API management
  - `DOCKER_STEPCA_INIT_ACME`: Enables ACME protocol support
- **`volumes`**: Persists CA keys, certificates, and configuration
- **`ports`**: Exposes port 9000 for ACME protocol and management

### Building and Running Staging

**Build and start all services:**

```bash
docker-compose -f docker-compose.staging.yml up -d --build
```

**Run database migrations:**

```bash
docker-compose -f docker-compose.staging.yml exec web python manage.py migrate --noinput
```

**Collect static files:**

```bash
docker-compose -f docker-compose.staging.yml exec web python manage.py collectstatic --no-input --clear
```

**Create Django superuser:**

```bash
docker-compose -f docker-compose.staging.yml exec web python manage.py createsuperuser
```

### Accessing the Staging Environment

- **HTTP**: `http://yourdomain.com` (automatically redirects to HTTPS)
- **HTTPS**: `https://yourdomain.com`
- **Step CA UI**: `https://localhost:9000`

---

## Nginx Configuration for nginx-proxy

The nginx-proxy setup requires custom configurations for serving static files and setting proxy-wide options.

### Directory Structure

Your "nginx" directory should look like this:

```
└── nginx
    ├── Dockerfile
    ├── custom.conf
    └── vhost.d
        └── default
```

### Static and Media Files Configuration

Create a directory called `vhost.d` in the nginx folder. Add a file called `default` inside that directory to serve static and media files:

**`nginx/vhost.d/default`:**

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

**How it works:**
- Requests matching these patterns are served directly from static/media folders
- They won't be proxied to other containers
- The web and nginx-proxy containers share volumes for static and media files:
  ```
  static_volume:/home/app/web/staticfiles
  media_volume:/home/app/web/mediafiles
  ```

### Custom Proxy-Wide Configuration

Add a `custom.conf` file to the "nginx" folder for proxy-wide settings:

**`nginx/custom.conf`:**

```nginx
client_max_body_size 10M;
```

This sets the maximum upload size for all proxied applications.

### Nginx Dockerfile

Update `nginx/Dockerfile` to copy custom configurations:

```dockerfile
FROM nginxproxy/nginx-proxy
COPY vhost.d/default /etc/nginx/vhost.d/default
COPY custom.conf /etc/nginx/conf.d/custom.conf
```

**Important:** Remove the old `nginx.conf` file as nginx-proxy generates configuration automatically.

---

## Step 24: Production with Automatic SSL Certificate Management

The `docker-compose.prod.yml` includes the same nginx-proxy + acme-companion architecture optimized for production environments.

### Key Differences from Staging

- Uses `.env.prod` and `.env.prod.db` instead of `.env.staging*`
- Can use public CA (Let's Encrypt) or self-hosted Step CA
- Production-grade environment variables and secrets management
- Enhanced security configurations

### Production Environment Configuration

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

### Production Docker Compose Configuration

The `docker-compose.prod.yml` file is similar to staging but configured for production use:

**`docker-compose.prod.yml`:**

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
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 5s
      timeout: 5s
      retries: 10

  nginx-proxy:
    container_name: nginx-proxy
    build: ./nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes_from:
      - web
    volumes:
      - vhost:/etc/nginx/vhost.d
      - conf:/etc/nginx/conf.d
      - html:/usr/share/nginx/html
      - certs:/etc/nginx/certs:ro
    depends_on:
      - web

  docker-gen:
    image: nginxproxy/docker-gen
    container_name: nginx-proxy-gen
    command: -notify-sighup nginx-proxy -watch -wait 5s:30s /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf
    volumes_from:
      - nginx-proxy
    volumes:
      - ./nginx/nginx.tmpl:/etc/docker-gen/templates/nginx.tmpl:ro
      - /var/run/docker.sock:/tmp/docker.sock:ro
    depends_on:
      - nginx-proxy

  acme-companion:
    image: nginxproxy/acme-companion:2.6
    container_name: nginx-proxy-acme
    environment:
      - NGINX_PROXY_CONTAINER=nginx-proxy-gen
      - DEFAULT_EMAIL=prod@prod.com
      # Replace ACME_CA_URI with your CA URL if not using self-hosted CA
      - ACME_CA_URI=https://step-ca:9000/acme/acme/directory
      # Remove CA_BUNDLE line if not using self-hosted CA
      - CA_BUNDLE=/home/step/certs/root_ca.crt
    volumes_from:
      - nginx-proxy
    volumes:
      - certs:/etc/nginx/certs:rw
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - acme:/etc/acme.sh
      - step-data:/home/step:ro
    depends_on:
      - nginx-proxy
      - step-ca

  # Remove step-ca service if not using self-hosted CA
  step-ca:
    image: smallstep/step-ca:latest
    container_name: step-ca
    restart: always
    environment:
      - DOCKER_STEPCA_INIT_NAME=Smallstep
      - DOCKER_STEPCA_INIT_DNS_NAMES=localhost,step-ca
      - DOCKER_STEPCA_INIT_REMOTE_MANAGEMENT=true
      - DOCKER_STEPCA_INIT_ACME=true
    volumes:
      - step-data:/home/step
    ports:
      - "9000:9000"

volumes:
  step-data:
  postgres_data:
  static_volume:
  media_volume:
  certs:
  html:
  vhost:
  conf:
  acme:
```

### Production Services Configuration

The production configuration uses the same services as staging with these key differences:

#### Production-Specific Environment Files
- **`.env.prod`**: Production Django settings (DEBUG=0, production SECRET_KEY)
- **`.env.prod.db`**: Production database credentials

#### Service Differences from Staging

**`acme-companion` Service (Production)**
- **`DEFAULT_EMAIL`**: Set to production email for certificate notifications
- **`ACME_CA_URI`**: Can be switched to Let's Encrypt by removing this line
- **`CA_BUNDLE`**: Optional - remove if using public CA like Let's Encrypt

**Comments in Production File:**
- Includes guidance for switching between self-hosted Step CA and public CA
- Shows which lines to remove/modify for Let's Encrypt integration
- Security reminders for SECRET_KEY and database passwords

All other services (`web`, `db`, `nginx-proxy`, `docker-gen`, `step-ca`) function identically to staging but use production environment variables and credentials.

### Certificate Authority Options

#### Option 1: Self-Hosted Step CA

**Configuration:**
- Keep `step-ca` service in docker-compose
- Set `ACME_CA_URI=https://step-ca:9000/acme/acme/directory`
- Set `CA_BUNDLE=/home/step/certs/root_ca.crt`

**Benefits:**
- Full control over certificate policies
- No external dependencies
- Works in air-gapped environments

#### Option 2: Public CA (Let's Encrypt)

**Configuration:**
- Remove `step-ca` service from docker-compose
- Remove `ACME_CA_URI` from acme-companion environment
- Remove `CA_BUNDLE` from acme-companion environment
- Update `DEFAULT_EMAIL` in acme-companion to your email address

**Benefits:**
- Publicly trusted certificates
- No infrastructure to manage
- Free certificate issuance

### Building and Running Production

**Build and start all services:**

```bash
docker-compose -f docker-compose.prod.yml up -d --build
```

**First-time setup:**

```bash
# Run database migrations
docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput

# Collect static files
docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input --clear

# Create Django superuser
docker-compose -f docker-compose.prod.yml exec web python manage.py createsuperuser
```

### Verify SSL Certificate

Check acme-companion logs to verify certificate issuance:

```bash
docker-compose -f docker-compose.prod.yml logs acme-companion
```

### Certificate Renewal

Certificates automatically renew via acme-companion. **No manual intervention needed.**

The acme-companion service monitors certificate expiration and requests renewal before expiry (typically 30 days before expiration for Let's Encrypt certificates).

---

## Nginx Template Configuration

The [`nginx/nginx.tmpl`](https://raw.githubusercontent.com/nginx-proxy/nginx-proxy/main/nginx.tmpl) file is used by docker-gen to generate Nginx configurations dynamically.

### Template Functionality

The template automatically:
- Auto-detects containers with `VIRTUAL_HOST` labels
- Configures upstream servers for load balancing
- Sets up SSL with certificates from acme-companion
- Handles HTTP to HTTPS redirects (when `HTTPS_METHOD=redirect`)
- Supports multiple applications on the same host
- Configures proxy headers for proper request forwarding

### Configuration Management

**No manual editing required** - all configuration happens via environment variables in your Django container:

- `VIRTUAL_HOST` - controls routing
- `VIRTUAL_PORT` - sets upstream port
- `LETSENCRYPT_HOST` - triggers SSL certificate issuance
- `HTTPS_METHOD` - controls redirect behavior

The template is downloaded automatically by nginx-proxy and used by docker-gen to generate live configurations.

---

## Troubleshooting Auto-SSL

### Viewing Logs

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

### Inspecting Configuration

**View generated Nginx config:**
```bash
docker exec nginx-proxy cat /etc/nginx/conf.d/default.conf
```

This shows the actual Nginx configuration generated by docker-gen.

### Common Issues

**Certificate not issuing:**
- Verify `VIRTUAL_HOST` matches `LETSENCRYPT_HOST`
- Check acme-companion can reach Step CA (network connectivity)
- Ensure Step CA is initialized (check logs for "step-ca is ready")
- Verify CA bundle path if using self-hosted CA
- Check that DNS/hosts file entries are correct

**HTTP works but HTTPS fails:**
- Check certificate was issued: `docker logs nginx-proxy-acme`
- Verify SSL certificate location: `docker exec nginx-proxy ls -la /etc/nginx/certs/`
- Ensure `HTTPS_METHOD=redirect` is set if you want automatic redirects

**Static files not serving:**
- Verify volumes are shared between web and nginx-proxy containers
- Check `vhost.d/default` file exists in nginx container
- Run `collectstatic` command to populate static files
- Inspect nginx logs for 404 errors

**Multiple domains not working:**
- Ensure both `VIRTUAL_HOST` and `LETSENCRYPT_HOST` have all domains comma-separated
- Example: `VIRTUAL_HOST=example.com,www.example.com`
- Example: `LETSENCRYPT_HOST=example.com,www.example.com`

### Testing SSL Certificates

**Test with curl:**
```bash
curl -I https://yourdomain.com
```

**Check certificate details:**
```bash
openssl s_client -connect yourdomain.com:443 -servername yourdomain.com
```

**Verify certificate expiration:**
```bash
echo | openssl s_client -servername yourdomain.com -connect yourdomain.com:443 2>/dev/null | openssl x509 -noout -dates
```

---

## Additional Resources

- [nginx-proxy - Automated Nginx Reverse Proxy for Docker](https://github.com/nginx-proxy/nginx-proxy)
- [acme-companion - LetsEncrypt companion for nginx-proxy](https://github.com/nginx-proxy/acme-companion)
- [Step CA - Private ACME Server](https://github.com/smallstep/certificates)
- [ACME Protocol Specification](https://datatracker.ietf.org/doc/html/rfc8555)

---

## Summary

This guide covered:
- Setting up local domains for SSL testing
- Configuring Django for HTTPS proxy environments
- Understanding nginx-proxy and acme-companion architecture
- Staging environment deployment with Step CA
- Custom Nginx configurations for static file serving
- Production deployment with automatic SSL management
- Troubleshooting common SSL and proxy issues

With this setup, you have:
- ✅ Automated reverse proxy configuration via nginx-proxy
- ✅ Automatic SSL certificate management via acme-companion
- ✅ Private CA option for staging/internal environments via Step CA
- ✅ Production-ready SSL with Let's Encrypt support
- ✅ Zero-downtime certificate renewals
