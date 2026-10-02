# Part 084: Drupal Performance & Scaling

## ระดับ: มืออาชีพ | ขั้นตอนที่ 841-880

---

## วัตถุประสงค์การเรียนรู้

เมื่อจบบทเรียนนี้ ผู้เรียนจะสามารถ:

1. เข้าใจสถาปัตยกรรม Cache API ของ Drupal 10 และใช้งาน cache bins, tags, และ contexts ได้อย่างถูกต้อง
2. ติดตั้งและกำหนดค่า BigPipe module เพื่อเพิ่มความเร็วในการโหลดหน้าเว็บแบบ progressive rendering
3. ตั้งค่า Varnish และ Reverse Proxy caching สำหรับ Drupal production environment
4. ผสานรวม Redis กับ Drupal เพื่อจัดการ session และ cache ในสภาพแวดล้อมแบบ distributed
5. วิเคราะห์และปรับปรุง database queries โดยใช้ Query tag และ slow query log
6. ปรับแต่ง CSS/JS aggregation เพื่อลด HTTP requests
7. ออกแบบระบบ horizontal scaling พร้อม load balancer สำหรับ Drupal
8. ทำ performance audit และสร้างรายงานการปรับปรุงประสิทธิภาพ

---

## ทำไม Performance ถึงสำคัญสำหรับ Drupal?

ในโลกของการพัฒนาเว็บไซต์ระดับองค์กร Drupal ถูกใช้งานในเว็บไซต์ขนาดใหญ่ที่มีผู้เข้าชมหลักล้านคนต่อวัน การปรับแต่งประสิทธิภาพจึงไม่ใช่แค่ตัวเลือก แต่เป็นความจำเป็น

**ผลกระทบของ Poor Performance:**
- Google PageSpeed Score ต่ำ → SEO ranking ลดลง
- Time to First Byte (TTFB) สูง → User experience แย่ลง
- Server load สูง → ค่าใช้จ่าย infrastructure เพิ่มขึ้น
- Database bottleneck → Application timeout

**เครื่องมือวัด Performance:**
- Google PageSpeed Insights / Lighthouse
- WebPageTest.org
- New Relic / Datadog APM
- Drupal's built-in performance module
- Xdebug + KCacheGrind

---

## ขั้นตอนที่ 841-845: Drupal Cache API

### 841. ทำความเข้าใจ Cache System ของ Drupal

Drupal 10 มี Cache system ที่ซับซ้อนและยืดหยุ่นมาก ประกอบด้วย 3 องค์ประกอบหลัก:

1. **Cache Bins** - พื้นที่จัดเก็บ cache แยกตามประเภท
2. **Cache Tags** - ป้ายกำกับสำหรับ invalidate cache
3. **Cache Contexts** - บริบทที่กำหนดว่า cache นี้ใช้ได้กับใคร/อะไร

```
Cache Bins ที่มีใน Drupal:
- cache.default      → ทั่วไป
- cache.bootstrap    → bootstrap ระบบ
- cache.render       → HTML render cache
- cache.data         → module data
- cache.discovery    → plugin discovery
- cache.dynamic_page_cache → dynamic page cache
- cache.page         → full page cache
```

### 842. การใช้งาน Cache API พื้นฐาน

```php
<?php
// web/modules/custom/mymodule/src/Service/DataService.php

namespace Drupal\mymodule\Service;

use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Cache\CacheTagsInvalidatorInterface;

/**
 * Service สำหรับจัดการข้อมูลพร้อม caching
 */
class DataService {

  /**
   * @var \Drupal\Core\Cache\CacheBackendInterface
   */
  protected CacheBackendInterface $cache;

  /**
   * @var \Drupal\Core\Cache\CacheTagsInvalidatorInterface
   */
  protected CacheTagsInvalidatorInterface $cacheTagsInvalidator;

  public function __construct(
    CacheBackendInterface $cache,
    CacheTagsInvalidatorInterface $cacheTagsInvalidator
  ) {
    $this->cache = $cache;
    $this->cacheTagsInvalidator = $cacheTagsInvalidator;
  }

  /**
   * ดึงข้อมูลพร้อม cache
   */
  public function getExpensiveData(int $nodeId): array {
    $cid = 'mymodule:expensive_data:' . $nodeId;

    // ลองดึงจาก cache ก่อน
    if ($cached = $this->cache->get($cid)) {
      return $cached->data;
    }

    // คำนวณข้อมูลที่ใช้เวลานาน
    $data = $this->calculateExpensiveData($nodeId);

    // บันทึก cache พร้อม tags และ expiry
    $this->cache->set(
      $cid,
      $data,
      // หมดอายุใน 1 ชั่วโมง
      time() + 3600,
      // Cache tags - เมื่อ node นี้ถูกแก้ไข cache จะถูกลบ
      ['node:' . $nodeId, 'mymodule_data']
    );

    return $data;
  }

  /**
   * Invalidate cache ทั้งหมดที่เกี่ยวข้องกับ module
   */
  public function invalidateAllModuleCache(): void {
    $this->cacheTagsInvalidator->invalidateTags(['mymodule_data']);
  }

  /**
   * Invalidate cache สำหรับ node เฉพาะ
   */
  public function invalidateNodeCache(int $nodeId): void {
    $this->cacheTagsInvalidator->invalidateTags(['node:' . $nodeId]);
  }

  /**
   * จำลองการคำนวณที่ใช้เวลานาน
   */
  private function calculateExpensiveData(int $nodeId): array {
    // จำลอง expensive operation
    sleep(1);
    return [
      'node_id' => $nodeId,
      'computed_at' => date('Y-m-d H:i:s'),
      'result' => rand(1, 1000),
    ];
  }

}
```

### 843. Cache Contexts - การ Cache ตามบริบท

Cache Contexts ใช้สำหรับบอก Drupal ว่า cache version ไหนควรใช้กับ request ไหน

```php
<?php
// web/modules/custom/mymodule/src/Plugin/Block/UserSpecificBlock.php

namespace Drupal\mymodule\Plugin\Block;

use Drupal\Core\Block\BlockBase;
use Drupal\Core\Cache\Cache;

/**
 * @Block(
 *   id = "user_specific_block",
 *   admin_label = @Translation("User Specific Block"),
 * )
 */
class UserSpecificBlock extends BlockBase {

  /**
   * {@inheritdoc}
   */
  public function build(): array {
    $build = [
      '#theme' => 'mymodule_user_block',
      '#data' => $this->getUserData(),
    ];

    return $build;
  }

  /**
   * กำหนด cache metadata
   */
  public function getCacheContexts(): array {
    return Cache::mergeContexts(
      parent::getCacheContexts(),
      [
        // Cache แยกตาม user
        'user',
        // Cache แยกตาม URL
        'url',
        // Cache แยกตาม ภาษา
        'languages:language_interface',
        // Cache แยกตาม role
        'user.roles',
      ]
    );
  }

  /**
   * กำหนด cache tags
   */
  public function getCacheTags(): array {
    return Cache::mergeTags(
      parent::getCacheTags(),
      ['user:' . \Drupal::currentUser()->id()]
    );
  }

  /**
   * กำหนดอายุ cache (วินาที)
   * -1 = ไม่หมดอายุ (invalidate ด้วย tags เท่านั้น)
   *  0 = ไม่ cache เลย
   */
  public function getCacheMaxAge(): int {
    return 300; // 5 นาที
  }

  private function getUserData(): array {
    $user = \Drupal::currentUser();
    return [
      'name' => $user->getDisplayName(),
      'roles' => $user->getRoles(),
    ];
  }

}
```

### 844. Custom Cache Backend

การสร้าง Custom Cache Backend สำหรับความต้องการพิเศษ:

```php
<?php
// web/modules/custom/mymodule/src/Cache/EncryptedCacheBackend.php

namespace Drupal\mymodule\Cache;

use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Cache\Cache;

/**
 * Cache Backend ที่เข้ารหัสข้อมูลก่อนบันทึก
 * ใช้สำหรับ sensitive data ที่ต้องการ cache
 */
class EncryptedCacheBackend implements CacheBackendInterface {

  /**
   * @var \Drupal\Core\Cache\CacheBackendInterface
   */
  protected CacheBackendInterface $backend;

  /**
   * @var string
   */
  protected string $encryptionKey;

  public function __construct(
    CacheBackendInterface $backend,
    string $encryptionKey
  ) {
    $this->backend = $backend;
    $this->encryptionKey = $encryptionKey;
  }

  /**
   * {@inheritdoc}
   */
  public function get($cid, $allow_invalid = FALSE) {
    $item = $this->backend->get($cid, $allow_invalid);
    if ($item) {
      $item->data = $this->decrypt($item->data);
    }
    return $item;
  }

  /**
   * {@inheritdoc}
   */
  public function getMultiple(&$cids, $allow_invalid = FALSE) {
    $items = $this->backend->getMultiple($cids, $allow_invalid);
    foreach ($items as &$item) {
      $item->data = $this->decrypt($item->data);
    }
    return $items;
  }

  /**
   * {@inheritdoc}
   */
  public function set($cid, $data, $expire = Cache::PERMANENT, array $tags = []) {
    $encrypted = $this->encrypt($data);
    $this->backend->set($cid, $encrypted, $expire, $tags);
  }

  /**
   * {@inheritdoc}
   */
  public function setMultiple(array $items) {
    foreach ($items as &$item) {
      $item['data'] = $this->encrypt($item['data']);
    }
    $this->backend->setMultiple($items);
  }

  /**
   * {@inheritdoc}
   */
  public function delete($cid) {
    $this->backend->delete($cid);
  }

  /**
   * {@inheritdoc}
   */
  public function deleteMultiple(array $cids) {
    $this->backend->deleteMultiple($cids);
  }

  /**
   * {@inheritdoc}
   */
  public function deleteAll() {
    $this->backend->deleteAll();
  }

  /**
   * {@inheritdoc}
   */
  public function invalidate($cid) {
    $this->backend->invalidate($cid);
  }

  /**
   * {@inheritdoc}
   */
  public function invalidateMultiple(array $cids) {
    $this->backend->invalidateMultiple($cids);
  }

  /**
   * {@inheritdoc}
   */
  public function invalidateAll() {
    $this->backend->invalidateAll();
  }

  /**
   * {@inheritdoc}
   */
  public function invalidateTags(array $tags) {
    $this->backend->invalidateTags($tags);
  }

  /**
   * {@inheritdoc}
   */
  public function removeBin() {
    $this->backend->removeBin();
  }

  /**
   * เข้ารหัสข้อมูล
   */
  private function encrypt(mixed $data): string {
    $serialized = serialize($data);
    $iv = random_bytes(16);
    $encrypted = openssl_encrypt(
      $serialized,
      'AES-256-CBC',
      $this->encryptionKey,
      0,
      $iv
    );
    return base64_encode($iv . $encrypted);
  }

  /**
   * ถอดรหัสข้อมูล
   */
  private function decrypt(mixed $data): mixed {
    if (!is_string($data)) {
      return $data;
    }
    try {
      $decoded = base64_decode($data);
      $iv = substr($decoded, 0, 16);
      $encrypted = substr($decoded, 16);
      $decrypted = openssl_decrypt(
        $encrypted,
        'AES-256-CBC',
        $this->encryptionKey,
        0,
        $iv
      );
      return unserialize($decrypted);
    }
    catch (\Exception $e) {
      return $data;
    }
  }

}
```

