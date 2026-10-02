# Part 052: WordPress Theme Basics

**ระดับ:** Intermediate  
**เวลาเรียน:** 4-5 ชั่วโมง  
**Prerequisites:** Part 051 (WordPress Installation)

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. เข้าใจ Template Hierarchy ของ WordPress
2. สร้างและจัดการ functions.php
3. Enqueue Scripts และ Styles อย่างถูกต้อง
4. ใช้งาน WordPress Action และ Filter Hooks
5. สร้าง Theme จาก Scratch

---

## 1. Template Hierarchy

Template Hierarchy คือลำดับการค้นหา Template File ที่ WordPress ใช้เพื่อแสดงผลหน้าต่างๆ

```
WordPress Template Hierarchy (แสดงแบบย่อ)

Home/Blog Page:
  home.php -> index.php

Single Post:
  single-{post-type}-{slug}.php
  -> single-{post-type}.php
  -> single.php
  -> singular.php
  -> index.php

Single Page:
  {custom-template}.php (จาก Page Attributes)
  -> page-{slug}.php
  -> page-{id}.php
  -> page.php
  -> singular.php
  -> index.php

Category Archive:
  category-{slug}.php
  -> category-{id}.php
  -> category.php
  -> archive.php
  -> index.php

Tag Archive:
  tag-{slug}.php
  -> tag-{id}.php
  -> tag.php
  -> archive.php
  -> index.php

Author Archive:
  author-{nicename}.php
  -> author-{id}.php
  -> author.php
  -> archive.php
  -> index.php

Date Archive:
  date.php -> archive.php -> index.php

Search:
  search.php -> index.php

404:
  404.php -> index.php
```

### 1.1 ตัวอย่าง Template Files

```php
<?php
// ==========================================
// index.php - Main Template (Fallback)
// ==========================================
get_header(); // โหลด header.php
?>

<main id="main-content">
    <?php
    // WordPress Loop
    if ( have_posts() ) {
        while ( have_posts() ) {
            the_post();
            get_template_part( 'template-parts/content', get_post_type() );
        }
        
        the_posts_pagination(); // แสดง Pagination
    } else {
        get_template_part( 'template-parts/content', 'none' );
    }
    ?>
</main>

<?php
get_sidebar(); // โหลด sidebar.php
get_footer();  // โหลด footer.php
?>
```

```php
<?php
// ==========================================
// single.php - Single Post Template
// ==========================================
get_header();
?>

<main id="main-content">
    <?php
    while ( have_posts() ) {
        the_post();
        ?>
        
        <article id="post-<?php the_ID(); ?>" <?php post_class(); ?>>
            <header class="entry-header">
                <?php the_title( '<h1 class="entry-title">', '</h1>' ); ?>
                
                <div class="entry-meta">
                    <span class="author">
                        <?php the_author_posts_link(); ?>
                    </span>
                    <span class="date">
                        <?php echo get_the_date(); ?>
                    </span>
                    <span class="categories">
                        <?php the_category( ', ' ); ?>
                    </span>
                </div>
            </header>
            
            <?php if ( has_post_thumbnail() ) : ?>
                <div class="post-thumbnail">
                    <?php the_post_thumbnail( 'large' ); ?>
                </div>
            <?php endif; ?>
            
            <div class="entry-content">
                <?php the_content(); ?>
            </div>
            
            <footer class="entry-footer">
                <?php the_tags( '<div class="tags">Tags: ', ', ', '</div>' ); ?>
            </footer>
        </article>
        
        <?php
        // Previous/Next Post Navigation
        the_post_navigation(
            array(
                'prev_text' => '&larr; %title',
                'next_text' => '%title &rarr;',
            )
        );
        
        // Comments
        if ( comments_open() || get_comments_number() ) {
            comments_template();
        }
    }
    ?>
</main>

<?php get_footer(); ?>
```

```php
<?php
// ==========================================
// page.php - Page Template
// ==========================================
get_header();
?>

<div class="container">
    <main id="primary">
        <?php
        while ( have_posts() ) {
            the_post();
            ?>
            
            <article id="page-<?php the_ID(); ?>" <?php post_class(); ?>>
                <?php the_title( '<h1>', '</h1>' ); ?>
                
                <div class="page-content">
                    <?php
                    the_content();
                    wp_link_pages();
                    ?>
                </div>
            </article>
            
            <?php
            if ( comments_open() ) {
                comments_template();
            }
        }
        ?>
    </main>
    
    <?php get_sidebar(); ?>
</div>

<?php get_footer(); ?>
```

