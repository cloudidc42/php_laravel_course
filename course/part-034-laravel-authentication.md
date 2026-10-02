# Part 034: Laravel Authentication

## ระดับ: Intermediate-Advanced
## เวลาที่ใช้: 5-6 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ติดตั้งและใช้งาน Laravel Breeze สำหรับ web authentication
- ใช้ Laravel Sanctum สำหรับ API token authentication
- กำหนด Guards และ Providers สำหรับ multi-auth
- ใช้ Policies และ Gates สำหรับ authorization
- ตั้งค่า Email Verification
- สร้างระบบ Authentication ครบวงจร

---

## 1. Laravel Authentication Overview

Laravel มีเครื่องมือสำหรับ Authentication หลายแบบ:

| เครื่องมือ | ใช้สำหรับ | ประเภท |
|-----------|---------|-------|
| **Breeze** | เริ่มต้นทำ web auth | Starter Kit |
| **Jetstream** | Auth พร้อม Teams, 2FA | Starter Kit |
| **Sanctum** | SPA และ Mobile API | Package |
| **Passport** | Full OAuth2 Server | Package |
| **Fortify** | Backend auth headless | Package |

---

## 2. Laravel Breeze

### 2.1 การติดตั้ง

```bash
# ติดตั้ง Breeze
composer require laravel/breeze --dev

# Publish และติดตั้ง
php artisan breeze:install

# เลือก stack:
# blade (default)
# vue (Inertia.js + Vue)
# react (Inertia.js + React)
# api (headless, ไม่มี views)
# livewire

# สำหรับ Blade stack
php artisan breeze:install blade

# ติดตั้ง dependencies และ build
npm install && npm run dev

# รัน migration
php artisan migrate
```

### 2.2 Routes ที่ Breeze สร้างให้

```php
<?php
// routes/auth.php (สร้างโดย Breeze)

Route::middleware('guest')->group(function () {
    Route::get('register', [RegisteredUserController::class, 'create'])
        ->name('register');
    Route::post('register', [RegisteredUserController::class, 'store']);
    
    Route::get('login', [AuthenticatedSessionController::class, 'create'])
        ->name('login');
    Route::post('login', [AuthenticatedSessionController::class, 'store']);
    
    Route::get('forgot-password', [PasswordResetLinkController::class, 'create'])
        ->name('password.request');
    Route::post('forgot-password', [PasswordResetLinkController::class, 'store'])
        ->name('password.email');
    
    Route::get('reset-password/{token}', [NewPasswordController::class, 'create'])
        ->name('password.reset');
    Route::post('reset-password', [NewPasswordController::class, 'store'])
        ->name('password.store');
});

Route::middleware('auth')->group(function () {
    Route::get('verify-email', EmailVerificationPromptController::class)
        ->name('verification.notice');
    Route::get('verify-email/{id}/{hash}', VerifyEmailController::class)
        ->middleware(['signed', 'throttle:6,1'])
        ->name('verification.verify');
    Route::post('email/verification-notification', [EmailVerificationNotificationController::class, 'store'])
        ->middleware('throttle:6,1')
        ->name('verification.send');
    
    Route::get('confirm-password', [ConfirmablePasswordController::class, 'show'])
        ->name('password.confirm');
    Route::post('confirm-password', [ConfirmablePasswordController::class, 'store']);
    
    Route::put('password', [PasswordController::class, 'update'])->name('password.update');
    Route::post('logout', [AuthenticatedSessionController::class, 'destroy'])
        ->name('logout');
});
```

### 2.3 Auth Helper Functions

```php
<?php
// การใช้งาน Auth

// ดึง user ที่ login อยู่
$user = auth()->user();
$user = Auth::user();

// ดึง user ID
$userId = auth()->id();
$userId = Auth::id();

// เช็คว่า login อยู่หรือเปล่า
if (auth()->check()) {
    echo "Login แล้ว";
}

if (Auth::check()) {
    echo "Login แล้ว";
}

// เช็คว่ายังไม่ได้ login
if (auth()->guest()) {
    return redirect()->route('login');
}

// Login
Auth::login($user);
Auth::login($user, $remember = true); // remember me

// Login ด้วย credentials
$credentials = ['email' => 'user@example.com', 'password' => 'secret'];
if (Auth::attempt($credentials)) {
    // Login สำเร็จ
}

// Login once (ไม่ save session)
Auth::once($credentials);

// Logout
Auth::logout();
$request->session()->invalidate();
$request->session()->regenerateToken();
```

---

## 3. Laravel Sanctum

### 3.1 การติดตั้ง

