# Part 060: WordPress Advanced - Multisite, Caching, Security & Performance

**ระดับ:** Expert  
**เวลาเรียน:** 6-8 ชั่วโมง  
**Prerequisites:** Part 051-059

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. ตั้งค่าและจัดการ WordPress Multisite
2. Implement Caching Strategies
3. Harden WordPress Security
4. Optimize Performance
5. ใช้ Object Cache และ Page Cache

---

## 1. WordPress Multisite

```php
<?php
/**
 * WordPress Multisite (Network)
 */

// ==========================================
// การเปิดใช้งาน Multisite
// ==========================================

// 1. เพิ่มใน wp-config.php (ก่อน require wp-settings.php)
/*
define('WP_ALLOW_MULTISITE', true);
*/

// 2. ไปที่ Tools > Network Setup และตั้งค่า

// 3. เพิ่มใน wp-config.php หลัง Setup
/*
define('MULTISITE', true);
define('SUBDOMAIN_INSTALL', false); // false = subdirectory, true = subdomain
define('DOMAIN_CURRENT_SITE', 'example.com');
define('PATH_CURRENT_SITE', '/');
define('SITE_ID_CURRENT_SITE', 1);
define('BLOG_ID_CURRENT_SITE', 1);
*/

// ==========================================
// Multisite Functions
// ==========================================

// ตรวจสอบว่าอยู่ใน Network
if (is_multisite()) {
    
    // ดึงข้อมูล Sites ทั้งหมด
    $sites = get_sites(array(
        'number'     => 100,
        'site__in'   => array(),
        'public'     => 1,
        'archived'   => 0,
        'spam'       => 0,
        'deleted'    => 0,
    ));
    
    foreach ($sites as $site) {
        switch_to_blog($site->blog_id);
        
        echo get_bloginfo('name') . ': ' . get_site_url() . "\n";
        
        // ดึง Posts ของแต่ละ Site
        $posts = get_posts(array(
            'post_type'      => 'post',
            'posts_per_page' => 5,
        ));
        
        restore_current_blog(); // สำคัญ!
    }
    
    // เพิ่ม Site ใหม่
    $new_site_id = wpmu_create_blog(
        'example.com',     // Domain
        '/new-site/',      // Path
        'New Site Name',   // Title
        1,                 // User ID (Admin)
        array(
            'public' => 1,
        ),
        1                  // Network ID
    );
    
    // ลบ Site
    wpmu_delete_blog($new_site_id, true); // true = delete files too
}

// ==========================================
// Network Plugin (mu-plugins)
// ==========================================

// สร้าง Plugin ที่ทำงานทั้ง Network
// File: wp-content/mu-plugins/my-network-plugin.php

/*
<?php
Plugin Name: My Network Plugin
Network: true  // ระบุว่าเป็น Network Plugin
*/

// Network Admin Hooks
add_action('network_admin_menu', function() {
    add_menu_page(
        'Network Settings',
        'Network Settings',
        'manage_network',
        'my-network-settings',
        function() {
            echo '<div class="wrap"><h1>Network Settings</h1></div>';
        }
    );
});

// กำหนด Default Options สำหรับ Site ใหม่
add_action('wpmu_new_blog', function($blog_id, $user_id) {
    switch_to_blog($blog_id);
    
    // ติดตั้ง Plugins สำหรับ Site ใหม่อัตโนมัติ
    activate_plugin('woocommerce/woocommerce.php');
    
    // ตั้งค่า Default
    update_option('blogdescription', 'Default tagline');
    update_option('posts_per_page', 12);
    
    restore_current_blog();
}, 10, 2);

// ==========================================
// Cross-Site Queries
// ==========================================

function get_all_network_posts($post_type = 'post', $limit = 10) {
    global $wpdb;
    
    // ดึงข้อมูลจากหลาย Sites โดยตรงใน Database
    $sites = get_sites(array('number' => 100));
    $all_posts = array();
    
    foreach ($sites as $site) {
        $posts_table = $wpdb->get_blog_prefix($site->blog_id) . 'posts';
        
        $posts = $wpdb->get_results($wpdb->prepare(
            "SELECT ID, post_title, post_date, post_type, {$site->blog_id} as site_id 
             FROM {$posts_table} 
             WHERE post_type = %s 
             AND post_status = 'publish'
             ORDER BY post_date DESC 
             LIMIT %d",
            $post_type,
            $limit
        ));
        
        $all_posts = array_merge($all_posts, $posts);
    }
    
    // เรียงตาม Date
    usort($all_posts, function($a, $b) {
        return strtotime($b->post_date) - strtotime($a->post_date);
    });
    
    return array_slice($all_posts, 0, $limit);
}
```

