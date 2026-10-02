# Part 082: Drupal Entity API

## ระดับ: สูง | ขั้นตอนที่ 761-800

---

## วัตถุประสงค์การเรียนรู้

หลังจากศึกษาบทนี้แล้ว ผู้เรียนจะสามารถ:

1. **เข้าใจ Entity System** ของ Drupal 10 และสถาปัตยกรรมพื้นฐาน
2. **สร้าง Custom Content Entity** พร้อม annotations และ field definitions
3. **สร้าง Custom Config Entity** สำหรับจัดการการตั้งค่าระบบ
4. **ดำเนินการ CRUD operations** ผ่าน Entity API
5. **ใช้ Entity Hooks และ Events** เพื่อขยายฟังก์ชันการทำงาน
6. **บูรณาการ Field API** กับ Custom Entities
7. **เขียน Entity Query ขั้นสูง** เพื่อดึงข้อมูลอย่างมีประสิทธิภาพ
8. **สร้าง Product Catalog Entity** ในโปรเจกต์ Workshop จริง

---

## บทนำ: ทำไม Drupal Entity API ถึงสำคัญ

Drupal Entity API เป็นหัวใจสำคัญของ Drupal 8, 9 และ 10 ซึ่งแทนที่ระบบ node เพียงอย่างเดียวด้วยระบบ entity ที่ยืดหยุ่นกว่ามาก Entity คือวัตถุข้อมูลหลักใน Drupal ไม่ว่าจะเป็น nodes, users, taxonomy terms, comments, หรือ custom entities ที่คุณสร้างขึ้นเอง

```
Entity Hierarchy ใน Drupal 10:
┌─────────────────────────────────┐
│         EntityInterface          │
├─────────────────────────────────┤
│   ContentEntityInterface         │  ← Node, User, File, Comment
│   ConfigEntityInterface          │  ← Vocabulary, Role, View
└─────────────────────────────────┘
```

### ข้อดีของ Entity System

- **Uniform API**: ใช้วิธีการเดียวกันกับ entity ทุกประเภท
- **Field API Integration**: เพิ่ม fields ได้โดยไม่ต้องแก้ไข schema
- **Access Control**: ระบบ access control ที่สมบูรณ์
- **Revision Support**: รองรับ versioning โดยธรรมชาติ
- **Multilingual**: รองรับหลายภาษาในตัว
- **RESTful**: expose เป็น REST API ได้ทันที

---

## ส่วนที่ 1: ภาพรวม Entity System

### 1.1 Entity Types และ Bundles

ใน Drupal Entity System มีโครงสร้างดังนี้:

```
Entity Type (ชนิด entity เช่น "node")
    └── Bundle (ประเภทย่อย เช่น "article", "page")
            └── Fields (ข้อมูลที่เก็บ เช่น "title", "body")
```

**Entity Type** คือ class PHP ที่ define โครงสร้างหลัก  
**Bundle** คือกลุ่มย่อยที่มี field configuration แตกต่างกัน  
**Fields** คือข้อมูลที่ entity เก็บ

### 1.2 Entity Managers และ Services

```php
<?php
// การเข้าถึง Entity Services ใน Drupal 10

// วิธีที่ 1: ผ่าน Dependency Injection (แนะนำ)
use Drupal\Core\Entity\EntityTypeManagerInterface;

class MyService {
    public function __construct(
        private EntityTypeManagerInterface $entityTypeManager
    ) {}
    
    public function getNode(int $nid): ?NodeInterface {
        return $this->entityTypeManager
            ->getStorage('node')
            ->load($nid);
    }
}

// วิธีที่ 2: ผ่าน \Drupal facade (ใช้ใน .module files)
$entityTypeManager = \Drupal::entityTypeManager();
$nodeStorage = $entityTypeManager->getStorage('node');
$node = $nodeStorage->load(1);

// วิธีที่ 3: ผ่าน service container
$entityTypeManager = \Drupal::service('entity_type.manager');
```

### 1.3 Entity Interfaces ที่สำคัญ

```php
<?php
// EntityInterface - interface พื้นฐานสำหรับ entity ทุกชนิด
use Drupal\Core\Entity\EntityInterface;

// ContentEntityInterface - สำหรับ content entities (มี fields)
use Drupal\Core\Entity\ContentEntityInterface;

// ConfigEntityInterface - สำหรับ config entities
use Drupal\Core\Config\Entity\ConfigEntityInterface;

// RevisionableInterface - สำหรับ entities ที่รองรับ revisions
use Drupal\Core\Entity\RevisionableInterface;

// TranslatableInterface - สำหรับ entities ที่รองรับหลายภาษา
use Drupal\Core\TypedData\TranslatableInterface;
```

### 1.4 Entity Storage Handlers

```php
<?php
// EntityStorageInterface methods ที่ใช้บ่อย
$storage = \Drupal::entityTypeManager()->getStorage('node');

// Load entity เดียว
$node = $storage->load($id);

// Load entities หลายตัว
$nodes = $storage->loadMultiple([1, 2, 3]);

// Load entities ตาม properties
$nodes = $storage->loadByProperties([
    'type' => 'article',
    'status' => 1,
]);

// Create entity ใหม่
$node = $storage->create([
    'type' => 'article',
    'title' => 'My Article',
]);

// Save entity
$node->save();

// Delete entity
$node->delete();

// Delete entities หลายตัว
$storage->delete($nodes);
```

### 1.5 Entity Query พื้นฐาน

```php
<?php
// การใช้ EntityQuery เพื่อค้นหา entities
$query = \Drupal::entityQuery('node')
    ->condition('type', 'article')
    ->condition('status', 1)
    ->sort('created', 'DESC')
    ->range(0, 10)
    ->accessCheck(TRUE);

$nids = $query->execute();
$nodes = \Drupal::entityTypeManager()
    ->getStorage('node')
    ->loadMultiple($nids);
```

---

## ส่วนที่ 2: Custom Content Entities

### 2.1 โครงสร้างไฟล์ของ Custom Module

```
modules/custom/product_catalog/
├── product_catalog.info.yml
├── product_catalog.module
├── product_catalog.routing.yml
├── product_catalog.links.menu.yml
├── product_catalog.links.action.yml
├── product_catalog.permissions.yml
├── config/
│   └── install/
│       └── core.entity_form_display.product.product.default.yml
└── src/
    ├── Entity/
    │   ├── Product.php           ← Entity class หลัก
    │   └── ProductType.php       ← Bundle entity
    ├── Entity/Handler/
    │   ├── ProductStorage.php    ← Storage handler
    │   ├── ProductAccessControlHandler.php
    │   ├── ProductListBuilder.php
    │   └── ProductViewBuilder.php
    ├── Form/
    │   ├── ProductForm.php       ← Add/Edit form
    │   └── ProductDeleteForm.php ← Delete confirmation
    └── ProductInterface.php      ← Entity interface
```

### 2.2 สร้าง Entity Class หลัก

