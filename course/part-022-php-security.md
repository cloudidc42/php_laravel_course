# Part 022: PHP Security

## ระดับ: Advanced
## เวลาเรียน: 4-5 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:
- ป้องกัน XSS (Cross-Site Scripting)
- ป้องกัน SQL Injection
- Implement CSRF Protection
- ใช้ password_hash() และ Argon2 ได้ถูกต้อง
- Validate และ Sanitize input
- สร้าง Secure Login System

---

## 1. XSS Prevention

### ทำความเข้าใจ XSS

XSS (Cross-Site Scripting) คือการที่ผู้โจมตีแทรก JavaScript ลงในหน้าเว็บ

```php
<?php
// ช่องโหว่ XSS - อย่าทำแบบนี้!
$name = $_GET['name'] ?? '';
echo "สวัสดี, {$name}!"; // ถ้า name = <script>alert('XSS')</script> จะรัน!

// URL attack:
// https://example.com/greet.php?name=<script>document.cookie</script>
```

### htmlspecialchars() - วิธีหลักในการป้องกัน XSS

```php
<?php
// htmlspecialchars แปลง special characters เป็น HTML entities
$userInput = '<script>alert("XSS")</script>';
$safe = htmlspecialchars($userInput, ENT_QUOTES | ENT_HTML5, 'UTF-8');
echo $safe; // &lt;script&gt;alert(&quot;XSS&quot;)&lt;/script&gt;

// ENT_QUOTES - แปลง " และ ' ด้วย
// ENT_HTML5 - ใช้ HTML5 entities
// UTF-8 - charset สำคัญมาก!

// วิธีเขียนให้ถูกต้อง
function e(string $str): string {
    return htmlspecialchars($str, ENT_QUOTES | ENT_HTML5, 'UTF-8');
}

// ใช้งาน
$name = $_GET['name'] ?? '';
echo "สวัสดี, " . e($name) . "!";

// ใน template
?>
<div class="user-info">
    <h1><?= e($user['name']) ?></h1>
    <p><?= e($user['bio']) ?></p>
    <a href="<?= e($user['website']) ?>">Website</a>
</div>
```

### Content Security Policy (CSP)

```php
<?php
// ส่ง CSP header เพื่อบอก browser ว่าโหลด resources จากที่ไหนได้
function setSecurityHeaders(): void {
    // Content Security Policy
    $csp = implode('; ', [
        "default-src 'self'",
        "script-src 'self' https://cdnjs.cloudflare.com",
        "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
        "img-src 'self' data: https:",
        "font-src 'self' https://fonts.gstatic.com",
        "connect-src 'self' https://api.example.com",
        "frame-ancestors 'none'",
        "form-action 'self'",
    ]);
    
    header("Content-Security-Policy: {$csp}");
    header("X-Content-Type-Options: nosniff");
    header("X-Frame-Options: DENY");
    header("X-XSS-Protection: 1; mode=block");
    header("Referrer-Policy: strict-origin-when-cross-origin");
    header("Permissions-Policy: camera=(), microphone=(), geolocation=()");
    
    // HSTS (HTTPS only)
    if (isset($_SERVER['HTTPS'])) {
        header("Strict-Transport-Security: max-age=31536000; includeSubDomains; preload");
    }
}

setSecurityHeaders();
```

### XSS ในแอตทริบิวต์ HTML

```php
<?php
// อันตราย: ค่าใน attribute
$color = $_GET['color'] ?? 'blue';
echo "<div style='color: {$color}'>"; // XSS ได้ถ้า color = red; background: url(javascript:...)

// ปลอดภัย: validate ก่อน
$allowedColors = ['red', 'blue', 'green', 'black', 'white'];
$color = in_array($_GET['color'] ?? '', $allowedColors) ? $_GET['color'] : 'blue';
echo "<div style='color: " . e($color) . "'>";

// หรือใช้ htmlspecialchars สำหรับ attribute values ด้วย
function attr(string $value): string {
    return htmlspecialchars($value, ENT_QUOTES | ENT_HTML5, 'UTF-8');
}

// สำหรับ URL attributes ต้อง validate ด้วย
function safeUrl(string $url): string {
    // อนุญาตเฉพาะ http:// และ https://
    if (!preg_match('/^https?:\/\//', $url)) {
        return '#'; // default safe value
    }
    return htmlspecialchars($url, ENT_QUOTES, 'UTF-8');
}

$link = $_GET['url'] ?? '';
echo '<a href="' . safeUrl($link) . '">Click</a>';
```

