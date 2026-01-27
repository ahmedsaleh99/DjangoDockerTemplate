# SSL/HTTPS Setup with Let's Encrypt

This guide covers setting up SSL/HTTPS for your Django application using Let's Encrypt and Certbot with Docker and Nginx.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Understanding SSL/TLS](#understanding-ssltls)
- [Let's Encrypt Overview](#lets-encrypt-overview)
- [Setup Methods](#setup-methods)
- [Method 1: Using Certbot Docker Container](#method-1-using-certbot-docker-container)
- [Method 2: Using Certbot Standalone](#method-2-using-certbot-standalone)
- [Nginx SSL Configuration](#nginx-ssl-configuration)
- [Auto-Renewal](#auto-renewal)
- [Testing SSL Configuration](#testing-ssl-configuration)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before setting up SSL, ensure you have:

- A registered domain name
- DNS records pointing to your server's IP address
- Port 80 and 443 open on your firewall
- A working Django application with Nginx
- Docker and Docker Compose installed

### Verify DNS Configuration

```bash
# Check if your domain points to your server
nslookup your_domain.com

# Or use dig
dig your_domain.com +short
```

The output should show your server's IP address.

## Understanding SSL/TLS

**SSL (Secure Sockets Layer)** and **TLS (Transport Layer Security)** encrypt data between client and server.

Benefits:
- 🔒 **Encrypted communication** - Protects sensitive data
- ✅ **Authentication** - Verifies your website's identity
- 🔍 **SEO boost** - Google favors HTTPS sites
- 🛡️ **Browser trust** - Avoids "Not Secure" warnings
- 📱 **Required for modern features** - Service Workers, HTTP/2, etc.

## Let's Encrypt Overview

**Let's Encrypt** is a free, automated, and open Certificate Authority.

Key features:
- ✅ Free SSL/TLS certificates
- 🔄 Automated certificate issuance and renewal
- 🌍 Trusted by all major browsers
- ⏰ 90-day certificate validity (encourages automation)

## Setup Methods

We'll cover two methods:
1. **Certbot Docker Container** (Recommended) - Fully containerized
2. **Certbot Standalone** - Traditional approach

## Method 1: Using Certbot Docker Container

This method keeps everything containerized and is easier to manage.

### Step 1: Update Docker Compose

Update `docker-compose.prod.yml`:

```yaml
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
    depends_on:
      - db

  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    env_file:
      - ./.env.prod.db

  nginx:
    build: ./nginx
    volumes:
      - static_volume:/home/app/web/staticfiles
      - media_volume:/home/app/web/mediafiles
      - certbot_conf:/etc/letsencrypt
      - certbot_www:/var/www/certbot
    ports:
      - 80:80
      - 443:443
    depends_on:
      - web
    command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g \"daemon off;\"'"

  certbot:
    image: certbot/certbot:latest
    volumes:
      - certbot_conf:/etc/letsencrypt
      - certbot_www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

volumes:
  postgres_data:
  static_volume:
  media_volume:
  certbot_conf:
  certbot_www:
```

**Key additions:**
- `certbot_conf` volume for certificates
- `certbot_www` volume for ACME challenge
- `certbot` service for certificate management
- Nginx reload command for certificate renewal

### Step 2: Initial Nginx Configuration

Create `nginx/nginx.conf` for initial setup:

```nginx
upstream django {
    server web:8000;
}

server {
    listen 80;
    server_name your_domain.com www.your_domain.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name your_domain.com www.your_domain.com;

    ssl_certificate /etc/letsencrypt/live/your_domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your_domain.com/privkey.pem;

    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://django;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
        alias /home/app/web/staticfiles/;
    }

    location /media/ {
        alias /home/app/web/mediafiles/;
    }

    client_max_body_size 100M;
}
```

### Step 3: Initial Certificate Request

Create `init-letsencrypt.sh` script:

```bash
#!/bin/bash

if ! [ -x "$(command -v docker-compose)" ]; then
  echo 'Error: docker-compose is not installed.' >&2
  exit 1
fi

domains=(your_domain.com www.your_domain.com)
rsa_key_size=4096
data_path="./certbot"
email="your-email@example.com" # Adding a valid address is strongly recommended
staging=0 # Set to 1 if you're testing your setup to avoid hitting request limits

if [ -d "$data_path" ]; then
  read -p "Existing data found for $domains. Continue and replace existing certificate? (y/N) " decision
  if [ "$decision" != "Y" ] && [ "$decision" != "y" ]; then
    exit
  fi
fi

if [ ! -e "$data_path/conf/options-ssl-nginx.conf" ] || [ ! -e "$data_path/conf/ssl-dhparams.pem" ]; then
  echo "### Downloading recommended TLS parameters ..."
  mkdir -p "$data_path/conf"
  curl -s https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf > "$data_path/conf/options-ssl-nginx.conf"
  curl -s https://raw.githubusercontent.com/certbot/certbot/master/certbot/certbot/ssl-dhparams.pem > "$data_path/conf/ssl-dhparams.pem"
  echo
fi

echo "### Creating dummy certificate for $domains ..."
path="/etc/letsencrypt/live/$domains"
mkdir -p "$data_path/conf/live/$domains"
docker-compose -f docker-compose.prod.yml run --rm --entrypoint "\
  openssl req -x509 -nodes -newkey rsa:$rsa_key_size -days 1\
    -keyout '$path/privkey.pem' \
    -out '$path/fullchain.pem' \
    -subj '/CN=localhost'" certbot
echo

echo "### Starting nginx ..."
docker-compose -f docker-compose.prod.yml up --force-recreate -d nginx
echo

echo "### Deleting dummy certificate for $domains ..."
docker-compose -f docker-compose.prod.yml run --rm --entrypoint "\
  rm -Rf /etc/letsencrypt/live/$domains && \
  rm -Rf /etc/letsencrypt/archive/$domains && \
  rm -Rf /etc/letsencrypt/renewal/$domains.conf" certbot
echo

echo "### Requesting Let's Encrypt certificate for $domains ..."
# Join $domains to -d args
domain_args=""
for domain in "${domains[@]}"; do
  domain_args="$domain_args -d $domain"
done

# Select appropriate email arg
case "$email" in
  "") email_arg="--register-unsafely-without-email" ;;
  *) email_arg="--email $email" ;;
esac

# Enable staging mode if needed
if [ $staging != "0" ]; then staging_arg="--staging"; fi

docker-compose -f docker-compose.prod.yml run --rm --entrypoint "\
  certbot certonly --webroot -w /var/www/certbot \
    $staging_arg \
    $email_arg \
    $domain_args \
    --rsa-key-size $rsa_key_size \
    --agree-tos \
    --force-renewal" certbot
echo

echo "### Reloading nginx ..."
docker-compose -f docker-compose.prod.yml exec nginx nginx -s reload
```

Make it executable:

```bash
chmod +x init-letsencrypt.sh
```

### Step 4: Run Initial Setup

```bash
# Edit the script with your domain and email
nano init-letsencrypt.sh

# Run the script
sudo ./init-letsencrypt.sh
```

This script will:
1. Download recommended TLS parameters
2. Create a dummy certificate
3. Start Nginx
4. Delete the dummy certificate
5. Request a real certificate from Let's Encrypt
6. Reload Nginx

### Step 5: Update Nginx Volume Mounts

Ensure Nginx can access the certificates by updating volume mounts in `docker-compose.prod.yml` if needed.

## Method 2: Using Certbot Standalone

This method installs Certbot directly on the host system.

### Step 1: Install Certbot

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install certbot python3-certbot-nginx

# CentOS/RHEL
sudo yum install certbot python3-certbot-nginx
```

### Step 2: Stop Nginx

```bash
docker-compose -f docker-compose.prod.yml stop nginx
```

### Step 3: Request Certificate

```bash
sudo certbot certonly --standalone -d your_domain.com -d www.your_domain.com
```

Follow the prompts:
- Enter your email address
- Agree to Terms of Service
- Choose whether to share email with EFF

### Step 4: Copy Certificates to Docker

```bash
# Create certificate directory
sudo mkdir -p /etc/letsencrypt

# Certificates are stored in /etc/letsencrypt/live/your_domain.com/
```

### Step 5: Update Docker Compose

Mount certificates in `docker-compose.prod.yml`:

```yaml
nginx:
  build: ./nginx
  volumes:
    - static_volume:/home/app/web/staticfiles
    - media_volume:/home/app/web/mediafiles
    - /etc/letsencrypt:/etc/letsencrypt:ro
  ports:
    - 80:80
    - 443:443
  depends_on:
    - web
```

### Step 6: Start Nginx

```bash
docker-compose -f docker-compose.prod.yml up -d nginx
```

## Nginx SSL Configuration

### Complete SSL Configuration

Here's a production-ready `nginx/nginx.conf` with SSL:

```nginx
upstream django {
    server web:8000;
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name your_domain.com www.your_domain.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS Server
server {
    listen 443 ssl http2;
    server_name your_domain.com www.your_domain.com;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/your_domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your_domain.com/privkey.pem;
    
    # SSL Security Settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    
    # SSL Session Settings
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/your_domain.com/chain.pem;
    
    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Proxy Settings
    location / {
        proxy_pass http://django;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $host;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Static Files
    location /static/ {
        alias /home/app/web/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Media Files
    location /media/ {
        alias /home/app/web/mediafiles/;
    }

    # File Upload Limit
    client_max_body_size 100M;
    
    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss application/json;
    gzip_disable "MSIE [1-6]\.";
}
```

### SSL Security Best Practices

1. **Use Modern TLS Versions**: Only TLS 1.2 and 1.3
2. **Strong Cipher Suites**: Prioritize modern, secure ciphers
3. **HSTS**: Force HTTPS for all future visits
4. **OCSP Stapling**: Improve SSL handshake performance
5. **Security Headers**: Protect against common attacks

## Auto-Renewal

Let's Encrypt certificates expire after 90 days. Set up auto-renewal.

### Method 1: Docker-based Auto-renewal

Already configured in the Docker Compose setup above. The Certbot container runs renewal checks twice daily.

Verify the renewal process:

```bash
docker-compose -f docker-compose.prod.yml run --rm certbot renew --dry-run
```

### Method 2: Cron-based Auto-renewal

If using standalone Certbot:

```bash
# Edit crontab
sudo crontab -e

# Add this line to run twice daily
0 0,12 * * * certbot renew --quiet --post-hook "docker-compose -f /path/to/docker-compose.prod.yml exec nginx nginx -s reload"
```

### Test Renewal

```bash
# Dry run (doesn't actually renew)
sudo certbot renew --dry-run

# Or with Docker
docker-compose -f docker-compose.prod.yml run --rm certbot renew --dry-run
```

## Django HTTPS Settings

Update `settings.py` for HTTPS:

```python
# HTTPS Settings (production only)
if not DEBUG:
    # Security settings
    SECURE_SSL_REDIRECT = True
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
    SECURE_BROWSER_XSS_FILTER = True
    SECURE_CONTENT_TYPE_NOSNIFF = True
    SECURE_HSTS_SECONDS = 31536000
    SECURE_HSTS_INCLUDE_SUBDOMAINS = True
    SECURE_HSTS_PRELOAD = True
    
    # Proxy settings
    SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
    
    # Additional security headers
    X_FRAME_OPTIONS = 'DENY'
```

## Testing SSL Configuration

### 1. SSL Labs Test

Visit [SSL Labs](https://www.ssllabs.com/ssltest/) and enter your domain. Aim for an A+ rating.

### 2. Command Line Test

```bash
# Check certificate
openssl s_client -connect your_domain.com:443 -servername your_domain.com

# Check specific cipher
openssl s_client -connect your_domain.com:443 -tls1_2

# Verify certificate chain
curl -vI https://your_domain.com
```

### 3. Browser Test

1. Visit `https://your_domain.com`
2. Click the padlock icon
3. Verify certificate details
4. Check for any mixed content warnings

### 4. Security Headers Test

Visit [Security Headers](https://securityheaders.com/) and check your domain.

## Certificate Management

### View Certificate Details

```bash
# Using Docker
docker-compose -f docker-compose.prod.yml run --rm certbot certificates

# Or directly
sudo certbot certificates
```

### Manual Renewal

```bash
# Force renewal (Docker)
docker-compose -f docker-compose.prod.yml run --rm certbot renew --force-renewal

# Standalone
sudo certbot renew --force-renewal
```

### Revoke Certificate

```bash
# If you need to revoke (security breach, etc.)
docker-compose -f docker-compose.prod.yml run --rm certbot revoke --cert-path /etc/letsencrypt/live/your_domain.com/cert.pem
```

## Troubleshooting

### Certificate Request Failed

**Problem**: "Failed to obtain certificate"

**Solutions**:
```bash
# Check DNS
dig your_domain.com +short

# Check if port 80 is accessible
curl http://your_domain.com/.well-known/acme-challenge/test

# Check Nginx logs
docker-compose -f docker-compose.prod.yml logs nginx

# Try with verbose output
docker-compose -f docker-compose.prod.yml run --rm certbot certonly --webroot -w /var/www/certbot -d your_domain.com --verbose
```

### Mixed Content Warnings

**Problem**: HTTPS site loading HTTP resources

**Solutions**:
1. Update all URLs to use HTTPS
2. Use protocol-relative URLs: `//example.com/script.js`
3. Check Django settings: `SECURE_SSL_REDIRECT = True`

### Nginx Won't Start with SSL

**Problem**: "Certificate file not found"

**Solutions**:
```bash
# Check certificate path
docker-compose -f docker-compose.prod.yml exec nginx ls -la /etc/letsencrypt/live/

# Verify certificate permissions
docker-compose -f docker-compose.prod.yml exec nginx cat /etc/letsencrypt/live/your_domain.com/fullchain.pem

# Check Nginx configuration
docker-compose -f docker-compose.prod.yml exec nginx nginx -t
```

### Rate Limits

Let's Encrypt has rate limits:
- 50 certificates per registered domain per week
- 5 duplicate certificates per week

**Solution**: Use staging environment for testing:
```bash
# Add --staging flag
docker-compose -f docker-compose.prod.yml run --rm certbot certonly --webroot -w /var/www/certbot -d your_domain.com --staging
```

### Auto-renewal Not Working

**Problem**: Certificates expiring

**Solutions**:
```bash
# Test renewal
docker-compose -f docker-compose.prod.yml run --rm certbot renew --dry-run

# Check Certbot container logs
docker-compose -f docker-compose.prod.yml logs certbot

# Manually renew
docker-compose -f docker-compose.prod.yml run --rm certbot renew --force-renewal

# Restart Nginx
docker-compose -f docker-compose.prod.yml exec nginx nginx -s reload
```

## Wildcard Certificates

For subdomains (`*.your_domain.com`):

```bash
docker-compose -f docker-compose.prod.yml run --rm certbot certonly \
  --manual \
  --preferred-challenges=dns \
  --email your-email@example.com \
  --server https://acme-v02.api.letsencrypt.org/directory \
  --agree-tos \
  -d your_domain.com \
  -d *.your_domain.com
```

This requires DNS verification (adding TXT records).

## Additional Security

### 1. Certificate Transparency Monitoring

Monitor your certificates at:
- [crt.sh](https://crt.sh/)
- [Certificate Transparency Search](https://transparencyreport.google.com/https/certificates)

### 2. HSTS Preload

Submit your domain to [HSTS Preload List](https://hstspreload.org/)

### 3. CAA Records

Add CAA DNS records to restrict which CAs can issue certificates:

```
your_domain.com. CAA 0 issue "letsencrypt.org"
your_domain.com. CAA 0 issuewild "letsencrypt.org"
```

## Next Steps

- [Best Practices and Security](best-practices.md)
- [Troubleshooting Guide](troubleshooting.md)
- [Production Deployment](production-deployment.md)
