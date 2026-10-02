# Part 057: WordPress REST API

**ระดับ:** Advanced  
**เวลาเรียน:** 5-6 ชั่วโมง  
**Prerequisites:** Part 056 (WordPress Custom Fields)

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. ใช้งาน WordPress REST API พื้นฐาน
2. สร้าง Custom REST Endpoints
3. จัดการ Authentication ใน REST API
4. ทำ CRUD Operations ผ่าน API
5. Extend Default Endpoints

---

## 1. WordPress REST API พื้นฐาน

```
Base URL: https://yoursite.com/wp-json/
WordPress Core: https://yoursite.com/wp-json/wp/v2/

Endpoints หลัก:
GET    /wp/v2/posts          - ดึง Posts ทั้งหมด
GET    /wp/v2/posts/{id}     - ดึง Post เดียว
POST   /wp/v2/posts          - สร้าง Post ใหม่
PUT    /wp/v2/posts/{id}     - อัปเดต Post
DELETE /wp/v2/posts/{id}     - ลบ Post

GET    /wp/v2/pages          - ดึง Pages
GET    /wp/v2/categories     - ดึง Categories
GET    /wp/v2/tags           - ดึง Tags
GET    /wp/v2/users          - ดึง Users
GET    /wp/v2/media          - ดึง Media Files
GET    /wp/v2/types          - ดึง Post Types
GET    /wp/v2/taxonomies     - ดึง Taxonomies
```

### 1.1 ทดสอบ REST API

```bash
# ดึง Posts ทั้งหมด
curl https://yoursite.com/wp-json/wp/v2/posts

# ดึงพร้อม Parameters
curl "https://yoursite.com/wp-json/wp/v2/posts?per_page=5&page=1&_embed"

# ค้นหา
curl "https://yoursite.com/wp-json/wp/v2/posts?search=wordpress"

# ดึง Custom Post Type
curl "https://yoursite.com/wp-json/wp/v2/books"

# ดึงพร้อม Authentication (Basic Auth - Development Only)
curl -u admin:password https://yoursite.com/wp-json/wp/v2/posts

# ดึงด้วย JWT Token
curl -H "Authorization: Bearer YOUR_JWT_TOKEN" https://yoursite.com/wp-json/wp/v2/posts
```

### 1.2 REST API Parameters ที่ใช้บ่อย

```javascript
// JavaScript Fetch API
const response = await fetch('https://yoursite.com/wp-json/wp/v2/posts?' + new URLSearchParams({
    per_page: 10,          // จำนวนต่อหน้า (max 100)
    page: 1,               // หน้าที่
    search: 'keyword',     // ค้นหา
    categories: '5,10',    // Filter ตาม Category ID
    tags: '2,3',           // Filter ตาม Tag ID
    author: 1,             // Filter ตาม Author
    orderby: 'date',       // เรียงตาม: date, id, title, modified
    order: 'desc',         // asc หรือ desc
    status: 'publish',     // publish, draft, private
    _embed: true,          // Include embedded data (featured_media, author, etc.)
    _fields: 'id,title,excerpt,link', // เลือก Fields ที่ต้องการ
}));

const posts = await response.json();

// ดึงจำนวน Total จาก Headers
const totalPosts = response.headers.get('X-WP-Total');
const totalPages = response.headers.get('X-WP-TotalPages');
```

---

## 2. Custom REST Endpoints

