# Part 058: WordPress Gutenberg Block Development

**ระดับ:** Advanced  
**เวลาเรียน:** 6-8 ชั่วโมง  
**Prerequisites:** Part 057, JavaScript ES6+, React Basics

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. เข้าใจ Gutenberg Block Architecture
2. สร้าง Static Block
3. สร้าง Dynamic Block (Render ด้วย PHP)
4. ใช้งาน Block Attributes
5. สร้าง Block Patterns

---

## 1. Gutenberg Block Architecture

```
Gutenberg Block Structure:
- block.json     - Block metadata (name, version, attributes, etc.)
- index.js       - Entry point (registers block)
- edit.js        - Editor Component (อะไรแสดงใน Editor)
- save.js        - Saved HTML (อะไรบันทึกลง Database)
- style.scss     - Frontend styles
- editor.scss    - Editor-only styles
- render.php     - Dynamic block PHP render (ถ้าใช้ Dynamic)
```

### 1.1 Setup Block Development Environment

```bash
# สร้าง Block Plugin ด้วย @wordpress/create-block
npx @wordpress/create-block my-custom-block --template @wordpress/create-block-tutorial-template

# หรือสร้างแบบ Custom
npx @wordpress/create-block my-block

# เข้าไปใน Plugin Directory
cd my-block

# ติดตั้ง Dependencies
npm install

# Development (Watch Mode)
npm start

# Build Production
npm run build

# Plugin Structure หลังสร้าง:
# my-block/
# ├── my-block.php         <- Plugin file
# ├── block.json           <- Block metadata
# ├── src/
# │   ├── index.js
# │   ├── edit.js
# │   ├── save.js
# │   ├── style.scss
# │   └── editor.scss
# └── build/               <- Compiled files
```

### 1.2 block.json

```json
{
    "$schema": "https://schemas.wp.org/trunk/block.json",
    "apiVersion": 3,
    "name": "my-plugin/hero-section",
    "version": "1.0.0",
    "title": "Hero Section",
    "category": "design",
    "icon": "cover-image",
    "description": "A full-width hero section with title, subtitle, and CTA button.",
    "keywords": [ "hero", "banner", "cover" ],
    "textdomain": "my-plugin",
    "supports": {
        "html": false,
        "color": {
            "background": true,
            "text": true
        },
        "spacing": {
            "padding": true,
            "margin": true
        },
        "typography": {
            "fontSize": true,
            "lineHeight": true
        },
        "align": [ "wide", "full" ]
    },
    "attributes": {
        "title": {
            "type": "string",
            "source": "html",
            "selector": "h1",
            "default": "Welcome to Our Website"
        },
        "subtitle": {
            "type": "string",
            "source": "html",
            "selector": "p.subtitle",
            "default": "We create amazing experiences"
        },
        "buttonText": {
            "type": "string",
            "default": "Get Started"
        },
        "buttonUrl": {
            "type": "string",
            "default": "#"
        },
        "backgroundImage": {
            "type": "object",
            "default": null
        },
        "overlayOpacity": {
            "type": "number",
            "default": 0.5
        },
        "textAlign": {
            "type": "string",
            "default": "center"
        },
        "minHeight": {
            "type": "number",
            "default": 400
        }
    },
    "editorScript": "file:./index.js",
    "style": "file:./style-index.css",
    "editorStyle": "file:./index.css",
    "render": "file:./render.php"
}
```

---

## 2. Static Block

```javascript
// ==========================================
// src/index.js
// ==========================================

import { registerBlockType } from '@wordpress/blocks';
import './style.scss';
import Edit from './edit';
import save from './save';
import metadata from './block.json';

registerBlockType( metadata.name, {
    edit: Edit,
    save,
} );
```

