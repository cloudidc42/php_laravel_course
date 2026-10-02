# 🔐 Part 11: PHP Sessions & Cookies - การจัดการ Session และ Cookie

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจความแตกต่างระหว่าง Session และ Cookie
- ใช้งาน PHP Sessions ได้อย่างถูกต้องและปลอดภัย
- จัดการ Cookies ได้
- ทำ Session Security ได้
- ใช้ Flash Messages ได้
- สร้างระบบ Authentication ด้วย Sessions

---

## 📌 1. Sessions

### 1.1 พื้นฐาน Session

Session คือกลไกที่ PHP ใช้เก็บข้อมูลผู้ใช้ไว้ที่ **server** ระหว่าง requests

```
[Client Browser] ←──── HTTP (stateless) ────→ [PHP Server]
       │                                              │
       │  Request + Session Cookie (PHPSESSID)       │
       └─────────────────────────────────────────────►│
                                                      │
                                              ดึงข้อมูลจาก
                                              $_SESSION['user']
                                                      │
       │◄─────────────────────────────────────────────┘
       Response + Set-Cookie: PHPSESSID=abc123
```

```php
<?php
// sessions-basic.php

// ต้องเรียก session_start() ก่อนใช้ $_SESSION
// และต้องเรียกก่อนส่ง output ใดๆ
session_start();

// ======================================
// เขียนข้อมูลลง Session
// ======================================
$_SESSION['username'] = 'สมชาย';
$_SESSION['user_id'] = 42;
$_SESSION['is_logged_in'] = true;
$_SESSION['role'] = 'admin';
$_SESSION['login_time'] = time();

// เก็บ array ใน session
$_SESSION['cart'] = [
    ['product_id' => 1, 'name' => 'MacBook', 'qty' => 1, 'price' => 89000],
    ['product_id' => 2, 'name' => 'iPhone', 'qty' => 2, 'price' => 45000],
];

// ======================================
// อ่านข้อมูลจาก Session
// ======================================
if (isset($_SESSION['username'])) {
    echo "สวัสดี, " . $_SESSION['username'] . "!\n";
}

$username = $_SESSION['username'] ?? 'Guest';
$userId = $_SESSION['user_id'] ?? null;
$isLoggedIn = $_SESSION['is_logged_in'] ?? false;

// ======================================
// ลบข้อมูลจาก Session
// ======================================
unset($_SESSION['cart']); // ลบ key เดียว

// ======================================
// ล้าง Session ทั้งหมด
// ======================================
// $_SESSION = []; // ล้างข้อมูลทั้งหมด

// ======================================
// ทำลาย Session
// ======================================
// session_destroy(); // ทำลาย session file

// ======================================
// ข้อมูล Session
// ======================================
echo "Session ID: " . session_id() . "\n";
echo "Session Name: " . session_name() . "\n"; // PHPSESSID
echo "Session Status: " . session_status() . "\n"; // PHP_SESSION_ACTIVE = 2

// เปลี่ยน Session ID (ป้องกัน Session Fixation)
session_regenerate_id(true);
?>
```

### 1.2 Session Configuration

```php
<?php
// session-config.php

// ตั้งค่า Session ก่อน session_start()

// กำหนดอายุ Session (วินาที)
ini_set('session.gc_maxlifetime', 3600); // 1 ชั่วโมง

// ตั้งค่า Cookie
ini_set('session.cookie_lifetime', 0);  // ปิด browser = logout
ini_set('session.cookie_httponly', 1);  // ป้องกัน JavaScript เข้าถึง
ini_set('session.cookie_secure', 1);    // ส่งผ่าน HTTPS เท่านั้น
ini_set('session.cookie_samesite', 'Lax'); // ป้องกัน CSRF

// ตั้งค่าที่เก็บ Session (default: files)
ini_set('session.save_handler', 'files');
ini_set('session.save_path', '/tmp/sessions');

session_start();

// หรือใช้ session_set_cookie_params()
session_set_cookie_params([
    'lifetime' => 0,
    'path'     => '/',
    'domain'   => '',
    'secure'   => true,
    'httponly' => true,
    'samesite' => 'Lax',
]);

session_start();
?>
```

### 1.3 Session Manager Class

