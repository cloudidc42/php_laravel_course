# Part 001: การติดตั้งและตั้งค่า PHP Environment
## ระดับ: พื้นฐาน | ขั้นตอนที่ 1-20

---

## 🎯 เป้าหมายของ Part นี้

เมื่อเรียนจบ Part นี้คุณจะสามารถ:
- ติดตั้ง PHP บน Windows, macOS และ Linux
- ตั้งค่า Web Server (Apache/Nginx)
- ใช้งาน XAMPP, MAMP, Laragon
- เข้าใจโครงสร้างของ PHP Environment
- รัน PHP Program แรกของคุณ

---

## 📖 เนื้อหา

### 1. PHP คืออะไร?

PHP (Hypertext Preprocessor) คือภาษา scripting ฝั่ง server-side ที่ออกแบบมาสำหรับการพัฒนาเว็บโดยเฉพาะ

**ประวัติย่อ:**
- 1994: Rasmus Lerdorf สร้าง PHP/FI (Personal Home Page Forms Interpreter)
- 1997: PHP 3 เปิดตัว - เขียนใหม่ทั้งหมด
- 2000: PHP 4 - Zend Engine 1.0
- 2004: PHP 5 - OOP เต็มรูปแบบ
- 2015: PHP 7 - ความเร็วเพิ่มขึ้น 2x
- 2020: PHP 8 - JIT Compiler, Named Arguments, Match Expression
- 2022: PHP 8.2 - Readonly Classes, Disjunctive Normal Form Types
- 2023: PHP 8.3 - Typed Class Constants

**PHP ใช้ทำอะไรได้บ้าง:**
```
✅ พัฒนาเว็บไซต์ Dynamic
✅ สร้าง REST API
✅ Command Line Scripts
✅ Desktop Applications (PHP-GTK)
✅ Backend สำหรับ Mobile Apps
```

**ตัวเลขที่น่าสนใจ:**
- PHP ขับเคลื่อนเว็บไซต์กว่า **79%** ของ server-side language ทั้งหมด
- WordPress, Facebook (ช่วงแรก), Wikipedia ใช้ PHP
- มี Package กว่า **300,000+** ใน Packagist

---

### 2. การติดตั้ง PHP บน Windows

#### วิธีที่ 1: ใช้ XAMPP (แนะนำสำหรับผู้เริ่มต้น)

XAMPP คือ package ที่รวม Apache + MySQL + PHP + phpMyAdmin ไว้ด้วยกัน

**ขั้นตอนติดตั้ง:**

```
1. ไปที่ https://www.apachefriends.org
2. ดาวน์โหลด XAMPP สำหรับ Windows
3. รันไฟล์ .exe ที่ดาวน์โหลดมา
4. เลือก Components ที่ต้องการ (Apache, MySQL, PHP, phpMyAdmin)
5. เลือก Installation Directory (แนะนำ: C:\xampp)
6. คลิก Install
7. รอการติดตั้งเสร็จสิ้น
```

**โครงสร้างไดเรกทอรี XAMPP:**
```
C:\xampp\
├── apache\          # Apache Web Server
├── htdocs\          # Document Root (วางไฟล์ PHP ที่นี่)
├── mysql\           # MySQL Database
├── php\             # PHP
├── phpMyAdmin\      # phpMyAdmin
└── xampp-control.exe  # Control Panel
```

**การเริ่มใช้งาน XAMPP:**
1. เปิด XAMPP Control Panel
2. คลิก Start ที่ Apache
3. คลิก Start ที่ MySQL
4. เปิด Browser ไปที่ `http://localhost`

#### วิธีที่ 2: ใช้ Laragon (แนะนำสำหรับ Developer)

Laragon เป็น development environment ที่เบากว่า XAMPP และใช้งานง่ายกว่า

**ขั้นตอนติดตั้ง:**
```
1. ดาวน์โหลดจาก https://laragon.org/download/
2. เลือก Laragon Full (มี PHP, MySQL, Node.js)
3. ติดตั้งตามขั้นตอน
4. เปิด Laragon และกด Start All
```

**ข้อดีของ Laragon:**
```
✅ Lightweight - ใช้ทรัพยากรน้อย
✅ Auto Virtual Host - สร้าง domain .test อัตโนมัติ
✅ สลับ PHP version ได้ง่าย
✅ Built-in Terminal (Git Bash, Cmder)
✅ รองรับ Laravel, WordPress, Drupal ได้ทันที
```

