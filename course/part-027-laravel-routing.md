# Part 027: Laravel Routing

**ระดับ: กลาง (Intermediate)**

---

## เป้าหมายของ Part นี้

หลังจากเรียน Part นี้จบ คุณจะสามารถ:
- กำหนด routes พื้นฐานทุกรูปแบบ HTTP method ได้
- ใช้ route parameters ทั้ง required และ optional
- ตั้งชื่อ routes และใช้ named routes
- จัดกลุ่ม routes ด้วย Route Groups
- ใช้ Middleware กับ routes
- สร้าง API routes ที่ถูกต้อง
- ใช้ Resource routes สำหรับ CRUD

---

## 1. Basic Routing

Routes ใน Laravel กำหนดใน `routes/web.php` (สำหรับ web) และ `routes/api.php` (สำหรับ API)

### HTTP Methods ทั้งหมด

```php
// routes/web.php
<?php

use Illuminate\Support\Facades\Route;

// GET - ดึงข้อมูล
Route::get('/posts', function () {
    return 'รายการบทความ';
});

// POST - สร้างข้อมูล
Route::post('/posts', function () {
    return 'สร้างบทความ';
});

// PUT - อัปเดตข้อมูลทั้งหมด
Route::put('/posts/{id}', function ($id) {
    return "อัปเดตบทความ {$id}";
});

// PATCH - อัปเดตข้อมูลบางส่วน
Route::patch('/posts/{id}', function ($id) {
    return "อัปเดตบางส่วน {$id}";
});

// DELETE - ลบข้อมูล
Route::delete('/posts/{id}', function ($id) {
    return "ลบบทความ {$id}";
});

// OPTIONS - ตรวจสอบ allowed methods (ส่วนใหญ่ไม่ต้องกำหนดเอง)
Route::options('/posts', function () {
    return response()->json(['methods' => ['GET', 'POST']]);
});

// รับทุก HTTP method
Route::any('/webhook', function () {
    return 'รับ webhook';
});

// รับหลาย HTTP methods
Route::match(['get', 'post'], '/login', function () {
    return 'หน้า Login';
});
```

### Closure vs Controller

```php
// วิธีที่ 1: ใช้ Closure (เหมาะสำหรับ logic เล็กน้อย)
Route::get('/hello', function () {
    return view('hello');
});

// วิธีที่ 2: ใช้ Controller (แนะนำสำหรับ logic ซับซ้อน)
use App\Http\Controllers\PostController;

Route::get('/posts', [PostController::class, 'index']);
Route::get('/posts/{post}', [PostController::class, 'show']);
Route::post('/posts', [PostController::class, 'store']);

// Invokable Controller (Single Action)
use App\Http\Controllers\ShowDashboard;
Route::get('/dashboard', ShowDashboard::class);
```

### Return Types

```php
// Return string
Route::get('/string', function () {
    return 'Hello, World!';
});

// Return array (auto JSON)
Route::get('/json', function () {
    return ['name' => 'John', 'email' => 'john@example.com'];
});

// Return view
Route::get('/home', function () {
    return view('home', ['title' => 'หน้าแรก']);
});

// Return JSON response
Route::get('/api/users', function () {
    return response()->json([
        'data' => ['users'],
        'status' => 'success'
    ]);
});

// Return redirect
Route::get('/old-page', function () {
    return redirect('/new-page');
});

// Return with status code
Route::get('/not-found', function () {
    return response('ไม่พบข้อมูล', 404);
});
```

---

## 2. Route Parameters

### Required Parameters

```php
// Parameter เดี่ยว
Route::get('/posts/{id}', function ($id) {
    return "แสดงบทความ ID: {$id}";
});

// หลาย Parameters
Route::get('/users/{userId}/posts/{postId}', function ($userId, $postId) {
    return "บทความ {$postId} ของ User {$userId}";
});

// ใช้กับ Controller
Route::get('/posts/{id}', [PostController::class, 'show']);
// Controller method:
// public function show($id) { ... }
```

### Optional Parameters

```php
// ใส่ ? หลัง parameter name และ default ใน method
Route::get('/posts/{category?}', function ($category = 'all') {
    return "หมวดหมู่: {$category}";
});

// หลาย optional parameters
Route::get('/posts/{year?}/{month?}', function ($year = null, $month = null) {
    if (!$year) {
        return "บทความทั้งหมด";
    }
    if (!$month) {
        return "บทความปี {$year}";
    }
    return "บทความเดือน {$month}/{$year}";
});
```

