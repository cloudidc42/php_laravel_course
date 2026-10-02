# Part 081: Drupal Commerce

## ระดับ: สูง | ขั้นตอนที่ 721-760

---

## วัตถุประสงค์การเรียนรู้

หลังจากเรียนจบบทนี้ ผู้เรียนจะสามารถ:

1. ติดตั้งและตั้งค่า Drupal Commerce บน Drupal 10 ได้อย่างถูกต้อง
2. สร้างและจัดการ Product Types และ Product Variations ได้
3. เข้าใจ Order Management Workflow และกระบวนการสั่งซื้อ
4. เชื่อมต่อ Payment Gateway โดยเฉพาะ Stripe Integration
5. ตั้งค่าระบบ Shipping Methods และ Rates
6. สร้าง Promotions และ Discounts ได้
7. พัฒนา Custom Commerce Module ได้
8. สร้างร้านค้าออนไลน์ที่สมบูรณ์แบบด้วย Drupal Commerce

---

## บทนำ: Drupal Commerce คืออะไร?

Drupal Commerce เป็น e-commerce framework ที่สร้างบน Drupal CMS ซึ่งได้รับความนิยมอย่างมากในการพัฒนาร้านค้าออนไลน์ระดับ enterprise โดยมีข้อดีดังนี้:

- **ความยืดหยุ่นสูง**: สามารถปรับแต่งได้ทุกส่วนของระบบ e-commerce
- **Integration ที่ดี**: เชื่อมต่อกับ payment gateways, shipping providers, และ third-party services ได้หลากหลาย
- **Scalability**: รองรับการขยายขนาดได้ดี เหมาะสำหรับธุรกิจที่กำลังเติบโต
- **Community Support**: มีชุมชนนักพัฒนาที่แข็งแกร่งและ modules เสริมมากมาย
- **Security**: ได้รับการดูแลด้านความปลอดภัยอย่างสม่ำเสมอ

---

## ส่วนที่ 1: การติดตั้งและตั้งค่า Drupal Commerce

### 1.1 ความต้องการของระบบ

ก่อนติดตั้ง Drupal Commerce ต้องมีสิ่งเหล่านี้:

- PHP 8.1 หรือสูงกว่า
- MySQL 5.7.8+ / MariaDB 10.3.7+ / PostgreSQL 10+
- Composer 2.x
- Drupal 10.x

### 1.2 การติดตั้งผ่าน Composer

```bash
# สร้าง Drupal project ใหม่
composer create-project drupal/recommended-project my-commerce-site
cd my-commerce-site

# ติดตั้ง Drupal Commerce
composer require drupal/commerce

# ติดตั้ง modules เสริมที่จำเป็น
composer require drupal/commerce_stripe
composer require drupal/commerce_shipping
composer require drupal/commerce_promotion

# ติดตั้ง development dependencies
composer require --dev drupal/devel drupal/admin_toolbar
```

### 1.3 การเปิดใช้งาน Modules

```bash
# เปิดใช้งาน Commerce modules หลัก
drush en commerce commerce_product commerce_order commerce_cart commerce_checkout commerce_payment commerce_tax

# เปิดใช้งาน Payment Gateway
drush en commerce_stripe

# เปิดใช้งาน Shipping
drush en commerce_shipping

# เปิดใช้งาน Promotions
drush en commerce_promotion

# Clear cache
drush cr
```

### 1.4 การตั้งค่า Store

```php
<?php
// web/modules/custom/my_commerce_setup/src/Setup/StoreSetup.php

namespace Drupal\my_commerce_setup\Setup;

use Drupal\commerce_store\Entity\Store;
use Drupal\commerce_price\Price;

/**
 * การตั้งค่าร้านค้าเริ่มต้น
 */
class StoreSetup {

  /**
   * สร้างร้านค้าใหม่
   */
  public function createStore(): Store {
    $store = Store::create([
      'type' => 'online',
      'uid' => 1,
      'name' => 'ร้านค้าออนไลน์ของฉัน',
      'mail' => 'shop@example.com',
      'address' => [
        'country_code' => 'TH',
        'address_line1' => '123 ถนนสุขุมวิท',
        'locality' => 'กรุงเทพมหานคร',
        'postal_code' => '10110',
      ],
      'default_currency' => 'THB',
      'timezone' => 'Asia/Bangkok',
      'billing_countries' => ['TH'],
    ]);
    $store->save();

    return $store;
  }

  /**
   * ตั้งค่า store เป็น default
   */
  public function setDefaultStore(Store $store): void {
    $store->setDefault(TRUE);
    $store->save();
  }

}
```

### 1.5 Configuration Management

```yaml
# config/sync/commerce_store.commerce_store.online.yml
uuid: '12345678-1234-1234-1234-123456789012'
langcode: th
status: true
dependencies:
  module:
    - commerce_store
name: 'ร้านค้าออนไลน์'
type: online
mail: shop@example.com
default_currency: THB
timezone: 'Asia/Bangkok'
address:
  country_code: TH
  address_line1: '123 ถนนสุขุมวิท'
  locality: กรุงเทพมหานคร
  postal_code: '10110'
billing_countries:
  - TH
```

---

## ส่วนที่ 2: Product Types และ Product Variations

### 2.1 การสร้าง Product Type ใหม่

```php
<?php
// web/modules/custom/my_products/src/ProductTypeManager.php

namespace Drupal\my_products;

use Drupal\commerce_product\Entity\ProductType;
use Drupal\commerce_product\Entity\ProductVariationType;

/**
 * จัดการ Product Types
 */
class ProductTypeManager {

  /**
   * สร้าง Product Type สำหรับเสื้อผ้า
   */
  public function createClothingProductType(): void {
    // สร้าง Variation Type ก่อน
    $variationType = ProductVariationType::create([
      'id' => 'clothing_variation',
      'label' => 'เสื้อผ้า Variation',
      'orderItemType' => 'default',
      'generateTitle' => TRUE,
      'traits' => ['purchasable'],
    ]);
    $variationType->save();

    // สร้าง Product Type
    $productType = ProductType::create([
      'id' => 'clothing',
      'label' => 'เสื้อผ้า',
      'description' => 'สินค้าประเภทเสื้อผ้าและเครื่องแต่งกาย',
      'variationType' => 'clothing_variation',
      'injectVariationFields' => TRUE,
      'traits' => [],
    ]);
    $productType->save();
  }

}
```

### 2.2 การสร้าง Custom Product Fields

```php
<?php
// web/modules/custom/my_products/my_products.install

use Drupal\field\Entity\FieldStorageConfig;
use Drupal\field\Entity\FieldConfig;

/**
 * Implements hook_install().
 */
function my_products_install(): void {
  // เพิ่ม field สีสินค้า
  _my_products_create_color_field();

  // เพิ่ม field ขนาดสินค้า
  _my_products_create_size_field();
}

/**
 * สร้าง field สำหรับสีสินค้า
 */
function _my_products_create_color_field(): void {
  // Field Storage
  if (!FieldStorageConfig::loadByName('commerce_product_variation', 'field_color')) {
    FieldStorageConfig::create([
      'field_name' => 'field_color',
      'entity_type' => 'commerce_product_variation',
      'type' => 'list_string',
      'settings' => [
        'allowed_values' => [
          'red' => 'แดง',
          'blue' => 'น้ำเงิน',
          'green' => 'เขียว',
          'black' => 'ดำ',
          'white' => 'ขาว',
          'yellow' => 'เหลือง',
        ],
      ],
    ])->save();
  }

  // Field Instance
  if (!FieldConfig::loadByName('commerce_product_variation', 'clothing_variation', 'field_color')) {
    FieldConfig::create([
      'field_name' => 'field_color',
      'entity_type' => 'commerce_product_variation',
      'bundle' => 'clothing_variation',
      'label' => 'สี',
      'required' => TRUE,
    ])->save();
  }
}

/**
 * สร้าง field สำหรับขนาดสินค้า
 */
function _my_products_create_size_field(): void {
  // Field Storage
  if (!FieldStorageConfig::loadByName('commerce_product_variation', 'field_size')) {
    FieldStorageConfig::create([
      'field_name' => 'field_size',
      'entity_type' => 'commerce_product_variation',
      'type' => 'list_string',
      'settings' => [
        'allowed_values' => [
          'XS' => 'XS (เล็กพิเศษ)',
          'S' => 'S (เล็ก)',
          'M' => 'M (กลาง)',
          'L' => 'L (ใหญ่)',
          'XL' => 'XL (ใหญ่พิเศษ)',
          'XXL' => 'XXL (ใหญ่มากพิเศษ)',
        ],
      ],
    ])->save();
  }

  // Field Instance
  if (!FieldConfig::loadByName('commerce_product_variation', 'clothing_variation', 'field_size')) {
    FieldConfig::create([
      'field_name' => 'field_size',
      'entity_type' => 'commerce_product_variation',
      'bundle' => 'clothing_variation',
      'label' => 'ขนาด',
      'required' => TRUE,
    ])->save();
  }
}
```

### 2.3 Product Variation Type ที่สมบูรณ์

```php
<?php
// web/modules/custom/my_products/src/Entity/ProductVariation/ClothingVariation.php

namespace Drupal\my_products\Entity\ProductVariation;

use Drupal\commerce_product\Entity\ProductVariation;

/**
 * กำหนด Entity Class สำหรับ Clothing Variation
 *
 * @ContentEntityType(
 *   id = "commerce_product_variation",
 *   bundle_label = @Translation("Product variation type"),
 *   handlers = {
 *     "storage" = "Drupal\commerce\CommerceContentEntityStorage",
 *     "view_builder" = "Drupal\Core\Entity\EntityViewBuilder",
 *     "views_data" = "Drupal\commerce_product\ProductVariationViewsData",
 *   },
 * )
 */
class ClothingVariation extends ProductVariation {

  /**
   * ดึงข้อมูลสีสินค้า
   */
  public function getColor(): ?string {
    if ($this->hasField('field_color') && !$this->get('field_color')->isEmpty()) {
      return $this->get('field_color')->value;
    }
    return NULL;
  }

  /**
   * ดึงข้อมูลขนาดสินค้า
   */
  public function getSize(): ?string {
    if ($this->hasField('field_size') && !$this->get('field_size')->isEmpty()) {
      return $this->get('field_size')->value;
    }
    return NULL;
  }

  /**
   * ตั้งค่าสีสินค้า
   */
  public function setColor(string $color): static {
    $this->set('field_color', $color);
    return $this;
  }

  /**
   * ตั้งค่าขนาดสินค้า
   */
  public function setSize(string $size): static {
    $this->set('field_size', $size);
    return $this;
  }

  /**
   * ดึงชื่อสินค้าพร้อมรายละเอียด
   */
  public function getTitle(): string {
    $title = parent::getTitle();
    $color = $this->getColor();
    $size = $this->getSize();

    $parts = array_filter([$title, $color, $size]);
    return implode(' - ', $parts);
  }

}
```

