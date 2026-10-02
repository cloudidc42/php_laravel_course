# 📝 Part 9: PHP Forms - การทำงานกับ HTML Forms

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้างและจัดการ HTML Forms ใน PHP ได้
- ใช้งาน `$_GET` และ `$_POST` superglobals ได้อย่างถูกต้อง
- ทำ Form Validation ทั้งฝั่ง client และ server ได้
- จัดการการ Upload ไฟล์ได้อย่างปลอดภัย
- ป้องกัน CSRF Attack ได้
- สร้าง Contact Form ที่ปลอดภัยและใช้งานได้จริง

---

## 📌 1. HTML Forms พื้นฐาน

### 1.1 โครงสร้าง HTML Form

Form คือ องค์ประกอบ HTML ที่ใช้รับข้อมูลจากผู้ใช้แล้วส่งไปยัง server

```html
<!-- basic-form.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Basic Form</title>
</head>
<body>
    <!-- method="post" หรือ method="get" -->
    <!-- action คือ URL ที่จะส่งข้อมูลไป -->
    <form action="process.php" method="post">
        
        <!-- Text Input -->
        <label for="name">ชื่อ:</label>
        <input type="text" id="name" name="name" placeholder="กรอกชื่อของคุณ">
        
        <!-- Email Input -->
        <label for="email">อีเมล:</label>
        <input type="email" id="email" name="email">
        
        <!-- Password Input -->
        <label for="password">รหัสผ่าน:</label>
        <input type="password" id="password" name="password">
        
        <!-- Number Input -->
        <label for="age">อายุ:</label>
        <input type="number" id="age" name="age" min="1" max="150">
        
        <!-- Textarea -->
        <label for="message">ข้อความ:</label>
        <textarea id="message" name="message" rows="4" cols="50"></textarea>
        
        <!-- Select Dropdown -->
        <label for="country">ประเทศ:</label>
        <select id="country" name="country">
            <option value="">-- เลือกประเทศ --</option>
            <option value="th">ไทย</option>
            <option value="us">สหรัฐอเมริกา</option>
            <option value="jp">ญี่ปุ่น</option>
        </select>
        
        <!-- Radio Buttons -->
        <p>เพศ:</p>
        <input type="radio" id="male" name="gender" value="male">
        <label for="male">ชาย</label>
        <input type="radio" id="female" name="gender" value="female">
        <label for="female">หญิง</label>
        
        <!-- Checkboxes -->
        <p>งานอดิเรก:</p>
        <input type="checkbox" id="reading" name="hobbies[]" value="reading">
        <label for="reading">อ่านหนังสือ</label>
        <input type="checkbox" id="gaming" name="hobbies[]" value="gaming">
        <label for="gaming">เล่นเกม</label>
        <input type="checkbox" id="cooking" name="hobbies[]" value="cooking">
        <label for="cooking">ทำอาหาร</label>
        
        <!-- Hidden Input -->
        <input type="hidden" name="form_id" value="contact_form">
        
        <!-- Submit Button -->
        <button type="submit">ส่งข้อมูล</button>
        <button type="reset">ล้างข้อมูล</button>
    </form>
</body>
</html>
```

### 1.2 HTTP Methods: GET vs POST

| คุณสมบัติ | GET | POST |
|-----------|-----|------|
| ข้อมูลใน URL | ✅ ใช่ | ❌ ไม่ |
| ความปลอดภัย | ต่ำ | สูงกว่า |
| ขนาดข้อมูล | จำกัด (~2000 chars) | ไม่จำกัด |
| การ Bookmark | ได้ | ไม่ได้ |
| การใช้งาน | ค้นหา, Filter | ส่งข้อมูลสำคัญ |

---

## 📌 2. $_GET และ $_POST Superglobals

### 2.1 การรับข้อมูลจาก GET

```php
<?php
// get-example.php
// URL: get-example.php?name=สมชาย&age=25&city=Bangkok

// ตรวจสอบว่ามีค่าใน $_GET หรือไม่
if (isset($_GET['name'])) {
    $name = $_GET['name'];
    echo "สวัสดี, " . htmlspecialchars($name);
}

// ใช้ null coalescing operator
$age = $_GET['age'] ?? 'ไม่ระบุ';
$city = $_GET['city'] ?? 'ไม่ระบุ';

echo "<br>อายุ: " . htmlspecialchars($age);
echo "<br>เมือง: " . htmlspecialchars($city);

// รับค่าหลายตัว
$search = $_GET['q'] ?? '';
$page = (int)($_GET['page'] ?? 1);
$limit = (int)($_GET['limit'] ?? 10);

echo "<br>ค้นหา: " . htmlspecialchars($search);
echo "<br>หน้า: $page";
echo "<br>แสดง: $limit รายการ";
?>
```

```html
<!-- search-form.html -->
<form action="search.php" method="get">
    <input type="text" name="q" placeholder="ค้นหา...">
    <select name="category">
        <option value="all">ทั้งหมด</option>
        <option value="news">ข่าว</option>
        <option value="product">สินค้า</option>
    </select>
    <button type="submit">ค้นหา</button>
</form>
```

```php
<?php
// search.php
$query = $_GET['q'] ?? '';
$category = $_GET['category'] ?? 'all';

// แสดงผลการค้นหา
if (!empty($query)) {
    echo "<h2>ผลการค้นหา: " . htmlspecialchars($query) . "</h2>";
    echo "<p>หมวดหมู่: " . htmlspecialchars($category) . "</p>";
    
    // จำลองผลการค้นหา
    $results = [
        ['title' => 'ผลลัพธ์ที่ 1', 'url' => '#'],
        ['title' => 'ผลลัพธ์ที่ 2', 'url' => '#'],
        ['title' => 'ผลลัพธ์ที่ 3', 'url' => '#'],
    ];
    
    foreach ($results as $result) {
        echo "<div>";
        echo "<a href='" . $result['url'] . "'>" . $result['title'] . "</a>";
        echo "</div>";
    }
} else {
    echo "<p>กรุณากรอกคำค้นหา</p>";
}
?>
```

