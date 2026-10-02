# Part 059: WordPress WooCommerce

**ระดับ:** Advanced  
**เวลาเรียน:** 6-8 ชั่วโมง  
**Prerequisites:** Part 055 (Custom Post Types), Part 058

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. Customize WooCommerce ด้วย Hooks
2. สร้าง Custom Product Types
3. Customize Checkout Flow
4. สร้าง Custom Payment Gateway
5. ใช้ WooCommerce REST API

---

## 1. WooCommerce Hooks

```php
<?php
/**
 * WooCommerce Action และ Filter Hooks
 */

// ==========================================
// Product Page Hooks
// ==========================================

// เพิ่มเนื้อหาก่อน Product Title
add_action( 'woocommerce_before_single_product_summary', function() {
    echo '<div class="product-badge">สินค้าขายดี</div>';
}, 5 );

// เพิ่มข้อมูลหลัง Price
add_action( 'woocommerce_single_product_summary', function() {
    global $product;
    if ( $product->is_in_stock() ) {
        $stock_qty = $product->get_stock_quantity();
        echo '<p class="stock-info">เหลือสินค้า: ' . $stock_qty . ' ชิ้น</p>';
    }
}, 15 );

// เพิ่ม Delivery Info
add_action( 'woocommerce_after_add_to_cart_button', function() {
    echo '<p class="delivery-info">🚚 จัดส่งฟรีสำหรับคำสั่งซื้อ 500 บาทขึ้นไป</p>';
} );

// ==========================================
// Product Loop Hooks
// ==========================================

// ลบ Rating จาก Shop Page
remove_action( 'woocommerce_after_shop_loop_item_title', 'woocommerce_template_loop_rating', 5 );

// เพิ่ม Badge ใหม่
add_action( 'woocommerce_before_shop_loop_item', function() {
    global $product;
    
    $tags = wc_get_product_tag_ids( $product->get_id() );
    
    if ( has_term( 'new', 'product_tag' ) ) {
        echo '<span class="badge new-badge">ใหม่</span>';
    }
    
    if ( $product->is_on_sale() ) {
        $regular = $product->get_regular_price();
        $sale    = $product->get_sale_price();
        $percent = round( ( ($regular - $sale) / $regular ) * 100 );
        echo '<span class="badge sale-badge">-' . $percent . '%</span>';
    }
}, 5 );

// ==========================================
// Cart Hooks
// ==========================================

// เพิ่มข้อความใน Cart
add_action( 'woocommerce_before_cart_table', function() {
    $cart_total = WC()->cart->get_cart_contents_total();
    $free_shipping_threshold = 500;
    
    if ( $cart_total < $free_shipping_threshold ) {
        $remaining = $free_shipping_threshold - $cart_total;
        echo '<div class="free-shipping-notice">';
        echo 'เพิ่มอีก ' . wc_price($remaining) . ' เพื่อรับส่งฟรี!';
        echo '</div>';
    }
} );

// ==========================================
// Checkout Hooks
// ==========================================

// เพิ่ม Field ใน Checkout
add_action( 'woocommerce_after_order_notes', function( $checkout ) {
    echo '<div class="custom-checkout-field">';
    
    woocommerce_form_field( 'delivery_time', array(
        'type'        => 'select',
        'class'       => array('form-row-wide'),
        'label'       => __('เวลาจัดส่งที่ต้องการ', 'woocommerce'),
        'options'     => array(
            ''         => 'เลือกช่วงเวลา',
            'morning'  => '08:00 - 12:00',
            'afternoon' => '13:00 - 17:00',
            'evening'  => '18:00 - 21:00',
        ),
    ), $checkout->get_value('delivery_time') );
    
    echo '</div>';
} );

// Validate Custom Field
add_action( 'woocommerce_checkout_process', function() {
    if ( empty($_POST['delivery_time']) ) {
        wc_add_notice( 'กรุณาเลือกช่วงเวลาจัดส่ง', 'error' );
    }
} );

// บันทึก Custom Field กับ Order
add_action( 'woocommerce_checkout_update_order_meta', function( $order_id ) {
    if ( ! empty($_POST['delivery_time']) ) {
        update_post_meta(
            $order_id,
            '_delivery_time',
            sanitize_text_field($_POST['delivery_time'])
        );
    }
} );

// แสดงใน Admin Order Detail
add_action( 'woocommerce_admin_order_data_after_billing_address', function( $order ) {
    $delivery_time = get_post_meta($order->get_id(), '_delivery_time', true);
    if ($delivery_time) {
        echo '<p><strong>เวลาจัดส่ง:</strong> ' . esc_html($delivery_time) . '</p>';
    }
} );
```