### 2.4 การสร้าง Product Programmatically

```php
<?php
// web/modules/custom/my_products/src/ProductFactory.php

namespace Drupal\my_products;

use Drupal\commerce_price\Price;
use Drupal\commerce_product\Entity\Product;
use Drupal\commerce_product\Entity\ProductVariation;
use Drupal\commerce_store\Entity\StoreInterface;

/**
 * Factory สำหรับสร้าง Products
 */
class ProductFactory {

  /**
   * สร้างสินค้าพร้อม variations
   */
  public function createClothingProduct(
    string $title,
    array $variations,
    StoreInterface $store
  ): Product {
    // สร้าง variations ก่อน
    $variationEntities = [];
    foreach ($variations as $variationData) {
      $variation = ProductVariation::create([
        'type' => 'clothing_variation',
        'sku' => $variationData['sku'],
        'price' => new Price($variationData['price'], 'THB'),
        'field_color' => $variationData['color'],
        'field_size' => $variationData['size'],
        'status' => TRUE,
      ]);
      $variation->save();
      $variationEntities[] = $variation;
    }

    // สร้าง Product
    $product = Product::create([
      'type' => 'clothing',
      'title' => $title,
      'stores' => [$store],
      'variations' => $variationEntities,
      'status' => TRUE,
    ]);
    $product->save();

    return $product;
  }

  /**
   * สร้าง Bundle Product (ชุดสินค้า)
   */
  public function createBundleProduct(
    string $title,
    array $items,
    float $bundlePrice,
    StoreInterface $store
  ): Product {
    $variation = ProductVariation::create([
      'type' => 'bundle',
      'sku' => 'BUNDLE-' . strtoupper(str_replace(' ', '-', $title)),
      'price' => new Price((string) $bundlePrice, 'THB'),
      'field_bundle_items' => $items,
      'status' => TRUE,
    ]);
    $variation->save();

    $product = Product::create([
      'type' => 'bundle',
      'title' => $title,
      'stores' => [$store],
      'variations' => [$variation],
      'status' => TRUE,
    ]);
    $product->save();

    return $product;
  }

}
```

---

## ส่วนที่ 3: Order Management Workflow

### 3.1 Order States และ Transitions

```php
<?php
// web/modules/custom/my_commerce/src/OrderWorkflow/CustomOrderWorkflow.php

namespace Drupal\my_commerce\OrderWorkflow;

use Drupal\commerce_order\Entity\OrderInterface;
use Drupal\state_machine\Plugin\Workflow\WorkflowInterface;

/**
 * Custom Order Workflow
 */
class CustomOrderWorkflow {

  /**
   * กำหนด states ของ order
   */
  public static function getStates(): array {
    return [
      'draft' => [
        'label' => 'ร่าง',
        'color' => '#grey',
      ],
      'pending' => [
        'label' => 'รอดำเนินการ',
        'color' => '#orange',
      ],
      'processing' => [
        'label' => 'กำลังดำเนินการ',
        'color' => '#blue',
      ],
      'packed' => [
        'label' => 'บรรจุสินค้าแล้ว',
        'color' => '#purple',
      ],
      'shipped' => [
        'label' => 'จัดส่งแล้ว',
        'color' => '#teal',
      ],
      'completed' => [
        'label' => 'เสร็จสมบูรณ์',
        'color' => '#green',
      ],
      'canceled' => [
        'label' => 'ยกเลิกแล้ว',
        'color' => '#red',
      ],
      'refunded' => [
        'label' => 'คืนเงินแล้ว',
        'color' => '#darkred',
      ],
    ];
  }

  /**
   * กำหนด transitions
   */
  public static function getTransitions(): array {
    return [
      'place' => [
        'label' => 'สั่งซื้อ',
        'from' => ['draft'],
        'to' => 'pending',
      ],
      'start_processing' => [
        'label' => 'เริ่มดำเนินการ',
        'from' => ['pending'],
        'to' => 'processing',
      ],
      'pack' => [
        'label' => 'บรรจุสินค้า',
        'from' => ['processing'],
        'to' => 'packed',
      ],
      'ship' => [
        'label' => 'จัดส่ง',
        'from' => ['packed'],
        'to' => 'shipped',
      ],
      'complete' => [
        'label' => 'เสร็จสิ้น',
        'from' => ['shipped'],
        'to' => 'completed',
      ],
      'cancel' => [
        'label' => 'ยกเลิก',
        'from' => ['pending', 'processing', 'packed'],
        'to' => 'canceled',
      ],
      'refund' => [
        'label' => 'คืนเงิน',
        'from' => ['completed'],
        'to' => 'refunded',
      ],
    ];
  }

}
```

### 3.2 Custom Order Processor

```php
<?php
// web/modules/custom/my_commerce/src/OrderProcessor/CustomOrderProcessor.php

namespace Drupal\my_commerce\OrderProcessor;

use Drupal\commerce_order\Adjustment;
use Drupal\commerce_order\Entity\OrderInterface;
use Drupal\commerce_order\OrderProcessorInterface;
use Drupal\commerce_price\Price;

/**
 * Custom Order Processor สำหรับคำนวณราคาพิเศษ
 *
 * @CommerceOrderProcessor(
 *   id = "my_commerce_custom_processor",
 *   label = "Custom Order Processor",
 *   priority = 300,
 * )
 */
class CustomOrderProcessor implements OrderProcessorInterface {

  /**
   * {@inheritdoc}
   */
  public function process(OrderInterface $order): void {
    // ตรวจสอบสมาชิก VIP
    $customer = $order->getCustomer();
    if ($customer && $this->isVipCustomer($customer)) {
      $this->applyVipDiscount($order);
    }

    // ตรวจสอบการสั่งซื้อจำนวนมาก
    $this->applyBulkDiscount($order);

    // คำนวณค่าจัดส่งฟรี
    $this->checkFreeShipping($order);
  }

  /**
   * ตรวจสอบว่าเป็นลูกค้า VIP หรือไม่
   */
  private function isVipCustomer($customer): bool {
    return $customer->hasRole('vip_customer');
  }

  /**
   * ใช้ส่วนลด VIP 10%
   */
  private function applyVipDiscount(OrderInterface $order): void {
    $subtotal = $order->getSubtotalPrice();
    if (!$subtotal) {
      return;
    }

    $discountAmount = $subtotal->multiply('0.10');
    $adjustment = new Adjustment([
      'type' => 'custom',
      'label' => 'ส่วนลดสมาชิก VIP (10%)',
      'amount' => $discountAmount->multiply('-1'),
      'source_id' => 'vip_discount',
    ]);

    $order->addAdjustment($adjustment);
  }

  /**
   * ส่วนลดสำหรับการสั่งซื้อจำนวนมาก
   */
  private function applyBulkDiscount(OrderInterface $order): void {
    $items = $order->getItems();
    $totalQuantity = 0;

    foreach ($items as $item) {
      $totalQuantity += $item->getQuantity();
    }

    if ($totalQuantity >= 10) {
      $subtotal = $order->getSubtotalPrice();
      if (!$subtotal) {
        return;
      }

      $discountRate = $totalQuantity >= 20 ? '0.15' : '0.05';
      $discountLabel = $totalQuantity >= 20 ? 'ส่วนลดสั่งมาก (15%)' : 'ส่วนลดสั่งมาก (5%)';

      $discountAmount = $subtotal->multiply($discountRate);
      $adjustment = new Adjustment([
        'type' => 'custom',
        'label' => $discountLabel,
        'amount' => $discountAmount->multiply('-1'),
        'source_id' => 'bulk_discount',
      ]);

      $order->addAdjustment($adjustment);
    }
  }

  /**
   * ตรวจสอบเงื่อนไขจัดส่งฟรี
   */
  private function checkFreeShipping(OrderInterface $order): void {
    $subtotal = $order->getSubtotalPrice();
    if (!$subtotal) {
      return;
    }

    // จัดส่งฟรีเมื่อสั่งซื้อ 1,000 บาทขึ้นไป
    $freeShippingThreshold = new Price('1000', 'THB');
    if ($subtotal->greaterThanOrEqual($freeShippingThreshold)) {
      // ล้าง shipping adjustments ที่มีอยู่
      $adjustments = $order->getAdjustments(['shipping']);
      foreach ($adjustments as $key => $adjustment) {
        $order->removeAdjustment($key);
      }

      // เพิ่ม free shipping label
      $adjustment = new Adjustment([
        'type' => 'shipping',
        'label' => 'จัดส่งฟรี (สั่งซื้อครบ 1,000 บาท)',
        'amount' => new Price('0', 'THB'),
        'source_id' => 'free_shipping',
      ]);
      $order->addAdjustment($adjustment);
    }
  }

}
```

### 3.3 Order Event Subscribers

