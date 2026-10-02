# Part 061: WordPress Security & Performance
## ระดับ: มืออาชีพ | ขั้นตอนที่ 600-640

---

## 🎯 เป้าหมายของ Part นี้

เมื่อเรียนจบ Part นี้คุณจะสามารถ:
- Harden WordPress ให้ปลอดภัยระดับ Production
- ใช้ Redis/Memcached สำหรับ Object Caching
- Optimize Database queries และ Assets
- ตั้งค่า Security Headers และ WAF
- Monitor และ Profile WordPress performance

---

## 📖 เนื้อหา

### 1. WordPress Security Hardening

#### wp-config.php Security

```php
<?php
// wp-config.php - Security configurations

// Database credentials (ใช้ environment variables ถ้าทำได้)
define('DB_NAME', getenv('DB_NAME') ?: 'wordpress_db');
define('DB_USER', getenv('DB_USER') ?: 'wp_user');
define('DB_PASSWORD', getenv('DB_PASSWORD') ?: '');
define('DB_HOST', getenv('DB_HOST') ?: 'localhost');
define('DB_CHARSET', 'utf8mb4');
define('DB_COLLATE', 'utf8mb4_unicode_ci');

// Security Keys (สร้างจาก https://api.wordpress.org/secret-key/1.1/salt/)
define('AUTH_KEY',         'your-unique-phrase-here');
define('SECURE_AUTH_KEY',  'your-unique-phrase-here');
define('LOGGED_IN_KEY',    'your-unique-phrase-here');
define('NONCE_KEY',        'your-unique-phrase-here');
define('AUTH_SALT',        'your-unique-phrase-here');
define('SECURE_AUTH_SALT', 'your-unique-phrase-here');
define('LOGGED_IN_SALT',   'your-unique-phrase-here');
define('NONCE_SALT',       'your-unique-phrase-here');

// Security Settings
define('DISALLOW_FILE_EDIT', true);      // ปิด Theme/Plugin editor
define('DISALLOW_FILE_MODS', false);     // ปิดการติดตั้ง plugin/theme (Production)
define('FORCE_SSL_ADMIN', true);         // บังคับ HTTPS สำหรับ Admin
define('WP_AUTO_UPDATE_CORE', true);     // auto-update core

// Debug (ปิดใน Production)
define('WP_DEBUG', false);
define('WP_DEBUG_LOG', false);
define('WP_DEBUG_DISPLAY', false);
define('SCRIPT_DEBUG', false);

// Limit Post Revisions
define('WP_POST_REVISIONS', 5);

// Trash ล้างอัตโนมัติใน 7 วัน
define('EMPTY_TRASH_DAYS', 7);

// Memory Limit
define('WP_MEMORY_LIMIT', '256M');
define('WP_MAX_MEMORY_LIMIT', '512M');

// ย้าย wp-content directory
// define('WP_CONTENT_DIR', dirname(__FILE__) . '/content');
// define('WP_CONTENT_URL', 'https://example.com/content');

$table_prefix = 'wp_sec_'; // เปลี่ยน prefix จาก wp_ เพื่อป้องกัน SQL injection
```

#### .htaccess Security Rules