### 2.2 การรับข้อมูลจาก POST

```php
<?php
// post-example.php

// ตรวจสอบว่า Form ถูก Submit หรือไม่
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    
    // รับข้อมูลพื้นฐาน
    $name = $_POST['name'] ?? '';
    $email = $_POST['email'] ?? '';
    $message = $_POST['message'] ?? '';
    
    // รับข้อมูลจาก Select
    $country = $_POST['country'] ?? '';
    
    // รับข้อมูลจาก Radio
    $gender = $_POST['gender'] ?? '';
    
    // รับข้อมูลจาก Checkbox (array)
    $hobbies = $_POST['hobbies'] ?? [];
    
    // แสดงข้อมูลที่รับมา
    echo "<h2>ข้อมูลที่ได้รับ:</h2>";
    echo "<p>ชื่อ: " . htmlspecialchars($name) . "</p>";
    echo "<p>อีเมล: " . htmlspecialchars($email) . "</p>";
    echo "<p>ข้อความ: " . htmlspecialchars($message) . "</p>";
    echo "<p>ประเทศ: " . htmlspecialchars($country) . "</p>";
    echo "<p>เพศ: " . htmlspecialchars($gender) . "</p>";
    
    echo "<p>งานอดิเรก: ";
    if (!empty($hobbies)) {
        echo implode(', ', array_map('htmlspecialchars', $hobbies));
    } else {
        echo "ไม่ระบุ";
    }
    echo "</p>";
    
} else {
    echo "<p>ไม่มีข้อมูลถูกส่งมา</p>";
}
?>
```

### 2.3 ข้อมูลจาก Server Variables

```php
<?php
// server-info.php

// ข้อมูลเกี่ยวกับ Request
echo "HTTP Method: " . $_SERVER['REQUEST_METHOD'] . "<br>";
echo "Request URI: " . $_SERVER['REQUEST_URI'] . "<br>";
echo "Script Name: " . $_SERVER['SCRIPT_NAME'] . "<br>";

// ข้อมูลเกี่ยวกับ Client
echo "IP Address: " . $_SERVER['REMOTE_ADDR'] . "<br>";
echo "User Agent: " . $_SERVER['HTTP_USER_AGENT'] . "<br>";

// ข้อมูลเกี่ยวกับ Server
echo "Server Name: " . $_SERVER['SERVER_NAME'] . "<br>";
echo "Server Port: " . $_SERVER['SERVER_PORT'] . "<br>";
echo "Document Root: " . $_SERVER['DOCUMENT_ROOT'] . "<br>";

// Protocol
$protocol = isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on' ? 'https' : 'http';
$currentUrl = $protocol . '://' . $_SERVER['HTTP_HOST'] . $_SERVER['REQUEST_URI'];
echo "Current URL: " . $currentUrl . "<br>";
?>
```

---

## 📌 3. Form Validation

### 3.1 Server-side Validation พื้นฐาน

