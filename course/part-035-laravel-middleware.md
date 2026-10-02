# Part 035: Laravel Middleware

## ระดับ: Intermediate
## เวลาที่ใช้: 3-4 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจการทำงานของ Middleware ใน HTTP lifecycle
- สร้าง Middleware ที่ใช้งานได้จริง
- กำหนด Global, Route, และ Group Middleware
- ใช้ Terminable Middleware
- สร้างระบบ Rate Limiting, Locale Middleware, และ Admin Check

---

## 1. Middleware คืออะไร?

Middleware เป็น "ตัวกรอง" ที่ตรวจสอบ HTTP request ก่อนที่จะส่งต่อไปยัง Controller หรือ Response ก่อนส่งออกไปให้ Client

```
Client Request
    │
    ▼
┌─────────────────────────┐
│     Global Middleware   │  (ทุก request)
│  - Encryption Cookies   │
│  - Start Session        │
│  - Verify CSRF Token    │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│    Route Middleware     │  (เฉพาะ routes ที่กำหนด)
│  - auth                 │
│  - verified             │
│  - throttle             │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│       Controller        │
│    (Your Code Here)     │
└─────────────────────────┘
    │
    ▼
Response -> Back to Client (ผ่าน Middleware ย้อนกลับ)
```

### Built-in Middleware ของ Laravel

```php
<?php
// bootstrap/app.php (Laravel 11)

// Global Middleware ที่ Laravel ใส่ให้อัตโนมัติ:
// - InertiaMiddleware
// - HandleCors
// - PreventRequestsDuringMaintenance
// - TrimStrings
// - ConvertEmptyStringsToNull

// Web Group:
// - EncryptCookies
// - AddQueuedCookiesToResponse
// - StartSession
// - ShareErrorsFromSession
// - VerifyCsrfToken
// - SubstituteBindings

// API Group:
// - ThrottleRequests
// - SubstituteBindings
```

---

## 2. การสร้าง Middleware

### 2.1 คำสั่ง Artisan

```bash
# สร้าง Middleware
php artisan make:middleware CheckUserActive
php artisan make:middleware EnsureEmailVerified
php artisan make:middleware SetLocale
php artisan make:middleware AdminOnly
php artisan make:middleware LogRequests
```

### 2.2 โครงสร้าง Middleware

```php
<?php
// app/Http/Middleware/CheckUserActive.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class CheckUserActive
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        // โค้ดก่อน request ถึง Controller (Before Middleware)
        if (auth()->check() && !auth()->user()->is_active) {
            auth()->logout();
            return redirect()->route('login')
                ->with('error', 'บัญชีของคุณถูกระงับ');
        }

        // ส่งต่อ request ไปยัง step ถัดไป
        $response = $next($request);

        // โค้ดหลัง Controller ทำงานแล้ว (After Middleware)
        // $response->headers->set('X-Custom-Header', 'value');

        return $response;
    }
}
```

### 2.3 Before vs After Middleware

```php
<?php
// Before Middleware - รันก่อน Controller
class BeforeMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        // โค้ดรันก่อน
        echo "ก่อน request";

        return $next($request);
    }
}

// After Middleware - รันหลัง Controller
class AfterMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request); // รัน Controller ก่อน

        // โค้ดรันหลัง
        echo "หลัง response";

        return $response;
    }
}

// Combined: ทั้งก่อนและหลัง
class CombinedMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        // ก่อน
        $startTime = microtime(true);

        $response = $next($request);

        // หลัง
        $duration = microtime(true) - $startTime;
        $response->headers->set('X-Response-Time', $duration . 'ms');

        return $response;
    }
}
```

---

## 3. การ Register Middleware

### 3.1 Laravel 11 (bootstrap/app.php)

