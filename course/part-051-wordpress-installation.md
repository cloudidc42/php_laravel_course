# Part 051: WordPress Installation & Project Structure

**ระดับ:** Intermediate  
**เวลาเรียน:** 3-4 ชั่วโมง  
**Prerequisites:** PHP พื้นฐาน, MySQL, Web Server (Apache/Nginx)

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. ติดตั้ง WordPress บน Local Environment ได้
2. ใช้ WP-CLI เพื่อจัดการ WordPress ผ่าน Command Line
3. เข้าใจโครงสร้างไฟล์และโฟลเดอร์ของ WordPress
4. เข้าใจโครงสร้าง Database ของ WordPress
5. ตั้งค่า WordPress สำหรับ Development Environment

---

## 1. การติดตั้ง WordPress

### 1.1 ความต้องการของระบบ (System Requirements)

WordPress ต้องการ:
- **PHP:** 7.4 หรือใหม่กว่า (แนะนำ 8.0+)
- **MySQL:** 5.7+ หรือ MariaDB 10.3+
- **Web Server:** Apache หรือ Nginx
- **RAM:** อย่างน้อย 512MB (แนะนำ 1GB+)

### 1.2 การดาวน์โหลดและติดตั้ง WordPress

#### วิธีที่ 1: Manual Installation

```bash
# ดาวน์โหลด WordPress ล่าสุด
wget https://wordpress.org/latest.tar.gz

# แตกไฟล์
tar -xzvf latest.tar.gz

# ย้ายไปยัง web root
sudo mv wordpress /var/www/html/mysite

# ตั้งสิทธิ์ไฟล์
sudo chown -R www-data:www-data /var/www/html/mysite
sudo chmod -R 755 /var/www/html/mysite
```

#### วิธีที่ 2: Composer (แนะนำสำหรับ Development)

```bash
# สร้างโปรเจกต์ใหม่ด้วย Composer
composer create-project roots/bedrock mysite

# หรือใช้ WordPress packagist
composer require johnpbloch/wordpress
```

#### การสร้าง Database

```sql
-- สร้าง Database สำหรับ WordPress
CREATE DATABASE wordpress_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- สร้าง User และให้สิทธิ์
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'strong_password_here';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
```

### 1.3 การตั้งค่า wp-config.php

```php
<?php
/**
 * The base configuration for WordPress
 * 
 * ไฟล์ wp-config.php เก็บการตั้งค่าหลักของ WordPress
 */

// ** Database settings ** //
define( 'DB_NAME', 'wordpress_db' );        // ชื่อ Database
define( 'DB_USER', 'wp_user' );             // Username ของ Database
define( 'DB_PASSWORD', 'strong_password' ); // Password ของ Database
define( 'DB_HOST', 'localhost' );           // Database Host
define( 'DB_CHARSET', 'utf8mb4' );          // Character Set
define( 'DB_COLLATE', '' );                 // Database Collation

// Authentication Keys and Salts
// สร้างใหม่ได้ที่: https://api.wordpress.org/secret-key/1.1/salt/
define( 'AUTH_KEY',         'your-unique-phrase-here' );
define( 'SECURE_AUTH_KEY',  'your-unique-phrase-here' );
define( 'LOGGED_IN_KEY',    'your-unique-phrase-here' );
define( 'NONCE_KEY',        'your-unique-phrase-here' );
define( 'AUTH_SALT',        'your-unique-phrase-here' );
define( 'SECURE_AUTH_SALT', 'your-unique-phrase-here' );
define( 'LOGGED_IN_SALT',   'your-unique-phrase-here' );
define( 'NONCE_SALT',       'your-unique-phrase-here' );

// Database Table Prefix
$table_prefix = 'wp_';

// WordPress Debug Mode (เปิดระหว่าง Development)
define( 'WP_DEBUG', true );
define( 'WP_DEBUG_LOG', true );   // บันทึก error ใน wp-content/debug.log
define( 'WP_DEBUG_DISPLAY', false ); // ไม่แสดง error บนหน้าจอ

// URL Settings
define( 'WP_HOME', 'http://localhost/mysite' );
define( 'WP_SITEURL', 'http://localhost/mysite' );

// WordPress Memory Limit
define( 'WP_MEMORY_LIMIT', '256M' );

// การจำกัด Post Revisions (ประหยัด Database)
define( 'WP_POST_REVISIONS', 5 );

// Auto-save ทุก 60 วินาที
define( 'AUTOSAVE_INTERVAL', 60 );

// Disable File Editing ใน Admin (Security)
define( 'DISALLOW_FILE_EDIT', true );

// Absolute path to WordPress directory
if ( ! defined( 'ABSPATH' ) ) {
    define( 'ABSPATH', __DIR__ . '/' );
}

// Sets up WordPress vars and included files
require_once ABSPATH . 'wp-settings.php';
```