---

## 2. WooCommerce Filters

```php
<?php
/**
 * WooCommerce Filters
 */

// ==========================================
// Product Price Filters
// ==========================================

// เพิ่ม Custom Discount
add_filter( 'woocommerce_product_get_price', function( $price, $product ) {
    
    // Discount 10% สำหรับ Member
    if ( is_user_logged_in() && wc_current_user_has_role('member') ) {
        return $price * 0.9;
    }
    
    return $price;
    
}, 10, 2 );

// เปลี่ยนรูปแบบ Price Display
add_filter( 'woocommerce_price_format', function( $format, $currency_pos ) {
    return '%2$s %1$s'; // Symbol หลัง Amount
}, 10, 2 );

// ==========================================
// Product Query Filters
// ==========================================

// กรองสินค้าใน Shop Loop
add_filter( 'woocommerce_product_query_meta_query', function( $meta_query ) {
    
    if ( ! is_user_logged_in() ) {
        // ซ่อนสินค้า "สมาชิกเท่านั้น" สำหรับ Guest
        $meta_query[] = array(
            'relation' => 'OR',
            array(
                'key'     => '_members_only',
                'compare' => 'NOT EXISTS',
            ),
            array(
                'key'     => '_members_only',
                'value'   => '1',
                'compare' => '!=',
            ),
        );
    }
    
    return $meta_query;
} );

// ==========================================
// Cart and Order Filters
// ==========================================

// เพิ่ม Custom Fee
add_action( 'woocommerce_cart_calculate_fees', function( $cart ) {
    
    if ( is_admin() && ! defined('DOING_AJAX') ) return;
    
    // เพิ่มค่าบริการ 50 บาท
    $cart->add_fee( 'ค่าบริการ', 50, true );
    
    // ลดราคาถ้าสั่งมากกว่า 5 ชิ้น
    $item_count = $cart->get_cart_contents_count();
    if ($item_count >= 5) {
        $cart->add_fee( 'ส่วนลดสั่งจำนวนมาก', -100, false );
    }
} );

// ==========================================
// Email Filters
// ==========================================

// เปลี่ยน From Email
add_filter( 'woocommerce_email_from_address', function() {
    return 'orders@myshop.com';
} );

// เพิ่ม Heading ใน Order Email
add_filter( 'woocommerce_email_heading', function( $heading, $email ) {
    if ( $email->id === 'new_order' ) {
        return 'คำสั่งซื้อใหม่ #' . $email->object->get_order_number();
    }
    return $heading;
}, 10, 2 );

// ==========================================
// Admin Columns Filters
// ==========================================

// เพิ่ม Column ใน Order List
add_filter( 'manage_woocommerce_page_wc-orders_columns', function( $columns ) {
    $new_columns = array();
    
    foreach ($columns as $key => $value) {
        $new_columns[$key] = $value;
        if ($key === 'order_status') {
            $new_columns['delivery_time'] = 'เวลาจัดส่ง';
        }
    }
    
    return $new_columns;
} );

add_action( 'manage_woocommerce_page_wc-orders_custom_column', function( $column, $order_or_id ) {
    if ($column === 'delivery_time') {
        $order_id = is_object($order_or_id) ? $order_or_id->get_id() : $order_or_id;
        $time = get_post_meta($order_id, '_delivery_time', true);
        echo $time ? esc_html($time) : '—';
    }
}, 10, 2 );
```

---

## 3. Custom Product Type