```php
<?php
// validation-basic.php

$errors = [];
$data = [];

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    
    // ======================================
    // Validation ชื่อ
    // ======================================
    $name = trim($_POST['name'] ?? '');
    
    if (empty($name)) {
        $errors['name'] = 'กรุณากรอกชื่อ';
    } elseif (strlen($name) < 2) {
        $errors['name'] = 'ชื่อต้องมีอย่างน้อย 2 ตัวอักษร';
    } elseif (strlen($name) > 50) {
        $errors['name'] = 'ชื่อต้องไม่เกิน 50 ตัวอักษร';
    } else {
        $data['name'] = $name;
    }
    
    // ======================================
    // Validation อีเมล
    // ======================================
    $email = trim($_POST['email'] ?? '');
    
    if (empty($email)) {
        $errors['email'] = 'กรุณากรอกอีเมล';
    } elseif (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        $errors['email'] = 'รูปแบบอีเมลไม่ถูกต้อง';
    } else {
        $data['email'] = $email;
    }
    
    // ======================================
    // Validation เบอร์โทร
    // ======================================
    $phone = trim($_POST['phone'] ?? '');
    
    if (!empty($phone)) {
        // ตรวจสอบรูปแบบเบอร์โทรไทย
        if (!preg_match('/^(0[689]\d{8}|0[2-9]\d{7})$/', $phone)) {
            $errors['phone'] = 'รูปแบบเบอร์โทรไม่ถูกต้อง (เช่น 0891234567)';
        } else {
            $data['phone'] = $phone;
        }
    }
    
    // ======================================
    // Validation รหัสผ่าน
    // ======================================
    $password = $_POST['password'] ?? '';
    $confirm_password = $_POST['confirm_password'] ?? '';
    
    if (empty($password)) {
        $errors['password'] = 'กรุณากรอกรหัสผ่าน';
    } elseif (strlen($password) < 8) {
        $errors['password'] = 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
    } elseif (!preg_match('/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/', $password)) {
        $errors['password'] = 'รหัสผ่านต้องมีตัวอักษรพิมพ์เล็ก พิมพ์ใหญ่ และตัวเลข';
    } else {
        if ($password !== $confirm_password) {
            $errors['confirm_password'] = 'รหัสผ่านไม่ตรงกัน';
        } else {
            $data['password'] = password_hash($password, PASSWORD_BCRYPT);
        }
    }
    
    // ======================================
    // Validation วันเกิด
    // ======================================
    $birthdate = $_POST['birthdate'] ?? '';
    
    if (!empty($birthdate)) {
        $date = DateTime::createFromFormat('Y-m-d', $birthdate);
        if (!$date || $date->format('Y-m-d') !== $birthdate) {
            $errors['birthdate'] = 'รูปแบบวันที่ไม่ถูกต้อง';
        } else {
            $today = new DateTime();
            $age = $today->diff($date)->y;
            if ($age < 13) {
                $errors['birthdate'] = 'ต้องมีอายุอย่างน้อย 13 ปี';
            } elseif ($age > 120) {
                $errors['birthdate'] = 'วันเกิดไม่ถูกต้อง';
            } else {
                $data['birthdate'] = $birthdate;
            }
        }
    }
    
    // ======================================
    // Validation URL
    // ======================================
    $website = trim($_POST['website'] ?? '');
    
    if (!empty($website)) {
        if (!filter_var($website, FILTER_VALIDATE_URL)) {
            $errors['website'] = 'รูปแบบ URL ไม่ถูกต้อง';
        } else {
            $data['website'] = $website;
        }
    }
    
    // ======================================
    // ถ้าไม่มี Error ให้บันทึกข้อมูล
    // ======================================
    if (empty($errors)) {
        // บันทึกข้อมูลลง Database หรือทำอย่างอื่น
        echo "<div style='color: green;'>✅ ข้อมูลถูกต้อง! บันทึกสำเร็จ</div>";
        echo "<pre>" . print_r($data, true) . "</pre>";
    }
}
?>

<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Form Validation</title>
    <style>
        .error { color: red; font-size: 0.85em; }
        .form-group { margin-bottom: 15px; }
        input.invalid { border-color: red; }
        input.valid { border-color: green; }
    </style>
</head>
<body>
    <form action="" method="post" novalidate>
        <div class="form-group">
            <label>ชื่อ:</label>
            <input type="text" name="name" 
                   value="<?= htmlspecialchars($_POST['name'] ?? '') ?>"
                   class="<?= isset($errors['name']) ? 'invalid' : '' ?>">
            <?php if (isset($errors['name'])): ?>
                <span class="error"><?= $errors['name'] ?></span>
            <?php endif; ?>
        </div>
        
        <div class="form-group">
            <label>อีเมล:</label>
            <input type="email" name="email" 
                   value="<?= htmlspecialchars($_POST['email'] ?? '') ?>"
                   class="<?= isset($errors['email']) ? 'invalid' : '' ?>">
            <?php if (isset($errors['email'])): ?>
                <span class="error"><?= $errors['email'] ?></span>
            <?php endif; ?>
        </div>
        
        <div class="form-group">
            <label>เบอร์โทร:</label>
            <input type="text" name="phone" 
                   value="<?= htmlspecialchars($_POST['phone'] ?? '') ?>"
                   class="<?= isset($errors['phone']) ? 'invalid' : '' ?>">
            <?php if (isset($errors['phone'])): ?>
                <span class="error"><?= $errors['phone'] ?></span>
            <?php endif; ?>
        </div>
        
        <div class="form-group">
            <label>รหัสผ่าน:</label>
            <input type="password" name="password"
                   class="<?= isset($errors['password']) ? 'invalid' : '' ?>">
            <?php if (isset($errors['password'])): ?>
                <span class="error"><?= $errors['password'] ?></span>
            <?php endif; ?>
        </div>
        
        <div class="form-group">
            <label>ยืนยันรหัสผ่าน:</label>
            <input type="password" name="confirm_password"
                   class="<?= isset($errors['confirm_password']) ? 'invalid' : '' ?>">
            <?php if (isset($errors['confirm_password'])): ?>
                <span class="error"><?= $errors['confirm_password'] ?></span>
            <?php endif; ?>
        </div>
        
        <button type="submit">ลงทะเบียน</button>
    </form>
</body>
</html>
```

### 3.2 Validation Class แบบ Reusable

```php
<?php
// Validator.php

class Validator
{
    private array $errors = [];
    private array $data = [];
    
    /**
     * Validate ข้อมูลตาม rules ที่กำหนด
     */
    public function validate(array $input, array $rules): bool
    {
        $this->errors = [];
        $this->data = [];
        
        foreach ($rules as $field => $ruleString) {
            $value = $input[$field] ?? null;
            $fieldRules = explode('|', $ruleString);
            
            foreach ($fieldRules as $rule) {
                $ruleName = $rule;
                $ruleParam = null;
                
                // แยก rule ออกจาก parameter (เช่น min:3)
                if (strpos($rule, ':') !== false) {
                    [$ruleName, $ruleParam] = explode(':', $rule, 2);
                }
                
                $result = $this->applyRule($field, $value, $ruleName, $ruleParam);
                
                if ($result !== true) {
                    $this->errors[$field] = $result;
                    break; // หยุดที่ error แรก
                }
            }
            
            if (!isset($this->errors[$field])) {
                $this->data[$field] = $value;
            }
        }
        
        return empty($this->errors);
    }
    
    private function applyRule(string $field, $value, string $rule, $param): bool|string
    {
        $fieldLabel = ucfirst(str_replace('_', ' ', $field));
        
        switch ($rule) {
            case 'required':
                if ($value === null || $value === '') {
                    return "$fieldLabel is required";
                }
                break;
                
            case 'min':
                if (strlen((string)$value) < (int)$param) {
                    return "$fieldLabel must be at least $param characters";
                }
                break;
                
            case 'max':
                if (strlen((string)$value) > (int)$param) {
                    return "$fieldLabel must not exceed $param characters";
                }
                break;
                
            case 'email':
                if (!empty($value) && !filter_var($value, FILTER_VALIDATE_EMAIL)) {
                    return "$fieldLabel must be a valid email address";
                }
                break;
                
            case 'numeric':
                if (!empty($value) && !is_numeric($value)) {
                    return "$fieldLabel must be a number";
                }
                break;
                
            case 'integer':
                if (!empty($value) && !filter_var($value, FILTER_VALIDATE_INT)) {
                    return "$fieldLabel must be an integer";
                }
                break;
                
            case 'url':
                if (!empty($value) && !filter_var($value, FILTER_VALIDATE_URL)) {
                    return "$fieldLabel must be a valid URL";
                }
                break;
                
            case 'regex':
                if (!empty($value) && !preg_match($param, $value)) {
                    return "$fieldLabel format is invalid";
                }
                break;
                
            case 'in':
                $allowedValues = explode(',', $param);
                if (!empty($value) && !in_array($value, $allowedValues)) {
                    return "$fieldLabel must be one of: " . implode(', ', $allowedValues);
                }
                break;
        }
        
        return true;
    }
    
    public function errors(): array
    {
        return $this->errors;
    }
    
    public function getData(): array
    {
        return $this->data;
    }
    
    public function hasError(string $field): bool
    {
        return isset($this->errors[$field]);
    }
    
    public function getError(string $field): string
    {
        return $this->errors[$field] ?? '';
    }
}

// ตัวอย่างการใช้งาน
$validator = new Validator();

$isValid = $validator->validate($_POST, [
    'name'     => 'required|min:2|max:50',
    'email'    => 'required|email',
    'age'      => 'required|integer',
    'website'  => 'url',
    'gender'   => 'required|in:male,female,other',
]);

if ($isValid) {
    $data = $validator->getData();
    // ดำเนินการต่อ...
} else {
    $errors = $validator->errors();
    // แสดง errors...
}
?>
```

