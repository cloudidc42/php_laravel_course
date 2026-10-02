# Part 062: WordPress Headless CMS

## ระดับ: มืออาชีพ | ขั้นตอนที่ 641-680

---

## วัตถุประสงค์การเรียนรู้

เมื่อจบบทเรียนนี้ ผู้เรียนจะสามารถ:

1. อธิบายแนวคิด Headless CMS และความแตกต่างจาก Traditional CMS
2. ใช้งาน WordPress REST API อย่างละเอียดและครบถ้วน
3. ตั้งค่า JWT Authentication สำหรับระบบ Headless
4. เชื่อมต่อ Next.js กับ WordPress เป็น Backend
5. ใช้งาน WPGraphQL plugin สำหรับ GraphQL queries
6. จัดการข้อมูล ACF (Advanced Custom Fields) ผ่าน REST API
7. Deploy ระบบ Headless WordPress บน Vercel และ WordPress Hosting

---

## บทนำ: ทำไมต้องใช้ Headless CMS?

ในยุคปัจจุบัน การพัฒนา Web Application มีความซับซ้อนมากขึ้น ผู้ใช้งานต้องการประสบการณ์ที่รวดเร็ว ราบรื่น และทำงานได้บนทุกอุปกรณ์ แนวทาง Traditional CMS อย่าง WordPress ที่ใช้ Theme และ Plugin เพื่อ Render HTML ฝั่ง Server กำลังถูกแทนที่ด้วย Headless Architecture ที่แยก Frontend และ Backend ออกจากกันอย่างชัดเจน

**ข้อดีของ Headless CMS:**
- **Performance สูง** - Frontend สามารถใช้ Static Generation และ CDN ได้เต็มที่
- **Flexibility** - ใช้ Framework ใดก็ได้ (Next.js, Nuxt, Gatsby, SvelteKit)
- **Omnichannel** - Content เดียวกันส่งถึง Web, Mobile App, IoT ได้
- **Security** - WordPress Admin Panel ไม่ถูก Expose โดยตรง
- **Scalability** - Frontend และ Backend Scale แยกกันได้

---

## ขั้นตอนที่ 641-645: Headless CMS Concept

### 641. Traditional vs Headless Architecture

**Traditional WordPress (Coupled):**
```
Browser → WordPress PHP → Theme (Blade/PHP) → HTML Response
```

**Headless WordPress (Decoupled):**
```
Browser → Next.js (Frontend) → WordPress REST API/GraphQL → JSON Data
```

### แผนผัง Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Headless Architecture                  │
├─────────────────┬───────────────────────────────────────┤
│   Frontend      │            Backend (WordPress)         │
│   (Next.js)     │                                        │
│                 │  ┌─────────────┐  ┌────────────────┐  │
│  ┌──────────┐   │  │ WordPress   │  │   Database     │  │
│  │  Pages   │◄──┼──│ REST API /  │◄─│   (MySQL)      │  │
│  │  React   │   │  │  GraphQL    │  │                │  │
│  │Components│   │  └─────────────┘  └────────────────┘  │
│  └──────────┘   │                                        │
│       │         │  ┌─────────────┐                       │
│  ┌──────────┐   │  │  WP Admin   │                       │
│  │  Vercel  │   │  │  Dashboard  │                       │
│  │   CDN    │   │  └─────────────┘                       │
└─────────────────┴───────────────────────────────────────┘
```

### 642. การเตรียม WordPress สำหรับ Headless

ติดตั้ง WordPress ตามปกติ จากนั้นตั้งค่า:

```php
<?php
// wp-config.php - เพิ่มการตั้งค่าสำหรับ Headless

// อนุญาต CORS จาก Frontend Domain
define('HEADLESS_FRONTEND_URL', 'https://your-nextjs-app.vercel.app');

// ปิด WordPress Frontend (Optional)
define('WP_USE_THEMES', false);
```

เพิ่ม CORS Headers ใน `functions.php` หรือ Plugin:

```php
<?php
// functions.php หรือสร้าง Plugin ใหม่

add_action('init', function() {
    // อนุญาต Cross-Origin Requests จาก Frontend
    $allowed_origins = [
        'https://your-nextjs-app.vercel.app',
        'http://localhost:3000', // สำหรับ Development
    ];
    
    $origin = $_SERVER['HTTP_ORIGIN'] ?? '';
    
    if (in_array($origin, $allowed_origins)) {
        header("Access-Control-Allow-Origin: {$origin}");
        header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS');
        header('Access-Control-Allow-Headers: Content-Type, Authorization, X-WP-Nonce');
        header('Access-Control-Allow-Credentials: true');
    }
    
    // จัดการ Preflight Request
    if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
        status_header(200);
        exit();
    }
});
```

### 643. ทดสอบ WordPress REST API เบื้องต้น

WordPress REST API พร้อมใช้งานที่ `/wp-json/wp/v2/`:

```bash
# ดึงรายการ Posts ทั้งหมด
curl https://your-wordpress.com/wp-json/wp/v2/posts

# ดึง Post ตาม ID
curl https://your-wordpress.com/wp-json/wp/v2/posts/1

# ดึงด้วย Parameters
curl "https://your-wordpress.com/wp-json/wp/v2/posts?per_page=10&page=1&_embed=true"

# ดึง Categories
curl https://your-wordpress.com/wp-json/wp/v2/categories

# ดึง Tags
curl https://your-wordpress.com/wp-json/wp/v2/tags

# ดึง Pages
curl https://your-wordpress.com/wp-json/wp/v2/pages

# ดึง Media
curl https://your-wordpress.com/wp-json/wp/v2/media
```

### 644. Response Structure ของ REST API

```json
{
  "id": 1,
  "date": "2024-01-15T10:30:00",
  "date_gmt": "2024-01-15T03:30:00",
  "guid": {
    "rendered": "https://your-wordpress.com/?p=1"
  },
  "modified": "2024-01-15T10:30:00",
  "slug": "hello-world",
  "status": "publish",
  "type": "post",
  "link": "https://your-wordpress.com/hello-world/",
  "title": {
    "rendered": "Hello World"
  },
  "content": {
    "rendered": "<p>Welcome to WordPress...</p>",
    "protected": false
  },
  "excerpt": {
    "rendered": "<p>Welcome to WordPress...</p>",
    "protected": false
  },
  "author": 1,
  "featured_media": 5,
  "categories": [1],
  "tags": [],
  "_embedded": {
    "author": [{
      "id": 1,
      "name": "Admin",
      "avatar_urls": {
        "96": "https://..."
      }
    }],
    "wp:featuredmedia": [{
      "id": 5,
      "source_url": "https://your-wordpress.com/wp-content/uploads/...",
      "media_details": {
        "width": 1200,
        "height": 800,
        "sizes": {
          "thumbnail": { "source_url": "...", "width": 150, "height": 150 },
          "medium": { "source_url": "...", "width": 300, "height": 200 },
          "large": { "source_url": "...", "width": 1024, "height": 683 }
        }
      }
    }]
  }
}
```

### 645. WordPress REST API Namespace และ Routes

```bash
# ดู Routes ทั้งหมดที่มีอยู่
curl https://your-wordpress.com/wp-json/

# ดู Routes ใน Namespace wp/v2
curl https://your-wordpress.com/wp-json/wp/v2/

# ค้นหา Posts
curl "https://your-wordpress.com/wp-json/wp/v2/posts?search=keyword"

# Filter ด้วย Taxonomy
curl "https://your-wordpress.com/wp-json/wp/v2/posts?categories=5,10"
curl "https://your-wordpress.com/wp-json/wp/v2/posts?tags=3"

# Order และ Sort
curl "https://your-wordpress.com/wp-json/wp/v2/posts?orderby=date&order=desc"
curl "https://your-wordpress.com/wp-json/wp/v2/posts?orderby=title&order=asc"

# Select Fields เฉพาะที่ต้องการ
curl "https://your-wordpress.com/wp-json/wp/v2/posts?_fields=id,title,slug,excerpt,date"
```

---

## ขั้นตอนที่ 646-655: WordPress REST API Deep Dive

### 646. Custom Post Types กับ REST API

```php
<?php
// functions.php - ลงทะเบียน Custom Post Type พร้อม REST API Support

function register_product_post_type() {
    $args = [
        'public'       => true,
        'label'        => 'Products',
        'show_in_rest' => true, // สำคัญมาก! เปิดใช้ REST API
        'rest_base'    => 'products', // URL Endpoint: /wp-json/wp/v2/products
        'rest_controller_class' => 'WP_REST_Posts_Controller',
        'supports'     => ['title', 'editor', 'thumbnail', 'excerpt', 'custom-fields'],
        'has_archive'  => true,
        'rewrite'      => ['slug' => 'products'],
    ];
    
    register_post_type('product', $args);
}
add_action('init', 'register_product_post_type');

// Custom Taxonomy พร้อม REST API
function register_product_category() {
    $args = [
        'hierarchical' => true,
        'label'        => 'Product Categories',
        'show_in_rest' => true, // เปิด REST API
        'rest_base'    => 'product-categories',
        'rewrite'      => ['slug' => 'product-category'],
    ];
    
    register_taxonomy('product_category', ['product'], $args);
}
add_action('init', 'register_product_category');
```

### 647. เพิ่ม Custom Fields ลงใน REST API Response

```php
<?php
// เพิ่ม Meta Fields เข้า REST API Response

function register_product_meta_fields() {
    // ลงทะเบียน Meta Field
    register_post_meta('product', '_price', [
        'show_in_rest' => true,
        'single'       => true,
        'type'         => 'number',
        'description'  => 'Product price',
    ]);
    
    register_post_meta('product', '_sku', [
        'show_in_rest' => true,
        'single'       => true,
        'type'         => 'string',
        'description'  => 'Product SKU',
    ]);
    
    register_post_meta('product', '_stock', [
        'show_in_rest' => true,
        'single'       => true,
        'type'         => 'integer',
        'description'  => 'Stock quantity',
    ]);
}
add_action('init', 'register_product_meta_fields');

// เพิ่มข้อมูลพิเศษเข้า REST Response ด้วย register_rest_field
function add_product_fields_to_rest() {
    register_rest_field('product', 'product_data', [
        'get_callback' => function($post_arr) {
            $post_id = $post_arr['id'];
            
            return [
                'price'     => (float) get_post_meta($post_id, '_price', true),
                'sku'       => get_post_meta($post_id, '_sku', true),
                'stock'     => (int) get_post_meta($post_id, '_stock', true),
                'in_stock'  => (int) get_post_meta($post_id, '_stock', true) > 0,
                'gallery'   => get_post_meta($post_id, '_gallery', false), // Array
            ];
        },
        'update_callback' => null, // Read Only
        'schema' => [
            'description' => 'Product specific data',
            'type'        => 'object',
        ],
    ]);
}
add_action('rest_api_init', 'add_product_fields_to_rest');
```

### 648. Custom REST API Endpoints

```php
<?php
// สร้าง Custom REST API Endpoint

class Products_REST_Controller extends WP_REST_Controller {
    
    public function __construct() {
        $this->namespace = 'myapi/v1';
        $this->rest_base = 'products';
    }
    
