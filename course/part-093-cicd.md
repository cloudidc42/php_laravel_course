# Part 93: CI/CD สำหรับ PHP/Laravel

## บทนำ

CI/CD (Continuous Integration / Continuous Delivery) คือแนวทางการพัฒนาซอฟต์แวร์ที่ Automate กระบวนการ Build, Test, และ Deploy โดย:
- **CI**: ทดสอบ Code ทุกครั้งที่ Push
- **CD**: Deploy ไป Production โดยอัตโนมัติเมื่อผ่าน Tests

---

## GitHub Actions สำหรับ PHP/Laravel

### Basic Workflow

```yaml
# .github/workflows/ci.yml
name: Laravel CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  PHP_VERSION: '8.3'
  NODE_VERSION: '20'

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: secret
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3
      
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd="redis-cli ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          extensions: mbstring, bcmath, ctype, fileinfo, json, openssl, pdo, pdo_mysql, tokenizer, xml, redis
          coverage: xdebug
          tools: composer:v2
      
      - name: Cache Composer dependencies
        uses: actions/cache@v3
        with:
          path: vendor
          key: composer-${{ hashFiles('**/composer.lock') }}
          restore-keys: composer-
      
      - name: Install PHP dependencies
        run: composer install --no-interaction --no-progress --prefer-dist
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install Node dependencies
        run: npm ci
      
      - name: Build assets
        run: npm run build
      
      - name: Copy environment file
        run: cp .env.testing .env
      
      - name: Generate application key
        run: php artisan key:generate
      
      - name: Run database migrations
        run: php artisan migrate --force
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: secret
      
      - name: Run PHP tests
        run: php artisan test --parallel --coverage-clover=coverage.xml
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: secret
          REDIS_HOST: 127.0.0.1
          REDIS_PORT: 6379
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: coverage.xml
          fail_ci_if_error: true

  code-quality:
    name: Code Quality
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          tools: composer:v2
      
      - name: Install dependencies
        run: composer install --no-interaction
      
      - name: Run PHP CS Fixer
        run: vendor/bin/php-cs-fixer fix --dry-run --diff
      
      - name: Run PHPStan
        run: vendor/bin/phpstan analyse --error-format=github
      
      - name: Run Pint (Laravel)
        run: vendor/bin/pint --test
      
      - name: Check for security vulnerabilities
        run: composer audit

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [test, code-quality]
    if: github.ref == 'refs/heads/develop'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /var/www/staging
            git pull origin develop
            composer install --no-dev --optimize-autoloader
            php artisan migrate --force
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache
            sudo supervisorctl restart laravel-worker:*
            echo "Deployment completed!"
```

---

### Advanced: Matrix Testing

```yaml
# .github/workflows/matrix-test.yml
name: Matrix Tests

on: [push, pull_request]

jobs:
  test:
    name: PHP ${{ matrix.php }} - Laravel ${{ matrix.laravel }}
    runs-on: ubuntu-latest
    
    strategy:
      fail-fast: false
      matrix:
        php: ['8.1', '8.2', '8.3']
        laravel: ['10.*', '11.*']
        exclude:
          - php: '8.1'
            laravel: '11.*'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP ${{ matrix.php }}
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
      
      - name: Update Laravel version
        run: |
          composer require "laravel/framework:${{ matrix.laravel }}" --no-update
          composer update --no-interaction
      
      - name: Run tests
        run: php artisan test
```

---

### Deployment Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  release:
    types: [published]

