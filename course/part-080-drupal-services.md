# Part 080: Drupal Services, Plugins, Entity API & Config API

**ระดับ:** Advanced  
**เวลาเรียน:** 6-7 ชั่วโมง  
**Prerequisites:** Part 079, PHP OOP, Dependency Injection concept

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. สร้างและใช้ Services ด้วย Dependency Injection
2. สร้าง Block Plugins และ Field Formatter Plugins
3. ใช้ Entity API เข้าถึง entities, fields, entity queries
4. ใช้ Config API สำหรับ configuration management
5. สร้าง Custom Block Plugin และ Custom Field Formatter

---

## 1. Services & Dependency Injection

### 1.1 Drupal Service Container

Drupal ใช้ Symfony Service Container (Dependency Injection Container) เพื่อจัดการ services

```php
// ดึง service โดยตรง (global access - ใช้เฉพาะใน procedural code)
$service = \Drupal::service('service.name');

// ดีกว่า: Dependency Injection ใน class
public function __construct(ServiceInterface $service) {
  $this->service = $service;
}
```

### 1.2 สร้าง Service

```yaml
# mymodule.services.yml
services:
  # Service หลัก
  mymodule.calculator:
    class: Drupal\mymodule\Calculator
    arguments:
      - '@config.factory'     # inject config factory
      - '@entity_type.manager'  # inject entity type manager
      - '@cache.default'      # inject cache

  # Service แบบ Singleton (default)
  mymodule.formatter:
    class: Drupal\mymodule\ContentFormatter
    arguments: ['@renderer']

  # Service แบบ Lazy loading
  mymodule.heavy_service:
    class: Drupal\mymodule\HeavyService
    lazy: true
    
  # Tagged service (สำหรับ plugin system)
  mymodule.event_subscriber:
    class: Drupal\mymodule\EventSubscriber\NodeSubscriber
    tags:
      - { name: event_subscriber }
```

### 1.3 สร้าง Service Class

```php
<?php
// src/Calculator.php

namespace Drupal\mymodule;

use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Cache\CacheBackendInterface;

/**
 * Calculator service.
 */
class Calculator {

  /**
   * @var \Drupal\Core\Config\ImmutableConfig
   */
  protected $config;

  /**
   * @var \Drupal\Core\Entity\EntityTypeManagerInterface
   */
  protected $entityTypeManager;

  /**
   * @var \Drupal\Core\Cache\CacheBackendInterface
   */
  protected $cache;

  /**
   * Constructor.
   */
  public function __construct(
    ConfigFactoryInterface $config_factory,
    EntityTypeManagerInterface $entity_type_manager,
    CacheBackendInterface $cache
  ) {
    $this->config = $config_factory->get('mymodule.settings');
    $this->entityTypeManager = $entity_type_manager;
    $this->cache = $cache;
  }

  /**
   * คำนวณ statistics สำหรับ node.
   *
   * @param int $nid
   * @return array
   */
  public function calculateNodeStats(int $nid): array {
    $cache_id = 'mymodule:node_stats:' . $nid;
    
    // ลองดึงจาก cache ก่อน
    if ($cached = $this->cache->get($cache_id)) {
      return $cached->data;
    }
    
    // คำนวณใหม่
    $node = $this->entityTypeManager->getStorage('node')->load($nid);
    if (!$node) {
      return [];
    }
    
    $stats = [
      'word_count' => $this->countWords($node),
      'reading_time' => $this->estimateReadingTime($node),
      'image_count' => $this->countImages($node),
    ];
    
    // บันทึก cache พร้อม cache tags
    $this->cache->set(
      $cache_id,
      $stats,
      \Drupal\Core\Cache\CacheBackendInterface::CACHE_PERMANENT,
      ['node:' . $nid]  // Invalidate เมื่อ node เปลี่ยน
    );
    
    return $stats;
  }

  protected function countWords($node): int {
    $body = $node->get('body')->value ?? '';
    return str_word_count(strip_tags($body));
  }

  protected function estimateReadingTime($node): int {
    $words = $this->countWords($node);
    return max(1, ceil($words / 200));
  }

  protected function countImages($node): int {
    $body = $node->get('body')->value ?? '';
    preg_match_all('/<img/i', $body, $matches);
    return count($matches[0]);
  }
}
```

