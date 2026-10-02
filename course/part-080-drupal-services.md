# Part 080: Drupal Services, Plugin System & Entity API

**ระดับ:** สูง (Advanced)  
**เวลาเรียน:** 6-8 ชั่วโมง  
**Prerequisites:** Part 079 - Drupal Module Basics

---

## เป้าหมายของ Part นี้

1. เข้าใจ Services และ Dependency Injection ใน Drupal
2. สร้าง Custom Service Class
3. ใช้งาน Plugin System (Block Plugins, Field Formatter)
4. ทำงานกับ Entity API อย่างครบถ้วน
5. จัดการ Config ผ่าน Config API
6. Workshop: Custom Block + Thai Currency Field Formatter

---

## 1. Services & Dependency Injection

### 1.1 แนวคิด Services Container

Drupal ใช้ **Symfony's DependencyInjection Component** ซึ่งเป็น Service Container ที่จัดการ object creation และ dependencies อัตโนมัติ

ทำไมต้องใช้ DI?

```php
// ❌ วิธีเก่า - สร้าง dependency เองใน class
class MyService {
  public function doSomething(): void {
    $database = new Database(); // tight coupling
    $logger   = new Logger();   // ทดสอบยาก
    // ...
  }
}

// ✅ วิธีใหม่ - Inject dependencies จาก container
class MyService {
  public function __construct(
    private \Drupal\Core\Database\Connection $database,
    private \Psr\Log\LoggerInterface $logger,
  ) {}
}
```

**ข้อดีของ DI:**
- ทดสอบง่าย (mock dependencies ได้)
- Code ยืดหยุ่น และ maintainable
- Single Responsibility Principle

### 1.2 ไฟล์ mymodule.services.yml

```yaml
# web/modules/custom/mymodule/mymodule.services.yml

services:
  # Service พื้นฐาน
  mymodule.helper:
    class: Drupal\mymodule\Service\MyModuleHelper
    arguments:
      - '@entity_type.manager'
      - '@config.factory'
      - '@logger.factory'
      - '@current_user'

  # Service ที่ไม่มี dependencies
  mymodule.calculator:
    class: Drupal\mymodule\Service\Calculator

  # Service ที่ใช้ parent
  mymodule.cache_helper:
    class: Drupal\mymodule\Service\CacheHelper
    arguments:
      - '@cache.default'
      - '@cache.render'

  # Tagged service
  mymodule.content_processor:
    class: Drupal\mymodule\Service\ContentProcessor
    arguments:
      - '@entity_type.manager'
      - '@database'
    tags:
      - { name: event_subscriber }

  # Alias
  mymodule:
    alias: mymodule.helper
```

### 1.3 สร้าง Service Class

```php
<?php
// web/modules/custom/mymodule/src/Service/MyModuleHelper.php

namespace Drupal\mymodule\Service;

use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Logger\LoggerChannelFactoryInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\node\NodeInterface;

/**
 * Helper service for My Module.
 */
class MyModuleHelper {

  /**
   * Constructor with Dependency Injection.
   */
  public function __construct(
    protected EntityTypeManagerInterface $entityTypeManager,
    protected ConfigFactoryInterface $configFactory,
    protected LoggerChannelFactoryInterface $loggerFactory,
    protected AccountInterface $currentUser,
  ) {}

  /**
   * Get published nodes by type.
   *
   * @param string $type Content type machine name.
   * @param int    $limit Maximum number of nodes.
   *
   * @return \Drupal\node\NodeInterface[]
   */
  public function getPublishedNodes(string $type, int $limit = 10): array {
    $storage = $this->entityTypeManager->getStorage('node');

    $nids = $storage->getQuery()
      ->condition('status', NodeInterface::PUBLISHED)
      ->condition('type', $type)
      ->sort('created', 'DESC')
      ->range(0, $limit)
      ->accessCheck(TRUE)
      ->execute();

    if (empty($nids)) {
      return [];
    }

    return $storage->loadMultiple($nids);
  }

  /**
   * Get module configuration.
   */
  public function getConfig(string $key): mixed {
    return $this->configFactory
      ->get('mymodule.settings')
      ->get($key);
  }

  /**
   * Log module activity.
   */
  public function log(string $message, array $context = [], string $level = 'info'): void {
    $logger = $this->loggerFactory->get('mymodule');
    $logger->{$level}($message, $context);
  }

  /**
   * Check if current user can access feature.
   */
  public function canAccessFeature(string $permission): bool {
    return $this->currentUser->hasPermission($permission);
  }

  /**
   * Format nodes to array for API response.
   */
  public function formatNodesForApi(array $nodes): array {
    $result = [];

    foreach ($nodes as $node) {
      if (!$node instanceof NodeInterface) {
        continue;
      }

      $item = [
        'id'      => $node->id(),
        'title'   => $node->label(),
        'type'    => $node->bundle(),
        'url'     => $node->toUrl('canonical')->toString(),
        'created' => $node->getCreatedTime(),
        'author'  => [
          'uid'  => $node->getOwnerId(),
          'name' => $node->getOwner()->getDisplayName(),
        ],
      ];

      // เพิ่ม field values
      if ($node->hasField('body') && !$node->get('body')->isEmpty()) {
        $item['summary'] = $node->get('body')->summary
          ?? mb_substr(strip_tags($node->get('body')->value), 0, 200);
      }

      if ($node->hasField('field_image') && !$node->get('field_image')->isEmpty()) {
        $file = $node->get('field_image')->entity;
        if ($file) {
          $item['image'] = \Drupal::service('file_url_generator')
            ->generateAbsoluteString($file->getFileUri());
        }
      }

      $result[] = $item;
    }

    return $result;
  }

}
```