```php
<?php
/**
 * Custom Product Type: Digital Download with License
 */

// ==========================================
// สร้าง Product Class
// ==========================================

class WC_Product_Software extends WC_Product {
    
    public function __construct( $product ) {
        $this->product_type = 'software';
        parent::__construct($product);
    }
    
    // Override: Product Type
    public function get_type() {
        return 'software';
    }
    
    // Custom Methods
    public function get_license_type() {
        return $this->get_meta('_license_type');
    }
    
    public function set_license_type( $value ) {
        $this->add_meta_data('_license_type', $value, true);
    }
    
    public function get_max_activations() {
        return (int) $this->get_meta('_max_activations');
    }
    
    public function get_supported_platforms() {
        return $this->get_meta('_supported_platforms');
    }
    
    // Override: ไม่มีการจัดส่ง
    public function needs_shipping() {
        return false;
    }
    
    // Override: ดาวน์โหลดได้
    public function is_downloadable() {
        return true;
    }
    
    // Override: Virtual Product
    public function is_virtual() {
        return true;
    }
}

// ==========================================
// Register Product Type
// ==========================================

// เพิ่ม Product Type ใน WooCommerce
add_filter( 'product_type_selector', function( $types ) {
    $types['software'] = __('Software / License', 'woocommerce');
    return $types;
} );

// Load Class เมื่อ WooCommerce Load
add_filter( 'woocommerce_product_class', function( $class_name, $product_type ) {
    if ($product_type === 'software') {
        return 'WC_Product_Software';
    }
    return $class_name;
}, 10, 2 );

// ==========================================
// Custom Product Data Tab
// ==========================================

// เพิ่ม Tab ใน Product Data
add_filter( 'woocommerce_product_data_tabs', function( $tabs ) {
    
    $tabs['software_settings'] = array(
        'label'  => __('Software Settings', 'woocommerce'),
        'target' => 'software_settings_data',
        'class'  => array('show_if_software'),
        'priority' => 21,
    );
    
    return $tabs;
} );

// แสดง Tab Content
add_action( 'woocommerce_product_data_panels', function() {
    
    global $woocommerce, $post;
    
    echo '<div id="software_settings_data" class="panel woocommerce_options_panel">';
    
    woocommerce_wp_select( array(
        'id'      => '_license_type',
        'label'   => __('License Type', 'woocommerce'),
        'options' => array(
            'single'      => 'Single User',
            'multi'       => 'Multi User',
            'unlimited'   => 'Unlimited',
            'subscription' => 'Subscription',
        ),
    ) );
    
    woocommerce_wp_text_input( array(
        'id'          => '_max_activations',
        'label'       => __('Max Activations', 'woocommerce'),
        'type'        => 'number',
        'description' => 'จำนวนเครื่องที่ activate ได้',
        'custom_attributes' => array(
            'min' => 1,
            'step' => 1,
        ),
    ) );
    
    woocommerce_wp_checkbox( array(
        'id'          => '_windows_support',
        'label'       => 'Windows',
        'description' => 'รองรับ Windows',
    ) );
    
    woocommerce_wp_checkbox( array(
        'id'          => '_mac_support',
        'label'       => 'macOS',
        'description' => 'รองรับ macOS',
    ) );
    
    echo '</div>';
} );

// บันทึก Custom Product Data
add_action( 'woocommerce_process_product_meta_software', function( $post_id ) {
    
    update_post_meta($post_id, '_license_type', sanitize_text_field($_POST['_license_type'] ?? ''));
    update_post_meta($post_id, '_max_activations', absint($_POST['_max_activations'] ?? 1));
    update_post_meta($post_id, '_windows_support', isset($_POST['_windows_support']) ? 'yes' : 'no');
    update_post_meta($post_id, '_mac_support', isset($_POST['_mac_support']) ? 'yes' : 'no');
} );
```

---

## 4. Custom Payment Gateway