### 1.4 Event Subscriber

```php
<?php
// src/EventSubscriber/NodeSubscriber.php

namespace Drupal\mymodule\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Drupal\core_event_dispatcher\Event\Entity\EntityPresaveEvent;
use Drupal\core_event_dispatcher\HookEventDispatcherInterface;

/**
 * Subscribes to node events.
 */
class NodeSubscriber implements EventSubscriberInterface {

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents() {
    return [
      HookEventDispatcherInterface::ENTITY_PRE_SAVE => 'onEntityPresave',
    ];
  }

  /**
   * Called on entity pre save.
   */
  public function onEntityPresave(EntityPresaveEvent $event) {
    $entity = $event->getEntity();
    if ($entity->getEntityTypeId() === 'node' && $entity->bundle() === 'article') {
      // Do something before node save
    }
  }
}
```

---

## 2. Plugin System

### 2.1 สร้าง Block Plugin

```php
<?php
// src/Plugin/Block/RelatedPostsBlock.php

namespace Drupal\related_posts\Plugin\Block;

use Drupal\Core\Block\BlockBase;
use Drupal\Core\Block\BlockPluginInterface;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Drupal\related_posts\RelatedPostsService;
use Drupal\Core\Routing\RouteMatchInterface;

/**
 * Provides a 'Related Posts' Block.
 *
 * @Block(
 *   id = "related_posts_block",
 *   admin_label = @Translation("Related Posts"),
 *   category = @Translation("Content"),
 * )
 */
class RelatedPostsBlock extends BlockBase implements ContainerFactoryPluginInterface {

  /**
   * The related posts service.
   *
   * @var \Drupal\related_posts\RelatedPostsService
   */
  protected $relatedPostsService;

  /**
   * The route match.
   *
   * @var \Drupal\Core\Routing\RouteMatchInterface
   */
  protected $routeMatch;

  /**
   * Constructor.
   */
  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    RelatedPostsService $related_posts_service,
    RouteMatchInterface $route_match
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
    $this->relatedPostsService = $related_posts_service;
    $this->routeMatch = $route_match;
  }

  /**
   * {@inheritdoc}
   */
  public static function create(
    ContainerInterface $container,
    array $configuration,
    $plugin_id,
    $plugin_definition
  ) {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('related_posts.service'),
      $container->get('current_route_match')
    );
  }

  /**
   * {@inheritdoc}
   * Block configuration form.
   */
  public function blockForm($form, FormStateInterface $form_state) {
    $form = parent::blockForm($form, $form_state);
    
    $config = $this->getConfiguration();
    
    $form['max_posts'] = [
      '#type' => 'number',
      '#title' => $this->t('จำนวน posts สูงสุด'),
      '#default_value' => $config['max_posts'] ?? 4,
      '#min' => 1,
      '#max' => 12,
    ];
    
    $form['show_image'] = [
      '#type' => 'checkbox',
      '#title' => $this->t('แสดงรูปภาพ'),
      '#default_value' => $config['show_image'] ?? TRUE,
    ];
    
    $form['title_override'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Block title override'),
      '#default_value' => $config['title_override'] ?? 'บทความที่เกี่ยวข้อง',
    ];
    
    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function blockSubmit($form, FormStateInterface $form_state) {
    parent::blockSubmit($form, $form_state);
    $this->configuration['max_posts'] = $form_state->getValue('max_posts');
    $this->configuration['show_image'] = $form_state->getValue('show_image');
    $this->configuration['title_override'] = $form_state->getValue('title_override');
  }

  /**
   * {@inheritdoc}
   */
  public function build() {
    // ดึง current node จาก route
    $node = $this->routeMatch->getParameter('node');
    
    if (!$node) {
      return [];
    }
    
    $config = $this->getConfiguration();
    $max_posts = $config['max_posts'] ?? 4;
    
    $related_posts = $this->relatedPostsService->getRelatedPosts($node->id(), $max_posts);
    
    if (empty($related_posts)) {
      return [];
    }
    
    // สร้าง render array
    $items = [];
    foreach ($related_posts as $related_node) {
      $items[] = [
        '#theme' => 'related_posts_item',
        '#node' => $related_node,
        '#show_image' => $config['show_image'] ?? TRUE,
        '#url' => $related_node->toUrl()->toString(),
      ];
    }
    
    $build = [
      '#theme' => 'related_posts_block',
      '#nodes' => $related_posts,
      '#items' => $items,
      '#title' => $config['title_override'] ?? 'บทความที่เกี่ยวข้อง',
      // Cache settings
      '#cache' => [
        'tags' => array_merge(
          ['node:' . $node->id()],
          array_map(fn($n) => 'node:' . $n->id(), $related_posts)
        ),
        'contexts' => ['route'],
        'max-age' => 3600,
      ],
    ];
    
    return $build;
  }
}
```

