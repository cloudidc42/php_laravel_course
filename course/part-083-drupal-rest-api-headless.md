# Part 083: Drupal REST API & Headless CMS

## ระดับ: มืออาชีพ | ขั้นตอนที่ 801-840

---

## วัตถุประสงค์การเรียนรู้

หลังจากศึกษา Part นี้แล้ว ผู้เรียนจะสามารถ:

1. ติดตั้งและกำหนดค่า Drupal REST API module ได้อย่างถูกต้อง
2. ใช้งาน JSON:API module ที่มาพร้อมกับ Drupal 8+ ได้
3. สร้าง Custom REST Resource Plugin สำหรับ endpoint เฉพาะทาง
4. ตั้งค่าระบบ Authentication ด้วย OAuth2, JWT และ API Key
5. พัฒนา Decoupled Drupal โดยเชื่อมต่อกับ Next.js และ Nuxt.js
6. ใช้ Subrequests module เพื่อรวม API call หลายๆ ครั้งเป็นครั้งเดียว
7. ติดตั้งและใช้งาน GraphQL module สำหรับ Drupal
8. สร้างโปรเจกต์ Decoupled Blog ด้วย Drupal + Next.js

---

## บทนำ: Headless CMS และ Drupal

### Headless CMS คืออะไร?

Headless CMS คือระบบจัดการเนื้อหาที่แยก **Backend (Content Repository)** ออกจาก **Frontend (Presentation Layer)** อย่างสมบูรณ์ ทำให้นักพัฒนาสามารถใช้ frontend framework ใดก็ได้ในการแสดงผลเนื้อหา

```
┌─────────────────────────────────────────────────────────────────┐
│                    Traditional CMS Architecture                  │
├─────────────────────────────────────────────────────────────────┤
│  Content Editor → Drupal Backend → Drupal Theme → Browser       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Headless CMS Architecture                     │
├─────────────────────────────────────────────────────────────────┤
│  Content Editor → Drupal Backend → REST/GraphQL API →           │
│                                     ├── Next.js (Web)           │
│                                     ├── React Native (Mobile)   │
│                                     ├── Vue.js (SPA)            │
│                                     └── Any Framework           │
└─────────────────────────────────────────────────────────────────┘
```

### ทำไมต้องใช้ Drupal เป็น Headless CMS?

Drupal มีข้อได้เปรียบหลายประการในฐานะ Headless CMS:

- **Content Modeling ที่ยืดหยุ่น**: Field API ช่วยสร้างโครงสร้างข้อมูลที่ซับซ้อน
- **Permission System ที่ละเอียด**: ควบคุมการเข้าถึงได้ในระดับ entity
- **Built-in JSON:API**: ตั้งแต่ Drupal 8.7+ รวม JSON:API ไว้ใน core
- **REST API ที่ขยายได้**: สร้าง custom endpoints ได้ง่าย
- **GraphQL Support**: ผ่าน contributed module
- **Enterprise-grade**: รองรับโปรเจกต์ขนาดใหญ่

---

## ขั้นตอนที่ 801: การติดตั้ง Drupal REST API Module

### 1.1 เปิดใช้งาน RESTful Web Services Module

Drupal มี REST API module มาใน core แต่ต้องเปิดใช้งานก่อน:

```bash
# ใช้ Drush เปิด module
drush en rest restui serialization hal basic_auth -y

# หรือผ่าน Composer แล้วเปิด
composer require drupal/restui
drush en restui -y
```

### 1.2 กำหนดค่า REST Resources ผ่าน UI

หลังจากติดตั้ง REST UI module แล้ว:

1. ไปที่ **Admin → Configuration → Web Services → REST**
2. เลือก Resource ที่ต้องการเปิดใช้งาน เช่น "Content"
3. กำหนดค่า:
   - **Granularity**: Resource หรือ Method
   - **Methods**: GET, POST, PATCH, DELETE
   - **Accepted request formats**: json, hal_json
   - **Authentication providers**: cookie, basic_auth

### 1.3 กำหนดค่าผ่าน YAML (สำหรับ Production)

```yaml
# config/sync/rest.resource.entity.node.yml
langcode: en
status: true
dependencies:
  module:
    - basic_auth
    - node
    - serialization
    - user
id: entity.node
plugin_id: 'entity:node'
granularity: resource
configuration:
  methods:
    - GET
    - POST
    - PATCH
    - DELETE
  formats:
    - json
    - hal+json
  authentication:
    - cookie
    - basic_auth
```

### 1.4 ทดสอบ REST API เบื้องต้น

```bash
# GET: ดึงข้อมูล node
curl -X GET \
  "https://your-drupal.com/node/1?_format=json" \
  -H "Accept: application/json"

# Response ตัวอย่าง:
# {
#   "nid": [{"value": 1}],
#   "uuid": [{"value": "abc123..."}],
#   "title": [{"value": "Hello World"}],
#   "body": [{"value": "...", "format": "basic_html", "summary": ""}],
#   ...
# }
```

### 1.5 สร้าง Node ผ่าน REST API

```bash
# POST: สร้าง node ใหม่
curl -X POST \
  "https://your-drupal.com/node?_format=json" \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: [token]" \
  -u "admin:password" \
  -d '{
    "type": [{"target_id": "article"}],
    "title": [{"value": "บทความทดสอบ"}],
    "body": [{"value": "เนื้อหาบทความ", "format": "basic_html"}]
  }'
```

---

## ขั้นตอนที่ 802: JSON:API Module (Built-in Drupal 8+)

### 2.1 JSON:API คืออะไร?

JSON:API เป็น specification มาตรฐานสำหรับ RESTful APIs ที่ใช้ JSON เป็น format โดยกำหนดวิธีการ:
- ดึงข้อมูล (fetching) พร้อม relationships
- กรองข้อมูล (filtering)
- เรียงลำดับ (sorting)
- แบ่งหน้า (pagination)
- รวม relationships ใน response เดียว (sparse fieldsets)

### 2.2 เปิดใช้งาน JSON:API

```bash
# JSON:API มาใน Drupal core ตั้งแต่ 8.7+
drush en jsonapi -y

# ติดตั้ง extras สำหรับความสามารถเพิ่มเติม
composer require drupal/jsonapi_extras
drush en jsonapi_extras -y
```

### 2.3 โครงสร้าง JSON:API Endpoints

```
# Base URL
GET /jsonapi

# Content Types
GET /jsonapi/node/article          # ดึง articles ทั้งหมด
GET /jsonapi/node/article/{uuid}   # ดึง article เดียว
GET /jsonapi/node/page             # ดึง pages ทั้งหมด

# Taxonomy
GET /jsonapi/taxonomy_term/tags    # ดึง tags ทั้งหมด

# Users
GET /jsonapi/user/user             # ดึง users ทั้งหมด

# Media
GET /jsonapi/media/image           # ดึง images ทั้งหมด

# Files
GET /jsonapi/file/file             # ดึง files ทั้งหมด
```

### 2.4 การใช้งาน Filtering

```bash
# กรองด้วย field เดียว
GET /jsonapi/node/article?filter[title]=Hello

# กรองด้วยหลาย field
GET /jsonapi/node/article?filter[title]=Hello&filter[status]=1

# กรองด้วย operator
GET /jsonapi/node/article?filter[title][operator]=CONTAINS&filter[title][value]=Drupal

# กรองด้วย relationship
GET /jsonapi/node/article?filter[field_tags.name]=PHP

# กรองแบบ complex (AND/OR)
GET /jsonapi/node/article?filter[and-group][group][conjunction]=AND&filter[title-filter][condition][path]=title&filter[title-filter][condition][value]=Drupal&filter[title-filter][condition][memberOf]=and-group&filter[status-filter][condition][path]=status&filter[status-filter][condition][value]=1&filter[status-filter][condition][memberOf]=and-group
```

### 2.5 การใช้งาน Includes (Relationships)

```bash
# รวม relationships ใน response
GET /jsonapi/node/article?include=field_image,field_tags,uid

# Response จะมี "included" section
{
  "data": [{
    "type": "node--article",
    "id": "abc123",
    "attributes": {...},
    "relationships": {
      "field_image": {
        "data": {"type": "file--file", "id": "img123"}
      }
    }
  }],
  "included": [
    {
      "type": "file--file",
      "id": "img123",
      "attributes": {
        "uri": {"url": "/sites/default/files/image.jpg"},
        "filename": "image.jpg"
      }
    }
  ]
}
```

### 2.6 การใช้งาน Sparse Fieldsets

```bash
# เลือกเฉพาะ fields ที่ต้องการ
GET /jsonapi/node/article?fields[node--article]=title,body,field_image

# ลด payload ขนาดลงอย่างมาก
```

### 2.7 การสร้าง Resource ผ่าน JSON:API

```bash
# POST: สร้าง article
curl -X POST \
  "https://your-drupal.com/jsonapi/node/article" \
  -H "Content-Type: application/vnd.api+json" \
  -H "Accept: application/vnd.api+json" \
  -H "X-CSRF-Token: [token]" \
  -u "admin:password" \
  -d '{
    "data": {
      "type": "node--article",
      "attributes": {
        "title": "บทความใหม่",
        "body": {
          "value": "<p>เนื้อหาบทความ</p>",
          "format": "basic_html"
        },
        "status": true
      },
      "relationships": {
        "field_tags": {
          "data": [
            {"type": "taxonomy_term--tags", "id": "tag-uuid-here"}
          ]
        }
      }
    }
  }'
```

### 2.8 JSON:API Extras Configuration

```php
<?php
// web/modules/custom/my_module/src/Plugin/jsonapi/FieldEnhancer/DateEnhancer.php

namespace Drupal\my_module\Plugin\jsonapi\FieldEnhancer;

use Drupal\jsonapi_extras\Plugin\ResourceFieldEnhancerBase;
use Shaper\Util\Context;

/**
 * แปลง Date field เป็น Thai format
 *
 * @ResourceFieldEnhancer(
 *   id = "thai_date",
 *   label = @Translation("Thai Date Format"),
 *   description = @Translation("แปลงวันที่เป็นรูปแบบไทย")
 * )
 */
class ThaiDateEnhancer extends ResourceFieldEnhancerBase {

  /**
   * {@inheritdoc}
   */
  public function prepareCache($data, Context $context) {
    return $data;
  }

  /**
   * {@inheritdoc}
   */
  protected function doUndoTransform($data, Context $context) {
    if (empty($data)) {
      return $data;
    }

    $timestamp = strtotime($data);
    $thaiMonths = [
      1 => 'มกราคม', 2 => 'กุมภาพันธ์', 3 => 'มีนาคม',
      4 => 'เมษายน', 5 => 'พฤษภาคม', 6 => 'มิถุนายน',
      7 => 'กรกฎาคม', 8 => 'สิงหาคม', 9 => 'กันยายน',
      10 => 'ตุลาคม', 11 => 'พฤศจิกายน', 12 => 'ธันวาคม',
    ];

    $day = date('j', $timestamp);
    $month = $thaiMonths[(int)date('n', $timestamp)];
    $year = date('Y', $timestamp) + 543; // แปลงเป็น พ.ศ.

    return "{$day} {$month} {$year}";
  }

  /**
   * {@inheritdoc}
   */
  protected function doTransform($value, Context $context) {
    return $value;
  }

  /**
   * {@inheritdoc}
   */
  public function getOutputJsonSchema() {
    return ['type' => 'string'];
  }

}
```