```yaml
# web/modules/custom/mymodule/mymodule.services.yml

services:
  # กำหนด cache backend ปกติ
  cache.mymodule:
    class: Drupal\Core\Cache\DatabaseBackend
    factory: cache_factory:get
    arguments: ['mymodule']
    tags:
      - { name: cache.bin }

  # กำหนด encrypted cache backend
  cache.mymodule_secure:
    class: Drupal\mymodule\Cache\EncryptedCacheBackend
    arguments:
      - '@cache.mymodule'
      - '%env(CACHE_ENCRYPTION_KEY)%'

  # DataService ใช้ cache.mymodule
  mymodule.data_service:
    class: Drupal\mymodule\Service\DataService
    arguments:
      - '@cache.mymodule'
      - '@cache_tags.invalidator'
```

### 845. Cache Invalidation Strategies

กลยุทธ์การ invalidate cache ที่ถูกต้อง:

```php
<?php
// web/modules/custom/mymodule/src/EventSubscriber/NodeCacheSubscriber.php

namespace Drupal\mymodule\EventSubscriber;

use Drupal\Core\Cache\CacheTagsInvalidatorInterface;
use Drupal\hook_event_dispatcher\HookEventDispatcherInterface;
use Drupal\node\NodeInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

/**
 * Subscriber สำหรับจัดการ cache invalidation เมื่อ node เปลี่ยนแปลง
 */
class NodeCacheSubscriber implements EventSubscriberInterface {

  public function __construct(
    protected CacheTagsInvalidatorInterface $cacheTagsInvalidator
  ) {}

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents(): array {
    return [
      'hook_event_dispatcher.node.insert' => 'onNodeInsert',
      'hook_event_dispatcher.node.update' => 'onNodeUpdate',
      'hook_event_dispatcher.node.delete' => 'onNodeDelete',
    ];
  }

  /**
   * เมื่อสร้าง node ใหม่
   */
  public function onNodeInsert($event): void {
    $node = $event->getEntity();
    if ($node instanceof NodeInterface) {
      // Invalidate listing pages
      $this->cacheTagsInvalidator->invalidateTags([
        'node_list',
        'node_list:' . $node->getType(),
      ]);
    }
  }

  /**
   * เมื่ออัปเดต node
   */
  public function onNodeUpdate($event): void {
    $node = $event->getEntity();
    if ($node instanceof NodeInterface) {
      $tags = [
        'node:' . $node->id(),
        'node_list',
        'node_list:' . $node->getType(),
      ];

      // ถ้า URL alias เปลี่ยน ต้อง invalidate route cache ด้วย
      if ($node->isPublished() !== $node->original->isPublished()) {
        $tags[] = 'config:system.site';
      }

      $this->cacheTagsInvalidator->invalidateTags($tags);
    }
  }

  /**
   * เมื่อลบ node
   */
  public function onNodeDelete($event): void {
    $node = $event->getEntity();
    if ($node instanceof NodeInterface) {
      $this->cacheTagsInvalidator->invalidateTags([
        'node:' . $node->id(),
        'node_list',
        'node_list:' . $node->getType(),
      ]);
    }
  }

}
```

---

## ขั้นตอนที่ 846-850: BigPipe Module

### 846. BigPipe คืออะไร?

BigPipe เป็นเทคนิค Progressive Rendering ที่ Facebook พัฒนาขึ้น และ Drupal 8+ ได้นำมาใช้เป็น core module

**หลักการทำงาน:**
1. Server ส่ง HTML skeleton ไปให้ Browser ทันที
2. Browser เริ่มแสดงผลและโหลด CSS/JS ทันที
3. ส่วนที่ใช้เวลาคำนวณนาน (personalized content) ถูกส่งมาทีหลัง
4. JavaScript แทนที่ placeholder ด้วยเนื้อหาจริง

```
ก่อน BigPipe:
Request → [คำนวณทุกอย่าง] → ส่ง HTML ทั้งหมด → แสดงผล
ผล: TTFB = 3-5 วินาที

หลัง BigPipe:
Request → ส่ง HTML skeleton → แสดงผลทันที → ทยอยส่ง content
ผล: TTFB = 0.1-0.3 วินาที
```

### 847. เปิดใช้งาน BigPipe

```bash
# เปิดใช้งาน BigPipe module
drush en big_pipe -y

# ตรวจสอบสถานะ
drush status big_pipe
```

### 848. สร้าง BigPipe Placeholder

```php
<?php
// web/modules/custom/mymodule/src/Plugin/Block/PersonalizedBlock.php

namespace Drupal\mymodule\Plugin\Block;

use Drupal\Core\Block\BlockBase;
use Drupal\Core\Cache\Cache;
use Drupal\Core\Security\TrustedCallbackInterface;

/**
 * @Block(
 *   id = "personalized_content_block",
 *   admin_label = @Translation("Personalized Content Block"),
 * )
 *
 * Block นี้แสดงเนื้อหาที่เฉพาะเจาะจงสำหรับแต่ละ user
 * โดยใช้ BigPipe สำหรับ lazy loading
 */
class PersonalizedBlock extends BlockBase implements TrustedCallbackInterface {

  /**
   * {@inheritdoc}
   */
  public function build(): array {
    return [
      // ใช้ #lazy_builder เพื่อให้ BigPipe จัดการ
      '#lazy_builder' => [
        // Callback ที่จะถูกเรียกแบบ lazy
        static::class . '::buildContent',
        // Arguments
        [\Drupal::currentUser()->id()],
      ],
      // สำคัญ: สร้าง placeholder placeholder
      '#create_placeholder' => TRUE,
    ];
  }

  /**
   * Lazy builder callback - ถูกเรียกโดย BigPipe
   *
   * @param int $userId
   *   User ID
   *
   * @return array
   *   Render array
   */
  public static function buildContent(int $userId): array {
    // จำลอง expensive operation
    $userData = static::fetchUserPersonalizedData($userId);

    return [
      '#theme' => 'mymodule_personalized_block',
      '#user_data' => $userData,
      '#cache' => [
        'contexts' => ['user'],
        'tags' => ['user:' . $userId],
        'max-age' => 300,
      ],
    ];
  }

  /**
   * {@inheritdoc}
   */
  public static function trustedCallbacks(): array {
    return ['buildContent'];
  }

  /**
   * ดึงข้อมูล personalized สำหรับ user
   */
  private static function fetchUserPersonalizedData(int $userId): array {
    // Simulate database query
    $database = \Drupal::database();
    $result = $database->select('users_field_data', 'u')
      ->fields('u', ['name', 'mail', 'created'])
      ->condition('u.uid', $userId)
      ->execute()
      ->fetchAssoc();

    return $result ?: ['name' => 'Anonymous', 'mail' => '', 'created' => 0];
  }

  /**
   * Cache contexts สำหรับ block นี้
   */
  public function getCacheContexts(): array {
    return Cache::mergeContexts(
      parent::getCacheContexts(),
      ['user', 'session']
    );
  }

  /**
   * ไม่ cache block wrapper (เนื้อหาถูก cache ใน lazy builder แทน)
   */
  public function getCacheMaxAge(): int {
    return 0;
  }

}
```

### 849. Custom Lazy Builder Service

```php
<?php
// web/modules/custom/mymodule/src/LazyBuilder/ProductRecommendationLazyBuilder.php

namespace Drupal\mymodule\LazyBuilder;

use Drupal\Core\Security\TrustedCallbackInterface;
use Drupal\Core\StringTranslation\StringTranslationTrait;

/**
 * Lazy builder สำหรับ product recommendations
 * ใช้กับ BigPipe เพื่อโหลด recommendations แบบ asynchronous
 */
class ProductRecommendationLazyBuilder implements TrustedCallbackInterface {

  use StringTranslationTrait;

  /**
   * สร้าง product recommendations render array
   *
   * @param int $userId
   *   User ID
   * @param int $limit
   *   จำนวน recommendations ที่แสดง
   *
   * @return array
   *   Render array สำหรับ recommendations
   */
  public static function buildRecommendations(int $userId, int $limit = 5): array {
    // ดึง recommendations จาก ML service หรือ database
    $recommendations = static::getRecommendations($userId, $limit);

    if (empty($recommendations)) {
      return [
        '#markup' => t('ไม่มีคำแนะนำสำหรับคุณในขณะนี้'),
      ];
    }

    return [
      '#theme' => 'item_list',
      '#title' => t('สินค้าแนะนำสำหรับคุณ'),
      '#items' => array_map(function ($product) {
        return [
          '#type' => 'link',
          '#title' => $product['title'],
          '#url' => \Drupal\Core\Url::fromRoute(
            'entity.node.canonical',
            ['node' => $product['nid']]
          ),
        ];
      }, $recommendations),
      '#cache' => [
        'contexts' => ['user'],
        'tags' => array_map(fn($p) => 'node:' . $p['nid'], $recommendations),
        'max-age' => 600, // 10 นาที
      ],
    ];
  }

  /**
   * {@inheritdoc}
   */
  public static function trustedCallbacks(): array {
    return ['buildRecommendations'];
  }

  /**
   * ดึง recommendations จาก database
   */
  private static function getRecommendations(int $userId, int $limit): array {
    $database = \Drupal::database();

    // ตัวอย่าง: ดึงสินค้าที่ user อื่นที่มี profile คล้ายกันซื้อ
    return $database->select('product_recommendations', 'pr')
      ->fields('pr', ['nid', 'title', 'score'])
      ->condition('pr.uid', $userId)
      ->orderBy('pr.score', 'DESC')
      ->range(0, $limit)
      ->execute()
      ->fetchAll(\PDO::FETCH_ASSOC);
  }

}
```

```php
<?php
// ใช้งาน Lazy Builder ใน template หรือ hook

/**
 * Implements hook_preprocess_node().
 */
function mymodule_preprocess_node(array &$variables): void {
  $node = $variables['node'];

  if ($node->getType() === 'product') {
    $userId = \Drupal::currentUser()->id();

    // เพิ่ม recommendations แบบ lazy
    $variables['recommendations'] = [
      '#lazy_builder' => [
        'mymodule.product_recommendation_lazy_builder:buildRecommendations',
        [$userId, 5],
      ],
      '#create_placeholder' => TRUE,
    ];
  }
}
```