```php
<?php
// web/modules/custom/my_commerce/src/EventSubscriber/OrderEventSubscriber.php

namespace Drupal\my_commerce\EventSubscriber;

use Drupal\commerce_order\Event\OrderEvent;
use Drupal\commerce_order\Event\OrderEvents;
use Drupal\commerce_order\Event\OrderItemEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Drupal\Core\Mail\MailManagerInterface;
use Drupal\Core\Language\LanguageManagerInterface;
use Psr\Log\LoggerInterface;

/**
 * Order Event Subscriber
 */
class OrderEventSubscriber implements EventSubscriberInterface {

  public function __construct(
    private readonly MailManagerInterface $mailManager,
    private readonly LanguageManagerInterface $languageManager,
    private readonly LoggerInterface $logger
  ) {}

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents(): array {
    return [
      OrderEvents::ORDER_PLACE => ['onOrderPlace', 0],
      OrderEvents::ORDER_PRESHIP => ['onOrderPreShip', 0],
      OrderEvents::ORDER_CANCEL => ['onOrderCancel', 0],
      OrderEvents::ORDER_ITEM_INSERT => ['onOrderItemInsert', 0],
      OrderEvents::ORDER_ITEM_UPDATE => ['onOrderItemUpdate', 0],
    ];
  }

  /**
   * เมื่อมีการสั่งซื้อใหม่
   */
  public function onOrderPlace(OrderEvent $event): void {
    $order = $event->getOrder();

    // บันทึก log
    $this->logger->info('คำสั่งซื้อใหม่ #@order_id จาก @customer', [
      '@order_id' => $order->id(),
      '@customer' => $order->getCustomer()->getDisplayName(),
    ]);

    // ส่งอีเมลยืนยัน
    $this->sendOrderConfirmation($order);

    // อัปเดตสต็อกสินค้า
    $this->updateInventory($order);
  }

  /**
   * เมื่อจัดส่งสินค้า
   */
  public function onOrderPreShip(OrderEvent $event): void {
    $order = $event->getOrder();

    // ส่งอีเมลแจ้งจัดส่ง
    $this->sendShippingNotification($order);
  }

  /**
   * เมื่อยกเลิกคำสั่งซื้อ
   */
  public function onOrderCancel(OrderEvent $event): void {
    $order = $event->getOrder();

    // คืนสต็อกสินค้า
    $this->restoreInventory($order);

    // ส่งอีเมลแจ้งยกเลิก
    $this->sendCancellationNotification($order);

    $this->logger->warning('ยกเลิกคำสั่งซื้อ #@order_id', [
      '@order_id' => $order->id(),
    ]);
  }

  /**
   * เมื่อเพิ่มสินค้าในตะกร้า
   */
  public function onOrderItemInsert(OrderItemEvent $event): void {
    $orderItem = $event->getOrderItem();

    $this->logger->debug('เพิ่มสินค้าในตะกร้า: @title x @qty', [
      '@title' => $orderItem->getTitle(),
      '@qty' => $orderItem->getQuantity(),
    ]);
  }

  /**
   * เมื่ออัปเดตสินค้าในตะกร้า
   */
  public function onOrderItemUpdate(OrderItemEvent $event): void {
    $orderItem = $event->getOrderItem();

    $this->logger->debug('อัปเดตสินค้าในตะกร้า: @title x @qty', [
      '@title' => $orderItem->getTitle(),
      '@qty' => $orderItem->getQuantity(),
    ]);
  }

  /**
   * ส่งอีเมลยืนยันคำสั่งซื้อ
   */
  private function sendOrderConfirmation($order): void {
    $customer = $order->getCustomer();
    if (!$customer || !$customer->getEmail()) {
      return;
    }

    $params = [
      'order' => $order,
      'subject' => 'ยืนยันคำสั่งซื้อ #' . $order->id(),
    ];

    $this->mailManager->mail(
      'my_commerce',
      'order_confirmation',
      $customer->getEmail(),
      $this->languageManager->getCurrentLanguage()->getId(),
      $params
    );
  }

  /**
   * ส่งอีเมลแจ้งการจัดส่ง
   */
  private function sendShippingNotification($order): void {
    $customer = $order->getCustomer();
    if (!$customer || !$customer->getEmail()) {
      return;
    }

    $params = [
      'order' => $order,
      'subject' => 'สินค้าของคุณถูกจัดส่งแล้ว #' . $order->id(),
    ];

    $this->mailManager->mail(
      'my_commerce',
      'order_shipped',
      $customer->getEmail(),
      $this->languageManager->getCurrentLanguage()->getId(),
      $params
    );
  }

  /**
   * ส่งอีเมลแจ้งยกเลิก
   */
  private function sendCancellationNotification($order): void {
    $customer = $order->getCustomer();
    if (!$customer || !$customer->getEmail()) {
      return;
    }

    $params = [
      'order' => $order,
      'subject' => 'ยกเลิกคำสั่งซื้อ #' . $order->id(),
    ];

    $this->mailManager->mail(
      'my_commerce',
      'order_canceled',
      $customer->getEmail(),
      $this->languageManager->getCurrentLanguage()->getId(),
      $params
    );
  }

  /**
   * อัปเดตสต็อกสินค้า
   */
  private function updateInventory($order): void {
    foreach ($order->getItems() as $item) {
      $purchasable = $item->getPurchasedEntity();
      if ($purchasable && $purchasable->hasField('field_stock')) {
        $currentStock = (int) $purchasable->get('field_stock')->value;
        $newStock = max(0, $currentStock - (int) $item->getQuantity());
        $purchasable->set('field_stock', $newStock);
        $purchasable->save();
      }
    }
  }

  /**
   * คืนสต็อกสินค้า
   */
  private function restoreInventory($order): void {
    foreach ($order->getItems() as $item) {
      $purchasable = $item->getPurchasedEntity();
      if ($purchasable && $purchasable->hasField('field_stock')) {
        $currentStock = (int) $purchasable->get('field_stock')->value;
        $newStock = $currentStock + (int) $item->getQuantity();
        $purchasable->set('field_stock', $newStock);
        $purchasable->save();
      }
    }
  }

}
```

---

## ส่วนที่ 4: Payment Gateways (Stripe Integration)

### 4.1 Stripe Payment Gateway Plugin

```php
<?php
// web/modules/custom/my_commerce/src/Plugin/Commerce/PaymentGateway/StripeCustomGateway.php

namespace Drupal\my_commerce\Plugin\Commerce\PaymentGateway;

use Drupal\commerce_payment\Entity\PaymentInterface;
use Drupal\commerce_payment\Exception\PaymentGatewayException;
use Drupal\commerce_payment\Plugin\Commerce\PaymentGateway\OnsitePaymentGatewayBase;
use Drupal\commerce_price\Price;
use Drupal\Core\Form\FormStateInterface;
use Stripe\Exception\ApiErrorException;
use Stripe\PaymentIntent;
use Stripe\Stripe;

/**
 * Custom Stripe Payment Gateway
 *
 * @CommercePaymentGateway(
 *   id = "my_stripe_gateway",
 *   label = "Stripe (Custom)",
 *   display_label = "บัตรเครดิต/เดบิต (Stripe)",
 *   forms = {
 *     "add-payment-method" = "Drupal\my_commerce\PluginForm\StripePaymentMethodAddForm",
 *   },
 *   payment_method_types = {"credit_card"},
 *   credit_card_types = {
 *     "amex", "dinersclub", "discover", "jcb", "maestro", "mastercard",
 *     "mir", "unionpay", "visa",
 *   },
 * )
 */
class StripeCustomGateway extends OnsitePaymentGatewayBase {

  /**
   * {@inheritdoc}
   */
  public function defaultConfiguration(): array {
    return [
      'secret_key' => '',
      'publishable_key' => '',
      'webhook_secret' => '',
    ] + parent::defaultConfiguration();
  }

  /**
   * {@inheritdoc}
   */
  public function buildConfigurationForm(array $form, FormStateInterface $form_state): array {
    $form = parent::buildConfigurationForm($form, $form_state);

    $form['secret_key'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Secret Key'),
      '#description' => $this->t('Stripe Secret Key (sk_test_... หรือ sk_live_...)'),
      '#default_value' => $this->configuration['secret_key'],
      '#required' => TRUE,
    ];

    $form['publishable_key'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Publishable Key'),
      '#description' => $this->t('Stripe Publishable Key (pk_test_... หรือ pk_live_...)'),
      '#default_value' => $this->configuration['publishable_key'],
      '#required' => TRUE,
    ];

    $form['webhook_secret'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Webhook Secret'),
      '#description' => $this->t('Webhook Signing Secret สำหรับ verify events'),
      '#default_value' => $this->configuration['webhook_secret'],
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function submitConfigurationForm(array &$form, FormStateInterface $form_state): void {
    parent::submitConfigurationForm($form, $form_state);

    if (!$form_state->getErrors()) {
      $values = $form_state->getValue($form['#parents']);
      $this->configuration['secret_key'] = $values['secret_key'];
      $this->configuration['publishable_key'] = $values['publishable_key'];
      $this->configuration['webhook_secret'] = $values['webhook_secret'];
    }
  }

  /**
   * สร้าง Payment Intent
   */
  public function createPaymentIntent(PaymentInterface $payment): array {
    $this->initializeStripe();

    $amount = $payment->getAmount();
    $amountInCents = $this->toMinorUnits($amount);

    try {
      $intent = PaymentIntent::create([
        'amount' => $amountInCents,
        'currency' => strtolower($amount->getCurrencyCode()),
        'payment_method' => $payment->getRemoteId(),
        'confirmation_method' => 'manual',
        'confirm' => TRUE,
        'metadata' => [
          'order_id' => $payment->getOrderId(),
          'store_id' => $payment->getOrder()->getStoreId(),
        ],
      ]);

      return [
        'client_secret' => $intent->client_secret,
        'status' => $intent->status,
        'id' => $intent->id,
      ];
    }
    catch (ApiErrorException $e) {
      throw new PaymentGatewayException('Stripe Error: ' . $e->getMessage(), $e->getCode(), $e);
    }
  }

  /**
   * {@inheritdoc}
   */
  public function createPayment(PaymentInterface $payment, bool $capture = TRUE): void {
    $this->assertPaymentState($payment, ['new']);
    $paymentMethod = $payment->getPaymentMethod();
    $this->assertPaymentMethod($paymentMethod);

    $this->initializeStripe();

    $amount = $payment->getAmount();
    $amountInCents = $this->toMinorUnits($amount);

    try {
      $paymentIntent = PaymentIntent::create([
        'amount' => $amountInCents,
        'currency' => strtolower($amount->getCurrencyCode()),
        'payment_method' => $paymentMethod->getRemoteId(),
        'confirm' => TRUE,
        'capture_method' => $capture ? 'automatic' : 'manual',
        'metadata' => [
          'order_id' => $payment->getOrderId(),
          'drupal_site' => \Drupal::request()->getHost(),
        ],
      ]);

      if ($paymentIntent->status === 'succeeded') {
        $payment->setState($capture ? 'completed' : 'authorization');
        $payment->setRemoteId($paymentIntent->id);
        $payment->save();
      }
      elseif ($paymentIntent->status === 'requires_action') {
        throw new PaymentGatewayException('ต้องการการยืนยันเพิ่มเติม (3D Secure)');
      }
      else {
        throw new PaymentGatewayException('การชำระเงินล้มเหลว: ' . $paymentIntent->status);
      }
    }
    catch (ApiErrorException $e) {
      throw new PaymentGatewayException('Stripe Error: ' . $e->getMessage());
    }
  }

  /**
   * {@inheritdoc}
   */
  public function capturePayment(PaymentInterface $payment, Price $amount = NULL): void {
    $this->assertPaymentState($payment, ['authorization']);
    $remoteId = $payment->getRemoteId();

    $this->initializeStripe();

    $amountToCapture = $amount ?: $payment->getAmount();
    $amountInCents = $this->toMinorUnits($amountToCapture);

    try {
      $intent = PaymentIntent::retrieve($remoteId);
      $intent->capture(['amount_to_capture' => $amountInCents]);

      $payment->setState('completed');
      $payment->setAmount($amountToCapture);
      $payment->save();
    }
    catch (ApiErrorException $e) {
      throw new PaymentGatewayException('ไม่สามารถเรียกเก็บเงินได้: ' . $e->getMessage());
    }
  }

  /**
   * {@inheritdoc}
   */
  public function voidPayment(PaymentInterface $payment): void {
    $this->assertPaymentState($payment, ['authorization']);
    $remoteId = $payment->getRemoteId();

    $this->initializeStripe();

    try {
      $intent = PaymentIntent::retrieve($remoteId);
      $intent->cancel();

      $payment->setState('authorization_voided');
      $payment->save();
    }
    catch (ApiErrorException $e) {
      throw new PaymentGatewayException('ไม่สามารถยกเลิกได้: ' . $e->getMessage());
    }
  }

  /**
   * {@inheritdoc}
   */
  public function refundPayment(PaymentInterface $payment, Price $amount = NULL): void {
    $this->assertPaymentState($payment, ['completed', 'partially_refunded']);

    $amountToRefund = $amount ?: $payment->getAmount();
    $this->assertRefundAmount($payment, $amountToRefund);

    $this->initializeStripe();

    $amountInCents = $this->toMinorUnits($amountToRefund);

    try {
      \Stripe\Refund::create([
        'payment_intent' => $payment->getRemoteId(),
        'amount' => $amountInCents,
        'reason' => 'requested_by_customer',
      ]);

      $oldRefundedAmount = $payment->getRefundedAmount();
      $newRefundedAmount = $oldRefundedAmount->add($amountToRefund);
      $payment->setRefundedAmount($newRefundedAmount);

      if ($newRefundedAmount->lessThan($payment->getAmount())) {
        $payment->setState('partially_refunded');
      }
      else {
        $payment->setState('refunded');
      }
      $payment->save();
    }
    catch (ApiErrorException $e) {
      throw new PaymentGatewayException('ไม่สามารถคืนเงินได้: ' . $e->getMessage());
    }
  }

  /**
   * Initialize Stripe SDK
   */
  private function initializeStripe(): void {
    Stripe::setApiKey($this->configuration['secret_key']);
    Stripe::setAppInfo(
      'Drupal Commerce Custom Gateway',
      '1.0.0',
      'https://example.com'
    );
  }

  /**
   * แปลงราคาเป็น minor units (satangs สำหรับ THB)
   */
  private function toMinorUnits(Price $price): int {
    $amount = $price->getNumber();
    // THB ใช้ 2 decimal places
    return (int) round((float) $amount * 100);
  }

}
```