---

## 📌 4. File Upload

### 4.1 HTML Form สำหรับ Upload

```html
<!-- upload-form.html -->
<!-- ต้องมี enctype="multipart/form-data" สำหรับ file upload -->
<form action="upload.php" method="post" enctype="multipart/form-data">
    <label for="avatar">เลือกรูปภาพโปรไฟล์:</label>
    <input type="file" id="avatar" name="avatar" accept="image/*">
    
    <label for="documents">อัปโหลดเอกสาร (หลายไฟล์):</label>
    <input type="file" id="documents" name="documents[]" multiple accept=".pdf,.doc,.docx">
    
    <button type="submit">อัปโหลด</button>
</form>
```

### 4.2 PHP File Upload Handler

```php
<?php
// upload.php

class FileUploader
{
    private string $uploadDir;
    private array $allowedTypes;
    private int $maxSize; // bytes
    private array $errors = [];
    
    public function __construct(
        string $uploadDir = 'uploads/',
        array $allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'],
        int $maxSize = 5 * 1024 * 1024 // 5MB
    ) {
        $this->uploadDir = rtrim($uploadDir, '/') . '/';
        $this->allowedTypes = $allowedTypes;
        $this->maxSize = $maxSize;
        
        // สร้าง directory ถ้ายังไม่มี
        if (!is_dir($this->uploadDir)) {
            mkdir($this->uploadDir, 0755, true);
        }
    }
    
    /**
     * อัปโหลดไฟล์เดียว
     */
    public function upload(array $file): string|false
    {
        $this->errors = [];
        
        // ตรวจสอบ error จาก PHP
        if ($file['error'] !== UPLOAD_ERR_OK) {
            $this->errors[] = $this->getUploadErrorMessage($file['error']);
            return false;
        }
        
        // ตรวจสอบขนาดไฟล์
        if ($file['size'] > $this->maxSize) {
            $this->errors[] = 'ไฟล์มีขนาดเกิน ' . $this->formatBytes($this->maxSize);
            return false;
        }
        
        // ตรวจสอบประเภทไฟล์ (ใช้ finfo สำหรับความปลอดภัยสูงสุด)
        $finfo = new finfo(FILEINFO_MIME_TYPE);
        $mimeType = $finfo->file($file['tmp_name']);
        
        if (!in_array($mimeType, $this->allowedTypes)) {
            $this->errors[] = 'ประเภทไฟล์ไม่ได้รับอนุญาต: ' . $mimeType;
            return false;
        }
        
        // ตรวจสอบว่าเป็นไฟล์ที่ upload มาจริงๆ
        if (!is_uploaded_file($file['tmp_name'])) {
            $this->errors[] = 'ไฟล์ไม่ได้ถูก upload อย่างถูกต้อง';
            return false;
        }
        
        // สร้างชื่อไฟล์ที่ปลอดภัย
        $extension = $this->getExtensionFromMime($mimeType);
        $filename = $this->generateSafeFilename($extension);
        $destination = $this->uploadDir . $filename;
        
        // ย้ายไฟล์ไปยัง destination
        if (!move_uploaded_file($file['tmp_name'], $destination)) {
            $this->errors[] = 'ไม่สามารถบันทึกไฟล์ได้';
            return false;
        }
        
        return $filename;
    }
    
    /**
     * อัปโหลดหลายไฟล์
     */
    public function uploadMultiple(array $files): array
    {
        $results = [];
        
        // จัดรูปแบบ array ของไฟล์
        $fileList = $this->normalizeFileArray($files);
        
        foreach ($fileList as $index => $file) {
            $result = $this->upload($file);
            if ($result !== false) {
                $results[] = ['success' => true, 'filename' => $result];
            } else {
                $results[] = ['success' => false, 'errors' => $this->errors];
            }
        }
        
        return $results;
    }
    
    /**
     * แปลง $_FILES array format เป็น array ของ files
     */
    private function normalizeFileArray(array $files): array
    {
        $normalized = [];
        
        if (is_array($files['name'])) {
            for ($i = 0; $i < count($files['name']); $i++) {
                $normalized[] = [
                    'name'     => $files['name'][$i],
                    'type'     => $files['type'][$i],
                    'tmp_name' => $files['tmp_name'][$i],
                    'error'    => $files['error'][$i],
                    'size'     => $files['size'][$i],
                ];
            }
        } else {
            $normalized[] = $files;
        }
        
        return $normalized;
    }
    
    private function generateSafeFilename(string $extension): string
    {
        return bin2hex(random_bytes(16)) . '.' . $extension;
    }
    
    private function getExtensionFromMime(string $mimeType): string
    {
        $mimeToExt = [
            'image/jpeg' => 'jpg',
            'image/png'  => 'png',
            'image/gif'  => 'gif',
            'image/webp' => 'webp',
            'application/pdf' => 'pdf',
            'text/plain' => 'txt',
        ];
        
        return $mimeToExt[$mimeType] ?? 'bin';
    }
    
    private function getUploadErrorMessage(int $errorCode): string
    {
        $messages = [
            UPLOAD_ERR_INI_SIZE   => 'ไฟล์มีขนาดเกินที่ php.ini กำหนด',
            UPLOAD_ERR_FORM_SIZE  => 'ไฟล์มีขนาดเกินที่ Form กำหนด',
            UPLOAD_ERR_PARTIAL    => 'ไฟล์ถูก upload เพียงบางส่วน',
            UPLOAD_ERR_NO_FILE    => 'ไม่มีไฟล์ถูก upload',
            UPLOAD_ERR_NO_TMP_DIR => 'ไม่พบ temporary folder',
            UPLOAD_ERR_CANT_WRITE => 'ไม่สามารถเขียนไฟล์ลงดิสก์ได้',
            UPLOAD_ERR_EXTENSION  => 'PHP extension หยุดการ upload ไฟล์',
        ];
        
        return $messages[$errorCode] ?? "Upload error code: $errorCode";
    }
    
    private function formatBytes(int $bytes): string
    {
        $units = ['B', 'KB', 'MB', 'GB'];
        $i = 0;
        while ($bytes >= 1024 && $i < count($units) - 1) {
            $bytes /= 1024;
            $i++;
        }
        return round($bytes, 2) . ' ' . $units[$i];
    }
    
    public function getErrors(): array
    {
        return $this->errors;
    }
}

// ======================================
// การใช้งาน
// ======================================
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    
    // Upload รูปภาพเดียว
    if (isset($_FILES['avatar']) && $_FILES['avatar']['error'] !== UPLOAD_ERR_NO_FILE) {
        $uploader = new FileUploader(
            uploadDir: 'uploads/avatars/',
            allowedTypes: ['image/jpeg', 'image/png', 'image/webp'],
            maxSize: 2 * 1024 * 1024 // 2MB
        );
        
        $filename = $uploader->upload($_FILES['avatar']);
        
        if ($filename) {
            echo "✅ อัปโหลดสำเร็จ: $filename<br>";
            echo "<img src='uploads/avatars/$filename' width='200'>";
        } else {
            foreach ($uploader->getErrors() as $error) {
                echo "❌ $error<br>";
            }
        }
    }
    
    // Upload หลายไฟล์
    if (isset($_FILES['documents'])) {
        $uploader = new FileUploader(
            uploadDir: 'uploads/documents/',
            allowedTypes: ['application/pdf', 'image/jpeg', 'image/png'],
            maxSize: 10 * 1024 * 1024 // 10MB
        );
        
        $results = $uploader->uploadMultiple($_FILES['documents']);
        
        foreach ($results as $i => $result) {
            if ($result['success']) {
                echo "✅ ไฟล์ที่ " . ($i + 1) . ": " . $result['filename'] . "<br>";
            } else {
                echo "❌ ไฟล์ที่ " . ($i + 1) . ": " . implode(', ', $result['errors']) . "<br>";
            }
        }
    }
}
?>
```