---

## 2. Caching Strategies

```php
<?php
/**
 * WordPress Caching
 */

// ==========================================
// Object Cache (Persistent)
// ==========================================

// WordPress ใช้ WP_Object_Cache โดย default เก็บใน Memory
// เพิ่ม Persistent Cache ด้วย Redis หรือ Memcached

// wp-config.php
/*
define('WP_CACHE', true);
define('WP_REDIS_HOST', '127.0.0.1');
define('WP_REDIS_PORT', 6379);
define('WP_REDIS_TIMEOUT', 1);
define('WP_REDIS_READ_TIMEOUT', 1);
define('WP_REDIS_DATABASE', 0);
*/

// ==========================================
// ใช้งาน WordPress Object Cache
// ==========================================

function get_expensive_data($key) {
    
    $cache_group = 'my_plugin_data';
    $cached = wp_cache_get($key, $cache_group);
    
    if (false !== $cached) {
        return $cached; // Return from cache
    }
    
    // Expensive operation
    $data = fetch_data_from_database($key);
    
    // Store in cache for 1 hour
    wp_cache_set($key, $data, $cache_group, HOUR_IN_SECONDS);
    
    return $data;
}

// ล้าง Cache เมื่อ Data เปลี่ยน
add_action('save_post', function($post_id) {
    wp_cache_delete('post_' . $post_id, 'my_plugin_data');
    wp_cache_delete('all_posts', 'my_plugin_data');
});

// ==========================================
// Transients (Database Cache)
// ==========================================

function get_weather_data($city) {
    $cache_key = 'weather_' . sanitize_title($city);
    $cached    = get_transient($cache_key);
    
    if (false !== $cached) {
        return $cached;
    }
    
    // Fetch from API
    $response = wp_remote_get("https://api.weather.com/v1?city={$city}");
    
    if (is_wp_error($response)) {
        return false;
    }
    
    $data = json_decode(wp_remote_retrieve_body($response), true);
    
    // Cache 30 นาที
    set_transient($cache_key, $data, 30 * MINUTE_IN_SECONDS);
    
    return $data;
}

// ==========================================
// Page Cache Headers
// ==========================================

function set_cache_headers() {
    
    if (is_user_logged_in() || is_admin()) {
        header('Cache-Control: no-cache, no-store, must-revalidate');
        header('Pragma: no-cache');
        return;
    }
    
    if (is_singular() || is_archive() || is_home()) {
        // Cache 1 ชั่วโมง
        header('Cache-Control: public, max-age=3600');
        header('Surrogate-Control: max-age=3600');
        header('Vary: Accept-Encoding, Cookie');
    }
}
add_action('send_headers', 'set_cache_headers');

// ==========================================
// Fragment Caching
// ==========================================

function cached_widget($key, $callback, $ttl = 3600) {
    $cached = wp_cache_get($key, 'fragments');
    
    if (false !== $cached) {
        echo $cached;
        return;
    }
    
    ob_start();
    call_user_func($callback);
    $output = ob_get_clean();
    
    wp_cache_set($key, $output, 'fragments', $ttl);
    echo $output;
}

// Usage:
cached_widget('popular_posts', function() {
    $posts = get_posts(array(
        'posts_per_page' => 5,
        'meta_key'       => '_view_count',
        'orderby'        => 'meta_value_num',
        'order'          => 'DESC',
    ));
    
    echo '<ul class="popular-posts">';
    foreach ($posts as $post) {
        echo '<li><a href="' . get_permalink($post) . '">' . esc_html($post->post_title) . '</a></li>';
    }
    echo '</ul>';
}, HOUR_IN_SECONDS);

// ==========================================
// Query Optimization
// ==========================================

// ป้องกัน Slow Queries
add_filter('posts_clauses', function($clauses, $query) {
    
    // จำกัด Query ที่ไม่จำเป็น
    if (!is_admin() && $query->is_main_query()) {
        
        // ไม่ต้องนับ Total Posts ถ้าไม่ต้องการ Pagination
        if (isset($query->query_vars['no_found_rows'])) {
            $clauses['groupby'] = '';
        }
    }
    
    return $clauses;
    
}, 10, 2);

// ใช้ 'no_found_rows' => true เมื่อไม่ต้องการ Pagination
$posts = new WP_Query(array(
    'post_type'      => 'post',
    'posts_per_page' => 5,
    'no_found_rows'  => true,  // เร็วขึ้นมาก
    'update_post_meta_cache' => false, // ไม่ Cache Post Meta
    'update_post_term_cache' => false, // ไม่ Cache Terms
));
```