### 1.4 ใช้ Service ใน Controller

```php
<?php
// web/modules/custom/mymodule/src/Controller/ApiController.php

namespace Drupal\mymodule\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\mymodule\Service\MyModuleHelper;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Symfony\Component\HttpFoundation\JsonResponse;

/**
 * API Controller using DI.
 */
class ApiController extends ControllerBase {

  /**
   * Constructor.
   */
  public function __construct(
    protected MyModuleHelper $helper,
  ) {}

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('mymodule.helper')
    );
  }

  /**
   * Return recent articles as JSON.
   */
  public function recentArticles(): JsonResponse {
    if (!$this->helper->canAccessFeature('access content')) {
      return new JsonResponse(['error' => 'Access denied'], 403);
    }

    $nodes = $this->helper->getPublishedNodes('article', 10);
    $data  = $this->helper->formatNodesForApi($nodes);

    return new JsonResponse([
      'status' => 'ok',
      'count'  => count($data),
      'items'  => $data,
    ]);
  }

}
```

### 1.5 ใช้ Service แบบ Static (ไม่แนะนำ แต่ใช้ได้ในบางกรณี)

```php
// ใช้ใน .module file หรือ static context
$helper = \Drupal::service('mymodule.helper');
$nodes  = $helper->getPublishedNodes('article');

// Services ที่มีใน Drupal core
\Drupal::entityTypeManager()      // entity_type.manager
\Drupal::database()               // database
\Drupal::config('name')           // config.factory
\Drupal::state()                  // state
\Drupal::currentUser()            // current_user
\Drupal::request()                // request_stack
\Drupal::cache()                  // cache.default
\Drupal::logger('channel')        // logger.factory
\Drupal::moduleHandler()          // module_handler
\Drupal::languageManager()        // language_manager
\Drupal::token()                  // token
\Drupal::messenger()              // messenger
```

---

## 2. Plugin System

Plugin System คือระบบที่ให้ Drupal discover และใช้งาน plugins แบบ dynamic โดยไม่ต้องแก้ code เดิม

### 2.1 ประเภท Plugins ที่สำคัญ

| Plugin Type | ใช้สำหรับ | Base Class |
|-------------|-----------|------------|
| Block | Custom Blocks | `BlockBase` |
| Field Formatter | แสดงผล Field | `FormatterBase` |
| Field Widget | Form input สำหรับ Field | `WidgetBase` |
| Action | Batch actions | `ActionBase` |
| Condition | Access/Visibility conditions | `ConditionPluginBase` |
| QueueWorker | Background jobs | `QueueWorkerBase` |

### 2.2 Block Plugin

```php
<?php
// web/modules/custom/mymodule/src/Plugin/Block/RecentPostsBlock.php

namespace Drupal\mymodule\Plugin\Block;

use Drupal\Core\Block\BlockBase;
use Drupal\Core\Block\BlockPluginInterface;
use Drupal\Core\Cache\Cache;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\mymodule\Service\MyModuleHelper;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Provides a 'Recent Posts' block.
 *
 * @Block(
 *   id = "mymodule_recent_posts",
 *   admin_label = @Translation("Recent Posts"),
 *   category = @Translation("Custom"),
 * )
 */
class RecentPostsBlock extends BlockBase implements ContainerFactoryPluginInterface {

  /**
   * The helper service.
   */
  protected MyModuleHelper $helper;

  /**
   * Constructor.
   */
  public function __construct(
    array $configuration,
    string $plugin_id,
    mixed $plugin_definition,
    MyModuleHelper $helper,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
    $this->helper = $helper;
  }

  /**
   * {@inheritdoc}
   */
  public static function create(
    ContainerInterface $container,
    array $configuration,
    string $plugin_id,
    mixed $plugin_definition,
  ): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('mymodule.helper')
    );
  }

  /**
   * Block configuration form.
   */
  public function blockForm(array $form, FormStateInterface $form_state): array {
    $form = parent::blockForm($form, $form_state);

    $config = $this->getConfiguration();

    $form['count'] = [
      '#type'          => 'number',
      '#title'         => $this->t('Number of posts'),
      '#default_value' => $config['count'] ?? 5,
      '#min'           => 1,
      '#max'           => 20,
    ];

    $form['content_type'] = [
      '#type'          => 'select',
      '#title'         => $this->t('Content Type'),
      '#options'       => $this->getContentTypeOptions(),
      '#default_value' => $config['content_type'] ?? 'article',
    ];

    $form['show_image'] = [
      '#type'          => 'checkbox',
      '#title'         => $this->t('Show featured image'),
      '#default_value' => $config['show_image'] ?? TRUE,
    ];

    return $form;
  }

  /**
   * Save block configuration.
   */
  public function blockSubmit(array $form, FormStateInterface $form_state): void {
    parent::blockSubmit($form, $form_state);

    $this->configuration['count']        = $form_state->getValue('count');
    $this->configuration['content_type'] = $form_state->getValue('content_type');
    $this->configuration['show_image']   = $form_state->getValue('show_image');
  }

  /**
   * Build the block.
   */
  public function build(): array {
    $config       = $this->getConfiguration();
    $count        = $config['count'] ?? 5;
    $content_type = $config['content_type'] ?? 'article';
    $show_image   = $config['show_image'] ?? TRUE;

    $nodes = $this->helper->getPublishedNodes($content_type, $count);

    if (empty($nodes)) {
      return [
        '#markup' => $this->t('No posts found.'),
      ];
    }

    $items = [];
    foreach ($nodes as $node) {
      $item = [
        'title' => $node->label(),
        'url'   => $node->toUrl('canonical'),
        'date'  => \Drupal::service('date.formatter')
          ->format($node->getCreatedTime(), 'medium'),
      ];

      if ($show_image && $node->hasField('field_image') && !$node->get('field_image')->isEmpty()) {
        $item['image'] = $node->get('field_image')->entity;
      }

      $items[] = $item;
    }

    return [
      '#theme'     => 'mymodule_recent_posts_block',
      '#items'     => $items,
      '#cache'     => [
        'tags'    => ['node_list'],
        'contexts' => ['languages'],
        'max-age'  => 3600,
      ],
    ];
  }

  /**
   * {@inheritdoc}
   */
  public function getCacheTags(): array {
    return Cache::mergeTags(parent::getCacheTags(), ['node_list']);
  }

  /**
   * Get content type select options.
   */
  private function getContentTypeOptions(): array {
    $options = [];
    $types   = \Drupal::entityTypeManager()->getStorage('node_type')->loadMultiple();
    foreach ($types as $type) {
      $options[$type->id()] = $type->label();
    }
    return $options;
  }

}
```

