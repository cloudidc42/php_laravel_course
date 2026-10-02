# Part 077: Drupal Content Management - Content Types, Fields, Taxonomy, Views

**ระดับ:** Intermediate  
**เวลาเรียน:** 5-6 ชั่วโมง  
**Prerequisites:** Part 076 - Drupal Installation

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. สร้างและจัดการ Content Types
2. เพิ่ม Fields ประเภทต่างๆ
3. สร้าง Taxonomy และ Vocabulary
4. สร้าง Views เพื่อแสดงผลข้อมูล
5. จัดการ Content ด้วย PHP API

---

## 1. Content Types

Content Types ใน Drupal คือ Blueprint สำหรับสร้าง Content โดย Default มี:
- **Article** - สำหรับบทความ (มี Tags, Image)
- **Basic Page** - สำหรับหน้าทั่วไป

### 1.1 สร้าง Content Type ผ่าน UI

```
Admin > Structure > Content types > Add content type
```

### 1.2 สร้าง Content Type ด้วย PHP

```php
<?php
/**
 * File: web/modules/custom/product_catalog/product_catalog.install
 * 
 * สร้าง Content Type ด้วยโค้ด
 */

use Drupal\node\Entity\NodeType;
use Drupal\field\Entity\FieldStorageConfig;
use Drupal\field\Entity\FieldConfig;

/**
 * Implements hook_install().
 */
function product_catalog_install() {
    // สร้าง Content Type "Product"
    $node_type = NodeType::create([
        'type'        => 'product',
        'name'        => 'Product',
        'description' => 'A product in the catalog.',
        'help'        => 'Fill in all required fields.',
        'new_revision' => TRUE,
        'display_submitted' => FALSE,
    ]);
    $node_type->save();

    // เพิ่ม Body Field
    node_add_body_field($node_type, 'Product Description');

    // สร้าง Field Storage: price
    FieldStorageConfig::create([
        'field_name'  => 'field_price',
        'entity_type' => 'node',
        'type'        => 'decimal',
        'settings'    => [
            'precision' => 10,
            'scale'     => 2,
        ],
    ])->save();

    // เพิ่ม Field ลงใน Content Type
    FieldConfig::create([
        'field_name'   => 'field_price',
        'entity_type'  => 'node',
        'bundle'       => 'product',
        'label'        => 'Price',
        'required'     => TRUE,
        'settings'     => [
            'min'    => 0,
            'prefix' => '฿',
        ],
    ])->save();

    // สร้าง Field Storage: product_image
    FieldStorageConfig::create([
        'field_name'  => 'field_product_image',
        'entity_type' => 'node',
        'type'        => 'image',
        'settings'    => [
            'uri_scheme'  => 'public',
            'target_type' => 'file',
        ],
        'cardinality' => -1, // Unlimited
    ])->save();

    FieldConfig::create([
        'field_name'  => 'field_product_image',
        'entity_type' => 'node',
        'bundle'      => 'product',
        'label'       => 'Product Images',
        'required'    => FALSE,
        'settings'    => [
            'file_extensions' => 'png gif jpg jpeg webp',
            'alt_field'       => TRUE,
            'title_field'     => FALSE,
            'max_resolution'  => '2000x2000',
        ],
    ])->save();

    // สร้าง Field Storage: sku
    FieldStorageConfig::create([
        'field_name'  => 'field_sku',
        'entity_type' => 'node',
        'type'        => 'string',
        'settings'    => ['max_length' => 64],
    ])->save();

    FieldConfig::create([
        'field_name'  => 'field_sku',
        'entity_type' => 'node',
        'bundle'      => 'product',
        'label'       => 'SKU',
        'required'    => TRUE,
    ])->save();

    // ตั้งค่า Display Form
    /** @var \Drupal\Core\Entity\Display\EntityFormDisplayInterface $form_display */
    $form_display = \Drupal::service('entity_display.repository')
        ->getFormDisplay('node', 'product', 'default');

    $form_display
        ->setComponent('field_sku', [
            'type'     => 'string_textfield',
            'weight'   => 0,
            'settings' => ['size' => 32],
        ])
        ->setComponent('field_price', [
            'type'     => 'number',
            'weight'   => 1,
        ])
        ->setComponent('field_product_image', [
            'type'     => 'image_image',
            'weight'   => 2,
        ])
        ->save();

    // ตั้งค่า View Display
    $view_display = \Drupal::service('entity_display.repository')
        ->getViewDisplay('node', 'product', 'default');

    $view_display
        ->setComponent('field_sku', [
            'label'  => 'above',
            'type'   => 'string',
            'weight' => 0,
        ])
        ->setComponent('field_price', [
            'label'  => 'above',
            'type'   => 'number_decimal',
            'weight' => 1,
            'settings' => [
                'prefix_suffix' => TRUE,
            ],
        ])
        ->setComponent('field_product_image', [
            'label'  => 'hidden',
            'type'   => 'image',
            'weight' => -1,
            'settings' => [
                'image_style' => 'large',
                'image_link'  => 'content',
            ],
        ])
        ->save();
}

/**
 * Implements hook_uninstall().
 */
function product_catalog_uninstall() {
    // ลบ Nodes ทั้งหมด
    $nids = \Drupal::entityQuery('node')
        ->condition('type', 'product')
        ->execute();

    $nodes = \Drupal\node\Entity\Node::loadMultiple($nids);
    foreach ($nodes as $node) {
        $node->delete();
    }

    // ลบ Content Type
    $node_type = \Drupal\node\Entity\NodeType::load('product');
    if ($node_type) {
        $node_type->delete();
    }
}
```