```php
<?php
/**
 * Custom Payment Gateway
 * 
 * ตัวอย่าง: Bank Transfer Gateway พร้อม QR Code
 */

add_filter( 'woocommerce_payment_gateways', function( $gateways ) {
    $gateways[] = 'WC_Gateway_QR_Payment';
    return $gateways;
} );

class WC_Gateway_QR_Payment extends WC_Payment_Gateway {
    
    public function __construct() {
        $this->id                 = 'qr_payment';
        $this->icon               = plugins_url('images/qr-icon.png', __FILE__);
        $this->has_fields         = true;
        $this->method_title       = __('QR Code Payment', 'woocommerce');
        $this->method_description = __('รับชำระเงินผ่าน QR Code', 'woocommerce');
        $this->supports           = array('products');
        
        // Load Settings
        $this->init_form_fields();
        $this->init_settings();
        
        // Settings Values
        $this->title       = $this->get_option('title');
        $this->description = $this->get_option('description');
        $this->enabled     = $this->get_option('enabled');
        $this->bank_name   = $this->get_option('bank_name');
        $this->account_no  = $this->get_option('account_no');
        $this->account_name = $this->get_option('account_name');
        
        // Hooks
        add_action(
            'woocommerce_update_options_payment_gateways_' . $this->id,
            array($this, 'process_admin_options')
        );
        add_action(
            'woocommerce_receipt_' . $this->id,
            array($this, 'receipt_page')
        );
        add_action(
            'woocommerce_api_' . $this->id,
            array($this, 'handle_webhook')
        );
    }
    
    public function init_form_fields() {
        $this->form_fields = array(
            'enabled' => array(
                'title'   => 'Enable/Disable',
                'type'    => 'checkbox',
                'label'   => 'Enable QR Code Payment',
                'default' => 'yes',
            ),
            'title' => array(
                'title'   => 'Title',
                'type'    => 'text',
                'default' => 'โอนเงินผ่าน QR Code',
            ),
            'description' => array(
                'title'   => 'Description',
                'type'    => 'textarea',
                'default' => 'สแกน QR Code เพื่อชำระเงิน',
            ),
            'bank_name' => array(
                'title'   => 'Bank Name',
                'type'    => 'text',
                'default' => 'ธนาคารกสิกรไทย',
            ),
            'account_no' => array(
                'title'   => 'Account Number',
                'type'    => 'text',
            ),
            'account_name' => array(
                'title'   => 'Account Name',
                'type'    => 'text',
            ),
            'qr_image' => array(
                'title'       => 'QR Code Image',
                'type'        => 'text',
                'description' => 'URL ของรูป QR Code',
            ),
        );
    }
    
    // แสดง Payment Fields ในหน้า Checkout
    public function payment_fields() {
        ?>
        <div class="qr-payment-fields">
            <p><?php echo esc_html($this->description); ?></p>
            
            <div class="bank-info">
                <strong>ธนาคาร:</strong> <?php echo esc_html($this->bank_name); ?><br>
                <strong>เลขบัญชี:</strong> <?php echo esc_html($this->account_no); ?><br>
                <strong>ชื่อบัญชี:</strong> <?php echo esc_html($this->account_name); ?>
            </div>
            
            <p class="upload-slip">กรุณาอัปโหลดสลิปการโอนเงิน</p>
            
            <div class="form-row">
                <input type="file" 
                       name="transfer_slip" 
                       accept="image/*" 
                       id="transfer_slip">
                <label for="transfer_slip">เลือกไฟล์สลิป</label>
            </div>
            
            <input type="hidden" name="transfer_date" value="<?php echo date('Y-m-d'); ?>">
        </div>
        <?php
    }
    
    // Validate Fields ก่อน Process
    public function validate_fields() {
        
        if ( ! isset($_FILES['transfer_slip']) || $_FILES['transfer_slip']['error'] !== UPLOAD_ERR_OK ) {
            wc_add_notice('กรุณาอัปโหลดสลิปการโอนเงิน', 'error');
            return false;
        }
        
        // ตรวจสอบ File Type
        $allowed_types = array('image/jpeg', 'image/png', 'image/gif', 'image/webp');
        $file_type = mime_content_type($_FILES['transfer_slip']['tmp_name']);
        
        if ( ! in_array($file_type, $allowed_types) ) {
            wc_add_notice('ไฟล์ต้องเป็นรูปภาพ (JPG, PNG, GIF)', 'error');
            return false;
        }
        
        return true;
    }
    
    // Process Payment
    public function process_payment( $order_id ) {
        
        $order = wc_get_order($order_id);
        
        // อัปโหลด Slip
        if ( isset($_FILES['transfer_slip']) && $_FILES['transfer_slip']['error'] === UPLOAD_ERR_OK ) {
            $slip_id = $this->upload_slip($_FILES['transfer_slip'], $order_id);
            if ($slip_id) {
                $order->update_meta_data('_transfer_slip_id', $slip_id);
            }
        }
        
        // ตั้งสถานะ Order เป็น Pending Payment
        $order->update_status('pending', __('รอตรวจสอบการชำระเงิน', 'woocommerce'));
        
        // บันทึก Order Note
        $order->add_order_note(
            sprintf('ลูกค้าแจ้งชำระเงินวันที่ %s รอการตรวจสอบ', sanitize_text_field($_POST['transfer_date'] ?? ''))
        );
        
        // ลด Cart
        WC()->cart->empty_cart();
        
        // ส่ง Email แจ้ง Admin
        WC()->mailer()->emails['WC_Email_New_Order']->trigger($order_id);
        
        return array(
            'result'   => 'success',
            'redirect' => $order->get_checkout_order_received_url(),
        );
    }
    
    // อัปโหลดสลิป
    private function upload_slip( $file, $order_id ) {
        
        require_once ABSPATH . 'wp-admin/includes/file.php';
        require_once ABSPATH . 'wp-admin/includes/media.php';
        require_once ABSPATH . 'wp-admin/includes/image.php';
        
        $upload = wp_handle_upload($file, array('test_form' => false));
        
        if ( isset($upload['error']) ) {
            return false;
        }
        
        $attachment_id = wp_insert_attachment(array(
            'guid'           => $upload['url'],
            'post_mime_type' => $upload['type'],
            'post_title'     => 'Transfer Slip - Order #' . $order_id,
            'post_content'   => '',
            'post_status'    => 'inherit',
        ), $upload['file'], $order_id);
        
        wp_generate_attachment_metadata($attachment_id, $upload['file']);
        
        return $attachment_id;
    }
    
    // หน้า Receipt หลังชำระเงิน
    public function receipt_page( $order_id ) {
        
        $order   = wc_get_order($order_id);
        $qr_url  = $this->get_option('qr_image');
        $amount  = $order->get_total();
        
        ?>
        <div class="qr-receipt">
            <h2>ขอบคุณสำหรับคำสั่งซื้อ!</h2>
            
            <?php if ($qr_url) : ?>
                <img src="<?php echo esc_url($qr_url); ?>" alt="QR Code">
            <?php endif; ?>
            
            <p>
                <strong>ยอดที่ต้องชำระ:</strong> 
                <?php echo wc_price($amount); ?>
            </p>
            
            <p>
                <strong>เลขคำสั่งซื้อ:</strong> 
                <?php echo $order->get_order_number(); ?>
            </p>
            
            <div class="payment-note">
                <p>กรุณาโอนเงินและอัปโหลดสลิปเพื่อยืนยันการชำระเงิน</p>
                
                <form method="post" enctype="multipart/form-data">
                    <input type="file" name="slip_upload" accept="image/*">
                    <?php wp_nonce_field('upload_slip_' . $order_id); ?>
                    <button type="submit">อัปโหลดสลิป</button>
                </form>
            </div>
        </div>
        <?php
    }
    
    // Webhook Handler (สำหรับ Payment Notification)
    public function handle_webhook() {
        
        $body = file_get_contents('php://input');
        $data = json_decode($body, true);
        
        if ( ! $data ) {
            wp_die('Invalid request', 400);
        }
        
        // ตรวจสอบ Signature
        $signature = $_SERVER['HTTP_X_SIGNATURE'] ?? '';
        $secret    = $this->get_option('webhook_secret');
        
        if ( ! hash_equals(hash_hmac('sha256', $body, $secret), $signature) ) {
            wp_die('Invalid signature', 401);
        }
        
        // หา Order
        $order_id = $data['order_id'] ?? null;
        $order    = wc_get_order($order_id);
        
        if ( ! $order ) {
            wp_die('Order not found', 404);
        }
        
        // Update Order Status
        if ( $data['status'] === 'confirmed' ) {
            $order->payment_complete($data['transaction_id']);
            $order->add_order_note('ยืนยันการชำระเงินแล้ว - ' . $data['transaction_id']);
        }
        
        echo json_encode(array('success' => true));
        exit;
    }
}
```

