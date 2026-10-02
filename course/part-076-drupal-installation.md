# Part 076: Drupal 10 Installation & Project Structure

**ระดับ:** Intermediate  
**เวลาเรียน:** 4-5 ชั่วโมง  
**Prerequisites:** PHP 8.1+, Composer, MySQL/PostgreSQL, Web Server

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. ติดตั้ง Drupal 10 ด้วย Composer ได้อย่างถูกต้อง
2. ใช้ Drush CLI เพื่อจัดการ Drupal ผ่าน Command Line
3. เข้าใจโครงสร้างโปรเจกต์ Drupal (web/, modules/, themes/, config/)
4. ใช้ Configuration Management เพื่อ sync config ระหว่าง environments
5. สร้าง Drupal project แรกพร้อม local dev environment

---

## 1. ทำความรู้จัก Drupal 10

Drupal เป็น CMS (Content Management System) ระดับ Enterprise ที่ทรงพลัง ใช้งานโดยองค์กรขนาดใหญ่ทั่วโลก เช่น NASA, The White House, The Economist

### 1.1 ทำไมต้องเลือก Drupal?

- **ความยืดหยุ่นสูง:** สามารถสร้างระบบซับซ้อนได้โดยไม่ต้องเขียนโค้ดมาก
- **Security:** มีทีม Security ดูแลโดยเฉพาะ
- **Multilingual:** รองรับหลายภาษาในตัว
- **API-first:** สามารถใช้เป็น Headless CMS ได้
- **Scalable:** ขยายขนาดได้ตามความต้องการ

### 1.2 ความต้องการของระบบ (System Requirements)

```
PHP: 8.1 หรือใหม่กว่า
Database: MySQL 5.7.8+ / MariaDB 10.3.7+ / PostgreSQL 14+
Web Server: Apache 2.4.7+ / Nginx
Composer: 2.x
Drush: 12.x (สำหรับ Drupal 10)
```

---

## 2. ติดตั้ง Drupal 10 ด้วย Composer

### 2.1 สร้างโปรเจกต์ใหม่

```bash
# สร้าง Drupal project ด้วย Composer
composer create-project drupal/recommended-project my-drupal-site

# เข้าไปในโฟลเดอร์โปรเจกต์
cd my-drupal-site

# ดูโครงสร้างไฟล์
ls -la
```

### 2.2 โครงสร้างโปรเจกต์หลัก

```
my-drupal-site/
├── composer.json          # Dependencies หลัก
├── composer.lock          # Lock file
├── vendor/                # PHP libraries
└── web/                   # Document root
    ├── core/              # Drupal core
    ├── modules/           # Contributed & custom modules
    │   ├── contrib/       # Modules จาก drupal.org
    │   └── custom/        # Custom modules ที่เราเขียนเอง
    ├── themes/            # Themes
    │   ├── contrib/       # Contributed themes
    │   └── custom/        # Custom themes
    ├── profiles/          # Installation profiles
    ├── sites/             # Site-specific files
    │   └── default/
    │       ├── files/     # Uploaded files
    │       └── settings.php
    ├── index.php
    └── .htaccess
```

### 2.3 ติดตั้ง Drush

```bash
# ติดตั้ง Drush ผ่าน Composer
composer require drush/drush

# ตรวจสอบเวอร์ชัน
vendor/bin/drush --version

# หรือใช้ global alias
alias drush='vendor/bin/drush'
```

### 2.4 ตั้งค่า Database

```bash
# สร้าง database
mysql -u root -p
```

```sql
CREATE DATABASE drupal10_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'drupal_user'@'localhost' IDENTIFIED BY 'secure_password';
GRANT ALL PRIVILEGES ON drupal10_db.* TO 'drupal_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 2.5 ติดตั้งผ่าน Web Installer

เปิด browser ไปที่ `http://localhost/my-drupal-site/web`

ขั้นตอน:
1. เลือกภาษา
2. เลือก Installation profile (Standard / Minimal / Demo)
3. กรอก Database credentials
4. ตั้งค่า Site name, admin email, password

### 2.6 ติดตั้งผ่าน Drush (แนะนำสำหรับ Development)