```php
<?php
/**
 * สร้าง Custom REST API Endpoints
 */

// ==========================================
// Simple Custom Endpoint
// ==========================================

add_action( 'rest_api_init', function() {
    
    // GET /wp-json/my-plugin/v1/stats
    register_rest_route(
        'my-plugin/v1',        // Namespace
        '/stats',              // Route
        array(
            'methods'             => WP_REST_Server::READABLE, // GET
            'callback'            => 'get_site_stats',
            'permission_callback' => '__return_true', // ไม่ต้อง Auth
        )
    );
    
} );

function get_site_stats( WP_REST_Request $request ) {
    
    return array(
        'total_posts' => wp_count_posts()->publish,
        'total_pages' => wp_count_posts('page')->publish,
        'total_users' => count_users()['total_users'],
        'total_comments' => wp_count_comments()->approved,
        'site_name'   => get_bloginfo('name'),
        'wp_version'  => get_bloginfo('version'),
    );
}

// ==========================================
// Full CRUD Endpoint (Books API)
// ==========================================

add_action( 'rest_api_init', function() {
    
    $namespace = 'bookstore/v1';
    
    // List and Create
    register_rest_route( $namespace, '/books', array(
        array(
            'methods'             => WP_REST_Server::READABLE,
            'callback'            => 'api_get_books',
            'permission_callback' => '__return_true',
            'args'                => array(
                'per_page' => array(
                    'default'           => 10,
                    'sanitize_callback' => 'absint',
                    'validate_callback' => function($value) {
                        return $value > 0 && $value <= 100;
                    },
                ),
                'page' => array(
                    'default'           => 1,
                    'sanitize_callback' => 'absint',
                ),
                'search' => array(
                    'sanitize_callback' => 'sanitize_text_field',
                ),
                'genre' => array(
                    'sanitize_callback' => 'sanitize_text_field',
                ),
            ),
        ),
        array(
            'methods'             => WP_REST_Server::CREATABLE,
            'callback'            => 'api_create_book',
            'permission_callback' => function() {
                return current_user_can('publish_posts');
            },
            'args'                => array(
                'title' => array(
                    'required'          => true,
                    'sanitize_callback' => 'sanitize_text_field',
                    'validate_callback' => function($value) {
                        return ! empty(trim($value));
                    },
                ),
                'author' => array(
                    'required'          => true,
                    'sanitize_callback' => 'sanitize_text_field',
                ),
                'price' => array(
                    'required'          => true,
                    'validate_callback' => function($value) {
                        return is_numeric($value) && $value >= 0;
                    },
                    'sanitize_callback' => function($value) {
                        return floatval($value);
                    },
                ),
                'content' => array(
                    'sanitize_callback' => 'wp_kses_post',
                ),
                'genre_ids' => array(
                    'type'    => 'array',
                    'items'   => array( 'type' => 'integer' ),
                ),
            ),
        ),
    ) );
    
    // Single Book
    register_rest_route( $namespace, '/books/(?P<id>\d+)', array(
        array(
            'methods'             => WP_REST_Server::READABLE,
            'callback'            => 'api_get_book',
            'permission_callback' => '__return_true',
            'args'                => array(
                'id' => array(
                    'validate_callback' => function($param) {
                        return is_numeric($param);
                    },
                ),
            ),
        ),
        array(
            'methods'             => WP_REST_Server::EDITABLE,
            'callback'            => 'api_update_book',
            'permission_callback' => function( $request ) {
                return current_user_can('edit_post', $request['id']);
            },
        ),
        array(
            'methods'             => WP_REST_Server::DELETABLE,
            'callback'            => 'api_delete_book',
            'permission_callback' => function( $request ) {
                return current_user_can('delete_post', $request['id']);
            },
        ),
    ) );
    
    // Book Search
    register_rest_route( $namespace, '/books/search', array(
        'methods'             => WP_REST_Server::READABLE,
        'callback'            => 'api_search_books',
        'permission_callback' => '__return_true',
        'args'                => array(
            'q' => array(
                'required'          => true,
                'sanitize_callback' => 'sanitize_text_field',
            ),
        ),
    ) );
    
} );

// ==========================================
// Callback Functions
// ==========================================

function api_get_books( WP_REST_Request $request ) {
    
    $per_page = $request->get_param('per_page') ?: 10;
    $page     = $request->get_param('page') ?: 1;
    $search   = $request->get_param('search');
    $genre    = $request->get_param('genre');
    
    $args = array(
        'post_type'      => 'book',
        'posts_per_page' => $per_page,
        'paged'          => $page,
        'post_status'    => 'publish',
    );
    
    if ( $search ) {
        $args['s'] = $search;
    }
    
    if ( $genre ) {
        $args['tax_query'] = array(
            array(
                'taxonomy' => 'genre',
                'field'    => 'slug',
                'terms'    => $genre,
            ),
        );
    }
    
    $query  = new WP_Query($args);
    $books  = array();
    
    foreach ($query->posts as $post) {
        $books[] = format_book_for_api($post);
    }
    
    $response = new WP_REST_Response($books, 200);
    $response->header('X-WP-Total', $query->found_posts);
    $response->header('X-WP-TotalPages', $query->max_num_pages);
    
    return $response;
}

function api_get_book( WP_REST_Request $request ) {
    
    $id   = (int) $request['id'];
    $post = get_post($id);
    
    if ( ! $post || $post->post_type !== 'book' || $post->post_status !== 'publish' ) {
        return new WP_Error(
            'book_not_found',
            __('Book not found', 'my-plugin'),
            array('status' => 404)
        );
    }
    
    return new WP_REST_Response(format_book_for_api($post), 200);
}

function api_create_book( WP_REST_Request $request ) {
    
    $title   = $request->get_param('title');
    $content = $request->get_param('content') ?: '';
    $author  = $request->get_param('author');
    $price   = $request->get_param('price');
    $genre_ids = $request->get_param('genre_ids') ?: array();
    
    // สร้าง Post
    $post_id = wp_insert_post(array(
        'post_title'   => $title,
        'post_content' => $content,
        'post_type'    => 'book',
        'post_status'  => 'publish',
        'post_author'  => get_current_user_id(),
    ), true);
    
    if (is_wp_error($post_id)) {
        return new WP_Error(
            'create_failed',
            $post_id->get_error_message(),
            array('status' => 500)
        );
    }
    
    // บันทึก Meta
    update_post_meta($post_id, '_book_author', $author);
    update_post_meta($post_id, '_book_price', $price);
    
    // Set Taxonomy
    if (!empty($genre_ids)) {
        wp_set_object_terms($post_id, array_map('intval', $genre_ids), 'genre');
    }
    
    $post = get_post($post_id);
    
    return new WP_REST_Response(format_book_for_api($post), 201);
}

function api_update_book( WP_REST_Request $request ) {
    
    $id   = (int) $request['id'];
    $post = get_post($id);
    
    if (!$post || $post->post_type !== 'book') {
        return new WP_Error('not_found', 'Book not found', array('status' => 404));
    }
    
    $update_data = array('ID' => $id);
    
    if ($request->has_param('title')) {
        $update_data['post_title'] = sanitize_text_field($request->get_param('title'));
    }
    
    if ($request->has_param('content')) {
        $update_data['post_content'] = wp_kses_post($request->get_param('content'));
    }
    
    if (count($update_data) > 1) {
        $result = wp_update_post($update_data, true);
        if (is_wp_error($result)) {
            return $result;
        }
    }
    
    if ($request->has_param('author')) {
        update_post_meta($id, '_book_author', sanitize_text_field($request->get_param('author')));
    }
    
    if ($request->has_param('price')) {
        update_post_meta($id, '_book_price', floatval($request->get_param('price')));
    }
    
    return new WP_REST_Response(format_book_for_api(get_post($id)), 200);
}

function api_delete_book( WP_REST_Request $request ) {
    
    $id    = (int) $request['id'];
    $force = $request->get_param('force') === 'true';
    
    $post = get_post($id);
    if (!$post || $post->post_type !== 'book') {
        return new WP_Error('not_found', 'Book not found', array('status' => 404));
    }
    
    $result = wp_delete_post($id, $force);
    
    if (!$result) {
        return new WP_Error('delete_failed', 'Failed to delete', array('status' => 500));
    }
    
    return new WP_REST_Response(array(
        'deleted'  => true,
        'previous' => format_book_for_api($post),
    ), 200);
}

function api_search_books( WP_REST_Request $request ) {
    
    $query = $request->get_param('q');
    
    $posts = get_posts(array(
        'post_type'      => 'book',
        's'              => $query,
        'posts_per_page' => 20,
        'post_status'    => 'publish',
    ));
    
    return new WP_REST_Response(
        array_map('format_book_for_api', $posts),
        200
    );
}

// ==========================================
// Helper: Format Book Data
// ==========================================

function format_book_for_api( $post ) {
    
    $price     = get_post_meta($post->ID, '_book_price', true);
    $author    = get_post_meta($post->ID, '_book_author', true);
    $isbn      = get_post_meta($post->ID, '_book_isbn', true);
    $available = get_post_meta($post->ID, '_book_available', true);
    $rating    = get_post_meta($post->ID, '_book_rating', true);
    
    $genres    = wp_get_post_terms($post->ID, 'genre', array('fields' => 'names'));
    $thumbnail = get_the_post_thumbnail_url($post->ID, 'medium');
    
    return array(
        'id'           => $post->ID,
        'title'        => $post->post_title,
        'slug'         => $post->post_name,
        'content'      => $post->post_content,
        'excerpt'      => $post->post_excerpt,
        'author'       => $author,
        'isbn'         => $isbn,
        'price'        => $price ? (float) $price : null,
        'available'    => (bool) $available,
        'rating'       => $rating ? (float) $rating : null,
        'genres'       => $genres,
        'thumbnail'    => $thumbnail ?: null,
        'link'         => get_permalink($post->ID),
        'created_at'   => $post->post_date_gmt,
        'modified_at'  => $post->post_modified_gmt,
    );
}
```

