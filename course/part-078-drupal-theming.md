# Part 078: Drupal Theming ด้วย Twig

**ระดับ:** Intermediate  
**เวลาเรียน:** 5-6 ชั่วโมง  
**Prerequisites:** Part 077 (Drupal Content), HTML/CSS พื้นฐาน

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. เข้าใจ Twig template engine และ template suggestions
2. สร้าง theme.info.yml และ libraries.yml ที่ถูกต้อง
3. ใช้ Preprocess hooks เพื่อเตรียมตัวแปรสำหรับ templates
4. จัดการ Responsive images และ breakpoints
5. สร้าง Custom Theme สำหรับ Blog

---

## 1. Twig Template Engine

### 1.1 ทำความรู้จัก Twig

Drupal ใช้ Twig เป็น template engine ตั้งแต่เวอร์ชัน 8+ มีความปลอดภัยสูงเพราะ:
- Auto-escape HTML ป้องกัน XSS
- ไม่อนุญาตให้รัน PHP โดยตรง
- Cache templates โดยอัตโนมัติ

### 1.2 Twig Syntax พื้นฐาน

```twig
{# นี่คือ comment ใน Twig #}

{# แสดงผลตัวแปร #}
{{ variable }}
{{ node.title }}
{{ content.field_image }}

{# Block ควบคุม logic #}
{% if condition %}
  ...
{% elseif other_condition %}
  ...
{% else %}
  ...
{% endif %}

{# Loop #}
{% for item in items %}
  {{ item.label }}
{% else %}
  <p>ไม่มีข้อมูล</p>
{% endfor %}

{# Set ตัวแปร #}
{% set greeting = 'สวัสดี' %}
{% set items = ['a', 'b', 'c'] %}

{# Filter #}
{{ title|upper }}
{{ body|striptags|trim }}
{{ date|date('d/m/Y') }}
{{ text|t }}  {# translate #}
{{ amount|number_format(2, '.', ',') }}

{# Function #}
{{ path('node.view', {node: node.id}) }}
{{ url('user.login') }}
{{ file_url(node.field_image.entity.uri.value) }}
```

### 1.3 Twig Filters ที่ใช้บ่อยใน Drupal

```twig
{# แปลงข้อความ (i18n) #}
{{ 'Hello'|t }}
{{ 'Hello @name'|t({'@name': user.name}) }}

{# Clean string สำหรับ HTML attribute #}
{{ title|clean_class }}
{{ node.type|clean_id }}

{# Safe join #}
{{ items|safe_join(', ') }}

{# Check if iterable #}
{% if items is iterable %}

{# Without - แสดงทุกอย่างยกเว้น fields ที่ระบุ #}
{{ content|without('field_sidebar', 'field_footer') }}

{# Render #}
{{ content.field_image|render }}
```

### 1.4 Twig Functions ที่ใช้บ่อย

```twig
{# สร้าง URL #}
<a href="{{ path('node.view', {node: node.id}) }}">อ่านต่อ</a>
<a href="{{ url('<front>') }}">หน้าแรก</a>

{# Link function #}
{{ link(title, url, {'class': ['btn', 'btn-primary']}) }}

{# File URL #}
<img src="{{ file_url(node.field_image.entity.uri.value) }}" />

{# Attach library #}
{{ attach_library('mytheme/main') }}

{# Create attribute object #}
{% set attrs = create_attribute({'class': ['node', 'node--type-article']}) %}
<article {{ attrs }}>
```

---

## 2. Template Suggestions

Template suggestions คือระบบที่ Drupal ใช้หา template ที่ "เฉพาะเจาะจง" ที่สุด

### 2.1 Template Hierarchy สำหรับ Node

Drupal ค้นหา templates ตามลำดับนี้ (specific → general):

```
node--{nid}.html.twig              # node ID เฉพาะ
node--{type}--{view-mode}.html.twig
node--{type}.html.twig             # เช่น node--article.html.twig
node--{view-mode}.html.twig        # เช่น node--teaser.html.twig
node.html.twig                     # default fallback
```