### 850. BigPipe กับ Authenticated Users

```php
<?php
// web/modules/custom/mymodule/src/Plugin/Block/CartBlock.php

namespace Drupal\mymodule\Plugin\Block;

use Drupal\Core\Block\BlockBase;
use Drupal\Core\Cache\Cache;
use Drupal\Core\Security\TrustedCallbackInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * @Block(
 *   id = "shopping_cart_block",
 *   admin_label = @Translation("Shopping Cart Block"),
 * )
 */
class CartBlock extends BlockBase implements TrustedCallbackInterface, ContainerFactoryPluginInterface {

  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    protected AccountInterface $currentUser
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
  }

  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('current_user')
    );
  }

  /**
   * {@inheritdoc}
   */
  public function build(): array {
    // Anonymous users เห็น empty cart ทันที
    if ($this->currentUser->isAnonymous()) {
      return [
        '#markup' => '<div class="cart-icon">ตะกร้าสินค้า (0)</div>',
        '#cache' => [
          'contexts' => ['user.roles:anonymous'],
        ],
      ];
    }

    // Authenticated users ใช้ BigPipe lazy loading
    return [
      '#lazy_builder' => [
        static::class . '::buildCartContent',
        [$this->currentUser->id()],
      ],
      '#create_placeholder' => TRUE,
    ];
  }

  /**
   * Lazy builder สำหรับ cart content
   */
  public static function buildCartContent(int $userId): array {
    $cartItems = static::getCartItems($userId);
    $count = count($cartItems);
    $total = array_sum(array_column($cartItems, 'price'));

    return [
      '#theme' => 'shopping_cart_block',
      '#items' => $cartItems,
      '#count' => $count,
      '#total' => $total,
      '#cache' => [
        'contexts' => ['user'],
        'tags' => ['cart:' . $userId],
        'max-age' => 60, // 1 นาที
      ],
    ];
  }

  /**
   * {@inheritdoc}
   */
  public static function trustedCallbacks(): array {
    return ['buildCartContent'];
  }

  public function getCacheContexts(): array {
    return Cache::mergeContexts(parent::getCacheContexts(), ['user']);
  }

  public function getCacheMaxAge(): int {
    return 0; // ให้ lazy builder จัดการ cache
  }

  private static function getCartItems(int $userId): array {
    // ดึง cart items จาก database/session
    return \Drupal::database()->select('cart_items', 'ci')
      ->fields('ci', ['product_id', 'title', 'price', 'quantity'])
      ->condition('ci.uid', $userId)
      ->execute()
      ->fetchAll(\PDO::FETCH_ASSOC);
  }

}
```

---

## ขั้นตอนที่ 851-855: Varnish & Reverse Proxy

### 851. Varnish Architecture

Varnish Cache ทำหน้าที่เป็น HTTP reverse proxy ที่อยู่หน้า Drupal:

```
Internet → Varnish (port 80/443) → Drupal (port 8080)
                ↓
         [Cache Hit: ตอบทันที]
                ↓
         [Cache Miss: ส่งต่อไป Drupal แล้ว cache ผลลัพธ์]
```

### 852. ตั้งค่า Drupal ให้ทำงานกับ Varnish

```php
<?php
// web/sites/default/settings.php หรือ settings.local.php

// บอก Drupal ว่ามี reverse proxy อยู่
$settings['reverse_proxy'] = TRUE;

// IP ของ Varnish servers
$settings['reverse_proxy_addresses'] = ['10.0.0.1', '10.0.0.2'];

// Header ที่ Varnish ส่งมาบอก IP จริงของ client
$settings['reverse_proxy_header'] = 'X-Forwarded-For';

// Protocol header
$settings['reverse_proxy_proto_header'] = 'X-Forwarded-Proto';

// Port header
$settings['reverse_proxy_port_header'] = 'X-Forwarded-Port';

// Host header
$settings['reverse_proxy_host_header'] = 'X-Forwarded-Host';

// Trusted Host Patterns
$settings['trusted_host_patterns'] = [
  '^www\.example\.com$',
  '^example\.com$',
];
```

### 853. Varnish Configuration (VCL)

```vcl
# /etc/varnish/default.vcl
# Varnish Configuration Language สำหรับ Drupal

vcl 4.0;

import std;
import directors;

# กำหนด Drupal backend
backend drupal1 {
  .host = "drupal-app-1";
  .port = "8080";
  .connect_timeout = 60s;
  .send_timeout = 300s;
  .first_byte_timeout = 300s;
  .probe = {
    .url = "/health-check";
    .interval = 5s;
    .timeout = 2s;
    .window = 5;
    .threshold = 3;
  }
}

backend drupal2 {
  .host = "drupal-app-2";
  .port = "8080";
  .connect_timeout = 60s;
  .send_timeout = 300s;
  .first_byte_timeout = 300s;
}

# Load balancer
sub vcl_init {
  new drupal_cluster = directors.round_robin();
  drupal_cluster.add_backend(drupal1);
  drupal_cluster.add_backend(drupal2);
}

# Request processing
sub vcl_recv {
  set req.backend_hint = drupal_cluster.backend();

  # ลบ cookies ที่ไม่จำเป็น (ยกเว้น Drupal session)
  if (req.http.Cookie) {
    # เก็บเฉพาะ Drupal session cookies
    set req.http.Cookie = ";" + req.http.Cookie;
    set req.http.Cookie = regsuball(req.http.Cookie, "; +", ";");
    set req.http.Cookie = regsuball(req.http.Cookie, ";(SESS[a-z0-9]+|SSESS[a-z0-9]+|NO_CACHE)=", "; \1=");
    set req.http.Cookie = regsuball(req.http.Cookie, ";[^ ][^;]*", "");
    set req.http.Cookie = regsuball(req.http.Cookie, "^[; ]+|[; ]+$", "");

    if (req.http.Cookie == "") {
      # ไม่มี Drupal cookies → anonymous user → cache ได้
      unset req.http.Cookie;
    }
  }

  # ไม่ cache ถ้ามี Authorization header
  if (req.http.Authorization) {
    return(pass);
  }

  # ไม่ cache สำหรับ admin paths
  if (req.url ~ "^/admin" || req.url ~ "^/user") {
    return(pass);
  }

  # ไม่ cache สำหรับ POST requests
  if (req.method == "POST") {
    return(pass);
  }

  return(hash);
}

# Backend response processing
sub vcl_backend_response {
  # Cache Drupal pages
  if (beresp.http.Cache-Control ~ "no-cache|no-store|private") {
    return(pass);
  }

  # ลบ Set-Cookie headers สำหรับ static assets
  if (bereq.url ~ "\.(css|js|png|jpg|gif|svg|woff|woff2)$") {
    unset beresp.http.Set-Cookie;
    set beresp.ttl = 7d;
    return(deliver);
  }

  # Grace period: ให้ serve stale content ระหว่างรอ backend ตอบ
  set beresp.grace = 6h;
  set beresp.keep = 24h;

  return(deliver);
}

# Deliver response
sub vcl_deliver {
  # เพิ่ม debug headers (ลบออกใน production)
  if (obj.hits > 0) {
    set resp.http.X-Cache = "HIT";
    set resp.http.X-Cache-Hits = obj.hits;
  }
  else {
    set resp.http.X-Cache = "MISS";
  }

  return(deliver);
}

# Cache hash - ใช้ URL + Host เป็น cache key
sub vcl_hash {
  hash_data(req.url);

  if (req.http.host) {
    hash_data(req.http.host);
  }
  else {
    hash_data(server.ip);
  }

  return(lookup);
}
```

### 854. Drupal Purge Module

Purge module ใช้สำหรับ invalidate Varnish cache เมื่อ content เปลี่ยน:

```bash
# ติดตั้ง purge modules
composer require drupal/purge drupal/varnish_purger

# เปิดใช้งาน
drush en purge purge_drush purge_ui purge_queuer_coretags purge_processor_cron varnish_purger -y
```

```php
<?php
// web/modules/custom/mymodule/src/Service/CachePurgeService.php

namespace Drupal\mymodule\Service;

use Drupal\purge\Plugin\Purge\Purger\PurgersServiceInterface;
use Drupal\purge\Plugin\Purge\Invalidation\InvalidationsServiceInterface;

/**
 * Service สำหรับ purge Varnish cache
 */
class CachePurgeService {

  public function __construct(
    protected PurgersServiceInterface $purgersService,
    protected InvalidationsServiceInterface $invalidationsService
  ) {}

  /**
   * Purge URL เฉพาะ
   */
  public function purgeUrl(string $url): void {
    $invalidations = [
      $this->invalidationsService->get('url', $url),
    ];
    $this->purgersService->invalidate($invalidations);
  }

  /**
   * Purge ด้วย cache tag
   */
  public function purgeByTag(string $tag): void {
    $invalidations = [
      $this->invalidationsService->get('tag', $tag),
    ];
    $this->purgersService->invalidate($invalidations);
  }

  /**
   * Purge ทั้งหมด (ใช้ระวัง!)
   */
  public function purgeEverything(): void {
    $invalidations = [
      $this->invalidationsService->get('everything'),
    ];
    $this->purgersService->invalidate($invalidations);
  }

}
```

### 855. HTTP Cache Headers ใน Drupal

```php
<?php
// web/modules/custom/mymodule/src/EventSubscriber/CacheHeaderSubscriber.php

namespace Drupal\mymodule\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

/**
 * Subscriber สำหรับปรับแต่ง Cache-Control headers
 */
class CacheHeaderSubscriber implements EventSubscriberInterface {

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents(): array {
    return [
      KernelEvents::RESPONSE => ['onKernelResponse', -100],
    ];
  }

  /**
   * ปรับแต่ง cache headers
   */
  public function onKernelResponse(ResponseEvent $event): void {
    if (!$event->isMainRequest()) {
      return;
    }

    $request = $event->getRequest();
    $response = $event->getResponse();
    $path = $request->getPathInfo();

    // Static assets: cache นาน
    if (preg_match('/\.(css|js|png|jpg|gif|svg|woff2?)$/', $path)) {
      $response->headers->set('Cache-Control', 'public, max-age=31536000, immutable');
      $response->headers->set('Vary', 'Accept-Encoding');
      return;
    }

    // API endpoints: cache สั้น
    if (str_starts_with($path, '/api/')) {
      $response->headers->set('Cache-Control', 'public, max-age=60, s-maxage=120');
      $response->headers->set('Vary', 'Accept, Accept-Language, Accept-Encoding');
      return;
    }

    // Admin pages: ไม่ cache
    if (str_starts_with($path, '/admin') || str_starts_with($path, '/user')) {
      $response->headers->set('Cache-Control', 'no-cache, no-store, must-revalidate');
      $response->headers->set('Pragma', 'no-cache');
      $response->headers->set('Expires', '0');
    }
  }

}
```

---