#### วิธีที่ 3: ติดตั้ง PHP โดยตรง

```powershell
# ใช้ Chocolatey (Windows Package Manager)
# ก่อนอื่นติดตั้ง Chocolatey:
Set-ExecutionPolicy Bypass -Scope Process -Force
iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

# ติดตั้ง PHP
choco install php

# ตรวจสอบ version
php --version
```

หรือดาวน์โหลด PHP โดยตรง:
```
1. ไปที่ https://windows.php.net/download/
2. ดาวน์โหลด PHP x64 Thread Safe
3. แตกไฟล์ไปที่ C:\php
4. เพิ่ม C:\php ใน System PATH
5. คัดลอก php.ini-development เป็น php.ini
```

---

### 3. การติดตั้ง PHP บน macOS

#### วิธีที่ 1: ใช้ Homebrew (แนะนำ)

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง PHP
brew install php

# ตรวจสอบ version
php --version

# ติดตั้ง PHP version เฉพาะ
brew install php@8.2

# สลับ PHP version
brew unlink php && brew link php@8.2

# ติดตั้ง MySQL
brew install mysql

# เริ่มใช้งาน MySQL
brew services start mysql
```

#### วิธีที่ 2: ใช้ MAMP

```
1. ดาวน์โหลดจาก https://www.mamp.info
2. ติดตั้งและเปิดใช้งาน
3. กด Start Servers
4. เปิด Browser ไปที่ http://localhost:8888
```

**Document Root ของ MAMP:**
```
/Applications/MAMP/htdocs/
```

#### วิธีที่ 3: ใช้ Valet (สำหรับ Laravel Developer)

```bash
# ติดตั้ง Composer ก่อน
brew install composer

# ติดตั้ง Valet
composer global require laravel/valet

# ติดตั้ง Valet
valet install

# กำหนด Directory สำหรับ project
mkdir ~/Sites
cd ~/Sites
valet park

# สร้าง Project - เข้าถึงผ่าน project-name.test
mkdir myproject
# เข้าถึงได้ที่ http://myproject.test
```

---

### 4. การติดตั้ง PHP บน Linux (Ubuntu/Debian)

#### ติดตั้งผ่าน APT

```bash
# อัพเดท package list
sudo apt update

# ติดตั้ง PHP และ extensions ที่จำเป็น
sudo apt install php php-cli php-common php-mysql php-zip php-gd \
    php-mbstring php-curl php-xml php-bcmath php-json -y

# ตรวจสอบ version
php --version

# ติดตั้ง Apache
sudo apt install apache2 -y
sudo systemctl start apache2
sudo systemctl enable apache2

# ติดตั้ง MySQL
sudo apt install mysql-server -y
sudo systemctl start mysql

# เชื่อม PHP กับ Apache
sudo apt install libapache2-mod-php -y
sudo systemctl restart apache2
```

#### ติดตั้ง PHP 8.3 บน Ubuntu

```bash
# เพิ่ม PPA repository
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update

# ติดตั้ง PHP 8.3
sudo apt install php8.3 php8.3-cli php8.3-common php8.3-mysql \
    php8.3-zip php8.3-gd php8.3-mbstring php8.3-curl \
    php8.3-xml php8.3-bcmath -y

# ตรวจสอบ
php8.3 --version

# ตั้งค่า default PHP version
sudo update-alternatives --set php /usr/bin/php8.3
```

#### ติดตั้งบน Fedora/CentOS/RHEL

```bash
# Fedora
sudo dnf install php php-cli php-mysqlnd php-zip php-gd \
    php-mbstring php-curl php-xml php-bcmath -y

# CentOS 8+ / RHEL 8+
sudo dnf install epel-release -y
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-8.rpm -y
sudo dnf module enable php:remi-8.3 -y
sudo dnf install php php-cli php-mysqlnd -y
```

---

### 5. การตั้งค่า PHP (php.ini)

`php.ini` คือไฟล์ configuration หลักของ PHP

**ค้นหา php.ini:**
```bash
php --ini
# หรือ
php -i | grep "Configuration File"
```

**การตั้งค่าที่สำคัญ:**

```ini
; php.ini

; ========================
; Error Reporting
; ========================
; Development - แสดง error ทั้งหมด
error_reporting = E_ALL
display_errors = On
display_startup_errors = On
log_errors = On
error_log = /var/log/php_errors.log

; Production - ซ่อน error จากผู้ใช้
; error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
; display_errors = Off
; log_errors = On