```bash
# ติดตั้ง Sanctum
composer require laravel/sanctum

# Publish config และ migration
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"

# รัน migration (สร้าง personal_access_tokens table)
php artisan migrate
```

### 3.2 การตั้งค่า

```php
<?php
// app/Models/User.php

use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
    // ...
}
```

```php
<?php
// config/sanctum.php

return [
    'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', sprintf(
        '%s%s',
        'localhost,localhost:3000,127.0.0.1,127.0.0.1:8000,::1',
        Sanctum::currentApplicationUrlWithPort()
    ))),

    'guard' => ['web'],

    'expiration' => null, // null = ไม่หมดอายุ, หรือใส่จำนวนนาที

    'token_prefix' => env('SANCTUM_TOKEN_PREFIX', ''),

    'middleware' => [
        'authenticate_session' => Laravel\Sanctum\Http\Middleware\AuthenticateSession::class,
        'encrypt_cookies'      => App\Http\Middleware\EncryptCookies::class,
        'verify_csrf_token'    => App\Http\Middleware\VerifyCsrfToken::class,
    ],
];
```

### 3.3 API Authentication Controller

```php
<?php
// app/Http/Controllers/Api/AuthController.php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rules\Password;

class AuthController extends Controller
{
    /**
     * สมัครสมาชิก
     */
    public function register(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'name'     => 'required|string|max:255',
            'email'    => 'required|email|unique:users',
            'password' => ['required', 'confirmed', Password::defaults()],
        ]);

        $user = User::create([
            'name'     => $validated['name'],
            'email'    => $validated['email'],
            'password' => Hash::make($validated['password']),
        ]);

        $token = $user->createToken('auth_token')->plainTextToken;

        return response()->json([
            'message'      => 'สมัครสมาชิกสำเร็จ',
            'user'         => $user,
            'access_token' => $token,
            'token_type'   => 'Bearer',
        ], 201);
    }

    /**
     * Login
     */
    public function login(Request $request): JsonResponse
    {
        $credentials = $request->validate([
            'email'    => 'required|email',
            'password' => 'required|string',
        ]);

        if (!Auth::attempt($credentials)) {
            return response()->json([
                'message' => 'อีเมลหรือรหัสผ่านไม่ถูกต้อง',
            ], 401);
        }

        $user = Auth::user();

        // ลบ tokens เก่าก่อน (optional)
        // $user->tokens()->delete();

        // สร้าง token ใหม่
        $token = $user->createToken(
            name: 'auth_token',
            abilities: ['*'],                        // permissions ทั้งหมด
            expiresAt: now()->addDays(30),           // หมดอายุใน 30 วัน
        )->plainTextToken;

        return response()->json([
            'message'      => 'Login สำเร็จ',
            'user'         => $user,
            'access_token' => $token,
            'token_type'   => 'Bearer',
            'expires_at'   => now()->addDays(30)->toDateTimeString(),
        ]);
    }

    /**
     * ดึงข้อมูล user ปัจจุบัน
     */
    public function me(Request $request): JsonResponse
    {
        return response()->json([
            'user' => $request->user(),
        ]);
    }

    /**
     * Logout (revoke current token)
     */
    public function logout(Request $request): JsonResponse
    {
        $request->user()->currentAccessToken()->delete();

        return response()->json([
            'message' => 'Logout สำเร็จ',
        ]);
    }

    /**
     * Logout ทุก devices
     */
    public function logoutAll(Request $request): JsonResponse
    {
        $request->user()->tokens()->delete();

        return response()->json([
            'message' => 'Logout จากทุก devices สำเร็จ',
        ]);
    }

    /**
     * ดู tokens ทั้งหมด
     */
    public function tokens(Request $request): JsonResponse
    {
        $tokens = $request->user()->tokens()->select([
            'id', 'name', 'abilities', 'last_used_at', 'expires_at', 'created_at'
        ])->get();

        return response()->json(['tokens' => $tokens]);
    }

    /**
     * ลบ token ที่กำหนด
     */
    public function revokeToken(Request $request, int $tokenId): JsonResponse
    {
        $request->user()->tokens()->where('id', $tokenId)->delete();

        return response()->json(['message' => 'ลบ token สำเร็จ']);
    }
}
```

### 3.4 Token Abilities (Scopes)