```php
<?php
// bootstrap/app.php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use App\Http\Middleware\CheckUserActive;
use App\Http\Middleware\SetLocale;
use App\Http\Middleware\AdminOnly;
use App\Http\Middleware\LogRequests;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        
        // Global Middleware - รันทุก request
        $middleware->append(LogRequests::class);
        $middleware->prepend(SetLocale::class); // เพิ่มต้น pipeline

        // เพิ่มใน Web group
        $middleware->appendToGroup('web', [
            CheckUserActive::class,
        ]);

        // เพิ่มใน API group
        $middleware->appendToGroup('api', [
            \App\Http\Middleware\ForceJsonResponse::class,
        ]);

        // กำหนด Middleware aliases สำหรับใช้ในไฟล์ routes
        $middleware->alias([
            'admin'       => AdminOnly::class,
            'role'        => \App\Http\Middleware\RequireRole::class,
            'active'      => CheckUserActive::class,
            'locale'      => SetLocale::class,
        ]);

        // Priority (ลำดับการทำงาน)
        $middleware->priority([
            \Illuminate\Cookie\Middleware\EncryptCookies::class,
            \Illuminate\Session\Middleware\StartSession::class,
            // ...
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

### 3.2 Laravel 10 (Kernel.php)

```php
<?php
// app/Http/Kernel.php (Laravel 10 และก่อนหน้า)

namespace App\Http;

use Illuminate\Foundation\Http\Kernel as HttpKernel;

class Kernel extends HttpKernel
{
    /**
     * Global HTTP Middleware - รันทุก request
     */
    protected $middleware = [
        \App\Http\Middleware\TrustProxies::class,
        \Illuminate\Http\Middleware\HandleCors::class,
        \App\Http\Middleware\PreventRequestsDuringMaintenance::class,
        \Illuminate\Foundation\Http\Middleware\ValidatePostSize::class,
        \App\Http\Middleware\TrimStrings::class,
        \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
        \App\Http\Middleware\LogRequests::class,  // เพิ่มเอง
    ];

    /**
     * Middleware Groups
     */
    protected $middlewareGroups = [
        'web' => [
            \App\Http\Middleware\EncryptCookies::class,
            \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
            \Illuminate\Session\Middleware\StartSession::class,
            \Illuminate\View\Middleware\ShareErrorsFromSession::class,
            \App\Http\Middleware\VerifyCsrfToken::class,
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
            \App\Http\Middleware\CheckUserActive::class,  // เพิ่มเอง
        ],

        'api' => [
            \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
            \Illuminate\Routing\Middleware\ThrottleRequests::class.':api',
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
        ],
    ];

    /**
     * Route Middleware - ใช้ใน routes โดยชื่อย่อ
     */
    protected $routeMiddleware = [
        'auth'             => \App\Http\Middleware\Authenticate::class,
        'auth.basic'       => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
        'cache.headers'    => \Illuminate\Http\Middleware\SetCacheHeaders::class,
        'can'              => \Illuminate\Auth\Middleware\Authorize::class,
        'guest'            => \App\Http\Middleware\RedirectIfAuthenticated::class,
        'password.confirm' => \Illuminate\Auth\Middleware\RequirePassword::class,
        'signed'           => \App\Http\Middleware\ValidateSignature::class,
        'throttle'         => \Illuminate\Routing\Middleware\ThrottleRequests::class,
        'verified'         => \Illuminate\Auth\Middleware\EnsureEmailIsVerified::class,
        
        // Custom
        'admin'            => \App\Http\Middleware\AdminOnly::class,
        'role'             => \App\Http\Middleware\RequireRole::class,
        'active'           => \App\Http\Middleware\CheckUserActive::class,
    ];
}
```

---

## 4. การใช้ Middleware ใน Routes

### 4.1 ใน routes/web.php

```php
<?php
use App\Http\Middleware\AdminOnly;

// Route เดี่ยว
Route::get('/admin', function () {
    // ...
})->middleware('admin');

// หลาย middleware
Route::get('/dashboard', function () {
    // ...
})->middleware(['auth', 'verified', 'active']);

// Middleware group
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
    Route::get('/profile',   [ProfileController::class, 'edit']);
});

// Nested groups
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/posts', [PostController::class, 'index']);
    
    // Admin-only nested group
    Route::middleware('admin')->prefix('admin')->name('admin.')->group(function () {
        Route::resource('users', Admin\UserController::class);
        Route::resource('settings', Admin\SettingController::class);
    });
});

// Middleware พร้อม parameters
Route::get('/posts', function () {})->middleware('throttle:60,1');
Route::get('/admin', function () {})->middleware('role:admin,super_admin');