---

## 2. SQL Injection Prevention

### ทำความเข้าใจ SQL Injection

```php
<?php
// ช่องโหว่ SQL Injection - อย่าทำแบบนี้!
$username = $_POST['username']; // "admin' --"
$password = $_POST['password'];

// query ที่ถูก inject:
// SELECT * FROM users WHERE username = 'admin' -- ' AND password = '...'
$query = "SELECT * FROM users WHERE username = '{$username}' AND password = '{$password}'";
// ผู้โจมตีสามารถ login ได้โดยไม่รู้ password!
```

### PDO Prepared Statements

```php
<?php
// วิธีที่ถูกต้อง: ใช้ Prepared Statements
class Database {
    private static ?PDO $instance = null;
    
    public static function getInstance(): PDO {
        if (self::$instance === null) {
            self::$instance = new PDO(
                'mysql:host=localhost;dbname=mydb;charset=utf8mb4',
                'username',
                'password',
                [
                    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
                    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
                    PDO::ATTR_EMULATE_PREPARES => false, // ปิด emulation!
                    PDO::MYSQL_ATTR_FOUND_ROWS => true,
                ]
            );
        }
        return self::$instance;
    }
}

class UserRepository {
    private PDO $db;
    
    public function __construct() {
        $this->db = Database::getInstance();
    }
    
    // Prepared statement พื้นฐาน
    public function findByCredentials(string $username, string $password): ?array {
        $stmt = $this->db->prepare(
            'SELECT id, username, password_hash, role, active 
             FROM users 
             WHERE username = :username AND active = 1
             LIMIT 1'
        );
        
        $stmt->execute([':username' => $username]);
        $user = $stmt->fetch();
        
        if (!$user) return null;
        
        // ตรวจสอบ password hash (ไม่เปรียบเทียบ hash โดยตรง!)
        if (!password_verify($password, $user['password_hash'])) {
            return null;
        }
        
        unset($user['password_hash']); // ลบ hash ออก
        return $user;
    }
    
    // Dynamic queries อย่างปลอดภัย
    public function search(array $filters, string $orderBy = 'id', string $direction = 'ASC'): array {
        // Whitelist สำหรับ column names (ป้องกัน injection ใน ORDER BY)
        $allowedColumns = ['id', 'username', 'email', 'created_at'];
        $allowedDirections = ['ASC', 'DESC'];
        
        if (!in_array($orderBy, $allowedColumns)) {
            $orderBy = 'id';
        }
        
        if (!in_array(strtoupper($direction), $allowedDirections)) {
            $direction = 'ASC';
        }
        
        $where = ['1=1'];
        $params = [];
        
        if (!empty($filters['username'])) {
            $where[] = 'username LIKE :username';
            $params[':username'] = '%' . $filters['username'] . '%';
        }
        
        if (!empty($filters['role'])) {
            $where[] = 'role = :role';
            $params[':role'] = $filters['role'];
        }
        
        if (!empty($filters['active'])) {
            $where[] = 'active = :active';
            $params[':active'] = (int) $filters['active'];
        }
        
        $sql = "SELECT id, username, email, role, active 
                FROM users 
                WHERE " . implode(' AND ', $where) . "
                ORDER BY {$orderBy} {$direction}"; // ปลอดภัยเพราะ whitelist แล้ว
        
        $stmt = $this->db->prepare($sql);
        $stmt->execute($params);
        return $stmt->fetchAll();
    }
    
    // Bulk insert อย่างปลอดภัย
    public function bulkInsert(array $users): int {
        $placeholders = implode(', ', array_fill(0, count($users), '(?, ?)'));
        $sql = "INSERT INTO users (username, email) VALUES {$placeholders}";
        
        $params = [];
        foreach ($users as $user) {
            $params[] = $user['username'];
            $params[] = $user['email'];
        }
        
        $stmt = $this->db->prepare($sql);
        $stmt->execute($params);
        return $stmt->rowCount();
    }
    
    // Transaction
    public function createWithProfile(array $userData, array $profileData): int {
        $this->db->beginTransaction();
        
        try {
            $stmt = $this->db->prepare(
                'INSERT INTO users (username, email, password_hash) VALUES (:username, :email, :hash)'
            );
            $stmt->execute([
                ':username' => $userData['username'],
                ':email' => $userData['email'],
                ':hash' => password_hash($userData['password'], PASSWORD_ARGON2ID),
            ]);
            
            $userId = (int) $this->db->lastInsertId();
            
            $stmt2 = $this->db->prepare(
                'INSERT INTO user_profiles (user_id, bio, avatar) VALUES (:uid, :bio, :avatar)'
            );
            $stmt2->execute([
                ':uid' => $userId,
                ':bio' => $profileData['bio'] ?? '',
                ':avatar' => $profileData['avatar'] ?? '',
            ]);
            
            $this->db->commit();
            return $userId;
            
        } catch (\Throwable $e) {
            $this->db->rollBack();
            throw new \RuntimeException("Failed to create user: " . $e->getMessage(), 0, $e);
        }
    }
}
```

