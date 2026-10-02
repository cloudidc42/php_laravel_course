# Part 054: WordPress Plugin Basics

**ระดับ:** Intermediate  
**เวลาเรียน:** 4-5 ชั่วโมง  
**Prerequisites:** Part 052 (WordPress Theme Basics)

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. เข้าใจโครงสร้าง Plugin และ Plugin Header
2. ใช้ Activation/Deactivation/Uninstall Hooks
3. สร้าง Admin Menu และ Settings Page
4. ใช้ Action และ Filter Hooks ใน Plugin
5. สร้าง Plugin ที่มีโครงสร้างแบบ OOP

---

## 1. โครงสร้าง Plugin

### 1.1 Minimum Plugin

```php
<?php
/**
 * Plugin Name: My First Plugin
 * Plugin URI: https://example.com/my-plugin
 * Description: This is my first WordPress plugin.
 * Version: 1.0.0
 * Requires at least: 5.9
 * Requires PHP: 7.4
 * Author: Your Name
 * Author URI: https://example.com
 * License: GPL-2.0-or-later
 * License URI: https://www.gnu.org/licenses/gpl-2.0.html
 * Text Domain: my-plugin
 * Domain Path: /languages
 * Network: false
 */

// ป้องกันการเข้าถึงโดยตรง
if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

// Plugin Code ที่นี่
add_action( 'init', function() {
    // TODO
} );
```

### 1.2 โครงสร้าง Plugin ที่ดี (OOP)

```
my-plugin/
├── my-plugin.php           # Main Plugin File (Plugin Header)
├── uninstall.php           # Runs on uninstall
├── readme.txt              # WordPress.org readme
├── composer.json           # Composer dependencies
├── package.json            # NPM dependencies
│
├── includes/               # PHP Classes
│   ├── class-my-plugin.php          # Main class
│   ├── class-my-plugin-admin.php    # Admin functionality
│   ├── class-my-plugin-public.php   # Public functionality
│   └── class-my-plugin-activator.php # Activation
│
├── admin/                  # Admin-specific assets
│   ├── css/
│   ├── js/
│   └── partials/           # Admin HTML templates
│
├── public/                 # Public-facing assets
│   ├── css/
│   ├── js/
│   └── partials/
│
├── languages/              # Translation files
│   ├── my-plugin.pot
│   └── my-plugin-th_TH.po
│
└── templates/              # Frontend templates
```

### 1.3 Plugin Main File

```php
<?php
/**
 * Plugin Name: Advanced Custom Plugin
 * Plugin URI: https://example.com
 * Description: A comprehensive custom plugin example.
 * Version: 1.0.0
 * Author: Your Name
 * License: GPL-2.0-or-later
 * Text Domain: acp
 * Domain Path: /languages
 */

if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

// Define Constants
define( 'ACP_VERSION',     '1.0.0' );
define( 'ACP_PLUGIN_DIR',  plugin_dir_path( __FILE__ ) );
define( 'ACP_PLUGIN_URL',  plugin_dir_url( __FILE__ ) );
define( 'ACP_PLUGIN_FILE', __FILE__ );
define( 'ACP_PLUGIN_BASE', plugin_basename( __FILE__ ) );

// Autoload
if ( file_exists( ACP_PLUGIN_DIR . 'vendor/autoload.php' ) ) {
    require_once ACP_PLUGIN_DIR . 'vendor/autoload.php';
}

// Include Classes
require_once ACP_PLUGIN_DIR . 'includes/class-acp-activator.php';
require_once ACP_PLUGIN_DIR . 'includes/class-acp.php';

// Activation Hook
register_activation_hook( __FILE__, array( 'ACP_Activator', 'activate' ) );

// Deactivation Hook
register_deactivation_hook( __FILE__, array( 'ACP_Activator', 'deactivate' ) );

// Initialize Plugin
function run_acp() {
    $plugin = new ACP();
    $plugin->run();
}

run_acp();
```

---

## 2. Activation, Deactivation, Uninstall