```php
<?php
// ==========================================
// archive.php - Archive Template
// ==========================================
get_header();
?>

<main>
    <header class="page-header">
        <?php
        the_archive_title( '<h1 class="page-title">', '</h1>' );
        the_archive_description( '<div class="archive-description">', '</div>' );
        ?>
    </header>
    
    <?php if ( have_posts() ) : ?>
        <div class="posts-grid">
            <?php
            while ( have_posts() ) {
                the_post();
                get_template_part( 'template-parts/content', get_post_type() );
            }
            ?>
        </div>
        
        <?php the_posts_pagination(); ?>
        
    <?php else : ?>
        <p>ไม่พบเนื้อหา</p>
    <?php endif; ?>
</main>

<?php get_footer(); ?>
```

```php
<?php
// ==========================================
// header.php - Header Template
// ==========================================
?>
<!DOCTYPE html>
<html <?php language_attributes(); ?>>
<head>
    <meta charset="<?php bloginfo( 'charset' ); ?>">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <?php wp_head(); ?> <!-- จำเป็น! Scripts/Styles จะถูก inject ที่นี่ -->
</head>

<body <?php body_class(); ?>>
    <?php wp_body_open(); ?> <!-- Hook สำหรับเพิ่ม Content หลัง <body> -->
    
    <header id="masthead">
        <div class="site-branding">
            <?php if ( has_custom_logo() ) : ?>
                <?php the_custom_logo(); ?>
            <?php else : ?>
                <h1 class="site-title">
                    <a href="<?php echo esc_url( home_url( '/' ) ); ?>">
                        <?php bloginfo( 'name' ); ?>
                    </a>
                </h1>
            <?php endif; ?>
        </div>
        
        <nav id="site-navigation">
            <?php
            wp_nav_menu( array(
                'theme_location' => 'primary',
                'menu_id'        => 'primary-menu',
                'menu_class'     => 'nav-menu',
            ) );
            ?>
        </nav>
    </header>
```

```php
<?php
// ==========================================
// footer.php - Footer Template
// ==========================================
?>
    <footer id="colophon">
        <div class="site-info">
            <p>&copy; <?php echo date('Y'); ?> <?php bloginfo('name'); ?></p>
            <?php wp_nav_menu( array( 'theme_location' => 'footer' ) ); ?>
        </div>
    </footer>
    
    <?php wp_footer(); ?> <!-- จำเป็น! Scripts จะถูก inject ที่นี่ -->
</body>
</html>
```

---

## 2. functions.php

`functions.php` คือไฟล์หัวใจของ Theme ใช้เพิ่มฟีเจอร์และกำหนดการทำงานของ Theme