```php
<?php
// สร้าง token พร้อม abilities
$token = $user->createToken('mobile-app', [
    'products:read',
    'products:write',
    'orders:read',
    // ไม่มี 'admin:*'
])->plainTextToken;

// ใน Middleware/Controller เช็ค ability
Route::middleware(['auth:sanctum'])->group(function () {
    
    Route::get('/products', function (Request $request) {
        // เช็ค ability
        if (!$request->user()->tokenCan('products:read')) {
            abort(403, 'Token ไม่มีสิทธิ์');
        }
        // ...
    });

    Route::post('/products', function (Request $request) {
        if (!$request->user()->tokenCan('products:write')) {
            abort(403, 'Token ไม่มีสิทธิ์');
        }
        // ...
    });
});
```

### 3.5 Routes สำหรับ API

```php
<?php
// routes/api.php

use App\Http\Controllers\Api\AuthController;

// Public routes
Route::prefix('auth')->group(function () {
    Route::post('register', [AuthController::class, 'register']);
    Route::post('login',    [AuthController::class, 'login']);
});

// Protected routes
Route::middleware('auth:sanctum')->group(function () {
    Route::prefix('auth')->group(function () {
        Route::get('me',              [AuthController::class, 'me']);
        Route::post('logout',         [AuthController::class, 'logout']);
        Route::post('logout-all',     [AuthController::class, 'logoutAll']);
        Route::get('tokens',          [AuthController::class, 'tokens']);
        Route::delete('tokens/{id}',  [AuthController::class, 'revokeToken']);
    });

    // Other protected routes
    Route::apiResource('products', ProductController::class);
    Route::apiResource('orders', OrderController::class);
});
```

---

## 4. Guards & Providers

### 4.1 ทำความเข้าใจ Guards

Guard กำหนดวิธีที่ User ถูก authenticate ต่อแต่ละ request

```php
<?php
// config/auth.php

return [
    'defaults' => [
        'guard'     => 'web',
        'passwords' => 'users',
    ],

    'guards' => [
        // Web guard: ใช้ session
        'web' => [
            'driver'   => 'session',
            'provider' => 'users',
        ],
        
        // API guard: ใช้ token (Sanctum)
        'api' => [
            'driver'   => 'sanctum',
            'provider' => 'users',
        ],

        // Admin guard: session แยกสำหรับ admin
        'admin' => [
            'driver'   => 'session',
            'provider' => 'admins',
        ],
    ],

    'providers' => [
        // Users provider: ดึงข้อมูลจาก User model
        'users' => [
            'driver' => 'eloquent',
            'model'  => App\Models\User::class,
        ],

        // Admins provider: ดึงข้อมูลจาก Admin model แยก
        'admins' => [
            'driver' => 'eloquent',
            'model'  => App\Models\Admin::class,
        ],
    ],
];
```

### 4.2 Multi-Auth System

```php
<?php
// app/Models/Admin.php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class Admin extends Authenticatable
{
    use Notifiable;

    protected $table = 'admins';
    protected $fillable = ['name', 'email', 'password', 'role'];
    protected $hidden = ['password', 'remember_token'];
    protected $casts = ['password' => 'hashed'];
}
```

```php
<?php
// app/Http/Controllers/Admin/AuthController.php

namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class AuthController extends Controller
{
    public function showLoginForm()
    {
        return view('admin.auth.login');
    }

    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email'    => 'required|email',
            'password' => 'required',
        ]);

        // ใช้ 'admin' guard
        if (Auth::guard('admin')->attempt($credentials)) {
            $request->session()->regenerate();
            return redirect()->route('admin.dashboard');
        }

        return back()->withErrors([
            'email' => 'Email หรือ password ไม่ถูกต้อง',
        ]);
    }

    public function logout(Request $request)
    {
        Auth::guard('admin')->logout();
        $request->session()->invalidate();
        return redirect()->route('admin.login');
    }
}
```

```php
<?php
// app/Http/Middleware/AdminAuth.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class AdminAuth
{
    public function handle(Request $request, Closure $next)
    {
        if (!Auth::guard('admin')->check()) {
            return redirect()->route('admin.login')
                ->with('error', 'กรุณาเข้าสู่ระบบ Admin');
        }

        return $next($request);
    }
}
```

```php
<?php
// routes/admin.php

Route::prefix('admin')->name('admin.')->group(function () {
    // Public admin routes
    Route::middleware('guest:admin')->group(function () {
        Route::get('login',  [Admin\AuthController::class, 'showLoginForm'])->name('login');
        Route::post('login', [Admin\AuthController::class, 'login']);
    });

    // Protected admin routes
    Route::middleware('admin.auth')->group(function () {
        Route::get('dashboard', [Admin\DashboardController::class, 'index'])->name('dashboard');
        Route::resource('users', Admin\UserController::class);
        Route::post('logout', [Admin\AuthController::class, 'logout'])->name('logout');
    });
});
```

---