```php
<?php
// src/Entity/Product.php
namespace Drupal\product_catalog\Entity;

use Drupal\Core\Entity\ContentEntityBase;
use Drupal\Core\Entity\EntityChangedTrait;
use Drupal\Core\Entity\EntityPublishedTrait;
use Drupal\Core\Entity\EntityStorageInterface;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Field\BaseFieldDefinition;
use Drupal\product_catalog\ProductInterface;
use Drupal\user\EntityOwnerTrait;

/**
 * Defines the Product entity type.
 *
 * @ContentEntityType(
 *   id = "product",
 *   label = @Translation("Product"),
 *   label_collection = @Translation("Products"),
 *   label_singular = @Translation("product"),
 *   label_plural = @Translation("products"),
 *   label_count = @PluralTranslation(
 *     singular = "@count product",
 *     plural = "@count products",
 *   ),
 *   bundle_label = @Translation("Product type"),
 *   handlers = {
 *     "storage" = "Drupal\product_catalog\Entity\Handler\ProductStorage",
 *     "view_builder" = "Drupal\Core\Entity\EntityViewBuilder",
 *     "list_builder" = "Drupal\product_catalog\Entity\Handler\ProductListBuilder",
 *     "views_data" = "Drupal\views\EntityViewsData",
 *     "access" = "Drupal\product_catalog\Entity\Handler\ProductAccessControlHandler",
 *     "form" = {
 *       "add" = "Drupal\product_catalog\Form\ProductForm",
 *       "edit" = "Drupal\product_catalog\Form\ProductForm",
 *       "delete" = "Drupal\product_catalog\Form\ProductDeleteForm",
 *     },
 *     "route_provider" = {
 *       "html" = "Drupal\Core\Entity\Routing\AdminHtmlRouteProvider",
 *     },
 *   },
 *   base_table = "product",
 *   revision_table = "product_revision",
 *   revision_data_table = "product_field_revision",
 *   show_revision_ui = TRUE,
 *   translatable = TRUE,
 *   admin_permission = "administer product types",
 *   entity_keys = {
 *     "id" = "id",
 *     "revision" = "revision_id",
 *     "langcode" = "langcode",
 *     "bundle" = "type",
 *     "label" = "title",
 *     "uuid" = "uuid",
 *     "owner" = "uid",
 *     "published" = "status",
 *   },
 *   revision_metadata_keys = {
 *     "revision_user" = "revision_uid",
 *     "revision_created" = "revision_timestamp",
 *     "revision_log_message" = "revision_log",
 *     "revision_default" = "revision_default",
 *   },
 *   links = {
 *     "collection" = "/admin/content/products",
 *     "add-form" = "/product/add/{product_type}",
 *     "add-page" = "/product/add",
 *     "canonical" = "/product/{product}",
 *     "edit-form" = "/product/{product}/edit",
 *     "delete-form" = "/product/{product}/delete",
 *     "delete-multiple-form" = "/admin/content/products/delete",
 *     "revision" = "/product/{product}/revisions/{product_revision}/view",
 *     "revision-revert-form" = "/product/{product}/revisions/{product_revision}/revert",
 *     "revision-delete-form" = "/product/{product}/revisions/{product_revision}/delete",
 *     "revisions" = "/product/{product}/revisions",
 *   },
 *   bundle_entity_type = "product_type",
 *   field_ui_base_route = "entity.product_type.edit_form",
 * )
 */
class Product extends ContentEntityBase implements ProductInterface {

    use EntityChangedTrait;
    use EntityPublishedTrait;
    use EntityOwnerTrait;

    /**
     * {@inheritdoc}
     */
    public function preSave(EntityStorageInterface $storage): void {
        parent::preSave($storage);
        
        if (!$this->getOwnerId()) {
            // Default owner เป็น current user
            $this->setOwnerId(\Drupal::currentUser()->id());
        }
    }

    /**
     * {@inheritdoc}
     */
    public function getTitle(): string {
        return $this->get('title')->value;
    }

    /**
     * {@inheritdoc}
     */
    public function setTitle(string $title): static {
        $this->set('title', $title);
        return $this;
    }

    /**
     * {@inheritdoc}
     */
    public function getPrice(): float {
        return (float) $this->get('price')->value;
    }

    /**
     * {@inheritdoc}
     */
    public function setPrice(float $price): static {
        $this->set('price', $price);
        return $this;
    }

    /**
     * {@inheritdoc}
     */
    public function getSku(): string {
        return $this->get('sku')->value ?? '';
    }

    /**
     * {@inheritdoc}
     */
    public function setSku(string $sku): static {
        $this->set('sku', $sku);
        return $this;
    }

    /**
     * {@inheritdoc}
     */
    public function getCreatedTime(): int {
        return (int) $this->get('created')->value;
    }

    /**
     * {@inheritdoc}
     */
    public function setCreatedTime(int $timestamp): static {
        $this->set('created', $timestamp);
        return $this;
    }

    /**
     * กำหนด base fields สำหรับ Product entity
     *
     * {@inheritdoc}
     */
    public static function baseFieldDefinitions(EntityTypeInterface $entity_type): array {
        $fields = parent::baseFieldDefinitions($entity_type);

        // เพิ่ม published fields
        $fields += static::publishedBaseFieldDefinitions($entity_type);
        // เพิ่ม owner fields
        $fields += static::ownerBaseFieldDefinitions($entity_type);

        // Title field
        $fields['title'] = BaseFieldDefinition::create('string')
            ->setLabel(t('Title'))
            ->setDescription(t('ชื่อสินค้า'))
            ->setRevisionable(TRUE)
            ->setTranslatable(TRUE)
            ->setRequired(TRUE)
            ->setSetting('max_length', 255)
            ->setDisplayOptions('view', [
                'label' => 'hidden',
                'type' => 'string',
                'weight' => -5,
            ])
            ->setDisplayOptions('form', [
                'type' => 'string_textfield',
                'weight' => -5,
            ])
            ->setDisplayConfigurable('form', TRUE)
            ->setDisplayConfigurable('view', TRUE);

        // SKU field
        $fields['sku'] = BaseFieldDefinition::create('string')
            ->setLabel(t('SKU'))
            ->setDescription(t('รหัสสินค้า (Stock Keeping Unit)'))
            ->setRevisionable(TRUE)
            ->setRequired(TRUE)
            ->setSetting('max_length', 64)
            ->addConstraint('UniqueField')
            ->setDisplayOptions('view', [
                'label' => 'above',
                'type' => 'string',
                'weight' => -4,
            ])
            ->setDisplayOptions('form', [
                'type' => 'string_textfield',
                'weight' => -4,
            ])
            ->setDisplayConfigurable('form', TRUE)
            ->setDisplayConfigurable('view', TRUE);

        // Price field
        $fields['price'] = BaseFieldDefinition::create('decimal')
            ->setLabel(t('Price'))
            ->setDescription(t('ราคาสินค้า'))
            ->setRevisionable(TRUE)
            ->setRequired(TRUE)
            ->setSettings([
                'precision' => 10,
                'scale' => 2,
            ])
            ->setDisplayOptions('view', [
                'label' => 'above',
                'type' => 'number_decimal',
                'weight' => -3,
                'settings' => ['thousand_separator' => ','],
            ])
            ->setDisplayOptions('form', [
                'type' => 'number',
                'weight' => -3,
            ])
            ->setDisplayConfigurable('form', TRUE)
            ->setDisplayConfigurable('view', TRUE);

        // Stock quantity field
        $fields['stock_quantity'] = BaseFieldDefinition::create('integer')
            ->setLabel(t('Stock Quantity'))
            ->setDescription(t('จำนวนสินค้าในคลัง'))
            ->setRevisionable(TRUE)
            ->setDefaultValue(0)
            ->setSetting('min', 0)
            ->setDisplayOptions('view', [
                'label' => 'above',
                'type' => 'number_integer',
                'weight' => -2,
            ])
            ->setDisplayOptions('form', [
                'type' => 'number',
                'weight' => -2,
            ])
            ->setDisplayConfigurable('form', TRUE)
            ->setDisplayConfigurable('view', TRUE);

        // Description field
        $fields['description'] = BaseFieldDefinition::create('text_long')
            ->setLabel(t('Description'))
            ->setDescription(t('รายละเอียดสินค้า'))
            ->setRevisionable(TRUE)
            ->setTranslatable(TRUE)
            ->setDisplayOptions('view', [
                'label' => 'above',
                'type' => 'text_default',
                'weight' => 0,
            ])
            ->setDisplayOptions('form', [
                'type' => 'text_textarea',
                'weight' => 0,
                'rows' => 6,
            ])
            ->setDisplayConfigurable('form', TRUE)
            ->setDisplayConfigurable('view', TRUE);

        // Product image field
        $fields['image'] = BaseFieldDefinition::create('image')
            ->setLabel(t('Image'))
            ->setDescription(t('รูปภาพสินค้า'))
            ->setRevisionable(TRUE)
            ->setTranslatable(TRUE)
            ->setSettings([
                'file_directory' => 'products/[date:custom:Y]-[date:custom:m]',
                'file_extensions' => 'png gif jpg jpeg webp',
                'max_filesize' => '5 MB',
                'alt_field' => TRUE,
                'alt_field_required' => FALSE,
                'title_field' => FALSE,
            ])
            ->setDisplayOptions('view', [
                'label' => 'hidden',
                'type' => 'image',
                'weight' => -1,
                'settings' => ['image_style' => 'medium'],
            ])
            ->setDisplayOptions('form', [
                'type' => 'image_image',
                'weight' => -1,
            ])
            ->setDisplayConfigurable('form', TRUE)
            ->setDisplayConfigurable('view', TRUE);

        // Status field (published/unpublished)
        $fields['status']
            ->setDisplayOptions('form', [
                'type' => 'boolean_checkbox',
                'settings' => ['display_label' => TRUE],
                'weight' => 120,
            ])
            ->setDisplayConfigurable('form', TRUE);

        // Created timestamp
        $fields['created'] = BaseFieldDefinition::create('created')
            ->setLabel(t('Created'))
            ->setDescription(t('เวลาที่สร้างสินค้า'))
            ->setRevisionable(TRUE)
            ->setDisplayOptions('view', [
                'label' => 'above',
                'type' => 'timestamp',
                'weight' => 20,
            ])
            ->setDisplayConfigurable('form', TRUE)
            ->setDisplayConfigurable('view', TRUE);

        // Changed timestamp
        $fields['changed'] = BaseFieldDefinition::create('changed')
            ->setLabel(t('Changed'))
            ->setDescription(t('เวลาที่แก้ไขล่าสุด'))
            ->setRevisionable(TRUE);

        return $fields;
    }
}
```

### 2.3 สร้าง Entity Interface

```php
<?php
// src/ProductInterface.php
namespace Drupal\product_catalog;

use Drupal\Core\Entity\ContentEntityInterface;
use Drupal\Core\Entity\EntityChangedInterface;
use Drupal\Core\Entity\EntityPublishedInterface;
use Drupal\user\EntityOwnerInterface;

/**
 * Interface สำหรับ Product entity
 */
interface ProductInterface extends 
    ContentEntityInterface, 
    EntityChangedInterface,
    EntityPublishedInterface,
    EntityOwnerInterface {

    /**
     * ดึงชื่อสินค้า
     */
    public function getTitle(): string;

    /**
     * ตั้งชื่อสินค้า
     */
    public function setTitle(string $title): static;

    /**
     * ดึงราคาสินค้า
     */
    public function getPrice(): float;

    /**
     * ตั้งราคาสินค้า
     */
    public function setPrice(float $price): static;

    /**
     * ดึง SKU
     */
    public function getSku(): string;

    /**
     * ตั้ง SKU
     */
    public function setSku(string $sku): static;

    /**
     * ดึงเวลาที่สร้าง
     */
    public function getCreatedTime(): int;

    /**
     * ตั้งเวลาที่สร้าง
     */
    public function setCreatedTime(int $timestamp): static;
}
```