### 2.2 Template Suggestions สำหรับ element อื่นๆ

```
{# Block #}
block--{module}.html.twig
block--{region}--{module}.html.twig
block--{block-id}.html.twig
block.html.twig

{# Field #}
field--{field-name}--{node-type}.html.twig
field--{field-name}.html.twig
field--{field-type}.html.twig
field.html.twig

{# Views #}
views-view--{view-name}--{display-id}.html.twig
views-view--{view-name}.html.twig
views-view.html.twig

views-view-unformatted--{view-name}--{display-id}.html.twig
views-view-unformatted--{view-name}.html.twig

{# Page #}
page--front.html.twig
page--node--{nid}.html.twig
page--node--{type}.html.twig
page--node.html.twig
page.html.twig
```

### 2.3 Debug เพื่อหา Template Suggestions

เปิด Twig debug ใน settings.local.php:

```php
$settings['twig_debug'] = TRUE;
$settings['twig_auto_reload'] = TRUE;
$settings['twig_cache'] = FALSE;
```

จากนั้น inspect HTML source จะเห็น:

```html
<!-- THEME DEBUG -->
<!-- THEME HOOK: 'node' -->
<!-- FILE NAME SUGGESTIONS:
   * node--1--full.html.twig
   * node--1.html.twig
   x node--article--full.html.twig
   * node--article.html.twig
   * node--full.html.twig
   * node.html.twig
-->
<!-- BEGIN OUTPUT from 'themes/mytheme/templates/node--article--full.html.twig' -->
```

`x` หมายถึง template ที่ถูก active ใช้อยู่

---

## 3. สร้าง Custom Theme

### 3.1 โครงสร้าง Theme

```
web/themes/custom/mytheme/
├── mytheme.info.yml          # Theme definition
├── mytheme.libraries.yml     # CSS/JS libraries
├── mytheme.theme             # PHP preprocess hooks
├── screenshot.png            # Theme screenshot (optional)
├── config/
│   └── install/
│       └── mytheme.settings.yml
├── templates/
│   ├── layout/
│   │   ├── page.html.twig
│   │   └── html.html.twig
│   ├── node/
│   │   ├── node.html.twig
│   │   ├── node--article.html.twig
│   │   └── node--article--teaser.html.twig
│   ├── block/
│   │   └── block.html.twig
│   ├── field/
│   │   └── field--field-image.html.twig
│   └── views/
│       └── views-view--news-listing.html.twig
├── css/
│   ├── base/
│   │   ├── reset.css
│   │   └── typography.css
│   ├── components/
│   │   ├── navigation.css
│   │   ├── cards.css
│   │   └── hero.css
│   └── layout/
│       └── layout.css
├── js/
│   ├── main.js
│   └── mobile-menu.js
└── images/
    └── logo.svg
```

### 3.2 mytheme.info.yml

```yaml
name: 'My Blog Theme'
type: theme
description: 'Custom theme for Blog website'
package: Custom
core_version_requirement: ^10
base theme: stable9

libraries:
  - mytheme/global-styling
  - mytheme/global-scripts

regions:
  header: Header
  primary_menu: 'Primary Menu'
  breadcrumb: Breadcrumb
  hero: 'Hero Area'
  content: Content
  sidebar_first: 'Sidebar (First)'
  sidebar_second: 'Sidebar (Second)'
  featured: 'Featured Content'
  footer_first: 'Footer Column 1'
  footer_second: 'Footer Column 2'
  footer_third: 'Footer Column 3'
  footer_bottom: 'Footer Bottom'

# ตั้งค่า default logo และ favicon
logo:
  use: images/logo.svg

# Libraries Override (ลบ libraries จาก core หรือ modules)
libraries-override:
  core/drupal.dialog.ajax: false
  
# Libraries Extend (เพิ่ม CSS/JS ใน existing library)
libraries-extend:
  core/drupal:
    - mytheme/drupal-tweaks
```

### 3.3 mytheme.libraries.yml

