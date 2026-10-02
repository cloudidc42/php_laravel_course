# Part 079: Drupal Module Basics - Custom Module Development

**ระดับ:** สูง (Advanced)  
**เวลาเรียน:** 6-8 ชั่วโมง  
**Prerequisites:** Part 078 - Drupal Theming

---

## เป้าหมายของ Part นี้

1. เข้าใจโครงสร้างของ Custom Module ใน Drupal 10
2. สร้าง Routing System และ Controllers
3. ใช้งาน Forms API (FormBase, ConfigFormBase)
4. เขียน Hooks ที่สำคัญใน Drupal
5. จัดการ Permissions และ Access Control
6. Workshop: สร้าง Custom Module "Related Posts"

---

## 1. โครงสร้าง Custom Module

### 1.1 ที่อยู่ของ Module

Drupal จัดเก็บ Custom Modules ไว้ใน directory:

```
web/modules/custom/mymodule/
```

โครงสร้างไฟล์พื้นฐาน:

```
web/modules/custom/mymodule/
├── mymodule.info.yml          # ข้อมูล Module (จำเป็นต้องมี)
├── mymodule.module            # Hook implementations
├── mymodule.routing.yml       # URL Routes
├── mymodule.permissions.yml   # สิทธิ์การใช้งาน
├── mymodule.services.yml      # Dependency Injection Services
├── mymodule.libraries.yml     # CSS/JS Libraries
├── mymodule.links.menu.yml    # Menu Links
├── config/
│   └── install/
│       └── mymodule.settings.yml   # Default Config
└── src/
    ├── Controller/
    │   └── MyModuleController.php
    ├── Form/
    │   ├── MyModuleForm.php
    │   └── MyModuleSettingsForm.php
    └── Plugin/
        └── Block/
            └── MyModuleBlock.php
```

### 1.2 ไฟล์ mymodule.info.yml

ไฟล์นี้คือหัวใจของ Module บอก Drupal ว่านี่คือ Module อะไร:

```yaml
# web/modules/custom/mymodule/mymodule.info.yml

name: My Module
type: module
description: 'A custom module example for Drupal 10'
package: Custom
core_version_requirement: ^10
version: '1.0.0'
dependencies:
  - drupal:node
  - drupal:user
  - drupal:views
```

**อธิบาย field สำคัญ:**

| Field | ความหมาย |
|-------|----------|
| `name` | ชื่อที่แสดงใน Admin UI |
| `type` | ต้องเป็น `module` |
| `description` | คำอธิบายสั้นๆ |
| `core_version_requirement` | รองรับ Drupal version ใด |
| `dependencies` | Module ที่ต้องการให้ enable ก่อน |

### 1.3 ไฟล์ mymodule.module

ไฟล์นี้ใช้สำหรับ implement Hooks:

```php
<?php

/**
 * @file
 * Primary module hooks for My Module.
 */

use Drupal\Core\Entity\EntityInterface;
use Drupal\node\NodeInterface;
use Drupal\Core\Form\FormStateInterface;

/**
 * Implements hook_help().
 */
function mymodule_help($route_name, \Drupal\Core\Routing\RouteMatchInterface $route_match) {
  switch ($route_name) {
    case 'help.page.mymodule':
      $output = '';
      $output .= '<h3>' . t('About') . '</h3>';
      $output .= '<p>' . t('This module provides custom functionality.') . '</p>';
      return $output;
  }
}
```

---

## 2. Routing System

### 2.1 ไฟล์ mymodule.routing.yml

Routing กำหนด URL ที่ Module จัดการ:

```yaml
# web/modules/custom/mymodule/mymodule.routing.yml

mymodule.hello:
  path: '/hello'
  defaults:
    _controller: '\Drupal\mymodule\Controller\MyModuleController::hello'
    _title: 'Hello Page'
  requirements:
    _permission: 'access content'

mymodule.user_page:
  path: '/user/{uid}/profile'
  defaults:
    _controller: '\Drupal\mymodule\Controller\MyModuleController::userProfile'
    _title_callback: '\Drupal\mymodule\Controller\MyModuleController::userProfileTitle'
  requirements:
    _permission: 'access content'
    uid: '\d+'

mymodule.settings:
  path: '/admin/config/mymodule/settings'
  defaults:
    _form: '\Drupal\mymodule\Form\MyModuleSettingsForm'
    _title: 'My Module Settings'
  requirements:
    _permission: 'administer mymodule'
```

### 2.2 Controller