### 2.4 สร้าง Bundle Entity (Product Type)

```php
<?php
// src/Entity/ProductType.php
namespace Drupal\product_catalog\Entity;

use Drupal\Core\Config\Entity\ConfigEntityBundleBase;

/**
 * Defines the Product type entity.
 *
 * @ConfigEntityType(
 *   id = "product_type",
 *   label = @Translation("Product type"),
 *   label_collection = @Translation("Product types"),
 *   label_singular = @Translation("product type"),
 *   label_plural = @Translation("product types"),
 *   label_count = @PluralTranslation(
 *     singular = "@count product type",
 *     plural = "@count product types",
 *   ),
 *   handlers = {
 *     "form" = {
 *       "add" = "Drupal\product_catalog\Form\ProductTypeForm",
 *       "edit" = "Drupal\product_catalog\Form\ProductTypeForm",
 *       "delete" = "Drupal\Core\Entity\EntityDeleteForm",
 *     },
 *     "list_builder" = "Drupal\Core\Entity\EntityListBuilder",
 *     "route_provider" = {
 *       "html" = "Drupal\Core\Entity\Routing\AdminHtmlRouteProvider",
 *     },
 *   },
 *   admin_permission = "administer product types",
 *   bundle_of = "product",
 *   config_prefix = "type",
 *   entity_keys = {
 *     "id" = "id",
 *     "label" = "label",
 *     "uuid" = "uuid",
 *   },
 *   links = {
 *     "add-form" = "/admin/structure/product-types/add",
 *     "edit-form" = "/admin/structure/product-types/manage/{product_type}",
 *     "delete-form" = "/admin/structure/product-types/manage/{product_type}/delete",
 *     "collection" = "/admin/structure/product-types",
 *   },
 *   config_export = {
 *     "id",
 *     "label",
 *     "description",
 *   },
 * )
 */
class ProductType extends ConfigEntityBundleBase {

    /**
     * ID ของ product type
     */
    protected string $id;

    /**
     * Label ของ product type
     */
    protected string $label;

    /**
     * คำอธิบาย product type
     */
    protected string $description = '';

    /**
     * ดึงคำอธิบาย
     */
    public function getDescription(): string {
        return $this->description;
    }

    /**
     * ตั้งคำอธิบาย
     */
    public function setDescription(string $description): static {
        $this->description = $description;
        return $this;
    }
}
```

---

## ส่วนที่ 3: Custom Config Entities

### 3.1 ความแตกต่างระหว่าง Content Entity และ Config Entity

| คุณสมบัติ | Content Entity | Config Entity |
|-----------|---------------|---------------|
| เก็บข้อมูลใน | Database tables | Configuration system (YAML) |
| ตัวอย่าง | Node, User, Comment | Vocabulary, View, Role |
| รองรับ Fields | ใช่ | ไม่ |
| Deploy ได้ | ไม่โดยตรง | ใช่ (via Features/config sync) |
| Revisions | รองรับ | ไม่รองรับ |

### 3.2 สร้าง Config Entity สำหรับ Discount Rule

```php
<?php
// src/Entity/DiscountRule.php
namespace Drupal\product_catalog\Entity;

use Drupal\Core\Config\Entity\ConfigEntityBase;
use Drupal\Core\Entity\EntityWithPluginCollectionInterface;

/**
 * Defines the Discount Rule config entity.
 *
 * @ConfigEntityType(
 *   id = "discount_rule",
 *   label = @Translation("Discount Rule"),
 *   handlers = {
 *     "list_builder" = "Drupal\product_catalog\Entity\Handler\DiscountRuleListBuilder",
 *     "form" = {
 *       "add" = "Drupal\product_catalog\Form\DiscountRuleForm",
 *       "edit" = "Drupal\product_catalog\Form\DiscountRuleForm",
 *       "delete" = "Drupal\Core\Entity\EntityDeleteForm",
 *     },
 *     "route_provider" = {
 *       "html" = "Drupal\Core\Entity\Routing\AdminHtmlRouteProvider",
 *     },
 *   },
 *   config_prefix = "discount_rule",
 *   admin_permission = "administer discount rules",
 *   entity_keys = {
 *     "id" = "id",
 *     "label" = "label",
 *     "uuid" = "uuid",
 *     "status" = "status",
 *   },
 *   links = {
 *     "collection" = "/admin/config/catalog/discount-rules",
 *     "add-form" = "/admin/config/catalog/discount-rules/add",
 *     "edit-form" = "/admin/config/catalog/discount-rules/{discount_rule}",
 *     "delete-form" = "/admin/config/catalog/discount-rules/{discount_rule}/delete",
 *     "enable" = "/admin/config/catalog/discount-rules/{discount_rule}/enable",
 *     "disable" = "/admin/config/catalog/discount-rules/{discount_rule}/disable",
 *   },
 *   config_export = {
 *     "id",
 *     "label",
 *     "description",
 *     "discount_type",
 *     "discount_value",
 *     "conditions",
 *     "status",
 *   },
 * )
 */
class DiscountRule extends ConfigEntityBase {

    protected string $id;
    protected string $label;
    protected string $description = '';
    
    /**
     * ประเภทส่วนลด: 'percentage' หรือ 'fixed'
     */
    protected string $discount_type = 'percentage';
    
    /**
     * ค่าส่วนลด
     */
    protected float $discount_value = 0.0;
    
    /**
     * เงื่อนไขการใช้ส่วนลด (array)
     */
    protected array $conditions = [];

    public function getDiscountType(): string {
        return $this->discount_type;
    }

    public function getDiscountValue(): float {
        return $this->discount_value;
    }

    /**
     * คำนวณราคาหลังส่วนลด
     */
    public function applyDiscount(float $original_price): float {
        if ($this->discount_type === 'percentage') {
            return $original_price * (1 - $this->discount_value / 100);
        }
        return max(0, $original_price - $this->discount_value);
    }

    /**
     * ตรวจสอบว่า rule นี้ใช้กับสินค้านี้ได้หรือไม่
     */
    public function appliesTo(ProductInterface $product): bool {
        if (empty($this->conditions)) {
            return TRUE;
        }
        
        foreach ($this->conditions as $condition) {
            if (!$this->evaluateCondition($condition, $product)) {
                return FALSE;
            }
        }
        
        return TRUE;
    }

    private function evaluateCondition(array $condition, ProductInterface $product): bool {
        return match($condition['type']) {
            'min_price' => $product->getPrice() >= $condition['value'],
            'product_type' => $product->bundle() === $condition['value'],
            default => TRUE,
        };
    }
}
```

---

## ส่วนที่ 4: Entity CRUD Operations

### 4.1 Create (สร้าง Entity)

```php
<?php
// การสร้าง Product entity ใหม่

use Drupal\product_catalog\Entity\Product;

// วิธีที่ 1: ใช้ static create method
$product = Product::create([
    'type' => 'physical',           // bundle
    'title' => 'iPhone 15 Pro',
    'sku' => 'IPHONE-15-PRO-256',
    'price' => 45900.00,
    'stock_quantity' => 50,
    'description' => [
        'value' => '<p>สมาร์ทโฟนรุ่นล่าสุดจาก Apple</p>',
        'format' => 'basic_html',
    ],
    'status' => 1,
]);

// ตั้งค่า owner
$product->setOwnerId(\Drupal::currentUser()->id());

// บันทึกและตรวจสอบ errors
try {
    $product->save();
    \Drupal::logger('product_catalog')
        ->notice('Created product @id: @title', [
            '@id' => $product->id(),
            '@title' => $product->getTitle(),
        ]);
} catch (\Exception $e) {
    \Drupal::logger('product_catalog')
        ->error('Failed to create product: @message', [
            '@message' => $e->getMessage(),
        ]);
    throw $e;
}

// วิธีที่ 2: ผ่าน EntityTypeManager
$storage = \Drupal::entityTypeManager()->getStorage('product');
$product = $storage->create([
    'type' => 'digital',
    'title' => 'Photoshop License',
    'sku' => 'PS-LICENSE-2024',
    'price' => 899.00,
]);
$product->save();
```

### 4.2 Read (อ่าน Entity)

```php
<?php
// การอ่านข้อมูล Product entity

$storage = \Drupal::entityTypeManager()->getStorage('product');

// Load entity เดียว
$product = $storage->load(42);
if ($product) {
    echo $product->getTitle();
    echo $product->getPrice();
    echo $product->getSku();
}

// Load หลาย entities
$products = $storage->loadMultiple([1, 2, 3, 4, 5]);
foreach ($products as $id => $product) {
    echo "{$product->id()}: {$product->getTitle()} - {$product->getPrice()} บาท\n";
}

// Load โดย properties
$published_products = $storage->loadByProperties([
    'type' => 'physical',
    'status' => 1,
]);

// Load revision
$revision = $storage->loadRevision($revision_id);

// Load entity โดยตรง (หากรู้ entity type)
$product = \Drupal::entityTypeManager()
    ->getStorage('product')
    ->load($product_id);

// เข้าถึง fields
$title = $product->get('title')->value;
$price = $product->get('price')->value;

// เข้าถึง entity reference
$uid = $product->get('uid')->target_id;
$owner = $product->get('uid')->entity; // Load user entity

// เข้าถึง image field
$image_field = $product->get('image');
if (!$image_field->isEmpty()) {
    $image_item = $image_field->first();
    $file = $image_item->entity;
    $uri = $file->getFileUri();
    $url = \Drupal::service('file_url_generator')->generateAbsoluteString($uri);
}
```