```bash
# ติดตั้ง Drupal ด้วย Drush
vendor/bin/drush site:install standard \
  --db-url=mysql://drupal_user:secure_password@localhost/drupal10_db \
  --site-name="My Drupal Site" \
  --account-name=admin \
  --account-pass=admin123 \
  --yes

# สร้าง settings.php local override
cp web/sites/default/default.settings.php web/sites/default/settings.php
```

### 2.7 settings.php พื้นฐาน

```php
<?php
// web/sites/default/settings.php

// Database configuration
$databases['default']['default'] = [
  'database' => 'drupal10_db',
  'username' => 'drupal_user',
  'password' => 'secure_password',
  'host' => 'localhost',
  'port' => '3306',
  'driver' => 'mysql',
  'prefix' => '',
  'collation' => 'utf8mb4_general_ci',
];

// Hash salt (auto-generated during install)
$settings['hash_salt'] = 'your-unique-hash-salt-here';

// Trusted host patterns
$settings['trusted_host_patterns'] = [
  '^localhost$',
  '^127\.0\.0\.1$',
  '^mysite\.local$',
];

// Configuration sync directory
$settings['config_sync_directory'] = '../config/sync';

// File paths
$settings['file_public_path'] = 'sites/default/files';
$settings['file_private_path'] = '../private';
$settings['file_temp_path'] = '/tmp';

// Development settings
if (file_exists($app_root . '/' . $site_path . '/settings.local.php')) {
  include $app_root . '/' . $site_path . '/settings.local.php';
}
```

### 2.8 settings.local.php สำหรับ Development

```php
<?php
// web/sites/default/settings.local.php

// Disable caching for development
$settings['cache']['bins']['render'] = 'cache.backend.null';
$settings['cache']['bins']['page'] = 'cache.backend.null';
$settings['cache']['bins']['dynamic_page_cache'] = 'cache.backend.null';

// Enable verbose error display
$config['system.logging']['error_level'] = 'verbose';

// Disable aggregation
$config['system.performance']['css']['preprocess'] = FALSE;
$config['system.performance']['js']['preprocess'] = FALSE;

// Enable Twig debug
$settings['twig_debug'] = TRUE;
$settings['twig_auto_reload'] = TRUE;
$settings['twig_cache'] = FALSE;
```

---

## 3. Drush CLI

Drush (Drupal Shell) เป็นเครื่องมือ Command Line ที่ขาดไม่ได้สำหรับ Drupal developers

### 3.1 Cache Management

```bash
# Clear all caches (สำคัญมาก! ใช้บ่อยที่สุด)
drush cache:rebuild
# หรือย่อ
drush cr

# Clear specific cache bin
drush cache:clear render
drush cache:clear menu

# ดู cache bins ทั้งหมด
drush cache:clear
```

### 3.2 Database Updates

```bash
# Run database updates (หลังอัปเดต modules/core)
drush updatedb
# หรือย่อ
drush updb

# ดู pending updates
drush updatedb --simulate

# Run specific update hook
drush php-eval "mymodule_update_10001();"
```

### 3.3 Configuration Management

```bash
# Export configuration จาก database ไป config files
drush config:export
# หรือย่อ
drush cex

# Import configuration จาก config files เข้า database
drush config:import
# หรือย่อ
drush cim

# ดู config ที่มีความแตกต่าง
drush config:status
# หรือย่อ
drush cst

# Get config value
drush config:get system.site name

# Set config value
drush config:set system.site name "New Site Name"
```

### 3.4 User Management

```bash
# สร้าง one-time login link (ไม่ต้องใช้ password!)
drush user:login
# หรือย่อ
drush uli

# Login as specific user
drush uli --uid=1
drush uli admin

# สร้าง user ใหม่
drush user:create john --mail=john@example.com --password=pass123

# เปลี่ยน password
drush user:password admin newpassword123

# Block/Unblock user
drush user:block suspicious_user
drush user:unblock good_user

# Add role
drush user:role:add editor john
```

### 3.5 Module Management