---

## 3. Authentication

```php
<?php
/**
 * REST API Authentication
 */

// ==========================================
// Method 1: Cookie Authentication (แบบ Default)
// ใช้สำหรับ Request จากหน้า WordPress เดียวกัน
// ==========================================

add_action('wp_head', function() {
    ?>
    <script>
    // JavaScript สามารถใช้ WordPress nonce ได้
    const wpApiSettings = {
        root: '<?php echo esc_url_raw(rest_url()); ?>',
        nonce: '<?php echo wp_create_nonce('wp_rest'); ?>'
    };
    </script>
    <?php
});

// JavaScript ใช้งาน:
/*
fetch('/wp-json/wp/v2/posts', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-WP-Nonce': wpApiSettings.nonce
    },
    body: JSON.stringify({ title: 'New Post', status: 'publish' })
});
*/

// ==========================================
// Method 2: Application Passwords (WordPress 5.6+)
// ==========================================

// สร้าง Application Password ใน:
// Users > Your Profile > Application Passwords

// ใช้งาน:
/*
curl -u "username:xxxx xxxx xxxx xxxx xxxx xxxx" \
     https://yoursite.com/wp-json/wp/v2/posts
*/

// PHP ใช้ Basic Auth:
$response = wp_remote_get('https://yoursite.com/wp-json/wp/v2/posts', array(
    'headers' => array(
        'Authorization' => 'Basic ' . base64_encode('username:app_password'),
    ),
));

// ==========================================
// Method 3: JWT Authentication
// (ต้องติดตั้ง Plugin: JWT Authentication for WP REST API)
// ==========================================

// ขั้นตอน:
// 1. ขอ Token
/*
POST /wp-json/jwt-auth/v1/token
{
    "username": "admin",
    "password": "password"
}
Response: { "token": "eyJ...", "user_email": "...", ... }
*/

// 2. ใช้ Token
/*
GET /wp-json/wp/v2/posts
Authorization: Bearer eyJ...
*/

// ==========================================
// Method 4: OAuth 2.0
// ==========================================

// ต้องติดตั้ง Plugin: WP OAuth Server

// ==========================================
// Custom Authentication Middleware
// ==========================================

add_filter( 'rest_authentication_errors', function( $result ) {
    
    // ถ้า Auth สำเร็จแล้วหรือ Error
    if ( ! empty($result) ) {
        return $result;
    }
    
    // ตรวจสอบ Custom API Key
    $api_key = sanitize_text_field($_SERVER['HTTP_X_API_KEY'] ?? '');
    
    if ( $api_key ) {
        // ค้นหา User ที่มี API Key นี้
        $users = get_users(array(
            'meta_key'   => '_api_key',
            'meta_value' => $api_key,
            'number'     => 1,
        ));
        
        if (!empty($users)) {
            wp_set_current_user($users[0]->ID);
            return true;
        }
        
        return new WP_Error(
            'invalid_api_key',
            'Invalid API Key',
            array('status' => 401)
        );
    }
    
    return $result;
});

// ==========================================
// Rate Limiting
// ==========================================

add_filter( 'rest_pre_dispatch', function( $result, $server, $request ) {
    
    $ip      = $_SERVER['REMOTE_ADDR'] ?? '';
    $key     = 'api_rate_limit_' . md5($ip);
    $count   = (int) get_transient($key);
    $limit   = 100; // 100 requests per hour
    
    if ($count >= $limit) {
        return new WP_Error(
            'rate_limit_exceeded',
            'Too many requests. Please try again later.',
            array(
                'status' => 429,
                'retry_after' => 3600,
            )
        );
    }
    
    set_transient($key, $count + 1, HOUR_IN_SECONDS);
    
    return $result;
    
}, 10, 3);
```