```php
<?php
// web/modules/custom/mymodule/src/Controller/MyModuleController.php

namespace Drupal\mymodule\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\Core\Session\AccountInterface;
use Drupal\user\UserInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

/**
 * Controller for My Module routes.
 */
class MyModuleController extends ControllerBase {

  /**
   * The current user.
   *
   * @var \Drupal\Core\Session\AccountInterface
   */
  protected AccountInterface $currentUser;

  /**
   * Constructor.
   */
  public function __construct(AccountInterface $current_user) {
    $this->currentUser = $current_user;
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('current_user')
    );
  }

  /**
   * Hello page.
   */
  public function hello(): array {
    return [
      '#markup' => $this->t('Hello, @name!', [
        '@name' => $this->currentUser->getDisplayName(),
      ]),
    ];
  }

  /**
   * User profile page.
   */
  public function userProfile(int $uid): array {
    $user = $this->entityTypeManager()
      ->getStorage('user')
      ->load($uid);

    if (!$user instanceof UserInterface) {
      throw new NotFoundHttpException();
    }

    return [
      '#theme' => 'mymodule_user_profile',
      '#user' => $user,
      '#cache' => [
        'tags' => ['user:' . $uid],
      ],
    ];
  }

  /**
   * Title callback for user profile.
   */
  public function userProfileTitle(int $uid): string {
    $user = $this->entityTypeManager()
      ->getStorage('user')
      ->load($uid);

    if ($user instanceof UserInterface) {
      return $this->t('@name Profile', ['@name' => $user->getDisplayName()]);
    }

    return $this->t('User Profile');
  }

  /**
   * JSON API endpoint.
   */
  public function apiData(int $nid): JsonResponse {
    $node = $this->entityTypeManager()
      ->getStorage('node')
      ->load($nid);

    if (!$node) {
      return new JsonResponse(['error' => 'Not found'], 404);
    }

    return new JsonResponse([
      'id'    => $node->id(),
      'title' => $node->label(),
      'type'  => $node->bundle(),
    ]);
  }

}
```

### 2.3 Route Parameters และ Upcasting

Drupal สามารถ upcast parameter อัตโนมัติได้:

```yaml
# mymodule.routing.yml
mymodule.node_view:
  path: '/mymodule/node/{node}'
  defaults:
    _controller: '\Drupal\mymodule\Controller\MyModuleController::nodeView'
    _title: 'Node View'
  requirements:
    _permission: 'access content'
    node: \d+
  options:
    parameters:
      node:
        type: entity:node
```

```php
// Controller ได้รับ NodeInterface โดยตรง
public function nodeView(\Drupal\node\NodeInterface $node): array {
  return [
    '#markup' => $this->t('Node: @title', ['@title' => $node->label()]),
  ];
}
```

---

## 3. Forms API

### 3.1 FormBase - Form ทั่วไป

```php
<?php
// web/modules/custom/mymodule/src/Form/ContactForm.php

namespace Drupal\mymodule\Form;

use Drupal\Core\Form\FormBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Mail\MailManagerInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Provides a contact form.
 */
class ContactForm extends FormBase {

  /**
   * The mail manager.
   */
  protected MailManagerInterface $mailManager;

  /**
   * Constructor.
   */
  public function __construct(MailManagerInterface $mail_manager) {
    $this->mailManager = $mail_manager;
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('plugin.manager.mail')
    );
  }

  /**
   * {@inheritdoc}
   */
  public function getFormId(): string {
    return 'mymodule_contact_form';
  }

  /**
   * {@inheritdoc}
   */
  public function buildForm(array $form, FormStateInterface $form_state): array {

    $form['name'] = [
      '#type'        => 'textfield',
      '#title'       => $this->t('Your Name'),
      '#required'    => TRUE,
      '#maxlength'   => 100,
      '#placeholder' => $this->t('Enter your name'),
    ];

    $form['email'] = [
      '#type'     => 'email',
      '#title'    => $this->t('Email Address'),
      '#required' => TRUE,
    ];

    $form['subject'] = [
      '#type'     => 'textfield',
      '#title'    => $this->t('Subject'),
      '#required' => TRUE,
    ];

    $form['message'] = [
      '#type'     => 'textarea',
      '#title'    => $this->t('Message'),
      '#required' => TRUE,
      '#rows'     => 6,
    ];

    $form['category'] = [
      '#type'    => 'select',
      '#title'   => $this->t('Category'),
      '#options' => [
        'general'  => $this->t('General Inquiry'),
        'support'  => $this->t('Technical Support'),
        'billing'  => $this->t('Billing'),
        'feedback' => $this->t('Feedback'),
      ],
      '#default_value' => 'general',
    ];

    $form['newsletter'] = [
      '#type'  => 'checkbox',
      '#title' => $this->t('Subscribe to newsletter'),
    ];

    $form['actions'] = [
      '#type' => 'actions',
    ];

    $form['actions']['submit'] = [
      '#type'  => 'submit',
      '#value' => $this->t('Send Message'),
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function validateForm(array &$form, FormStateInterface $form_state): void {
    $name = $form_state->getValue('name');
    if (strlen($name) < 2) {
      $form_state->setErrorByName('name', $this->t('Name must be at least 2 characters.'));
    }

    $email = $form_state->getValue('email');
    if (!\Drupal::service('email.validator')->isValid($email)) {
      $form_state->setErrorByName('email', $this->t('Please enter a valid email address.'));
    }
  }

  /**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $values = $form_state->getValues();

    // บันทึก log
    $this->logger('mymodule')->info('Contact form submitted by @name', [
      '@name' => $values['name'],
    ]);

    // ส่งอีเมล
    $params = [
      'subject' => $values['subject'],
      'message' => $values['message'],
      'name'    => $values['name'],
    ];

    $this->mailManager->mail(
      'mymodule',
      'contact',
      $values['email'],
      $this->languageManager()->getDefaultLanguage()->getId(),
      $params
    );

    // แสดง success message
    $this->messenger()->addStatus($this->t('Your message has been sent successfully!'));

    // Redirect ไปหน้าอื่น
    $form_state->setRedirect('<front>');
  }

}
```

