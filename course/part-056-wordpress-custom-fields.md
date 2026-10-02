# Part 056: WordPress Custom Fields Advanced

**ระดับ:** Advanced  
**เวลาเรียน:** 5-6 ชั่วโมง  
**Prerequisites:** Part 055 (WordPress Custom Post Types)

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
1. ใช้ post_meta API อย่างมืออาชีพ
2. สร้าง Repeater Fields เอง
3. สร้าง Field Groups คล้าย ACF
4. จัดการ Complex Data Structures
5. Validate และ Sanitize Custom Field Data

---

## 1. Post Meta API พื้นฐาน

```php
<?php
/**
 * WordPress Post Meta API
 */

// ==========================================
// add_post_meta vs update_post_meta
// ==========================================

$post_id = get_the_ID();

// add_post_meta: เพิ่มเสมอ (สร้าง Record ใหม่)
// ถ้าไม่ใช่ unique จะสร้างหลาย records ได้
add_post_meta( $post_id, 'my_key', 'value1' );
add_post_meta( $post_id, 'my_key', 'value2' ); // สร้าง record ที่ 2

// add_post_meta แบบ unique (ป้องกัน duplicate)
add_post_meta( $post_id, 'my_key', 'value', true ); // false ถ้ามีอยู่แล้ว

// update_post_meta: อัปเดต หรือสร้างใหม่ถ้าไม่มี
update_post_meta( $post_id, 'my_key', 'new_value' );

// update เฉพาะ value เก่าที่ระบุ (ป้องกัน race condition)
update_post_meta( $post_id, 'my_key', 'new_value', 'old_value' );

// ==========================================
// get_post_meta
// ==========================================

// ดึงค่าเดียว
$value = get_post_meta( $post_id, 'my_key', true );

// ดึงทุก values (array)
$values = get_post_meta( $post_id, 'my_key', false );

// ดึง Meta ทั้งหมดของ Post
$all_meta = get_post_meta( $post_id );

// ==========================================
// delete_post_meta
// ==========================================

// ลบทุก records ที่มี key นี้
delete_post_meta( $post_id, 'my_key' );

// ลบเฉพาะ record ที่มีค่าเฉพาะ
delete_post_meta( $post_id, 'my_key', 'specific_value' );

// ==========================================
// Meta Keys Convention
// ==========================================

// ใช้ underscore prefix (_) สำหรับ Private/Hidden Fields
// (จะไม่แสดงใน Custom Fields meta box ใน Admin)
update_post_meta( $post_id, '_book_author', 'ชื่อผู้แต่ง' );
update_post_meta( $post_id, '_featured', true );
update_post_meta( $post_id, '_price', 299.99 );

// Fields ที่ไม่มี _ prefix จะแสดงใน Custom Fields box
update_post_meta( $post_id, 'my_notes', 'หมายเหตุ' );
```

---

## 2. Advanced Meta Box System