// Exclude middleware จาก route
Route::withoutMiddleware([VerifyCsrfToken::class])->group(function () {
    Route::post('/webhook', [WebhookController::class, 'handle']);
});
```

### 4.2 ใน Controller

```php
<?php
// app/Http/Controllers/PostController.php

class PostController extends Controller
{
    public function __construct()
    {
        // Apply middleware ทุก method
        $this->middleware('auth');
        
        // Apply เฉพาะ methods ที่กำหนด
        $this->middleware('verified')->only(['store', 'update', 'destroy']);
        
        // Apply ทุก method ยกเว้นที่กำหนด
        $this->middleware('admin')->except(['index', 'show']);
        
        // Middleware พร้อม parameters
        $this->middleware('throttle:10,1')->only(['store']);
    }

    public function index()
    {
        return Post::paginate(15);
    }

    public function store(Request $request)
    {
        // ...
    }
}
```

---

## 5. Middleware Parameters

```php
<?php
// app/Http/Middleware/RequireRole.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class RequireRole
{
    /**
     * Handle an incoming request.
     * รับ parameter ผ่าน method signature
     */
    public function handle(Request $request, Closure $next, string ...$roles): Response
    {
        if (!$request->user()) {
            return redirect()->route('login');
        }

        $userRole = $request->user()->role->value;

        if (!in_array($userRole, $roles)) {
            if ($request->expectsJson()) {
                return response()->json([
                    'message' => 'ไม่มีสิทธิ์เข้าถึง',
                ], 403);
            }

            abort(403, 'ไม่มีสิทธิ์เข้าถึง');
        }

        return $next($request);
    }
}
```

```php
<?php
// การใช้งาน Middleware Parameters ใน Routes

// Single role
Route::get('/admin')->middleware('role:admin');

// Multiple roles (comma-separated)
Route::get('/content')->middleware('role:admin,editor');

// หรือใน PHP
Route::get('/content')->middleware(['role:admin,editor']);
```

---

## 6. Middleware Groups

```php
<?php
// สร้าง Custom Middleware Group (Laravel 11)

// bootstrap/app.php
$middleware->group('api.auth', [
    'auth:sanctum',
    \App\Http\Middleware\CheckApiVersion::class,
    \App\Http\Middleware\ForceJsonResponse::class,
]);

// ใช้งาน
Route::middleware('api.auth')->group(function () {
    // ...
});
```

---

## 7. Terminable Middleware

Terminable Middleware รันหลังจาก response ถูกส่งไปยัง client แล้ว เหมาะสำหรับ tasks ที่ไม่ต้องรอผล

```php
<?php
// app/Http/Middleware/LogRequestTime.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\Response;

class LogRequestTime
{
    private float $startTime;

    public function handle(Request $request, Closure $next): Response
    {
        $this->startTime = microtime(true);
        
        return $next($request);
    }

    /**
     * Terminable: รันหลังจาก response ส่งให้ client แล้ว
     * ต้องมี terminate() method
     */
    public function terminate(Request $request, Response $response): void
    {
        $duration = round((microtime(true) - $this->startTime) * 1000, 2);
        
        Log::info('Request completed', [
            'method'     => $request->method(),
            'url'        => $request->fullUrl(),
            'status'     => $response->getStatusCode(),
            'duration_ms' => $duration,
            'user_id'    => auth()->id(),
            'ip'         => $request->ip(),
        ]);
    }
}
```

```php
<?php
// อีกตัวอย่าง: Session Cleanup
class CleanupSession
{
    public function handle(Request $request, Closure $next): Response
    {
        return $next($request);
    }

    public function terminate(Request $request, Response $response): void
    {
        // ทำ cleanup หลัง response ส่งแล้ว
        $request->session()->forget('_flash_messages');
        
        // หรือ update statistics
        if (auth()->check()) {
            auth()->user()->increment('total_requests');
        }
    }
}
```

---

## 8. Workshop: Middleware ในโลกจริง

### 8.1 Rate Limiting Middleware

```php
<?php
// app/Http/Middleware/ApiRateLimit.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Cache\RateLimiter;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Symfony\Component\HttpFoundation\Response as BaseResponse;

class ApiRateLimit
{
    public function __construct(
        protected RateLimiter $limiter
    ) {}