---

## 5. WooCommerce REST API

```php
<?php
/**
 * WooCommerce REST API
 */

// ==========================================
// WooCommerce REST API Endpoints
// ==========================================

/*
GET    /wc/v3/products               - รายการสินค้า
GET    /wc/v3/products/{id}          - สินค้าเดียว
POST   /wc/v3/products               - สร้างสินค้า
PUT    /wc/v3/products/{id}          - อัปเดตสินค้า
DELETE /wc/v3/products/{id}          - ลบสินค้า

GET    /wc/v3/orders                 - รายการคำสั่งซื้อ
GET    /wc/v3/orders/{id}            - คำสั่งซื้อเดียว
POST   /wc/v3/orders                 - สร้างคำสั่งซื้อ
PUT    /wc/v3/orders/{id}            - อัปเดตสถานะ

GET    /wc/v3/customers              - รายการลูกค้า
GET    /wc/v3/reports/sales          - รายงานยอดขาย
*/

// ==========================================
// PHP: ใช้ WooCommerce API
// ==========================================

function create_woo_order( $data ) {
    
    // กำหนด Order Data
    $order = wc_create_order();
    
    // เพิ่ม Product
    $product = wc_get_product($data['product_id']);
    $order->add_product($product, $data['quantity']);
    
    // ตั้ง Billing Address
    $order->set_billing_first_name($data['first_name']);
    $order->set_billing_last_name($data['last_name']);
    $order->set_billing_email($data['email']);
    $order->set_billing_phone($data['phone']);
    $order->set_billing_address_1($data['address']);
    $order->set_billing_city($data['city']);
    $order->set_billing_postcode($data['postcode']);
    $order->set_billing_country('TH');
    
    // ตั้งค่า Shipping (same as billing)
    $order->set_shipping_first_name($data['first_name']);
    $order->set_shipping_last_name($data['last_name']);
    $order->set_shipping_address_1($data['address']);
    
    // ตั้ง Payment Method
    $order->set_payment_method('qr_payment');
    $order->set_payment_method_title('QR Code Payment');
    
    // คำนวณ Totals
    $order->calculate_totals();
    
    // ตั้ง Status
    $order->update_status('pending', 'สร้างจาก API');
    
    return $order->get_id();
}

// ==========================================
// Extend WooCommerce REST API
// ==========================================

add_action('rest_api_init', function() {
    
    // เพิ่ม Endpoint สำหรับ Product Stock Update
    register_rest_route('my-plugin/v1', '/stock-update', array(
        'methods'             => 'POST',
        'callback'            => function(WP_REST_Request $request) {
            
            $product_id = $request->get_param('product_id');
            $quantity   = $request->get_param('quantity');
            
            $product = wc_get_product($product_id);
            if (!$product) {
                return new WP_Error('not_found', 'Product not found', array('status' => 404));
            }
            
            $product->set_stock_quantity($quantity);
            $product->save();
            
            return array(
                'product_id'    => $product_id,
                'new_stock'     => $product->get_stock_quantity(),
                'in_stock'      => $product->is_in_stock(),
            );
        },
        'permission_callback' => function() {
            return current_user_can('edit_products');
        },
    ));
    
});
```