```php
<?php
/**
 * Advanced Meta Box Field System
 * 
 * ระบบสร้าง Custom Fields แบบ Flexible
 */

class Meta_Box_Builder {
    
    private $boxes = array();
    
    /**
     * เพิ่ม Meta Box Definition
     */
    public function add_box( $id, $args ) {
        $defaults = array(
            'title'    => '',
            'post_type' => 'post',
            'context'  => 'normal',
            'priority' => 'default',
            'fields'   => array(),
        );
        
        $this->boxes[$id] = wp_parse_args( $args, $defaults );
        return $this;
    }
    
    /**
     * Register Meta Boxes กับ WordPress
     */
    public function register() {
        add_action( 'add_meta_boxes', array( $this, 'register_boxes' ) );
        add_action( 'save_post', array( $this, 'save_all_boxes' ), 10, 2 );
    }
    
    public function register_boxes() {
        foreach ( $this->boxes as $id => $box ) {
            $post_types = (array) $box['post_type'];
            foreach ( $post_types as $post_type ) {
                add_meta_box(
                    $id,
                    $box['title'],
                    array( $this, 'render_box' ),
                    $post_type,
                    $box['context'],
                    $box['priority'],
                    array( 'id' => $id, 'fields' => $box['fields'] )
                );
            }
        }
    }
    
    public function render_box( $post, $metabox ) {
        $id     = $metabox['args']['id'];
        $fields = $metabox['args']['fields'];
        
        wp_nonce_field( 'save_meta_box_' . $id, 'meta_box_nonce_' . $id );
        
        echo '<table class="form-table">';
        foreach ( $fields as $field ) {
            $this->render_field( $post, $field );
        }
        echo '</table>';
    }
    
    private function render_field( $post, $field ) {
        $value = get_post_meta( $post->ID, $field['key'], true );
        
        echo '<tr>';
        echo '<th><label for="' . esc_attr($field['key']) . '">' . esc_html($field['label']) . '</label></th>';
        echo '<td>';
        
        switch ( $field['type'] ) {
            
            case 'text':
                echo '<input type="text" 
                    id="' . esc_attr($field['key']) . '" 
                    name="' . esc_attr($field['key']) . '" 
                    value="' . esc_attr($value) . '" 
                    class="regular-text"
                    ' . ($field['required'] ?? false ? 'required' : '') . '>';
                break;
                
            case 'textarea':
                echo '<textarea 
                    id="' . esc_attr($field['key']) . '" 
                    name="' . esc_attr($field['key']) . '" 
                    rows="' . ($field['rows'] ?? 4) . '"
                    class="large-text">' . esc_textarea($value) . '</textarea>';
                break;
                
            case 'number':
                echo '<input type="number" 
                    id="' . esc_attr($field['key']) . '" 
                    name="' . esc_attr($field['key']) . '" 
                    value="' . esc_attr($value) . '"
                    min="' . ($field['min'] ?? '') . '"
                    max="' . ($field['max'] ?? '') . '"
                    step="' . ($field['step'] ?? 1) . '">';
                break;
                
            case 'select':
                echo '<select id="' . esc_attr($field['key']) . '" name="' . esc_attr($field['key']) . '">';
                if ( isset($field['placeholder']) ) {
                    echo '<option value="">' . esc_html($field['placeholder']) . '</option>';
                }
                foreach ( ($field['options'] ?? array()) as $opt_value => $opt_label ) {
                    echo '<option value="' . esc_attr($opt_value) . '" ' . selected($value, $opt_value, false) . '>';
                    echo esc_html($opt_label);
                    echo '</option>';
                }
                echo '</select>';
                break;
                
            case 'checkbox':
                echo '<label>';
                echo '<input type="checkbox" 
                    id="' . esc_attr($field['key']) . '" 
                    name="' . esc_attr($field['key']) . '" 
                    value="1" ' . checked($value, '1', false) . '>';
                echo ' ' . esc_html($field['description'] ?? '');
                echo '</label>';
                break;
                
            case 'date':
                echo '<input type="date" 
                    id="' . esc_attr($field['key']) . '" 
                    name="' . esc_attr($field['key']) . '" 
                    value="' . esc_attr($value) . '">';
                break;
                
            case 'color':
                echo '<input type="color" 
                    id="' . esc_attr($field['key']) . '" 
                    name="' . esc_attr($field['key']) . '" 
                    value="' . esc_attr($value ?: '#000000') . '">';
                break;
                
            case 'image':
                $image_url = $value ? wp_get_attachment_image_url($value, 'thumbnail') : '';
                ?>
                <div class="meta-image-field">
                    <input type="hidden" 
                           id="<?php echo esc_attr($field['key']); ?>" 
                           name="<?php echo esc_attr($field['key']); ?>" 
                           value="<?php echo esc_attr($value); ?>">
                    <div class="image-preview">
                        <?php if ($image_url) : ?>
                            <img src="<?php echo esc_url($image_url); ?>" style="max-width:150px">
                        <?php endif; ?>
                    </div>
                    <button type="button" class="button upload-image-btn" 
                            data-target="<?php echo esc_attr($field['key']); ?>">
                        เลือกรูปภาพ
                    </button>
                    <?php if ($value) : ?>
                        <button type="button" class="button remove-image-btn"
                                data-target="<?php echo esc_attr($field['key']); ?>">
                            ลบรูป
                        </button>
                    <?php endif; ?>
                </div>
                <?php
                break;
                
            case 'wysiwyg':
                wp_editor( $value, $field['key'], array(
                    'textarea_name' => $field['key'],
                    'textarea_rows' => $field['rows'] ?? 10,
                    'media_buttons' => $field['media_buttons'] ?? true,
                ) );
                break;
        }
        
        if ( isset($field['description']) && $field['type'] !== 'checkbox' ) {
            echo '<p class="description">' . esc_html($field['description']) . '</p>';
        }
        
        echo '</td></tr>';
    }
    
    public function save_all_boxes( $post_id, $post ) {
        
        if ( defined('DOING_AUTOSAVE') && DOING_AUTOSAVE ) return;
        if ( wp_is_post_revision($post_id) ) return;
        
        foreach ( $this->boxes as $id => $box ) {
            $nonce_key = 'meta_box_nonce_' . $id;
            $nonce_action = 'save_meta_box_' . $id;
            
            if ( ! isset($_POST[$nonce_key]) ||
                 ! wp_verify_nonce($_POST[$nonce_key], $nonce_action) ) {
                continue;
            }
            
            if ( ! current_user_can('edit_post', $post_id) ) {
                continue;
            }
            
            $post_types = (array) $box['post_type'];
            if ( ! in_array($post->post_type, $post_types) ) {
                continue;
            }
            
            foreach ( $box['fields'] as $field ) {
                $this->save_field( $post_id, $field );
            }
        }
    }
    
    private function save_field( $post_id, $field ) {
        $key = $field['key'];
        
        if ( $field['type'] === 'checkbox' ) {
            update_post_meta( $post_id, $key, isset($_POST[$key]) ? '1' : '0' );
            return;
        }
        
        if ( ! isset($_POST[$key]) ) {
            delete_post_meta( $post_id, $key );
            return;
        }
        
        $value = $_POST[$key];
        
        // Sanitize ตาม Type
        switch ($field['type']) {
            case 'text':
                $value = sanitize_text_field($value);
                break;
            case 'textarea':
                $value = sanitize_textarea_field($value);
                break;
            case 'wysiwyg':
                $value = wp_kses_post($value);
                break;
            case 'number':
                $value = is_numeric($value) ? (float) $value : 0;
                break;
            case 'date':
                $value = sanitize_text_field($value);
                // Validate date format
                $d = DateTime::createFromFormat('Y-m-d', $value);
                if (!$d || $d->format('Y-m-d') !== $value) {
                    $value = '';
                }
                break;
            case 'color':
                $value = sanitize_hex_color($value);
                break;
            case 'image':
                $value = absint($value);
                break;
            case 'select':
                $allowed = array_keys($field['options'] ?? array());
                $value = in_array($value, $allowed) ? $value : '';
                break;
            default:
                $value = sanitize_text_field($value);
        }
        
        if ( $value !== '' && $value !== null ) {
            update_post_meta( $post_id, $key, $value );
        } else {
            delete_post_meta( $post_id, $key );
        }
    }
}

// ==========================================
// การใช้งาน Meta Box Builder
// ==========================================

$meta_builder = new Meta_Box_Builder();

$meta_builder->add_box( 'book_details', array(
    'title'     => 'รายละเอียดหนังสือ',
    'post_type' => 'book',
    'fields'    => array(
        array(
            'key'      => '_book_author',
            'label'    => 'ผู้แต่ง',
            'type'     => 'text',
            'required' => true,
        ),
        array(
            'key'   => '_book_isbn',
            'label' => 'ISBN',
            'type'  => 'text',
        ),
        array(
            'key'   => '_book_pages',
            'label' => 'จำนวนหน้า',
            'type'  => 'number',
            'min'   => 1,
        ),
        array(
            'key'     => '_book_language',
            'label'   => 'ภาษา',
            'type'    => 'select',
            'options' => array(
                'thai'    => 'ไทย',
                'english' => 'English',
            ),
            'placeholder' => 'เลือกภาษา',
        ),
        array(
            'key'         => '_book_description',
            'label'       => 'เนื้อหาย่อ',
            'type'        => 'textarea',
            'rows'        => 5,
            'description' => 'สรุปเนื้อหาของหนังสือ',
        ),
    ),
) );

$meta_builder->register();
```