```javascript
// ==========================================
// src/edit.js - Editor Interface
// ==========================================

import { __ } from '@wordpress/i18n';
import {
    useBlockProps,
    RichText,
    InspectorControls,
    MediaUpload,
    MediaUploadCheck,
    ColorPaletteControl,
    PanelColorSettings,
} from '@wordpress/block-editor';
import {
    PanelBody,
    TextControl,
    RangeControl,
    SelectControl,
    Button,
    ToggleControl,
} from '@wordpress/components';

export default function Edit( { attributes, setAttributes } ) {
    
    const {
        title,
        subtitle,
        buttonText,
        buttonUrl,
        backgroundImage,
        overlayOpacity,
        textAlign,
        minHeight,
    } = attributes;
    
    const blockProps = useBlockProps( {
        className: `hero-section text-align-${textAlign}`,
        style: {
            minHeight: `${minHeight}px`,
            backgroundImage: backgroundImage
                ? `url(${backgroundImage.url})`
                : 'none',
        },
    } );
    
    return (
        <>
            {/* Inspector Controls (Sidebar Panel) */}
            <InspectorControls>
                
                {/* Layout Panel */}
                <PanelBody title={ __( 'Layout', 'my-plugin' ) } initialOpen={ true }>
                    
                    <RangeControl
                        label={ __( 'Min Height (px)', 'my-plugin' ) }
                        value={ minHeight }
                        onChange={ ( value ) => setAttributes( { minHeight: value } ) }
                        min={ 200 }
                        max={ 1000 }
                        step={ 10 }
                    />
                    
                    <SelectControl
                        label={ __( 'Text Alignment', 'my-plugin' ) }
                        value={ textAlign }
                        options={ [
                            { label: 'Left', value: 'left' },
                            { label: 'Center', value: 'center' },
                            { label: 'Right', value: 'right' },
                        ] }
                        onChange={ ( value ) => setAttributes( { textAlign: value } ) }
                    />
                    
                </PanelBody>
                
                {/* Background Image Panel */}
                <PanelBody title={ __( 'Background', 'my-plugin' ) }>
                    
                    <MediaUploadCheck>
                        <MediaUpload
                            onSelect={ ( media ) => setAttributes( {
                                backgroundImage: {
                                    id: media.id,
                                    url: media.url,
                                    alt: media.alt,
                                }
                            } ) }
                            allowedTypes={ ['image'] }
                            value={ backgroundImage?.id }
                            render={ ( { open } ) => (
                                <div>
                                    { backgroundImage && (
                                        <img
                                            src={ backgroundImage.url }
                                            alt={ backgroundImage.alt }
                                            style={ { width: '100%', marginBottom: '8px' } }
                                        />
                                    ) }
                                    <Button
                                        onClick={ open }
                                        isPrimary={ !backgroundImage }
                                        isSecondary={ !!backgroundImage }
                                    >
                                        { backgroundImage
                                            ? __( 'Change Image', 'my-plugin' )
                                            : __( 'Select Image', 'my-plugin' )
                                        }
                                    </Button>
                                    { backgroundImage && (
                                        <Button
                                            isDestructive
                                            onClick={ () => setAttributes( { backgroundImage: null } ) }
                                            style={ { marginLeft: '8px' } }
                                        >
                                            { __( 'Remove', 'my-plugin' ) }
                                        </Button>
                                    ) }
                                </div>
                            ) }
                        />
                    </MediaUploadCheck>
                    
                    { backgroundImage && (
                        <RangeControl
                            label={ __( 'Overlay Opacity', 'my-plugin' ) }
                            value={ overlayOpacity }
                            onChange={ ( value ) => setAttributes( { overlayOpacity: value } ) }
                            min={ 0 }
                            max={ 1 }
                            step={ 0.05 }
                        />
                    ) }
                    
                </PanelBody>
                
                {/* Button Panel */}
                <PanelBody title={ __( 'Button', 'my-plugin' ) }>
                    
                    <TextControl
                        label={ __( 'Button Text', 'my-plugin' ) }
                        value={ buttonText }
                        onChange={ ( value ) => setAttributes( { buttonText: value } ) }
                    />
                    
                    <TextControl
                        label={ __( 'Button URL', 'my-plugin' ) }
                        value={ buttonUrl }
                        onChange={ ( value ) => setAttributes( { buttonUrl: value } ) }
                        type="url"
                    />
                    
                </PanelBody>
                
            </InspectorControls>
            
            {/* Block Content (Editor View) */}
            <div { ...blockProps }>
                
                { backgroundImage && (
                    <div
                        className="hero-overlay"
                        style={ { opacity: overlayOpacity } }
                    />
                ) }
                
                <div className="hero-content">
                    
                    <RichText
                        tagName="h1"
                        value={ title }
                        onChange={ ( value ) => setAttributes( { title: value } ) }
                        placeholder={ __( 'Enter hero title...', 'my-plugin' ) }
                        allowedFormats={ ['core/bold', 'core/italic'] }
                    />
                    
                    <RichText
                        tagName="p"
                        className="subtitle"
                        value={ subtitle }
                        onChange={ ( value ) => setAttributes( { subtitle: value } ) }
                        placeholder={ __( 'Enter subtitle...', 'my-plugin' ) }
                    />
                    
                    { buttonText && (
                        <a href={ buttonUrl } className="hero-button">
                            { buttonText }
                        </a>
                    ) }
                    
                </div>
            </div>
        </>
    );
}
```

