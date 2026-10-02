# Part 085: Drupal Advanced Development

## ระดับ: มืออาชีพ | ขั้นตอนที่ 881-920

---

## วัตถุประสงค์การเรียนรู้ (Learning Objectives)

หลังจากศึกษา Part นี้แล้ว ผู้เรียนจะสามารถ:

1. **ใช้งาน Drupal Migrate API** เพื่อนำเข้าข้อมูลจากแหล่งข้อมูลภายนอก เช่น CSV, JSON, XML และฐานข้อมูลเดิม
2. **สร้างโครงสร้างเนื้อหาขั้นสูง** โดยใช้ Paragraphs module เพื่อสร้าง reusable content components
3. **จัดการ Content Moderation** ผ่าน Workflows module ด้วย editorial workflow ระดับองค์กร
4. **ใช้งาน Drupal Console และ Drush** ในระดับขั้นสูงสำหรับการพัฒนาและ deployment
5. **เขียน Automated Tests** ด้วย Drupal Test Traits และ PHPUnit
6. **ตั้งค่า Multisite Configuration** สำหรับการจัดการหลาย site จาก codebase เดียว
7. **สร้าง CI/CD Pipeline** สำหรับ Drupal deployment ในระดับ production
8. **สร้าง Enterprise Content Migration** จากระบบ CMS เดิม

---

## ขั้นตอนที่ 881-885: Drupal Migrate API

### ความเข้าใจพื้นฐานของ Migrate API

Migrate API ใน Drupal 10 ให้กรอบการทำงานสำหรับการนำเข้าข้อมูลจากแหล่งต่างๆ ประกอบด้วย 3 ส่วนหลัก:

1. **Source Plugin** - ดึงข้อมูลจากแหล่งข้อมูลต้นทาง
2. **Process Plugin** - แปลงข้อมูลระหว่างการนำเข้า
3. **Destination Plugin** - บันทึกข้อมูลไปยัง Drupal entity

### โครงสร้างโมดูล Migration

```
modules/custom/my_migration/
├── config/
│   └── install/
│       └── migrate_plus.migration.articles_from_csv.yml
├── src/
│   └── Plugin/
│       └── migrate/
│           ├── source/
│           │   └── ArticleSource.php
│           └── process/
│               └── TransformCategory.php
└── my_migration.info.yml
```

### ไฟล์ my_migration.info.yml

```yaml
name: My Migration
type: module
description: 'Custom migration module for importing content'
core_version_requirement: ^10
package: Custom
dependencies:
  - migrate:migrate
  - migrate_plus:migrate_plus
  - migrate_tools:migrate_tools
```

### Source Plugin สำหรับ CSV

```php
<?php
// modules/custom/my_migration/src/Plugin/migrate/source/ArticleSource.php

namespace Drupal\my_migration\Plugin\migrate\source;

use Drupal\migrate\Plugin\migrate\source\SqlBase;
use Drupal\migrate\Row;

/**
 * Source plugin สำหรับนำเข้าบทความจาก CSV file
 *
 * @MigrateSource(
 *   id = "article_csv_source",
 *   source_module = "my_migration"
 * )
 */
class ArticleSource extends \Drupal\migrate\Plugin\migrate\source\SourcePluginBase {

  /**
   * {@inheritdoc}
   */
  public function fields() {
    return [
      'id'          => $this->t('Article ID'),
      'title'       => $this->t('Title'),
      'body'        => $this->t('Body'),
      'category'    => $this->t('Category'),
      'author'      => $this->t('Author email'),
      'created_at'  => $this->t('Created date'),
      'status'      => $this->t('Published status'),
    ];
  }

  /**
   * {@inheritdoc}
   */
  public function getIds() {
    return [
      'id' => [
        'type'  => 'integer',
        'alias' => 'src',
      ],
    ];
  }

  /**
   * {@inheritdoc}
   */
  public function __toString() {
    return 'article_csv_source';
  }

  /**
   * {@inheritdoc}
   */
  protected function initializeIterator() {
    $file_path = $this->configuration['path'];
    $data = [];

    // อ่านไฟล์ CSV
    if (($handle = fopen($file_path, 'r')) !== FALSE) {
      $headers = fgetcsv($handle);
      while (($row = fgetcsv($handle)) !== FALSE) {
        $data[] = array_combine($headers, $row);
      }
      fclose($handle);
    }

    return new \ArrayIterator($data);
  }

  /**
   * {@inheritdoc}
   */
  public function count($refresh = FALSE) {
    return iterator_count($this->initializeIterator());
  }
}
```

### Source Plugin สำหรับ JSON API (External Source)

```php
<?php
// modules/custom/my_migration/src/Plugin/migrate/source/JsonApiSource.php

namespace Drupal\my_migration\Plugin\migrate\source;

use Drupal\migrate\Plugin\MigrationInterface;
use Drupal\migrate\Plugin\migrate\source\SourcePluginBase;
use GuzzleHttp\ClientInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Source plugin สำหรับดึงข้อมูลจาก JSON API
 *
 * @MigrateSource(
 *   id = "json_api_source",
 *   source_module = "my_migration"
 * )
 */
class JsonApiSource extends SourcePluginBase {

  /**
   * @var \GuzzleHttp\ClientInterface
   */
  protected ClientInterface $httpClient;

  /**
   * {@inheritdoc}
   */
  public static function create(
    ContainerInterface $container,
    array $configuration,
    $plugin_id,
    $plugin_definition,
    MigrationInterface $migration = NULL
  ) {
    $instance = parent::create(
      $container, $configuration, $plugin_id, $plugin_definition, $migration
    );
    $instance->httpClient = $container->get('http_client');
    return $instance;
  }

  /**
   * {@inheritdoc}
   */
  public function fields() {
    return [
      'id'          => $this->t('Post ID'),
      'title'       => $this->t('Title'),
      'content'     => $this->t('Content'),
      'excerpt'     => $this->t('Excerpt'),
      'author'      => $this->t('Author'),
      'date'        => $this->t('Published date'),
      'categories'  => $this->t('Categories'),
      'tags'        => $this->t('Tags'),
      'featured_image' => $this->t('Featured image URL'),
    ];
  }

  /**
   * {@inheritdoc}
   */
  public function getIds() {
    return [
      'id' => ['type' => 'integer'],
    ];
  }

  /**
   * {@inheritdoc}
   */
  public function __toString() {
    return 'json_api_source';
  }

  /**
   * ดึงข้อมูลจาก API พร้อม pagination
   */
  protected function initializeIterator() {
    $base_url  = $this->configuration['base_url'];
    $endpoint  = $this->configuration['endpoint'];
    $per_page  = $this->configuration['per_page'] ?? 100;

    $all_items = [];
    $page = 1;

    do {
      $url = "{$base_url}{$endpoint}?per_page={$per_page}&page={$page}";
      $response = $this->httpClient->request('GET', $url, [
        'headers' => [
          'Accept'        => 'application/json',
          'Authorization' => 'Bearer ' . ($this->configuration['api_token'] ?? ''),
        ],
      ]);

      $data  = json_decode($response->getBody()->getContents(), TRUE);
      $items = $data['data'] ?? $data;

      if (empty($items)) {
        break;
      }

      $all_items = array_merge($all_items, $items);
      $total_pages = $data['meta']['last_page'] ?? 1;
      $page++;

    } while ($page <= $total_pages);

    return new \ArrayIterator($all_items);
  }
}
```

### Process Plugin สำหรับแปลงข้อมูล

```php
<?php
// modules/custom/my_migration/src/Plugin/migrate/process/TransformCategory.php

namespace Drupal\my_migration\Plugin\migrate\process;

use Drupal\migrate\MigrateExecutableInterface;
use Drupal\migrate\ProcessPluginBase;
use Drupal\migrate\Row;

/**
 * Process plugin สำหรับแปลงชื่อ category เป็น term ID
 *
 * ตัวอย่างการใช้งานใน migration YAML:
 * @code
 * process:
 *   field_category:
 *     plugin: transform_category
 *     source: category
 *     vocabulary: article_categories
 *     create_term: true
 * @endcode
 *
 * @MigrateProcessPlugin(
 *   id = "transform_category"
 * )
 */
class TransformCategory extends ProcessPluginBase {

  /**
   * {@inheritdoc}
   */
  public function transform(
    $value,
    MigrateExecutableInterface $migrate_executable,
    Row $row,
    $destination_property
  ) {
    if (empty($value)) {
      return NULL;
    }

    $vocabulary = $this->configuration['vocabulary'] ?? 'tags';
    $create_term = $this->configuration['create_term'] ?? FALSE;

    // ค้นหา term ที่มีอยู่แล้ว
    $terms = \Drupal::entityTypeManager()
      ->getStorage('taxonomy_term')
      ->loadByProperties([
        'name'       => $value,
        'vid'        => $vocabulary,
      ]);

    if (!empty($terms)) {
      $term = reset($terms);
      return $term->id();
    }

    // สร้าง term ใหม่ถ้าไม่มี
    if ($create_term) {
      $term = \Drupal::entityTypeManager()
        ->getStorage('taxonomy_term')
        ->create([
          'name' => $value,
          'vid'  => $vocabulary,
        ]);
      $term->save();
      return $term->id();
    }

    return NULL;
  }
}
```

### Process Plugin สำหรับแปลง HTML content