```php
<?php
// SessionManager.php

class SessionManager
{
    private int $lifetime;
    private bool $secure;
    
    public function __construct(int $lifetime = 3600, bool $secure = false)
    {
        $this->lifetime = $lifetime;
        $this->secure = $secure;
        
        if (session_status() === PHP_SESSION_NONE) {
            $this->configure();
            session_start();
        }
    }
    
    private function configure(): void
    {
        session_set_cookie_params([
            'lifetime' => $this->lifetime,
            'path'     => '/',
            'domain'   => '',
            'secure'   => $this->secure,
            'httponly' => true,
            'samesite' => 'Lax',
        ]);
        
        ini_set('session.gc_maxlifetime', $this->lifetime);
        ini_set('session.use_strict_mode', 1);
    }
    
    /**
     * เก็บค่า
     */
    public function set(string $key, mixed $value): void
    {
        $_SESSION[$key] = $value;
    }
    
    /**
     * ดึงค่า
     */
    public function get(string $key, mixed $default = null): mixed
    {
        return $_SESSION[$key] ?? $default;
    }
    
    /**
     * ตรวจสอบว่ามีค่าหรือไม่
     */
    public function has(string $key): bool
    {
        return isset($_SESSION[$key]);
    }
    
    /**
     * ลบค่า
     */
    public function remove(string $key): void
    {
        unset($_SESSION[$key]);
    }
    
    /**
     * ดึงค่าแล้วลบ (flash-like)
     */
    public function pull(string $key, mixed $default = null): mixed
    {
        $value = $this->get($key, $default);
        $this->remove($key);
        return $value;
    }
    
    /**
     * ล้างทั้งหมด
     */
    public function clear(): void
    {
        $_SESSION = [];
    }
    
    /**
     * ทำลาย session
     */
    public function destroy(): void
    {
        $_SESSION = [];
        
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
    
    /**
     * สร้าง Session ID ใหม่
     */
    public function regenerate(): void
    {
        session_regenerate_id(true);
    }
    
    /**
     * ดึง Session ID
     */
    public function getId(): string
    {
        return session_id();
    }
    
    /**
     * ดึงข้อมูลทั้งหมด
     */
    public function all(): array
    {
        return $_SESSION;
    }
}
?>
```

---

## 📌 2. Cookies

### 2.1 พื้นฐาน Cookie

Cookie คือข้อมูลที่เก็บไว้ที่ **client browser**

```php
<?php
// cookies-basic.php

// ======================================
// สร้าง Cookie
// ======================================

// setcookie(name, value, expire, path, domain, secure, httponly)
setcookie('username', 'สมชาย', time() + (86400 * 30), '/'); // 30 วัน
setcookie('theme', 'dark', time() + (86400 * 365), '/');    // 1 ปี

// PHP 7.3+ รองรับ SameSite
setcookie('user_pref', 'th', [
    'expires'  => time() + 86400 * 30,
    'path'     => '/',
    'domain'   => '',
    'secure'   => false,
    'httponly' => true,
    'samesite' => 'Lax',
]);

// ======================================
// อ่าน Cookie
// ======================================
$username = $_COOKIE['username'] ?? 'Guest';
$theme = $_COOKIE['theme'] ?? 'light';

echo "สวัสดี, $username! (theme: $theme)\n";

// ======================================
// ลบ Cookie
// ======================================
// ตั้ง expire เป็นอดีต
setcookie('username', '', time() - 3600, '/');
setcookie('theme', '', time() - 3600, '/');

// ======================================
// ข้อมูลทั้งหมดใน Cookie
// ======================================
foreach ($_COOKIE as $name => $value) {
    echo htmlspecialchars($name) . " = " . htmlspecialchars($value) . "\n";
}
?>
```

### 2.2 Cookie Manager