---

## 6. WooCommerce Email Customization

```php
<?php
/**
 * Custom WooCommerce Email Template
 */

// Override Email Template โดยสร้างไฟล์ใน Theme:
// my-theme/woocommerce/emails/customer-completed-order.php

// เพิ่ม Custom Email
add_action('init', function() {
    
    // สร้าง Email Class
    class WC_Email_Transfer_Confirmed extends WC_Email {
        
        public function __construct() {
            $this->id             = 'transfer_confirmed';
            $this->title          = 'การยืนยันการโอนเงิน';
            $this->description    = 'ส่งให้ลูกค้าเมื่อยืนยันการโอนเงินแล้ว';
            $this->heading        = 'ยืนยันการชำระเงินแล้ว';
            $this->subject        = 'ยืนยันการชำระเงิน - #{order_number}';
            $this->template_html  = 'emails/transfer-confirmed.php';
            $this->template_plain = 'emails/plain/transfer-confirmed.php';
            
            parent::__construct();
        }
        
        public function trigger($order_id) {
            if (!$order_id) return;
            
            $this->setup_locale();
            
            $order = wc_get_order($order_id);
            if (!is_a($order, 'WC_Order')) return;
            
            $this->object = $order;
            $this->recipient = $order->get_billing_email();
            $this->placeholders['{order_number}'] = $order->get_order_number();
            
            if ($this->is_enabled() && $this->get_recipient()) {
                $this->send(
                    $this->get_recipient(),
                    $this->get_subject(),
                    $this->get_content(),
                    $this->get_headers(),
                    $this->get_attachments()
                );
            }
            
            $this->restore_locale();
        }
        
        public function get_content_html() {
            return wc_get_template_html(
                $this->template_html,
                array(
                    'order'         => $this->object,
                    'email_heading' => $this->get_heading(),
                    'sent_to_admin' => false,
                    'plain_text'    => false,
                    'email'         => $this,
                )
            );
        }
    }
});

// Register Custom Email
add_filter('woocommerce_email_classes', function($emails) {
    require_once 'class-wc-email-transfer-confirmed.php';
    $emails['WC_Email_Transfer_Confirmed'] = new WC_Email_Transfer_Confirmed();
    return $emails;
});
```