```apache
# .htaccess - Apache Security

# ป้องกัน wp-config.php
<files wp-config.php>
order allow,deny
deny from all
</files>

# ป้องกัน .htaccess ตัวเอง
<Files .htaccess>
order allow,deny
deny from all
</Files>

# ป้องกัน xmlrpc.php (ถ้าไม่ใช้)
<Files xmlrpc.php>
order deny,allow
deny from all
</Files>

# ป้องกัน wp-login.php Brute Force - จำกัด IP
<Files wp-login.php>
order deny,allow
deny from all
allow from 192.168.1.0/24   # IP Office
allow from 203.0.113.0/24   # IP บ้าน
</Files>

# ป้องกัน Directory Listing
Options -Indexes

# ป้องกัน Access ไปยัง sensitive files
<FilesMatch "(^#.*#|\.(bak|conf|dist|fla|in[ci]|log|orig|psd|sh|sql|sw[op])|~)$">
order allow,deny
deny from all
satisfy all
</FilesMatch>

# Security Headers
<IfModule mod_headers.c>
    Header set X-XSS-Protection "1; mode=block"
    Header set X-Frame-Options "SAMEORIGIN"
    Header set X-Content-Type-Options "nosniff"
    Header set Referrer-Policy "strict-origin-when-cross-origin"
    Header set Permissions-Policy "geolocation=(), microphone=(), camera=()"
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
    Header set Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' *.googleapis.com *.gstatic.com; style-src 'self' 'unsafe-inline' *.googleapis.com; img-src 'self' data: *.gravatar.com; font-src 'self' *.gstatic.com; frame-ancestors 'self';"
</IfModule>

# Block Bad Bots
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteCond %{HTTP_USER_AGENT} (havij|libwww-perl|wget|python|nikto|curl|scan|java|winhttp|clshttp|loader) [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} (%0A|%0D|%27|%3C|%3E|%00) [NC]
    RewriteRule .* - [F,L]
</IfModule>

# Prevent image hotlinking
<IfModule mod_rewrite.c>
    RewriteEngine on
    RewriteCond %{HTTP_REFERER} !^$
    RewriteCond %{HTTP_REFERER} !^https://(www\.)?yourdomain.com [NC]
    RewriteRule \.(gif|jpg|jpeg|png|webp)$ - [NC,F,L]
</IfModule>
```

---

### 2. WordPress Login Security

```php
<?php
// functions.php หรือ mu-plugin

/**
 * เปลี่ยน URL ของ wp-login.php
 * ต้องใช้ plugin เช่น WPS Hide Login หรือ code นี้
 */
add_action('init', function() {
    // Redirect จาก /wp-login.php ไปหน้าอื่น
    if (strpos($_SERVER['REQUEST_URI'], 'wp-login.php') !== false
        && !isset($_GET['action'])
        && $_SERVER['REQUEST_METHOD'] !== 'POST') {
        wp_redirect(home_url('/404'));
        exit;
    }
});

/**
 * จำกัดจำนวนครั้งที่ login ผิด
 */
class LoginLimiter {
    private static $transient_prefix = 'login_fail_';
    private static $max_attempts = 5;
    private static $lockout_time = 900; // 15 นาที

    public static function init(): void {
        add_action('wp_login_failed', [self::class, 'onLoginFailed']);
        add_filter('authenticate', [self::class, 'checkLockout'], 30, 3);
    }

    public static function onLoginFailed(string $username): void {
        $ip = self::getClientIP();
        $key = self::$transient_prefix . md5($ip);
        $attempts = (int) get_transient($key);
        set_transient($key, $attempts + 1, self::$lockout_time);
    }

    public static function checkLockout($user, string $username, string $password) {
        $ip = self::getClientIP();
        $key = self::$transient_prefix . md5($ip);
        $attempts = (int) get_transient($key);

        if ($attempts >= self::$max_attempts) {
            return new WP_Error(
                'too_many_retries',
                sprintf(
                    'IP ของคุณถูก block ชั่วคราว กรุณารอ %d นาที',
                    self::$lockout_time / 60
                )
            );
        }

        return $user;
    }

    private static function getClientIP(): string {
        $ip_headers = ['HTTP_CF_CONNECTING_IP', 'HTTP_X_FORWARDED_FOR', 'REMOTE_ADDR'];
        foreach ($ip_headers as $header) {
            if (!empty($_SERVER[$header])) {
                return filter_var($_SERVER[$header], FILTER_VALIDATE_IP) ?: '0.0.0.0';
            }
        }
        return '0.0.0.0';
    }
}

LoginLimiter::init();

/**
 * ซ่อน WordPress version
 */
remove_action('wp_head', 'wp_generator');
add_filter('the_generator', '__return_empty_string');

/**
 * ลบข้อมูล version จาก scripts/styles
 */
add_filter('style_loader_src', function(string $src): string {
    return strpos($src, '?ver=') ? remove_query_arg('ver', $src) : $src;
});

add_filter('script_loader_src', function(string $src): string {
    return strpos($src, '?ver=') ? remove_query_arg('ver', $src) : $src;
});

/**
 * ปิด REST API สำหรับ unauthenticated users (ถ้าไม่ต้องการ public API)
 */
add_filter('rest_authentication_errors', function($result) {
    if (!empty($result)) {
        return $result;
    }
    if (!is_user_logged_in()) {
        return new WP_Error('rest_not_logged_in', 'REST API requires authentication.', ['status' => 401]);
    }
    return $result;
});
```