```php
<?php
// CookieManager.php

class CookieManager
{
    private array $defaults;
    
    public function __construct(array $defaults = [])
    {
        $this->defaults = array_merge([
            'expire'   => 0,
            'path'     => '/',
            'domain'   => '',
            'secure'   => false,
            'httponly' => true,
            'samesite' => 'Lax',
        ], $defaults);
    }
    
    /**
     * ตั้งค่า Cookie
     */
    public function set(string $name, mixed $value, int $minutes = 60 * 24 * 30, array $options = []): bool
    {
        $options = array_merge($this->defaults, $options);
        $options['expires'] = time() + ($minutes * 60);
        
        // เข้ารหัสค่าถ้าเป็น array หรือ object
        if (is_array($value) || is_object($value)) {
            $value = json_encode($value);
        }
        
        return setcookie($name, (string)$value, $options);
    }
    
    /**
     * ดึงค่า Cookie
     */
    public function get(string $name, mixed $default = null): mixed
    {
        if (!isset($_COOKIE[$name])) {
            return $default;
        }
        
        $value = $_COOKIE[$name];
        
        // decode JSON ถ้าเป็น JSON string
        $decoded = json_decode($value, true);
        if (json_last_error() === JSON_ERROR_NONE) {
            return $decoded;
        }
        
        return $value;
    }
    
    /**
     * ตรวจสอบว่ามี Cookie หรือไม่
     */
    public function has(string $name): bool
    {
        return isset($_COOKIE[$name]);
    }
    
    /**
     * ลบ Cookie
     */
    public function remove(string $name): bool
    {
        if (!$this->has($name)) {
            return false;
        }
        
        unset($_COOKIE[$name]);
        return setcookie($name, '', time() - 86400, $this->defaults['path'], $this->defaults['domain']);
    }
    
    /**
     * ลบทุก Cookie
     */
    public function removeAll(): void
    {
        foreach ($_COOKIE as $name => $value) {
            $this->remove($name);
        }
    }
    
    /**
     * Remember Me Cookie (encrypted)
     */
    public function rememberUser(int $userId, string $token, int $days = 30): void
    {
        $value = base64_encode(json_encode([
            'user_id' => $userId,
            'token'   => $token,
            'created' => time(),
        ]));
        
        $this->set('remember_me', $value, 60 * 24 * $days, [
            'secure'   => true,
            'httponly' => true,
            'samesite' => 'Strict',
        ]);
    }
    
    /**
     * อ่าน Remember Me Cookie
     */
    public function getRememberData(): ?array
    {
        $cookie = $this->get('remember_me');
        if (!$cookie || !is_string($cookie)) return null;
        
        $decoded = base64_decode($cookie, true);
        if (!$decoded) return null;
        
        $data = json_decode($decoded, true);
        if (!$data || json_last_error() !== JSON_ERROR_NONE) return null;
        
        return $data;
    }
}
?>
```

---

## 📌 3. Session Security

### 3.1 Session Fixation & Hijacking Prevention

```php
<?php
// session-security.php

class SecureSession
{
    private const FINGERPRINT_KEY = '_fingerprint';
    private const LAST_ACTIVITY_KEY = '_last_activity';
    private const TIMEOUT = 1800; // 30 นาที
    
    public static function start(): void
    {
        ini_set('session.use_strict_mode', 1);
        ini_set('session.use_only_cookies', 1);
        ini_set('session.cookie_httponly', 1);
        ini_set('session.cookie_secure', 1);
        ini_set('session.cookie_samesite', 'Lax');
        
        session_start();
        
        // ตรวจสอบ Session Fixation
        if (!isset($_SESSION[self::FINGERPRINT_KEY])) {
            self::initFingerprint();
        } else {
            self::validateFingerprint();
        }
        
        // ตรวจสอบ Timeout
        self::checkTimeout();
        
        // Regenerate ID เป็นระยะ
        self::periodicRegenerate();
    }
    
    /**
     * สร้าง Fingerprint ครั้งแรก
     */
    private static function initFingerprint(): void
    {
        $_SESSION[self::FINGERPRINT_KEY] = self::generateFingerprint();
        $_SESSION[self::LAST_ACTIVITY_KEY] = time();
        session_regenerate_id(true);
    }
    
    /**
     * ตรวจสอบ Fingerprint
     */
    private static function validateFingerprint(): void
    {
        $currentFingerprint = self::generateFingerprint();
        
        if ($_SESSION[self::FINGERPRINT_KEY] !== $currentFingerprint) {
            // Fingerprint ไม่ตรง = อาจเป็น Session Hijacking
            self::invalidate();
            die('Session security violation detected. Please login again.');
        }
    }
    
    /**
     * สร้าง Browser Fingerprint
     */
    private static function generateFingerprint(): string
    {
        $data = implode('|', [
            $_SERVER['HTTP_USER_AGENT'] ?? '',
            $_SERVER['HTTP_ACCEPT_LANGUAGE'] ?? '',
            // ไม่ใช้ IP เพราะ mobile users เปลี่ยน IP บ่อย
        ]);
        
        return hash('sha256', $data . session_id());
    }
    
    /**
     * ตรวจสอบ Timeout
     */
    private static function checkTimeout(): void
    {
        $lastActivity = $_SESSION[self::LAST_ACTIVITY_KEY] ?? 0;
        
        if (time() - $lastActivity > self::TIMEOUT) {
            self::invalidate();
            // Redirect ไปหน้า login พร้อมแจ้งเตือน
            $_SESSION['timeout_message'] = 'Session หมดอายุ กรุณาเข้าสู่ระบบใหม่';
            header('Location: /login.php');
            exit;
        }
        
        $_SESSION[self::LAST_ACTIVITY_KEY] = time();
    }
    
    /**
     * Regenerate Session ID เป็นระยะ (ทุก 5 นาที)
     */
    private static function periodicRegenerate(): void
    {
        $regenerateKey = '_last_regenerate';
        $lastRegenerate = $_SESSION[$regenerateKey] ?? 0;
        
        if (time() - $lastRegenerate > 300) { // 5 นาที
            session_regenerate_id(true);
            $_SESSION[$regenerateKey] = time();
            // อัปเดต fingerprint หลัง regenerate
            $_SESSION[self::FINGERPRINT_KEY] = self::generateFingerprint();
        }
    }
    
    /**
     * ยกเลิก Session
     */
    public static function invalidate(): void
    {
        $_SESSION = [];
        
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
        
        session_destroy();
        session_start();
    }
    
    /**
     * Login
     */
    public static function login(array $user): void
    {
        // Regenerate ID หลัง login เพื่อป้องกัน Session Fixation
        session_regenerate_id(true);
        
        $_SESSION['user'] = [
            'id'         => $user['id'],
            'username'   => $user['username'],
            'email'      => $user['email'],
            'role'       => $user['role'],
            'logged_in'  => true,
            'login_time' => time(),
        ];
        
        $_SESSION[self::FINGERPRINT_KEY] = self::generateFingerprint();
        $_SESSION[self::LAST_ACTIVITY_KEY] = time();
    }
    
    /**
     * Logout
     */
    public static function logout(): void
    {
        self::invalidate();
    }
    
    /**
     * ตรวจสอบว่า Login อยู่หรือไม่
     */
    public static function isLoggedIn(): bool
    {
        return isset($_SESSION['user']['logged_in']) && $_SESSION['user']['logged_in'] === true;
    }
    
    /**
     * ดึงข้อมูลผู้ใช้
     */
    public static function user(): ?array
    {
        return $_SESSION['user'] ?? null;
    }
    
    /**
     * ตรวจสอบ Role
     */
    public static function hasRole(string $role): bool
    {
        $user = self::user();
        return $user !== null && $user['role'] === $role;
    }
}
?>
```