---

## ขั้นตอนที่ 803: Custom REST Resource Plugins

### 3.1 โครงสร้างของ Custom REST Plugin

```php
<?php
// web/modules/custom/my_api/src/Plugin/rest/resource/ArticleSearchResource.php

namespace Drupal\my_api\Plugin\rest\resource;

use Drupal\rest\Plugin\ResourceBase;
use Drupal\rest\ResourceResponse;
use Drupal\Core\Session\AccountProxyInterface;
use Psr\Log\LoggerInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Symfony\Component\HttpKernel\Exception\BadRequestHttpException;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
use Symfony\Component\HttpFoundation\Request;

/**
 * REST Resource สำหรับการค้นหาบทความ
 *
 * @RestResource(
 *   id = "article_search",
 *   label = @Translation("Article Search Resource"),
 *   uri_paths = {
 *     "canonical" = "/api/v1/articles/search"
 *   }
 * )
 */
class ArticleSearchResource extends ResourceBase {

  /**
   * @var AccountProxyInterface
   */
  protected $currentUser;

  /**
   * @var Request
   */
  protected $currentRequest;

  /**
   * Constructor
   */
  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    array $serializer_formats,
    LoggerInterface $logger,
    AccountProxyInterface $current_user,
    Request $current_request
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition, $serializer_formats, $logger);
    $this->currentUser = $current_user;
    $this->currentRequest = $current_request;
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition) {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->getParameter('serializer.formats'),
      $container->get('logger.factory')->get('my_api'),
      $container->get('current_user'),
      $container->get('request_stack')->getCurrentRequest()
    );
  }

  /**
   * ดึงข้อมูลบทความ
   *
   * @return ResourceResponse
   */
  public function get() {
    // ตรวจสอบ permission
    if (!$this->currentUser->hasPermission('access content')) {
      throw new \Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException();
    }

    // รับ query parameters
    $keyword = $this->currentRequest->query->get('keyword', '');
    $page = (int) $this->currentRequest->query->get('page', 0);
    $limit = (int) $this->currentRequest->query->get('limit', 10);
    $tag = $this->currentRequest->query->get('tag', '');

    if (empty($keyword) && empty($tag)) {
      throw new BadRequestHttpException('กรุณาระบุ keyword หรือ tag ในการค้นหา');
    }

    // ค้นหาบทความ
    $articles = $this->searchArticles($keyword, $tag, $page, $limit);

    if (empty($articles['items'])) {
      return new ResourceResponse([
        'data' => [],
        'meta' => [
          'total' => 0,
          'page' => $page,
          'limit' => $limit,
          'message' => 'ไม่พบบทความที่ตรงกับเงื่อนไข',
        ],
      ], 200);
    }

    $response = new ResourceResponse([
      'data' => $articles['items'],
      'meta' => [
        'total' => $articles['total'],
        'page' => $page,
        'limit' => $limit,
        'pages' => ceil($articles['total'] / $limit),
      ],
    ], 200);

    // กำหนด Cache tags
    $response->addCacheableDependency(\Drupal\Core\Cache\CacheableMetadata::createFromRenderArray([
      '#cache' => [
        'tags' => ['node_list'],
        'contexts' => ['url.query_args'],
        'max-age' => 300, // 5 นาที
      ],
    ]));

    return $response;
  }

  /**
   * ค้นหาบทความจากฐานข้อมูล
   */
  private function searchArticles(string $keyword, string $tag, int $page, int $limit): array {
    $entity_type_manager = \Drupal::entityTypeManager();
    $query = $entity_type_manager->getStorage('node')->getQuery();

    $query->condition('type', 'article')
          ->condition('status', 1)
          ->accessCheck(TRUE);

    // ค้นหาด้วย keyword
    if (!empty($keyword)) {
      $orGroup = $query->orConditionGroup()
        ->condition('title', '%' . $keyword . '%', 'LIKE')
        ->condition('body.value', '%' . $keyword . '%', 'LIKE');
      $query->condition($orGroup);
    }

    // กรองด้วย tag
    if (!empty($tag)) {
      $query->condition('field_tags.entity.name', $tag);
    }

    // นับจำนวนทั้งหมด
    $countQuery = clone $query;
    $total = $countQuery->count()->execute();

    // Pagination
    $query->range($page * $limit, $limit);
    $query->sort('created', 'DESC');

    $nids = $query->execute();
    $nodes = $entity_type_manager->getStorage('node')->loadMultiple($nids);

    $items = [];
    foreach ($nodes as $node) {
      $items[] = $this->formatArticle($node);
    }

    return ['items' => $items, 'total' => (int) $total];
  }

  /**
   * จัดรูปแบบข้อมูลบทความ
   */
  private function formatArticle($node): array {
    $imageUrl = null;
    if ($node->hasField('field_image') && !$node->get('field_image')->isEmpty()) {
      $file = $node->get('field_image')->entity;
      if ($file) {
        $imageUrl = \Drupal::service('file_url_generator')
          ->generateAbsoluteString($file->getFileUri());
      }
    }

    $tags = [];
    if ($node->hasField('field_tags') && !$node->get('field_tags')->isEmpty()) {
      foreach ($node->get('field_tags') as $tag) {
        if ($tag->entity) {
          $tags[] = [
            'id' => $tag->entity->id(),
            'name' => $tag->entity->getName(),
          ];
        }
      }
    }

    return [
      'id' => $node->id(),
      'uuid' => $node->uuid(),
      'title' => $node->getTitle(),
      'summary' => $node->get('body')->summary ?: substr(strip_tags($node->get('body')->value), 0, 200),
      'image' => $imageUrl,
      'tags' => $tags,
      'author' => [
        'name' => $node->getOwner()->getDisplayName(),
        'uid' => $node->getOwnerId(),
      ],
      'created' => date('Y-m-d\TH:i:s', $node->getCreatedTime()),
      'url' => $node->toUrl('canonical', ['absolute' => TRUE])->toString(),
    ];
  }

}
```

### 3.2 Custom POST Endpoint

```php
<?php
// web/modules/custom/my_api/src/Plugin/rest/resource/ContactFormResource.php

namespace Drupal\my_api\Plugin\rest\resource;

use Drupal\rest\Plugin\ResourceBase;
use Drupal\rest\ModifiedResourceResponse;
use Drupal\rest\ResourceResponse;
use Symfony\Component\HttpKernel\Exception\BadRequestHttpException;
use Symfony\Component\HttpKernel\Exception\UnprocessableEntityHttpException;

/**
 * REST Resource สำหรับ Contact Form
 *
 * @RestResource(
 *   id = "contact_form_submit",
 *   label = @Translation("Contact Form Submit"),
 *   uri_paths = {
 *     "create" = "/api/v1/contact"
 *   }
 * )
 */
class ContactFormResource extends ResourceBase {

  /**
   * POST: ส่งข้อมูล contact form
   *
   * @param array $data
   *   ข้อมูลจาก request body
   *
   * @return ModifiedResourceResponse
   */
  public function post(array $data) {
    // Validate ข้อมูล
    $errors = $this->validateContactData($data);
    if (!empty($errors)) {
      throw new UnprocessableEntityHttpException(json_encode(['errors' => $errors]));
    }

    // ตรวจสอบ Honeypot (spam protection)
    if (!empty($data['website'])) {
      // Field นี้ถูกซ่อนจาก user จริง ถ้ามีค่ามาแสดงว่าเป็น bot
      $this->logger->warning('Honeypot triggered from IP: @ip', [
        '@ip' => \Drupal::request()->getClientIp(),
      ]);
      // Return 200 เพื่อหลอก bot
      return new ModifiedResourceResponse(['status' => 'ok'], 200);
    }

    // บันทึกข้อมูลลง database
    $connection = \Drupal::database();
    $connection->insert('contact_messages')->fields([
      'name' => $data['name'],
      'email' => $data['email'],
      'subject' => $data['subject'],
      'message' => $data['message'],
      'created' => \Drupal::time()->getRequestTime(),
      'ip_address' => \Drupal::request()->getClientIp(),
    ])->execute();

    // ส่ง Email แจ้งเตือน
    $this->sendNotificationEmail($data);

    return new ModifiedResourceResponse([
      'status' => 'success',
      'message' => 'ส่งข้อมูลเรียบร้อยแล้ว เราจะติดต่อกลับเร็วๆ นี้',
    ], 201);
  }

  /**
   * Validate ข้อมูล contact form
   */
  private function validateContactData(array $data): array {
    $errors = [];

    if (empty($data['name'])) {
      $errors[] = ['field' => 'name', 'message' => 'กรุณากรอกชื่อ'];
    } elseif (strlen($data['name']) > 100) {
      $errors[] = ['field' => 'name', 'message' => 'ชื่อต้องไม่เกิน 100 ตัวอักษร'];
    }

    if (empty($data['email'])) {
      $errors[] = ['field' => 'email', 'message' => 'กรุณากรอก Email'];
    } elseif (!filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
      $errors[] = ['field' => 'email', 'message' => 'รูปแบบ Email ไม่ถูกต้อง'];
    }

    if (empty($data['subject'])) {
      $errors[] = ['field' => 'subject', 'message' => 'กรุณากรอกหัวเรื่อง'];
    }

    if (empty($data['message'])) {
      $errors[] = ['field' => 'message', 'message' => 'กรุณากรอกข้อความ'];
    } elseif (strlen($data['message']) < 10) {
      $errors[] = ['field' => 'message', 'message' => 'ข้อความต้องมีอย่างน้อย 10 ตัวอักษร'];
    }

    return $errors;
  }

  /**
   * ส่ง Email แจ้งเตือน
   */
  private function sendNotificationEmail(array $data): void {
    $mailManager = \Drupal::service('plugin.manager.mail');
    $params = [
      'name' => $data['name'],
      'email' => $data['email'],
      'subject' => $data['subject'],
      'message' => $data['message'],
    ];

    $mailManager->mail(
      'my_api',
      'contact_notification',
      \Drupal::config('system.site')->get('mail'),
      \Drupal::currentUser()->getPreferredLangcode(),
      $params
    );
  }

}
```

### 3.3 Services สำหรับ REST Plugin

```yaml
# web/modules/custom/my_api/my_api.services.yml
services:
  my_api.article_service:
    class: Drupal\my_api\Service\ArticleService
    arguments:
      - '@entity_type.manager'
      - '@cache.default'
      - '@logger.factory'

  my_api.response_builder:
    class: Drupal\my_api\Service\ResponseBuilder
    arguments:
      - '@serializer'
```