---

## 2. WP-CLI (WordPress Command Line Interface)

WP-CLI เป็นเครื่องมือสำคัญสำหรับ WordPress Developer ช่วยให้จัดการ WordPress ผ่าน Terminal ได้อย่างรวดเร็ว

### 2.1 การติดตั้ง WP-CLI

```bash
# ดาวน์โหลด WP-CLI
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar

# ทดสอบว่าทำงานได้
php wp-cli.phar --info

# ทำให้ใช้ได้ทั่วระบบ
chmod +x wp-cli.phar
sudo mv wp-cli.phar /usr/local/bin/wp

# ตรวจสอบ version
wp --version
```

### 2.2 คำสั่ง WP-CLI พื้นฐาน

```bash
# ==========================================
# WordPress Core
# ==========================================

# ติดตั้ง WordPress
wp core download --locale=th

# ตรวจสอบ version
wp core version

# อัปเดต WordPress
wp core update

# ==========================================
# การจัดการ Database
# ==========================================

# สร้าง Database Tables
wp db create

# Import SQL file
wp db import backup.sql

# Export Database
wp db export backup.sql

# รัน SQL query
wp db query "SELECT * FROM wp_users LIMIT 5"

# ==========================================
# การจัดการ User
# ==========================================

# สร้าง Admin User ใหม่
wp user create admin admin@example.com \
    --role=administrator \
    --user_pass=SecurePassword123!

# ดู User ทั้งหมด
wp user list

# Reset Password
wp user update admin --user_pass=NewPassword456!

# ==========================================
# การจัดการ Plugin
# ==========================================

# ติดตั้ง Plugin
wp plugin install woocommerce --activate

# ดู Plugin ทั้งหมด
wp plugin list

# อัปเดต Plugin ทั้งหมด
wp plugin update --all

# ลบ Plugin
wp plugin delete hello-dolly

# ==========================================
# การจัดการ Theme
# ==========================================

# ติดตั้ง Theme
wp theme install twentytwentyfour --activate

# ดู Theme ทั้งหมด
wp theme list

# สร้าง Child Theme
wp scaffold child-theme mytheme --parent_theme=twentytwentyfour

# ==========================================
# การจัดการ Post
# ==========================================

# สร้าง Post
wp post create \
    --post_title="Hello World" \
    --post_content="Content here" \
    --post_status=publish

# ดู Post ทั้งหมด
wp post list --post_type=post --format=table

# ลบ Post
wp post delete 1 --force

# ==========================================
# Search and Replace
# ==========================================

# เปลี่ยน URL (เมื่อ Migrate)
wp search-replace 'http://old-domain.com' 'http://new-domain.com'

# Dry run ก่อน (ไม่บันทึกจริง)
wp search-replace 'old' 'new' --dry-run

# ==========================================
# การ Generate Code
# ==========================================

# สร้าง Plugin skeleton
wp scaffold plugin my-plugin

# สร้าง Post Type
wp scaffold post-type book --plugin=my-plugin

# สร้าง Taxonomy
wp scaffold taxonomy genre --post_types=book

# Generate PHPUnit tests
wp scaffold plugin-tests my-plugin
```

### 2.3 WP-CLI สำหรับ Development Workflow