---

### 3. Object Caching ด้วย Redis

#### ติดตั้ง Redis Object Cache

```bash
# ติดตั้ง Redis server
sudo apt install redis-server -y
sudo systemctl start redis
sudo systemctl enable redis

# ติดตั้ง PHP Redis extension
sudo apt install php-redis -y

# ทดสอบ Redis
redis-cli ping  # PONG
```

#### wp-config.php สำหรับ Redis

```php
// Redis Object Cache configuration
define('WP_REDIS_HOST', '127.0.0.1');
define('WP_REDIS_PORT', 6379);
define('WP_REDIS_DATABASE', 0);
define('WP_REDIS_PREFIX', 'wp_' . DB_NAME . '_'); // แยก cache ต่าง site
define('WP_REDIS_MAXTTL', 86400); // 24 ชั่วโมง
define('WP_REDIS_TIMEOUT', 1);
define('WP_REDIS_READ_TIMEOUT', 1);
// define('WP_REDIS_PASSWORD', 'your-redis-password');
```

#### Custom Object Cache Implementation

```php
<?php
// wp-content/object-cache.php

/**
 * WordPress Object Cache ด้วย Redis
 * ไฟล์นี้จะถูก WordPress โหลดอัตโนมัติ
 */

class WP_Object_Cache {
    private Redis $redis;
    private array $local_cache = [];
    private string $prefix;
    private bool $connected = false;
    
    public function __construct() {
        $this->prefix = defined('WP_REDIS_PREFIX') ? WP_REDIS_PREFIX : 'wp_';
        $this->connect();
    }
    
    private function connect(): void {
        try {
            $this->redis = new Redis();
            $host = defined('WP_REDIS_HOST') ? WP_REDIS_HOST : '127.0.0.1';
            $port = defined('WP_REDIS_PORT') ? WP_REDIS_PORT : 6379;
            $timeout = defined('WP_REDIS_TIMEOUT') ? WP_REDIS_TIMEOUT : 1;
            
            $this->redis->connect($host, $port, $timeout);
            
            if (defined('WP_REDIS_PASSWORD') && WP_REDIS_PASSWORD) {
                $this->redis->auth(WP_REDIS_PASSWORD);
            }
            
            if (defined('WP_REDIS_DATABASE')) {
                $this->redis->select(WP_REDIS_DATABASE);
            }
            
            $this->connected = true;
        } catch (Exception $e) {
            $this->connected = false;
            error_log('Redis connection failed: ' . $e->getMessage());
        }
    }
    
    public function add(string $key, mixed $data, string $group = 'default', int $expire = 0): bool {
        if ($this->get($key, $group) !== false) {
            return false;
        }
        return $this->set($key, $data, $group, $expire);
    }
    
    public function set(string $key, mixed $data, string $group = 'default', int $expire = 0): bool {
        $cache_key = $this->buildKey($key, $group);
        $this->local_cache[$cache_key] = $data;
        
        if (!$this->connected) {
            return true;
        }
        
        $serialized = serialize($data);
        
        if ($expire > 0) {
            return (bool) $this->redis->setex($cache_key, $expire, $serialized);
        }
        
        return (bool) $this->redis->set($cache_key, $serialized);
    }
    
    public function get(string $key, string $group = 'default', bool $force = false, &$found = null): mixed {
        $cache_key = $this->buildKey($key, $group);
        
        // ตรวจ local cache ก่อน
        if (!$force && array_key_exists($cache_key, $this->local_cache)) {
            $found = true;
            return $this->local_cache[$cache_key];
        }
        
        if (!$this->connected) {
            $found = false;
            return false;
        }
        
        $value = $this->redis->get($cache_key);
        
        if ($value === false) {
            $found = false;
            return false;
        }
        
        $found = true;
        $data = unserialize($value);
        $this->local_cache[$cache_key] = $data;
        
        return $data;
    }
    
    public function delete(string $key, string $group = 'default'): bool {
        $cache_key = $this->buildKey($key, $group);
        unset($this->local_cache[$cache_key]);
        
        if (!$this->connected) {
            return true;
        }
        
        return (bool) $this->redis->del($cache_key);
    }
    
    public function flush(): bool {
        $this->local_cache = [];
        
        if (!$this->connected) {
            return true;
        }
        
        // ลบเฉพาะ keys ที่มี prefix ของ site นี้
        $keys = $this->redis->keys($this->prefix . '*');
        if (!empty($keys)) {
            $this->redis->del($keys);
        }
        
        return true;
    }
    
    private function buildKey(string $key, string $group): string {
        return $this->prefix . $group . ':' . $key;
    }
    
    public function stats(): array {
        if (!$this->connected) {
            return ['status' => 'disconnected'];
        }
        
        return $this->redis->info();
    }
}

// WordPress Object Cache API
global $wp_object_cache;

function wp_cache_init(): void {
    global $wp_object_cache;
    $wp_object_cache = new WP_Object_Cache();
}

function wp_cache_add(string $key, mixed $data, string $group = '', int $expire = 0): bool {
    global $wp_object_cache;
    return $wp_object_cache->add($key, $data, $group ?: 'default', $expire);
}

function wp_cache_set(string $key, mixed $data, string $group = '', int $expire = 0): bool {
    global $wp_object_cache;
    return $wp_object_cache->set($key, $data, $group ?: 'default', $expire);
}

function wp_cache_get(string $key, string $group = '', bool $force = false, &$found = null): mixed {
    global $wp_object_cache;
    return $wp_object_cache->get($key, $group ?: 'default', $force, $found);
}

function wp_cache_delete(string $key, string $group = ''): bool {
    global $wp_object_cache;
    return $wp_object_cache->delete($key, $group ?: 'default');
}

function wp_cache_flush(): bool {
    global $wp_object_cache;
    return $wp_object_cache->flush();
}
```

