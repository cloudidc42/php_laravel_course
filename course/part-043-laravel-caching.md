# Part 043: Laravel Caching — ระบบแคชใน Laravel

**ระดับ: สูง (Advanced)**
**เวลาเรียน: 4-5 ชั่วโมง**

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจและใช้งาน Cache Drivers ต่างๆ (Redis, Memcached, File)
- ใช้งาน Cache Tags เพื่อจัดกลุ่มและลบ Cache
- ตั้งค่า Rate Limiting ด้วย Cache
- ออกแบบ Cache Invalidation strategies ที่เหมาะสม
- สร้าง Caching layer สำหรับ API ที่มีประสิทธิภาพ

---

## 1. ทำไมต้องใช้ Cache?

Cache คือการเก็บข้อมูลที่ใช้บ่อยไว้ในที่เข้าถึงได้เร็วกว่า เพื่อลดเวลาในการประมวลผลและลดภาระของฐานข้อมูล

**ปัญหาที่ Cache แก้ได้:**
- Query ฐานข้อมูลที่ช้าและทำซ้ำบ่อย
- การคำนวณที่ใช้เวลานาน
- การเรียก API ภายนอกที่มี Rate Limit
- การโหลดข้อมูลที่ไม่ค่อยเปลี่ยนแปลง

```php
// ตัวอย่างแบบไม่มี Cache - query ทุก request
public function getPopularProducts()
{
    return Product::where('is_active', true)
        ->withCount('orders')
        ->orderBy('orders_count', 'desc')
        ->take(10)
        ->get();
}

// ตัวอย่างแบบมี Cache - query เฉพาะครั้งแรก
public function getPopularProducts()
{
    return Cache::remember('popular_products', now()->addHours(1), function () {
        return Product::where('is_active', true)
            ->withCount('orders')
            ->orderBy('orders_count', 'desc')
            ->take(10)
            ->get();
    });
}
```

---

## 2. Cache Drivers ใน Laravel

Laravel รองรับ Cache Drivers หลายประเภท แต่ละแบบเหมาะกับการใช้งานที่แตกต่างกัน

### 2.1 File Cache (Default)

เก็บ Cache เป็นไฟล์ในโฟลเดอร์ `storage/framework/cache`

**ข้อดี:** ไม่ต้องติดตั้งซอฟต์แวร์เพิ่มเติม
**ข้อเสีย:** ช้ากว่า In-memory drivers, ไม่เหมาะกับ Multi-server

```bash
# ตั้งค่าใน .env
CACHE_DRIVER=file
```

```php
// config/cache.php
'file' => [
    'driver' => 'file',
    'path' => storage_path('framework/cache/data'),
    'lock_path' => storage_path('framework/cache/data'),
],
```

### 2.2 Redis Cache

Redis เป็น In-memory data structure store ที่ได้รับความนิยมมากที่สุดสำหรับ Cache

**ติดตั้ง:**
```bash
composer require predis/predis
# หรือใช้ phpredis extension
```

**ตั้งค่า .env:**
```bash
CACHE_DRIVER=redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
```

**ตั้งค่า config/database.php:**
```php
'redis' => [
    'client' => env('REDIS_CLIENT', 'phpredis'),
    
    'default' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'username' => env('REDIS_USERNAME'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_DB', '0'),
    ],
    
    'cache' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'username' => env('REDIS_USERNAME'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_CACHE_DB', '1'),
    ],
],
```

**ใช้งาน Redis Cache:**
```php
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Redis;

// เก็บค่าแบบธรรมดา
Cache::put('user:1:profile', $userData, now()->addMinutes(30));

// ดึงค่า
$user = Cache::get('user:1:profile');

// เก็บถาวร (ไม่หมดอายุ)
Cache::forever('settings', $settings);

// ตรวจสอบว่ามีหรือไม่
if (Cache::has('user:1:profile')) {
    // ...
}

// ลบ Cache
Cache::forget('user:1:profile');

// ใช้ Redis โดยตรงสำหรับ operations พิเศษ
Redis::incr('page:views:' . $pageId);
Redis::expire('page:views:' . $pageId, 86400);
$views = Redis::get('page:views:' . $pageId);
```

**Redis Connection Pooling และ Sentinel:**
```php
// config/database.php - Redis Sentinel สำหรับ High Availability
'redis' => [
    'client' => 'predis',
    
    'clusters' => [
        'default' => [
            [
                'host' => env('REDIS_HOST', '127.0.0.1'),
                'password' => env('REDIS_PASSWORD', null),
                'port' => env('REDIS_PORT', 6379),
                'database' => 0,
            ],
        ],
    ],
    
    'options' => [
        'cluster' => env('REDIS_CLUSTER', 'redis'),
        'prefix' => env('REDIS_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_database_'),
    ],
],
```