## 5. Policies & Gates

### 5.1 Gates (Simple Authorization)

```php
<?php
// app/Providers/AppServiceProvider.php

use Illuminate\Support\Facades\Gate;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Define gate
        Gate::define('view-admin-dashboard', function ($user) {
            return in_array($user->role, ['admin', 'super_admin']);
        });

        Gate::define('edit-post', function ($user, Post $post) {
            return $user->id === $post->user_id || $user->role === 'admin';
        });

        Gate::define('delete-post', function ($user, Post $post) {
            return $user->id === $post->user_id && $user->role !== 'viewer';
        });

        // Before callback: ถ้า return true จะ pass ทุก gate
        Gate::before(function ($user, $ability) {
            if ($user->isSuperAdmin()) {
                return true;
            }
        });

        // After callback
        Gate::after(function ($user, $ability, $result, $arguments) {
            // log authorization
        });
    }
}
```

```php
<?php
// การใช้งาน Gates

// ใน Controller
if (Gate::allows('edit-post', $post)) {
    // อนุญาต
}

if (Gate::denies('edit-post', $post)) {
    abort(403);
}

// ใช้ authorize (throw AuthorizationException ถ้าไม่ผ่าน)
Gate::authorize('edit-post', $post);

// ใน Blade
@can('edit-post', $post)
    <button>แก้ไข</button>
@endcan

@cannot('delete-post', $post)
    <p>ไม่มีสิทธิ์ลบ</p>
@endcannot

@canany(['edit-post', 'delete-post'], $post)
    <div>Admin actions</div>
@endcanany

// ใน Controller ผ่าน $request->user()
if ($request->user()->can('edit-post', $post)) {
    // ...
}

if ($request->user()->cannot('delete-post', $post)) {
    abort(403);
}
```

### 5.2 Policies (Resource-based Authorization)

```bash
# สร้าง Policy
php artisan make:policy PostPolicy
php artisan make:policy PostPolicy --model=Post
```

```php
<?php
// app/Policies/PostPolicy.php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\Response;

class PostPolicy
{
    /**
     * ถ้า return true/false ก่อน method อื่นๆ จะ override ทั้งหมด
     */
    public function before(User $user, string $ability): bool|null
    {
        // Super admin bypass ทุก policy
        if ($user->role === 'super_admin') {
            return true;
        }

        // return null = ดำเนินการต่อไป
        return null;
    }

    /**
     * ดู posts ทั้งหมด
     */
    public function viewAny(User $user): bool
    {
        return true; // ทุกคนดูได้
    }

    /**
     * ดู post เดี่ยว
     */
    public function view(User $user, Post $post): bool
    {
        return $post->status === 'published' || $user->id === $post->user_id;
    }

    /**
     * สร้าง post
     */
    public function create(User $user): bool
    {
        return in_array($user->role, ['admin', 'editor', 'author']);
    }

    /**
     * อัพเดต post
     */
    public function update(User $user, Post $post): bool|Response
    {
        if ($user->id === $post->user_id) {
            return true;
        }

        if ($user->role === 'admin') {
            return true;
        }

        // คืน Response object สำหรับ custom message
        return Response::deny('คุณสามารถแก้ไขเฉพาะ post ของตัวเองเท่านั้น');
    }

    /**
     * ลบ post
     */
    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id || $user->role === 'admin';
    }

    /**
     * Restore soft deleted post
     */
    public function restore(User $user, Post $post): bool
    {
        return $user->role === 'admin';
    }

    /**
     * Force delete
     */
    public function forceDelete(User $user, Post $post): bool
    {
        return $user->role === 'super_admin';
    }

    /**
     * Publish post
     */
    public function publish(User $user, Post $post): bool
    {
        return ($user->id === $post->user_id && $user->role !== 'author')
            || $user->role === 'admin';
    }
}
```

### 5.3 Register Policy

```php
<?php
// app/Providers/AppServiceProvider.php

use App\Models\Post;
use App\Policies\PostPolicy;
use Illuminate\Support\Facades\Gate;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Register manually
        Gate::policy(Post::class, PostPolicy::class);
    }
}
```

หรือใช้ auto-discovery (Laravel 9+):
```php
<?php
// app/Providers/AuthServiceProvider.php (Laravel 8 และก่อนหน้า)

protected $policies = [
    Post::class => PostPolicy::class,
];
```

### 5.4 การใช้งาน Policies