### 4.2 Stripe Webhook Handler

```php
<?php
// web/modules/custom/my_commerce/src/Controller/StripeWebhookController.php

namespace Drupal\my_commerce\Controller;

use Drupal\Core\Controller\ControllerBase;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Stripe\Exception\SignatureVerificationException;
use Stripe\Webhook;
use Psr\Log\LoggerInterface;

/**
 * Controller สำหรับ Stripe Webhooks
 */
class StripeWebhookController extends ControllerBase {

  public function __construct(
    private readonly LoggerInterface $logger,
    private readonly string $webhookSecret
  ) {}

  /**
   * Handle Stripe Webhook
   */
  public function handleWebhook(Request $request): Response {
    $payload = $request->getContent();
    $sigHeader = $request->headers->get('Stripe-Signature');

    // Verify webhook signature
    try {
      $event = Webhook::constructEvent($payload, $sigHeader, $this->webhookSecret);
    }
    catch (SignatureVerificationException $e) {
      $this->logger->error('Stripe webhook signature verification failed: @msg', [
        '@msg' => $e->getMessage(),
      ]);
      return new Response('Invalid signature', 400);
    }
    catch (\UnexpectedValueException $e) {
      return new Response('Invalid payload', 400);
    }

    // Handle events
    switch ($event->type) {
      case 'payment_intent.succeeded':
        $this->handlePaymentSucceeded($event->data->object);
        break;

      case 'payment_intent.payment_failed':
        $this->handlePaymentFailed($event->data->object);
        break;

      case 'charge.refunded':
        $this->handleChargeRefunded($event->data->object);
        break;

      case 'customer.subscription.updated':
        $this->handleSubscriptionUpdated($event->data->object);
        break;

      default:
        $this->logger->debug('Unhandled Stripe event: @type', [
          '@type' => $event->type,
        ]);
    }

    return new Response('OK', 200);
  }

  /**
   * จัดการเมื่อชำระเงินสำเร็จ
   */
  private function handlePaymentSucceeded($paymentIntent): void {
    $orderId = $paymentIntent->metadata->order_id ?? NULL;
    if (!$orderId) {
      return;
    }

    $this->logger->info('Payment succeeded for order #@order_id', [
      '@order_id' => $orderId,
    ]);

    // อัปเดต order state
    $order = \Drupal::entityTypeManager()
      ->getStorage('commerce_order')
      ->load($orderId);

    if ($order && $order->getState()->getId() === 'pending') {
      $order->getState()->applyTransitionById('place');
      $order->save();
    }
  }

  /**
   * จัดการเมื่อชำระเงินล้มเหลว
   */
  private function handlePaymentFailed($paymentIntent): void {
    $orderId = $paymentIntent->metadata->order_id ?? NULL;
    $this->logger->error('Payment failed for order #@order_id: @reason', [
      '@order_id' => $orderId ?? 'unknown',
      '@reason' => $paymentIntent->last_payment_error->message ?? 'Unknown error',
    ]);
  }

  /**
   * จัดการเมื่อมีการคืนเงิน
   */
  private function handleChargeRefunded($charge): void {
    $this->logger->info('Charge refunded: @id', ['@id' => $charge->id]);
  }

  /**
   * จัดการเมื่อ subscription อัปเดต
   */
  private function handleSubscriptionUpdated($subscription): void {
    $this->logger->info('Subscription updated: @id', ['@id' => $subscription->id]);
  }

}
```

---

## ส่วนที่ 5: Shipping Methods และ Rates

### 5.1 Custom Shipping Method Plugin

```php
<?php
// web/modules/custom/my_commerce/src/Plugin/Commerce/ShippingMethod/ThaiPostShipping.php

namespace Drupal\my_commerce\Plugin\Commerce\ShippingMethod;

use Drupal\commerce_price\Price;
use Drupal\commerce_shipping\Entity\ShipmentInterface;
use Drupal\commerce_shipping\Plugin\Commerce\ShippingMethod\ShippingMethodBase;
use Drupal\commerce_shipping\ShippingRate;
use Drupal\commerce_shipping\ShippingService;
use Drupal\Core\Form\FormStateInterface;

/**
 * Thai Post Shipping Method
 *
 * @CommerceShippingMethod(
 *   id = "thai_post",
 *   label = @Translation("ไปรษณีย์ไทย"),
 *   services = {
 *     "ems" = @Translation("EMS (ด่วนพิเศษ)"),
 *     "registered" = @Translation("ลงทะเบียน"),
 *     "parcel" = @Translation("พัสดุ"),
 *   },
 * )
 */
class ThaiPostShipping extends ShippingMethodBase {

  /**
   * {@inheritdoc}
   */
  public function defaultConfiguration(): array {
    return [
      'ems_base_rate' => '80',
      'registered_base_rate' => '40',
      'parcel_base_rate' => '30',
      'weight_rate' => '5',
      'free_shipping_threshold' => '1000',
    ] + parent::defaultConfiguration();
  }

  /**
   * {@inheritdoc}
   */
  public function buildConfigurationForm(array $form, FormStateInterface $form_state): array {
    $form = parent::buildConfigurationForm($form, $form_state);

    $form['rates'] = [
      '#type' => 'fieldset',
      '#title' => $this->t('อัตราค่าจัดส่ง'),
    ];

    $form['rates']['ems_base_rate'] = [
      '#type' => 'number',
      '#title' => $this->t('ค่า EMS เริ่มต้น (บาท)'),
      '#default_value' => $this->configuration['ems_base_rate'],
      '#min' => 0,
      '#step' => 0.01,
    ];

    $form['rates']['registered_base_rate'] = [
      '#type' => 'number',
      '#title' => $this->t('ค่าลงทะเบียนเริ่มต้น (บาท)'),
      '#default_value' => $this->configuration['registered_base_rate'],
      '#min' => 0,
    ];

    $form['rates']['parcel_base_rate'] = [
      '#type' => 'number',
      '#title' => $this->t('ค่าพัสดุเริ่มต้น (บาท)'),
      '#default_value' => $this->configuration['parcel_base_rate'],
      '#min' => 0,
    ];

    $form['rates']['weight_rate'] = [
      '#type' => 'number',
      '#title' => $this->t('ค่าน้ำหนักต่อ 100 กรัม (บาท)'),
      '#default_value' => $this->configuration['weight_rate'],
      '#min' => 0,
    ];

    $form['free_shipping_threshold'] = [
      '#type' => 'number',
      '#title' => $this->t('ยอดสั่งซื้อขั้นต่ำสำหรับจัดส่งฟรี (บาท)'),
      '#description' => $this->t('ใส่ 0 เพื่อปิดการจัดส่งฟรี'),
      '#default_value' => $this->configuration['free_shipping_threshold'],
      '#min' => 0,
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function calculateRates(ShipmentInterface $shipment): array {
    $rates = [];
    $weight = $this->getShipmentWeight($shipment);
    $orderTotal = $shipment->getOrder()->getSubtotalPrice();

    // ตรวจสอบจัดส่งฟรี
    $freeThreshold = (float) $this->configuration['free_shipping_threshold'];
    if ($freeThreshold > 0 && $orderTotal) {
      $thresholdPrice = new Price((string) $freeThreshold, 'THB');
      if ($orderTotal->greaterThanOrEqual($thresholdPrice)) {
        $rates[] = new ShippingRate([
          'shipping_method_id' => $this->parentEntity->id(),
          'service' => new ShippingService('free', 'จัดส่งฟรี'),
          'amount' => new Price('0', 'THB'),
        ]);
        return $rates;
      }
    }

    // คำนวณค่าจัดส่งตามบริการ
    foreach (['ems', 'registered', 'parcel'] as $service) {
      $rate = $this->calculateServiceRate($service, $weight);
      $rates[] = new ShippingRate([
        'shipping_method_id' => $this->parentEntity->id(),
        'service' => new ShippingService($service, $this->services[$service]),
        'amount' => new Price((string) $rate, 'THB'),
      ]);
    }

    return $rates;
  }

  /**
   * คำนวณค่าบริการตามน้ำหนัก
   */
  private function calculateServiceRate(string $service, float $weightInGrams): float {
    $baseRate = (float) $this->configuration[$service . '_base_rate'];
    $weightRate = (float) $this->configuration['weight_rate'];

    // คำนวณค่าน้ำหนักเพิ่ม (ทุก 100 กรัม)
    $extraWeight = max(0, $weightInGrams - 100);
    $extraWeightCharge = ceil($extraWeight / 100) * $weightRate;

    return $baseRate + $extraWeightCharge;
  }

  /**
   * ดึงน้ำหนักของ shipment
   */
  private function getShipmentWeight(ShipmentInterface $shipment): float {
    $weight = $shipment->getWeight();
    if ($weight) {
      // แปลงเป็น grams
      return match ($weight->getUnit()) {
        'kg' => $weight->getNumber() * 1000,
        'lb' => $weight->getNumber() * 453.592,
        'oz' => $weight->getNumber() * 28.3495,
        default => $weight->getNumber(), // ถือว่าเป็น grams
      };
    }
    return 500; // น้ำหนักเริ่มต้น 500 กรัม
  }

}
```