---

## 3. CSRF Protection

```php
<?php
class CsrfProtection {
    private string $sessionKey = '_csrf_tokens';
    private int $tokenLifetime = 3600; // 1 hour
    private int $maxTokens = 20;
    
    public function __construct() {
        if (session_status() === PHP_SESSION_NONE) {
            session_start();
        }
    }
    
    public function generateToken(string $formName = 'default'): string {
        $token = bin2hex(random_bytes(32)); // 64 chars, cryptographically secure
        
        if (!isset($_SESSION[$this->sessionKey])) {
            $_SESSION[$this->sessionKey] = [];
        }
        
        // Cleanup old tokens
        $this->cleanup();
        
        $_SESSION[$this->sessionKey][$formName] = [
            'token' => $token,
            'expires' => time() + $this->tokenLifetime,
        ];
        
        return $token;
    }
    
    public function validateToken(string $token, string $formName = 'default'): bool {
        if (empty($token)) return false;
        
        $stored = $_SESSION[$this->sessionKey][$formName] ?? null;
        if (!$stored) return false;
        
        // ตรวจสอบ expiry
        if (time() > $stored['expires']) {
            unset($_SESSION[$this->sessionKey][$formName]);
            return false;
        }
        
        // ใช้ hash_equals เพื่อป้องกัน timing attack
        if (!hash_equals($stored['token'], $token)) {
            return false;
        }
        
        // One-time token: ลบหลังใช้
        unset($_SESSION[$this->sessionKey][$formName]);
        return true;
    }
    
    public function getHiddenInput(string $formName = 'default'): string {
        $token = $this->generateToken($formName);
        return '<input type="hidden" name="_csrf_token" value="' . htmlspecialchars($token) . '">';
    }
    
    public function getMetaTag(string $formName = 'default'): string {
        $token = $this->generateToken($formName);
        return '<meta name="csrf-token" content="' . htmlspecialchars($token) . '">';
    }
    
    private function cleanup(): void {
        if (!isset($_SESSION[$this->sessionKey])) return;
        
        // ลบ token ที่หมดอายุ
        foreach ($_SESSION[$this->sessionKey] as $key => $data) {
            if (time() > $data['expires']) {
                unset($_SESSION[$this->sessionKey][$key]);
            }
        }
        
        // จำกัดจำนวน token
        while (count($_SESSION[$this->sessionKey]) >= $this->maxTokens) {
            array_shift($_SESSION[$this->sessionKey]);
        }
    }
    
    // Middleware สำหรับ API
    public function verifyRequestToken(): bool {
        // Check from header (for AJAX)
        $headerToken = $_SERVER['HTTP_X_CSRF_TOKEN'] ?? '';
        if (!empty($headerToken) && $this->validateToken($headerToken, 'api')) {
            return true;
        }
        
        // Check from POST body
        $postToken = $_POST['_csrf_token'] ?? '';
        return !empty($postToken) && $this->validateToken($postToken);
    }
}

// ใช้งานใน form
$csrf = new CsrfProtection();
?>
<form method="POST" action="/submit">
    <?= $csrf->getHiddenInput('register') ?>
    <input type="text" name="username">
    <button type="submit">Register</button>
</form>

<?php
// ตรวจสอบใน handler
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $token = $_POST['_csrf_token'] ?? '';
    
    if (!$csrf->validateToken($token, 'register')) {
        http_response_code(403);
        die('CSRF validation failed!');
    }
    
    // ดำเนินการต่อ...
}
?>
```