```php
<?php
// ใน Controller
class PostController extends Controller
{
    public function index()
    {
        // $this->authorize('viewAny', Post::class);
        return Post::all();
    }

    public function show(Post $post)
    {
        $this->authorize('view', $post);
        return $post;
    }

    public function store(Request $request)
    {
        $this->authorize('create', Post::class);
        // ...
    }

    public function update(Request $request, Post $post)
    {
        $this->authorize('update', $post);
        // ถ้าไม่ผ่าน จะ throw AuthorizationException -> HTTP 403
        $post->update($request->validated());
        return $post;
    }

    public function destroy(Post $post)
    {
        $this->authorize('delete', $post);
        $post->delete();
        return response()->noContent();
    }
}
```

```php
<?php
// ใน Blade Templates

@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">แก้ไข</a>
@endcan

@can('delete', $post)
    <form method="POST" action="{{ route('posts.destroy', $post) }}">
        @csrf
        @method('DELETE')
        <button type="submit">ลบ</button>
    </form>
@endcan

@can('publish', $post)
    <button>Publish</button>
@endcan

// ตรวจสอบหลาย abilities
@canany(['update', 'delete'], $post)
    <div class="admin-actions">
        ...
    </div>
@endcanany
```

```php
<?php
// ใช้กับ Resource Controller (automatic policy mapping)
class PostController extends Controller
{
    public function __construct()
    {
        // ทำ authorization โดยอัตโนมัติสำหรับทุก method
        $this->authorizeResource(Post::class, 'post');
        
        // index -> viewAny
        // show -> view
        // create/store -> create
        // edit/update -> update
        // destroy -> delete
    }
}
```

---

## 6. Email Verification

### 6.1 ตั้งค่า

```php
<?php
// app/Models/User.php

use Illuminate\Contracts\Auth\MustVerifyEmail;

class User extends Authenticatable implements MustVerifyEmail
{
    // ...
}
```

```php
<?php
// routes/web.php

Auth::routes(['verify' => true]);

// หรือกับ Breeze:
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
});
```

### 6.2 Custom Verification Email

```php
<?php
// app/Notifications/VerifyEmailNotification.php

namespace App\Notifications;

use Illuminate\Auth\Notifications\VerifyEmail;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Support\Carbon;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\URL;

class VerifyEmailNotification extends VerifyEmail
{
    protected function buildMailMessage($url): MailMessage
    {
        return (new MailMessage)
            ->subject('ยืนยันอีเมลของคุณ')
            ->greeting("สวัสดีคุณ {$this->notifiable->name}!")
            ->line('กรุณากดปุ่มด้านล่างเพื่อยืนยันอีเมลของคุณ')
            ->action('ยืนยันอีเมล', $url)
            ->line('ลิงก์นี้จะหมดอายุใน 60 นาที')
            ->line('หากคุณไม่ได้สมัครสมาชิก กรุณาเพิกเฉยต่ออีเมลนี้');
    }

    protected function verificationUrl($notifiable): string
    {
        return URL::temporarySignedRoute(
            'verification.verify',
            Carbon::now()->addMinutes(Config::get('auth.verification.expire', 60)),
            [
                'id'   => $notifiable->getKey(),
                'hash' => sha1($notifiable->getEmailForVerification()),
            ]
        );
    }
}
```

```php
<?php
// app/Models/User.php

use App\Notifications\VerifyEmailNotification;

class User extends Authenticatable implements MustVerifyEmail
{
    public function sendEmailVerificationNotification(): void
    {
        $this->notify(new VerifyEmailNotification());
    }
}
```

---

## 7. Workshop: สร้าง Auth System ครบวงจร

### 7.1 User Roles System

```php
<?php
// database/migrations/add_role_to_users_table.php

Schema::table('users', function (Blueprint $table) {
    $table->enum('role', ['user', 'author', 'editor', 'admin', 'super_admin'])
        ->default('user')
        ->after('email');
    $table->boolean('is_active')->default(true)->after('role');
    $table->timestamp('last_login_at')->nullable();
    $table->string('last_login_ip', 45)->nullable();
});
```

```php
<?php
// app/Enums/UserRole.php

namespace App\Enums;

enum UserRole: string
{
    case User       = 'user';
    case Author     = 'author';
    case Editor     = 'editor';
    case Admin      = 'admin';
    case SuperAdmin = 'super_admin';

    public function label(): string
    {
        return match($this) {
            self::User      => 'ผู้ใช้งาน',
            self::Author    => 'นักเขียน',
            self::Editor    => 'บรรณาธิการ',
            self::Admin     => 'ผู้ดูแล',
            self::SuperAdmin => 'ผู้ดูแลสูงสุด',
        };
    }

    public function permissions(): array
    {
        return match($this) {
            self::User      => ['read'],
            self::Author    => ['read', 'post:create', 'post:edit-own'],
            self::Editor    => ['read', 'post:create', 'post:edit', 'post:publish'],
            self::Admin     => ['read', 'post:*', 'user:read', 'user:edit'],
            self::SuperAdmin => ['*'],
        };
    }

    public static function values(): array
    {
        return array_column(self::cases(), 'value');
    }
}
```