## ขั้นตอนที่ 856-860: Redis Integration

### 856. ทำไมต้องใช้ Redis?

**ปัญหาของ Database Cache:**
- MySQL cache ต้องเขียน/อ่าน disk
- การ lock table ทำให้ช้า
- ไม่เหมาะกับ high-concurrency

**ข้อดีของ Redis:**
- In-memory storage → เร็วมาก (< 1ms)
- Built-in data structures (hash, list, sorted set)
- Pub/Sub สำหรับ real-time features
- Clustering รองรับ horizontal scaling
- Automatic expiry

### 857. ติดตั้ง Redis สำหรับ Drupal

```bash
# ติดตั้ง predis/predis หรือ phpredis
composer require predis/predis

# หรือติดตั้ง phpredis extension
pecl install redis

# ติดตั้ง Drupal Redis module
composer require drupal/redis

# เปิดใช้งาน
drush en redis -y
```

### 858. ตั้งค่า Redis Cache Backend

```php
<?php
// web/sites/default/settings.php

// Redis configuration
$settings['redis.connection']['interface'] = 'PhpRedis'; // หรือ 'Predis'
$settings['redis.connection']['host'] = 'redis-server'; // Redis hostname
$settings['redis.connection']['port'] = 6379;
// $settings['redis.connection']['password'] = 'your-redis-password';
$settings['redis.connection']['base'] = 0; // Redis database number

// ใช้ Redis เป็น cache backend หลัก
$settings['cache']['default'] = 'cache.backend.redis';

// กำหนด cache bins เฉพาะ
$settings['cache']['bins']['render'] = 'cache.backend.redis';
$settings['cache']['bins']['dynamic_page_cache'] = 'cache.backend.redis';
$settings['cache']['bins']['page'] = 'cache.backend.redis';

// ไม่ใช้ Redis สำหรับ bootstrap (เพื่อความเร็วใน bootstrapping)
$settings['cache']['bins']['bootstrap'] = 'cache.backend.chainedfast';
$settings['cache']['bins']['discovery'] = 'cache.backend.chainedfast';
$settings['cache']['bins']['config'] = 'cache.backend.chainedfast';

// Redis prefix (ป้องกัน collision กับ apps อื่น)
$settings['cache_prefix']['default'] = 'drupal_prod_';

// PhpRedis serializer (เร็วกว่า PHP serialize)
$settings['redis.connection']['serializer'] = Redis::SERIALIZER_IGBINARY;
```

### 859. Redis Sentinel สำหรับ High Availability

```php
<?php
// web/sites/default/settings.php - Redis Sentinel Configuration

// ใช้ Redis Sentinel สำหรับ HA
$settings['redis.connection']['interface'] = 'PhpRedis';
$settings['redis.connection']['replication'] = 'sentinel';
$settings['redis.connection']['service'] = 'mymaster'; // Sentinel service name

// Sentinel nodes
$settings['redis.connection']['host'] = [
  '10.0.1.1:26379', // sentinel 1
  '10.0.1.2:26379', // sentinel 2
  '10.0.1.3:26379', // sentinel 3
];
```

### 860. ใช้ Redis สำหรับ Session Storage

```php
<?php
// web/sites/default/settings.php

// ใช้ Redis สำหรับ session storage
ini_set('session.save_handler', 'redis');
ini_set('session.save_path', 'tcp://redis-server:6379?auth=password&prefix=drupal_sess_&database=1');

// หรือใช้ Drupal session handler
$settings['session.storage.options'] = [
  'cookie_lifetime' => 0,
  'gc_probability' => 1,
  'gc_divisor' => 100,
  'gc_maxlifetime' => 200000,
];
```

```php
<?php
// web/modules/custom/mymodule/src/Service/RedisService.php

namespace Drupal\mymodule\Service;

use Drupal\redis\ClientFactory;

/**
 * Service สำหรับใช้งาน Redis โดยตรง
 * ใช้ในกรณีที่ต้องการ Redis features ที่ไม่มีใน Cache API
 */
class RedisService {

  /**
   * @var \Redis
   */
  protected $redis;

  public function __construct(ClientFactory $clientFactory) {
    $this->redis = $clientFactory->getClient();
  }

  /**
   * Rate limiting โดยใช้ Redis
   * คืนค่า true ถ้าอยู่ใน limit, false ถ้าเกิน limit
   */
  public function checkRateLimit(string $identifier, int $limit = 100, int $window = 3600): bool {
    $key = 'rate_limit:' . $identifier;

    // ใช้ Redis pipeline เพื่อ atomicity
    $this->redis->multi();
    $this->redis->incr($key);
    $this->redis->expire($key, $window);
    $results = $this->redis->exec();

    $currentCount = $results[0];
    return $currentCount <= $limit;
  }

  /**
   * Distributed lock โดยใช้ Redis
   */
  public function acquireLock(string $lockName, int $ttl = 30): bool {
    $key = 'lock:' . $lockName;
    $value = uniqid('lock_', TRUE);

    // SET NX EX - atomic lock acquire
    $result = $this->redis->set($key, $value, ['NX', 'EX' => $ttl]);

    return $result === TRUE;
  }

  /**
   * Leaderboard โดยใช้ Redis Sorted Set
   */
  public function updateLeaderboard(string $userId, float $score): void {
    $this->redis->zAdd('leaderboard', $score, $userId);
    // เก็บแค่ top 1000
    $this->redis->zRemRangeByRank('leaderboard', 0, -1001);
  }

  /**
   * ดึง top N จาก leaderboard
   */
  public function getTopLeaderboard(int $n = 10): array {
    return $this->redis->zRevRange('leaderboard', 0, $n - 1, TRUE);
  }

  /**
   * Pub/Sub: publish event
   */
  public function publishEvent(string $channel, array $data): void {
    $this->redis->publish($channel, json_encode($data));
  }

  /**
   * Counter with expiry
   */
  public function incrementCounter(string $name, int $amount = 1, int $ttl = null): int {
    $key = 'counter:' . $name;
    $value = $this->redis->incrBy($key, $amount);

    if ($ttl !== null && $value === $amount) {
      // ตั้ง expiry เฉพาะครั้งแรก
      $this->redis->expire($key, $ttl);
    }

    return $value;
  }

}
```

---

## ขั้นตอนที่ 861-865: Database Optimization

### 861. Query Tag สำหรับ Profiling

```php
<?php
// web/modules/custom/mymodule/src/Service/OptimizedQueryService.php

namespace Drupal\mymodule\Service;

use Drupal\Core\Database\Connection;

/**
 * Service สำหรับ optimized database queries
 */
class OptimizedQueryService {

  public function __construct(
    protected Connection $database
  ) {}

  /**
   * Query ที่ optimized พร้อม tagging
   */
  public function getActiveArticles(int $limit = 10): array {
    $query = $this->database->select('node_field_data', 'n');
    $query->fields('n', ['nid', 'title', 'created', 'changed']);
    $query->condition('n.type', 'article');
    $query->condition('n.status', 1);
    $query->orderBy('n.created', 'DESC');
    $query->range(0, $limit);

    // เพิ่ม tags เพื่อ tracking และ debugging
    $query->addTag('node_access');
    $query->addTag('mymodule_articles');

    // เพิ่ม metadata สำหรับ debugging
    $query->addMetaData('account', \Drupal::currentUser());

    return $query->execute()->fetchAll(\PDO::FETCH_ASSOC);
  }

  /**
   * ใช้ Query ด้วย JOIN ที่ optimized
   */
  public function getArticlesWithAuthor(int $limit = 10): array {
    $query = $this->database->select('node_field_data', 'n');

    // JOIN กับ users table
    $query->leftJoin('users_field_data', 'u', 'n.uid = u.uid');

    $query->addField('n', 'nid');
    $query->addField('n', 'title');
    $query->addField('n', 'created');
    $query->addField('u', 'name', 'author_name');

    $query->condition('n.type', 'article');
    $query->condition('n.status', 1);
    $query->orderBy('n.created', 'DESC');
    $query->range(0, $limit);

    $query->addTag('node_access');

    return $query->execute()->fetchAll(\PDO::FETCH_ASSOC);
  }

  /**
   * Batch query สำหรับ large dataset
   */
  public function processLargeDataset(callable $processor): void {
    $batchSize = 1000;
    $offset = 0;

    do {
      $query = $this->database->select('node_field_data', 'n');
      $query->fields('n', ['nid', 'title']);
      $query->condition('n.type', 'article');
      $query->range($offset, $batchSize);

      $results = $query->execute()->fetchAll(\PDO::FETCH_ASSOC);

      if (empty($results)) {
        break;
      }

      // Process batch
      $processor($results);

      $offset += $batchSize;

      // ให้ DB หายใจ
      if ($offset % 10000 === 0) {
        sleep(1);
      }

    } while (count($results) === $batchSize);
  }

  /**
   * Query ที่ใช้ Index อย่างถูกต้อง
   */
  public function getNodesByDateRange(string $startDate, string $endDate): array {
    // ใช้ timestamp (indexed) แทน DATE() function
    $start = strtotime($startDate);
    $end = strtotime($endDate . ' 23:59:59');

    return $this->database->select('node_field_data', 'n')
      ->fields('n', ['nid', 'title', 'created'])
      ->condition('n.type', 'article')
      ->condition('n.status', 1)
      ->condition('n.created', [$start, $end], 'BETWEEN')
      ->orderBy('n.created', 'DESC')
      ->execute()
      ->fetchAll(\PDO::FETCH_ASSOC);
  }

}
```

### 862. Hook สำหรับ Query Alteration

```php
<?php
// web/modules/custom/mymodule/mymodule.module

/**
 * Implements hook_query_TAG_alter().
 * แก้ไข query ที่มี tag 'mymodule_articles'
 */
function mymodule_query_mymodule_articles_alter(\Drupal\Core\Database\Query\AlterableInterface $query): void {
  // เพิ่มเงื่อนไขเพิ่มเติมสำหรับ query ทุกตัวที่มี tag นี้
  $config = \Drupal::config('mymodule.settings');

  // กรองตาม featured flag
  if ($config->get('show_featured_only')) {
    $query->condition('n.promote', 1);
  }

  // Log query สำหรับ debugging ใน development
  if (\Drupal::state()->get('mymodule.debug_queries', FALSE)) {
    \Drupal::logger('mymodule')->debug(
      'Query: @query',
      ['@query' => (string) $query]
    );
  }
}
```

### 863. Database Index Management