### 4.3 Update (อัพเดท Entity)

```php
<?php
// การอัพเดท Product entity

$storage = \Drupal::entityTypeManager()->getStorage('product');
$product = $storage->load(42);

if ($product) {
    // อัพเดทผ่าน interface methods
    $product->setTitle('iPhone 15 Pro Max');
    $product->setPrice(52900.00);
    
    // อัพเดทผ่าน set method
    $product->set('stock_quantity', 100);
    $product->set('description', [
        'value' => '<p>อัพเดทรายละเอียดสินค้า</p>',
        'format' => 'basic_html',
    ]);
    
    // สร้าง revision ใหม่
    $product->setNewRevision(TRUE);
    $product->setRevisionLogMessage('อัพเดทราคาและ stock');
    $product->setRevisionUserId(\Drupal::currentUser()->id());
    $product->setRevisionCreationTime(\Drupal::time()->getRequestTime());
    
    $product->save();
}

// Batch update หลาย entities
$products = $storage->loadByProperties(['type' => 'physical']);
foreach ($products as $product) {
    // เพิ่มราคา 10%
    $new_price = $product->getPrice() * 1.1;
    $product->setPrice(round($new_price, 2));
    $product->save();
}
```

### 4.4 Delete (ลบ Entity)

```php
<?php
// การลบ Product entity

$storage = \Drupal::entityTypeManager()->getStorage('product');

// ลบ entity เดียว
$product = $storage->load(42);
if ($product) {
    $product->delete();
}

// ลบหลาย entities
$products = $storage->loadMultiple([1, 2, 3]);
$storage->delete($products);

// ลบ entity ที่ไม่ published และอายุเกิน 1 ปี
$one_year_ago = \Drupal::time()->getRequestTime() - (365 * 24 * 60 * 60);

$query = \Drupal::entityQuery('product')
    ->condition('status', 0)
    ->condition('created', $one_year_ago, '<')
    ->accessCheck(FALSE);

$ids = $query->execute();
if ($ids) {
    $products_to_delete = $storage->loadMultiple($ids);
    $storage->delete($products_to_delete);
    
    \Drupal::logger('product_catalog')
        ->notice('Deleted @count old unpublished products', [
            '@count' => count($products_to_delete),
        ]);
}
```

---

## ส่วนที่ 5: Entity Storage Handler

### 5.1 Custom Storage Handler

```php
<?php
// src/Entity/Handler/ProductStorage.php
namespace Drupal\product_catalog\Entity\Handler;

use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Database\Connection;
use Drupal\Core\Entity\EntityFieldManagerInterface;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Entity\EntityTypeBundleInfoInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Entity\Sql\SqlContentEntityStorage;
use Drupal\Core\Language\LanguageManagerInterface;
use Drupal\product_catalog\ProductInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Custom storage handler สำหรับ Product entity
 */
class ProductStorage extends SqlContentEntityStorage {

    public function __construct(
        EntityTypeInterface $entity_type,
        Connection $database,
        EntityFieldManagerInterface $entity_field_manager,
        CacheBackendInterface $cache,
        LanguageManagerInterface $language_manager,
        EntityTypeBundleInfoInterface $entity_type_bundle_info,
        EntityTypeManagerInterface $entity_type_manager,
        private readonly CacheBackendInterface $productCache
    ) {
        parent::__construct(
            $entity_type, $database, $entity_field_manager,
            $cache, $language_manager, $entity_type_bundle_info,
            $entity_type_manager
        );
    }

    /**
     * ดึงสินค้าที่ stock เกือบหมด
     */
    public function getLowStockProducts(int $threshold = 5): array {
        $query = $this->getQuery()
            ->condition('status', 1)
            ->condition('stock_quantity', $threshold, '<=')
            ->condition('stock_quantity', 0, '>')
            ->sort('stock_quantity', 'ASC')
            ->accessCheck(FALSE);

        $ids = $query->execute();
        return $this->loadMultiple($ids);
    }

    /**
     * ดึงสินค้าตาม price range
     */
    public function getProductsByPriceRange(float $min, float $max): array {
        $query = $this->getQuery()
            ->condition('status', 1)
            ->condition('price', $min, '>=')
            ->condition('price', $max, '<=')
            ->sort('price', 'ASC')
            ->accessCheck(TRUE);

        $ids = $query->execute();
        return $this->loadMultiple($ids);
    }

    /**
     * ดึงสินค้าใหม่ล่าสุด
     */
    public function getLatestProducts(int $count = 10): array {
        $cid = "product_catalog:latest:{$count}";
        
        // ลองดึงจาก cache ก่อน
        $cached = $this->productCache->get($cid);
        if ($cached) {
            return $cached->data;
        }

        $query = $this->getQuery()
            ->condition('status', 1)
            ->sort('created', 'DESC')
            ->range(0, $count)
            ->accessCheck(TRUE);

        $ids = $query->execute();
        $products = $this->loadMultiple($ids);

        // เก็บใน cache 1 ชั่วโมง
        $this->productCache->set($cid, $products, time() + 3600, [
            'product_catalog_list',
        ]);

        return $products;
    }

    /**
     * อัพเดท stock quantity โดยตรง (ไม่ผ่าน entity save)
     * ใช้สำหรับ high-performance stock updates
     */
    public function updateStock(int $product_id, int $quantity_change): bool {
        $result = $this->database->update('product')
            ->expression('stock_quantity', 'stock_quantity + :change', [
                ':change' => $quantity_change,
            ])
            ->condition('id', $product_id)
            ->condition('stock_quantity + :check_change', 0, '>=', [
                ':check_change' => $quantity_change,
            ])
            ->execute();

        if ($result) {
            // Invalidate cache สำหรับ entity นี้
            $this->resetCache([$product_id]);
        }

        return (bool) $result;
    }

    /**
     * {@inheritdoc}
     */
    protected function doDelete(array $entities): void {
        // Custom logic ก่อนลบ
        foreach ($entities as $entity) {
            \Drupal::moduleHandler()->invokeAll(
                'product_pre_delete', [$entity]
            );
        }
        
        parent::doDelete($entities);
    }
}
```

---

## ส่วนที่ 6: Entity Access Control Handler

### 6.1 Custom Access Control Handler

```php
<?php
// src/Entity/Handler/ProductAccessControlHandler.php
namespace Drupal\product_catalog\Entity\Handler;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Entity\EntityAccessControlHandler;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Session\AccountInterface;

/**
 * Custom access control สำหรับ Product entity
 */
class ProductAccessControlHandler extends EntityAccessControlHandler {

    /**
     * {@inheritdoc}
     */
    protected function checkAccess(
        EntityInterface $entity,
        string $operation,
        AccountInterface $account
    ): AccessResultInterface {
        
        /** @var \Drupal\product_catalog\ProductInterface $entity */
        
        switch ($operation) {
            case 'view':
                // ดูได้ถ้า published หรือมีสิทธิ์ administer
                if (!$entity->isPublished()) {
                    return AccessResult::allowedIfHasPermission(
                        $account, 'administer products'
                    )->cachePerPermissions()
                     ->addCacheableDependency($entity);
                }
                return AccessResult::allowedIfHasPermission(
                    $account, 'view products'
                )->cachePerPermissions();

            case 'update':
                // แก้ไขได้ถ้าเป็น owner หรือมีสิทธิ์ administer
                if ($account->hasPermission('administer products')) {
                    return AccessResult::allowed()
                        ->cachePerPermissions();
                }
                
                // ตรวจสอบ ownership
                $is_owner = ($entity->getOwnerId() === $account->id());
                return AccessResult::allowedIf(
                    $is_owner && $account->hasPermission('edit own products')
                )->cachePerPermissions()
                 ->cachePerUser()
                 ->addCacheableDependency($entity);

            case 'delete':
                return AccessResult::allowedIfHasPermission(
                    $account, 'delete products'
                )->cachePerPermissions();

            case 'view revision':
            case 'revert revision':
            case 'delete revision':
                return AccessResult::allowedIfHasPermission(
                    $account, 'administer products'
                )->cachePerPermissions();
        }

        return parent::checkAccess($entity, $operation, $account);
    }

    /**
     * {@inheritdoc}
     */
    protected function checkCreateAccess(
        AccountInterface $account,
        array $context,
        ?string $entity_bundle = NULL
    ): AccessResultInterface {
        
        return AccessResult::allowedIfHasPermissions(
            $account, ['create products', 'administer products'], 'OR'
        )->cachePerPermissions();
    }
}
```

---

## ส่วนที่ 7: Entity Form Handler

### 7.1 Custom Entity Form