```bash
# Enable module
drush pm:enable views
drush en views

# Disable module
drush pm:uninstall mymodule
drush pmu mymodule

# ดู installed modules
drush pm:list --status=enabled

# ดู available updates
drush pm:security
```

### 3.6 Maintenance Mode

```bash
# เปิด Maintenance Mode
drush state:set system.maintenance_mode 1 --input-format=integer
drush cr

# ปิด Maintenance Mode
drush state:set system.maintenance_mode 0 --input-format=integer
drush cr
```

### 3.7 Entity & SQL Operations

```bash
# Run SQL query
drush sql:query "SELECT nid, title FROM node_field_data WHERE type = 'article' LIMIT 5"

# SQL connect (เปิด MySQL CLI)
drush sql:connect

# Dump database
drush sql:dump > backup.sql

# Import database
drush sql:query < backup.sql

# Entity queries
drush php-eval "
  \$nids = \Drupal::entityQuery('node')
    ->condition('type', 'article')
    ->condition('status', 1)
    ->range(0, 5)
    ->execute();
  print_r(\$nids);
"
```

### 3.8 Generate Code (Drush Generate)

```bash
# สร้าง module skeleton
drush generate module

# สร้าง controller
drush generate controller

# สร้าง form
drush generate form

# สร้าง service
drush generate service

# สร้าง plugin
drush generate plugin:block
```

---

## 4. โครงสร้างโปรเจกต์เชิงลึก

### 4.1 web/core/

```
web/core/
├── lib/Drupal/Core/       # Core PHP classes
├── modules/               # Core modules (node, user, views, etc.)
├── themes/                # Core themes (claro, olivero, etc.)
├── profiles/              # Core profiles
└── assets/                # CSS, JS ของ core
```

### 4.2 web/modules/

```
web/modules/
├── contrib/               # Modules ที่ติดตั้งผ่าน Composer
│   ├── views/
│   ├── pathauto/
│   └── token/
└── custom/                # Custom modules ที่เราเขียนเอง
    └── my_module/
        ├── my_module.info.yml
        ├── my_module.module
        ├── my_module.routing.yml
        └── src/
            ├── Controller/
            ├── Form/
            └── Plugin/
```

### 4.3 web/themes/

```
web/themes/
├── contrib/
│   └── bootstrap5/
└── custom/
    └── mytheme/
        ├── mytheme.info.yml
        ├── mytheme.libraries.yml
        ├── mytheme.theme
        ├── templates/
        └── css/
```

### 4.4 config/

```
config/
└── sync/                  # Configuration YAML files
    ├── system.site.yml
    ├── node.type.article.yml
    ├── views.view.frontpage.yml
    └── field.field.node.article.body.yml
```

---

## 5. Configuration Management เชิงลึก

Configuration Management เป็นหัวใจสำคัญของ Drupal 8+ ช่วยให้ย้าย config ระหว่าง environments ได้อย่างปลอดภัย

### 5.1 วิธีการทำงาน

```
Development → Export → Version Control → Import → Staging → Import → Production
```

**Active Config** = ข้อมูล config ที่อยู่ใน database (ใช้งานจริง)  
**Sync Config** = ไฟล์ YAML ใน config/sync/ (source of truth)

### 5.2 Config Files ตัวอย่าง

**system.site.yml**
```yaml
uuid: 12345678-1234-1234-1234-123456789012
name: 'My Drupal Site'
mail: admin@example.com
slogan: 'Built with Drupal 10'
page:
  403: ''
  404: ''
  front: /node
admin_compact_mode: false
weight_select_max: 100
langcode: en
default_langcode: en
```

**node.type.article.yml**
```yaml
langcode: en
status: true
dependencies: {  }
name: Article
type: article
description: 'Use articles for time-sensitive content...'
help: ''
new_revision: true
preview_mode: 1
display_submitted: true
```

### 5.3 Deployment Workflow

```bash
# === Development Environment ===
# 1. ทำการเปลี่ยนแปลง config ผ่าน UI
# 2. Export config ออกมา
drush cex -y

# 3. ตรวจสอบ changes
git diff config/sync/

# 4. Commit
git add config/sync/
git commit -m "Add article content type with custom fields"

# === Staging/Production Environment ===
# 5. Pull code
git pull origin main

# 6. Run database updates ก่อน (ถ้ามี)
drush updb -y

# 7. Import config
drush cim -y

# 8. Clear cache
drush cr
```