```php
<?php
// web/modules/custom/mymodule/mymodule.install

/**
 * Implements hook_schema().
 */
function mymodule_schema(): array {
  $schema = [];

  $schema['mymodule_analytics'] = [
    'description' => 'Analytics data table',
    'fields' => [
      'id' => [
        'type' => 'serial',
        'unsigned' => TRUE,
        'not null' => TRUE,
      ],
      'nid' => [
        'type' => 'int',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'default' => 0,
      ],
      'uid' => [
        'type' => 'int',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'default' => 0,
      ],
      'event_type' => [
        'type' => 'varchar',
        'length' => 64,
        'not null' => TRUE,
        'default' => '',
      ],
      'timestamp' => [
        'type' => 'int',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'default' => 0,
      ],
      'data' => [
        'type' => 'blob',
        'size' => 'big',
        'not null' => FALSE,
      ],
    ],
    'primary key' => ['id'],
    // สร้าง indexes สำหรับ queries ที่ใช้บ่อย
    'indexes' => [
      // Index สำหรับ query by nid
      'nid' => ['nid'],
      // Index สำหรับ query by user
      'uid_timestamp' => ['uid', 'timestamp'],
      // Index สำหรับ query by event type และ timestamp
      'event_timestamp' => ['event_type', 'timestamp'],
      // Composite index สำหรับ analytics queries
      'nid_event_timestamp' => ['nid', 'event_type', 'timestamp'],
    ],
  ];

  return $schema;
}

/**
 * เพิ่ม index สำหรับ table ที่มีอยู่แล้ว
 */
function mymodule_update_10001(): void {
  $database = \Drupal::database();

  // ตรวจสอบว่า index มีอยู่แล้วหรือไม่
  $schema = $database->schema();
  if (!$schema->indexExists('mymodule_analytics', 'event_timestamp')) {
    $schema->addIndex(
      'mymodule_analytics',
      'event_timestamp',
      ['event_type', 'timestamp'],
      [
        'fields' => [
          'event_type' => ['type' => 'varchar', 'length' => 64],
          'timestamp' => ['type' => 'int'],
        ],
        'indexes' => [
          'event_timestamp' => ['event_type', 'timestamp'],
        ],
      ]
    );
  }
}
```

### 864. Slow Query Monitoring

```php
<?php
// web/modules/custom/mymodule/src/EventSubscriber/SlowQuerySubscriber.php

namespace Drupal\mymodule\EventSubscriber;

use Drupal\Core\Database\Connection;
use Psr\Log\LoggerInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\TerminateEvent;
use Symfony\Component\HttpKernel\KernelEvents;

/**
 * Monitor slow database queries
 */
class SlowQuerySubscriber implements EventSubscriberInterface {

  /**
   * Slow query threshold (milliseconds)
   */
  const SLOW_QUERY_THRESHOLD_MS = 1000;

  public function __construct(
    protected Connection $database,
    protected LoggerInterface $logger
  ) {}

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents(): array {
    return [
      KernelEvents::TERMINATE => 'onKernelTerminate',
    ];
  }

  /**
   * ตรวจสอบ slow queries หลัง request เสร็จ
   */
  public function onKernelTerminate(TerminateEvent $event): void {
    if (!\Drupal::state()->get('mymodule.monitor_slow_queries', FALSE)) {
      return;
    }

    // ดึง query log จาก Drupal database logger
    $queries = $this->database->getLogger()?->get('queries') ?? [];

    foreach ($queries as $query) {
      $duration = $query['time'] * 1000; // แปลงเป็น ms

      if ($duration >= self::SLOW_QUERY_THRESHOLD_MS) {
        $this->logger->warning(
          'Slow query detected (@duration ms): @query | Args: @args',
          [
            '@duration' => round($duration, 2),
            '@query' => $query['query'],
            '@args' => print_r($query['args'], TRUE),
          ]
        );
      }
    }
  }

}
```

```php
<?php
// ใช้ EXPLAIN สำหรับ analyze query

namespace Drupal\mymodule\Service;

use Drupal\Core\Database\Connection;

/**
 * Service สำหรับ analyze query performance
 */
class QueryAnalyzerService {

  public function __construct(protected Connection $database) {}

  /**
   * Analyze query ด้วย EXPLAIN
   */
  public function analyzeQuery(string $sql, array $args = []): array {
    $explain = $this->database->query(
      'EXPLAIN ' . $sql,
      $args
    )->fetchAll(\PDO::FETCH_ASSOC);

    $issues = [];

    foreach ($explain as $row) {
      // ตรวจสอบว่าใช้ index หรือไม่
      if ($row['key'] === null) {
        $issues[] = [
          'severity' => 'warning',
          'message' => 'Full table scan on table ' . $row['table'],
          'rows_examined' => $row['rows'],
        ];
      }

      // ตรวจสอบ rows ที่ต้อง scan
      if ((int) $row['rows'] > 10000) {
        $issues[] = [
          'severity' => 'info',
          'message' => 'Large row scan: ' . $row['rows'] . ' rows on ' . $row['table'],
        ];
      }
    }

    return [
      'explain' => $explain,
      'issues' => $issues,
    ];
  }

}
```

### 865. Entity Query Optimization

```php
<?php
// web/modules/custom/mymodule/src/Service/OptimizedEntityService.php

namespace Drupal\mymodule\Service;

use Drupal\Core\Entity\EntityTypeManagerInterface;

/**
 * Service สำหรับ Entity queries ที่ optimized
 */
class OptimizedEntityService {

  public function __construct(
    protected EntityTypeManagerInterface $entityTypeManager
  ) {}

  /**
   * ดึง node IDs เท่านั้น (เร็วกว่า load entities)
   */
  public function getArticleIds(int $limit = 100): array {
    return $this->entityTypeManager
      ->getStorage('node')
      ->getQuery()
      ->condition('type', 'article')
      ->condition('status', 1)
      ->sort('created', 'DESC')
      ->range(0, $limit)
      ->accessCheck(TRUE)
      ->execute();
  }

  /**
   * Load entities แบบ batch เพื่อลด memory
   */
  public function processArticlesInBatches(callable $processor, int $batchSize = 50): void {
    $query = $this->entityTypeManager
      ->getStorage('node')
      ->getQuery()
      ->condition('type', 'article')
      ->condition('status', 1)
      ->sort('nid', 'ASC')
      ->accessCheck(TRUE);

    $lastId = 0;

    do {
      // ใช้ cursor-based pagination แทน offset
      $batchQuery = clone $query;
      $batchQuery->condition('nid', $lastId, '>');
      $batchQuery->range(0, $batchSize);

      $ids = $batchQuery->execute();

      if (empty($ids)) {
        break;
      }

      // Load entities
      $nodes = $this->entityTypeManager
        ->getStorage('node')
        ->loadMultiple($ids);

      $processor($nodes);

      // Reset static cache หลัง process แต่ละ batch
      $this->entityTypeManager->getStorage('node')->resetCache();

      $lastId = max(array_keys($ids));

    } while (count($ids) === $batchSize);
  }

  /**
   * ใช้ loadByProperties สำหรับ simple queries
   */
  public function getFeaturedArticles(): array {
    return $this->entityTypeManager
      ->getStorage('node')
      ->loadByProperties([
        'type' => 'article',
        'status' => 1,
        'promote' => 1,
      ]);
  }

}
```

---

## ขั้นตอนที่ 866-870: Asset Optimization

### 866. CSS/JS Aggregation

```php
<?php
// web/sites/default/settings.php

// เปิดใช้ CSS/JS aggregation
$config['system.performance']['css']['preprocess'] = TRUE;
$config['system.performance']['js']['preprocess'] = TRUE;

// กำหนด cache lifetime
$config['system.performance']['cache']['page']['max_age'] = 3600; // 1 ชั่วโมง
```

```yaml
# web/modules/custom/mymodule/mymodule.libraries.yml

# กำหนด library ที่ optimized
mymodule/main:
  version: 1.0.0
  css:
    theme:
      css/mymodule.css: {}
  js:
    js/mymodule.js:
      # เพิ่ม minify
      minified: true
  dependencies:
    - core/jquery
    - core/drupal

# Library สำหรับ critical CSS (โหลดก่อน)
mymodule/critical:
  version: 1.0.0
  css:
    component:
      # preload CSS สำคัญ
      css/critical.css:
        attributes:
          rel: preload
          as: style
          onload: "this.onload=null;this.rel='stylesheet'"

# Library ที่โหลด async
mymodule/analytics:
  version: 1.0.0
  js:
    js/analytics.js:
      attributes:
        async: true
        defer: true
  # โหลดท้าย page
  header: false
```

### 867. Asset Preloading

```php
<?php
// web/modules/custom/mymodule/mymodule.module

/**
 * Implements hook_page_attachments_alter().
 * เพิ่ม resource hints สำหรับ performance
 */
function mymodule_page_attachments_alter(array &$attachments): void {
  $route = \Drupal::routeMatch()->getRouteName();

  // DNS prefetch สำหรับ third-party domains
  $attachments['#attached']['html_head'][] = [
    [
      '#tag' => 'link',
      '#attributes' => [
        'rel' => 'dns-prefetch',
        'href' => '//fonts.googleapis.com',
      ],
    ],
    'dns_prefetch_fonts',
  ];

  // Preconnect สำหรับ critical resources
  $attachments['#attached']['html_head'][] = [
    [
      '#tag' => 'link',
      '#attributes' => [
        'rel' => 'preconnect',
        'href' => 'https://fonts.gstatic.com',
        'crossorigin' => TRUE,
      ],
    ],
    'preconnect_fonts_gstatic',
  ];

  // Preload critical font
  $attachments['#attached']['html_head'][] = [
    [
      '#tag' => 'link',
      '#attributes' => [
        'rel' => 'preload',
        'href' => '/themes/custom/mytheme/fonts/main.woff2',
        'as' => 'font',
        'type' => 'font/woff2',
        'crossorigin' => TRUE,
      ],
    ],
    'preload_main_font',
  ];

  // Preload hero image บน homepage
  if ($route === '<front>') {
    $attachments['#attached']['html_head'][] = [
      [
        '#tag' => 'link',
        '#attributes' => [
          'rel' => 'preload',
          'href' => '/sites/default/files/hero-image.webp',
          'as' => 'image',
          'type' => 'image/webp',
        ],
      ],
      'preload_hero_image',
    ];
  }
}
```

### 868. Image Optimization