---

## 📌 4. Flash Messages

Flash Messages คือข้อความที่แสดงครั้งเดียวแล้วหายไป (เช่น "บันทึกสำเร็จ!")

```php
<?php
// FlashMessage.php

class Flash
{
    private const SESSION_KEY = '_flash';
    
    public static function init(): void
    {
        if (session_status() === PHP_SESSION_NONE) {
            session_start();
        }
        
        if (!isset($_SESSION[self::SESSION_KEY])) {
            $_SESSION[self::SESSION_KEY] = [];
        }
    }
    
    /**
     * เพิ่มข้อความ
     */
    public static function add(string $type, string $message): void
    {
        self::init();
        $_SESSION[self::SESSION_KEY][] = [
            'type'    => $type,
            'message' => $message,
        ];
    }
    
    // Shortcut methods
    public static function success(string $message): void { self::add('success', $message); }
    public static function error(string $message): void   { self::add('error', $message); }
    public static function warning(string $message): void { self::add('warning', $message); }
    public static function info(string $message): void    { self::add('info', $message); }
    
    /**
     * ดึงข้อความทั้งหมดแล้วล้าง
     */
    public static function get(): array
    {
        self::init();
        $messages = $_SESSION[self::SESSION_KEY];
        $_SESSION[self::SESSION_KEY] = [];
        return $messages;
    }
    
    /**
     * ตรวจสอบว่ามีข้อความหรือไม่
     */
    public static function has(): bool
    {
        self::init();
        return !empty($_SESSION[self::SESSION_KEY]);
    }
    
    /**
     * แสดงข้อความ HTML
     */
    public static function render(): string
    {
        $messages = self::get();
        if (empty($messages)) return '';
        
        $colors = [
            'success' => '#d4edda',
            'error'   => '#f8d7da',
            'warning' => '#fff3cd',
            'info'    => '#cce5ff',
        ];
        
        $icons = [
            'success' => '✅',
            'error'   => '❌',
            'warning' => '⚠️',
            'info'    => 'ℹ️',
        ];
        
        $html = '';
        foreach ($messages as $msg) {
            $type = $msg['type'];
            $bg = $colors[$type] ?? '#f5f5f5';
            $icon = $icons[$type] ?? '•';
            
            $html .= sprintf(
                '<div style="background:%s; padding:12px 16px; margin:8px 0; border-radius:4px; border-left:4px solid currentColor;">%s %s</div>',
                $bg,
                $icon,
                htmlspecialchars($msg['message'])
            );
        }
        
        return $html;
    }
}

// ======================================
// ตัวอย่างการใช้งาน
// ======================================

// ในหน้าที่บันทึกข้อมูล
Flash::success('บันทึกข้อมูลสำเร็จ!');
Flash::info('อีเมลยืนยันถูกส่งไปแล้ว');
header('Location: dashboard.php');
exit;

// ในหน้า dashboard.php
echo Flash::render();
?>
```