---

## 📌 5. CSRF Protection

### 5.1 ทำความเข้าใจ CSRF Attack

CSRF (Cross-Site Request Forgery) คือการที่ attacker หลอกให้ browser ของ victim ส่ง request ที่ไม่ได้ต้องการไปยัง server

```
ตัวอย่าง CSRF Attack:
1. ผู้ใช้ login เข้าเว็บธนาคาร bank.com
2. ผู้ใช้เข้าเว็บ evil.com ในแท็บอื่น
3. evil.com มีโค้ด: <img src="https://bank.com/transfer?to=hacker&amount=10000">
4. Browser ส่ง request ไปยัง bank.com พร้อม session cookie อัตโนมัติ
5. ธนาคารโอนเงินให้ hacker เพราะคิดว่าเป็นผู้ใช้สั่ง
```

### 5.2 CSRF Token Implementation

```php
<?php
// CsrfProtection.php

class CsrfProtection
{
    private const TOKEN_LENGTH = 32;
    private const SESSION_KEY = 'csrf_token';
    
    /**
     * เริ่ม session และสร้าง token
     */
    public static function init(): void
    {
        if (session_status() === PHP_SESSION_NONE) {
            session_start();
        }
        
        if (!isset($_SESSION[self::SESSION_KEY])) {
            self::regenerate();
        }
    }
    
    /**
     * สร้าง CSRF token ใหม่
     */
    public static function regenerate(): void
    {
        $_SESSION[self::SESSION_KEY] = bin2hex(random_bytes(self::TOKEN_LENGTH));
    }
    
    /**
     * ดึง token ปัจจุบัน
     */
    public static function token(): string
    {
        self::init();
        return $_SESSION[self::SESSION_KEY];
    }
    
    /**
     * สร้าง HTML hidden input
     */
    public static function field(): string
    {
        return '<input type="hidden" name="_token" value="' . self::token() . '">';
    }
    
    /**
     * ตรวจสอบ token
     */
    public static function verify(string $token): bool
    {
        self::init();
        
        if (!isset($_SESSION[self::SESSION_KEY])) {
            return false;
        }
        
        // ใช้ hash_equals เพื่อป้องกัน timing attack
        return hash_equals($_SESSION[self::SESSION_KEY], $token);
    }
    
    /**
     * ตรวจสอบและ throw exception ถ้าไม่ผ่าน
     */
    public static function validateRequest(): void
    {
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            $token = $_POST['_token'] ?? '';
            
            if (!self::verify($token)) {
                http_response_code(419);
                die('CSRF token mismatch. Request rejected.');
            }
        }
    }
}

// ======================================
// การใช้งาน
// ======================================
CsrfProtection::init();

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // ตรวจสอบ CSRF token
    CsrfProtection::validateRequest();
    
    // ดำเนินการต่อถ้าผ่าน
    $name = htmlspecialchars($_POST['name'] ?? '');
    echo "ข้อมูลที่ได้รับ: $name";
}
?>

<form action="" method="post">
    <!-- เพิ่ม CSRF token field -->
    <?= CsrfProtection::field() ?>
    
    <input type="text" name="name" placeholder="ชื่อของคุณ">
    <button type="submit">ส่งข้อมูล</button>
</form>
```