---

## 4. Extend Default Endpoints

```php
<?php
/**
 * Extend WordPress REST API Endpoints
 */

// ==========================================
// เพิ่ม Fields ให้ /wp/v2/posts
// ==========================================

add_action( 'rest_api_init', function() {
    
    // เพิ่ม Custom Field
    register_rest_field(
        'post',           // Post Type
        'reading_time',   // Field Name
        array(
            'get_callback' => function( $post_data ) {
                $content    = get_post_field('post_content', $post_data['id']);
                $word_count = str_word_count(strip_tags($content));
                return (int) ceil($word_count / 200);
            },
            'schema' => array(
                'description' => 'Estimated reading time in minutes',
                'type'        => 'integer',
            ),
        )
    );
    
    // เพิ่ม View Count
    register_rest_field(
        'post',
        'view_count',
        array(
            'get_callback' => function( $post_data ) {
                return (int) get_post_meta($post_data['id'], '_view_count', true);
            },
            'update_callback' => function( $value, $post ) {
                update_post_meta($post->ID, '_view_count', absint($value));
            },
            'schema' => array(
                'type' => 'integer',
            ),
        )
    );
    
    // เพิ่ม Related Posts
    register_rest_field(
        array('post', 'book'),
        'related',
        array(
            'get_callback' => function( $post_data ) {
                $post_id = $post_data['id'];
                $tags    = wp_get_post_tags($post_id, array('fields' => 'ids'));
                
                if (empty($tags)) return array();
                
                $related = get_posts(array(
                    'tag__in'        => $tags,
                    'post__not_in'   => array($post_id),
                    'posts_per_page' => 3,
                    'post_status'    => 'publish',
                ));
                
                return array_map(function($p) {
                    return array(
                        'id'    => $p->ID,
                        'title' => $p->post_title,
                        'link'  => get_permalink($p->ID),
                    );
                }, $related);
            },
        )
    );
    
} );

// ==========================================
// Modify Response
// ==========================================

// เพิ่มข้อมูลใน Response ของ Posts
add_filter( 'rest_prepare_post', function( $response, $post, $request ) {
    
    // เพิ่ม Featured Image URL แบบ Array ขนาดต่างๆ
    $thumbnail_id = get_post_thumbnail_id($post->ID);
    if ($thumbnail_id) {
        $sizes = array();
        foreach (get_intermediate_image_sizes() as $size) {
            $img = wp_get_attachment_image_src($thumbnail_id, $size);
            if ($img) {
                $sizes[$size] = array(
                    'url'    => $img[0],
                    'width'  => $img[1],
                    'height' => $img[2],
                );
            }
        }
        $response->data['thumbnail_sizes'] = $sizes;
    }
    
    // เพิ่ม Author Info
    $author_id = $post->post_author;
    $response->data['author_details'] = array(
        'id'          => $author_id,
        'name'        => get_the_author_meta('display_name', $author_id),
        'email'       => get_the_author_meta('user_email', $author_id),
        'avatar'      => get_avatar_url($author_id, array('size' => 80)),
        'description' => get_the_author_meta('description', $author_id),
    );
    
    return $response;
    
}, 10, 3);

// ==========================================
// ลบ Sensitive Data จาก Response
// ==========================================

// ลบ Email จาก Users Endpoint (สำหรับ Public)
add_filter( 'rest_prepare_user', function( $response, $user, $request ) {
    
    if ( ! current_user_can('list_users') ) {
        // ผู้ใช้ทั่วไปไม่ควรเห็น Email และข้อมูล Sensitive
        unset($response->data['email']);
        unset($response->data['capabilities']);
        unset($response->data['extra_capabilities']);
        unset($response->data['registered_date']);
    }
    
    return $response;
    
}, 10, 3);
```