### 3.2 ConfigFormBase - Form สำหรับ Settings

```php
<?php
// web/modules/custom/mymodule/src/Form/MyModuleSettingsForm.php

namespace Drupal\mymodule\Form;

use Drupal\Core\Form\ConfigFormBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Settings form for My Module.
 */
class MyModuleSettingsForm extends ConfigFormBase {

  /**
   * Config settings name.
   */
  const SETTINGS = 'mymodule.settings';

  /**
   * {@inheritdoc}
   */
  public function getFormId(): string {
    return 'mymodule_settings_form';
  }

  /**
   * {@inheritdoc}
   */
  protected function getEditableConfigNames(): array {
    return [static::SETTINGS];
  }

  /**
   * {@inheritdoc}
   */
  public function buildForm(array $form, FormStateInterface $form_state): array {
    $config = $this->config(static::SETTINGS);

    $form['general'] = [
      '#type'  => 'details',
      '#title' => $this->t('General Settings'),
      '#open'  => TRUE,
    ];

    $form['general']['items_per_page'] = [
      '#type'          => 'number',
      '#title'         => $this->t('Items Per Page'),
      '#description'   => $this->t('Number of items to show per page.'),
      '#default_value' => $config->get('items_per_page') ?? 10,
      '#min'           => 1,
      '#max'           => 100,
    ];

    $form['general']['cache_lifetime'] = [
      '#type'          => 'select',
      '#title'         => $this->t('Cache Lifetime'),
      '#options'       => [
        0     => $this->t('No cache'),
        3600  => $this->t('1 Hour'),
        86400 => $this->t('1 Day'),
        604800 => $this->t('1 Week'),
      ],
      '#default_value' => $config->get('cache_lifetime') ?? 3600,
    ];

    $form['general']['enable_feature'] = [
      '#type'          => 'checkbox',
      '#title'         => $this->t('Enable Advanced Feature'),
      '#default_value' => $config->get('enable_feature') ?? FALSE,
    ];

    $form['advanced'] = [
      '#type'  => 'details',
      '#title' => $this->t('Advanced Settings'),
      '#open'  => FALSE,
    ];

    $form['advanced']['api_endpoint'] = [
      '#type'          => 'url',
      '#title'         => $this->t('API Endpoint'),
      '#description'   => $this->t('External API endpoint URL.'),
      '#default_value' => $config->get('api_endpoint') ?? '',
    ];

    $form['advanced']['api_key'] = [
      '#type'          => 'textfield',
      '#title'         => $this->t('API Key'),
      '#default_value' => $config->get('api_key') ?? '',
      '#attributes'    => ['autocomplete' => 'off'],
    ];

    $form['advanced']['content_types'] = [
      '#type'          => 'checkboxes',
      '#title'         => $this->t('Enabled Content Types'),
      '#options'       => $this->getNodeTypeOptions(),
      '#default_value' => $config->get('content_types') ?? [],
    ];

    return parent::buildForm($form, $form_state);
  }

  /**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $this->config(static::SETTINGS)
      ->set('items_per_page', $form_state->getValue('items_per_page'))
      ->set('cache_lifetime', $form_state->getValue('cache_lifetime'))
      ->set('enable_feature', $form_state->getValue('enable_feature'))
      ->set('api_endpoint', $form_state->getValue('api_endpoint'))
      ->set('api_key', $form_state->getValue('api_key'))
      ->set('content_types', array_filter($form_state->getValue('content_types')))
      ->save();

    parent::submitForm($form, $form_state);
  }

  /**
   * Get node type options.
   */
  private function getNodeTypeOptions(): array {
    $options = [];
    $types = \Drupal::entityTypeManager()
      ->getStorage('node_type')
      ->loadMultiple();

    foreach ($types as $type) {
      $options[$type->id()] = $type->label();
    }

    return $options;
  }

}
```

### 3.3 Default Config File

```yaml
# web/modules/custom/mymodule/config/install/mymodule.settings.yml

items_per_page: 10
cache_lifetime: 3600
enable_feature: false
api_endpoint: ''
api_key: ''
content_types: []
```

### 3.4 Form Elements ที่ใช้บ่อย