---

## 4. Password Hashing

### password_hash() และ Argon2

```php
<?php
class PasswordManager {
    
    // Argon2id - แนะนำที่สุดสำหรับ PHP 8+
    public static function hash(string $password): string {
        return password_hash($password, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536,  // 64 MB
            'time_cost'   => 4,      // iterations
            'threads'     => 1,      // parallel threads
        ]);
    }
    
    // Bcrypt - fallback ถ้า Argon2 ไม่ available
    public static function hashBcrypt(string $password): string {
        return password_hash($password, PASSWORD_BCRYPT, [
            'cost' => 12, // 2^12 iterations (ค่า default คือ 10)
        ]);
    }
    
    public static function verify(string $password, string $hash): bool {
        return password_verify($password, $hash);
    }
    
    // ตรวจสอบว่า hash ต้องการ rehash หรือไม่
    public static function needsRehash(string $hash): bool {
        return password_needs_rehash($hash, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536,
            'time_cost'   => 4,
            'threads'     => 1,
        ]);
    }
    
    public static function getInfo(string $hash): array {
        return password_get_info($hash);
    }
    
    // ตรวจสอบความแข็งแกร่งของ password
    public static function validateStrength(string $password): array {
        $errors = [];
        
        if (strlen($password) < 8) {
            $errors[] = "ต้องมีความยาวอย่างน้อย 8 ตัวอักษร";
        }
        
        if (!preg_match('/[A-Z]/', $password)) {
            $errors[] = "ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว";
        }
        
        if (!preg_match('/[a-z]/', $password)) {
            $errors[] = "ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว";
        }
        
        if (!preg_match('/[0-9]/', $password)) {
            $errors[] = "ต้องมีตัวเลขอย่างน้อย 1 ตัว";
        }
        
        if (!preg_match('/[@$!%*?&]/', $password)) {
            $errors[] = "ต้องมีอักขระพิเศษอย่างน้อย 1 ตัว (@$!%*?&)";
        }
        
        // ตรวจสอบ common passwords
        $commonPasswords = ['password', '12345678', 'qwerty123', 'password1'];
        if (in_array(strtolower($password), $commonPasswords)) {
            $errors[] = "รหัสผ่านนี้ง่ายเกินไป";
        }
        
        return $errors;
    }
}

// ทดสอบ
$password = "MySecure@Pass1";
$hash = PasswordManager::hash($password);

echo "Hash: {$hash}\n";
echo "Verify: " . (PasswordManager::verify($password, $hash) ? "Valid" : "Invalid") . "\n";
echo "Wrong password: " . (PasswordManager::verify("wrong", $hash) ? "Valid" : "Invalid") . "\n";
echo "Needs rehash: " . (PasswordManager::needsRehash($hash) ? "Yes" : "No") . "\n";

$info = PasswordManager::getInfo($hash);
echo "Algorithm: " . $info['algoName'] . "\n";

// ตรวจสอบความแข็งแกร่ง
$weakPassword = "pass";
$errors = PasswordManager::validateStrength($weakPassword);
if (!empty($errors)) {
    echo "Weak password errors:\n";
    foreach ($errors as $error) {
        echo "  - {$error}\n";
    }
}
```

---

## 5. Input Validation & Sanitization