```yaml
# Global styles & scripts
global-styling:
  version: 1.0
  css:
    base:
      css/base/reset.css: {}
      css/base/typography.css: {}
    layout:
      css/layout/layout.css: {}
    component:
      css/components/navigation.css: {}
      css/components/cards.css: {}
      css/components/hero.css: {}
      css/components/footer.css: {}
  dependencies:
    - core/normalize

global-scripts:
  version: 1.0
  js:
    js/main.js:
      scope: footer  # โหลด JS ที่ footer
      weight: 10
  dependencies:
    - core/drupal
    - core/jquery  # ถ้าต้องการ jQuery

# Specific component library
article-page:
  version: 1.0
  css:
    component:
      css/components/article.css: {}
      css/components/comments.css: {}
  js:
    js/article.js:
      scope: footer

# Drupal behaviors tweaks
drupal-tweaks:
  version: 1.0
  js:
    js/drupal-tweaks.js:
      scope: header
      weight: -100

# Third-party library
slick-slider:
  version: 1.8.1
  css:
    theme:
      //cdnjs.cloudflare.com/ajax/libs/slick-carousel/1.8.1/slick.min.css:
        type: external
  js:
    //cdnjs.cloudflare.com/ajax/libs/slick-carousel/1.8.1/slick.min.js:
      type: external
      scope: footer
  dependencies:
    - core/jquery
```

---

## 4. Preprocess Hooks

Preprocess hooks ใช้เตรียมตัวแปรก่อนส่งไปยัง Twig templates

### 4.1 hook_preprocess_node()

```php
<?php
// mytheme.theme

use Drupal\node\NodeInterface;
use Drupal\file\Entity\File;

/**
 * Implements hook_preprocess_node().
 */
function mytheme_preprocess_node(&$variables) {
  /** @var \Drupal\node\NodeInterface $node */
  $node = $variables['node'];
  
  // เพิ่ม CSS classes ตาม content type
  $variables['attributes']['class'][] = 'node';
  $variables['attributes']['class'][] = 'node--type-' . $node->bundle();
  
  if ($node->isPromoted()) {
    $variables['attributes']['class'][] = 'node--promoted';
  }
  
  // เพิ่ม custom variables
  $variables['node_type'] = $node->bundle();
  $variables['is_front'] = \Drupal::service('path.matcher')->isFrontPage();
  
  // ประมวลผล read time สำหรับ article
  if ($node->bundle() === 'article') {
    mytheme_preprocess_article($variables, $node);
  }
}

/**
 * Preprocess สำหรับ article.
 */
function mytheme_preprocess_article(&$variables, NodeInterface $node) {
  // คำนวณ read time
  if ($node->hasField('body') && !$node->get('body')->isEmpty()) {
    $body = $node->get('body')->value;
    $word_count = str_word_count(strip_tags($body));
    $read_time = max(1, ceil($word_count / 200));
    $variables['read_time'] = $read_time;
  }
  
  // เพิ่ม featured image URL
  if ($node->hasField('field_featured_image') && !$node->get('field_featured_image')->isEmpty()) {
    $image_field = $node->get('field_featured_image')->first();
    if ($file = $image_field->entity) {
      $variables['featured_image_url'] = \Drupal::service('file_url_generator')
        ->generateAbsoluteString($file->getFileUri());
      $variables['featured_image_alt'] = $image_field->alt;
    }
  }
  
  // Author info
  $author = $node->getOwner();
  $variables['author_name'] = $author->getDisplayName();
  $variables['author_url'] = $author->toUrl()->toString();
}
```

### 4.2 hook_preprocess_page()