```php
// ประเภท Form Elements ใน Drupal Forms API

// Text input
$form['textfield'] = [
  '#type'     => 'textfield',
  '#title'    => $this->t('Text'),
  '#required' => TRUE,
];

// Textarea
$form['textarea'] = [
  '#type' => 'textarea',
  '#rows' => 5,
  '#cols' => 60,
];

// Select dropdown
$form['select'] = [
  '#type'    => 'select',
  '#options' => ['key' => 'Label'],
  '#empty_option' => $this->t('- Select -'),
];

// Checkboxes (multiple)
$form['checkboxes'] = [
  '#type'    => 'checkboxes',
  '#options' => ['a' => 'A', 'b' => 'B'],
];

// Radios
$form['radios'] = [
  '#type'    => 'radios',
  '#options' => ['yes' => 'Yes', 'no' => 'No'],
];

// Date
$form['date'] = [
  '#type' => 'date',
];

// Number
$form['number'] = [
  '#type' => 'number',
  '#min'  => 0,
  '#max'  => 100,
  '#step' => 1,
];

// Entity autocomplete
$form['node'] = [
  '#type'            => 'entity_autocomplete',
  '#target_type'     => 'node',
  '#selection_settings' => [
    'target_bundles' => ['article'],
  ],
];

// File upload
$form['file'] = [
  '#type'              => 'managed_file',
  '#upload_location'   => 'public://uploads/',
  '#upload_validators' => [
    'file_validate_extensions' => ['jpg jpeg png gif'],
    'file_validate_size'       => [1024 * 1024 * 5],
  ],
];

// AJAX
$form['ajax_field'] = [
  '#type'  => 'select',
  '#ajax'  => [
    'callback' => '::ajaxCallback',
    'wrapper'  => 'ajax-wrapper',
    'event'    => 'change',
  ],
];

$form['ajax_result'] = [
  '#type'       => 'container',
  '#attributes' => ['id' => 'ajax-wrapper'],
];
```

---

## 4. Hook System

Hooks คือ "จุดต่อ" ที่ Drupal เปิดให้ Module อื่นเข้ามาแก้ไขพฤติกรรม

### 4.1 hook_node_presave

Hook นี้ถูกเรียกก่อนที่ Node จะถูก save:

```php
<?php
// web/modules/custom/mymodule/mymodule.module

use Drupal\node\NodeInterface;

/**
 * Implements hook_node_presave().
 */
function mymodule_node_presave(NodeInterface $node): void {
  // เรียกเฉพาะ article content type
  if ($node->bundle() !== 'article') {
    return;
  }

  // Auto-generate summary จาก body
  if ($node->hasField('body') && $node->hasField('field_summary')) {
    $body = $node->get('body')->value;
    $summary = $node->get('field_summary')->value;

    if (empty($summary) && !empty($body)) {
      // ตัดคำมา 200 ตัวอักษร
      $plain_text = strip_tags($body);
      $auto_summary = mb_substr($plain_text, 0, 200);
      if (mb_strlen($plain_text) > 200) {
        $auto_summary .= '...';
      }
      $node->set('field_summary', $auto_summary);
    }
  }

  // บันทึกเวลา publish ครั้งแรก
  if ($node->isNew() && $node->isPublished()) {
    if ($node->hasField('field_first_published')) {
      $node->set('field_first_published', \Drupal::time()->getRequestTime());
    }
  }

  // เพิ่ม tag อัตโนมัติตาม category
  if ($node->hasField('field_category') && $node->hasField('field_tags')) {
    $category = $node->get('field_category')->value;
    $auto_tags = mymodule_get_auto_tags($category);

    if (!empty($auto_tags)) {
      $existing_tags = array_column($node->get('field_tags')->getValue(), 'target_id');
      $new_tags = array_unique(array_merge($existing_tags, $auto_tags));

      $tag_values = array_map(fn($tid) => ['target_id' => $tid], $new_tags);
      $node->set('field_tags', $tag_values);
    }
  }
}

/**
 * Helper function to get auto tags.
 */
function mymodule_get_auto_tags(string $category): array {
  $mapping = [
    'technology' => [1, 5, 12],
    'business'   => [2, 6, 13],
    'lifestyle'  => [3, 7, 14],
  ];
  return $mapping[$category] ?? [];
}
```

### 4.2 hook_form_alter

Hook นี้ใช้แก้ไข Form ที่มีอยู่แล้วใน Drupal:

```php
/**
 * Implements hook_form_alter().
 *
 * แก้ไข form ทุก form ใน Drupal
 */
function mymodule_form_alter(array &$form, FormStateInterface $form_state, string $form_id): void {
  // เพิ่ม class ให้ submit button ทุก form
  if (isset($form['actions']['submit'])) {
    $form['actions']['submit']['#attributes']['class'][] = 'btn-primary';
  }
}

/**
 * Implements hook_form_FORM_ID_alter().
 *
 * แก้ไขเฉพาะ form ที่กำหนด (node article form)
 */
function mymodule_form_node_article_form_alter(array &$form, FormStateInterface $form_state, string $form_id): void {
  // เพิ่ม validation
  $form['#validate'][] = 'mymodule_article_form_validate';

  // ซ่อน field บางตัวสำหรับ non-admin
  $current_user = \Drupal::currentUser();
  if (!$current_user->hasPermission('administer nodes')) {
    if (isset($form['revision_log'])) {
      $form['revision_log']['#access'] = FALSE;
    }
    if (isset($form['uid'])) {
      $form['uid']['#access'] = FALSE;
    }
  }

  // เพิ่ม field group
  $form['seo_settings'] = [
    '#type'   => 'details',
    '#title'  => t('SEO Settings'),
    '#group'  => 'advanced',
    '#weight' => 100,
  ];

  $form['seo_settings']['field_meta_description'] = [
    '#type'       => 'textarea',
    '#title'      => t('Meta Description'),
    '#rows'       => 3,
    '#maxlength'  => 160,
    '#description' => t('Meta description for SEO (max 160 characters).'),
  ];

  // เพิ่ม custom submit handler
  $form['actions']['submit']['#submit'][] = 'mymodule_article_form_submit';
}

/**
 * Custom validation for article form.
 */
function mymodule_article_form_validate(array &$form, FormStateInterface $form_state): void {
  $title = $form_state->getValue('title');
  if ($title && strlen($title[0]['value']) < 10) {
    $form_state->setErrorByName('title', t('Title must be at least 10 characters.'));
  }
}

/**
 * Custom submit handler for article form.
 */
function mymodule_article_form_submit(array &$form, FormStateInterface $form_state): void {
  \Drupal::logger('mymodule')->info('Article saved by user @uid', [
    '@uid' => \Drupal::currentUser()->id(),
  ]);
}
```

### 4.3 hook_menu_links_discovered_alter

```php
/**
 * Implements hook_menu_links_discovered_alter().
 *
 * แก้ไข Menu Links ที่ถูกค้นพบโดย Drupal
 */
function mymodule_menu_links_discovered_alter(array &$links): void {
  // เปลี่ยน title ของ menu link
  if (isset($links['user.page'])) {
    $links['user.page']['title'] = t('My Account');
  }

  // ซ่อน link บางตัว
  if (isset($links['user.logout'])) {
    $links['user.logout']['route_name'] = 'mymodule.custom_logout';
  }

  // เพิ่ม custom link
  $links['mymodule.dashboard'] = [
    'title'      => t('Dashboard'),
    'route_name' => 'mymodule.dashboard',
    'menu_name'  => 'main',
    'parent'     => '',
    'weight'     => -50,
  ];
}
```

### 4.4 Hooks อื่นที่ใช้บ่อย

```php
/**
 * Implements hook_page_attachments().
 * เพิ่ม CSS/JS ทุกหน้า
 */
function mymodule_page_attachments(array &$attachments): void {
  $attachments['#attached']['library'][] = 'mymodule/global';
}

/**
 * Implements hook_node_view().
 * แก้ไข render array ของ node ก่อนแสดงผล
 */
function mymodule_node_view(array &$build, \Drupal\Core\Entity\EntityInterface $entity, \Drupal\Core\Entity\Display\EntityViewDisplayInterface $display, string $view_mode): void {
  if ($entity->bundle() === 'article' && $view_mode === 'full') {
    $build['related_posts'] = [
      '#theme'   => 'mymodule_related_posts',
      '#node_id' => $entity->id(),
      '#weight'  => 100,
    ];
  }
}

/**
 * Implements hook_theme().
 * ลงทะเบียน Theme templates
 */
function mymodule_theme(array $existing, string $type, string $theme, string $path): array {
  return [
    'mymodule_related_posts' => [
      'variables' => [
        'node_id'   => NULL,
        'nodes'     => [],
        'title'     => NULL,
      ],
      'template'  => 'mymodule-related-posts',
    ],
  ];
}

/**
 * Implements hook_ENTITY_TYPE_delete().
 * ทำงานเมื่อ node ถูกลบ
 */
function mymodule_node_delete(\Drupal\node\NodeInterface $node): void {
  // ลบ related data
  \Drupal::database()
    ->delete('mymodule_related')
    ->condition('nid', $node->id())
    ->execute();
}

/**
 * Implements hook_cron().
 * รันทุกครั้งที่ cron ทำงาน
 */
function mymodule_cron(): void {
  $last_run = \Drupal::state()->get('mymodule.last_cron', 0);
  $interval = 3600; // 1 hour

  if (\Drupal::time()->getRequestTime() - $last_run >= $interval) {
    // ทำงานที่ต้องการ
    mymodule_process_queue();
    \Drupal::state()->set('mymodule.last_cron', \Drupal::time()->getRequestTime());
  }
}
```

---

## 5. Permissions & Access Control

### 5.1 ไฟล์ mymodule.permissions.yml

```yaml
# web/modules/custom/mymodule/mymodule.permissions.yml

administer mymodule:
  title: 'Administer My Module'
  description: 'Full administration access to My Module.'
  restrict access: true

view mymodule content:
  title: 'View My Module Content'
  description: 'Access to view module content.'

create mymodule content:
  title: 'Create My Module Content'
  description: 'Can create new module content.'

edit own mymodule content:
  title: 'Edit Own Content'
  description: 'Can edit own module content only.'

delete any mymodule content:
  title: 'Delete Any Content'
  description: 'Can delete any module content.'
  restrict access: true
```

