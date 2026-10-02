# Part 063: WordPress Advanced Custom Plugin
## ระดับ: มืออาชีพ | ขั้นตอนที่ 681-720

---

## 🎯 เป้าหมายของ Part นี้

เมื่อเรียนจบ Part นี้คุณจะสามารถ:
- สร้าง Enterprise Plugin ด้วย OOP Architecture
- ใช้ Settings API และ Custom Database Tables
- สร้าง Admin UI ด้วย React
- ออกแบบ Plugin Hook System
- เขียน Plugin Tests ด้วย WP_Mock

---

## 📖 เนื้อหา

### 1. Enterprise Plugin Architecture

#### Plugin Main Class (Singleton)

```php
<?php
// my-booking-plugin.php

/**
 * Plugin Name: My Booking System
 * Plugin URI: https://example.com
 * Description: Enterprise booking management system
 * Version: 1.0.0
 * Author: Your Name
 * Text Domain: my-booking
 * Domain Path: /languages
 */

if (!defined('ABSPATH')) exit;

define('MBP_VERSION', '1.0.0');
define('MBP_PLUGIN_FILE', __FILE__);
define('MBP_PLUGIN_DIR', plugin_dir_path(__FILE__));
define('MBP_PLUGIN_URL', plugin_dir_url(__FILE__));

final class MyBookingPlugin {
    private static ?self $instance = null;
    
    private function __construct() {
        $this->loadDependencies();
        $this->setLocale();
        $this->defineAdminHooks();
        $this->definePublicHooks();
    }
    
    public static function getInstance(): self {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }
    
    private function loadDependencies(): void {
        require_once MBP_PLUGIN_DIR . 'includes/class-database.php';
        require_once MBP_PLUGIN_DIR . 'includes/class-booking.php';
        require_once MBP_PLUGIN_DIR . 'includes/class-admin.php';
        require_once MBP_PLUGIN_DIR . 'includes/class-api.php';
        require_once MBP_PLUGIN_DIR . 'public/class-public.php';
    }
    
    private function setLocale(): void {
        add_action('plugins_loaded', function() {
            load_plugin_textdomain('my-booking', false, MBP_PLUGIN_DIR . 'languages/');
        });
    }
    
    private function defineAdminHooks(): void {
        $admin = new MBP_Admin();
        add_action('admin_enqueue_scripts', [$admin, 'enqueueStyles']);
        add_action('admin_enqueue_scripts', [$admin, 'enqueueScripts']);
        add_action('admin_menu', [$admin, 'addMenuPages']);
    }
    
    private function definePublicHooks(): void {
        $public = new MBP_Public();
        add_action('wp_enqueue_scripts', [$public, 'enqueueStyles']);
        add_action('wp_enqueue_scripts', [$public, 'enqueueScripts']);
        add_shortcode('booking_form', [$public, 'renderBookingForm']);
    }
    
    public function run(): void {
        // Plugin is running
    }
    
    // ป้องกัน clone และ unserialize
    private function __clone() {}
    public function __wakeup(): never {
        throw new \Exception('Cannot unserialize singleton');
    }
}

// Activation / Deactivation / Uninstall hooks
register_activation_hook(__FILE__, ['MBP_Activator', 'activate']);
register_deactivation_hook(__FILE__, ['MBP_Deactivator', 'deactivate']);

function run_my_booking_plugin(): void {
    $plugin = MyBookingPlugin::getInstance();
    $plugin->run();
}
run_my_booking_plugin();
```

---

### 2. Custom Database Tables