---

## 🛠️ Workshop: สร้าง Authentication ด้วย Sessions

### โครงสร้างโปรเจค

```
auth-system/
├── login.php
├── logout.php
├── dashboard.php
├── register.php
├── profile.php
├── includes/
│   ├── auth.php        (Auth functions)
│   ├── db.php          (Database simulation)
│   └── flash.php
└── data/
    └── users.json
```

### includes/auth.php

```php
<?php
// includes/auth.php

class Auth
{
    private static ?Auth $instance = null;
    private array $users = [];
    private string $usersFile;
    
    private function __construct(string $usersFile = 'data/users.json')
    {
        $this->usersFile = $usersFile;
        $this->loadUsers();
        
        if (session_status() === PHP_SESSION_NONE) {
            session_set_cookie_params([
                'lifetime' => 0,
                'path'     => '/',
                'httponly' => true,
                'samesite' => 'Lax',
            ]);
            session_start();
        }
    }
    
    public static function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }
    
    private function loadUsers(): void
    {
        if (!file_exists($this->usersFile)) {
            // สร้าง admin user เริ่มต้น
            $adminUser = [
                'id'         => 1,
                'username'   => 'admin',
                'email'      => 'admin@example.com',
                'password'   => password_hash('Admin@1234', PASSWORD_BCRYPT),
                'role'       => 'admin',
                'name'       => 'ผู้ดูแลระบบ',
                'created_at' => date('Y-m-d H:i:s'),
                'last_login' => null,
            ];
            
            $dir = dirname($this->usersFile);
            if (!is_dir($dir)) mkdir($dir, 0755, true);
            
            file_put_contents($this->usersFile, json_encode([$adminUser], JSON_PRETTY_PRINT));
        }
        
        $this->users = json_decode(file_get_contents($this->usersFile), true) ?? [];
    }
    
    private function saveUsers(): void
    {
        file_put_contents(
            $this->usersFile,
            json_encode($this->users, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE),
            LOCK_EX
        );
    }
    
    /**
     * ลงทะเบียนผู้ใช้ใหม่
     */
    public function register(array $data): array
    {
        $errors = [];
        
        // Validate
        if (empty($data['username'])) {
            $errors['username'] = 'กรุณากรอกชื่อผู้ใช้';
        } elseif (strlen($data['username']) < 3 || strlen($data['username']) > 20) {
            $errors['username'] = 'ชื่อผู้ใช้ต้องมี 3-20 ตัวอักษร';
        } elseif (!preg_match('/^[a-zA-Z0-9_]+$/', $data['username'])) {
            $errors['username'] = 'ชื่อผู้ใช้ใช้ได้เฉพาะ a-z, 0-9, _';
        } elseif ($this->findByUsername($data['username'])) {
            $errors['username'] = 'ชื่อผู้ใช้นี้ถูกใช้งานแล้ว';
        }
        
        if (empty($data['email'])) {
            $errors['email'] = 'กรุณากรอกอีเมล';
        } elseif (!filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            $errors['email'] = 'รูปแบบอีเมลไม่ถูกต้อง';
        } elseif ($this->findByEmail($data['email'])) {
            $errors['email'] = 'อีเมลนี้ถูกใช้งานแล้ว';
        }
        
        if (empty($data['password'])) {
            $errors['password'] = 'กรุณากรอกรหัสผ่าน';
        } elseif (strlen($data['password']) < 8) {
            $errors['password'] = 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
        } elseif (!preg_match('/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/', $data['password'])) {
            $errors['password'] = 'รหัสผ่านต้องมีตัวพิมพ์เล็ก พิมพ์ใหญ่ และตัวเลข';
        }
        
        if (($data['password'] ?? '') !== ($data['password_confirm'] ?? '')) {
            $errors['password_confirm'] = 'รหัสผ่านไม่ตรงกัน';
        }
        
        if (!empty($errors)) {
            return ['success' => false, 'errors' => $errors];
        }
        
        // สร้างผู้ใช้ใหม่
        $maxId = empty($this->users) ? 0 : max(array_column($this->users, 'id'));
        
        $newUser = [
            'id'         => $maxId + 1,
            'username'   => $data['username'],
            'email'      => $data['email'],
            'password'   => password_hash($data['password'], PASSWORD_BCRYPT),
            'role'       => 'user',
            'name'       => $data['name'] ?? $data['username'],
            'created_at' => date('Y-m-d H:i:s'),
            'last_login' => null,
        ];
        
        $this->users[] = $newUser;
        $this->saveUsers();
        
        return ['success' => true, 'user' => $newUser];
    }
    
    /**
     * เข้าสู่ระบบ
     */
    public function login(string $username, string $password, bool $remember = false): array
    {
        $user = $this->findByUsername($username);
        
        if (!$user) {
            $user = $this->findByEmail($username); // ลอง login ด้วย email
        }
        
        if (!$user) {
            return ['success' => false, 'message' => 'ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง'];
        }
        
        if (!password_verify($password, $user['password'])) {
            return ['success' => false, 'message' => 'ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง'];
        }
        
        // สร้าง Session ใหม่ (ป้องกัน Session Fixation)
        session_regenerate_id(true);
        
        // เก็บข้อมูลลง Session
        $_SESSION['user'] = [
            'id'       => $user['id'],
            'username' => $user['username'],
            'email'    => $user['email'],
            'name'     => $user['name'],
            'role'     => $user['role'],
        ];
        $_SESSION['logged_in'] = true;
        $_SESSION['login_time'] = time();
        $_SESSION['_last_activity'] = time();
        
        // Remember Me
        if ($remember) {
            $token = bin2hex(random_bytes(32));
            setcookie('remember_token', $token, [
                'expires'  => time() + 86400 * 30,
                'path'     => '/',
                'httponly' => true,
                'samesite' => 'Strict',
            ]);
        }
        
        // อัปเดต last_login
        $this->updateLastLogin($user['id']);
        
        return ['success' => true, 'user' => $user];
    }
    
    /**
     * ออกจากระบบ
     */
    public function logout(): void
    {
        $_SESSION = [];
        
        $params = session_get_cookie_params();
        setcookie(session_name(), '', time() - 86400, $params['path'], $params['domain']);
        setcookie('remember_token', '', time() - 86400, '/');
        
        session_destroy();
    }
    
    /**
     * ตรวจสอบว่า Login อยู่หรือไม่
     */
    public function isLoggedIn(): bool
    {
        if (!isset($_SESSION['logged_in']) || !$_SESSION['logged_in']) {
            return false;
        }
        
        // ตรวจสอบ timeout (30 นาที)
        $lastActivity = $_SESSION['_last_activity'] ?? 0;
        if (time() - $lastActivity > 1800) {
            $this->logout();
            return false;
        }
        
        $_SESSION['_last_activity'] = time();
        return true;
    }
    
    /**
     * บังคับ Login
     */
    public function requireLogin(string $redirect = '/login.php'): void
    {
        if (!$this->isLoggedIn()) {
            Flash::warning('กรุณาเข้าสู่ระบบก่อน');
            header("Location: $redirect");
            exit;
        }
    }
    
    /**
     * บังคับ Role
     */
    public function requireRole(string $role, string $redirect = '/dashboard.php'): void
    {
        $this->requireLogin();
        
        $user = $this->currentUser();
        if (!$user || $user['role'] !== $role) {
            Flash::error('คุณไม่มีสิทธิ์เข้าถึงหน้านี้');
            header("Location: $redirect");
            exit;
        }
    }
    
    /**
     * ดึงข้อมูลผู้ใช้ปัจจุบัน
     */
    public function currentUser(): ?array
    {
        return $_SESSION['user'] ?? null;
    }
    
    private function findByUsername(string $username): ?array
    {
        foreach ($this->users as $user) {
            if (strtolower($user['username']) === strtolower($username)) {
                return $user;
            }
        }
        return null;
    }
    
    private function findByEmail(string $email): ?array
    {
        foreach ($this->users as $user) {
            if (strtolower($user['email']) === strtolower($email)) {
                return $user;
            }
        }
        return null;
    }
    
    private function updateLastLogin(int $userId): void
    {
        foreach ($this->users as &$user) {
            if ($user['id'] === $userId) {
                $user['last_login'] = date('Y-m-d H:i:s');
                break;
            }
        }
        $this->saveUsers();
    }
}
```