```php
<?php
/**
 * My Theme Functions
 * 
 * @package MyTheme
 * @version 1.0.0
 */

// ป้องกันการเข้าถึงโดยตรง
if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

// ==========================================
// Theme Constants
// ==========================================

define( 'MY_THEME_VERSION', '1.0.0' );
define( 'MY_THEME_DIR', get_template_directory() );
define( 'MY_THEME_URI', get_template_directory_uri() );

// ==========================================
// Theme Setup
// ==========================================

function mytheme_setup() {
    
    // เปิดใช้งาน Translation
    load_theme_textdomain( 'mytheme', MY_THEME_DIR . '/languages' );
    
    // Add Default Posts and Comments RSS Feed Links
    add_theme_support( 'automatic-feed-links' );
    
    // Let WordPress manage <title> tag
    add_theme_support( 'title-tag' );
    
    // เปิดใช้งาน Post Thumbnails (Featured Images)
    add_theme_support( 'post-thumbnails' );
    
    // กำหนด Image Sizes
    add_image_size( 'mytheme-featured', 800, 450, true );   // Cropped
    add_image_size( 'mytheme-thumbnail', 400, 300, true );
    add_image_size( 'mytheme-square', 300, 300, true );
    add_image_size( 'mytheme-wide', 1200, 600, false );     // Not cropped
    
    // Register Navigation Menus
    register_nav_menus( array(
        'primary' => __( 'Primary Menu', 'mytheme' ),
        'footer'  => __( 'Footer Menu', 'mytheme' ),
        'social'  => __( 'Social Links Menu', 'mytheme' ),
    ) );
    
    // เปิดใช้งาน HTML5 Support
    add_theme_support( 'html5', array(
        'search-form',
        'comment-form',
        'comment-list',
        'gallery',
        'caption',
        'style',
        'script',
    ) );
    
    // Post Formats
    add_theme_support( 'post-formats', array(
        'aside',
        'gallery',
        'link',
        'image',
        'quote',
        'status',
        'video',
        'audio',
        'chat',
    ) );
    
    // Custom Logo
    add_theme_support( 'custom-logo', array(
        'height'      => 100,
        'width'       => 400,
        'flex-height' => true,
        'flex-width'  => true,
    ) );
    
    // Custom Background
    add_theme_support( 'custom-background', array(
        'default-color' => 'ffffff',
        'default-image' => '',
    ) );
    
    // Custom Header
    add_theme_support( 'custom-header', array(
        'default-image'          => '',
        'default-text-color'     => '000000',
        'width'                  => 1000,
        'height'                 => 250,
        'flex-width'             => true,
        'flex-height'            => true,
    ) );
    
    // WooCommerce Support
    add_theme_support( 'woocommerce' );
    add_theme_support( 'wc-product-gallery-zoom' );
    add_theme_support( 'wc-product-gallery-lightbox' );
    add_theme_support( 'wc-product-gallery-slider' );
    
    // Block Editor Settings
    add_theme_support( 'align-wide' );
    add_theme_support( 'editor-styles' );
    add_editor_style( 'assets/css/editor-style.css' );
    
    // Responsive Embeds
    add_theme_support( 'responsive-embeds' );
}
add_action( 'after_setup_theme', 'mytheme_setup' );

// ==========================================
// Register Sidebars/Widget Areas
// ==========================================

function mytheme_widgets_init() {
    
    // Primary Sidebar
    register_sidebar( array(
        'name'          => __( 'Primary Sidebar', 'mytheme' ),
        'id'            => 'sidebar-1',
        'description'   => __( 'Add widgets here.', 'mytheme' ),
        'before_widget' => '<section id="%1$s" class="widget %2$s">',
        'after_widget'  => '</section>',
        'before_title'  => '<h2 class="widget-title">',
        'after_title'   => '</h2>',
    ) );
    
    // Footer Widget Areas
    for ( $i = 1; $i <= 4; $i++ ) {
        register_sidebar( array(
            'name'          => sprintf( __( 'Footer Widget %d', 'mytheme' ), $i ),
            'id'            => 'footer-' . $i,
            'before_widget' => '<div class="footer-widget %2$s">',
            'after_widget'  => '</div>',
            'before_title'  => '<h3 class="widget-title">',
            'after_title'   => '</h3>',
        ) );
    }
    
    // Shop Sidebar (WooCommerce)
    register_sidebar( array(
        'name'          => __( 'Shop Sidebar', 'mytheme' ),
        'id'            => 'sidebar-shop',
        'before_widget' => '<div class="widget %2$s">',
        'after_widget'  => '</div>',
        'before_title'  => '<h3>',
        'after_title'   => '</h3>',
    ) );
}
add_action( 'widgets_init', 'mytheme_widgets_init' );
```

---

## 3. Enqueue Scripts and Styles