### 1.3 Field Types ที่มีใน Drupal

```php
<?php
/**
 * ประเภท Field ที่สำคัญใน Drupal
 */

// Text Fields
// - string           : Plain text (สั้น)
// - string_long      : Plain text (ยาว)
// - text             : Formatted text
// - text_long        : Formatted text (ยาว)
// - text_with_summary: Text + Summary

// Number Fields
// - integer   : จำนวนเต็ม
// - decimal   : ทศนิยม
// - float     : Float

// Reference Fields
// - entity_reference         : อ้างอิง Entity
// - entity_reference_revisions: อ้างอิง Entity พร้อม Revision

// Date/Time Fields
// - datetime    : วันที่และเวลา
// - daterange   : ช่วงเวลา
// - timestamp   : Unix Timestamp

// File Fields
// - file  : ไฟล์
// - image : รูปภาพ

// Boolean
// - boolean : True/False

// Address (ต้องติดตั้ง Module)
// - address : ที่อยู่แบบมาตรฐาน

// Link
// - link : URL และ Link Text

// Email
// - email : Email Address

// Telephone
// - telephone : เบอร์โทรศัพท์

// List Fields
// - list_string  : Dropdown (string values)
// - list_integer : Dropdown (integer values)
// - list_float   : Dropdown (float values)
```

---

## 2. Taxonomy

Taxonomy ใน Drupal ใช้สำหรับจัดหมวดหมู่ Content คล้ายกับ Categories/Tags ใน WordPress

### 2.1 Vocabulary และ Terms