### 2.2 สร้าง Field Formatter Plugin

```php
<?php
// src/Plugin/Field/FieldFormatter/ReadTimeFormatter.php

namespace Drupal\related_posts\Plugin\Field\FieldFormatter;

use Drupal\Core\Field\FormatterBase;
use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Form\FormStateInterface;

/**
 * Plugin implementation of the 'read_time' formatter.
 *
 * @FieldFormatter(
 *   id = "read_time_formatter",
 *   label = @Translation("Read Time"),
 *   field_types = {
 *     "text",
 *     "text_long",
 *     "text_with_summary",
 *   }
 * )
 */
class ReadTimeFormatter extends FormatterBase {

  /**
   * {@inheritdoc}
   */
  public static function defaultSettings() {
    return [
      'words_per_minute' => 200,
      'show_icon' => TRUE,
      'format' => 'minutes',
    ] + parent::defaultSettings();
  }

  /**
   * {@inheritdoc}
   */
  public function settingsForm(array $form, FormStateInterface $form_state) {
    $form = parent::settingsForm($form, $form_state);
    
    $form['words_per_minute'] = [
      '#type' => 'number',
      '#title' => $this->t('ความเร็วการอ่าน (words/minute)'),
      '#default_value' => $this->getSetting('words_per_minute'),
      '#min' => 100,
      '#max' => 500,
    ];
    
    $form['show_icon'] = [
      '#type' => 'checkbox',
      '#title' => $this->t('แสดง icon นาฬิกา'),
      '#default_value' => $this->getSetting('show_icon'),
    ];
    
    $form['format'] = [
      '#type' => 'select',
      '#title' => $this->t('รูปแบบ'),
      '#options' => [
        'minutes' => $this->t('X นาที'),
        'minutes_seconds' => $this->t('X นาที Y วินาที'),
        'verbose' => $this->t('ใช้เวลาอ่านประมาณ X นาที'),
      ],
      '#default_value' => $this->getSetting('format'),
    ];
    
    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function settingsSummary() {
    $summary = [];
    $summary[] = $this->t('Speed: @wpm words/min', [
      '@wpm' => $this->getSetting('words_per_minute'),
    ]);
    return $summary;
  }

  /**
   * {@inheritdoc}
   */
  public function viewElements(FieldItemListInterface $items, $langcode) {
    $elements = [];
    $wpm = $this->getSetting('words_per_minute');
    $format = $this->getSetting('format');
    $show_icon = $this->getSetting('show_icon');
    
    foreach ($items as $delta => $item) {
      $text = strip_tags($item->value);
      $word_count = str_word_count($text);
      $seconds = round($word_count / $wpm * 60);
      $minutes = floor($seconds / 60);
      $remaining_seconds = $seconds % 60;
      
      $display_time = match($format) {
        'minutes' => max(1, $minutes) . ' นาที',
        'minutes_seconds' => $minutes . ' นาที ' . $remaining_seconds . ' วินาที',
        'verbose' => 'ใช้เวลาอ่านประมาณ ' . max(1, $minutes) . ' นาที',
      };
      
      $elements[$delta] = [
        '#markup' => ($show_icon ? '⏱ ' : '') . $display_time,
        '#cache' => ['max-age' => \Drupal\Core\Cache\Cache::PERMANENT],
      ];
    }
    
    return $elements;
  }
}
```

### 2.3 Field Widget Plugin

