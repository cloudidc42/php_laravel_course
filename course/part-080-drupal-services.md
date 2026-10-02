# Part 080: Drupal Services, Dependency Injection, Plugin System & Entity API

**ระดับ:** Advanced  
**เวลาเรียน:** 6-8 ชั่วโมง  
**Prerequisites:** Part 079 - Drupal Module Basics

---

## เป้าหมายของ Part นี้

1. เข้าใจ Drupal Service Container
2. ใช้ Dependency Injection (DI)
3. สร้าง Custom Services
4. ใช้ Plugin System
5. Entity API ขั้นสูง
6. Event Subscribers

---

## 1. Service Container

Drupal ใช้ Symfony Service Container สำหรับจัดการ Dependencies

### 1.1 ใช้ Services

```php
<?php
/**
 * วิธีเรียกใช้ Services ใน Drupal
 */

// วิธีที่ 1: Static Method (ไม่แนะนำในโค้ด OOP)
$entityTypeManager = \Drupal::entityTypeManager();
$currentUser       = \Drupal::currentUser();
$config            = \Drupal::config('system.site');
$database          = \Drupal::database();
$logger            = \Drupal::logger('my_module');
$messenger         = \Drupal::messenger();
$cache             = \Drupal::cache();

// วิธีที่ 2: Dependency Injection (แนะนำ)
// ใน Controller/Service/Plugin ให้รับ Services ผ่าน Constructor
```

### 1.2 Services ที่สำคัญ

```php
<?php
/**
 * Services ที่ใช้บ่อยใน Drupal
 */

// Entity Type Manager - จัดการ Entities
$entityTypeManager = \Drupal::service('entity_type.manager');
$nodes = $entityTypeManager->getStorage('node')->loadMultiple([1,2,3]);

// Current User
$currentUser = \Drupal::service('current_user');
$uid = $currentUser->id();
$roles = $currentUser->getRoles();

// Route Match - ดู Current Route
$routeMatch = \Drupal::service('current_route_match');
$route_name = $routeMatch->getRouteName();
$node = $routeMatch->getParameter('node');

// Path Current
$pathCurrent = \Drupal::service('path.current');
$current_path = $pathCurrent->getPath();

// Path Alias Manager
$pathAliasManager = \Drupal::service('path_alias.manager');
$alias = $pathAliasManager->getAliasByPath('/node/1');
$path  = $pathAliasManager->getPathByAlias('/my-page');

// Config Factory
$configFactory = \Drupal::service('config.factory');
$config = $configFactory->get('system.site');
$name   = $config->get('name');

// State (Key-Value Store สำหรับ Runtime State)
$state = \Drupal::service('state');
$state->set('my_module.last_run', time());
$last_run = $state->get('my_module.last_run');

// Cache
$cache = \Drupal::service('cache.default');
$cached = $cache->get('my_cache_key');
if (!$cached) {
    $data = expensive_operation();
    $cache->set('my_cache_key', $data, time() + 3600, ['node_list']);
} else {
    $data = $cached->data;
}

// Language Manager
$languageManager = \Drupal::service('language_manager');
$current_lang = $languageManager->getCurrentLanguage()->getId();

// File URL Generator
$fileUrlGenerator = \Drupal::service('file_url_generator');
$url = $fileUrlGenerator->generateAbsoluteString('public://images/photo.jpg');

// Module Handler
$moduleHandler = \Drupal::service('module_handler');
$is_enabled = $moduleHandler->moduleExists('views');

// Token Service
$token = \Drupal::service('token');
$replaced = $token->replace('[site:name] - [node:title]', ['node' => $node]);
```

---

## 2. Custom Services

### 2.1 services.yml

```yaml
# web/modules/custom/product_catalog/product_catalog.services.yml

services:
  # Main Product Service
  product_catalog.product_manager:
    class: Drupal\product_catalog\Service\ProductManager
    arguments:
      - '@entity_type.manager'
      - '@database'
      - '@cache.default'
      - '@logger.factory'
    tags:
      - { name: service_collector }

  # Price Calculator Service
  product_catalog.price_calculator:
    class: Drupal\product_catalog\Service\PriceCalculator
    arguments:
      - '@config.factory'
      - '@current_user'

  # Cart Service (Shared - ไม่ต้อง Singleton ใหม่ทุกครั้ง)
  product_catalog.cart:
    class: Drupal\product_catalog\Service\CartService
    arguments:
      - '@session'
      - '@current_user'
      - '@entity_type.manager'
    shared: true  # Default: สร้าง Instance เดียว

  # Event Subscriber
  product_catalog.event_subscriber:
    class: Drupal\product_catalog\EventSubscriber\ProductEventSubscriber
    arguments:
      - '@logger.factory'
    tags:
      - { name: event_subscriber }
```