    public function handle(Request $request, Closure $next, int $maxAttempts = 60, int $decayMinutes = 1): BaseResponse
    {
        $key = $this->resolveRequestSignature($request);

        if ($this->limiter->tooManyAttempts($key, $maxAttempts)) {
            return $this->buildTooManyAttemptsResponse($key, $maxAttempts);
        }

        $this->limiter->hit($key, $decayMinutes * 60);

        $response = $next($request);

        return $this->addHeaders(
            $response,
            $maxAttempts,
            $this->calculateRemainingAttempts($key, $maxAttempts)
        );
    }

    protected function resolveRequestSignature(Request $request): string
    {
        // Key: ip + user_id (ถ้า login) หรือแค่ ip
        if ($user = $request->user()) {
            return sha1($request->route()?->getDomain() . '|' . $user->id);
        }

        return sha1($request->ip() . '|' . $request->route()?->getDomain());
    }

    protected function buildTooManyAttemptsResponse(string $key, int $maxAttempts): BaseResponse
    {
        $retryAfter = $this->limiter->availableIn($key);

        return response()->json([
            'message'     => 'Too Many Requests - กรุณาลองใหม่ภายหลัง',
            'retry_after' => $retryAfter,
        ], 429)->withHeaders([
            'Retry-After'         => $retryAfter,
            'X-RateLimit-Limit'   => $maxAttempts,
            'X-RateLimit-Remaining' => 0,
        ]);
    }

    protected function addHeaders(BaseResponse $response, int $maxAttempts, int $remainingAttempts): BaseResponse
    {
        $response->headers->add([
            'X-RateLimit-Limit'     => $maxAttempts,
            'X-RateLimit-Remaining' => $remainingAttempts,
        ]);

        return $response;
    }

    protected function calculateRemainingAttempts(string $key, int $maxAttempts): int
    {
        return $maxAttempts - $this->limiter->attempts($key) + 1;
    }
}
```

### 8.2 Rate Limiter สำหรับ Authentication

```php
<?php
// app/Providers/AppServiceProvider.php

use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Rate limiter สำหรับ API ทั่วไป
        RateLimiter::for('api', function (Request $request) {
            return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
        });

        // Rate limiter สำหรับ login
        RateLimiter::for('login', function (Request $request) {
            return [
                Limit::perMinute(5)->by($request->ip()),                    // 5 ครั้ง/นาที ต่อ IP
                Limit::perMinute(3)->by($request->input('email') . '|' . $request->ip()), // 3 ครั้ง/นาที ต่อ email+IP
            ];
        });

        // Rate limiter สำหรับ forgot password
        RateLimiter::for('forgot-password', function (Request $request) {
            return Limit::perHour(3)->by($request->input('email') . '|' . $request->ip());
        });

        // Rate limiter แบบ dynamic ตาม user role
        RateLimiter::for('uploads', function (Request $request) {
            return $request->user()?->role === 'admin'
                ? Limit::none()
                : Limit::perMinute(10)->by($request->user()?->id ?: $request->ip());
        });
    }
}
```

```php
<?php
// routes/api.php

Route::middleware('throttle:login')->post('/auth/login', [AuthController::class, 'login']);
Route::middleware('throttle:forgot-password')->post('/auth/forgot-password', [ForgotPasswordController::class, 'store']);

Route::middleware(['auth:sanctum', 'throttle:uploads'])->post('/upload', [UploadController::class, 'store']);
```

### 8.3 Locale Middleware

```php
<?php
// app/Http/Middleware/SetLocale.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\App;
use Illuminate\Support\Facades\Session;
use Symfony\Component\HttpFoundation\Response;

class SetLocale
{
    private array $supportedLocales = ['th', 'en', 'zh', 'ja', 'ko'];
    private string $defaultLocale = 'th';

    public function handle(Request $request, Closure $next): Response
    {
        $locale = $this->determineLocale($request);

        App::setLocale($locale);
        Session::put('locale', $locale);

        return $next($request);
    }