```javascript
// ==========================================
// src/save.js - Saved HTML
// ==========================================

import { useBlockProps, RichText } from '@wordpress/block-editor';

export default function save( { attributes } ) {
    
    const {
        title,
        subtitle,
        buttonText,
        buttonUrl,
        backgroundImage,
        overlayOpacity,
        textAlign,
        minHeight,
    } = attributes;
    
    const blockProps = useBlockProps.save( {
        className: `hero-section text-align-${textAlign}`,
        style: {
            minHeight: `${minHeight}px`,
            backgroundImage: backgroundImage
                ? `url(${backgroundImage.url})`
                : undefined,
        },
    } );
    
    return (
        <div { ...blockProps }>
            
            { backgroundImage && (
                <div
                    className="hero-overlay"
                    style={ { opacity: overlayOpacity } }
                />
            ) }
            
            <div className="hero-content">
                
                <RichText.Content
                    tagName="h1"
                    value={ title }
                />
                
                <RichText.Content
                    tagName="p"
                    className="subtitle"
                    value={ subtitle }
                />
                
                { buttonText && (
                    <a href={ buttonUrl } className="hero-button">
                        { buttonText }
                    </a>
                ) }
                
            </div>
        </div>
    );
}
```

---

## 3. Dynamic Block (PHP Render)

```json
// block.json สำหรับ Dynamic Block
{
    "name": "my-plugin/latest-posts",
    "title": "Latest Posts (Dynamic)",
    "category": "widgets",
    "attributes": {
        "numberOfPosts": {
            "type": "number",
            "default": 5
        },
        "categoryId": {
            "type": "number",
            "default": 0
        },
        "showExcerpt": {
            "type": "boolean",
            "default": true
        },
        "showThumbnail": {
            "type": "boolean",
            "default": true
        },
        "layout": {
            "type": "string",
            "default": "list"
        }
    },
    "editorScript": "file:./index.js",
    "render": "file:./render.php"
}
```

```php
<?php
// render.php - PHP Template สำหรับ Dynamic Block
// $attributes มาจาก Block Attributes
// $content มาจาก Inner Blocks (ถ้ามี)
// $block คือ WP_Block instance

$number_of_posts = (int) ($attributes['numberOfPosts'] ?? 5);
$category_id     = (int) ($attributes['categoryId'] ?? 0);
$show_excerpt    = (bool) ($attributes['showExcerpt'] ?? true);
$show_thumbnail  = (bool) ($attributes['showThumbnail'] ?? true);
$layout          = sanitize_text_field($attributes['layout'] ?? 'list');

$query_args = array(
    'post_type'      => 'post',
    'posts_per_page' => $number_of_posts,
    'post_status'    => 'publish',
);

if ($category_id) {
    $query_args['cat'] = $category_id;
}

$posts = get_posts($query_args);

if (empty($posts)) {
    echo '<p>ไม่พบบทความ</p>';
    return;
}

$wrapper_attributes = get_block_wrapper_attributes(array(
    'class' => 'latest-posts layout-' . $layout,
));

?>

<div <?php echo $wrapper_attributes; ?>>
    <?php foreach ($posts as $post) : ?>
        <article class="post-item">
            
            <?php if ($show_thumbnail && has_post_thumbnail($post->ID)) : ?>
                <a href="<?php echo get_permalink($post->ID); ?>" class="post-thumbnail">
                    <?php echo get_the_post_thumbnail($post->ID, 'medium'); ?>
                </a>
            <?php endif; ?>
            
            <div class="post-info">
                <h3>
                    <a href="<?php echo esc_url(get_permalink($post->ID)); ?>">
                        <?php echo esc_html($post->post_title); ?>
                    </a>
                </h3>
                
                <time datetime="<?php echo esc_attr(get_the_date('Y-m-d', $post)); ?>">
                    <?php echo esc_html(get_the_date('', $post)); ?>
                </time>
                
                <?php if ($show_excerpt) : ?>
                    <p><?php echo esc_html(get_the_excerpt($post)); ?></p>
                <?php endif; ?>
            </div>
            
        </article>
    <?php endforeach; ?>
</div>
```