### 2.3 Memcached Cache

Memcached เป็น In-memory caching system ที่เรียบง่ายและเร็ว

**ติดตั้ง:**
```bash
# Ubuntu/Debian
sudo apt-get install php-memcached

# macOS
brew install libmemcached
pecl install memcached
```

**ตั้งค่า .env:**
```bash
CACHE_DRIVER=memcached
MEMCACHED_HOST=127.0.0.1
MEMCACHED_PORT=11211
```

**config/cache.php:**
```php
'memcached' => [
    'driver' => 'memcached',
    'persistent_id' => env('MEMCACHED_PERSISTENT_ID'),
    'sasl' => [
        env('MEMCACHED_USERNAME'),
        env('MEMCACHED_PASSWORD'),
    ],
    'options' => [
        // Memcached::OPT_CONNECT_TIMEOUT => 2000,
    ],
    'servers' => [
        [
            'host' => env('MEMCACHED_HOST', '127.0.0.1'),
            'port' => env('MEMCACHED_PORT', 11211),
            'weight' => 100,
        ],
    ],
],
```

### 2.4 Array Cache (สำหรับ Testing)

Array Cache เก็บข้อมูลใน PHP array ใช้สำหรับ Testing เท่านั้น

```php
// ตั้งค่าใน phpunit.xml
<php>
    <env name="CACHE_DRIVER" value="array"/>
</php>
```

### 2.5 Database Cache

เก็บ Cache ในฐานข้อมูล เหมาะสำหรับ Application ที่ยังไม่พร้อม Redis

```bash
# สร้าง Cache table
php artisan cache:table
php artisan migrate
```

---

## 3. Cache Operations พื้นฐาน

### 3.1 การเก็บและดึง Cache

```php
use Illuminate\Support\Facades\Cache;

class ProductController extends Controller
{
    // remember() - ดึงหรือสร้าง Cache
    public function index()
    {
        $products = Cache::remember('products.all', now()->addHour(), function () {
            return Product::with(['category', 'images'])
                ->where('is_active', true)
                ->get();
        });
        
        return ProductResource::collection($products);
    }
    
    // rememberForever() - Cache ไม่หมดอายุ
    public function getCategories()
    {
        return Cache::rememberForever('categories.all', function () {
            return Category::orderBy('name')->get();
        });
    }
    
    // put() - เก็บ Cache
    public function store(Request $request)
    {
        $product = Product::create($request->validated());
        
        // อัพเดต Cache
        Cache::put('product.' . $product->id, $product, now()->addDay());
        
        // ลบ Cache ที่เกี่ยวข้อง
        Cache::forget('products.all');
        
        return new ProductResource($product);
    }
    
    // get() with default value
    public function show($id)
    {
        $product = Cache::get('product.' . $id, function () use ($id) {
            return Product::findOrFail($id);
        });
        
        return new ProductResource($product);
    }
    
    // pull() - ดึงและลบ Cache
    public function getOneTimeData($token)
    {
        $data = Cache::pull('one_time:' . $token);
        
        if (!$data) {
            abort(404, 'Token not found or already used');
        }
        
        return response()->json($data);
    }
    
    // add() - เพิ่มเฉพาะกรณีที่ยังไม่มี
    public function incrementViews($productId)
    {
        Cache::add('product.' . $productId . '.views', 0, now()->addDay());
        Cache::increment('product.' . $productId . '.views');
    }
}
```

### 3.2 Atomic Locks

ป้องกัน Race Condition ด้วย Cache Locks:

```php
use Illuminate\Support\Facades\Cache;

class OrderService
{
    public function processOrder(int $orderId): void
    {
        $lock = Cache::lock('order:processing:' . $orderId, 30); // 30 วินาที
        
        if ($lock->get()) {
            try {
                // ประมวลผล Order
                $order = Order::findOrFail($orderId);
                $order->process();
            } finally {
                $lock->release();
            }
        } else {
            throw new \RuntimeException('Order is already being processed');
        }
    }
    
    // block() - รอจนกว่าจะได้ Lock
    public function processWithWait(int $orderId): void
    {
        $lock = Cache::lock('order:processing:' . $orderId, 30);
        
        $lock->block(10, function () use ($orderId) {
            // รอ 10 วินาที แล้วประมวลผล
            $order = Order::findOrFail($orderId);
            $order->process();
        });
    }
}
```

---

## 4. Cache Tags

Cache Tags ช่วยจัดกลุ่ม Cache Keys เพื่อลบหมู่ได้ง่าย (ใช้ได้กับ Redis, Memcached เท่านั้น)