### login.php

```php
<?php
// login.php
require_once 'includes/flash.php';
require_once 'includes/auth.php';

$auth = Auth::getInstance();

if ($auth->isLoggedIn()) {
    header('Location: dashboard.php');
    exit;
}

$errors = [];
$old = [];

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $username = trim($_POST['username'] ?? '');
    $password = $_POST['password'] ?? '';
    $remember = isset($_POST['remember']);
    
    if (empty($username) || empty($password)) {
        $errors[] = 'กรุณากรอกชื่อผู้ใช้และรหัสผ่าน';
    } else {
        $result = $auth->login($username, $password, $remember);
        
        if ($result['success']) {
            Flash::success('เข้าสู่ระบบสำเร็จ! ยินดีต้อนรับ ' . $result['user']['name']);
            header('Location: dashboard.php');
            exit;
        } else {
            $errors[] = $result['message'];
            $old['username'] = $username;
        }
    }
}
?>
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>เข้าสู่ระบบ</title>
    <style>
        body { font-family: sans-serif; display: flex; justify-content: center; align-items: center; min-height: 100vh; background: #f0f2f5; margin: 0; }
        .login-box { background: white; padding: 40px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); width: 100%; max-width: 400px; }
        h1 { text-align: center; color: #333; margin-bottom: 30px; }
        .form-group { margin-bottom: 20px; }
        label { display: block; margin-bottom: 6px; font-weight: bold; color: #555; }
        input[type=text], input[type=password] { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 4px; font-size: 15px; box-sizing: border-box; }
        input:focus { outline: none; border-color: #4CAF50; }
        .btn { width: 100%; padding: 13px; background: #4CAF50; color: white; border: none; border-radius: 4px; font-size: 16px; cursor: pointer; }
        .btn:hover { background: #45a049; }
        .error { background: #fee; color: #c00; padding: 10px; border-radius: 4px; margin-bottom: 20px; }
        .register-link { text-align: center; margin-top: 20px; }
        .remember { display: flex; align-items: center; gap: 8px; }
    </style>
</head>
<body>
    <div class="login-box">
        <h1>🔐 เข้าสู่ระบบ</h1>
        
        <?= Flash::render() ?>
        
        <?php if (!empty($errors)): ?>
        <div class="error">
            <?php foreach ($errors as $error): ?>
                <div>❌ <?= htmlspecialchars($error) ?></div>
            <?php endforeach; ?>
        </div>
        <?php endif; ?>
        
        <form action="" method="post">
            <div class="form-group">
                <label for="username">ชื่อผู้ใช้หรืออีเมล</label>
                <input type="text" id="username" name="username" 
                       value="<?= htmlspecialchars($old['username'] ?? '') ?>"
                       placeholder="username หรือ email@example.com"
                       autofocus>
            </div>
            
            <div class="form-group">
                <label for="password">รหัสผ่าน</label>
                <input type="password" id="password" name="password" placeholder="••••••••">
            </div>
            
            <div class="form-group">
                <label class="remember">
                    <input type="checkbox" name="remember">
                    จดจำการเข้าสู่ระบบ (30 วัน)
                </label>
            </div>
            
            <button type="submit" class="btn">เข้าสู่ระบบ</button>
        </form>
        
        <div class="register-link">
            ยังไม่มีบัญชี? <a href="register.php">ลงทะเบียน</a>
        </div>
    </div>
</body>
</html>
```