```php
<?php
/**
 * File: web/modules/custom/product_catalog/product_catalog.install
 * 
 * สร้าง Taxonomy Vocabulary
 */

use Drupal\taxonomy\Entity\Vocabulary;
use Drupal\taxonomy\Entity\Term;

// สร้าง Vocabulary
function product_catalog_create_taxonomy() {

    // สร้าง Vocabulary "Product Category"
    $vocabulary = Vocabulary::create([
        'vid'         => 'product_category',
        'name'        => 'Product Category',
        'description' => 'Categories for products',
        'weight'      => 0,
    ]);
    $vocabulary->save();

    // สร้าง Taxonomy Terms
    $categories = [
        'Electronics' => ['Smartphones', 'Laptops', 'Tablets'],
        'Clothing'    => ["Men's", "Women's", 'Kids'],
        'Food'        => ['Fresh', 'Packaged', 'Beverages'],
    ];

    foreach ($categories as $parent_name => $children) {
        // สร้าง Parent Term
        $parent = Term::create([
            'name'   => $parent_name,
            'vid'    => 'product_category',
            'weight' => 0,
        ]);
        $parent->save();

        // สร้าง Child Terms
        foreach ($children as $child_name) {
            $child = Term::create([
                'name'   => $child_name,
                'vid'    => 'product_category',
                'parent' => [$parent->id()],
            ]);
            $child->save();
        }
    }

    // เพิ่ม Taxonomy Field ไปที่ Product
    FieldStorageConfig::create([
        'field_name'  => 'field_product_category',
        'entity_type' => 'node',
        'type'        => 'entity_reference',
        'settings'    => ['target_type' => 'taxonomy_term'],
        'cardinality' => -1,
    ])->save();

    FieldConfig::create([
        'field_name'   => 'field_product_category',
        'entity_type'  => 'node',
        'bundle'       => 'product',
        'label'        => 'Categories',
        'settings'     => [
            'handler'          => 'default:taxonomy_term',
            'handler_settings' => [
                'target_bundles'         => ['product_category' => 'product_category'],
                'sort'                   => ['field' => '_none'],
                'auto_create'            => FALSE,
                'auto_create_bundle'     => FALSE,
            ],
        ],
    ])->save();
}
```

### 2.2 Taxonomy API

```php
<?php
/**
 * Taxonomy Term Operations
 */

// โหลด Term
$term = \Drupal\taxonomy\Entity\Term::load(1);
echo $term->getName();
echo $term->id();
echo $term->getVocabularyId();

// โหลด Term จากชื่อ
$terms = \Drupal::entityTypeManager()
    ->getStorage('taxonomy_term')
    ->loadByProperties([
        'name' => 'Electronics',
        'vid'  => 'product_category',
    ]);

// Query Terms
$tids = \Drupal::entityQuery('taxonomy_term')
    ->condition('vid', 'product_category')
    ->condition('status', 1)
    ->sort('weight')
    ->execute();

$terms = \Drupal\taxonomy\Entity\Term::loadMultiple($tids);

// ดู Parent Terms
$parent_terms = \Drupal::entityTypeManager()
    ->getStorage('taxonomy_term')
    ->loadParents($term->id());

// ดู Children Terms
$children = \Drupal::entityTypeManager()
    ->getStorage('taxonomy_term')
    ->loadChildren($term->id());

// ดู Tree ทั้งหมด
$tree = \Drupal::entityTypeManager()
    ->getStorage('taxonomy_term')
    ->loadTree('product_category', 0, NULL, TRUE);

foreach ($tree as $term) {
    $depth = $term->depth;
    $indent = str_repeat('-', $depth);
    echo $indent . $term->getName() . "\n";
}
```

---

## 3. Views Module

Views เป็นหนึ่งใน Most Powerful Modules ของ Drupal ช่วยสร้างหน้าแสดงผลข้อมูล

### 3.1 สร้าง View ผ่าน UI

```
Admin > Structure > Views > Add view
```

### 3.2 สร้าง View ด้วยโค้ด (YAML Config)