```php
use Illuminate\Support\Facades\Cache;

class ArticleController extends Controller
{
    // เก็บ Cache พร้อม Tags
    public function index()
    {
        return Cache::tags(['articles', 'published'])
            ->remember('articles.published', now()->addHour(), function () {
                return Article::where('status', 'published')
                    ->with(['author', 'tags'])
                    ->latest()
                    ->paginate(15);
            });
    }
    
    public function show($id)
    {
        return Cache::tags(['articles', 'article:' . $id])
            ->remember('article.' . $id, now()->addHours(6), function () use ($id) {
                return Article::with(['author', 'comments', 'tags'])
                    ->findOrFail($id);
            });
    }
    
    // เมื่ออัพเดต Article - ลบ Cache ที่เกี่ยวข้องทั้งหมด
    public function update(Request $request, Article $article)
    {
        $article->update($request->validated());
        
        // ลบ Cache ของ Article นี้ทั้งหมด
        Cache::tags(['article:' . $article->id])->flush();
        
        // ลบ Cache List ของ Articles
        Cache::tags(['articles'])->flush();
        
        return new ArticleResource($article);
    }
    
    // เมื่อลบ Article
    public function destroy(Article $article)
    {
        // ลบ Cache ก่อน
        Cache::tags(['article:' . $article->id, 'articles'])->flush();
        
        $article->delete();
        
        return response()->noContent();
    }
}
```

**ตัวอย่างการใช้ Tags ขั้นสูง:**
```php
class CacheService
{
    // Cache ข้อมูล User พร้อม Tags หลายระดับ
    public function getUserData(int $userId): array
    {
        return Cache::tags([
            'users',
            'user:' . $userId,
            'user:' . $userId . ':profile'
        ])->remember('user.' . $userId . '.full', now()->addMinutes(30), function () use ($userId) {
            $user = User::with([
                'profile',
                'roles',
                'permissions',
                'preferences'
            ])->findOrFail($userId);
            
            return [
                'user' => $user->toArray(),
                'stats' => $this->getUserStats($userId),
                'recent_activity' => $this->getRecentActivity($userId),
            ];
        });
    }
    
    // ลบ Cache เมื่อ User อัพเดตข้อมูล
    public function invalidateUserCache(int $userId, string $type = 'all'): void
    {
        match($type) {
            'profile' => Cache::tags(['user:' . $userId . ':profile'])->flush(),
            'all' => Cache::tags(['user:' . $userId])->flush(),
            'users_list' => Cache::tags(['users'])->flush(),
            default => Cache::tags(['user:' . $userId])->flush(),
        };
    }
}
```

---

## 5. Rate Limiting

Laravel ใช้ Cache สำหรับ Rate Limiting เพื่อจำกัดจำนวน Request

### 5.1 ใช้ RateLimiter Facade

```php
use Illuminate\Support\Facades\RateLimiter;

class LoginController extends Controller
{
    public function login(Request $request)
    {
        $key = 'login:' . $request->ip();
        
        if (RateLimiter::tooManyAttempts($key, 5)) {
            $seconds = RateLimiter::availableIn($key);
            
            return response()->json([
                'message' => 'Too many login attempts.',
                'retry_after' => $seconds,
            ], 429);
        }
        
        // ตรวจสอบ credentials
        if (!Auth::attempt($request->only('email', 'password'))) {
            RateLimiter::hit($key, 60); // เพิ่มจำนวนครั้ง, หมดอายุใน 60 วินาที
            
            return response()->json([
                'message' => 'Invalid credentials.',
                'attempts_remaining' => RateLimiter::remaining($key, 5),
            ], 401);
        }
        
        RateLimiter::clear($key); // ล้างเมื่อ Login สำเร็จ
        
        return response()->json([
            'token' => $request->user()->createToken('auth')->plainTextToken,
        ]);
    }
}
```

### 5.2 ตั้งค่า Rate Limiter ใน RouteServiceProvider

```php
// app/Providers/RouteServiceProvider.php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

protected function configureRateLimiting(): void
{
    // API Rate Limit ทั่วไป
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });
    
    // Rate Limit สำหรับ Authenticated Users
    RateLimiter::for('api-auth', function (Request $request) {
        return $request->user()
            ? Limit::perMinute(120)->by($request->user()->id)
            : Limit::perMinute(30)->by($request->ip());
    });
    
    // Rate Limit สำหรับ Upload Files
    RateLimiter::for('uploads', function (Request $request) {
        return Limit::perMinute(10)->by($request->user()?->id ?: $request->ip())
            ->response(function () {
                return response()->json([
                    'error' => 'Upload limit exceeded. Please wait before uploading again.',
                ], 429);
            });
    });
    
    // Rate Limit แบบหลายระดับ (Tiered)
    RateLimiter::for('tier', function (Request $request) {
        return [
            Limit::perMinute(100)->by($request->user()?->id ?: $request->ip()),
            Limit::perDay(1000)->by($request->user()?->id ?: $request->ip()),
        ];
    });
}
```

### 5.3 ใช้ Rate Limiting ใน Routes