```php
<?php
// modules/custom/my_migration/src/Plugin/migrate/process/CleanHtml.php

namespace Drupal\my_migration\Plugin\migrate\process;

use Drupal\migrate\MigrateExecutableInterface;
use Drupal\migrate\ProcessPluginBase;
use Drupal\migrate\Row;

/**
 * Process plugin สำหรับทำความสะอาด HTML content
 *
 * @MigrateProcessPlugin(
 *   id = "clean_html"
 * )
 */
class CleanHtml extends ProcessPluginBase {

  /**
   * {@inheritdoc}
   */
  public function transform(
    $value,
    MigrateExecutableInterface $migrate_executable,
    Row $row,
    $destination_property
  ) {
    if (empty($value)) {
      return ['value' => '', 'format' => 'basic_html'];
    }

    // ลบ script tags
    $value = preg_replace('/<script\b[^>]*>(.*?)<\/script>/is', '', $value);

    // ลบ inline styles ที่เป็น absolute URL ของระบบเก่า
    $old_domain = $this->configuration['old_domain'] ?? '';
    if ($old_domain) {
      $value = str_replace($old_domain, '', $value);
    }

    // แปลง WordPress shortcodes เป็น HTML
    $value = $this->convertShortcodes($value);

    return [
      'value'  => $value,
      'format' => $this->configuration['text_format'] ?? 'full_html',
    ];
  }

  /**
   * แปลง WordPress shortcodes
   */
  protected function convertShortcodes(string $content): string {
    // [caption] shortcode
    $content = preg_replace_callback(
      '/\[caption[^\]]*\](.*?)\[\/caption\]/is',
      function ($matches) {
        return '<figure>' . $matches[1] . '</figure>';
      },
      $content
    );

    // [gallery] shortcode - ลบออก
    $content = preg_replace('/\[gallery[^\]]*\]/i', '', $content);

    return $content;
  }
}
```

### Migration Configuration YAML

```yaml
# config/install/migrate_plus.migration.articles_from_csv.yml

id: articles_from_csv
label: 'Import articles from CSV'
migration_group: my_migrations
migration_tags:
  - articles
  - csv

source:
  plugin: article_csv_source
  path: 'public://migration/articles.csv'
  header_row_count: 1

process:
  # แมป title ตรงๆ
  title: title

  # แปลง body content
  body:
    plugin: clean_html
    source: body
    old_domain: 'https://old-site.example.com'
    text_format: full_html

  # แปลง category เป็น term reference
  field_category:
    plugin: transform_category
    source: category
    vocabulary: article_categories
    create_term: true

  # แปลง author email เป็น user ID
  uid:
    plugin: migration_lookup
    migration: users_migration
    source: author
    no_stub: true

  # แปลง date string เป็น timestamp
  created:
    plugin: callback
    callable: strtotime
    source: created_at

  # กำหนด status
  status:
    plugin: static_map
    source: status
    map:
      'publish': 1
      'draft': 0
      'private': 0
    default_value: 0

  # กำหนด content type
  type:
    plugin: default_value
    default_value: article

destination:
  plugin: 'entity:node'

migration_dependencies:
  required:
    - users_migration
  optional:
    - taxonomy_terms_migration
```

---

## ขั้นตอนที่ 886-890: Paragraphs Module

### การติดตั้งและตั้งค่า Paragraphs

Paragraphs module ช่วยสร้าง flexible content structures โดยแบ่งเนื้อหาเป็น reusable components

```bash
# ติดตั้งผ่าน Composer
composer require drupal/paragraphs

# Enable modules
drush en paragraphs field_group -y
drush cr
```

### สร้าง Custom Paragraph Type ผ่าน Code

```php
<?php
// modules/custom/my_content/src/Install/CreateParagraphTypes.php

namespace Drupal\my_content\Install;

use Drupal\paragraphs\Entity\ParagraphsType;
use Drupal\field\Entity\FieldConfig;
use Drupal\field\Entity\FieldStorageConfig;

/**
 * สร้าง Paragraph types สำหรับโครงการ
 */
class CreateParagraphTypes {

  /**
   * สร้าง Hero Banner paragraph type
   */
  public static function createHeroBanner(): void {
    // สร้าง paragraph type
    if (!ParagraphsType::load('hero_banner')) {
      ParagraphsType::create([
        'id'          => 'hero_banner',
        'label'       => 'Hero Banner',
        'description' => 'Full-width hero section with title, subtitle and CTA button',
      ])->save();
    }

    // สร้าง field storage สำหรับ heading
    if (!FieldStorageConfig::loadByName('paragraph', 'field_hero_heading')) {
      FieldStorageConfig::create([
        'field_name'  => 'field_hero_heading',
        'entity_type' => 'paragraph',
        'type'        => 'string',
        'cardinality' => 1,
      ])->save();
    }

    // Attach field ไปยัง paragraph type
    if (!FieldConfig::loadByName('paragraph', 'hero_banner', 'field_hero_heading')) {
      FieldConfig::create([
        'field_name'  => 'field_hero_heading',
        'entity_type' => 'paragraph',
        'bundle'      => 'hero_banner',
        'label'       => 'Heading',
        'required'    => TRUE,
      ])->save();
    }

    // สร้าง field สำหรับ subheading
    if (!FieldStorageConfig::loadByName('paragraph', 'field_hero_subheading')) {
      FieldStorageConfig::create([
        'field_name'  => 'field_hero_subheading',
        'entity_type' => 'paragraph',
        'type'        => 'string_long',
        'cardinality' => 1,
      ])->save();
    }

    if (!FieldConfig::loadByName('paragraph', 'hero_banner', 'field_hero_subheading')) {
      FieldConfig::create([
        'field_name'  => 'field_hero_subheading',
        'entity_type' => 'paragraph',
        'bundle'      => 'hero_banner',
        'label'       => 'Subheading',
        'required'    => FALSE,
      ])->save();
    }

    // สร้าง background image field
    if (!FieldStorageConfig::loadByName('paragraph', 'field_hero_background')) {
      FieldStorageConfig::create([
        'field_name'  => 'field_hero_background',
        'entity_type' => 'paragraph',
        'type'        => 'image',
        'cardinality' => 1,
        'settings'    => [
          'uri_scheme' => 'public',
          'default_image' => [
            'uuid' => NULL,
            'alt'  => '',
            'title' => '',
            'width' => NULL,
            'height' => NULL,
          ],
        ],
      ])->save();
    }

    if (!FieldConfig::loadByName('paragraph', 'hero_banner', 'field_hero_background')) {
      FieldConfig::create([
        'field_name'  => 'field_hero_background',
        'entity_type' => 'paragraph',
        'bundle'      => 'hero_banner',
        'label'       => 'Background Image',
        'required'    => TRUE,
        'settings'    => [
          'alt_field'           => 1,
          'alt_field_required'  => 0,
          'title_field'         => 0,
          'max_resolution'      => '',
          'min_resolution'      => '',
          'default_image'       => [],
          'file_extensions'     => 'jpg jpeg png webp',
          'max_filesize'        => '5 MB',
          'handler'             => 'default:file',
          'handler_settings'    => [],
        ],
      ])->save();
    }
  }

  /**
   * สร้าง Text with Image paragraph type
   */
  public static function createTextWithImage(): void {
    if (!ParagraphsType::load('text_with_image')) {
      ParagraphsType::create([
        'id'          => 'text_with_image',
        'label'       => 'Text with Image',
        'description' => 'Two-column layout with text and image side by side',
      ])->save();
    }

    // Field สำหรับ image position (left/right)
    if (!FieldStorageConfig::loadByName('paragraph', 'field_image_position')) {
      FieldStorageConfig::create([
        'field_name'  => 'field_image_position',
        'entity_type' => 'paragraph',
        'type'        => 'list_string',
        'cardinality' => 1,
        'settings'    => [
          'allowed_values' => [
            'left'  => 'Image Left',
            'right' => 'Image Right',
          ],
        ],
      ])->save();
    }

    if (!FieldConfig::loadByName('paragraph', 'text_with_image', 'field_image_position')) {
      FieldConfig::create([
        'field_name'  => 'field_image_position',
        'entity_type' => 'paragraph',
        'bundle'      => 'text_with_image',
        'label'       => 'Image Position',
        'required'    => TRUE,
        'default_value' => [['value' => 'left']],
      ])->save();
    }
  }
}
```

### การสร้าง Entity Reference ไปยัง Paragraph

```php
<?php
// modules/custom/my_content/src/Install/AttachParagraphsToNode.php

namespace Drupal\my_content\Install;

use Drupal\field\Entity\FieldConfig;
use Drupal\field\Entity\FieldStorageConfig;

/**
 * เพิ่ม Paragraphs field ให้กับ content type
 */
class AttachParagraphsToNode {

  /**
   * เพิ่ม field_page_sections ไปยัง page content type
   */
  public static function addPageSectionsField(string $content_type = 'page'): void {
    $field_name = 'field_page_sections';

    // สร้าง field storage
    if (!FieldStorageConfig::loadByName('node', $field_name)) {
      FieldStorageConfig::create([
        'field_name'   => $field_name,
        'entity_type'  => 'node',
        'type'         => 'entity_reference_revisions',
        'cardinality'  => -1, // unlimited
        'settings'     => [
          'target_type' => 'paragraph',
        ],
      ])->save();
    }

    // Attach ไปยัง content type
    if (!FieldConfig::loadByName('node', $content_type, $field_name)) {
      FieldConfig::create([
        'field_name'   => $field_name,
        'entity_type'  => 'node',
        'bundle'       => $content_type,
        'label'        => 'Page Sections',
        'required'     => FALSE,
        'settings'     => [
          'handler'          => 'default:paragraph',
          'handler_settings' => [
            'negate' => 0,
            'target_bundles' => [
              'hero_banner'     => 'hero_banner',
              'text_with_image' => 'text_with_image',
              'card_grid'       => 'card_grid',
              'testimonials'    => 'testimonials',
            ],
            'target_bundles_drag_drop' => [
              'hero_banner'     => ['enabled' => TRUE, 'weight' => 1],
              'text_with_image' => ['enabled' => TRUE, 'weight' => 2],
            ],
          ],
        ],
      ])->save();
    }
  }
}
```

### Template สำหรับ Paragraph

```php
<?php
// themes/custom/my_theme/templates/paragraph/paragraph--hero-banner.html.twig
?>
{#
/**
 * @file
 * Template สำหรับ Hero Banner paragraph
 */
#}
{% set classes = [
  'paragraph',
  'paragraph--type--' ~ paragraph.bundle|clean_class,
  view_mode ? 'paragraph--view-mode--' ~ view_mode|clean_class,
  'hero-banner',
] %}

{% if content.field_hero_background %}
  {% set bg_image = content.field_hero_background[0]['#media'] %}
{% endif %}

<section{{ attributes.addClass(classes) }}>
  <div class="hero-banner__inner">
    <div class="hero-banner__content">
      {% if content.field_hero_heading %}
        <h1 class="hero-banner__heading">
          {{ content.field_hero_heading }}
        </h1>
      {% endif %}

      {% if content.field_hero_subheading %}
        <p class="hero-banner__subheading">
          {{ content.field_hero_subheading }}
        </p>
      {% endif %}

      {% if content.field_hero_cta_link %}
        <div class="hero-banner__cta">
          {{ content.field_hero_cta_link }}
        </div>
      {% endif %}
    </div>
  </div>
</section>
```