```php
<?php
/**
 * Enqueue Scripts และ Styles อย่างถูกต้อง
 */

// ==========================================
// Frontend Assets
// ==========================================

function mytheme_enqueue_scripts() {
    
    // ==========================================
    // Stylesheets
    // ==========================================
    
    // Main Stylesheet (style.css ใน root ของ Theme)
    wp_enqueue_style(
        'mytheme-style',                           // Handle
        get_stylesheet_uri(),                      // URL
        array(),                                   // Dependencies
        MY_THEME_VERSION,                          // Version
        'all'                                      // Media
    );
    
    // Google Fonts
    wp_enqueue_style(
        'mytheme-google-fonts',
        'https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;600;700&family=Prompt:wght@400;600&display=swap',
        array(),
        null
    );
    
    // Custom CSS
    wp_enqueue_style(
        'mytheme-custom',
        MY_THEME_URI . '/assets/css/main.css',
        array( 'mytheme-style' ),
        MY_THEME_VERSION
    );
    
    // ==========================================
    // JavaScript
    // ==========================================
    
    // jQuery ถูก Bundle มากับ WordPress
    // wp_enqueue_script( 'jquery' ); // โหลด jQuery
    
    // Slick Slider (CDN)
    wp_enqueue_script(
        'slick-slider',
        'https://cdn.jsdelivr.net/npm/slick-carousel@1.8.1/slick/slick.min.js',
        array( 'jquery' ),
        '1.8.1',
        true  // โหลดใน footer
    );
    
    // Main JavaScript
    wp_enqueue_script(
        'mytheme-main',
        MY_THEME_URI . '/assets/js/main.js',
        array( 'jquery', 'slick-slider' ),
        MY_THEME_VERSION,
        true
    );
    
    // ==========================================
    // Pass PHP data to JavaScript
    // ==========================================
    
    wp_localize_script(
        'mytheme-main',        // ต้องใช้ handle เดียวกับ script
        'mythemeVars',         // ชื่อ Object ใน JavaScript
        array(
            'ajaxUrl'   => admin_url( 'admin-ajax.php' ),
            'homeUrl'   => home_url(),
            'nonce'     => wp_create_nonce( 'mytheme-nonce' ),
            'themeUrl'  => MY_THEME_URI,
            'isLoggedIn' => is_user_logged_in(),
            'currentUser' => get_current_user_id(),
            'strings'   => array(
                'loading'  => __( 'กำลังโหลด...', 'mytheme' ),
                'error'    => __( 'เกิดข้อผิดพลาด', 'mytheme' ),
                'success'  => __( 'สำเร็จ!', 'mytheme' ),
            ),
        )
    );
    
    // ==========================================
    // Conditional Loading
    // ==========================================
    
    // โหลดเฉพาะหน้า Single Post
    if ( is_single() ) {
        wp_enqueue_script(
            'mytheme-comments',
            MY_THEME_URI . '/assets/js/comments.js',
            array( 'jquery' ),
            MY_THEME_VERSION,
            true
        );
    }
    
    // โหลดเฉพาะหน้า WooCommerce
    if ( function_exists( 'is_woocommerce' ) && is_woocommerce() ) {
        wp_enqueue_style(
            'mytheme-woocommerce',
            MY_THEME_URI . '/assets/css/woocommerce.css',
            array(),
            MY_THEME_VERSION
        );
    }
    
    // Comment Reply Script (เฉพาะหน้าที่มี comments)
    if ( is_singular() && comments_open() && get_option( 'thread_comments' ) ) {
        wp_enqueue_script( 'comment-reply' );
    }
}
add_action( 'wp_enqueue_scripts', 'mytheme_enqueue_scripts' );

// ==========================================
// Admin Assets
// ==========================================

function mytheme_admin_scripts( $hook ) {
    
    // โหลดเฉพาะหน้า Post Edit
    if ( in_array( $hook, array( 'post.php', 'post-new.php' ) ) ) {
        wp_enqueue_style(
            'mytheme-admin-post',
            MY_THEME_URI . '/assets/css/admin-post.css',
            array(),
            MY_THEME_VERSION
        );
        
        wp_enqueue_script(
            'mytheme-admin-post',
            MY_THEME_URI . '/assets/js/admin-post.js',
            array( 'jquery' ),
            MY_THEME_VERSION,
            true
        );
    }
    
    // โหลดเฉพาะหน้า Settings
    if ( $hook === 'settings_page_mytheme-settings' ) {
        wp_enqueue_media(); // WordPress Media Uploader
        wp_enqueue_script( 'wp-color-picker' );
        wp_enqueue_style( 'wp-color-picker' );
    }
}
add_action( 'admin_enqueue_scripts', 'mytheme_admin_scripts' );

// ==========================================
// Block Editor Assets (Gutenberg)
// ==========================================

function mytheme_block_editor_scripts() {
    wp_enqueue_script(
        'mytheme-editor',
        MY_THEME_URI . '/assets/js/editor.js',
        array( 'wp-blocks', 'wp-element', 'wp-editor' ),
        MY_THEME_VERSION
    );
    
    wp_enqueue_style(
        'mytheme-editor-styles',
        MY_THEME_URI . '/assets/css/editor.css',
        array( 'wp-edit-blocks' ),
        MY_THEME_VERSION
    );
}
add_action( 'enqueue_block_editor_assets', 'mytheme_block_editor_scripts' );
```