```php
/**
 * Implements hook_preprocess_page().
 */
function mytheme_preprocess_page(&$variables) {
  // เพิ่ม site name
  $config = \Drupal::config('system.site');
  $variables['site_name'] = $config->get('name');
  $variables['site_slogan'] = $config->get('slogan');
  
  // ตรวจสอบ page type
  $variables['is_front'] = \Drupal::service('path.matcher')->isFrontPage();
  
  // เพิ่ม layout class ตาม sidebar
  $layout_class = 'layout--no-sidebar';
  if (!empty($variables['page']['sidebar_first']) && !empty($variables['page']['sidebar_second'])) {
    $layout_class = 'layout--two-sidebars';
  } elseif (!empty($variables['page']['sidebar_first'])) {
    $layout_class = 'layout--sidebar-first';
  } elseif (!empty($variables['page']['sidebar_second'])) {
    $layout_class = 'layout--sidebar-second';
  }
  $variables['layout_class'] = $layout_class;
  
  // Current user info
  $current_user = \Drupal::currentUser();
  $variables['logged_in'] = $current_user->isAuthenticated();
  $variables['user_name'] = $current_user->getDisplayName();
}
```

### 4.3 hook_preprocess_html()

```php
/**
 * Implements hook_preprocess_html().
 */
function mytheme_preprocess_html(&$variables) {
  // เพิ่ม body classes
  $node = \Drupal::routeMatch()->getParameter('node');
  if ($node) {
    $variables['attributes']['class'][] = 'page-node-' . $node->id();
    $variables['attributes']['class'][] = 'node-type-' . $node->bundle();
  }
  
  // Dark mode support
  $variables['attributes']['class'][] = 'theme-light'; // default
  
  // Language direction
  $language = \Drupal::languageManager()->getCurrentLanguage();
  $variables['attributes']['dir'] = $language->getDirection();
  $variables['attributes']['lang'] = $language->getId();
}
```

### 4.4 hook_preprocess_block()

```php
/**
 * Implements hook_preprocess_block().
 */
function mytheme_preprocess_block(&$variables) {
  // เพิ่ม block machine name เป็น class
  if (!empty($variables['elements']['#id'])) {
    $variables['attributes']['class'][] = 'block-' . str_replace('_', '-', $variables['elements']['#id']);
  }
  
  // เพิ่ม region class
  if (!empty($variables['elements']['#configuration']['region'])) {
    $variables['attributes']['class'][] = 'block-region-' . $variables['elements']['#configuration']['region'];
  }
}
```

### 4.5 hook_preprocess_field()

```php
/**
 * Implements hook_preprocess_field().
 */
function mytheme_preprocess_field(&$variables) {
  $field_name = $variables['field_name'];
  $entity_type = $variables['element']['#entity_type'];
  
  // เพิ่ม custom variables สำหรับ specific fields
  if ($field_name === 'field_tags' && $entity_type === 'node') {
    $variables['is_tags'] = TRUE;
  }
}
```

### 4.6 hook_theme_suggestions_HOOK_alter()

```php
/**
 * Implements hook_theme_suggestions_node_alter().
 */
function mytheme_theme_suggestions_node_alter(&$suggestions, $variables) {
  $node = $variables['elements']['#node'];
  $view_mode = $variables['elements']['#view_mode'];
  
  // เพิ่ม custom suggestion ตาม field value
  if ($node->bundle() === 'article') {
    if ($node->hasField('field_is_featured') && $node->get('field_is_featured')->value) {
      array_splice($suggestions, 1, 0, 'node__article__featured');
    }
  }
}

/**
 * Implements hook_theme_suggestions_page_alter().
 */
function mytheme_theme_suggestions_page_alter(&$suggestions, $variables) {
  // เพิ่ม suggestions ตาม taxonomy term
  if ($term = \Drupal::routeMatch()->getParameter('taxonomy_term')) {
    $vocabulary = $term->bundle();
    $suggestions[] = 'page__taxonomy__' . $vocabulary;
    $suggestions[] = 'page__taxonomy__' . $vocabulary . '__' . $term->id();
  }
}
```

---

## 5. Templates

### 5.1 page.html.twig