### dashboard.php

```php
<?php
// dashboard.php
require_once 'includes/flash.php';
require_once 'includes/auth.php';

$auth = Auth::getInstance();
$auth->requireLogin('/login.php');

$user = $auth->currentUser();
?>
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Dashboard</title>
    <style>
        body { font-family: sans-serif; margin: 0; background: #f0f2f5; }
        .header { background: #333; color: white; padding: 15px 30px; display: flex; justify-content: space-between; align-items: center; }
        .content { padding: 30px; max-width: 1000px; margin: 0 auto; }
        .card { background: white; padding: 25px; border-radius: 8px; box-shadow: 0 2px 5px rgba(0,0,0,0.05); margin-bottom: 20px; }
        .stats { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; margin-bottom: 30px; }
        .stat-card { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 5px rgba(0,0,0,0.05); text-align: center; }
        .stat-number { font-size: 2em; font-weight: bold; color: #4CAF50; }
        .logout-btn { background: #e74c3c; color: white; padding: 8px 16px; border: none; border-radius: 4px; cursor: pointer; text-decoration: none; }
    </style>
</head>
<body>
    <div class="header">
        <h2>📊 Dashboard</h2>
        <div>
            👤 <?= htmlspecialchars($user['name']) ?> 
            (<?= htmlspecialchars($user['role']) ?>)
            &nbsp;
            <a href="logout.php" class="logout-btn">ออกจากระบบ</a>
        </div>
    </div>
    
    <div class="content">
        <?= Flash::render() ?>
        
        <div class="stats">
            <div class="stat-card">
                <div class="stat-number">156</div>
                <div>ผู้ใช้ทั้งหมด</div>
            </div>
            <div class="stat-card">
                <div class="stat-number">42</div>
                <div>โพสต์วันนี้</div>
            </div>
            <div class="stat-card">
                <div class="stat-number">1,234</div>
                <div>ยอดเข้าชม</div>
            </div>
        </div>
        
        <div class="card">
            <h3>ข้อมูล Session ปัจจุบัน</h3>
            <p>Session ID: <code><?= session_id() ?></code></p>
            <p>เข้าสู่ระบบเมื่อ: <?= date('d/m/Y H:i:s', $_SESSION['login_time'] ?? time()) ?></p>
            <p>กิจกรรมล่าสุด: <?= date('d/m/Y H:i:s', $_SESSION['_last_activity'] ?? time()) ?></p>
        </div>
    </div>
</body>
</html>
```