; ========================
; Resource Limits
; ========================
max_execution_time = 300          ; เวลาสูงสุดในการรัน script (วินาที)
max_input_time = 300              ; เวลาสูงสุดในการรับ input
memory_limit = 256M               ; หน่วยความจำสูงสุด

; ========================
; File Uploads
; ========================
file_uploads = On
upload_max_filesize = 20M         ; ขนาดไฟล์ upload สูงสุด
max_file_uploads = 20             ; จำนวนไฟล์ upload สูงสุด
post_max_size = 25M               ; ขนาด POST data สูงสุด

; ========================
; Date/Time
; ========================
date.timezone = "Asia/Bangkok"    ; Timezone สำหรับไทย

; ========================
; Extensions
; ========================
extension=pdo_mysql
extension=mbstring
extension=gd
extension=curl
extension=zip
extension=xml
extension=bcmath
extension=openssl

; ========================
; Session
; ========================
session.gc_maxlifetime = 3600     ; Session หมดอายุใน 1 ชั่วโมง
session.cookie_httponly = 1       ; ป้องกัน XSS
session.cookie_secure = 1         ; ใช้ HTTPS เท่านั้น (Production)
session.use_strict_mode = 1

; ========================
; OPcache (Performance)
; ========================
opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=4000
opcache.revalidate_freq=60
opcache.fast_shutdown=1
```

---

### 6. การติดตั้ง Composer

Composer คือ Package Manager สำหรับ PHP ที่ขาดไม่ได้

#### ติดตั้งบน Windows:
```
1. ดาวน์โหลด Composer-Setup.exe จาก https://getcomposer.org
2. รันและทำตามขั้นตอน
3. เลือก PHP executable ที่ติดตั้งไว้
4. ทดสอบ: composer --version
```

#### ติดตั้งบน macOS/Linux:
```bash
# ดาวน์โหลด installer
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"

# ตรวจสอบ hash (ดู hash ล่าสุดที่ https://composer.github.io/pubkeys.html)
php -r "if (hash_file('sha384', 'composer-setup.php') === 'HASH_HERE') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"

# ติดตั้ง
php composer-setup.php --install-dir=/usr/local/bin --filename=composer

# ลบ installer
php -r "unlink('composer-setup.php');"

# ทดสอบ
composer --version
```

**Composer Commands พื้นฐาน:**
```bash
composer --version              # ดู version
composer self-update            # อัพเดท composer
composer create-project         # สร้าง project ใหม่
composer require package/name   # ติดตั้ง package
composer install                # ติดตั้ง packages จาก composer.json
composer update                 # อัพเดท packages
composer dump-autoload          # อัพเดท autoload
composer show                   # แสดง packages ที่ติดตั้ง
```

---

### 7. การตั้งค่า VS Code สำหรับ PHP

VS Code เป็น Editor ที่แนะนำสำหรับการพัฒนา PHP

**Extensions ที่จำเป็น:**

```
1. PHP Intelephense - Code intelligence สำหรับ PHP
2. PHP Debug - Debug PHP ด้วย Xdebug
3. PHP Sniffer & Beautifier - Code formatting
4. Laravel Blade Snippets - สำหรับ Laravel
5. GitLens - Git integration
6. Thunder Client - REST API testing
7. Auto Rename Tag - HTML tag renaming
```

**settings.json สำหรับ PHP:**
```json
{
    "php.validate.executablePath": "/usr/bin/php",
    "php.suggest.basic": false,
    "[php]": {
        "editor.defaultFormatter": "bmewburn.vscode-intelephense-client",
        "editor.formatOnSave": true,
        "editor.tabSize": 4,
        "editor.insertSpaces": true
    },
    "intelephense.environment.phpVersion": "8.3.0",
    "intelephense.files.maxSize": 5000000,
    "editor.rulers": [80, 120],
    "files.associations": {
        "*.php": "php"
    }
}
```

**การติดตั้ง Xdebug:**
```bash
# Linux/macOS
sudo apt install php-xdebug  # Ubuntu

# หรือ ใช้ PECL
pecl install xdebug

# เพิ่มใน php.ini
zend_extension=xdebug
xdebug.mode=debug
xdebug.start_with_request=yes
xdebug.client_port=9003
xdebug.client_host=localhost
```

---

### 8. การสร้าง PHP Project แรก

#### โครงสร้าง Project ง่ายๆ:

```
my-first-php/
├── index.php
├── config/
│   └── database.php
├── includes/
│   ├── header.php
│   └── footer.php
├── assets/
│   ├── css/
│   │   └── style.css
│   └── js/
│       └── main.js
└── pages/
    ├── home.php
    └── about.php