### 2.2 Product Manager Service

```php
<?php
/**
 * File: src/Service/ProductManager.php
 */

namespace Drupal\product_catalog\Service;

use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Database\Connection;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Logger\LoggerChannelFactoryInterface;
use Drupal\Core\Logger\LoggerChannelInterface;

class ProductManager {

    private EntityTypeManagerInterface $entityTypeManager;
    private Connection $database;
    private CacheBackendInterface $cache;
    private LoggerChannelInterface $logger;

    public function __construct(
        EntityTypeManagerInterface $entityTypeManager,
        Connection $database,
        CacheBackendInterface $cache,
        LoggerChannelFactoryInterface $loggerFactory
    ) {
        $this->entityTypeManager = $entityTypeManager;
        $this->database          = $database;
        $this->cache             = $cache;
        $this->logger            = $loggerFactory->get('product_catalog');
    }

    /**
     * ดึง Featured Products
     */
    public function getFeaturedProducts(int $count = 4, ?int $category_id = NULL): array {
        $cache_key = "product_catalog:featured:{$count}:{$category_id}";
        $cached    = $this->cache->get($cache_key);

        if ($cached) {
            return $cached->data;
        }

        try {
            $query = $this->entityTypeManager
                ->getStorage('node')
                ->getQuery()
                ->condition('type', 'product')
                ->condition('status', 1)
                ->sort('created', 'DESC')
                ->range(0, $count)
                ->accessCheck(TRUE);

            if ($category_id) {
                $query->condition('field_product_category', $category_id);
            }

            $nids  = $query->execute();
            $nodes = $this->entityTypeManager
                ->getStorage('node')
                ->loadMultiple($nids);

            $products = [];
            foreach ($nodes as $node) {
                $products[] = $this->buildProductData($node);
            }

            $this->cache->set($cache_key, $products, time() + 3600, [
                'node_list:product',
                'taxonomy_term:' . $category_id,
            ]);

            return $products;

        } catch (\Exception $e) {
            $this->logger->error('Error fetching featured products: @message', [
                '@message' => $e->getMessage(),
            ]);
            return [];
        }
    }

    /**
     * ดึงสถิติ Sales
     */
    public function getProductStats(int $nid): array {
        $cache_key = "product_catalog:stats:{$nid}";
        $cached    = $this->cache->get($cache_key);

        if ($cached) {
            return $cached->data;
        }

        $result = $this->database->query(
            "SELECT COUNT(*) as total_orders, SUM(quantity) as total_sold
             FROM {product_order_items}
             WHERE product_id = :nid",
            [':nid' => $nid]
        )->fetchAssoc();

        $stats = [
            'total_orders' => (int) ($result['total_orders'] ?? 0),
            'total_sold'   => (int) ($result['total_sold'] ?? 0),
        ];

        $this->cache->set($cache_key, $stats, time() + 300, ["node:{$nid}"]);

        return $stats;
    }

    private function buildProductData(\Drupal\node\NodeInterface $node): array {
        return [
            'id'    => $node->id(),
            'title' => $node->getTitle(),
            'price' => (float) $node->get('field_price')->value,
            'sku'   => $node->get('field_sku')->value,
            'url'   => $node->toUrl()->toString(),
        ];
    }
}
```

### 2.3 Inject Service ใน Controller

```php
<?php
/**
 * Controller ที่ใช้ DI
 */

namespace Drupal\product_catalog\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\product_catalog\Service\ProductManager;
use Symfony\Component\DependencyInjection\ContainerInterface;

class ProductController extends ControllerBase {

    public function __construct(
        private readonly ProductManager $productManager,
    ) {}

    public static function create(ContainerInterface $container): static {
        return new static(
            $container->get('product_catalog.product_manager'),
        );
    }

    public function featured(): array {
        $products = $this->productManager->getFeaturedProducts(8);

        return [
            '#theme'    => 'product_list',
            '#products' => $products,
            '#cache'    => ['tags' => ['node_list:product']],
        ];
    }
}
```