```php
<?php
/**
 * File: includes/class-acp-activator.php
 * 
 * การจัดการ Lifecycle ของ Plugin
 */

class ACP_Activator {
    
    /**
     * Runs on Plugin Activation
     * 
     * ใช้สร้าง Database Tables, Options, เพิ่ม Roles
     */
    public static function activate() {
        
        global $wpdb;
        
        // ตรวจสอบ PHP Version
        if ( version_compare( PHP_VERSION, '7.4', '<' ) ) {
            deactivate_plugins( ACP_PLUGIN_BASE );
            wp_die(
                __( 'Plugin นี้ต้องการ PHP 7.4 หรือสูงกว่า', 'acp' ),
                __( 'Plugin Activation Error', 'acp' ),
                array( 'back_link' => true )
            );
        }
        
        // ตรวจสอบ WordPress Version
        if ( version_compare( get_bloginfo('version'), '5.9', '<' ) ) {
            deactivate_plugins( ACP_PLUGIN_BASE );
            wp_die(
                __( 'Plugin นี้ต้องการ WordPress 5.9 หรือสูงกว่า', 'acp' ),
                __( 'Plugin Activation Error', 'acp' ),
                array( 'back_link' => true )
            );
        }
        
        // สร้าง Database Tables
        self::create_tables();
        
        // บันทึก Default Options
        self::set_default_options();
        
        // สร้าง Custom Roles
        self::create_roles();
        
        // Flush Rewrite Rules (สำหรับ Custom Post Types)
        flush_rewrite_rules();
        
        // บันทึก Activation Time
        update_option( 'acp_activated_at', current_time( 'timestamp' ) );
        update_option( 'acp_version', ACP_VERSION );
        
        // Set Flag สำหรับ Redirect หลัง Activation
        set_transient( 'acp_activation_redirect', true, 30 );
    }
    
    /**
     * Runs on Plugin Deactivation
     */
    public static function deactivate() {
        
        // Flush Rewrite Rules
        flush_rewrite_rules();
        
        // ลบ Cron Jobs
        wp_clear_scheduled_hook( 'acp_daily_cleanup' );
        
        // บันทึก Deactivation Time
        update_option( 'acp_deactivated_at', current_time( 'timestamp' ) );
        
        // หมายเหตุ: ไม่ลบข้อมูลใน Deactivation
        // การลบข้อมูลควรทำใน uninstall.php เท่านั้น
    }
    
    /**
     * สร้าง Custom Database Tables
     */
    private static function create_tables() {
        
        global $wpdb;
        
        $charset_collate = $wpdb->get_charset_collate();
        
        // ตาราง acp_logs
        $table_logs = $wpdb->prefix . 'acp_logs';
        
        $sql_logs = "CREATE TABLE IF NOT EXISTS $table_logs (
            id          bigint(20) unsigned NOT NULL AUTO_INCREMENT,
            user_id     bigint(20) unsigned NOT NULL DEFAULT '0',
            action      varchar(100) NOT NULL DEFAULT '',
            message     text,
            ip_address  varchar(45) NOT NULL DEFAULT '',
            created_at  datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
            PRIMARY KEY (id),
            KEY user_id (user_id),
            KEY action (action),
            KEY created_at (created_at)
        ) $charset_collate;";
        
        // ตาราง acp_bookings
        $table_bookings = $wpdb->prefix . 'acp_bookings';
        
        $sql_bookings = "CREATE TABLE IF NOT EXISTS $table_bookings (
            id          bigint(20) unsigned NOT NULL AUTO_INCREMENT,
            post_id     bigint(20) unsigned NOT NULL,
            user_id     bigint(20) unsigned NOT NULL,
            booking_date date NOT NULL,
            status      varchar(20) NOT NULL DEFAULT 'pending',
            notes       text,
            created_at  datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
            updated_at  datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
            PRIMARY KEY (id),
            KEY post_id (post_id),
            KEY user_id (user_id),
            KEY status (status)
        ) $charset_collate;";
        
        // dbDelta ใช้ Create/Update Tables อย่างปลอดภัย
        require_once ABSPATH . 'wp-admin/includes/upgrade.php';
        dbDelta( $sql_logs );
        dbDelta( $sql_bookings );
        
        // บันทึก DB Version
        update_option( 'acp_db_version', '1.0.0' );
    }
    
    /**
     * ตั้งค่า Default Options
     */
    private static function set_default_options() {
        
        $defaults = array(
            'enable_notifications' => true,
            'notification_email'   => get_option( 'admin_email' ),
            'max_bookings'         => 10,
            'booking_days_ahead'   => 30,
            'currency'             => 'THB',
            'date_format'          => 'Y-m-d',
        );
        
        // add_option ไม่ overwrite ถ้ามีอยู่แล้ว
        add_option( 'acp_settings', $defaults );
    }
    
    /**
     * สร้าง Custom User Roles
     */
    private static function create_roles() {
        
        // สร้าง Role: Booking Manager
        add_role(
            'booking_manager',
            __( 'Booking Manager', 'acp' ),
            array(
                'read'           => true,
                'manage_bookings' => true,
                'edit_posts'     => false,
            )
        );
        
        // เพิ่ม Capability ให้ Administrator
        $admin = get_role( 'administrator' );
        if ( $admin ) {
            $admin->add_cap( 'manage_bookings' );
            $admin->add_cap( 'view_booking_reports' );
        }
    }
}
```