```javascript
// edit.js สำหรับ Dynamic Block
import { __ } from '@wordpress/i18n';
import { useBlockProps, InspectorControls } from '@wordpress/block-editor';
import { PanelBody, RangeControl, ToggleControl, SelectControl } from '@wordpress/components';
import { useSelect } from '@wordpress/data';
import { store as coreStore } from '@wordpress/core-data';
import { RawHTML } from '@wordpress/element';
import { useEntityProp } from '@wordpress/core-data';

export default function Edit( { attributes, setAttributes } ) {
    
    const { numberOfPosts, categoryId, showExcerpt, showThumbnail, layout } = attributes;
    
    // ดึง Categories
    const categories = useSelect(
        select =>
            select(coreStore).getEntityRecords('taxonomy', 'category', {
                per_page: -1,
            }),
        []
    );
    
    // ดึง Posts Preview
    const posts = useSelect(
        select =>
            select(coreStore).getEntityRecords('postType', 'post', {
                per_page: numberOfPosts,
                _embed: true,
                ...(categoryId ? { categories: categoryId } : {}),
            }),
        [numberOfPosts, categoryId]
    );
    
    const blockProps = useBlockProps();
    
    const categoryOptions = [
        { label: __('All Categories', 'my-plugin'), value: 0 },
        ...(categories || []).map(cat => ({
            label: cat.name,
            value: cat.id,
        })),
    ];
    
    return (
        <>
            <InspectorControls>
                <PanelBody title={ __('Settings', 'my-plugin') }>
                    
                    <RangeControl
                        label={ __('Number of Posts', 'my-plugin') }
                        value={ numberOfPosts }
                        onChange={ value => setAttributes({ numberOfPosts: value }) }
                        min={ 1 }
                        max={ 20 }
                    />
                    
                    <SelectControl
                        label={ __('Category', 'my-plugin') }
                        value={ categoryId }
                        options={ categoryOptions }
                        onChange={ value => setAttributes({ categoryId: parseInt(value) }) }
                    />
                    
                    <SelectControl
                        label={ __('Layout', 'my-plugin') }
                        value={ layout }
                        options={ [
                            { label: 'List', value: 'list' },
                            { label: 'Grid', value: 'grid' },
                        ] }
                        onChange={ value => setAttributes({ layout: value }) }
                    />
                    
                    <ToggleControl
                        label={ __('Show Excerpt', 'my-plugin') }
                        checked={ showExcerpt }
                        onChange={ value => setAttributes({ showExcerpt: value }) }
                    />
                    
                    <ToggleControl
                        label={ __('Show Thumbnail', 'my-plugin') }
                        checked={ showThumbnail }
                        onChange={ value => setAttributes({ showThumbnail: value }) }
                    />
                    
                </PanelBody>
            </InspectorControls>
            
            <div { ...blockProps }>
                { !posts && <p>{ __('Loading...', 'my-plugin') }</p> }
                
                { posts && posts.length === 0 && (
                    <p>{ __('No posts found.', 'my-plugin') }</p>
                ) }
                
                { posts && posts.length > 0 && (
                    <ul className={ `posts-preview layout-${layout}` }>
                        { posts.map(post => (
                            <li key={ post.id }>
                                { showThumbnail && post._embedded?.['wp:featuredmedia']?.[0]?.source_url && (
                                    <img
                                        src={ post._embedded['wp:featuredmedia'][0].source_url }
                                        alt=""
                                        style={ { width: '100%', maxWidth: '200px' } }
                                    />
                                ) }
                                <h3>
                                    <RawHTML>{ post.title.rendered }</RawHTML>
                                </h3>
                                { showExcerpt && (
                                    <RawHTML>{ post.excerpt.rendered }</RawHTML>
                                ) }
                            </li>
                        ) ) }
                    </ul>
                ) }
            </div>
        </>
    );
}
```