---

## 4. WordPress Hooks (Action & Filter)

### 4.1 Action Hooks

Action Hooks ให้เรา "เพิ่ม" โค้ดที่จุดต่างๆ ใน WordPress

```php
<?php
/**
 * Action Hooks
 * 
 * ใช้ add_action() เพื่อ "เพิ่ม" การกระทำ
 * Syntax: add_action( $hook, $callback, $priority, $accepted_args )
 */

// ==========================================
// ตัวอย่าง Action Hooks พื้นฐาน
// ==========================================

// เพิ่ม Meta Tags ใน <head>
add_action( 'wp_head', function() {
    echo '<meta name="theme-color" content="#ffffff">';
    echo '<link rel="preconnect" href="https://fonts.googleapis.com">';
} );

// เพิ่ม Content ก่อน Article
add_action( 'before_single_post', 'mytheme_before_post' );
function mytheme_before_post() {
    echo '<div class="post-notice">บทความนี้อาจมีการอัปเดต</div>';
}

// Custom Action Hook ของเรา
// ประกาศ Hook
function mytheme_custom_section() {
    do_action( 'mytheme_before_content' ); // ประกาศ Custom Action
    echo '<div class="content">';
    do_action( 'mytheme_inside_content' );
    echo '</div>';
    do_action( 'mytheme_after_content' );
}

// ใช้ Custom Hook ที่เราสร้าง
add_action( 'mytheme_before_content', function() {
    echo '<div class="breadcrumb">Home > Blog</div>';
}, 10 );

add_action( 'mytheme_before_content', function() {
    if ( is_single() ) {
        echo '<div class="social-share">Share: ...</div>';
    }
}, 20 ); // Priority 20 = ทำงานหลัง Priority 10

// ==========================================
// ลบ Action Hooks
// ==========================================

// ลบ WordPress default actions
remove_action( 'wp_head', 'wp_generator' );              // ซ่อน WP Version
remove_action( 'wp_head', 'rsd_link' );                  // RSD Link
remove_action( 'wp_head', 'wlwmanifest_link' );          // Manifest
remove_action( 'wp_head', 'wp_shortlink_wp_head' );      // Shortlink

// ลบ Emoji Scripts (ประหยัด Loading)
remove_action( 'wp_head', 'print_emoji_detection_script', 7 );
remove_action( 'wp_print_styles', 'print_emoji_styles' );
add_filter( 'emoji_svg_url', '__return_false' );

// ==========================================
// Hooks ในช่วง Save Post
// ==========================================

add_action( 'save_post', function( $post_id, $post, $update ) {
    
    // ไม่ทำงานกับ Auto-save
    if ( defined( 'DOING_AUTOSAVE' ) && DOING_AUTOSAVE ) {
        return;
    }
    
    // ตรวจสอบ Nonce
    if ( ! isset( $_POST['my_nonce'] ) || 
         ! wp_verify_nonce( $_POST['my_nonce'], 'save_my_meta' ) ) {
        return;
    }
    
    // ตรวจสอบสิทธิ์
    if ( ! current_user_can( 'edit_post', $post_id ) ) {
        return;
    }
    
    // บันทึกข้อมูล Custom Field
    if ( isset( $_POST['my_custom_field'] ) ) {
        update_post_meta(
            $post_id,
            '_my_custom_field',
            sanitize_text_field( $_POST['my_custom_field'] )
        );
    }
    
}, 10, 3 );
```

### 4.2 Filter Hooks

Filter Hooks ให้เรา "แก้ไข" ค่าที่ส่งผ่าน WordPress