```php
<?php
// src/Plugin/Field/FieldWidget/StarRatingWidget.php

namespace Drupal\mymodule\Plugin\Field\FieldWidget;

use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\WidgetBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Plugin implementation of the 'star_rating' widget.
 *
 * @FieldWidget(
 *   id = "star_rating_widget",
 *   label = @Translation("Star Rating"),
 *   field_types = {
 *     "integer",
 *     "decimal",
 *   }
 * )
 */
class StarRatingWidget extends WidgetBase {

  /**
   * {@inheritdoc}
   */
  public function formElement(
    FieldItemListInterface $items,
    $delta,
    array $element,
    array &$form,
    FormStateInterface $form_state
  ) {
    $value = $items[$delta]->value ?? 0;
    
    $element['value'] = $element + [
      '#type' => 'radios',
      '#options' => [
        1 => '⭐',
        2 => '⭐⭐',
        3 => '⭐⭐⭐',
        4 => '⭐⭐⭐⭐',
        5 => '⭐⭐⭐⭐⭐',
      ],
      '#default_value' => $value,
      '#attributes' => ['class' => ['star-rating-widget']],
    ];
    
    return $element;
  }
}
```

---

## 3. Entity API

### 3.1 โหลด Entities

```php
<?php

use Drupal\node\Entity\Node;
use Drupal\user\Entity\User;
use Drupal\taxonomy\Entity\Term;

// โหลด node เดี่ยว
$node = Node::load(1);
// หรือ
$node = \Drupal::entityTypeManager()->getStorage('node')->load(1);

// โหลดหลาย nodes
$nodes = Node::loadMultiple([1, 2, 3]);

// โหลด user
$user = User::load(\Drupal::currentUser()->id());

// โหลด taxonomy term
$term = Term::load(5);
```

### 3.2 อ่านค่า Fields

```php
// ดึงค่า text field
$title = $node->getTitle();
$body = $node->get('body')->value;
$summary = $node->get('body')->summary;
$format = $node->get('body')->format;

// ดึงค่า integer/decimal field
$price = $node->get('field_price')->value;

// ดึงค่า boolean field
$is_featured = (bool) $node->get('field_is_featured')->value;

// ดึงค่า image field
$image_field = $node->get('field_featured_image');
if (!$image_field->isEmpty()) {
  $file = $image_field->entity;  // File entity
  $uri = $file->getFileUri();     // 'public://image.jpg'
  $url = \Drupal::service('file_url_generator')->generateAbsoluteString($uri);
  $alt = $image_field->alt;
  $title = $image_field->title;
  $width = $image_field->width;
  $height = $image_field->height;
}

// ดึงค่า entity reference field
$author = $node->get('uid')->entity;  // User entity
$author_name = $author->getDisplayName();

// ดึงค่า taxonomy term reference
$categories = $node->get('field_category');
foreach ($categories as $category_ref) {
  $term = $category_ref->entity;
  echo $term->getName();
}

// ดึง multiple values (unlimited cardinality)
$tags = $node->get('field_tags');
$tag_ids = array_column($tags->getValue(), 'target_id');
$tag_terms = Term::loadMultiple($tag_ids);

// ดึงค่า link field
$link = $node->get('field_website');
$url = $link->uri;
$link_title = $link->title;
```

### 3.3 แก้ไขและบันทึก Entity

```php
// แก้ไข field values
$node->set('title', 'New Title');
$node->set('body', [
  'value' => '<p>New body content</p>',
  'format' => 'full_html',
]);
$node->set('field_price', 299.99);

// เพิ่ม taxonomy term
$term = Term::load(5);
$node->get('field_tags')->appendItem(['target_id' => $term->id()]);

// บันทึก node (auto creates new revision ถ้าตั้งค่าไว้)
$node->save();

// สร้าง node ใหม่
$new_node = Node::create([
  'type' => 'article',
  'title' => 'New Article',
  'body' => [
    'value' => '<p>Content</p>',
    'format' => 'full_html',
  ],
  'uid' => \Drupal::currentUser()->id(),
  'status' => 1,
  'field_category' => ['target_id' => 5],
]);
$new_node->save();

// ลบ node
$node->delete();
```

### 3.4 Entity Queries