```php
<?php
// includes/class-database.php

class MBP_Database {
    private static string $version_option = 'mbp_db_version';
    private static string $db_version = '1.2';
    
    public static function createTables(): void {
        global $wpdb;
        $charset_collate = $wpdb->get_charset_collate();
        
        $bookings_table = $wpdb->prefix . 'mbp_bookings';
        $services_table = $wpdb->prefix . 'mbp_services';
        
        $sql = "
        CREATE TABLE {$bookings_table} (
            id bigint(20) NOT NULL AUTO_INCREMENT,
            service_id bigint(20) NOT NULL,
            user_id bigint(20) DEFAULT NULL,
            customer_name varchar(100) NOT NULL,
            customer_email varchar(150) NOT NULL,
            customer_phone varchar(20) DEFAULT NULL,
            booking_date date NOT NULL,
            booking_time time NOT NULL,
            status varchar(20) NOT NULL DEFAULT 'pending',
            notes text DEFAULT NULL,
            total_price decimal(10,2) NOT NULL DEFAULT 0.00,
            created_at datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
            updated_at datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
            PRIMARY KEY (id),
            KEY service_id (service_id),
            KEY booking_date (booking_date),
            KEY status (status)
        ) {$charset_collate};
        
        CREATE TABLE {$services_table} (
            id bigint(20) NOT NULL AUTO_INCREMENT,
            name varchar(100) NOT NULL,
            description text DEFAULT NULL,
            duration_minutes int(11) NOT NULL DEFAULT 60,
            price decimal(10,2) NOT NULL DEFAULT 0.00,
            max_bookings_per_day int(11) NOT NULL DEFAULT 10,
            is_active tinyint(1) NOT NULL DEFAULT 1,
            created_at datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
            PRIMARY KEY (id)
        ) {$charset_collate};
        ";
        
        require_once ABSPATH . 'wp-admin/includes/upgrade.php';
        dbDelta($sql);
        
        update_option(self::$version_option, self::$db_version);
    }
    
    public static function dropTables(): void {
        global $wpdb;
        $wpdb->query("DROP TABLE IF EXISTS {$wpdb->prefix}mbp_bookings");
        $wpdb->query("DROP TABLE IF EXISTS {$wpdb->prefix}mbp_services");
        delete_option(self::$version_option);
    }
    
    public static function maybeUpgrade(): void {
        $installed_version = get_option(self::$version_option, '0');
        if (version_compare($installed_version, self::$db_version, '<')) {
            self::createTables();
        }
    }
}

// Activator
class MBP_Activator {
    public static function activate(): void {
        MBP_Database::createTables();
        self::createDefaultOptions();
        self::scheduleCleanup();
    }
    
    private static function createDefaultOptions(): void {
        $defaults = [
            'mbp_company_name'     => get_bloginfo('name'),
            'mbp_email_from'       => get_bloginfo('admin_email'),
            'mbp_booking_window'   => 30, // วันล่วงหน้าที่จองได้
            'mbp_cancel_before'    => 24, // ชั่วโมงก่อนยกเลิก
            'mbp_notification_email' => get_bloginfo('admin_email'),
        ];
        
        foreach ($defaults as $option => $value) {
            add_option($option, $value);
        }
    }
    
    private static function scheduleCleanup(): void {
        if (!wp_next_scheduled('mbp_cleanup_old_bookings')) {
            wp_schedule_event(time(), 'daily', 'mbp_cleanup_old_bookings');
        }
    }
}

// Deactivator
class MBP_Deactivator {
    public static function deactivate(): void {
        wp_clear_scheduled_hook('mbp_cleanup_old_bookings');
    }
}

// Uninstall (ใน uninstall.php)
// MBP_Database::dropTables();
// foreach (array_keys($defaults) as $option) { delete_option($option); }
```

---

### 3. Booking Model (Repository Pattern)