---

### 4. WordPress Performance Optimization

#### Database Query Optimization

```php
<?php
// functions.php

/**
 * ลด Autoloaded Options ที่ไม่จำเป็น
 */
function cleanup_autoloaded_options(): void {
    global $wpdb;
    
    // ดู options ที่ใหญ่ที่สุด
    $large_options = $wpdb->get_results("
        SELECT option_name, length(option_value) as size
        FROM {$wpdb->options}
        WHERE autoload = 'yes'
        ORDER BY size DESC
        LIMIT 20
    ");
    
    // ตัวอย่าง: ปิด autoload สำหรับ options ที่ไม่ต้องการ
    $non_autoload = [
        'recently_activated',
        '_site_transient_update_plugins',
        '_site_transient_update_themes',
    ];
    
    foreach ($non_autoload as $option) {
        $wpdb->update(
            $wpdb->options,
            ['autoload' => 'no'],
            ['option_name' => $option]
        );
    }
}

/**
 * ลด Query ที่ไม่จำเป็น
 */
// ปิด Heartbeat API (ลด AJAX requests)
add_action('init', function() {
    if (!is_admin()) {
        wp_deregister_script('heartbeat');
    }
});

// ปิด Emoji Scripts
remove_action('wp_head', 'print_emoji_detection_script', 7);
remove_action('wp_print_styles', 'print_emoji_styles');
remove_action('admin_print_scripts', 'print_emoji_detection_script');
remove_action('admin_print_styles', 'print_emoji_styles');

// ปิด Embeds
remove_action('wp_head', 'wp_oembed_add_discovery_links');
remove_action('wp_head', 'wp_oembed_add_host_js');

// ปิด wlwmanifest
remove_action('wp_head', 'wlwmanifest_link');

// ปิด RSD link
remove_action('wp_head', 'rsd_link');

// ปิด shortlink
remove_action('wp_head', 'wp_shortlink_wp_head');

/**
 * Lazy load images
 */
add_filter('the_content', function(string $content): string {
    if (is_feed()) return $content;
    return preg_replace('/<img(.*?)src=/i', '<img$1loading="lazy" src=', $content);
});

/**
 * Dequeue unused scripts/styles
 */
add_action('wp_enqueue_scripts', function() {
    // ลบ Contact Form 7 scripts ถ้าไม่มี form
    if (!is_page([10, 15])) { // page IDs ที่มี form
        wp_dequeue_script('contact-form-7');
        wp_dequeue_style('contact-form-7');
    }
    
    // ลบ WooCommerce scripts บนหน้าที่ไม่ใช่ shop
    if (function_exists('is_woocommerce') && !is_woocommerce() && !is_cart() && !is_checkout()) {
        wp_dequeue_style('woocommerce-general');
        wp_dequeue_style('woocommerce-smallscreen');
        wp_dequeue_style('woocommerce-layout');
    }
}, 99);

/**
 * Optimize WP_Query - ใช้ fields เพื่อดึงเฉพาะที่ต้องการ
 */
function get_recent_post_ids(int $limit = 5): array {
    $query = new WP_Query([
        'post_type'      => 'post',
        'post_status'    => 'publish',
        'posts_per_page' => $limit,
        'fields'         => 'ids', // ดึงแค่ IDs ไม่ใช่ post objects ทั้งหมด
        'no_found_rows'  => true,  // ไม่คำนวณ pagination
        'update_post_meta_cache' => false, // ไม่โหลด post meta
        'update_post_term_cache' => false, // ไม่โหลด terms
    ]);
    
    return $query->posts;
}
```