```twig
{#
/**
 * @file
 * Theme override to display a single page.
 */
#}
<!DOCTYPE html>
<html{{ html_attributes }}>
  <head>
    <head-placeholder token="{{ placeholder_token }}">
    <title>{{ head_title|safe_join(' | ') }}</title>
    <css-placeholder token="{{ placeholder_token }}">
    <js-placeholder token="{{ placeholder_token }}">
  </head>
  <body{{ attributes }}>
    <a href="#main-content" class="visually-hidden focusable skip-link">
      {{ 'Skip to main content'|t }}
    </a>
    
    {{ page_top }}
    
    <div class="page-wrapper">
      {# Header #}
      <header class="site-header" role="banner">
        <div class="container">
          <div class="site-branding">
            {% if site_logo %}
              <a href="{{ path('<front>') }}" rel="home" class="site-logo">
                <img src="{{ site_logo }}" alt="{{ 'Home'|t }}" />
              </a>
            {% endif %}
            <div class="site-name-slogan">
              {% if site_name %}
                <span class="site-name">{{ site_name }}</span>
              {% endif %}
              {% if site_slogan %}
                <span class="site-slogan">{{ site_slogan }}</span>
              {% endif %}
            </div>
          </div>
          
          {% if page.primary_menu %}
            <nav class="site-navigation" role="navigation">
              {{ page.primary_menu }}
            </nav>
          {% endif %}
        </div>
      </header>
      
      {# Main content #}
      <main id="main-content" class="main-content {{ layout_class }}">
        <div class="container">
          {% if page.breadcrumb %}
            <div class="breadcrumb-wrapper">
              {{ page.breadcrumb }}
            </div>
          {% endif %}
          
          <div class="content-wrapper">
            <div class="main-column">
              {{ page.content }}
            </div>
            
            {% if page.sidebar_first %}
              <aside class="sidebar sidebar--first" role="complementary">
                {{ page.sidebar_first }}
              </aside>
            {% endif %}
            
            {% if page.sidebar_second %}
              <aside class="sidebar sidebar--second" role="complementary">
                {{ page.sidebar_second }}
              </aside>
            {% endif %}
          </div>
        </div>
      </main>
      
      {# Footer #}
      <footer class="site-footer" role="contentinfo">
        <div class="container">
          {% if page.footer_first or page.footer_second or page.footer_third %}
            <div class="footer-columns">
              {% if page.footer_first %}
                <div class="footer-column">{{ page.footer_first }}</div>
              {% endif %}
              {% if page.footer_second %}
                <div class="footer-column">{{ page.footer_second }}</div>
              {% endif %}
              {% if page.footer_third %}
                <div class="footer-column">{{ page.footer_third }}</div>
              {% endif %}
            </div>
          {% endif %}
          
          {% if page.footer_bottom %}
            <div class="footer-bottom">{{ page.footer_bottom }}</div>
          {% endif %}
        </div>
      </footer>
    </div>
    
    {{ page_bottom }}
    <js-bottom-placeholder token="{{ placeholder_token }}">
  </body>
</html>
```

### 5.2 node--article.html.twig

```twig
{#
/**
 * @file
 * Theme override for Article node.
 */
#}
<article{{ attributes.addClass('article-node') }}>
  
  {# Featured Image #}
  {% if content.field_featured_image %}
    <div class="article-hero">
      {{ content.field_featured_image }}
    </div>
  {% endif %}
  
  <div class="article-content">
    {# Meta information #}
    <div class="article-meta">
      {% if content.field_category %}
        <span class="article-category">
          {{ content.field_category }}
        </span>
      {% endif %}
      
      <time class="article-date" datetime="{{ node.created.value|date('c') }}">
        {{ node.created.value|date('d F Y') }}
      </time>
      
      {% if read_time %}
        <span class="article-read-time">
          {{ read_time }} {{ 'min read'|t }}
        </span>
      {% endif %}
    </div>
    
    {# Title #}
    {{ title_prefix }}
    {% if not page %}
      <h2{{ title_attributes }}>
        <a href="{{ url }}" rel="bookmark">{{ label }}</a>
      </h2>
    {% else %}
      <h1{{ title_attributes }}>{{ label }}</h1>
    {% endif %}
    {{ title_suffix }}
    
    {# Subtitle #}
    {% if content.field_subtitle %}
      <p class="article-subtitle">
        {{ content.field_subtitle }}
      </p>
    {% endif %}
    
    {# Author #}
    {% if display_submitted %}
      <div class="article-author">
        <a href="{{ author_url }}">{{ author_name }}</a>
      </div>
    {% endif %}
    
    {# Body #}
    <div class="article-body">
      {{ content.body }}
    </div>
    
    {# Tags #}
    {% if content.field_tags %}
      <div class="article-tags">
        <span class="tags-label">{{ 'Tags:'|t }}</span>
        {{ content.field_tags }}
      </div>
    {% endif %}
    
    {# Remaining fields #}
    {{ content|without('field_featured_image', 'field_category', 'field_subtitle', 'body', 'field_tags') }}
  </div>
  
</article>
```