```php
<?php
// web/modules/custom/my_api/src/Service/ArticleService.php

namespace Drupal\my_api\Service;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Cache\CacheBackendInterface;
use Psr\Log\LoggerInterface;

/**
 * Service สำหรับจัดการบทความ
 */
class ArticleService {

  protected EntityTypeManagerInterface $entityTypeManager;
  protected CacheBackendInterface $cache;
  protected LoggerInterface $logger;

  public function __construct(
    EntityTypeManagerInterface $entity_type_manager,
    CacheBackendInterface $cache,
    LoggerInterface $logger
  ) {
    $this->entityTypeManager = $entity_type_manager;
    $this->cache = $cache;
    $this->logger = $logger;
  }

  /**
   * ดึงบทความยอดนิยม
   *
   * @param int $limit จำนวนบทความ
   * @return array
   */
  public function getPopularArticles(int $limit = 5): array {
    $cacheKey = "my_api:popular_articles:{$limit}";

    // ตรวจสอบ Cache
    if ($cached = $this->cache->get($cacheKey)) {
      return $cached->data;
    }

    $query = $this->entityTypeManager->getStorage('node')->getQuery();
    $nids = $query
      ->condition('type', 'article')
      ->condition('status', 1)
      ->sort('field_view_count', 'DESC')
      ->range(0, $limit)
      ->accessCheck(TRUE)
      ->execute();

    $nodes = $this->entityTypeManager->getStorage('node')->loadMultiple($nids);
    $articles = [];

    foreach ($nodes as $node) {
      $articles[] = [
        'nid' => $node->id(),
        'uuid' => $node->uuid(),
        'title' => $node->getTitle(),
        'views' => $node->get('field_view_count')->value ?? 0,
        'created' => $node->getCreatedTime(),
      ];
    }

    // เก็บ Cache 10 นาที
    $this->cache->set($cacheKey, $articles, time() + 600, ['node_list']);

    return $articles;
  }

}
```

---

## ขั้นตอนที่ 804: Authentication (OAuth2, JWT, API Key)

### 4.1 OAuth2 Authentication

```bash
# ติดตั้ง OAuth2 module
composer require drupal/simple_oauth
drush en simple_oauth -y

# สร้าง keys
drush simple-oauth:generate-keys /path/to/keys
```

```php
<?php
// กำหนดค่า OAuth2 Client ผ่าน code

use Drupal\consumers\Entity\Consumer;

$consumer = Consumer::create([
  'label' => 'My Frontend App',
  'client_id' => 'my-frontend-app',
  'new_secret' => 'my-secret-key',
  'is_default' => FALSE,
  'redirect' => 'https://frontend.example.com/oauth/callback',
  'grant_types' => ['authorization_code', 'refresh_token'],
  'scopes' => ['authenticated'],
  'user_id' => 1,
]);
$consumer->save();
```

```bash
# ขอ Access Token ด้วย Password Grant
curl -X POST "https://your-drupal.com/oauth/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password&client_id=my-frontend-app&client_secret=my-secret-key&username=admin&password=admin123"

# Response:
# {
#   "token_type": "Bearer",
#   "expires_in": 300,
#   "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9...",
#   "refresh_token": "def502..."
# }

# ใช้ Access Token
curl -X GET "https://your-drupal.com/jsonapi/node/article" \
  -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9..."
```

### 4.2 JWT Authentication

```bash
# ติดตั้ง JWT module
composer require drupal/jwt
drush en jwt jwt_auth_consumer jwt_auth_issuer -y
```

```php
<?php
// web/modules/custom/my_api/src/Authentication/Provider/ApiKeyAuthProvider.php

namespace Drupal\my_api\Authentication\Provider;

use Drupal\Core\Authentication\AuthenticationProviderInterface;
use Drupal\Core\Database\Connection;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Symfony\Component\HttpFoundation\Request;

/**
 * Authentication Provider สำหรับ API Key
 */
class ApiKeyAuthProvider implements AuthenticationProviderInterface {

  protected Connection $database;
  protected EntityTypeManagerInterface $entityTypeManager;

  public function __construct(
    Connection $database,
    EntityTypeManagerInterface $entity_type_manager
  ) {
    $this->database = $database;
    $this->entityTypeManager = $entity_type_manager;
  }

  /**
   * ตรวจสอบว่า request นี้ใช้ API Key authentication
   *
   * {@inheritdoc}
   */
  public function applies(Request $request): bool {
    return $request->headers->has('X-API-Key');
  }

  /**
   * ตรวจสอบ API Key และคืน User object
   *
   * {@inheritdoc}
   */
  public function authenticate(Request $request) {
    $apiKey = $request->headers->get('X-API-Key');

    if (empty($apiKey)) {
      return NULL;
    }

    // ค้นหา API Key ในฐานข้อมูล
    $result = $this->database->select('api_keys', 'ak')
      ->fields('ak', ['uid', 'key_hash', 'expires', 'permissions'])
      ->condition('ak.key_hash', hash('sha256', $apiKey))
      ->condition('ak.status', 1)
      ->execute()
      ->fetchObject();

    if (!$result) {
      return NULL;
    }

    // ตรวจสอบวันหมดอายุ
    if ($result->expires && $result->expires < time()) {
      return NULL;
    }

    // อัปเดต last_used
    $this->database->update('api_keys')
      ->fields(['last_used' => time()])
      ->condition('key_hash', hash('sha256', $apiKey))
      ->execute();

    // คืน User account
    $user = $this->entityTypeManager->getStorage('user')->load($result->uid);

    return $user ?: NULL;
  }

}
```

### 4.3 Services สำหรับ Authentication Provider

```yaml
# web/modules/custom/my_api/my_api.services.yml
services:
  my_api.api_key_auth_provider:
    class: Drupal\my_api\Authentication\Provider\ApiKeyAuthProvider
    arguments:
      - '@database'
      - '@entity_type.manager'
    tags:
      - name: authentication_provider
        provider_id: api_key
        priority: 100
```

### 4.4 API Key Management Service

```php
<?php
// web/modules/custom/my_api/src/Service/ApiKeyService.php

namespace Drupal\my_api\Service;

use Drupal\Core\Database\Connection;

/**
 * Service สำหรับจัดการ API Keys
 */
class ApiKeyService {

  protected Connection $database;

  public function __construct(Connection $database) {
    $this->database = $database;
  }

  /**
   * สร้าง API Key ใหม่
   *
   * @param int $uid User ID
   * @param string $name ชื่อ Key
   * @param int $expires วันหมดอายุ (timestamp), 0 = ไม่หมดอายุ
   * @param array $permissions สิทธิ์ที่อนุญาต
   *
   * @return string API Key ที่สร้างใหม่
   */
  public function createApiKey(int $uid, string $name, int $expires = 0, array $permissions = []): string {
    // สร้าง random key
    $apiKey = 'dk_' . bin2hex(random_bytes(32)); // 67 ตัวอักษร

    $this->database->insert('api_keys')->fields([
      'uid' => $uid,
      'name' => $name,
      'key_hash' => hash('sha256', $apiKey),
      'key_prefix' => substr($apiKey, 0, 10) . '...', // สำหรับแสดงใน UI
      'expires' => $expires,
      'permissions' => json_encode($permissions),
      'status' => 1,
      'created' => time(),
      'last_used' => 0,
    ])->execute();

    return $apiKey; // คืน key เพียงครั้งเดียว
  }

  /**
   * เพิกถอน API Key
   *
   * @param int $uid User ID
   * @param string $keyPrefix Key prefix
   */
  public function revokeApiKey(int $uid, string $keyPrefix): bool {
    $result = $this->database->update('api_keys')
      ->fields(['status' => 0, 'revoked_at' => time()])
      ->condition('uid', $uid)
      ->condition('key_prefix', $keyPrefix)
      ->execute();

    return $result > 0;
  }

  /**
   * ดึง API Keys ทั้งหมดของ User
   *
   * @param int $uid User ID
   * @return array
   */
  public function getUserApiKeys(int $uid): array {
    return $this->database->select('api_keys', 'ak')
      ->fields('ak', ['id', 'name', 'key_prefix', 'expires', 'permissions', 'status', 'created', 'last_used'])
      ->condition('uid', $uid)
      ->orderBy('created', 'DESC')
      ->execute()
      ->fetchAll(\PDO::FETCH_ASSOC);
  }

}
```

### 4.5 Schema สำหรับ API Keys Table

```php
<?php
// web/modules/custom/my_api/my_api.install

/**
 * Implements hook_schema()
 */
function my_api_schema(): array {
  $schema['api_keys'] = [
    'description' => 'เก็บ API Keys',
    'fields' => [
      'id' => [
        'type' => 'serial',
        'not null' => TRUE,
      ],
      'uid' => [
        'type' => 'int',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'description' => 'User ID เจ้าของ Key',
      ],
      'name' => [
        'type' => 'varchar',
        'length' => 255,
        'not null' => TRUE,
        'description' => 'ชื่อ Key',
      ],
      'key_hash' => [
        'type' => 'varchar',
        'length' => 64,
        'not null' => TRUE,
        'description' => 'SHA256 hash ของ key',
      ],
      'key_prefix' => [
        'type' => 'varchar',
        'length' => 20,
        'not null' => TRUE,
        'description' => 'ส่วนแรกของ key สำหรับแสดงใน UI',
      ],
      'expires' => [
        'type' => 'int',
        'not null' => FALSE,
        'default' => 0,
        'description' => 'Timestamp วันหมดอายุ, 0 = ไม่หมดอายุ',
      ],
      'permissions' => [
        'type' => 'text',
        'description' => 'JSON array ของ permissions',
      ],
      'status' => [
        'type' => 'int',
        'size' => 'tiny',
        'default' => 1,
      ],
      'created' => [
        'type' => 'int',
        'not null' => TRUE,
      ],
      'last_used' => [
        'type' => 'int',
        'default' => 0,
      ],
      'revoked_at' => [
        'type' => 'int',
        'default' => 0,
      ],
    ],
    'primary key' => ['id'],
    'indexes' => [
      'key_hash' => ['key_hash'],
      'uid' => ['uid'],
    ],
  ];

  return $schema;
}
```

---

## ขั้นตอนที่ 805: Decoupled Drupal กับ Next.js

### 5.1 การตั้งค่า Next.js Project

```bash
# สร้าง Next.js project
npx create-next-app@latest drupal-frontend --typescript --tailwind --app
cd drupal-frontend

# ติดตั้ง dependencies
npm install next-drupal zod
```

### 5.2 การกำหนดค่า Environment Variables

```bash
# .env.local
NEXT_PUBLIC_DRUPAL_BASE_URL=https://your-drupal.com
DRUPAL_SITE_ID=your-site-id
DRUPAL_CLIENT_ID=your-oauth-client-id
DRUPAL_CLIENT_SECRET=your-oauth-client-secret
DRUPAL_FRONT_PAGE=/node
DRUPAL_PREVIEW_SECRET=your-preview-secret
REVALIDATE_SECRET=your-revalidate-secret
```