```php
// Basic query
$query = \Drupal::entityQuery('node');
$nids = $query
  ->condition('type', 'article')
  ->condition('status', 1)
  ->accessCheck(TRUE)
  ->sort('created', 'DESC')
  ->range(0, 10)
  ->execute();

// Query with multiple conditions (AND)
$nids = \Drupal::entityQuery('node')
  ->condition('type', 'article')
  ->condition('status', 1)
  ->condition('field_category.entity.name', 'Technology')
  ->condition('created', strtotime('-30 days'), '>')
  ->accessCheck(TRUE)
  ->count()
  ->execute();  // returns count

// Query with OR conditions
$nids = \Drupal::entityQuery('node')
  ->condition('type', 'article')
  ->condition(
    \Drupal::entityQuery('node')
      ->orConditionGroup()
      ->condition('title', '%drupal%', 'LIKE')
      ->condition('title', '%php%', 'LIKE')
  )
  ->accessCheck(TRUE)
  ->execute();

// Query taxonomy terms
$tids = \Drupal::entityQuery('taxonomy_term')
  ->condition('vid', 'news_category')
  ->condition('status', 1)
  ->sort('weight')
  ->accessCheck(FALSE)
  ->execute();

// Complex query: articles ที่มี specific tag
$nids = \Drupal::entityQuery('node')
  ->condition('type', 'article')
  ->condition('field_tags.entity.name', 'PHP')
  ->accessCheck(TRUE)
  ->execute();

$nodes = \Drupal\node\Entity\Node::loadMultiple($nids);
```

### 3.5 Direct Database Queries (เมื่อ Entity Query ไม่เพียงพอ)

```php
// ใช้ database layer โดยตรง
$database = \Drupal::database();

// Simple SELECT
$result = $database->query(
  "SELECT nid, title FROM {node_field_data} WHERE type = :type AND status = 1",
  [':type' => 'article']
)->fetchAll();

// Query builder
$query = $database->select('node_field_data', 'n');
$query->join('node__field_category', 'c', 'n.nid = c.entity_id');
$query->join('taxonomy_term_field_data', 't', 'c.field_category_target_id = t.tid');
$query->fields('n', ['nid', 'title', 'created']);
$query->fields('t', ['name']);
$query->condition('n.type', 'article');
$query->condition('n.status', 1);
$query->orderBy('n.created', 'DESC');
$query->range(0, 10);

$results = $query->execute()->fetchAll();

// INSERT
$database->insert('related_posts_manual')
  ->fields([
    'source_nid' => 1,
    'related_nid' => 5,
    'weight' => 0,
  ])
  ->execute();

// UPDATE
$database->update('related_posts_manual')
  ->fields(['weight' => 10])
  ->condition('source_nid', 1)
  ->condition('related_nid', 5)
  ->execute();

// DELETE
$database->delete('related_posts_manual')
  ->condition('source_nid', 1)
  ->execute();

// Transaction
$transaction = $database->startTransaction();
try {
  $database->insert('related_posts_manual')->fields([...])->execute();
  $database->update('node_field_data')->fields([...])->execute();
  // หากสำเร็จ transaction จะ commit อัตโนมัติ
} catch (\Exception $e) {
  $transaction->rollBack();
  throw $e;
}
```

---

## 4. Config API

### 4.1 อ่าน Config

```php
// อ่าน config object (immutable)
$config = \Drupal::config('system.site');
$site_name = $config->get('name');
$site_mail = $config->get('mail');
$slogan = $config->get('slogan');

// อ่าน nested values
$front_page = \Drupal::config('system.site')->get('page.front');

// อ่าน config ใน class ที่ใช้ DI
use Drupal\Core\Config\ConfigFactoryInterface;

class MyService {
  public function __construct(private ConfigFactoryInterface $configFactory) {}
  
  public function getSettings(): array {
    $config = $this->configFactory->get('mymodule.settings');
    return [
      'limit' => $config->get('limit') ?? 10,
      'enabled' => $config->get('enabled') ?? FALSE,
    ];
  }
}
```

### 4.2 เขียน Config (Mutable)

```php
// ดึง mutable config
$config = \Drupal::service('config.factory')->getEditable('mymodule.settings');

// ตั้งค่า
$config->set('limit', 20);
$config->set('enabled', TRUE);
$config->set('nested.key', 'value');

// บันทึก
$config->save();

// หรือแบบ chain
\Drupal::service('config.factory')
  ->getEditable('mymodule.settings')
  ->set('key', 'value')
  ->save();
```

### 4.3 State API (Runtime state ไม่ sync กับ config management)

