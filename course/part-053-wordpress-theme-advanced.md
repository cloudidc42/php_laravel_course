# Part 053: WordPress Theme Advanced

**ระดับ:** Advanced  
**เวลาเรียน:** 5-6 ชั่วโมง  
**Prerequisites:** Part 052 (WordPress Theme Basics)

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. สร้าง Custom Page Templates
2. สร้าง Custom Navigation ด้วย Walker Classes
3. ใช้งาน WordPress Customizer API
4. สร้าง Mega Menu และ Custom Walker
5. จัดการ Theme Options อย่างมืออาชีพ

---

## 1. Custom Page Templates

Custom Page Templates ช่วยให้เราสร้าง Layout พิเศษสำหรับหน้าเฉพาะ

```php
<?php
/**
 * Template Name: Full Width Page
 * Template Post Type: page
 * 
 * @package MyTheme
 */

get_header();
?>

<main id="primary" class="full-width">
    <?php
    while ( have_posts() ) {
        the_post();
        ?>
        <article id="page-<?php the_ID(); ?>" <?php post_class(); ?>>
            <?php the_title( '<h1>', '</h1>' ); ?>
            <div class="entry-content">
                <?php the_content(); ?>
            </div>
        </article>
        <?php
    }
    ?>
</main>
<!-- ไม่มี Sidebar -->

<?php get_footer(); ?>
```

```php
<?php
/**
 * Template Name: Landing Page
 * Template Post Type: page
 * 
 * Template พิเศษสำหรับ Landing Page
 * ไม่มี Header/Footer ของ Theme
 */

// ปิด Header ปกติ
?>
<!DOCTYPE html>
<html <?php language_attributes(); ?>>
<head>
    <meta charset="<?php bloginfo('charset'); ?>">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <?php wp_head(); ?>
    <style>
        body { margin: 0; padding: 0; }
        .hero-section { min-height: 100vh; }
    </style>
</head>
<body <?php body_class('landing-page'); ?>>
<?php wp_body_open(); ?>

<div class="landing-wrapper">
    <!-- Hero Section -->
    <section class="hero-section">
        <?php the_post_thumbnail('full'); ?>
        <div class="hero-content">
            <?php the_title('<h1>', '</h1>'); ?>
            <?php the_content(); ?>
        </div>
    </section>
    
    <!-- Custom Sections จาก ACF Fields -->
    <?php
    // ใช้ ACF หรือ Custom Meta
    $sections = get_post_meta( get_the_ID(), '_landing_sections', true );
    if ( $sections ) :
        foreach ( $sections as $section ) : ?>
            <section class="content-section <?php echo esc_attr($section['layout']); ?>">
                <h2><?php echo esc_html($section['title']); ?></h2>
                <?php echo wp_kses_post($section['content']); ?>
            </section>
        <?php endforeach;
    endif;
    ?>
</div>

<?php wp_footer(); ?>
</body>
</html>
```

```php
<?php
/**
 * Template Name: Portfolio
 * Template Post Type: page
 */
get_header();

// ดึง Portfolio Items
$portfolio_args = array(
    'post_type'      => 'portfolio',
    'posts_per_page' => -1,
    'orderby'        => 'menu_order',
    'order'          => 'ASC',
    'meta_query'     => array(
        array(
            'key'     => '_featured',
            'value'   => '1',
            'compare' => '=',
        ),
    ),
);

$portfolio_query = new WP_Query( $portfolio_args );
?>

<main class="portfolio-page">
    <?php the_title('<h1>', '</h1>'); ?>
    
    <!-- Filter Buttons -->
    <?php
    $categories = get_terms( array(
        'taxonomy' => 'portfolio_category',
        'hide_empty' => true,
    ) );
    ?>
    
    <div class="portfolio-filter">
        <button class="filter-btn active" data-filter="all">ทั้งหมด</button>
        <?php foreach ( $categories as $cat ) : ?>
            <button class="filter-btn" data-filter="<?php echo esc_attr($cat->slug); ?>">
                <?php echo esc_html($cat->name); ?>
            </button>
        <?php endforeach; ?>
    </div>
    
    <!-- Portfolio Grid -->
    <div class="portfolio-grid">
        <?php
        while ( $portfolio_query->have_posts() ) {
            $portfolio_query->the_post();
            $cats = wp_get_post_terms( get_the_ID(), 'portfolio_category', array('fields' => 'slugs') );
            $cat_classes = implode(' ', $cats);
            ?>
            
            <div class="portfolio-item <?php echo esc_attr($cat_classes); ?>">
                <?php if ( has_post_thumbnail() ) : ?>
                    <figure>
                        <?php the_post_thumbnail('medium_large'); ?>
                    </figure>
                <?php endif; ?>
                
                <div class="portfolio-info">
                    <?php the_title('<h3>', '</h3>'); ?>
                    <?php the_excerpt(); ?>
                    <a href="<?php the_permalink(); ?>">ดูเพิ่มเติม</a>
                </div>
            </div>
            
            <?php
        }
        wp_reset_postdata();
        ?>
    </div>
</main>

<?php get_footer(); ?>
```