### 5.3 Drupal Client Configuration

```typescript
// lib/drupal.ts
import { DrupalClient } from "next-drupal";

export const drupal = new DrupalClient(
  process.env.NEXT_PUBLIC_DRUPAL_BASE_URL!,
  {
    auth: {
      clientId: process.env.DRUPAL_CLIENT_ID!,
      clientSecret: process.env.DRUPAL_CLIENT_SECRET!,
    },
    previewSecret: process.env.DRUPAL_PREVIEW_SECRET!,
    frontPage: process.env.DRUPAL_FRONT_PAGE!,
  }
);
```

### 5.4 TypeScript Types สำหรับ Drupal Content

```typescript
// types/drupal.ts

export interface DrupalNode {
  id: string;
  type: string;
  langcode: string;
  status: boolean;
  title: string;
  created: string;
  changed: string;
  path: {
    alias: string;
    pid: number;
    langcode: string;
  };
}

export interface DrupalArticle extends DrupalNode {
  body: {
    value: string;
    format: string;
    processed: string;
    summary: string;
  };
  field_image?: DrupalFile;
  field_tags?: DrupalTaxonomyTerm[];
  uid: {
    id: string;
    display_name: string;
  };
}

export interface DrupalFile {
  id: string;
  filename: string;
  uri: {
    value: string;
    url: string;
  };
  filemime: string;
  filesize: number;
  resourceIdObjMeta?: {
    alt?: string;
    title?: string;
    width?: number;
    height?: number;
  };
}

export interface DrupalTaxonomyTerm {
  id: string;
  name: string;
  description?: {
    value: string;
    processed: string;
  };
  path: {
    alias: string;
    pid: number;
    langcode: string;
  };
}

export interface DrupalApiResponse<T> {
  data: T[];
  meta: {
    count: number;
  };
  links: {
    self: { href: string };
    next?: { href: string };
    prev?: { href: string };
  };
}
```

### 5.5 การดึงข้อมูลจาก Drupal ใน Next.js App Router

```typescript
// app/blog/page.tsx
import { drupal } from "@/lib/drupal";
import { DrupalArticle } from "@/types/drupal";
import { ArticleCard } from "@/components/ArticleCard";

interface BlogPageProps {
  searchParams: { page?: string };
}

export default async function BlogPage({ searchParams }: BlogPageProps) {
  const page = parseInt(searchParams.page || "0");
  const pageSize = 10;

  // ดึง articles จาก Drupal JSON:API
  const articles = await drupal.getResourceCollection<DrupalArticle[]>(
    "node--article",
    {
      params: {
        "filter[status]": 1,
        "filter[langcode]": "th",
        "fields[node--article]": "title,body,field_image,field_tags,path,created,uid",
        include: "field_image,field_tags,uid",
        sort: "-created",
        "page[limit]": pageSize,
        "page[offset]": page * pageSize,
      },
    }
  );

  return (
    <main className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">บทความทั้งหมด</h1>
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {articles.map((article) => (
          <ArticleCard key={article.id} article={article} />
        ))}
      </div>

      {/* Pagination */}
      <div className="flex justify-center mt-8 gap-4">
        {page > 0 && (
          <a href={`/blog?page=${page - 1}`} className="btn btn-outline">
            ← หน้าก่อนหน้า
          </a>
        )}
        {articles.length === pageSize && (
          <a href={`/blog?page=${page + 1}`} className="btn btn-primary">
            หน้าถัดไป →
          </a>
        )}
      </div>
    </main>
  );
}

// ISR: Revalidate ทุก 60 วินาที
export const revalidate = 60;
```

### 5.6 Dynamic Route สำหรับ Article

```typescript
// app/blog/[slug]/page.tsx
import { drupal } from "@/lib/drupal";
import { DrupalArticle } from "@/types/drupal";
import Image from "next/image";
import { notFound } from "next/navigation";

interface ArticlePageProps {
  params: { slug: string };
}

export default async function ArticlePage({ params }: ArticlePageProps) {
  const node = await drupal.getResourceByPath<DrupalArticle>(
    `/blog/${params.slug}`,
    {
      params: {
        include: "field_image,field_tags,uid",
      },
    }
  );

  if (!node) {
    notFound();
  }

  const imageUrl = node.field_image?.uri.url
    ? `${process.env.NEXT_PUBLIC_DRUPAL_BASE_URL}${node.field_image.uri.url}`
    : null;

  return (
    <article className="container mx-auto px-4 py-8 max-w-4xl">
      <header className="mb-8">
        <h1 className="text-4xl font-bold mb-4">{node.title}</h1>
        <div className="flex items-center gap-4 text-gray-600">
          <span>โดย {node.uid?.display_name}</span>
          <span>•</span>
          <time dateTime={node.created}>
            {new Date(node.created).toLocaleDateString("th-TH", {
              year: "numeric",
              month: "long",
              day: "numeric",
            })}
          </time>
        </div>

        {node.field_tags && node.field_tags.length > 0 && (
          <div className="flex gap-2 mt-4">
            {node.field_tags.map((tag) => (
              <span
                key={tag.id}
                className="px-3 py-1 bg-blue-100 text-blue-800 rounded-full text-sm"
              >
                {tag.name}
              </span>
            ))}
          </div>
        )}
      </header>

      {imageUrl && (
        <div className="relative w-full h-96 mb-8 rounded-lg overflow-hidden">
          <Image
            src={imageUrl}
            alt={node.field_image?.resourceIdObjMeta?.alt || node.title}
            fill
            className="object-cover"
            priority
          />
        </div>
      )}

      <div
        className="prose prose-lg max-w-none"
        dangerouslySetInnerHTML={{ __html: node.body?.processed || "" }}
      />
    </article>
  );
}

// Static Site Generation
export async function generateStaticParams() {
  const articles = await drupal.getResourceCollection("node--article", {
    params: {
      "filter[status]": 1,
      "fields[node--article]": "path",
    },
  });

  return articles.map((article: any) => ({
    slug: article.path.alias.replace("/blog/", ""),
  }));
}

export const revalidate = 3600; // 1 ชั่วโมง
```

### 5.7 ISR Webhook สำหรับ On-Demand Revalidation

```typescript
// app/api/revalidate/route.ts
import { NextRequest, NextResponse } from "next/server";
import { revalidatePath } from "next/cache";

export async function POST(request: NextRequest) {
  // ตรวจสอบ secret
  const secret = request.headers.get("x-revalidate-secret");
  if (secret !== process.env.REVALIDATE_SECRET) {
    return NextResponse.json({ message: "Unauthorized" }, { status: 401 });
  }

  try {
    const body = await request.json();
    const { type, slug } = body;

    if (type === "node--article") {
      // Revalidate specific article
      revalidatePath(`/blog/${slug}`);
      // Revalidate blog listing
      revalidatePath("/blog");

      return NextResponse.json({
        revalidated: true,
        message: `Revalidated /blog/${slug}`,
      });
    }

    return NextResponse.json({
      revalidated: false,
      message: "Unknown content type",
    });
  } catch (error) {
    return NextResponse.json(
      { message: "Error revalidating" },
      { status: 500 }
    );
  }
}
```

### 5.8 Drupal Webhook Module

```php
<?php
// web/modules/custom/next_drupal_webhook/src/EventSubscriber/NodeSaveSubscriber.php

namespace Drupal\next_drupal_webhook\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Drupal\Core\Entity\EntityInterface;
use GuzzleHttp\ClientInterface;

/**
 * Event Subscriber สำหรับส่ง webhook เมื่อ Node ถูกบันทึก
 */
class NodeSaveSubscriber implements EventSubscriberInterface {

  protected ClientInterface $httpClient;

  public function __construct(ClientInterface $http_client) {
    $this->httpClient = $http_client;
  }

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents(): array {
    return [
      'hook_entity_update' => 'onEntityUpdate',
      'hook_entity_insert' => 'onEntityInsert',
    ];
  }

  /**
   * เมื่อ Node ถูกอัปเดต
   */
  public function onEntityUpdate(EntityInterface $entity): void {
    if ($entity->getEntityTypeId() === 'node') {
      $this->sendWebhook($entity, 'update');
    }
  }

  /**
   * เมื่อ Node ถูกสร้างใหม่
   */
  public function onEntityInsert(EntityInterface $entity): void {
    if ($entity->getEntityTypeId() === 'node') {
      $this->sendWebhook($entity, 'insert');
    }
  }

  /**
   * ส่ง Webhook ไปยัง Next.js
   */
  private function sendWebhook($node, string $action): void {
    $config = \Drupal::config('next_drupal_webhook.settings');
    $webhookUrl = $config->get('webhook_url');
    $secret = $config->get('webhook_secret');

    if (empty($webhookUrl)) {
      return;
    }

    $payload = [
      'type' => $node->bundle() ? 'node--' . $node->bundle() : 'node',
      'id' => $node->uuid(),
      'action' => $action,
      'slug' => $node->hasField('path') ? ltrim($node->get('path')->alias, '/') : null,
    ];

    try {
      $this->httpClient->post($webhookUrl, [
        'json' => $payload,
        'headers' => [
          'X-Revalidate-Secret' => $secret,
          'Content-Type' => 'application/json',
        ],
        'timeout' => 5,
      ]);
    } catch (\Exception $e) {
      \Drupal::logger('next_drupal_webhook')->error(
        'ส่ง webhook ล้มเหลว: @error', ['@error' => $e->getMessage()]
      );
    }
  }

}
```

---

## ขั้นตอนที่ 806: Decoupled Drupal กับ Nuxt.js

### 6.1 การตั้งค่า Nuxt.js Project

```bash
# สร้าง Nuxt.js project
npx nuxi@latest init drupal-nuxt
cd drupal-nuxt

# ติดตั้ง dependencies
npm install @nuxtjs/drupal-ce @vueuse/nuxt
```