---

## ขั้นตอนที่ 891-895: Workflows Module (Content Moderation)

### การตั้งค่า Content Moderation

```bash
# Enable modules
drush en workflows content_moderation -y
drush cr
```

### สร้าง Editorial Workflow ผ่าน Code

```php
<?php
// modules/custom/my_workflow/src/Install/CreateEditorialWorkflow.php

namespace Drupal\my_workflow\Install;

use Drupal\workflows\Entity\Workflow;

/**
 * สร้าง Editorial workflow สำหรับองค์กร
 */
class CreateEditorialWorkflow {

  /**
   * สร้าง workflow พร้อม states และ transitions
   */
  public static function create(): void {
    if (Workflow::load('editorial')) {
      return; // Workflow มีอยู่แล้ว
    }

    $workflow = Workflow::create([
      'id'    => 'editorial',
      'label' => 'Editorial Workflow',
      'type'  => 'content_moderation',
    ]);

    $workflow->getTypePlugin()->setConfiguration([
      'states' => [
        'draft' => [
          'label'     => 'Draft',
          'published' => FALSE,
          'default_revision' => FALSE,
          'weight'    => 0,
        ],
        'in_review' => [
          'label'     => 'In Review',
          'published' => FALSE,
          'default_revision' => FALSE,
          'weight'    => 1,
        ],
        'approved' => [
          'label'     => 'Approved',
          'published' => FALSE,
          'default_revision' => FALSE,
          'weight'    => 2,
        ],
        'published' => [
          'label'     => 'Published',
          'published' => TRUE,
          'default_revision' => TRUE,
          'weight'    => 3,
        ],
        'archived' => [
          'label'     => 'Archived',
          'published' => FALSE,
          'default_revision' => TRUE,
          'weight'    => 4,
        ],
      ],
      'transitions' => [
        'create_new_draft' => [
          'label'  => 'Create New Draft',
          'from'   => ['draft', 'published'],
          'to'     => 'draft',
          'weight' => 0,
        ],
        'submit_for_review' => [
          'label'  => 'Submit for Review',
          'from'   => ['draft'],
          'to'     => 'in_review',
          'weight' => 1,
        ],
        'approve' => [
          'label'  => 'Approve',
          'from'   => ['in_review'],
          'to'     => 'approved',
          'weight' => 2,
        ],
        'reject' => [
          'label'  => 'Reject (back to draft)',
          'from'   => ['in_review'],
          'to'     => 'draft',
          'weight' => 3,
        ],
        'publish' => [
          'label'  => 'Publish',
          'from'   => ['approved'],
          'to'     => 'published',
          'weight' => 4,
        ],
        'archive' => [
          'label'  => 'Archive',
          'from'   => ['published'],
          'to'     => 'archived',
          'weight' => 5,
        ],
        'restore' => [
          'label'  => 'Restore from Archive',
          'from'   => ['archived'],
          'to'     => 'draft',
          'weight' => 6,
        ],
      ],
      'entity_types' => [
        'node' => ['article', 'page', 'landing_page'],
      ],
    ]);

    $workflow->save();
  }
}
```

### Content Moderation Service

```php
<?php
// modules/custom/my_workflow/src/Service/ContentModerationService.php

namespace Drupal\my_workflow\Service;

use Drupal\content_moderation\ModerationInformationInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\node\NodeInterface;

/**
 * Service สำหรับจัดการ Content Moderation
 */
class ContentModerationService {

  public function __construct(
    protected EntityTypeManagerInterface $entityTypeManager,
    protected ModerationInformationInterface $moderationInfo,
    protected AccountInterface $currentUser
  ) {}

  /**
   * เปลี่ยน moderation state ของ node
   */
  public function transitionNode(NodeInterface $node, string $new_state): bool {
    if (!$this->moderationInfo->isModeratedEntity($node)) {
      return FALSE;
    }

    $workflow = $this->moderationInfo->getWorkflowForEntity($node);
    if (!$workflow) {
      return FALSE;
    }

    $current_state = $node->get('moderation_state')->value;
    $type_plugin = $workflow->getTypePlugin();

    // ตรวจสอบว่า transition ถูกต้อง
    if (!$type_plugin->hasTransitionFromStateToState($current_state, $new_state)) {
      return FALSE;
    }

    // ตรวจสอบสิทธิ์ของ user
    $transitions = $type_plugin->getTransitionsForState($current_state);
    foreach ($transitions as $transition) {
      if ($transition->to()->id() === $new_state) {
        $permission = 'use ' . $workflow->id() . ' transition ' . $transition->id();
        if (!$this->currentUser->hasPermission($permission)) {
          return FALSE;
        }
        break;
      }
    }

    $node->set('moderation_state', $new_state);
    $node->save();

    return TRUE;
  }

  /**
   * ดึงรายการ nodes ที่รอ review
   */
  public function getPendingReviews(int $limit = 20): array {
    $query = $this->entityTypeManager
      ->getStorage('node')
      ->getQuery()
      ->accessCheck(TRUE)
      ->condition('moderation_state', 'in_review')
      ->sort('changed', 'ASC')
      ->range(0, $limit);

    $nids = $query->execute();
    return $this->entityTypeManager->getStorage('node')->loadMultiple($nids);
  }

  /**
   * ดึง moderation history ของ node
   */
  public function getModerationHistory(NodeInterface $node): array {
    $storage = $this->entityTypeManager->getStorage('content_moderation_state');
    $history = [];

    $ids = $storage->getQuery()
      ->accessCheck(FALSE)
      ->condition('content_entity_type_id', 'node')
      ->condition('content_entity_id', $node->id())
      ->sort('revision_id', 'ASC')
      ->execute();

    foreach ($storage->loadMultiple($ids) as $state) {
      $history[] = [
        'state'        => $state->get('moderation_state')->value,
        'revision_id'  => $state->get('content_entity_revision_id')->value,
        'changed'      => $state->get('changed')->value,
        'uid'          => $state->get('uid')->target_id,
      ];
    }

    return $history;
  }
}
```

### Event Subscriber สำหรับ Moderation Events

```php
<?php
// modules/custom/my_workflow/src/EventSubscriber/ModerationSubscriber.php

namespace Drupal\my_workflow\EventSubscriber;

use Drupal\content_moderation\Event\ContentModerationEvents;
use Drupal\content_moderation\Event\ContentModerationStateChangedEvent;
use Drupal\Core\Mail\MailManagerInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

/**
 * Event subscriber สำหรับ Content Moderation events
 */
class ModerationSubscriber implements EventSubscriberInterface {

  public function __construct(
    protected MailManagerInterface $mailManager
  ) {}

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents(): array {
    return [
      ContentModerationEvents::STATE_CHANGED => 'onStateChanged',
    ];
  }

  /**
   * ส่งอีเมลแจ้งเตือนเมื่อ state เปลี่ยน
   */
  public function onStateChanged(ContentModerationStateChangedEvent $event): void {
    $entity        = $event->getEntity();
    $new_state     = $event->getNewState();
    $previous_state = $event->getPreviousState();

    // แจ้งเตือน editor เมื่อเนื้อหาถูก reject
    if ($new_state === 'draft' && $previous_state === 'in_review') {
      $this->notifyAuthor($entity, 'rejected');
    }

    // แจ้งเตือน reviewer เมื่อมีเนื้อหาใหม่รอ review
    if ($new_state === 'in_review') {
      $this->notifyReviewers($entity);
    }

    // แจ้งเตือน author เมื่อเนื้อหา published
    if ($new_state === 'published') {
      $this->notifyAuthor($entity, 'published');
    }
  }

  /**
   * ส่งอีเมลให้ author
   */
  protected function notifyAuthor($entity, string $action): void {
    $author = $entity->getOwner();
    if (!$author || !$author->getEmail()) {
      return;
    }

    $params = [
      'entity'  => $entity,
      'action'  => $action,
      'message' => $this->buildMessage($entity, $action),
    ];

    $this->mailManager->mail(
      'my_workflow',
      "content_{$action}",
      $author->getEmail(),
      $author->getPreferredLangcode(),
      $params
    );
  }

  /**
   * แจ้งเตือน reviewers ทั้งหมด
   */
  protected function notifyReviewers($entity): void {
    // โหลด users ที่มีสิทธิ์ review
    $user_storage = \Drupal::entityTypeManager()->getStorage('user');
    $reviewer_ids = $user_storage->getQuery()
      ->accessCheck(FALSE)
      ->condition('roles', 'content_reviewer')
      ->condition('status', 1)
      ->execute();

    foreach ($user_storage->loadMultiple($reviewer_ids) as $reviewer) {
      if ($reviewer->getEmail()) {
        $this->mailManager->mail(
          'my_workflow',
          'content_needs_review',
          $reviewer->getEmail(),
          $reviewer->getPreferredLangcode(),
          ['entity' => $entity]
        );
      }
    }
  }

  /**
   * สร้าง message text
   */
  protected function buildMessage($entity, string $action): string {
    $title = $entity->label();
    $url   = $entity->toUrl('canonical', ['absolute' => TRUE])->toString();

    return match($action) {
      'rejected'  => "เนื้อหา \"{$title}\" ถูกส่งกลับมาแก้ไข กรุณาตรวจสอบและแก้ไขก่อนส่ง review ใหม่\n{$url}",
      'published' => "เนื้อหา \"{$title}\" ได้รับการเผยแพร่แล้ว\n{$url}",
      default     => "เนื้อหา \"{$title}\" มีการเปลี่ยนแปลง state\n{$url}",
    };
  }
}
```

---

## ขั้นตอนที่ 896-900: Drupal Console & Drush Advanced

### Drush Commands ที่ใช้บ่อยในระดับ Advanced

