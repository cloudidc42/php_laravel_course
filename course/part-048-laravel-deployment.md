# Part 048: Laravel Deployment — Deploy สู่ Production

**ระดับ: มืออาชีพ (Professional)**
**เวลาเรียน: 5-6 ชั่วโมง**

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เตรียม Server สำหรับ Laravel Production
- Deploy ด้วย Laravel Forge
- ทำ Zero-Downtime Deployment ด้วย Envoyer
- Deploy ด้วย Docker และ Docker Compose
- ตั้งค่า CI/CD ด้วย GitHub Actions
- Deploy Laravel App ไปสู่ Production อย่างมั่นใจ

---

## 1. Server Requirements

### 1.1 System Requirements

```
PHP >= 8.1
BCMath Extension
Ctype Extension
cURL Extension
DOM Extension
Fileinfo Extension
JSON Extension
Mbstring Extension
OpenSSL Extension
PCRE Extension
PDO Extension
Tokenizer Extension
XML Extension

Web Server: Nginx หรือ Apache
Database: MySQL 8.0+ หรือ PostgreSQL 13+
Cache: Redis 6.0+
Queue: Redis หรือ Beanstalkd
Storage: Minimum 10GB SSD
RAM: Minimum 2GB (4GB แนะนำ)
```

### 1.2 Nginx Configuration

```nginx
# /etc/nginx/sites-available/laravel
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;
    
    root /var/www/html/public;
    index index.php;
    
    # SSL
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # Gzip
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    gzip_min_length 1024;
    
    # Logs
    access_log /var/log/nginx/laravel-access.log;
    error_log /var/log/nginx/laravel-error.log;
    
    charset utf-8;
    
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }
    
    error_page 404 /index.php;
    
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        
        fastcgi_read_timeout 300;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
    }
    
    # Deny .htaccess access
    location ~ /\.ht {
        deny all;
    }
    
    # Cache static assets
    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
    
    # Block access to sensitive files
    location ~ /\.(env|git|svn) {
        deny all;
        return 404;
    }
}
```

### 1.3 PHP-FPM Configuration

```ini
; /etc/php/8.2/fpm/pool.d/www.conf
[www]
user = www-data
group = www-data
listen = /var/run/php/php8.2-fpm.sock
listen.owner = www-data
listen.group = www-data

pm = dynamic
pm.max_children = 50
pm.start_servers = 10
pm.min_spare_servers = 5
pm.max_spare_servers = 20
pm.max_requests = 500

php_admin_value[error_log] = /var/log/php/php-fpm-error.log
php_admin_flag[log_errors] = on

; Security
php_admin_value[open_basedir] = /var/www/html
php_admin_value[disable_functions] = exec,passthru,shell_exec,system,proc_open
```

---

## 2. Laravel Forge

Forge เป็นบริการจาก Laravel ที่ช่วย Provision และจัดการ Server

### 2.1 สิ่งที่ Forge จัดการ

```
- Server Provisioning (AWS, DigitalOcean, Linode, Vultr, Hetzner)
- Nginx Configuration
- PHP-FPM Management
- SSL Certificates (Let's Encrypt)
- MySQL/PostgreSQL/Redis Setup
- Queue Workers
- Scheduler
- Deployment Scripts
```

### 2.2 Forge Deploy Script

```bash
#!/bin/bash
# Forge Deploy Script

# ไปยัง Project Directory
cd /home/forge/example.com

# Pull changes
git pull origin main

# Install Composer dependencies
$FORGE_COMPOSER install --no-interaction --prefer-dist --optimize-autoloader --no-dev

# Install npm dependencies และ build assets
npm ci
npm run build

# Clear and cache configs
$FORGE_PHP artisan config:cache
$FORGE_PHP artisan route:cache
$FORGE_PHP artisan view:cache
$FORGE_PHP artisan event:cache

# Run migrations
$FORGE_PHP artisan migrate --force

# Restart queue workers
$FORGE_PHP artisan queue:restart

# Reload PHP-FPM
sudo -S service php8.2-fpm reload

# Clear Application Cache
$FORGE_PHP artisan cache:clear

echo "Deployment completed at $(date)"
```

### 2.3 Environment Management ใน Forge

```bash
# .env สำหรับ Production
APP_NAME="My Application"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://example.com

LOG_CHANNEL=stack
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=error

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=production_db
DB_USERNAME=forge
DB_PASSWORD=your-secure-password

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis
SESSION_LIFETIME=120

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=smtp.mailgun.org
MAIL_PORT=587

BROADCAST_DRIVER=pusher
```