### 6.2 Nuxt Config สำหรับ Drupal

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  devtools: { enabled: true },

  runtimeConfig: {
    drupalClientId: process.env.DRUPAL_CLIENT_ID,
    drupalClientSecret: process.env.DRUPAL_CLIENT_SECRET,
    public: {
      drupalBaseUrl: process.env.DRUPAL_BASE_URL || "https://your-drupal.com",
    },
  },

  modules: ["@nuxtjs/tailwindcss", "@vueuse/nuxt"],
});
```

### 6.3 Drupal Composable สำหรับ Nuxt

```typescript
// composables/useDrupal.ts
export const useDrupal = () => {
  const config = useRuntimeConfig();
  const baseUrl = config.public.drupalBaseUrl;

  const getHeaders = async (): Promise<HeadersInit> => {
    // ดึง token จาก OAuth2
    const tokenResponse = await $fetch<{access_token: string}>("/api/drupal-token");
    return {
      Authorization: `Bearer ${tokenResponse.access_token}`,
      Accept: "application/vnd.api+json",
    };
  };

  /**
   * ดึง articles
   */
  const getArticles = async (options: {
    page?: number;
    limit?: number;
    tag?: string;
  } = {}) => {
    const { page = 0, limit = 10, tag } = options;

    const params = new URLSearchParams({
      "filter[status]": "1",
      "fields[node--article]": "title,body,field_image,field_tags,path,created",
      include: "field_image,field_tags",
      sort: "-created",
      "page[limit]": limit.toString(),
      "page[offset]": (page * limit).toString(),
    });

    if (tag) {
      params.append("filter[field_tags.name]", tag);
    }

    const headers = await getHeaders();

    return await $fetch<{data: any[], meta: any}>(
      `${baseUrl}/jsonapi/node/article?${params.toString()}`,
      { headers }
    );
  };

  /**
   * ดึง article เดียว
   */
  const getArticle = async (uuid: string) => {
    const params = new URLSearchParams({
      include: "field_image,field_tags,uid",
    });

    const headers = await getHeaders();

    return await $fetch<{data: any}>(
      `${baseUrl}/jsonapi/node/article/${uuid}?${params.toString()}`,
      { headers }
    );
  };

  return {
    getArticles,
    getArticle,
  };
};
```

### 6.4 Vue Component สำหรับแสดงบทความ

```vue
<!-- components/ArticleList.vue -->
<template>
  <div>
    <div v-if="pending" class="flex justify-center py-12">
      <div class="loading loading-spinner loading-lg"></div>
    </div>

    <div v-else-if="error" class="alert alert-error">
      เกิดข้อผิดพลาด: {{ error.message }}
    </div>

    <div v-else>
      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        <ArticleCard
          v-for="article in articles"
          :key="article.id"
          :article="article"
        />
      </div>

      <!-- Infinite Scroll -->
      <div ref="loadMoreRef" class="py-4 text-center">
        <span v-if="loadingMore">กำลังโหลด...</span>
        <span v-else-if="!hasMore">โหลดครบแล้ว</span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useIntersectionObserver } from "@vueuse/core";

const { getArticles } = useDrupal();

const page = ref(0);
const allArticles = ref<any[]>([]);
const hasMore = ref(true);
const loadingMore = ref(false);
const loadMoreRef = ref<HTMLElement>();

const { data, pending, error } = await useAsyncData("articles", () =>
  getArticles({ page: 0, limit: 10 })
);

if (data.value) {
  allArticles.value = data.value.data;
}

const articles = computed(() => allArticles.value);

// Infinite scroll
const loadMore = async () => {
  if (loadingMore.value || !hasMore.value) return;

  loadingMore.value = true;
  page.value++;

  const result = await getArticles({ page: page.value, limit: 10 });

  if (result.data.length < 10) {
    hasMore.value = false;
  }

  allArticles.value.push(...result.data);
  loadingMore.value = false;
};

useIntersectionObserver(loadMoreRef, ([{ isIntersecting }]) => {
  if (isIntersecting) {
    loadMore();
  }
});
</script>
```

---

## ขั้นตอนที่ 807: Subrequests Module

### 7.1 Subrequests คืออะไร?

Subrequests เป็น module ที่ช่วยรวม API calls หลายๆ ครั้งเป็น batch request เดียว ลดจำนวน HTTP requests และปัญหา N+1 queries

```bash
# ติดตั้ง
composer require drupal/subrequests
drush en subrequests -y
```

### 7.2 การใช้งาน Subrequests

```bash
# ส่ง batch request
curl -X POST "https://your-drupal.com/subrequests?_format=json" \
  -H "Content-Type: application/json" \
  -d '[
    {
      "requestId": "req-articles",
      "action": "view",
      "uri": "/jsonapi/node/article?filter[status]=1&page[limit]=5",
      "headers": {
        "Accept": "application/vnd.api+json"
      }
    },
    {
      "requestId": "req-categories",
      "action": "view",
      "uri": "/jsonapi/taxonomy_term/category",
      "headers": {
        "Accept": "application/vnd.api+json"
      }
    },
    {
      "requestId": "req-site-config",
      "action": "view",
      "uri": "/jsonapi/block_content/basic/{{req-articles.body@$.data[0].id}}",
      "waitFor": ["req-articles"],
      "headers": {
        "Accept": "application/vnd.api+json"
      }
    }
  ]'
```

### 7.3 Subrequests ใน TypeScript

```typescript
// lib/subrequests.ts
interface SubrequestItem {
  requestId: string;
  action: "view" | "create" | "update" | "replace" | "delete" | "exists" | "discover";
  uri: string;
  headers?: Record<string, string>;
  body?: string;
  waitFor?: string[];
}

interface SubrequestResponse {
  [requestId: string]: {
    headers: Record<string, string[]>;
    body: string;
    status: number;
  };
}

export class DrupalSubrequests {
  private baseUrl: string;
  private accessToken: string;

  constructor(baseUrl: string, accessToken: string) {
    this.baseUrl = baseUrl;
    this.accessToken = accessToken;
  }