---

## 4. Register Block ใน PHP

```php
<?php
/**
 * Block Registration (PHP Side)
 */

function register_custom_blocks() {
    
    // Register Block จาก block.json
    register_block_type( __DIR__ . '/blocks/hero-section' );
    register_block_type( __DIR__ . '/blocks/latest-posts' );
    
    // Register Block ด้วย Manual Arguments
    register_block_type( 'my-plugin/testimonial', array(
        'render_callback' => 'render_testimonial_block',
        'attributes'      => array(
            'quote'   => array( 'type' => 'string', 'default' => '' ),
            'author'  => array( 'type' => 'string', 'default' => '' ),
            'rating'  => array( 'type' => 'number', 'default' => 5 ),
        ),
        'editor_script'   => 'my-plugin-blocks',
        'editor_style'    => 'my-plugin-blocks-editor',
        'style'           => 'my-plugin-blocks-style',
    ) );
}
add_action( 'init', 'register_custom_blocks' );

// ==========================================
// Block Categories
// ==========================================

add_filter( 'block_categories_all', function( $categories ) {
    
    // เพิ่ม Custom Category
    array_unshift( $categories, array(
        'slug'  => 'my-plugin',
        'title' => __( 'My Plugin Blocks', 'my-plugin' ),
        'icon'  => 'star-filled',
    ) );
    
    return $categories;
} );

// ==========================================
// Block Transforms
// ==========================================

// ใน JavaScript:
/*
import { createBlock } from '@wordpress/blocks';

registerBlockType('my-plugin/hero', {
    transforms: {
        from: [
            {
                type: 'block',
                blocks: ['core/cover'],
                transform: ({ url, alt, overlayColor }) => {
                    return createBlock('my-plugin/hero', {
                        backgroundImage: { url, alt },
                        // map attributes
                    });
                },
            },
        ],
        to: [
            {
                type: 'block',
                blocks: ['core/cover'],
                transform: ({ backgroundImage, title }) => {
                    return createBlock('core/cover', {
                        url: backgroundImage?.url,
                    });
                },
            },
        ],
    },
});
*/
```

---

## 5. Block Patterns