---

## 5. REST API กับ JavaScript/React

```javascript
// ==========================================
// WordPress API Class
// ==========================================

class WordPressAPI {
    
    constructor(baseUrl, nonce = null) {
        this.baseUrl = baseUrl;
        this.nonce   = nonce;
        this.headers = {
            'Content-Type': 'application/json',
        };
        
        if (nonce) {
            this.headers['X-WP-Nonce'] = nonce;
        }
    }
    
    async request(endpoint, options = {}) {
        const url = `${this.baseUrl}${endpoint}`;
        
        const config = {
            headers: this.headers,
            ...options,
        };
        
        if (config.body && typeof config.body === 'object') {
            config.body = JSON.stringify(config.body);
        }
        
        const response = await fetch(url, config);
        
        if (!response.ok) {
            const error = await response.json();
            throw new Error(error.message || `HTTP Error ${response.status}`);
        }
        
        return {
            data: await response.json(),
            headers: response.headers,
            status: response.status,
        };
    }
    
    async getPosts(params = {}) {
        const query = new URLSearchParams(params).toString();
        return this.request(`/wp/v2/posts?${query}`);
    }
    
    async getPost(id) {
        return this.request(`/wp/v2/posts/${id}?_embed`);
    }
    
    async createPost(data) {
        return this.request('/wp/v2/posts', {
            method: 'POST',
            body: data,
        });
    }
    
    async updatePost(id, data) {
        return this.request(`/wp/v2/posts/${id}`, {
            method: 'PUT',
            body: data,
        });
    }
    
    async deletePost(id) {
        return this.request(`/wp/v2/posts/${id}`, {
            method: 'DELETE',
        });
    }
    
    // Custom Endpoint
    async getBooks(params = {}) {
        const query = new URLSearchParams(params).toString();
        return this.request(`/bookstore/v1/books?${query}`);
    }
}

// ==========================================
// การใช้งาน
// ==========================================

const api = new WordPressAPI('https://yoursite.com/wp-json', wpApiSettings.nonce);

// ดึง Posts
async function loadPosts() {
    try {
        const { data: posts, headers } = await api.getPosts({
            per_page: 10,
            _embed: true,
        });
        
        const total = headers.get('X-WP-Total');
        console.log(`Total: ${total} posts`);
        
        posts.forEach(post => {
            console.log(post.title.rendered, post.link);
        });
        
    } catch (error) {
        console.error('Error loading posts:', error.message);
    }
}

// สร้าง Post
async function createNewPost(title, content) {
    try {
        const { data: newPost } = await api.createPost({
            title: title,
            content: content,
            status: 'publish',
        });
        
        console.log('Created post:', newPost.id, newPost.link);
        return newPost;
        
    } catch (error) {
        console.error('Failed to create post:', error.message);
    }
}

// ==========================================
// React Component Example
// ==========================================

/*
import React, { useState, useEffect } from 'react';

function BookList() {
    const [books, setBooks] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    const [page, setPage] = useState(1);
    const [totalPages, setTotalPages] = useState(0);
    
    useEffect(() => {
        loadBooks();
    }, [page]);
    
    async function loadBooks() {
        setLoading(true);
        try {
            const res = await fetch(
                `/wp-json/bookstore/v1/books?page=${page}&per_page=12`
            );
            
            if (!res.ok) throw new Error('Failed to fetch');
            
            const data = await res.json();
            setBooks(data);
            setTotalPages(parseInt(res.headers.get('X-WP-TotalPages') || '0'));
            
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    }
    
    if (loading) return <div>กำลังโหลด...</div>;
    if (error) return <div>เกิดข้อผิดพลาด: {error}</div>;
    
    return (
        <div>
            <div className="books-grid">
                {books.map(book => (
                    <div key={book.id} className="book-card">
                        {book.thumbnail && <img src={book.thumbnail} alt={book.title} />}
                        <h3>{book.title}</h3>
                        <p>โดย {book.author}</p>
                        <p>{book.price} บาท</p>
                    </div>
                ))}
            </div>
            
            <div className="pagination">
                {page > 1 && <button onClick={() => setPage(p => p - 1)}>ก่อนหน้า</button>}
                <span>หน้า {page} / {totalPages}</span>
                {page < totalPages && <button onClick={() => setPage(p => p + 1)}>ถัดไป</button>}
            </div>
        </div>
    );
}
*/
```