---

## 3. Envoyer — Zero Downtime Deployment

Envoyer ทำให้ Deploy โดยไม่มี Downtime ด้วยการ Atomic Deployments

### 3.1 วิธีการทำงาน

```
Project Directory Structure:
/home/forge/example.com/
├── current -> releases/20240115120000/   (Symlink ไปยัง current release)
├── releases/
│   ├── 20240115120000/  (Current)
│   ├── 20240114120000/  (Previous)
│   └── 20240113120000/  (Older)
└── storage/             (Shared across releases)
```

### 3.2 Envoyer Deployment Process

```
1. Clone Repository ไปยัง release directory ใหม่
2. Install Composer Dependencies
3. Run Build Hooks (npm install, npm run build, artisan commands)
4. Link Shared Directories (storage, .env)
5. Run Deployment Hooks
6. Activate Release (สลับ symlink current -> ใหม่)
7. Purge Old Releases
```

### 3.3 Deployment Hooks

```bash
# Before Activation
cd {{ release }}
php artisan config:cache
php artisan route:cache
php artisan view:cache

# หลังจาก Activation
php artisan migrate --force
php artisan queue:restart
php artisan horizon:terminate
```

### 3.4 สร้าง Custom Deployment Script

```php
// app/Console/Commands/PostDeploy.php
namespace App\Console\Commands;

use Illuminate\Console\Command;

class PostDeploy extends Command
{
    protected $signature = 'app:post-deploy';
    protected $description = 'Run post-deployment tasks';
    
    public function handle(): int
    {
        $this->info('Running post-deployment tasks...');
        
        // ล้าง Cache
        $this->call('cache:clear');
        $this->call('config:cache');
        $this->call('route:cache');
        $this->call('view:cache');
        
        // Run Migrations
        $this->call('migrate', ['--force' => true]);
        
        // Warm Caches
        $this->call('cache:warm');
        
        // Restart Workers
        $this->call('queue:restart');
        
        $this->info('Post-deployment tasks completed!');
        
        return 0;
    }
}
```

---

## 4. Docker Deployment

### 4.1 Dockerfile

```dockerfile
# Dockerfile
FROM php:8.2-fpm-alpine AS base

# Install system dependencies
RUN apk add --no-cache \
    git \
    curl \
    libpng-dev \
    libzip-dev \
    zip \
    unzip \
    nodejs \
    npm \
    nginx \
    supervisor

# Install PHP extensions
RUN docker-php-ext-install \
    pdo_mysql \
    mbstring \
    exif \
    pcntl \
    bcmath \
    gd \
    zip \
    opcache

# Install Redis extension
RUN pecl install redis && docker-php-ext-enable redis

# Install Swoole for Octane
RUN pecl install swoole && docker-php-ext-enable swoole

# Configure OPcache
COPY docker/php/opcache.ini /usr/local/etc/php/conf.d/opcache.ini

# Get Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

# Copy application
COPY . .

# ---- Development Stage ----
FROM base AS development
RUN composer install
RUN npm install && npm run build

# ---- Production Stage ----
FROM base AS production

# Install dependencies (no dev)
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist

COPY . .

RUN composer dump-autoload --optimize --classmap-authoritative

# Build assets
RUN npm ci && npm run build && rm -rf node_modules

# Set permissions
RUN chown -R www-data:www-data storage bootstrap/cache \
    && chmod -R 775 storage bootstrap/cache

# Copy configurations
COPY docker/nginx/nginx.conf /etc/nginx/nginx.conf
COPY docker/nginx/default.conf /etc/nginx/conf.d/default.conf
COPY docker/supervisor/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY docker/php/php.ini /usr/local/etc/php/php.ini

EXPOSE 80

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

### 4.2 Docker Compose Production

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      target: production
    image: myapp:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./storage:/var/www/html/storage
      - ./.env:/var/www/html/.env
      - ssl_certs:/etc/letsencrypt
    environment:
      - APP_ENV=production
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 512M
  
  db:
    image: mysql:8.0
    restart: unless-stopped
    volumes:
      - db_data:/var/lib/mysql
      - ./docker/mysql/my.cnf:/etc/mysql/conf.d/my.cnf
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 20s
      retries: 10
    networks:
      - app-network
  
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD} --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      timeout: 5s
      retries: 3
    networks:
      - app-network
  
  queue:
    build:
      context: .
      target: production
    restart: unless-stopped
    command: php artisan horizon
    volumes:
      - ./storage:/var/www/html/storage
      - ./.env:/var/www/html/.env
    depends_on:
      - redis
      - db
    networks:
      - app-network
  
  scheduler:
    build:
      context: .
      target: production
    restart: unless-stopped
    command: sh -c "while true; do php artisan schedule:run --verbose --no-interaction & sleep 60; done"
    volumes:
      - ./storage:/var/www/html/storage
      - ./.env:/var/www/html/.env
    depends_on:
      - db
      - redis
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  db_data:
  redis_data:
  ssl_certs:
```