### 2.3 Field Formatter Plugin

```php
<?php
// web/modules/custom/mymodule/src/Plugin/Field/FieldFormatter/ThaiCurrencyFormatter.php

namespace Drupal\mymodule\Plugin\Field\FieldFormatter;

use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\FormatterBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Plugin implementation of the Thai Currency field formatter.
 *
 * @FieldFormatter(
 *   id = "mymodule_thai_currency",
 *   label = @Translation("Thai Currency (฿)"),
 *   field_types = {
 *     "decimal",
 *     "float",
 *     "integer",
 *   }
 * )
 */
class ThaiCurrencyFormatter extends FormatterBase {

  /**
   * {@inheritdoc}
   */
  public static function defaultSettings(): array {
    return [
      'prefix'          => '฿',
      'suffix'          => '',
      'decimal_places'  => 2,
      'thousand_sep'    => ',',
      'show_symbol'     => TRUE,
      'color_negative'  => TRUE,
    ] + parent::defaultSettings();
  }

  /**
   * Settings form.
   */
  public function settingsForm(array $form, FormStateInterface $form_state): array {
    $elements = parent::settingsForm($form, $form_state);

    $elements['prefix'] = [
      '#type'          => 'textfield',
      '#title'         => $this->t('Prefix'),
      '#default_value' => $this->getSetting('prefix'),
      '#size'          => 10,
    ];

    $elements['suffix'] = [
      '#type'          => 'textfield',
      '#title'         => $this->t('Suffix (optional)'),
      '#default_value' => $this->getSetting('suffix'),
      '#size'          => 10,
    ];

    $elements['decimal_places'] = [
      '#type'          => 'number',
      '#title'         => $this->t('Decimal Places'),
      '#default_value' => $this->getSetting('decimal_places'),
      '#min'           => 0,
      '#max'           => 4,
    ];

    $elements['thousand_sep'] = [
      '#type'          => 'select',
      '#title'         => $this->t('Thousands Separator'),
      '#options'       => [
        ','  => $this->t('Comma (1,000)'),
        '.'  => $this->t('Dot (1.000)'),
        ' '  => $this->t('Space (1 000)'),
        ''   => $this->t('None (1000)'),
      ],
      '#default_value' => $this->getSetting('thousand_sep'),
    ];

    $elements['color_negative'] = [
      '#type'          => 'checkbox',
      '#title'         => $this->t('Highlight negative values in red'),
      '#default_value' => $this->getSetting('color_negative'),
    ];

    return $elements;
  }

  /**
   * Settings summary.
   */
  public function settingsSummary(): array {
    $summary = [];
    $example = $this->formatCurrency(1234567.89);
    $summary[] = $this->t('Example: @example', ['@example' => $example]);
    return $summary;
  }

  /**
   * {@inheritdoc}
   */
  public function viewElements(FieldItemListInterface $items, string $langcode): array {
    $elements = [];

    foreach ($items as $delta => $item) {
      $value    = (float) $item->value;
      $formatted = $this->formatCurrency($value);
      $negative  = $value < 0;

      $elements[$delta] = [
        '#markup' => $this->buildMarkup($formatted, $negative),
      ];
    }

    return $elements;
  }

  /**
   * Format a number as Thai currency.
   */
  protected function formatCurrency(float $value): string {
    $prefix        = $this->getSetting('prefix');
    $suffix        = $this->getSetting('suffix');
    $decimals      = (int) $this->getSetting('decimal_places');
    $thousands_sep = $this->getSetting('thousand_sep');
    $dec_point     = $thousands_sep === '.' ? ',' : '.';

    $formatted = number_format(abs($value), $decimals, $dec_point, $thousands_sep);
    $sign      = $value < 0 ? '-' : '';

    return $sign . $prefix . $formatted . $suffix;
  }

  /**
   * Build HTML markup for the formatted value.
   */
  protected function buildMarkup(string $formatted, bool $negative): string {
    $color_negative = $this->getSetting('color_negative');

    if ($negative && $color_negative) {
      return '<span class="currency-negative" style="color: #e74c3c;">'
        . htmlspecialchars($formatted, ENT_QUOTES, 'UTF-8')
        . '</span>';
    }

    return '<span class="currency-value">'
      . htmlspecialchars($formatted, ENT_QUOTES, 'UTF-8')
      . '</span>';
  }

}
```

---

## 3. Entity API

### 3.1 entityTypeManager

`entityTypeManager` เป็น Service หลักในการทำงานกับ Entities:

```php
// ใน Controller หรือ Service
$entity_type_manager = $this->entityTypeManager();
// หรือ inject ผ่าน constructor:
// $container->get('entity_type.manager')

// ดึง Storage handler
$node_storage = $entity_type_manager->getStorage('node');
$user_storage = $entity_type_manager->getStorage('user');
$term_storage = $entity_type_manager->getStorage('taxonomy_term');
$file_storage = $entity_type_manager->getStorage('file');

// ดึง View Builder
$view_builder = $entity_type_manager->getViewBuilder('node');
$rendered = $view_builder->view($node, 'teaser');

// ดึง Entity Definition
$definition = $entity_type_manager->getDefinition('node');
```

### 3.2 Entity Query

```php
// entityQuery - ค้นหา entity IDs

// แบบ basic
$query = $entity_type_manager->getStorage('node')->getQuery();
$nids  = $query
  ->condition('status', 1)
  ->condition('type', 'article')
  ->sort('created', 'DESC')
  ->range(0, 10)
  ->accessCheck(TRUE)
  ->execute();

// ค้นหาด้วย field
$nids = $entity_type_manager->getStorage('node')->getQuery()
  ->condition('status', 1)
  ->condition('type', 'article')
  ->condition('field_category', 'technology')
  ->condition('field_tags.entity.name', 'PHP')
  ->sort('title', 'ASC')
  ->range(0, 20)
  ->accessCheck(TRUE)
  ->execute();

// ค้นหาแบบ OR
$query   = $entity_type_manager->getStorage('node')->getQuery();
$or_group = $query->orConditionGroup()
  ->condition('title', '%laravel%', 'LIKE')
  ->condition('body', '%laravel%', 'LIKE');

$nids = $query
  ->condition('status', 1)
  ->condition($or_group)
  ->sort('created', 'DESC')
  ->range(0, 10)
  ->accessCheck(TRUE)
  ->execute();

// นับ entities
$count = $entity_type_manager->getStorage('node')->getQuery()
  ->condition('status', 1)
  ->condition('type', 'article')
  ->count()
  ->execute();

// ค้นหา user
$uids = $entity_type_manager->getStorage('user')->getQuery()
  ->condition('status', 1)
  ->condition('roles', 'editor', 'IN')
  ->sort('created', 'DESC')
  ->execute();
```

### 3.3 Load และ loadMultiple

```php
// โหลด node เดียว
$node = $entity_type_manager->getStorage('node')->load(42);
if ($node instanceof \Drupal\node\NodeInterface) {
  echo $node->label();       // Title
  echo $node->id();          // Node ID
  echo $node->bundle();      // Content type
  echo $node->getCreatedTime(); // Timestamp
  echo $node->getOwnerId();  // UID
}

// โหลดหลาย nodes
$nodes = $entity_type_manager->getStorage('node')->loadMultiple([1, 2, 3, 4]);
foreach ($nodes as $nid => $node) {
  echo $node->label();
}

// โหลด taxonomy terms
$term = $entity_type_manager->getStorage('taxonomy_term')->load(5);
echo $term->label();         // Term name
echo $term->bundle();        // Vocabulary machine name
echo $term->getDescription(); // Description

// โหลด user
$user = $entity_type_manager->getStorage('user')->load(1);
echo $user->getDisplayName();
echo $user->getEmail();
foreach ($user->getRoles() as $role) {
  echo $role;
}
```

### 3.4 อ่านและเขียน Field Values

```php
// อ่าน field values
$node = $entity_type_manager->getStorage('node')->load(42);

// Text fields
$title   = $node->get('title')->value;
$body    = $node->get('body')->value;
$summary = $node->get('body')->summary;
$format  = $node->get('body')->format;

// Number fields
$price   = $node->get('field_price')->value;

// Date fields
$date    = $node->get('field_date')->value; // ISO format string

// Boolean
$featured = (bool) $node->get('field_featured')->value;

// Entity Reference (single)
$category_id = $node->get('field_category')->target_id;
$category    = $node->get('field_category')->entity; // หรือ load โดยตรง

// Entity Reference (multiple)
foreach ($node->get('field_tags') as $tag_ref) {
  $tid  = $tag_ref->target_id;
  $term = $tag_ref->entity;
  echo $term->label();
}

// Image field
$image_field = $node->get('field_image');
if (!$image_field->isEmpty()) {
  $file     = $image_field->entity;
  $alt      = $image_field->alt;
  $title    = $image_field->title;
  $uri      = $file->getFileUri(); // e.g., public://uploads/photo.jpg
  $url      = \Drupal::service('file_url_generator')->generateString($uri);
}

// เขียน field values
$node->set('title', 'New Title');
$node->set('body', [
  'value'   => '<p>Body content</p>',
  'summary' => 'Short summary',
  'format'  => 'basic_html',
]);
$node->set('field_price', 999.99);
$node->set('field_featured', TRUE);

// Set entity reference
$node->set('field_category', 5); // term ID

// Set multiple entity references
$node->set('field_tags', [
  ['target_id' => 1],
  ['target_id' => 2],
  ['target_id' => 3],
]);

// บันทึก
$node->save();
```

### 3.5 สร้างและลบ Entity