```php
<?php
// web/modules/custom/mymodule/src/Service/ImageOptimizationService.php

namespace Drupal\mymodule\Service;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\File\FileSystemInterface;
use Drupal\file\FileInterface;

/**
 * Service สำหรับ optimize images
 */
class ImageOptimizationService {

  public function __construct(
    protected EntityTypeManagerInterface $entityTypeManager,
    protected FileSystemInterface $fileSystem
  ) {}

  /**
   * สร้าง responsive image styles
   */
  public function getResponsiveImageMarkup(int $fid, string $style = 'large'): array {
    $file = $this->entityTypeManager->getStorage('file')->load($fid);

    if (!$file instanceof FileInterface) {
      return [];
    }

    $imageStyleStorage = $this->entityTypeManager->getStorage('image_style');

    return [
      '#theme' => 'responsive_image',
      '#responsive_image_style_id' => 'responsive_' . $style,
      '#uri' => $file->getFileUri(),
      '#attributes' => [
        'loading' => 'lazy', // Lazy load images
        'decoding' => 'async', // Async decode
      ],
    ];
  }

  /**
   * Convert image เป็น WebP format
   */
  public function convertToWebP(string $sourcePath): ?string {
    if (!function_exists('imagewebp')) {
      return null; // GD extension ไม่รองรับ WebP
    }

    $extension = pathinfo($sourcePath, PATHINFO_EXTENSION);
    $webpPath = preg_replace('/\.' . $extension . '$/', '.webp', $sourcePath);

    $image = match (strtolower($extension)) {
      'jpg', 'jpeg' => imagecreatefromjpeg($sourcePath),
      'png' => imagecreatefrompng($sourcePath),
      'gif' => imagecreatefromgif($sourcePath),
      default => null,
    };

    if (!$image) {
      return null;
    }

    imagewebp($image, $webpPath, 80); // Quality 80%
    imagedestroy($image);

    return $webpPath;
  }

}
```

### 869. Critical CSS Generation

```php
<?php
// web/modules/custom/mymodule/src/Service/CriticalCssService.php

namespace Drupal\mymodule\Service;

use Drupal\Core\Cache\CacheBackendInterface;

/**
 * Service สำหรับจัดการ Critical CSS
 * Critical CSS คือ CSS ที่จำเป็นสำหรับ above-the-fold content
 */
class CriticalCssService {

  public function __construct(
    protected CacheBackendInterface $cache
  ) {}

  /**
   * ดึง Critical CSS สำหรับ path ที่กำหนด
   */
  public function getCriticalCss(string $path): string {
    $cid = 'mymodule:critical_css:' . md5($path);

    if ($cached = $this->cache->get($cid)) {
      return $cached->data;
    }

    $css = $this->generateCriticalCss($path);

    $this->cache->set($cid, $css, time() + 86400, ['mymodule_critical_css']);

    return $css;
  }

  /**
   * สร้าง Critical CSS (simplified version)
   * ใน production ควรใช้ tools อย่าง critical หรือ penthouse
   */
  private function generateCriticalCss(string $path): string {
    // ใน production environment ควรใช้ headless browser
    // สร้าง critical CSS โดย puppeteer หรือ playwright
    // แต่ในที่นี้ return pre-generated CSS

    $criticalCssPath = DRUPAL_ROOT . '/themes/custom/mytheme/css/critical/' .
      md5($path) . '.css';

    if (file_exists($criticalCssPath)) {
      return file_get_contents($criticalCssPath);
    }

    // Fallback: ใช้ base critical CSS
    $baseCriticalPath = DRUPAL_ROOT . '/themes/custom/mytheme/css/critical/base.css';
    return file_exists($baseCriticalPath) ? file_get_contents($baseCriticalPath) : '';
  }

}
```

### 870. HTTP/2 Server Push

```php
<?php
// web/modules/custom/mymodule/src/EventSubscriber/Http2PushSubscriber.php

namespace Drupal\mymodule\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

/**
 * Subscriber สำหรับ HTTP/2 Server Push
 * ส่ง critical resources ล่วงหน้าก่อนที่ browser จะ request
 */
class Http2PushSubscriber implements EventSubscriberInterface {

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents(): array {
    return [
      KernelEvents::RESPONSE => ['onKernelResponse', -200],
    ];
  }

  /**
   * เพิ่ม Link headers สำหรับ HTTP/2 push
   */
  public function onKernelResponse(ResponseEvent $event): void {
    if (!$event->isMainRequest()) {
      return;
    }

    $response = $event->getResponse();
    $request = $event->getRequest();

    // เฉพาะ HTML responses
    $contentType = $response->headers->get('Content-Type', '');
    if (!str_contains($contentType, 'text/html')) {
      return;
    }

    $pushLinks = [];

    // Push critical CSS
    $pushLinks[] = '</themes/custom/mytheme/css/critical.css>; rel=preload; as=style';

    // Push main JavaScript
    $pushLinks[] = '</themes/custom/mytheme/js/main.js>; rel=preload; as=script';

    // Push logo image
    $pushLinks[] = '</themes/custom/mytheme/images/logo.svg>; rel=preload; as=image';

    // เพิ่ม Link header
    foreach ($pushLinks as $link) {
      $response->headers->set('Link', $link, FALSE); // FALSE = append ไม่ replace
    }
  }

}
```

---

## ขั้นตอนที่ 871-875: Horizontal Scaling

### 871. Architecture ของ Horizontal Scaling

```
                    ┌─────────────────┐
                    │   Load Balancer  │
                    │  (HAProxy/Nginx) │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────▼──┐  ┌────────▼──┐  ┌───────▼───┐
    │  Drupal 1  │  │  Drupal 2  │  │  Drupal 3  │
    │  (App Node)│  │  (App Node)│  │  (App Node)│
    └─────────┬──┘  └────────┬──┘  └───────┬───┘
              │              │              │
              └──────────────┼──────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────▼──┐  ┌────────▼──┐  ┌───────▼───┐
    │   MariaDB   │  │   Redis   │  │  NFS/GlusterFS│
    │  (Primary)  │  │  Cluster  │  │  (Shared Files)│
    └────────────┘  └──────────┘  └──────────────┘
```

### 872. Shared File System Configuration

```php
<?php
// web/sites/default/settings.php - Shared File Configuration

// ใช้ NFS mount สำหรับ shared files
$settings['file_public_path'] = 'sites/default/files';
$settings['file_private_path'] = '/mnt/shared/drupal/private';
$settings['file_temp_path'] = '/tmp';

// Custom stream wrapper สำหรับ S3 (ถ้าใช้ S3)
// ต้องติดตั้ง s3fs module
$settings['s3fs.access_key'] = getenv('AWS_ACCESS_KEY_ID');
$settings['s3fs.secret_key'] = getenv('AWS_SECRET_ACCESS_KEY');
$settings['s3fs.bucket'] = getenv('S3_BUCKET_NAME');
$settings['s3fs.region'] = getenv('AWS_REGION');
```

### 873. Database Replication Configuration

```php
<?php
// web/sites/default/settings.php - Database Replication

// Primary database (writes)
$databases['default']['default'] = [
  'driver' => 'mysql',
  'database' => getenv('DB_NAME'),
  'username' => getenv('DB_USER'),
  'password' => getenv('DB_PASSWORD'),
  'host' => getenv('DB_PRIMARY_HOST'),
  'port' => '3306',
  'prefix' => '',
  'collation' => 'utf8mb4_general_ci',
];

// Read replicas (reads)
$databases['default']['replica'][] = [
  'driver' => 'mysql',
  'database' => getenv('DB_NAME'),
  'username' => getenv('DB_REPLICA_USER'),
  'password' => getenv('DB_REPLICA_PASSWORD'),
  'host' => getenv('DB_REPLICA_1_HOST'),
  'port' => '3306',
  'prefix' => '',
  'collation' => 'utf8mb4_general_ci',
];

$databases['default']['replica'][] = [
  'driver' => 'mysql',
  'database' => getenv('DB_NAME'),
  'username' => getenv('DB_REPLICA_USER'),
  'password' => getenv('DB_REPLICA_PASSWORD'),
  'host' => getenv('DB_REPLICA_2_HOST'),
  'port' => '3306',
  'prefix' => '',
  'collation' => 'utf8mb4_general_ci',
];
```

### 874. Session Handling ใน Multi-Server Setup

```php
<?php
// web/sites/default/settings.php

// ใช้ Redis สำหรับ session ใน multi-server setup
// ป้องกัน session loss เมื่อ request ไปยัง server ต่างกัน
ini_set('session.save_handler', 'redis');
ini_set('session.save_path', implode(',', [
  'tcp://redis-1:6379?auth=' . getenv('REDIS_PASSWORD') . '&database=0&weight=1',
  'tcp://redis-2:6379?auth=' . getenv('REDIS_PASSWORD') . '&database=0&weight=1',
]));

// Session cookie settings
ini_set('session.cookie_secure', '1'); // HTTPS only
ini_set('session.cookie_httponly', '1'); // No JS access
ini_set('session.cookie_samesite', 'Strict');
ini_set('session.use_strict_mode', '1');
```

### 875. Load Balancer Configuration (HAProxy)

```
# /etc/haproxy/haproxy.cfg

global
    maxconn 50000
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    option  http-server-close
    option  forwardfor except 127.0.0.0/8
    option  redispatch
    retries 3
    timeout connect  5000ms
    timeout client  50000ms
    timeout server  50000ms

# Frontend: รับ HTTP traffic
frontend drupal_http
    bind *:80
    redirect scheme https code 301 if !{ ssl_fc }

# Frontend: รับ HTTPS traffic
frontend drupal_https
    bind *:443 ssl crt /etc/ssl/certs/cert.pem
    
    # Health check endpoint
    acl is_health_check path_beg /health
    use_backend health_backend if is_health_check
    
    # Admin paths ไปที่ server เดิมเสมอ (sticky session)
    acl is_admin path_beg /admin /user
    use_backend drupal_admin if is_admin
    
    # ทั่วไปใช้ round-robin
    default_backend drupal_pool

# Backend: Drupal nodes ทั่วไป
backend drupal_pool
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200
    
    # Sticky session based on cookie
    cookie SERVERID insert indirect nocache
    
    server drupal1 10.0.1.1:8080 check cookie drupal1
    server drupal2 10.0.1.2:8080 check cookie drupal2
    server drupal3 10.0.1.3:8080 check cookie drupal3

# Backend: สำหรับ admin operations (single server)
backend drupal_admin
    balance source
    server drupal1 10.0.1.1:8080 check

# Health check endpoint
backend health_backend
    server drupal1 10.0.1.1:8080

# Statistics page
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:securepassword
```

---

## ขั้นตอนที่ 876-880: Performance Monitoring

### 876. Performance Monitoring Module