```bash
# ==========================================
# Development Workflow Script
# ==========================================

#!/bin/bash
# setup-dev.sh - ตั้งค่า WordPress สำหรับ Development

# ดาวน์โหลด WordPress
wp core download --locale=th

# ตั้งค่า wp-config.php
wp config create \
    --dbname=wordpress_dev \
    --dbuser=root \
    --dbpass=password \
    --dbhost=localhost \
    --extra-php <<PHP
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);
define('WP_DEBUG_DISPLAY', false);
define('SAVEQUERIES', true);
PHP

# ติดตั้ง WordPress
wp core install \
    --url=http://localhost/mysite \
    --title="My WordPress Site" \
    --admin_user=admin \
    --admin_password=admin123 \
    --admin_email=admin@example.com

# ติดตั้ง Plugin สำหรับ Development
wp plugin install query-monitor --activate
wp plugin install debug-bar --activate

# ลบ Plugin ที่ไม่ต้องการ
wp plugin delete akismet hello

# ตั้งค่า Permalink
wp rewrite structure '/%postname%/' --hard

# Import Dummy Content
wp plugin install --activate wordpress-importer
wp import dummy-content.xml --authors=create

echo "WordPress Development Environment Ready!"
```

---

## 3. โครงสร้างไฟล์และโฟลเดอร์ WordPress

```
wordpress/
├── wp-admin/                    # Admin Interface
│   ├── css/                     # Stylesheets สำหรับ Admin
│   ├── images/                  # รูปภาพใน Admin
│   ├── includes/                # PHP Files ของ Admin
│   ├── js/                      # JavaScript ใน Admin
│   └── index.php                # Admin Entry Point
│
├── wp-content/                  # Content Directory (สำคัญมาก!)
│   ├── plugins/                 # Plugin Files
│   │   ├── my-plugin/           # Custom Plugin ของเรา
│   │   └── woocommerce/         # Third-party Plugin
│   │
│   ├── themes/                  # Theme Files
│   │   ├── twentytwentyfour/    # Default Theme
│   │   └── my-theme/            # Custom Theme ของเรา
│   │
│   ├── uploads/                 # Media Files (รูปภาพ, วิดีโอ)
│   │   └── 2024/
│   │       └── 01/
│   │           ├── image.jpg
│   │           └── image-300x200.jpg  # Thumbnail
│   │
│   ├── mu-plugins/              # Must-Use Plugins (โหลดเสมอ)
│   └── upgrade/                 # Temp files ระหว่าง Update
│
├── wp-includes/                 # WordPress Core Files
│   ├── js/                      # Core JavaScript
│   ├── css/                     # Core CSS
│   ├── class-wp.php             # Main WP Class
│   ├── functions.php            # Core Functions
│   ├── post.php                 # Post Functions
│   ├── query.php                # WP_Query Class
│   ├── template.php             # Template Functions
│   └── ...
│
├── index.php                    # Main Entry Point
├── wp-config.php                # Configuration File
├── wp-login.php                 # Login Page
├── wp-cron.php                  # Cron Job Handler
├── wp-signup.php                # User Registration
├── xmlrpc.php                   # XML-RPC API (Legacy)
└── .htaccess                    # Apache Configuration
```

### 3.1 ความสำคัญของแต่ละโฟลเดอร์

```php
<?php
// ค่า Constants ที่สำคัญใน WordPress
// สามารถใช้ในโค้ดของเราได้

echo ABSPATH;           // /var/www/html/mysite/
echo WPINC;             // wp-includes
echo WP_CONTENT_DIR;    // /var/www/html/mysite/wp-content
echo WP_CONTENT_URL;    // http://mysite.com/wp-content
echo WP_PLUGIN_DIR;     // /var/www/html/mysite/wp-content/plugins
echo WP_PLUGIN_URL;     // http://mysite.com/wp-content/plugins
echo get_template_directory();      // Path ของ Active Theme
echo get_template_directory_uri();  // URL ของ Active Theme
echo get_stylesheet_directory();    // Path ของ Child Theme (ถ้ามี)
```

---

## 4. โครงสร้าง Database WordPress

WordPress ใช้ตาราง (Tables) เริ่มต้น 12 ตาราง:

### 4.1 ตารางหลัก