---

## 2. Walker Classes สำหรับ Custom Navigation

Walker Class ใช้สำหรับ Customize HTML Output ของ WordPress Menus และ Lists

### 2.1 Custom Nav Walker

```php
<?php
/**
 * Custom Navigation Walker
 * 
 * สร้าง Bootstrap 5 Compatible Menu
 */

class Bootstrap_Nav_Walker extends Walker_Nav_Menu {
    
    /**
     * Starts the list before the elements are added.
     * 
     * @param string   $output  ตัวแปรเก็บ HTML Output
     * @param int      $depth   ระดับความลึกของ Menu
     * @param stdClass $args    Menu Arguments
     */
    public function start_lvl( &$output, $depth = 0, $args = null ) {
        
        if ( isset( $args->item_spacing ) && 'discard' === $args->item_spacing ) {
            $t = '';
            $n = '';
        } else {
            $t = "\t";
            $n = "\n";
        }
        
        $indent = str_repeat( $t, $depth );
        
        // เพิ่ม Bootstrap Dropdown Classes
        $classes   = array( 'dropdown-menu' );
        $class_names = implode( ' ', apply_filters( 'nav_menu_submenu_css_class', $classes, $args, $depth ) );
        $atts  = '';
        $atts .= ! empty( $class_names ) ? ' class="' . esc_attr( $class_names ) . '"' : '';
        
        $output .= "{$n}{$indent}<ul{$atts}>{$n}";
    }
    
    /**
     * Starts the element output.
     */
    public function start_el( &$output, $item, $depth = 0, $args = null, $id = 0 ) {
        
        if ( isset( $args->item_spacing ) && 'discard' === $args->item_spacing ) {
            $t = '';
            $n = '';
        } else {
            $t = "\t";
            $n = "\n";
        }
        
        $indent = ( $depth ) ? str_repeat( $t, $depth ) : '';
        
        $classes   = empty( $item->classes ) ? array() : (array) $item->classes;
        $classes[] = 'nav-item';
        
        // เช็คว่ามี Submenu ไหม
        $has_children = in_array( 'menu-item-has-children', $classes );
        if ( $has_children ) {
            $classes[] = 'dropdown';
        }
        
        $class_names = implode( ' ', apply_filters( 'nav_menu_css_class', array_filter( $classes ), $item, $args, $depth ) );
        $class_names = $class_names ? ' class="' . esc_attr( $class_names ) . '"' : '';
        
        $id = apply_filters( 'nav_menu_item_id', 'menu-item-' . $item->ID, $item, $args, $depth );
        $id = $id ? ' id="' . esc_attr( $id ) . '"' : '';
        
        $output .= $indent . '<li' . $id . $class_names . '>';
        
        $atts           = array();
        $atts['title']  = ! empty( $item->attr_title ) ? $item->attr_title : '';
        $atts['target'] = ! empty( $item->target ) ? $item->target : '';
        
        if ( '_blank' === $item->target && empty( $item->xfn ) ) {
            $atts['rel'] = 'noopener noreferrer';
        } else {
            $atts['rel'] = $item->xfn;
        }
        
        $atts['href']         = ! empty( $item->url ) ? $item->url : '';
        $atts['aria-current'] = $item->current ? 'page' : '';
        
        // Bootstrap Classes สำหรับ Link
        if ( $depth === 0 ) {
            $atts['class'] = 'nav-link';
            if ( $has_children ) {
                $atts['class']           .= ' dropdown-toggle';
                $atts['data-bs-toggle']   = 'dropdown';
                $atts['aria-expanded']    = 'false';
                $atts['role']             = 'button';
            }
        } else {
            $atts['class'] = 'dropdown-item';
        }
        
        $atts = apply_filters( 'nav_menu_link_attributes', $atts, $item, $args, $depth );
        
        $attributes = '';
        foreach ( $atts as $attr => $value ) {
            if ( is_scalar( $value ) && '' !== $value && false !== $value ) {
                $value       = ( 'href' === $attr ) ? esc_url( $value ) : esc_attr( $value );
                $attributes .= ' ' . $attr . '="' . $value . '"';
            }
        }
        
        $title = apply_filters( 'the_title', $item->title, $item->ID );
        $title = apply_filters( 'nav_menu_item_title', $title, $item, $args, $depth );
        
        $item_output  = isset( $args->before ) ? $args->before : '';
        $item_output .= '<a' . $attributes . '>';
        $item_output .= ( isset( $args->link_before ) ? $args->link_before : '' ) . $title . ( isset( $args->link_after ) ? $args->link_after : '' );
        
        // เพิ่ม Dropdown Arrow
        if ( $has_children && $depth === 0 ) {
            $item_output .= ' <svg class="dropdown-arrow" width="10" height="10" viewBox="0 0 10 10"><path d="M0 3l5 5 5-5z" fill="currentColor"/></svg>';
        }
        
        $item_output .= '</a>';
        $item_output .= isset( $args->after ) ? $args->after : '';
        
        $output .= apply_filters( 'walker_nav_menu_start_el', $item_output, $item, $depth, $args );
    }
}
```