---

## 3. Security Hardening

```php
<?php
/**
 * WordPress Security Hardening
 */

// ==========================================
// ซ่อน WordPress Version
// ==========================================

remove_action('wp_head', 'wp_generator');

add_filter('the_generator', '__return_empty_string');

// ซ่อน Version ใน Script/Style
add_filter('style_loader_src', function($src) {
    if (strpos($src, 'ver=')) {
        $src = remove_query_arg('ver', $src);
    }
    return $src;
});

add_filter('script_loader_src', function($src) {
    if (strpos($src, 'ver=')) {
        $src = remove_query_arg('ver', $src);
    }
    return $src;
});

// ==========================================
// ปิด XML-RPC
// ==========================================

add_filter('xmlrpc_enabled', '__return_false');

// ลบ Link จาก Header
remove_action('wp_head', 'rsd_link');
remove_action('wp_head', 'wlwmanifest_link');

// ==========================================
// Login Security
// ==========================================

// เปลี่ยน Login URL
// (ต้องใช้ Plugin เช่น WPS Hide Login)

// จำกัด Login Attempts
class Login_Attempts_Limiter {
    
    private $max_attempts = 5;
    private $lockout_duration = 900; // 15 นาที
    
    public function __construct() {
        add_action('wp_login_failed', array($this, 'record_failed_attempt'));
        add_filter('authenticate', array($this, 'check_lockout'), 30, 3);
    }
    
    public function record_failed_attempt($username) {
        $ip  = $this->get_client_ip();
        $key = 'failed_login_' . md5($ip . $username);
        
        $attempts = (int) get_transient($key);
        $attempts++;
        
        set_transient($key, $attempts, $this->lockout_duration);
        
        // Log ถ้า Suspicious
        if ($attempts >= 3) {
            error_log("Multiple failed logins for {$username} from {$ip}");
        }
    }
    
    public function check_lockout($user, $username, $password) {
        if (empty($username)) return $user;
        
        $ip  = $this->get_client_ip();
        $key = 'failed_login_' . md5($ip . $username);
        
        $attempts = (int) get_transient($key);
        
        if ($attempts >= $this->max_attempts) {
            $ttl     = $this->lockout_duration;
            $minutes = ceil($ttl / 60);
            
            return new WP_Error(
                'too_many_attempts',
                sprintf(
                    'บัญชีถูกล็อค กรุณาลองใหม่ใน %d นาที',
                    $minutes
                )
            );
        }
        
        return $user;
    }
    
    private function get_client_ip() {
        $ip_keys = array(
            'HTTP_X_FORWARDED_FOR',
            'HTTP_X_REAL_IP',
            'HTTP_CLIENT_IP',
            'REMOTE_ADDR',
        );
        
        foreach ($ip_keys as $key) {
            if (!empty($_SERVER[$key])) {
                $ips = explode(',', $_SERVER[$key]);
                return trim($ips[0]);
            }
        }
        
        return '0.0.0.0';
    }
}

new Login_Attempts_Limiter();

// ==========================================
// Nonce Validation
// ==========================================

// สร้าง Nonce
$nonce = wp_create_nonce('my_action');

// ตรวจสอบ Nonce
if (!wp_verify_nonce($_REQUEST['nonce'], 'my_action')) {
    wp_die('Security check failed');
}

// URL Nonce
$url = wp_nonce_url(admin_url('admin.php?action=delete&id=5'), 'delete_item_5');

// Form Nonce
wp_nonce_field('save_settings', 'settings_nonce');

// ==========================================
// Input Sanitization และ Validation
// ==========================================

class Security_Utils {
    
    // Sanitize HTML
    public static function sanitize_html($input) {
        $allowed_tags = wp_kses_allowed_html('post');
        return wp_kses($input, $allowed_tags);
    }
    
    // Sanitize Text
    public static function sanitize_text($input) {
        return sanitize_text_field($input);
    }
    
    // Sanitize Email
    public static function sanitize_email($input) {
        $email = sanitize_email($input);
        if (!is_email($email)) {
            return '';
        }
        return $email;
    }
    
    // Sanitize URL
    public static function sanitize_url($input) {
        return esc_url_raw($input);
    }
    
    // Sanitize Integer
    public static function sanitize_int($input, $min = null, $max = null) {
        $value = intval($input);
        if ($min !== null) $value = max($min, $value);
        if ($max !== null) $value = min($max, $value);
        return $value;
    }
    
    // Validate Thai Phone
    public static function validate_thai_phone($phone) {
        return preg_match('/^(0[689]\d{8}|0[2-7]\d{7,8})$/', $phone);
    }
    
    // Validate Thai ID Card
    public static function validate_thai_id($id) {
        if (!preg_match('/^\d{13}$/', $id)) return false;
        
        $sum = 0;
        for ($i = 0; $i < 12; $i++) {
            $sum += (int)$id[$i] * (13 - $i);
        }
        $checksum = (11 - ($sum % 11)) % 10;
        
        return $checksum == (int)$id[12];
    }
}

// ==========================================
// File Upload Security
// ==========================================

function secure_file_upload($file, $allowed_types = array()) {
    
    // Default allowed types
    if (empty($allowed_types)) {
        $allowed_types = array('jpg', 'jpeg', 'png', 'gif', 'pdf', 'doc', 'docx');
    }
    
    // ตรวจสอบ MIME Type จริง (ไม่ใช่ Extension)
    $finfo = finfo_open(FILEINFO_MIME_TYPE);
    $mime  = finfo_file($finfo, $file['tmp_name']);
    finfo_close($finfo);
    
    $mime_to_ext = array(
        'image/jpeg' => array('jpg', 'jpeg'),
        'image/png'  => array('png'),
        'image/gif'  => array('gif'),
        'application/pdf' => array('pdf'),
    );
    
    $ext = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
    
    // ตรวจสอบว่า MIME Type ตรงกับ Extension
    $valid = false;
    foreach ($mime_to_ext as $type => $exts) {
        if ($mime === $type && in_array($ext, $exts)) {
            $valid = true;
            break;
        }
    }
    
    if (!$valid || !in_array($ext, $allowed_types)) {
        return new WP_Error('invalid_file', 'ประเภทไฟล์ไม่ถูกต้อง');
    }
    
    // ตรวจสอบขนาดไฟล์
    $max_size = 5 * 1024 * 1024; // 5MB
    if ($file['size'] > $max_size) {
        return new WP_Error('file_too_large', 'ไฟล์ใหญ่เกินไป (สูงสุด 5MB)');
    }
    
    // สร้าง Safe Filename
    $filename = sanitize_file_name($file['name']);
    $filename = preg_replace('/[^a-zA-Z0-9._-]/', '', $filename);
    
    // ย้ายไป Upload
    $upload_dir = wp_upload_dir();
    $dest = $upload_dir['path'] . '/' . $filename;
    
    if (!move_uploaded_file($file['tmp_name'], $dest)) {
        return new WP_Error('upload_failed', 'อัปโหลดล้มเหลว');
    }
    
    return $dest;
}

// ==========================================
// Content Security Policy (CSP)
// ==========================================

add_action('send_headers', function() {
    
    if (is_admin()) return;
    
    $csp = implode('; ', array(
        "default-src 'self'",
        "script-src 'self' 'unsafe-inline' https://www.google-analytics.com https://www.googletagmanager.com",
        "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
        "font-src 'self' https://fonts.gstatic.com",
        "img-src 'self' data: https:",
        "connect-src 'self' https://www.google-analytics.com",
        "frame-src 'self' https://www.youtube.com https://player.vimeo.com",
    ));
    
    header("Content-Security-Policy: {$csp}");
    header("X-Content-Type-Options: nosniff");
    header("X-Frame-Options: SAMEORIGIN");
    header("X-XSS-Protection: 1; mode=block");
    header("Referrer-Policy: strict-origin-when-cross-origin");
    header("Permissions-Policy: geolocation=(), microphone=(), camera=()");
    
    if (is_ssl()) {
        header("Strict-Transport-Security: max-age=31536000; includeSubDomains");
    }
});
```