```sql
-- ==========================================
-- wp_posts - เก็บ Content ทุกประเภท
-- ==========================================
DESCRIBE wp_posts;
/*
ID                    bigint(20) unsigned AUTO_INCREMENT
post_author           bigint(20) unsigned (FK -> wp_users.ID)
post_date             datetime
post_date_gmt         datetime
post_content          longtext
post_title            text
post_excerpt          text
post_status           varchar(20)  [publish, draft, private, trash, ...]
comment_status        varchar(20)
ping_status           varchar(20)
post_password         varchar(255)
post_name             varchar(200)  (slug)
post_modified         datetime
post_modified_gmt     datetime
post_parent           bigint(20) unsigned
guid                  varchar(255)
menu_order            int(11)
post_type             varchar(20)  [post, page, attachment, ...]
post_mime_type        varchar(100)
comment_count         bigint(20)
*/

-- ==========================================
-- wp_postmeta - เก็บ Custom Fields
-- ==========================================
DESCRIBE wp_postmeta;
/*
meta_id     bigint(20) unsigned AUTO_INCREMENT
post_id     bigint(20) unsigned (FK -> wp_posts.ID)
meta_key    varchar(255)
meta_value  longtext
*/

-- ==========================================
-- wp_users - เก็บข้อมูล Users
-- ==========================================
DESCRIBE wp_users;
/*
ID                  bigint(20) unsigned
user_login          varchar(60)
user_pass           varchar(255)  (hashed password)
user_nicename       varchar(50)
user_email          varchar(100)
user_url            varchar(100)
user_registered     datetime
user_activation_key varchar(255)
user_status         int(11)
display_name        varchar(250)
*/

-- ==========================================
-- wp_usermeta - เก็บข้อมูลเพิ่มเติมของ User
-- ==========================================
DESCRIBE wp_usermeta;
/*
umeta_id    bigint(20) unsigned
user_id     bigint(20) unsigned (FK -> wp_users.ID)
meta_key    varchar(255)
meta_value  longtext
*/

-- ==========================================
-- wp_terms - เก็บ Categories, Tags, Taxonomies
-- ==========================================
DESCRIBE wp_terms;
/*
term_id     bigint(20) unsigned
name        varchar(200)
slug        varchar(200)
term_group  bigint(10)
*/

-- ==========================================
-- wp_term_taxonomy - เชื่อม Term กับ Taxonomy
-- ==========================================
DESCRIBE wp_term_taxonomy;
/*
term_taxonomy_id bigint(20) unsigned
term_id          bigint(20) unsigned (FK -> wp_terms.term_id)
taxonomy         varchar(32)  [category, post_tag, ...]
description      longtext
parent           bigint(20) unsigned
count            bigint(20)
*/

-- ==========================================
-- wp_term_relationships - เชื่อม Post กับ Term
-- ==========================================
DESCRIBE wp_term_relationships;
/*
object_id        bigint(20) unsigned (FK -> wp_posts.ID)
term_taxonomy_id bigint(20) unsigned
term_order       int(11)
*/

-- ==========================================
-- wp_options - เก็บ Settings ทั้งหมด
-- ==========================================
DESCRIBE wp_options;
/*
option_id    bigint(20) unsigned
option_name  varchar(191)  (unique)
option_value longtext
autoload     varchar(20)   [yes/no]
*/

-- ==========================================
-- wp_comments - เก็บ Comments
-- ==========================================
DESCRIBE wp_comments;
/*
comment_ID           bigint(20) unsigned
comment_post_ID      bigint(20) unsigned (FK -> wp_posts.ID)
comment_author       tinytext
comment_author_email varchar(100)
comment_author_url   varchar(200)
comment_author_IP    varchar(100)
comment_date         datetime
comment_date_gmt     datetime
comment_content      text
comment_karma        int(11)
comment_approved     varchar(20)
comment_agent        varchar(255)
comment_type         varchar(20)
comment_parent       bigint(20) unsigned
user_id              bigint(20) unsigned
*/
```

### 4.2 การ Query Database ด้วย wpdb