### 2.2 การใช้งาน Custom Walker

```php
<?php
// ใช้ Bootstrap Walker ใน Template
wp_nav_menu( array(
    'theme_location' => 'primary',
    'container'      => 'nav',
    'container_class' => 'navbar-nav',
    'menu_class'     => 'nav',
    'walker'         => new Bootstrap_Nav_Walker(),
    'fallback_cb'    => 'Bootstrap_Nav_Walker::fallback',
) );
```

### 2.3 Mega Menu Walker

```php
<?php
/**
 * Mega Menu Walker
 * 
 * สร้าง Mega Menu ที่แสดง Sub-Items แบบ Grid
 */

class Mega_Menu_Walker extends Walker_Nav_Menu {
    
    private $mega_menu_items = array();
    
    public function start_lvl( &$output, $depth = 0, $args = null ) {
        if ( $depth === 0 ) {
            // First Level = Mega Menu
            $output .= '<div class="mega-menu">';
            $output .= '<div class="mega-menu-inner">';
        } else {
            $output .= '<ul class="sub-menu">';
        }
    }
    
    public function end_lvl( &$output, $depth = 0, $args = null ) {
        if ( $depth === 0 ) {
            $output .= '</div></div>';
        } else {
            $output .= '</ul>';
        }
    }
    
    public function start_el( &$output, $item, $depth = 0, $args = null, $id = 0 ) {
        $classes      = (array) $item->classes;
        $is_mega      = in_array( 'mega-menu', $classes );
        $has_children = in_array( 'menu-item-has-children', $classes );
        
        $li_class = implode(' ', $classes);
        
        if ( $depth === 0 ) {
            $output .= '<li class="' . esc_attr($li_class) . '">';
            $output .= '<a href="' . esc_url($item->url) . '" class="nav-link">';
            $output .= esc_html($item->title);
            if ( $has_children ) {
                $output .= '<span class="arrow">▼</span>';
            }
            $output .= '</a>';
        } else {
            $output .= '<li>';
            $output .= '<a href="' . esc_url($item->url) . '">';
            $output .= esc_html($item->title);
            $output .= '</a>';
        }
    }
    
    public function end_el( &$output, $item, $depth = 0, $args = null ) {
        $output .= '</li>';
    }
}
```

---

## 3. WordPress Customizer API

