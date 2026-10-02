# Part 055: WordPress Custom Post Types & Taxonomies

**ระดับ:** Intermediate  
**เวลาเรียน:** 4-5 ชั่วโมง  
**Prerequisites:** Part 054 (WordPress Plugin Basics)

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. สร้าง Custom Post Type (CPT) ด้วย `register_post_type()`
2. สร้าง Custom Taxonomy ด้วย `register_taxonomy()`
3. ใช้งาน Custom Fields และ Meta Boxes
4. Query CPT ด้วย WP_Query
5. สร้าง Template สำหรับ CPT

---

## 1. Custom Post Types (CPT)

### 1.1 register_post_type พื้นฐาน

```php
<?php
/**
 * สร้าง Custom Post Type: Book
 */

function register_book_post_type() {
    
    $labels = array(
        'name'                  => _x( 'Books', 'Post type general name', 'my-plugin' ),
        'singular_name'         => _x( 'Book', 'Post type singular name', 'my-plugin' ),
        'menu_name'             => _x( 'Books', 'Admin Menu text', 'my-plugin' ),
        'name_admin_bar'        => _x( 'Book', 'Add New on Toolbar', 'my-plugin' ),
        'add_new'               => __( 'Add New', 'my-plugin' ),
        'add_new_item'          => __( 'Add New Book', 'my-plugin' ),
        'new_item'              => __( 'New Book', 'my-plugin' ),
        'edit_item'             => __( 'Edit Book', 'my-plugin' ),
        'view_item'             => __( 'View Book', 'my-plugin' ),
        'all_items'             => __( 'All Books', 'my-plugin' ),
        'search_items'          => __( 'Search Books', 'my-plugin' ),
        'parent_item_colon'     => __( 'Parent Books:', 'my-plugin' ),
        'not_found'             => __( 'No books found.', 'my-plugin' ),
        'not_found_in_trash'    => __( 'No books found in Trash.', 'my-plugin' ),
        'featured_image'        => _x( 'Book Cover', 'Overrides the "Featured Image" phrase.', 'my-plugin' ),
        'set_featured_image'    => _x( 'Set cover image', 'Overrides the "Set featured image" phrase.', 'my-plugin' ),
        'remove_featured_image' => _x( 'Remove cover image', 'my-plugin' ),
        'use_featured_image'    => _x( 'Use as cover image', 'my-plugin' ),
        'archives'              => _x( 'Book archives', 'my-plugin' ),
        'insert_into_item'      => _x( 'Insert into book', 'my-plugin' ),
        'uploaded_to_this_item' => _x( 'Uploaded to this book', 'my-plugin' ),
        'filter_items_list'     => _x( 'Filter books list', 'my-plugin' ),
        'items_list_navigation' => _x( 'Books list navigation', 'my-plugin' ),
        'items_list'            => _x( 'Books list', 'my-plugin' ),
    );
    
    $args = array(
        'labels'             => $labels,
        'public'             => true,              // แสดงบน Frontend
        'publicly_queryable' => true,              // ค้นหาได้
        'show_ui'            => true,              // แสดงใน Admin
        'show_in_menu'       => true,              // แสดงใน Admin Menu
        'query_var'          => true,              // รองรับ ?book=slug
        'rewrite'            => array( 
            'slug'       => 'books',               // URL: /books/my-book
            'with_front' => false,                 // ไม่เพิ่ม Front Base
        ),
        'capability_type'    => 'post',            // ใช้ Post Capabilities
        'has_archive'        => 'books',           // /books/ page
        'hierarchical'       => false,             // false = like posts, true = like pages
        'menu_position'      => 20,
        'menu_icon'          => 'dashicons-book-alt',
        'supports'           => array(
            'title',
            'editor',
            'author',
            'thumbnail',
            'excerpt',
            'comments',
            'revisions',
            'custom-fields',
            'page-attributes',
        ),
        'show_in_rest'       => true,              // Block Editor Support
        'rest_base'          => 'books',           // REST API Endpoint
        'rest_controller_class' => 'WP_REST_Posts_Controller',
        'taxonomies'         => array( 'genre', 'book_tag' ), // ผูก Taxonomies
    );
    
    register_post_type( 'book', $args );
}
add_action( 'init', 'register_book_post_type' );

// ==========================================
// สร้าง Custom Post Types หลายตัวพร้อมกัน
// ==========================================

function register_all_custom_post_types() {
    
    $post_types = array(
        
        'portfolio' => array(
            'singular' => 'Portfolio',
            'plural'   => 'Portfolio',
            'icon'     => 'dashicons-portfolio',
            'slug'     => 'portfolio',
            'supports' => array( 'title', 'editor', 'thumbnail', 'excerpt' ),
        ),
        
        'testimonial' => array(
            'singular' => 'Testimonial',
            'plural'   => 'Testimonials',
            'icon'     => 'dashicons-format-quote',
            'slug'     => 'testimonials',
            'supports' => array( 'title', 'editor', 'thumbnail' ),
        ),
        
        'event' => array(
            'singular' => 'Event',
            'plural'   => 'Events',
            'icon'     => 'dashicons-calendar',
            'slug'     => 'events',
            'has_archive' => true,
            'supports' => array( 'title', 'editor', 'thumbnail', 'excerpt', 'author' ),
        ),
        
        'faq' => array(
            'singular'  => 'FAQ',
            'plural'    => 'FAQs',
            'icon'      => 'dashicons-editor-help',
            'slug'      => 'faq',
            'hierarchical' => false,
            'supports'  => array( 'title', 'editor' ),
        ),
    );
    
    foreach ( $post_types as $type => $config ) {
        
        $labels = array(
            'name'          => $config['plural'],
            'singular_name' => $config['singular'],
            'add_new_item'  => 'Add New ' . $config['singular'],
            'edit_item'     => 'Edit ' . $config['singular'],
            'all_items'     => 'All ' . $config['plural'],
            'not_found'     => 'No ' . strtolower($config['plural']) . ' found.',
        );
        
        register_post_type( $type, array(
            'labels'          => $labels,
            'public'          => true,
            'show_ui'         => true,
            'show_in_menu'    => true,
            'menu_icon'       => $config['icon'],
            'has_archive'     => $config['has_archive'] ?? $config['slug'],
            'rewrite'         => array( 'slug' => $config['slug'] ),
            'supports'        => $config['supports'],
            'hierarchical'    => $config['hierarchical'] ?? false,
            'show_in_rest'    => true,
        ) );
    }
}
add_action( 'init', 'register_all_custom_post_types' );
```