---

## 🛠️ Workshop: สร้าง Contact Form ที่ปลอดภัย

### โครงสร้างโปรเจค

```
contact-form/
├── index.php          (หน้า Form)
├── process.php        (ประมวลผล)
├── success.php        (หน้าสำเร็จ)
├── includes/
│   ├── csrf.php
│   ├── validator.php
│   └── mailer.php
└── uploads/
    └── attachments/
```

### ไฟล์หลัก: index.php

```php
<?php
// index.php
session_start();

// สร้าง CSRF Token
if (!isset($_SESSION['csrf_token'])) {
    $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
}

// รับ errors และ old input จาก session
$errors = $_SESSION['errors'] ?? [];
$old = $_SESSION['old_input'] ?? [];

// ล้าง session
unset($_SESSION['errors'], $_SESSION['old_input']);

function old(string $field, string $default = ''): string {
    global $old;
    return htmlspecialchars($old[$field] ?? $default);
}

function hasError(string $field): bool {
    global $errors;
    return isset($errors[$field]);
}

function getError(string $field): string {
    global $errors;
    return $errors[$field] ?? '';
}
?>
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ติดต่อเรา</title>
    <style>
        * { box-sizing: border-box; }
        body { font-family: 'Sarabun', sans-serif; max-width: 600px; margin: 40px auto; padding: 20px; }
        h1 { color: #333; }
        .form-group { margin-bottom: 20px; }
        label { display: block; margin-bottom: 5px; font-weight: bold; }
        input, textarea, select {
            width: 100%; padding: 10px; border: 1px solid #ddd;
            border-radius: 4px; font-size: 16px;
        }
        input.error, textarea.error, select.error { border-color: #e74c3c; }
        .error-msg { color: #e74c3c; font-size: 0.85em; margin-top: 5px; }
        .success-msg { background: #d4edda; color: #155724; padding: 15px; border-radius: 4px; }
        .btn {
            background: #3498db; color: white; padding: 12px 30px;
            border: none; border-radius: 4px; cursor: pointer; font-size: 16px;
        }
        .btn:hover { background: #2980b9; }
        .char-count { font-size: 0.8em; color: #666; text-align: right; }
    </style>
</head>
<body>
    <h1>📬 ติดต่อเรา</h1>
    
    <?php if (isset($_SESSION['success'])): ?>
        <div class="success-msg">✅ <?= $_SESSION['success'] ?></div>
        <?php unset($_SESSION['success']); ?>
    <?php endif; ?>
    
    <form action="process.php" method="post" enctype="multipart/form-data" id="contactForm">
        <!-- CSRF Token -->
        <input type="hidden" name="csrf_token" value="<?= $_SESSION['csrf_token'] ?>">
        
        <!-- ชื่อ-นามสกุล -->
        <div class="form-group">
            <label for="name">ชื่อ-นามสกุล <span style="color:red">*</span></label>
            <input type="text" id="name" name="name" 
                   value="<?= old('name') ?>"
                   class="<?= hasError('name') ? 'error' : '' ?>"
                   placeholder="เช่น สมชาย ใจดี"
                   maxlength="100">
            <?php if (hasError('name')): ?>
                <div class="error-msg">⚠️ <?= getError('name') ?></div>
            <?php endif; ?>
        </div>
        
        <!-- อีเมล -->
        <div class="form-group">
            <label for="email">อีเมล <span style="color:red">*</span></label>
            <input type="email" id="email" name="email" 
                   value="<?= old('email') ?>"
                   class="<?= hasError('email') ? 'error' : '' ?>"
                   placeholder="example@email.com">
            <?php if (hasError('email')): ?>
                <div class="error-msg">⚠️ <?= getError('email') ?></div>
            <?php endif; ?>
        </div>
        
        <!-- เบอร์โทร -->
        <div class="form-group">
            <label for="phone">เบอร์โทรศัพท์</label>
            <input type="tel" id="phone" name="phone" 
                   value="<?= old('phone') ?>"
                   class="<?= hasError('phone') ? 'error' : '' ?>"
                   placeholder="0891234567">
            <?php if (hasError('phone')): ?>
                <div class="error-msg">⚠️ <?= getError('phone') ?></div>
            <?php endif; ?>
        </div>
        
        <!-- หัวข้อ -->
        <div class="form-group">
            <label for="subject">หัวข้อ <span style="color:red">*</span></label>
            <select id="subject" name="subject" class="<?= hasError('subject') ? 'error' : '' ?>">
                <option value="">-- เลือกหัวข้อ --</option>
                <option value="general" <?= old('subject') === 'general' ? 'selected' : '' ?>>ข้อมูลทั่วไป</option>
                <option value="support" <?= old('subject') === 'support' ? 'selected' : '' ?>>ขอความช่วยเหลือ</option>
                <option value="complaint" <?= old('subject') === 'complaint' ? 'selected' : '' ?>>ร้องเรียน</option>
                <option value="feedback" <?= old('subject') === 'feedback' ? 'selected' : '' ?>>ให้คำแนะนำ</option>
            </select>
            <?php if (hasError('subject')): ?>
                <div class="error-msg">⚠️ <?= getError('subject') ?></div>
            <?php endif; ?>
        </div>
        
        <!-- ข้อความ -->
        <div class="form-group">
            <label for="message">ข้อความ <span style="color:red">*</span></label>
            <textarea id="message" name="message" rows="6"
                      class="<?= hasError('message') ? 'error' : '' ?>"
                      placeholder="กรอกข้อความของคุณที่นี่..."
                      maxlength="2000"
                      oninput="updateCharCount()"><?= old('message') ?></textarea>
            <div class="char-count"><span id="charCount">0</span>/2000</div>
            <?php if (hasError('message')): ?>
                <div class="error-msg">⚠️ <?= getError('message') ?></div>
            <?php endif; ?>
        </div>
        
        <!-- แนบไฟล์ -->
        <div class="form-group">
            <label for="attachment">แนบไฟล์ (ไม่บังคับ, PDF/JPG/PNG สูงสุด 5MB)</label>
            <input type="file" id="attachment" name="attachment" 
                   accept=".pdf,.jpg,.jpeg,.png"
                   class="<?= hasError('attachment') ? 'error' : '' ?>">
            <?php if (hasError('attachment')): ?>
                <div class="error-msg">⚠️ <?= getError('attachment') ?></div>
            <?php endif; ?>
        </div>
        
        <button type="submit" class="btn">📤 ส่งข้อความ</button>
    </form>
    
    <script>
    function updateCharCount() {
        const textarea = document.getElementById('message');
        document.getElementById('charCount').textContent = textarea.value.length;
    }
    
    // Client-side validation (เพิ่มเติม)
    document.getElementById('contactForm').addEventListener('submit', function(e) {
        let hasError = false;
        const name = document.getElementById('name').value.trim();
        const email = document.getElementById('email').value.trim();
        
        if (name.length < 2) {
            alert('กรุณากรอกชื่อให้ครบถ้วน');
            hasError = true;
        }
        
        if (hasError) {
            e.preventDefault();
        }
    });
    </script>
</body>
</html>
```