```php
// routes/api.php
Route::middleware(['auth:sanctum', 'throttle:api-auth'])->group(function () {
    Route::get('/user', [UserController::class, 'profile']);
    Route::put('/user', [UserController::class, 'update']);
});

Route::middleware(['throttle:uploads'])->group(function () {
    Route::post('/upload', [UploadController::class, 'store']);
});

// Custom Rate Limiter
Route::middleware(['throttle:tier'])->group(function () {
    Route::get('/reports', [ReportController::class, 'index']);
});
```

### 5.4 สร้าง Custom Rate Limiter Middleware

```php
// app/Http/Middleware/ApiRateLimit.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

class ApiRateLimit
{
    public function handle(Request $request, Closure $next, string $type = 'default')
    {
        $limits = $this->getLimits($request, $type);
        
        foreach ($limits as $limit) {
            $key = sha1($limit->key);
            
            if (RateLimiter::tooManyAttempts($key, $limit->maxAttempts)) {
                return $this->buildRateLimitResponse($key, $limit);
            }
            
            RateLimiter::hit($key, $limit->decaySeconds);
        }
        
        $response = $next($request);
        
        return $this->addRateLimitHeaders($response, $key, $limits[0]);
    }
    
    private function getLimits(Request $request, string $type): array
    {
        $userId = $request->user()?->id ?? $request->ip();
        
        return match($type) {
            'premium' => [
                Limit::perMinute(300)->by('premium:' . $userId),
                Limit::perDay(10000)->by('premium:daily:' . $userId),
            ],
            'free' => [
                Limit::perMinute(30)->by('free:' . $userId),
                Limit::perDay(500)->by('free:daily:' . $userId),
            ],
            default => [
                Limit::perMinute(60)->by('default:' . $userId),
            ],
        };
    }
    
    private function buildRateLimitResponse(string $key, Limit $limit): \Illuminate\Http\JsonResponse
    {
        $retryAfter = RateLimiter::availableIn($key);
        
        return response()->json([
            'error' => 'Too Many Requests',
            'message' => 'Rate limit exceeded. Please try again later.',
            'retry_after' => $retryAfter,
        ], 429)->withHeaders([
            'Retry-After' => $retryAfter,
            'X-RateLimit-Limit' => $limit->maxAttempts,
            'X-RateLimit-Remaining' => 0,
        ]);
    }
    
    private function addRateLimitHeaders($response, string $key, Limit $limit)
    {
        $remaining = RateLimiter::remaining($key, $limit->maxAttempts);
        
        return $response->withHeaders([
            'X-RateLimit-Limit' => $limit->maxAttempts,
            'X-RateLimit-Remaining' => max(0, $remaining),
        ]);
    }
}
```

---

## 6. Cache Invalidation Strategies

Cache Invalidation เป็นหนึ่งในปัญหาที่ยากที่สุดใน Computer Science มีกลยุทธ์หลายแบบ

### 6.1 Time-Based Invalidation (TTL)

วิธีง่ายที่สุด - กำหนดเวลาหมดอายุ

```php
class ProductService
{
    // Short TTL สำหรับข้อมูลที่เปลี่ยนบ่อย
    public function getProductPrice(int $productId): float
    {
        return Cache::remember(
            'product:' . $productId . ':price',
            now()->addMinutes(5), // 5 นาที
            fn() => Product::find($productId)->current_price
        );
    }
    
    // Long TTL สำหรับข้อมูลที่เปลี่ยนน้อย
    public function getProductDetails(int $productId): array
    {
        return Cache::remember(
            'product:' . $productId . ':details',
            now()->addDays(7), // 7 วัน
            fn() => Product::with(['images', 'specs'])->find($productId)->toArray()
        );
    }
}
```

### 6.2 Event-Driven Invalidation

ลบ Cache เมื่อมีการเปลี่ยนแปลงข้อมูล

```php
// app/Models/Product.php
class Product extends Model
{
    protected static function booted(): void
    {
        static::updated(function (Product $product) {
            // ลบ Cache เมื่ออัพเดต
            Cache::forget('product:' . $product->id . ':details');
            Cache::forget('product:' . $product->id . ':price');
            
            // ลบ Cache ที่เกี่ยวข้อง
            if ($product->isDirty('category_id')) {
                Cache::tags(['category:' . $product->category_id])->flush();
                Cache::tags(['category:' . $product->getOriginal('category_id')])->flush();
            }
        });
        
        static::deleted(function (Product $product) {
            Cache::tags(['product:' . $product->id])->flush();
            Cache::tags(['products:list'])->flush();
        });
    }
}
```