    public function register_routes() {
        // GET /wp-json/myapi/v1/products
        register_rest_route($this->namespace, '/' . $this->rest_base, [
            [
                'methods'             => WP_REST_Server::READABLE,
                'callback'            => [$this, 'get_items'],
                'permission_callback' => [$this, 'get_items_permissions_check'],
                'args'                => $this->get_collection_params(),
            ],
            [
                'methods'             => WP_REST_Server::CREATABLE,
                'callback'            => [$this, 'create_item'],
                'permission_callback' => [$this, 'create_item_permissions_check'],
                'args'                => $this->get_endpoint_args_for_item_schema(WP_REST_Server::CREATABLE),
            ],
        ]);
        
        // GET/PUT/DELETE /wp-json/myapi/v1/products/{id}
        register_rest_route($this->namespace, '/' . $this->rest_base . '/(?P<id>[\d]+)', [
            [
                'methods'             => WP_REST_Server::READABLE,
                'callback'            => [$this, 'get_item'],
                'permission_callback' => [$this, 'get_item_permissions_check'],
                'args'                => ['id' => ['validate_callback' => 'rest_validate_request_arg']],
            ],
            [
                'methods'             => WP_REST_Server::EDITABLE,
                'callback'            => [$this, 'update_item'],
                'permission_callback' => [$this, 'update_item_permissions_check'],
            ],
            [
                'methods'             => WP_REST_Server::DELETABLE,
                'callback'            => [$this, 'delete_item'],
                'permission_callback' => [$this, 'delete_item_permissions_check'],
            ],
        ]);
        
        // Custom Endpoint: /wp-json/myapi/v1/products/featured
        register_rest_route($this->namespace, '/' . $this->rest_base . '/featured', [
            'methods'             => WP_REST_Server::READABLE,
            'callback'            => [$this, 'get_featured_products'],
            'permission_callback' => '__return_true', // Public access
        ]);
    }
    
    public function get_items($request) {
        $args = [
            'post_type'      => 'product',
            'post_status'    => 'publish',
            'posts_per_page' => $request->get_param('per_page') ?? 10,
            'paged'          => $request->get_param('page') ?? 1,
            'orderby'        => $request->get_param('orderby') ?? 'date',
            'order'          => $request->get_param('order') ?? 'DESC',
        ];
        
        // Filter by category
        if ($category = $request->get_param('category')) {
            $args['tax_query'] = [[
                'taxonomy' => 'product_category',
                'field'    => 'slug',
                'terms'    => $category,
            ]];
        }
        
        // Filter by price range
        if ($min_price = $request->get_param('min_price')) {
            $args['meta_query'][] = [
                'key'     => '_price',
                'value'   => $min_price,
                'compare' => '>=',
                'type'    => 'NUMERIC',
            ];
        }
        
        if ($max_price = $request->get_param('max_price')) {
            $args['meta_query'][] = [
                'key'     => '_price',
                'value'   => $max_price,
                'compare' => '<=',
                'type'    => 'NUMERIC',
            ];
        }
        
        $query = new WP_Query($args);
        $products = [];
        
        foreach ($query->posts as $post) {
            $products[] = $this->prepare_product_for_response($post, $request);
        }
        
        $response = rest_ensure_response($products);
        
        // เพิ่ม Pagination Headers
        $total = $query->found_posts;
        $total_pages = ceil($total / $args['posts_per_page']);
        
        $response->header('X-WP-Total', $total);
        $response->header('X-WP-TotalPages', $total_pages);
        
        return $response;
    }
    
    public function get_item($request) {
        $id   = $request->get_param('id');
        $post = get_post($id);
        
        if (!$post || $post->post_type !== 'product') {
            return new WP_Error(
                'rest_product_not_found',
                __('Product not found.'),
                ['status' => 404]
            );
        }
        
        return $this->prepare_product_for_response($post, $request);
    }
    
    public function get_featured_products($request) {
        $args = [
            'post_type'      => 'product',
            'post_status'    => 'publish',
            'posts_per_page' => 6,
            'meta_query'     => [[
                'key'   => '_featured',
                'value' => '1',
            ]],
        ];
        
        $query    = new WP_Query($args);
        $products = [];
        
        foreach ($query->posts as $post) {
            $products[] = $this->prepare_product_for_response($post, $request);
        }
        
        return rest_ensure_response($products);
    }
    
    private function prepare_product_for_response($post, $request) {
        return [
            'id'          => $post->ID,
            'title'       => $post->post_title,
            'slug'        => $post->post_name,
            'excerpt'     => wp_strip_all_tags($post->post_excerpt),
            'content'     => apply_filters('the_content', $post->post_content),
            'date'        => $post->post_date,
            'price'       => (float) get_post_meta($post->ID, '_price', true),
            'sku'         => get_post_meta($post->ID, '_sku', true),
            'stock'       => (int) get_post_meta($post->ID, '_stock', true),
            'featured'    => (bool) get_post_meta($post->ID, '_featured', true),
            'thumbnail'   => get_the_post_thumbnail_url($post->ID, 'large'),
            'categories'  => wp_get_post_terms($post->ID, 'product_category', ['fields' => 'names']),
            'link'        => get_permalink($post->ID),
        ];
    }
    
    public function get_items_permissions_check($request) {
        return true; // Public access
    }
    
    public function get_item_permissions_check($request) {
        return true;
    }
    
    public function create_item_permissions_check($request) {
        return current_user_can('edit_posts');
    }
    
    public function update_item_permissions_check($request) {
        return current_user_can('edit_posts');
    }
    
    public function delete_item_permissions_check($request) {
        return current_user_can('delete_posts');
    }
}

// ลงทะเบียน Controller
function register_products_rest_routes() {
    $controller = new Products_REST_Controller();
    $controller->register_routes();
}
add_action('rest_api_init', 'register_products_rest_routes');
```

### 649. REST API Filtering และ Query Optimization

```php
<?php
// เพิ่ม Custom Query Parameters สำหรับ Posts

add_filter('rest_post_query', function($args, $request) {
    // Filter by multiple categories
    if ($categories = $request->get_param('category_slugs')) {
        $slugs = explode(',', $categories);
        $args['tax_query'] = [[
            'taxonomy' => 'category',
            'field'    => 'slug',
            'terms'    => $slugs,
            'operator' => 'IN',
        ]];
    }
    
    // Filter by date range
    if ($after = $request->get_param('after_date')) {
        $args['date_query']['after'] = $after;
    }
    
    if ($before = $request->get_param('before_date')) {
        $args['date_query']['before'] = $before;
    }
    
    // Custom Meta Query
    if ($featured = $request->get_param('featured')) {
        $args['meta_query'] = [[
            'key'   => '_featured',
            'value' => '1',
        ]];
    }
    
    return $args;
}, 10, 2);

// เพิ่ม Custom Parameters
add_filter('rest_post_collection_params', function($params) {
    $params['category_slugs'] = [
        'description' => 'Filter by category slugs (comma-separated)',
        'type'        => 'string',
    ];
    
    $params['after_date'] = [
        'description' => 'Filter posts after this date',
        'type'        => 'string',
        'format'      => 'date-time',
    ];
    
    $params['before_date'] = [
        'description' => 'Filter posts before this date',
        'type'        => 'string',
        'format'      => 'date-time',
    ];
    
    $params['featured'] = [
        'description' => 'Show only featured posts',
        'type'        => 'boolean',
    ];
    
    return $params;
});
```

### 650. REST API Caching ด้วย Transients

```php
<?php
// Cache REST API Responses ด้วย WordPress Transients

add_filter('rest_pre_dispatch', function($result, $server, $request) {
    // Cache เฉพาะ GET Requests
    if ($request->get_method() !== 'GET') {
        return $result;
    }
    
    $cache_key = 'rest_cache_' . md5($request->get_route() . serialize($request->get_params()));
    $cached    = get_transient($cache_key);
    
    if ($cached !== false) {
        return $cached;
    }
    
    return $result;
}, 10, 3);

add_filter('rest_post_dispatch', function($response, $server, $request) {
    // Cache เฉพาะ Successful GET Requests
    if ($request->get_method() !== 'GET' || $response->get_status() !== 200) {
        return $response;
    }
    
    // ไม่ Cache Authenticated Requests
    if (is_user_logged_in()) {
        return $response;
    }
    
    $cache_key = 'rest_cache_' . md5($request->get_route() . serialize($request->get_params()));
    $ttl       = 300; // 5 นาที
    
    // Cache เฉพาะ Public Endpoints
    $cacheable_routes = [
        '/wp/v2/posts',
        '/wp/v2/pages',
        '/wp/v2/categories',
        '/myapi/v1/products',
    ];
    
    foreach ($cacheable_routes as $route) {
        if (strpos($request->get_route(), $route) !== false) {
            set_transient($cache_key, $response, $ttl);
            $response->header('X-Cache', 'MISS');
            break;
        }
    }
    
    return $response;
}, 10, 3);

// ล้าง Cache เมื่อมีการอัพเดต Post
add_action('save_post', function($post_id) {
    // ล้าง Cache ทั้งหมดที่เกี่ยวกับ Posts
    global $wpdb;
    $wpdb->query("DELETE FROM {$wpdb->options} WHERE option_name LIKE '_transient_rest_cache_%'");
    $wpdb->query("DELETE FROM {$wpdb->options} WHERE option_name LIKE '_transient_timeout_rest_cache_%'");
});
```

### 651. Webhooks สำหรับ Real-time Updates

```php
<?php
// ส่ง Webhook เมื่อมีการเผยแพร่ Post ใหม่

function send_post_published_webhook($new_status, $old_status, $post) {
    if ($new_status !== 'publish' || $old_status === 'publish') {
        return;
    }
    
    if ($post->post_type !== 'post') {
        return;
    }
    
    $webhook_url = get_option('headless_webhook_url');
    
    if (!$webhook_url) {
        return;
    }
    
    $payload = [
        'event'    => 'post.published',
        'post_id'  => $post->ID,
        'slug'     => $post->post_name,
        'title'    => $post->post_title,
        'date'     => $post->post_date,
        'author'   => get_the_author_meta('display_name', $post->post_author),
        'link'     => get_permalink($post->ID),
    ];
    
    wp_remote_post($webhook_url, [
        'method'  => 'POST',
        'headers' => [
            'Content-Type'  => 'application/json',
            'X-WP-Webhook'  => 'post.published',
            'Authorization' => 'Bearer ' . get_option('headless_webhook_secret'),
        ],
        'body'    => json_encode($payload),
        'timeout' => 10,
    ]);
}
add_action('transition_post_status', 'send_post_published_webhook', 10, 3);
```

### 652. Rate Limiting สำหรับ REST API

```php
<?php
// Rate Limiting สำหรับ REST API

class REST_Rate_Limiter {
    
    private $limit    = 100; // requests per window
    private $window   = 3600; // 1 hour in seconds
    
    public function __construct() {
        add_filter('rest_pre_dispatch', [$this, 'check_rate_limit'], 10, 3);
    }
    