---

## 3. Event Subscribers

```php
<?php
/**
 * File: src/EventSubscriber/ProductEventSubscriber.php
 */

namespace Drupal\product_catalog\EventSubscriber;

use Drupal\Core\Logger\LoggerChannelFactoryInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\KernelEvents;

class ProductEventSubscriber implements EventSubscriberInterface {

    private \Drupal\Core\Logger\LoggerChannelInterface $logger;

    public function __construct(LoggerChannelFactoryInterface $loggerFactory) {
        $this->logger = $loggerFactory->get('product_catalog');
    }

    public static function getSubscribedEvents(): array {
        return [
            KernelEvents::REQUEST => [
                ['onRequest', 100],
            ],
            // Drupal-specific Events
            'entity.presave' => 'onEntityPreSave',
        ];
    }

    public function onRequest(RequestEvent $event): void {
        $request = $event->getRequest();
        $path    = $request->getPathInfo();

        // Log Product Page Views
        if (str_starts_with($path, '/products/')) {
            $this->logger->info('Product page accessed: @path', ['@path' => $path]);
        }
    }

    public function onEntityPreSave(\Drupal\Core\Entity\EntityInterface $entity): void {
        if ($entity->getEntityTypeId() !== 'node' || $entity->bundle() !== 'product') {
            return;
        }

        // Log product saves
        $this->logger->info('Product saved: @title', [
            '@title' => $entity->label(),
        ]);
    }
}
```

---

## 4. Entity API ขั้นสูง

### 4.1 Custom Entity Type

```php
<?php
/**
 * File: src/Entity/ProductReview.php
 * 
 * สร้าง Custom Entity
 */

namespace Drupal\product_catalog\Entity;

use Drupal\Core\Entity\ContentEntityBase;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Field\BaseFieldDefinition;

/**
 * @ContentEntityType(
 *   id = "product_review",
 *   label = @Translation("Product Review"),
 *   base_table = "product_review",
 *   entity_keys = {
 *     "id" = "id",
 *     "uid" = "uid",
 *     "uuid" = "uuid",
 *   },
 *   handlers = {
 *     "storage" = "Drupal\Core\Entity\Sql\SqlContentEntityStorage",
 *     "list_builder" = "Drupal\Core\Entity\EntityListBuilder",
 *     "form" = {
 *       "default" = "Drupal\Core\Entity\ContentEntityForm",
 *       "delete" = "Drupal\Core\Entity\ContentEntityDeleteForm",
 *     },
 *   },
 *   links = {
 *     "canonical" = "/product-review/{product_review}",
 *     "edit-form" = "/product-review/{product_review}/edit",
 *     "delete-form" = "/product-review/{product_review}/delete",
 *   },
 * )
 */
class ProductReview extends ContentEntityBase {

    public static function baseFieldDefinitions(EntityTypeInterface $entity_type): array {
        $fields = parent::baseFieldDefinitions($entity_type);

        // Node Reference
        $fields['nid'] = BaseFieldDefinition::create('entity_reference')
            ->setLabel(t('Product'))
            ->setRequired(TRUE)
            ->setSetting('target_type', 'node')
            ->setSetting('handler_settings', ['target_bundles' => ['product']])
            ->setDisplayOptions('form', ['type' => 'entity_reference_autocomplete'])
            ->setDisplayConfigurable('form', TRUE);

        // Rating
        $fields['rating'] = BaseFieldDefinition::create('integer')
            ->setLabel(t('Rating'))
            ->setRequired(TRUE)
            ->setSetting('min', 1)
            ->setSetting('max', 5)
            ->setDisplayOptions('view', ['type' => 'number_integer'])
            ->setDisplayOptions('form', ['type' => 'number'])
            ->setDisplayConfigurable('view', TRUE)
            ->setDisplayConfigurable('form', TRUE);

        // Review Text
        $fields['review'] = BaseFieldDefinition::create('text_long')
            ->setLabel(t('Review'))
            ->setRequired(TRUE)
            ->setDisplayOptions('view', [
                'label' => 'above',
                'type'  => 'text_default',
            ])
            ->setDisplayOptions('form', ['type' => 'text_textarea'])
            ->setDisplayConfigurable('view', TRUE)
            ->setDisplayConfigurable('form', TRUE);

        // Author
        $fields['uid'] = BaseFieldDefinition::create('entity_reference')
            ->setLabel(t('Author'))
            ->setSetting('target_type', 'user')
            ->setDefaultValueCallback(static::class . '::getDefaultEntityOwner');

        // Created
        $fields['created'] = BaseFieldDefinition::create('created')
            ->setLabel(t('Created'));

        // Status
        $fields['status'] = BaseFieldDefinition::create('boolean')
            ->setLabel(t('Published'))
            ->setDefaultValue(FALSE);

        return $fields;
    }

    public function getProduct(): ?\Drupal\node\NodeInterface {
        return $this->get('nid')->entity;
    }

    public function getRating(): int {
        return (int) $this->get('rating')->value;
    }

    public function getReview(): string {
        return $this->get('review')->value ?? '';
    }

    public function isApproved(): bool {
        return (bool) $this->get('status')->value;
    }
}
```

