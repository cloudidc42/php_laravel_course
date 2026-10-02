# Part 078: Drupal Theming - Twig, Libraries, Preprocess

**ระดับ:** Intermediate  
**เวลาเรียน:** 5-6 ชั่วโมง  
**Prerequisites:** Part 077 - Drupal Content Management

---

## เป้าหมายของ Part นี้

1. สร้าง Custom Theme ใน Drupal
2. ใช้ Twig Templates
3. สร้าง Libraries (CSS/JS)
4. เขียน Preprocess Functions
5. Override Templates

---

## 1. สร้าง Custom Theme

### 1.1 โครงสร้าง Theme

```
web/themes/custom/my_theme/
├── my_theme.info.yml           # Theme Info
├── my_theme.libraries.yml      # CSS/JS Libraries
├── my_theme.theme              # Theme Functions (PHP)
├── logo.svg                    # Theme Logo
├── screenshot.png              # Theme Screenshot
├── config/
│   └── install/                # Default Config
├── css/
│   ├── base/
│   │   └── base.css
│   ├── components/
│   │   └── card.css
│   └── layout/
│       └── layout.css
├── js/
│   └── script.js
├── images/
│   └── logo.png
└── templates/
    ├── layout/
    │   ├── html.html.twig
    │   └── page.html.twig
    ├── navigation/
    │   └── menu.html.twig
    ├── content/
    │   ├── node.html.twig
    │   └── node--article.html.twig
    └── field/
        └── field--field-price.html.twig
```

### 1.2 theme.info.yml

```yaml
# web/themes/custom/my_theme/my_theme.info.yml

name: My Theme
type: theme
description: 'A custom Drupal theme for our site.'
core_version_requirement: ^10
base theme: stable9

# Version
version: 1.0.0

# Libraries ที่โหลดในทุกหน้า
libraries:
  - my_theme/global-styling
  - my_theme/global-scripts

# Breakpoints
breakpoints:
  my_theme.mobile:
    label: mobile
    mediaQuery: ''
    weight: 2
    multipliers:
      - 1x
  my_theme.tablet:
    label: tablet
    mediaQuery: 'all and (min-width: 640px)'
    weight: 1
    multipliers:
      - 1x
  my_theme.desktop:
    label: desktop
    mediaQuery: 'all and (min-width: 1024px)'
    weight: 0
    multipliers:
      - 1x

# Regions
regions:
  header:        Header
  primary_menu:  'Primary Menu'
  secondary_menu: 'Secondary Menu'
  breadcrumb:    Breadcrumb
  highlighted:   Highlighted
  help:          Help
  content:       Content
  sidebar_first: 'Left Sidebar'
  sidebar_second: 'Right Sidebar'
  footer_first:  'Footer First'
  footer_second: 'Footer Second'
  footer_third:  'Footer Third'

# Feature Flags (ของ Drupal Core)
features:
  - comment_user_picture
  - node_user_picture
  - favicon
  - main_menu
  - secondary_menu
```

### 1.3 Libraries

```yaml
# web/themes/custom/my_theme/my_theme.libraries.yml

# Global Styles
global-styling:
  version: 1.0
  css:
    base:
      css/base/base.css: {}
    layout:
      css/layout/layout.css: {}
    component:
      css/components/card.css: {}
      css/components/button.css: {}
  dependencies:
    - core/normalize

# Global Scripts
global-scripts:
  version: 1.0
  js:
    js/script.js: {}
  dependencies:
    - core/jquery
    - core/drupal

# Bootstrap Library (CDN)
bootstrap:
  version: 5.3
  css:
    theme:
      https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css:
        type: external
        minified: true
  js:
    https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js:
      type: external
      minified: true

# Page-specific Library
product-catalog:
  version: 1.0
  css:
    component:
      css/components/product-catalog.css: {}
  js:
    js/product-catalog.js: {}
  dependencies:
    - my_theme/global-scripts

# Drupal Behaviors ตัวอย่าง
behaviors:
  version: 1.0
  js:
    js/behaviors.js: {}
  dependencies:
    - core/drupal
    - core/once
```