```php
<?php
// app/Models/User.php

use App\Enums\UserRole;

class User extends Authenticatable implements MustVerifyEmail
{
    use HasApiTokens, HasFactory, Notifiable;

    protected $fillable = [
        'name', 'email', 'password', 'role', 'is_active',
    ];

    protected $hidden = ['password', 'remember_token'];

    protected $casts = [
        'email_verified_at' => 'datetime',
        'last_login_at'     => 'datetime',
        'password'          => 'hashed',
        'is_active'         => 'boolean',
        'role'              => UserRole::class,
    ];

    // Role check methods
    public function hasRole(UserRole|string $role): bool
    {
        $roleValue = $role instanceof UserRole ? $role->value : $role;
        return $this->role->value === $roleValue;
    }

    public function hasAnyRole(array $roles): bool
    {
        foreach ($roles as $role) {
            if ($this->hasRole($role)) {
                return true;
            }
        }
        return false;
    }

    public function isAdmin(): bool
    {
        return $this->hasAnyRole(['admin', 'super_admin']);
    }

    public function isSuperAdmin(): bool
    {
        return $this->hasRole(UserRole::SuperAdmin);
    }

    public function hasPermission(string $permission): bool
    {
        $permissions = $this->role->permissions();
        
        if (in_array('*', $permissions)) {
            return true;
        }

        foreach ($permissions as $p) {
            if ($p === $permission) {
                return true;
            }

            // Wildcard: post:* matches post:create, post:edit, etc.
            if (str_ends_with($p, ':*')) {
                $prefix = rtrim($p, ':*');
                if (str_starts_with($permission, $prefix . ':')) {
                    return true;
                }
            }
        }

        return false;
    }

    // Update last login
    public function updateLastLogin(string $ip): void
    {
        $this->update([
            'last_login_at' => now(),
            'last_login_ip' => $ip,
        ]);
    }
}
```

### 7.2 Authentication Middleware

```php
<?php
// app/Http/Middleware/CheckUserActive.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class CheckUserActive
{
    public function handle(Request $request, Closure $next)
    {
        if (Auth::check() && !Auth::user()->is_active) {
            Auth::logout();
            $request->session()->invalidate();
            
            return redirect()->route('login')
                ->with('error', 'บัญชีของคุณถูกระงับ กรุณาติดต่อผู้ดูแลระบบ');
        }

        return $next($request);
    }
}
```

```php
<?php
// app/Http/Middleware/TrackLastLogin.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class TrackLastLogin
{
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);

        // อัพเดต last login หลังจาก login สำเร็จ
        if (Auth::check() && $request->isMethod('POST') && $request->routeIs('login')) {
            Auth::user()->updateLastLogin($request->ip());
        }

        return $response;
    }
}
```

```php
<?php
// app/Http/Middleware/RequireRole.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class RequireRole
{
    public function handle(Request $request, Closure $next, string ...$roles)
    {
        if (!Auth::check()) {
            return redirect()->route('login');
        }

        $user = Auth::user();
        
        foreach ($roles as $role) {
            if ($user->hasRole($role)) {
                return $next($request);
            }
        }

        abort(403, 'ไม่มีสิทธิ์เข้าถึง');
    }
}
```

### 7.3 Register Middleware

```php
<?php
// bootstrap/app.php (Laravel 11)

use App\Http\Middleware\CheckUserActive;
use App\Http\Middleware\RequireRole;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(...)
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->appendToGroup('web', [
            CheckUserActive::class,
        ]);

        $middleware->alias([
            'role' => RequireRole::class,
        ]);
    })
    ->create();
```

### 7.4 Routes

```php
<?php
// routes/web.php

// Auth routes (Breeze)
require __DIR__.'/auth.php';

// Protected routes
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
    Route::get('/profile',   [ProfileController::class, 'edit'])->name('profile.edit');
    Route::patch('/profile', [ProfileController::class, 'update'])->name('profile.update');
});

// Role-based routes
Route::middleware(['auth', 'verified', 'role:author,editor,admin'])->group(function () {
    Route::resource('posts', PostController::class);
});

Route::middleware(['auth', 'verified', 'role:admin,super_admin'])->prefix('admin')->name('admin.')->group(function () {
    Route::get('/', [Admin\DashboardController::class, 'index'])->name('dashboard');
    Route::resource('users', Admin\UserController::class);
});
```