```php
<?php
/**
 * Register Block Patterns
 */

function register_block_patterns() {
    
    // Register Pattern Category
    register_block_pattern_category( 'my-plugin', array(
        'label' => __( 'My Plugin', 'my-plugin' ),
    ) );
    
    // Register Pattern: Hero Section
    register_block_pattern(
        'my-plugin/hero-with-cta',
        array(
            'title'       => __( 'Hero with CTA', 'my-plugin' ),
            'description' => __( 'A full-width hero section with call-to-action button.', 'my-plugin' ),
            'categories'  => array( 'my-plugin', 'hero' ),
            'keywords'    => array( 'hero', 'banner', 'landing' ),
            'content'     => '<!-- wp:my-plugin/hero-section {"title":"ยินดีต้อนรับสู่เว็บของเรา","subtitle":"เราสร้างประสบการณ์ที่น่าจดจำ","buttonText":"เริ่มต้น","buttonUrl":"#","textAlign":"center","minHeight":500} /-->',
        )
    );
    
    // Register Pattern: Pricing Cards
    register_block_pattern(
        'my-plugin/pricing-cards',
        array(
            'title'      => __( 'Pricing Cards', 'my-plugin' ),
            'categories' => array( 'my-plugin' ),
            'content'    => '
                <!-- wp:columns {"className":"pricing-section"} -->
                <div class="wp-block-columns pricing-section">
                    
                    <!-- wp:column -->
                    <div class="wp-block-column">
                        <!-- wp:group {"className":"pricing-card"} -->
                        <div class="wp-block-group pricing-card">
                            <!-- wp:heading {"level":3} -->
                            <h3 class="wp-block-heading">แพ็กเกจฟรี</h3>
                            <!-- /wp:heading -->
                            <!-- wp:paragraph {"className":"price"} -->
                            <p class="price">0 บาท/เดือน</p>
                            <!-- /wp:paragraph -->
                        </div>
                        <!-- /wp:group -->
                    </div>
                    <!-- /wp:column -->
                    
                    <!-- wp:column -->
                    <div class="wp-block-column">
                        <!-- wp:group {"className":"pricing-card featured"} -->
                        <div class="wp-block-group pricing-card featured">
                            <!-- wp:heading {"level":3} -->
                            <h3 class="wp-block-heading">แพ็กเกจมาตรฐาน</h3>
                            <!-- /wp:heading -->
                            <!-- wp:paragraph {"className":"price"} -->
                            <p class="price">499 บาท/เดือน</p>
                            <!-- /wp:paragraph -->
                        </div>
                        <!-- /wp:group -->
                    </div>
                    <!-- /wp:column -->
                    
                </div>
                <!-- /wp:columns -->
            ',
        )
    );
    
    // Register Pattern จากไฟล์
    // (WordPress 6.0+ รองรับ patterns/ directory ใน Theme)
}
add_action( 'init', 'register_block_patterns' );

// ==========================================
// Theme Block Patterns (Theme Support)
// ==========================================

// สร้างไฟล์ใน: patterns/hero.php
/*
<?php
/**
 * Title: Full Width Hero
 * Slug: my-theme/full-width-hero
 * Categories: my-theme
 * Keywords: hero, banner
 * Block Types: core/cover
 * Inserter: true
 */
?>
<!-- wp:cover {"dimRatio":50,"isDark":false,"align":"full"} -->
<div class="wp-block-cover alignfull">
    <span class="wp-block-cover__background"></span>
    <div class="wp-block-cover__inner-container">
        <!-- wp:heading {"textAlign":"center","level":1} -->
        <h1 class="wp-block-heading has-text-align-center">ยินดีต้อนรับ</h1>
        <!-- /wp:heading -->
    </div>
</div>
<!-- /wp:cover -->
*/
```

---

## 6. Inner Blocks

```javascript
// Block ที่รับ Child Blocks (Inner Blocks)
import {
    InnerBlocks,
    useBlockProps,
    InspectorControls,
} from '@wordpress/block-editor';

const TEMPLATE = [
    ['core/heading', { level: 2, placeholder: 'Card Title...' }],
    ['core/paragraph', { placeholder: 'Card description...' }],
    ['core/buttons', {}, [
        ['core/button', { text: 'Read More' }],
    ]],
];

const ALLOWED_BLOCKS = [
    'core/heading',
    'core/paragraph',
    'core/image',
    'core/buttons',
    'core/list',
];

export function CardEdit({ attributes, setAttributes }) {
    const blockProps = useBlockProps({ className: 'card-block' });
    
    return (
        <div {...blockProps}>
            <div className="card-inner">
                <InnerBlocks
                    template={TEMPLATE}
                    templateLock={false}  // false = ไม่ lock, 'all' = lock ทั้งหมด
                    allowedBlocks={ALLOWED_BLOCKS}
                />
            </div>
        </div>
    );
}

export function CardSave() {
    const blockProps = useBlockProps.save({ className: 'card-block' });
    
    return (
        <div {...blockProps}>
            <div className="card-inner">
                <InnerBlocks.Content />
            </div>
        </div>
    );
}
```

---

## Workshop: สร้าง Testimonials Block

```bash
# สร้าง Block
npx @wordpress/create-block testimonials-block
cd testimonials-block
```