    private function determineLocale(Request $request): string
    {
        // Priority: URL param -> Session -> Browser -> Default

        // 1. URL parameter: ?lang=en
        if ($request->has('lang')) {
            $lang = $request->query('lang');
            if (in_array($lang, $this->supportedLocales)) {
                return $lang;
            }
        }

        // 2. Session
        if (Session::has('locale')) {
            $sessionLocale = Session::get('locale');
            if (in_array($sessionLocale, $this->supportedLocales)) {
                return $sessionLocale;
            }
        }

        // 3. User preference (ถ้า login)
        if (auth()->check() && !empty(auth()->user()->preferred_locale)) {
            $userLocale = auth()->user()->preferred_locale;
            if (in_array($userLocale, $this->supportedLocales)) {
                return $userLocale;
            }
        }

        // 4. Accept-Language header
        $browserLocale = $this->getLocaleFromBrowser($request);
        if ($browserLocale) {
            return $browserLocale;
        }

        return $this->defaultLocale;
    }

    private function getLocaleFromBrowser(Request $request): ?string
    {
        $acceptLanguage = $request->header('Accept-Language', '');
        
        if (empty($acceptLanguage)) {
            return null;
        }

        // Parse Accept-Language: th,en-US;q=0.9,en;q=0.8
        $languages = [];
        foreach (explode(',', $acceptLanguage) as $item) {
            $parts = explode(';q=', trim($item));
            $lang = strtolower(trim(substr($parts[0], 0, 2)));
            $q = isset($parts[1]) ? (float) $parts[1] : 1.0;
            $languages[$lang] = $q;
        }

        arsort($languages); // เรียงตาม quality

        foreach (array_keys($languages) as $lang) {
            if (in_array($lang, $this->supportedLocales)) {
                return $lang;
            }
        }

        return null;
    }
}
```

### 8.4 Admin Check Middleware

```php
<?php
// app/Http/Middleware/AdminOnly.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class AdminOnly
{
    public function handle(Request $request, Closure $next): Response
    {
        if (!$request->user()) {
            if ($request->expectsJson()) {
                return response()->json(['message' => 'กรุณาเข้าสู่ระบบก่อน'], 401);
            }
            return redirect()->route('login');
        }

        if (!$request->user()->isAdmin()) {
            if ($request->expectsJson()) {
                return response()->json(['message' => 'ไม่มีสิทธิ์เข้าถึง'], 403);
            }
            abort(403, 'ไม่มีสิทธิ์เข้าถึงส่วนนี้');
        }

        return $next($request);
    }
}
```

### 8.5 API Version Middleware

```php
<?php
// app/Http/Middleware/CheckApiVersion.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class CheckApiVersion
{
    private array $supportedVersions = ['v1', 'v2'];
    private string $latestVersion = 'v2';

    public function handle(Request $request, Closure $next, string $minVersion = 'v1'): Response
    {
        // ดึง version จาก URL path: /api/v1/...
        $version = $this->extractVersion($request);

        if (!$version || !in_array($version, $this->supportedVersions)) {
            return response()->json([
                'message'            => 'API version ไม่รองรับ',
                'supported_versions' => $this->supportedVersions,
            ], 400);
        }

        // เช็ค minimum version
        if (version_compare(ltrim($version, 'v'), ltrim($minVersion, 'v'), '<')) {
            return response()->json([
                'message'       => "กรุณาใช้ API version {$minVersion} ขึ้นไป",
                'latest_version' => $this->latestVersion,
            ], 426); // Upgrade Required
        }

        // เพิ่ม version info ใน response header
        $response = $next($request);
        $response->headers->set('X-API-Version', $version);
        $response->headers->set('X-API-Latest', $this->latestVersion);

        // Warning header ถ้าใช้ version เก่า
        if ($version !== $this->latestVersion) {
            $response->headers->set(
                'Warning',
                "299 - \"API version {$version} is deprecated. Please upgrade to {$this->latestVersion}\""
            );
        }

        return $response;
    }

    private function extractVersion(Request $request): ?string
    {
        $segments = explode('/', trim($request->path(), '/'));
        
        foreach ($segments as $segment) {
            if (preg_match('/^v\d+$/', $segment)) {
                return $segment;
            }
        }

        return null;
    }
}
```

### 8.6 Maintenance Mode Middleware

```php
<?php
// app/Http/Middleware/PreventAccessDuringMaintenance.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class PreventAccessDuringMaintenance
{
    public function handle(Request $request, Closure $next): Response
    {
        // อ่าน maintenance mode จาก config หรือ cache
        $maintenanceMode = \Cache::get('maintenance_mode', false);

        if ($maintenanceMode) {
            // อนุญาตเฉพาะ admin IPs
            $allowedIps = config('maintenance.allowed_ips', []);
            
            if (in_array($request->ip(), $allowedIps)) {
                return $next($request);
            }

            // อนุญาต admin ที่ login
            if ($request->user()?->isSuperAdmin()) {
                return $next($request);
            }

            // ยกเว้น health check endpoint
            if ($request->is('up', 'health')) {
                return $next($request);
            }

            $maintenanceInfo = \Cache::get('maintenance_info', [
                'message'      => 'ระบบปิดปรับปรุงชั่วคราว',
                'expected_back' => null,
            ]);

            if ($request->expectsJson()) {
                return response()->json([
                    'message'      => $maintenanceInfo['message'],
                    'expected_back' => $maintenanceInfo['expected_back'],
                ], 503);
            }

            return response()
                ->view('errors.maintenance', $maintenanceInfo, 503);
        }

        return $next($request);
    }
}
```

### 8.7 Security Headers Middleware

```php
<?php
// app/Http/Middleware/SecurityHeaders.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class SecurityHeaders
{
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);

        // Security Headers
        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('X-Frame-Options', 'SAMEORIGIN');
        $response->headers->set('X-XSS-Protection', '1; mode=block');
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
        
        // Permissions Policy
        $response->headers->set('Permissions-Policy', 'camera=(), microphone=(), geolocation=()');
        
        // Content Security Policy (ปรับตาม needs)
        $response->headers->set('Content-Security-Policy', implode('; ', [
            "default-src 'self'",
            "script-src 'self' 'unsafe-inline' https://cdnjs.cloudflare.com",
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
            "font-src 'self' https://fonts.gstatic.com",
            "img-src 'self' data: https:",
            "connect-src 'self'",
        ]));

        // HSTS (HTTPS only)
        if ($request->isSecure()) {
            $response->headers->set(
                'Strict-Transport-Security',
                'max-age=31536000; includeSubDomains'
            );
        }

        return $response;
    }
}
```

### 8.8 CORS Middleware

```php
<?php
// app/Http/Middleware/HandleCors.php
// ปกติ Laravel มีให้อยู่แล้ว แต่นี่เป็นตัวอย่าง custom

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class HandleCors
{
    private array $allowedOrigins = [
        'http://localhost:3000',
        'https://app.example.com',
    ];

    private array $allowedMethods = ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'];

    private array $allowedHeaders = [
        'Content-Type', 'Authorization', 'X-Requested-With', 'Accept',
    ];

    public function handle(Request $request, Closure $next): Response
    {
        // Handle preflight request
        if ($request->isMethod('OPTIONS')) {
            $response = response('', 200);
            return $this->addCorsHeaders($request, $response);
        }

        $response = $next($request);
        return $this->addCorsHeaders($request, $response);
    }

    private function addCorsHeaders(Request $request, Response $response): Response
    {
        $origin = $request->header('Origin');

        if (in_array($origin, $this->allowedOrigins)) {
            $response->headers->set('Access-Control-Allow-Origin', $origin);
            $response->headers->set('Access-Control-Allow-Methods', implode(', ', $this->allowedMethods));
            $response->headers->set('Access-Control-Allow-Headers', implode(', ', $this->allowedHeaders));
            $response->headers->set('Access-Control-Allow-Credentials', 'true');
            $response->headers->set('Access-Control-Max-Age', '86400'); // Cache preflight 24 ชั่วโมง
        }

        return $response;
    }
}
```

### 8.9 Request Logging Middleware

```php
<?php
// app/Http/Middleware/LogRequests.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\Response;