```php
<?php
class InputSanitizer {
    
    // Sanitize text input
    public static function text(string $input, int $maxLength = 255): string {
        // Strip null bytes
        $input = str_replace("\0", '', $input);
        
        // Normalize whitespace
        $input = preg_replace('/\s+/', ' ', $input);
        $input = trim($input);
        
        // Limit length
        if (mb_strlen($input) > $maxLength) {
            $input = mb_substr($input, 0, $maxLength);
        }
        
        return $input;
    }
    
    // Sanitize HTML content (allow safe tags)
    public static function html(string $input): string {
        // แนะนำใช้ HTML Purifier library แทน
        $allowedTags = '<p><br><b><i><strong><em><ul><ol><li><h1><h2><h3><a>';
        $clean = strip_tags($input, $allowedTags);
        
        // Remove event handlers from allowed tags
        $clean = preg_replace('/\s+on\w+="[^"]*"/', '', $clean);
        $clean = preg_replace("/\s+on\w+='[^']*'/", '', $clean);
        $clean = preg_replace('/\s+on\w+=\w+/', '', $clean);
        
        // Remove javascript: in href
        $clean = preg_replace('/href\s*=\s*["\']?\s*javascript:/i', 'href="#"', $clean);
        
        return $clean;
    }
    
    // Sanitize integer
    public static function int(mixed $input, int $min = PHP_INT_MIN, int $max = PHP_INT_MAX): ?int {
        $filtered = filter_var($input, FILTER_VALIDATE_INT, [
            'options' => ['min_range' => $min, 'max_range' => $max],
        ]);
        return $filtered !== false ? $filtered : null;
    }
    
    // Sanitize float
    public static function float(mixed $input): ?float {
        $filtered = filter_var($input, FILTER_VALIDATE_FLOAT);
        return $filtered !== false ? $filtered : null;
    }
    
    // Sanitize email
    public static function email(string $input): ?string {
        $filtered = filter_var(strtolower(trim($input)), FILTER_VALIDATE_EMAIL);
        return $filtered ?: null;
    }
    
    // Sanitize URL
    public static function url(string $input): ?string {
        $filtered = filter_var(trim($input), FILTER_VALIDATE_URL);
        if (!$filtered) return null;
        
        // อนุญาตเฉพาะ http/https
        $scheme = parse_url($filtered, PHP_URL_SCHEME);
        if (!in_array($scheme, ['http', 'https'])) return null;
        
        return $filtered;
    }
    
    // Sanitize filename
    public static function filename(string $input): string {
        // ลบ path traversal
        $input = basename($input);
        
        // ลบ characters อันตราย
        $input = preg_replace('/[^a-zA-Z0-9._\-]/', '_', $input);
        
        // ลบ leading dots (hidden files)
        $input = ltrim($input, '.');
        
        // จำกัดความยาว
        if (strlen($input) > 255) {
            $ext = pathinfo($input, PATHINFO_EXTENSION);
            $name = pathinfo($input, PATHINFO_FILENAME);
            $input = substr($name, 0, 250 - strlen($ext)) . '.' . $ext;
        }
        
        return $input ?: 'file';
    }
    
    // Sanitize array recursively
    public static function array(array $input, string $type = 'text'): array {
        array_walk_recursive($input, function(&$value) use ($type) {
            if (is_string($value)) {
                $value = match($type) {
                    'email' => self::email($value) ?? $value,
                    'url' => self::url($value) ?? $value,
                    default => self::text($value),
                };
            }
        });
        return $input;
    }
    
    // Mask sensitive data สำหรับ logging
    public static function maskSensitive(array $data, array $sensitiveKeys = []): array {
        $defaultSensitive = ['password', 'password_confirmation', 'token', 'secret', 'key', 'credit_card'];
        $sensitive = array_merge($defaultSensitive, $sensitiveKeys);
        
        array_walk_recursive($data, function(&$value, $key) use ($sensitive) {
            if (in_array(strtolower($key), $sensitive) && is_string($value)) {
                $value = '***';
            }
        });
        
        return $data;
    }
}

// ทดสอบ
$userInput = [
    'name' => '  <script>alert("xss")</script>สมชาย  ',
    'email' => '  SOMCHAI@EXAMPLE.COM  ',
    'age' => '25abc',
    'website' => 'javascript:alert(1)',
    'password' => 'Secret123!',
];

echo "Name: " . InputSanitizer::text($userInput['name']) . "\n";
// ไม่มี script tag, trim แล้ว

echo "Email: " . (InputSanitizer::email($userInput['email']) ?? 'Invalid') . "\n";
// somchai@example.com

echo "Age: " . (InputSanitizer::int($userInput['age'], 1, 120) ?? 'Invalid') . "\n";
// null (invalid)

echo "Website: " . (InputSanitizer::url($userInput['website']) ?? 'Invalid') . "\n";
// null (javascript: not allowed)

// Mask for logging
$safeLog = InputSanitizer::maskSensitive($userInput);
echo "Safe log: " . json_encode($safeLog, JSON_UNESCAPED_UNICODE) . "\n";
// password จะเป็น ***
```