```php
<?php
/**
 * WordPress Customizer API
 * 
 * ให้ User ปรับแต่ง Theme ได้แบบ Real-time
 */

function mytheme_customize_register( $wp_customize ) {
    
    // ==========================================
    // เพิ่ม Panel (กลุ่มของ Section)
    // ==========================================
    
    $wp_customize->add_panel( 'mytheme_options', array(
        'title'       => __( 'Theme Options', 'mytheme' ),
        'description' => __( 'ตั้งค่า Theme', 'mytheme' ),
        'priority'    => 10,
    ) );
    
    // ==========================================
    // Header Settings Section
    // ==========================================
    
    $wp_customize->add_section( 'mytheme_header', array(
        'title'    => __( 'Header Settings', 'mytheme' ),
        'panel'    => 'mytheme_options',
        'priority' => 10,
    ) );
    
    // Header Background Color
    $wp_customize->add_setting( 'header_bg_color', array(
        'default'           => '#ffffff',
        'transport'         => 'postMessage', // Live Preview
        'sanitize_callback' => 'sanitize_hex_color',
    ) );
    
    $wp_customize->add_control( new WP_Customize_Color_Control(
        $wp_customize,
        'header_bg_color',
        array(
            'label'   => __( 'Header Background Color', 'mytheme' ),
            'section' => 'mytheme_header',
        )
    ) );
    
    // Header Height
    $wp_customize->add_setting( 'header_height', array(
        'default'           => '80',
        'transport'         => 'postMessage',
        'sanitize_callback' => 'absint',
    ) );
    
    $wp_customize->add_control( 'header_height', array(
        'type'        => 'range',
        'label'       => __( 'Header Height (px)', 'mytheme' ),
        'section'     => 'mytheme_header',
        'input_attrs' => array(
            'min'  => 50,
            'max'  => 200,
            'step' => 5,
        ),
    ) );
    
    // Sticky Header Toggle
    $wp_customize->add_setting( 'sticky_header', array(
        'default'           => false,
        'transport'         => 'refresh',
        'sanitize_callback' => 'wp_validate_boolean',
    ) );
    
    $wp_customize->add_control( 'sticky_header', array(
        'type'    => 'checkbox',
        'label'   => __( 'Enable Sticky Header', 'mytheme' ),
        'section' => 'mytheme_header',
    ) );
    
    // ==========================================
    // Typography Section
    // ==========================================
    
    $wp_customize->add_section( 'mytheme_typography', array(
        'title' => __( 'Typography', 'mytheme' ),
        'panel' => 'mytheme_options',
    ) );
    
    // Body Font
    $wp_customize->add_setting( 'body_font', array(
        'default'           => 'Sarabun',
        'transport'         => 'postMessage',
        'sanitize_callback' => 'sanitize_text_field',
    ) );
    
    $wp_customize->add_control( 'body_font', array(
        'type'    => 'select',
        'label'   => __( 'Body Font', 'mytheme' ),
        'section' => 'mytheme_typography',
        'choices' => array(
            'Sarabun'  => 'Sarabun',
            'Prompt'   => 'Prompt',
            'Kanit'    => 'Kanit',
            'Mitr'     => 'Mitr',
            'Noto Sans Thai' => 'Noto Sans Thai',
        ),
    ) );
    
    // Font Size
    $wp_customize->add_setting( 'base_font_size', array(
        'default'           => '16',
        'transport'         => 'postMessage',
        'sanitize_callback' => 'absint',
    ) );
    
    $wp_customize->add_control( 'base_font_size', array(
        'type'        => 'range',
        'label'       => __( 'Base Font Size (px)', 'mytheme' ),
        'section'     => 'mytheme_typography',
        'input_attrs' => array(
            'min'  => 12,
            'max'  => 24,
            'step' => 1,
        ),
    ) );
    
    // ==========================================
    // Footer Section
    // ==========================================
    
    $wp_customize->add_section( 'mytheme_footer', array(
        'title' => __( 'Footer Settings', 'mytheme' ),
        'panel' => 'mytheme_options',
    ) );
    
    // Footer Text
    $wp_customize->add_setting( 'footer_text', array(
        'default'           => 'Copyright © 2024. All rights reserved.',
        'transport'         => 'postMessage',
        'sanitize_callback' => 'wp_kses_post',
    ) );
    
    $wp_customize->add_control( 'footer_text', array(
        'type'    => 'textarea',
        'label'   => __( 'Footer Copyright Text', 'mytheme' ),
        'section' => 'mytheme_footer',
    ) );
    
    // Footer Background Color
    $wp_customize->add_setting( 'footer_bg_color', array(
        'default'           => '#333333',
        'sanitize_callback' => 'sanitize_hex_color',
    ) );
    
    $wp_customize->add_control( new WP_Customize_Color_Control(
        $wp_customize,
        'footer_bg_color',
        array(
            'label'   => __( 'Footer Background Color', 'mytheme' ),
            'section' => 'mytheme_footer',
        )
    ) );
    
    // Custom Logo (เพิ่มเติมจาก default)
    $wp_customize->add_setting( 'footer_logo', array(
        'default'           => '',
        'sanitize_callback' => 'absint',
    ) );
    
    $wp_customize->add_control( new WP_Customize_Media_Control(
        $wp_customize,
        'footer_logo',
        array(
            'label'     => __( 'Footer Logo', 'mytheme' ),
            'section'   => 'mytheme_footer',
            'mime_type' => 'image',
        )
    ) );
    
    // ==========================================
    // Social Media Section
    // ==========================================
    
    $wp_customize->add_section( 'mytheme_social', array(
        'title' => __( 'Social Media', 'mytheme' ),
        'panel' => 'mytheme_options',
    ) );
    
    $social_networks = array(
        'facebook'  => 'Facebook URL',
        'twitter'   => 'Twitter/X URL',
        'instagram' => 'Instagram URL',
        'youtube'   => 'YouTube URL',
        'linkedin'  => 'LinkedIn URL',
        'line'      => 'LINE URL',
    );
    
    foreach ( $social_networks as $key => $label ) {
        $wp_customize->add_setting( 'social_' . $key, array(
            'default'           => '',
            'sanitize_callback' => 'esc_url_raw',
        ) );
        
        $wp_customize->add_control( 'social_' . $key, array(
            'type'    => 'url',
            'label'   => $label,
            'section' => 'mytheme_social',
        ) );
    }
}
add_action( 'customize_register', 'mytheme_customize_register' );

// ==========================================
// Live Preview JavaScript Bindings
// ==========================================

function mytheme_customize_preview_js() {
    wp_enqueue_script(
        'mytheme-customizer',
        MY_THEME_URI . '/assets/js/customizer.js',
        array( 'customize-preview' ),
        MY_THEME_VERSION,
        true
    );
}
add_action( 'customize_preview_init', 'mytheme_customize_preview_js' );

// ==========================================
// Output Custom CSS จาก Customizer Settings
// ==========================================

function mytheme_customizer_css() {
    $header_bg    = get_theme_mod( 'header_bg_color', '#ffffff' );
    $header_height = get_theme_mod( 'header_height', 80 );
    $footer_bg    = get_theme_mod( 'footer_bg_color', '#333333' );
    $body_font    = get_theme_mod( 'body_font', 'Sarabun' );
    $font_size    = get_theme_mod( 'base_font_size', 16 );
    
    ?>
    <style id="mytheme-customizer-css">
        :root {
            --header-bg: <?php echo esc_attr( $header_bg ); ?>;
            --header-height: <?php echo absint( $header_height ); ?>px;
            --footer-bg: <?php echo esc_attr( $footer_bg ); ?>;
            --body-font: '<?php echo esc_attr( $body_font ); ?>', sans-serif;
            --base-font-size: <?php echo absint( $font_size ); ?>px;
        }
        
        #masthead {
            background-color: var(--header-bg);
            min-height: var(--header-height);
        }
        
        body {
            font-family: var(--body-font);
            font-size: var(--base-font-size);
        }
        
        #colophon {
            background-color: var(--footer-bg);
        }
    </style>
    <?php
}
add_action( 'wp_head', 'mytheme_customizer_css' );
```