### 4.2 Entity Query ขั้นสูง

```php
<?php
/**
 * Entity Query ขั้นสูง
 */

// Query ด้วย Multiple Conditions
$nids = \Drupal::entityQuery('node')
    ->condition('type', 'product')
    ->condition('status', 1)
    // Nested AND/OR
    ->condition(
        \Drupal::entityQuery('node')
            ->orConditionGroup()
            ->condition('field_price', 100, '<')
            ->condition('title', '%sale%', 'LIKE')
    )
    ->sort('field_price', 'ASC')
    ->range(0, 10)
    ->accessCheck(TRUE)
    ->execute();

// Aggregate Query
$result = \Drupal::entityQuery('node')
    ->condition('type', 'product')
    ->condition('status', 1)
    ->aggregate('field_price', 'AVG')
    ->execute();
$avg_price = $result[0]['field_price_value_avg'] ?? 0;

// Count
$count = \Drupal::entityQuery('node')
    ->condition('type', 'product')
    ->accessCheck(FALSE)
    ->count()
    ->execute();
```

---

## 5. Plugin System

```php
<?php
/**
 * File: src/Plugin/PaymentGateway/QrPayment.php
 * 
 * Custom Plugin
 */

namespace Drupal\product_catalog\Plugin\PaymentGateway;

use Drupal\Core\Plugin\PluginBase;
use Drupal\product_catalog\PaymentGatewayInterface;

/**
 * @PaymentGateway(
 *   id = "qr_payment",
 *   label = @Translation("QR Payment"),
 *   description = @Translation("Pay via QR Code"),
 * )
 */
class QrPayment extends PluginBase implements PaymentGatewayInterface {

    public function getLabel(): string {
        return $this->pluginDefinition['label'];
    }

    public function processPayment(array $order_data): array {
        // สร้าง QR Code
        $qr_url = $this->generateQrCode($order_data['amount'], $order_data['order_id']);

        return [
            'success' => TRUE,
            'qr_url'  => $qr_url,
            'message' => 'Scan QR to pay',
        ];
    }

    private function generateQrCode(float $amount, string $order_id): string {
        // PromptPay QR Format
        $promptpay_id = $this->configuration['promptpay_id'] ?? '';
        return "https://promptpay.io/{$promptpay_id}/{$amount}";
    }
}
```

```php
<?php
/**
 * Plugin Manager
 */

namespace Drupal\product_catalog;

use Drupal\Core\Plugin\DefaultPluginManager;
use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Extension\ModuleHandlerInterface;

class PaymentGatewayManager extends DefaultPluginManager {

    public function __construct(
        \Traversable $namespaces,
        CacheBackendInterface $cache_backend,
        ModuleHandlerInterface $module_handler
    ) {
        parent::__construct(
            'Plugin/PaymentGateway',          // Subdirectory
            $namespaces,
            $module_handler,
            PaymentGatewayInterface::class,   // Interface
            PaymentGateway::class             // Annotation Class
        );

        $this->alterInfo('payment_gateway_info');
        $this->setCacheBackend($cache_backend, 'payment_gateway_plugins');
    }
}
```

---

## Workshop: สร้าง Complete E-commerce Module