  async batch(requests: SubrequestItem[]): Promise<SubrequestResponse> {
    const response = await fetch(
      `${this.baseUrl}/subrequests?_format=json`,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${this.accessToken}`,
        },
        body: JSON.stringify(requests),
      }
    );

    if (!response.ok) {
      throw new Error(`Subrequests failed: ${response.statusText}`);
    }

    return response.json();
  }

  /**
   * ดึงข้อมูล Home Page ในครั้งเดียว
   */
  async getHomepageData() {
    const results = await this.batch([
      {
        requestId: "featured-articles",
        action: "view",
        uri: "/jsonapi/node/article?filter[field_featured]=1&page[limit]=3&include=field_image&fields[node--article]=title,path,field_image,created",
        headers: { Accept: "application/vnd.api+json" },
      },
      {
        requestId: "latest-articles",
        action: "view",
        uri: "/jsonapi/node/article?filter[status]=1&sort=-created&page[limit]=6&include=field_image&fields[node--article]=title,path,field_image,created",
        headers: { Accept: "application/vnd.api+json" },
      },
      {
        requestId: "categories",
        action: "view",
        uri: "/jsonapi/taxonomy_term/category?sort=weight&fields[taxonomy_term--category]=name,path,field_icon",
        headers: { Accept: "application/vnd.api+json" },
      },
    ]);

    return {
      featuredArticles: JSON.parse(results["featured-articles"].body),
      latestArticles: JSON.parse(results["latest-articles"].body),
      categories: JSON.parse(results["categories"].body),
    };
  }

}
```

---

## ขั้นตอนที่ 808: GraphQL Module สำหรับ Drupal

### 8.1 ติดตั้ง GraphQL Module

```bash
# ติดตั้ง GraphQL Compose (recommended สำหรับ Drupal 10)
composer require drupal/graphql_compose
drush en graphql graphql_compose graphql_compose_edges graphql_compose_extra -y
```

### 8.2 กำหนดค่า GraphQL Schema

```php
<?php
// web/modules/custom/my_graphql/src/Plugin/GraphQL/SchemaExtension/ArticleSchemaExtension.php

namespace Drupal\my_graphql\Plugin\GraphQL\SchemaExtension;

use Drupal\graphql\GraphQL\ResolverBuilder;
use Drupal\graphql\GraphQL\ResolverRegistryInterface;
use Drupal\graphql\Plugin\GraphQL\SchemaExtension\SdlSchemaExtensionPluginBase;

/**
 * GraphQL Schema Extension สำหรับ Article
 *
 * @SchemaExtension(
 *   id = "article_extension",
 *   name = "Article Extension",
 *   description = "เพิ่ม Article types เข้าไปใน GraphQL schema",
 *   schema = "default"
 * )
 */
class ArticleSchemaExtension extends SdlSchemaExtensionPluginBase {

  /**
   * {@inheritdoc}
   */
  public function registerResolvers(ResolverRegistryInterface $registry): void {
    $builder = new ResolverBuilder();

    // Query resolvers
    $this->addQueryResolvers($registry, $builder);

    // Type resolvers
    $this->addArticleResolvers($registry, $builder);
    $this->addTagResolvers($registry, $builder);
  }

  /**
   * เพิ่ม Query resolvers
   */
  private function addQueryResolvers(ResolverRegistryInterface $registry, ResolverBuilder $builder): void {
    // articles query
    $registry->addFieldResolver('Query', 'articles',
      $builder->produce('query_articles')
        ->map('filter', $builder->fromArgument('filter'))
        ->map('sort', $builder->fromArgument('sort'))
        ->map('page', $builder->fromArgument('page'))
    );

    // article query (by ID หรือ alias)
    $registry->addFieldResolver('Query', 'article',
      $builder->produce('article_by_id')
        ->map('id', $builder->fromArgument('id'))
    );
  }

  /**
   * เพิ่ม Article field resolvers
   */
  private function addArticleResolvers(ResolverRegistryInterface $registry, ResolverBuilder $builder): void {
    $registry->addFieldResolver('Article', 'id',
      $builder->produce('entity_id')->map('entity', $builder->fromParent())
    );

    $registry->addFieldResolver('Article', 'uuid',
      $builder->produce('entity_uuid')->map('entity', $builder->fromParent())
    );

    $registry->addFieldResolver('Article', 'title',
      $builder->produce('entity_label')->map('entity', $builder->fromParent())
    );

    $registry->addFieldResolver('Article', 'body',
      $builder->produce('property_path')
        ->map('type', $builder->fromValue('entity:node'))
        ->map('value', $builder->fromParent())
        ->map('path', $builder->fromValue('body.processed'))
    );

    $registry->addFieldResolver('Article', 'summary',
      $builder->produce('property_path')
        ->map('type', $builder->fromValue('entity:node'))
        ->map('value', $builder->fromParent())
        ->map('path', $builder->fromValue('body.summary'))
    );

    $registry->addFieldResolver('Article', 'created',
      $builder->produce('entity_created')
        ->map('entity', $builder->fromParent())
        ->map('format', $builder->fromValue('Y-m-d\TH:i:s+07:00'))
    );

    $registry->addFieldResolver('Article', 'image',
      $builder->produce('entity_reference')
        ->map('entity', $builder->fromParent())
        ->map('field', $builder->fromValue('field_image'))
    );

    $registry->addFieldResolver('Article', 'tags',
      $builder->produce('entity_reference')
        ->map('entity', $builder->fromParent())
        ->map('field', $builder->fromValue('field_tags'))
    );
  }

  /**
   * เพิ่ม Tag field resolvers
   */
  private function addTagResolvers(ResolverRegistryInterface $registry, ResolverBuilder $builder): void {
    $registry->addFieldResolver('Tag', 'id',
      $builder->produce('entity_id')->map('entity', $builder->fromParent())
    );

    $registry->addFieldResolver('Tag', 'name',
      $builder->produce('entity_label')->map('entity', $builder->fromParent())
    );
  }

}
```

### 8.3 GraphQL SDL Schema

```graphql
# web/modules/custom/my_graphql/graphql/article.extension.graphqls

extend type Query {
  """
  ดึง articles ทั้งหมด
  """
  articles(
    filter: ArticleFilter
    sort: ArticleSort
    page: PaginationInput
  ): ArticleConnection!

  """
  ดึง article เดียว
  """
  article(id: ID!): Article
}

type Article {
  id: ID!
  uuid: String!
  title: String!
  body: String
  summary: String
  created: String!
  changed: String!
  status: Boolean!
  path: String
  image: Image
  tags: [Tag!]
  author: User
  viewCount: Int
}

type Tag {
  id: ID!
  uuid: String!
  name: String!
  description: String
  path: String
  articleCount: Int
}

type Image {
  id: ID!
  url: String!
  alt: String
  title: String
  width: Int
  height: Int
}

type User {
  id: ID!
  name: String!
  displayName: String!
  picture: Image
  bio: String
}

type ArticleConnection {
  nodes: [Article!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  currentPage: Int!
  totalPages: Int!
}

input ArticleFilter {
  status: Boolean
  keyword: String
  tags: [String!]
  author: ID
  dateFrom: String
  dateTo: String
}

input ArticleSort {
  field: ArticleSortField
  direction: SortDirection
}

enum ArticleSortField {
  CREATED
  CHANGED
  TITLE
  VIEW_COUNT
}

enum SortDirection {
  ASC
  DESC
}

input PaginationInput {
  page: Int
  limit: Int
}
```

### 8.4 การ Query GraphQL จาก Next.js

```typescript
// lib/graphql-client.ts
import { GraphQLClient, gql } from "graphql-request";

const endpoint = `${process.env.NEXT_PUBLIC_DRUPAL_BASE_URL}/graphql`;

export const graphqlClient = new GraphQLClient(endpoint, {
  headers: {
    Authorization: `Bearer ${process.env.DRUPAL_GRAPHQL_TOKEN}`,
  },
});

// Queries
export const GET_ARTICLES = gql`
  query GetArticles($page: Int, $limit: Int, $tag: String) {
    articles(
      filter: { status: true }
      sort: { field: CREATED, direction: DESC }
      page: { page: $page, limit: $limit }
    ) {
      nodes {
        id
        uuid
        title
        summary
        created
        path
        image {
          url
          alt
        }
        tags {
          id
          name
        }
        author {
          displayName
        }
      }
      totalCount
      pageInfo {
        hasNextPage
        hasPreviousPage
        currentPage
        totalPages
      }
    }
  }
`;

export const GET_ARTICLE = gql`
  query GetArticle($id: ID!) {
    article(id: $id) {
      id
      uuid
      title
      body
      created
      changed
      path
      image {
        url
        alt
        width
        height
      }
      tags {
        id
        name
        path
      }
      author {
        displayName
        bio
        picture {
          url
        }
      }
    }
  }
`;
```

```typescript
// app/blog-graphql/page.tsx (ใช้ GraphQL แทน JSON:API)
import { graphqlClient, GET_ARTICLES } from "@/lib/graphql-client";

export default async function BlogGraphQLPage() {
  const data = await graphqlClient.request(GET_ARTICLES, {
    page: 0,
    limit: 10,
  });

  const { nodes: articles, totalCount, pageInfo } = data.articles;

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-4">บทความ (GraphQL)</h1>
      <p className="text-gray-500 mb-8">ทั้งหมด {totalCount} บทความ</p>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {articles.map((article: any) => (
          <div key={article.id} className="card shadow-lg">
            {article.image && (
              <img
                src={`${process.env.NEXT_PUBLIC_DRUPAL_BASE_URL}${article.image.url}`}
                alt={article.image.alt || article.title}
                className="w-full h-48 object-cover rounded-t-lg"
              />
            )}
            <div className="p-4">
              <h2 className="text-xl font-semibold mb-2">{article.title}</h2>
              <p className="text-gray-600 text-sm mb-4">{article.summary}</p>
              <a href={article.path} className="btn btn-sm btn-primary">
                อ่านต่อ
              </a>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## ขั้นตอนที่ 809: Custom JSON:API Filter

### 9.1 สร้าง Custom Filter Operator

```php
<?php
// web/modules/custom/my_api/src/Plugin/jsonapi/FieldEnhancer/ThaiSearchEnhancer.php

namespace Drupal\my_api\Plugin\jsonapi\FieldEnhancer;

use Drupal\jsonapi_extras\Plugin\ResourceFieldEnhancerBase;
use Shaper\Util\Context;

/**
 * Custom enhancer สำหรับการค้นหาภาษาไทย
 *
 * @ResourceFieldEnhancer(
 *   id = "thai_search_enhancer",
 *   label = @Translation("Thai Search Enhancer"),
 *   description = @Translation("เพิ่มความสามารถค้นหาภาษาไทย")
 * )
 */
class ThaiSearchEnhancer extends ResourceFieldEnhancerBase {

  /**
   * {@inheritdoc}
   */
  protected function doUndoTransform($data, Context $context) {
    if (empty($data)) {
      return $data;
    }

    // เพิ่ม metadata สำหรับ Thai search
    return [
      'original' => $data,
      'normalized' => $this->normalizeThai($data),
      'length' => mb_strlen($data, 'UTF-8'),
    ];
  }

  /**
   * {@inheritdoc}
   */
  protected function doTransform($value, Context $context) {
    if (is_array($value) && isset($value['original'])) {
      return $value['original'];
    }
    return $value;
  }

  /**
   * Normalize ข้อความภาษาไทย
   */
  private function normalizeThai(string $text): string {
    // ลบ HTML tags
    $text = strip_tags($text);
    // ลบ extra spaces
    $text = preg_replace('/\s+/', ' ', $text);
    // Convert to lowercase (สำหรับ English)
    $text = mb_strtolower($text, 'UTF-8');

    return trim($text);
  }

  /**
   * {@inheritdoc}
   */
  public function prepareCache($data, Context $context) {
    return $data;
  }

  /**
   * {@inheritdoc}
   */
  public function getOutputJsonSchema() {
    return [
      'type' => 'object',
      'properties' => [
        'original' => ['type' => 'string'],
        'normalized' => ['type' => 'string'],
        'length' => ['type' => 'integer'],
      ],
    ];
  }

}
```

### 9.2 Custom Filter Access

```php
<?php
// web/modules/custom/my_api/src/Access/ArticleApiAccess.php

namespace Drupal\my_api\Access;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Routing\Access\AccessInterface;
use Drupal\Core\Session\AccountInterface;
use Symfony\Component\HttpFoundation\Request;

/**
 * Access control สำหรับ Article API
 */
class ArticleApiAccess implements AccessInterface {

  /**
   * ตรวจสอบสิทธิ์การเข้าถึง
   */
  public function access(AccountInterface $account, Request $request): AccessResult {
    // ตรวจสอบ rate limiting
    if (!$this->checkRateLimit($request)) {
      return AccessResult::forbidden('Rate limit exceeded');
    }

    // ตรวจสอบ IP whitelist สำหรับ admin endpoints
    $path = $request->getPathInfo();
    if (str_starts_with($path, '/api/v1/admin/')) {
      if (!$this->isAllowedIp($request->getClientIp())) {
        return AccessResult::forbidden('IP not allowed');
      }
    }

    return AccessResult::allowed()->cachePerUser()->cachePerRequest();
  }

  /**
   * ตรวจสอบ Rate Limiting
   */
  private function checkRateLimit(Request $request): bool {
    $ip = $request->getClientIp();
    $cacheKey = "rate_limit:{$ip}";

    $cache = \Drupal::cache('bootstrap');
    $count = $cache->get($cacheKey)?->data ?? 0;

    if ($count >= 100) { // 100 requests ต่อนาที
      return FALSE;
    }

    $cache->set($cacheKey, $count + 1, time() + 60);
    return TRUE;
  }

  /**
   * ตรวจสอบ IP Whitelist
   */
  private function isAllowedIp(string $ip): bool {
    $allowedIps = \Drupal::config('my_api.settings')->get('admin_allowed_ips') ?? [];
    return in_array($ip, $allowedIps);
  }

}
```

---

## Workshop: สร้าง Decoupled Blog ด้วย Drupal + Next.js

### Workshop Overview

ใน Workshop นี้เราจะสร้าง Decoupled Blog ที่มีฟีเจอร์:
- Drupal 10 เป็น Backend CMS
- Next.js 14 เป็น Frontend
- OAuth2 Authentication
- ISR (Incremental Static Regeneration)
- Real-time Preview Mode
- Comment System ผ่าน REST API

### Step 1: ติดตั้ง Drupal 10

```bash
# สร้าง Drupal 10 project
composer create-project drupal/recommended-project:^10 my-drupal-blog
cd my-drupal-blog

# ติดตั้ง modules ที่จำเป็น
composer require drupal/simple_oauth drupal/jsonapi_extras drupal/subrequests drupal/graphql_compose drupal/restui drupal/cors drupal/next

# ติดตั้ง Drupal
drush site-install --db-url=mysql://user:pass@localhost/drupal_blog -y

# เปิด modules
drush en simple_oauth jsonapi_extras subrequests graphql graphql_compose cors next -y
```

### Step 2: สร้าง Content Types

```php
<?php
// web/modules/custom/blog_setup/blog_setup.install

/**
 * Implements hook_install().
 * สร้าง Content Types สำหรับ Blog
 */
function blog_setup_install(): void {
  // สร้าง Article content type
  $nodeType = \Drupal\node\Entity\NodeType::create([
    'type' => 'blog_post',
    'name' => 'Blog Post',
    'description' => 'บทความสำหรับ Blog',
  ]);
  $nodeType->save();

  // เพิ่ม fields
  _blog_setup_add_fields();
}

/**
 * เพิ่ม Fields ให้กับ Blog Post
 */
function _blog_setup_add_fields(): void {
  // Featured Image
  $fieldStorage = \Drupal\field\Entity\FieldStorageConfig::create([
    'field_name' => 'field_featured_image',
    'entity_type' => 'node',
    'type' => 'image',
    'cardinality' => 1,
  ]);
  $fieldStorage->save();

  \Drupal\field\Entity\FieldConfig::create([
    'field_storage' => $fieldStorage,
    'bundle' => 'blog_post',
    'label' => 'Featured Image',
    'required' => FALSE,
  ])->save();

  // Tags (taxonomy reference)
  $fieldStorage = \Drupal\field\Entity\FieldStorageConfig::create([
    'field_name' => 'field_blog_tags',
    'entity_type' => 'node',
    'type' => 'entity_reference',
    'settings' => ['target_type' => 'taxonomy_term'],
    'cardinality' => -1,
  ]);
  $fieldStorage->save();

  \Drupal\field\Entity\FieldConfig::create([
    'field_storage' => $fieldStorage,
    'bundle' => 'blog_post',
    'label' => 'Tags',
    'settings' => [
      'handler' => 'default:taxonomy_term',
      'handler_settings' => [
        'target_bundles' => ['blog_tags' => 'blog_tags'],
      ],
    ],
  ])->save();

  // Reading Time (computed field)
  $fieldStorage = \Drupal\field\Entity\FieldStorageConfig::create([
    'field_name' => 'field_reading_time',
    'entity_type' => 'node',
    'type' => 'integer',
    'cardinality' => 1,
  ]);
  $fieldStorage->save();

  \Drupal\field\Entity\FieldConfig::create([
    'field_storage' => $fieldStorage,
    'bundle' => 'blog_post',
    'label' => 'Reading Time (minutes)',
  ])->save();

  // View Count
  $fieldStorage = \Drupal\field\Entity\FieldStorageConfig::create([
    'field_name' => 'field_view_count',
    'entity_type' => 'node',
    'type' => 'integer',
    'cardinality' => 1,
  ]);
  $fieldStorage->save();

  \Drupal\field\Entity\FieldConfig::create([
    'field_storage' => $fieldStorage,
    'bundle' => 'blog_post',
    'label' => 'View Count',
    'default_value' => [['value' => 0]],
  ])->save();
}
```

### Step 3: สร้าง Comment System REST API

```php
<?php
// web/modules/custom/blog_api/src/Plugin/rest/resource/CommentResource.php

namespace Drupal\blog_api\Plugin\rest\resource;

use Drupal\rest\Plugin\ResourceBase;
use Drupal\rest\ModifiedResourceResponse;
use Drupal\rest\ResourceResponse;
use Symfony\Component\HttpFoundation\Request;

/**
 * REST Resource สำหรับ Comments
 *
 * @RestResource(
 *   id = "blog_comment",
 *   label = @Translation("Blog Comment Resource"),
 *   uri_paths = {
 *     "canonical" = "/api/v1/posts/{nid}/comments",
 *     "create" = "/api/v1/posts/{nid}/comments"
 *   }
 * )
 */
class CommentResource extends ResourceBase {

  /**
   * GET: ดึง comments ของ post
   */
  public function get(int $nid) {
    $node = \Drupal::entityTypeManager()->getStorage('node')->load($nid);

    if (!$node || $node->bundle() !== 'blog_post') {
      throw new \Symfony\Component\HttpKernel\Exception\NotFoundHttpException('ไม่พบบทความ');
    }

    // ดึง comments
    $commentStorage = \Drupal::entityTypeManager()->getStorage('comment');
    $query = $commentStorage->getQuery()
      ->condition('entity_id', $nid)
      ->condition('entity_type', 'node')
      ->condition('status', 1)
      ->sort('created', 'ASC')
      ->accessCheck(TRUE);

    $cids = $query->execute();
    $comments = $commentStorage->loadMultiple($cids);

    $data = [];
    foreach ($comments as $comment) {
      $data[] = [
        'id' => $comment->id(),
        'author' => [
          'name' => $comment->getAuthorName(),
          'uid' => $comment->getOwnerId(),
          'isAnonymous' => $comment->getOwnerId() === 0,
        ],
        'subject' => $comment->getSubject(),
        'body' => $comment->get('comment_body')->value,
        'created' => date('Y-m-d\TH:i:s', $comment->getCreatedTime()),
        'replies' => [], // TODO: nested comments
      ];
    }

    $response = new ResourceResponse(['data' => $data, 'total' => count($data)]);
    $response->addCacheableDependency($node);
    return $response;
  }

  /**
   * POST: เพิ่ม comment ใหม่
   */
  public function post(int $nid, array $data) {
    $node = \Drupal::entityTypeManager()->getStorage('node')->load($nid);

    if (!$node || $node->bundle() !== 'blog_post') {
      throw new \Symfony\Component\HttpKernel\Exception\NotFoundHttpException('ไม่พบบทความ');
    }

    // Validate
    if (empty($data['body'])) {
      throw new \Symfony\Component\HttpKernel\Exception\BadRequestHttpException('กรุณากรอกข้อความ comment');
    }

    // สร้าง Comment entity
    $comment = \Drupal\comment\Entity\Comment::create([
      'entity_id' => $nid,
      'entity_type' => 'node',
      'field_name' => 'field_comments',
      'comment_type' => 'comment',
      'subject' => $data['subject'] ?? '',
      'comment_body' => [
        'value' => strip_tags($data['body']),
        'format' => 'plain_text',
      ],
      'status' => 1, // Auto-approve (ในโปรเจกต์จริงอาจต้อง moderation)
      'uid' => \Drupal::currentUser()->id(),
    ]);
    $comment->save();

    return new ModifiedResourceResponse([
      'id' => $comment->id(),
      'message' => 'เพิ่ม comment เรียบร้อยแล้ว',
    ], 201);
  }

}
```

### Step 4: Next.js Blog Components

```typescript
// components/BlogPost.tsx
import Image from "next/image";
import { DrupalArticle } from "@/types/drupal";
import { CommentSection } from "./CommentSection";
import { ReadingProgress } from "./ReadingProgress";

interface BlogPostProps {
  post: DrupalArticle;
  drupalBaseUrl: string;
}

export function BlogPost({ post, drupalBaseUrl }: BlogPostProps) {
  const imageUrl = post.field_featured_image?.uri?.url
    ? `${drupalBaseUrl}${post.field_featured_image.uri.url}`
    : null;

  return (
    <article className="max-w-4xl mx-auto px-4">
      <ReadingProgress />

      <header className="mb-8">
        <div className="flex gap-2 mb-4">
          {post.field_blog_tags?.map((tag: any) => (
            <a
              key={tag.id}
              href={`/tag/${tag.path?.alias?.replace("/tag/", "")}`}
              className="badge badge-primary"
            >
              {tag.name}
            </a>
          ))}
        </div>

        <h1 className="text-4xl md:text-5xl font-bold mb-4 leading-tight">
          {post.title}
        </h1>

        <div className="flex items-center gap-6 text-gray-500 text-sm">
          <div className="flex items-center gap-2">
            <span>โดย</span>
            <strong className="text-gray-800">{post.uid?.display_name}</strong>
          </div>
          <time>
            {new Date(post.created).toLocaleDateString("th-TH", {
              year: "numeric",
              month: "long",
              day: "numeric",
            })}
          </time>
          {post.field_reading_time && (
            <span>{post.field_reading_time} นาทีในการอ่าน</span>
          )}
        </div>
      </header>

      {imageUrl && (
        <div className="relative w-full h-[400px] md:h-[500px] mb-8 rounded-2xl overflow-hidden">
          <Image
            src={imageUrl}
            alt={post.field_featured_image?.resourceIdObjMeta?.alt || post.title}
            fill
            className="object-cover"
            priority
          />
        </div>
      )}

      <div
        className="prose prose-lg max-w-none prose-headings:font-bold prose-a:text-primary"
        dangerouslySetInnerHTML={{ __html: post.body?.processed || "" }}
      />

      <CommentSection postId={post.id} drupalBaseUrl={drupalBaseUrl} />
    </article>
  );
}
```

```typescript
// components/CommentSection.tsx
"use client";

import { useState, useEffect } from "react";
import { useForm } from "react-hook-form";

interface Comment {
  id: string;
  author: {
    name: string;
    isAnonymous: boolean;
  };
  body: string;
  created: string;
}

interface CommentFormData {
  name: string;
  email: string;
  body: string;
}

interface CommentSectionProps {
  postId: string;
  drupalBaseUrl: string;
}

export function CommentSection({ postId, drupalBaseUrl }: CommentSectionProps) {
  const [comments, setComments] = useState<Comment[]>([]);
  const [loading, setLoading] = useState(true);
  const [submitting, setSubmitting] = useState(false);
  const [success, setSuccess] = useState(false);

  const { register, handleSubmit, reset, formState: { errors } } = useForm<CommentFormData>();

  useEffect(() => {
    fetchComments();
  }, [postId]);

  const fetchComments = async () => {
    try {
      const response = await fetch(
        `${drupalBaseUrl}/api/v1/posts/${postId}/comments?_format=json`
      );
      const data = await response.json();
      setComments(data.data || []);
    } catch (error) {
      console.error("Error fetching comments:", error);
    } finally {
      setLoading(false);
    }
  };

  const onSubmit = async (data: CommentFormData) => {
    setSubmitting(true);

    try {
      // ดึง CSRF token
      const tokenResponse = await fetch(`${drupalBaseUrl}/session/token`);
      const token = await tokenResponse.text();

      const response = await fetch(
        `${drupalBaseUrl}/api/v1/posts/${postId}/comments?_format=json`,
        {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            "X-CSRF-Token": token,
          },
          body: JSON.stringify({
            subject: `Comment จาก ${data.name}`,
            body: data.body,
          }),
        }
      );

      if (response.ok) {
        setSuccess(true);
        reset();
        await fetchComments();
        setTimeout(() => setSuccess(false), 3000);
      }
    } catch (error) {
      console.error("Error posting comment:", error);
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <section className="mt-16 pt-8 border-t">
      <h2 className="text-2xl font-bold mb-8">
        ความคิดเห็น ({comments.length})
      </h2>

      {/* Comment List */}
      {loading ? (
        <p>กำลังโหลด...</p>
      ) : comments.length === 0 ? (
        <p className="text-gray-500">ยังไม่มีความคิดเห็น เป็นคนแรกที่แสดงความคิดเห็น!</p>
      ) : (
        <div className="space-y-6 mb-8">
          {comments.map((comment) => (
            <div key={comment.id} className="bg-gray-50 rounded-lg p-4">
              <div className="flex justify-between mb-2">
                <strong>{comment.author.name}</strong>
                <time className="text-sm text-gray-500">
                  {new Date(comment.created).toLocaleDateString("th-TH")}
                </time>
              </div>
              <p className="text-gray-700">{comment.body}</p>
            </div>
          ))}
        </div>
      )}

      {/* Comment Form */}
      <div className="bg-white border rounded-lg p-6">
        <h3 className="text-lg font-semibold mb-4">แสดงความคิดเห็น</h3>

        {success && (
          <div className="alert alert-success mb-4">
            ส่งความคิดเห็นเรียบร้อยแล้ว ขอบคุณ!
          </div>
        )}

        <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
          <div className="grid grid-cols-2 gap-4">
            <div>
              <label className="block text-sm font-medium mb-1">ชื่อ *</label>
              <input
                {...register("name", { required: "กรุณากรอกชื่อ" })}
                className="input input-bordered w-full"
                placeholder="ชื่อของคุณ"
              />
              {errors.name && (
                <p className="text-red-500 text-sm mt-1">{errors.name.message}</p>
              )}
            </div>
            <div>
              <label className="block text-sm font-medium mb-1">Email</label>
              <input
                {...register("email", {
                  pattern: {
                    value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
                    message: "รูปแบบ Email ไม่ถูกต้อง",
                  },
                })}
                type="email"
                className="input input-bordered w-full"
                placeholder="email@example.com"
              />
              {errors.email && (
                <p className="text-red-500 text-sm mt-1">{errors.email.message}</p>
              )}
            </div>
          </div>

          <div>
            <label className="block text-sm font-medium mb-1">ความคิดเห็น *</label>
            <textarea
              {...register("body", {
                required: "กรุณากรอกความคิดเห็น",
                minLength: { value: 10, message: "ความคิดเห็นต้องมีอย่างน้อย 10 ตัวอักษร" },
              })}
              className="textarea textarea-bordered w-full"
              rows={4}
              placeholder="แสดงความคิดเห็นของคุณ..."
            />
            {errors.body && (
              <p className="text-red-500 text-sm mt-1">{errors.body.message}</p>
            )}
          </div>

          <button
            type="submit"
            disabled={submitting}
            className="btn btn-primary"
          >
            {submitting ? "กำลังส่ง..." : "ส่งความคิดเห็น"}
          </button>
        </form>
      </div>
    </section>
  );
}
```

### Step 5: CORS Configuration สำหรับ Drupal

```yaml
# web/sites/default/services.yml (เพิ่ม CORS settings)
parameters:
  cors.config:
    enabled: true
    allowedHeaders:
      - '*'
    allowedMethods:
      - GET
      - POST
      - PATCH
      - DELETE
      - OPTIONS
    allowedOrigins:
      - 'https://your-nextjs-app.vercel.app'
      - 'http://localhost:3000'
    exposedHeaders:
      - ''
    maxAge: false
    supportsCredentials: false
```

### Step 6: Preview Mode Configuration

```php
<?php
// web/modules/custom/blog_api/src/Plugin/rest/resource/PreviewResource.php

namespace Drupal\blog_api\Plugin\rest\resource;

use Drupal\rest\Plugin\ResourceBase;
use Drupal\rest\ResourceResponse;

/**
 * REST Resource สำหรับ Preview Mode
 *
 * @RestResource(
 *   id = "preview_token",
 *   label = @Translation("Preview Token Resource"),
 *   uri_paths = {
 *     "canonical" = "/api/v1/preview-token/{nid}"
 *   }
 * )
 */
class PreviewResource extends ResourceBase {

  /**
   * GET: ขอ Preview Token
   */
  public function get(int $nid) {
    // ตรวจสอบ permission
    if (!\Drupal::currentUser()->hasPermission('administer nodes')) {
      throw new \Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException();
    }

    $node = \Drupal::entityTypeManager()->getStorage('node')->load($nid);
    if (!$node) {
      throw new \Symfony\Component\HttpKernel\Exception\NotFoundHttpException();
    }

    // สร้าง token ที่มีอายุ 1 ชั่วโมง
    $token = bin2hex(random_bytes(32));
    $expires = time() + 3600;

    \Drupal::cache()->set(
      "preview_token:{$token}",
      ['nid' => $nid, 'expires' => $expires],
      $expires
    );

    $path = $node->toUrl('canonical')->toString();
    $previewUrl = \Drupal::config('blog_api.settings')->get('nextjs_url')
      . "/api/preview?secret={$token}&slug={$path}";

    return new ResourceResponse([
      'token' => $token,
      'expires' => $expires,
      'preview_url' => $previewUrl,
    ]);
  }

}
```

```typescript
// app/api/preview/route.ts (Next.js)
import { NextRequest, NextResponse } from "next/server";
import { draftMode } from "next/headers";
import { redirect } from "next/navigation";

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const secret = searchParams.get("secret");
  const slug = searchParams.get("slug");

  // ตรวจสอบ secret กับ Drupal
  const verifyResponse = await fetch(
    `${process.env.NEXT_PUBLIC_DRUPAL_BASE_URL}/api/v1/verify-preview`,
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${process.env.DRUPAL_PREVIEW_TOKEN}`,
      },
      body: JSON.stringify({ token: secret }),
    }
  );

  if (!verifyResponse.ok) {
    return NextResponse.json({ message: "Invalid preview token" }, { status: 401 });
  }

  // เปิด Draft Mode
  draftMode().enable();

  // Redirect ไปยัง path ที่ต้องการ preview
  redirect(slug || "/");
}
```

---

## แบบทดสอบ (Quiz)

**คำถาม 1:** JSON:API ใน Drupal 8+ แตกต่างจาก REST API module อย่างไร?

**ตอบ:** JSON:API ใน Drupal เป็น specification มาตรฐาน (JSON:API 1.0 spec) ที่มีฟีเจอร์ครบกว่า REST API module ประกอบด้วย:
- **Filtering**: รองรับ complex filters ด้วย AND/OR groups
- **Sparse Fieldsets**: เลือก fields ที่ต้องการในแต่ละ request
- **Includes**: ดึง relationship data พร้อมกันในครั้งเดียว
- **Sorting**: เรียงลำดับข้อมูลได้หลาย fields
- **Pagination**: รองรับ cursor-based pagination
- **Standard Response Format**: response format มาตรฐานที่ consistent

ส่วน REST API module ยืดหยุ่นกว่าในการสร้าง custom endpoints แต่ต้องเขียน code มากกว่า

---

**คำถาม 2:** อธิบาย Decoupled Drupal Architecture และข้อดีข้อเสีย

**ตอบ:** 

**Decoupled Drupal** คือการแยก Drupal ออกเป็น API server และใช้ frontend framework แยกต่างหาก

**ข้อดี:**
- เลือก frontend framework ได้อิสระ (React, Vue, Angular, etc.)
- Performance ดีขึ้นเพราะ frontend เป็น SPA/SSG
- Scaling แยกกันได้ระหว่าง backend และ frontend
- Developer Experience ดีขึ้นสำหรับ frontend developers
- รองรับ Multi-channel delivery (Web, Mobile, IoT)

**ข้อเสีย:**
- ซับซ้อนขึ้น ต้องจัดการ 2 application แยกกัน
- SEO ต้องการ SSR/SSG สำหรับ SPAs
- Authentication ซับซ้อนขึ้น
- ต้นทุนการ host เพิ่มขึ้น
- Content Preview mode ต้องตั้งค่าพิเศษ

---

**คำถาม 3:** Custom REST Resource Plugin ควรมีโครงสร้างอย่างไร?

**ตอบ:** Custom REST Resource Plugin ต้องมี:

1. **Annotation** ที่ระบุ:
   - `id`: unique identifier
   - `label`: ชื่อที่แสดง
   - `uri_paths`: URL paths สำหรับแต่ละ operation

2. **Methods** ที่ต้องการ:
   - `get()` สำหรับ GET requests
   - `post()` สำหรับ POST requests
   - `patch()` สำหรับ PATCH requests
   - `delete()` สำหรับ DELETE requests

3. **Dependency Injection** ผ่าน:
   - `__construct()` รับ services
   - `create()` static method สร้าง instance จาก container

4. **Response** ต้องคืน:
   - `ResourceResponse` สำหรับ GET/DELETE
   - `ModifiedResourceResponse` สำหรับ POST/PATCH (มี Location header)

5. **Cacheability Metadata** สำหรับ GET responses

---

**คำถาม 4:** Subrequests module แก้ปัญหาอะไรและใช้เมื่อไหร่?

**ตอบ:**

**ปัญหาที่แก้:**
- **Waterfall requests**: frontend ต้องทำ request หลายๆ ครั้ง รอแต่ละ request เสร็จก่อน
- **Over-fetching/Under-fetching**: ดึงข้อมูลมากเกินไปหรือน้อยเกินไป
- **N+1 Problem**: ต้องทำ request N ครั้งสำหรับ N items

**เมื่อไหรควรใช้:**
- หน้า Home page ที่ต้องการข้อมูลหลายส่วนพร้อมกัน
- Sidebar ที่ต้องการ categories, recent posts, featured posts
- Dashboard ที่มีหลาย widgets
- ลด initial load time ของ page

**ตัวอย่าง**: แทนที่จะทำ 5 requests แยกกัน (articles, tags, users, media, config) ทำได้ใน batch request เดียว ลด latency รวมจาก 5 × 100ms = 500ms เหลือ ~110ms

---

**คำถาม 5:** อธิบายการทำงานของ OAuth2 ใน Drupal และ Next.js integration

**ตอบ:**

**OAuth2 Flow ใน Drupal + Next.js:**

1. **Next.js Server** ส่ง request ขอ token จาก Drupal:
   ```
   POST /oauth/token
   grant_type=client_credentials
   client_id=nextjs-app
   client_secret=secret
   ```

2. **Drupal** ตรวจสอบ credentials และคืน Access Token (JWT):
   ```json
   {
     "token_type": "Bearer",
     "expires_in": 300,
     "access_token": "eyJ..."
   }
   ```

3. **Next.js** ใช้ token ใน API calls:
   ```
   Authorization: Bearer eyJ...
   ```

4. **Token Refresh**: เมื่อ token หมดอายุ ใช้ refresh token ขอ token ใหม่

**Security Best Practices:**
- เก็บ client_secret ใน environment variables เท่านั้น
- ไม่ส่ง token ไปยัง client-side (browser)
- ใช้ HTTPS เสมอ
- ตั้ง token expiry ให้สั้นพอสมควร (5-15 นาที)
- Rotate credentials เป็นประจำ

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **REST API Module**: การตั้งค่าและใช้งาน RESTful endpoints ใน Drupal
2. **JSON:API**: มาตรฐาน API ที่ทรงพลังพร้อม filtering, includes, sparse fieldsets
3. **Custom REST Plugins**: สร้าง custom endpoints ที่ตรงกับความต้องการ
4. **Authentication**: OAuth2, JWT, API Key สำหรับรักษาความปลอดภัย
5. **Decoupled Architecture**: การเชื่อมต่อ Drupal กับ Next.js และ Nuxt.js
6. **Subrequests**: การรวม API calls เพื่อลด latency
7. **GraphQL**: การใช้ graphql_compose module สำหรับ flexible queries
8. **Workshop**: สร้าง Decoupled Blog จริงๆ ด้วย Drupal + Next.js

### เทคนิคที่ควรจำ

- ใช้ **JSON:API** สำหรับ CRUD operations ทั่วไป - มีมาตรฐานชัดเจนและใช้งานง่าย
- ใช้ **Custom REST Plugin** เมื่อต้องการ business logic พิเศษ หรือ aggregate data
- ใช้ **GraphQL** เมื่อ frontend ต้องการ flexibility ในการ query
- ใช้ **Subrequests** เพื่อ optimize page load performance
- ตั้งค่า **Cache** ให้ถูกต้องเพื่อ performance ที่ดี
- ใช้ **ISR** ใน Next.js เพื่อให้ได้ทั้ง performance และ freshness

### ทรัพยากรเพิ่มเติม

- [Drupal JSON:API Documentation](https://www.drupal.org/docs/core-modules-and-themes/core-modules/jsonapi-module)
- [Next-Drupal Documentation](https://next-drupal.org/docs)
- [GraphQL Compose Module](https://www.drupal.org/project/graphql_compose)
- [Simple OAuth Module](https://www.drupal.org/project/simple_oauth)
- [Decoupled Drupal](https://www.drupal.org/docs/develop/decoupled-drupal)

---

*Part 083 | ระดับมืออาชีพ | Drupal REST API & Headless*