```php
// บันทึก state
\Drupal::state()->set('mymodule.last_run', time());
\Drupal::state()->set('mymodule.counter', 42);

// อ่าน state
$last_run = \Drupal::state()->get('mymodule.last_run', 0);

// ลบ state
\Drupal::state()->delete('mymodule.last_run');

// Multiple values
\Drupal::state()->setMultiple([
  'mymodule.key1' => 'value1',
  'mymodule.key2' => 'value2',
]);
$values = \Drupal::state()->getMultiple(['mymodule.key1', 'mymodule.key2']);
```

### 4.4 Default Config Files

```yaml
# config/install/mymodule.settings.yml
# ไฟล์นี้จะถูก import เมื่อ module ถูก install ครั้งแรก

limit: 10
enabled: false
title: 'Default Title'
allowed_content_types:
  - article
  - page
display:
  show_image: true
  show_date: true
  view_mode: teaser
```

---

## Workshop: Custom Block Plugin + Custom Field Formatter

### เป้าหมาย
สร้าง:
1. **ReadingStatsBlock** - Block ที่แสดง reading statistics ของ current node
2. **EstimatedReadTimeFormatter** - Field formatter สำหรับ body field แสดงเวลาอ่าน

### ขั้นตอนที่ 1: สร้าง Module Structure

```bash
mkdir -p web/modules/custom/reading_stats/src/Plugin/{Block,Field/FieldFormatter}
touch web/modules/custom/reading_stats/reading_stats.info.yml
touch web/modules/custom/reading_stats/reading_stats.services.yml
```

### ขั้นตอนที่ 2: reading_stats.info.yml

```yaml
name: 'Reading Stats'
type: module
description: 'Provides reading statistics for articles'
package: Custom
core_version_requirement: ^10
dependencies:
  - drupal:node
```

### ขั้นตอนที่ 3: reading_stats.services.yml

```yaml
services:
  reading_stats.calculator:
    class: Drupal\reading_stats\ReadingCalculator
    arguments:
      - '@config.factory'
```

### ขั้นตอนที่ 4: ReadingCalculator Service

```php
<?php
// src/ReadingCalculator.php

namespace Drupal\reading_stats;

use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\node\NodeInterface;

class ReadingCalculator {

  protected $wordsPerMinute;

  public function __construct(ConfigFactoryInterface $config_factory) {
    $config = $config_factory->get('reading_stats.settings');
    $this->wordsPerMinute = $config->get('words_per_minute') ?? 200;
  }

  public function calculate(NodeInterface $node): array {
    $body = '';
    if ($node->hasField('body') && !$node->get('body')->isEmpty()) {
      $body = $node->get('body')->value;
    }
    
    $clean_text = strip_tags($body);
    $word_count = str_word_count($clean_text);
    $char_count = mb_strlen($clean_text);
    $reading_seconds = round($word_count / $this->wordsPerMinute * 60);
    $reading_minutes = max(1, ceil($reading_seconds / 60));
    
    // นับรูปภาพ
    preg_match_all('/<img/i', $body, $img_matches);
    $image_count = count($img_matches[0]);
    
    // นับ paragraphs
    preg_match_all('/<p[^>]*>/i', $body, $p_matches);
    $paragraph_count = max(1, count($p_matches[0]));
    
    return [
      'word_count' => $word_count,
      'char_count' => $char_count,
      'reading_minutes' => $reading_minutes,
      'reading_seconds' => $reading_seconds,
      'image_count' => $image_count,
      'paragraph_count' => $paragraph_count,
    ];
  }
}
```

### ขั้นตอนที่ 5: ReadingStatsBlock