    public function check_rate_limit($result, $server, $request) {
        $ip         = $_SERVER['REMOTE_ADDR'];
        $cache_key  = 'rate_limit_' . md5($ip);
        $count      = (int) get_transient($cache_key);
        
        if ($count >= $this->limit) {
            return new WP_Error(
                'rest_rate_limit_exceeded',
                'Rate limit exceeded. Try again later.',
                [
                    'status'      => 429,
                    'retry_after' => $this->get_remaining_time($cache_key),
                ]
            );
        }
        
        if ($count === 0) {
            set_transient($cache_key, 1, $this->window);
        } else {
            set_transient($cache_key, $count + 1, $this->get_remaining_time($cache_key));
        }
        
        // เพิ่ม Rate Limit Headers
        add_action('rest_send_allow_header', function() use ($count) {
            header('X-RateLimit-Limit: ' . $this->limit);
            header('X-RateLimit-Remaining: ' . ($this->limit - $count - 1));
        });
        
        return $result;
    }
    
    private function get_remaining_time($cache_key) {
        global $wpdb;
        $expiration = $wpdb->get_var($wpdb->prepare(
            "SELECT option_value FROM {$wpdb->options} WHERE option_name = %s",
            '_transient_timeout_' . $cache_key
        ));
        
        return $expiration ? $expiration - time() : $this->window;
    }
}

new REST_Rate_Limiter();
```

### 653. Versioning ของ Custom API

```php
<?php
// สร้าง Versioned API Endpoints

class API_V1_Controller {
    public function register_routes() {
        register_rest_route('myapi/v1', '/posts', [
            'methods'             => 'GET',
            'callback'            => [$this, 'get_posts_v1'],
            'permission_callback' => '__return_true',
        ]);
    }
    
    public function get_posts_v1($request) {
        // V1 Response Format
        return ['data' => $this->fetch_posts($request), 'version' => 'v1'];
    }
    
    protected function fetch_posts($request) {
        // ... implementation
    }
}

class API_V2_Controller extends API_V1_Controller {
    public function register_routes() {
        register_rest_route('myapi/v2', '/posts', [
            'methods'             => 'GET',
            'callback'            => [$this, 'get_posts_v2'],
            'permission_callback' => '__return_true',
        ]);
    }
    
    public function get_posts_v2($request) {
        // V2 Response Format - เพิ่มข้อมูลมากขึ้น
        $posts    = $this->fetch_posts($request);
        $enhanced = array_map(function($post) {
            $post['reading_time'] = $this->calculate_reading_time($post['content']);
            $post['social_share'] = $this->get_share_counts($post['id']);
            return $post;
        }, $posts);
        
        return [
            'data'     => $enhanced,
            'version'  => 'v2',
            'meta'     => [
                'total' => count($enhanced),
            ],
        ];
    }
    
    private function calculate_reading_time($content) {
        $word_count    = str_word_count(strip_tags($content));
        $reading_speed = 200; // words per minute
        return ceil($word_count / $reading_speed);
    }
    
    private function get_share_counts($post_id) {
        return (int) get_post_meta($post_id, '_share_count', true);
    }
}

add_action('rest_api_init', function() {
    (new API_V1_Controller())->register_routes();
    (new API_V2_Controller())->register_routes();
});
```

---

## ขั้นตอนที่ 654-660: JWT Authentication

### 654. ติดตั้งและตั้งค่า JWT Authentication

ติดตั้ง Plugin: **JWT Authentication for WP REST API**

```bash
# ติดตั้งผ่าน WP-CLI
wp plugin install jwt-authentication-for-wp-rest-api --activate
```

ตั้งค่าใน `wp-config.php`:

```php
<?php
// wp-config.php

// Secret Key สำหรับ JWT (ใช้ String ที่ Random และยาวพอ)
define('JWT_AUTH_SECRET_KEY', 'your-super-secret-key-here-make-it-long-and-random');

// เปิดใช้ CORS Support
define('JWT_AUTH_CORS_ENABLE', true);
```

ตั้งค่าใน `.htaccess`:

```apache
RewriteEngine on
RewriteCond %{HTTP:Authorization} ^(.*)
RewriteRule ^(.*) - [E=HTTP_AUTHORIZATION:%1]

# หรือสำหรับ FastCGI
SetEnvIf Authorization "(.*)" HTTP_AUTHORIZATION=$1
```

### 655. Authentication Flow

```javascript
// JavaScript - Login และรับ JWT Token

class WordPressAuth {
    constructor(baseUrl) {
        this.baseUrl = baseUrl;
        this.token   = null;
    }
    