### 1.4 JavaScript ด้วย Drupal Behaviors

```javascript
// web/themes/custom/my_theme/js/behaviors.js

(function (Drupal, once) {
  'use strict';

  /**
   * Drupal Behavior: Product Gallery
   * 
   * Drupal Behaviors รันทุกครั้งที่ DOM เปลี่ยน
   * (ต่างจาก $(document).ready() ที่รันครั้งเดียว)
   */
  Drupal.behaviors.productGallery = {
    attach: function (context, settings) {
      // ใช้ once() เพื่อไม่ให้ attach ซ้ำ
      once('product-gallery', '.product-images', context).forEach(function (element) {
        const thumbnails = element.querySelectorAll('.thumbnail');
        const mainImage = element.querySelector('.main-image img');

        thumbnails.forEach(function (thumb) {
          thumb.addEventListener('click', function () {
            const src = this.getAttribute('data-full');
            mainImage.src = src;

            thumbnails.forEach(t => t.classList.remove('active'));
            this.classList.add('active');
          });
        });
      });
    },

    detach: function (context, trigger) {
      // cleanup เมื่อ Detach
      once.remove('product-gallery', '.product-images', context);
    }
  };

  /**
   * Ajax Cart Button
   */
  Drupal.behaviors.addToCart = {
    attach: function (context, settings) {
      once('add-to-cart', '.btn-add-to-cart', context).forEach(function (button) {
        button.addEventListener('click', function (e) {
          e.preventDefault();

          const productId = this.dataset.productId;
          const quantity = document.querySelector('#quantity')?.value || 1;

          // Drupal Ajax URL
          const url = Drupal.url('api/cart/add');

          fetch(url, {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
              'X-CSRF-Token': drupalSettings.csrfToken || '',
            },
            body: JSON.stringify({ product_id: productId, quantity }),
          })
            .then(response => response.json())
            .then(data => {
              if (data.success) {
                Drupal.announce(Drupal.t('Item added to cart'));
                document.querySelector('.cart-count').textContent = data.cart_count;
              }
            });
        });
      });
    }
  };

})(Drupal, once);
```

---

## 2. Twig Templates

### 2.1 Template Inheritance

```twig
{# web/themes/custom/my_theme/templates/layout/page.html.twig #}

<!DOCTYPE html>
<html {{ html_attributes }}>
  <head>
    <head-placeholder token="{{ placeholder_token }}">
    <title>{{ head_title|safe_join(' | ') }}</title>
    <css-placeholder token="{{ placeholder_token }}">
    <js-placeholder token="{{ placeholder_token }}">
  </head>
  <body{{ attributes.addClass('layout-container') }}>
    <a href="#main-content" class="visually-hidden focusable">
      {{ 'Skip to main content'|t }}
    </a>

    {# Header #}
    {% if page.header %}
      <header id="header" class="site-header" role="banner">
        <div class="container">
          {{ page.header }}
        </div>
      </header>
    {% endif %}

    {# Navigation #}
    {% if page.primary_menu %}
      <nav id="main-nav" aria-label="{{ 'Main Navigation'|t }}">
        <div class="container">
          {{ page.primary_menu }}
        </div>
      </nav>
    {% endif %}

    {# Main Content #}
    <main id="main-content" role="main" tabindex="-1">
      <div class="container">
        {% if page.breadcrumb %}
          {{ page.breadcrumb }}
        {% endif %}

        {% if page.highlighted %}
          <div class="highlighted">{{ page.highlighted }}</div>
        {% endif %}

        <div class="layout-main-wrapper">
          {% if page.sidebar_first %}
            <aside class="layout-sidebar-first" role="complementary">
              {{ page.sidebar_first }}
            </aside>
          {% endif %}

          <div class="layout-main">
            {{ page.content }}
          </div>

          {% if page.sidebar_second %}
            <aside class="layout-sidebar-second" role="complementary">
              {{ page.sidebar_second }}
            </aside>
          {% endif %}
        </div>
      </div>
    </main>

    {# Footer #}
    {% if page.footer_first or page.footer_second %}
      <footer class="site-footer" role="contentinfo">
        <div class="container">
          <div class="footer-grid">
            {% if page.footer_first %}
              <div class="footer-col">{{ page.footer_first }}</div>
            {% endif %}
            {% if page.footer_second %}
              <div class="footer-col">{{ page.footer_second }}</div>
            {% endif %}
          </div>
        </div>
      </footer>
    {% endif %}

    <js-bottom-placeholder token="{{ placeholder_token }}">
  </body>
</html>
```