---

## ส่วนที่ 6: Promotions และ Discounts

### 6.1 Custom Condition Plugin

```php
<?php
// web/modules/custom/my_commerce/src/Plugin/Commerce/Condition/CustomerBirthday.php

namespace Drupal\my_commerce\Plugin\Commerce\Condition;

use Drupal\commerce\Plugin\Commerce\Condition\ConditionBase;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Form\FormStateInterface;

/**
 * Condition สำหรับวันเกิดลูกค้า
 *
 * @CommerceCondition(
 *   id = "order_customer_birthday",
 *   label = @Translation("วันเกิดของลูกค้า"),
 *   display_label = @Translation("ตรงกับวันเกิดลูกค้า"),
 *   category = @Translation("ลูกค้า"),
 *   entity_type = "commerce_order",
 * )
 */
class CustomerBirthday extends ConditionBase {

  /**
   * {@inheritdoc}
   */
  public function defaultConfiguration(): array {
    return [
      'days_before' => 0,
      'days_after' => 0,
    ] + parent::defaultConfiguration();
  }

  /**
   * {@inheritdoc}
   */
  public function buildConfigurationForm(array $form, FormStateInterface $form_state): array {
    $form = parent::buildConfigurationForm($form, $form_state);

    $form['days_before'] = [
      '#type' => 'number',
      '#title' => $this->t('วันก่อนวันเกิด'),
      '#default_value' => $this->configuration['days_before'],
      '#min' => 0,
      '#max' => 30,
    ];

    $form['days_after'] = [
      '#type' => 'number',
      '#title' => $this->t('วันหลังวันเกิด'),
      '#default_value' => $this->configuration['days_after'],
      '#min' => 0,
      '#max' => 30,
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function evaluate(EntityInterface $entity): bool {
    $this->assertEntity($entity);
    $order = $entity;
    $customer = $order->getCustomer();

    if (!$customer || $customer->isAnonymous()) {
      return FALSE;
    }

    // ตรวจสอบ field วันเกิด
    if (!$customer->hasField('field_birthday') || $customer->get('field_birthday')->isEmpty()) {
      return FALSE;
    }

    $birthday = new \DateTime($customer->get('field_birthday')->value);
    $today = new \DateTime();

    // ตรวจสอบว่าวันนี้อยู่ในช่วงวันเกิดหรือไม่
    $birthdayThisYear = new \DateTime(
      $today->format('Y') . '-' . $birthday->format('m-d')
    );

    $daysBefore = (int) $this->configuration['days_before'];
    $daysAfter = (int) $this->configuration['days_after'];

    $startDate = clone $birthdayThisYear;
    $startDate->modify("-{$daysBefore} days");

    $endDate = clone $birthdayThisYear;
    $endDate->modify("+{$daysAfter} days");

    return $today >= $startDate && $today <= $endDate;
  }

}
```

### 6.2 Custom Offer Plugin

```php
<?php
// web/modules/custom/my_commerce/src/Plugin/Commerce/PromotionOffer/BuyXGetYFree.php

namespace Drupal\my_commerce\Plugin\Commerce\PromotionOffer;

use Drupal\commerce_order\Adjustment;
use Drupal\commerce_order\Entity\OrderItemInterface;
use Drupal\commerce_price\Price;
use Drupal\commerce_promotion\Entity\PromotionInterface;
use Drupal\commerce_promotion\Plugin\Commerce\PromotionOffer\OrderItemPromotionOfferBase;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Form\FormStateInterface;

/**
 * Buy X Get Y Free Offer
 *
 * @CommercePromotionOffer(
 *   id = "order_item_buy_x_get_y",
 *   label = @Translation("ซื้อ X ชิ้น แถม Y ชิ้น"),
 *   entity_type = "commerce_order_item",
 * )
 */
class BuyXGetYFree extends OrderItemPromotionOfferBase {

  /**
   * {@inheritdoc}
   */
  public function defaultConfiguration(): array {
    return [
      'buy_quantity' => 2,
      'get_quantity' => 1,
      'max_sets' => 0,
    ] + parent::defaultConfiguration();
  }

  /**
   * {@inheritdoc}
   */
  public function buildConfigurationForm(array $form, FormStateInterface $form_state): array {
    $form = parent::buildConfigurationForm($form, $form_state);

    $form['buy_quantity'] = [
      '#type' => 'number',
      '#title' => $this->t('จำนวนที่ต้องซื้อ (X)'),
      '#default_value' => $this->configuration['buy_quantity'],
      '#min' => 1,
      '#required' => TRUE,
    ];

    $form['get_quantity'] = [
      '#type' => 'number',
      '#title' => $this->t('จำนวนที่ได้รับฟรี (Y)'),
      '#default_value' => $this->configuration['get_quantity'],
      '#min' => 1,
      '#required' => TRUE,
    ];

    $form['max_sets'] = [
      '#type' => 'number',
      '#title' => $this->t('จำนวนครั้งสูงสุด (0 = ไม่จำกัด)'),
      '#default_value' => $this->configuration['max_sets'],
      '#min' => 0,
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function apply(EntityInterface $entity, PromotionInterface $promotion): void {
    $this->assertEntity($entity);
    $orderItem = $entity;
    $quantity = (int) $orderItem->getQuantity();

    $buyQty = (int) $this->configuration['buy_quantity'];
    $getQty = (int) $this->configuration['get_quantity'];
    $maxSets = (int) $this->configuration['max_sets'];
    $setSize = $buyQty + $getQty;

    if ($quantity < $setSize) {
      return;
    }

    // คำนวณจำนวนชุด
    $sets = floor($quantity / $setSize);
    if ($maxSets > 0) {
      $sets = min($sets, $maxSets);
    }

    // คำนวณส่วนลด
    $freeQuantity = $sets * $getQty;
    $unitPrice = $orderItem->getUnitPrice();
    $discountAmount = $unitPrice->multiply((string) $freeQuantity);

    $adjustment = new Adjustment([
      'type' => 'promotion',
      'label' => $promotion->label() . " (ฟรี {$freeQuantity} ชิ้น)",
      'amount' => $discountAmount->multiply('-1'),
      'source_id' => $promotion->id(),
      'included' => FALSE,
    ]);

    $orderItem->addAdjustment($adjustment);
  }

}
```

---

## ส่วนที่ 7: Custom Commerce Module

### 7.1 โครงสร้าง Module

```
web/modules/custom/my_commerce/
├── my_commerce.info.yml
├── my_commerce.module
├── my_commerce.services.yml
├── my_commerce.routing.yml
├── my_commerce.permissions.yml
├── src/
│   ├── Controller/
│   │   ├── StripeWebhookController.php
│   │   └── OrderDashboardController.php
│   ├── EventSubscriber/
│   │   └── OrderEventSubscriber.php
│   ├── Form/
│   │   ├── CheckoutCustomPane.php
│   │   └── PromotionApplyForm.php
│   ├── OrderProcessor/
│   │   └── CustomOrderProcessor.php
│   ├── Plugin/
│   │   └── Commerce/
│   │       ├── Condition/
│   │       │   └── CustomerBirthday.php
│   │       ├── PaymentGateway/
│   │       │   └── StripeCustomGateway.php
│   │       ├── PromotionOffer/
│   │       │   └── BuyXGetYFree.php
│   │       └── ShippingMethod/
│   │           └── ThaiPostShipping.php
│   └── Service/
│       ├── InventoryManager.php
│       └── OrderReportService.php
└── templates/
    ├── commerce-order-confirmation.html.twig
    └── commerce-order-receipt.html.twig
```

### 7.2 Module Info และ Services

```yaml
# web/modules/custom/my_commerce/my_commerce.info.yml
name: 'My Commerce'
type: module
description: 'Custom Drupal Commerce module สำหรับร้านค้าออนไลน์'
core_version_requirement: ^10
package: 'Custom'
dependencies:
  - commerce:commerce
  - commerce:commerce_order
  - commerce:commerce_product
  - commerce:commerce_payment
  - commerce:commerce_shipping
  - commerce:commerce_promotion
  - commerce:commerce_checkout
```

```yaml
# web/modules/custom/my_commerce/my_commerce.services.yml
services:
  my_commerce.order_processor:
    class: Drupal\my_commerce\OrderProcessor\CustomOrderProcessor
    tags:
      - { name: commerce_order.order_processor, priority: 300 }

  my_commerce.order_event_subscriber:
    class: Drupal\my_commerce\EventSubscriber\OrderEventSubscriber
    arguments:
      - '@plugin.manager.mail'
      - '@language_manager'
      - '@logger.channel.my_commerce'
    tags:
      - { name: event_subscriber }

  my_commerce.inventory_manager:
    class: Drupal\my_commerce\Service\InventoryManager
    arguments:
      - '@entity_type.manager'
      - '@logger.channel.my_commerce'

  my_commerce.order_report_service:
    class: Drupal\my_commerce\Service\OrderReportService
    arguments:
      - '@database'
      - '@entity_type.manager'

  logger.channel.my_commerce:
    parent: logger.channel_base
    arguments: ['my_commerce']
```

### 7.3 Checkout Pane