### ไฟล์ process.php

```php
<?php
// process.php
session_start();

// ตรวจสอบ Request Method
if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    header('Location: index.php');
    exit;
}

// ======================================
// CSRF Validation
// ======================================
$token = $_POST['csrf_token'] ?? '';
if (!isset($_SESSION['csrf_token']) || !hash_equals($_SESSION['csrf_token'], $token)) {
    die('❌ Invalid CSRF token');
}

// สร้าง token ใหม่หลังใช้งาน
$_SESSION['csrf_token'] = bin2hex(random_bytes(32));

// ======================================
// Sanitize และ Validate Input
// ======================================
$errors = [];
$data = [];

// ชื่อ-นามสกุล
$name = trim($_POST['name'] ?? '');
if (empty($name)) {
    $errors['name'] = 'กรุณากรอกชื่อ-นามสกุล';
} elseif (strlen($name) < 2 || strlen($name) > 100) {
    $errors['name'] = 'ชื่อต้องมี 2-100 ตัวอักษร';
} else {
    $data['name'] = $name;
}

// อีเมล
$email = trim($_POST['email'] ?? '');
if (empty($email)) {
    $errors['email'] = 'กรุณากรอกอีเมล';
} elseif (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
    $errors['email'] = 'รูปแบบอีเมลไม่ถูกต้อง';
} else {
    $data['email'] = $email;
}

// เบอร์โทร (optional)
$phone = trim($_POST['phone'] ?? '');
if (!empty($phone)) {
    $phone = preg_replace('/[^0-9+\-\s]/', '', $phone);
    if (!preg_match('/^[0-9]{9,10}$/', preg_replace('/[\s\-]/', '', $phone))) {
        $errors['phone'] = 'เบอร์โทรไม่ถูกต้อง';
    } else {
        $data['phone'] = $phone;
    }
}

// หัวข้อ
$subject = $_POST['subject'] ?? '';
$allowedSubjects = ['general', 'support', 'complaint', 'feedback'];
if (!in_array($subject, $allowedSubjects)) {
    $errors['subject'] = 'กรุณาเลือกหัวข้อ';
} else {
    $data['subject'] = $subject;
}

// ข้อความ
$message = trim($_POST['message'] ?? '');
if (empty($message)) {
    $errors['message'] = 'กรุณากรอกข้อความ';
} elseif (strlen($message) < 10) {
    $errors['message'] = 'ข้อความต้องมีอย่างน้อย 10 ตัวอักษร';
} elseif (strlen($message) > 2000) {
    $errors['message'] = 'ข้อความต้องไม่เกิน 2000 ตัวอักษร';
} else {
    $data['message'] = $message;
}

// ไฟล์แนบ (optional)
$attachmentPath = null;
if (isset($_FILES['attachment']) && $_FILES['attachment']['error'] !== UPLOAD_ERR_NO_FILE) {
    $file = $_FILES['attachment'];
    
    if ($file['error'] !== UPLOAD_ERR_OK) {
        $errors['attachment'] = 'เกิดข้อผิดพลาดในการอัปโหลดไฟล์';
    } elseif ($file['size'] > 5 * 1024 * 1024) {
        $errors['attachment'] = 'ไฟล์มีขนาดเกิน 5MB';
    } else {
        $finfo = new finfo(FILEINFO_MIME_TYPE);
        $mimeType = $finfo->file($file['tmp_name']);
        $allowedMimes = ['application/pdf', 'image/jpeg', 'image/png'];
        
        if (!in_array($mimeType, $allowedMimes)) {
            $errors['attachment'] = 'ประเภทไฟล์ไม่ได้รับอนุญาต';
        } else {
            $ext = ['application/pdf' => 'pdf', 'image/jpeg' => 'jpg', 'image/png' => 'png'][$mimeType];
            $filename = 'attachment_' . time() . '_' . bin2hex(random_bytes(8)) . '.' . $ext;
            $uploadDir = 'uploads/attachments/';
            
            if (!is_dir($uploadDir)) {
                mkdir($uploadDir, 0755, true);
            }
            
            if (move_uploaded_file($file['tmp_name'], $uploadDir . $filename)) {
                $attachmentPath = $uploadDir . $filename;
                $data['attachment'] = $filename;
            } else {
                $errors['attachment'] = 'ไม่สามารถบันทึกไฟล์ได้';
            }
        }
    }
}

// ======================================
// มี Error: redirect กลับพร้อมข้อมูล
// ======================================
if (!empty($errors)) {
    $_SESSION['errors'] = $errors;
    $_SESSION['old_input'] = $_POST;
    header('Location: index.php');
    exit;
}

// ======================================
// บันทึกข้อมูลและส่งอีเมล
// ======================================

// จำลองการบันทึกลง Database
$contactData = [
    'id'         => uniqid('contact_'),
    'name'       => $data['name'],
    'email'      => $data['email'],
    'phone'      => $data['phone'] ?? null,
    'subject'    => $data['subject'],
    'message'    => $data['message'],
    'attachment' => $data['attachment'] ?? null,
    'ip'         => $_SERVER['REMOTE_ADDR'],
    'user_agent' => $_SERVER['HTTP_USER_AGENT'],
    'created_at' => date('Y-m-d H:i:s'),
];

// บันทึกลงไฟล์ (ในที่จริงใช้ Database)
$logFile = 'contacts.json';
$contacts = [];
if (file_exists($logFile)) {
    $contacts = json_decode(file_get_contents($logFile), true) ?? [];
}
$contacts[] = $contactData;
file_put_contents($logFile, json_encode($contacts, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE));

// จำลองการส่งอีเมล
// mail($data['email'], 'ขอบคุณที่ติดต่อเรา', "เราได้รับข้อความของคุณแล้ว...");

$_SESSION['success'] = "ขอบคุณ {$data['name']}! เราได้รับข้อความของคุณแล้ว และจะติดต่อกลับภายใน 24 ชั่วโมง";
header('Location: index.php');
exit;
?>
```