### 5.2 Access Control ใน Routing

```yaml
# แบบต่างๆ ของ access control ใน routing

# ตรวจสอบ permission
mymodule.page1:
  path: '/mymodule/page1'
  defaults:
    _controller: '...'
  requirements:
    _permission: 'view mymodule content'

# ต้อง login
mymodule.page2:
  path: '/mymodule/page2'
  defaults:
    _controller: '...'
  requirements:
    _user_is_logged_in: 'TRUE'

# ต้องเป็น admin
mymodule.page3:
  path: '/admin/mymodule'
  defaults:
    _controller: '...'
  requirements:
    _role: 'administrator'

# Custom access check
mymodule.page4:
  path: '/mymodule/page4'
  defaults:
    _controller: '...'
  requirements:
    _custom_access: '\Drupal\mymodule\Access\MyModuleAccess::checkAccess'
```

### 5.3 Custom Access Check

```php
<?php
// web/modules/custom/mymodule/src/Access/MyModuleAccess.php

namespace Drupal\mymodule\Access;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Routing\Access\AccessInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\node\NodeInterface;

/**
 * Custom access check for My Module.
 */
class MyModuleAccess implements AccessInterface {

  /**
   * Check access.
   */
  public function checkAccess(AccountInterface $account): AccessResultInterface {
    // ตรวจสอบเงื่อนไขหลายอย่าง
    if ($account->hasPermission('administer mymodule')) {
      return AccessResult::allowed()->cachePerPermissions();
    }

    // ตรวจสอบเงื่อนไขเพิ่มเติม
    $is_business_hours = $this->isBusinessHours();
    if ($account->hasPermission('view mymodule content') && $is_business_hours) {
      return AccessResult::allowed()
        ->cachePerPermissions()
        ->setCacheMaxAge(3600);
    }

    return AccessResult::forbidden()->cachePerPermissions();
  }

  /**
   * Check if it is business hours (8AM - 6PM).
   */
  private function isBusinessHours(): bool {
    $hour = (int) date('G');
    return $hour >= 8 && $hour < 18;
  }

  /**
   * Check node-specific access.
   */
  public function checkNodeAccess(NodeInterface $node, AccountInterface $account): AccessResultInterface {
    // เจ้าของสามารถเข้าถึงได้เสมอ
    if ($node->getOwnerId() === (int) $account->id()) {
      return AccessResult::allowed()
        ->cachePerUser()
        ->addCacheableDependency($node);
    }

    return AccessResult::neutral();
  }

}
```

### 5.4 เพิ่ม Link Menu

```yaml
# web/modules/custom/mymodule/mymodule.links.menu.yml

mymodule.settings:
  title: 'My Module Settings'
  description: 'Configure My Module settings.'
  route_name: mymodule.settings
  parent: system.admin_config_system
  weight: 100
```

---

## 6. Workshop: Custom Module "Related Posts"

### เป้าหมาย
สร้าง Module ที่แสดงบทความที่เกี่ยวข้องในหน้า node โดยอิงจาก tags ที่เหมือนกัน

### 6.1 โครงสร้างไฟล์

```
web/modules/custom/related_posts/
├── related_posts.info.yml
├── related_posts.module
├── related_posts.routing.yml
├── related_posts.permissions.yml
├── config/
│   └── install/
│       └── related_posts.settings.yml
├── src/
│   ├── Controller/
│   │   └── RelatedPostsController.php
│   └── Form/
│       └── RelatedPostsSettingsForm.php
└── templates/
    └── related-posts.html.twig
```

### 6.2 related_posts.info.yml

```yaml
name: Related Posts
type: module
description: 'Shows related posts based on shared taxonomy tags.'
package: Custom
core_version_requirement: ^10
version: '1.0.0'
dependencies:
  - drupal:node
  - drupal:taxonomy
```

### 6.3 related_posts.module

```php
<?php

/**
 * @file
 * Primary module hooks for Related Posts module.
 */

use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Entity\Display\EntityViewDisplayInterface;
use Drupal\node\NodeInterface;

/**
 * Implements hook_node_view().
 */
function related_posts_node_view(array &$build, EntityInterface $entity, EntityViewDisplayInterface $display, string $view_mode): void {
  // แสดงเฉพาะ full view mode
  if ($view_mode !== 'full') {
    return;
  }

  // ตรวจสอบว่าเป็น node ที่รองรับ
  if (!$entity instanceof NodeInterface) {
    return;
  }

  $config = \Drupal::config('related_posts.settings');
  $enabled_types = $config->get('content_types') ?? ['article'];

  if (!in_array($entity->bundle(), $enabled_types)) {
    return;
  }

  // ตรวจสอบ permission
  if (!\Drupal::currentUser()->hasPermission('view related posts')) {
    return;
  }

  // เพิ่ม Related Posts block
  $build['related_posts'] = [
    '#lazy_builder' => [
      'related_posts.lazy_builder:renderRelatedPosts',
      [$entity->id()],
    ],
    '#create_placeholder' => TRUE,
    '#weight' => 100,
  ];
}

/**
 * Implements hook_theme().
 */
function related_posts_theme(array $existing, string $type, string $theme, string $path): array {
  return [
    'related_posts' => [
      'variables' => [
        'title' => NULL,
        'nodes' => [],
        'count' => 0,
      ],
      'template'  => 'related-posts',
    ],
  ];
}
```