```php
<?php
// routes/api.php

use App\Http\Controllers\Api\AuthController;

// Public
Route::prefix('v1')->group(function () {
    Route::post('auth/register', [AuthController::class, 'register']);
    Route::post('auth/login',    [AuthController::class, 'login']);
    Route::post('auth/forgot-password', [ForgotPasswordController::class, 'store']);
    Route::post('auth/reset-password',  [ResetPasswordController::class, 'store']);
});

// Protected
Route::prefix('v1')->middleware(['auth:sanctum'])->group(function () {
    Route::prefix('auth')->group(function () {
        Route::get('me',         [AuthController::class, 'me']);
        Route::post('logout',    [AuthController::class, 'logout']);
        Route::post('logout-all',[AuthController::class, 'logoutAll']);
        Route::put('password',   [ChangePasswordController::class, 'update']);
        Route::post('refresh',   [AuthController::class, 'refresh']);
    });

    // User routes
    Route::get('profile', [ProfileController::class, 'show']);
    Route::put('profile', [ProfileController::class, 'update']);

    // Posts
    Route::apiResource('posts', PostController::class);
});
```

### 7.5 Complete Auth Flow (API)

```php
<?php
// app/Http/Controllers/Api/AuthController.php (ฉบับสมบูรณ์)

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Password;
use Illuminate\Support\Str;
use Illuminate\Auth\Events\PasswordReset;
use Illuminate\Validation\Rules\Password as PasswordRule;

class AuthController extends Controller
{
    public function register(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'name'     => 'required|string|max:255',
            'email'    => 'required|email|unique:users',
            'password' => ['required', 'confirmed', PasswordRule::defaults()],
        ]);

        $user = User::create([
            'name'     => $validated['name'],
            'email'    => $validated['email'],
            'password' => Hash::make($validated['password']),
        ]);

        $user->sendEmailVerificationNotification();

        $token = $user->createToken('auth_token')->plainTextToken;

        return response()->json([
            'message'      => 'สมัครสมาชิกสำเร็จ กรุณายืนยันอีเมลของคุณ',
            'user'         => $user,
            'access_token' => $token,
            'token_type'   => 'Bearer',
        ], 201);
    }

    public function login(Request $request): JsonResponse
    {
        $credentials = $request->validate([
            'email'    => 'required|email',
            'password' => 'required|string',
        ]);

        if (!Auth::attempt($credentials)) {
            return response()->json([
                'message' => 'อีเมลหรือรหัสผ่านไม่ถูกต้อง',
            ], 401);
        }

        $user = Auth::user();

        if (!$user->is_active) {
            Auth::logout();
            return response()->json([
                'message' => 'บัญชีของคุณถูกระงับ',
            ], 403);
        }

        $user->updateLastLogin($request->ip());

        $token = $user->createToken(
            name: 'auth_token',
            expiresAt: now()->addDays(30),
        )->plainTextToken;

        return response()->json([
            'message'      => 'Login สำเร็จ',
            'user'         => $user,
            'access_token' => $token,
            'token_type'   => 'Bearer',
        ]);
    }

    public function forgotPassword(Request $request): JsonResponse
    {
        $request->validate(['email' => 'required|email']);

        $status = Password::sendResetLink($request->only('email'));

        return $status === Password::RESET_LINK_SENT
            ? response()->json(['message' => 'ส่งลิงก์รีเซ็ตรหัสผ่านไปยังอีเมลของคุณแล้ว'])
            : response()->json(['message' => 'ไม่พบอีเมลนี้ในระบบ'], 404);
    }

    public function resetPassword(Request $request): JsonResponse
    {
        $request->validate([
            'token'                 => 'required',
            'email'                 => 'required|email',
            'password'              => ['required', 'confirmed', PasswordRule::defaults()],
        ]);

        $status = Password::reset(
            $request->only('email', 'password', 'password_confirmation', 'token'),
            function (User $user, string $password) {
                $user->forceFill(['password' => Hash::make($password)])
                     ->setRememberToken(Str::random(60));
                $user->save();
                event(new PasswordReset($user));
            }
        );

        return $status === Password::PASSWORD_RESET
            ? response()->json(['message' => 'รีเซ็ตรหัสผ่านสำเร็จ'])
            : response()->json(['message' => 'ลิงก์หมดอายุหรือไม่ถูกต้อง'], 422);
    }

    public function me(Request $request): JsonResponse
    {
        return response()->json(['user' => $request->user()]);
    }

    public function logout(Request $request): JsonResponse
    {
        $request->user()->currentAccessToken()->delete();
        return response()->json(['message' => 'Logout สำเร็จ']);
    }
}
```