**ใช้ Observer Pattern:**
```php
// app/Observers/ProductObserver.php
namespace App\Observers;

use App\Models\Product;
use App\Services\CacheService;
use Illuminate\Support\Facades\Cache;

class ProductObserver
{
    public function __construct(
        private readonly CacheService $cacheService
    ) {}
    
    public function created(Product $product): void
    {
        $this->cacheService->invalidateProductList();
        $this->cacheService->invalidateCategoryCache($product->category_id);
    }
    
    public function updated(Product $product): void
    {
        Cache::tags(['product:' . $product->id])->flush();
        $this->cacheService->invalidateProductList();
        
        if ($product->wasChanged('category_id')) {
            $this->cacheService->invalidateCategoryCache($product->category_id);
            $this->cacheService->invalidateCategoryCache($product->getOriginal('category_id'));
        }
    }
    
    public function deleted(Product $product): void
    {
        Cache::tags([
            'product:' . $product->id,
            'products:list',
            'category:' . $product->category_id,
        ])->flush();
    }
    
    public function restored(Product $product): void
    {
        $this->cacheService->invalidateProductList();
    }
}

// Register Observer ใน AppServiceProvider
// app/Providers/AppServiceProvider.php
public function boot(): void
{
    Product::observe(ProductObserver::class);
}
```

### 6.3 Write-Through Cache

อัพเดต Cache พร้อมกับ Database เสมอ

```php
class ProductRepository
{
    public function update(int $id, array $data): Product
    {
        $product = Product::findOrFail($id);
        $product->update($data);
        
        // Write-through: อัพเดต Cache ทันที
        $cacheKey = 'product:' . $id;
        Cache::put($cacheKey, $product->fresh()->load(['category', 'images']), now()->addDay());
        
        return $product;
    }
    
    public function find(int $id): Product
    {
        $cacheKey = 'product:' . $id;
        
        // Read-through: ดึงจาก Cache ก่อน
        return Cache::remember($cacheKey, now()->addDay(), function () use ($id) {
            return Product::with(['category', 'images'])->findOrFail($id);
        });
    }
}
```

### 6.4 Cache Aside Pattern

```php
class UserProfileService
{
    public function getProfile(int $userId): UserProfile
    {
        $cacheKey = 'user:' . $userId . ':profile';
        
        // 1. ตรวจสอบ Cache ก่อน
        $profile = Cache::get($cacheKey);
        
        if ($profile !== null) {
            return $profile; // Cache Hit
        }
        
        // 2. Cache Miss - ดึงจาก Database
        $profile = UserProfile::where('user_id', $userId)
            ->with(['settings', 'social_links'])
            ->firstOrFail();
        
        // 3. เก็บใน Cache
        Cache::put($cacheKey, $profile, now()->addHours(2));
        
        return $profile;
    }
    
    public function updateProfile(int $userId, array $data): UserProfile
    {
        $profile = UserProfile::where('user_id', $userId)->firstOrFail();
        $profile->update($data);
        
        // Invalidate Cache
        Cache::forget('user:' . $userId . ':profile');
        
        return $profile->fresh();
    }
}
```

### 6.5 Stampede Protection (Cache Stampede)

ป้องกันปัญหา "Thundering Herd" เมื่อ Cache หมดอายุพร้อมกัน

```php
class CacheStampedeProtection
{
    /**
     * Probabilistic early expiration
     * ต่ออายุ Cache ก่อนหมดเวลาด้วย Probability
     */
    public function remember(string $key, int $ttl, callable $callback, float $beta = 1.0): mixed
    {
        $cached = Cache::get($key);
        
        if ($cached !== null) {
            // ตรวจสอบว่าควร early expiration หรือไม่
            $expiry = Cache::get($key . ':expiry', 0);
            $elapsed = now()->timestamp - $expiry + $ttl;
            
            if ($expiry === 0 || (-$beta * log(random_int(1, 100) / 100)) < ($elapsed - $ttl)) {
                return $cached; // ยังคืนค่า Cache เก่า
            }
        }
        
        // คำนวณค่าใหม่
        $value = $callback();
        
        // เพิ่ม jitter เพื่อกระจาย expiration time
        $jitter = random_int(-30, 30);
        Cache::put($key, $value, now()->addSeconds($ttl + $jitter));
        Cache::put($key . ':expiry', now()->timestamp + $ttl, now()->addSeconds($ttl + $jitter + 60));
        
        return $value;
    }
}
```

---

## 7. Advanced Cache Patterns

### 7.1 Versioned Cache Keys

```php
class VersionedCache
{
    public function remember(string $key, $ttl, callable $callback): mixed
    {
        $version = Cache::get('version:' . $key, 1);
        $versionedKey = $key . ':v' . $version;
        
        return Cache::remember($versionedKey, $ttl, $callback);
    }
    
    public function invalidate(string $key): void
    {
        // เพิ่ม version แทนการลบ - เร็วกว่า
        Cache::increment('version:' . $key);
    }
}
```