---

## Workshop: สร้าง Custom WooCommerce Feature

### Exercise: Wishlist System

```php
<?php
// TODO: สร้าง Wishlist Feature:
// 1. Custom Table: wc_wishlists
// 2. AJAX: เพิ่ม/ลบ Product
// 3. Shortcode: [my_wishlist]
// 4. Product Loop: เพิ่มปุ่ม "Add to Wishlist"

function add_to_wishlist() {
    
    check_ajax_referer('wishlist_nonce', 'nonce');
    
    if (!is_user_logged_in()) {
        wp_send_json_error(array('message' => 'Please login'));
    }
    
    $product_id = absint($_POST['product_id'] ?? 0);
    $user_id    = get_current_user_id();
    
    // TODO: Add to database
    
    wp_send_json_success(array('message' => 'Added to wishlist'));
}
add_action('wp_ajax_add_to_wishlist', 'add_to_wishlist');
add_action('wp_ajax_nopriv_add_to_wishlist', 'add_to_wishlist');
```

---

## Quiz

**คำถามที่ 1:** Hook ใดใช้เพิ่ม Custom Fields ใน WooCommerce Checkout?

A) `woocommerce_checkout_fields`  
B) `woocommerce_after_order_notes`  
C) `checkout_fields`  
D) `woocommerce_custom_fields`  

**เฉลย: B) woocommerce_after_order_notes (หรือ woocommerce_before_order_notes, woocommerce_checkout_billing)**

---

**คำถามที่ 2:** `WC_Payment_Gateway` method ที่ต้อง implement คืออะไร?

A) `execute_payment()`  
B) `process_payment($order_id)`  
C) `handle_payment()`  
D) `pay()`  

**เฉลย: B) process_payment($order_id) - ต้อง return array('result', 'redirect')**

---

**คำถามที่ 3:** `woocommerce_cart_calculate_fees` action ใช้ทำอะไร?

A) Calculate Product Price  
B) เพิ่ม/ลด ค่าธรรมเนียมพิเศษใน Cart  
C) คำนวณ Tax  
D) คำนวณ Shipping  

**เฉลย: B) เพิ่ม Custom Fees หรือ Discounts ใน Cart**

---

**คำถามที่ 4:** Custom Product Type ต้อง extend Class ใด?

A) WC_Product_Simple  
B) WC_Abstract_Product  
C) WC_Product  
D) WC_Custom_Product  

**เฉลย: C) WC_Product (หรือ WC_Product_Simple, WC_Product_Variable ขึ้นอยู่กับความต้องการ)**

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- WooCommerce Action/Filter Hooks
- Custom Checkout Fields
- Custom Product Types
- Custom Payment Gateway
- WooCommerce REST API
- Email Customization

---

## ต่อไป

➡️ **[Part 060: WordPress Advanced](part-060-wordpress-advanced.md)**

เรียนรู้เกี่ยวกับ:
- WordPress Multisite
- Caching Strategies
- Security Hardening
- Performance Optimization