### 4.3 Docker Supervisor Config

```ini
; docker/supervisor/supervisord.conf
[supervisord]
nodaemon=true
logfile=/var/log/supervisor/supervisord.log
pidfile=/var/run/supervisord.pid

[unix_http_server]
file=/var/run/supervisor.sock
chmod=0700

[supervisorctl]
serverurl=unix:///var/run/supervisor.sock

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[program:nginx]
command=nginx -g "daemon off;"
autostart=true
autorestart=true
stderr_logfile=/var/log/supervisor/nginx-error.log
stdout_logfile=/var/log/supervisor/nginx-access.log

[program:php-fpm]
command=php-fpm -F
autostart=true
autorestart=true
stderr_logfile=/var/log/supervisor/php-fpm-error.log
stdout_logfile=/var/log/supervisor/php-fpm-access.log
```

### 4.4 Makefile สำหรับ Docker Commands

```makefile
# Makefile
.PHONY: build up down logs shell migrate

build:
	docker-compose build

up:
	docker-compose up -d

down:
	docker-compose down

logs:
	docker-compose logs -f

shell:
	docker-compose exec app bash

migrate:
	docker-compose exec app php artisan migrate --force

deploy:
	docker-compose pull
	docker-compose up -d --build
	docker-compose exec app php artisan migrate --force
	docker-compose exec app php artisan config:cache
	docker-compose exec app php artisan route:cache
	docker-compose exec app php artisan view:cache
	docker-compose exec app php artisan queue:restart
```

---

## 5. GitHub Actions CI/CD

### 5.1 Basic CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3
      
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: --health-cmd="redis-cli ping" --health-interval=10s --health-timeout=5s
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, sqlite, pdo_sqlite, bcmath, soap, intl, gd, exif, iconv, imagick, redis
          coverage: xdebug
      
      - name: Cache Composer dependencies
        uses: actions/cache@v3
        with:
          path: ~/.composer/cache
          key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
          restore-keys: ${{ runner.os }}-composer-
      
      - name: Install dependencies
        run: composer install --no-progress --prefer-dist --optimize-autoloader
      
      - name: Copy .env
        run: cp .env.testing .env
      
      - name: Generate key
        run: php artisan key:generate
      
      - name: Cache Node modules
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      
      - name: Install npm dependencies
        run: npm ci
      
      - name: Build assets
        run: npm run build
      
      - name: Create test database
        run: php artisan migrate --force
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: password
      
      - name: Run Tests with Coverage
        run: php artisan test --coverage --min=70
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: password
          REDIS_HOST: 127.0.0.1
      
      - name: Run Static Analysis (PHPStan)
        run: ./vendor/bin/phpstan analyse --memory-limit=2G
      
      - name: Run Code Style Check
        run: ./vendor/bin/pint --test
```

### 5.2 CD Pipeline (Deploy to Production)

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  test:
    uses: ./.github/workflows/ci.yml
  
  deploy:
    name: Deploy
    needs: test
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.8.0
        with:
          ssh-private-key: ${{ secrets.DEPLOY_SSH_KEY }}
      
      - name: Add server to known hosts
        run: |
          ssh-keyscan -H ${{ secrets.DEPLOY_HOST }} >> ~/.ssh/known_hosts
      
      - name: Deploy to server
        run: |
          ssh ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} '
            cd /var/www/html
            
            echo "Pulling latest changes..."
            git pull origin main
            
            echo "Installing dependencies..."
            composer install --no-dev --optimize-autoloader --no-interaction
            
            echo "Building assets..."
            npm ci && npm run build
            
            echo "Caching..."
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache
            
            echo "Running migrations..."
            php artisan migrate --force
            
            echo "Restarting services..."
            php artisan queue:restart
            sudo service php8.2-fpm reload
            
            echo "Deployment completed at $(date)"
          '
      
      - name: Notify Slack on Success
        if: success()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": "Deployment to Production succeeded! :rocket:",
              "attachments": [
                {
                  "color": "good",
                  "fields": [
                    {"title": "Repository", "value": "${{ github.repository }}", "short": true},
                    {"title": "Branch", "value": "${{ github.ref_name }}", "short": true},
                    {"title": "Deployed by", "value": "${{ github.actor }}", "short": true}
                  ]
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      
      - name: Notify Slack on Failure
        if: failure()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": "Deployment to Production FAILED! :x:",
              "attachments": [
                {
                  "color": "danger",
                  "text": "Check the GitHub Actions workflow for details."
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 5.3 Docker Build และ Push to Registry

```yaml
# .github/workflows/docker.yml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*.*.*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          target: production
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Deploy to Production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          script: |
            cd /var/www
            docker-compose pull
            docker-compose up -d --remove-orphans
            docker-compose exec -T app php artisan migrate --force
            docker-compose exec -T app php artisan config:cache
            docker system prune -f