```php
<?php
/**
 * File: uninstall.php
 * 
 * รันเมื่อ Delete Plugin จาก Admin
 * ลบข้อมูลทั้งหมดที่ Plugin สร้างไว้
 */

// ต้องตรวจสอบ WP_UNINSTALL_PLUGIN
if ( ! defined( 'WP_UNINSTALL_PLUGIN' ) ) {
    exit;
}

global $wpdb;

// ลบ Database Tables
$tables = array(
    $wpdb->prefix . 'acp_logs',
    $wpdb->prefix . 'acp_bookings',
);

foreach ( $tables as $table ) {
    $wpdb->query( "DROP TABLE IF EXISTS {$table}" );
}

// ลบ Options
delete_option( 'acp_settings' );
delete_option( 'acp_version' );
delete_option( 'acp_db_version' );
delete_option( 'acp_activated_at' );
delete_option( 'acp_deactivated_at' );

// ลบ Transients
delete_transient( 'acp_activation_redirect' );

// ลบ User Meta
$wpdb->delete( $wpdb->usermeta, array( 'meta_key' => 'acp_user_settings' ) );

// ลบ Post Meta
$wpdb->delete( $wpdb->postmeta, array( 'meta_key' => '_acp_booking_data' ) );

// ลบ Custom Role
remove_role( 'booking_manager' );

// ลบ Capabilities จาก Admin
$admin = get_role( 'administrator' );
if ( $admin ) {
    $admin->remove_cap( 'manage_bookings' );
    $admin->remove_cap( 'view_booking_reports' );
}

// ลบ Scheduled Events
wp_clear_scheduled_hook( 'acp_daily_cleanup' );
```

---

## 3. Main Plugin Class