### 2.2 Node Template

```twig
{# web/themes/custom/my_theme/templates/content/node--product--full.html.twig #}
{#
  Template: node--product--full.html.twig
  แสดง Product ในรูปแบบ Full
  
  Variables:
  - node: Node entity
  - content: Rendered fields
  - label: Node title
  - url: Node URL
#}

{% set classes = [
  'node',
  'node--type-' ~ node.bundle|clean_class,
  node.isPromoted ? 'node--promoted',
  node.isSticky ? 'node--sticky',
  not node.isPublished ? 'node--unpublished',
  view_mode ? 'node--view-mode-' ~ view_mode|clean_class,
] %}

<article{{ attributes.addClass(classes) }}>
  <div class="product-detail">

    {# Images Gallery #}
    {% if content.field_product_image %}
      <div class="product-images" data-behavior="product-gallery">
        {% for item in node.field_product_image %}
          {% set image_uri = item.entity.uri.value %}
          {% if loop.first %}
            <div class="main-image">
              <img src="{{ image_uri | image_style('large') }}"
                   alt="{{ item.alt }}"
                   loading="lazy">
            </div>
          {% endif %}
        {% endfor %}

        <div class="thumbnails">
          {% for item in node.field_product_image %}
            {% set image_uri = item.entity.uri.value %}
            <div class="thumbnail {{ loop.first ? 'active' : '' }}"
                 data-full="{{ image_uri | image_style('large') }}">
              <img src="{{ image_uri | image_style('thumbnail') }}"
                   alt="{{ item.alt }}">
            </div>
          {% endfor %}
        </div>
      </div>
    {% endif %}

    {# Product Info #}
    <div class="product-info">
      <h1 class="product-title">{{ label }}</h1>

      {# SKU #}
      {% if content.field_sku %}
        <p class="product-sku">
          {{ 'SKU'|t }}: <span>{{ node.field_sku.value }}</span>
        </p>
      {% endif %}

      {# Price #}
      {% if content.field_price %}
        <div class="product-price">
          <span class="price-label">{{ 'Price'|t }}:</span>
          <span class="price-value">
            ฿{{ node.field_price.value|number_format(2) }}
          </span>
        </div>
      {% endif %}

      {# Categories #}
      {% if content.field_product_category %}
        <div class="product-categories">
          <span>{{ 'Categories'|t }}:</span>
          {% for term_ref in node.field_product_category %}
            <a href="{{ term_ref.entity.url }}" class="category-link">
              {{ term_ref.entity.name.value }}
            </a>
            {% if not loop.last %}, {% endif %}
          {% endfor %}
        </div>
      {% endif %}

      {# Description #}
      {% if content.body %}
        <div class="product-description">
          {{ content.body }}
        </div>
      {% endif %}

      {# Add to Cart #}
      <div class="product-actions">
        <div class="quantity-selector">
          <label for="quantity">{{ 'Quantity'|t }}:</label>
          <input type="number" id="quantity" name="quantity"
                 value="1" min="1" max="99">
        </div>
        <button class="btn-add-to-cart button button--primary"
                data-product-id="{{ node.id() }}">
          {{ 'Add to Cart'|t }}
        </button>
      </div>
    </div>
  </div>
</article>
```

### 2.3 Custom Template สำหรับ Teaser

```twig
{# web/themes/custom/my_theme/templates/content/node--product--teaser.html.twig #}

<article{{ attributes.addClass('product-card') }}>
  {% if content.field_product_image %}
    <div class="product-card__image">
      <a href="{{ url }}">
        {{ content.field_product_image }}
      </a>
    </div>
  {% endif %}

  <div class="product-card__body">
    <h2 class="product-card__title">
      <a href="{{ url }}">{{ label }}</a>
    </h2>

    <div class="product-card__price">
      ฿{{ node.field_price.value|number_format(2) }}
    </div>

    <a href="{{ url }}" class="button">{{ 'View Product'|t }}</a>
  </div>
</article>
```