```javascript
// src/edit.js - สำหรับ Exercise
// TODO: สร้าง Testimonial Block ที่มี:
// - Repeater-like UI สำหรับเพิ่ม Testimonials
// - Fields: quote, author name, author title, rating
// - Inspector Controls: columns layout (1-3)
// - Style: card layout

import { __ } from '@wordpress/i18n';
import { useBlockProps, InspectorControls } from '@wordpress/block-editor';
import { PanelBody, RangeControl, Button, TextControl, TextareaControl } from '@wordpress/components';

export default function Edit({ attributes, setAttributes }) {
    const { testimonials = [], columns = 2 } = attributes;
    
    const addTestimonial = () => {
        setAttributes({
            testimonials: [
                ...testimonials,
                { id: Date.now(), quote: '', author: '', title: '', rating: 5 }
            ]
        });
    };
    
    const updateTestimonial = (id, field, value) => {
        setAttributes({
            testimonials: testimonials.map(t =>
                t.id === id ? { ...t, [field]: value } : t
            )
        });
    };
    
    const removeTestimonial = (id) => {
        setAttributes({
            testimonials: testimonials.filter(t => t.id !== id)
        });
    };
    
    const blockProps = useBlockProps({ className: `testimonials-grid cols-${columns}` });
    
    return (
        <>
            <InspectorControls>
                <PanelBody title="Layout">
                    <RangeControl
                        label="Columns"
                        value={columns}
                        onChange={value => setAttributes({ columns: value })}
                        min={1}
                        max={3}
                    />
                </PanelBody>
            </InspectorControls>
            
            <div {...blockProps}>
                {testimonials.map(t => (
                    <div key={t.id} className="testimonial-item">
                        <TextareaControl
                            placeholder="Enter quote..."
                            value={t.quote}
                            onChange={v => updateTestimonial(t.id, 'quote', v)}
                        />
                        <TextControl
                            placeholder="Author name"
                            value={t.author}
                            onChange={v => updateTestimonial(t.id, 'author', v)}
                        />
                        <Button isDestructive onClick={() => removeTestimonial(t.id)}>
                            Remove
                        </Button>
                    </div>
                ))}
                <Button isPrimary onClick={addTestimonial}>
                    + Add Testimonial
                </Button>
            </div>
        </>
    );
}
```

---

## Quiz

**คำถามที่ 1:** ความแตกต่างระหว่าง Static Block และ Dynamic Block คืออะไร?

A) Static เร็วกว่า  
B) Static บันทึก HTML ลง Database, Dynamic Render ด้วย PHP ทุกครั้ง  
C) Dynamic แก้ไขได้, Static แก้ไขไม่ได้  
D) Static ใช้ JavaScript, Dynamic ใช้ PHP  

**เฉลย: B) Static Block บันทึก HTML ใน post_content, Dynamic Block Render PHP ทุก Request**

---

**คำถามที่ 2:** `block.json` ใช้สำหรับอะไร?

A) เก็บ Block Content  
B) กำหนด Block Metadata เช่น name, attributes, supports  
C) JavaScript configuration  
D) Database Schema  

**เฉลย: B) กำหนด Metadata ของ Block**

---

**คำถามที่ 3:** `useBlockProps` ใช้ทำอะไร?

A) ดึง Block Attributes  
B) สร้าง Block DOM Attributes ที่ถูกต้อง (class, id สำหรับ Gutenberg)  
C) Register Block  
D) Update Attributes  

**เฉลย: B) Generate Block wrapper attributes ที่จำเป็นสำหรับ Gutenberg**

---

**คำถามที่ 4:** Block Pattern คืออะไร?

A) PHP Design Pattern  
B) CSS Pattern  
C) Template ของ Blocks ที่กำหนดไว้ล่วงหน้า  
D) Block Animation  

**เฉลย: C) Block Patterns คือ Pre-made Block Layouts ที่ User เลือกใช้ได้**

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- Gutenberg Block Architecture
- Static Block ด้วย JavaScript/React
- Dynamic Block ด้วย PHP
- InspectorControls และ Block Controls
- Inner Blocks
- Block Patterns

---

## ต่อไป

➡️ **[Part 059: WordPress WooCommerce](part-059-wordpress-woocommerce.md)**

เรียนรู้เกี่ยวกับ:
- WooCommerce Customization
- Custom Product Types
- Checkout Hooks
- Payment Gateway Development