```

**index.php:**
```php
<?php
// บรรทัดแรกของ PHP ต้องเริ่มด้วย <?php
// ไม่ต้องมี closing tag ?> ถ้าไฟล์มีแค่ PHP

// แสดงข้อความ
echo "สวัสดีโลก PHP!";
echo "<br>";
echo "นี่คือ PHP version: " . PHP_VERSION;
echo "<br>";
echo "วันที่: " . date('d/m/Y');

// phpinfo() - แสดงข้อมูล PHP ทั้งหมด (ใช้แค่ตอน development)
// phpinfo();
```

**การรัน PHP Built-in Server:**
```bash
# รัน PHP development server
cd my-first-php
php -S localhost:8000

# หรือกำหนด document root
php -S localhost:8000 -t public/

# เปิด Browser: http://localhost:8000
```

---

### 9. PHP Tags และรูปแบบต่างๆ

```php
<?php
// Standard PHP Tags - แนะนำ
echo "Standard Tag";
?>

<?= "Short Echo Tag" ?>
<!-- เทียบเท่ากับ <?php echo "Short Echo Tag"; ?> -->

<?php
// การฝัง PHP ใน HTML
$name = "สมชาย";
$age = 25;
?>

<!DOCTYPE html>
<html>
<head>
    <title>PHP Test</title>
</head>
<body>
    <h1>สวัสดี <?= $name ?></h1>
    <p>อายุ <?= $age ?> ปี</p>
    
    <?php if ($age >= 18): ?>
        <p>คุณเป็นผู้ใหญ่แล้ว</p>
    <?php else: ?>
        <p>คุณยังเป็นเยาวชน</p>
    <?php endif; ?>
</body>
</html>
```

---

### 10. การทดสอบ Environment

สร้างไฟล์ `test-environment.php`:

```php
<?php
declare(strict_types=1);

echo "<h1>PHP Environment Test</h1>";
echo "<hr>";

// PHP Version
echo "<h2>PHP Information</h2>";
echo "PHP Version: " . PHP_VERSION . "<br>";
echo "PHP Major Version: " . PHP_MAJOR_VERSION . "<br>";
echo "PHP OS: " . PHP_OS . "<br>";
echo "PHP SAPI: " . PHP_SAPI . "<br>";
echo "Script: " . __FILE__ . "<br>";
echo "Directory: " . __DIR__ . "<br>";
echo "<hr>";

// Extensions
echo "<h2>Loaded Extensions</h2>";
$important_extensions = [
    'pdo', 'pdo_mysql', 'mbstring', 'gd', 
    'curl', 'zip', 'xml', 'bcmath', 'openssl', 'json'
];

echo "<table border='1'>";
echo "<tr><th>Extension</th><th>Status</th></tr>";
foreach ($important_extensions as $ext) {
    $loaded = extension_loaded($ext);
    $status = $loaded ? '✅ Loaded' : '❌ Not Loaded';
    echo "<tr><td>$ext</td><td>$status</td></tr>";
}
echo "</table>";
echo "<hr>";

// PHP Configuration
echo "<h2>PHP Configuration</h2>";
$configs = [
    'memory_limit',
    'max_execution_time',
    'upload_max_filesize',
    'post_max_size',
    'date.timezone',
    'display_errors',
    'error_reporting',
];

echo "<table border='1'>";
echo "<tr><th>Setting</th><th>Value</th></tr>";
foreach ($configs as $config) {
    $value = ini_get($config);
    echo "<tr><td>$config</td><td>$value</td></tr>";
}
echo "</table>";
echo "<hr>";