### 2.4 Twig Functions และ Filters ที่ Drupal เพิ่ม

```twig
{# Twig Functions ของ Drupal #}

{# แปลข้อความ #}
{{ 'Hello World'|t }}
{{ 'Hello @name'|t({'@name': user.name}) }}

{# URL Generation #}
<a href="{{ url('user.login') }}">Login</a>
<a href="{{ path('entity.node.canonical', {'node': node.id()}) }}">{{ node.label }}</a>

{# Image Styles #}
{% set uri = node.field_image.entity.uri.value %}
<img src="{{ uri | image_style('thumbnail') }}" alt="">
<img src="{{ file_url(uri) }}" alt="">

{# Check Access #}
{% if is_admin %}
  <a href="{{ path('entity.node.edit_form', {'node': node.id()}) }}">Edit</a>
{% endif %}

{# Render Element #}
{{ content.field_price }}

{# Drupal Settings (drupalSettings) #}
{{ drupal_settings_twig|json_encode }}

{# Custom Block #}
{{ drupal_block('system_branding_block') }}

{# View #}
{{ drupal_view('product_listing', 'block_1') }}

{# Token #}
{{ drupal_token('site:name') }}

{# Attributes #}
<div{{ attributes.addClass('my-class').setAttribute('id', 'main') }}>

{# Clean Class #}
<div class="{{ 'My Class Name'|clean_class }}"> {# = my-class-name #}

{# Create Attributes Object #}
{% set image_attributes = create_attribute() %}
{% set image_attributes = image_attributes.addClass('img-fluid') %}
<img{{ image_attributes }} src="...">
```

---

## 3. Preprocess Functions