### 5.3 node--article--teaser.html.twig

```twig
{#
/**
 * @file
 * Teaser template for Article.
 */
#}
<article{{ attributes.addClass('article-card') }}>
  
  {% if content.field_featured_image %}
    <div class="card-image">
      <a href="{{ url }}">
        {{ content.field_featured_image }}
      </a>
    </div>
  {% endif %}
  
  <div class="card-body">
    {% if content.field_category %}
      <div class="card-category">{{ content.field_category }}</div>
    {% endif %}
    
    {{ title_prefix }}
    <h3{{ title_attributes.addClass('card-title') }}>
      <a href="{{ url }}" rel="bookmark">{{ label }}</a>
    </h3>
    {{ title_suffix }}
    
    <div class="card-excerpt">
      {{ content.body }}
    </div>
    
    <div class="card-footer">
      <time datetime="{{ node.created.value|date('c') }}">
        {{ node.created.value|date('d M Y') }}
      </time>
      
      {% if read_time %}
        <span>{{ read_time }} {{ 'min'|t }}</span>
      {% endif %}
      
      <a href="{{ url }}" class="read-more">{{ 'อ่านต่อ'|t }}</a>
    </div>
  </div>
  
</article>
```

---

## 6. Responsive Images & Breakpoints

### 6.1 breakpoints.yml

```yaml
# mytheme.breakpoints.yml
mytheme.mobile:
  label: Mobile
  mediaQuery: '(max-width: 767px)'
  weight: 1
  multipliers:
    - 1x
    - 2x

mytheme.tablet:
  label: Tablet
  mediaQuery: '(min-width: 768px) and (max-width: 1023px)'
  weight: 2
  multipliers:
    - 1x
    - 2x

mytheme.desktop:
  label: Desktop
  mediaQuery: '(min-width: 1024px)'
  weight: 3
  multipliers:
    - 1x
    - 2x

mytheme.wide:
  label: 'Wide Desktop'
  mediaQuery: '(min-width: 1440px)'
  weight: 4
  multipliers:
    - 1x
    - 2x
```

### 6.2 สร้าง Image Styles

ไปที่: **Configuration → Media → Image styles → Add image style**

```
hero_desktop: Scale and crop 1920x600
hero_tablet: Scale and crop 1024x400
hero_mobile: Scale and crop 768x300
article_featured: Scale and crop 800x450
article_thumbnail: Scale and crop 400x225
article_card: Scale and crop 600x338
```

### 6.3 สร้าง Responsive Image Style

ไปที่: **Configuration → Media → Responsive image styles → Add responsive image style**

```
Name: Article Featured
Breakpoint group: mytheme

Mappings:
  mytheme.wide (1x):    hero_desktop
  mytheme.desktop (1x): hero_desktop
  mytheme.tablet (1x):  hero_tablet
  mytheme.mobile (1x):  hero_mobile
  Fallback image style: article_featured
```

### 6.4 ใช้ Responsive Image ใน Template

```twig
{# ใช้ responsive_image function #}
{% set image_uri = node.field_featured_image.entity.uri.value %}
{% if image_uri %}
  {{ responsive_image(image_uri, 'article_featured') }}
{% endif %}
```

---

## Workshop: สร้าง Custom Theme สำหรับ Blog