```php
<?php
// includes/class-booking.php

class MBP_Booking {
    private static function getTable(): string {
        global $wpdb;
        return $wpdb->prefix . 'mbp_bookings';
    }
    
    public static function create(array $data): int|false {
        global $wpdb;
        
        $defaults = [
            'status'     => 'pending',
            'created_at' => current_time('mysql'),
        ];
        
        $insert_data = array_merge($defaults, $data);
        
        // Validate required fields
        $required = ['service_id', 'customer_name', 'customer_email', 'booking_date', 'booking_time'];
        foreach ($required as $field) {
            if (empty($insert_data[$field])) {
                return false;
            }
        }
        
        $result = $wpdb->insert(
            self::getTable(),
            $insert_data,
            ['%d', '%s', '%s', '%s', '%s', '%s', '%f']
        );
        
        if ($result === false) {
            return false;
        }
        
        $booking_id = $wpdb->insert_id;
        
        // Fire action hook
        do_action('mbp_booking_created', $booking_id, $insert_data);
        
        return $booking_id;
    }
    
    public static function get(int $id): ?object {
        global $wpdb;
        return $wpdb->get_row(
            $wpdb->prepare("SELECT * FROM " . self::getTable() . " WHERE id = %d", $id)
        );
    }
    
    public static function getAll(array $args = []): array {
        global $wpdb;
        
        $defaults = [
            'status'     => '',
            'date_from'  => '',
            'date_to'    => '',
            'per_page'   => 20,
            'page'       => 1,
            'orderby'    => 'booking_date',
            'order'      => 'DESC',
        ];
        
        $args = wp_parse_args($args, $defaults);
        
        $where = ['1=1'];
        $values = [];
        
        if (!empty($args['status'])) {
            $where[] = 'status = %s';
            $values[] = $args['status'];
        }
        
        if (!empty($args['date_from'])) {
            $where[] = 'booking_date >= %s';
            $values[] = $args['date_from'];
        }
        
        if (!empty($args['date_to'])) {
            $where[] = 'booking_date <= %s';
            $values[] = $args['date_to'];
        }
        
        $where_clause = implode(' AND ', $where);
        $order = in_array(strtoupper($args['order']), ['ASC', 'DESC']) ? $args['order'] : 'DESC';
        $allowed_orderby = ['booking_date', 'created_at', 'status', 'id'];
        $orderby = in_array($args['orderby'], $allowed_orderby) ? $args['orderby'] : 'booking_date';
        
        $offset = ($args['page'] - 1) * $args['per_page'];
        
        $query = "SELECT * FROM " . self::getTable() . " WHERE {$where_clause} ORDER BY {$orderby} {$order} LIMIT %d OFFSET %d";
        $values[] = $args['per_page'];
        $values[] = $offset;
        
        if (!empty($values)) {
            $query = $wpdb->prepare($query, $values);
        }
        
        return $wpdb->get_results($query) ?: [];
    }
    
    public static function updateStatus(int $id, string $status): bool {
        global $wpdb;
        
        $allowed_statuses = ['pending', 'confirmed', 'cancelled', 'completed'];
        if (!in_array($status, $allowed_statuses)) {
            return false;
        }
        
        $old_booking = self::get($id);
        if (!$old_booking) return false;
        
        $result = $wpdb->update(
            self::getTable(),
            ['status' => $status, 'updated_at' => current_time('mysql')],
            ['id' => $id],
            ['%s', '%s'],
            ['%d']
        );
        
        if ($result !== false) {
            do_action('mbp_booking_status_changed', $id, $status, $old_booking->status);
        }
        
        return $result !== false;
    }
    
    public static function delete(int $id): bool {
        global $wpdb;
        $result = $wpdb->delete(self::getTable(), ['id' => $id], ['%d']);
        if ($result) {
            do_action('mbp_booking_deleted', $id);
        }
        return $result !== false;
    }
    
    public static function count(array $args = []): int {
        global $wpdb;
        $where = '1=1';
        if (!empty($args['status'])) {
            $where .= $wpdb->prepare(' AND status = %s', $args['status']);
        }
        return (int) $wpdb->get_var("SELECT COUNT(*) FROM " . self::getTable() . " WHERE {$where}");
    }
}
```

---

### 4. Admin UI ด้วย React

```php
<?php
// includes/class-admin.php

class MBP_Admin {
    public function addMenuPages(): void {
        add_menu_page(
            __('Booking System', 'my-booking'),
            __('Bookings', 'my-booking'),
            'manage_options',
            'mbp-bookings',
            [$this, 'renderMainPage'],
            'dashicons-calendar-alt',
            30
        );
        
        add_submenu_page('mbp-bookings', __('All Bookings', 'my-booking'), __('All Bookings', 'my-booking'), 'manage_options', 'mbp-bookings', [$this, 'renderMainPage']);
        add_submenu_page('mbp-bookings', __('Services', 'my-booking'), __('Services', 'my-booking'), 'manage_options', 'mbp-services', [$this, 'renderServicesPage']);
        add_submenu_page('mbp-bookings', __('Settings', 'my-booking'), __('Settings', 'my-booking'), 'manage_options', 'mbp-settings', [$this, 'renderSettingsPage']);
    }
    
    public function enqueueScripts(string $hook): void {
        if (strpos($hook, 'mbp-') === false) return;
        
        $asset_file = MBP_PLUGIN_DIR . 'admin/build/index.asset.php';
        $asset = file_exists($asset_file) ? require($asset_file) : ['dependencies' => [], 'version' => MBP_VERSION];
        
        wp_enqueue_script(
            'mbp-admin',
            MBP_PLUGIN_URL . 'admin/build/index.js',
            $asset['dependencies'],
            $asset['version'],
            true
        );
        
        wp_localize_script('mbp-admin', 'mbpAdmin', [
            'apiUrl'   => rest_url('mbp/v1/'),
            'nonce'    => wp_create_nonce('wp_rest'),
            'i18n'     => [
                'confirm_delete' => __('Are you sure?', 'my-booking'),
            ],
        ]);
    }
    
    public function enqueueStyles(string $hook): void {
        if (strpos($hook, 'mbp-') === false) return;
        wp_enqueue_style('mbp-admin', MBP_PLUGIN_URL . 'admin/build/index.css', [], MBP_VERSION);
    }
    
    public function renderMainPage(): void {
        echo '<div id="mbp-admin-root"></div>';
    }
    
    public function renderServicesPage(): void {
        echo '<div id="mbp-services-root"></div>';
    }
    
    public function renderSettingsPage(): void {
        ?>
        <div class="wrap">
            <h1><?php _e('Booking Settings', 'my-booking'); ?></h1>
            <form method="post" action="options.php">
                <?php
                settings_fields('mbp_settings_group');
                do_settings_sections('mbp-settings');
                submit_button();
                ?>
            </form>
        </div>
        <?php
    }
}
```