---

## 4. Performance Optimization

```php
<?php
/**
 * WordPress Performance Optimization
 */

// ==========================================
// Disable Unnecessary Features
// ==========================================

function disable_unnecessary_features() {
    
    // ปิด Heartbeat API (ลด AJAX Calls)
    if (!is_admin()) {
        wp_deregister_script('heartbeat');
    }
    
    // ปิด Emoji
    remove_action('wp_head', 'print_emoji_detection_script', 7);
    remove_action('wp_print_styles', 'print_emoji_styles');
    remove_action('admin_print_scripts', 'print_emoji_detection_script');
    remove_action('admin_print_styles', 'print_emoji_styles');
    add_filter('emoji_svg_url', '__return_false');
    
    // ปิด oEmbed
    remove_action('wp_head', 'wp_oembed_add_discovery_links');
    remove_action('wp_head', 'wp_oembed_add_host_js');
    
    // ปิด REST API Link
    remove_action('wp_head', 'rest_output_link_wp_head');
    remove_action('wp_head', 'wp_resource_hints', 2);
    
    // ปิด Block Library CSS สำหรับ Classic Themes
    // add_filter('should_load_separate_core_block_assets', '__return_false');
    
    // ปิด Load Scripts Globally ที่ไม่ต้องการ
    if (!is_single() && !is_page()) {
        wp_dequeue_style('wp-block-library');
        wp_dequeue_style('wp-block-library-theme');
    }
}
add_action('wp_enqueue_scripts', 'disable_unnecessary_features', 100);

// ==========================================
// Lazy Loading
// ==========================================

// WordPress 5.5+ มี Native Lazy Loading
// เพิ่ม loading="lazy" ให้ Images ที่ไม่มีโดยอัตโนมัติ
add_filter('wp_lazy_loading_enabled', '__return_true');

// Custom Lazy Load Implementation
add_filter('the_content', function($content) {
    if (is_admin()) return $content;
    
    // เพิ่ม loading="lazy" ให้ iframes
    $content = preg_replace(
        '/<iframe([^>]*)>/i',
        '<iframe$1 loading="lazy">',
        $content
    );
    
    return $content;
});

// ==========================================
// Script Optimization
// ==========================================

// Defer Scripts
add_filter('script_loader_tag', function($tag, $handle) {
    
    $defer_scripts = array(
        'my-theme-main',
        'google-analytics',
    );
    
    if (in_array($handle, $defer_scripts)) {
        return str_replace(' src', ' defer src', $tag);
    }
    
    return $tag;
    
}, 10, 2);

// Preload Critical Resources
add_action('wp_head', function() {
    ?>
    <link rel="preload" href="<?php echo get_template_directory_uri(); ?>/assets/fonts/sarabun.woff2" 
          as="font" type="font/woff2" crossorigin>
    <link rel="preload" href="<?php echo get_template_directory_uri(); ?>/assets/css/critical.css" 
          as="style">
    <link rel="dns-prefetch" href="//fonts.googleapis.com">
    <link rel="dns-prefetch" href="//www.google-analytics.com">
    <?php
}, 1);

// ==========================================
// Database Optimization
// ==========================================

// ล้าง Post Revisions เก่า
function cleanup_post_revisions() {
    global $wpdb;
    
    // ลบ Revisions เก่ากว่า 30 วัน (เก็บแค่ 3 อัน)
    $wpdb->query("
        DELETE FROM {$wpdb->posts}
        WHERE post_type = 'revision'
        AND ID NOT IN (
            SELECT ID FROM (
                SELECT ID FROM {$wpdb->posts}
                WHERE post_type = 'revision'
                ORDER BY post_date DESC
                LIMIT 3
            ) as keep
        )
        AND post_date < DATE_SUB(NOW(), INTERVAL 30 DAY)
    ");
    
    // ล้าง Orphaned Post Meta
    $wpdb->query("
        DELETE pm FROM {$wpdb->postmeta} pm
        LEFT JOIN {$wpdb->posts} p ON pm.post_id = p.ID
        WHERE p.ID IS NULL
    ");
    
    // ล้าง Expired Transients
    $wpdb->query("
        DELETE FROM {$wpdb->options}
        WHERE option_name LIKE '_transient_timeout_%'
        AND option_value < " . time()
    );
    
    $wpdb->query("
        DELETE FROM {$wpdb->options}
        WHERE option_name LIKE '_transient_%'
        AND option_name NOT LIKE '_transient_timeout_%'
        AND option_name NOT IN (
            SELECT CONCAT('_transient_', SUBSTRING(option_name, 20))
            FROM {$wpdb->options}
            WHERE option_name LIKE '_transient_timeout_%'
        )
    ");
}

// Schedule Cleanup
add_action('wp', function() {
    if (!wp_next_scheduled('my_db_cleanup')) {
        wp_schedule_event(time(), 'weekly', 'my_db_cleanup');
    }
});
add_action('my_db_cleanup', 'cleanup_post_revisions');

// ==========================================
// Image Optimization
// ==========================================

// ลด Image Sizes ที่ไม่จำเป็น
function cleanup_image_sizes() {
    
    // ลบ Default Image Sizes ที่ไม่ต้องการ
    remove_image_size('1536x1536');
    remove_image_size('2048x2048');
    
    // ปรับ Default Sizes
    update_option('thumbnail_size_w', 150);
    update_option('thumbnail_size_h', 150);
    update_option('medium_size_w', 400);
    update_option('medium_size_h', 300);
    update_option('large_size_w', 800);
    update_option('large_size_h', 600);
}
add_action('after_setup_theme', 'cleanup_image_sizes');

// Convert Images to WebP (WordPress 5.8+)
add_filter('wp_editor_set_quality', function() {
    return 85; // JPEG Quality
});

// ==========================================
// WP_Query Optimization
// ==========================================

// ใช้ get_posts() แทน WP_Query สำหรับ Simple Queries
$recent_posts = get_posts(array(
    'post_type'              => 'post',
    'posts_per_page'         => 5,
    'no_found_rows'          => true,
    'update_post_meta_cache' => false,
    'update_post_term_cache' => false,
    'ignore_sticky_posts'    => true,
));

// ใช้ Fields เฉพาะที่ต้องการ
$post_ids = $wpdb->get_col(
    "SELECT ID FROM {$wpdb->posts} 
     WHERE post_type = 'post' 
     AND post_status = 'publish' 
     ORDER BY post_date DESC 
     LIMIT 10"
);

// ==========================================
// CDN Integration
// ==========================================

function integrate_cdn($url) {
    
    $cdn_url = 'https://cdn.example.com';
    $site_url = site_url();
    
    // เปลี่ยน URL ของ Assets ให้ชี้ไป CDN
    if (strpos($url, $site_url . '/wp-content') !== false) {
        $url = str_replace($site_url, $cdn_url, $url);
    }
    
    return $url;
}

// เฉพาะ Production
if (defined('WP_ENV') && WP_ENV === 'production') {
    add_filter('wp_get_attachment_url', 'integrate_cdn');
    add_filter('get_stylesheet_directory_uri', 'integrate_cdn');
    add_filter('plugins_url', 'integrate_cdn');
}
```