```php
<?php
/**
 * File: includes/class-acp.php
 */

class ACP {
    
    protected $loader;
    protected $plugin_name;
    protected $version;
    
    public function __construct() {
        $this->plugin_name = 'advanced-custom-plugin';
        $this->version     = ACP_VERSION;
        
        $this->load_dependencies();
        $this->set_locale();
        $this->define_admin_hooks();
        $this->define_public_hooks();
    }
    
    private function load_dependencies() {
        require_once ACP_PLUGIN_DIR . 'includes/class-acp-loader.php';
        require_once ACP_PLUGIN_DIR . 'includes/class-acp-i18n.php';
        require_once ACP_PLUGIN_DIR . 'admin/class-acp-admin.php';
        require_once ACP_PLUGIN_DIR . 'public/class-acp-public.php';
        
        $this->loader = new ACP_Loader();
    }
    
    private function set_locale() {
        $plugin_i18n = new ACP_i18n();
        $this->loader->add_action( 'plugins_loaded', $plugin_i18n, 'load_plugin_textdomain' );
    }
    
    private function define_admin_hooks() {
        $plugin_admin = new ACP_Admin( $this->get_plugin_name(), $this->get_version() );
        
        $this->loader->add_action( 'admin_enqueue_scripts', $plugin_admin, 'enqueue_styles' );
        $this->loader->add_action( 'admin_enqueue_scripts', $plugin_admin, 'enqueue_scripts' );
        $this->loader->add_action( 'admin_menu', $plugin_admin, 'add_plugin_admin_menu' );
        $this->loader->add_action( 'admin_init', $plugin_admin, 'options_update' );
        
        // Redirect หลัง Activation
        $this->loader->add_action( 'admin_init', $plugin_admin, 'activation_redirect' );
        
        // Plugin Action Links
        $this->loader->add_filter(
            'plugin_action_links_' . ACP_PLUGIN_BASE,
            $plugin_admin,
            'add_action_links'
        );
    }
    
    private function define_public_hooks() {
        $plugin_public = new ACP_Public( $this->get_plugin_name(), $this->get_version() );
        
        $this->loader->add_action( 'wp_enqueue_scripts', $plugin_public, 'enqueue_styles' );
        $this->loader->add_action( 'wp_enqueue_scripts', $plugin_public, 'enqueue_scripts' );
        
        // Shortcodes
        $this->loader->add_action( 'init', $plugin_public, 'register_shortcodes' );
        
        // AJAX
        $this->loader->add_action( 'wp_ajax_acp_submit', $plugin_public, 'handle_ajax_submit' );
        $this->loader->add_action( 'wp_ajax_nopriv_acp_submit', $plugin_public, 'handle_ajax_submit' );
    }
    
    public function run() {
        $this->loader->run();
    }
    
    public function get_plugin_name() {
        return $this->plugin_name;
    }
    
    public function get_version() {
        return $this->version;
    }
}

// ==========================================
// Loader Class
// ==========================================

class ACP_Loader {
    
    protected $actions = array();
    protected $filters = array();
    
    public function add_action( $hook, $component, $callback, $priority = 10, $accepted_args = 1 ) {
        $this->actions = $this->add( $this->actions, $hook, $component, $callback, $priority, $accepted_args );
    }
    
    public function add_filter( $hook, $component, $callback, $priority = 10, $accepted_args = 1 ) {
        $this->filters = $this->add( $this->filters, $hook, $component, $callback, $priority, $accepted_args );
    }
    
    private function add( $hooks, $hook, $component, $callback, $priority, $accepted_args ) {
        $hooks[] = array(
            'hook'          => $hook,
            'component'     => $component,
            'callback'      => $callback,
            'priority'      => $priority,
            'accepted_args' => $accepted_args,
        );
        return $hooks;
    }
    
    public function run() {
        foreach ( $this->filters as $hook ) {
            add_filter(
                $hook['hook'],
                array( $hook['component'], $hook['callback'] ),
                $hook['priority'],
                $hook['accepted_args']
            );
        }
        
        foreach ( $this->actions as $hook ) {
            add_action(
                $hook['hook'],
                array( $hook['component'], $hook['callback'] ),
                $hook['priority'],
                $hook['accepted_args']
            );
        }
    }
}
```

---

## 4. Admin Class