#### Asset Optimization

```php
<?php
// functions.php

/**
 * Critical CSS - Inline CSS ที่จำเป็น
 */
add_action('wp_head', function() {
    $critical_css_file = get_template_directory() . '/assets/css/critical.css';
    if (file_exists($critical_css_file)) {
        echo '<style id="critical-css">';
        echo file_get_contents($critical_css_file);
        echo '</style>';
    }
}, 1);

/**
 * Defer non-critical CSS
 */
add_filter('style_loader_tag', function(string $html, string $handle): string {
    $defer_styles = ['google-fonts', 'font-awesome', 'non-critical-style'];
    
    if (in_array($handle, $defer_styles)) {
        // ใช้ preload pattern สำหรับ CSS
        $html = str_replace(
            "rel='stylesheet'",
            "rel='preload' as='style' onload=\"this.onload=null;this.rel='stylesheet'\"",
            $html
        );
        $html .= "<noscript>" . str_replace("rel='preload' as='style' onload=\"this.onload=null;this.rel='stylesheet'\"", "rel='stylesheet'", $html) . "</noscript>";
    }
    
    return $html;
}, 10, 2);

/**
 * Preload key resources
 */
add_action('wp_head', function() {
    $font_url = get_template_directory_uri() . '/assets/fonts/primary-font.woff2';
    echo "<link rel='preload' href='{$font_url}' as='font' type='font/woff2' crossorigin='anonymous'>\n";
}, 1);
```

---

### 5. Caching Layers

#### Full Page Cache