```

---

## 6. Production Checklist

### 6.1 Security Checklist

```bash
# ตรวจสอบ Security Headers
curl -I https://example.com | grep -E "X-Frame-Options|X-Content-Type|Strict-Transport|Content-Security"

# ตรวจสอบ SSL
curl -s https://ssl-checker.io/api/v1/check/example.com | jq

# ตรวจสอบ Rate Limiting
for i in {1..100}; do curl -s -o /dev/null -w "%{http_code}" https://example.com/api/login; done
```

```php
// Security Configuration
// config/app.php
'debug' => env('APP_DEBUG', false), // false ใน production

// .env
APP_DEBUG=false
APP_ENV=production

// ปิด Directory Listing
// .htaccess หรือ nginx config:
Options -Indexes
```

### 6.2 Performance Checklist

```bash
#!/bin/bash
# production-check.sh

echo "=== Laravel Production Check ==="

# Check APP_DEBUG
if grep -q "APP_DEBUG=true" .env; then
    echo "FAIL: APP_DEBUG is true in production!"
else
    echo "OK: APP_DEBUG is false"
fi

# Check configs are cached
if php artisan config:show --raw > /dev/null 2>&1; then
    echo "OK: Config is cached"
fi

# Check routes are cached  
if [ -f "bootstrap/cache/routes-v7.php" ]; then
    echo "OK: Routes are cached"
else
    echo "WARNING: Routes are not cached"
fi

# Check storage symlink
if [ -L "public/storage" ]; then
    echo "OK: Storage symlink exists"
else
    echo "FAIL: Storage symlink missing"
fi

# Check .env permissions
PERMS=$(stat -c %a .env)
if [ "$PERMS" = "600" ] || [ "$PERMS" = "640" ]; then
    echo "OK: .env permissions are secure"
else
    echo "WARNING: .env permissions should be 600 or 640 (current: $PERMS)"
fi

echo "=== Check Complete ==="
```

---

## 7. Workshop: Deploy Laravel App ไป Production

### Step 1: เตรียม Server (Ubuntu 22.04)

```bash
#!/bin/bash
# server-setup.sh

# Update system
apt-get update && apt-get upgrade -y

# Install Nginx
apt-get install -y nginx

# Install PHP 8.2
add-apt-repository ppa:ondrej/php -y
apt-get update
apt-get install -y php8.2-fpm php8.2-mysql php8.2-mbstring \
    php8.2-xml php8.2-bcmath php8.2-curl php8.2-gd \
    php8.2-intl php8.2-zip php8.2-redis php8.2-opcache

# Install MySQL
apt-get install -y mysql-server

# Install Redis
apt-get install -y redis-server

# Install Composer
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer

# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs

# Install Certbot (SSL)
apt-get install -y certbot python3-certbot-nginx

# Setup Firewall
ufw allow OpenSSH
ufw allow Nginx Full
ufw enable

# Create deploy user
useradd -m -s /bin/bash deploy
mkdir -p /var/www/html
chown -R deploy:www-data /var/www/html

echo "Server setup complete!"
```

### Step 2: สร้าง Deploy Script

```bash
#!/bin/bash
# scripts/deploy.sh

set -e  # Exit on error

DEPLOY_DIR="/var/www/html"
CURRENT_RELEASE=$(date +%Y%m%d%H%M%S)
RELEASES_DIR="$DEPLOY_DIR/releases"
CURRENT_DIR="$DEPLOY_DIR/current"
SHARED_DIR="$DEPLOY_DIR/shared"

echo "Starting deployment: $CURRENT_RELEASE"