### Parameter Constraints (Regex)

```php
// กำหนดว่า parameter ต้องเป็นตัวเลขเท่านั้น
Route::get('/posts/{id}', function ($id) {
    return "Post {$id}";
})->where('id', '[0-9]+');

// ตัวอักษรเท่านั้น
Route::get('/categories/{name}', function ($name) {
    return "Category: {$name}";
})->where('name', '[a-zA-Z]+');

// Slug format
Route::get('/posts/{slug}', function ($slug) {
    return "Post: {$slug}";
})->where('slug', '[a-z0-9\-]+');

// หลาย constraints
Route::get('/users/{id}/posts/{slug}', function ($id, $slug) {
    return "User {$id} Post: {$slug}";
})->where(['id' => '[0-9]+', 'slug' => '[a-z0-9\-]+']);
```

### Global Constraints

```php
// app/Providers/RouteServiceProvider.php
public function boot(): void
{
    // กำหนด global constraint สำหรับ parameter ชื่อ 'id'
    Route::pattern('id', '[0-9]+');
    Route::pattern('slug', '[a-z0-9\-]+');
    
    parent::boot();
}
```

### Shorthand Constraints

```php
// Laravel มี shorthand methods
Route::get('/posts/{id}', function ($id) {
    // ...
})->whereNumber('id'); // เฉพาะตัวเลข

Route::get('/posts/{slug}', function ($slug) {
    // ...
})->whereAlpha('slug'); // เฉพาะตัวอักษร

Route::get('/posts/{slug}', function ($slug) {
    // ...
})->whereAlphaNumeric('slug'); // ตัวอักษรและตัวเลข

Route::get('/posts/{uuid}', function ($uuid) {
    // ...
})->whereUuid('uuid'); // UUID format
```

### Route Model Binding

```php
// แทนที่จะใช้ $id ดึง Post เอง ให้ Laravel ดึงให้อัตโนมัติ
use App\Models\Post;

// Implicit Binding - ใช้ type hint
Route::get('/posts/{post}', function (Post $post) {
    return $post; // Laravel ดึง Post::find($id) ให้อัตโนมัติ
});

// ถ้าไม่เจอ Post จะ return 404 อัตโนมัติ

// กำหนด binding key (แทน id ใช้ slug)
Route::get('/posts/{post:slug}', function (Post $post) {
    return $post; // Laravel ดึง Post::where('slug', $slug)->first()
});

// หรือ กำหนดใน Model
// app/Models/Post.php
public function getRouteKeyName(): string
{
    return 'slug'; // ใช้ slug แทน id สำหรับ route binding
}
```

---

## 3. Named Routes

Named routes ช่วยให้เรา reference routes ด้วยชื่อแทน URL จริง

```php
// กำหนดชื่อ route
Route::get('/posts', [PostController::class, 'index'])->name('posts.index');
Route::get('/posts/create', [PostController::class, 'create'])->name('posts.create');
Route::get('/posts/{post}', [PostController::class, 'show'])->name('posts.show');
Route::post('/posts', [PostController::class, 'store'])->name('posts.store');
Route::get('/posts/{post}/edit', [PostController::class, 'edit'])->name('posts.edit');
Route::put('/posts/{post}', [PostController::class, 'update'])->name('posts.update');
Route::delete('/posts/{post}', [PostController::class, 'destroy'])->name('posts.destroy');
```

### การใช้ Named Routes

```php
// สร้าง URL จาก route name
$url = route('posts.index');
// http://localhost/posts

$url = route('posts.show', ['post' => 1]);
// http://localhost/posts/1

$url = route('posts.show', $post); // ส่ง Model โดยตรง
// http://localhost/posts/1

// Redirect ด้วย route name
return redirect()->route('posts.index');
return redirect()->route('posts.show', $post);

// ใน Blade template
<a href="{{ route('posts.index') }}">ดูบทความ</a>
<a href="{{ route('posts.show', $post) }}">{{ $post->title }}</a>

// ตรวจสอบ current route
if (request()->routeIs('posts.*')) {
    // อยู่ใน posts routes
}

if (request()->routeIs('posts.index')) {
    // อยู่ที่หน้า index
}
```

---

## 4. Route Groups

### Prefix - จัดกลุ่มด้วย URL prefix