### 6.4 RelatedPostsController.php

```php
<?php

namespace Drupal\related_posts\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\Core\Database\Database;
use Drupal\node\NodeInterface;

/**
 * Controller for Related Posts.
 */
class RelatedPostsController extends ControllerBase {

  /**
   * Get related posts for a node.
   */
  public function getRelatedPosts(NodeInterface $node): array {
    $config = $this->config('related_posts.settings');
    $max_items = $config->get('max_items') ?? 5;

    // ดึง tags ของ node ปัจจุบัน
    $tags = $this->getNodeTags($node);

    if (empty($tags)) {
      return [];
    }

    // ค้นหา nodes ที่มี tags เหมือนกัน
    $related_nids = $this->findRelatedNodes($node->id(), $tags, $max_items);

    if (empty($related_nids)) {
      return [];
    }

    // โหลด nodes
    $nodes = $this->entityTypeManager()
      ->getStorage('node')
      ->loadMultiple($related_nids);

    return $nodes;
  }

  /**
   * Get taxonomy tags of a node.
   */
  private function getNodeTags(NodeInterface $node): array {
    $tags = [];

    // ตรวจสอบ field_tags
    if ($node->hasField('field_tags')) {
      foreach ($node->get('field_tags')->referencedEntities() as $term) {
        $tags[] = $term->id();
      }
    }

    return $tags;
  }

  /**
   * Find related nodes by tags.
   */
  private function findRelatedNodes(int $current_nid, array $tags, int $limit): array {
    // ใช้ Entity Query แทน raw SQL (แนะนำ)
    $query = $this->entityTypeManager()
      ->getStorage('node')
      ->getQuery();

    $related_nids = $query
      ->condition('status', 1)
      ->condition('type', 'article')
      ->condition('nid', $current_nid, '!=')
      ->condition('field_tags.target_id', $tags, 'IN')
      ->sort('created', 'DESC')
      ->range(0, $limit)
      ->accessCheck(TRUE)
      ->execute();

    return array_values($related_nids);
  }

  /**
   * Page callback to show related posts via AJAX.
   */
  public function ajaxRelatedPosts(NodeInterface $node): array {
    $nodes = $this->getRelatedPosts($node);

    return [
      '#theme' => 'related_posts',
      '#title' => $this->t('Related Posts'),
      '#nodes' => $nodes,
      '#count' => count($nodes),
      '#cache' => [
        'tags'    => ['node:' . $node->id(), 'node_list'],
        'contexts' => ['user.permissions'],
        'max-age'  => 3600,
      ],
    ];
  }

}
```

### 6.5 Twig Template

```twig
{# web/modules/custom/related_posts/templates/related-posts.html.twig #}

{% if nodes is not empty %}
  <section class="related-posts" aria-label="{{ 'Related Posts'|t }}">
    <h3 class="related-posts__title">
      {{ title|default('Related Posts'|t) }}
    </h3>

    <div class="related-posts__grid">
      {% for node in nodes %}
        <article class="related-posts__item">
          {% if node.field_image.entity %}
            <div class="related-posts__image">
              <a href="{{ path('entity.node.canonical', {'node': node.id}) }}">
                {{ node.field_image.entity|view('thumbnail') }}
              </a>
            </div>
          {% endif %}

          <div class="related-posts__content">
            <h4 class="related-posts__item-title">
              <a href="{{ path('entity.node.canonical', {'node': node.id}) }}">
                {{ node.label }}
              </a>
            </h4>

            {% if node.body.summary %}
              <p class="related-posts__summary">
                {{ node.body.summary|striptags|trim|slice(0, 120) }}...
              </p>
            {% endif %}

            <time class="related-posts__date"
                  datetime="{{ node.created.value|date('Y-m-d') }}">
              {{ node.created.value|date('d M Y') }}
            </time>
          </div>
        </article>
      {% endfor %}
    </div>
  </section>
{% endif %}
```

### 6.6 related_posts.permissions.yml

```yaml
# web/modules/custom/related_posts/related_posts.permissions.yml

view related posts:
  title: 'View Related Posts'
  description: 'Can see related posts on node pages.'

administer related posts:
  title: 'Administer Related Posts'
  description: 'Configure Related Posts settings.'
  restrict access: true
```

### 6.7 Settings Form