### 5.4 Config Override (settings.php)

```php
// Override config ใน settings.php (ไม่ถูก export/import)
$config['system.performance']['css']['preprocess'] = FALSE;
$config['system.site']['name'] = 'Site Name (Dev)';

// Override สำหรับ environment-specific settings
if (getenv('APP_ENV') === 'production') {
  $config['system.performance']['css']['preprocess'] = TRUE;
  $config['system.performance']['js']['preprocess'] = TRUE;
}
```

### 5.5 Config Split (สำหรับ Per-environment Config)

```bash
# ติดตั้ง config_split module
composer require drupal/config_split

# สร้าง config split สำหรับ development
drush config:set config_split.config_split.development status true
```

```yaml
# config/sync/config_split.config_split.development.yml
id: development
label: Development
description: 'Configuration for development environment'
weight: 0
status: false  # disabled by default
folder: config/dev
modules:
  devel: devel
  kint: kint
themes: {  }
blacklist: {  }
graylist: {  }
graylist_dependents: true
graylist_skip_equal: true
storage: database
```

---

## 6. Local Development Environment

### 6.1 ด้วย DDEV (แนะนำ)

```bash
# ติดตั้ง DDEV
brew install ddev/ddev/ddev  # macOS

# สร้าง DDEV config
cd my-drupal-site
ddev config --project-type=drupal10 --docroot=web --create-docroot

# เริ่ม environment
ddev start

# ติดตั้ง Drupal
ddev drush site:install --account-name=admin --account-pass=admin -y

# เปิด browser
ddev launch
```

### 6.2 ด้วย Lando

```yaml
# .lando.yml
name: my-drupal-site
recipe: drupal10
config:
  webroot: web
  php: '8.2'
  database: mysql:8.0
  drush: true
  xdebug: false
services:
  mailhog:
    type: mailhog
    hogfrom:
      - appserver
tooling:
  drush:
    service: appserver
    cmd: vendor/bin/drush
```

```bash
# เริ่ม Lando
lando start

# ใช้ drush ผ่าน Lando
lando drush cr
lando drush cim
```

### 6.3 ด้วย Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  drupal:
    image: drupal:10-php8.2-apache
    ports:
      - "8080:80"
    volumes:
      - ./web:/var/www/html
    environment:
      - DRUPAL_DB_HOST=db
      - DRUPAL_DB_NAME=drupal
      - DRUPAL_DB_USER=drupal
      - DRUPAL_DB_PASSWORD=drupal
    depends_on:
      - db

  db:
    image: mysql:8.0
    environment:
      - MYSQL_DATABASE=drupal
      - MYSQL_USER=drupal
      - MYSQL_PASSWORD=drupal
      - MYSQL_ROOT_PASSWORD=root
    volumes:
      - drupal_db:/var/lib/mysql

volumes:
  drupal_db:
```

---

## Workshop: สร้าง Drupal Project แรก

### เป้าหมาย Workshop
สร้าง Drupal 10 project สำหรับ "TechBlog" พร้อม local development environment ที่สมบูรณ์

### ขั้นตอนที่ 1: สร้างโปรเจกต์

```bash
# สร้างโปรเจกต์
composer create-project drupal/recommended-project techblog

cd techblog

# ติดตั้ง Drush
composer require drush/drush

# ติดตั้ง useful modules
composer require drupal/admin_toolbar
composer require drupal/ctools
composer require drupal/token
composer require drupal/pathauto
composer require drupal/metatag
```

### ขั้นตอนที่ 2: ตั้งค่า Database และ Settings

```bash
# สร้าง database
mysql -u root -p -e "CREATE DATABASE techblog_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -u root -p -e "CREATE USER 'techblog'@'localhost' IDENTIFIED BY 'techblog123';"
mysql -u root -p -e "GRANT ALL ON techblog_db.* TO 'techblog'@'localhost';"
```

```bash
# ติดตั้ง Drupal
vendor/bin/drush site:install standard \
  --db-url="mysql://techblog:techblog123@localhost/techblog_db" \
  --site-name="TechBlog" \
  --account-name=admin \
  --account-pass=Admin@1234 \
  --yes