---

## 3. Repeater Fields

```php
<?php
/**
 * Repeater Fields
 * 
 * Fields ที่เพิ่ม/ลบได้ (เช่น Gallery, List of Items)
 */

class Repeater_Field {
    
    private $key;
    private $fields;
    
    public function __construct( $key, $fields ) {
        $this->key    = $key;
        $this->fields = $fields;
    }
    
    public function render( $post ) {
        $rows = get_post_meta( $post->ID, $this->key, true );
        if ( ! is_array($rows) ) $rows = array();
        
        ?>
        <div class="repeater-field" data-key="<?php echo esc_attr($this->key); ?>">
            
            <div class="repeater-rows">
                <?php if (empty($rows)) : ?>
                    <p class="no-rows">ยังไม่มีข้อมูล คลิก "เพิ่มแถว" เพื่อเริ่ม</p>
                <?php endif; ?>
                
                <?php foreach ($rows as $index => $row) : ?>
                    <?php $this->render_row($index, $row); ?>
                <?php endforeach; ?>
            </div>
            
            <button type="button" class="button add-repeater-row">+ เพิ่มแถว</button>
            
            <!-- Template Row (Hidden) -->
            <div class="repeater-template" style="display:none">
                <?php $this->render_row('__INDEX__', array()); ?>
            </div>
        </div>
        
        <script>
        jQuery(function($) {
            var key = '<?php echo esc_js($this->key); ?>';
            var index = <?php echo count($rows); ?>;
            
            // เพิ่มแถวใหม่
            $('.add-repeater-row').on('click', function() {
                var template = $('.repeater-template').html();
                template = template.replace(/__INDEX__/g, index);
                $('.repeater-rows').append(template);
                index++;
            });
            
            // ลบแถว
            $(document).on('click', '.remove-repeater-row', function() {
                if (confirm('ต้องการลบแถวนี้ใช่ไหม?')) {
                    $(this).closest('.repeater-row').remove();
                }
            });
            
            // ลาก-วาง เรียงลำดับ
            if (typeof $.fn.sortable !== 'undefined') {
                $('.repeater-rows').sortable({
                    handle: '.row-handle',
                    axis: 'y',
                });
            }
        });
        </script>
        <?php
    }
    
    private function render_row( $index, $row_data ) {
        ?>
        <div class="repeater-row" style="border:1px solid #ddd; padding:15px; margin-bottom:10px; background:#f9f9f9">
            <span class="row-handle" style="cursor:move; margin-right:10px">⠿</span>
            <button type="button" class="button remove-repeater-row" style="float:right">ลบ</button>
            
            <table class="form-table" style="clear:both">
                <?php foreach ($this->fields as $field) :
                    $field_key = $this->key . '[' . $index . '][' . $field['key'] . ']';
                    $value = $row_data[$field['key']] ?? '';
                    ?>
                    <tr>
                        <th><?php echo esc_html($field['label']); ?></th>
                        <td>
                            <?php if ($field['type'] === 'text') : ?>
                                <input type="text" 
                                       name="<?php echo esc_attr($field_key); ?>" 
                                       value="<?php echo esc_attr($value); ?>"
                                       class="regular-text">
                                       
                            <?php elseif ($field['type'] === 'textarea') : ?>
                                <textarea name="<?php echo esc_attr($field_key); ?>" 
                                          rows="3"
                                          class="large-text"><?php echo esc_textarea($value); ?></textarea>
                                          
                            <?php elseif ($field['type'] === 'number') : ?>
                                <input type="number" 
                                       name="<?php echo esc_attr($field_key); ?>" 
                                       value="<?php echo esc_attr($value); ?>"
                                       min="<?php echo esc_attr($field['min'] ?? ''); ?>"
                                       max="<?php echo esc_attr($field['max'] ?? ''); ?>">
                                       
                            <?php elseif ($field['type'] === 'image') : ?>
                                <input type="hidden" 
                                       name="<?php echo esc_attr($field_key); ?>" 
                                       value="<?php echo esc_attr($value); ?>"
                                       class="image-id">
                                <?php if ($value) : ?>
                                    <img src="<?php echo esc_url(wp_get_attachment_image_url($value, 'thumbnail')); ?>"
                                         style="max-width:100px">
                                <?php endif; ?>
                                <button type="button" class="button select-image">เลือกรูป</button>
                            <?php endif; ?>
                        </td>
                    </tr>
                <?php endforeach; ?>
            </table>
        </div>
        <?php
    }
    
    public function save( $post_id ) {
        if ( ! isset($_POST[$this->key]) ) {
            delete_post_meta($post_id, $this->key);
            return;
        }
        
        $rows = $_POST[$this->key];
        $sanitized = array();
        
        foreach ($rows as $row) {
            $clean_row = array();
            foreach ($this->fields as $field) {
                $value = $row[$field['key']] ?? '';
                
                switch ($field['type']) {
                    case 'text':
                        $clean_row[$field['key']] = sanitize_text_field($value);
                        break;
                    case 'textarea':
                        $clean_row[$field['key']] = sanitize_textarea_field($value);
                        break;
                    case 'number':
                        $clean_row[$field['key']] = floatval($value);
                        break;
                    case 'image':
                        $clean_row[$field['key']] = absint($value);
                        break;
                    default:
                        $clean_row[$field['key']] = sanitize_text_field($value);
                }
            }
            $sanitized[] = $clean_row;
        }
        
        update_post_meta($post_id, $this->key, $sanitized);
    }
}

// ==========================================
// ตัวอย่างการใช้ Repeater
// ==========================================

// สร้าง Repeater สำหรับ Team Members
$team_repeater = new Repeater_Field(
    '_team_members',
    array(
        array( 'key' => 'name',     'label' => 'ชื่อ',       'type' => 'text' ),
        array( 'key' => 'position', 'label' => 'ตำแหน่ง',   'type' => 'text' ),
        array( 'key' => 'bio',      'label' => 'ประวัติย่อ', 'type' => 'textarea' ),
        array( 'key' => 'photo',    'label' => 'รูปภาพ',     'type' => 'image' ),
        array( 'key' => 'order',    'label' => 'ลำดับ',       'type' => 'number', 'min' => 1 ),
    )
);

// ใน Meta Box render function
add_meta_box('team_members', 'ทีมงาน', function($post) use ($team_repeater) {
    wp_nonce_field('save_team_members', 'team_nonce');
    $team_repeater->render($post);
}, 'page');

// ใน save_post
add_action('save_post', function($post_id) use ($team_repeater) {
    if (!isset($_POST['team_nonce']) || 
        !wp_verify_nonce($_POST['team_nonce'], 'save_team_members')) {
        return;
    }
    $team_repeater->save($post_id);
});

// ==========================================
// ดึงข้อมูล Repeater ใน Template
// ==========================================

$team_members = get_post_meta(get_the_ID(), '_team_members', true);

if (is_array($team_members) && !empty($team_members)) :
    echo '<div class="team-grid">';
    foreach ($team_members as $member) :
        ?>
        <div class="team-card">
            <?php if (!empty($member['photo'])) : ?>
                <?php echo wp_get_attachment_image($member['photo'], 'medium', false, array('class' => 'team-photo')); ?>
            <?php endif; ?>
            <h3><?php echo esc_html($member['name']); ?></h3>
            <p class="position"><?php echo esc_html($member['position']); ?></p>
            <p><?php echo esc_html($member['bio']); ?></p>
        </div>
        <?php
    endforeach;
    echo '</div>';
endif;
```