```php
// ทุก route ใน group จะมี prefix /admin
Route::prefix('admin')->group(function () {
    Route::get('/dashboard', [AdminController::class, 'dashboard']);
    // URL: /admin/dashboard
    
    Route::get('/users', [AdminController::class, 'users']);
    // URL: /admin/users
    
    Route::get('/settings', [AdminController::class, 'settings']);
    // URL: /admin/settings
});
```

### Namespace - จัดกลุ่ม Controller namespace

```php
// Laravel 10+ ไม่ต้องกำหนด namespace แล้ว แต่สามารถใช้ได้
Route::prefix('admin')
    ->name('admin.')
    ->group(function () {
        Route::get('/dashboard', [AdminDashboardController::class, 'index'])
            ->name('dashboard');
        // Name: admin.dashboard
        // URL: /admin/dashboard
    });
```

### Name Prefix - จัดกลุ่มด้วย route name prefix

```php
Route::name('admin.')->group(function () {
    Route::get('/admin/users', [AdminController::class, 'users'])
        ->name('users');
    // Name: admin.users
    
    Route::get('/admin/posts', [AdminController::class, 'posts'])
        ->name('posts');
    // Name: admin.posts
});
```

### รวมหลาย options

```php
Route::prefix('admin')
    ->name('admin.')
    ->middleware(['auth', 'admin'])
    ->group(function () {
        Route::get('/dashboard', [AdminDashboardController::class, 'index'])
            ->name('dashboard');
        // URL: /admin/dashboard, Name: admin.dashboard, Middleware: auth, admin
        
        Route::resource('users', AdminUserController::class);
        // URLs: /admin/users/*, Names: admin.users.*
        
        Route::resource('posts', AdminPostController::class);
        // URLs: /admin/posts/*, Names: admin.posts.*
    });
```

---

## 5. Route Middleware

Middleware คือ code ที่รันก่อน/หลัง HTTP request

### Built-in Middleware

```php
// auth - ต้อง login
Route::get('/dashboard', [DashboardController::class, 'index'])
    ->middleware('auth');

// guest - ต้องไม่ได้ login
Route::get('/login', [AuthController::class, 'showLogin'])
    ->middleware('guest');

// verified - ต้อง verify email
Route::get('/premium', [PremiumController::class, 'index'])
    ->middleware(['auth', 'verified']);

// throttle - rate limiting
Route::post('/api/submit', [ApiController::class, 'submit'])
    ->middleware('throttle:60,1'); // 60 requests ต่อ 1 นาที
```

### Middleware ใน Groups

```php
Route::middleware(['auth'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
    Route::get('/profile', [ProfileController::class, 'show']);
    Route::put('/profile', [ProfileController::class, 'update']);
});

Route::middleware(['auth', 'admin'])->prefix('admin')->name('admin.')->group(function () {
    Route::resource('users', AdminUserController::class);
    Route::resource('posts', AdminPostController::class);
});
```

### สร้าง Custom Middleware

```bash
php artisan make:middleware CheckAge
```

```php
// app/Http/Middleware/CheckAge.php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class CheckAge
{
    public function handle(Request $request, Closure $next, int $minAge = 18): Response
    {
        $age = $request->input('age') ?? auth()->user()?->age;
        
        if ($age < $minAge) {
            return redirect('/')->with('error', "ต้องมีอายุ {$minAge} ปีขึ้นไป");
        }
        
        return $next($request);
    }
}
```

```php
// bootstrap/app.php (Laravel 11+) หรือ app/Http/Kernel.php (Laravel 10)
// ลงทะเบียน middleware alias
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'age' => \App\Http\Middleware\CheckAge::class,
    ]);
})
```

```php
// ใช้งาน
Route::get('/adult-content', [ContentController::class, 'adult'])
    ->middleware('age:21'); // ต้องอายุ 21+

Route::get('/mature-content', [ContentController::class, 'mature'])
    ->middleware('age'); // default 18
```

---

## 6. API Routes