class LogRequests
{
    // Fields ที่ไม่ควร log (sensitive data)
    private array $sensitiveFields = [
        'password', 'password_confirmation', 'current_password',
        'new_password', 'token', 'secret', 'credit_card', 'cvv',
    ];

    public function handle(Request $request, Closure $next): Response
    {
        $startTime = microtime(true);

        $response = $next($request);

        $duration = round((microtime(true) - $startTime) * 1000, 2);

        $this->log($request, $response, $duration);

        return $response;
    }

    private function log(Request $request, Response $response, float $duration): void
    {
        // ข้าม health check
        if ($request->is('up', 'health', '_debugbar/*')) {
            return;
        }

        $data = [
            'method'      => $request->method(),
            'url'         => $request->fullUrl(),
            'status'      => $response->getStatusCode(),
            'duration_ms' => $duration,
            'user_id'     => auth()->id(),
            'ip'          => $request->ip(),
            'user_agent'  => $request->userAgent(),
        ];

        // Log body สำหรับ non-GET requests (ยกเว้น sensitive fields)
        if (!$request->isMethod('GET') && $request->isJson()) {
            $body = $request->all();
            $data['body'] = $this->maskSensitiveFields($body);
        }

        // Log ตาม status code
        if ($response->getStatusCode() >= 500) {
            Log::error('Server Error', $data);
        } elseif ($response->getStatusCode() >= 400) {
            Log::warning('Client Error', $data);
        } elseif ($duration > 1000) { // slow request (> 1 second)
            Log::warning('Slow Request', $data);
        } else {
            Log::info('Request', $data);
        }
    }