---

## 4. Custom Theme Options Page

```php
<?php
/**
 * Theme Options Page ใน Admin
 */

class Mytheme_Options {
    
    private $options;
    
    public function __construct() {
        $this->options = get_option( 'mytheme_options', array() );
        add_action( 'admin_menu', array( $this, 'add_options_page' ) );
        add_action( 'admin_init', array( $this, 'register_settings' ) );
    }
    
    public function add_options_page() {
        add_theme_page(
            __( 'Theme Options', 'mytheme' ),
            __( 'Theme Options', 'mytheme' ),
            'manage_options',
            'mytheme-options',
            array( $this, 'render_options_page' )
        );
    }
    
    public function register_settings() {
        register_setting(
            'mytheme_options_group',
            'mytheme_options',
            array( $this, 'sanitize_options' )
        );
        
        // General Section
        add_settings_section(
            'mytheme_general',
            __( 'General Settings', 'mytheme' ),
            null,
            'mytheme-options'
        );
        
        add_settings_field(
            'google_analytics_id',
            __( 'Google Analytics ID', 'mytheme' ),
            array( $this, 'render_text_field' ),
            'mytheme-options',
            'mytheme_general',
            array( 'key' => 'google_analytics_id' )
        );
        
        add_settings_field(
            'posts_per_page',
            __( 'Posts Per Page', 'mytheme' ),
            array( $this, 'render_number_field' ),
            'mytheme-options',
            'mytheme_general',
            array( 'key' => 'posts_per_page', 'min' => 1, 'max' => 100 )
        );
    }
    
    public function render_text_field( $args ) {
        $key   = $args['key'];
        $value = isset( $this->options[ $key ] ) ? $this->options[ $key ] : '';
        echo '<input type="text" name="mytheme_options[' . esc_attr($key) . ']" value="' . esc_attr($value) . '" class="regular-text">';
    }
    
    public function render_number_field( $args ) {
        $key   = $args['key'];
        $value = isset( $this->options[ $key ] ) ? $this->options[ $key ] : '';
        $min   = isset( $args['min'] ) ? $args['min'] : '';
        $max   = isset( $args['max'] ) ? $args['max'] : '';
        echo '<input type="number" name="mytheme_options[' . esc_attr($key) . ']" value="' . esc_attr($value) . '" min="' . esc_attr($min) . '" max="' . esc_attr($max) . '">';
    }
    
    public function sanitize_options( $input ) {
        $sanitized = array();
        
        if ( isset( $input['google_analytics_id'] ) ) {
            $sanitized['google_analytics_id'] = sanitize_text_field( $input['google_analytics_id'] );
        }
        
        if ( isset( $input['posts_per_page'] ) ) {
            $sanitized['posts_per_page'] = absint( $input['posts_per_page'] );
        }
        
        return $sanitized;
    }
    
    public function render_options_page() {
        ?>
        <div class="wrap">
            <h1><?php esc_html_e( 'Theme Options', 'mytheme' ); ?></h1>
            
            <form method="post" action="options.php">
                <?php
                settings_fields( 'mytheme_options_group' );
                do_settings_sections( 'mytheme-options' );
                submit_button();
                ?>
            </form>
        </div>
        <?php
    }
}

new Mytheme_Options();
```