### 7.2 Hierarchical Cache

```php
class HierarchicalCache
{
    private const LEVELS = [
        'L1' => ['driver' => 'array', 'ttl' => 60],        // In-process, 1 นาที
        'L2' => ['driver' => 'redis', 'ttl' => 3600],       // Redis, 1 ชั่วโมง
        'L3' => ['driver' => 'database', 'ttl' => 86400],   // Database, 1 วัน
    ];
    
    public function get(string $key): mixed
    {
        foreach (self::LEVELS as $level => $config) {
            $store = Cache::store($config['driver']);
            $value = $store->get($key);
            
            if ($value !== null) {
                // Populate higher levels
                $this->populateLevels($key, $value, $level);
                return $value;
            }
        }
        
        return null;
    }
    
    private function populateLevels(string $key, mixed $value, string $hitLevel): void
    {
        $found = false;
        
        foreach (self::LEVELS as $level => $config) {
            if ($level === $hitLevel) {
                $found = true;
                continue;
            }
            
            if (!$found) {
                Cache::store($config['driver'])->put($key, $value, $config['ttl']);
            }
        }
    }
}
```

---

## 8. Workshop: สร้าง Caching Layer สำหรับ API

### Workshop Overview

สร้าง E-commerce API ที่มี Caching Layer ครบถ้วน

### Step 1: สร้าง CacheService

```php
// app/Services/CacheService.php
namespace App\Services;

use Illuminate\Support\Facades\Cache;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Collection;

class CacheService
{
    public function __construct(
        private readonly int $defaultTtl = 3600,
        private readonly string $prefix = 'app'
    ) {}
    
    public function key(string ...$parts): string
    {
        return $this->prefix . ':' . implode(':', $parts);
    }
    
    public function remember(string $key, callable $callback, ?int $ttl = null): mixed
    {
        return Cache::tags($this->extractTags($key))
            ->remember($key, $ttl ?? $this->defaultTtl, $callback);
    }
    
    public function invalidate(string ...$tags): void
    {
        Cache::tags($tags)->flush();
    }
    
    private function extractTags(string $key): array
    {
        $parts = explode(':', $key);
        $tags = [];
        
        for ($i = 0; $i < count($parts) - 1; $i++) {
            $tags[] = implode(':', array_slice($parts, 0, $i + 1));
        }
        
        return array_unique($tags) ?: [$key];
    }
}
```

### Step 2: สร้าง API Controllers พร้อม Cache

```php
// app/Http/Controllers/Api/ProductController.php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Resources\ProductResource;
use App\Models\Product;
use App\Services\CacheService;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Cache;

class ProductController extends Controller
{
    public function __construct(
        private readonly CacheService $cache
    ) {}
    
    public function index(Request $request)
    {
        $filters = $request->only(['category', 'min_price', 'max_price', 'sort', 'page']);
        $cacheKey = 'api:products:list:' . md5(serialize($filters));
        
        $products = $this->cache->remember($cacheKey, function () use ($request) {
            $query = Product::query()
                ->with(['category', 'primaryImage'])
                ->where('is_active', true);
            
            if ($request->has('category')) {
                $query->whereHas('category', fn($q) => $q->where('slug', $request->category));
            }
            
            if ($request->has('min_price')) {
                $query->where('price', '>=', $request->min_price);
            }
            
            if ($request->has('max_price')) {
                $query->where('price', '<=', $request->max_price);
            }
            
            $sortField = $request->get('sort', 'created_at');
            $query->orderBy($sortField, 'desc');
            
            return $query->paginate(20);
        }, 300); // 5 นาที
        
        return ProductResource::collection($products);
    }
    
    public function show(int $id)
    {
        $cacheKey = 'api:products:' . $id;
        
        $product = $this->cache->remember($cacheKey, function () use ($id) {
            return Product::with([
                'category',
                'images',
                'specifications',
                'reviews' => fn($q) => $q->approved()->latest()->take(5),
            ])->findOrFail($id);
        }, 1800); // 30 นาที
        
        return new ProductResource($product);
    }
    
    public function store(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'description' => 'required|string',
            'price' => 'required|numeric|min:0',
            'category_id' => 'required|exists:categories,id',
            'stock' => 'required|integer|min:0',
        ]);
        
        $product = Product::create($validated);
        
        // Invalidate list caches
        $this->cache->invalidate('api:products:list', 'api:categories');
        
        return new ProductResource($product);
    }
    
    public function update(Request $request, Product $product)
    {
        $validated = $request->validate([
            'name' => 'sometimes|string|max:255',
            'description' => 'sometimes|string',
            'price' => 'sometimes|numeric|min:0',
            'category_id' => 'sometimes|exists:categories,id',
        ]);
        
        $product->update($validated);
        
        // Invalidate specific product and lists
        Cache::forget('api:products:' . $product->id);
        $this->cache->invalidate('api:products:list');
        
        return new ProductResource($product->fresh());
    }
    
    public function destroy(Product $product)
    {
        // Invalidate before delete
        Cache::forget('api:products:' . $product->id);
        $this->cache->invalidate('api:products:list');
        
        $product->delete();
        
        return response()->noContent();
    }
}
```