```php
<?php
// src/Form/ProductForm.php
namespace Drupal\product_catalog\Form;

use Drupal\Core\Entity\ContentEntityForm;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Messenger\MessengerInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Form handler สำหรับ add และ edit product
 */
class ProductForm extends ContentEntityForm {

    public function __construct(
        // ... parent dependencies ...
        private readonly MessengerInterface $messenger
    ) {
        // parent constructor
    }

    public static function create(ContainerInterface $container): static {
        return new static(
            $container->get('entity.repository'),
            $container->get('entity_type.bundle.info'),
            $container->get('datetime.time'),
            $container->get('messenger')
        );
    }

    /**
     * {@inheritdoc}
     */
    public function form(array $form, FormStateInterface $form_state): array {
        $form = parent::form($form, $form_state);
        
        /** @var \Drupal\product_catalog\ProductInterface $product */
        $product = $this->entity;
        
        // เพิ่ม information section
        $form['product_info'] = [
            '#type' => 'details',
            '#title' => $this->t('ข้อมูลสินค้า'),
            '#open' => TRUE,
            '#weight' => -10,
        ];
        
        // ย้าย fields เข้า section
        if (isset($form['title'])) {
            $form['title']['#group'] = 'product_info';
        }
        if (isset($form['sku'])) {
            $form['sku']['#group'] = 'product_info';
        }
        if (isset($form['price'])) {
            $form['price']['#group'] = 'product_info';
        }

        // เพิ่ม inventory section
        $form['inventory'] = [
            '#type' => 'details',
            '#title' => $this->t('คลังสินค้า'),
            '#open' => TRUE,
            '#weight' => -5,
        ];
        
        if (isset($form['stock_quantity'])) {
            $form['stock_quantity']['#group'] = 'inventory';
        }
        
        // เพิ่ม sidebar สำหรับ options
        $form['advanced'] = [
            '#type' => 'vertical_tabs',
            '#title' => $this->t('การตั้งค่าขั้นสูง'),
            '#weight' => 99,
        ];
        
        $form['publishing_options'] = [
            '#type' => 'details',
            '#title' => $this->t('ตัวเลือกการเผยแพร่'),
            '#group' => 'advanced',
            '#weight' => 10,
        ];
        
        if (isset($form['status'])) {
            $form['status']['#group'] = 'publishing_options';
        }

        // เพิ่ม custom validation message สำหรับ SKU
        if (isset($form['sku']['widget'][0]['value'])) {
            $form['sku']['widget'][0]['value']['#description'] = $this->t(
                'รหัสสินค้าต้องไม่ซ้ำกัน ใช้ตัวอักษรพิมพ์ใหญ่และตัวเลขเท่านั้น'
            );
        }

        return $form;
    }

    /**
     * {@inheritdoc}
     */
    public function validateForm(array &$form, FormStateInterface $form_state): void {
        parent::validateForm($form, $form_state);
        
        // ตรวจสอบ SKU format
        $sku = $form_state->getValue(['sku', 0, 'value']);
        if ($sku && !preg_match('/^[A-Z0-9\-_]+$/', $sku)) {
            $form_state->setErrorByName('sku', $this->t(
                'SKU ต้องประกอบด้วยตัวอักษรพิมพ์ใหญ่ (A-Z), ตัวเลข (0-9), เครื่องหมาย (-) หรือ (_) เท่านั้น'
            ));
        }
        
        // ตรวจสอบ price
        $price = $form_state->getValue(['price', 0, 'value']);
        if ($price !== NULL && $price < 0) {
            $form_state->setErrorByName('price', $this->t(
                'ราคาต้องไม่ต่ำกว่า 0'
            ));
        }
    }

    /**
     * {@inheritdoc}
     */
    public function save(array $form, FormStateInterface $form_state): int {
        $product = $this->entity;
        $status = $product->save();
        
        if ($status === SAVED_NEW) {
            $this->messenger->addStatus($this->t(
                'สร้างสินค้า %title เรียบร้อยแล้ว',
                ['%title' => $product->getTitle()]
            ));
        } else {
            $this->messenger->addStatus($this->t(
                'อัพเดทสินค้า %title เรียบร้อยแล้ว',
                ['%title' => $product->getTitle()]
            ));
        }
        
        $form_state->setRedirectUrl($product->toUrl());
        return $status;
    }
}
```

---

## ส่วนที่ 8: Entity List Builder

### 8.1 Custom List Builder

```php
<?php
// src/Entity/Handler/ProductListBuilder.php
namespace Drupal\product_catalog\Entity\Handler;

use Drupal\Core\Datetime\DateFormatterInterface;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Entity\EntityListBuilder;
use Drupal\Core\Entity\EntityStorageInterface;
use Drupal\Core\Entity\EntityTypeInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * List builder สำหรับ Product entity
 */
class ProductListBuilder extends EntityListBuilder {

    public function __construct(
        EntityTypeInterface $entity_type,
        EntityStorageInterface $storage,
        private readonly DateFormatterInterface $dateFormatter
    ) {
        parent::__construct($entity_type, $storage);
    }

    public static function createInstance(
        ContainerInterface $container,
        EntityTypeInterface $entity_type
    ): static {
        return new static(
            $entity_type,
            $container->get('entity_type.manager')->getStorage($entity_type->id()),
            $container->get('date.formatter')
        );
    }

    /**
     * กำหนด header ของตาราง
     * {@inheritdoc}
     */
    public function buildHeader(): array {
        $header = [
            'id' => $this->t('ID'),
            'title' => $this->t('ชื่อสินค้า'),
            'sku' => $this->t('SKU'),
            'price' => $this->t('ราคา'),
            'stock' => $this->t('คลัง'),
            'status' => $this->t('สถานะ'),
            'created' => $this->t('วันที่สร้าง'),
        ];
        return $header + parent::buildHeader();
    }

    /**
     * กำหนดข้อมูลในแต่ละแถว
     * {@inheritdoc}
     */
    public function buildRow(EntityInterface $entity): array {
        /** @var \Drupal\product_catalog\ProductInterface $entity */
        $row = [
            'id' => $entity->id(),
            'title' => [
                'data' => [
                    '#type' => 'link',
                    '#title' => $entity->getTitle(),
                    '#url' => $entity->toUrl(),
                ],
            ],
            'sku' => $entity->getSku(),
            'price' => number_format($entity->getPrice(), 2) . ' บาท',
            'stock' => $entity->get('stock_quantity')->value,
            'status' => $entity->isPublished() 
                ? $this->t('เผยแพร่') 
                : $this->t('ไม่เผยแพร่'),
            'created' => $this->dateFormatter->format(
                $entity->getCreatedTime(),
                'short'
            ),
        ];
        return $row + parent::buildRow($entity);
    }

    /**
     * เพิ่ม filter form ด้านบน
     * {@inheritdoc}
     */
    public function render(): array {
        $build = parent::render();
        
        // เพิ่มข้อมูลสรุป
        $build['summary'] = [
            '#markup' => $this->t('แสดง @count สินค้า', [
                '@count' => count($this->load()),
            ]),
            '#weight' => -10,
        ];
        
        return $build;
    }
}
```

---

## ส่วนที่ 9: Entity Hooks และ Events

### 9.1 Entity Hooks ใน .module file

```php
<?php
// product_catalog.module

/**
 * Implements hook_entity_presave().
 * เรียกก่อน save entity ทุกประเภท
 */
function product_catalog_entity_presave(\Drupal\Core\Entity\EntityInterface $entity): void {
    // ตรวจสอบว่าเป็น product entity หรือไม่
    if ($entity->getEntityTypeId() !== 'product') {
        return;
    }
    
    // Auto-generate SKU ถ้าไม่มี
    /** @var \Drupal\product_catalog\ProductInterface $entity */
    if (!$entity->getSku() && $entity->isNew()) {
        $sku = 'PROD-' . strtoupper(uniqid());
        $entity->setSku($sku);
    }
}

/**
 * Implements hook_ENTITY_TYPE_presave() for product entities.
 * เรียกก่อน save เฉพาะ product entity
 */
function product_catalog_product_presave(\Drupal\product_catalog\ProductInterface $entity): void {
    // Log การสร้างสินค้าใหม่
    if ($entity->isNew()) {
        \Drupal::logger('product_catalog')->info(
            'กำลังสร้างสินค้าใหม่: @title (SKU: @sku)',
            ['@title' => $entity->getTitle(), '@sku' => $entity->getSku()]
        );
    }
}

/**
 * Implements hook_ENTITY_TYPE_insert() for product entities.
 * เรียกหลัง insert entity ใหม่
 */
function product_catalog_product_insert(\Drupal\product_catalog\ProductInterface $entity): void {
    // ส่ง notification เมื่อสร้างสินค้าใหม่
    $config = \Drupal::config('product_catalog.settings');
    if ($config->get('notify_on_create')) {
        _product_catalog_send_notification('new_product', $entity);
    }
    
    // Invalidate cache lists
    \Drupal\Core\Cache\Cache::invalidateTags(['product_catalog_list']);
}

/**
 * Implements hook_ENTITY_TYPE_update() for product entities.
 * เรียกหลัง update entity
 */
function product_catalog_product_update(\Drupal\product_catalog\ProductInterface $entity): void {
    // ตรวจสอบการเปลี่ยนแปลงราคา
    $original = $entity->original;
    if ($original && $entity->getPrice() !== $original->getPrice()) {
        $old_price = $original->getPrice();
        $new_price = $entity->getPrice();
        
        \Drupal::logger('product_catalog')->notice(
            'สินค้า @title: ราคาเปลี่ยนจาก @old เป็น @new',
            [
                '@title' => $entity->getTitle(),
                '@old' => number_format($old_price, 2),
                '@new' => number_format($new_price, 2),
            ]
        );
    }
    
    // Invalidate cache
    \Drupal\Core\Cache\Cache::invalidateTags([
        'product:' . $entity->id(),
        'product_catalog_list',
    ]);
}

/**
 * Implements hook_ENTITY_TYPE_delete() for product entities.
 * เรียกหลัง delete entity
 */
function product_catalog_product_delete(\Drupal\product_catalog\ProductInterface $entity): void {
    // ลบ related data
    \Drupal::database()->delete('product_catalog_analytics')
        ->condition('product_id', $entity->id())
        ->execute();
    
    // Invalidate cache
    \Drupal\Core\Cache\Cache::invalidateTags(['product_catalog_list']);
}

/**
 * Implements hook_ENTITY_TYPE_view() for product entities.
 * เรียกเมื่อ render entity view
 */
function product_catalog_product_view(
    array &$build,
    \Drupal\product_catalog\ProductInterface $entity,
    \Drupal\Core\Entity\Display\EntityViewDisplayInterface $display,
    string $view_mode
): void {
    // เพิ่ม "สินค้าหมดแล้ว" badge ถ้า stock เป็น 0
    $stock = $entity->get('stock_quantity')->value;
    if ($stock == 0) {
        $build['out_of_stock'] = [
            '#markup' => '<span class="badge badge-danger">สินค้าหมด</span>',
            '#weight' => -100,
        ];
    }
    
    // เพิ่ม breadcrumb cache tag
    $build['#cache']['tags'][] = 'product:' . $entity->id();
}
```