#### React Admin Component

```jsx
// admin/src/components/BookingList.jsx
import { useState, useEffect } from '@wordpress/element';
import apiFetch from '@wordpress/api-fetch';
import { Button, Badge, Spinner } from '@wordpress/components';

const STATUS_COLORS = {
    pending:   'warning',
    confirmed: 'success',
    cancelled: 'error',
    completed: 'info',
};

export default function BookingList() {
    const [bookings, setBookings] = useState([]);
    const [loading, setLoading] = useState(true);
    const [filter, setFilter] = useState('all');
    const [page, setPage] = useState(1);
    
    useEffect(() => {
        fetchBookings();
    }, [filter, page]);
    
    const fetchBookings = async () => {
        setLoading(true);
        try {
            const params = new URLSearchParams({ page, per_page: 20 });
            if (filter !== 'all') params.set('status', filter);
            
            const data = await apiFetch({ path: `/mbp/v1/bookings?${params}` });
            setBookings(data);
        } catch (error) {
            console.error('Failed to fetch bookings:', error);
        } finally {
            setLoading(false);
        }
    };
    
    const updateStatus = async (id, status) => {
        if (!confirm(mbpAdmin.i18n.confirm_delete)) return;
        
        try {
            await apiFetch({
                path: `/mbp/v1/bookings/${id}`,
                method: 'PATCH',
                data: { status },
            });
            fetchBookings();
        } catch (error) {
            console.error('Failed to update status:', error);
        }
    };
    
    if (loading) return <Spinner />;
    
    return (
        <div className="mbp-booking-list">
            <div className="mbp-filters">
                {['all', 'pending', 'confirmed', 'cancelled', 'completed'].map(s => (
                    <Button
                        key={s}
                        variant={filter === s ? 'primary' : 'secondary'}
                        onClick={() => setFilter(s)}
                    >
                        {s.charAt(0).toUpperCase() + s.slice(1)}
                    </Button>
                ))}
            </div>
            
            <table className="wp-list-table widefat fixed striped">
                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Customer</th>
                        <th>Service</th>
                        <th>Date & Time</th>
                        <th>Status</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {bookings.map(booking => (
                        <tr key={booking.id}>
                            <td>#{booking.id}</td>
                            <td>
                                <strong>{booking.customer_name}</strong>
                                <br /><small>{booking.customer_email}</small>
                            </td>
                            <td>{booking.service_name}</td>
                            <td>{booking.booking_date} {booking.booking_time}</td>
                            <td>
                                <Badge status={STATUS_COLORS[booking.status]}>
                                    {booking.status}
                                </Badge>
                            </td>
                            <td>
                                {booking.status === 'pending' && (
                                    <>
                                        <Button isSmall onClick={() => updateStatus(booking.id, 'confirmed')}>
                                            Confirm
                                        </Button>
                                        <Button isSmall isDestructive onClick={() => updateStatus(booking.id, 'cancelled')}>
                                            Cancel
                                        </Button>
                                    </>
                                )}
                            </td>
                        </tr>
                    ))}
                </tbody>
            </table>
        </div>
    );
}
```