---

## 2. Custom Taxonomies

```php
<?php
/**
 * สร้าง Custom Taxonomy
 */

function register_book_taxonomies() {
    
    // ==========================================
    // Genre - Hierarchical (เหมือน Category)
    // ==========================================
    
    $genre_labels = array(
        'name'              => _x( 'Genres', 'taxonomy general name', 'my-plugin' ),
        'singular_name'     => _x( 'Genre', 'taxonomy singular name', 'my-plugin' ),
        'search_items'      => __( 'Search Genres', 'my-plugin' ),
        'all_items'         => __( 'All Genres', 'my-plugin' ),
        'parent_item'       => __( 'Parent Genre', 'my-plugin' ),
        'parent_item_colon' => __( 'Parent Genre:', 'my-plugin' ),
        'edit_item'         => __( 'Edit Genre', 'my-plugin' ),
        'update_item'       => __( 'Update Genre', 'my-plugin' ),
        'add_new_item'      => __( 'Add New Genre', 'my-plugin' ),
        'new_item_name'     => __( 'New Genre Name', 'my-plugin' ),
        'menu_name'         => __( 'Genre', 'my-plugin' ),
    );
    
    register_taxonomy( 'genre', array( 'book' ), array(
        'hierarchical'      => true,     // true = Categories, false = Tags
        'labels'            => $genre_labels,
        'show_ui'           => true,
        'show_admin_column' => true,     // แสดงใน Post List
        'query_var'         => true,
        'rewrite'           => array( 'slug' => 'genre' ),
        'show_in_rest'      => true,
        'rest_base'         => 'genres',
    ) );
    
    // ==========================================
    // Book Tag - Non-Hierarchical (เหมือน Tag)
    // ==========================================
    
    $book_tag_labels = array(
        'name'                       => _x( 'Book Tags', 'taxonomy general name', 'my-plugin' ),
        'singular_name'              => _x( 'Book Tag', 'taxonomy singular name', 'my-plugin' ),
        'search_items'               => __( 'Search Book Tags', 'my-plugin' ),
        'popular_items'              => __( 'Popular Book Tags', 'my-plugin' ),
        'all_items'                  => __( 'All Book Tags', 'my-plugin' ),
        'edit_item'                  => __( 'Edit Book Tag', 'my-plugin' ),
        'update_item'                => __( 'Update Book Tag', 'my-plugin' ),
        'add_new_item'               => __( 'Add New Book Tag', 'my-plugin' ),
        'new_item_name'              => __( 'New Book Tag Name', 'my-plugin' ),
        'separate_items_with_commas' => __( 'Separate tags with commas', 'my-plugin' ),
        'add_or_remove_items'        => __( 'Add or remove tags', 'my-plugin' ),
        'choose_from_most_used'      => __( 'Choose from the most used tags', 'my-plugin' ),
        'not_found'                  => __( 'No tags found.', 'my-plugin' ),
        'menu_name'                  => __( 'Book Tags', 'my-plugin' ),
    );
    
    register_taxonomy( 'book_tag', array( 'book' ), array(
        'hierarchical'          => false,
        'labels'                => $book_tag_labels,
        'show_ui'               => true,
        'show_admin_column'     => true,
        'update_count_callback' => '_update_post_term_count',
        'query_var'             => true,
        'rewrite'               => array( 'slug' => 'book-tag' ),
        'show_in_rest'          => true,
    ) );
}
add_action( 'init', 'register_book_taxonomies' );

// ==========================================
// ผูก Taxonomy กับ Post Type ภายหลัง
// ==========================================

function attach_taxonomy_to_post_type() {
    register_taxonomy_for_object_type( 'category', 'book' ); // ใช้ Category ปกติ
    register_taxonomy_for_object_type( 'post_tag', 'book' );  // ใช้ Tag ปกติ
}
add_action( 'init', 'attach_taxonomy_to_post_type' );
```