---

## 5. Advanced Template Functions

```php
<?php
/**
 * Advanced Template Helper Functions
 */

// ==========================================
// Breadcrumbs
// ==========================================

function mytheme_breadcrumbs() {
    echo '<nav class="breadcrumbs" aria-label="Breadcrumb">';
    echo '<ol itemscope itemtype="https://schema.org/BreadcrumbList">';
    
    $position = 1;
    
    // Home
    echo '<li itemprop="itemListElement" itemscope itemtype="https://schema.org/ListItem">';
    echo '<a itemprop="item" href="' . home_url() . '"><span itemprop="name">หน้าแรก</span></a>';
    echo '<meta itemprop="position" content="' . $position . '">';
    echo '</li>';
    
    if ( is_single() ) {
        $position++;
        $category = get_the_category();
        if ( $category ) {
            $cat = $category[0];
            echo '<li itemprop="itemListElement" itemscope itemtype="https://schema.org/ListItem">';
            echo '<a itemprop="item" href="' . get_category_link($cat->term_id) . '"><span itemprop="name">' . esc_html($cat->name) . '</span></a>';
            echo '<meta itemprop="position" content="' . $position . '">';
            echo '</li>';
        }
        
        $position++;
        echo '<li class="current" itemprop="itemListElement" itemscope itemtype="https://schema.org/ListItem">';
        echo '<span itemprop="name">' . get_the_title() . '</span>';
        echo '<meta itemprop="position" content="' . $position . '">';
        echo '</li>';
        
    } elseif ( is_page() ) {
        
        global $post;
        $parents = array();
        $parent_id = $post->post_parent;
        
        while ( $parent_id ) {
            $parents[] = $parent_id;
            $parent = get_page( $parent_id );
            $parent_id = $parent->post_parent;
        }
        
        foreach ( array_reverse( $parents ) as $parent_id ) {
            $position++;
            echo '<li itemprop="itemListElement">';
            echo '<a href="' . get_permalink($parent_id) . '">' . get_the_title($parent_id) . '</a>';
            echo '</li>';
        }
        
        $position++;
        echo '<li class="current">' . get_the_title() . '</li>';
        
    } elseif ( is_category() ) {
        $position++;
        echo '<li class="current">' . single_cat_title('', false) . '</li>';
        
    } elseif ( is_tag() ) {
        $position++;
        echo '<li class="current">Tag: ' . single_tag_title('', false) . '</li>';
        
    } elseif ( is_search() ) {
        $position++;
        echo '<li class="current">ค้นหา: ' . get_search_query() . '</li>';
        
    } elseif ( is_404() ) {
        $position++;
        echo '<li class="current">404 - ไม่พบหน้า</li>';
    }
    
    echo '</ol></nav>';
}

// ==========================================
// Social Share Buttons
// ==========================================

function mytheme_social_share() {
    $url     = urlencode( get_permalink() );
    $title   = urlencode( get_the_title() );
    $image   = urlencode( get_the_post_thumbnail_url() );
    
    echo '<div class="social-share">';
    echo '<h4>แชร์บทความนี้</h4>';
    echo '<ul class="share-buttons">';
    
    // Facebook
    echo '<li><a href="https://www.facebook.com/sharer/sharer.php?u=' . $url . '" target="_blank" rel="noopener" class="share-facebook">Facebook</a></li>';
    
    // Twitter/X
    echo '<li><a href="https://twitter.com/intent/tweet?url=' . $url . '&text=' . $title . '" target="_blank" rel="noopener" class="share-twitter">Twitter</a></li>';
    
    // LINE
    echo '<li><a href="https://social-plugins.line.me/lineit/share?url=' . $url . '" target="_blank" rel="noopener" class="share-line">LINE</a></li>';
    
    // Copy Link
    echo '<li><button class="share-copy" data-url="' . esc_attr(get_permalink()) . '">คัดลอกลิงก์</button></li>';
    
    echo '</ul>';
    echo '</div>';
}

// ==========================================
// Related Posts
// ==========================================

function mytheme_related_posts( $post_id = null, $count = 4 ) {
    
    if ( ! $post_id ) {
        $post_id = get_the_ID();
    }
    
    $categories = wp_get_post_categories( $post_id );
    $tags       = wp_get_post_tags( $post_id, array('fields' => 'ids') );
    
    $args = array(
        'post_type'      => get_post_type(),
        'posts_per_page' => $count,
        'post__not_in'   => array( $post_id ),
        'orderby'        => 'rand',
        'no_found_rows'  => true,
    );
    
    if ( $tags ) {
        $args['tax_query'] = array(
            array(
                'taxonomy' => 'post_tag',
                'field'    => 'term_id',
                'terms'    => $tags,
            ),
        );
    } elseif ( $categories ) {
        $args['category__in'] = $categories;
    }
    
    $related = new WP_Query( $args );
    
    if ( $related->have_posts() ) {
        echo '<div class="related-posts">';
        echo '<h3>' . __('บทความที่เกี่ยวข้อง', 'mytheme') . '</h3>';
        echo '<div class="related-grid">';
        
        while ( $related->have_posts() ) {
            $related->the_post();
            ?>
            <article class="related-post">
                <?php if (has_post_thumbnail()) : ?>
                    <a href="<?php the_permalink(); ?>" class="related-thumbnail">
                        <?php the_post_thumbnail('mytheme-thumbnail'); ?>
                    </a>
                <?php endif; ?>
                <div class="related-info">
                    <?php the_title('<h4><a href="' . get_permalink() . '">', '</a></h4>'); ?>
                    <span class="related-date"><?php echo get_the_date(); ?></span>
                </div>
            </article>
            <?php
        }
        
        echo '</div></div>';
        wp_reset_postdata();
    }
}
```