```php
<?php
/**
 * การใช้ $wpdb - WordPress Database Object
 * 
 * ใช้ $wpdb แทนการ Query MySQL โดยตรงเพื่อความปลอดภัย
 */

global $wpdb;

// ==========================================
// SELECT - ดึงข้อมูล
// ==========================================

// ดึงหลาย rows
$posts = $wpdb->get_results(
    "SELECT ID, post_title FROM {$wpdb->posts} WHERE post_status = 'publish'",
    ARRAY_A  // OBJECT (default), ARRAY_A (associative), ARRAY_N (numeric)
);

// ดึง row เดียว
$post = $wpdb->get_row(
    $wpdb->prepare(
        "SELECT * FROM {$wpdb->posts} WHERE ID = %d",
        42
    )
);

// ดึงค่าเดียว
$count = $wpdb->get_var(
    "SELECT COUNT(*) FROM {$wpdb->posts} WHERE post_type = 'post'"
);

// ดึงหนึ่ง column
$titles = $wpdb->get_col(
    "SELECT post_title FROM {$wpdb->posts} WHERE post_status = 'publish'"
);

// ==========================================
// INSERT - เพิ่มข้อมูล
// ==========================================

$result = $wpdb->insert(
    $wpdb->postmeta,         // ชื่อตาราง
    array(                    // ข้อมูลที่จะเพิ่ม
        'post_id'    => 42,
        'meta_key'   => '_my_custom_field',
        'meta_value' => 'my_value',
    ),
    array( '%d', '%s', '%s' ) // Format ของแต่ละค่า
);

$inserted_id = $wpdb->insert_id; // ID ของ row ที่เพิ่งเพิ่ม

// ==========================================
// UPDATE - แก้ไขข้อมูล
// ==========================================

$result = $wpdb->update(
    $wpdb->postmeta,
    array( 'meta_value' => 'new_value' ),  // ข้อมูลที่อัปเดต
    array( 
        'post_id'  => 42,
        'meta_key' => '_my_custom_field'
    ),                                       // WHERE clause
    array( '%s' ),                           // Format ของข้อมูลที่อัปเดต
    array( '%d', '%s' )                      // Format ของ WHERE
);

// ==========================================
// DELETE - ลบข้อมูล
// ==========================================

$result = $wpdb->delete(
    $wpdb->postmeta,
    array( 
        'post_id'  => 42,
        'meta_key' => '_my_custom_field'
    ),
    array( '%d', '%s' )
);

// ==========================================
// Prepared Statements - ป้องกัน SQL Injection
// ==========================================

// ใช้ prepare() เสมอเมื่อรับข้อมูลจาก User
$user_input = $_GET['search'] ?? '';

$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT * FROM {$wpdb->posts} 
         WHERE post_title LIKE %s 
         AND post_type = %s 
         AND post_status = %s",
        '%' . $wpdb->esc_like( $user_input ) . '%',
        'post',
        'publish'
    )
);

// ==========================================
// Direct Query - สำหรับ Complex Queries
// ==========================================

$wpdb->query(
    $wpdb->prepare(
        "UPDATE {$wpdb->posts} SET post_status = %s WHERE ID = %d",
        'trash',
        42
    )
);

// ตรวจสอบ Error
if ( $wpdb->last_error ) {
    error_log( 'DB Error: ' . $wpdb->last_error );
}

// แสดง Query ล่าสุด (เมื่อ Debug)
if ( WP_DEBUG ) {
    error_log( 'Last Query: ' . $wpdb->last_query );
}
```

---

## 5. WordPress Loading Process

```php
<?php
/**
 * WordPress Bootstrap Process
 * 
 * เข้าใจลำดับการโหลดของ WordPress จะช่วยให้ Debug ได้ง่ายขึ้น
 */

// 1. index.php - Entry Point
//    โหลด wp-blog-header.php

// 2. wp-blog-header.php
//    โหลด wp-load.php -> wp-config.php -> wp-settings.php

// 3. wp-settings.php (กระบวนการหลัก)
//    a. กำหนด Constants ต่างๆ
//    b. โหลด wp-includes/functions.php
//    c. เริ่มต้น Database ($wpdb)
//    d. โหลด Must-Use Plugins (mu-plugins)
//    e. โหลด Active Plugins
//    f. โหลด Active Theme (functions.php)
//    g. Fire 'init' action
//    h. Fire 'wp_loaded' action

// 4. การ Handle Request
//    a. Parse Request URL
//    b. Query Database
//    c. Load Template

// ==========================================
// WordPress Action/Filter Hooks Timeline
// ==========================================

// plugins_loaded - หลัง Plugins โหลดแล้ว
add_action( 'plugins_loaded', function() {
    // โหลด Text Domain สำหรับ Translation
    load_plugin_textdomain( 'my-plugin', false, dirname( plugin_basename( __FILE__ ) ) . '/languages' );
});

// init - หลัง WordPress โหลดเสร็จ
add_action( 'init', function() {
    // Register Custom Post Types, Taxonomies
    // เพิ่ม Rewrite Rules
});

// wp_loaded - หลัง WordPress และ Plugins โหลดทั้งหมด
add_action( 'wp_loaded', function() {
    // ทุกอย่างพร้อมแล้ว
});

// wp - หลัง Query ถูก setup
add_action( 'wp', function() {
    // เข้าถึง $wp_query ได้แล้ว
});

// template_redirect - ก่อน Template Load
add_action( 'template_redirect', function() {
    // Redirect หรือ Modify Template ได้
});

// wp_head - ใน <head> ของ HTML
add_action( 'wp_head', function() {
    echo '<meta name="custom" content="value">';
});

// wp_footer - ก่อน </body>
add_action( 'wp_footer', function() {
    echo '<script>console.log("WordPress loaded!");</script>';
});
```