### Step 3: สร้าง Cache Middleware

```php
// app/Http/Middleware/CacheResponse.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Illuminate\Support\Facades\Cache;

class CacheResponse
{
    public function handle(Request $request, Closure $next, int $ttl = 300)
    {
        // ไม่ Cache ถ้าไม่ใช่ GET หรือมี auth token ส่วนตัว
        if (!$request->isMethod('GET') || $request->has('no-cache')) {
            return $next($request);
        }
        
        $cacheKey = $this->getCacheKey($request);
        
        // ตรวจสอบ Cache
        if (Cache::has($cacheKey)) {
            $cached = Cache::get($cacheKey);
            
            return response($cached['content'], $cached['status'])
                ->withHeaders($cached['headers'])
                ->header('X-Cache', 'HIT')
                ->header('X-Cache-TTL', Cache::getTimeToLive($cacheKey));
        }
        
        $response = $next($request);
        
        // Cache เฉพาะ 200 responses
        if ($response->getStatusCode() === 200) {
            Cache::put($cacheKey, [
                'content' => $response->getContent(),
                'status' => $response->getStatusCode(),
                'headers' => $response->headers->all(),
            ], now()->addSeconds($ttl));
        }
        
        return $response->header('X-Cache', 'MISS');
    }
    
    private function getCacheKey(Request $request): string
    {
        $url = $request->fullUrl();
        $userId = $request->user()?->id ?? 'guest';
        
        return 'response:' . sha1($url . ':' . $userId);
    }
}
```

### Step 4: ทดสอบ Caching

```php
// tests/Feature/ProductCacheTest.php
namespace Tests\Feature;

use App\Models\Product;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Cache;
use Tests\TestCase;

class ProductCacheTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_products_are_cached_on_first_request(): void
    {
        Product::factory()->count(5)->create(['is_active' => true]);
        
        // ครั้งแรก - ควร MISS
        $response1 = $this->getJson('/api/products');
        $response1->assertHeader('X-Cache', 'MISS');
        
        // ครั้งที่สอง - ควร HIT
        $response2 = $this->getJson('/api/products');
        $response2->assertHeader('X-Cache', 'HIT');
        
        $this->assertEquals($response1->json(), $response2->json());
    }
    
    public function test_cache_is_invalidated_on_product_update(): void
    {
        $product = Product::factory()->create(['is_active' => true]);
        
        // Cache ผลลัพธ์แรก
        $this->getJson('/api/products/' . $product->id);
        
        // อัพเดต Product
        $this->putJson('/api/products/' . $product->id, [
            'name' => 'Updated Product Name',
        ]);
        
        // Cache ควรถูก Invalidate
        $response = $this->getJson('/api/products/' . $product->id);
        $response->assertHeader('X-Cache', 'MISS');
        $response->assertJsonPath('data.name', 'Updated Product Name');
    }
    
    public function test_rate_limiting_works(): void
    {
        $this->artisan('cache:clear');
        
        for ($i = 0; $i < 60; $i++) {
            $this->getJson('/api/products');
        }
        
        // Request ที่ 61 ควรถูก Rate Limit
        $response = $this->getJson('/api/products');
        $response->assertStatus(429);
        $response->assertJsonStructure(['message', 'retry_after']);
    }
}
```

### Step 5: เพิ่ม Cache Warming Command

```php
// app/Console/Commands/WarmCache.php
namespace App\Console\Commands;

use App\Models\Product;
use App\Services\CacheService;
use Illuminate\Console\Command;

class WarmCache extends Command
{
    protected $signature = 'cache:warm {--type=all : Type of cache to warm}';
    protected $description = 'Warm up application caches';
    
    public function __construct(
        private readonly CacheService $cacheService
    ) {
        parent::__construct();
    }
    
    public function handle(): int
    {
        $type = $this->option('type');
        
        $this->info('Starting cache warming...');
        
        match($type) {
            'products' => $this->warmProductsCache(),
            'categories' => $this->warmCategoriesCache(),
            'all' => $this->warmAllCaches(),
            default => $this->error('Unknown cache type: ' . $type),
        };
        
        $this->info('Cache warming complete!');
        
        return 0;
    }
    
    private function warmProductsCache(): void
    {
        $this->info('Warming products cache...');
        
        $bar = $this->output->createProgressBar(Product::count());
        
        Product::with(['category', 'images'])->chunk(100, function ($products) use ($bar) {
            foreach ($products as $product) {
                $this->cacheService->remember(
                    'api:products:' . $product->id,
                    fn() => $product,
                    1800
                );
                $bar->advance();
            }
        });
        
        $bar->finish();
        $this->newLine();
    }
    
    private function warmCategoriesCache(): void
    {
        $this->info('Warming categories cache...');
        // Implementation...
    }
    
    private function warmAllCaches(): void
    {
        $this->warmProductsCache();
        $this->warmCategoriesCache();
    }
}
```