```php
<?php
/**
 * File: admin/class-acp-admin.php
 */

class ACP_Admin {
    
    private $plugin_name;
    private $version;
    
    public function __construct( $plugin_name, $version ) {
        $this->plugin_name = $plugin_name;
        $this->version     = $version;
    }
    
    // ==========================================
    // Enqueue Admin Assets
    // ==========================================
    
    public function enqueue_styles( $hook ) {
        if ( strpos( $hook, $this->plugin_name ) !== false ) {
            wp_enqueue_style(
                $this->plugin_name,
                ACP_PLUGIN_URL . 'admin/css/acp-admin.css',
                array(),
                $this->version
            );
        }
    }
    
    public function enqueue_scripts( $hook ) {
        if ( strpos( $hook, $this->plugin_name ) !== false ) {
            wp_enqueue_script(
                $this->plugin_name,
                ACP_PLUGIN_URL . 'admin/js/acp-admin.js',
                array( 'jquery' ),
                $this->version,
                true
            );
            
            wp_localize_script(
                $this->plugin_name,
                'acpAdmin',
                array(
                    'ajaxUrl' => admin_url( 'admin-ajax.php' ),
                    'nonce'   => wp_create_nonce( 'acp-admin-nonce' ),
                )
            );
        }
    }
    
    // ==========================================
    // Admin Menu
    // ==========================================
    
    public function add_plugin_admin_menu() {
        
        // Main Menu
        add_menu_page(
            __( 'ACP Plugin', 'acp' ),           // Page Title
            __( 'ACP Plugin', 'acp' ),           // Menu Title
            'manage_options',                     // Capability
            $this->plugin_name,                   // Menu Slug
            array( $this, 'display_dashboard' ),  // Callback
            'dashicons-calendar-alt',             // Icon
            26                                    // Position
        );
        
        // Sub Menu
        add_submenu_page(
            $this->plugin_name,
            __( 'Dashboard', 'acp' ),
            __( 'Dashboard', 'acp' ),
            'manage_options',
            $this->plugin_name,
            array( $this, 'display_dashboard' )
        );
        
        add_submenu_page(
            $this->plugin_name,
            __( 'Bookings', 'acp' ),
            __( 'Bookings', 'acp' ),
            'manage_bookings',
            $this->plugin_name . '-bookings',
            array( $this, 'display_bookings' )
        );
        
        add_submenu_page(
            $this->plugin_name,
            __( 'Reports', 'acp' ),
            __( 'Reports', 'acp' ),
            'view_booking_reports',
            $this->plugin_name . '-reports',
            array( $this, 'display_reports' )
        );
        
        add_submenu_page(
            $this->plugin_name,
            __( 'Settings', 'acp' ),
            __( 'Settings', 'acp' ),
            'manage_options',
            $this->plugin_name . '-settings',
            array( $this, 'display_settings' )
        );
    }
    
    // ==========================================
    // Settings
    // ==========================================
    
    public function options_update() {
        register_setting(
            $this->plugin_name,
            $this->plugin_name,
            array( $this, 'validate_settings' )
        );
    }
    
    public function validate_settings( $input ) {
        $valid = array();
        
        $valid['enable_notifications'] = (bool) ( $input['enable_notifications'] ?? false );
        $valid['notification_email']   = sanitize_email( $input['notification_email'] ?? '' );
        $valid['max_bookings']         = absint( $input['max_bookings'] ?? 10 );
        
        if ( ! is_email( $valid['notification_email'] ) ) {
            add_settings_error(
                $this->plugin_name,
                'invalid_email',
                __( 'Invalid notification email address.', 'acp' )
            );
        }
        
        return $valid;
    }
    
    // ==========================================
    // Page Displays
    // ==========================================
    
    public function display_dashboard() {
        include ACP_PLUGIN_DIR . 'admin/partials/acp-dashboard.php';
    }
    
    public function display_bookings() {
        include ACP_PLUGIN_DIR . 'admin/partials/acp-bookings.php';
    }
    
    public function display_reports() {
        include ACP_PLUGIN_DIR . 'admin/partials/acp-reports.php';
    }
    
    public function display_settings() {
        include ACP_PLUGIN_DIR . 'admin/partials/acp-settings.php';
    }
    
    // ==========================================
    // Plugin Action Links
    // ==========================================
    
    public function add_action_links( $links ) {
        $settings_link = '<a href="' . 
            admin_url( 'admin.php?page=' . $this->plugin_name . '-settings' ) . 
            '">' . __( 'Settings', 'acp' ) . '</a>';
        
        array_unshift( $links, $settings_link );
        return $links;
    }
    
    // ==========================================
    // Activation Redirect
    // ==========================================
    
    public function activation_redirect() {
        if ( get_transient( 'acp_activation_redirect' ) ) {
            delete_transient( 'acp_activation_redirect' );
            wp_redirect( admin_url( 'admin.php?page=' . $this->plugin_name . '&activated=true' ) );
            exit;
        }
    }
}
```

---

## 5. Shortcodes