---

### 5. REST API Endpoints

```php
<?php
// includes/class-api.php

class MBP_API {
    public function __construct() {
        add_action('rest_api_init', [$this, 'registerRoutes']);
    }
    
    public function registerRoutes(): void {
        $namespace = 'mbp/v1';
        
        register_rest_route($namespace, '/bookings', [
            [
                'methods'             => WP_REST_Server::READABLE,
                'callback'            => [$this, 'getBookings'],
                'permission_callback' => [$this, 'adminPermission'],
                'args'                => $this->getCollectionArgs(),
            ],
            [
                'methods'             => WP_REST_Server::CREATABLE,
                'callback'            => [$this, 'createBooking'],
                'permission_callback' => '__return_true', // Public
                'args'                => $this->getCreateArgs(),
            ],
        ]);
        
        register_rest_route($namespace, '/bookings/(?P<id>\d+)', [
            [
                'methods'             => WP_REST_Server::READABLE,
                'callback'            => [$this, 'getBooking'],
                'permission_callback' => [$this, 'adminPermission'],
            ],
            [
                'methods'             => 'PATCH',
                'callback'            => [$this, 'updateBooking'],
                'permission_callback' => [$this, 'adminPermission'],
            ],
            [
                'methods'             => WP_REST_Server::DELETABLE,
                'callback'            => [$this, 'deleteBooking'],
                'permission_callback' => [$this, 'adminPermission'],
            ],
        ]);
        
        register_rest_route($namespace, '/availability', [
            'methods'             => WP_REST_Server::READABLE,
            'callback'            => [$this, 'getAvailability'],
            'permission_callback' => '__return_true',
            'args'                => [
                'service_id' => ['required' => true, 'type' => 'integer'],
                'date'       => ['required' => true, 'type' => 'string', 'format' => 'date'],
            ],
        ]);
    }
    
    public function getBookings(WP_REST_Request $request): WP_REST_Response {
        $args = [
            'status'    => $request->get_param('status') ?? '',
            'date_from' => $request->get_param('date_from') ?? '',
            'date_to'   => $request->get_param('date_to') ?? '',
            'per_page'  => min((int)($request->get_param('per_page') ?? 20), 100),
            'page'      => max((int)($request->get_param('page') ?? 1), 1),
        ];
        
        $bookings = MBP_Booking::getAll($args);
        $total = MBP_Booking::count($args);
        
        $response = new WP_REST_Response($bookings, 200);
        $response->header('X-WP-Total', $total);
        $response->header('X-WP-TotalPages', ceil($total / $args['per_page']));
        
        return $response;
    }
    
    public function createBooking(WP_REST_Request $request): WP_REST_Response|WP_Error {
        $data = [
            'service_id'    => $request->get_param('service_id'),
            'customer_name' => sanitize_text_field($request->get_param('customer_name')),
            'customer_email'=> sanitize_email($request->get_param('customer_email')),
            'customer_phone'=> sanitize_text_field($request->get_param('customer_phone') ?? ''),
            'booking_date'  => sanitize_text_field($request->get_param('booking_date')),
            'booking_time'  => sanitize_text_field($request->get_param('booking_time')),
            'notes'         => sanitize_textarea_field($request->get_param('notes') ?? ''),
        ];
        
        // ตรวจสอบ availability
        $available = apply_filters('mbp_check_availability', true, $data['service_id'], $data['booking_date'], $data['booking_time']);
        if (!$available) {
            return new WP_Error('not_available', __('Selected time slot is not available.', 'my-booking'), ['status' => 409]);
        }
        
        $id = MBP_Booking::create($data);
        if (!$id) {
            return new WP_Error('create_failed', __('Failed to create booking.', 'my-booking'), ['status' => 500]);
        }
        
        return new WP_REST_Response(MBP_Booking::get($id), 201);
    }
    
    public function updateBooking(WP_REST_Request $request): WP_REST_Response|WP_Error {
        $id = (int) $request->get_param('id');
        $status = sanitize_text_field($request->get_param('status'));
        
        if (!MBP_Booking::updateStatus($id, $status)) {
            return new WP_Error('update_failed', __('Failed to update booking.', 'my-booking'), ['status' => 400]);
        }
        
        return new WP_REST_Response(MBP_Booking::get($id), 200);
    }
    
    public function deleteBooking(WP_REST_Request $request): WP_REST_Response|WP_Error {
        $id = (int) $request->get_param('id');
        if (!MBP_Booking::delete($id)) {
            return new WP_Error('delete_failed', 'Failed to delete.', ['status' => 500]);
        }
        return new WP_REST_Response(['deleted' => true], 200);
    }
    
    public function adminPermission(): bool {
        return current_user_can('manage_options');
    }
    
    private function getCollectionArgs(): array {
        return [
            'status'    => ['type' => 'string', 'default' => ''],
            'date_from' => ['type' => 'string'],
            'date_to'   => ['type' => 'string'],
            'per_page'  => ['type' => 'integer', 'default' => 20, 'minimum' => 1, 'maximum' => 100],
            'page'      => ['type' => 'integer', 'default' => 1, 'minimum' => 1],
        ];
    }
    
    private function getCreateArgs(): array {
        return [
            'service_id'     => ['required' => true, 'type' => 'integer'],
            'customer_name'  => ['required' => true, 'type' => 'string', 'minLength' => 2],
            'customer_email' => ['required' => true, 'type' => 'string', 'format' => 'email'],
            'booking_date'   => ['required' => true, 'type' => 'string', 'format' => 'date'],
            'booking_time'   => ['required' => true, 'type' => 'string'],
        ];
    }
}

new MBP_API();
```