---

## 3. Meta Boxes

```php
<?php
/**
 * Meta Boxes สำหรับ Book Post Type
 */

class Book_Meta_Boxes {
    
    public function __construct() {
        add_action( 'add_meta_boxes', array( $this, 'add_meta_boxes' ) );
        add_action( 'save_post_book', array( $this, 'save_meta' ), 10, 2 );
    }
    
    public function add_meta_boxes() {
        
        // Book Details Meta Box
        add_meta_box(
            'book_details',              // ID
            __( 'Book Details', 'my-plugin' ), // Title
            array( $this, 'render_book_details' ), // Callback
            'book',                      // Post Type
            'normal',                    // Context: normal, side, advanced
            'high'                       // Priority: high, default, low, core
        );
        
        // Book Pricing Meta Box (ด้านข้าง)
        add_meta_box(
            'book_pricing',
            __( 'Pricing', 'my-plugin' ),
            array( $this, 'render_book_pricing' ),
            'book',
            'side',
            'default'
        );
        
        // ลบ Meta Box ที่ไม่ต้องการ
        remove_meta_box( 'commentsdiv', 'book', 'normal' );
    }
    
    public function render_book_details( $post ) {
        
        // สร้าง Nonce
        wp_nonce_field( 'save_book_details', 'book_details_nonce' );
        
        // ดึงค่าที่บันทึกไว้
        $author   = get_post_meta( $post->ID, '_book_author', true );
        $isbn     = get_post_meta( $post->ID, '_book_isbn', true );
        $pages    = get_post_meta( $post->ID, '_book_pages', true );
        $publisher = get_post_meta( $post->ID, '_book_publisher', true );
        $pub_date = get_post_meta( $post->ID, '_book_publish_date', true );
        $language = get_post_meta( $post->ID, '_book_language', true );
        $rating   = get_post_meta( $post->ID, '_book_rating', true );
        
        ?>
        <style>
            .book-meta-table { width: 100%; border-collapse: collapse; }
            .book-meta-table td { padding: 8px; vertical-align: top; }
            .book-meta-table label { font-weight: 600; }
            .book-meta-table input, .book-meta-table select { width: 100%; }
        </style>
        
        <table class="book-meta-table">
            <tr>
                <td width="30%"><label for="_book_author">ผู้แต่ง *</label></td>
                <td>
                    <input type="text" 
                           id="_book_author" 
                           name="_book_author" 
                           value="<?php echo esc_attr( $author ); ?>"
                           placeholder="ชื่อผู้แต่ง">
                </td>
            </tr>
            
            <tr>
                <td><label for="_book_isbn">ISBN</label></td>
                <td>
                    <input type="text" 
                           id="_book_isbn" 
                           name="_book_isbn" 
                           value="<?php echo esc_attr( $isbn ); ?>"
                           placeholder="978-X-XXXX-XXXX-X">
                </td>
            </tr>
            
            <tr>
                <td><label for="_book_pages">จำนวนหน้า</label></td>
                <td>
                    <input type="number" 
                           id="_book_pages" 
                           name="_book_pages" 
                           value="<?php echo absint( $pages ); ?>"
                           min="1">
                </td>
            </tr>
            
            <tr>
                <td><label for="_book_publisher">สำนักพิมพ์</label></td>
                <td>
                    <input type="text" 
                           id="_book_publisher" 
                           name="_book_publisher" 
                           value="<?php echo esc_attr( $publisher ); ?>">
                </td>
            </tr>
            
            <tr>
                <td><label for="_book_publish_date">วันที่พิมพ์</label></td>
                <td>
                    <input type="date" 
                           id="_book_publish_date" 
                           name="_book_publish_date" 
                           value="<?php echo esc_attr( $pub_date ); ?>">
                </td>
            </tr>
            
            <tr>
                <td><label for="_book_language">ภาษา</label></td>
                <td>
                    <select id="_book_language" name="_book_language">
                        <option value="thai" <?php selected( $language, 'thai' ); ?>>ไทย</option>
                        <option value="english" <?php selected( $language, 'english' ); ?>>English</option>
                        <option value="japanese" <?php selected( $language, 'japanese' ); ?>>日本語</option>
                    </select>
                </td>
            </tr>
            
            <tr>
                <td><label for="_book_rating">คะแนน (1-5)</label></td>
                <td>
                    <input type="number" 
                           id="_book_rating" 
                           name="_book_rating" 
                           value="<?php echo esc_attr( $rating ); ?>"
                           min="1" max="5" step="0.1">
                </td>
            </tr>
        </table>
        <?php
    }
    
    public function render_book_pricing( $post ) {
        
        wp_nonce_field( 'save_book_pricing', 'book_pricing_nonce' );
        
        $price        = get_post_meta( $post->ID, '_book_price', true );
        $sale_price   = get_post_meta( $post->ID, '_book_sale_price', true );
        $is_available = get_post_meta( $post->ID, '_book_available', true );
        
        ?>
        <p>
            <label for="_book_price"><strong>ราคาปกติ (บาท)</strong></label><br>
            <input type="number" 
                   id="_book_price" 
                   name="_book_price" 
                   value="<?php echo esc_attr( $price ); ?>"
                   min="0" step="0.01"
                   style="width:100%">
        </p>
        
        <p>
            <label for="_book_sale_price"><strong>ราคาลด (บาท)</strong></label><br>
            <input type="number" 
                   id="_book_sale_price" 
                   name="_book_sale_price" 
                   value="<?php echo esc_attr( $sale_price ); ?>"
                   min="0" step="0.01"
                   style="width:100%">
        </p>
        
        <p>
            <label>
                <input type="checkbox"
                       name="_book_available"
                       value="1"
                       <?php checked( $is_available, '1' ); ?>>
                <strong>มีจำหน่าย</strong>
            </label>
        </p>
        <?php
    }
    
    public function save_meta( $post_id, $post ) {
        
        // ==========================================
        // Security Checks
        // ==========================================
        
        // Auto-save
        if ( defined( 'DOING_AUTOSAVE' ) && DOING_AUTOSAVE ) {
            return;
        }
        
        // Revision
        if ( wp_is_post_revision( $post_id ) ) {
            return;
        }
        
        // Book Details Nonce
        if ( isset( $_POST['book_details_nonce'] ) && 
             wp_verify_nonce( $_POST['book_details_nonce'], 'save_book_details' ) ) {
            
            if ( current_user_can( 'edit_post', $post_id ) ) {
                
                // บันทึกข้อมูล
                $fields = array(
                    '_book_author'       => 'sanitize_text_field',
                    '_book_isbn'         => 'sanitize_text_field',
                    '_book_publisher'    => 'sanitize_text_field',
                    '_book_publish_date' => 'sanitize_text_field',
                    '_book_language'     => 'sanitize_text_field',
                );
                
                foreach ( $fields as $key => $sanitize ) {
                    if ( isset( $_POST[$key] ) ) {
                        update_post_meta( $post_id, $key, $sanitize( $_POST[$key] ) );
                    }
                }
                
                // Number Fields
                if ( isset( $_POST['_book_pages'] ) ) {
                    update_post_meta( $post_id, '_book_pages', absint( $_POST['_book_pages'] ) );
                }
                
                if ( isset( $_POST['_book_rating'] ) ) {
                    $rating = floatval( $_POST['_book_rating'] );
                    $rating = max( 1, min( 5, $rating ) );
                    update_post_meta( $post_id, '_book_rating', $rating );
                }
            }
        }
        
        // Book Pricing Nonce
        if ( isset( $_POST['book_pricing_nonce'] ) && 
             wp_verify_nonce( $_POST['book_pricing_nonce'], 'save_book_pricing' ) ) {
            
            if ( current_user_can( 'edit_post', $post_id ) ) {
                
                update_post_meta( 
                    $post_id, 
                    '_book_price', 
                    floatval( $_POST['_book_price'] ?? 0 ) 
                );
                
                update_post_meta( 
                    $post_id, 
                    '_book_sale_price', 
                    floatval( $_POST['_book_sale_price'] ?? 0 ) 
                );
                
                update_post_meta( 
                    $post_id, 
                    '_book_available', 
                    isset( $_POST['_book_available'] ) ? '1' : '0' 
                );
            }
        }
    }
}

new Book_Meta_Boxes();
```