---

## Workshop: สร้าง Mobile App API

### Exercise: สร้าง News App API

```php
<?php
// สร้าง REST Endpoints สำหรับ Mobile App:
// GET  /news-app/v1/feed          - ข่าวล่าสุด
// GET  /news-app/v1/categories    - หมวดหมู่
// GET  /news-app/v1/article/{id}  - บทความเดียว
// POST /news-app/v1/bookmark      - บันทึก Bookmark (ต้อง Login)
// GET  /news-app/v1/bookmarks     - ดู Bookmarks ของตัวเอง

add_action('rest_api_init', function() {
    
    // TODO: implement endpoints
    
    register_rest_route('news-app/v1', '/feed', array(
        'methods'             => 'GET',
        'callback'            => function($request) {
            // TODO: return formatted posts
        },
        'permission_callback' => '__return_true',
    ));
    
});
```

---

## Quiz

**คำถามที่ 1:** `register_rest_route()` namespace convention ที่ถูกต้องคือ?

A) `my-plugin`  
B) `my-plugin/v1`  
C) `wp/my-plugin`  
D) `api/my-plugin/v1`  

**เฉลย: B) my-plugin/v1 - Namespace ตามด้วย Version**

---

**คำถามที่ 2:** `WP_REST_Server::READABLE` เท่ากับ HTTP Method ใด?