```php
// สร้าง node ใหม่
$node = $entity_type_manager->getStorage('node')->create([
  'type'   => 'article',
  'title'  => 'New Article',
  'status' => 1,
  'uid'    => \Drupal::currentUser()->id(),
  'body'   => [
    'value'  => '<p>Article body here.</p>',
    'format' => 'basic_html',
  ],
  'field_tags' => [
    ['target_id' => 1],
  ],
]);
$node->save();
$new_nid = $node->id();

// สร้าง taxonomy term
$term = $entity_type_manager->getStorage('taxonomy_term')->create([
  'name' => 'New Term',
  'vid'  => 'tags', // vocabulary machine name
  'description' => [
    'value'  => 'Term description',
    'format' => 'plain_text',
  ],
]);
$term->save();

// สร้าง user
$user = $entity_type_manager->getStorage('user')->create([
  'name'   => 'newuser',
  'mail'   => 'newuser@example.com',
  'status' => 1,
  'roles'  => ['editor'],
]);
$user->setPassword('SecurePassword123!');
$user->save();

// ลบ entity
$node = $entity_type_manager->getStorage('node')->load(42);
$node->delete();

// ลบหลาย entities
$nodes = $entity_type_manager->getStorage('node')->loadMultiple([1, 2, 3]);
$entity_type_manager->getStorage('node')->delete($nodes);
```

---

## 4. Config API

### 4.1 อ่าน Config

```php
// อ่าน immutable config (read-only)
$config = \Drupal::config('mymodule.settings');
$items_per_page = $config->get('items_per_page'); // ค่าเดียว
$all_settings   = $config->getRawData();           // ทุก keys

// อ่าน nested config
$api_settings = $config->get('api');               // array
$endpoint     = $config->get('api.endpoint');      // nested key

// ผ่าน injection
use Drupal\Core\Config\ConfigFactoryInterface;

class MyService {
  public function __construct(
    protected ConfigFactoryInterface $configFactory,
  ) {}

  public function getSetting(string $key): mixed {
    return $this->configFactory->get('mymodule.settings')->get($key);
  }
}
```

### 4.2 เขียน Config

```php
// ต้องใช้ mutable config (editable)
$config = \Drupal::service('config.factory')
  ->getEditable('mymodule.settings');

$config->set('items_per_page', 20)->save();

// เขียนหลาย values พร้อมกัน
$config
  ->set('items_per_page', 20)
  ->set('cache_lifetime', 7200)
  ->set('enable_feature', TRUE)
  ->set('api.endpoint', 'https://api.example.com')
  ->set('api.key', 'secret-key')
  ->save();

// ลบ key
$config->clear('deprecated_setting')->save();

// ลบ config ทั้งหมด
$config->delete();
```

### 4.3 State API (สำหรับ runtime data)

Config เหมาะสำหรับ settings ที่ export ได้  
State เหมาะสำหรับข้อมูล runtime ที่ไม่ต้อง export

```php
// State API
$state = \Drupal::state();

// บันทึก
$state->set('mymodule.last_import', time());
$state->setMultiple([
  'mymodule.counter'    => 0,
  'mymodule.last_run'   => time(),
]);

// อ่าน
$last_import = $state->get('mymodule.last_import');
$counter     = $state->get('mymodule.counter', 0); // default value

// ลบ
$state->delete('mymodule.counter');
$state->deleteMultiple(['mymodule.counter', 'mymodule.last_run']);
```

---

## 5. Workshop: Custom Block + Thai Currency Formatter

### Workshop A: Recent Posts Block พร้อม Caching

#### 5.1 โครงสร้างไฟล์

```
web/modules/custom/custom_blocks/
├── custom_blocks.info.yml
├── custom_blocks.services.yml
├── custom_blocks.module
└── src/
    ├── Plugin/
    │   └── Block/
    │       └── RecentPostsBlock.php
    └── Service/
        └── PostFetcher.php
```

#### 5.2 custom_blocks.info.yml

```yaml
name: Custom Blocks
type: module
description: 'Provides custom block plugins.'
package: Custom
core_version_requirement: ^10
dependencies:
  - drupal:node
  - drupal:block
```

#### 5.3 PostFetcher Service

```php
<?php
// src/Service/PostFetcher.php

namespace Drupal\custom_blocks\Service;

use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\node\NodeInterface;

/**
 * Service to fetch posts with caching.
 */
class PostFetcher {

  const CACHE_BIN = 'cache.default';
  const CACHE_TTL = 3600; // 1 hour

  public function __construct(
    protected EntityTypeManagerInterface $entityTypeManager,
    protected CacheBackendInterface $cache,
  ) {}

  /**
   * Get recent posts with caching.
   */
  public function getRecentPosts(string $type, int $count): array {
    $cid = "custom_blocks:recent_posts:{$type}:{$count}";

    // ลอง get จาก cache ก่อน
    if ($cached = $this->cache->get($cid)) {
      return $cached->data;
    }

    // ถ้าไม่มี cache ดึงจาก database
    $storage = $this->entityTypeManager->getStorage('node');

    $nids = $storage->getQuery()
      ->condition('status', NodeInterface::PUBLISHED)
      ->condition('type', $type)
      ->sort('created', 'DESC')
      ->range(0, $count)
      ->accessCheck(TRUE)
      ->execute();

    $nodes = $storage->loadMultiple($nids);

    // บันทึกลง cache
    $this->cache->set(
      $cid,
      $nodes,
      time() + static::CACHE_TTL,
      ['node_list', "node_type:{$type}"]
    );

    return $nodes;
  }

  /**
   * Invalidate cache for a node type.
   */
  public function invalidateCache(string $type): void {
    \Drupal::service('cache_tags.invalidator')
      ->invalidateTags(["node_type:{$type}"]);
  }

}
```

#### 5.4 custom_blocks.services.yml

```yaml
services:
  custom_blocks.post_fetcher:
    class: Drupal\custom_blocks\Service\PostFetcher
    arguments:
      - '@entity_type.manager'
      - '@cache.default'
```

#### 5.5 RecentPostsBlock Plugin (สมบูรณ์)