```php
// routes/api.php - routes นี้จะมี prefix /api โดยอัตโนมัติ
<?php

use App\Http\Controllers\Api\PostController;
use App\Http\Controllers\Api\AuthController;
use Illuminate\Support\Facades\Route;

// Public routes
Route::post('/auth/login', [AuthController::class, 'login']);
Route::post('/auth/register', [AuthController::class, 'register']);

// Protected routes (ต้อง authenticate ด้วย token)
Route::middleware('auth:sanctum')->group(function () {
    Route::post('/auth/logout', [AuthController::class, 'logout']);
    Route::get('/auth/me', [AuthController::class, 'me']);
    
    Route::apiResource('posts', PostController::class);
    // สร้าง routes สำหรับ API (ไม่มี create/edit)
    // GET    /api/posts          -> index
    // POST   /api/posts          -> store
    // GET    /api/posts/{post}   -> show
    // PUT    /api/posts/{post}   -> update
    // DELETE /api/posts/{post}   -> destroy
});

// API Versioning
Route::prefix('v1')->group(function () {
    Route::apiResource('posts', \App\Http\Controllers\Api\V1\PostController::class);
});

Route::prefix('v2')->group(function () {
    Route::apiResource('posts', \App\Http\Controllers\Api\V2\PostController::class);
});
```

### API Route กับ Rate Limiting

```php
// app/Providers/RouteServiceProvider.php (Laravel 10)
// หรือ bootstrap/app.php (Laravel 11)

protected function configureRateLimiting(): void
{
    // Rate limit 60 requests ต่อนาทีต่อ user
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });
    
    // Rate limit พิเศษสำหรับ upload
    RateLimiter::for('uploads', function (Request $request) {
        return [
            Limit::perMinute(10)->by($request->user()->id),
            Limit::perDay(100)->by($request->user()->id),
        ];
    });
}
```

```php
// ใช้งาน rate limit
Route::middleware('throttle:api')->group(function () {
    Route::apiResource('posts', PostController::class);
});

Route::post('/upload', [UploadController::class, 'store'])
    ->middleware('throttle:uploads');
```

---

## 7. Resource Routes

Resource routes สร้าง 7 routes มาตรฐานสำหรับ CRUD ในครั้งเดียว

```php
Route::resource('posts', PostController::class);
```

สร้าง routes ดังนี้:

| Method | URL | Action | Route Name |
|--------|-----|--------|------------|
| GET | /posts | index | posts.index |
| GET | /posts/create | create | posts.create |
| POST | /posts | store | posts.store |
| GET | /posts/{post} | show | posts.show |
| GET | /posts/{post}/edit | edit | posts.edit |
| PUT/PATCH | /posts/{post} | update | posts.update |
| DELETE | /posts/{post} | destroy | posts.destroy |

### จำกัด Routes

```php
// เฉพาะบาง actions
Route::resource('posts', PostController::class)->only([
    'index', 'show'
]);

// ยกเว้นบาง actions
Route::resource('posts', PostController::class)->except([
    'create', 'edit'
]);

// API Resource (ไม่มี create/edit เพราะ API ไม่ต้องการ form)
Route::apiResource('posts', PostController::class);
```

### Nested Resources

```php
// blog.post = post ภายใน blog
Route::resource('blogs.posts', BlogPostController::class);
// URL: /blogs/{blog}/posts
// URL: /blogs/{blog}/posts/{post}
// Name: blogs.posts.index, blogs.posts.show, ...

// Shallow nesting (ลด nesting ลง)
Route::resource('blogs.posts', BlogPostController::class)->shallow();
// GET /blogs/{blog}/posts (index)
// GET /blogs/{blog}/posts/create (create)
// POST /blogs/{blog}/posts (store)
// GET /posts/{post} (show) - ไม่มี blog ใน URL
// GET /posts/{post}/edit (edit)
// PUT /posts/{post} (update)
// DELETE /posts/{post} (destroy)
```

### ตั้งชื่อ Parameter

```php
// แทน {post} ใช้ {article}
Route::resource('posts', PostController::class)->parameters([
    'posts' => 'article'
]);
// URL: /posts/{article}
```

### Multiple Resource Controllers

```php
Route::resources([
    'posts'    => PostController::class,
    'comments' => CommentController::class,
    'tags'     => TagController::class,
]);
```

---

## 8. Route ขั้นสูง

### Fallback Route (404 handler)

```php
// ต้องกำหนดท้ายสุด
Route::fallback(function () {
    return response()->view('errors.404', [], 404);
});
```

### Redirect Routes

```php
// Redirect ถาวร (301)
Route::redirect('/here', '/there');
Route::redirect('/here', '/there', 301);

// Redirect ชั่วคราว (302)
Route::redirect('/here', '/there', 302);

// Permanent redirect
Route::permanentRedirect('/here', '/there');
```