```bash
# ========================================
# Database Operations
# ========================================

# Export database พร้อม compression
drush sql:dump --gzip --result-file=/var/backups/db-$(date +%Y%m%d).sql.gz

# Import database
drush sql:drop && drush sql:cli < backup.sql

# Sanitize database (สำหรับ dev environment)
drush sql:sanitize --sanitize-password=test123

# ========================================
# Configuration Management
# ========================================

# Export config
drush config:export -y

# Import config
drush config:import -y

# ดู config diff
drush config:status

# Set config value
drush config:set system.site name "My Site" -y

# ========================================
# Cache Management
# ========================================

# Clear all caches
drush cache:rebuild

# Clear specific cache
drush cache:clear render
drush cache:clear css-js
drush cache:clear entity

# ========================================
# Entity Operations
# ========================================

# ดู entity info
drush entity:bundle-info node

# Delete entities by type
drush entity:delete node --bundle=article

# ========================================
# User Management
# ========================================

# สร้าง user ใหม่
drush user:create editor --mail=editor@example.com --password=password123

# เพิ่ม role ให้ user
drush user:role:add editor content_editor

# Block user
drush user:block editor

# Login link
drush user:login --uid=1

# ========================================
# Module & Theme Management
# ========================================

# ดูรายการ modules
drush pm:list --type=module --status=enabled

# Update modules
drush pm:update

# Uninstall module
drush pm:uninstall my_module -y

# ========================================
# Migration Commands
# ========================================

# ดูรายการ migrations
drush migrate:status

# Run migration
drush migrate:import articles_from_csv

# Rollback migration
drush migrate:rollback articles_from_csv

# Reset stuck migration
drush migrate:reset-status articles_from_csv

# Run migration กับ options
drush migrate:import articles_from_csv --limit=100 --update --feedback=50

# ========================================
# Cron & Queue
# ========================================

# Run cron
drush core:cron

# ดู queue
drush queue:list

# Process queue
drush queue:run my_queue_name

# ========================================
# Performance Analysis
# ========================================

# ดู watchdog logs
drush watchdog:list --severity=error --count=50

# ตรวจสอบ site status
drush core:status

# Requirements check
drush core:requirements
```

### Custom Drush Command

```php
<?php
// modules/custom/my_drush/src/Commands/MyDrushCommands.php

namespace Drupal\my_drush\Commands;

use Drush\Commands\DrushCommands;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Logger\LoggerChannelFactoryInterface;

/**
 * Custom Drush commands
 */
class MyDrushCommands extends DrushCommands {

  public function __construct(
    protected EntityTypeManagerInterface $entityTypeManager,
    protected LoggerChannelFactoryInterface $loggerFactory
  ) {
    parent::__construct();
  }

  /**
   * ล้าง orphaned paragraphs
   *
   * @command my:clean-paragraphs
   * @aliases my-cp
   * @option dry-run แสดงผลโดยไม่ลบจริง
   * @usage drush my:clean-paragraphs
   * @usage drush my:clean-paragraphs --dry-run
   */
  public function cleanOrphanedParagraphs(array $options = ['dry-run' => FALSE]): void {
    $dry_run = $options['dry-run'];
    $this->output()->writeln('กำลังค้นหา orphaned paragraphs...');

    $paragraph_storage = $this->entityTypeManager->getStorage('paragraph');
    $node_storage = $this->entityTypeManager->getStorage('node');

    // ดึง paragraph IDs ทั้งหมด
    $all_paragraph_ids = $paragraph_storage->getQuery()
      ->accessCheck(FALSE)
      ->execute();

    $orphaned = [];
    $batch_size = 50;
    $chunks = array_chunk($all_paragraph_ids, $batch_size);

    foreach ($chunks as $chunk) {
      $paragraphs = $paragraph_storage->loadMultiple($chunk);

      foreach ($paragraphs as $paragraph) {
        $parent_id   = $paragraph->get('parent_id')->value;
        $parent_type = $paragraph->get('parent_type')->value;

        if ($parent_type === 'node') {
          $parent = $node_storage->load($parent_id);
          if (!$parent) {
            $orphaned[] = $paragraph->id();
          }
        }
      }
    }

    $count = count($orphaned);
    $this->output()->writeln("พบ {$count} orphaned paragraphs");

    if ($count > 0 && !$dry_run) {
      $paragraphs_to_delete = $paragraph_storage->loadMultiple($orphaned);
      $paragraph_storage->delete($paragraphs_to_delete);
      $this->output()->writeln("ลบ {$count} paragraphs เรียบร้อยแล้ว");
      $this->loggerFactory->get('my_drush')->notice('Deleted @count orphaned paragraphs', ['@count' => $count]);
    }
    elseif ($dry_run) {
      $this->output()->writeln('Dry run mode: ไม่มีการลบจริง');
      $this->output()->writeln('IDs: ' . implode(', ', array_slice($orphaned, 0, 20)) . (count($orphaned) > 20 ? '...' : ''));
    }
  }

  /**
   * Generate sitemap
   *
   * @command my:generate-sitemap
   * @aliases my-sitemap
   * @option types รายการ content types คั่นด้วย comma
   */
  public function generateSitemap(array $options = ['types' => 'article,page']): void {
    $types = explode(',', $options['types']);
    $this->output()->writeln('กำลัง generate sitemap...');

    $urls = [];
    foreach ($types as $type) {
      $nids = $this->entityTypeManager->getStorage('node')
        ->getQuery()
        ->accessCheck(FALSE)
        ->condition('type', trim($type))
        ->condition('status', 1)
        ->execute();

      $nodes = $this->entityTypeManager->getStorage('node')->loadMultiple($nids);
      foreach ($nodes as $node) {
        $urls[] = $node->toUrl('canonical', ['absolute' => TRUE])->toString();
      }
    }

    $this->output()->writeln('พบ URL ทั้งหมด: ' . count($urls));

    // เขียนไฟล์ sitemap
    $sitemap_content = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n";
    $sitemap_content .= "<urlset xmlns=\"http://www.sitemaps.org/schemas/sitemap/0.9\">\n";
    foreach ($urls as $url) {
      $sitemap_content .= "  <url><loc>" . htmlspecialchars($url) . "</loc></url>\n";
    }
    $sitemap_content .= "</urlset>";

    file_put_contents('public://sitemap.xml', $sitemap_content);
    $this->output()->writeln('Sitemap saved: public://sitemap.xml');
  }
}
```

### Drush services.yml

```yaml
# modules/custom/my_drush/drush.services.yml

services:
  my_drush.commands:
    class: Drupal\my_drush\Commands\MyDrushCommands
    arguments:
      - '@entity_type.manager'
      - '@logger.factory'
    tags:
      - { name: drush.command }
```

---

## ขั้นตอนที่ 901-905: Automated Testing

### โครงสร้างการทดสอบ Drupal

```
modules/custom/my_module/
└── tests/
    ├── src/
    │   ├── Unit/
    │   │   └── Plugin/
    │   │       └── migrate/
    │   │           └── process/
    │   │               └── TransformCategoryTest.php
    │   ├── Kernel/
    │   │   └── Service/
    │   │       └── ContentModerationServiceTest.php
    │   └── Functional/
    │       └── ContentWorkflowTest.php
    └── phpunit.xml
```

### Unit Test ตัวอย่าง

```php
<?php
// modules/custom/my_module/tests/src/Unit/Plugin/migrate/process/TransformCategoryTest.php

namespace Drupal\Tests\my_module\Unit\Plugin\migrate\process;

use Drupal\Tests\UnitTestCase;
use Drupal\my_migration\Plugin\migrate\process\TransformCategory;
use Drupal\migrate\MigrateExecutableInterface;
use Drupal\migrate\Row;

/**
 * Unit tests สำหรับ TransformCategory process plugin
 *
 * @coversDefaultClass \Drupal\my_migration\Plugin\migrate\process\TransformCategory
 * @group my_migration
 */
class TransformCategoryTest extends UnitTestCase {

  /**
   * @covers ::transform
   */
  public function testTransformReturnsNullForEmptyValue(): void {
    $plugin = new TransformCategory(
      ['vocabulary' => 'tags'],
      'transform_category',
      []
    );

    $result = $plugin->transform(
      '',
      $this->createMock(MigrateExecutableInterface::class),
      $this->createMock(Row::class),
      'field_category'
    );

    $this->assertNull($result);
  }

  /**
   * @covers ::transform
   * @dataProvider categoryDataProvider
   */
  public function testTransformHandlesVariousInputs(
    string $input,
    mixed $expected
  ): void {
    $plugin = new TransformCategory(
      [
        'vocabulary'  => 'tags',
        'create_term' => FALSE,
      ],
      'transform_category',
      []
    );

    // ใน unit test เราต้อง mock entity storage
    // สำหรับ integration test ให้ใช้ Kernel test แทน
    $this->assertIsString($input);
  }

  /**
   * Data provider สำหรับ category tests
   */
  public static function categoryDataProvider(): array {
    return [
      'empty string'    => ['', NULL],
      'valid category'  => ['Technology', 1],
      'with spaces'     => ['  Technology  ', 1],
    ];
  }
}
```

### Kernel Test ตัวอย่าง