---

## 4. User Meta

```php
<?php
/**
 * User Meta - Custom Fields สำหรับ Users
 */

// ==========================================
// เพิ่ม Fields ใน User Profile
// ==========================================

function add_user_profile_fields( $user ) {
    
    $phone      = get_user_meta( $user->ID, 'phone', true );
    $address    = get_user_meta( $user->ID, 'address', true );
    $birth_date = get_user_meta( $user->ID, 'birth_date', true );
    $avatar_id  = get_user_meta( $user->ID, 'custom_avatar', true );
    
    ?>
    <h3>ข้อมูลเพิ่มเติม</h3>
    
    <table class="form-table">
        <tr>
            <th><label for="phone">เบอร์โทรศัพท์</label></th>
            <td>
                <input type="tel" 
                       id="phone" 
                       name="phone" 
                       value="<?php echo esc_attr($phone); ?>"
                       class="regular-text">
            </td>
        </tr>
        
        <tr>
            <th><label for="address">ที่อยู่</label></th>
            <td>
                <textarea id="address" name="address" rows="4" class="regular-text">
                    <?php echo esc_textarea($address); ?>
                </textarea>
            </td>
        </tr>
        
        <tr>
            <th><label for="birth_date">วันเกิด</label></th>
            <td>
                <input type="date" 
                       id="birth_date" 
                       name="birth_date" 
                       value="<?php echo esc_attr($birth_date); ?>">
            </td>
        </tr>
    </table>
    
    <?php wp_nonce_field( 'save_user_profile', 'user_profile_nonce' ); ?>
    <?php
}
add_action( 'show_user_profile', 'add_user_profile_fields' );
add_action( 'edit_user_profile', 'add_user_profile_fields' );

// ==========================================
// บันทึก User Meta
// ==========================================

function save_user_profile_fields( $user_id ) {
    
    if ( ! isset($_POST['user_profile_nonce']) ||
         ! wp_verify_nonce($_POST['user_profile_nonce'], 'save_user_profile') ) {
        return;
    }
    
    if ( ! current_user_can('edit_user', $user_id) ) {
        return;
    }
    
    if ( isset($_POST['phone']) ) {
        update_user_meta( $user_id, 'phone', sanitize_text_field($_POST['phone']) );
    }
    
    if ( isset($_POST['address']) ) {
        update_user_meta( $user_id, 'address', sanitize_textarea_field($_POST['address']) );
    }
    
    if ( isset($_POST['birth_date']) ) {
        update_user_meta( $user_id, 'birth_date', sanitize_text_field($_POST['birth_date']) );
    }
}
add_action( 'personal_options_update', 'save_user_profile_fields' );
add_action( 'edit_user_profile_update', 'save_user_profile_fields' );

// ==========================================
// Term Meta (Taxonomy Meta)
// ==========================================

// เพิ่ม Field ให้ Category
add_action( 'category_add_form_fields', function() {
    ?>
    <div class="form-field">
        <label for="category_image">รูปภาพ Category</label>
        <input type="hidden" id="category_image" name="category_image" value="">
        <img id="category_image_preview" src="" style="display:none; max-width:150px">
        <button type="button" class="button" id="upload_category_image">เลือกรูป</button>
    </div>
    <?php
} );

add_action( 'created_category', function( $term_id ) {
    if ( isset($_POST['category_image']) ) {
        update_term_meta( $term_id, 'image_id', absint($_POST['category_image']) );
    }
} );

add_action( 'edit_category_form_fields', function( $term ) {
    $image_id = get_term_meta( $term->term_id, 'image_id', true );
    $image_url = $image_id ? wp_get_attachment_image_url($image_id, 'thumbnail') : '';
    ?>
    <tr class="form-field">
        <th><label for="category_image">รูปภาพ Category</label></th>
        <td>
            <input type="hidden" id="category_image" name="category_image" 
                   value="<?php echo esc_attr($image_id); ?>">
            <?php if ($image_url) : ?>
                <img id="category_image_preview" src="<?php echo esc_url($image_url); ?>" 
                     style="max-width:150px">
            <?php endif; ?>
            <button type="button" class="button" id="upload_category_image">เลือกรูป</button>
        </td>
    </tr>
    <?php
} );

add_action( 'edited_category', function( $term_id ) {
    if ( isset($_POST['category_image']) ) {
        update_term_meta( $term_id, 'image_id', absint($_POST['category_image']) );
    }
} );

// ดึงค่า Term Meta
$image_id = get_term_meta( $term_id, 'image_id', true );
```