### 9.2 Entity Events ด้วย Symfony Event System

```php
<?php
// src/Event/ProductPriceChangedEvent.php
namespace Drupal\product_catalog\Event;

use Drupal\product_catalog\ProductInterface;
use Symfony\Contracts\EventDispatcher\Event;

/**
 * Event ที่เกิดขึ้นเมื่อราคาสินค้าเปลี่ยน
 */
class ProductPriceChangedEvent extends Event {

    const EVENT_NAME = 'product_catalog.price_changed';

    public function __construct(
        private readonly ProductInterface $product,
        private readonly float $oldPrice,
        private readonly float $newPrice
    ) {}

    public function getProduct(): ProductInterface {
        return $this->product;
    }

    public function getOldPrice(): float {
        return $this->oldPrice;
    }

    public function getNewPrice(): float {
        return $this->newPrice;
    }

    public function getPriceChange(): float {
        return $this->newPrice - $this->oldPrice;
    }

    public function getPriceChangePercent(): float {
        if ($this->oldPrice == 0) {
            return 0;
        }
        return (($this->newPrice - $this->oldPrice) / $this->oldPrice) * 100;
    }
}

// src/EventSubscriber/ProductEventSubscriber.php
namespace Drupal\product_catalog\EventSubscriber;

use Drupal\product_catalog\Event\ProductPriceChangedEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

/**
 * Event subscriber สำหรับ product events
 */
class ProductEventSubscriber implements EventSubscriberInterface {

    public static function getSubscribedEvents(): array {
        return [
            ProductPriceChangedEvent::EVENT_NAME => [
                ['onPriceChanged', 0],
            ],
        ];
    }

    public function onPriceChanged(ProductPriceChangedEvent $event): void {
        $product = $event->getProduct();
        $change = $event->getPriceChangePercent();
        
        // แจ้งเตือนถ้าราคาลดมากกว่า 20%
        if ($change < -20) {
            \Drupal::logger('product_catalog')->warning(
                'สินค้า @title มีการลดราคามากกว่า 20% (@percent%)',
                [
                    '@title' => $product->getTitle(),
                    '@percent' => round(abs($change), 1),
                ]
            );
        }
    }
}
```

---

## ส่วนที่ 10: Field API Integration

### 10.1 สร้าง Custom Field Type

```php
<?php
// src/Plugin/Field/FieldType/PriceFieldItem.php
namespace Drupal\product_catalog\Plugin\Field\FieldType;

use Drupal\Core\Field\FieldItemBase;
use Drupal\Core\Field\FieldStorageDefinitionInterface;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\TypedData\DataDefinition;

/**
 * Field type สำหรับราคาพร้อม currency
 *
 * @FieldType(
 *   id = "product_price",
 *   label = @Translation("Product Price"),
 *   description = @Translation("ราคาสินค้าพร้อมสกุลเงิน"),
 *   default_widget = "product_price_widget",
 *   default_formatter = "product_price_formatter",
 * )
 */
class PriceFieldItem extends FieldItemBase {

    /**
     * {@inheritdoc}
     */
    public static function schema(FieldStorageDefinitionInterface $field_definition): array {
        return [
            'columns' => [
                'amount' => [
                    'type' => 'numeric',
                    'precision' => 10,
                    'scale' => 2,
                    'not null' => FALSE,
                ],
                'currency_code' => [
                    'type' => 'varchar',
                    'length' => 3,
                    'not null' => FALSE,
                ],
            ],
        ];
    }

    /**
     * {@inheritdoc}
     */
    public static function propertyDefinitions(
        FieldStorageDefinitionInterface $field_definition
    ): array {
        $properties['amount'] = DataDefinition::create('float')
            ->setLabel(t('Amount'))
            ->setRequired(TRUE);

        $properties['currency_code'] = DataDefinition::create('string')
            ->setLabel(t('Currency Code'))
            ->setRequired(TRUE);

        return $properties;
    }

    /**
     * {@inheritdoc}
     */
    public function isEmpty(): bool {
        $amount = $this->get('amount')->getValue();
        return $amount === NULL || $amount === '';
    }

    /**
     * {@inheritdoc}
     */
    public static function defaultFieldSettings(): array {
        return [
            'available_currencies' => ['THB', 'USD', 'EUR'],
            'default_currency' => 'THB',
        ] + parent::defaultFieldSettings();
    }

    /**
     * ดึงราคาในรูปแบบ formatted string
     */
    public function getFormattedAmount(): string {
        $amount = $this->get('amount')->getValue();
        $currency = $this->get('currency_code')->getValue();
        return number_format($amount, 2) . ' ' . $currency;
    }
}
```

### 10.2 สร้าง Field Widget

```php
<?php
// src/Plugin/Field/FieldWidget/PriceWidget.php
namespace Drupal\product_catalog\Plugin\Field\FieldWidget;

use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\WidgetBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Widget สำหรับ product_price field
 *
 * @FieldWidget(
 *   id = "product_price_widget",
 *   label = @Translation("Price Input"),
 *   field_types = {
 *     "product_price",
 *   }
 * )
 */
class PriceWidget extends WidgetBase {

    /**
     * {@inheritdoc}
     */
    public function formElement(
        FieldItemListInterface $items,
        int $delta,
        array $element,
        array &$form,
        FormStateInterface $form_state
    ): array {
        $item = $items[$delta];
        
        $element['amount'] = [
            '#type' => 'number',
            '#title' => $this->t('ราคา'),
            '#default_value' => $item->amount ?? '',
            '#min' => 0,
            '#step' => '0.01',
            '#required' => $element['#required'],
        ];
        
        $settings = $this->getFieldSetting('available_currencies') 
            ?? ['THB', 'USD', 'EUR'];
        
        $options = array_combine($settings, $settings);
        
        $element['currency_code'] = [
            '#type' => 'select',
            '#title' => $this->t('สกุลเงิน'),
            '#options' => $options,
            '#default_value' => $item->currency_code 
                ?? $this->getFieldSetting('default_currency') 
                ?? 'THB',
        ];
        
        return $element;
    }
}
```

### 10.3 สร้าง Field Formatter

```php
<?php
// src/Plugin/Field/FieldFormatter/PriceFormatter.php
namespace Drupal\product_catalog\Plugin\Field\FieldFormatter;

use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\FormatterBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Formatter สำหรับ product_price field
 *
 * @FieldFormatter(
 *   id = "product_price_formatter",
 *   label = @Translation("Price Display"),
 *   field_types = {
 *     "product_price",
 *   }
 * )
 */
class PriceFormatter extends FormatterBase {

    /**
     * {@inheritdoc}
     */
    public static function defaultSettings(): array {
        return [
            'show_currency_symbol' => TRUE,
            'currency_position' => 'after',
        ] + parent::defaultSettings();
    }

    /**
     * {@inheritdoc}
     */
    public function viewElements(FieldItemListInterface $items, string $langcode): array {
        $elements = [];
        
        foreach ($items as $delta => $item) {
            if ($item->isEmpty()) {
                continue;
            }
            
            $amount = number_format($item->amount, 2, '.', ',');
            $currency = $item->currency_code;
            
            $formatted = $this->getSetting('currency_position') === 'before'
                ? "{$currency} {$amount}"
                : "{$amount} {$currency}";
            
            $elements[$delta] = [
                '#markup' => '<span class="product-price">' . $formatted . '</span>',
                '#cache' => ['contexts' => ['languages:language_interface']],
            ];
        }
        
        return $elements;
    }
}
```

---

## ส่วนที่ 11: Entity Query ขั้นสูง

### 11.1 Complex Entity Queries

```php
<?php
// การใช้ EntityQuery ขั้นสูง

$entityQuery = \Drupal::entityQuery('product');

// Query ที่ซับซ้อน: OR conditions
$group = $entityQuery->orConditionGroup()
    ->condition('type', 'physical')
    ->condition('type', 'digital');

$products = $entityQuery
    ->condition($group)
    ->condition('status', 1)
    ->condition('price', 100, '>=')
    ->sort('price', 'ASC')
    ->range(0, 20)
    ->accessCheck(TRUE)
    ->execute();

// Query พร้อม field conditions
$query = \Drupal::entityQuery('product')
    ->condition('status', 1)
    ->condition('field_category.entity.name', 'Electronics')  // Entity reference condition
    ->condition('field_tags.entity.name', ['Sale', 'Featured'], 'IN')  // Multiple values
    ->notExists('field_expiry_date')  // Field ที่ไม่มีค่า
    ->sort('created', 'DESC');

// Pager query
$query = \Drupal::entityQuery('product')
    ->condition('status', 1)
    ->sort('title', 'ASC')
    ->pager(10)  // 10 items per page
    ->accessCheck(TRUE);

$ids = $query->execute();

// Count query
$count = \Drupal::entityQuery('product')
    ->condition('status', 1)
    ->count()
    ->accessCheck(FALSE)
    ->execute();

echo "มีสินค้าที่เผยแพร่ {$count} รายการ";

// Query พร้อม aggregate (ผ่าน database query)
$connection = \Drupal::database();
$result = $connection->select('product', 'p')
    ->fields('p', ['type'])
    ->groupBy('type')
    ->execute()
    ->fetchAll();

// หา entity IDs ที่ user นี้เป็นเจ้าของ
$uid = \Drupal::currentUser()->id();
$my_products = \Drupal::entityQuery('product')
    ->condition('uid', $uid)
    ->condition('status', 1)
    ->sort('changed', 'DESC')
    ->accessCheck(TRUE)
    ->execute();
```