```php
<?php
/**
 * File: web/themes/custom/my_theme/my_theme.theme
 * 
 * Theme Preprocess Functions
 */

use Drupal\Core\Template\Attribute;

/**
 * Implements hook_preprocess_HOOK() for html.
 */
function my_theme_preprocess_html(&$variables) {
    // เพิ่ม Class ให้ Body ตาม Path
    $current_path = \Drupal::service('path.current')->getPath();
    $path_alias = \Drupal::service('path_alias.manager')
        ->getAliasByPath($current_path);

    $path_class = preg_replace('/[^a-z0-9-]/', '-', ltrim($path_alias, '/'));
    $variables['attributes']['class'][] = 'page-' . $path_class;

    // เพิ่ม Node Type Class
    $node = \Drupal::routeMatch()->getParameter('node');
    if ($node) {
        $variables['attributes']['class'][] = 'node-type-' . $node->bundle();
    }
}

/**
 * Implements hook_preprocess_HOOK() for page.
 */
function my_theme_preprocess_page(&$variables) {
    // เพิ่ม Site Name และ Logo
    $config = \Drupal::config('system.site');
    $variables['site_name'] = $config->get('name');
    $variables['site_slogan'] = $config->get('slogan');

    // ดึง Social Media Links จาก Config
    $theme_settings = theme_get_setting('social_links');
    $variables['social_links'] = $theme_settings ?? [];

    // เช็ค User Role
    $current_user = \Drupal::currentUser();
    $variables['is_admin'] = $current_user->hasPermission('access administration pages');
    $variables['is_logged_in'] = $current_user->isAuthenticated();
}

/**
 * Implements hook_preprocess_HOOK() for node.
 */
function my_theme_preprocess_node(&$variables) {
    $node = $variables['node'];

    // เพิ่ม Variables พิเศษสำหรับ Product
    if ($node->bundle() === 'product') {
        // Format Price
        $price = $node->get('field_price')->value;
        $variables['formatted_price'] = number_format($price, 2);

        // ดึง Category Names
        $categories = [];
        foreach ($node->get('field_product_category') as $term_ref) {
            $term = $term_ref->entity;
            if ($term) {
                $categories[] = [
                    'name' => $term->getName(),
                    'url'  => $term->toUrl()->toString(),
                ];
            }
        }
        $variables['product_categories'] = $categories;

        // เพิ่ม Schema.org Microdata
        $variables['attributes']['itemscope'] = '';
        $variables['attributes']['itemtype'] = 'https://schema.org/Product';

        // Related Products
        if ($variables['view_mode'] === 'full') {
            $variables['related_products'] = my_theme_get_related_products($node);
        }
    }
}

/**
 * ดึง Related Products
 */
function my_theme_get_related_products(\Drupal\node\NodeInterface $node): array {
    $category_ids = [];
    foreach ($node->get('field_product_category') as $item) {
        $category_ids[] = $item->target_id;
    }

    if (empty($category_ids)) {
        return [];
    }

    $nids = \Drupal::entityQuery('node')
        ->condition('type', 'product')
        ->condition('status', 1)
        ->condition('field_product_category', $category_ids, 'IN')
        ->condition('nid', $node->id(), '!=')
        ->range(0, 4)
        ->accessCheck(TRUE)
        ->execute();

    if (empty($nids)) {
        return [];
    }

    $view_builder = \Drupal::entityTypeManager()->getViewBuilder('node');
    $nodes = \Drupal\node\Entity\Node::loadMultiple($nids);
    $related = [];

    foreach ($nodes as $related_node) {
        $related[] = $view_builder->view($related_node, 'teaser');
    }

    return $related;
}

/**
 * Implements hook_preprocess_HOOK() for field.
 */
function my_theme_preprocess_field(&$variables) {
    // เพิ่ม Class พิเศษให้ Price Field
    if ($variables['field_name'] === 'field_price') {
        $variables['attributes']['class'][] = 'price-field';

        // เพิ่ม Currency Symbol
        foreach ($variables['items'] as &$item) {
            $item['content']['#prefix'] = '<span class="currency">฿</span>';
        }
    }
}

/**
 * Implements hook_preprocess_HOOK() for views.
 */
function my_theme_preprocess_views_view__product_listing(&$variables) {
    // เพิ่ม Variables เฉพาะ View
    $variables['total_count'] = $variables['view']->total_rows ?? 0;
    $variables['current_filters'] = $variables['view']->getExposedInput();
}

/**
 * Implements hook_theme_suggestions_HOOK_alter() for node.
 */
function my_theme_theme_suggestions_node_alter(array &$suggestions, array $variables) {
    $node = $variables['elements']['#node'];
    $view_mode = $variables['elements']['#view_mode'];

    // เพิ่ม Template Suggestion ตาม View Mode
    $suggestions[] = 'node__' . $node->bundle() . '__' . $view_mode;

    // เพิ่ม Suggestion ตาม Term
    if ($node->bundle() === 'product') {
        foreach ($node->get('field_product_category') as $term_ref) {
            $term = $term_ref->entity;
            if ($term) {
                $suggestions[] = 'node__product__category_' . $term->id();
            }
        }
    }
}

/**
 * Implements hook_preprocess_HOOK() for menu.
 */
function my_theme_preprocess_menu(&$variables) {
    if ($variables['menu_name'] === 'main') {
        // เพิ่ม Active Class
        $current_path = \Drupal::service('path.current')->getPath();
        foreach ($variables['items'] as &$item) {
            if ($item['url']->toString() === $current_path) {
                $item['attributes']->addClass('active');
            }
        }
    }
}
```

---

## 4. Theme Settings

```php
<?php
/**
 * Implements hook_form_FORM_ID_alter() for system_theme_settings.
 * เพิ่ม Theme Settings
 */
function my_theme_form_system_theme_settings_alter(&$form, \Drupal\Core\Form\FormStateInterface $form_state) {
    $form['my_theme_settings'] = [
        '#type'  => 'details',
        '#title' => t('My Theme Settings'),
        '#open'  => TRUE,
    ];

    $form['my_theme_settings']['social_facebook'] = [
        '#type'          => 'url',
        '#title'         => t('Facebook URL'),
        '#default_value' => theme_get_setting('social_facebook'),
    ];

    $form['my_theme_settings']['social_instagram'] = [
        '#type'          => 'url',
        '#title'         => t('Instagram URL'),
        '#default_value' => theme_get_setting('social_instagram'),
    ];

    $form['my_theme_settings']['footer_text'] = [
        '#type'          => 'textarea',
        '#title'         => t('Footer Text'),
        '#default_value' => theme_get_setting('footer_text'),
        '#rows'          => 3,
    ];

    $form['my_theme_settings']['show_breadcrumb'] = [
        '#type'          => 'checkbox',
        '#title'         => t('Show Breadcrumb'),
        '#default_value' => theme_get_setting('show_breadcrumb') ?? TRUE,
    ];
}
```