jobs:
  deploy:
    name: Zero-Downtime Deployment
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
      
      - name: Install dependencies
        run: composer install --no-dev --optimize-autoloader
      
      - name: Build assets
        run: |
          npm ci
          npm run build
      
      - name: Deploy with zero downtime
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            set -e
            
            # Variables
            APP_DIR="/var/www/myapp"
            RELEASES_DIR="/var/www/releases"
            RELEASE_DIR="${RELEASES_DIR}/$(date +%Y%m%d_%H%M%S)"
            SHARED_DIR="/var/www/shared"
            
            # Create new release directory
            mkdir -p $RELEASE_DIR
            
            # Clone repository
            git clone --depth=1 --branch=${{ github.ref_name }} \
              https://github.com/${{ github.repository }}.git $RELEASE_DIR
            
            # Install dependencies
            cd $RELEASE_DIR
            composer install --no-dev --optimize-autoloader
            
            # Link shared files
            ln -sfn $SHARED_DIR/.env $RELEASE_DIR/.env
            ln -sfn $SHARED_DIR/storage $RELEASE_DIR/storage
            
            # Run migrations
            php artisan migrate --force
            
            # Optimize
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache
            php artisan icons:cache
            
            # Atomic switch (Zero Downtime!)
            ln -sfn $RELEASE_DIR $APP_DIR
            
            # Reload PHP-FPM (not restart)
            sudo kill -USR2 $(cat /var/run/php/php8.3-fpm.pid)
            
            # Restart Queue workers
            sudo supervisorctl restart laravel-worker:*
            
            # Clean up old releases (keep last 5)
            ls -dt $RELEASES_DIR/* | tail -n +6 | xargs rm -rf
            
            echo "✅ Deployment successful: $RELEASE_DIR"
      
      - name: Verify deployment
        run: |
          sleep 10
          response=$(curl -s -o /dev/null -w "%{http_code}" https://myapp.com/health)
          if [ "$response" != "200" ]; then
            echo "❌ Health check failed!"
            exit 1
          fi
          echo "✅ Health check passed!"
      
      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": "${{ job.status == 'success' && '✅' || '❌' }} Deployment ${{ job.status }} - ${{ github.repository }} @ ${{ github.ref_name }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## GitLab CI

```yaml
# .gitlab-ci.yml
image: php:8.3-fpm

variables:
  MYSQL_ROOT_PASSWORD: secret
  MYSQL_DATABASE: testing
  REDIS_HOST: redis

stages:
  - install
  - test
  - quality
  - build
  - deploy

# Cache สำหรับ Dependencies
.cache_config: &cache_config
  cache:
    key:
      files:
        - composer.lock
        - package-lock.json
    paths:
      - vendor/
      - node_modules/

# Install Stage
install:dependencies:
  stage: install
  <<: *cache_config
  script:
    - composer install --no-interaction --no-progress
    - npm ci
  artifacts:
    paths:
      - vendor/
      - node_modules/

# Test Stage
test:unit:
  stage: test
  <<: *cache_config
  services:
    - name: mysql:8.0
      alias: mysql
    - name: redis:7
      alias: redis
  variables:
    DB_HOST: mysql
    DB_DATABASE: testing
    DB_USERNAME: root
    DB_PASSWORD: secret
  script:
    - cp .env.testing .env
    - php artisan key:generate
    - php artisan migrate --force
    - php artisan test --coverage-text
  coverage: '/Lines:\s+\d+\.\d+%/'
  artifacts:
    reports:
      junit: storage/logs/phpunit-report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

test:integration:
  stage: test
  services:
    - mysql:8.0
    - redis:7
  script:
    - php artisan test --testsuite=Feature

# Quality Stage
quality:phpstan:
  stage: quality
  script:
    - vendor/bin/phpstan analyse --error-format=gitlab > phpstan-report.json
  artifacts:
    reports:
      codequality: phpstan-report.json

quality:psalm:
  stage: quality
  script:
    - vendor/bin/psalm --output-format=github

quality:security:
  stage: quality
  script:
    - composer audit
  allow_failure: false

# Build Stage
build:docker:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA -t $CI_REGISTRY_IMAGE:latest .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main

# Deploy Stage
deploy:staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.myapp.com
  script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$STAGING_SSH_KEY" | tr -d '\r' | ssh-add -
    - ssh -o StrictHostKeyChecking=no $STAGING_USER@$STAGING_HOST "
        cd /var/www/staging &&
        git pull &&
        composer install --no-dev &&
        php artisan migrate --force &&
        php artisan optimize"
  only:
    - develop

deploy:production:
  stage: deploy
  environment:
    name: production
    url: https://myapp.com
  when: manual
  script:
    - echo "Deploying to production..."
  only:
    - main
```

---

## Docker Deployment

### Multi-stage Dockerfile

```dockerfile
# Dockerfile
# Build stage
FROM node:20-alpine AS node-builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY resources/ ./resources/
COPY vite.config.js ./
COPY tailwind.config.js ./

RUN npm run build

# PHP Build stage
FROM composer:2 AS composer-builder

WORKDIR /app
COPY composer*.json ./
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist

COPY . .
RUN composer dump-autoload --optimize

# Production stage
FROM php:8.3-fpm-alpine AS production

# Install system dependencies
RUN apk add --no-cache \
    nginx \
    supervisor \
    curl \
    git \
    libpng-dev \
    libjpeg-turbo-dev \
    freetype-dev \
    libwebp-dev

# Install PHP extensions
RUN docker-php-ext-configure gd \
    --with-freetype \
    --with-jpeg \
    --with-webp

RUN docker-php-ext-install \
    pdo_mysql \
    opcache \
    pcntl \
    bcmath \
    gd \
    exif

# Install Redis extension
RUN pecl install redis && docker-php-ext-enable redis

# OPcache configuration
RUN cat > /usr/local/etc/php/conf.d/opcache.ini << 'EOF'
opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=20000
opcache.revalidate_freq=0
opcache.validate_timestamps=0
opcache.save_comments=1
opcache.jit=tracing
opcache.jit_buffer_size=100M
EOF

# Copy application
WORKDIR /var/www/html

COPY --from=composer-builder /app/vendor ./vendor
COPY --from=composer-builder /app .
COPY --from=node-builder /app/public/build ./public/build

# Permissions
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/storage \
    && chmod -R 755 /var/www/html/bootstrap/cache

# Copy configs
COPY docker/nginx.conf /etc/nginx/http.d/default.conf
COPY docker/php-fpm.conf /usr/local/etc/php-fpm.d/www.conf
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/health || exit 1

CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

```ini
; docker/supervisord.conf
[supervisord]
nodaemon=true
user=root
logfile=/var/log/supervisor/supervisord.log

[program:nginx]
command=nginx -g 'daemon off;'
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0

[program:php-fpm]
command=php-fpm
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0

[program:laravel-worker]
command=php /var/www/html/artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
numprocs=4
process_name=%(program_name)s_%(process_num)02d
stdout_logfile=/var/log/supervisor/worker.log
```

---

## Zero-Downtime Deployment

```bash
#!/bin/bash
# deploy.sh - Zero-downtime deployment script

set -e

APP_DIR="/var/www/myapp"
RELEASES_DIR="/var/www/releases"
SHARED_DIR="/var/www/shared"
KEEP_RELEASES=5
REPO="git@github.com:mycompany/myapp.git"
BRANCH="${1:-main}"

echo "🚀 Starting deployment of branch: $BRANCH"

# Create release directory
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RELEASE_DIR="$RELEASES_DIR/$TIMESTAMP"
mkdir -p "$RELEASE_DIR"

# Clone repository
echo "📦 Cloning repository..."
git clone --depth=1 --branch="$BRANCH" "$REPO" "$RELEASE_DIR"

# Install dependencies
echo "📚 Installing dependencies..."
cd "$RELEASE_DIR"
composer install --no-dev --optimize-autoloader --quiet

# Link shared resources
echo "🔗 Linking shared resources..."
rm -rf "$RELEASE_DIR/storage"
ln -sfn "$SHARED_DIR/storage" "$RELEASE_DIR/storage"
ln -sfn "$SHARED_DIR/.env" "$RELEASE_DIR/.env"

# Run migrations
echo "🗄️ Running migrations..."
php artisan migrate --force

# Optimize
echo "⚡ Optimizing..."
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# Atomic switch
echo "🔄 Switching to new release..."
ln -sfn "$RELEASE_DIR" "$APP_DIR"

# Gracefully reload PHP-FPM
echo "🔧 Reloading PHP-FPM..."
sudo kill -USR2 $(cat /var/run/php/php8.3-fpm.pid)

# Restart queue workers gracefully
echo "👷 Restarting queue workers..."
php artisan queue:restart

# Clean old releases
echo "🧹 Cleaning old releases..."
ls -dt "$RELEASES_DIR"/* | tail -n "+$((KEEP_RELEASES + 1))" | xargs rm -rf

echo "✅ Deployment completed: $RELEASE_DIR"
echo "📊 Release: $TIMESTAMP"
```

---

## Workshop: CI/CD Pipeline ครบวงจร

```yaml
# .github/workflows/complete-pipeline.yml
name: Complete CI/CD Pipeline

on:
  push:
    branches: ['**']
  pull_request:
    branches: [main, develop]
  release:
    types: [published]

jobs:
  # Stage 1: Code Quality
  lint:
    name: 🔍 Lint & Format Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          tools: cs2pr
      - run: composer install --no-interaction
      - run: vendor/bin/pint --test
      - run: vendor/bin/phpstan analyse --error-format=checkstyle | cs2pr

  # Stage 2: Security Check
  security:
    name: 🔐 Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
      - run: composer audit --format=json > security-report.json || true
      - name: Check for critical vulnerabilities
        run: |
          if grep -q '"severity":"critical"' security-report.json; then
            echo "❌ Critical vulnerabilities found!"
            cat security-report.json
            exit 1
          fi

  # Stage 3: Tests
  test:
    name: 🧪 Test Suite
    runs-on: ubuntu-latest
    needs: [lint]
    
    strategy:
      matrix:
        php: ['8.2', '8.3']
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: testing
          MYSQL_DATABASE: laravel_test
        options: --health-cmd="mysqladmin ping"
      redis:
        image: redis:7
        options: --health-cmd="redis-cli ping"
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP ${{ matrix.php }}
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
          extensions: redis, xdebug
          coverage: xdebug
      
      - name: Install Dependencies
        run: composer install --no-interaction
      
      - name: Prepare Environment
        run: |
          cp .env.testing .env
          php artisan key:generate
          php artisan migrate --force
        env:
          DB_HOST: 127.0.0.1
          DB_DATABASE: laravel_test
          DB_USERNAME: root
          DB_PASSWORD: testing
      
      - name: Run Tests with Coverage
        run: php artisan test --parallel --coverage-clover=coverage.xml
        env:
          DB_HOST: 127.0.0.1
          DB_DATABASE: laravel_test
          DB_USERNAME: root
          DB_PASSWORD: testing
          REDIS_HOST: 127.0.0.1
      
      - name: Assert Coverage >= 80%
        run: |
          coverage=$(php -r "
            \$xml = simplexml_load_file('coverage.xml');
            \$metrics = \$xml->xpath('//metrics');
            \$statements = 0;
            \$covered = 0;
            foreach (\$metrics as \$m) {
              \$statements += (int)\$m['statements'];
              \$covered += (int)\$m['coveredstatements'];
            }
            echo round(\$covered / \$statements * 100, 2);
          ")
          echo "Coverage: ${coverage}%"
          if (( $(echo "${coverage} < 80" | bc -l) )); then
            echo "❌ Coverage below 80%!"
            exit 1
          fi

  # Stage 4: Build Docker Image
  build:
    name: 🐳 Build Docker Image
    runs-on: ubuntu-latest
    needs: [test, security]
    if: github.event_name != 'pull_request'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # Stage 5: Deploy
  deploy-production:
    name: 🚀 Deploy to Production
    runs-on: ubuntu-latest
    needs: [build]
    if: github.event_name == 'release'
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - name: Deploy
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            ./deploy.sh ${{ github.ref_name }}
      
      - name: Smoke Test
        run: |
          sleep 15
          for endpoint in / /api/health; do
            status=$(curl -s -o /dev/null -w "%{http_code}" https://myapp.com$endpoint)
            [ "$status" = "200" ] || { echo "❌ $endpoint returned $status"; exit 1; }
          done
          echo "✅ All smoke tests passed!"
```

---

## สรุป

| Stage | เครื่องมือ | วัตถุประสงค์ |
|-------|----------|------------|
| Lint | PHP-CS-Fixer, Pint | Code Style |
| Static Analysis | PHPStan, Psalm | Type Safety, Bugs |
| Security | composer audit, Snyk | Vulnerabilities |
| Tests | PHPUnit, Pest | Correctness |
| Coverage | Xdebug, PCOV | Test completeness |
| Build | Docker | Packaging |
| Deploy | SSH, Kubernetes | Release |

---

*CI/CD ช่วยให้ Deploy บ่อยขึ้น เร็วขึ้น และปลอดภัยขึ้น - "If it hurts, do it more often"*