```php
<?php
// web/modules/custom/my_commerce/src/Plugin/Commerce/CheckoutPane/GiftMessagePane.php

namespace Drupal\my_commerce\Plugin\Commerce\CheckoutPane;

use Drupal\commerce_checkout\Plugin\Commerce\CheckoutPane\CheckoutPaneBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Gift Message Checkout Pane
 *
 * @CommerceCheckoutPane(
 *   id = "gift_message_pane",
 *   label = @Translation("ข้อความของขวัญ"),
 *   default_step = "order_information",
 *   wrapper_element = "fieldset",
 * )
 */
class GiftMessagePane extends CheckoutPaneBase {

  /**
   * {@inheritdoc}
   */
  public function isVisible(): bool {
    return TRUE;
  }

  /**
   * {@inheritdoc}
   */
  public function buildPaneSummary(): array {
    $summary = [];
    $giftMessage = $this->order->getData('gift_message');

    if ($giftMessage) {
      $isGift = $this->order->getData('is_gift');
      $summary[] = [
        '#markup' => $isGift
          ? $this->t('เป็นของขวัญ: @message', ['@message' => $giftMessage])
          : $this->t('หมายเหตุ: @message', ['@message' => $giftMessage]),
      ];
    }

    return $summary;
  }

  /**
   * {@inheritdoc}
   */
  public function buildPaneForm(array $pane_form, FormStateInterface $form_state, array &$complete_form): array {
    $isGift = $this->order->getData('is_gift', FALSE);
    $giftMessage = $this->order->getData('gift_message', '');

    $pane_form['is_gift'] = [
      '#type' => 'checkbox',
      '#title' => $this->t('ต้องการให้เป็นของขวัญ'),
      '#default_value' => $isGift,
    ];

    $pane_form['gift_message'] = [
      '#type' => 'textarea',
      '#title' => $this->t('ข้อความ/หมายเหตุ'),
      '#description' => $this->t('ข้อความสำหรับผู้รับหรือหมายเหตุเพิ่มเติม (ไม่เกิน 200 ตัวอักษร)'),
      '#default_value' => $giftMessage,
      '#maxlength' => 200,
      '#rows' => 3,
      '#states' => [
        'visible' => [
          ':input[name="gift_message_pane[is_gift]"]' => ['checked' => TRUE],
        ],
      ],
    ];

    $pane_form['gift_wrapping'] = [
      '#type' => 'select',
      '#title' => $this->t('บรรจุภัณฑ์'),
      '#options' => [
        '' => $this->t('-- เลือกบรรจุภัณฑ์ --'),
        'standard' => $this->t('กล่องมาตรฐาน (ฟรี)'),
        'premium' => $this->t('กล่องพรีเมียม (+50 บาท)'),
        'luxury' => $this->t('กล่องลักซ์ชวรี่ (+150 บาท)'),
      ],
      '#default_value' => $this->order->getData('gift_wrapping', ''),
      '#states' => [
        'visible' => [
          ':input[name="gift_message_pane[is_gift]"]' => ['checked' => TRUE],
        ],
      ],
    ];

    return $pane_form;
  }

  /**
   * {@inheritdoc}
   */
  public function validatePaneForm(array &$pane_form, FormStateInterface $form_state, array &$complete_form): void {
    $values = $form_state->getValue($pane_form['#parents']);
    $isGift = !empty($values['is_gift']);

    if ($isGift && empty($values['gift_message'])) {
      $form_state->setError($pane_form['gift_message'], $this->t('กรุณาระบุข้อความสำหรับผู้รับ'));
    }
  }

  /**
   * {@inheritdoc}
   */
  public function submitPaneForm(array &$pane_form, FormStateInterface $form_state, array &$complete_form): void {
    $values = $form_state->getValue($pane_form['#parents']);

    $this->order->setData('is_gift', !empty($values['is_gift']));
    $this->order->setData('gift_message', $values['gift_message'] ?? '');
    $this->order->setData('gift_wrapping', $values['gift_wrapping'] ?? '');

    // เพิ่มค่าบรรจุภัณฑ์
    if (!empty($values['is_gift']) && !empty($values['gift_wrapping'])) {
      $this->applyWrappingCharge($values['gift_wrapping']);
    }
  }

  /**
   * เพิ่มค่าบรรจุภัณฑ์
   */
  private function applyWrappingCharge(string $wrappingType): void {
    $charges = [
      'premium' => '50',
      'luxury' => '150',
    ];

    if (!isset($charges[$wrappingType])) {
      return;
    }

    $amount = new \Drupal\commerce_price\Price($charges[$wrappingType], 'THB');
    $adjustment = new \Drupal\commerce_order\Adjustment([
      'type' => 'custom',
      'label' => 'ค่าบรรจุภัณฑ์ของขวัญ',
      'amount' => $amount,
      'source_id' => 'gift_wrapping',
    ]);

    $this->order->addAdjustment($adjustment);
  }

}
```

### 7.4 Order Report Service

```php
<?php
// web/modules/custom/my_commerce/src/Service/OrderReportService.php

namespace Drupal\my_commerce\Service;

use Drupal\Core\Database\Connection;
use Drupal\Core\Entity\EntityTypeManagerInterface;

/**
 * Service สำหรับรายงาน Order
 */
class OrderReportService {

  public function __construct(
    private readonly Connection $database,
    private readonly EntityTypeManagerInterface $entityTypeManager
  ) {}

  /**
   * ดึงยอดขายรายวัน
   */
  public function getDailySales(\DateTime $date): array {
    $startOfDay = clone $date;
    $startOfDay->setTime(0, 0, 0);

    $endOfDay = clone $date;
    $endOfDay->setTime(23, 59, 59);

    $query = $this->database->select('commerce_order', 'o')
      ->condition('o.state', 'completed')
      ->condition('o.placed', $startOfDay->getTimestamp(), '>=')
      ->condition('o.placed', $endOfDay->getTimestamp(), '<=');

    $query->addExpression('COUNT(o.order_id)', 'order_count');
    $query->addExpression('SUM(ot.amount__number)', 'total_amount');
    $query->join('commerce_order__total_price', 'ot', 'ot.entity_id = o.order_id');

    $result = $query->execute()->fetchAssoc();

    return [
      'date' => $date->format('Y-m-d'),
      'order_count' => (int) ($result['order_count'] ?? 0),
      'total_amount' => (float) ($result['total_amount'] ?? 0),
      'currency' => 'THB',
    ];
  }

  /**
   * ดึงสินค้าขายดี
   */
  public function getTopSellingProducts(int $limit = 10, int $days = 30): array {
    $since = new \DateTime("-{$days} days");

    $query = $this->database->select('commerce_order_item', 'oi')
      ->condition('o.state', 'completed')
      ->condition('o.placed', $since->getTimestamp(), '>=')
      ->groupBy('oi.purchased_entity__target_id')
      ->orderBy('total_quantity', 'DESC')
      ->range(0, $limit);

    $query->join('commerce_order', 'o', 'o.order_id = oi.order_id');
    $query->addField('oi', 'purchased_entity__target_id', 'variation_id');
    $query->addExpression('SUM(oi.quantity)', 'total_quantity');
    $query->addExpression('SUM(oi.total_price__number)', 'total_revenue');

    $results = $query->execute()->fetchAll();
    $products = [];

    foreach ($results as $row) {
      $variation = $this->entityTypeManager
        ->getStorage('commerce_product_variation')
        ->load($row->variation_id);

      if ($variation) {
        $products[] = [
          'variation_id' => $row->variation_id,
          'title' => $variation->getTitle(),
          'sku' => $variation->getSku(),
          'total_quantity' => (int) $row->total_quantity,
          'total_revenue' => (float) $row->total_revenue,
        ];
      }
    }

    return $products;
  }

  /**
   * ดึงสถิติรายเดือน
   */
  public function getMonthlyStats(int $year, int $month): array {
    $startDate = new \DateTime("$year-$month-01");
    $endDate = new \DateTime($startDate->format('Y-m-t'));

    $query = $this->database->select('commerce_order', 'o')
      ->condition('o.state', ['completed', 'refunded'], 'IN')
      ->condition('o.placed', $startDate->getTimestamp(), '>=')
      ->condition('o.placed', $endDate->getTimestamp(), '<=');

    $query->addExpression('COUNT(CASE WHEN o.state = :completed THEN 1 END)', 'completed_orders', [':completed' => 'completed']);
    $query->addExpression('COUNT(CASE WHEN o.state = :refunded THEN 1 END)', 'refunded_orders', [':refunded' => 'refunded']);
    $query->addExpression('SUM(CASE WHEN o.state = :completed2 THEN ot.amount__number ELSE 0 END)', 'gross_revenue', [':completed2' => 'completed']);

    $query->join('commerce_order__total_price', 'ot', 'ot.entity_id = o.order_id');

    $result = $query->execute()->fetchAssoc();

    return [
      'year' => $year,
      'month' => $month,
      'completed_orders' => (int) ($result['completed_orders'] ?? 0),
      'refunded_orders' => (int) ($result['refunded_orders'] ?? 0),
      'gross_revenue' => (float) ($result['gross_revenue'] ?? 0),
      'currency' => 'THB',
    ];
  }

}
```

---

## ส่วนที่ 8: Workshop - สร้างร้านค้าออนไลน์สมบูรณ์แบบ

### Workshop: ร้านขายเสื้อผ้าออนไลน์

ในส่วนนี้เราจะสร้างร้านขายเสื้อผ้าออนไลน์แบบสมบูรณ์ตั้งแต่ต้นจนจบ

#### ขั้นตอนที่ 1: เตรียมสภาพแวดล้อม

```bash
# 1. สร้าง Drupal project
composer create-project drupal/recommended-project thai-fashion-store
cd thai-fashion-store

# 2. ติดตั้ง Commerce และ dependencies
composer require \
  drupal/commerce \
  drupal/commerce_stripe \
  drupal/commerce_shipping \
  drupal/commerce_promotion \
  drupal/commerce_tax \
  drupal/address \
  drupal/inline_entity_form \
  drupal/views_infinite_scroll

# 3. ติดตั้ง Stripe PHP SDK
composer require stripe/stripe-php

# 4. Setup database (แก้ไข settings.php ก่อน)
drush site:install --account-name=admin --account-pass=admin123

# 5. Enable modules
drush en commerce commerce_product commerce_order commerce_cart \
  commerce_checkout commerce_payment commerce_stripe commerce_shipping \
  commerce_promotion commerce_tax
```

#### ขั้นตอนที่ 2: สร้าง Custom Module

```bash
# สร้างโครงสร้าง module
mkdir -p web/modules/custom/thai_fashion
cd web/modules/custom/thai_fashion

# สร้าง info file
cat > thai_fashion.info.yml << 'EOF'
name: 'Thai Fashion Store'
type: module
description: 'ร้านขายเสื้อผ้าออนไลน์ไทย'
core_version_requirement: ^10
package: 'Custom'
dependencies:
  - commerce:commerce
  - commerce:commerce_product
  - commerce:commerce_order
  - commerce:commerce_payment
  - commerce:commerce_shipping
  - commerce:commerce_promotion
EOF
```