```php
<?php
/**
 * Filter Hooks
 * 
 * ใช้ add_filter() เพื่อ "แก้ไข" ข้อมูล
 * Syntax: add_filter( $hook, $callback, $priority, $accepted_args )
 */

// ==========================================
// Filter ที่ใช้บ่อย
// ==========================================

// เปลี่ยน Excerpt Length
add_filter( 'excerpt_length', function( $length ) {
    return 25; // จำนวนคำ
} );

// เปลี่ยน Excerpt More Text
add_filter( 'excerpt_more', function( $more ) {
    return '... <a href="' . get_permalink() . '">' . 
           __( 'อ่านต่อ', 'mytheme' ) . '</a>';
} );

// เพิ่ม Custom Class ให้ Body
add_filter( 'body_class', function( $classes ) {
    if ( is_home() ) {
        $classes[] = 'blog-home';
    }
    
    if ( is_user_logged_in() ) {
        $classes[] = 'user-logged-in';
    }
    
    // เพิ่ม Browser Class
    global $is_chrome, $is_firefox, $is_safari;
    if ( $is_chrome ) {
        $classes[] = 'browser-chrome';
    }
    
    return $classes;
} );

// ==========================================
// Filter เนื้อหา Post
// ==========================================

// เพิ่ม Content หลัง Post
add_filter( 'the_content', function( $content ) {
    
    if ( is_single() && is_main_query() ) {
        $author_box = '<div class="author-box">';
        $author_box .= get_avatar( get_the_author_meta( 'email' ), 80 );
        $author_box .= '<div class="author-info">';
        $author_box .= '<h4>' . get_the_author() . '</h4>';
        $author_box .= '<p>' . get_the_author_meta( 'description' ) . '</p>';
        $author_box .= '</div></div>';
        
        $content .= $author_box;
    }
    
    return $content;
} );

// แปลง YouTube URL เป็น Embed
add_filter( 'the_content', function( $content ) {
    $pattern = '/https?:\/\/(?:www\.)?youtube\.com\/watch\?v=([a-zA-Z0-9_-]+)/';
    $replacement = '<div class="video-wrapper"><iframe src="https://www.youtube.com/embed/$1" allowfullscreen></iframe></div>';
    return preg_replace( $pattern, $replacement, $content );
} );

// ==========================================
// Filter Menu Items
// ==========================================

add_filter( 'wp_nav_menu_items', function( $items, $args ) {
    
    if ( $args->theme_location === 'primary' ) {
        // เพิ่ม Search Icon ใน Menu
        $items .= '<li class="menu-item menu-search">';
        $items .= '<a href="#" id="search-toggle"><span class="dashicons dashicons-search"></span></a>';
        $items .= '</li>';
        
        // เพิ่ม Login/Logout Link
        if ( is_user_logged_in() ) {
            $items .= '<li class="menu-item">';
            $items .= '<a href="' . wp_logout_url( home_url() ) . '">ออกจากระบบ</a>';
            $items .= '</li>';
        } else {
            $items .= '<li class="menu-item">';
            $items .= '<a href="' . wp_login_url( get_permalink() ) . '">เข้าสู่ระบบ</a>';
            $items .= '</li>';
        }
    }
    
    return $items;
    
}, 10, 2 );

// ==========================================
// Filter Mail Settings
// ==========================================

// เปลี่ยน From Email
add_filter( 'wp_mail_from', function( $email ) {
    return 'noreply@mysite.com';
} );

// เปลี่ยน From Name
add_filter( 'wp_mail_from_name', function( $name ) {
    return get_bloginfo( 'name' );
} );

// ==========================================
// Filter Login URL
// ==========================================

// เปลี่ยน URL หลัง Login
add_filter( 'login_redirect', function( $redirect_to, $request, $user ) {
    if ( isset( $user->roles ) && is_array( $user->roles ) ) {
        if ( in_array( 'administrator', $user->roles ) ) {
            return admin_url();
        } elseif ( in_array( 'editor', $user->roles ) ) {
            return admin_url( 'edit.php' );
        } else {
            return home_url( '/dashboard/' );
        }
    }
    return $redirect_to;
}, 10, 3 );
```

---

## 5. WordPress Loop และ Template Tags