```php
<?php
// src/Plugin/Block/RecentPostsBlock.php

namespace Drupal\custom_blocks\Plugin\Block;

use Drupal\Core\Block\BlockBase;
use Drupal\Core\Cache\Cache;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\custom_blocks\Service\PostFetcher;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Recent Posts Block.
 *
 * @Block(
 *   id = "custom_blocks_recent_posts",
 *   admin_label = @Translation("Recent Posts"),
 *   category = @Translation("Custom Blocks"),
 * )
 */
class RecentPostsBlock extends BlockBase implements ContainerFactoryPluginInterface {

  public function __construct(
    array $configuration,
    string $plugin_id,
    mixed $plugin_definition,
    protected PostFetcher $postFetcher,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
  }

  public static function create(
    ContainerInterface $container,
    array $configuration,
    string $plugin_id,
    mixed $plugin_definition,
  ): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('custom_blocks.post_fetcher')
    );
  }

  public function defaultConfiguration(): array {
    return [
      'count'        => 5,
      'content_type' => 'article',
      'show_date'    => TRUE,
      'show_image'   => TRUE,
    ];
  }

  public function blockForm(array $form, FormStateInterface $form_state): array {
    $form = parent::blockForm($form, $form_state);
    $config = $this->getConfiguration();

    // โหลด node types
    $types = \Drupal::entityTypeManager()->getStorage('node_type')->loadMultiple();
    $type_options = [];
    foreach ($types as $type) {
      $type_options[$type->id()] = $type->label();
    }

    $form['count'] = [
      '#type'          => 'number',
      '#title'         => $this->t('Number of items'),
      '#default_value' => $config['count'],
      '#min'           => 1,
      '#max'           => 20,
    ];

    $form['content_type'] = [
      '#type'          => 'select',
      '#title'         => $this->t('Content Type'),
      '#options'       => $type_options,
      '#default_value' => $config['content_type'],
    ];

    $form['show_date'] = [
      '#type'          => 'checkbox',
      '#title'         => $this->t('Show published date'),
      '#default_value' => $config['show_date'],
    ];

    $form['show_image'] = [
      '#type'          => 'checkbox',
      '#title'         => $this->t('Show featured image'),
      '#default_value' => $config['show_image'],
    ];

    return $form;
  }

  public function blockSubmit(array $form, FormStateInterface $form_state): void {
    parent::blockSubmit($form, $form_state);
    $this->configuration['count']        = $form_state->getValue('count');
    $this->configuration['content_type'] = $form_state->getValue('content_type');
    $this->configuration['show_date']    = $form_state->getValue('show_date');
    $this->configuration['show_image']   = $form_state->getValue('show_image');
  }

  public function build(): array {
    $config       = $this->getConfiguration();
    $count        = (int) ($config['count'] ?? 5);
    $content_type = $config['content_type'] ?? 'article';
    $show_date    = (bool) ($config['show_date'] ?? TRUE);
    $show_image   = (bool) ($config['show_image'] ?? TRUE);

    $nodes = $this->postFetcher->getRecentPosts($content_type, $count);

    if (empty($nodes)) {
      return ['#markup' => $this->t('No posts available.')];
    }

    $date_formatter = \Drupal::service('date.formatter');
    $file_url_gen   = \Drupal::service('file_url_generator');

    $items = [];
    foreach ($nodes as $node) {
      $item = [
        'nid'   => $node->id(),
        'title' => $node->label(),
        'url'   => $node->toUrl('canonical')->toString(),
        'date'  => $show_date
          ? $date_formatter->format($node->getCreatedTime(), 'custom', 'd M Y')
          : NULL,
      ];

      if ($show_image
          && $node->hasField('field_image')
          && !$node->get('field_image')->isEmpty()
          && $file = $node->get('field_image')->entity) {
        $item['image_url'] = $file_url_gen->generateString($file->getFileUri());
        $item['image_alt'] = $node->get('field_image')->alt ?? $node->label();
      }

      $items[] = $item;
    }

    return [
      '#theme'        => 'custom_blocks_recent_posts',
      '#items'        => $items,
      '#show_date'    => $show_date,
      '#show_image'   => $show_image,
      '#attached'     => [
        'library' => ['custom_blocks/recent-posts'],
      ],
    ];
  }

  public function getCacheTags(): array {
    return Cache::mergeTags(parent::getCacheTags(), ['node_list']);
  }

  public function getCacheContexts(): array {
    return Cache::mergeContexts(parent::getCacheContexts(), ['languages']);
  }

}
```

### Workshop B: Thai Currency Field Formatter (สมบูรณ์)

#### 5.6 ThaiCurrencyFormatter พร้อม Options เต็ม