```php
<?php
/**
 * Shortcodes
 */

class ACP_Public {
    
    private $plugin_name;
    private $version;
    
    public function __construct( $plugin_name, $version ) {
        $this->plugin_name = $plugin_name;
        $this->version     = $version;
    }
    
    public function register_shortcodes() {
        add_shortcode( 'acp_booking_form', array( $this, 'booking_form_shortcode' ) );
        add_shortcode( 'acp_booking_list', array( $this, 'booking_list_shortcode' ) );
        add_shortcode( 'acp_calendar', array( $this, 'calendar_shortcode' ) );
    }
    
    /**
     * [acp_booking_form id="123" title="Book Now"]
     */
    public function booking_form_shortcode( $atts ) {
        
        $atts = shortcode_atts(
            array(
                'id'    => 0,
                'title' => __( 'Make a Booking', 'acp' ),
                'class' => '',
            ),
            $atts,
            'acp_booking_form'
        );
        
        // ตรวจสอบ ID
        $post_id = absint( $atts['id'] );
        if ( ! $post_id ) {
            $post_id = get_the_ID();
        }
        
        ob_start();
        include ACP_PLUGIN_DIR . 'public/partials/booking-form.php';
        return ob_get_clean();
    }
    
    /**
     * [acp_booking_list user_id="5" limit="10"]
     */
    public function booking_list_shortcode( $atts ) {
        
        $atts = shortcode_atts(
            array(
                'user_id' => get_current_user_id(),
                'limit'   => 10,
                'status'  => 'all',
            ),
            $atts,
            'acp_booking_list'
        );
        
        // ตรวจสอบสิทธิ์
        if ( ! is_user_logged_in() ) {
            return '<p>' . __( 'กรุณาเข้าสู่ระบบเพื่อดูรายการ Booking', 'acp' ) . '</p>';
        }
        
        global $wpdb;
        
        $query = "SELECT * FROM {$wpdb->prefix}acp_bookings WHERE user_id = %d";
        $params = array( absint( $atts['user_id'] ) );
        
        if ( $atts['status'] !== 'all' ) {
            $query  .= ' AND status = %s';
            $params[] = sanitize_text_field( $atts['status'] );
        }
        
        $query .= ' ORDER BY created_at DESC LIMIT %d';
        $params[] = absint( $atts['limit'] );
        
        $bookings = $wpdb->get_results( $wpdb->prepare( $query, $params ) );
        
        ob_start();
        include ACP_PLUGIN_DIR . 'public/partials/booking-list.php';
        return ob_get_clean();
    }
    
    // ==========================================
    // AJAX Handler
    // ==========================================
    
    public function handle_ajax_submit() {
        
        // ตรวจสอบ Nonce
        if ( ! check_ajax_referer( 'acp-booking-nonce', 'nonce', false ) ) {
            wp_send_json_error( array(
                'message' => __( 'Security check failed', 'acp' ),
            ), 403 );
        }
        
        // Validate Input
        $post_id = absint( $_POST['post_id'] ?? 0 );
        $date    = sanitize_text_field( $_POST['booking_date'] ?? '' );
        $notes   = sanitize_textarea_field( $_POST['notes'] ?? '' );
        
        if ( ! $post_id || ! $date ) {
            wp_send_json_error( array(
                'message' => __( 'กรุณากรอกข้อมูลให้ครบ', 'acp' ),
            ) );
        }
        
        // ตรวจสอบ Date Format
        $date_obj = DateTime::createFromFormat( 'Y-m-d', $date );
        if ( ! $date_obj || $date_obj->format('Y-m-d') !== $date ) {
            wp_send_json_error( array(
                'message' => __( 'รูปแบบวันที่ไม่ถูกต้อง', 'acp' ),
            ) );
        }
        
        // บันทึก Booking
        global $wpdb;
        
        $result = $wpdb->insert(
            $wpdb->prefix . 'acp_bookings',
            array(
                'post_id'      => $post_id,
                'user_id'      => get_current_user_id(),
                'booking_date' => $date,
                'status'       => 'pending',
                'notes'        => $notes,
            ),
            array( '%d', '%d', '%s', '%s', '%s' )
        );
        
        if ( $result === false ) {
            wp_send_json_error( array(
                'message' => __( 'เกิดข้อผิดพลาดในการบันทึก', 'acp' ),
            ) );
        }
        
        $booking_id = $wpdb->insert_id;
        
        // ส่ง Email แจ้ง Admin
        $this->send_booking_notification( $booking_id );
        
        wp_send_json_success( array(
            'message'    => __( 'จองสำเร็จ!', 'acp' ),
            'booking_id' => $booking_id,
        ) );
    }
    
    private function send_booking_notification( $booking_id ) {
        
        $settings = get_option( 'advanced-custom-plugin', array() );
        
        if ( empty( $settings['enable_notifications'] ) ) {
            return;
        }
        
        $to      = $settings['notification_email'] ?? get_option( 'admin_email' );
        $subject = sprintf( __( 'การจองใหม่ #%d', 'acp' ), $booking_id );
        $message = sprintf( 
            __( 'มีการจองใหม่เข้ามา กรุณาตรวจสอบที่ %s', 'acp' ),
            admin_url( 'admin.php?page=advanced-custom-plugin-bookings' )
        );
        
        wp_mail( $to, $subject, $message );
    }
    
    public function enqueue_styles() {
        wp_enqueue_style(
            $this->plugin_name,
            ACP_PLUGIN_URL . 'public/css/acp-public.css',
            array(),
            $this->version
        );
    }
    
    public function enqueue_scripts() {
        wp_enqueue_script(
            $this->plugin_name,
            ACP_PLUGIN_URL . 'public/js/acp-public.js',
            array( 'jquery' ),
            $this->version,
            true
        );
        
        wp_localize_script(
            $this->plugin_name,
            'acpPublic',
            array(
                'ajaxUrl' => admin_url( 'admin-ajax.php' ),
                'nonce'   => wp_create_nonce( 'acp-booking-nonce' ),
            )
        );
    }
}
```