```php
<?php
// src/Plugin/Block/ReadingStatsBlock.php

namespace Drupal\reading_stats\Plugin\Block;

use Drupal\Core\Block\BlockBase;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Drupal\reading_stats\ReadingCalculator;
use Drupal\Core\Routing\RouteMatchInterface;

/**
 * @Block(
 *   id = "reading_stats_block",
 *   admin_label = @Translation("Reading Statistics"),
 *   category = @Translation("Content")
 * )
 */
class ReadingStatsBlock extends BlockBase implements ContainerFactoryPluginInterface {

  protected $calculator;
  protected $routeMatch;

  public function __construct(
    array $configuration, $plugin_id, $plugin_definition,
    ReadingCalculator $calculator,
    RouteMatchInterface $route_match
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
    $this->calculator = $calculator;
    $this->routeMatch = $route_match;
  }

  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition) {
    return new static(
      $configuration, $plugin_id, $plugin_definition,
      $container->get('reading_stats.calculator'),
      $container->get('current_route_match')
    );
  }

  public function build() {
    $node = $this->routeMatch->getParameter('node');
    if (!$node || $node->bundle() !== 'article') {
      return [];
    }

    $stats = $this->calculator->calculate($node);

    return [
      '#theme' => 'reading_stats_block',
      '#stats' => $stats,
      '#cache' => [
        'tags' => ['node:' . $node->id()],
        'contexts' => ['route'],
      ],
    ];
  }
}
```

### ขั้นตอนที่ 6: Enable และทดสอบ

```bash
drush en reading_stats -y
drush cr

# เพิ่ม block ใน Block Layout
# Admin > Structure > Block layout > Add block > Reading Statistics
```

---

## Quiz

**ข้อ 1:** ความแตกต่างระหว่าง `\Drupal::config()` และ `\Drupal::service('config.factory')->getEditable()` คืออะไร?
- a) ทำงานเหมือนกันทุกอย่าง
- b) `\Drupal::config()` return immutable object (อ่านอย่างเดียว), `getEditable()` return mutable object (แก้ไขได้)
- c) `\Drupal::config()` ช้ากว่า
- d) `getEditable()` ใช้ได้เฉพาะในการ install module

**เฉลย:** b) `\Drupal::config()` = อ่านอย่างเดียว, `getEditable()` = อ่านและเขียนได้

---

**ข้อ 2:** `ContainerFactoryPluginInterface` ใน Block plugin ใช้ทำอะไร?
- a) ทำให้ block สามารถ export เป็น config ได้
- b) ช่วยให้ plugin สามารถรับ dependencies ผ่าน service container (Dependency Injection)
- c) ทำให้ block ทำงานได้เร็วขึ้น
- d) ป้องกัน circular dependencies

**เฉลย:** b) ContainerFactoryPluginInterface ช่วยให้ plugin รับ services จาก DI container ผ่าน static `create()` method

---

**ข้อ 3:** Entity Query ต่างจาก Direct SQL Query อย่างไร?
- a) Entity Query ช้ากว่าเสมอ
- b) Entity Query จัดการ access control และ entity cache โดยอัตโนมัติ Direct SQL ข้ามสิ่งเหล่านี้
- c) Direct SQL ใช้ได้เฉพาะ nodes เท่านั้น
- d) Entity Query ใช้ได้กับ taxonomy terms เท่านั้น

**เฉลย:** b) Entity Query มี access checking, translation support, และ cache invalidation built-in

---

**ข้อ 4:** Plugin Annotation ใน Drupal คืออะไร?
- a) Comment ธรรมดาใน PHP
- b) PHP docblock ที่ Drupal อ่านเพื่อ register plugin metadata เช่น `@Block(id="...", label=...)`
- c) Configuration file แยกต่างหาก
- d) Database record ที่เก็บ plugin info

**เฉลย:** b) Annotation เป็น docblock comments ที่ Drupal parse เพื่อ discover และ register plugins

---

**ข้อ 5:** State API ต่างจาก Config API อย่างไร?
- a) State API เร็วกว่า Config API
- b) State API สำหรับ runtime state ที่ไม่ควร sync ระหว่าง environments (เช่น cron timestamps), Config API สำหรับ configuration ที่ sync ได้
- c) Config API ใช้ database, State API ใช้ file system
- d) ทั้งสองอย่างเหมือนกัน

**เฉลย:** b) State = per-environment runtime data (ไม่ export/import), Config = exportable configuration (sync ระหว่าง environments ได้)

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- Services และ Dependency Injection ด้วย services.yml
- Block Plugin และ Field Formatter Plugin
- Entity API สำหรับโหลด, อ่าน, แก้ไข, บันทึก entities
- Entity Queries สำหรับ complex database queries
- Config API และ State API สำหรับจัดเก็บข้อมูล configuration

**Part ถัดไป:** Part 097 - PHP Internals & Performance