---

## 4. WP_Query กับ Custom Post Types

```php
<?php
/**
 * Querying Custom Post Types
 */

// ==========================================
// ดึง Books ทั้งหมด
// ==========================================

$books = new WP_Query( array(
    'post_type'      => 'book',
    'posts_per_page' => 12,
    'post_status'    => 'publish',
) );

// ==========================================
// ดึง Books ตาม Genre
// ==========================================

$fiction_books = new WP_Query( array(
    'post_type'      => 'book',
    'posts_per_page' => 6,
    'tax_query'      => array(
        array(
            'taxonomy' => 'genre',
            'field'    => 'slug',
            'terms'    => array( 'fiction', 'science-fiction' ),
            'operator' => 'IN', // IN, NOT IN, AND
        ),
    ),
) );

// ==========================================
// ดึง Books ตาม Meta (Custom Fields)
// ==========================================

$available_books = new WP_Query( array(
    'post_type'      => 'book',
    'posts_per_page' => 10,
    'meta_query'     => array(
        'relation' => 'AND',
        array(
            'key'     => '_book_available',
            'value'   => '1',
            'compare' => '=',
        ),
        array(
            'key'     => '_book_price',
            'value'   => array( 100, 500 ),
            'type'    => 'NUMERIC',
            'compare' => 'BETWEEN',
        ),
    ),
) );

// ==========================================
// เรียงตาม Meta Value
// ==========================================

$books_by_price = new WP_Query( array(
    'post_type'      => 'book',
    'posts_per_page' => 10,
    'meta_key'       => '_book_price',
    'orderby'        => 'meta_value_num',
    'order'          => 'ASC',
) );

// ==========================================
// ดึง Books ที่เกี่ยวข้อง
// ==========================================

function get_related_books( $post_id, $count = 4 ) {
    
    $genres = wp_get_object_terms( $post_id, 'genre', array('fields' => 'ids') );
    
    if ( empty( $genres ) ) {
        return array();
    }
    
    return new WP_Query( array(
        'post_type'      => 'book',
        'posts_per_page' => $count,
        'post__not_in'   => array( $post_id ),
        'tax_query'      => array(
            array(
                'taxonomy' => 'genre',
                'field'    => 'term_id',
                'terms'    => $genres,
            ),
        ),
        'orderby'        => 'rand',
    ) );
}

// ==========================================
// Loop ผ่าน Custom Post Type
// ==========================================

$books_query = new WP_Query( array(
    'post_type'      => 'book',
    'posts_per_page' => 12,
) );

if ( $books_query->have_posts() ) :
    while ( $books_query->have_posts() ) :
        $books_query->the_post();
        
        $price       = get_post_meta( get_the_ID(), '_book_price', true );
        $author      = get_post_meta( get_the_ID(), '_book_author', true );
        $genres      = get_the_terms( get_the_ID(), 'genre' );
        $is_available = get_post_meta( get_the_ID(), '_book_available', true );
        
        ?>
        <div class="book-card <?php echo $is_available ? 'available' : 'unavailable'; ?>">
            <?php the_post_thumbnail('medium'); ?>
            
            <div class="book-info">
                <?php the_title('<h3>', '</h3>'); ?>
                
                <?php if ($author) : ?>
                    <p class="book-author">โดย <?php echo esc_html($author); ?></p>
                <?php endif; ?>
                
                <?php if ($genres && !is_wp_error($genres)) : ?>
                    <div class="book-genres">
                        <?php foreach ($genres as $genre) : ?>
                            <a href="<?php echo esc_url(get_term_link($genre)); ?>">
                                <?php echo esc_html($genre->name); ?>
                            </a>
                        <?php endforeach; ?>
                    </div>
                <?php endif; ?>
                
                <?php if ($price) : ?>
                    <p class="book-price"><?php echo number_format($price); ?> บาท</p>
                <?php endif; ?>
                
                <a href="<?php the_permalink(); ?>" class="btn-read-more">ดูรายละเอียด</a>
            </div>
        </div>
        <?php
    endwhile;
    wp_reset_postdata();
endif;
```