---

## Workshop: สร้าง Theme สมบูรณ์

### Exercise 1: CSS Layout

```css
/* web/themes/custom/my_theme/css/layout/layout.css */

:root {
  --color-primary: #0077cc;
  --color-secondary: #005fa3;
  --color-text: #333;
  --color-bg: #fff;
  --font-base: 'Sarabun', sans-serif;
  --container-width: 1200px;
  --spacing-xs: 0.25rem;
  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;
  --spacing-lg: 2rem;
  --spacing-xl: 4rem;
}

* { box-sizing: border-box; }

body {
  font-family: var(--font-base);
  color: var(--color-text);
  background: var(--color-bg);
  margin: 0;
  line-height: 1.6;
}

.container {
  max-width: var(--container-width);
  margin: 0 auto;
  padding: 0 var(--spacing-md);
}

/* Grid Layout */
.layout-main-wrapper {
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--spacing-lg);
}

@media (min-width: 768px) {
  .layout-main-wrapper.has-sidebar-first {
    grid-template-columns: 280px 1fr;
  }
  .layout-main-wrapper.has-sidebar-second {
    grid-template-columns: 1fr 280px;
  }
  .layout-main-wrapper.has-both-sidebars {
    grid-template-columns: 240px 1fr 240px;
  }
}
```

---

## Quiz

**คำถามที่ 1:** `hook_preprocess_node()` ใช้ทำอะไร?

A) สร้าง Node ใหม่  
B) เพิ่ม Variables ให้ Twig Template ก่อน Render  
C) ลบ Node  
D) เปลี่ยน Node Type  

**เฉลย: B) Preprocess เพิ่ม/แก้ไข Variables ที่ส่งไปให้ Twig Template**

---

**คำถามที่ 2:** ใน Twig Template ของ Drupal จะแปลข้อความเป็นภาษาอื่นด้วยวิธีใด?

A) `{{ translate('text') }}`  
B) `{{ 'text'|t }}`  
C) `{{ i18n('text') }}`  
D) `{{ lang('text') }}`  

**เฉลย: B) ใช้ `|t` filter ซึ่งเรียก t() function**

---

**คำถามที่ 3:** Libraries ใน Drupal คืออะไร?

A) PHP Packages  
B) กลุ่มของ CSS/JS Files ที่ Attach ให้ Pages  
C) Drupal Modules  
D) Twig Extensions  

**เฉลย: B) Libraries ใน .libraries.yml กำหนดกลุ่ม CSS/JS ที่สามารถ Attach ได้**

---

**คำถามที่ 4:** `once()` ใน Drupal JavaScript Behaviors ใช้ทำอะไร?

A) รันโค้ดตอน Page Load ครั้งแรกเท่านั้น  
B) ป้องกัน Event Listener ถูก Attach ซ้ำบน Element เดียวกัน  
C) สร้าง Ajax Request  
D) โหลด Library  

**เฉลย: B) `once()` tracking ว่า Element ไหนถูก Process แล้ว ป้องกัน Attach ซ้ำ**

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- โครงสร้าง Custom Theme และ theme.info.yml
- Libraries สำหรับ CSS/JS
- Twig Templates และ Functions
- Preprocess Hooks
- Theme Settings

---

## ต่อไป

➡️ **[Part 079: Drupal Module Basics](part-079-drupal-module-basics.md)**

เรียนรู้เกี่ยวกับ:
- Custom Module Structure
- Routing System
- Controllers
- Forms API