```php
<?php
// src/Plugin/Field/FieldFormatter/ThaiCurrencyFormatter.php

namespace Drupal\custom_blocks\Plugin\Field\FieldFormatter;

use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\FormatterBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Thai Currency Formatter.
 *
 * @FieldFormatter(
 *   id = "thai_currency",
 *   label = @Translation("Thai Currency"),
 *   field_types = {
 *     "decimal",
 *     "float",
 *     "integer",
 *     "list_float",
 *   }
 * )
 */
class ThaiCurrencyFormatter extends FormatterBase {

  const BAHT_SYMBOL = '฿';

  public static function defaultSettings(): array {
    return [
      'currency'        => 'THB',
      'show_symbol'     => TRUE,
      'decimal_places'  => 2,
      'thousand_sep'    => ',',
      'negative_format' => 'minus',  // minus, parentheses, color
      'zero_display'    => 'value',  // value, dash, free
    ] + parent::defaultSettings();
  }

  public function settingsForm(array $form, FormStateInterface $form_state): array {
    $elements = parent::settingsForm($form, $form_state);

    $elements['currency'] = [
      '#type'          => 'select',
      '#title'         => $this->t('Currency'),
      '#options'       => [
        'THB' => $this->t('Thai Baht (฿)'),
        'USD' => $this->t('US Dollar ($)'),
        'EUR' => $this->t('Euro (€)'),
        'JPY' => $this->t('Japanese Yen (¥)'),
      ],
      '#default_value' => $this->getSetting('currency'),
    ];

    $elements['show_symbol'] = [
      '#type'          => 'checkbox',
      '#title'         => $this->t('Show currency symbol'),
      '#default_value' => $this->getSetting('show_symbol'),
    ];

    $elements['decimal_places'] = [
      '#type'          => 'select',
      '#title'         => $this->t('Decimal Places'),
      '#options'       => [0 => '0', 1 => '1', 2 => '2', 3 => '3'],
      '#default_value' => $this->getSetting('decimal_places'),
    ];

    $elements['thousand_sep'] = [
      '#type'          => 'select',
      '#title'         => $this->t('Thousands Separator'),
      '#options'       => [
        ','  => '1,000',
        '.'  => '1.000',
        ' '  => '1 000',
        ''   => '1000',
      ],
      '#default_value' => $this->getSetting('thousand_sep'),
    ];

    $elements['negative_format'] = [
      '#type'          => 'select',
      '#title'         => $this->t('Negative Value Format'),
      '#options'       => [
        'minus'       => '-฿1,000',
        'parentheses' => '(฿1,000)',
        'color'       => $this->t('Red color'),
      ],
      '#default_value' => $this->getSetting('negative_format'),
    ];

    $elements['zero_display'] = [
      '#type'          => 'select',
      '#title'         => $this->t('Zero Value Display'),
      '#options'       => [
        'value'  => '฿0.00',
        'dash'   => '-',
        'free'   => $this->t('Free'),
      ],
      '#default_value' => $this->getSetting('zero_display'),
    ];

    return $elements;
  }

  public function settingsSummary(): array {
    $symbols = [
      'THB' => '฿',
      'USD' => '$',
      'EUR' => '€',
      'JPY' => '¥',
    ];
    $currency = $this->getSetting('currency');
    $symbol   = $symbols[$currency] ?? '฿';
    $example  = $this->formatValue(1234567.89);

    return [$this->t('Example: @example', ['@example' => $example])];
  }

  public function viewElements(FieldItemListInterface $items, string $langcode): array {
    $elements = [];

    foreach ($items as $delta => $item) {
      $value = (float) $item->value;

      // Handle zero
      if ($value == 0) {
        $zero_display = $this->getSetting('zero_display');
        $markup = match ($zero_display) {
          'dash' => '<span class="currency-zero">-</span>',
          'free' => '<span class="currency-free">' . $this->t('Free') . '</span>',
          default => '<span class="currency-zero">' . htmlspecialchars($this->formatValue(0)) . '</span>',
        };
        $elements[$delta] = ['#markup' => $markup];
        continue;
      }

      $formatted        = $this->formatValue($value);
      $negative_format  = $this->getSetting('negative_format');
      $is_negative      = $value < 0;

      if ($is_negative && $negative_format === 'parentheses') {
        $abs_formatted = $this->formatValue(abs($value));
        $formatted     = "($abs_formatted)";
        $elements[$delta] = [
          '#markup' => '<span class="currency-negative currency-parentheses">'
            . htmlspecialchars($formatted) . '</span>',
        ];
      }
      elseif ($is_negative && $negative_format === 'color') {
        $elements[$delta] = [
          '#markup' => '<span class="currency-negative" style="color:#e74c3c;">'
            . htmlspecialchars($formatted) . '</span>',
        ];
      }
      else {
        $class = $is_negative ? 'currency-negative' : 'currency-positive';
        $elements[$delta] = [
          '#markup' => '<span class="currency-value ' . $class . '">'
            . htmlspecialchars($formatted) . '</span>',
        ];
      }
    }

    return $elements;
  }

  /**
   * Format a numeric value as currency.
   */
  protected function formatValue(float $value): string {
    $symbols = [
      'THB' => '฿',
      'USD' => '$',
      'EUR' => '€',
      'JPY' => '¥',
    ];

    $currency     = $this->getSetting('currency');
    $show_symbol  = (bool) $this->getSetting('show_symbol');
    $decimals     = (int) $this->getSetting('decimal_places');
    $thousand_sep = $this->getSetting('thousand_sep');
    $dec_point    = $thousand_sep === '.' ? ',' : '.';
    $symbol       = $show_symbol ? ($symbols[$currency] ?? '฿') : '';
    $sign         = $value < 0 ? '-' : '';

    $formatted = number_format(abs($value), $decimals, $dec_point, $thousand_sep);

    return $sign . $symbol . $formatted;
  }

}
```

#### 5.7 เพิ่ม block theme ใน .module

```php
<?php
// custom_blocks.module

/**
 * Implements hook_theme().
 */
function custom_blocks_theme(array $existing, string $type, string $theme, string $path): array {
  return [
    'custom_blocks_recent_posts' => [
      'variables' => [
        'items'      => [],
        'show_date'  => TRUE,
        'show_image' => TRUE,
      ],
      'template'  => 'custom-blocks-recent-posts',
    ],
  ];
}
```

#### 5.8 Twig Template สำหรับ Block