    private function maskSensitiveFields(array $data): array
    {
        foreach ($data as $key => $value) {
            if (in_array(strtolower($key), $this->sensitiveFields)) {
                $data[$key] = '[HIDDEN]';
            } elseif (is_array($value)) {
                $data[$key] = $this->maskSensitiveFields($value);
            }
        }

        return $data;
    }
}
```

### 8.10 Force JSON Response (สำหรับ API)

```php
<?php
// app/Http/Middleware/ForceJsonResponse.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class ForceJsonResponse
{
    public function handle(Request $request, Closure $next): Response
    {
        // บังคับให้ request accept JSON
        $request->headers->set('Accept', 'application/json');

        return $next($request);
    }
}
```

---

## 9. ตัวอย่างการใช้งานครบระบบ

### 9.1 E-commerce API Middleware Stack

```php
<?php
// bootstrap/app.php

->withMiddleware(function (Middleware $middleware) {

    // Global Security Headers
    $middleware->append(\App\Http\Middleware\SecurityHeaders::class);
    $middleware->append(\App\Http\Middleware\LogRequests::class);

    // Web Group
    $middleware->appendToGroup('web', [
        \App\Http\Middleware\SetLocale::class,
        \App\Http\Middleware\CheckUserActive::class,
    ]);

    // API Group
    $middleware->appendToGroup('api', [
        \App\Http\Middleware\ForceJsonResponse::class,
        \App\Http\Middleware\CheckApiVersion::class,
    ]);

    // Aliases
    $middleware->alias([
        'admin'        => \App\Http\Middleware\AdminOnly::class,
        'role'         => \App\Http\Middleware\RequireRole::class,
        'locale'       => \App\Http\Middleware\SetLocale::class,
        'api.version'  => \App\Http\Middleware\CheckApiVersion::class,
        'active'       => \App\Http\Middleware\CheckUserActive::class,
    ]);
})
```

```php
<?php
// routes/api.php

// Public
Route::middleware('throttle:login')->post('auth/login', [AuthController::class, 'login']);
Route::middleware('throttle:60,1')->post('auth/register', [AuthController::class, 'register']);

// Protected API v1
Route::prefix('v1')
    ->middleware(['auth:sanctum', 'active', 'throttle:api', 'api.version:v1'])
    ->group(function () {

        // Products (ดูได้ทุกคน)
        Route::get('products', [ProductController::class, 'index']);
        Route::get('products/{id}', [ProductController::class, 'show']);

        // Orders (ต้อง login)
        Route::apiResource('orders', OrderController::class);

        // Admin only
        Route::middleware('admin')->prefix('admin')->group(function () {
            Route::apiResource('products', Admin\ProductController::class);
            Route::apiResource('users', Admin\UserController::class);
        });
    });