### เป้าหมาย
สร้าง theme "BlogTheme" ที่มี:
- Modern card layout สำหรับ article listing
- Hero section สำหรับ homepage
- Responsive design
- Custom preprocess hooks

### ขั้นตอนที่ 1: สร้างโครงสร้าง Theme

```bash
mkdir -p web/themes/custom/blogtheme/{templates/{layout,node,block,views,field},css/{base,components,layout},js,images}

# สร้างไฟล์หลัก
touch web/themes/custom/blogtheme/blogtheme.info.yml
touch web/themes/custom/blogtheme/blogtheme.libraries.yml
touch web/themes/custom/blogtheme/blogtheme.theme
touch web/themes/custom/blogtheme/blogtheme.breakpoints.yml
```

### ขั้นตอนที่ 2: blogtheme.info.yml

```yaml
name: 'Blog Theme'
type: theme
description: 'Modern blog theme with card layout'
package: Custom
core_version_requirement: ^10
base theme: stable9

libraries:
  - blogtheme/global-styling
  - blogtheme/global-scripts

regions:
  header: Header
  primary_menu: Navigation
  hero: Hero
  content: Content
  sidebar: Sidebar
  footer: Footer
```

### ขั้นตอนที่ 3: blogtheme.libraries.yml

```yaml
global-styling:
  css:
    base:
      css/base/variables.css: {}
      css/base/reset.css: {}
      css/base/typography.css: {}
    layout:
      css/layout/grid.css: {}
      css/layout/layout.css: {}
    component:
      css/components/header.css: {}
      css/components/navigation.css: {}
      css/components/hero.css: {}
      css/components/cards.css: {}
      css/components/footer.css: {}

global-scripts:
  js:
    js/main.js:
      scope: footer
  dependencies:
    - core/drupal
```

### ขั้นตอนที่ 4: CSS Variables

```css
/* css/base/variables.css */
:root {
  --color-primary: #2563eb;
  --color-secondary: #1e40af;
  --color-accent: #f59e0b;
  --color-text: #1f2937;
  --color-text-light: #6b7280;
  --color-bg: #ffffff;
  --color-bg-light: #f9fafb;
  --color-border: #e5e7eb;
  
  --font-sans: 'Inter', -apple-system, sans-serif;
  --font-serif: 'Merriweather', Georgia, serif;
  
  --spacing-xs: 0.25rem;
  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;
  --spacing-lg: 1.5rem;
  --spacing-xl: 2rem;
  --spacing-2xl: 3rem;
  
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 16px;
  
  --shadow-sm: 0 1px 2px rgba(0,0,0,0.05);
  --shadow-md: 0 4px 6px rgba(0,0,0,0.07);
  --shadow-lg: 0 10px 15px rgba(0,0,0,0.1);
}
```

### ขั้นตอนที่ 5: Card CSS

```css
/* css/components/cards.css */
.article-card {
  background: var(--color-bg);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-sm);
  overflow: hidden;
  transition: transform 0.2s, box-shadow 0.2s;
}

.article-card:hover {
  transform: translateY(-4px);
  box-shadow: var(--shadow-lg);
}

.article-card .card-image img {
  width: 100%;
  height: 200px;
  object-fit: cover;
}

.article-card .card-body {
  padding: var(--spacing-lg);
}

.article-card .card-category {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: var(--color-primary);
  margin-bottom: var(--spacing-sm);
}

.article-card .card-title a {
  color: var(--color-text);
  text-decoration: none;
  font-size: 1.125rem;
  font-weight: 700;
  line-height: 1.4;
}

.article-card .card-title a:hover {
  color: var(--color-primary);
}

.article-card .card-footer {
  display: flex;
  align-items: center;
  gap: var(--spacing-md);
  margin-top: var(--spacing-md);
  padding-top: var(--spacing-md);
  border-top: 1px solid var(--color-border);
  font-size: 0.875rem;
  color: var(--color-text-light);
}

.read-more {
  margin-left: auto;
  color: var(--color-primary);
  font-weight: 500;
  text-decoration: none;
}
```

### ขั้นตอนที่ 6: blogtheme.theme