```php
<?php
/**
 * WordPress Loop
 */

// ==========================================
// Main Loop
// ==========================================

if ( have_posts() ) :
    while ( have_posts() ) :
        the_post();
        ?>
        
        <article id="post-<?php the_ID(); ?>" <?php post_class( 'post-card' ); ?>>
            
            <!-- Featured Image -->
            <?php if ( has_post_thumbnail() ) : ?>
                <a href="<?php the_permalink(); ?>" class="post-thumbnail">
                    <?php the_post_thumbnail( 'mytheme-featured' ); ?>
                </a>
            <?php endif; ?>
            
            <!-- Post Meta -->
            <div class="post-header">
                <div class="post-meta">
                    <!-- Author -->
                    <span class="author">
                        <?php the_author_posts_link(); ?>
                    </span>
                    
                    <!-- Date -->
                    <time datetime="<?php echo get_the_date( 'Y-m-d' ); ?>">
                        <?php echo get_the_date( 'd M Y' ); ?>
                    </time>
                    
                    <!-- Category -->
                    <?php the_category( ' | ' ); ?>
                    
                    <!-- Reading Time -->
                    <?php
                    $content   = get_the_content();
                    $word_count = str_word_count( strip_tags( $content ) );
                    $reading_time = ceil( $word_count / 200 );
                    echo '<span>' . $reading_time . ' นาที</span>';
                    ?>
                </div>
                
                <!-- Title -->
                <?php the_title( '<h2 class="post-title"><a href="' . esc_url( get_permalink() ) . '">', '</a></h2>' ); ?>
            </div>
            
            <!-- Excerpt -->
            <div class="post-excerpt">
                <?php the_excerpt(); ?>
            </div>
            
            <!-- Tags -->
            <?php if ( has_tag() ) : ?>
                <div class="post-tags">
                    <?php the_tags( '', ' ', '' ); ?>
                </div>
            <?php endif; ?>
            
        </article>
        
        <?php
    endwhile;
    
    // Pagination
    the_posts_pagination( array(
        'mid_size'  => 2,
        'prev_text' => '&larr; หน้าก่อน',
        'next_text' => 'หน้าถัดไป &rarr;',
    ) );
    
else :
    echo '<p>ไม่พบบทความ</p>';
endif;

// ==========================================
// Custom Loop ด้วย WP_Query
// ==========================================

$custom_query = new WP_Query( array(
    'post_type'      => 'post',
    'posts_per_page' => 5,
    'post_status'    => 'publish',
    'category_name'  => 'featured',
    'meta_key'       => '_featured_post',
    'meta_value'     => '1',
    'orderby'        => 'date',
    'order'          => 'DESC',
) );

if ( $custom_query->have_posts() ) :
    while ( $custom_query->have_posts() ) :
        $custom_query->the_post();
        // แสดงผล
        echo '<h3>' . get_the_title() . '</h3>';
    endwhile;
    wp_reset_postdata(); // สำคัญ! Reset global $post
endif;
```

---

## 6. Template Parts

```php
<?php
// ==========================================
// สร้าง Template Part
// ==========================================
// File: template-parts/content.php

/**
 * Template Part: Content
 * 
 * เรียกใช้งานด้วย: get_template_part( 'template-parts/content' )
 * หรือ: get_template_part( 'template-parts/content', 'post' )
 *   -> โหลด template-parts/content-post.php หรือ template-parts/content.php
 */
?>

<article id="post-<?php the_ID(); ?>" <?php post_class(); ?>>
    
    <?php if ( has_post_thumbnail() ) : ?>
        <figure class="post-thumbnail">
            <?php the_post_thumbnail( 'large' ); ?>
        </figure>
    <?php endif; ?>
    
    <header class="entry-header">
        <?php the_title( '<h2 class="entry-title"><a href="' . esc_url( get_permalink() ) . '">', '</a></h2>' ); ?>
        <?php // Template Meta ?>
        <div class="entry-meta">
            Posted on <?php the_date(); ?> by <?php the_author(); ?>
        </div>
    </header>
    
    <div class="entry-content">
        <?php the_excerpt(); ?>
    </div>
    
    <footer class="entry-footer">
        <a href="<?php the_permalink(); ?>" class="read-more">
            <?php esc_html_e( 'อ่านต่อ', 'mytheme' ); ?>
        </a>
    </footer>
    
</article>
```

```php
<?php
// ==========================================
// Passing Data to Template Parts
// ==========================================

// WordPress 5.5+ รองรับการส่ง Arguments
get_template_part(
    'template-parts/card',
    null,
    array(
        'title'    => 'Custom Title',
        'excerpt'  => 'Custom excerpt...',
        'image_id' => 123,
    )
);

// File: template-parts/card.php
$args    = isset( $args ) ? $args : array();
$title   = $args['title'] ?? get_the_title();
$excerpt = $args['excerpt'] ?? get_the_excerpt();

echo '<div class="card">';
echo '<h3>' . esc_html( $title ) . '</h3>';
echo '<p>' . esc_html( $excerpt ) . '</p>';
echo '</div>';
```

---