```yaml
# config/sync/views.view.product_listing.yml

langcode: en
status: true
dependencies:
  config:
    - core.entity_view_mode.node.teaser
    - node.type.product
  module:
    - node
    - user
id: product_listing
label: 'Product Listing'
module: views
description: 'Display a list of products'
tag: default
base_table: node_field_data
base_field: nid
display:
  default:
    id: default
    display_title: Default
    display_plugin: default
    position: 0
    display_options:
      access:
        type: perm
        options:
          perm: 'access content'
      cache:
        type: tag
        options: {  }
      query:
        type: views_query
        options:
          disable_sql_rewrite: false
          distinct: false
          replica: false
      exposed_form:
        type: basic
        options:
          submit_button: Apply
          reset_button: false
          exposed_sorts_label: 'Sort by'
      pager:
        type: mini
        options:
          items_per_page: 12
          offset: 0
      style:
        type: default
      row:
        type: 'entity:node'
        options:
          relationship: none
          view_mode: teaser
      fields:
        title:
          id: title
          table: node_field_data
          field: title
          label: ''
          link_to_node: true
        field_price:
          id: field_price
          table: node__field_price
          field: field_price
          label: Price
          settings:
            prefix_suffix: true
        field_product_image:
          id: field_product_image
          table: node__field_product_image
          field: field_product_image
          label: ''
          settings:
            image_style: medium
      filters:
        status:
          id: status
          table: node_field_data
          field: status
          value: '1'
        type:
          id: type
          table: node_field_data
          field: type
          value:
            product: product
        field_product_category_target_id:
          id: field_product_category_target_id
          table: node__field_product_category
          field: field_product_category_target_id
          exposed: true
          expose:
            operator_id: ''
            label: Category
            identifier: category
      sorts:
        created:
          id: created
          table: node_field_data
          field: created
          order: DESC
  page_1:
    id: page_1
    display_title: Page
    display_plugin: page
    position: 1
    display_options:
      display_extenders: {  }
      path: products
      menu:
        type: normal
        title: Products
        weight: 0
  block_1:
    id: block_1
    display_title: 'Featured Products Block'
    display_plugin: block
    position: 2
    display_options:
      display_extenders: {  }
      pager:
        type: some
        options:
          items_per_page: 4
```

### 3.3 Views API ใน PHP

```php
<?php
/**
 * ใช้ Views API ใน PHP
 */

// Execute View Programmatically
$view = \Drupal\views\Views::getView('product_listing');
if ($view) {
    $view->setDisplay('default');
    $view->setCurrentPage(0);
    
    // Set Exposed Filters
    $view->setExposedInput([
        'category' => 5, // Term ID
    ]);
    
    $view->execute();
    
    // ดู Results
    $results = $view->result;
    foreach ($results as $row) {
        $node = $row->_entity;
        echo $node->getTitle() . "\n";
    }
    
    // Render View
    $rendered = $view->render();
    $output = \Drupal::service('renderer')->render($rendered);
}

// Embed View ใน Template
// ใน Twig: {{ drupal_view('product_listing', 'block_1') }}
```

---

## 4. Node API