---

## 5. Option Pages (Theme/Plugin Settings)

```php
<?php
/**
 * Complex Options Page
 */

class Advanced_Options_Page {
    
    private $option_key = 'my_advanced_options';
    private $page_slug  = 'my-advanced-options';
    
    private $fields = array();
    
    public function __construct() {
        add_action( 'admin_menu', array($this, 'add_page') );
        add_action( 'admin_init', array($this, 'register_settings') );
        add_action( 'admin_enqueue_scripts', array($this, 'enqueue_scripts') );
        
        $this->define_fields();
    }
    
    private function define_fields() {
        $this->fields = array(
            'general' => array(
                'title'  => 'ทั่วไป',
                'fields' => array(
                    array(
                        'id'    => 'site_name_override',
                        'label' => 'ชื่อเว็บไซต์ (Override)',
                        'type'  => 'text',
                        'desc'  => 'ถ้าว่างจะใช้ค่าจาก Settings > General',
                    ),
                    array(
                        'id'    => 'maintenance_mode',
                        'label' => 'Maintenance Mode',
                        'type'  => 'checkbox',
                        'desc'  => 'ปิดหน้าเว็บชั่วคราว',
                    ),
                ),
            ),
            'api' => array(
                'title'  => 'API Settings',
                'fields' => array(
                    array(
                        'id'    => 'google_maps_key',
                        'label' => 'Google Maps API Key',
                        'type'  => 'password',
                    ),
                    array(
                        'id'      => 'api_environment',
                        'label'   => 'Environment',
                        'type'    => 'select',
                        'options' => array(
                            'production' => 'Production',
                            'staging'    => 'Staging',
                            'development' => 'Development',
                        ),
                    ),
                ),
            ),
        );
    }
    
    public function add_page() {
        add_options_page(
            'Advanced Options',
            'Advanced Options',
            'manage_options',
            $this->page_slug,
            array($this, 'render_page')
        );
    }
    
    public function register_settings() {
        register_setting(
            $this->page_slug,
            $this->option_key,
            array($this, 'sanitize')
        );
        
        foreach ($this->fields as $section_id => $section) {
            add_settings_section(
                $section_id,
                $section['title'],
                null,
                $this->page_slug
            );
            
            foreach ($section['fields'] as $field) {
                add_settings_field(
                    $field['id'],
                    $field['label'],
                    array($this, 'render_field'),
                    $this->page_slug,
                    $section_id,
                    $field
                );
            }
        }
    }
    
    public function render_field( $field ) {
        $options = get_option($this->option_key, array());
        $value   = $options[$field['id']] ?? '';
        $name    = $this->option_key . '[' . $field['id'] . ']';
        
        switch ($field['type']) {
            case 'text':
            case 'password':
                echo '<input type="' . esc_attr($field['type']) . '" name="' . esc_attr($name) . '" value="' . esc_attr($value) . '" class="regular-text">';
                break;
            case 'checkbox':
                echo '<input type="checkbox" name="' . esc_attr($name) . '" value="1" ' . checked($value, '1', false) . '>';
                break;
            case 'select':
                echo '<select name="' . esc_attr($name) . '">';
                foreach ($field['options'] as $opt_val => $opt_label) {
                    echo '<option value="' . esc_attr($opt_val) . '" ' . selected($value, $opt_val, false) . '>' . esc_html($opt_label) . '</option>';
                }
                echo '</select>';
                break;
        }
        
        if (!empty($field['desc'])) {
            echo '<p class="description">' . esc_html($field['desc']) . '</p>';
        }
    }
    
    public function sanitize( $input ) {
        $sanitized = array();
        
        foreach ($this->fields as $section) {
            foreach ($section['fields'] as $field) {
                $value = $input[$field['id']] ?? '';
                
                switch ($field['type']) {
                    case 'text':
                        $sanitized[$field['id']] = sanitize_text_field($value);
                        break;
                    case 'password':
                        $sanitized[$field['id']] = sanitize_text_field($value);
                        break;
                    case 'checkbox':
                        $sanitized[$field['id']] = $value ? '1' : '0';
                        break;
                    case 'select':
                        $allowed = array_keys($field['options']);
                        $sanitized[$field['id']] = in_array($value, $allowed) ? $value : '';
                        break;
                }
            }
        }
        
        return $sanitized;
    }
    
    public function render_page() {
        ?>
        <div class="wrap">
            <h1>Advanced Options</h1>
            <form method="post" action="options.php">
                <?php
                settings_fields($this->page_slug);
                do_settings_sections($this->page_slug);
                submit_button();
                ?>
            </form>
        </div>
        <?php
    }
    
    public function enqueue_scripts($hook) {
        if ($hook !== 'settings_page_' . $this->page_slug) return;
        wp_enqueue_media();
    }
    
    // Static getter
    public static function get($key, $default = null) {
        $options = get_option('my_advanced_options', array());
        return $options[$key] ?? $default;
    }
}

new Advanced_Options_Page();

// ดึงค่า
$google_key = Advanced_Options_Page::get('google_maps_key', '');
```