```php
<?php
// mu-plugins/page-cache.php

/**
 * Simple Full Page Cache
 * สำหรับ Production ควรใช้ Nginx FastCGI Cache หรือ Varnish แทน
 */
class SimplePageCache {
    private string $cache_dir;
    private int $cache_ttl;
    
    public function __construct(string $cache_dir = WP_CONTENT_DIR . '/page-cache', int $ttl = 3600) {
        $this->cache_dir = $cache_dir;
        $this->cache_ttl = $ttl;
        
        if (!is_dir($this->cache_dir)) {
            mkdir($this->cache_dir, 0755, true);
        }
    }
    
    public function serve(): void {
        if (!$this->isCacheable()) {
            return;
        }
        
        $cache_file = $this->getCacheFilePath();
        
        if ($this->isCacheValid($cache_file)) {
            $this->sendCacheHeaders();
            readfile($cache_file);
            exit;
        }
        
        ob_start([$this, 'savePage']);
    }
    
    public function savePage(string $buffer): string {
        if (strlen($buffer) > 0 && !is_404()) {
            $cache_file = $this->getCacheFilePath();
            $dir = dirname($cache_file);
            
            if (!is_dir($dir)) {
                mkdir($dir, 0755, true);
            }
            
            file_put_contents($cache_file, $buffer);
        }
        
        return $buffer;
    }
    
    public function clear(int $post_id = 0): void {
        if ($post_id > 0) {
            // Clear cache เฉพาะ URL ของ post นั้น
            $url = get_permalink($post_id);
            $cache_file = $this->getCacheFilePath($url);
            if (file_exists($cache_file)) {
                unlink($cache_file);
            }
        } else {
            // Clear cache ทั้งหมด
            $this->deleteDirectory($this->cache_dir);
            mkdir($this->cache_dir, 0755, true);
        }
    }
    
    private function isCacheable(): bool {
        // ไม่ cache ถ้า:
        if (is_user_logged_in()) return false;
        if (isset($_COOKIE['wordpress_logged_in_' . COOKIEHASH])) return false;
        if ($_SERVER['REQUEST_METHOD'] !== 'GET') return false;
        if (!empty($_GET)) return false;
        if (is_admin()) return false;
        
        return true;
    }
    
    private function getCacheFilePath(?string $url = null): string {
        $url = $url ?? (isset($_SERVER['HTTPS']) ? 'https' : 'http') . '://' . $_SERVER['HTTP_HOST'] . $_SERVER['REQUEST_URI'];
        $path = parse_url($url, PHP_URL_PATH);
        $hash = md5($url);
        
        return $this->cache_dir . $path . $hash . '.html';
    }
    
    private function isCacheValid(string $cache_file): bool {
        return file_exists($cache_file) && (time() - filemtime($cache_file) < $this->cache_ttl);
    }
    
    private function sendCacheHeaders(): void {
        header('X-Cache: HIT');
        header('Cache-Control: public, max-age=' . $this->cache_ttl);
    }
    
    private function deleteDirectory(string $dir): void {
        if (!is_dir($dir)) return;
        
        $files = array_diff(scandir($dir), ['.', '..']);
        foreach ($files as $file) {
            $path = $dir . '/' . $file;
            is_dir($path) ? $this->deleteDirectory($path) : unlink($path);
        }
        
        rmdir($dir);
    }
}

// เริ่มต้น cache
$page_cache = new SimplePageCache();
$page_cache->serve();

// Clear cache เมื่อ post ถูกอัปเดต
add_action('save_post', function(int $post_id) use ($page_cache) {
    $page_cache->clear($post_id);
});

add_action('comment_post', function() use ($page_cache) {
    $page_cache->clear();
});
```

---

### 6. Nginx Configuration สำหรับ WordPress

```nginx
# /etc/nginx/sites-available/wordpress

server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;
    root /var/www/wordpress;
    index index.php;

    # SSL
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml application/xml+rss text/javascript
               image/svg+xml application/font-woff2;

    # Browser Caching
    location ~* \.(jpg|jpeg|png|gif|ico|svg|webp|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    location ~* \.(css|js)$ {
        expires 1M;
        add_header Cache-Control "public";
        access_log off;
    }

    # FastCGI Cache
    fastcgi_cache_path /var/cache/nginx/wordpress
        levels=1:2
        keys_zone=WORDPRESS:100m
        inactive=60m
        max_size=1g;
    fastcgi_cache_key "$scheme$request_method$host$request_uri";

    # WordPress
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # PHP-FPM
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        
        # FastCGI Cache
        fastcgi_cache WORDPRESS;
        fastcgi_cache_valid 200 301 302 60m;
        fastcgi_cache_use_stale error timeout updating;
        fastcgi_cache_bypass $no_cache;
        fastcgi_no_cache $no_cache;
        
        set $no_cache 0;
        if ($http_cookie ~* "wordpress_logged_in|wp-postpass|woocommerce_cart_hash") {
            set $no_cache 1;
        }
        if ($request_method = POST) {
            set $no_cache 1;
        }
        
        add_header X-FastCGI-Cache $upstream_cache_status;
    }

    # Block access to sensitive files
    location ~* /(?:wp-config\.php|xmlrpc\.php|readme\.html|license\.txt)$ {
        deny all;
        return 404;
    }

    location ~ /\.ht {
        deny all;
    }

    # ป้องกัน PHP execution ใน uploads
    location /wp-content/uploads/ {
        location ~ \.php$ {
            deny all;
        }
    }
}
```

---

### 7. WordPress Performance Monitoring