```php
<?php
/**
 * Node CRUD Operations
 */

use Drupal\node\Entity\Node;

// สร้าง Node
function create_product(array $data): int {
    $node = Node::create([
        'type'                        => 'product',
        'title'                       => $data['title'],
        'body'                        => [
            'value'  => $data['description'],
            'format' => 'full_html',
        ],
        'field_sku'                   => $data['sku'],
        'field_price'                 => $data['price'],
        'field_product_category'      => $data['category_ids'], // Array of TIDs
        'status'                      => 1, // Published
        'uid'                         => \Drupal::currentUser()->id(),
        'langcode'                    => 'th',
    ]);
    $node->save();
    return $node->id();
}

// อ่าน Node
function get_product(int $nid): ?array {
    $node = Node::load($nid);
    if (!$node) {
        return NULL;
    }

    $categories = [];
    foreach ($node->get('field_product_category') as $item) {
        $term = $item->entity;
        $categories[] = [
            'id'   => $term->id(),
            'name' => $term->getName(),
        ];
    }

    return [
        'id'          => $node->id(),
        'title'       => $node->getTitle(),
        'description' => $node->get('body')->value,
        'sku'         => $node->get('field_sku')->value,
        'price'       => $node->get('field_price')->value,
        'categories'  => $categories,
        'created'     => $node->getCreatedTime(),
        'status'      => $node->isPublished(),
    ];
}

// อัปเดต Node
function update_product(int $nid, array $data): bool {
    $node = Node::load($nid);
    if (!$node) {
        return FALSE;
    }

    if (isset($data['title'])) {
        $node->setTitle($data['title']);
    }
    if (isset($data['price'])) {
        $node->set('field_price', $data['price']);
    }
    if (isset($data['status'])) {
        if ($data['status']) {
            $node->setPublished();
        } else {
            $node->setUnpublished();
        }
    }

    $node->setNewRevision(TRUE);
    $node->revision_log = 'Updated via API';
    $node->setRevisionUserId(\Drupal::currentUser()->id());
    $node->save();

    return TRUE;
}

// ลบ Node
function delete_product(int $nid): bool {
    $node = Node::load($nid);
    if (!$node) {
        return FALSE;
    }
    $node->delete();
    return TRUE;
}

// ค้นหา Nodes
function search_products(array $filters = [], int $page = 0, int $limit = 12): array {
    $query = \Drupal::entityQuery('node')
        ->condition('type', 'product')
        ->condition('status', 1)
        ->sort('created', 'DESC')
        ->range($page * $limit, $limit)
        ->accessCheck(TRUE);

    if (!empty($filters['category'])) {
        $query->condition('field_product_category', $filters['category']);
    }

    if (!empty($filters['min_price'])) {
        $query->condition('field_price', $filters['min_price'], '>=');
    }

    if (!empty($filters['max_price'])) {
        $query->condition('field_price', $filters['max_price'], '<=');
    }

    if (!empty($filters['search'])) {
        $query->condition('title', '%' . $filters['search'] . '%', 'LIKE');
    }

    $nids = $query->execute();
    $nodes = Node::loadMultiple($nids);

    $products = [];
    foreach ($nodes as $node) {
        $products[] = get_product($node->id());
    }

    return $products;
}
```

---

## 5. Entity Query

```php
<?php
/**
 * Advanced Entity Queries
 */

// OR Conditions
$query = \Drupal::entityQuery('node')
    ->condition('type', 'product')
    ->accessCheck(TRUE);

// OR Group
$or_group = $query->orConditionGroup()
    ->condition('field_price', 100, '<')
    ->condition('title', '%sale%', 'LIKE');

$query->condition($or_group);
$nids = $query->execute();

// Multiple Conditions
$query = \Drupal::entityQuery('node')
    ->condition('type', ['product', 'article'], 'IN')
    ->condition('status', 1)
    ->condition('created', strtotime('-30 days'), '>')
    ->sort('created', 'DESC')
    ->range(0, 10)
    ->accessCheck(TRUE);

// Taxonomy Reference
$query = \Drupal::entityQuery('node')
    ->condition('type', 'product')
    ->condition('field_product_category.entity.name', 'Electronics')
    ->accessCheck(TRUE);

// Count Query
$count = \Drupal::entityQuery('node')
    ->condition('type', 'product')
    ->condition('status', 1)
    ->accessCheck(TRUE)
    ->count()
    ->execute();

echo "Total products: $count";
```

---

## Workshop: สร้าง Product Catalog Module

### Exercise 1: Module ครบวงจร

```php
<?php
/**
 * File: web/modules/custom/product_catalog/product_catalog.module
 */

use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Entity\Display\EntityViewDisplayInterface;
use Drupal\node\NodeInterface;

/**
 * Implements hook_node_view().
 * เพิ่มข้อมูลพิเศษเมื่อดู Product
 */
function product_catalog_node_view(array &$build, EntityInterface $entity, EntityViewDisplayInterface $display, $view_mode) {
    if ($entity->bundle() !== 'product') {
        return;
    }

    if ($view_mode === 'full') {
        // เพิ่ม "Add to Cart" Button (ตัวอย่าง)
        $build['add_to_cart'] = [
            '#type'   => 'link',
            '#title'  => t('Add to Cart'),
            '#url'    => \Drupal\Core\Url::fromRoute('product_catalog.add_to_cart', [
                'node' => $entity->id(),
            ]),
            '#attributes' => [
                'class' => ['button', 'button--primary'],
            ],
        ];
    }
}

/**
 * Implements hook_node_presave().
 */
function product_catalog_node_presave(NodeInterface $node) {
    if ($node->bundle() !== 'product') {
        return;
    }

    // Auto-generate SKU ถ้าไม่ได้ใส่
    if (empty($node->get('field_sku')->value)) {
        $node->set('field_sku', 'SKU-' . strtoupper(uniqid()));
    }
}

/**
 * Implements hook_theme().
 */
function product_catalog_theme() {
    return [
        'product_card' => [
            'variables' => [
                'node'       => NULL,
                'image_url'  => NULL,
                'price'      => NULL,
                'categories' => [],
            ],
            'template' => 'product-card',
        ],
    ];
}
```