A) POST  
B) PUT  
C) GET  
D) DELETE  

**เฉลย: C) GET**

---

**คำถามที่ 3:** จะดึง Total จำนวน Posts ทั้งหมดจาก REST API ได้จากที่ไหน?

A) Response body field `total`  
B) Response header `X-WP-Total`  
C) Response body field `count`  
D) Response header `Content-Count`  

**เฉลย: B) X-WP-Total header**

---

**คำถามที่ 4:** `register_rest_field()` ใช้ทำอะไร?

A) สร้าง Endpoint ใหม่  
B) เพิ่ม Fields ให้ Existing Endpoints  
C) Validate Request Parameters  
D) เพิ่ม Authentication  

**เฉลย: B) เพิ่ม Custom Fields ใน Response ของ Endpoints ที่มีอยู่แล้ว**

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- WordPress REST API พื้นฐานและ Endpoints ที่มีให้
- สร้าง Custom REST Endpoints
- Authentication Methods ต่างๆ
- Extend Default Endpoints
- ใช้งาน REST API กับ JavaScript

---

## ต่อไป

➡️ **[Part 058: WordPress Gutenberg Blocks](part-058-wordpress-gutenberg.md)**

เรียนรู้เกี่ยวกับ:
- Gutenberg Block Development
- Block JSON Schema
- PHP/JavaScript Block Code
- Block Patterns