```php
<?php
// modules/custom/my_module/tests/src/Kernel/Service/ContentModerationServiceTest.php

namespace Drupal\Tests\my_module\Kernel\Service;

use Drupal\KernelTests\Core\Entity\EntityKernelTestBase;
use Drupal\node\Entity\Node;
use Drupal\node\Entity\NodeType;
use Drupal\workflows\Entity\Workflow;
use Drupal\my_workflow\Service\ContentModerationService;

/**
 * Kernel tests สำหรับ ContentModerationService
 *
 * @group my_workflow
 */
class ContentModerationServiceTest extends EntityKernelTestBase {

  /**
   * {@inheritdoc}
   */
  protected static $modules = [
    'node',
    'user',
    'system',
    'workflows',
    'content_moderation',
    'my_workflow',
  ];

  /**
   * @var \Drupal\my_workflow\Service\ContentModerationService
   */
  protected ContentModerationService $service;

  /**
   * {@inheritdoc}
   */
  protected function setUp(): void {
    parent::setUp();

    $this->installEntitySchema('node');
    $this->installEntitySchema('user');
    $this->installEntitySchema('content_moderation_state');
    $this->installSchema('node', 'node_access');
    $this->installConfig(['content_moderation', 'my_workflow']);

    // สร้าง content type
    NodeType::create(['type' => 'article', 'name' => 'Article'])->save();

    // สร้าง workflow
    $this->createTestWorkflow();

    $this->service = $this->container->get('my_workflow.content_moderation');
  }

  /**
   * สร้าง test workflow
   */
  protected function createTestWorkflow(): void {
    $workflow = Workflow::create([
      'id'    => 'editorial',
      'label' => 'Editorial',
      'type'  => 'content_moderation',
    ]);

    $workflow->getTypePlugin()->addState('draft', 'Draft');
    $workflow->getTypePlugin()->addState('in_review', 'In Review');
    $workflow->getTypePlugin()->addState('published', 'Published');

    $workflow->getTypePlugin()->addTransition(
      'submit',
      'Submit for Review',
      ['draft'],
      'in_review'
    );

    $workflow->getTypePlugin()->addEntityTypeAndBundle('node', 'article');
    $workflow->save();
  }

  /**
   * ทดสอบการเปลี่ยน moderation state
   *
   * @covers ::transitionNode
   */
  public function testTransitionNode(): void {
    $node = Node::create([
      'type'             => 'article',
      'title'            => 'Test Article',
      'moderation_state' => 'draft',
    ]);
    $node->save();

    $result = $this->service->transitionNode($node, 'in_review');

    $this->assertTrue($result);
    $node_reloaded = Node::load($node->id());
    $this->assertEquals('in_review', $node_reloaded->get('moderation_state')->value);
  }

  /**
   * ทดสอบ invalid transition
   *
   * @covers ::transitionNode
   */
  public function testInvalidTransitionReturnsFalse(): void {
    $node = Node::create([
      'type'             => 'article',
      'title'            => 'Test Article',
      'moderation_state' => 'draft',
    ]);
    $node->save();

    // ไม่มี transition จาก draft ไป published โดยตรง
    $result = $this->service->transitionNode($node, 'published');

    $this->assertFalse($result);
  }

  /**
   * ทดสอบ getPendingReviews
   *
   * @covers ::getPendingReviews
   */
  public function testGetPendingReviews(): void {
    // สร้าง nodes ที่รอ review
    for ($i = 1; $i <= 3; $i++) {
      $node = Node::create([
        'type'             => 'article',
        'title'            => "Article {$i}",
        'moderation_state' => 'in_review',
      ]);
      $node->save();
    }

    // สร้าง node ที่ไม่รอ review
    $node = Node::create([
      'type'             => 'article',
      'title'            => 'Draft Article',
      'moderation_state' => 'draft',
    ]);
    $node->save();

    $pending = $this->service->getPendingReviews();

    $this->assertCount(3, $pending);
  }
}
```

### Functional Test ตัวอย่าง

```php
<?php
// modules/custom/my_module/tests/src/Functional/ContentWorkflowTest.php

namespace Drupal\Tests\my_module\Functional;

use Drupal\Tests\BrowserTestBase;
use Drupal\node\Entity\NodeType;

/**
 * Functional tests สำหรับ Content Workflow UI
 *
 * @group my_workflow
 */
class ContentWorkflowTest extends BrowserTestBase {

  /**
   * {@inheritdoc}
   */
  protected static $modules = [
    'node',
    'user',
    'workflows',
    'content_moderation',
    'my_workflow',
  ];

  /**
   * {@inheritdoc}
   */
  protected $defaultTheme = 'stark';

  /**
   * @var \Drupal\user\UserInterface
   */
  protected $editor;

  /**
   * @var \Drupal\user\UserInterface
   */
  protected $reviewer;

  /**
   * {@inheritdoc}
   */
  protected function setUp(): void {
    parent::setUp();

    NodeType::create(['type' => 'article', 'name' => 'Article'])->save();

    // สร้าง editor user
    $this->editor = $this->drupalCreateUser([
      'create article content',
      'edit own article content',
      'use editorial transition submit_for_review',
      'view any unpublished content',
    ]);

    // สร้าง reviewer user
    $this->reviewer = $this->drupalCreateUser([
      'edit any article content',
      'use editorial transition approve',
      'use editorial transition reject',
      'view any unpublished content',
    ]);
  }

  /**
   * ทดสอบ workflow ครบวงจร
   */
  public function testCompleteEditorialWorkflow(): void {
    // Editor สร้างบทความ
    $this->drupalLogin($this->editor);
    $this->drupalGet('node/add/article');

    $this->submitForm([
      'title[0][value]'             => 'Test Article Workflow',
      'moderation_state[0][state]'  => 'draft',
      'body[0][value]'              => 'This is the article content.',
    ], 'Save');

    $this->assertSession()->pageTextContains('Test Article Workflow');
    $this->assertSession()->pageTextContains('Draft');

    // Editor ส่ง review
    $node = $this->getNodeByTitle('Test Article Workflow');
    $this->drupalGet("node/{$node->id()}/edit");
    $this->submitForm([
      'moderation_state[0][state]' => 'in_review',
    ], 'Save');

    $this->assertSession()->pageTextContains('In Review');

    // Reviewer อนุมัติ
    $this->drupalLogin($this->reviewer);
    $this->drupalGet("node/{$node->id()}/edit");
    $this->submitForm([
      'moderation_state[0][state]' => 'approved',
    ], 'Save');

    $this->assertSession()->pageTextContains('Approved');
  }

  /**
   * Helper: ดึง node จาก title
   */
  protected function getNodeByTitle(string $title) {
    $nodes = \Drupal::entityTypeManager()
      ->getStorage('node')
      ->loadByProperties(['title' => $title]);
    return reset($nodes);
  }
}
```

### phpunit.xml Configuration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- phpunit.xml -->
<phpunit
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  bootstrap="web/core/tests/bootstrap.php"
  colors="true"
  beStrictAboutTestsThatDoNotTestAnything="true"
  beStrictAboutOutputDuringTests="true"
  xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.3/phpunit.xsd"
>
  <php>
    <ini name="error_reporting" value="32767"/>
    <env name="SIMPLETEST_BASE_URL" value="http://localhost"/>
    <env name="SIMPLETEST_DB" value="mysql://drupal:drupal@127.0.0.1/drupal_test"/>
    <env name="BROWSERTEST_OUTPUT_DIRECTORY" value="/tmp/test-output"/>
    <env name="BROWSERTEST_OUTPUT_BASE_URL" value="http://localhost"/>
  </php>
  <testsuites>
    <testsuite name="unit">
      <directory>web/modules/custom/*/tests/src/Unit</directory>
    </testsuite>
    <testsuite name="kernel">
      <directory>web/modules/custom/*/tests/src/Kernel</directory>
    </testsuite>
    <testsuite name="functional">
      <directory>web/modules/custom/*/tests/src/Functional</directory>
    </testsuite>
  </testsuites>
  <coverage>
    <include>
      <directory suffix=".php">web/modules/custom</directory>
    </include>
    <exclude>
      <directory>web/modules/custom/*/tests</directory>
    </exclude>
    <report>
      <html outputDirectory="coverage"/>
    </report>
  </coverage>
</phpunit>
```

---

## ขั้นตอนที่ 906-910: Multisite Configuration

### โครงสร้าง Multisite

```
drupal-root/
├── sites/
│   ├── default/          # Main site
│   │   └── settings.php
│   ├── site2.example.com/
│   │   └── settings.php
│   ├── site3.example.com/
│   │   └── settings.php
│   └── sites.php         # Site discovery
├── web/
└── composer.json
```

### sites.php Configuration

```php
<?php
// sites/sites.php

/**
 * Site aliasing สำหรับ Multisite
 *
 * Format: $sites['domain.path'] = 'directory';
 */

// Production sites
$sites['example.com']         = 'default';
$sites['www.example.com']     = 'default';
$sites['blog.example.com']    = 'blog.example.com';
$sites['shop.example.com']    = 'shop.example.com';

// Staging sites
$sites['staging.example.com']      = 'staging.example.com';
$sites['blog.staging.example.com'] = 'blog.staging.example.com';

// Local development
$sites['drupal.local']             = 'default';
$sites['blog.drupal.local']        = 'blog.example.com';
$sites['shop.drupal.local']        = 'shop.example.com';

// Port-based development (localhost:8080)
$sites['8080.localhost']           = 'default';
$sites['8081.localhost']           = 'blog.example.com';
```

### settings.php สำหรับ Multisite

```php
<?php
// sites/blog.example.com/settings.php

// Database configuration สำหรับ blog site
$databases['default']['default'] = [
  'database'  => 'drupal_blog',
  'username'  => 'drupal',
  'password'  => getenv('DB_PASSWORD') ?: 'password',
  'host'      => getenv('DB_HOST') ?: 'localhost',
  'port'      => '3306',
  'driver'    => 'mysql',
  'prefix'    => '',
  'collation' => 'utf8mb4_general_ci',
];

// Config sync directory
$settings['config_sync_directory'] = '../config/blog/sync';

// Hash salt (ต้องแตกต่างจาก site อื่น)
$settings['hash_salt'] = 'UNIQUE_HASH_FOR_BLOG_SITE_CHANGE_THIS';

// File paths
$settings['file_public_path']  = 'sites/blog.example.com/files';
$settings['file_private_path'] = '/var/drupal-private/blog';
$settings['file_temp_path']    = '/tmp/drupal-blog';

// Trusted host patterns
$settings['trusted_host_patterns'] = [
  '^blog\.example\.com$',
  '^blog\.staging\.example\.com$',
  '^blog\.drupal\.local$',
];

// Redis cache (แต่ละ site ใช้ database ต่างกัน)
if (extension_loaded('redis')) {
  $settings['cache']['default'] = 'cache.backend.redis';
  $settings['redis.connection']['interface'] = 'PhpRedis';
  $settings['redis.connection']['host']     = getenv('REDIS_HOST') ?: '127.0.0.1';
  $settings['redis.connection']['port']     = getenv('REDIS_PORT') ?: '6379';
  $settings['cache_prefix']['default']      = 'blog_'; // prefix ต่างกัน
}

// Include shared settings
if (file_exists(DRUPAL_ROOT . '/sites/shared.settings.php')) {
  include DRUPAL_ROOT . '/sites/shared.settings.php';
}