---

## 6. WordPress Options API

```php
<?php
/**
 * WordPress Options API
 * 
 * ใช้เก็บ Settings และ Configuration
 */

// ==========================================
// get_option / update_option
// ==========================================

// เพิ่ม Option ใหม่
add_option( 'my_plugin_settings', array(
    'api_key'     => '',
    'debug_mode'  => false,
    'items_count' => 10,
) );

// ดึงค่า Option
$settings = get_option( 'my_plugin_settings', array() ); // array() = default value
$api_key = $settings['api_key'] ?? '';

// อัปเดต Option (สร้างใหม่ถ้ายังไม่มี)
update_option( 'my_plugin_settings', array(
    'api_key'     => 'new_api_key_here',
    'debug_mode'  => true,
    'items_count' => 20,
) );

// ลบ Option
delete_option( 'my_plugin_settings' );

// ==========================================
// Transients - เก็บข้อมูลชั่วคราว (Caching)
// ==========================================

// เก็บ Cache เป็นเวลา 1 ชั่วโมง
$cache_key = 'my_api_data';
$cached_data = get_transient( $cache_key );

if ( false === $cached_data ) {
    // ข้อมูลหมดอายุหรือไม่มี - ดึงใหม่
    $cached_data = fetch_data_from_api();
    
    // เก็บไว้ 1 ชั่วโมง (3600 วินาที)
    set_transient( $cache_key, $cached_data, HOUR_IN_SECONDS );
}

// ลบ Transient
delete_transient( $cache_key );

// WordPress Time Constants
// MINUTE_IN_SECONDS = 60
// HOUR_IN_SECONDS   = 3600
// DAY_IN_SECONDS    = 86400
// WEEK_IN_SECONDS   = 604800
// MONTH_IN_SECONDS  = 2592000
// YEAR_IN_SECONDS   = 31536000
```

---

## 7. Environment Configuration

### 7.1 การใช้ .env File กับ WordPress

```php
<?php
// wp-config.php - ดึงค่าจาก Environment Variables

// วิธีใช้ phpdotenv (Composer)
// composer require vlucas/phpdotenv

require_once __DIR__ . '/vendor/autoload.php';

$dotenv = Dotenv\Dotenv::createImmutable( __DIR__ );
$dotenv->load();

// ดึงค่าจาก .env
define( 'DB_NAME',     $_ENV['DB_NAME'] );
define( 'DB_USER',     $_ENV['DB_USER'] );
define( 'DB_PASSWORD', $_ENV['DB_PASSWORD'] );
define( 'DB_HOST',     $_ENV['DB_HOST'] );

// ตัวอย่าง .env file
/*
DB_NAME=wordpress_db
DB_USER=wp_user
DB_PASSWORD=secure_password
DB_HOST=localhost

WP_ENV=development
WP_HOME=http://localhost
WP_SITEURL=${WP_HOME}/wp

AUTH_KEY=your-auth-key
SECURE_AUTH_KEY=your-secure-auth-key
LOGGED_IN_KEY=your-logged-in-key
NONCE_KEY=your-nonce-key
*/
```

### 7.2 WordPress Bedrock Structure (Modern WordPress)

```
bedrock/
├── composer.json            # Composer Dependencies
├── composer.lock
├── .env                     # Environment Variables
├── .env.example
├── config/
│   ├── application.php      # Application Config (แทน wp-config.php)
│   └── environments/
│       ├── development.php  # Dev Config
│       ├── staging.php
│       └── production.php
├── vendor/                  # Composer packages
└── web/                     # Web Root
    ├── app/                 # แทน wp-content/
    │   ├── mu-plugins/
    │   ├── plugins/
    │   ├── themes/
    │   └── uploads/
    ├── wp/                  # WordPress Core
    └── index.php
```