---

## 6. Admin Partials

```php
<?php
/**
 * File: admin/partials/acp-settings.php
 */

// Security Check
if ( ! current_user_can( 'manage_options' ) ) {
    return;
}

$options = get_option( 'advanced-custom-plugin', array() );
?>

<div class="wrap">
    <h1><?php esc_html_e( 'ACP Plugin Settings', 'acp' ); ?></h1>
    
    <?php settings_errors( 'advanced-custom-plugin' ); ?>
    
    <?php if ( isset( $_GET['activated'] ) ) : ?>
        <div class="notice notice-success is-dismissible">
            <p><?php esc_html_e( 'ขอบคุณที่ใช้งาน ACP Plugin! ตั้งค่าเพิ่มเติมได้ที่นี่', 'acp' ); ?></p>
        </div>
    <?php endif; ?>
    
    <form method="post" action="options.php">
        <?php settings_fields( 'advanced-custom-plugin' ); ?>
        
        <table class="form-table" role="presentation">
            
            <tr>
                <th scope="row">
                    <label for="enable_notifications">
                        <?php esc_html_e( 'Email Notifications', 'acp' ); ?>
                    </label>
                </th>
                <td>
                    <input type="checkbox"
                           id="enable_notifications"
                           name="advanced-custom-plugin[enable_notifications]"
                           value="1"
                           <?php checked( $options['enable_notifications'] ?? false ); ?>>
                    <label for="enable_notifications">
                        <?php esc_html_e( 'ส่ง Email เมื่อมีการจองใหม่', 'acp' ); ?>
                    </label>
                </td>
            </tr>
            
            <tr>
                <th scope="row">
                    <label for="notification_email">
                        <?php esc_html_e( 'Notification Email', 'acp' ); ?>
                    </label>
                </th>
                <td>
                    <input type="email"
                           id="notification_email"
                           name="advanced-custom-plugin[notification_email]"
                           value="<?php echo esc_attr( $options['notification_email'] ?? get_option('admin_email') ); ?>"
                           class="regular-text">
                </td>
            </tr>
            
            <tr>
                <th scope="row">
                    <label for="max_bookings">
                        <?php esc_html_e( 'Max Bookings Per Day', 'acp' ); ?>
                    </label>
                </th>
                <td>
                    <input type="number"
                           id="max_bookings"
                           name="advanced-custom-plugin[max_bookings]"
                           value="<?php echo absint( $options['max_bookings'] ?? 10 ); ?>"
                           min="1"
                           max="100">
                </td>
            </tr>
            
        </table>
        
        <?php submit_button( __( 'บันทึกการตั้งค่า', 'acp' ) ); ?>
    </form>
</div>
```