---

## 📝 Quiz

### คำถาม

**ข้อ 1:** ข้อใดคือวิธีที่ถูกต้องในการป้องกัน XSS ใน PHP?
- A. `mysql_real_escape_string($_POST['input'])`
- B. `htmlspecialchars($_POST['input'], ENT_QUOTES, 'UTF-8')`
- C. `strip_tags($_POST['input'])`
- D. `addslashes($_POST['input'])`

**ข้อ 2:** ฟังก์ชันใดใช้ตรวจสอบว่าเป็นไฟล์ที่ถูก upload จริงๆ?
- A. `file_exists()`
- B. `is_file()`
- C. `is_uploaded_file()`
- D. `upload_file_exists()`

**ข้อ 3:** CSRF Token ต้องเปรียบเทียบด้วยฟังก์ชันใดเพื่อป้องกัน timing attack?
- A. `strcmp()`
- B. `==`
- C. `hash_equals()`
- D. `md5_compare()`

**ข้อ 4:** `enctype="multipart/form-data"` จำเป็นต้องใส่เมื่อใด?
- A. ทุกครั้งที่ใช้ method="post"
- B. เฉพาะเมื่อ form มี file input
- C. เฉพาะเมื่อ form มีข้อมูลมาก
- D. ไม่จำเป็นต้องใส่

**ข้อ 5:** วิธีตรวจสอบ MIME type ที่ปลอดภัยที่สุดคือ?
- A. ดูจาก `$_FILES['file']['type']`
- B. ดูจาก extension ของไฟล์
- C. ใช้ `finfo` extension
- D. ทั้ง A และ B

### เฉลย

| ข้อ | คำตอบ | อธิบาย |
|-----|-------|--------|
| 1 | B | `htmlspecialchars()` แปลง HTML special chars ป้องกัน XSS |
| 2 | C | `is_uploaded_file()` ตรวจสอบว่าไฟล์ถูก upload ผ่าน HTTP POST จริงๆ |
| 3 | C | `hash_equals()` ป้องกัน timing attack โดยใช้เวลาคงที่ไม่ว่า string จะต่างกันตรงไหน |
| 4 | B | `enctype="multipart/form-data"` จำเป็นเฉพาะ form ที่มี `<input type="file">` |
| 5 | C | `finfo` อ่าน magic bytes จริงๆ ในไฟล์ ไม่สามารถปลอมแปลงได้ง่าย |

---

## 🔗 แหล่งข้อมูลเพิ่มเติม

- [PHP Manual: $_FILES](https://www.php.net/manual/en/reserved.variables.files.php)
- [PHP Manual: filter_var](https://www.php.net/manual/en/function.filter-var.php)
- [OWASP: CSRF Prevention](https://owasp.org/www-community/attacks/csrf)

---

## ➡️ Part ถัดไป

**[Part 10: PHP File Handling - การจัดการไฟล์](part-010-php-file-handling.md)**

ใน Part ถัดไปเราจะเรียนรู้เกี่ยวกับ:
- การอ่านและเขียนไฟล์
- File system functions
- Directory operations
- การทำงานกับ CSV และ JSON
- Workshop: สร้าง File Manager