---

## Workshop: ติดตั้ง WordPress Development Environment

### Exercise 1: Manual Setup

```bash
# 1. สร้างโปรเจกต์ใหม่
mkdir wordpress-project
cd wordpress-project

# 2. ดาวน์โหลด WordPress
wp core download --locale=th

# 3. สร้าง Database
mysql -u root -p -e "CREATE DATABASE wp_dev CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# 4. ตั้งค่า wp-config.php
wp config create \
    --dbname=wp_dev \
    --dbuser=root \
    --dbpass=password \
    --extra-php <<PHP
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);
define('WP_DEBUG_DISPLAY', false);
PHP

# 5. ติดตั้ง WordPress
wp core install \
    --url=http://localhost/wordpress-project \
    --title="My Dev Site" \
    --admin_user=admin \
    --admin_password=Admin123! \
    --admin_email=dev@example.com

# 6. ติดตั้ง Plugin สำหรับ Development
wp plugin install query-monitor --activate

# 7. ตั้งค่า Permalink
wp rewrite structure '/%category%/%postname%/' --hard

echo "Setup complete!"
```

### Exercise 2: สร้าง Custom wp-config.php

สร้างไฟล์ `wp-config.php` ที่มีการตั้งค่าสำหรับ:
1. Development mode พร้อม Debug
2. Memory Limit 256MB
3. จำกัด Post Revisions เป็น 3
4. Disable auto-update Core

```php
<?php
// TODO: เติมโค้ดให้ครบ
define( 'WP_DEBUG', ??? );
define( 'WP_MEMORY_LIMIT', ??? );
define( 'WP_POST_REVISIONS', ??? );
define( 'AUTOMATIC_UPDATER_DISABLED', ??? );
```

**เฉลย:**
```php
<?php
define( 'WP_DEBUG', true );
define( 'WP_MEMORY_LIMIT', '256M' );
define( 'WP_POST_REVISIONS', 3 );
define( 'AUTOMATIC_UPDATER_DISABLED', true );
```

---

## Quiz

**คำถามที่ 1:** ไฟล์ไหนคือ Entry Point หลักของ WordPress?

A) wp-config.php  
B) wp-settings.php  
C) index.php  
D) wp-blog-header.php  

**เฉลย: C) index.php**

---

**คำถามที่ 2:** ตาราง WordPress ไหนที่เก็บ Custom Fields (Meta)?

A) wp_options  
B) wp_postmeta  
C) wp_terms  
D) wp_comments  

**เฉลย: B) wp_postmeta**

---

**คำถามที่ 3:** คำสั่ง WP-CLI ใดที่ใช้เปลี่ยน URL เมื่อ Migrate?

A) `wp option update siteurl`  
B) `wp db migrate`  
C) `wp search-replace`  
D) `wp core update`  

**เฉลย: C) wp search-replace**

---

**คำถามที่ 4:** ทำไมต้องใช้ `$wpdb->prepare()` เมื่อ Query Database?

A) ทำให้ Query เร็วขึ้น  
B) ป้องกัน SQL Injection  
C) ลดการใช้ Memory  
D) Auto-cache ผลลัพธ์  

**เฉลย: B) ป้องกัน SQL Injection**

---

**คำถามที่ 5:** Transients ใน WordPress คืออะไร?

A) ตาราง Database ชนิดใหม่  
B) ฟังก์ชัน Async  
C) ระบบ Cache ชั่วคราวใน Database  
D) Plugin Security  

**เฉลย: C) ระบบ Cache ชั่วคราวใน Database**

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- การติดตั้ง WordPress ทั้งแบบ Manual และ Composer
- การใช้ WP-CLI เพื่อจัดการ WordPress
- โครงสร้างไฟล์และโฟลเดอร์ที่สำคัญ
- โครงสร้าง Database และ Tables ต่างๆ
- การใช้ `$wpdb` อย่างปลอดภัย
- WordPress Loading Process
- Options API และ Transients

---

## ต่อไป

➡️ **[Part 052: WordPress Theme Basics](part-052-wordpress-theme-basics.md)**

เรียนรู้เกี่ยวกับ:
- Template Hierarchy
- functions.php
- Enqueue Scripts/Styles
- WordPress Hooks System