### View Routes

```php
// ถ้า route แค่ return view ไม่ต้องสร้าง Controller
Route::view('/about', 'about');
Route::view('/contact', 'contact', ['phone' => '02-xxx-xxxx']);
```

### Signed Routes (URL ที่มี signature)

```php
// สร้าง signed URL (สำหรับ email verification เป็นต้น)
use Illuminate\Support\Facades\URL;

$url = URL::signedRoute('unsubscribe', ['user' => 1]);
// http://localhost/unsubscribe/1?signature=xxxxx

// Temporary signed URL (หมดอายุ)
$url = URL::temporarySignedRoute(
    'unsubscribe',
    now()->addHours(24),
    ['user' => 1]
);

// Route ที่รับ signed URL
Route::get('/unsubscribe/{user}', function (Request $request) {
    if (!$request->hasValidSignature()) {
        abort(401);
    }
    // ดำเนินการ unsubscribe
})->name('unsubscribe');

// หรือใช้ middleware
Route::get('/unsubscribe/{user}', [EmailController::class, 'unsubscribe'])
    ->name('unsubscribe')
    ->middleware('signed');
```

---

## Workshop: สร้าง URL Structure สำหรับ Blog

### โจทย์

สร้าง routes สำหรับ Blog application ที่มีฟีเจอร์:
1. หน้าแรก
2. จัดการบทความ (CRUD) พร้อม authentication
3. จัดการ comments ภายใต้บทความ
4. Admin panel
5. API สำหรับ mobile app

### Solution

```php
// routes/web.php
<?php

use App\Http\Controllers\HomeController;
use App\Http\Controllers\PostController;
use App\Http\Controllers\CommentController;
use App\Http\Controllers\CategoryController;
use App\Http\Controllers\AuthController;
use App\Http\Controllers\Admin\DashboardController as AdminDashboard;
use App\Http\Controllers\Admin\PostController as AdminPostController;
use App\Http\Controllers\Admin\UserController as AdminUserController;
use Illuminate\Support\Facades\Route;

/*
|--------------------------------------------------------------------------
| Public Routes (ไม่ต้อง Login)
|--------------------------------------------------------------------------
*/

// หน้าแรก
Route::get('/', [HomeController::class, 'index'])->name('home');

// บทความ (อ่านอย่างเดียว)
Route::get('/posts', [PostController::class, 'index'])->name('posts.index');
Route::get('/posts/{post:slug}', [PostController::class, 'show'])->name('posts.show');

// หมวดหมู่
Route::get('/categories', [CategoryController::class, 'index'])->name('categories.index');
Route::get('/categories/{category:slug}', [CategoryController::class, 'show'])->name('categories.show');

// Authentication
Route::middleware('guest')->group(function () {
    Route::view('/login', 'auth.login')->name('login');
    Route::post('/login', [AuthController::class, 'login'])->name('auth.login');
    Route::view('/register', 'auth.register')->name('register');
    Route::post('/register', [AuthController::class, 'register'])->name('auth.register');
});

/*
|--------------------------------------------------------------------------
| Authenticated Routes (ต้อง Login)
|--------------------------------------------------------------------------
*/

Route::middleware('auth')->group(function () {
    // Logout
    Route::post('/logout', [AuthController::class, 'logout'])->name('auth.logout');
    
    // Profile
    Route::get('/profile', [ProfileController::class, 'show'])->name('profile.show');
    Route::put('/profile', [ProfileController::class, 'update'])->name('profile.update');
    
    // จัดการบทความของตัวเอง
    Route::prefix('my')->name('my.')->group(function () {
        Route::resource('posts', PostController::class)->except(['index', 'show']);
        // my.posts.create, my.posts.store, my.posts.edit, my.posts.update, my.posts.destroy
    });
    
    // Comments
    Route::post('/posts/{post}/comments', [CommentController::class, 'store'])
        ->name('comments.store');
    Route::delete('/comments/{comment}', [CommentController::class, 'destroy'])
        ->name('comments.destroy');
});

/*
|--------------------------------------------------------------------------
| Admin Routes
|--------------------------------------------------------------------------
*/

Route::middleware(['auth', 'admin'])
    ->prefix('admin')
    ->name('admin.')
    ->group(function () {
        // Dashboard
        Route::get('/', [AdminDashboard::class, 'index'])->name('dashboard');
        
        // จัดการบทความ
        Route::resource('posts', AdminPostController::class);
        
        // จัดการ users
        Route::resource('users', AdminUserController::class);
        
        // รายงาน
        Route::get('/reports', [AdminDashboard::class, 'reports'])->name('reports');
        Route::get('/reports/export', [AdminDashboard::class, 'export'])->name('reports.export');
    });
```