---

## 5. Monitoring และ Logging

```php
<?php
/**
 * WordPress Monitoring
 */

// ==========================================
// Custom Error Handler
// ==========================================

class WP_Error_Monitor {
    
    public static function init() {
        set_error_handler(array(__CLASS__, 'handle_error'));
        set_exception_handler(array(__CLASS__, 'handle_exception'));
        register_shutdown_function(array(__CLASS__, 'handle_shutdown'));
    }
    
    public static function handle_error($errno, $errstr, $errfile, $errline) {
        
        if (!(error_reporting() & $errno)) {
            return false;
        }
        
        $severity = self::get_severity($errno);
        
        self::log($severity, $errstr, array(
            'file'  => $errfile,
            'line'  => $errline,
            'errno' => $errno,
        ));
        
        return false;
    }
    
    public static function handle_exception($exception) {
        self::log('CRITICAL', $exception->getMessage(), array(
            'file'  => $exception->getFile(),
            'line'  => $exception->getLine(),
            'trace' => $exception->getTraceAsString(),
        ));
    }
    
    public static function handle_shutdown() {
        $error = error_get_last();
        if ($error && in_array($error['type'], array(E_ERROR, E_PARSE, E_CORE_ERROR))) {
            self::log('FATAL', $error['message'], array(
                'file' => $error['file'],
                'line' => $error['line'],
            ));
        }
    }
    
    private static function log($level, $message, $context = array()) {
        
        $log_entry = array(
            'timestamp' => current_time('mysql'),
            'level'     => $level,
            'message'   => $message,
            'url'       => $_SERVER['REQUEST_URI'] ?? '',
            'user_id'   => get_current_user_id(),
            'context'   => $context,
        );
        
        // บันทึกใน Database
        global $wpdb;
        $wpdb->insert($wpdb->prefix . 'error_logs', array(
            'level'      => $level,
            'message'    => $message,
            'context'    => json_encode($context),
            'created_at' => current_time('mysql'),
        ));
        
        // บันทึกใน File
        if (WP_DEBUG_LOG) {
            error_log("[{$level}] {$message} " . json_encode($context));
        }
        
        // Alert สำหรับ Critical Errors
        if (in_array($level, array('CRITICAL', 'FATAL'))) {
            wp_mail(
                get_option('admin_email'),
                "[{$level}] WordPress Error",
                $message . "\n\n" . print_r($context, true)
            );
        }
    }
    
    private static function get_severity($errno) {
        $map = array(
            E_ERROR        => 'ERROR',
            E_WARNING      => 'WARNING',
            E_NOTICE       => 'NOTICE',
            E_USER_ERROR   => 'ERROR',
            E_USER_WARNING => 'WARNING',
            E_USER_NOTICE  => 'NOTICE',
            E_DEPRECATED   => 'DEPRECATED',
        );
        return $map[$errno] ?? 'UNKNOWN';
    }
}

// Initialize (เฉพาะ Production)
if (defined('WP_ENV') && WP_ENV === 'production') {
    WP_Error_Monitor::init();
}
```