---

## 📝 Quiz

### คำถาม

**ข้อ 1:** เหตุใดจึงควรเรียก `session_regenerate_id(true)` หลัง Login?
- A. เพื่อให้ Session ทำงานเร็วขึ้น
- B. เพื่อป้องกัน Session Fixation Attack
- C. เพื่อต่ออายุ Session
- D. เพื่อลบ Session เก่า

**ข้อ 2:** Cookie ที่มี flag `httponly=true` คืออะไร?
- A. ส่งผ่าน HTTP เท่านั้น ไม่ใช้ HTTPS
- B. JavaScript ไม่สามารถอ่าน cookie ได้
- C. Cookie หมดอายุเมื่อปิด browser
- D. Cookie ใช้ได้เฉพาะ path /

**ข้อ 3:** Flash Message แตกต่างจาก Session ปกติอย่างไร?
- A. Flash Message ใช้ Cookie แทน Session
- B. Flash Message แสดงได้ครั้งเดียวแล้วถูกลบ
- C. Flash Message ไม่ต้อง session_start()
- D. Flash Message เก็บข้อมูลได้มากกว่า

**ข้อ 4:** `$_SESSION` และ `$_COOKIE` ต่างกันอย่างไร?
- A. `$_SESSION` เก็บที่ server, `$_COOKIE` เก็บที่ client
- B. `$_SESSION` เร็วกว่า `$_COOKIE`
- C. `$_COOKIE` ปลอดภัยกว่า `$_SESSION`
- D. ทั้งสองเก็บข้อมูลที่ server

**ข้อ 5:** Session Timeout ควรจัดการอย่างไร?
- A. เพิ่ม `session.lifetime` ให้มาก
- B. ตรวจสอบ `$_SESSION['_last_activity']` และเปรียบเทียบกับ `time()`
- C. ใช้ `session_destroy()` ทุก request
- D. ใช้ `setcookie()` กำหนดอายุ

### เฉลย

| ข้อ | คำตอบ | อธิบาย |
|-----|-------|--------|
| 1 | B | Session Fixation คือการที่ attacker กำหนด Session ID ให้ victim ก่อน login แล้วใช้ ID เดิมหลัง login |
| 2 | B | `httponly` ป้องกัน JavaScript (เช่น XSS) จากการอ่าน cookie ค่านี้ |
| 3 | B | Flash Message ใช้ pattern "เก็บแล้วลบ" เมื่อ render ครั้งเดียว |
| 4 | A | Session data อยู่ที่ server, browser เก็บแค่ Session ID ส่วน Cookie เก็บข้อมูลจริงที่ browser |
| 5 | B | เก็บเวลากิจกรรมล่าสุดใน Session แล้วตรวจสอบทุก request |

---

## ➡️ Part ถัดไป

**[Part 12: PHP MySQL - การทำงานกับฐานข้อมูล MySQL](part-012-php-mysql.md)**