---

## 9. การ Monitor Cache

```php
// ติดตาม Cache Hit Rate
class CacheMonitor
{
    public function recordHit(string $key): void
    {
        Redis::incr('cache:stats:hits');
        Redis::incr('cache:stats:hit:' . $this->getCategory($key));
    }
    
    public function recordMiss(string $key): void
    {
        Redis::incr('cache:stats:misses');
        Redis::incr('cache:stats:miss:' . $this->getCategory($key));
    }
    
    public function getStats(): array
    {
        $hits = (int) Redis::get('cache:stats:hits');
        $misses = (int) Redis::get('cache:stats:misses');
        $total = $hits + $misses;
        
        return [
            'hits' => $hits,
            'misses' => $misses,
            'hit_rate' => $total > 0 ? round($hits / $total * 100, 2) : 0,
            'total_requests' => $total,
        ];
    }
    
    private function getCategory(string $key): string
    {
        return explode(':', $key)[0] ?? 'unknown';
    }
}
```

---

## Quiz

**ข้อ 1:** Cache Driver ใดที่รองรับ Cache Tags?

a) File  
b) Database  
c) Redis  
d) Array

**เฉลย:** c) Redis (และ Memcached) รองรับ Cache Tags เท่านั้น เพราะต้องการความสามารถในการ scan keys

---

**ข้อ 2:** Cache Stampede คืออะไร?

a) การที่ Cache หมดอายุทีละ key  
b) การที่ Cache หลาย key หมดอายุพร้อมกัน ทำให้มี requests ไปยัง database พร้อมกันจำนวนมาก  
c) การที่ Cache เต็มและต้องลบข้อมูลเก่า  
d) การที่ Redis Server ล่ม

**เฉลย:** b) Cache Stampede หรือ Thundering Herd เกิดเมื่อ Cache หมดอายุพร้อมกัน ทำให้ requests จำนวนมากไปยัง database พร้อมกัน

---

**ข้อ 3:** ควรใช้ Cache Invalidation Strategy แบบไหนสำหรับข้อมูลที่เปลี่ยนบ่อย?

a) Write-Through Cache  
b) Time-Based Invalidation เวลาสั้น  
c) Event-Driven Invalidation  
d) ทั้ง a และ c

**เฉลย:** d) ควรใช้ทั้ง Event-Driven Invalidation (ลบ Cache ทันทีเมื่อข้อมูลเปลี่ยน) และ Write-Through Cache (อัพเดต Cache พร้อมกับ Database)

---

**ข้อ 4:** `Cache::remember()` vs `Cache::rememberForever()` ต่างกันอย่างไร?

a) `remember()` มีเวลาหมดอายุ, `rememberForever()` ไม่มี  
b) `remember()` คืนค่าเสมอ, `rememberForever()` อาจคืน null  
c) `remember()` ใช้ Redis, `rememberForever()` ใช้ File  
d) ไม่มีความแตกต่าง

**เฉลย:** a) `Cache::remember()` ต้องการ TTL (เวลาหมดอายุ) ส่วน `Cache::rememberForever()` เก็บ Cache ไว้จนกว่าจะลบด้วยตนเอง

---

**ข้อ 5:** Atomic Lock ใน Laravel ใช้แก้ปัญหาอะไร?

a) ป้องกัน Cache ล้น  
b) ป้องกัน Race Condition ในการประมวลผลพร้อมกัน  
c) เพิ่มความเร็วในการเขียน Cache  
d) ทำให้ Cache ปลอดภัยมากขึ้น

**เฉลย:** b) Atomic Lock ป้องกัน Race Condition โดยทำให้มีเพียง process เดียวที่สามารถทำงานกับ resource นั้นได้ในแต่ละครั้ง

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- **Cache Drivers:** File, Redis, Memcached แต่ละอันมีจุดเด่นต่างกัน
- **Cache Tags:** จัดกลุ่ม Cache เพื่อลบหมู่ได้ง่าย
- **Rate Limiting:** จำกัดจำนวน Request ด้วย Cache
- **Cache Invalidation:** กลยุทธ์การลบ Cache แบบต่างๆ
- **Workshop:** สร้าง Caching Layer ครบถ้วนสำหรับ API

---

## ไปต่อ

➡️ [Part 044: Laravel Broadcasting — Real-time Communication](./part-044-laravel-broadcasting.md)