## Workshop: สร้าง Basic Theme

### Exercise 1: สร้าง Theme Structure

```bash
# สร้างโครงสร้าง Theme
mkdir -p wp-content/themes/mytheme/{assets/{css,js,images},template-parts,languages}

# ไฟล์ที่ต้องสร้าง:
touch wp-content/themes/mytheme/style.css
touch wp-content/themes/mytheme/index.php
touch wp-content/themes/mytheme/functions.php
touch wp-content/themes/mytheme/header.php
touch wp-content/themes/mytheme/footer.php
touch wp-content/themes/mytheme/sidebar.php
touch wp-content/themes/mytheme/single.php
touch wp-content/themes/mytheme/page.php
touch wp-content/themes/mytheme/archive.php
touch wp-content/themes/mytheme/search.php
touch wp-content/themes/mytheme/404.php
touch wp-content/themes/mytheme/template-parts/content.php
```

### Exercise 2: style.css Theme Header

```css
/*
Theme Name: My Theme
Theme URI: https://example.com
Author: Your Name
Author URI: https://example.com
Description: Custom WordPress Theme
Version: 1.0.0
License: GPL-2.0-or-later
License URI: https://www.gnu.org/licenses/gpl-2.0.html
Text Domain: mytheme
Tags: custom-background, custom-logo, custom-menu, featured-images
*/
```

### Exercise 3: สร้าง functions.php ที่สมบูรณ์

สร้าง functions.php ที่มีฟีเจอร์:
1. Theme Support (post-thumbnails, title-tag, menus)
2. Enqueue main.css และ main.js
3. Register Primary Sidebar
4. เพิ่ม Custom Image Size 800x500

---

## Quiz

**คำถามที่ 1:** WordPress Template Hierarchy ใดที่โหลดสำหรับหน้า Category ที่ชื่อ "news"?

A) news.php  
B) category-news.php  
C) archive-news.php  
D) taxonomy-news.php  

**เฉลย: B) category-news.php** (ตามลำดับ: category-{slug}.php -> category-{id}.php -> category.php -> archive.php)

---

**คำถามที่ 2:** ฟังก์ชันใดถูกต้องในการโหลด Script ใน WordPress?

A) `include_script('script.js')`  
B) `wp_enqueue_script('handle', 'url', array(), '1.0', true)`  
C) `add_script('handle', 'url')`  
D) `load_script('script.js')`  

**เฉลย: B) wp_enqueue_script**

---

**คำถามที่ 3:** ความแตกต่างระหว่าง Action Hook และ Filter Hook คืออะไร?

A) Action เปลี่ยนข้อมูล, Filter เพิ่มโค้ด  
B) Action เพิ่มโค้ด, Filter แก้ไข/คืนค่าข้อมูล  
C) ไม่มีความแตกต่าง  
D) Action ทำงานเร็วกว่า Filter  

**เฉลย: B) Action เพิ่มโค้ดได้, Filter ต้องคืนค่า (return) ที่แก้ไขแล้ว**

---

**คำถามที่ 4:** `wp_reset_postdata()` ใช้ทำอะไร?

A) ลบ Post ทั้งหมดจาก Database  
B) Reset global `$post` หลังใช้ WP_Query  
C) Clear Cache ของ Post  
D) Reload หน้าเว็บ  

**เฉลย: B) Reset global $post กลับไปเป็น Post ปัจจุบัน หลังใช้ Custom WP_Query**

---

**คำถามที่ 5:** `wp_footer()` ต้องอยู่ที่ไหน?

A) ใน header.php  
B) ใน functions.php  
C) ก่อน `</body>` ใน footer.php  
D) ใน index.php  

**เฉลย: C) ก่อน `</body>` ใน footer.php - จำเป็นสำหรับ Scripts**

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- Template Hierarchy และลำดับการโหลด Template
- การสร้างและตั้งค่า functions.php
- การ Enqueue Scripts/Styles อย่างถูกต้อง
- Action Hooks และ Filter Hooks
- WordPress Loop และ Template Tags
- Template Parts

---

## ต่อไป

➡️ **[Part 053: WordPress Theme Advanced](part-053-wordpress-theme-advanced.md)**

เรียนรู้เกี่ยวกับ:
- Custom Page Templates
- Walker Classes สำหรับ Custom Navigation
- Theme Options
- Advanced Customization