---

## Quiz พร้อมเฉลย

### คำถามที่ 1
ความแตกต่างระหว่าง Policies และ Gates คืออะไร?

**เฉลย:**
- **Gates**: Simple authorization เหมาะสำหรับ actions ที่ไม่เกี่ยวกับ Model เฉพาะ หรือ global actions เช่น `view-admin-panel`
- **Policies**: Resource-based authorization เหมาะสำหรับ CRUD operations บน Models เฉพาะ เช่น Post, User
- Gates: ใช้ `Gate::define()` ใน ServiceProvider
- Policies: สร้างเป็น class แยก มี methods สำหรับแต่ละ action

### คำถามที่ 2
อธิบายความแตกต่างระหว่าง Breeze, Sanctum, Passport

**เฉลย:**
- **Breeze**: Starter Kit สำหรับ web authentication (session-based), มี views พร้อมใช้
- **Sanctum**: ไม่มี views, เหมาะสำหรับ SPA authentication และ Mobile API token, ง่ายกว่า Passport
- **Passport**: Full OAuth2 server implementation, เหมาะสำหรับ third-party apps ที่ต้องการ OAuth2 flows

### คำถามที่ 3
เขียนตัวอย่าง Gate ที่ตรวจสอบว่า user สามารถ edit post ของคนอื่นได้เฉพาะเมื่อเป็น admin

**เฉลย:**
```php
Gate::define('edit-post', function (User $user, Post $post) {
    // เจ้าของ post ทุกคน
    if ($user->id === $post->user_id) {
        return true;
    }
    // Admin edit ได้ทุก post
    return $user->role === 'admin';
});

// ใช้งาน:
if (Gate::allows('edit-post', $post)) { ... }
// หรือ
$this->authorize('edit-post', $post);
```

### คำถามที่ 4
Token Abilities ใน Sanctum ใช้ทำอะไร?

**เฉลย:**
Token Abilities คือ scopes ของ API token ที่กำหนดว่า token นั้นสามารถทำอะไรได้บ้าง เหมือน OAuth2 scopes:
```php
// สร้าง token พร้อม abilities
$token = $user->createToken('mobile-app', [
    'products:read',
    'orders:create',
])->plainTextToken;

// เช็ค ability
if ($user->tokenCan('products:read')) { ... }
```

### คำถามที่ 5
`before()` method ใน Policy ใช้ทำอะไร?

**เฉลย:**
`before()` method รันก่อน method อื่นๆ ทั้งหมดใน Policy ถ้า return `true` จะ pass ทุก authorization, ถ้า return `false` จะ deny ทุก authorization, ถ้า return `null` จะดำเนินการต่อใน method ปกติ ใช้สำหรับ Super Admin ที่ bypass authorization ทั้งหมด

---

## แบบฝึกหัด

### Exercise 1: API Rate Limiting
เพิ่ม rate limiting สำหรับ API authentication:
- Login: ไม่เกิน 5 ครั้งต่อ 1 นาที
- Register: ไม่เกิน 3 ครั้งต่อ 10 นาที
- Forgot password: ไม่เกิน 3 ครั้งต่อ 1 ชั่วโมง

### Exercise 2: Two-Factor Authentication
เพิ่ม 2FA ด้วย OTP ส่งผ่าน SMS:
- เมื่อ login สำเร็จ ส่ง OTP ไปที่เบอร์โทร
- ผู้ใช้ต้องกรอก OTP ภายใน 5 นาที
- OTP ใช้ได้ครั้งเดียว

### Exercise 3: Social Login
เพิ่ม Social Login ด้วย Google และ Facebook ด้วย Laravel Socialite

---

## สรุป

| หัวข้อ | เครื่องมือ | ใช้เมื่อ |
|--------|-----------|---------|
| Web Auth | Breeze, Fortify | Web app ที่ใช้ session |
| API Auth | Sanctum | SPA, Mobile App |
| OAuth2 | Passport | Third-party app integration |
| Authorization | Gates | Simple, non-model actions |
| Authorization | Policies | CRUD operations บน models |
| Email Verify | Built-in | ยืนยัน email |

---

## ลิงก์ไป Part ถัดไป

➡️ [Part 035: Laravel Middleware](part-035-laravel-middleware.md)

ใน Part ถัดไปเราจะเรียนเรื่อง Middleware ซึ่งเป็น "ประตูทาง" ก่อนที่ request จะถึง Controller