---

## 6. Workshop: Secure Login System

```php
<?php
// ============================================================
// Secure Login System - สมบูรณ์แบบ
// ============================================================

class SecureAuth {
    private PDO $db;
    private CsrfProtection $csrf;
    private int $maxLoginAttempts = 5;
    private int $lockoutDuration = 900; // 15 minutes
    
    public function __construct() {
        $this->db = Database::getInstance();
        $this->csrf = new CsrfProtection();
        
        if (session_status() === PHP_SESSION_NONE) {
            $this->startSecureSession();
        }
    }
    
    private function startSecureSession(): void {
        // Session security settings
        ini_set('session.cookie_httponly', '1');
        ini_set('session.cookie_secure', '1');   // HTTPS only
        ini_set('session.cookie_samesite', 'Strict');
        ini_set('session.use_strict_mode', '1');
        ini_set('session.use_only_cookies', '1');
        
        session_start();
        
        // Regenerate session ID เพื่อป้องกัน session fixation
        if (!isset($_SESSION['_initialized'])) {
            session_regenerate_id(true);
            $_SESSION['_initialized'] = true;
        }
    }
    
    public function login(string $username, string $password, string $csrfToken): array {
        // 1. CSRF Check
        if (!$this->csrf->validateToken($csrfToken, 'login')) {
            $this->logSecurityEvent('csrf_failure', ['username' => $username]);
            return ['success' => false, 'message' => 'Invalid request token'];
        }
        
        // 2. Input Validation
        $username = InputSanitizer::text($username, 50);
        if (empty($username) || empty($password)) {
            return ['success' => false, 'message' => 'กรุณากรอก username และ password'];
        }
        
        // 3. Rate Limiting - ตรวจสอบ brute force
        $ip = $this->getClientIp();
        if ($this->isRateLimited($username, $ip)) {
            $this->logSecurityEvent('rate_limit', ['username' => $username, 'ip' => $ip]);
            return [
                'success' => false,
                'message' => 'บัญชีถูกล็อคชั่วคราว กรุณารอ 15 นาที',
            ];
        }
        
        // 4. Find user
        $user = $this->findUser($username);
        
        // 5. Verify password (ใช้เวลาคงที่เพื่อป้องกัน timing attack)
        if (!$user || !$this->verifyPassword($password, $user['password_hash'])) {
            $this->recordFailedAttempt($username, $ip);
            $this->logSecurityEvent('login_failure', ['username' => $username, 'ip' => $ip]);
            
            // อย่า reveal ว่า username ผิดหรือ password ผิด
            return ['success' => false, 'message' => 'Username หรือ Password ไม่ถูกต้อง'];
        }
        
        // 6. ตรวจสอบสถานะบัญชี
        if (!$user['active']) {
            return ['success' => false, 'message' => 'บัญชีนี้ถูกปิดการใช้งาน'];
        }
        
        if ($user['email_verified_at'] === null) {
            return ['success' => false, 'message' => 'กรุณายืนยัน email ก่อน login'];
        }
        
        // 7. Rehash password ถ้าจำเป็น
        if (password_needs_rehash($user['password_hash'], PASSWORD_ARGON2ID)) {
            $this->updatePasswordHash($user['id'], $password);
        }
        
        // 8. สร้าง session ที่ปลอดภัย
        session_regenerate_id(true); // ป้องกัน session fixation
        
        $_SESSION['user_id'] = $user['id'];
        $_SESSION['username'] = $user['username'];
        $_SESSION['role'] = $user['role'];
        $_SESSION['logged_in_at'] = time();
        $_SESSION['ip'] = $ip;
        $_SESSION['user_agent'] = $_SERVER['HTTP_USER_AGENT'] ?? '';
        
        // 9. Clear failed attempts
        $this->clearFailedAttempts($username, $ip);
        
        // 10. Log success
        $this->logSecurityEvent('login_success', [
            'user_id' => $user['id'],
            'username' => $user['username'],
            'ip' => $ip,
        ]);
        
        // 11. Update last login
        $this->updateLastLogin($user['id']);
        
        return [
            'success' => true,
            'message' => 'เข้าสู่ระบบสำเร็จ',
            'user' => [
                'id' => $user['id'],
                'username' => $user['username'],
                'role' => $user['role'],
            ],
        ];
    }
    
    public function logout(): void {
        $this->logSecurityEvent('logout', [
            'user_id' => $_SESSION['user_id'] ?? null,
        ]);
        
        // ลบข้อมูลทั้งหมดใน session
        $_SESSION = [];
        
        // ลบ session cookie
        if (ini_get('session.use_cookies')) {
            $params = session_get_cookie_params();
            setcookie(
                session_name(),
                '',
                time() - 42000,
                $params['path'],
                $params['domain'],
                $params['secure'],
                $params['httponly']
            );
        }
        
        session_destroy();
    }
    
    public function isAuthenticated(): bool {
        if (!isset($_SESSION['user_id'], $_SESSION['logged_in_at'])) {
            return false;
        }
        
        // ตรวจสอบ session timeout (30 นาที)
        if (time() - $_SESSION['logged_in_at'] > 1800) {
            $this->logout();
            return false;
        }
        
        // ตรวจสอบ IP ไม่เปลี่ยน (optional - อาจเป็นปัญหากับ proxy)
        if ($_SESSION['ip'] !== $this->getClientIp()) {
            $this->logSecurityEvent('ip_mismatch', [
                'user_id' => $_SESSION['user_id'],
                'session_ip' => $_SESSION['ip'],
                'current_ip' => $this->getClientIp(),
            ]);
            $this->logout();
            return false;
        }
        
        // Refresh timeout
        $_SESSION['logged_in_at'] = time();
        
        return true;
    }
    
    public function requireAuth(): void {
        if (!$this->isAuthenticated()) {
            header('Location: /login?redirect=' . urlencode($_SERVER['REQUEST_URI']));
            exit;
        }
    }
    
    public function requireRole(string ...$roles): void {
        $this->requireAuth();
        
        if (!in_array($_SESSION['role'], $roles)) {
            http_response_code(403);
            die('ไม่มีสิทธิ์เข้าถึงหน้านี้');
        }
    }
    
    private function findUser(string $username): ?array {
        $stmt = $this->db->prepare(
            'SELECT id, username, password_hash, role, active, email_verified_at 
             FROM users 
             WHERE username = :username
             LIMIT 1'
        );
        $stmt->execute([':username' => $username]);
        return $stmt->fetch() ?: null;
    }
    
    private function verifyPassword(string $password, string $hash): bool {
        // password_verify ใช้เวลาคงที่ (constant time)
        return password_verify($password, $hash);
    }
    
    private function isRateLimited(string $username, string $ip): bool {
        $stmt = $this->db->prepare(
            'SELECT COUNT(*) as attempts 
             FROM login_attempts 
             WHERE (username = :username OR ip = :ip)
               AND attempted_at > :since
               AND success = 0'
        );
        $stmt->execute([
            ':username' => $username,
            ':ip' => $ip,
            ':since' => date('Y-m-d H:i:s', time() - $this->lockoutDuration),
        ]);
        
        $result = $stmt->fetch();
        return (int) $result['attempts'] >= $this->maxLoginAttempts;
    }
    
    private function recordFailedAttempt(string $username, string $ip): void {
        $stmt = $this->db->prepare(
            'INSERT INTO login_attempts (username, ip, attempted_at, success) 
             VALUES (:username, :ip, NOW(), 0)'
        );
        $stmt->execute([':username' => $username, ':ip' => $ip]);
    }
    
    private function clearFailedAttempts(string $username, string $ip): void {
        $stmt = $this->db->prepare(
            'DELETE FROM login_attempts WHERE username = :username OR ip = :ip'
        );
        $stmt->execute([':username' => $username, ':ip' => $ip]);
    }
    
    private function updateLastLogin(int $userId): void {
        $stmt = $this->db->prepare(
            'UPDATE users SET last_login_at = NOW() WHERE id = :id'
        );
        $stmt->execute([':id' => $userId]);
    }
    
    private function updatePasswordHash(int $userId, string $password): void {
        $stmt = $this->db->prepare(
            'UPDATE users SET password_hash = :hash WHERE id = :id'
        );
        $stmt->execute([
            ':hash' => PasswordManager::hash($password),
            ':id' => $userId,
        ]);
    }
    
    private function getClientIp(): string {
        // ระวัง: HTTP headers สามารถ spoof ได้!
        // ใช้ REMOTE_ADDR เป็น primary source
        return $_SERVER['REMOTE_ADDR'] ?? '0.0.0.0';
    }
    
    private function logSecurityEvent(string $event, array $context = []): void {
        // Log to database หรือ file
        $stmt = $this->db->prepare(
            'INSERT INTO security_logs (event, context, ip, user_agent, created_at) 
             VALUES (:event, :context, :ip, :agent, NOW())'
        );
        $stmt->execute([
            ':event' => $event,
            ':context' => json_encode($context),
            ':ip' => $this->getClientIp(),
            ':agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        ]);
    }
}

// SQL สำหรับสร้าง tables
$sql = "
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role ENUM('user', 'admin', 'moderator') DEFAULT 'user',
    active TINYINT(1) DEFAULT 1,
    email_verified_at DATETIME NULL,
    last_login_at DATETIME NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS login_attempts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    ip VARCHAR(45) NOT NULL,
    attempted_at DATETIME NOT NULL,
    success TINYINT(1) DEFAULT 0,
    INDEX idx_username (username),
    INDEX idx_ip (ip),
    INDEX idx_attempted (attempted_at)
);

CREATE TABLE IF NOT EXISTS security_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    event VARCHAR(50) NOT NULL,
    context JSON,
    ip VARCHAR(45),
    user_agent VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_event (event),
    INDEX idx_created (created_at)
);
";

echo "Schema ready!\n";
echo "Security tables: users, login_attempts, security_logs\n";
```