```php
<?php
// blogtheme.theme

function blogtheme_preprocess_node(&$variables) {
  $node = $variables['node'];
  
  if ($node->bundle() === 'article') {
    // คำนวณ read time
    if (!$node->get('body')->isEmpty()) {
      $body = $node->get('body')->value;
      $words = str_word_count(strip_tags($body));
      $variables['read_time'] = max(1, ceil($words / 200));
    }
    
    // Author display name
    $variables['author_name'] = $node->getOwner()->getDisplayName();
    
    // Format date in Thai
    $timestamp = $node->get('created')->value;
    $thai_months = [
      1 => 'ม.ค.', 2 => 'ก.พ.', 3 => 'มี.ค.', 4 => 'เม.ย.',
      5 => 'พ.ค.', 6 => 'มิ.ย.', 7 => 'ก.ค.', 8 => 'ส.ค.',
      9 => 'ก.ย.', 10 => 'ต.ค.', 11 => 'พ.ย.', 12 => 'ธ.ค.',
    ];
    $month = (int) date('n', $timestamp);
    $variables['thai_date'] = date('j', $timestamp) . ' ' . 
                              $thai_months[$month] . ' ' . 
                              (date('Y', $timestamp) + 543);
  }
}

function blogtheme_preprocess_page(&$variables) {
  $config = \Drupal::config('system.site');
  $variables['site_name'] = $config->get('name');
}
```

### ขั้นตอนที่ 7: Enable Theme

```bash
# Enable theme
drush theme:enable blogtheme

# Set as default theme
drush config:set system.theme default blogtheme -y

drush cr
```

---

## Quiz

**ข้อ 1:** Template suggestion ไหนมีความเฉพาะเจาะจงมากที่สุด (highest specificity)?
- a) `node.html.twig`
- b) `node--article.html.twig`
- c) `node--1.html.twig`
- d) `node--full.html.twig`

**เฉลย:** c) `node--{nid}.html.twig` มี specificity สูงที่สุดเพราะ target node ID เฉพาะ

---

**ข้อ 2:** `{{ content|without('body') }}` ทำอะไร?
- a) ลบ field body ออกจาก content type
- b) แสดงทุก fields ยกเว้น body
- c) Filter HTML tags ออกจาก body
- d) เปลี่ยน field body เป็น empty string

**เฉลย:** b) `without()` filter แสดง content เต็มแต่ยกเว้น fields ที่ระบุ

---

**ข้อ 3:** hook_preprocess_node() ใช้ทำอะไร?
- a) สร้าง node ใหม่
- b) เพิ่มหรือแก้ไข template variables ก่อนส่งไปยัง Twig
- c) Override node edit form
- d) เปลี่ยน node permissions

**เฉลย:** b) Preprocess hooks เตรียม/แก้ไข variables สำหรับ template rendering

---

**ข้อ 4:** libraries.yml ใช้สำหรับอะไร?
- a) Import PHP libraries
- b) ประกาศ CSS/JS assets และ dependencies ของ theme
- c) Register Drupal hooks
- d) สร้าง theme regions

**เฉลย:** b) libraries.yml ประกาศ CSS/JS groups และ dependencies เพื่อใช้กับ `attach_library()`

---

**ข้อ 5:** จะเปิด Twig debug mode ได้อย่างไร?
- a) แก้ไข mytheme.info.yml
- b) ตั้งค่าใน settings.local.php: `$settings['twig_debug'] = TRUE;`
- c) ใช้ drush command: `drush twig:debug`
- d) แก้ไข composer.json

**เฉลย:** b) ตั้งค่า `$settings['twig_debug'] = TRUE;` ใน settings.local.php

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- Twig syntax, filters, และ functions สำหรับ Drupal
- Template suggestion system และการ debug
- โครงสร้าง Custom Theme พร้อม info.yml และ libraries.yml
- Preprocess hooks สำหรับเตรียม template variables
- Responsive images และ breakpoints

**Part ถัดไป:** Part 079 - Drupal Custom Module Basics