```php
<?php

namespace Drupal\related_posts\Form;

use Drupal\Core\Form\ConfigFormBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Settings for Related Posts.
 */
class RelatedPostsSettingsForm extends ConfigFormBase {

  const SETTINGS = 'related_posts.settings';

  public function getFormId(): string {
    return 'related_posts_settings_form';
  }

  protected function getEditableConfigNames(): array {
    return [static::SETTINGS];
  }

  public function buildForm(array $form, FormStateInterface $form_state): array {
    $config = $this->config(static::SETTINGS);

    $form['max_items'] = [
      '#type'          => 'number',
      '#title'         => $this->t('Maximum Related Posts'),
      '#default_value' => $config->get('max_items') ?? 5,
      '#min'           => 1,
      '#max'           => 20,
    ];

    $form['content_types'] = [
      '#type'          => 'checkboxes',
      '#title'         => $this->t('Enable for Content Types'),
      '#options'       => $this->getNodeTypeOptions(),
      '#default_value' => $config->get('content_types') ?? ['article'],
    ];

    return parent::buildForm($form, $form_state);
  }

  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $this->config(static::SETTINGS)
      ->set('max_items', $form_state->getValue('max_items'))
      ->set('content_types', array_filter($form_state->getValue('content_types')))
      ->save();

    parent::submitForm($form, $form_state);
  }

  private function getNodeTypeOptions(): array {
    $options = [];
    $types = \Drupal::entityTypeManager()->getStorage('node_type')->loadMultiple();
    foreach ($types as $type) {
      $options[$type->id()] = $type->label();
    }
    return $options;
  }

}
```

### 6.8 Enable Module

```bash
# Enable module ผ่าน Drush
drush en related_posts -y

# Clear cache
drush cr

# ตรวจสอบ
drush pml --type=module | grep related
```

---

## Quiz

**ข้อ 1:** ไฟล์ใดที่จำเป็นต้องมีเพื่อให้ Drupal รู้จัก Module?

A) mymodule.module  
B) mymodule.info.yml ✓  
C) mymodule.routing.yml  
D) mymodule.services.yml  

**เฉลย:** B - ไฟล์ `.info.yml` เป็นไฟล์เดียวที่จำเป็นต้องมี ไฟล์อื่นเป็น optional

---

**ข้อ 2:** Hook ใดที่ถูกเรียกก่อนที่ Node จะถูก save ลง database?

A) hook_node_insert  
B) hook_node_update  
C) hook_node_presave ✓  
D) hook_node_save  

**เฉลย:** C - `hook_node_presave` ถูกเรียกก่อน save ทั้ง insert และ update

---

**ข้อ 3:** Class ใดใช้สำหรับสร้าง Form ที่บันทึกค่าลงใน Config System?

A) FormBase  
B) FormInterface  
C) ConfigFormBase ✓  
D) SettingsForm  

**เฉลย:** C - `ConfigFormBase` extend มาจาก `FormBase` และมี method `getEditableConfigNames()` สำหรับจัดการ Config

---

**ข้อ 4:** วิธีที่ถูกต้องในการ inject dependency เข้า Controller ใน Drupal คืออะไร?

A) ใช้ `\Drupal::service()` ตรงๆ ใน method  
B) ผ่าน constructor และ `create()` static method ✓  
C) ใช้ global variable  
D) ใช้ `new ClassName()` ตรงๆ  

**เฉลย:** B - Drupal ใช้ Dependency Injection Container ผ่าน `create(ContainerInterface $container)` และ constructor injection

---

**ข้อ 5:** ข้อใดคือวิธีที่ถูกต้องในการตรวจสอบ permission ใน Controller?

A) `if ($user->isAdmin())`  
B) `if ($this->currentUser()->hasPermission('my permission'))` ✓  
C) `if ($user->role === 'admin')`  
D) `if (user_access('my permission'))`  

**เฉลย:** B - ใช้ `hasPermission()` ผ่าน `currentUser()` ที่ได้จาก `ControllerBase`

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **โครงสร้าง Module** - ไฟล์ที่จำเป็นและ optional
2. **Routing System** - กำหนด URL และเชื่อมกับ Controller
3. **Controllers** - จัดการ request และ return render array
4. **Forms API** - `FormBase` สำหรับ form ทั่วไป, `ConfigFormBase` สำหรับ settings
5. **Hook System** - จุดต่อที่ Drupal เปิดให้แก้ไขพฤติกรรม
6. **Permissions** - จัดการสิทธิ์การเข้าถึง

---

## ลิงก์ที่เกี่ยวข้อง

- [Drupal.org - Creating Custom Modules](https://www.drupal.org/docs/creating-custom-modules)
- [Drupal.org - Routing System](https://www.drupal.org/docs/drupal-apis/routing-system)
- [Drupal.org - Form API Reference](https://api.drupal.org/api/drupal/core!lib!Drupal!Core!Form!FormBase.php/class/FormBase/10)
- [Drupal.org - Hook System](https://api.drupal.org/api/drupal/core!core.api.php/group/hooks/10)

---

## ไปต่อ

➡️ **[Part 080: Drupal Services & Plugin System](part-080-drupal-services.md)**

ใน Part ถัดไปเราจะเรียนรู้เกี่ยวกับ Services, Dependency Injection อย่างละเอียด, Plugin System รวมถึง Block Plugins และ Field Formatter Plugins พร้อม Workshop สร้าง Custom Block และ Thai Currency Formatter