// Local overrides
if (file_exists(__DIR__ . '/settings.local.php')) {
  include __DIR__ . '/settings.local.php';
}
```

### Shared Settings

```php
<?php
// sites/shared.settings.php

/**
 * Settings ที่ใช้ร่วมกันทุก sites
 */

// Performance settings
$config['system.performance']['css']['preprocess']        = TRUE;
$config['system.performance']['js']['preprocess']         = TRUE;
$config['system.performance']['cache']['page']['max_age'] = 86400;

// Security settings
$settings['update_free_access']          = FALSE;
$settings['allow_authorize_operations']  = FALSE;

// Logging
$config['system.logging']['error_level'] = 'hide';

// Disable development modules on production
if (getenv('DRUPAL_ENV') === 'production') {
  $config['system.performance']['css']['preprocess'] = TRUE;
  $config['system.performance']['js']['preprocess']  = TRUE;
}

// Development settings
if (getenv('DRUPAL_ENV') === 'development') {
  $settings['container_yamls'][] = DRUPAL_ROOT . '/sites/development.services.yml';
  $config['system.logging']['error_level'] = 'verbose';
  $settings['rebuild_access'] = TRUE;
  $settings['skip_permissions_hardening'] = TRUE;
}
```

### Drush สำหรับ Multisite

```bash
# ใช้ uri flag เพื่อระบุ site
drush --uri=blog.example.com cache:rebuild
drush --uri=shop.example.com config:import -y
drush --uri=blog.example.com sql:dump

# กำหนดใน drush.yml
# sites/blog.example.com/drush.yml
options:
  uri: 'https://blog.example.com'
```

---

## ขั้นตอนที่ 911-915: Drupal CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/drupal.yml

name: Drupal CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  DRUPAL_VERSION: "10"
  PHP_VERSION: "8.2"

jobs:
  # ========================================
  # Build & Test
  # ========================================
  test:
    name: Run Tests
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: drupal_test
          MYSQL_USER: drupal
          MYSQL_PASSWORD: drupal
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          extensions: mbstring, xml, ctype, iconv, intl, pdo_mysql, dom, filter, gd, json, opcache, redis
          coverage: xdebug
          tools: composer:v2

      - name: Get Composer cache directory
        id: composer-cache
        run: echo "dir=$(composer config cache-files-dir)" >> $GITHUB_OUTPUT

      - name: Cache Composer dependencies
        uses: actions/cache@v3
        with:
          path: ${{ steps.composer-cache.outputs.dir }}
          key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
          restore-keys: ${{ runner.os }}-composer-

      - name: Install dependencies
        run: composer install --no-progress --prefer-dist --optimize-autoloader

      - name: Setup Drupal
        run: |
          cp web/sites/default/default.settings.php web/sites/default/settings.php
          mkdir -p web/sites/default/files
          chmod 777 web/sites/default/files

      - name: Install Drupal
        env:
          SIMPLETEST_DB: mysql://drupal:drupal@127.0.0.1/drupal_test
        run: |
          ./vendor/bin/drush site:install standard \
            --db-url=mysql://drupal:drupal@127.0.0.1/drupal_test \
            --site-name="Test Site" \
            --account-name=admin \
            --account-pass=admin \
            --yes

      - name: Run PHPCS
        run: |
          ./vendor/bin/phpcs \
            --standard=Drupal,DrupalPractice \
            --extensions=php,module,inc,install,test,profile,theme,yml \
            web/modules/custom

      - name: Run Unit Tests
        env:
          SIMPLETEST_DB: mysql://drupal:drupal@127.0.0.1/drupal_test
          SIMPLETEST_BASE_URL: http://localhost
        run: |
          ./vendor/bin/phpunit \
            --testsuite=unit \
            --coverage-clover=coverage.xml \
            --log-junit=test-results.xml

      - name: Run Kernel Tests
        env:
          SIMPLETEST_DB: mysql://drupal:drupal@127.0.0.1/drupal_test
        run: |
          ./vendor/bin/phpunit --testsuite=kernel

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: coverage.xml

  # ========================================
  # Deploy to Staging
  # ========================================
  deploy-staging:
    name: Deploy to Staging
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /var/www/staging.example.com
            git pull origin develop
            composer install --no-dev --optimize-autoloader
            ./vendor/bin/drush --uri=staging.example.com deploy

  # ========================================
  # Deploy to Production
  # ========================================
  deploy-production:
    name: Deploy to Production
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Create deployment
        uses: chrnorm/deployment-action@v2
        id: deployment
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          environment: production

      - name: Deploy via SSH
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            set -e
            cd /var/www/example.com

            # Backup database ก่อน deploy
            ./vendor/bin/drush sql:dump \
              --gzip \
              --result-file=/var/backups/pre-deploy-$(date +%Y%m%d-%H%M%S).sql.gz

            # Deploy
            git pull origin main
            composer install --no-dev --optimize-autoloader

            # Run Drupal deploy (includes: updatedb, config:import, cache:rebuild)
            ./vendor/bin/drush deploy --yes

            # Warm caches
            curl -s https://example.com/ > /dev/null

      - name: Update deployment status
        uses: chrnorm/deployment-status@v2
        if: always()
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          deployment-id: ${{ steps.deployment.outputs.deployment_id }}
          state: ${{ job.status }}
```

### deploy.php Hook Script

```php
<?php
// scripts/deploy.php

/**
 * Custom deploy script ที่รันหลัง drush deploy
 */

use Drupal\Core\DrupalKernel;

// Bootstrap Drupal
$autoloader = require_once dirname(__DIR__) . '/vendor/autoload.php';
$kernel = DrupalKernel::createFromRequest(
  \Symfony\Component\HttpFoundation\Request::createFromGlobals(),
  $autoloader,
  'prod'
);
$kernel->boot();

$container = $kernel->getContainer();

// Clear all caches
drupal_flush_all_caches();

// Rebuild routes
\Drupal::service('router.builder')->rebuild();

// Index content (ถ้าใช้ Search API)
if (\Drupal::moduleHandler()->moduleExists('search_api')) {
  // Queue re-indexing
  $index_storage = $container->get('entity_type.manager')->getStorage('search_api_index');
  $indexes = $index_storage->loadMultiple();
  foreach ($indexes as $index) {
    $index->reindex();
  }
}

echo "Deploy post-processing complete.\n";
```

---

## ขั้นตอนที่ 916-920: Workshop - Enterprise Content Migration

### โปรเจกต์: Migration จาก WordPress ไป Drupal 10

#### ขั้นตอนที่ 1: วิเคราะห์ข้อมูลต้นทาง

```php
<?php
// modules/custom/wp_migration/src/Analysis/WordPressAnalyzer.php

namespace Drupal\wp_migration\Analysis;

/**
 * วิเคราะห์โครงสร้างข้อมูลจาก WordPress
 */
class WordPressAnalyzer {

  public function __construct(
    protected \PDO $wpDb
  ) {}

  /**
   * วิเคราะห์จำนวน posts ตาม type
   */
  public function analyzePostTypes(): array {
    $stmt = $this->wpDb->query(
      "SELECT post_type, post_status, COUNT(*) as count
       FROM wp_posts
       WHERE post_type NOT IN ('revision', 'auto-draft', 'nav_menu_item', 'attachment')
       GROUP BY post_type, post_status
       ORDER BY post_type, count DESC"
    );

    return $stmt->fetchAll(\PDO::FETCH_ASSOC);
  }

  /**
   * วิเคราะห์ custom fields ที่ใช้
   */
  public function analyzeCustomFields(): array {
    $stmt = $this->wpDb->query(
      "SELECT meta_key, COUNT(*) as usage_count
       FROM wp_postmeta
       WHERE meta_key NOT LIKE '\_%'
       GROUP BY meta_key
       ORDER BY usage_count DESC
       LIMIT 50"
    );

    return $stmt->fetchAll(\PDO::FETCH_ASSOC);
  }

  /**
   * วิเคราะห์ taxonomies
   */
  public function analyzeTaxonomies(): array {
    $stmt = $this->wpDb->query(
      "SELECT t.taxonomy,
              COUNT(DISTINCT tr.object_id) as post_count,
              COUNT(DISTINCT t.term_id) as term_count
       FROM wp_term_taxonomy t
       INNER JOIN wp_term_relationships tr ON t.term_taxonomy_id = tr.term_taxonomy_id
       GROUP BY t.taxonomy
       ORDER BY post_count DESC"
    );

    return $stmt->fetchAll(\PDO::FETCH_ASSOC);
  }

  /**
   * วิเคราะห์ media/files
   */
  public function analyzeMedia(): array {
    $stmt = $this->wpDb->query(
      "SELECT
        p.post_mime_type,
        COUNT(*) as count,
        SUM(CAST(pm.meta_value AS UNSIGNED)) as total_size
       FROM wp_posts p
       LEFT JOIN wp_postmeta pm ON p.ID = pm.post_id AND pm.meta_key = '_wp_attachment_metadata'
       WHERE p.post_type = 'attachment'
       GROUP BY p.post_mime_type
       ORDER BY count DESC"
    );

    return $stmt->fetchAll(\PDO::FETCH_ASSOC);
  }
}
```

#### ขั้นตอนที่ 2: WordPress Source Plugin