    async login(username, password) {
        try {
            const response = await fetch(`${this.baseUrl}/wp-json/jwt-auth/v1/token`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({ username, password }),
            });
            
            if (!response.ok) {
                const error = await response.json();
                throw new Error(error.message || 'Login failed');
            }
            
            const data = await response.json();
            this.token = data.token;
            
            // เก็บ Token ใน localStorage (หรือ Cookie ที่ Secure กว่า)
            localStorage.setItem('wp_jwt_token', data.token);
            localStorage.setItem('wp_user_email', data.user_email);
            localStorage.setItem('wp_user_display_name', data.user_display_name);
            
            return data;
        } catch (error) {
            console.error('Login error:', error);
            throw error;
        }
    }
    
    async validateToken() {
        const token = this.getToken();
        if (!token) return false;
        
        try {
            const response = await fetch(`${this.baseUrl}/wp-json/jwt-auth/v1/token/validate`, {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${token}`,
                },
            });
            
            return response.ok;
        } catch {
            return false;
        }
    }
    
    getToken() {
        return this.token || localStorage.getItem('wp_jwt_token');
    }
    
    logout() {
        this.token = null;
        localStorage.removeItem('wp_jwt_token');
        localStorage.removeItem('wp_user_email');
        localStorage.removeItem('wp_user_display_name');
    }
    
    getAuthHeaders() {
        const token = this.getToken();
        return token ? { 'Authorization': `Bearer ${token}` } : {};
    }
}

// ตัวอย่างการใช้งาน
const auth = new WordPressAuth('https://your-wordpress.com');

// Login
const userData = await auth.login('username', 'password');
console.log('Logged in as:', userData.user_display_name);

// ตรวจสอบ Token
const isValid = await auth.validateToken();
console.log('Token valid:', isValid);

// Authenticated Request
const response = await fetch('https://your-wordpress.com/wp-json/wp/v2/posts', {
    headers: {
        ...auth.getAuthHeaders(),
        'Content-Type': 'application/json',
    },
});
```

### 656. Custom JWT Implementation ด้วย PHP

```php
<?php
// สร้าง Custom JWT Authentication Plugin

class Custom_JWT_Auth {
    
    private $secret_key;
    
    public function __construct() {
        $this->secret_key = defined('JWT_AUTH_SECRET_KEY') 
            ? JWT_AUTH_SECRET_KEY 
            : 'fallback-secret';
        
        add_action('rest_api_init', [$this, 'register_routes']);
        add_filter('determine_current_user', [$this, 'determine_current_user'], 20);
    }
    
    public function register_routes() {
        register_rest_route('custom-jwt/v1', '/auth/login', [
            'methods'             => 'POST',
            'callback'            => [$this, 'login'],
            'permission_callback' => '__return_true',
        ]);
        
        register_rest_route('custom-jwt/v1', '/auth/refresh', [
            'methods'             => 'POST',
            'callback'            => [$this, 'refresh_token'],
            'permission_callback' => '__return_true',
        ]);
        
        register_rest_route('custom-jwt/v1', '/auth/logout', [
            'methods'             => 'POST',
            'callback'            => [$this, 'logout'],
            'permission_callback' => [$this, 'is_authenticated'],
        ]);
    }
    
    public function login($request) {
        $username = sanitize_text_field($request->get_param('username'));
        $password = $request->get_param('password');
        
        // ตรวจสอบ Credentials
        $user = wp_authenticate($username, $password);
        
        if (is_wp_error($user)) {
            return new WP_Error(
                'jwt_auth_failed',
                'Invalid credentials.',
                ['status' => 401]
            );
        }
        
        // สร้าง Access Token (15 นาที)
        $access_token = $this->generate_token($user->ID, '+15 minutes', 'access');
        
        // สร้าง Refresh Token (7 วัน)
        $refresh_token = $this->generate_token($user->ID, '+7 days', 'refresh');
        
        // เก็บ Refresh Token ใน Database (เพื่อ Revocation)
        update_user_meta($user->ID, 'jwt_refresh_token', hash('sha256', $refresh_token));
        
        return new WP_REST_Response([
            'access_token'  => $access_token,
            'refresh_token' => $refresh_token,
            'token_type'    => 'Bearer',
            'expires_in'    => 900, // 15 minutes
            'user'          => [
                'id'           => $user->ID,
                'email'        => $user->user_email,
                'display_name' => $user->display_name,
                'roles'        => $user->roles,
            ],
        ], 200);
    }
    
    public function refresh_token($request) {
        $refresh_token = $request->get_param('refresh_token');
        
        if (!$refresh_token) {
            return new WP_Error('jwt_refresh_missing', 'Refresh token required.', ['status' => 400]);
        }
        
        try {
            $decoded = $this->decode_token($refresh_token);
            
            if ($decoded['type'] !== 'refresh') {
                throw new Exception('Invalid token type');
            }
            
            $user_id = $decoded['sub'];
            $user    = get_user_by('id', $user_id);
            
            if (!$user) {
                throw new Exception('User not found');
            }
            
            // ตรวจสอบว่า Refresh Token ยังใช้ได้
            $stored_hash = get_user_meta($user_id, 'jwt_refresh_token', true);
            if (hash('sha256', $refresh_token) !== $stored_hash) {
                throw new Exception('Refresh token revoked');
            }
            
            // สร้าง Access Token ใหม่
            $new_access_token = $this->generate_token($user_id, '+15 minutes', 'access');
            
            return new WP_REST_Response([
                'access_token' => $new_access_token,
                'token_type'   => 'Bearer',
                'expires_in'   => 900,
            ], 200);
            
        } catch (Exception $e) {
            return new WP_Error('jwt_refresh_failed', $e->getMessage(), ['status' => 401]);
        }
    }
    
    public function logout($request) {
        $user_id = get_current_user_id();
        delete_user_meta($user_id, 'jwt_refresh_token');
        
        return new WP_REST_Response(['message' => 'Logged out successfully.'], 200);
    }
    
    public function determine_current_user($user) {
        $token = $this->get_token_from_request();
        
        if (!$token) {
            return $user;
        }
        
        try {
            $decoded = $this->decode_token($token);
            
            if ($decoded['type'] !== 'access') {
                return $user;
            }
            
            return $decoded['sub'];
        } catch (Exception $e) {
            return $user;
        }
    }
    
    private function generate_token($user_id, $expiry, $type = 'access') {
        $issued_at = time();
        $expiration = strtotime($expiry, $issued_at);
        
        $payload = [
            'iss'  => get_bloginfo('url'),
            'iat'  => $issued_at,
            'exp'  => $expiration,
            'sub'  => $user_id,
            'type' => $type,
        ];
        
        return $this->encode_jwt($payload);
    }
    
    private function encode_jwt($payload) {
        $header  = base64url_encode(json_encode(['alg' => 'HS256', 'typ' => 'JWT']));
        $payload = base64url_encode(json_encode($payload));
        $sig     = base64url_encode(hash_hmac('sha256', "$header.$payload", $this->secret_key, true));
        
        return "$header.$payload.$sig";
    }
    
    private function decode_token($token) {
        $parts = explode('.', $token);
        
        if (count($parts) !== 3) {
            throw new Exception('Invalid token format');
        }
        
        [$header, $payload, $sig] = $parts;
        
        // ตรวจสอบ Signature
        $expected_sig = base64url_encode(hash_hmac('sha256', "$header.$payload", $this->secret_key, true));
        
        if (!hash_equals($expected_sig, $sig)) {
            throw new Exception('Invalid token signature');
        }
        
        $decoded = json_decode(base64url_decode($payload), true);
        
        // ตรวจสอบ Expiration
        if ($decoded['exp'] < time()) {
            throw new Exception('Token has expired');
        }
        
        return $decoded;
    }
    
    private function get_token_from_request() {
        $auth_header = $_SERVER['HTTP_AUTHORIZATION'] ?? $_SERVER['REDIRECT_HTTP_AUTHORIZATION'] ?? '';
        
        if (preg_match('/Bearer\s+(.*)$/i', $auth_header, $matches)) {
            return $matches[1];
        }
        
        return null;
    }
    
    public function is_authenticated($request) {
        return is_user_logged_in();
    }
}

// Helper functions
function base64url_encode($data) {
    return rtrim(strtr(base64_encode($data), '+/', '-_'), '=');
}

function base64url_decode($data) {
    return base64_decode(strtr($data, '-_', '+/'));
}

new Custom_JWT_Auth();
```

---

## ขั้นตอนที่ 661-668: Next.js Integration

### 661. ตั้งค่า Next.js Project

```bash
# สร้าง Next.js Project ใหม่
npx create-next-app@latest my-wordpress-headless \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir

cd my-wordpress-headless
```

สร้างไฟล์ `.env.local`:

```env
NEXT_PUBLIC_WORDPRESS_URL=https://your-wordpress.com
WORDPRESS_API_URL=https://your-wordpress.com/wp-json
WORDPRESS_AUTH_REFRESH_TOKEN=your-refresh-token
REVALIDATION_SECRET=your-revalidation-secret
```

### 662. WordPress API Client

```typescript
// src/lib/wordpress.ts

const WP_URL = process.env.NEXT_PUBLIC_WORDPRESS_URL!;
const API_URL = `${WP_URL}/wp-json/wp/v2`;

// Types
export interface Post {
    id: number;
    slug: string;
    title: { rendered: string };
    content: { rendered: string };
    excerpt: { rendered: string };
    date: string;
    modified: string;
    author: number;
    featured_media: number;
    categories: number[];
    tags: number[];
    _embedded?: {
        author: Author[];
        'wp:featuredmedia': FeaturedMedia[];
        'wp:term': Term[][];
    };
}

export interface Author {
    id: number;
    name: string;
    description: string;
    avatar_urls: { [size: string]: string };
}

export interface FeaturedMedia {
    id: number;
    source_url: string;
    alt_text: string;
    media_details: {
        width: number;
        height: number;
        sizes: {
            thumbnail?: { source_url: string; width: number; height: number };
            medium?: { source_url: string; width: number; height: number };
            large?: { source_url: string; width: number; height: number };
            full?: { source_url: string; width: number; height: number };
        };
    };
}

export interface Category {
    id: number;
    name: string;
    slug: string;
    description: string;
    count: number;
    parent: number;
}

export interface Term {
    id: number;
    name: string;
    slug: string;
    taxonomy: string;
}

// Fetch Functions
export async function getAllPosts(params?: {
    page?: number;
    perPage?: number;
    categoryId?: number;
    tagId?: number;
    search?: string;
}): Promise<{ posts: Post[]; totalPages: number; total: number }> {
    const { page = 1, perPage = 10, categoryId, tagId, search } = params ?? {};
    
    const url = new URL(`${API_URL}/posts`);
    url.searchParams.set('page', String(page));
    url.searchParams.set('per_page', String(perPage));
    url.searchParams.set('_embed', 'true');
    url.searchParams.set('status', 'publish');
    
    if (categoryId) url.searchParams.set('categories', String(categoryId));
    if (tagId)       url.searchParams.set('tags', String(tagId));
    if (search)      url.searchParams.set('search', search);
    
    const response = await fetch(url.toString(), {
        next: { revalidate: 300 }, // ISR: Revalidate ทุก 5 นาที
    });
    
    if (!response.ok) {
        throw new Error(`Failed to fetch posts: ${response.statusText}`);
    }
    
    const posts: Post[]    = await response.json();
    const total            = parseInt(response.headers.get('X-WP-Total') ?? '0');
    const totalPages       = parseInt(response.headers.get('X-WP-TotalPages') ?? '1');
    
    return { posts, total, totalPages };
}

export async function getPostBySlug(slug: string): Promise<Post | null> {
    const response = await fetch(
        `${API_URL}/posts?slug=${slug}&_embed=true`,
        {
            next: {
                revalidate: 3600,
                tags: [`post-${slug}`], // On-Demand Revalidation
            },
        }
    );
    
    if (!response.ok) return null;
    
    const posts: Post[] = await response.json();
    return posts[0] ?? null;
}

export async function getAllPostSlugs(): Promise<string[]> {
    let page    = 1;
    let slugs: string[] = [];
    
    while (true) {
        const response = await fetch(
            `${API_URL}/posts?per_page=100&page=${page}&_fields=slug&status=publish`,
            { next: { revalidate: 3600 } }
        );
        
        if (!response.ok) break;
        
        const posts: Pick<Post, 'slug'>[] = await response.json();
        
        if (posts.length === 0) break;
        
        slugs = [...slugs, ...posts.map(p => p.slug)];
        
        const totalPages = parseInt(response.headers.get('X-WP-TotalPages') ?? '1');
        if (page >= totalPages) break;
        
        page++;
    }
    
    return slugs;
}

export async function getAllCategories(): Promise<Category[]> {
    const response = await fetch(
        `${API_URL}/categories?per_page=100&hide_empty=true`,
        { next: { revalidate: 3600 } }
    );
    
    if (!response.ok) return [];
    
    return response.json();
}

export async function getCategoryBySlug(slug: string): Promise<Category | null> {
    const response = await fetch(
        `${API_URL}/categories?slug=${slug}`,
        { next: { revalidate: 3600 } }
    );
    
    if (!response.ok) return null;
    
    const categories: Category[] = await response.json();
    return categories[0] ?? null;
}

// Helper Functions
export function getExcerpt(post: Post, length = 150): string {
    const text = post.excerpt.rendered.replace(/<[^>]*>/g, '').trim();
    return text.length > length ? `${text.substring(0, length)}...` : text;
}

export function getFeaturedImageUrl(post: Post, size: keyof FeaturedMedia['media_details']['sizes'] = 'large'): string | null {
    const media = post._embedded?.['wp:featuredmedia']?.[0];
    if (!media) return null;
    
    return media.media_details.sizes[size]?.source_url ?? media.source_url;
}

export function formatDate(dateString: string, locale = 'th-TH'): string {
    return new Date(dateString).toLocaleDateString(locale, {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
    });
}
```

### 663. Next.js Pages ด้วย App Router

```typescript
// src/app/page.tsx - หน้า Blog หลัก

import { getAllPosts, getAllCategories, getExcerpt, getFeaturedImageUrl, formatDate } from '@/lib/wordpress';
import PostCard from '@/components/PostCard';
import CategoryFilter from '@/components/CategoryFilter';
import Pagination from '@/components/Pagination';

interface HomePageProps {
    searchParams: {
        page?: string;
        category?: string;
        search?: string;
    };
}

export default async function HomePage({ searchParams }: HomePageProps) {
    const page    = parseInt(searchParams.page ?? '1');
    const search  = searchParams.search;
    
    // Fetch Posts และ Categories พร้อมกัน
    const [{ posts, total, totalPages }, categories] = await Promise.all([
        getAllPosts({ page, perPage: 12, search }),
        getAllCategories(),
    ]);
    
    return (
        <main className="container mx-auto px-4 py-8">
            <h1 className="text-4xl font-bold mb-8">บทความล่าสุด</h1>
            
            {/* Category Filter */}
            <CategoryFilter categories={categories} />
            
            {/* Posts Grid */}
            {posts.length > 0 ? (
                <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6 mt-8">
                    {posts.map(post => (
                        <PostCard
                            key={post.id}
                            title={post.title.rendered}
                            slug={post.slug}
                            excerpt={getExcerpt(post)}
                            imageUrl={getFeaturedImageUrl(post, 'medium')}
                            date={formatDate(post.date)}
                            author={post._embedded?.author?.[0]?.name}
                            categories={post._embedded?.['wp:term']?.[0]?.map(t => t.name) ?? []}
                        />
                    ))}
                </div>
            ) : (
                <p className="text-center text-gray-500 mt-12">ไม่พบบทความ</p>
            )}
            
            {/* Pagination */}
            <Pagination
                currentPage={page}
                totalPages={totalPages}
                total={total}
            />
        </main>
    );
}
```

```typescript
// src/app/posts/[slug]/page.tsx - หน้า Single Post

import { notFound } from 'next/navigation';
import Image from 'next/image';
import { Metadata } from 'next';
import { 
    getAllPostSlugs, 
    getPostBySlug, 
    getFeaturedImageUrl,
    formatDate 
} from '@/lib/wordpress';

interface PostPageProps {
    params: { slug: string };
}

// Static Generation - สร้าง Static Pages สำหรับทุก Post
export async function generateStaticParams() {
    const slugs = await getAllPostSlugs();
    return slugs.map(slug => ({ slug }));
}

// Dynamic Metadata
export async function generateMetadata({ params }: PostPageProps): Promise<Metadata> {
    const post = await getPostBySlug(params.slug);
    
    if (!post) return { title: 'Not Found' };
    
    const imageUrl = getFeaturedImageUrl(post);
    
    return {
        title: post.title.rendered,
        description: post.excerpt.rendered.replace(/<[^>]*>/g, '').substring(0, 160),
        openGraph: {
            title:       post.title.rendered,
            description: post.excerpt.rendered.replace(/<[^>]*>/g, '').substring(0, 160),
            type:        'article',
            publishedTime: post.date,
            modifiedTime:  post.modified,
            images: imageUrl ? [{ url: imageUrl, width: 1200, height: 630 }] : [],
        },
    };
}

export default async function PostPage({ params }: PostPageProps) {
    const post = await getPostBySlug(params.slug);
    
    if (!post) {
        notFound();
    }
    
    const imageUrl = getFeaturedImageUrl(post, 'full');
    const author   = post._embedded?.author?.[0];
    const terms    = post._embedded?.['wp:term'] ?? [];
    const categories = terms[0] ?? [];
    const tags       = terms[1] ?? [];
    
    return (
        <article className="max-w-3xl mx-auto px-4 py-8">
            {/* Article Header */}
            <header className="mb-8">
                <div className="flex gap-2 mb-4">
                    {categories.map(cat => (
                        <span key={cat.id} className="bg-blue-100 text-blue-800 text-sm px-3 py-1 rounded-full">
                            {cat.name}
                        </span>
                    ))}
                </div>
                
                <h1
                    className="text-4xl font-bold mb-4"
                    dangerouslySetInnerHTML={{ __html: post.title.rendered }}
                />
                
                <div className="flex items-center gap-4 text-gray-500">
                    {author && (
                        <div className="flex items-center gap-2">
                            <img
                                src={author.avatar_urls['48']}
                                alt={author.name}
                                className="w-8 h-8 rounded-full"
                            />
                            <span>{author.name}</span>
                        </div>
                    )}
                    <time dateTime={post.date}>{formatDate(post.date)}</time>
                </div>
            </header>
            
            {/* Featured Image */}
            {imageUrl && (
                <div className="relative aspect-video mb-8 rounded-lg overflow-hidden">
                    <Image
                        src={imageUrl}
                        alt={post.title.rendered}
                        fill
                        className="object-cover"
                        priority
                    />
                </div>
            )}
            
            {/* Article Content */}
            <div
                className="prose prose-lg max-w-none"
                dangerouslySetInnerHTML={{ __html: post.content.rendered }}
            />
            
            {/* Tags */}
            {tags.length > 0 && (
                <div className="mt-8 pt-4 border-t">
                    <p className="text-gray-500 mb-2">แท็ก:</p>
                    <div className="flex flex-wrap gap-2">
                        {tags.map(tag => (
                            <span key={tag.id} className="bg-gray-100 text-gray-700 text-sm px-3 py-1 rounded">
                                #{tag.name}
                            </span>
                        ))}
                    </div>
                </div>
            )}
        </article>
    );
}
```

### 664. On-Demand Revalidation Endpoint

```typescript
// src/app/api/revalidate/route.ts

import { revalidatePath, revalidateTag } from 'next/cache';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
    // ตรวจสอบ Secret Token
    const secret = request.headers.get('x-revalidation-secret');
    
    if (secret !== process.env.REVALIDATION_SECRET) {
        return NextResponse.json(
            { error: 'Invalid revalidation secret' },
            { status: 401 }
        );
    }
    
    try {
        const body = await request.json();
        const { type, slug, id } = body;
        
        switch (type) {
            case 'post':
                // Revalidate specific post
                revalidateTag(`post-${slug}`);
                revalidatePath(`/posts/${slug}`);
                // Revalidate home and category pages
                revalidatePath('/');
                revalidatePath('/posts');
                
                console.log(`Revalidated post: ${slug}`);
                break;
                
            case 'category':
                revalidatePath(`/category/${slug}`);
                revalidatePath('/');
                break;
                
            case 'all':
                revalidatePath('/', 'layout');
                break;
                
            default:
                return NextResponse.json(
                    { error: 'Unknown revalidation type' },
                    { status: 400 }
                );
        }
        
        return NextResponse.json({
            revalidated: true,
            timestamp: new Date().toISOString(),
            type,
            slug,
        });
        
    } catch (error) {
        return NextResponse.json(
            { error: 'Revalidation failed', details: String(error) },
            { status: 500 }
        );
    }
}
```

### 665. WordPress Webhook สำหรับ Revalidation

```php
<?php
// WordPress Plugin: Trigger Next.js Revalidation

class Headless_Revalidation {
    
    private $frontend_url;
    private $secret;
    
    public function __construct() {
        $this->frontend_url = get_option('headless_frontend_url', '');
        $this->secret       = get_option('headless_revalidation_secret', '');
        
        add_action('publish_post', [$this, 'revalidate_post'], 10, 2);
        add_action('edit_post',    [$this, 'revalidate_post'], 10, 2);
        add_action('delete_post',  [$this, 'revalidate_post'], 10, 2);
    }
    
    public function revalidate_post($post_id, $post = null) {
        if (!$post) {
            $post = get_post($post_id);
        }
        
        if (!$post || $post->post_type !== 'post') {
            return;
        }
        
        if (!$this->frontend_url || !$this->secret) {
            return;
        }
        
        $payload = [
            'type'  => 'post',
            'slug'  => $post->post_name,
            'id'    => $post_id,
        ];
        
        wp_remote_post("{$this->frontend_url}/api/revalidate", [
            'method'  => 'POST',
            'headers' => [
                'Content-Type'             => 'application/json',
                'X-Revalidation-Secret'    => $this->secret,
            ],
            'body'    => json_encode($payload),
            'timeout' => 15,
        ]);
    }
}

new Headless_Revalidation();
```

### 666. ISR (Incremental Static Regeneration) Patterns

```typescript
// src/app/posts/page.tsx - Posts Listing ด้วย ISR

import { Suspense } from 'react';
import { getAllPosts, getAllCategories } from '@/lib/wordpress';

// กำหนด Revalidation Time
export const revalidate = 300; // 5 นาที

// สร้าง Dynamic OG Image
export async function generateMetadata() {
    return {
        title:       'บทความทั้งหมด',
        description: 'รวบรวมบทความและข่าวสารที่น่าสนใจ',
        openGraph: {
            title:       'บทความทั้งหมด',
            description: 'รวบรวมบทความและข่าวสารที่น่าสนใจ',
        },
    };
}

async function PostsList({ page }: { page: number }) {
    const { posts, total, totalPages } = await getAllPosts({ page, perPage: 12 });
    
    return (
        <div>
            <p className="text-gray-500 mb-4">พบ {total} บทความ</p>
            {/* Render posts */}
        </div>
    );
}

export default function PostsPage({ searchParams }: { searchParams: { page?: string } }) {
    const page = parseInt(searchParams.page ?? '1');
    
    return (
        <Suspense fallback={<div>กำลังโหลด...</div>}>
            <PostsList page={page} />
        </Suspense>
    );
}
```

```typescript
// src/lib/wordpress-swr.ts - Client-side Data Fetching ด้วย SWR

'use client';

import useSWR from 'swr';
import useSWRInfinite from 'swr/infinite';

const fetcher = (url: string) => fetch(url).then(res => res.json());

export function usePosts(params?: { category?: string; search?: string }) {
    const query = new URLSearchParams();
    if (params?.category) query.set('category', params.category);
    if (params?.search)   query.set('search', params.search);
    
    const { data, error, isLoading } = useSWR(
        `/api/posts?${query.toString()}`,
        fetcher,
        {
            revalidateOnFocus: false,
            dedupingInterval:  60000, // 1 นาที
        }
    );
    
    return {
        posts:     data?.posts ?? [],
        total:     data?.total ?? 0,
        isLoading,
        isError:   !!error,
    };
}

export function useInfinitePosts() {
    const getKey = (pageIndex: number, previousPageData: any) => {
        if (previousPageData && !previousPageData.posts?.length) return null;
        return `/api/posts?page=${pageIndex + 1}`;
    };
    
    const { data, error, size, setSize, isValidating } = useSWRInfinite(
        getKey,
        fetcher
    );
    
    const posts    = data ? data.flatMap(d => d.posts ?? []) : [];
    const isLoading = !data && !error;
    const isEmpty  = data?.[0]?.posts?.length === 0;
    const isReachingEnd = isEmpty || (data && data[data.length - 1]?.posts?.length < 12);
    
    return {
        posts,
        isLoading,
        isError:       !!error,
        size,
        setSize,
        isValidating,
        isReachingEnd,
    };
}
```

---

## ขั้นตอนที่ 669-673: WPGraphQL

### 669. ติดตั้งและตั้งค่า WPGraphQL

```bash
# ติดตั้งผ่าน WP-CLI
wp plugin install wp-graphql --activate

# ติดตั้ง Extension เพิ่มเติม
wp plugin install wp-graphql-acf --activate  # สำหรับ ACF
wp plugin install wp-graphql-jwt-authentication --activate  # JWT
```

ตั้งค่า WPGraphQL ใน `wp-config.php`:

```php
<?php
define('GRAPHQL_JWT_AUTH_SECRET_KEY', 'your-graphql-jwt-secret');
```

### 670. GraphQL Queries พื้นฐาน

```graphql
# ดึง Posts ทั้งหมด
query GetPosts {
    posts(first: 10, where: { status: PUBLISH }) {
        nodes {
            id
            title
            slug
            date
            excerpt
            featuredImage {
                node {
                    sourceUrl
                    altText
                    mediaDetails {
                        width
                        height
                    }
                }
            }
            categories {
                nodes {
                    id
                    name
                    slug
                }
            }
            author {
                node {
                    name
                    avatar {
                        url
                    }
                }
            }
        }
        pageInfo {
            hasNextPage
            hasPreviousPage
            startCursor
            endCursor
        }
    }
}

# ดึง Post ด้วย Slug
query GetPostBySlug($slug: ID!) {
    post(id: $slug, idType: SLUG) {
        id
        title
        content
        date
        modified
        slug
        seo {
            title
            metaDesc
            opengraphImage {
                sourceUrl
            }
        }
        featuredImage {
            node {
                sourceUrl(size: LARGE)
                altText
            }
        }
        categories {
            nodes { name slug }
        }
        tags {
            nodes { name slug }
        }
        author {
            node {
                name
                description
                avatar { url }
            }
        }
    }
}

# Cursor-based Pagination
query GetPostsWithCursor($first: Int!, $after: String) {
    posts(first: $first, after: $after, where: { status: PUBLISH }) {
        nodes {
            id
            title
            slug
            excerpt
            date
        }
        pageInfo {
            hasNextPage
            endCursor
        }
    }
}
```

### 671. GraphQL Client ใน Next.js

```typescript
// src/lib/graphql-client.ts

const GRAPHQL_URL = `${process.env.NEXT_PUBLIC_WORDPRESS_URL}/graphql`;

interface GraphQLResponse<T> {
    data: T;
    errors?: Array<{ message: string; locations?: any[] }>;
}

export async function graphqlFetch<T>(
    query: string,
    variables?: Record<string, unknown>,
    options?: RequestInit
): Promise<T> {
    const response = await fetch(GRAPHQL_URL, {
        method:  'POST',
        headers: {
            'Content-Type': 'application/json',
            ...options?.headers,
        },
        body:    JSON.stringify({ query, variables }),
        ...options,
    });
    
    if (!response.ok) {
        throw new Error(`GraphQL request failed: ${response.statusText}`);
    }
    
    const result: GraphQLResponse<T> = await response.json();
    
    if (result.errors?.length) {
        throw new Error(result.errors.map(e => e.message).join(', '));
    }
    
    return result.data;
}

// Typed Queries
const GET_POSTS_QUERY = `
    query GetPosts($first: Int!, $after: String, $categorySlug: String) {
        posts(
            first: $first
            after: $after
            where: { 
                status: PUBLISH
                categoryName: $categorySlug
            }
        ) {
            nodes {
                id databaseId title slug date excerpt
                featuredImage {
                    node { sourceUrl altText }
                }
                categories {
                    nodes { name slug }
                }
                author {
                    node { name avatar { url } }
                }
            }
            pageInfo {
                hasNextPage
                endCursor
            }
        }
    }
`;

export interface GQLPost {
    id:             string;
    databaseId:     number;
    title:          string;
    slug:           string;
    date:           string;
    excerpt:        string;
    featuredImage?: { node: { sourceUrl: string; altText: string } };
    categories:     { nodes: Array<{ name: string; slug: string }> };
    author:         { node: { name: string; avatar: { url: string } } };
}

export async function getPostsGraphQL(params?: {
    first?:        number;
    after?:        string;
    categorySlug?: string;
}) {
    const { first = 12, after, categorySlug } = params ?? {};
    
    const data = await graphqlFetch<{
        posts: {
            nodes:    GQLPost[];
            pageInfo: { hasNextPage: boolean; endCursor: string };
        };
    }>(GET_POSTS_QUERY, { first, after, categorySlug }, {
        next: { revalidate: 300 },
    });
    
    return data.posts;
}
```

### 672. Mutations ด้วย GraphQL (ต้อง Authentication)

```typescript
// src/lib/graphql-mutations.ts

import { graphqlFetch } from './graphql-client';

// Login
const LOGIN_MUTATION = `
    mutation LoginUser($username: String!, $password: String!) {
        login(input: { username: $username, password: $password }) {
            authToken
            refreshToken
            user {
                id
                name
                email
            }
        }
    }
`;

export async function loginWithGraphQL(username: string, password: string) {
    return graphqlFetch<{
        login: {
            authToken:    string;
            refreshToken: string;
            user:         { id: string; name: string; email: string };
        };
    }>(LOGIN_MUTATION, { username, password });
}

// Create Comment
const CREATE_COMMENT_MUTATION = `
    mutation CreateComment(
        $postId: Int!
        $content: String!
        $author: String
        $authorEmail: String
    ) {
        createComment(input: {
            commentOn: $postId
            content: $content
            author: $author
            authorEmail: $authorEmail
        }) {
            comment {
                id
                content
                date
                author {
                    node { name }
                }
            }
        }
    }
`;

export async function createComment(
    authToken: string,
    params: {
        postId:       number;
        content:      string;
        author?:      string;
        authorEmail?: string;
    }
) {
    return graphqlFetch<{
        createComment: {
            comment: {
                id:      string;
                content: string;
                date:    string;
                author:  { node: { name: string } };
            };
        };
    }>(CREATE_COMMENT_MUTATION, params, {
        headers: { Authorization: `Bearer ${authToken}` },
    });
}
```

### 673. Fragment สำหรับ Code Reuse ใน GraphQL

```graphql
# src/lib/graphql-fragments.ts

fragment PostFields on Post {
    id
    databaseId
    title
    slug
    date
    modified
    excerpt
}

fragment FeaturedImageFields on MediaItem {
    sourceUrl
    altText
    mediaDetails {
        width
        height
    }
}

fragment AuthorFields on User {
    name
    description
    avatar { url }
}

fragment CategoryFields on Category {
    id
    name
    slug
}

# ใช้ Fragments ใน Query
query GetFeaturedPosts {
    posts(first: 6, where: { status: PUBLISH, categoryName: "featured" }) {
        nodes {
            ...PostFields
            featuredImage {
                node {
                    ...FeaturedImageFields
                }
            }
            author {
                node {
                    ...AuthorFields
                }
            }
            categories {
                nodes {
                    ...CategoryFields
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 674-677: ACF กับ REST API

### 674. ติดตั้งและตั้งค่า ACF Pro

```bash
# ติดตั้ง ACF Pro (ต้องมี License Key)
wp plugin install advanced-custom-fields-pro --activate
```

สร้าง Field Group ด้วย PHP (แนะนำสำหรับ Version Control):

```php
<?php
// acf-fields.php หรือ functions.php

add_action('acf/init', function() {
    
    // Field Group สำหรับ Product Post Type
    acf_add_local_field_group([
        'key'    => 'group_product_details',
        'title'  => 'Product Details',
        'fields' => [
            [
                'key'   => 'field_product_price',
                'label' => 'Price',
                'name'  => 'price',
                'type'  => 'number',
                'min'   => 0,
                'step'  => 0.01,
            ],
            [
                'key'     => 'field_product_sku',
                'label'   => 'SKU',
                'name'    => 'sku',
                'type'    => 'text',
                'required' => 1,
            ],
            [
                'key'   => 'field_product_gallery',
                'label' => 'Product Gallery',
                'name'  => 'gallery',
                'type'  => 'gallery',
                'return_format' => 'array',
            ],
            [
                'key'        => 'field_product_specifications',
                'label'      => 'Specifications',
                'name'       => 'specifications',
                'type'       => 'repeater',
                'sub_fields' => [
                    [
                        'key'   => 'field_spec_name',
                        'label' => 'Name',
                        'name'  => 'name',
                        'type'  => 'text',
                    ],
                    [
                        'key'   => 'field_spec_value',
                        'label' => 'Value',
                        'name'  => 'value',
                        'type'  => 'text',
                    ],
                ],
            ],
            [
                'key'     => 'field_product_related',
                'label'   => 'Related Products',
                'name'    => 'related_products',
                'type'    => 'relationship',
                'post_type' => ['product'],
                'return_format' => 'id',
            ],
        ],
        'location' => [[
            ['param' => 'post_type', 'operator' => '==', 'value' => 'product'],
        ]],
    ]);
});
```

### 675. Expose ACF Fields ใน REST API

```php
<?php
// เพิ่ม ACF Fields เข้า REST API

function add_acf_to_rest_api() {
    // สำหรับ Product Post Type
    add_filter('rest_prepare_product', function($response, $post, $request) {
        $acf_data = get_fields($post->ID);
        
        if ($acf_data) {
            // ประมวลผล Gallery Images
            if (!empty($acf_data['gallery'])) {
                $acf_data['gallery'] = array_map(function($image) {
                    return [
                        'id'     => $image['ID'],
                        'url'    => $image['url'],
                        'sizes'  => [
                            'thumbnail' => $image['sizes']['thumbnail'],
                            'medium'    => $image['sizes']['medium'],
                            'large'     => $image['sizes']['large'],
                        ],
                        'alt'    => $image['alt'],
                        'title'  => $image['title'],
                    ];
                }, $acf_data['gallery']);
            }
            
            // ประมวลผล Related Products
            if (!empty($acf_data['related_products'])) {
                $related = [];
                foreach ($acf_data['related_products'] as $product_id) {
                    $related_post = get_post($product_id);
                    if ($related_post) {
                        $related[] = [
                            'id'        => $product_id,
                            'title'     => $related_post->post_title,
                            'slug'      => $related_post->post_name,
                            'thumbnail' => get_the_post_thumbnail_url($product_id, 'medium'),
                            'price'     => (float) get_field('price', $product_id),
                        ];
                    }
                }
                $acf_data['related_products'] = $related;
            }
            
            $data = $response->get_data();
            $data['acf'] = $acf_data;
            $response->set_data($data);
        }
        
        return $response;
    }, 10, 3);
}
add_action('rest_api_init', 'add_acf_to_rest_api');
```

### 676. ACF กับ WPGraphQL

```php
<?php
// ตั้งค่า ACF Field Groups สำหรับ GraphQL

add_action('acf/init', function() {
    acf_add_local_field_group([
        'key'        => 'group_product_details',
        'title'      => 'Product Details',
        'graphql_field_name' => 'productDetails', // ชื่อสำหรับ GraphQL
        'show_in_graphql' => 1, // เปิดใช้ GraphQL
        'fields'     => [
            [
                'key'              => 'field_product_price',
                'label'            => 'Price',
                'name'             => 'price',
                'type'             => 'number',
                'show_in_graphql'  => 1,
                'graphql_field_name' => 'price',
            ],
            // ... other fields
        ],
        'location' => [[
            ['param' => 'post_type', 'operator' => '==', 'value' => 'product'],
        ]],
    ]);
});
```

```graphql
# Query สำหรับ ACF Fields ผ่าน GraphQL

query GetProductWithACF($slug: ID!) {
    product(id: $slug, idType: SLUG) {
        id
        title
        content
        productDetails {
            price
            sku
            gallery {
                id
                sourceUrl
                altText
                mediaDetails {
                    width
                    height
                }
            }
            specifications {
                name
                value
            }
            relatedProducts {
                nodes {
                    id
                    title
                    slug
                    productDetails {
                        price
                    }
                    featuredImage {
                        node { sourceUrl }
                    }
                }
            }
        }
    }
}
```

### 677. ACF Options Page สำหรับ Site Settings

```php
<?php
// สร้าง Options Page สำหรับ Site-wide Settings

if (function_exists('acf_add_options_page')) {
    
    acf_add_options_page([
        'page_title' => 'Site Settings',
        'menu_title' => 'Site Settings',
        'menu_slug'  => 'site-settings',
        'capability' => 'edit_posts',
        'icon_url'   => 'dashicons-admin-settings',
    ]);
    
    acf_add_options_sub_page([
        'page_title'  => 'Header Settings',
        'menu_title'  => 'Header',
        'parent_slug' => 'site-settings',
    ]);
    
    acf_add_options_sub_page([
        'page_title'  => 'Footer Settings',
        'menu_title'  => 'Footer',
        'parent_slug' => 'site-settings',
    ]);
}

// ลงทะเบียน Fields สำหรับ Options Page
add_action('acf/init', function() {
    acf_add_local_field_group([
        'key'             => 'group_site_settings',
        'title'           => 'Global Site Settings',
        'show_in_graphql' => 1,
        'graphql_field_name' => 'siteSettings',
        'fields'          => [
            [
                'key'              => 'field_site_logo',
                'label'            => 'Site Logo',
                'name'             => 'site_logo',
                'type'             => 'image',
                'show_in_graphql'  => 1,
                'graphql_field_name' => 'siteLogo',
                'return_format'    => 'array',
            ],
            [
                'key'              => 'field_footer_text',
                'label'            => 'Footer Text',
                'name'             => 'footer_text',
                'type'             => 'text',
                'show_in_graphql'  => 1,
                'graphql_field_name' => 'footerText',
            ],
            [
                'key'              => 'field_social_links',
                'label'            => 'Social Links',
                'name'             => 'social_links',
                'type'             => 'group',
                'show_in_graphql'  => 1,
                'graphql_field_name' => 'socialLinks',
                'sub_fields'       => [
                    [
                        'key'   => 'field_facebook_url',
                        'label' => 'Facebook',
                        'name'  => 'facebook',
                        'type'  => 'url',
                        'show_in_graphql' => 1,
                    ],
                    [
                        'key'   => 'field_twitter_url',
                        'label' => 'Twitter/X',
                        'name'  => 'twitter',
                        'type'  => 'url',
                        'show_in_graphql' => 1,
                    ],
                    [
                        'key'   => 'field_instagram_url',
                        'label' => 'Instagram',
                        'name'  => 'instagram',
                        'type'  => 'url',
                        'show_in_graphql' => 1,
                    ],
                ],
            ],
        ],
        'location' => [[
            ['param' => 'options_page', 'operator' => '==', 'value' => 'site-settings'],
        ]],
    ]);
});

// Expose Options ผ่าน REST API
add_action('rest_api_init', function() {
    register_rest_route('myapi/v1', '/site-settings', [
        'methods'             => 'GET',
        'callback'            => function() {
            return [
                'logo'         => acf_get_field('site_logo', 'option'),
                'footer_text'  => get_field('footer_text', 'option'),
                'social_links' => get_field('social_links', 'option'),
            ];
        },
        'permission_callback' => '__return_true',
    ]);
});
```

---

## ขั้นตอนที่ 678-680: Deployment

### 678. Deploy Next.js บน Vercel

```json
// vercel.json - การตั้งค่า Vercel

{
    "buildCommand": "next build",
    "outputDirectory": ".next",
    "devCommand": "next dev",
    "framework": "nextjs",
    "regions": ["sin1"],
    "env": {
        "NEXT_PUBLIC_WORDPRESS_URL": "@wordpress_url",
        "WORDPRESS_API_URL": "@wordpress_api_url",
        "REVALIDATION_SECRET": "@revalidation_secret"
    },
    "headers": [
        {
            "source": "/api/(.*)",
            "headers": [
                { "key": "X-Content-Type-Options", "value": "nosniff" },
                { "key": "X-Frame-Options", "value": "DENY" },
                { "key": "X-XSS-Protection", "value": "1; mode=block" }
            ]
        }
    ],
    "rewrites": [
        {
            "source": "/sitemap.xml",
            "destination": "/api/sitemap"
        }
    ]
}
```

```bash
# Deploy ด้วย Vercel CLI

# ติดตั้ง Vercel CLI
npm install -g vercel

# Login
vercel login

# Deploy Development
vercel

# Deploy Production
vercel --prod

# ตั้งค่า Environment Variables
vercel env add NEXT_PUBLIC_WORDPRESS_URL production
vercel env add REVALIDATION_SECRET production

# ดู Deployment Logs
vercel logs
```

### 679. WordPress Hosting Configuration

```nginx
# Nginx Configuration สำหรับ WordPress (Headless)

server {
    listen 80;
    server_name api.your-domain.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.your-domain.com;
    
    # SSL Certificate
    ssl_certificate     /etc/letsencrypt/live/api.your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.your-domain.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    
    root /var/www/html/wordpress;
    index index.php;
    
    # Security Headers
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options nosniff;
    add_header Referrer-Policy strict-origin-when-cross-origin;
    
    # CORS Headers สำหรับ REST API
    location ~* ^/wp-json/ {
        add_header Access-Control-Allow-Origin https://your-nextjs-app.vercel.app;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
        add_header Access-Control-Allow-Headers "Authorization, Content-Type, X-WP-Nonce";
        
        if ($request_method = 'OPTIONS') {
            add_header Access-Control-Max-Age 1728000;
            add_header Content-Type 'text/plain; charset=utf-8';
            add_header Content-Length 0;
            return 204;
        }
        
        try_files $uri $uri/ /index.php?$args;
    }
    
    # Disable WordPress Frontend (Headless Mode)
    location / {
        # อนุญาตเฉพาะ wp-admin, wp-json, wp-login
        if ($request_uri !~* "^/(wp-admin|wp-json|wp-login|wp-cron|graphql|xmlrpc.php|wp-content|wp-includes)") {
            return 404;
        }
        
        try_files $uri $uri/ /index.php?$args;
    }
    
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PHP_VALUE "upload_max_filesize = 64M \n post_max_size=64M";
        fastcgi_read_timeout 300;
    }
    
    # Cache Static Files
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|pdf|txt|woff|woff2|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
    
    # Block access to sensitive files
    location ~* /(wp-config\.php|\.htaccess|readme\.html|license\.txt) {
        deny all;
        return 404;
    }
}
```

```bash
# .github/workflows/deploy.yml - CI/CD Pipeline

name: Deploy Headless WordPress Site

on:
    push:
        branches: [main]
    pull_request:
        branches: [main]

jobs:
    test-frontend:
        runs-on: ubuntu-latest
        defaults:
            run:
                working-directory: ./frontend
        
        steps:
            - uses: actions/checkout@v4
            
            - name: Setup Node.js
              uses: actions/setup-node@v4
              with:
                  node-version: '20'
                  cache: 'npm'
            
            - name: Install dependencies
              run: npm ci
            
            - name: Run linter
              run: npm run lint
            
            - name: Type check
              run: npm run type-check
            
            - name: Run tests
              run: npm test
            
            - name: Build
              run: npm run build
              env:
                  NEXT_PUBLIC_WORDPRESS_URL: ${{ secrets.WORDPRESS_URL }}
    
    deploy-frontend:
        needs: test-frontend
        runs-on: ubuntu-latest
        if: github.ref == 'refs/heads/main'
        
        steps:
            - uses: actions/checkout@v4
            
            - name: Deploy to Vercel
              uses: amondnet/vercel-action@v25
              with:
                  vercel-token:   ${{ secrets.VERCEL_TOKEN }}
                  vercel-org-id:  ${{ secrets.ORG_ID }}
                  vercel-project-id: ${{ secrets.PROJECT_ID }}
                  working-directory: ./frontend
                  vercel-args: '--prod'
    
    deploy-wordpress:
        needs: test-frontend
        runs-on: ubuntu-latest
        if: github.ref == 'refs/heads/main'
        
        steps:
            - uses: actions/checkout@v4
            
            - name: Deploy WordPress to Server
              uses: easingthemes/ssh-deploy@main
              env:
                  SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
                  ARGS: "-rlgoDzvc -i --delete"
                  SOURCE: "./wordpress/"
                  REMOTE_HOST: ${{ secrets.REMOTE_HOST }}
                  REMOTE_USER: ${{ secrets.REMOTE_USER }}
                  TARGET: "/var/www/html/wordpress/"
                  EXCLUDE: "/wp-config.php, /wp-content/uploads/"
```

### 680. Monitoring และ Performance

```typescript
// src/lib/monitoring.ts - Performance Monitoring

export function measurePageLoad(pageName: string) {
    if (typeof window === 'undefined') return;
    
    // Web Vitals
    const observer = new PerformanceObserver((list) => {
        for (const entry of list.getEntries()) {
            const metric = {
                name:  entry.name,
                value: entry.startTime,
                page:  pageName,
            };
            
            // ส่ง Metric ไปยัง Analytics Service
            sendMetric(metric);
        }
    });
    
    observer.observe({ entryTypes: ['largest-contentful-paint', 'first-input', 'layout-shift'] });
}

function sendMetric(metric: { name: string; value: number; page: string }) {
    fetch('/api/metrics', {
        method:  'POST',
        headers: { 'Content-Type': 'application/json' },
        body:    JSON.stringify(metric),
        keepalive: true,
    });
}

// Sitemap Generator
// src/app/api/sitemap/route.ts
import { getAllPostSlugs, getAllCategories } from '@/lib/wordpress';

export async function GET() {
    const [slugs, categories] = await Promise.all([
        getAllPostSlugs(),
        getAllCategories(),
    ]);
    
    const baseUrl = process.env.NEXT_PUBLIC_SITE_URL ?? 'https://your-site.com';
    
    const sitemap = `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
    <url>
        <loc>${baseUrl}</loc>
        <changefreq>daily</changefreq>
        <priority>1.0</priority>
    </url>
    ${slugs.map(slug => `
    <url>
        <loc>${baseUrl}/posts/${slug}</loc>
        <changefreq>weekly</changefreq>
        <priority>0.8</priority>
    </url>`).join('')}
    ${categories.map(cat => `
    <url>
        <loc>${baseUrl}/category/${cat.slug}</loc>
        <changefreq>daily</changefreq>
        <priority>0.6</priority>
    </url>`).join('')}
</urlset>`;
    
    return new Response(sitemap, {
        headers: {
            'Content-Type': 'application/xml',
            'Cache-Control': 'public, max-age=3600',
        },
    });
}
```

---

## Workshop: สร้าง Headless Blog ครบระบบ

### โจทย์

สร้าง Headless Blog ที่ประกอบด้วย:
1. WordPress Backend พร้อม Custom Post Type "Article" และ ACF Fields
2. Next.js Frontend ที่ดึงข้อมูลจาก WordPress REST API
3. ระบบ Authentication สำหรับ Comment
4. On-Demand Revalidation เมื่อมีการอัพเดต Article
5. Sitemap อัตโนมัติ

### โครงสร้างโปรเจกต์

```
headless-blog/
├── wordpress/              # WordPress Backend
│   ├── wp-content/
│   │   └── plugins/
│   │       └── headless-blog-api/
│   │           ├── headless-blog-api.php
│   │           ├── includes/
│   │           │   ├── class-article-post-type.php
│   │           │   ├── class-rest-controller.php
│   │           │   ├── class-jwt-auth.php
│   │           │   └── class-revalidation.php
│   │           └── acf-fields/
│   │               └── article-fields.php
└── frontend/               # Next.js Frontend
    ├── src/
    │   ├── app/
    │   │   ├── layout.tsx
    │   │   ├── page.tsx
    │   │   ├── articles/
    │   │   │   ├── page.tsx
    │   │   │   └── [slug]/
    │   │   │       └── page.tsx
    │   │   └── api/
    │   │       ├── revalidate/
    │   │       │   └── route.ts
    │   │       └── sitemap/
    │   │           └── route.ts
    │   ├── components/
    │   │   ├── ArticleCard.tsx
    │   │   ├── ArticleContent.tsx
    │   │   ├── CommentForm.tsx
    │   │   └── Navigation.tsx
    │   └── lib/
    │       ├── wordpress.ts
    │       ├── graphql-client.ts
    │       └── auth.ts
    ├── next.config.ts
    └── .env.local
```

### ขั้นตอนการทำ Workshop

**ขั้นตอนที่ 1: สร้าง WordPress Plugin**

```php
<?php
/**
 * Plugin Name: Headless Blog API
 * Description: Custom API for Headless Blog
 * Version: 1.0.0
 */

// headless-blog-api.php

if (!defined('ABSPATH')) exit;

define('HEADLESS_BLOG_VERSION', '1.0.0');
define('HEADLESS_BLOG_PATH',    plugin_dir_path(__FILE__));
define('HEADLESS_BLOG_URL',     plugin_dir_url(__FILE__));

require_once HEADLESS_BLOG_PATH . 'includes/class-article-post-type.php';
require_once HEADLESS_BLOG_PATH . 'includes/class-rest-controller.php';
require_once HEADLESS_BLOG_PATH . 'includes/class-jwt-auth.php';
require_once HEADLESS_BLOG_PATH . 'includes/class-revalidation.php';

function headless_blog_init() {
    new Article_Post_Type();
    new Article_REST_Controller();
    new Headless_JWT_Auth();
    new Headless_Revalidation();
}
add_action('plugins_loaded', 'headless_blog_init');
```

**ขั้นตอนที่ 2: สร้าง Article Post Type**

```php
<?php
// includes/class-article-post-type.php

class Article_Post_Type {
    
    public function __construct() {
        add_action('init', [$this, 'register']);
        add_action('acf/init', [$this, 'register_fields']);
    }
    
    public function register() {
        register_post_type('article', [
            'public'       => true,
            'label'        => 'Articles',
            'show_in_rest' => true,
            'rest_base'    => 'articles',
            'supports'     => ['title', 'editor', 'thumbnail', 'excerpt', 'author'],
            'has_archive'  => true,
            'rewrite'      => ['slug' => 'articles'],
        ]);
        
        register_taxonomy('article_topic', ['article'], [
            'hierarchical' => true,
            'label'        => 'Topics',
            'show_in_rest' => true,
            'rest_base'    => 'topics',
            'rewrite'      => ['slug' => 'topic'],
        ]);
    }
    
    public function register_fields() {
        acf_add_local_field_group([
            'key'             => 'group_article',
            'title'           => 'Article Details',
            'show_in_graphql' => 1,
            'graphql_field_name' => 'articleDetails',
            'fields'          => [
                [
                    'key'   => 'field_reading_time',
                    'label' => 'Reading Time (minutes)',
                    'name'  => 'reading_time',
                    'type'  => 'number',
                    'show_in_graphql' => 1,
                ],
                [
                    'key'    => 'field_is_featured',
                    'label'  => 'Featured Article',
                    'name'   => 'is_featured',
                    'type'   => 'true_false',
                    'show_in_graphql' => 1,
                ],
                [
                    'key'    => 'field_seo_description',
                    'label'  => 'SEO Description',
                    'name'   => 'seo_description',
                    'type'   => 'textarea',
                    'rows'   => 3,
                    'show_in_graphql' => 1,
                ],
            ],
            'location' => [[
                ['param' => 'post_type', 'operator' => '==', 'value' => 'article'],
            ]],
        ]);
    }
}
```

**ขั้นตอนที่ 3: สร้าง Next.js Component สำหรับ Article**

```typescript
// src/components/ArticleCard.tsx

import Link from 'next/link';
import Image from 'next/image';

interface ArticleCardProps {
    id:          number;
    title:       string;
    slug:        string;
    excerpt:     string;
    imageUrl:    string | null;
    date:        string;
    author:      string;
    topics:      string[];
    readingTime: number;
    isFeatured:  boolean;
}

export default function ArticleCard({
    title,
    slug,
    excerpt,
    imageUrl,
    date,
    author,
    topics,
    readingTime,
    isFeatured,
}: ArticleCardProps) {
    return (
        <article className={`bg-white rounded-xl shadow-md overflow-hidden ${isFeatured ? 'border-2 border-blue-500' : ''}`}>
            {/* Featured Badge */}
            {isFeatured && (
                <div className="bg-blue-500 text-white text-xs font-bold px-3 py-1 text-center">
                    บทความแนะนำ
                </div>
            )}
            
            {/* Thumbnail */}
            {imageUrl && (
                <div className="relative aspect-video">
                    <Image
                        src={imageUrl}
                        alt={title}
                        fill
                        className="object-cover"
                        sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
                    />
                </div>
            )}
            
            <div className="p-6">
                {/* Topics */}
                <div className="flex flex-wrap gap-2 mb-3">
                    {topics.map(topic => (
                        <span key={topic} className="bg-gray-100 text-gray-600 text-xs px-2 py-1 rounded">
                            {topic}
                        </span>
                    ))}
                </div>
                
                {/* Title */}
                <h2 className="text-xl font-bold mb-2 line-clamp-2">
                    <Link href={`/articles/${slug}`} className="hover:text-blue-600">
                        {title}
                    </Link>
                </h2>
                
                {/* Excerpt */}
                <p className="text-gray-600 text-sm line-clamp-3 mb-4">{excerpt}</p>
                
                {/* Meta */}
                <div className="flex items-center justify-between text-sm text-gray-500">
                    <div className="flex items-center gap-2">
                        <span>{author}</span>
                        <span>•</span>
                        <time>{date}</time>
                    </div>
                    <span>{readingTime} นาที</span>
                </div>
            </div>
        </article>
    );
}
```

---

## แบบทดสอบ (Quiz)

### คำถามที่ 1

**Headless CMS แตกต่างจาก Traditional CMS อย่างไร?**

ก) Headless CMS ไม่มี Admin Panel ให้ใช้งาน  
ข) Headless CMS แยก Frontend และ Backend ออกจากกัน โดย Backend ส่งข้อมูลในรูปแบบ JSON ผ่าน API  
ค) Headless CMS ใช้เฉพาะ Next.js เท่านั้น  
ง) Headless CMS ช้ากว่า Traditional CMS เสมอ  

**เฉลย: ข**

อธิบาย: Headless CMS (เช่น WordPress Headless) แยก Content Management (Backend) ออกจาก Presentation Layer (Frontend) โดยสิ้นเชิง Backend จัดการเนื้อหาผ่าน Admin Panel และส่งข้อมูลเป็น JSON ผ่าน REST API หรือ GraphQL ส่วน Frontend ใช้ Framework ใดก็ได้ (Next.js, Nuxt, Gatsby, Svelte) เพื่อ Render ข้อมูลเหล่านั้น

---

### คำถามที่ 2

**JWT Token มีโครงสร้างอย่างไร?**

ก) แบ่งเป็น 2 ส่วน คือ Header และ Payload  
ข) แบ่งเป็น 3 ส่วน คือ Header, Payload และ Signature คั่นด้วย `.`  
ค) เป็น String ธรรมดาที่เข้ารหัสด้วย Base64  
ง) เป็น XML Document ที่เข้ารหัส  

**เฉลย: ข**

อธิบาย: JWT (JSON Web Token) ประกอบด้วย 3 ส่วนคั่นด้วยจุด (.) ได้แก่:
- **Header**: ระบุ Algorithm และ Token Type เช่น `{"alg": "HS256", "typ": "JWT"}`
- **Payload**: ข้อมูล Claims เช่น `{"sub": 1, "exp": 1234567890, "iss": "https://example.com"}`
- **Signature**: Hash ของ Header + Payload + Secret Key เพื่อยืนยันความถูกต้อง

แต่ละส่วนถูก Encode ด้วย Base64URL ทำให้ดูเหมือน `xxxxx.yyyyy.zzzzz`

---

### คำถามที่ 3

**ISR (Incremental Static Regeneration) ใน Next.js คืออะไร?**

ก) การสร้าง Static HTML ทุกครั้งที่มี Request ใหม่  
ข) การ Render ทุกอย่างฝั่ง Client เพื่อความเร็ว  
ค) การสร้าง Static Pages และอัพเดตเฉพาะหน้าที่จำเป็นในระหว่างที่ Site กำลังทำงาน โดยไม่ต้อง Rebuild ทั้งหมด  
ง) การใช้ Redis Cache เพื่อ Store ผลลัพธ์  

**เฉลย: ค**

อธิบาย: ISR ช่วยให้ Next.js สร้าง Static Pages ตอน Build Time และอัพเดตหน้าเหล่านั้นได้ใน Background หลังจากที่ Site Deploy แล้ว โดยกำหนด `revalidate` เป็นจำนวนวินาที เช่น `{ next: { revalidate: 300 } }` Next.js จะส่ง Static Page ที่ Cache ไว้ให้ User ทันที แต่ใน Background จะ Fetch ข้อมูลใหม่จาก WordPress ทุก 5 นาที และอัพเดต Cache สำหรับ Request ถัดไป

---

### คำถามที่ 4

**WPGraphQL Plugin ให้ประโยชน์อะไรเหนือกว่า REST API?**

ก) WPGraphQL เร็วกว่า REST API เสมอ  
ข) WPGraphQL ให้ Client ระบุได้ว่าต้องการ Field อะไรบ้างใน Query เดียว ลด Over-fetching และ Under-fetching  
ค) WPGraphQL รองรับ Authentication ส่วน REST API ไม่รองรับ  
ง) WPGraphQL ใช้ JSON ส่วน REST API ใช้ XML  

**เฉลย: ข**

อธิบาย: GraphQL แก้ปัญหา 2 อย่างหลักของ REST API:
- **Over-fetching**: REST API ส่งข้อมูลมากเกินความจำเป็น (เช่น ต้องการแค่ title และ slug แต่ได้ทั้ง Post Object)
- **Under-fetching**: ต้องทำหลาย Request เพื่อได้ข้อมูลครบ (เช่น ต้องเรียก /posts แล้วเรียก /users/1 แยกกัน)

GraphQL ให้ Client เลือก Field ที่ต้องการได้เอง และ Fetch ข้อมูลจากหลาย Resource ใน Query เดียว

---

### คำถามที่ 5

**On-Demand Revalidation ใน Next.js ทำงานอย่างไร?**

ก) หน้าเว็บจะ Rebuild ทั้งหมดทุกครั้งที่มีการอัพเดตใน WordPress  
ข) Next.js จะ Fetch ข้อมูลใหม่ทุก 60 วินาทีโดยอัตโนมัติ  
ค) WordPress เรียก Webhook ไปยัง Next.js API เมื่อมีการ Publish/Update Post และ Next.js จะ Invalidate Cache เฉพาะหน้าที่เกี่ยวข้อง  
ง) User ต้อง Refresh Browser เพื่อดูข้อมูลใหม่  

**เฉลย: ค**

อธิบาย: On-Demand Revalidation ทำงานด้วย `revalidatePath()` หรือ `revalidateTag()` ใน Next.js 13+:
1. WordPress มี Plugin/Hook ที่ส่ง POST Request ไปยัง `/api/revalidate` ของ Next.js เมื่อมีการ Publish หรือ Update Post
2. Next.js API Route ตรวจสอบ Secret Token ก่อน จากนั้นเรียก `revalidatePath('/posts/[slug]')` หรือ `revalidateTag('post-my-slug')`
3. Next.js จะ Re-fetch ข้อมูลจาก WordPress และอัพเดต Cache เฉพาะหน้านั้น
4. User คนต่อไปที่เข้าหน้านั้นจะได้รับข้อมูลใหม่ทันที

---

## สรุปบทเรียน

ในบทเรียนนี้เราได้เรียนรู้:

| หัวข้อ | สิ่งที่เรียน |
|--------|------------|
| Headless Architecture | การแยก Frontend/Backend, ข้อดี-ข้อเสีย, Use Cases |
| WordPress REST API | Endpoints, Custom Post Types, Custom Fields, Custom Controllers |
| JWT Authentication | Token Structure, Login Flow, Refresh Tokens, Security |
| Next.js Integration | App Router, ISR, On-Demand Revalidation, TypeScript Types |
| WPGraphQL | Queries, Mutations, Fragments, Cursor Pagination |
| ACF REST API | Field Groups, expose to REST/GraphQL, Options Pages |
| Deployment | Vercel, Nginx Config, CI/CD Pipeline, Monitoring |

### แนะนำการพัฒนาต่อ

1. **Performance**: ใช้ Redis สำหรับ API Caching บน WordPress
2. **Search**: ติดตั้ง ElasticSearch + ElasticPress Plugin
3. **Preview Mode**: ตั้งค่า Next.js Draft Mode สำหรับ Preview Post ก่อน Publish
4. **Image Optimization**: ใช้ Next.js `<Image>` component กับ WordPress Media
5. **Multi-language**: ติดตั้ง Polylang/WPML และสร้าง i18n ใน Next.js
6. **Commerce**: เชื่อมต่อ WooCommerce REST API สำหรับ E-commerce

### Resources เพิ่มเติม

- [WordPress REST API Handbook](https://developer.wordpress.org/rest-api/)
- [WPGraphQL Documentation](https://www.wpgraphql.com/docs)
- [Next.js Documentation](https://nextjs.org/docs)
- [ACF Documentation](https://www.advancedcustomfields.com/resources/)
- [JWT.io](https://jwt.io/) - JWT Debugger

---

*Part 062 | ระดับมืออาชีพ | WordPress Headless CMS*