---

### 6. Email Notifications (Action Hooks)

```php
<?php
// includes/class-notifications.php

class MBP_Notifications {
    public function __construct() {
        add_action('mbp_booking_created', [$this, 'sendConfirmationEmail'], 10, 2);
        add_action('mbp_booking_status_changed', [$this, 'sendStatusEmail'], 10, 3);
    }
    
    public function sendConfirmationEmail(int $booking_id, array $data): void {
        $booking = MBP_Booking::get($booking_id);
        if (!$booking) return;
        
        $subject = sprintf(
            __('[%s] Booking Confirmation #%d', 'my-booking'),
            get_bloginfo('name'),
            $booking_id
        );
        
        $message = $this->renderTemplate('booking-confirmation', [
            'booking'      => $booking,
            'company_name' => get_option('mbp_company_name'),
            'cancel_url'   => add_query_arg(['mbp_action' => 'cancel', 'id' => $booking_id, 'token' => $this->generateToken($booking_id)], home_url()),
        ]);
        
        wp_mail($booking->customer_email, $subject, $message, $this->getHeaders());
        
        // แจ้ง admin
        $admin_email = get_option('mbp_notification_email');
        if ($admin_email) {
            wp_mail($admin_email, 'New Booking: ' . $booking->customer_name, $message, $this->getHeaders());
        }
    }
    
    public function sendStatusEmail(int $booking_id, string $new_status, string $old_status): void {
        if ($new_status === $old_status) return;
        
        $booking = MBP_Booking::get($booking_id);
        if (!$booking) return;
        
        $subjects = [
            'confirmed'  => __('Your booking has been confirmed!', 'my-booking'),
            'cancelled'  => __('Your booking has been cancelled.', 'my-booking'),
            'completed'  => __('Thank you for your visit!', 'my-booking'),
        ];
        
        if (!isset($subjects[$new_status])) return;
        
        wp_mail(
            $booking->customer_email,
            $subjects[$new_status],
            $this->renderTemplate("booking-{$new_status}", ['booking' => $booking]),
            $this->getHeaders()
        );
    }
    
    private function renderTemplate(string $template, array $data): string {
        $template_file = MBP_PLUGIN_DIR . "templates/emails/{$template}.php";
        if (!file_exists($template_file)) return '';
        
        extract($data);
        ob_start();
        include $template_file;
        return ob_get_clean();
    }
    
    private function getHeaders(): array {
        return [
            'Content-Type: text/html; charset=UTF-8',
            'From: ' . get_option('mbp_company_name') . ' <' . get_option('mbp_email_from') . '>',
        ];
    }
    
    private function generateToken(int $booking_id): string {
        return wp_hash($booking_id . get_option('auth_key'));
    }
}

new MBP_Notifications();
```

---

### 7. Plugin Testing ด้วย WP_Mock