#### ขั้นตอนที่ 3: Product Configuration

```php
<?php
// web/modules/custom/thai_fashion/thai_fashion.install

use Drupal\field\Entity\FieldStorageConfig;
use Drupal\field\Entity\FieldConfig;
use Drupal\commerce_product\Entity\ProductType;
use Drupal\commerce_product\Entity\ProductVariationType;

/**
 * Implements hook_install().
 */
function thai_fashion_install(): void {
  // สร้าง Product Types
  _thai_fashion_create_product_types();

  // สร้าง Fields
  _thai_fashion_create_fields();

  // สร้าง Store
  _thai_fashion_create_store();
}

function _thai_fashion_create_product_types(): void {
  $types = [
    'shirt' => 'เสื้อ',
    'pants' => 'กางเกง',
    'dress' => 'ชุดเดรส',
    'accessories' => 'เครื่องประดับ',
  ];

  foreach ($types as $id => $label) {
    // Variation Type
    $variationType = ProductVariationType::create([
      'id' => $id . '_variation',
      'label' => $label . ' Variation',
      'orderItemType' => 'default',
      'generateTitle' => TRUE,
    ]);
    $variationType->save();

    // Product Type
    $productType = ProductType::create([
      'id' => $id,
      'label' => $label,
      'variationType' => $id . '_variation',
      'injectVariationFields' => TRUE,
    ]);
    $productType->save();
  }
}

function _thai_fashion_create_fields(): void {
  $clothingTypes = ['shirt_variation', 'pants_variation', 'dress_variation'];

  // Color field
  FieldStorageConfig::create([
    'field_name' => 'field_color',
    'entity_type' => 'commerce_product_variation',
    'type' => 'list_string',
    'settings' => [
      'allowed_values' => [
        'black' => 'ดำ', 'white' => 'ขาว', 'red' => 'แดง',
        'blue' => 'น้ำเงิน', 'green' => 'เขียว', 'yellow' => 'เหลือง',
        'pink' => 'ชมพู', 'purple' => 'ม่วง', 'orange' => 'ส้ม',
        'gray' => 'เทา', 'brown' => 'น้ำตาล', 'navy' => 'กรมท่า',
      ],
    ],
  ])->save();

  // Size field
  FieldStorageConfig::create([
    'field_name' => 'field_clothing_size',
    'entity_type' => 'commerce_product_variation',
    'type' => 'list_string',
    'settings' => [
      'allowed_values' => [
        'XS' => 'XS', 'S' => 'S', 'M' => 'M',
        'L' => 'L', 'XL' => 'XL', 'XXL' => 'XXL',
        'XXXL' => 'XXXL', 'F' => 'Free Size',
      ],
    ],
  ])->save();

  // Stock field
  FieldStorageConfig::create([
    'field_name' => 'field_stock',
    'entity_type' => 'commerce_product_variation',
    'type' => 'integer',
  ])->save();

  // เพิ่ม fields ให้แต่ละ variation type
  foreach ($clothingTypes as $bundle) {
    foreach (['field_color', 'field_clothing_size', 'field_stock'] as $fieldName) {
      $labels = [
        'field_color' => 'สี',
        'field_clothing_size' => 'ขนาด',
        'field_stock' => 'จำนวนในคลัง',
      ];

      FieldConfig::create([
        'field_name' => $fieldName,
        'entity_type' => 'commerce_product_variation',
        'bundle' => $bundle,
        'label' => $labels[$fieldName],
        'required' => in_array($fieldName, ['field_color', 'field_clothing_size']),
      ])->save();
    }
  }
}

function _thai_fashion_create_store(): void {
  $store = \Drupal\commerce_store\Entity\Store::create([
    'type' => 'online',
    'uid' => 1,
    'name' => 'Thai Fashion Store',
    'mail' => 'info@thai-fashion.com',
    'address' => [
      'country_code' => 'TH',
      'address_line1' => '456 ถนนพหลโยธิน',
      'locality' => 'กรุงเทพมหานคร',
      'postal_code' => '10900',
    ],
    'default_currency' => 'THB',
    'timezone' => 'Asia/Bangkok',
    'billing_countries' => ['TH'],
  ]);
  $store->save();
  $store->setDefault(TRUE);
  $store->save();
}
```

#### ขั้นตอนที่ 4: Tax Configuration

```php
<?php
// web/modules/custom/thai_fashion/src/TaxSetup.php

namespace Drupal\thai_fashion;

/**
 * ตั้งค่าภาษี VAT 7% สำหรับไทย
 */
class TaxSetup {

  public function configureTax(): void {
    // สร้าง Tax Type
    $taxType = \Drupal\commerce_tax\Entity\TaxType::create([
      'id' => 'thai_vat',
      'label' => 'VAT (ภาษีมูลค่าเพิ่ม)',
      'plugin' => 'thai_vat',
      'configuration' => [
        'display_inclusive' => TRUE,
        'rates' => [
          [
            'id' => 'standard',
            'label' => 'VAT 7%',
            'percentage' => '0.07',
          ],
        ],
        'territories' => [
          ['country_code' => 'TH'],
        ],
      ],
      'status' => TRUE,
    ]);
    $taxType->save();
  }

}
```

#### ขั้นตอนที่ 5: Payment Gateway Setup

```php
<?php
// ตั้งค่า Stripe Payment Gateway ผ่าน code

use Drupal\commerce_payment\Entity\PaymentGateway;

$gateway = PaymentGateway::create([
  'id' => 'stripe_checkout',
  'label' => 'Stripe - บัตรเครดิต/เดบิต',
  'plugin' => 'stripe_checkout',
  'configuration' => [
    'display_label' => 'บัตรเครดิต/เดบิต',
    'payment_method_types' => ['credit_card'],
    'publishable_key' => 'pk_test_YOUR_KEY',
    'secret_key' => 'sk_test_YOUR_SECRET',
    'webhook_secret' => 'whsec_YOUR_WEBHOOK_SECRET',
    'mode' => 'test',
  ],
  'status' => TRUE,
  'weight' => 0,
]);
$gateway->save();
```

#### ขั้นตอนที่ 6: Promotion Setup

```php
<?php
// สร้าง Promotions ต่างๆ

use Drupal\commerce_promotion\Entity\Coupon;
use Drupal\commerce_promotion\Entity\Promotion;

// โปรโมชั่นสำหรับสมาชิกใหม่
$newMemberPromotion = Promotion::create([
  'name' => 'ส่วนลดสมาชิกใหม่',
  'offer' => [
    'target_plugin_id' => 'order_percentage_off',
    'target_plugin_configuration' => [
      'percentage' => '0.15',
    ],
  ],
  'conditions' => [
    [
      'target_plugin_id' => 'order_total_price',
      'target_plugin_configuration' => [
        'operator' => '>=',
        'amount' => ['number' => '500', 'currency_code' => 'THB'],
      ],
    ],
  ],
  'start_date' => '2024-01-01T00:00:00',
  'status' => TRUE,
  'display_name' => 'ส่วนลด 15% สำหรับสมาชิกใหม่',
  'usage_limit' => 1,
  'usage_limit_customer' => 1,
]);
$newMemberPromotion->save();

// Coupon Code
$coupon = Coupon::create([
  'code' => 'WELCOME15',
  'promotion_id' => $newMemberPromotion->id(),
  'usage_limit' => 100,
  'status' => TRUE,
]);
$coupon->save();
```

#### ขั้นตอนที่ 7: Shipping Setup

```php
<?php
// ตั้งค่าการจัดส่ง

use Drupal\commerce_shipping\Entity\ShippingMethod;

$shippingMethod = ShippingMethod::create([
  'stores' => [1],
  'name' => 'ไปรษณีย์ไทย',
  'plugin' => [
    'target_plugin_id' => 'thai_post',
    'target_plugin_configuration' => [
      'ems_base_rate' => '80',
      'registered_base_rate' => '40',
      'parcel_base_rate' => '30',
      'weight_rate' => '5',
      'free_shipping_threshold' => '1000',
    ],
  ],
  'status' => TRUE,
  'weight' => 0,
]);
$shippingMethod->save();
```

#### ขั้นตอนที่ 8: Views Configuration

```yaml
# config/sync/views.view.product_catalog.yml
# (สร้าง Views สำหรับแสดงสินค้า)
id: product_catalog
label: 'แคตตาล็อกสินค้า'
module: views
description: 'แสดงสินค้าทั้งหมดในร้าน'
tag: commerce
base_table: commerce_product_field_data
base_field: product_id
display:
  default:
    display_options:
      access:
        type: none
      cache:
        type: tag
      pager:
        type: mini
        options:
          items_per_page: 12
      style:
        type: grid
      row:
        type: fields
      fields:
        title:
          table: commerce_product_field_data
          field: title
          label: ''
          type: string_link
        commerce_price:
          table: commerce_product_variation_field_data
          field: price
          label: ราคา
      filters:
        status:
          value: '1'
          table: commerce_product_field_data
          field: status
      sorts:
        created:
          order: DESC
```

#### ขั้นตอนที่ 9: Template Customization

```twig
{# web/themes/custom/thai_fashion_theme/templates/commerce/commerce-cart-block.html.twig #}

<div class="cart-block {{ attributes.class }}" {{ attributes }}>
  {% if count %}
    <div class="cart-block__icon">
      <a href="{{ url('<front>') }}/cart" class="cart-icon-link">
        <span class="cart-icon">🛒</span>
        <span class="cart-count badge">{{ count }}</span>
      </a>
    </div>
    <div class="cart-block__contents">
      <div class="cart-block__header">
        <h3>{{ 'ตะกร้าสินค้า'|t }}</h3>
        <span class="cart-total">{{ 'รวม'|t }}: {{ total_price }}</span>
      </div>
      <div class="cart-block__links">
        <a href="{{ url('<front>') }}/cart" class="btn btn-outline-primary btn-sm">
          {{ 'ดูตะกร้า'|t }}
        </a>
        <a href="{{ url('<front>') }}/checkout" class="btn btn-primary btn-sm">
          {{ 'ชำระเงิน'|t }}
        </a>
      </div>
    </div>
  {% else %}
    <div class="cart-block__empty">
      <span class="cart-icon">🛒</span>
      <span>{{ 'ตะกร้าสินค้าว่าง'|t }}</span>
    </div>
  {% endif %}
</div>
```

#### ขั้นตอนที่ 10: Testing