```php
// routes/api.php
<?php

use App\Http\Controllers\Api\AuthController;
use App\Http\Controllers\Api\PostController;
use App\Http\Controllers\Api\CommentController;
use Illuminate\Support\Facades\Route;

/*
|--------------------------------------------------------------------------
| API v1 Routes
|--------------------------------------------------------------------------
*/

Route::prefix('v1')->name('api.v1.')->group(function () {
    
    // Auth
    Route::post('/auth/login', [AuthController::class, 'login'])->name('auth.login');
    Route::post('/auth/register', [AuthController::class, 'register'])->name('auth.register');
    
    // Public endpoints
    Route::apiResource('posts', PostController::class)->only(['index', 'show']);
    
    // Protected endpoints
    Route::middleware('auth:sanctum')->group(function () {
        Route::post('/auth/logout', [AuthController::class, 'logout'])->name('auth.logout');
        Route::get('/auth/me', [AuthController::class, 'me'])->name('auth.me');
        
        // Posts (เขียน/แก้ไข)
        Route::apiResource('posts', PostController::class)->except(['index', 'show']);
        
        // Comments
        Route::apiResource('posts.comments', CommentController::class)->shallow();
    });
});
```

### ดู Routes ทั้งหมด

```bash
php artisan route:list
php artisan route:list --name=admin
php artisan route:list --path=api/v1
```

---

## Quiz

### คำถาม

**1.** Route ใดที่ใช้รับทุก HTTP method?

a) `Route::all()`
b) `Route::any()`
c) `Route::every()`
d) `Route::match('*')`

**2.** Named route `posts.show` สร้าง URL ด้วยคำสั่งใด?

a) `url('posts.show', $post)`
b) `route('posts.show', $post)`
c) `link('posts.show', $post)`
d) `path('posts.show', $post)`

**3.** `Route::resource('posts', PostController::class)` สร้างกี่ routes?

a) 4
b) 5
c) 7
d) 9

**4.** Route Model Binding ใช้กับ type hint แบบใดในตัวอย่างต่อไปนี้?

```php
Route::get('/posts/{post}', function (Post $post) {
    return $post;
});
```

a) `post` ต้องตรงกับ column ใน database
b) Laravel ดึง `Post::find($id)` ให้อัตโนมัติ
c) `$post` จะเป็น null ถ้าไม่พบ
d) ต้องกำหนด explicit binding เสมอ

**5.** Middleware `throttle:60,1` หมายความว่าอะไร?

a) หมดเวลา 60 วินาที, retry 1 ครั้ง
b) 60 requests ต่อ 1 นาที
c) 1 request ต่อ 60 วินาที
d) 60 MB per 1 request

### เฉลย

| ข้อ | คำตอบ | อธิบาย |
|-----|-------|--------|
| 1 | **b** | `Route::any()` รับทุก HTTP method |
| 2 | **b** | `route('posts.show', $post)` สร้าง URL จาก named route |
| 3 | **c** | Resource routes สร้าง 7 routes: index, create, store, show, edit, update, destroy |
| 4 | **b** | Implicit binding ดึง Model จาก database อัตโนมัติ หาก ไม่พบ return 404 |
| 5 | **b** | `throttle:60,1` = 60 requests ต่อ 1 นาที |

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- HTTP methods ทั้งหมดและการใช้งาน
- Route parameters ทั้ง required, optional และ constraints
- Route Model Binding ที่ดึง Model อัตโนมัติ
- Named routes สำหรับ reference URL ด้วยชื่อ
- Route Groups สำหรับจัดกลุ่ม routes
- Middleware กับ routes ทั้งระดับ individual และ group
- API routes และ rate limiting
- Resource routes สำหรับ CRUD

---

## ลิงก์ไป Part ถัดไป

➡️ **[Part 028: Laravel Controllers](./part-028-laravel-controllers.md)**

ใน Part ถัดไปเราจะเรียนรู้การสร้างและใช้งาน Controllers อย่างละเอียด รวมถึง Resource Controller, Single Action Controller และ Dependency Injection