---

## Quiz

### คำถาม 1
ทำไมต้องใช้ `hash_equals()` แทน `===` เมื่อเปรียบเทียบ token?
- A. hash_equals เร็วกว่า
- B. ป้องกัน timing attack ที่ผู้โจมตีวัดเวลาเพื่อทายค่า
- C. hash_equals รองรับ Unicode
- D. ไม่มีความแตกต่าง

**เฉลย: B** - การเปรียบเทียบแบบ `===` อาจ return เร็วขึ้นถ้า characters แรกต่างกัน ทำให้ผู้โจมตีวัดเวลาเพื่อ brute force ได้

### คำถาม 2
PDO::ATTR_EMULATE_PREPARES => false มีความสำคัญอย่างไร?
- A. เร็วกว่า
- B. บังคับให้ใช้ native prepared statements จริงๆ ไม่ใช่ emulation ที่อาจมีช่องโหว่
- C. รองรับ character set มากขึ้น
- D. ไม่มีผล

**เฉลย: B** - Emulation อาจมีช่องโหว่บางกรณี ควรปิดเพื่อความปลอดภัยสูงสุด

### คำถาม 3
PASSWORD_ARGON2ID ดีกว่า PASSWORD_BCRYPT อย่างไร?

**เฉลย:** Argon2id ป้องกัน GPU attacks และ side-channel attacks ได้ดีกว่า เพราะ:
- ใช้ memory มาก (memory-hard) ทำให้ GPU/ASIC crackers ทำงานได้ยาก
- มี time cost และ parallelism ที่ปรับได้
- Bcrypt เป็น CPU-bound เท่านั้น ทำให้ GPU crack ได้เร็วกว่า

### คำถาม 4
CSRF token ทำไมต้องสร้างใหม่ทุกครั้งที่ render form?

**เฉลย:** ถ้าใช้ token เดิมซ้ำๆ ผู้โจมตีอาจขโมย token ได้ (เช่น ผ่าน XSS หรือ browser history) และใช้ได้ไปเรื่อยๆ การสร้าง token ใหม่ทุกครั้งทำให้ token มีอายุสั้น และ one-time use ป้องกัน replay attacks ได้

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- **XSS Prevention** - htmlspecialchars, CSP headers, safe URL handling
- **SQL Injection** - Prepared statements, PDO, dynamic queries อย่างปลอดภัย
- **CSRF Protection** - Token generation/validation, one-time tokens
- **Password Hashing** - Argon2id, bcrypt, rehashing, password strength
- **Input Sanitization** - text, email, URL, filename sanitization
- **Secure Login** - Rate limiting, session security, timing attack prevention

---

## ➡️ Part ถัดไป

[Part 023: PHP Testing ด้วย PHPUnit](./part-023-php-testing.md)