### Exercise 2: Views Integration

```php
<?php
/**
 * ใช้ Views ผ่าน PHP ใน Controller
 */

namespace Drupal\product_catalog\Controller;

use Drupal\Core\Controller\ControllerBase;

class ProductController extends ControllerBase {

    public function catalog(): array {
        // Embed View ใน Render Array
        $view = \Drupal\views\Views::getView('product_listing');
        if ($view) {
            $view->setDisplay('page_1');
            $view->preExecute();
            $view->execute();
            $view_render = $view->buildRenderable('page_1');
        }

        return [
            '#type'     => 'container',
            '#children' => [
                'heading' => [
                    '#markup' => '<h1>' . $this->t('Product Catalog') . '</h1>',
                ],
                'products' => $view_render ?? [],
            ],
        ];
    }
}
```

---

## Quiz

**คำถามที่ 1:** Vocabulary ใน Drupal Taxonomy คืออะไร?

A) ชื่อของ Content Type  
B) กลุ่มของ Taxonomy Terms (เช่น Categories, Tags)  
C) ชื่อของ Module  
D) ประเภทของ Field  

**เฉลย: B) Vocabulary คือกลุ่มของ Terms เช่น "Product Category" vocabulary มี Terms เช่น Electronics, Clothing**

---

**คำถามที่ 2:** `cardinality: -1` ใน Field Storage Config หมายความว่าอะไร?

A) Field ใส่ค่าได้ 1 ค่า  
B) Field ใส่ค่าได้สูงสุด -1 ค่า  
C) Field ใส่ค่าได้ไม่จำกัด  
D) Field ไม่รองรับหลายค่า  

**เฉลย: C) -1 หมายถึง Unlimited/Infinite values**

---

**คำถามที่ 3:** Views Module ใช้ทำอะไรใน Drupal?

A) จัดการ User Permissions  
B) สร้าง Content Types  
C) สร้างหน้าแสดงผล/List Content แบบ Dynamic  
D) จัดการ CSS/JS  

**เฉลย: C) Views ช่วยสร้าง Queries และแสดงผล Content แบบ Dynamic โดยไม่ต้องเขียนโค้ด**

---

**คำถามที่ 4:** คำสั่งใดใช้ดึง Taxonomy Tree ทั้งหมด?

A) `Term::loadAll()`  
B) `\Drupal::entityTypeManager()->getStorage('taxonomy_term')->loadTree($vid)`  
C) `Views::getView('taxonomy')`  
D) `taxonomy_get_tree($vid)`  

**เฉลย: B) `loadTree()` method ของ Storage ดึง Term Tree แบบ Flat Array พร้อม depth**

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- การสร้าง Content Types และ Fields ด้วยโค้ด
- Taxonomy: Vocabulary และ Terms
- Views Module สำหรับแสดงผล Content
- Node API สำหรับ CRUD Operations
- Entity Query สำหรับค้นหาข้อมูล

---

## ต่อไป

➡️ **[Part 078: Drupal Theming](part-078-drupal-theming.md)**

เรียนรู้เกี่ยวกับ:
- Twig Templates
- theme.info.yml
- Preprocess Hooks
- Libraries (CSS/JS)