---

## Workshop: สร้าง Event Meta System

```php
<?php
// TODO: สร้าง Meta Box System สำหรับ Event Post Type
// Fields ที่ต้องการ:
// 1. Event Start Date/Time
// 2. Event End Date/Time
// 3. Venue (ชื่อสถานที่)
// 4. Address
// 5. Google Maps Link
// 6. Ticket Price (Repeater: ประเภทบัตร, ราคา, จำนวน)
// 7. Organizer Name
// 8. Organizer Email
// 9. Max Attendees

// Template ให้เติม:
function event_meta_render($post) {
    wp_nonce_field('save_event_meta', 'event_meta_nonce');
    
    $start_date    = get_post_meta($post->ID, '_event_start_date', true);
    $end_date      = get_post_meta($post->ID, '_event_end_date', true);
    $venue         = get_post_meta($post->ID, '_event_venue', true);
    // TODO: ดึง fields อื่นๆ
    
    ?>
    <table class="form-table">
        <tr>
            <th><label for="_event_start_date">วันที่เริ่ม</label></th>
            <td>
                <input type="datetime-local"
                       name="_event_start_date"
                       value="<?php echo esc_attr($start_date); ?>">
            </td>
        </tr>
        <!-- TODO: เพิ่ม fields อื่นๆ -->
    </table>
    <?php
}
```