---

## 5. Template สำหรับ Custom Post Type

```php
<?php
// ==========================================
// File: single-book.php
// Template สำหรับ Single Book
// ==========================================

get_header();

while ( have_posts() ) :
    the_post();
    
    $author    = get_post_meta( get_the_ID(), '_book_author', true );
    $isbn      = get_post_meta( get_the_ID(), '_book_isbn', true );
    $pages     = get_post_meta( get_the_ID(), '_book_pages', true );
    $publisher = get_post_meta( get_the_ID(), '_book_publisher', true );
    $pub_date  = get_post_meta( get_the_ID(), '_book_publish_date', true );
    $language  = get_post_meta( get_the_ID(), '_book_language', true );
    $price     = get_post_meta( get_the_ID(), '_book_price', true );
    $sale_price = get_post_meta( get_the_ID(), '_book_sale_price', true );
    $rating    = get_post_meta( get_the_ID(), '_book_rating', true );
    $available = get_post_meta( get_the_ID(), '_book_available', true );
    $genres    = get_the_terms( get_the_ID(), 'genre' );
    
    ?>
    
    <article id="book-<?php the_ID(); ?>" <?php post_class('single-book'); ?>>
        
        <div class="book-layout">
            
            <!-- Book Cover -->
            <div class="book-cover">
                <?php the_post_thumbnail('large'); ?>
                
                <!-- Rating Stars -->
                <?php if ($rating) : ?>
                    <div class="book-rating">
                        <?php
                        $full_stars = floor($rating);
                        $half_star  = ($rating - $full_stars) >= 0.5;
                        
                        for ($i = 1; $i <= 5; $i++) :
                            if ($i <= $full_stars) :
                                echo '<span class="star full">★</span>';
                            elseif ($i === $full_stars + 1 && $half_star) :
                                echo '<span class="star half">★</span>';
                            else :
                                echo '<span class="star empty">☆</span>';
                            endif;
                        endfor;
                        ?>
                        <span class="rating-value"><?php echo esc_html($rating); ?>/5</span>
                    </div>
                <?php endif; ?>
            </div>
            
            <!-- Book Details -->
            <div class="book-details">
                
                <?php the_title('<h1 class="book-title">', '</h1>'); ?>
                
                <?php if ($author) : ?>
                    <p class="book-author">
                        ผู้แต่ง: <strong><?php echo esc_html($author); ?></strong>
                    </p>
                <?php endif; ?>
                
                <!-- Genres -->
                <?php if ($genres && !is_wp_error($genres)) : ?>
                    <div class="book-genres">
                        <span>ประเภท:</span>
                        <?php foreach ($genres as $genre) : ?>
                            <a href="<?php echo esc_url(get_term_link($genre)); ?>" 
                               class="genre-tag">
                                <?php echo esc_html($genre->name); ?>
                            </a>
                        <?php endforeach; ?>
                    </div>
                <?php endif; ?>
                
                <!-- Price -->
                <div class="book-pricing">
                    <?php if ($sale_price && $sale_price < $price) : ?>
                        <span class="original-price">
                            ราคาปกติ: <del><?php echo number_format($price); ?> บาท</del>
                        </span>
                        <span class="sale-price">
                            ราคาลด: <strong><?php echo number_format($sale_price); ?> บาท</strong>
                        </span>
                    <?php elseif ($price) : ?>
                        <span class="regular-price">
                            ราคา: <strong><?php echo number_format($price); ?> บาท</strong>
                        </span>
                    <?php endif; ?>
                </div>
                
                <!-- Availability -->
                <div class="book-availability">
                    <?php if ($available) : ?>
                        <span class="badge available">✓ มีจำหน่าย</span>
                    <?php else : ?>
                        <span class="badge unavailable">✗ สินค้าหมด</span>
                    <?php endif; ?>
                </div>
                
                <!-- Book Info Table -->
                <table class="book-info-table">
                    <tbody>
                        <?php if ($isbn) : ?>
                            <tr><th>ISBN</th><td><?php echo esc_html($isbn); ?></td></tr>
                        <?php endif; ?>
                        <?php if ($pages) : ?>
                            <tr><th>จำนวนหน้า</th><td><?php echo absint($pages); ?> หน้า</td></tr>
                        <?php endif; ?>
                        <?php if ($publisher) : ?>
                            <tr><th>สำนักพิมพ์</th><td><?php echo esc_html($publisher); ?></td></tr>
                        <?php endif; ?>
                        <?php if ($pub_date) : ?>
                            <tr><th>วันที่พิมพ์</th><td><?php echo esc_html($pub_date); ?></td></tr>
                        <?php endif; ?>
                        <?php if ($language) : ?>
                            <tr><th>ภาษา</th><td><?php echo esc_html(ucfirst($language)); ?></td></tr>
                        <?php endif; ?>
                    </tbody>
                </table>
                
            </div>
        </div>
        
        <!-- Book Description -->
        <div class="book-description">
            <h2>รายละเอียด</h2>
            <?php the_content(); ?>
        </div>
        
        <!-- Related Books -->
        <div class="related-books">
            <h2>หนังสือที่เกี่ยวข้อง</h2>
            <?php
            $related = get_related_books( get_the_ID(), 4 );
            if ( $related && $related->have_posts() ) :
                echo '<div class="books-grid">';
                while ($related->have_posts()) :
                    $related->the_post();
                    get_template_part('template-parts/book', 'card');
                endwhile;
                echo '</div>';
                wp_reset_postdata();
            endif;
            ?>
        </div>
        
    </article>
    
    <?php
endwhile;

get_footer();
?>
```