### 11.2 Entity Query กับ Views

```php
<?php
// การสร้าง Views programmatically สำหรับ Product

// หรือใช้ Views API ด้วย PHP
$view = \Drupal\views\Views::getView('product_catalog');
if ($view) {
    $view->setDisplay('default');
    $view->setArguments([$product_type]);
    $view->execute();
    
    $results = $view->result;
    foreach ($results as $row) {
        $product = $row->_entity;
        // ประมวลผล...
    }
    
    // Render view
    $rendered = $view->buildRenderable('page_1', [$product_type]);
}
```

---

## Workshop: สร้าง Product Catalog Entity

### Workshop Overview

ในส่วนนี้เราจะสร้าง complete product catalog module ตั้งแต่ต้นจนจบ

### Step 1: product_catalog.info.yml

```yaml
# product_catalog.info.yml
name: Product Catalog
type: module
description: 'Custom product catalog using Drupal Entity API'
core_version_requirement: ^10
package: Custom
dependencies:
  - drupal:field
  - drupal:text
  - drupal:image
  - drupal:user
  - drupal:views
```

### Step 2: product_catalog.permissions.yml

```yaml
# product_catalog.permissions.yml
administer products:
  title: 'Administer products'
  description: 'Full access to product management'
  restrict access: true

administer product types:
  title: 'Administer product types'
  restrict access: true

create products:
  title: 'Create products'

edit own products:
  title: 'Edit own products'

delete products:
  title: 'Delete products'

view products:
  title: 'View published products'
```

### Step 3: product_catalog.routing.yml

```yaml
# product_catalog.routing.yml
entity.product.canonical:
  path: '/product/{product}'
  defaults:
    _entity_view: 'product'
    _title_callback: '\Drupal\Core\Entity\Controller\EntityController::title'
  requirements:
    _entity_access: 'product.view'

entity.product.add_page:
  path: '/product/add'
  defaults:
    _controller: '\Drupal\product_catalog\Controller\ProductController::addPage'
    _title: 'เพิ่มสินค้าใหม่'
  requirements:
    _permission: 'create products'

entity.product.add_form:
  path: '/product/add/{product_type}'
  defaults:
    _entity_form: 'product.add'
    _title: 'เพิ่มสินค้าใหม่'
  requirements:
    _entity_create_access: 'product'

entity.product.collection:
  path: '/admin/content/products'
  defaults:
    _entity_list: 'product'
    _title: 'รายการสินค้า'
  requirements:
    _permission: 'administer products'
```

### Step 4: Complete Product Entity Test

```php
<?php
// tests/src/Kernel/ProductEntityTest.php
namespace Drupal\Tests\product_catalog\Kernel;

use Drupal\KernelTests\Core\Entity\EntityKernelTestBase;
use Drupal\product_catalog\Entity\Product;

/**
 * Tests สำหรับ Product entity
 *
 * @group product_catalog
 */
class ProductEntityTest extends EntityKernelTestBase {

    protected static $modules = [
        'product_catalog',
        'image',
        'file',
    ];

    protected function setUp(): void {
        parent::setUp();
        $this->installEntitySchema('product');
        $this->installEntitySchema('product_type');
        $this->installConfig(['product_catalog']);
    }

    /**
     * ทดสอบการสร้าง Product entity
     */
    public function testProductCreation(): void {
        $product = Product::create([
            'type' => 'default',
            'title' => 'Test Product',
            'sku' => 'TEST-001',
            'price' => 99.99,
            'stock_quantity' => 10,
            'status' => 1,
        ]);

        $this->assertInstanceOf(Product::class, $product);
        $this->assertEquals('Test Product', $product->getTitle());
        $this->assertEquals('TEST-001', $product->getSku());
        $this->assertEquals(99.99, $product->getPrice());
        $this->assertTrue($product->isNew());

        $product->save();
        $this->assertFalse($product->isNew());
        $this->assertNotNull($product->id());
    }

    /**
     * ทดสอบ CRUD operations
     */
    public function testProductCrud(): void {
        // Create
        $product = Product::create([
            'type' => 'default',
            'title' => 'CRUD Test',
            'sku' => 'CRUD-001',
            'price' => 50.00,
        ]);
        $product->save();
        $id = $product->id();

        // Read
        $loaded = Product::load($id);
        $this->assertEquals('CRUD Test', $loaded->getTitle());

        // Update
        $loaded->setTitle('Updated Title');
        $loaded->setPrice(75.00);
        $loaded->save();

        $updated = Product::load($id);
        $this->assertEquals('Updated Title', $updated->getTitle());
        $this->assertEquals(75.00, $updated->getPrice());

        // Delete
        $updated->delete();
        $deleted = Product::load($id);
        $this->assertNull($deleted);
    }

    /**
     * ทดสอบ Entity Query
     */
    public function testProductQuery(): void {
        // สร้างข้อมูลทดสอบ
        for ($i = 1; $i <= 5; $i++) {
            Product::create([
                'type' => 'default',
                'title' => "Product {$i}",
                'sku' => "SKU-{$i}",
                'price' => $i * 100,
                'status' => 1,
            ])->save();
        }

        // Query เฉพาะ products ที่ราคา >= 300
        $ids = \Drupal::entityQuery('product')
            ->condition('price', 300, '>=')
            ->condition('status', 1)
            ->accessCheck(FALSE)
            ->execute();

        $this->assertCount(3, $ids);
    }
}
```

### Step 5: Service Definition

```yaml
# product_catalog.services.yml
services:
  product_catalog.product_manager:
    class: Drupal\product_catalog\ProductManager
    arguments:
      - '@entity_type.manager'
      - '@current_user'
      - '@logger.factory'
      - '@cache.default'

  Drupal\product_catalog\EventSubscriber\ProductEventSubscriber:
    tags:
      - { name: event_subscriber }
```

### Step 6: ProductManager Service

```php
<?php
// src/ProductManager.php
namespace Drupal\product_catalog;

use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Logger\LoggerChannelFactoryInterface;
use Drupal\Core\Session\AccountInterface;

/**
 * Service สำหรับจัดการ Product logic
 */
class ProductManager {

    public function __construct(
        private readonly EntityTypeManagerInterface $entityTypeManager,
        private readonly AccountInterface $currentUser,
        private readonly LoggerChannelFactoryInterface $loggerFactory,
        private readonly CacheBackendInterface $cache
    ) {}

    /**
     * ดึง featured products
     */
    public function getFeaturedProducts(int $limit = 4): array {
        $cid = "product_catalog:featured:{$limit}";
        
        $cached = $this->cache->get($cid);
        if ($cached !== FALSE) {
            return $cached->data;
        }

        $ids = $this->entityTypeManager
            ->getStorage('product')
            ->getQuery()
            ->condition('status', 1)
            ->condition('field_featured', 1)
            ->sort('changed', 'DESC')
            ->range(0, $limit)
            ->accessCheck(TRUE)
            ->execute();

        $products = $this->entityTypeManager
            ->getStorage('product')
            ->loadMultiple($ids);

        $this->cache->set($cid, $products, time() + 1800, [
            'product_catalog_list',
        ]);

        return $products;
    }

    /**
     * ค้นหาสินค้า
     */
    public function searchProducts(
        string $search_term,
        array $filters = [],
        int $page = 0,
        int $per_page = 20
    ): array {
        $storage = $this->entityTypeManager->getStorage('product');
        
        $query = $storage->getQuery()
            ->condition('status', 1)
            ->accessCheck(TRUE);

        // Full-text search ผ่าน title
        if (!empty($search_term)) {
            $query->condition('title', '%' . $search_term . '%', 'LIKE');
        }

        // Apply filters
        if (!empty($filters['type'])) {
            $query->condition('type', $filters['type']);
        }
        if (!empty($filters['min_price'])) {
            $query->condition('price', $filters['min_price'], '>=');
        }
        if (!empty($filters['max_price'])) {
            $query->condition('price', $filters['max_price'], '<=');
        }
        if (isset($filters['in_stock']) && $filters['in_stock']) {
            $query->condition('stock_quantity', 0, '>');
        }

        // Pagination
        $query->range($page * $per_page, $per_page);
        $query->sort('created', 'DESC');

        $ids = $query->execute();
        return $storage->loadMultiple($ids);
    }

    /**
     * ลดจำนวน stock เมื่อมีการสั่งซื้อ
     */
    public function decrementStock(int $product_id, int $quantity): bool {
        $storage = $this->entityTypeManager->getStorage('product');
        
        /** @var \Drupal\product_catalog\ProductInterface $product */
        $product = $storage->load($product_id);
        
        if (!$product) {
            return FALSE;
        }
        
        $current_stock = $product->get('stock_quantity')->value;
        
        if ($current_stock < $quantity) {
            $this->loggerFactory->get('product_catalog')->warning(
                'ไม่สามารถลด stock สินค้า @title: stock ปัจจุบัน (@current) น้อยกว่าที่ต้องการ (@required)',
                [
                    '@title' => $product->getTitle(),
                    '@current' => $current_stock,
                    '@required' => $quantity,
                ]
            );
            return FALSE;
        }
        
        $product->set('stock_quantity', $current_stock - $quantity);
        $product->save();
        
        return TRUE;
    }
}
```