```php
<?php
// mu-plugins/performance-monitor.php

/**
 * Performance Monitor สำหรับ Development
 */
if (defined('WP_DEBUG') && WP_DEBUG) {
    
    class WP_Performance_Monitor {
        private float $start_time;
        private int $start_memory;
        
        public function __construct() {
            $this->start_time = microtime(true);
            $this->start_memory = memory_get_usage();
            
            add_action('shutdown', [$this, 'report'], 999);
            add_filter('query', [$this, 'logQuery']);
        }
        
        private array $queries = [];
        
        public function logQuery(string $query): string {
            $this->queries[] = $query;
            return $query;
        }
        
        public function report(): void {
            $execution_time = microtime(true) - $this->start_time;
            $memory_used = memory_get_peak_usage(true) - $this->start_memory;
            $query_count = count($this->queries);
            
            $report = [
                'url'           => $_SERVER['REQUEST_URI'],
                'execution_ms'  => round($execution_time * 1000, 2),
                'memory_mb'     => round($memory_used / 1024 / 1024, 2),
                'query_count'   => $query_count,
                'timestamp'     => date('Y-m-d H:i:s'),
            ];
            
            // Log เฉพาะถ้าช้าเกิน 500ms
            if ($execution_time > 0.5) {
                error_log('SLOW PAGE: ' . json_encode($report));
                
                // Log slow queries
                foreach ($this->queries as $q) {
                    if (strlen($q) > 200) {
                        error_log('LONG QUERY: ' . substr($q, 0, 500));
                    }
                }
            }
        }
    }
    
    new WP_Performance_Monitor();
}
```

---

## 🛠️ Workshop: Security Audit

```bash
#!/bin/bash
# wp-security-audit.sh - ตรวจสอบความปลอดภัย WordPress

WP_PATH="/var/www/wordpress"
REPORT_FILE="/tmp/wp-security-report-$(date +%Y%m%d).txt"

echo "WordPress Security Audit - $(date)" > $REPORT_FILE
echo "=================================" >> $REPORT_FILE

# 1. ตรวจสอบ file permissions
echo -e "\n[1] File Permissions:" >> $REPORT_FILE
find $WP_PATH -name "*.php" -perm /o+w 2>/dev/null | head -20 >> $REPORT_FILE

# 2. ตรวจสอบ wp-config.php permissions
echo -e "\n[2] wp-config.php permissions:" >> $REPORT_FILE
ls -la $WP_PATH/wp-config.php >> $REPORT_FILE

# 3. ตรวจสอบ PHP version
echo -e "\n[3] PHP Version:" >> $REPORT_FILE
php -v >> $REPORT_FILE

# 4. ตรวจสอบ WordPress core files
echo -e "\n[4] WordPress Core Integrity:" >> $REPORT_FILE
wp core verify-checksums --path=$WP_PATH 2>&1 >> $REPORT_FILE

# 5. ตรวจสอบ plugin vulnerabilities
echo -e "\n[5] Plugin Security Check:" >> $REPORT_FILE
wp plugin list --path=$WP_PATH --format=table >> $REPORT_FILE

echo "Audit complete: $REPORT_FILE"
cat $REPORT_FILE
```

---

## 📝 Quiz

1. ควรตั้งค่า `DISALLOW_FILE_EDIT` เป็น `true` ทำไม?
2. `X-Frame-Options: SAMEORIGIN` ป้องกันการโจมตีประเภทใด?
3. Redis Object Cache ต่างจาก Transient API อย่างไร?
4. FastCGI Cache ควร bypass เมื่อใด?
5. ทำไมถึงควรเปลี่ยน `$table_prefix` จาก `wp_`?

**เฉลย:**
1. ป้องกันไม่ให้แก้ไข theme/plugin ผ่าน WordPress Admin (ลด attack surface)
2. Clickjacking - ป้องกันไม่ให้ embed site ใน iframe ของ site อื่น
3. Redis เก็บใน RAM ตลอดเวลา เร็วกว่า; Transient เก็บใน MySQL Options table
4. เมื่อ user logged in, POST request, หรือมี session cookies
5. SQL Injection attacks มักใช้ `wp_` prefix เดา table names

---

## ⏭️ Part ถัดไป

**Part 062: WordPress Headless CMS**

---

*Part 061 | ระดับมืออาชีพ | WordPress Security & Performance*