```php
<?php
// web/modules/custom/thai_fashion/tests/src/Functional/OrderFlowTest.php

namespace Drupal\Tests\thai_fashion\Functional;

use Drupal\Tests\commerce\Functional\CommerceBrowserTestBase;
use Drupal\commerce_price\Price;
use Drupal\commerce_product\Entity\Product;
use Drupal\commerce_product\Entity\ProductVariation;

/**
 * Test Order Flow
 *
 * @group thai_fashion
 */
class OrderFlowTest extends CommerceBrowserTestBase {

  protected static $modules = [
    'thai_fashion',
    'commerce_cart',
    'commerce_checkout',
    'commerce_payment',
    'commerce_payment_example',
  ];

  protected $defaultTheme = 'stark';

  /**
   * ทดสอบการเพิ่มสินค้าเข้าตะกร้า
   */
  public function testAddToCart(): void {
    // สร้างสินค้าทดสอบ
    $variation = ProductVariation::create([
      'type' => 'shirt_variation',
      'sku' => 'SHIRT-001-M-BLACK',
      'price' => new Price('599', 'THB'),
      'field_color' => 'black',
      'field_clothing_size' => 'M',
      'status' => TRUE,
    ]);
    $variation->save();

    $product = Product::create([
      'type' => 'shirt',
      'title' => 'เสื้อ Basic ทดสอบ',
      'stores' => [$this->store],
      'variations' => [$variation],
      'status' => TRUE,
    ]);
    $product->save();

    // เข้าหน้าสินค้า
    $this->drupalGet('/product/' . $product->id());
    $this->assertSession()->statusCodeEquals(200);
    $this->assertSession()->pageTextContains('เสื้อ Basic ทดสอบ');
    $this->assertSession()->pageTextContains('599');

    // เพิ่มเข้าตะกร้า
    $this->submitForm([
      'quantity[0][value]' => '2',
    ], 'เพิ่มลงในตะกร้า');

    // ตรวจสอบตะกร้า
    $this->drupalGet('/cart');
    $this->assertSession()->pageTextContains('เสื้อ Basic ทดสอบ');
    $this->assertSession()->pageTextContains('1,198'); // 599 * 2
  }

  /**
   * ทดสอบ checkout flow
   */
  public function testCheckoutFlow(): void {
    // ... ทดสอบกระบวนการ checkout
    $this->drupalGet('/checkout/1');
    $this->assertSession()->statusCodeEquals(200);

    // กรอกข้อมูลลูกค้า
    $this->submitForm([
      'contact_information[email]' => 'test@example.com',
      'shipping_information[shipping_profile][address][0][address][given_name]' => 'สมชาย',
      'shipping_information[shipping_profile][address][0][address][family_name]' => 'ใจดี',
      'shipping_information[shipping_profile][address][0][address][address_line1]' => '123 ถนนทดสอบ',
      'shipping_information[shipping_profile][address][0][address][locality]' => 'กรุงเทพ',
      'shipping_information[shipping_profile][address][0][address][postal_code]' => '10110',
      'shipping_information[shipping_profile][address][0][address][country_code]' => 'TH',
    ], 'ถัดไป');
  }

}
```

---

## สรุปการใช้งาน Drupal Commerce

### สถาปัตยกรรมของ Drupal Commerce

```
┌─────────────────────────────────────────────────┐
│                   Drupal Core                   │
├─────────────────────────────────────────────────┤
│              Commerce Framework                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │ Products │ │  Orders  │ │    Payments      │ │
│  └──────────┘ └──────────┘ └──────────────────┘ │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │Promotions│ │ Shipping │ │      Tax         │ │
│  └──────────┘ └──────────┘ └──────────────────┘ │
├─────────────────────────────────────────────────┤
│              Custom Modules                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │  Custom  │ │  Custom  │ │     Custom       │ │
│  │ Products │ │  Orders  │ │    Gateways      │ │
│  └──────────┘ └──────────┘ └──────────────────┘ │
└─────────────────────────────────────────────────┘
```

### Plugin System Overview

Drupal Commerce ใช้ Plugin system อย่างมาก ดังนี้:

| Plugin Type | ใช้สำหรับ | Base Class |
|------------|---------|-----------|
| PaymentGateway | Payment processing | OnsitePaymentGatewayBase |
| ShippingMethod | Shipping calculations | ShippingMethodBase |
| PromotionOffer | Discount calculations | OrderPromotionOfferBase |
| Condition | Order/customer conditions | ConditionBase |
| CheckoutPane | Checkout steps | CheckoutPaneBase |
| OrderProcessor | Order adjustments | OrderProcessorInterface |

### Best Practices

1. **ใช้ Dependency Injection**: อย่า hardcode dependencies ใช้ DI เสมอ
2. **Event-Driven Architecture**: ใช้ Event Subscribers แทนการ override methods
3. **Plugin System**: สร้าง custom plugins เพื่อเพิ่ม functionality
4. **Configuration Management**: เก็บ config ใน YAML files
5. **Testing**: เขียน unit และ functional tests เสมอ
6. **Performance**: ใช้ caching และ lazy loading
7. **Security**: validate input, sanitize output ทุกครั้ง

---

## แบบทดสอบ (Quiz)

### คำถามที่ 1

**คำถาม**: ใน Drupal Commerce คลาสใดที่ใช้เป็น Base Class สำหรับสร้าง Payment Gateway แบบ Onsite (ชำระเงินในเว็บไซต์)?

**ก.** `PaymentGatewayBase`
**ข.** `OnsitePaymentGatewayBase`
**ค.** `OffsitePaymentGatewayBase`
**ง.** `ManualPaymentGatewayBase`

**คำตอบ**: **ข. OnsitePaymentGatewayBase**

*อธิบาย*: `OnsitePaymentGatewayBase` ใช้สำหรับ Payment Gateways ที่ดำเนินการชำระเงินภายในเว็บไซต์โดยตรง (เช่น Stripe ที่ใช้ JavaScript ในการเก็บข้อมูลบัตร) ในขณะที่ `OffsitePaymentGatewayBase` ใช้สำหรับ Gateways ที่ redirect ผู้ใช้ไปยังหน้าชำระเงินของ provider (เช่น PayPal)

---

### คำถามที่ 2

**คำถาม**: Order Processor Interface ใน Drupal Commerce ต้องการ implement method ใด?

**ก.** `processOrder(OrderInterface $order)`
**ข.** `process(OrderInterface $order)`
**ค.** `handleOrder(OrderInterface $order)`
**ง.** `executeOrder(OrderInterface $order)`

**คำตอบ**: **ข. process(OrderInterface $order)**

*อธิบาย*: `OrderProcessorInterface` กำหนดให้ implement method `process(OrderInterface $order)` ซึ่งจะถูกเรียกใช้เมื่อ order ถูก recalculate เพื่อให้ processor สามารถเพิ่ม adjustments (ส่วนลด, ค่าเพิ่มเติม) ได้

---

### คำถามที่ 3

**คำถาม**: เมื่อต้องการส่ง notification email เมื่อ order ถูก place ใน Drupal Commerce ควรใช้วิธีใด?

**ก.** Override `OrderType::place()` method
**ข.** Implement `hook_commerce_order_place()`
**ค.** สร้าง Event Subscriber สำหรับ `OrderEvents::ORDER_PLACE`
**ง.** ใช้ `hook_entity_insert()` กับ `commerce_order` entity

**คำตอบ**: **ค. สร้าง Event Subscriber สำหรับ OrderEvents::ORDER_PLACE**

*อธิบาย*: Drupal Commerce ใช้ Symfony Event system ดังนั้นวิธีที่ถูกต้องและ maintainable ที่สุดคือการสร้าง Event Subscriber ที่ subscribe กับ `OrderEvents::ORDER_PLACE` event ซึ่งจะถูก dispatch เมื่อ order เข้าสู่ state "placed"

---

### คำถามที่ 4

**คำถาม**: ใน Drupal Commerce Promotion Offer Plugin annotation ใดที่ถูกต้องสำหรับการสร้าง offer ที่ทำงานกับ Order Items?

**ก.** `@CommercePromotion(entity_type = "commerce_order_item")`
**ข.** `@CommercePromotionOffer(entity_type = "commerce_order_item")`
**ค.** `@CommerceDiscount(entity_type = "commerce_order_item")`
**ง.** `@CommerceOffer(entity_type = "commerce_order_item")`

**คำตอบ**: **ข. @CommercePromotionOffer(entity_type = "commerce_order_item")**

*อธิบาย*: Plugin annotation ที่ถูกต้องสำหรับ Promotion Offer ที่ทำงานกับ Order Items คือ `@CommercePromotionOffer` พร้อมกับ `entity_type = "commerce_order_item"` หาก offer ทำงานกับทั้ง order ให้ใช้ `entity_type = "commerce_order"` แทน

---

### คำถามที่ 5

**คำถาม**: ใน Drupal Commerce CheckoutPane เมื่อต้องการแสดง Pane เฉพาะในเงื่อนไขบางอย่าง ควร override method ใด?

**ก.** `getVisible()`
**ข.** `isApplicable()`
**ค.** `isVisible()`
**ง.** `shouldDisplay()`

**คำตอบ**: **ค. isVisible()**

*อธิบาย*: `CheckoutPaneBase` มี method `isVisible(): bool` ที่สามารถ override ได้เพื่อกำหนดเงื่อนไขการแสดง Pane โดย default จะ return `TRUE` เสมอ แต่เราสามารถ override เพื่อตรวจสอบเงื่อนไข เช่น ตรวจสอบว่า order มีสินค้าประเภทที่ต้องการ gift wrapping หรือไม่

---

## แหล่งเรียนรู้เพิ่มเติม

### Documentation

- **Drupal Commerce Docs**: https://docs.drupalcommerce.org/
- **Commerce API**: https://api.drupalcommerce.org/
- **Commerce GitHub**: https://github.com/drupalcommerce/commerce

### Drupal.org Resources

- **Commerce Forum**: https://www.drupal.org/forum/contributed-modules/commerce
- **Issue Queue**: https://www.drupal.org/project/issues/commerce
- **Commerce Contributed Modules**: https://www.drupal.org/project/project_module?f[2]=im_vid_3%3A56

### Thai Community

- **Drupal Thailand**: https://www.drupal.org/thai
- **Facebook Group**: Drupal Thailand Community

### ตัวอย่างโปรเจ็กต์ที่แนะนำ

1. **Commerce Demo**: `composer require drupal/commerce_demo`
2. **Commerce Kickstart**: ตัวอย่าง configuration ที่สมบูรณ์
3. **Commerce Examples**: Code examples จาก Drupal Commerce team

---

*Part 081 | ระดับสูง | Drupal Commerce*