---

## Workshop: สร้าง Event Custom Post Type

### Exercise: สร้าง System สำหรับจัดการ Events

```bash
# Generate Post Type Scaffold
wp scaffold post-type event --plugin=my-events-plugin \
    --label="Event" \
    --supports="title editor excerpt thumbnail"
```

```php
<?php
// TODO: สร้าง Event Post Type พร้อม:
// 1. CPT: event
// 2. Taxonomy: event_type (Hierarchical)
// 3. Meta Box: วันที่เริ่ม, วันที่สิ้นสุด, สถานที่, ราคา
// 4. Template: single-event.php

function register_event_cpt() {
    register_post_type( 'event', array(
        'labels'      => array(
            'name'          => 'Events',
            'singular_name' => 'Event',
        ),
        'public'      => true,
        'has_archive' => true,
        'supports'    => array( 'title', 'editor', 'thumbnail', 'excerpt' ),
        'rewrite'     => array( 'slug' => 'events' ),
        'show_in_rest' => true,
        'menu_icon'   => 'dashicons-calendar',
    ) );
}
add_action( 'init', 'register_event_cpt' );
```

---

## Quiz

**คำถามที่ 1:** อาร์กิวเมนต์ใดใน register_post_type ที่ควบคุม URL ของ Archive Page?