---

## Workshop: สร้าง Advanced Theme Features

### Exercise 1: Custom Walker สำหรับ Navigation

สร้าง Custom Walker ที่เพิ่ม Font Awesome Icons ให้ Menu Items:

```php
<?php
class Icon_Nav_Walker extends Walker_Nav_Menu {
    
    public function start_el( &$output, $item, $depth = 0, $args = null, $id = 0 ) {
        
        $classes = (array) $item->classes;
        
        // ค้นหา Icon Class (เช่น icon-home, icon-about)
        $icon_class = '';
        foreach ( $classes as $class ) {
            if ( strpos($class, 'icon-') === 0 ) {
                $icon_class = str_replace('icon-', 'fa-', $class);
                break;
            }
        }
        
        $output .= '<li ' . $this->build_li_atts($item, $depth, $args) . '>';
        $output .= '<a href="' . esc_url($item->url) . '">';
        
        // TODO: เพิ่ม Icon ก่อน Title
        if ( $icon_class ) {
            $output .= '<i class="fas ' . esc_attr($icon_class) . '"></i> ';
        }
        
        $output .= esc_html($item->title);
        $output .= '</a>';
    }
    
    private function build_li_atts( $item, $depth, $args ) {
        // TODO: สร้าง attributes string
        $classes = (array) $item->classes;
        $class_str = implode(' ', array_filter($classes));
        return 'class="' . esc_attr($class_str) . '"';
    }
}
```