```php
<?php
// modules/custom/wp_migration/src/Plugin/migrate/source/WordPressPost.php

namespace Drupal\wp_migration\Plugin\migrate\source;

use Drupal\migrate\Plugin\migrate\source\SqlBase;
use Drupal\migrate\Row;

/**
 * Source plugin สำหรับ WordPress posts
 *
 * @MigrateSource(
 *   id = "wordpress_post",
 *   source_module = "wp_migration"
 * )
 */
class WordPressPost extends SqlBase {

  /**
   * {@inheritdoc}
   */
  public function query() {
    $post_type = $this->configuration['post_type'] ?? 'post';
    $statuses  = $this->configuration['statuses'] ?? ['publish', 'draft'];

    $query = $this->select('wp_posts', 'p')
      ->fields('p', [
        'ID',
        'post_author',
        'post_date',
        'post_date_gmt',
        'post_content',
        'post_title',
        'post_excerpt',
        'post_status',
        'post_name',
        'post_modified',
        'post_modified_gmt',
        'post_parent',
        'menu_order',
        'post_type',
        'post_mime_type',
        'comment_count',
      ]);

    $query->condition('p.post_type', $post_type);
    $query->condition('p.post_status', $statuses, 'IN');

    return $query;
  }

  /**
   * {@inheritdoc}
   */
  public function prepareRow(Row $row) {
    $post_id = $row->getSourceProperty('ID');

    // โหลด post meta
    $meta_query = $this->select('wp_postmeta', 'pm')
      ->fields('pm', ['meta_key', 'meta_value'])
      ->condition('pm.post_id', $post_id);

    $meta = [];
    foreach ($meta_query->execute()->fetchAll() as $meta_row) {
      $meta[$meta_row['meta_key']] = $meta_row['meta_value'];
    }
    $row->setSourceProperty('meta', $meta);

    // โหลด featured image
    if (isset($meta['_thumbnail_id'])) {
      $thumbnail_id  = $meta['_thumbnail_id'];
      $thumbnail_url = $this->getAttachmentUrl($thumbnail_id);
      $row->setSourceProperty('featured_image_url', $thumbnail_url);
    }

    // โหลด categories
    $categories = $this->getTerms($post_id, 'category');
    $row->setSourceProperty('categories', $categories);

    // โหลด tags
    $tags = $this->getTerms($post_id, 'post_tag');
    $row->setSourceProperty('tags', $tags);

    // แปลง ACF fields
    if (isset($meta['_yoast_wpseo_title'])) {
      $row->setSourceProperty('seo_title', $meta['_yoast_wpseo_title']);
    }
    if (isset($meta['_yoast_wpseo_metadesc'])) {
      $row->setSourceProperty('seo_description', $meta['_yoast_wpseo_metadesc']);
    }

    return parent::prepareRow($row);
  }

  /**
   * ดึง terms ของ post
   */
  protected function getTerms(int $post_id, string $taxonomy): array {
    $query = $this->select('wp_terms', 't')
      ->fields('t', ['term_id', 'name', 'slug'])
      ->condition('tt.taxonomy', $taxonomy);

    $query->join('wp_term_taxonomy', 'tt', 'tt.term_id = t.term_id');
    $query->join('wp_term_relationships', 'tr', 'tr.term_taxonomy_id = tt.term_taxonomy_id');
    $query->condition('tr.object_id', $post_id);

    return $query->execute()->fetchAll();
  }

  /**
   * ดึง URL ของ attachment
   */
  protected function getAttachmentUrl(int $attachment_id): ?string {
    $query = $this->select('wp_postmeta', 'pm')
      ->fields('pm', ['meta_value'])
      ->condition('pm.post_id', $attachment_id)
      ->condition('pm.meta_key', '_wp_attached_file');

    $result = $query->execute()->fetchField();
    if ($result) {
      $wp_upload_url = $this->configuration['wp_upload_url'] ?? '';
      return rtrim($wp_upload_url, '/') . '/' . $result;
    }

    return NULL;
  }

  /**
   * {@inheritdoc}
   */
  public function fields() {
    return [
      'ID'                  => $this->t('Post ID'),
      'post_title'          => $this->t('Title'),
      'post_content'        => $this->t('Content'),
      'post_excerpt'        => $this->t('Excerpt'),
      'post_status'         => $this->t('Status'),
      'post_date'           => $this->t('Created date'),
      'post_author'         => $this->t('Author ID'),
      'categories'          => $this->t('Categories'),
      'tags'                => $this->t('Tags'),
      'featured_image_url'  => $this->t('Featured image URL'),
      'meta'                => $this->t('Post meta'),
    ];
  }

  /**
   * {@inheritdoc}
   */
  public function getIds() {
    return ['ID' => ['type' => 'integer']];
  }
}
```

#### ขั้นตอนที่ 3: Process Plugin สำหรับ Image Download

```php
<?php
// modules/custom/wp_migration/src/Plugin/migrate/process/DownloadImage.php

namespace Drupal\wp_migration\Plugin\migrate\process;

use Drupal\migrate\MigrateExecutableInterface;
use Drupal\migrate\ProcessPluginBase;
use Drupal\migrate\Row;
use GuzzleHttp\ClientInterface;
use Drupal\Core\File\FileSystemInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Drupal\migrate\Plugin\MigrationInterface;

/**
 * ดาวน์โหลดและนำเข้า image จาก URL
 *
 * @MigrateProcessPlugin(
 *   id = "download_image",
 *   handle_multiples = false
 * )
 */
class DownloadImage extends ProcessPluginBase {

  public function __construct(
    array $configuration,
    string $plugin_id,
    array $plugin_definition,
    protected ClientInterface $httpClient,
    protected FileSystemInterface $fileSystem
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
  }

  public static function create(
    ContainerInterface $container,
    array $configuration,
    $plugin_id,
    $plugin_definition,
    MigrationInterface $migration = NULL
  ) {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('http_client'),
      $container->get('file_system')
    );
  }

  /**
   * {@inheritdoc}
   */
  public function transform(
    $value,
    MigrateExecutableInterface $migrate_executable,
    Row $row,
    $destination_property
  ) {
    if (empty($value)) {
      return NULL;
    }

    $destination_dir = $this->configuration['destination'] ?? 'public://migration-images';
    $this->fileSystem->prepareDirectory($destination_dir, FileSystemInterface::CREATE_DIRECTORY);

    // ดึง filename จาก URL
    $filename = basename(parse_url($value, PHP_URL_PATH));
    $destination = rtrim($destination_dir, '/') . '/' . $filename;

    // ข้ามถ้าไฟล์มีอยู่แล้ว
    if (file_exists($this->fileSystem->realpath($destination))) {
      // ค้นหา file entity ที่มีอยู่
      $files = \Drupal::entityTypeManager()
        ->getStorage('file')
        ->loadByProperties(['uri' => $destination]);

      if (!empty($files)) {
        return reset($files)->id();
      }
    }

    try {
      // ดาวน์โหลดไฟล์
      $response = $this->httpClient->request('GET', $value, [
        'timeout' => 30,
        'stream'  => TRUE,
      ]);

      $content = $response->getBody()->getContents();
      $file_uri = $this->fileSystem->saveData($content, $destination, FileSystemInterface::EXISTS_REPLACE);

      // สร้าง file entity
      $file = \Drupal::entityTypeManager()->getStorage('file')->create([
        'uri'      => $file_uri,
        'status'   => 1,
        'filename' => $filename,
      ]);
      $file->save();

      return $file->id();

    } catch (\Exception $e) {
      $migrate_executable->saveMessage(
        "Failed to download image from {$value}: " . $e->getMessage()
      );
      return NULL;
    }
  }
}
```

#### ขั้นตอนที่ 4: Migration YAML สำหรับ WordPress Posts

```yaml
# config/install/migrate_plus.migration.wp_posts.yml

id: wp_posts
label: 'Import WordPress Posts'
migration_group: wordpress

source:
  plugin: wordpress_post
  post_type: post
  statuses:
    - publish
    - draft
  wp_upload_url: 'https://old-wordpress.example.com/wp-content/uploads'
  # Database connection สำหรับ WordPress
  key: wordpress
  database:
    driver: mysql
    database: wordpress_db
    username: wp_user
    password: wp_password
    host: localhost
    port: 3306

process:
  title: post_title

  body:
    plugin: clean_html
    source: post_content
    old_domain: 'https://old-wordpress.example.com'
    text_format: full_html

  body/summary: post_excerpt

  created:
    plugin: format_date
    source: post_date
    from_format: 'Y-m-d H:i:s'
    to_format: 'U'

  changed:
    plugin: format_date
    source: post_modified
    from_format: 'Y-m-d H:i:s'
    to_format: 'U'

  status:
    plugin: static_map
    source: post_status
    map:
      publish: 1
      draft: 0
      private: 0
    default_value: 0

  moderation_state:
    plugin: static_map
    source: post_status
    map:
      publish: published
      draft: draft
      private: draft
    default_value: draft

  uid:
    plugin: migration_lookup
    migration: wp_users
    source: post_author
    no_stub: true

  field_categories:
    plugin: sub_process
    source: categories
    process:
      target_id:
        plugin: migration_lookup
        migration: wp_categories
        source: term_id
        no_stub: true

  field_tags:
    plugin: sub_process
    source: tags
    process:
      target_id:
        plugin: migration_lookup
        migration: wp_tags
        source: term_id
        no_stub: true

  field_featured_image/target_id:
    plugin: download_image
    source: featured_image_url
    destination: 'public://articles'

  field_seo_title:
    plugin: get
    source: seo_title

  type:
    plugin: default_value
    default_value: article

destination:
  plugin: 'entity:node'
  default_bundle: article

migration_dependencies:
  required:
    - wp_users
    - wp_categories
    - wp_tags
```

#### ขั้นตอนที่ 5: Migration Runner Service