---

## ส่วนที่ 12: Best Practices และ Performance Tips

### 12.1 Cache ที่ถูกต้อง

```php
<?php
// การตั้ง cache metadata ให้ถูกต้อง

// 1. Cache tags - invalidate เมื่อข้อมูลเปลี่ยน
$build = [
    '#markup' => $rendered_content,
    '#cache' => [
        'tags' => array_merge(
            $product->getCacheTags(),
            ['product_catalog_list']
        ),
        'contexts' => ['user.permissions', 'languages:language_interface'],
        'max-age' => 3600,  // 1 ชั่วโมง
    ],
];

// 2. ใช้ Entity::getCacheTags() สำหรับ entity-specific tags
$cache_tags = $product->getCacheTags();
// ผลลัพธ์: ['product:42']

// 3. Invalidate tags เมื่ออัพเดทข้อมูล
\Drupal\Core\Cache\Cache::invalidateTags(['product:42', 'product_catalog_list']);
```

### 12.2 Avoid N+1 Queries

```php
<?php
// ไม่ดี: N+1 queries
$products = $storage->loadMultiple($ids);
foreach ($products as $product) {
    $user = \Drupal::entityTypeManager()
        ->getStorage('user')
        ->load($product->getOwnerId()); // Query ทุกครั้ง!
    echo $user->getDisplayName();
}

// ดี: Load ทุกอย่างในคราวเดียว
$products = $storage->loadMultiple($ids);
$user_ids = array_map(fn($p) => $p->getOwnerId(), $products);
$users = \Drupal::entityTypeManager()
    ->getStorage('user')
    ->loadMultiple(array_unique($user_ids)); // Query เดียว!

foreach ($products as $product) {
    $user = $users[$product->getOwnerId()] ?? NULL;
    if ($user) {
        echo $user->getDisplayName();
    }
}
```

### 12.3 EntityQuery vs Direct SQL

```php
<?php
// ใช้ EntityQuery สำหรับ entity-level operations (รองรับ access control)
$ids = \Drupal::entityQuery('product')
    ->condition('status', 1)
    ->accessCheck(TRUE)  // เสมอ!
    ->execute();

// ใช้ Direct SQL สำหรับ reports และ aggregations
$connection = \Drupal::database();
$stats = $connection->select('product', 'p')
    ->fields('p', ['type'])
    ->condition('p.status', 1)
    ->groupBy('p.type')
    ->addExpression('COUNT(p.id)', 'count')
    ->addExpression('AVG(p.price)', 'avg_price')
    ->execute()
    ->fetchAll();
```

---

## แบบทดสอบ (Quiz)

### คำถามที่ 1

**ความแตกต่างระหว่าง Content Entity และ Config Entity คืออะไร?**

**คำตอบ:** Content Entity เก็บข้อมูลใน database tables รองรับ Field API, Revision, Translation และมีข้อมูลที่แปรเปลี่ยนตามเวลา (เช่น Node, User, Comment) ส่วน Config Entity เก็บใน Configuration Management System (ไฟล์ YAML) ใช้สำหรับ configuration ที่ต้องการ deploy ระหว่าง environments (เช่น Vocabulary, View, Role) ไม่รองรับ Field API แต่ deploy ง่ายกว่า

---

### คำถามที่ 2

**annotation `@ContentEntityType` คืออะไร และมีส่วนประกอบที่สำคัญอะไรบ้าง?**

**คำตอบ:** `@ContentEntityType` คือ Drupal annotation ที่ใช้ระบุ metadata สำหรับ Content Entity class โดยบอก Drupal ว่า class นี้เป็น entity type อะไร ส่วนประกอบที่สำคัญ ได้แก่:
- `id` - รหัสเฉพาะของ entity type
- `label` - ชื่อที่แสดงต่อผู้ใช้
- `handlers` - กำหนด class handlers เช่น storage, form, access, list_builder
- `entity_keys` - mapping ระหว่าง entity key กับ field name
- `links` - URL paths สำหรับ entity operations
- `base_table` - ชื่อ database table หลัก
- `bundle_entity_type` - entity type ที่เป็น bundle

---

### คำถามที่ 3

**`hook_ENTITY_TYPE_presave()` ต่างจาก `hook_entity_presave()` อย่างไร?**

**คำตอบ:** `hook_entity_presave()` ถูกเรียกเมื่อ entity ใด ๆ ก็ตามถูก save (รวมทุก entity types ทั้ง node, user, comment ฯลฯ) ต้องตรวจสอบ entity type ด้วยตนเอง ส่วน `hook_ENTITY_TYPE_presave()` (เช่น `hook_product_presave()`) ถูกเรียกเฉพาะเมื่อ entity type ที่ระบุชื่อถูก save เท่านั้น มีประสิทธิภาพและชัดเจนกว่า ไม่ต้องตรวจสอบ entity type เพิ่มเติม ตัวแปรรับยังเป็น type-hinted ได้ถูกต้อง

---

### คำถามที่ 4

**เพราะอะไร `->accessCheck(TRUE)` ใน EntityQuery จึงสำคัญ?**

**คำตอบ:** `->accessCheck(TRUE)` บังคับให้ EntityQuery ตรวจสอบสิทธิ์การเข้าถึงของ current user ก่อนคืนผลลัพธ์ ซึ่งสำคัญเพราะ:
1. **Security**: ป้องกันการรั่วไหลของข้อมูลที่ user ไม่มีสิทธิ์เข้าถึง
2. **มาตรฐาน Drupal**: เป็น best practice อย่างเป็นทางการ
3. **Access Control Compliance**: ทำให้ระบบ access control ทำงานได้อย่างถูกต้อง
การใช้ `accessCheck(FALSE)` ควรใช้เฉพาะใน trusted contexts เช่น cron jobs, batch operations หรือการดำเนินการ administrative เท่านั้น ห้ามใช้ในโค้ดที่ user อาจ trigger ได้โดยตรง

---

### คำถามที่ 5

**อธิบาย Entity Cache System และ Cache Tags ใน Drupal พร้อมตัวอย่าง**

**คำตอบ:** Drupal's Entity Cache System ใช้ Cache Tags เพื่อ invalidate cache อย่างแม่นยำโดยไม่ต้องล้าง cache ทั้งหมด:

- **Cache Tags** คือ label ที่ระบุว่า cache entry นี้ขึ้นอยู่กับข้อมูลชนิดใด เช่น `['product:42']` หมายความว่า cache นี้ขึ้นอยู่กับ product ที่มี id = 42
- เมื่อ product 42 ถูก update/delete Drupal จะ invalidate cache entries ทุกตัวที่มี tag `product:42` โดยอัตโนมัติ
- `['node_list']` invalidate เมื่อ node ใด ๆ เปลี่ยนแปลง (ใช้สำหรับ lists)
- ตัวอย่างการใช้งาน:
```php
// การตั้ง cache tags ใน render array
$build['#cache']['tags'] = array_merge(
    $product->getCacheTags(),        // ['product:42']
    ['product_catalog_list']          // custom list tag
);
// Invalidate เมื่ออัพเดท
Cache::invalidateTags(['product:42']);
```
ระบบนี้ทำให้ Drupal cache ได้อย่างมีประสิทธิภาพและ fresh พร้อมกัน

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:

1. **Entity System Architecture** - โครงสร้างพื้นฐานของ Drupal Entity System ทั้ง Content และ Config Entities
2. **Custom Content Entity** - การสร้าง entity พร้อม annotations, field definitions, และ handlers ครบชุด
3. **Custom Config Entity** - การสร้าง configuration entity สำหรับ Discount Rules
4. **CRUD Operations** - Create, Read, Update, Delete ผ่าน Entity API
5. **Storage Handler** - การเขียน custom storage handler พร้อม caching
6. **Access Control** - การควบคุมสิทธิ์การเข้าถึงอย่างละเอียด
7. **Entity Forms** - การสร้าง form handlers ที่สมบูรณ์
8. **List Builders** - การแสดงรายการ entities ใน admin UI
9. **Hooks และ Events** - การ hook เข้ากับ entity lifecycle
10. **Field API** - การสร้าง custom field types, widgets, และ formatters
11. **Advanced Entity Query** - การค้นหา entities อย่างมีประสิทธิภาพ
12. **Best Practices** - Cache management และ performance optimization

### แหล่งข้อมูลเพิ่มเติม

- [Drupal Entity API Documentation](https://www.drupal.org/docs/drupal-apis/entity-api)
- [Drupal 10 API Reference](https://api.drupal.org/api/drupal/10)
- [Drupal Entity Examples Module](https://www.drupal.org/project/examples)
- [Drupal Core Entity System](https://git.drupalcode.org/project/drupal/-/tree/10.x/core/lib/Drupal/Core/Entity)

---

*Part 082 | ระดับสูง | Drupal Entity API*