// Database Connection Test
echo "<h2>Database Connection Test</h2>";
try {
    $pdo = new PDO(
        'mysql:host=localhost;dbname=test;charset=utf8mb4',
        'root',
        '',
        [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
    );
    echo "✅ MySQL Connection: Success<br>";
    $version = $pdo->query('SELECT VERSION()')->fetchColumn();
    echo "MySQL Version: " . $version . "<br>";
} catch (PDOException $e) {
    echo "❌ MySQL Connection: Failed - " . $e->getMessage() . "<br>";
    echo "(นี่เป็นเรื่องปกติถ้ายังไม่ได้ตั้งค่า database)<br>";
}
echo "<hr>";

// Composer Check
echo "<h2>Composer Check</h2>";
if (file_exists(__DIR__ . '/vendor/autoload.php')) {
    echo "✅ Composer vendor directory found<br>";
} else {
    echo "⚠️ Composer vendor directory not found (รัน composer install ก่อน)<br>";
}

echo "<p><strong>Environment setup is ready! 🚀</strong></p>";
```

---

### 11. ตัวอย่าง Hello World ระดับต่างๆ

#### Hello World ระดับพื้นฐาน:
```php
<?php
echo "Hello, World!";
```

#### Hello World พร้อม HTML:
```php
<?php
$greeting = "สวัสดีโลก";
$language = "PHP";
$version = PHP_VERSION;
?>
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><?= $greeting ?></title>
    <style>
        body {
            font-family: 'Sarabun', sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
        }
        .card {
            background: white;
            padding: 40px;
            border-radius: 10px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.2);
            text-align: center;
        }
        h1 { color: #667eea; }
        .badge {
            background: #667eea;
            color: white;
            padding: 5px 15px;
            border-radius: 20px;
            font-size: 0.9em;
        }
    </style>
</head>
<body>
    <div class="card">
        <h1><?= $greeting ?></h1>
        <p>ยินดีต้อนรับสู่โลกของ <?= $language ?></p>
        <span class="badge">PHP <?= $version ?></span>
        <br><br>
        <small>วันที่: <?= date('d/m/Y H:i:s') ?></small>
    </div>
</body>
</html>
```

---

### 12. Environment Variables ด้วย .env

**.env file:**
```ini
APP_NAME="My PHP App"
APP_ENV=development
APP_DEBUG=true
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=myapp
DB_USERNAME=root
DB_PASSWORD=

MAIL_DRIVER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your@email.com
MAIL_PASSWORD=yourpassword
```

**การอ่าน .env ด้วย PHP:**
```php
<?php
// สร้างไฟล์ env.php
function loadEnv(string $path): void
{
    if (!file_exists($path)) {
        return;
    }
    
    $lines = file($path, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
    
    foreach ($lines as $line) {
        // ข้าม comments
        if (str_starts_with(trim($line), '#')) {
            continue;
        }
        
        // แยก key=value
        if (str_contains($line, '=')) {
            [$key, $value] = explode('=', $line, 2);
            $key = trim($key);
            $value = trim($value, " \t\n\r\0\x0B\"'");
            
            $_ENV[$key] = $value;
            putenv("$key=$value");
        }
    }
}

// โหลด .env
loadEnv(__DIR__ . '/.env');

// ใช้งาน
$appName = $_ENV['APP_NAME'] ?? 'Default App';
$dbHost = getenv('DB_HOST') ?: 'localhost';

echo "App: $appName" . PHP_EOL;
echo "DB Host: $dbHost" . PHP_EOL;
```

---

### 13. Docker สำหรับ PHP Development

สำหรับ Developer ระดับสูงขึ้น Docker เป็นทางเลือกที่ดีมาก

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  app:
    image: php:8.3-apache
    container_name: php_app
    ports:
      - "8080:80"
    volumes:
      - ./src:/var/www/html
    depends_on:
      - db
    environment:
      DB_HOST: db
      DB_DATABASE: myapp
      DB_USERNAME: root
      DB_PASSWORD: secret

  db:
    image: mysql:8.0
    container_name: php_db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: myapp
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql

  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    container_name: php_pma
    ports:
      - "8081:80"
    environment:
      PMA_HOST: db
      PMA_USER: root
      PMA_PASSWORD: secret

volumes:
  db_data:
```

**การใช้งาน:**
```bash
# เริ่ม containers
docker-compose up -d

# ดู containers ที่รัน
docker-compose ps

# เข้าถึง app container
docker-compose exec app bash

# หยุด containers
docker-compose down
```

**Dockerfile สำหรับ PHP:**
```dockerfile
FROM php:8.3-apache

# ติดตั้ง dependencies
RUN apt-get update && apt-get install -y \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    zip \
    unzip \
    git \
    curl

# ติดตั้ง PHP extensions
RUN docker-php-ext-configure gd --with-freetype --with-jpeg
RUN docker-php-ext-install -j$(nproc) gd pdo pdo_mysql mbstring zip bcmath

# ติดตั้ง Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# ตั้งค่า Apache
RUN a2enmod rewrite

# Copy source code
COPY . /var/www/html/

# ตั้งค่า permissions
RUN chown -R www-data:www-data /var/www/html
RUN chmod -R 755 /var/www/html

EXPOSE 80
```

---

### 14. สรุป Environment Setup Checklist

```
✅ PHP ติดตั้งแล้ว (version 8.0+)
✅ Web Server ทำงาน (Apache/Nginx)
✅ MySQL/MariaDB ติดตั้งแล้ว
✅ Composer ติดตั้งแล้ว
✅ VS Code พร้อม Extensions
✅ php.ini ตั้งค่าแล้ว
   - date.timezone = Asia/Bangkok
   - display_errors = On (Development)
   - memory_limit = 256M
✅ Extensions สำคัญ Load แล้ว
   - pdo_mysql, mbstring, gd, curl, zip
✅ สามารถรัน PHP ได้
✅ .env file setup (ถ้าต้องการ)
```

---

## 🛠️ Workshop: ตั้งค่า Environment

### Exercise 1: ติดตั้งและทดสอบ

```bash
# 1. ตรวจสอบ PHP version
php --version

# 2. ดู php.ini location
php --ini

# 3. รัน script จาก command line
php -r "echo 'PHP is working! Version: ' . PHP_VERSION . PHP_EOL;"

# 4. รัน built-in server
mkdir ~/php-test
echo '<?php echo "Hello World!";' > ~/php-test/index.php
cd ~/php-test
php -S localhost:8080
```

### Exercise 2: สร้าง PHP Info Page

```php
<?php
// บันทึกเป็น info.php
// เข้าถึง http://localhost:8080/info.php
phpinfo();
```

**⚠️ คำเตือน:** ลบ phpinfo() ออกเสมอก่อน Deploy ขึ้น Production เพราะจะเปิดเผยข้อมูล server

### Exercise 3: ทดสอบ Database Connection

```php
<?php
$host = 'localhost';
$dbname = 'test';
$username = 'root';
$password = '';

try {
    $pdo = new PDO(
        "mysql:host=$host;charset=utf8mb4",
        $username,
        $password,
        [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        ]
    );
    
    echo "✅ เชื่อมต่อ MySQL สำเร็จ!<br>";
    
    // สร้าง database
    $pdo->exec("CREATE DATABASE IF NOT EXISTS `test` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci");
    echo "✅ สร้าง/ตรวจสอบ database 'test' สำเร็จ!<br>";
    
    // ดู databases ทั้งหมด
    $stmt = $pdo->query("SHOW DATABASES");
    echo "<br>Databases:<br>";
    while ($row = $stmt->fetch(PDO::FETCH_NUM)) {
        echo "- " . $row[0] . "<br>";
    }
    
} catch (PDOException $e) {
    echo "❌ Error: " . $e->getMessage() . "<br>";
}
```

---

## 📝 Quiz: ทดสอบความเข้าใจ

1. PHP ย่อมาจากอะไร?
2. ไฟล์ configuration ของ PHP เรียกว่าอะไร?
3. Composer ใช้ทำอะไร?
4. ความแตกต่างระหว่าง `<?php echo` กับ `<?=` คืออะไร?
5. ทำไมถึงไม่ควรแสดง phpinfo() บน Production?
6. OPcache ช่วยอะไรใน PHP?
7. extension ใดที่ใช้เชื่อมต่อกับ MySQL?

**เฉลย:**
1. PHP: Hypertext Preprocessor
2. php.ini
3. Package Manager สำหรับ PHP - จัดการ dependencies
4. `<?=` เป็น shorthand ของ `<?php echo` - ทำงานเหมือนกัน
5. เปิดเผยข้อมูล server ที่อาจนำไปสู่ช่องโหว่ด้านความปลอดภัย
6. Cache compiled PHP bytecode เพื่อเพิ่มความเร็ว
7. `pdo_mysql` หรือ `mysqli`

---

## 🔗 แหล่งเรียนรู้เพิ่มเติม

- [PHP Manual (Official)](https://www.php.net/manual/en/)
- [PHP: The Right Way](https://phptherightway.com/)
- [Laracasts](https://laracasts.com/)
- [PHP Packagist](https://packagist.org/)

---

## ⏭️ Part ถัดไป

**Part 002: PHP Syntax พื้นฐาน - Variables & Data Types**

เราจะเรียนรู้:
- Variables และการตั้งชื่อ
- Data Types ทั้งหมดใน PHP
- Type Juggling และ Type Casting
- Constants
- Variable Variables

---

*Part 001 | ระดับพื้นฐาน | PHP Environment Setup*