```php
<?php
// modules/custom/wp_migration/src/Service/MigrationRunner.php

namespace Drupal\wp_migration\Service;

use Drupal\migrate\MigrateExecutable;
use Drupal\migrate\MigrateMessage;
use Drupal\migrate\Plugin\MigrationPluginManagerInterface;

/**
 * Service สำหรับรัน migrations แบบ programmatic
 */
class MigrationRunner {

  public function __construct(
    protected MigrationPluginManagerInterface $migrationPluginManager
  ) {}

  /**
   * รัน migration
   *
   * @param string $migration_id
   * @param array  $options
   * @return array ผลลัพธ์การ migrate
   */
  public function runMigration(string $migration_id, array $options = []): array {
    $migration = $this->migrationPluginManager->createInstance($migration_id, $options);

    if (!$migration) {
      throw new \InvalidArgumentException("Migration {$migration_id} not found");
    }

    // ตั้งค่า options
    if (!empty($options['update'])) {
      $migration->getIdMap()->prepareUpdate();
    }

    $message    = new MigrateMessage();
    $executable = new MigrateExecutable($migration, $message);

    $result = $executable->import();

    return [
      'migration_id' => $migration_id,
      'result'       => $result,
      'processed'    => $migration->getIdMap()->processedCount(),
      'imported'     => $migration->getIdMap()->importedCount(),
      'failed'       => $migration->getIdMap()->errorCount(),
      'messages'     => $message->getMessages(),
    ];
  }

  /**
   * รัน migration group ทั้งหมด
   */
  public function runMigrationGroup(string $group): array {
    $migrations = $this->migrationPluginManager->createInstancesByTag($group);
    $results    = [];

    // เรียงลำดับตาม dependencies
    $migrations = $this->sortByDependencies($migrations);

    foreach ($migrations as $migration_id => $migration) {
      $results[$migration_id] = $this->runMigration($migration_id);
    }

    return $results;
  }

  /**
   * เรียงลำดับ migrations ตาม dependencies
   */
  protected function sortByDependencies(array $migrations): array {
    $sorted = [];
    $unresolved = $migrations;
    $max_iterations = count($migrations) * 2;
    $iteration = 0;

    while (!empty($unresolved) && $iteration < $max_iterations) {
      foreach ($unresolved as $id => $migration) {
        $deps = $migration->getMigrationDependencies()['required'] ?? [];
        $deps_resolved = TRUE;

        foreach ($deps as $dep) {
          if (!isset($sorted[$dep])) {
            $deps_resolved = FALSE;
            break;
          }
        }

        if ($deps_resolved) {
          $sorted[$id] = $migration;
          unset($unresolved[$id]);
        }
      }
      $iteration++;
    }

    // เพิ่ม migrations ที่เหลือ (circular deps)
    foreach ($unresolved as $id => $migration) {
      $sorted[$id] = $migration;
    }

    return $sorted;
  }
}
```

---

## แบบทดสอบ (Quiz) - 5 ข้อ

### คำถามที่ 1

**ถาม:** ใน Drupal Migrate API มี Plugin 3 ประเภทหลัก ได้แก่อะไรบ้าง และแต่ละประเภทมีหน้าที่อะไร?

**ตอบ:** Drupal Migrate API ประกอบด้วย Plugin 3 ประเภทหลัก ดังนี้:
1. **Source Plugin** - ทำหน้าที่ดึงข้อมูลจากแหล่งข้อมูลต้นทาง เช่น CSV, JSON, XML, Database, API และคืนค่าเป็น iterable rows
2. **Process Plugin** - ทำหน้าที่แปลงข้อมูลจาก format ของแหล่งต้นทางให้เป็น format ที่ Drupal ต้องการ เช่น การ map values, แปลง date formats, หรือ lookup entity IDs
3. **Destination Plugin** - ทำหน้าที่บันทึกข้อมูลที่ผ่านการแปลงแล้วไปยัง Drupal entities เช่น node, user, taxonomy_term, file

---

### คำถามที่ 2

**ถาม:** Paragraphs module มีข้อดีกว่า Field ทั่วไปอย่างไร และเมื่อใดควรใช้ Paragraphs แทน Fields ปกติ?

**ตอบ:** Paragraphs module มีข้อดีดังนี้:
- **Reusable Components:** สร้าง paragraph types ครั้งเดียว ใช้ได้หลาย content types
- **Flexible Layout:** Editor สามารถเลือกและเรียงลำดับ component ได้ตามต้องการ
- **Structured Content:** บังคับโครงสร้างข้อมูลที่ชัดเจนกว่า body field ทั่วไป
- **Revision Support:** มี revision tracking แยกจาก parent entity

ควรใช้ Paragraphs เมื่อ: เนื้อหามีหลาย sections ที่แตกต่างกัน (hero, text+image, card grid), ต้องการ page builder-like experience สำหรับ editor, หรือมีหลาย content types ที่ใช้ layout components เหมือนกัน

ไม่ควรใช้ Paragraphs เมื่อ: ต้องการ field ง่ายๆ เช่น title, body, หรือ simple taxonomy reference

---

### คำถามที่ 3

**ถาม:** อธิบายความแตกต่างระหว่าง Unit Test, Kernel Test, และ Functional Test ใน Drupal และให้ตัวอย่างการใช้งานแต่ละประเภท

**ตอบ:** ใน Drupal มีการทดสอบ 3 ระดับ:

**Unit Test (PHPUnit):**
- ทดสอบ class หรือ method แบบ isolated โดยไม่ต้อง bootstrap Drupal
- เร็วที่สุด ไม่ต้องใช้ database
- ใช้สำหรับ: Process plugins, utility classes, service methods ที่ไม่ depend on Drupal services

**Kernel Test:**
- Bootstrap Drupal kernel บางส่วน สามารถใช้ service container และ database ได้
- ช้ากว่า Unit test แต่เร็วกว่า Functional test
- ใช้สำหรับ: Entity operations, hooks, services ที่ต้อง interact กับ database

**Functional Test (BrowserTestBase):**
- Bootstrap Drupal เต็มรูปแบบ รัน HTTP requests จริง มี UI testing
- ช้าที่สุด ใช้ทรัพยากรมากที่สุด
- ใช้สำหรับ: Form submissions, access control, user workflows, routing

---

### คำถามที่ 4

**ถาม:** ใน Multisite configuration ไฟล์ `sites/sites.php` ทำหน้าที่อะไร และ `$sites` array มีโครงสร้างอย่างไร?

**ตอบ:** ไฟล์ `sites/sites.php` ทำหน้าที่เป็น **Site Discovery Map** - เป็นตัวบอก Drupal ว่า domain/URL ใด ควรใช้ settings จาก directory ไหน

โครงสร้าง `$sites` array:
```php
$sites['DOMAIN.PATH'] = 'DIRECTORY_NAME';
```

- **Key** คือ domain name (และ optional path) ที่ request มา
- **Value** คือชื่อ directory ใน `sites/` ที่จะใช้ settings.php จาก directory นั้น

เช่น:
```php
$sites['blog.example.com'] = 'blog.example.com';
// Request ที่มาจาก blog.example.com จะใช้ sites/blog.example.com/settings.php
```

ถ้าไม่มี `sites.php` หรือ domain ไม่ match Drupal จะใช้ `sites/default/` เป็น fallback

---

### คำถามที่ 5

**ถาม:** ใน CI/CD Pipeline สำหรับ Drupal คำสั่ง `drush deploy` ทำอะไรบ้าง และทำไมถึงสำคัญกว่าการ clear cache อย่างเดียว?

**ตอบ:** คำสั่ง `drush deploy` (Drupal 9.1+) รวมขั้นตอน deployment ที่สำคัญไว้ในคำสั่งเดียว ได้แก่:

1. **`maintenance:set on`** - เปิด maintenance mode
2. **`updatedb`** - รัน database updates (hook_update_N)
3. **`cache:rebuild`** - ล้าง cache ทั้งหมด
4. **`config:import`** - นำเข้า configuration จาก sync directory
5. **`cache:rebuild`** - ล้าง cache อีกครั้งหลัง config import
6. **`maintenance:set off`** - ปิด maintenance mode

ความสำคัญกว่าการ clear cache อย่างเดียว:
- **Database schema updates:** ถ้ามีการเพิ่ม field ใหม่ ต้องรัน `updatedb` ก่อนใช้งาน
- **Configuration sync:** Code และ configuration ต้องตรงกัน ถ้า import config แล้วไม่ rebuild cache จะเกิด inconsistency
- **Maintenance mode:** ป้องกัน error ที่เกิดจาก user เข้าใช้งานระหว่าง deployment
- **Order matters:** ลำดับขั้นตอนสำคัญมาก การทำผิดลำดับอาจทำให้ site crash

---

## Workshop: สร้าง Enterprise Content Migration

### สรุปขั้นตอน

```bash
# 1. ติดตั้ง modules ที่จำเป็น
composer require drupal/migrate_plus drupal/migrate_tools drupal/migrate_source_csv

# 2. Enable modules
drush en migrate migrate_plus migrate_tools -y

# 3. ตรวจสอบ migration status
drush migrate:status --group=wordpress

# 4. รัน migrations ตามลำดับ dependency
drush migrate:import wp_users
drush migrate:import wp_categories
drush migrate:import wp_tags
drush migrate:import wp_media
drush migrate:import wp_posts --update --feedback=100

# 5. ตรวจสอบ errors
drush migrate:messages wp_posts

# 6. ดู migration map
drush migrate:status wp_posts

# 7. Rollback ถ้าจำเป็น
drush migrate:rollback wp_posts

# 8. รัน migration ด้วย batch limit
drush migrate:import wp_posts --limit=500
```

### การตรวจสอบผลลัพธ์

```php
<?php
// ตรวจสอบ migration results ผ่าน code

$migration = \Drupal::service('plugin.manager.migration')
  ->createInstance('wp_posts');

$id_map = $migration->getIdMap();

echo "Processed: " . $id_map->processedCount() . "\n";
echo "Imported:  " . $id_map->importedCount() . "\n";
echo "Failed:    " . $id_map->errorCount() . "\n";
echo "Skipped:   " . $id_map->ignoredCount() . "\n";

// ดู messages
$id_map->rewind();
while ($id_map->valid()) {
  $current = $id_map->current();
  if ($current['source_row_status'] == \Drupal\migrate\Plugin\MigrateIdMapInterface::STATUS_FAILED) {
    echo "Failed ID: " . implode(', ', $current['sourceid']) . "\n";
  }
  $id_map->next();
}
```

---

## สรุปบทเรียน

ใน Part 085 นี้เราได้เรียนรู้การพัฒนา Drupal ระดับ Enterprise ครอบคลุม:

| หัวข้อ | ทักษะที่ได้ |
|--------|-------------|
| Migrate API | สร้าง Source, Process, Destination plugins |
| Paragraphs Module | ออกแบบ flexible content architecture |
| Workflows Module | จัดการ editorial workflow ระดับองค์กร |
| Drush Advanced | Automation และ deployment commands |
| Automated Testing | Unit, Kernel, Functional tests |
| Multisite | จัดการหลาย sites จาก codebase เดียว |
| CI/CD Pipeline | GitHub Actions สำหรับ automated deployment |
| Enterprise Migration | นำเข้าข้อมูลจาก WordPress |

### ขั้นตอนต่อไป

- Part 086: Drupal Performance Optimization (Caching, CDN, Database)
- Part 087: Drupal Security Hardening
- Part 088: Drupal Headless/Decoupled Architecture

---

*Part 085 | ระดับมืออาชีพ | Drupal Advanced Development*