```php
<?php
/**
 * Cart Service - ตัวอย่างครบวงจร
 */

namespace Drupal\product_catalog\Service;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Session\AccountInterface;
use Symfony\Component\HttpFoundation\Session\SessionInterface;

class CartService {

    private array $cart = [];

    public function __construct(
        private readonly SessionInterface $session,
        private readonly AccountInterface $currentUser,
        private readonly EntityTypeManagerInterface $entityTypeManager
    ) {
        $this->loadCart();
    }

    private function loadCart(): void {
        $key = 'cart_' . $this->currentUser->id();
        $this->cart = $this->session->get($key, []);
    }

    private function saveCart(): void {
        $key = 'cart_' . $this->currentUser->id();
        $this->session->set($key, $this->cart);
    }

    public function addItem(int $product_id, int $quantity = 1): bool {
        $node = $this->entityTypeManager->getStorage('node')->load($product_id);
        if (!$node || $node->bundle() !== 'product') {
            return FALSE;
        }

        if (isset($this->cart[$product_id])) {
            $this->cart[$product_id]['quantity'] += $quantity;
        } else {
            $this->cart[$product_id] = [
                'product_id' => $product_id,
                'title'      => $node->getTitle(),
                'price'      => (float) $node->get('field_price')->value,
                'quantity'   => $quantity,
            ];
        }

        $this->saveCart();
        return TRUE;
    }

    public function removeItem(int $product_id): void {
        unset($this->cart[$product_id]);
        $this->saveCart();
    }

    public function getTotal(): float {
        return array_sum(array_map(
            fn($item) => $item['price'] * $item['quantity'],
            $this->cart
        ));
    }

    public function getItems(): array {
        return $this->cart;
    }

    public function clear(): void {
        $this->cart = [];
        $this->saveCart();
    }

    public function getCount(): int {
        return array_sum(array_column($this->cart, 'quantity'));
    }
}
```

---

## Quiz

**คำถามที่ 1:** Dependency Injection ใน Drupal ทำงานผ่านกลไกใด?

A) Global Variables  
B) Static Methods  
C) Service Container (Symfony)  
D) PHP magic methods  

**เฉลย: C) Drupal ใช้ Symfony Service Container inject dependencies**

---

**คำถามที่ 2:** `shared: true` ใน services.yml หมายความว่าอะไร?

A) Service นี้ใช้ได้ทุก Module  
B) สร้าง Instance เดียวตลอด Request (Singleton pattern)  
C) Cache ค่า Service ไว้  
D) อนุญาตให้ Override ได้  

**เฉลย: B) shared: true เป็น Default - สร้าง 1 instance ต่อ request**

---

**คำถามที่ 3:** `@ContentEntityType` Annotation ใช้ทำอะไร?

A) Mark class เป็น Service  
B) ลงทะเบียน Entity Type ใหม่กับ Drupal Entity System  
C) สร้าง Database Table อัตโนมัติ  
D) Define permissions  

**เฉลย: B) Annotation บอก Drupal Entity Type Manager เกี่ยวกับ Entity นี้**

---

**คำถามที่ 4:** Event Subscriber ใน Drupal ต้อง implement Interface ใด?

A) `PluginInterface`  
B) `EventSubscriberInterface`  
C) `HookInterface`  
D) `ServiceInterface`  

**เฉลย: B) `Symfony\Component\EventDispatcher\EventSubscriberInterface`**

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- Service Container และ Dependency Injection
- การสร้าง Custom Services
- Event Subscribers
- Custom Entity Types
- Plugin System

**ยินดีด้วย!** คุณผ่าน Part Drupal ทั้ง 5 Parts แล้ว ตอนนี้คุณมีพื้นฐาน Drupal ที่ครบครันสำหรับ Enterprise Development

---

## สรุปภาพรวม Drupal Series

| Part | หัวข้อ | สิ่งที่ได้เรียนรู้ |
|------|--------|-------------------|
| 076 | Installation | Composer, Drush, Settings |
| 077 | Content | Content Types, Fields, Taxonomy, Views |
| 078 | Theming | Twig, Libraries, Preprocess |
| 079 | Module Basics | Routing, Controllers, Forms |
| 080 | Services & DI | DI, Plugins, Entity API |

---

## แหล่งเรียนรู้เพิ่มเติม

- [Drupal.org Documentation](https://www.drupal.org/docs)
- [Drupal API Reference](https://api.drupal.org)
- [Drupalize.me](https://drupalize.me)