---

## Workshop: Performance Audit

### Exercise: Measure และ Optimize

```php
<?php
// 1. วัดเวลา Load ของ Page
$start_time = microtime(true);
add_action('wp_footer', function() use ($start_time) {
    $load_time = round((microtime(true) - $start_time) * 1000, 2);
    echo "<!-- Page loaded in {$load_time}ms -->";
});

// 2. วัดจำนวน Database Queries
add_action('wp_footer', function() {
    global $wpdb;
    echo "<!-- DB Queries: " . $wpdb->num_queries . " -->";
});

// 3. Check Memory Usage
add_action('wp_footer', function() {
    $memory = size_format(memory_get_peak_usage());
    echo "<!-- Memory Peak: {$memory} -->";
});
```

---

## Quiz

**คำถามที่ 1:** `wp_cache_set()` vs `set_transient()` ต่างกันอย่างไร?

A) ไม่มีความแตกต่าง  
B) wp_cache_set เก็บใน Memory (หายเมื่อ Request จบ), set_transient เก็บใน Database  
C) set_transient เร็วกว่า  
D) wp_cache_set ใช้กับ Redis เท่านั้น  

**เฉลย: B) wp_cache_set เก็บใน PHP Memory (per-request) หรือ Persistent Cache ถ้ามี Redis/Memcached, set_transient เก็บใน wp_options database**