```php
<?php
// web/modules/custom/mymodule/src/Service/PerformanceMonitorService.php

namespace Drupal\mymodule\Service;

use Drupal\Core\Database\Connection;
use Psr\Log\LoggerInterface;

/**
 * Service สำหรับ monitor performance metrics
 */
class PerformanceMonitorService {

  public function __construct(
    protected Connection $database,
    protected LoggerInterface $logger
  ) {}

  /**
   * บันทึก performance metrics
   */
  public function recordPageLoad(
    string $path,
    float $loadTime,
    int $memoryUsage,
    int $queryCount
  ): void {
    // บันทึกเฉพาะที่ใช้เวลานาน
    if ($loadTime > 2.0 || $queryCount > 50) {
      $this->database->insert('mymodule_performance_log')
        ->fields([
          'path' => $path,
          'load_time' => $loadTime,
          'memory_usage' => $memoryUsage,
          'query_count' => $queryCount,
          'timestamp' => time(),
        ])
        ->execute();

      if ($loadTime > 5.0) {
        $this->logger->warning(
          'Very slow page load: @path took @time seconds with @queries queries',
          [
            '@path' => $path,
            '@time' => round($loadTime, 2),
            '@queries' => $queryCount,
          ]
        );
      }
    }
  }

  /**
   * ดึง performance report
   */
  public function getPerformanceReport(int $hours = 24): array {
    $since = time() - ($hours * 3600);

    $results = $this->database->select('mymodule_performance_log', 'p')
      ->fields('p', ['path', 'load_time', 'memory_usage', 'query_count', 'timestamp'])
      ->condition('p.timestamp', $since, '>=')
      ->orderBy('p.load_time', 'DESC')
      ->range(0, 100)
      ->execute()
      ->fetchAll(\PDO::FETCH_ASSOC);

    // คำนวณ statistics
    $loadTimes = array_column($results, 'load_time');

    return [
      'total_requests' => count($results),
      'avg_load_time' => !empty($loadTimes) ? array_sum($loadTimes) / count($loadTimes) : 0,
      'max_load_time' => !empty($loadTimes) ? max($loadTimes) : 0,
      'p95_load_time' => $this->percentile($loadTimes, 95),
      'slow_pages' => $results,
    ];
  }

  /**
   * คำนวณ percentile
   */
  private function percentile(array $values, int $percentile): float {
    if (empty($values)) {
      return 0;
    }
    sort($values);
    $index = ceil(($percentile / 100) * count($values)) - 1;
    return $values[max(0, $index)];
  }

}
```

### 877. Drupal Performance Event Subscriber

```php
<?php
// web/modules/custom/mymodule/src/EventSubscriber/PerformanceSubscriber.php

namespace Drupal\mymodule\EventSubscriber;

use Drupal\mymodule\Service\PerformanceMonitorService;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\Event\TerminateEvent;
use Symfony\Component\HttpKernel\KernelEvents;

/**
 * Event subscriber สำหรับ performance monitoring
 */
class PerformanceSubscriber implements EventSubscriberInterface {

  /**
   * @var float
   */
  private float $startTime;

  /**
   * @var int
   */
  private int $startMemory;

  public function __construct(
    protected PerformanceMonitorService $performanceMonitor
  ) {}

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents(): array {
    return [
      KernelEvents::REQUEST => ['onKernelRequest', 999],
      KernelEvents::TERMINATE => ['onKernelTerminate', -999],
    ];
  }

  /**
   * บันทึกเวลาเริ่มต้น
   */
  public function onKernelRequest(RequestEvent $event): void {
    if (!$event->isMainRequest()) {
      return;
    }
    $this->startTime = microtime(TRUE);
    $this->startMemory = memory_get_usage(TRUE);
  }

  /**
   * บันทึก performance metrics หลัง request
   */
  public function onKernelTerminate(TerminateEvent $event): void {
    $request = $event->getRequest();
    $path = $request->getPathInfo();

    $loadTime = microtime(TRUE) - $this->startTime;
    $memoryUsage = memory_get_peak_usage(TRUE) - $this->startMemory;

    // นับ queries
    $queryCount = count(\Drupal::database()->getLogger()?->get('queries') ?? []);

    $this->performanceMonitor->recordPageLoad(
      $path,
      $loadTime,
      $memoryUsage,
      $queryCount
    );
  }

}
```

### 878. Drupal State API สำหรับ Performance Flags

```php
<?php
// web/modules/custom/mymodule/src/Controller/PerformanceController.php

namespace Drupal\mymodule\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\mymodule\Service\PerformanceMonitorService;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Symfony\Component\HttpFoundation\JsonResponse;

/**
 * Controller สำหรับ performance dashboard
 */
class PerformanceController extends ControllerBase {

  public static function create(ContainerInterface $container): static {
    $instance = parent::create($container);
    $instance->performanceMonitor = $container->get('mymodule.performance_monitor');
    return $instance;
  }

  /**
   * Performance dashboard
   */
  public function dashboard(): array {
    $report = $this->performanceMonitor->getPerformanceReport(24);

    return [
      '#theme' => 'mymodule_performance_dashboard',
      '#report' => $report,
      '#cache' => [
        'max-age' => 300,
        'contexts' => ['user.roles:administrator'],
      ],
    ];
  }

  /**
   * API endpoint สำหรับ metrics
   */
  public function metricsApi(): JsonResponse {
    $report = $this->performanceMonitor->getPerformanceReport(1);

    return new JsonResponse([
      'status' => 'ok',
      'metrics' => [
        'avg_response_time_ms' => round($report['avg_load_time'] * 1000, 2),
        'p95_response_time_ms' => round($report['p95_load_time'] * 1000, 2),
        'max_response_time_ms' => round($report['max_load_time'] * 1000, 2),
        'slow_page_count' => $report['total_requests'],
      ],
      'timestamp' => date('c'),
    ]);
  }

  /**
   * Toggle performance monitoring
   */
  public function toggleMonitoring(): JsonResponse {
    $current = \Drupal::state()->get('mymodule.monitor_slow_queries', FALSE);
    \Drupal::state()->set('mymodule.monitor_slow_queries', !$current);

    return new JsonResponse([
      'monitoring_enabled' => !$current,
    ]);
  }

}
```

### 879. Drush Commands สำหรับ Performance

```php
<?php
// web/modules/custom/mymodule/src/Commands/PerformanceCommands.php

namespace Drupal\mymodule\Commands;

use Drush\Commands\DrushCommands;
use Drupal\Core\Cache\CacheTagsInvalidatorInterface;

/**
 * Drush commands สำหรับ performance management
 */
class PerformanceCommands extends DrushCommands {

  public function __construct(
    protected CacheTagsInvalidatorInterface $cacheTagsInvalidator
  ) {}

  /**
   * Warm up cache สำหรับ important pages
   *
   * @command mymodule:cache-warmup
   * @aliases mm-cw
   * @usage mymodule:cache-warmup
   *   อุ่นเครื่อง cache สำหรับ pages สำคัญ
   */
  public function cacheWarmup(): void {
    $this->output()->writeln('Starting cache warmup...');

    // ดึง URLs ที่ต้องการ warm up
    $urls = $this->getImportantUrls();

    $client = \Drupal::httpClient();
    $baseUrl = \Drupal::request()->getSchemeAndHttpHost();

    $success = 0;
    $failed = 0;

    foreach ($urls as $url) {
      try {
        $fullUrl = $baseUrl . $url;
        $response = $client->get($fullUrl, [
          'timeout' => 30,
          'headers' => [
            'User-Agent' => 'Drupal Cache Warmer/1.0',
          ],
        ]);

        if ($response->getStatusCode() === 200) {
          $success++;
          $this->output()->writeln("✓ Warmed: $url");
        }
        else {
          $failed++;
          $this->output()->writeln("✗ Failed ($response->getStatusCode()): $url");
        }
      }
      catch (\Exception $e) {
        $failed++;
        $this->output()->writeln("✗ Error: $url - " . $e->getMessage());
      }
    }

    $this->output()->writeln("\nCache warmup complete: $success succeeded, $failed failed");
  }

  /**
   * แสดงรายงาน performance
   *
   * @command mymodule:perf-report
   * @aliases mm-pr
   * @option hours จำนวนชั่วโมงที่ต้องการดู (default: 24)
   * @usage mymodule:perf-report --hours=48
   *   แสดงรายงาน performance 48 ชั่วโมงที่ผ่านมา
   */
  public function performanceReport(array $options = ['hours' => 24]): void {
    $monitor = \Drupal::service('mymodule.performance_monitor');
    $report = $monitor->getPerformanceReport($options['hours']);

    $this->output()->writeln(sprintf(
      "\nPerformance Report (last %d hours):\n%s",
      $options['hours'],
      str_repeat('=', 50)
    ));

    $this->output()->writeln(sprintf(
      "Total slow requests: %d\nAvg load time: %.2f ms\nP95 load time: %.2f ms\nMax load time: %.2f ms",
      $report['total_requests'],
      $report['avg_load_time'] * 1000,
      $report['p95_load_time'] * 1000,
      $report['max_load_time'] * 1000
    ));

    if (!empty($report['slow_pages'])) {
      $this->output()->writeln("\nTop 10 Slowest Pages:");
      foreach (array_slice($report['slow_pages'], 0, 10) as $page) {
        $this->output()->writeln(sprintf(
          "  [%.2f ms] %s (queries: %d)",
          $page['load_time'] * 1000,
          $page['path'],
          $page['query_count']
        ));
      }
    }
  }

  /**
   * Flush cache แบบ smart (เฉพาะ cache ที่จำเป็น)
   *
   * @command mymodule:smart-flush
   * @aliases mm-sf
   */
  public function smartFlush(): void {
    // Flush เฉพาะ rendered content ไม่ flush config
    $this->cacheTagsInvalidator->invalidateTags(['rendered']);

    $this->output()->writeln('Smart cache flush completed (rendered content only)');
  }

  /**
   * ดึง URLs สำคัญที่ต้อง warm up
   */
  private function getImportantUrls(): array {
    $urls = ['/'];

    // เพิ่ม node URLs ที่มี traffic สูง
    $query = \Drupal::database()->select('node_field_data', 'n');
    $query->fields('n', ['nid']);
    $query->condition('n.status', 1);
    $query->condition('n.promote', 1);
    $query->orderBy('n.changed', 'DESC');
    $query->range(0, 50);

    $nids = $query->execute()->fetchCol();

    foreach ($nids as $nid) {
      $node = \Drupal::entityTypeManager()->getStorage('node')->load($nid);
      if ($node) {
        $urls[] = $node->toUrl()->toString();
      }
    }

    return $urls;
  }

}
```

### 880. Docker Configuration สำหรับ Drupal Scaling

```yaml
# docker-compose.yml สำหรับ development

version: '3.8'

services:
  drupal:
    build:
      context: .
      dockerfile: Dockerfile.drupal
    environment:
      - DRUPAL_DB_HOST=mariadb
      - DRUPAL_REDIS_HOST=redis
      - DRUPAL_SOLR_HOST=solr
    volumes:
      - ./web/sites/default/files:/var/www/html/web/sites/default/files
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512M
          cpus: '0.5'

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./web:/var/www/html/web:ro
    depends_on:
      - drupal

  mariadb:
    image: mariadb:10.11
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: drupal
      MYSQL_USER: drupal
      MYSQL_PASSWORD: drupalpassword
    volumes:
      - mariadb_data:/var/lib/mysql
      - ./mysql.cnf:/etc/mysql/conf.d/custom.cnf
    command: >
      --innodb-buffer-pool-size=1G
      --innodb-log-file-size=256M
      --max-connections=500

  redis:
    image: redis:7-alpine
    command: >
      redis-server
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --save ""
      --appendonly no
    volumes:
      - redis_data:/data

  varnish:
    image: varnish:7
    ports:
      - "6081:6081"
    volumes:
      - ./varnish.vcl:/etc/varnish/default.vcl
    environment:
      - VARNISH_SIZE=512m
    depends_on:
      - nginx

volumes:
  mariadb_data:
  redis_data:
```