---

## Quiz

**คำถามที่ 1:** Custom Page Template ต้องเพิ่มคอมเมนต์อะไรที่ด้านบนไฟล์?

A) `// Page Template`  
B) `/* Template Name: My Template */`  
C) `<?php // custom-template ?>`  
D) `/* @template my-template */`  

**เฉลย: B) `/* Template Name: My Template */`**

---

**คำถามที่ 2:** Walker Class ในการสร้าง Custom Menu ต้อง extend Class ใด?

A) WP_Nav_Menu  
B) Walker  
C) Walker_Nav_Menu  
D) WP_Walker  

**เฉลย: C) Walker_Nav_Menu**

---

**คำถามที่ 3:** `transport => 'postMessage'` ใน Customizer หมายความว่าอะไร?

A) ส่ง Email เมื่อ Settings เปลี่ยน  
B) Live Preview ทำงานโดยไม่ต้อง Refresh  
C) บันทึกข้อมูลทันที  
D) Disable Preview  

**เฉลย: B) Live Preview ทำงานผ่าน JavaScript โดยไม่ต้อง Reload**

---

**คำถามที่ 4:** `get_theme_mod()` ต่างจาก `get_option()` อย่างไร?

A) ไม่มีความแตกต่าง  
B) get_theme_mod ดึงค่าจาก Customizer, get_option จาก wp_options  
C) get_theme_mod เร็วกว่า  
D) get_theme_mod ใช้กับ Plugin เท่านั้น  

**เฉลย: B) get_theme_mod ดึงค่าที่ตั้งใน Customizer (theme_mods), get_option ดึงจาก wp_options ทั่วไป**

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- Custom Page Templates สำหรับ Layout พิเศษ
- Walker Classes สำหรับ Custom Navigation HTML
- WordPress Customizer API
- Theme Options Page ใน Admin
- Advanced Template Functions (Breadcrumbs, Social Share, Related Posts)

---

## ต่อไป

➡️ **[Part 054: WordPress Plugin Basics](part-054-wordpress-plugin-basics.md)**

เรียนรู้เกี่ยวกับ:
- โครงสร้าง Plugin
- Plugin Header
- Activation/Deactivation Hooks
- Action และ Filter Hooks ใน Plugin