---

**คำถามที่ 2:** `switch_to_blog()` ใน Multisite ต้องทำอะไรหลังใช้งาน?

A) delete_blog()  
B) restore_current_blog()  
C) close_blog()  
D) ไม่ต้องทำอะไร  

**เฉลย: B) restore_current_blog() ต้องเรียกเสมอเพื่อ reset กลับไป Blog ปัจจุบัน**

---

**คำถามที่ 3:** `no_found_rows => true` ใน WP_Query ช่วยอะไร?

A) ซ่อน Posts ที่ถูกลบ  
B) ข้าม SQL COUNT(*) query ทำให้เร็วขึ้นเมื่อไม่ต้องการ Pagination  
C) ล้าง Query Cache  
D) จำกัดจำนวน Results  

**เฉลย: B) ไม่ทำ SQL_CALC_FOUND_ROWS ทำให้ Query เร็วขึ้น แต่ใช้ Pagination ไม่ได้**

---

**คำถามที่ 4:** ควรเพิ่ม Security Headers ใน Hook ใด?

A) wp_head  
B) init  
C) send_headers  
D) wp_footer  

**เฉลย: C) send_headers action ส่ง HTTP Headers ก่อนส่ง Response**

---

## สรุป WordPress Section (Part 051-060)

ตลอด 10 Parts ที่ผ่านมาเราได้เรียนรู้:

| Part | หัวข้อ |
|------|--------|
| 051 | WordPress Installation, WP-CLI, Database |
| 052 | Theme Basics: Template Hierarchy, Hooks |
| 053 | Theme Advanced: Walker, Customizer |
| 054 | Plugin Development: OOP Structure |
| 055 | Custom Post Types & Taxonomies |
| 056 | Custom Fields & Meta Boxes |
| 057 | REST API |
| 058 | Gutenberg Blocks |
| 059 | WooCommerce |
| 060 | Advanced: Multisite, Cache, Security |

---

## ต่อไป

➡️ **[Part 076: Drupal Installation](part-076-drupal-installation.md)**

เริ่มต้นเรียน Drupal CMS:
- ติดตั้ง Drupal ด้วย Composer
- Drush CLI
- โครงสร้างโปรเจกต์