```twig
{# templates/custom-blocks-recent-posts.html.twig #}

<div class="recent-posts-block">
  {% for item in items %}
    <article class="recent-post-item">
      {% if show_image and item.image_url %}
        <div class="recent-post-image">
          <a href="{{ item.url }}">
            <img src="{{ item.image_url }}"
                 alt="{{ item.image_alt|e }}"
                 loading="lazy"
                 width="150"
                 height="100">
          </a>
        </div>
      {% endif %}

      <div class="recent-post-content">
        <h4 class="recent-post-title">
          <a href="{{ item.url }}">{{ item.title }}</a>
        </h4>

        {% if show_date and item.date %}
          <time class="recent-post-date">{{ item.date }}</time>
        {% endif %}
      </div>
    </article>
  {% else %}
    <p class="recent-posts-empty">{{ 'No posts found.'|t }}</p>
  {% endfor %}
</div>
```

### 5.9 Enable Module และ Configure Block

```bash
# Enable module
drush en custom_blocks -y
drush cr

# ตรวจสอบ block plugin
drush php-eval "
  \$manager = \Drupal::service('plugin.manager.block');
  \$definitions = \$manager->getDefinitions();
  foreach (\$definitions as \$id => \$def) {
    if (strpos(\$id, 'custom_blocks') !== false) {
      print \$id . ': ' . \$def['admin_label'] . PHP_EOL;
    }
  }
"
```

จากนั้นเพิ่ม Block ผ่าน Admin UI:
1. ไปที่ Structure > Block layout
2. คลิก "Place block" ใน region ที่ต้องการ
3. ค้นหา "Recent Posts" แล้วคลิก "Place block"
4. กำหนด settings และ Save

---

## Quiz

**ข้อ 1:** ข้อใดคือวิธีที่ถูกต้องในการ inject service เข้า Plugin (Block)?

A) ใช้ `\Drupal::service()` ใน `build()` method  
B) Implement `ContainerFactoryPluginInterface` และใช้ `create()` static method ✓  
C) ใช้ `@inject` annotation  
D) เพิ่มในไฟล์ `mymodule.services.yml` โดยตรง  

**เฉลย:** B - Plugin ที่ต้องการ inject service ต้อง implement `ContainerFactoryPluginInterface` และ override `create()` method เพื่อดึง services จาก container

---

**ข้อ 2:** Method ใดใน Entity Query ใช้สำหรับนับจำนวน entity?

A) `->total()`  
B) `->count()` ✓  
C) `->sum()`  
D) `->aggregate()`  

**เฉลย:** B - ใช้ `->count()->execute()` เพื่อ return จำนวน entities แทนที่จะเป็น array ของ IDs

---

**ข้อ 3:** Annotation ใดใช้กำหนด `field_types` ที่ Field Formatter รองรับ?

A) `@FieldType`  
B) `@FieldWidget`  
C) `@FieldFormatter` ✓  
D) `@Formatter`  

**เฉลย:** C - `@FieldFormatter` annotation ใช้กำหนด metadata ของ formatter รวมถึง `field_types` ที่รองรับ

---

**ข้อ 4:** ความแตกต่างระหว่าง Config API และ State API คืออะไร?

A) ไม่มีความแตกต่าง  
B) Config ใช้สำหรับ settings ที่ export/deploy ได้ ส่วน State ใช้สำหรับ runtime data ที่ไม่ควร export ✓  
C) State เร็วกว่า Config เสมอ  
D) Config เก็บใน database ส่วน State เก็บในไฟล์  

**เฉลย:** B - Config เหมาะสำหรับ settings ที่ต้องการ deploy ระหว่าง environment เช่น items_per_page ส่วน State เหมาะสำหรับ runtime data เช่น last cron run time

---

**ข้อ 5:** Cache tag `node_list` ใน Block plugin มีผลอย่างไร?

A) Cache block ไว้ตลอดไป  
B) Invalidate cache ของ block เมื่อมีการสร้าง แก้ไข หรือลบ node ใดๆ ✓  
C) Cache เฉพาะ node ที่แสดงอยู่  
D) ใช้สำหรับ cache ชนิด tag เท่านั้น  

**เฉลย:** B - Cache tag `node_list` ถูก invalidate ทุกครั้งที่มีการ save หรือลบ node ทำให้ block แสดงข้อมูลใหม่เสมอ

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **Services & DI** - การสร้าง Service Class และ inject ผ่าน Constructor
2. **mymodule.services.yml** - ลงทะเบียน Services และ dependencies
3. **Block Plugin** - สร้าง Block ที่กำหนดค่าได้พร้อม Caching
4. **Field Formatter Plugin** - สร้าง Formatter สำหรับ Thai Currency
5. **Entity API** - entityQuery, load, loadMultiple, field values
6. **Config API** - อ่านและเขียน configuration
7. **Workshop** - สร้าง Recent Posts Block และ Currency Formatter จริง

---

## ลิงก์ที่เกี่ยวข้อง

- [Drupal.org - Services and DI](https://www.drupal.org/docs/drupal-apis/services-and-dependency-injection)
- [Drupal.org - Plugin API](https://www.drupal.org/docs/drupal-apis/plugin-api)
- [Drupal.org - Entity API](https://www.drupal.org/docs/drupal-apis/entity-api)
- [Drupal.org - Configuration API](https://www.drupal.org/docs/drupal-apis/configuration-api)
- [Drupal.org - Block API](https://api.drupal.org/api/drupal/core!lib!Drupal!Core!Block!BlockBase.php/class/BlockBase/10)

---

## ไปต่อ

➡️ **[Part 086: Design Patterns](part-086-design-patterns.md)**

ใน Part ถัดไปเราจะเรียนรู้เกี่ยวกับ Design Patterns ที่สำคัญในการพัฒนา Software เช่น Singleton, Factory, Observer, Strategy และ Decorator Patterns พร้อมตัวอย่างการใช้งานจริงใน PHP