---

## Quiz

**คำถามที่ 1:** `get_post_meta($id, 'key', true)` vs `get_post_meta($id, 'key', false)` ต่างกันอย่างไร?

A) true = ดึงค่าเดียว (string), false = ดึงทุกค่า (array)  
B) true = cache, false = no cache  
C) true = hidden fields, false = all fields  
D) ไม่มีความแตกต่าง  

**เฉลย: A) true ส่งคืนค่าแรก (string), false ส่งคืน array ของทุก values**

---

**คำถามที่ 2:** Meta Key ที่ขึ้นต้นด้วย underscore `_` มีความหมายว่าอย่างไร?

A) Protected field ลบไม่ได้  
B) Hidden field ไม่แสดงใน Custom Fields box ใน Admin  
C) Required field  
D) Encrypted value  

**เฉลย: B) จะถูกซ่อนจาก Custom Fields meta box ใน Admin Panel**

---

**คำถามที่ 3:** เมื่อบันทึก Meta ที่มาจาก User Input ต้องทำอะไรก่อน update_post_meta?

A) Base64 encode  
B) Sanitize ข้อมูล (sanitize_text_field, absint, etc.)  
C) JSON encode  
D) Encrypt  

**เฉลย: B) Sanitize ข้อมูลเสมอก่อนบันทึก เพื่อป้องกัน XSS และ Data Corruption**

---

**คำถามที่ 4:** `get_term_meta()` ใช้สำหรับอะไร?

A) ดึง Meta ของ Post  
B) ดึง Meta ของ Term (Category, Tag, Custom Taxonomy)  
C) ดึง Meta ของ Comment  
D) ดึง Meta ของ User  

**เฉลย: B) get_term_meta ดึง Custom Fields ของ Taxonomy Terms**

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- Post Meta API อย่างละเอียด
- Advanced Meta Box Builder System
- Repeater Fields สำหรับ Multiple Values
- User Meta และ Term Meta
- Option Pages สำหรับ Plugin Settings

---

## ต่อไป

➡️ **[Part 057: WordPress REST API](part-057-wordpress-rest-api.md)**

เรียนรู้เกี่ยวกับ:
- WP REST API
- Custom Endpoints
- Authentication
- CRUD Operations ผ่าน API