---

## Workshop: สร้าง Plugin จาก Scratch

### Exercise: Contact Form Plugin

สร้าง Plugin สำหรับแบบฟอร์มติดต่อ:

```bash
# โครงสร้าง
wp scaffold plugin simple-contact-form
```

```php
<?php
/**
 * Plugin Name: Simple Contact Form
 * Description: A simple contact form plugin
 * Version: 1.0.0
 */

// ==========================================
// TODO: 
// 1. สร้าง Shortcode [contact_form]
// 2. แสดง Form: ชื่อ, อีเมล, ข้อความ
// 3. จัดการ Submission ด้วย AJAX
// 4. Validate ข้อมูล
// 5. ส่ง Email ไปยัง Admin
// 6. บันทึกใน Database
// ==========================================

add_shortcode( 'contact_form', function( $atts ) {
    ob_start();
    ?>
    <form id="contact-form" method="post">
        <?php wp_nonce_field( 'contact_form_submit', 'contact_nonce' ); ?>
        
        <div>
            <label for="contact-name">ชื่อ *</label>
            <input type="text" id="contact-name" name="contact_name" required>
        </div>
        
        <div>
            <label for="contact-email">อีเมล *</label>
            <input type="email" id="contact-email" name="contact_email" required>
        </div>
        
        <div>
            <label for="contact-message">ข้อความ *</label>
            <textarea id="contact-message" name="contact_message" required></textarea>
        </div>
        
        <button type="submit">ส่งข้อความ</button>
    </form>
    <?php
    return ob_get_clean();
} );

// TODO: เพิ่ม AJAX Handler
add_action( 'wp_ajax_submit_contact', 'handle_contact_submission' );
add_action( 'wp_ajax_nopriv_submit_contact', 'handle_contact_submission' );

function handle_contact_submission() {
    // TODO: implement
}
```

---

## Quiz

**คำถามที่ 1:** ฟังก์ชันใดใช้ Register Activation Hook ของ Plugin?

A) `plugin_activation_hook()`  
B) `register_activation_hook( __FILE__, 'my_function' )`  
C) `add_action( 'activate', 'my_function' )`  
D) `on_activate( 'my_function' )`  

**เฉลย: B) register_activation_hook( __FILE__, 'my_function' )**

---

**คำถามที่ 2:** ทำไมต้องใช้ `dbDelta()` แทน `$wpdb->query('CREATE TABLE ...')`?

A) เร็วกว่า  
B) dbDelta สามารถสร้างและอัปเดต Table Schema ได้อย่างปลอดภัย  
C) รองรับ Transactions  
D) ส่ง Notification ให้ Admin  

**เฉลย: B) dbDelta จัดการ ALTER TABLE อัตโนมัติเมื่อ Schema เปลี่ยน**

---

**คำถามที่ 3:** `uninstall.php` รันเมื่อไหร่?

A) เมื่อ Deactivate Plugin  
B) เมื่อ Update Plugin  
C) เมื่อ Delete Plugin จาก Admin  
D) เมื่อ Activate Plugin  

**เฉลย: C) เมื่อ Delete Plugin จาก WordPress Admin**

---

**คำถามที่ 4:** `wp_ajax_nopriv_` แตกต่างจาก `wp_ajax_` อย่างไร?

A) ไม่มีความแตกต่าง  
B) `nopriv` รองรับ User ที่ไม่ได้ Login  
C) `nopriv` เร็วกว่า  
D) `nopriv` ใช้กับ Admin เท่านั้น  

**เฉลย: B) wp_ajax_nopriv_ ทำงานสำหรับทั้ง Logged-in และ Non-logged-in Users**

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- โครงสร้าง Plugin และ Plugin Header
- Activation, Deactivation, Uninstall Hooks
- การสร้าง Database Tables ด้วย dbDelta
- Admin Menu และ Settings Page
- Shortcodes
- AJAX Handlers

---

## ต่อไป

➡️ **[Part 055: WordPress Custom Post Types](part-055-wordpress-custom-post-types.md)**

เรียนรู้เกี่ยวกับ:
- register_post_type
- register_taxonomy
- Custom Fields
- Meta Boxes