```

---

## Quiz พร้อมเฉลย

### คำถามที่ 1
อธิบายความแตกต่างระหว่าง Global Middleware, Group Middleware, และ Route Middleware

**เฉลย:**
- **Global Middleware**: รันทุก HTTP request ที่เข้ามา (เช่น TrimStrings, ConvertEmptyStringsToNull)
- **Group Middleware**: รันสำหรับ routes ในกลุ่มที่กำหนด (เช่น 'web' group, 'api' group)
- **Route Middleware**: รันเฉพาะ routes ที่กำหนดด้วย ->middleware()

### คำถามที่ 2
Terminable Middleware ต่างจาก After Middleware ธรรมดาอย่างไร?

**เฉลย:**
- **After Middleware**: รันหลัง Controller แต่ก่อนที่ response จะถูกส่งออกไป (บล็อก response)
- **Terminable Middleware**: รันหลังจาก response ส่งออกไปยัง client แล้ว (ไม่บล็อก) เหมาะสำหรับ background tasks เช่น logging, cleanup, analytics

### คำถามที่ 3
จงเขียน Middleware ที่ block requests จาก IP ที่กำหนดไว้ใน blacklist

**เฉลย:**
```php
class BlockIpMiddleware
{
    private array $blockedIps = ['192.168.1.1', '10.0.0.1'];

    public function handle(Request $request, Closure $next): Response
    {
        if (in_array($request->ip(), $this->blockedIps)) {
            abort(403, 'IP ของคุณถูกบล็อก');
        }

        return $next($request);
    }
}
```

### คำถามที่ 4
อธิบายวิธีส่ง parameters ไปยัง Middleware

**เฉลย:**
```php
// กำหนด middleware ให้รับ parameters
public function handle(Request $request, Closure $next, string ...$roles): Response

// ส่ง parameter ใน route definition
Route::get('/admin')->middleware('role:admin,editor');
// หรือ
Route::get('/admin')->middleware(['role:admin,editor']);

// ใน constructor ของ Controller
$this->middleware('throttle:10,1')->only(['store']);
```

### คำถามที่ 5
เมื่อไหรควรใช้ Middleware และเมื่อไหรควรใช้ Policy?

**เฉลย:**
- **Middleware**: ตรวจสอบ authentication, authorization ระดับ route (เช่น ต้อง login, ต้องเป็น admin, rate limiting)
- **Policy**: ตรวจสอบ authorization ระดับ model/resource (เช่น user คนนี้สามารถแก้ไข post นี้ได้ไหม)
- Middleware เหมาะกับการ guard ทั้ง route group
- Policy เหมาะกับการ check permissions บน specific instances

---

## แบบฝึกหัด

### Exercise 1: Subscription Middleware
สร้าง Middleware ที่ตรวจสอบว่า user มี active subscription หรือไม่ ถ้าไม่มี redirect ไปหน้า pricing

### Exercise 2: Country Block Middleware
สร้าง Middleware ที่ block requests จากประเทศที่กำหนด โดยใช้ IP geolocation

### Exercise 3: Cache Response Middleware
สร้าง Middleware ที่ cache GET responses สำหรับ routes ที่กำหนด พร้อม cache invalidation

### Exercise 4: Request Transform Middleware
สร้าง Middleware ที่แปลง camelCase request body เป็น snake_case ก่อนส่งไปยัง Controller

---

## สรุป

| ประเภท | เมื่อไหรรัน | ใช้สำหรับ |
|--------|-----------|---------|
| Global Middleware | ทุก request | Security headers, Logging |
| Group Middleware | Routes ในกลุ่ม | Session, CSRF, Auth |
| Route Middleware | Routes ที่กำหนด | Auth check, Rate limit, Role |
| Terminable | หลัง response ส่งแล้ว | Analytics, Cleanup, Async tasks |

### Best Practices:
1. **Keep Middleware focused** - 1 middleware ทำ 1 สิ่ง
2. **Use Terminable** สำหรับ operations ที่ไม่ต้องรอผล
3. **Order matters** - Middleware รันตามลำดับที่ register
4. **Test Middleware** แยกต่างหากจาก Controller tests
5. **Avoid heavy operations** ใน Global Middleware (รันทุก request)

---

## ลิงก์ไป Part ถัดไป

➡️ [Part 036: Laravel File Storage](part-036-laravel-file-storage.md)

ใน Part ถัดไปเราจะเรียนเรื่อง File Storage ซึ่งรวม Local, S3, Cloudflare R2 และการจัดการไฟล์ใน Laravel