# Create directories
mkdir -p "$RELEASES_DIR/$CURRENT_RELEASE"
mkdir -p "$SHARED_DIR/storage"
mkdir -p "$SHARED_DIR/bootstrap/cache"

# Clone repository
git clone --depth 1 --branch main \
    https://github.com/yourorg/yourapp.git \
    "$RELEASES_DIR/$CURRENT_RELEASE"

cd "$RELEASES_DIR/$CURRENT_RELEASE"

# Link shared directories
rm -rf storage
ln -nfs "$SHARED_DIR/storage" storage

rm -rf bootstrap/cache
ln -nfs "$SHARED_DIR/bootstrap/cache" bootstrap/cache

# Link .env
ln -nfs "$SHARED_DIR/.env" .env

# Install dependencies
composer install --no-dev --optimize-autoloader --no-interaction

# Build assets
npm ci && npm run build

# Optimize
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Run migrations
php artisan migrate --force

# Activate release (atomic symlink swap)
ln -nfs "$RELEASES_DIR/$CURRENT_RELEASE" "$CURRENT_DIR"

# Reload services
php artisan queue:restart
sudo service php8.2-fpm reload

# Purge old releases (keep last 5)
ls -t "$RELEASES_DIR" | tail -n +6 | xargs -I {} rm -rf "$RELEASES_DIR/{}"

echo "Deployment completed: $CURRENT_RELEASE"
```

### Step 3: Rollback Script

```bash
#!/bin/bash
# scripts/rollback.sh

DEPLOY_DIR="/var/www/html"
RELEASES_DIR="$DEPLOY_DIR/releases"
CURRENT_DIR="$DEPLOY_DIR/current"

# Get previous release
PREVIOUS=$(ls -t "$RELEASES_DIR" | sed -n '2p')

if [ -z "$PREVIOUS" ]; then
    echo "No previous release to rollback to!"
    exit 1
fi

echo "Rolling back to: $PREVIOUS"

# Activate previous release
ln -nfs "$RELEASES_DIR/$PREVIOUS" "$CURRENT_DIR"

# Reload services
php artisan queue:restart
sudo service php8.2-fpm reload

echo "Rollback completed: $PREVIOUS"
```

---

## Quiz

**ข้อ 1:** Zero-downtime Deployment คืออะไร?

a) Deploy ที่ไม่ใช้ Downtime เลย  
b) Deploy โดยการสลับ Symlink แบบ Atomic ทำให้ Old version ทำงานได้ระหว่าง Deploy  
c) Deploy ผ่าน Docker เท่านั้น  
d) Deploy ที่เร็วกว่า 1 วินาที

**เฉลย:** b) Zero-downtime Deploy ทำได้โดยเตรียม Release ใหม่ใน Directory แยก แล้วสลับ Symlink `current` แบบ Atomic ทำให้ไม่มีช่วงที่ Application หยุดทำงาน

---

**ข้อ 2:** เหตุใด `APP_DEBUG=false` จึงสำคัญใน Production?

a) ทำให้ Application เร็วขึ้น  
b) ป้องกันการเปิดเผย Stack Traces และ Error Details ที่อาจเป็น Security Vulnerability  
c) ลด Memory Usage  
d) ทำให้ Logs ทำงานได้

**เฉลย:** b) Stack Traces มีข้อมูลสำคัญเช่น File paths, Code, Environment variables ที่ Attacker สามารถนำไปใช้โจมตีได้

---

**ข้อ 3:** ใน GitHub Actions `environment: production` ใช้ทำอะไร?

a) กำหนด PHP version  
b) สร้าง Protected Environment ที่ต้องการ Approval ก่อน Deploy  
c) กำหนด Node.js version  
d) เชื่อมต่อ Database

**เฉลย:** b) GitHub Environments ช่วยสร้าง Protection Rules เช่น Required Reviewers, Wait Timer และจัดการ Secrets แยกต่างหากสำหรับแต่ละ Environment

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- **Server Setup:** Nginx, PHP-FPM, MySQL, Redis
- **Forge:** ตั้งค่า Server และ Deploy Script
- **Envoyer:** Zero-Downtime Deployment
- **Docker:** Containerization และ Docker Compose
- **GitHub Actions:** CI/CD Pipeline อัตโนมัติ
- **Workshop:** Deploy Script และ Rollback

---

## ไปต่อ

➡️ [Part 049: Laravel Advanced Patterns — Clean Code ระดับมืออาชีพ](./part-049-laravel-advanced-patterns.md)