A) `archive_url`  
B) `has_archive`  
C) `archive_slug`  
D) `rewrite`  

**เฉลย: B) has_archive - ตั้งเป็น true หรือ string slug**

---

**คำถามที่ 2:** ความแตกต่างระหว่าง `hierarchical => true` และ `false` ใน register_taxonomy?

A) ไม่มีความแตกต่าง  
B) true = แบบ Category (มี Parent), false = แบบ Tag  
C) true ทำงานเร็วกว่า  
D) false รองรับ Custom Fields  

**เฉลย: B) hierarchical true = Categories (tree structure), false = Tags (flat)**

---

**คำถามที่ 3:** `add_meta_box()` context `side` หมายความว่าอะไร?

A) แสดงข้างซ้ายของ Content  
B) แสดงใน Sidebar ด้านขวาของ Editor  
C) แสดงในแถบ Toolbar  
D) แสดงใต้ Title  

**เฉลย: B) แสดงใน Sidebar ด้านขวาของ Classic Editor**

---

**คำถามที่ 4:** ทำไม `flush_rewrite_rules()` จึงสำคัญหลัง register_post_type?

A) ล้าง Post Cache  
B) อัปเดต URL Rewrite Rules ใน Database เพื่อให้ Permalink ทำงาน  
C) Reload Theme  
D) Reset Settings  

**เฉลย: B) WordPress เก็บ Rewrite Rules ใน wp_options ต้อง Flush เพื่อให้ CPT Permalink ทำงาน**

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- สร้าง Custom Post Types ด้วย register_post_type
- สร้าง Custom Taxonomies (Hierarchical และ Non-Hierarchical)
- สร้าง Meta Boxes สำหรับ Custom Fields
- Query CPT ด้วย WP_Query พร้อม tax_query และ meta_query
- สร้าง Template สำหรับ CPT

---

## ต่อไป

➡️ **[Part 056: WordPress Custom Fields Advanced](part-056-wordpress-custom-fields.md)**

เรียนรู้เกี่ยวกับ:
- add_meta_box Advanced
- get/update_post_meta Patterns
- ACF-style Field Groups
- Repeater Fields