---

## Workshop: Performance Audit และการปรับปรุง

### Workshop 1: ทำ Performance Audit

```bash
# ขั้นตอนที่ 1: ตรวจสอบ Drupal caching
drush php-eval "
  \$render_cache = \Drupal::cache('render');
  \$info = \Drupal::service('cache_tags.invalidator');
  echo 'Cache system active' . PHP_EOL;
"

# ขั้นตอนที่ 2: Enable query logging
drush php-eval "
  \Drupal::state()->set('mymodule.monitor_slow_queries', TRUE);
  echo 'Query monitoring enabled' . PHP_EOL;
"

# ขั้นตอนที่ 3: ตรวจสอบ CSS/JS aggregation
drush config-get system.performance

# ขั้นตอนที่ 4: เปิดใช้ aggregation
drush config-set system.performance css.preprocess 1 -y
drush config-set system.performance js.preprocess 1 -y

# ขั้นตอนที่ 5: ทดสอบ performance
drush mymodule:perf-report

# ขั้นตอนที่ 6: Warm up cache
drush mymodule:cache-warmup
```

### Workshop 2: Redis Monitoring

```bash
# ตรวจสอบ Redis memory usage
redis-cli INFO memory | grep used_memory_human

# ดู cache hit/miss ratio
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"

# Monitor real-time commands
redis-cli MONITOR

# ดู keys ทั้งหมด
redis-cli KEYS "drupal_prod_*" | wc -l

# ดู memory ของ key เฉพาะ
redis-cli MEMORY USAGE "drupal_prod_cache.render:node:1"

# Flush cache เฉพาะ
redis-cli KEYS "drupal_prod_cache.render:*" | xargs redis-cli DEL
```

### Workshop 3: Database Optimization

```sql
-- ตรวจสอบ slow queries (ต้องเปิด slow_query_log ใน MySQL)
SELECT * FROM mysql.slow_log
ORDER BY query_time DESC
LIMIT 20;

-- ดู queries ที่ใช้ time นาน
SHOW PROCESSLIST;

-- ตรวจสอบ table indexes
SHOW INDEX FROM node_field_data;

-- Analyze table สำหรับ statistics
ANALYZE TABLE node_field_data;
ANALYZE TABLE node__body;

-- ตรวจสอบ table fragmentation
SELECT 
    table_name,
    ROUND(data_length/1024/1024, 2) AS 'Data MB',
    ROUND(index_length/1024/1024, 2) AS 'Index MB',
    ROUND(data_free/1024/1024, 2) AS 'Free MB'
FROM information_schema.tables
WHERE table_schema = 'drupal'
ORDER BY data_free DESC;

-- Optimize fragmented tables
OPTIMIZE TABLE node_field_data;
OPTIMIZE TABLE cache_render;
```

### Workshop 4: Profiling ด้วย Xdebug

```php
<?php
// php.ini settings สำหรับ profiling

// เปิดใช้ Xdebug profiler
xdebug.mode=profile
xdebug.output_dir=/tmp/xdebug
xdebug.profiler_output_name=cachegrind.out.%t.%p

// ใน code: trigger profiling สำหรับ specific request
if (isset($_GET['XDEBUG_PROFILE'])) {
    xdebug_start_profiling('/tmp/xdebug/profile_' . time() . '.out');
}
```

```bash
# วิเคราะห์ profiling output ด้วย KCacheGrind
kcachegrind /tmp/xdebug/cachegrind.out.12345

# หรือใช้ webgrind (web-based)
# git clone https://github.com/jokkedk/webgrind.git
# เปิด http://localhost:8080/webgrind
```

---

## สรุปบทเรียน

### Performance Checklist สำหรับ Production

```
✅ Drupal Core
  □ เปิด CSS/JS aggregation
  □ เปิด Page cache
  □ เปิด Dynamic page cache
  □ เปิด BigPipe
  □ ตั้งค่า cache lifetime ที่เหมาะสม

✅ Cache Backend
  □ ใช้ Redis แทน database cache
  □ ตั้งค่า Redis maxmemory policy
  □ แยก cache bins ตามความสำคัญ
  □ ใช้ Redis สำหรับ session storage

✅ Database
  □ สร้าง indexes สำหรับ queries ที่ใช้บ่อย
  □ ใช้ database replicas สำหรับ read queries
  □ Optimize table ทุกสัปดาห์
  □ Monitor slow queries

✅ Reverse Proxy
  □ ติดตั้ง Varnish หน้า Drupal
  □ ตั้งค่า VCL อย่างถูกต้อง
  □ ใช้ Purge module
  □ Monitor cache hit ratio

✅ Assets
  □ Compress CSS/JS (gzip/brotli)
  □ Optimize images (WebP format)
  □ ใช้ CDN สำหรับ static assets
  □ Implement HTTP/2

✅ Infrastructure
  □ ใช้ PHP OPcache
  □ ตั้งค่า PHP memory limit ที่เหมาะสม
  □ ใช้ PHP-FPM workers ที่เพียงพอ
  □ Monitor server resources

✅ Monitoring
  □ ตั้งค่า alerting สำหรับ slow responses
  □ ติดตั้ง APM (New Relic/Datadog)
  □ Monitor cache hit rates
  □ ตรวจสอบ error logs ทุกวัน
```

---

## แบบทดสอบ (Quiz)

### คำถามที่ 1

**Q: Cache Context 'user' ใน Drupal ทำงานอย่างไร?**

A) สร้าง cache version เดียวสำหรับทุก user  
B) สร้าง cache version แยกสำหรับแต่ละ user ที่ login  
C) ไม่ cache เลยสำหรับ authenticated users  
D) Cache เฉพาะ anonymous users เท่านั้น  

**คำตอบ: B**

**อธิบาย:** Cache Context 'user' บอกให้ Drupal สร้าง cache version แยกสำหรับแต่ละ user ID ดังนั้น User A และ User B จะเห็น cache คนละ version กัน ใช้เมื่อเนื้อหาแตกต่างกันตาม user เช่น ชื่อผู้ใช้ หรือสิทธิ์ที่ต่างกัน

---

### คำถามที่ 2

**Q: BigPipe ช่วยปรับปรุง performance อย่างไร?**

A) บีบอัด HTML ก่อนส่ง  
B) ส่ง HTML skeleton ไปก่อนแล้วค่อยส่งส่วนที่คำนวณนานทีหลัง  
C) Cache ทั้งหน้าในหน่วยความจำ  
D) ลด HTTP requests โดยรวม CSS/JS  

**คำตอบ: B**

**อธิบาย:** BigPipe ใช้หลักการ Progressive Rendering โดยส่ง HTML skeleton ไปให้ Browser แสดงผลทันที ส่วนที่ต้องคำนวณนาน (เช่น personalized content) จะถูกส่งมาทีหลังผ่าน Streaming และ JavaScript แทนที่ placeholder ด้วยเนื้อหาจริง ทำให้ผู้ใช้เห็นหน้าเว็บเร็วขึ้นมาก

---

### คำถามที่ 3

**Q: ข้อใดเป็นวิธีที่ถูกต้องในการ invalidate cache สำหรับ node ที่ถูกแก้ไข?**

A) ล้าง cache ทั้งหมด (`drupal_flush_all_caches()`)  
B) Invalidate tag 'node:{nid}' โดยใช้ CacheTagsInvalidator  
C) ลบ cache files ด้วย command line  
D) Restart web server  

**คำตอบ: B**

**อธิบาย:** การ invalidate ด้วย cache tag 'node:{nid}' เป็นวิธีที่ถูกต้องและมีประสิทธิภาพที่สุด เพราะ:
1. ลบเฉพาะ cache ที่เกี่ยวข้องกับ node นั้น
2. ไม่กระทบ cache ของ nodes อื่น
3. Varnish และ Drupal cache ทั้งหมดที่มี tag นี้จะถูก invalidate พร้อมกัน

---

### คำถามที่ 4

**Q: ทำไมจึงควรใช้ Redis แทน Database cache สำหรับ Drupal production?**

A) Redis ถูกกว่า MySQL  
B) Redis เป็น in-memory storage ทำให้เร็วกว่ามาก และรองรับ distributed caching ได้ดีกว่า  
C) Redis ใช้งานง่ายกว่า  
D) MySQL ไม่รองรับ cache  

**คำตอบ: B**

**อธิบาย:** Redis มีข้อดีหลายประการ:
- In-memory storage → latency < 1ms เทียบกับ MySQL ที่ 10-100ms
- ไม่มี table lock → รองรับ high concurrency ได้ดี
- Built-in data expiry → จัดการ TTL อัตโนมัติ
- Clustering → scale horizontally ได้
- ใช้เป็น shared cache ระหว่าง multiple app nodes ได้

---

### คำถามที่ 5

**Q: ใน multi-server Drupal setup ทำไมจึงต้องใช้ Redis สำหรับ session storage?**

A) Redis เก็บ session ได้มากกว่า  
B) เพื่อให้ session ของ user สามารถ access ได้จาก app node ใดก็ได้ ป้องกัน session loss  
C) Redis เร็วกว่า file system  
D) ข้อ B และ C ถูกทั้งคู่  

**คำตอบ: D**

**อธิบาย:** ใน multi-server setup:
- ถ้าใช้ file-based sessions: User ที่ login ที่ Server A จะไม่สามารถ access Session ที่ Server B ได้ (Session Loss)
- ถ้าใช้ Redis: Session ถูกเก็บแบบ centralized ทำให้ทุก App Node access Session เดียวกันได้
- Redis ยังเร็วกว่า disk-based session storage มาก (ข้อ C ก็ถูก)
- ดังนั้น ข้อ D เป็นคำตอบที่ครบถ้วนที่สุด

---

## เอกสารอ้างอิง

- [Drupal Cache API Documentation](https://www.drupal.org/docs/drupal-apis/cache-api)
- [BigPipe in Drupal 8+](https://www.drupal.org/docs/8/core/modules/big-pipe)
- [Drupal Redis Module](https://www.drupal.org/project/redis)
- [Varnish Cache Documentation](https://varnish-cache.org/docs/)
- [Drupal Performance Benchmarking](https://www.drupal.org/docs/administering-a-drupal-site/security-in-drupal/performance-and-scalability)
- [PHP OPcache Configuration](https://www.php.net/manual/en/opcache.configuration.php)
- [MySQL Performance Tuning](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)

---

*Part 084 | ระดับมืออาชีพ | Drupal Performance & Scaling*