```php
<?php
// tests/test-booking.php

use PHPUnit\Framework\TestCase;
use Mockery;
use WP_Mock;

class BookingTest extends TestCase {
    protected function setUp(): void {
        parent::setUp();
        WP_Mock::setUp();
    }
    
    protected function tearDown(): void {
        WP_Mock::tearDown();
        Mockery::close();
        parent::tearDown();
    }
    
    public function testCreateBookingFiresAction(): void {
        global $wpdb;
        
        // Mock wpdb
        $wpdb = Mockery::mock('\wpdb');
        $wpdb->prefix = 'wp_';
        $wpdb->shouldReceive('insert')->once()->andReturn(1);
        $wpdb->insert_id = 42;
        $wpdb->shouldReceive('prepare')->andReturnUsing(fn($q, ...$a) => vsprintf(str_replace('%d', '%s', $q), $a));
        $wpdb->shouldReceive('get_row')->andReturn((object)['id' => 42, 'customer_name' => 'Test']);
        
        // Mock do_action
        WP_Mock::expectAction('mbp_booking_created', 42, Mockery::type('array'));
        
        $data = [
            'service_id'     => 1,
            'customer_name'  => 'Test User',
            'customer_email' => 'test@example.com',
            'booking_date'   => '2025-01-15',
            'booking_time'   => '10:00:00',
        ];
        
        $result = MBP_Booking::create($data);
        
        $this->assertEquals(42, $result);
        WP_Mock::assertActionsCalled();
    }
    
    public function testCreateBookingReturnsFalseWithMissingFields(): void {
        $data = ['service_id' => 1]; // Missing required fields
        $result = MBP_Booking::create($data);
        $this->assertFalse($result);
    }
    
    public function testUpdateStatusRejectsInvalidStatus(): void {
        global $wpdb;
        $wpdb = Mockery::mock('\wpdb');
        $wpdb->prefix = 'wp_';
        $wpdb->shouldReceive('get_row')->andReturn((object)['id' => 1]);
        
        $result = MBP_Booking::updateStatus(1, 'invalid_status');
        $this->assertFalse($result);
    }
}
```

---

## 🛠️ Workshop: โครงสร้าง Plugin สมบูรณ์

```
my-booking-plugin/
├── my-booking-plugin.php          # Main plugin file
├── uninstall.php                  # Cleanup on uninstall
├── composer.json
├── package.json
├── includes/
│   ├── class-database.php
│   ├── class-booking.php
│   ├── class-api.php
│   ├── class-notifications.php
│   └── class-settings.php
├── admin/
│   ├── class-admin.php
│   ├── src/
│   │   ├── index.js
│   │   └── components/
│   │       ├── BookingList.jsx
│   │       └── ServiceManager.jsx
│   └── build/                     # compiled assets
├── public/
│   ├── class-public.php
│   └── css/booking-form.css
├── templates/
│   ├── booking-form.php
│   └── emails/
│       ├── booking-confirmation.php
│       └── booking-confirmed.php
├── languages/
│   └── my-booking.pot
└── tests/
    ├── bootstrap.php
    └── test-booking.php
```

---

## 📝 Quiz

1. ทำไมต้องใช้ Singleton pattern สำหรับ Main Plugin class?
2. `dbDelta()` ใช้ทำอะไร และต่างจาก `$wpdb->query()` อย่างไร?
3. `register_activation_hook()` ต่างจาก `add_action('init')` อย่างไร?
4. ทำไมต้อง sanitize input ก่อน insert ลง database?
5. WP_Mock ช่วยอะไรในการ test WordPress plugin?

**เฉลย:**
1. ป้องกัน Plugin ถูก instantiate หลายครั้ง เนื่องจาก WordPress load ไฟล์ plugin หลายครั้งได้
2. `dbDelta()` สร้างหรืออัปเดต table โดยไม่ลบข้อมูลเดิม เหมาะสำหรับ database migrations
3. `register_activation_hook()` ทำงานเพียงครั้งเดียวตอน activate; `add_action('init')` ทำงานทุก request
4. ป้องกัน SQL Injection และข้อมูลที่ malicious เข้าไปใน database
5. WP_Mock mock WordPress functions (wp_mail, add_action ฯลฯ) ให้ test ได้โดยไม่ต้องมี WordPress จริง

---

## ⏭️ Part ถัดไป

**Part 076: Drupal Installation & Configuration**

---

*Part 063 | ระดับมืออาชีพ | WordPress Advanced Custom Plugin*