```

### ขั้นตอนที่ 3: สร้าง settings.local.php

```bash
# สร้าง settings.local.php
cat > web/sites/default/settings.local.php << 'EOF'
<?php
$settings['cache']['bins']['render'] = 'cache.backend.null';
$settings['cache']['bins']['page'] = 'cache.backend.null';
$settings['cache']['bins']['dynamic_page_cache'] = 'cache.backend.null';
$config['system.logging']['error_level'] = 'verbose';
$config['system.performance']['css']['preprocess'] = FALSE;
$config['system.performance']['js']['preprocess'] = FALSE;
$settings['twig_debug'] = TRUE;
$settings['twig_auto_reload'] = TRUE;
$settings['twig_cache'] = FALSE;
EOF
```

### ขั้นตอนที่ 4: Enable Modules

```bash
# Enable contributed modules
vendor/bin/drush en admin_toolbar admin_toolbar_tools -y
vendor/bin/drush en token pathauto metatag -y

# Clear cache
vendor/bin/drush cr
```

### ขั้นตอนที่ 5: Setup Config Management

```bash
# สร้าง config/sync directory
mkdir -p config/sync

# อัปเดต settings.php ให้ชี้ไปที่ config/sync
# (ปกติ Drupal จะตั้งค่านี้ให้อัตโนมัติ)

# Export initial config
vendor/bin/drush cex -y

# ตรวจสอบว่ามี config files
ls config/sync/ | head -10
```

### ขั้นตอนที่ 6: เริ่ม Development

```bash
# เปิด login link
vendor/bin/drush uli

# เริ่ม PHP built-in server (สำหรับ development อย่างง่าย)
cd web
php -S localhost:8080
```

---

## Quiz

**ข้อ 1:** คำสั่ง Drush ใดที่ใช้ clear all caches?
- a) `drush cc all`
- b) `drush cache:rebuild`
- c) `drush cr:all`
- d) `drush flush:cache`

**เฉลย:** b) `drush cache:rebuild` (หรือย่อ `drush cr`)

---

**ข้อ 2:** Config Management ใน Drupal ทำงานอย่างไร?
- a) Export จาก files เข้า database / Import จาก database ออก files
- b) Export จาก database ออก YAML files / Import จาก YAML files เข้า database
- c) Sync โดยอัตโนมัติทุก 5 นาที
- d) ใช้ Git hooks ในการ sync

**เฉลย:** b) `drush cex` = Export ออก YAML, `drush cim` = Import เข้า database

---

**ข้อ 3:** ไฟล์ใดใน settings.php ที่ใช้สำหรับ local/development settings?
- a) settings.dev.php
- b) development.settings.php
- c) settings.local.php
- d) local.settings.php

**เฉลย:** c) `settings.local.php` - Drupal include ไฟล์นี้อัตโนมัติถ้ามีอยู่

---

**ข้อ 4:** Custom modules ควรวางไว้ที่ไหน?
- a) web/core/modules/custom/
- b) web/modules/custom/
- c) modules/custom/
- d) custom/modules/

**เฉลย:** b) `web/modules/custom/` - แยกจาก contrib modules

---

**ข้อ 5:** คำสั่ง Drush ใดที่ใช้สร้าง one-time login link?
- a) `drush user:login`
- b) `drush login:create`
- c) `drush uli`
- d) ทั้ง a และ c

**เฉลย:** d) ทั้ง `drush user:login` และ `drush uli` ใช้ได้เหมือนกัน

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- การติดตั้ง Drupal 10 ด้วย Composer อย่างถูกต้อง
- Drush CLI commands ที่จำเป็น (cr, updb, cex/cim, uli)
- โครงสร้างโปรเจกต์ Drupal ที่สำคัญ
- Configuration Management และ deployment workflow
- การตั้งค่า local development environment

**Part ถัดไป:** Part 077 - Drupal Content Types, Taxonomy & Views
