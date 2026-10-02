# Part 91: Performance Optimization ใน PHP

## บทนำ

Performance Optimization คือกระบวนการทำให้ Application ทำงานเร็วขึ้นและใช้ Resources น้อยลง ต้องเริ่มจากการ Measure ก่อนเสมอ เพราะ "Premature Optimization is the root of all evil"

**Workflow**:
1. Measure (วัด)
2. Identify Bottleneck
3. Fix
4. Measure Again

---

## OPcache Configuration

OPcache เก็บ Compiled PHP Scripts ใน Memory เพื่อไม่ต้อง Compile ซ้ำทุก Request

```ini
; php.ini - OPcache Configuration
[opcache]
opcache.enable=1
opcache.enable_cli=0
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=20000
opcache.revalidate_freq=0          ; 0 = never revalidate in production
opcache.validate_timestamps=0      ; 0 = no timestamp check in production
opcache.save_comments=1
opcache.fast_shutdown=1
opcache.jit=tracing
opcache.jit_buffer_size=100M
```

```php
<?php

// ตรวจสอบ OPcache Status
function getOpcacheStatus(): array
{
    if (!extension_loaded('Zend OPcache')) {
        return ['enabled' => false];
    }
    
    $status = opcache_get_status(false);
    $config = opcache_get_configuration();
    
    return [
        'enabled' => $status['opcache_enabled'],
        'hit_rate' => round($status['opcache_statistics']['opcache_hit_rate'], 2),
        'memory_usage' => [
            'used' => round($status['memory_usage']['used_memory'] / 1024 / 1024, 2) . ' MB',
            'free' => round($status['memory_usage']['free_memory'] / 1024 / 1024, 2) . ' MB',
            'wasted' => round($status['memory_usage']['wasted_memory'] / 1024 / 1024, 2) . ' MB',
        ],
        'cached_scripts' => $status['opcache_statistics']['num_cached_scripts'],
        'max_cached_keys' => $config['directives']['opcache.max_accelerated_files'],
    ];
}

// Preloading (PHP 7.4+)
// preload.php - โหลด Classes ที่ใช้บ่อยตั้งแต่ตอน Start
opcache_compile_file(__DIR__ . '/vendor/autoload.php');

// รายชื่อ Files ที่ต้อง Preload
$files = [
    'app/Models/User.php',
    'app/Models/Product.php',
    'app/Models/Order.php',
    'app/Services/PaymentService.php',
    // ... critical files
];

foreach ($files as $file) {
    opcache_compile_file(base_path($file));
}
```

---

## Redis Caching

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Cache;

class ProductCacheService
{
    private const PRODUCT_TTL = 3600; // 1 hour
    private const PRODUCT_LIST_TTL = 300; // 5 minutes
    
    public function getProduct(int $id): ?array
    {
        return Cache::remember(
            key: "product:{$id}",
            ttl: self::PRODUCT_TTL,
            callback: function () use ($id) {
                return Product::with(['variants', 'images', 'category'])
                    ->find($id)?->toArray();
            }
        );
    }
    
    public function getProductList(array $filters = [], int $page = 1): array
    {
        $cacheKey = 'products:list:' . md5(serialize($filters) . $page);
        
        return Cache::remember(
            key: $cacheKey,
            ttl: self::PRODUCT_LIST_TTL,
            callback: function () use ($filters, $page) {
                return Product::filter($filters)
                    ->with(['mainImage'])
                    ->paginate(20, page: $page)
                    ->toArray();
            }
        );
    }
    
    public function invalidateProduct(int $id): void
    {
        Cache::forget("product:{$id}");
        // Also invalidate list caches (แบบ Simple)
        Cache::tags(['products'])->flush();
    }
    
    public function invalidateAll(): void
    {
        Cache::tags(['products'])->flush();
    }
}

// Cache Tags (Redis only)
class TaggedCacheService
{
    public function getUser(int $id): ?array
    {
        return Cache::tags(['users', "user:{$id}"])
            ->remember("user:data:{$id}", 3600, function () use ($id) {
                return User::with(['profile', 'roles'])->find($id)?->toArray();
            });
    }
    
    public function getUserOrders(int $userId): array
    {
        return Cache::tags(['orders', "user:{$userId}:orders"])
            ->remember("user:{$userId}:orders", 600, function () use ($userId) {
                return Order::where('user_id', $userId)
                    ->with(['items'])
                    ->latest()
                    ->limit(10)
                    ->get()
                    ->toArray();
            });
    }
    
    public function invalidateUser(int $userId): void
    {
        // Invalidate everything related to this user
        Cache::tags(["user:{$userId}"])->flush();
        Cache::tags(["user:{$userId}:orders"])->flush();
    }
}

// Redis Advanced Patterns
class RedisCacheAdvanced
{
    public function __construct(private \Redis $redis) {}
    
    // Counter with Expiry
    public function incrementPageView(string $page): int
    {
        $key = "pageviews:{$page}:" . date('Y-m-d');
        $count = $this->redis->incr($key);
        $this->redis->expire($key, 86400 * 7); // 7 days
        return $count;
    }
    
    // Rate Limiting with Sliding Window
    public function isRateLimited(string $key, int $limit, int $window): bool
    {
        $now = microtime(true);
        $windowStart = $now - $window;
        
        $this->redis->zRemRangeByScore($key, '-inf', $windowStart);
        $count = $this->redis->zCard($key);
        
        if ($count < $limit) {
            $this->redis->zAdd($key, $now, $now);
            $this->redis->expire($key, $window);
            return false;
        }
        
        return true;
    }
    
    // Distributed Lock
    public function acquireLock(string $resource, int $ttl = 30): ?string
    {
        $token = bin2hex(random_bytes(16));
        
        $acquired = $this->redis->set(
            "lock:{$resource}",
            $token,
            ['NX', 'EX' => $ttl]
        );
        
        return $acquired ? $token : null;
    }
    
    public function releaseLock(string $resource, string $token): bool
    {
        $script = "
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
        ";
        
        return (bool)$this->redis->eval($script, ["lock:{$resource}", $token], 1);
    }
    
    // Cache Aside Pattern
    public function getOrSet(string $key, callable $callback, int $ttl = 3600): mixed
    {
        $cached = $this->redis->get($key);
        
        if ($cached !== false) {
            return unserialize($cached);
        }
        
        $value = $callback();
        $this->redis->setex($key, $ttl, serialize($value));
        
        return $value;
    }
}
```

---

## Database Query Optimization

```php
<?php

// ❌ N+1 Problem
$users = User::all(); // 1 query
foreach ($users as $user) {
    echo $user->profile->bio; // N queries!
}

// ✅ Eager Loading
$users = User::with(['profile', 'roles', 'orders' => function ($q) {
    $q->where('status', 'completed')->latest()->limit(5);
}])->get();

// ✅ Lazy Eager Loading (เมื่อรู้ว่าจะใช้ในภายหลัง)
$users = User::all();
$users->load(['profile', 'roles']);

// Select เฉพาะ Columns ที่ต้องการ
$users = User::select(['id', 'name', 'email'])
    ->with(['profile:user_id,bio,avatar'])
    ->get();

// Chunk สำหรับ Large Dataset
User::where('active', true)
    ->chunk(100, function ($users) {
        foreach ($users as $user) {
            // Process...
        }
    });

// chunkById - เร็วกว่า chunk สำหรับ Large Tables
User::chunkById(100, function ($users) {
    // Process batch
}, 'id');

// Lazy Collection - Memory Efficient
User::lazy()->each(function ($user) {
    // Process one at a time without loading all to memory
});

// Query Optimization ด้วย Indexes
// migrations/add_indexes.php
Schema::table('orders', function (Blueprint $table) {
    // Composite Index สำหรับ Query ที่ Filter หลาย Column
    $table->index(['user_id', 'status', 'created_at'], 'orders_user_status_date_idx');
    
    // Partial Index (MySQL 8.0+)
    $table->rawIndex('(status) WHERE status = "pending"', 'orders_pending_idx');
    
    // Full-text Index
    $table->fullText('description', 'products_description_fulltext');
});

// Explain Query
$query = Order::where('user_id', 1)
    ->where('status', 'completed')
    ->orderBy('created_at', 'desc');

$sql = $query->toSql();
$bindings = $query->getBindings();

// ดู Execution Plan
$explain = \DB::select("EXPLAIN " . $sql, $bindings);
dd($explain);

// Query Scopes สำหรับ Reusable Queries
class Order extends Model
{
    public function scopeRecent($query, int $days = 30)
    {
        return $query->where('created_at', '>=', now()->subDays($days));
    }
    
    public function scopeCompleted($query)
    {
        return $query->where('status', 'completed');
    }
    
    public function scopeHighValue($query, float $threshold = 1000)
    {
        return $query->where('total', '>=', $threshold);
    }
    
    public function scopeWithSummary($query)
    {
        return $query->select([
            'id', 'user_id', 'status', 'total', 'created_at'
        ])->with(['user:id,name']);
    }
}

// การใช้งาน Scopes
$orders = Order::recent(7)
    ->completed()
    ->highValue(5000)
    ->withSummary()
    ->paginate(20);

// Subqueries ที่มีประสิทธิภาพ
// ดึง User พร้อม Last Order Date
$users = User::addSelect([
    'last_order_at' => Order::select('created_at')
        ->whereColumn('user_id', 'users.id')
        ->latest()
        ->limit(1)
])->get();

// Union Queries
$pendingOrders = Order::where('status', 'pending')->select(['id', 'total', 'created_at']);
$completedOrders = Order::where('status', 'completed')->select(['id', 'total', 'created_at']);

$allOrders = $pendingOrders->union($completedOrders)->orderBy('created_at')->get();
```

---

## PHP Performance Profiling

### Xhprof

```php
<?php

// Setup Xhprof
if (extension_loaded('xhprof')) {
    xhprof_enable(XHPROF_FLAGS_CPU | XHPROF_FLAGS_MEMORY);
    
    // ... run your code ...
    
    $data = xhprof_disable();
    
    // Save profile data
    file_put_contents(
        '/tmp/xhprof/' . uniqid() . '.xhprof',
        serialize($data)
    );
}

// Laravel Middleware for Profiling
class XhprofMiddleware
{
    public function handle(Request $request, \Closure $next): Response
    {
        if (!app()->environment('local') || !extension_loaded('xhprof')) {
            return $next($request);
        }
        
        if ($request->header('X-Profile') !== 'true') {
            return $next($request);
        }
        
        xhprof_enable(XHPROF_FLAGS_CPU | XHPROF_FLAGS_MEMORY);
        
        $response = $next($request);
        
        $data = xhprof_disable();
        $id = uniqid();
        file_put_contents("/tmp/xhprof/{$id}.xhprof", serialize($data));
        
        $response->headers->set('X-Profile-Id', $id);
        
        return $response;
    }
}
```

### Blackfire Integration

```php
<?php

// ติดตั้ง Blackfire Probe
// .env
// BLACKFIRE_SERVER_ID=your_server_id
// BLACKFIRE_SERVER_TOKEN=your_server_token

// Blackfire SDK
$config = new \Blackfire\ClientConfiguration(
    serverToken: config('services.blackfire.server_token')
);
$client = new \Blackfire\Client($config);

$probe = $client->createProbe();
// ... code to profile ...
$client->endProbe($probe);

// Performance Monitoring Middleware
class PerformanceMonitorMiddleware
{
    public function handle(Request $request, \Closure $next): Response
    {
        $start = microtime(true);
        $startMemory = memory_get_usage();
        
        $response = $next($request);
        
        $duration = (microtime(true) - $start) * 1000;
        $memoryUsed = (memory_get_usage() - $startMemory) / 1024 / 1024;
        
        // Log slow requests
        if ($duration > 500) { // > 500ms
            logger()->warning('Slow request detected', [
                'url' => $request->fullUrl(),
                'method' => $request->method(),
                'duration_ms' => round($duration, 2),
                'memory_mb' => round($memoryUsed, 2),
                'db_queries' => count(\DB::getQueryLog()),
            ]);
        }
        
        $response->headers->set('X-Response-Time', round($duration, 2) . 'ms');
        
        return $response;
    }
}
```

---

## Application Level Caching

```php
<?php

// View Caching
class ProductListController extends Controller
{
    public function index(Request $request): Response
    {
        $cacheKey = 'products:list:' . md5($request->query->toString());
        
        // Full Response Caching
        return Cache::remember($cacheKey, 300, function () use ($request) {
            $products = Product::filter($request->all())
                ->with(['mainImage', 'category'])
                ->paginate(20);
            
            return response()->json($products);
        });
    }
}

// Memoization ใน Application
class StatisticsService
{
    private array $memo = [];
    
    public function getDailyRevenue(\DateTime $date): float
    {
        $key = $date->format('Y-m-d');
        
        if (!isset($this->memo[$key])) {
            $this->memo[$key] = Order::whereDate('created_at', $key)
                ->where('status', 'completed')
                ->sum('total');
        }
        
        return $this->memo[$key];
    }
}

// Background Job สำหรับ Heavy Computation
class GenerateReportJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public function handle(): void
    {
        $data = $this->computeReport();
        Cache::put('monthly_report:' . date('Y-m'), $data, 86400);
        
        // Notify user
        event(new ReportGeneratedEvent($data));
    }
    
    private function computeReport(): array
    {
        // Heavy computation...
        return DB::select(<<<SQL
            SELECT 
                DATE(created_at) as date,
                COUNT(*) as order_count,
                SUM(total) as revenue,
                AVG(total) as avg_order_value
            FROM orders
            WHERE 
                status = 'completed'
                AND created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
            GROUP BY DATE(created_at)
            ORDER BY date DESC
        SQL);
    }
}
```

---

## PHP 8.x Performance Features

```php
<?php

// JIT Compiler (PHP 8.0+)
// ตรวจสอบ JIT Status
$jitInfo = opcache_get_status()['jit'] ?? [];
echo "JIT enabled: " . ($jitInfo['enabled'] ? 'yes' : 'no') . "\n";

// Named Arguments (PHP 8.0)
function createUser(
    string $name,
    string $email,
    string $role = 'user',
    bool $active = true
): User {
    // ...
}

// แทนที่จะเขียน
createUser('John', 'john@example.com', 'user', true);

// ใช้ Named Arguments ชัดเจนกว่า
createUser(name: 'John', email: 'john@example.com', active: true);

// Fibers (PHP 8.1) - Cooperative Multitasking
$fiber = new \Fiber(function (): void {
    $value = \Fiber::suspend('first');
    echo "Resumed with: {$value}\n";
});

$value = $fiber->start();
echo "Fiber suspended with: {$value}\n"; // "first"
$fiber->resume('second');

// Enums (PHP 8.1) - เร็วกว่า Class Constants
enum OrderStatus: string
{
    case PENDING = 'pending';
    case CONFIRMED = 'confirmed';
    case SHIPPED = 'shipped';
    case DELIVERED = 'delivered';
    case CANCELLED = 'cancelled';
    
    public function label(): string
    {
        return match($this) {
            self::PENDING => 'รอดำเนินการ',
            self::CONFIRMED => 'ยืนยันแล้ว',
            self::SHIPPED => 'จัดส่งแล้ว',
            self::DELIVERED => 'ส่งถึงแล้ว',
            self::CANCELLED => 'ยกเลิกแล้ว',
        };
    }
    
    public function canTransitionTo(self $newStatus): bool
    {
        return match($this) {
            self::PENDING => in_array($newStatus, [self::CONFIRMED, self::CANCELLED]),
            self::CONFIRMED => in_array($newStatus, [self::SHIPPED, self::CANCELLED]),
            self::SHIPPED => $newStatus === self::DELIVERED,
            default => false
        };
    }
}

// Readonly Properties (PHP 8.1)
class ProductDTO
{
    public function __construct(
        public readonly int $id,
        public readonly string $name,
        public readonly float $price,
        public readonly string $currency = 'THB'
    ) {}
}

// First-class Callables (PHP 8.1)
$users = User::all();
$emails = array_map(fn($u) => $u->email, $users);

// PHP 8.1+
$emails = array_map(User::getEmailFn(), $users);
```

---

## Horizontal Scaling

```php
<?php

// Session ใน Redis สำหรับ Multi-server
// config/session.php
return [
    'driver' => 'redis',
    'connection' => 'session',
    // ...
];

// config/database.php - Redis connections
'redis' => [
    'session' => [
        'url' => env('REDIS_SESSION_URL'),
        'host' => env('REDIS_SESSION_HOST', '127.0.0.1'),
        'password' => env('REDIS_SESSION_PASSWORD'),
        'port' => env('REDIS_SESSION_PORT', 6379),
        'database' => 1,
    ],
    'cache' => [
        'url' => env('REDIS_CACHE_URL'),
        'host' => env('REDIS_CACHE_HOST', '127.0.0.1'),
        'password' => env('REDIS_CACHE_PASSWORD'),
        'port' => env('REDIS_CACHE_PORT', 6379),
        'database' => 2,
    ],
],

// File Storage ใน S3 สำหรับ Multi-server
// config/filesystems.php
's3' => [
    'driver' => 's3',
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION'),
    'bucket' => env('AWS_BUCKET'),
    'url' => env('AWS_URL'),
    'endpoint' => env('AWS_ENDPOINT'),
    'use_path_style_endpoint' => env('AWS_USE_PATH_STYLE_ENDPOINT', false),
],

// Load Balancer Health Check
Route::get('/health', function () {
    return response()->json([
        'status' => 'ok',
        'server' => gethostname(),
        'timestamp' => now()->toISOString(),
    ]);
});

// Sticky Sessions (สำหรับ WebSocket)
// ใช้ X-Forwarded-For header ใน Nginx
upstream websocket-servers {
    ip_hash; # Sticky sessions based on IP
    server ws1:6001;
    server ws2:6001;
    server ws3:6001;
}
```

---

## Workshop: Optimize Slow Application

### Step 1: วัดก่อน Optimize

```php
<?php

// Debug Bar สำหรับ Development
// composer require barryvdh/laravel-debugbar --dev

// AppServiceProvider.php
public function boot(): void
{
    if (config('app.debug')) {
        \DB::listen(function ($query) {
            if ($query->time > 100) { // > 100ms
                logger()->warning('Slow query', [
                    'sql' => $query->sql,
                    'bindings' => $query->bindings,
                    'time_ms' => $query->time,
                    'backtrace' => debug_backtrace(DEBUG_BACKTRACE_IGNORE_ARGS, 5),
                ]);
            }
        });
    }
}

// Performance Test
class PerformanceBenchmark
{
    public function run(callable $code, int $iterations = 1000): array
    {
        $times = [];
        $memoryUsages = [];
        
        for ($i = 0; $i < $iterations; $i++) {
            $start = microtime(true);
            $startMem = memory_get_usage();
            
            $code();
            
            $times[] = (microtime(true) - $start) * 1000;
            $memoryUsages[] = memory_get_usage() - $startMem;
        }
        
        return [
            'iterations' => $iterations,
            'total_ms' => array_sum($times),
            'avg_ms' => array_sum($times) / count($times),
            'min_ms' => min($times),
            'max_ms' => max($times),
            'avg_memory_kb' => array_sum($memoryUsages) / count($memoryUsages) / 1024,
        ];
    }
}

// ทดสอบ
$bench = new PerformanceBenchmark();

// Test Array vs Collection
$result1 = $bench->run(function () {
    $items = range(1, 1000);
    return array_filter($items, fn($n) => $n % 2 === 0);
});

$result2 = $bench->run(function () {
    return collect(range(1, 1000))->filter(fn($n) => $n % 2 === 0);
});

echo "Array: {$result1['avg_ms']}ms\n";
echo "Collection: {$result2['avg_ms']}ms\n";
```

### Step 2: Fix N+1 Queries

```php
<?php

// Before (N+1)
$orders = Order::where('status', 'pending')->get(); // 1 query
foreach ($orders as $order) {
    echo $order->user->name . ': ' . $order->total . "\n"; // N queries
    foreach ($order->items as $item) { // N queries
        echo "  - " . $item->product->name . "\n"; // N*M queries
    }
}

// After (Eager Loading)
$orders = Order::where('status', 'pending')
    ->with([
        'user:id,name,email',
        'items.product:id,name,price'
    ])
    ->get();
// Total: 3 queries instead of 1 + N + N*M

// เพิ่ม Global Scope ป้องกัน N+1
class Order extends Model
{
    // Always load these relations
    protected $with = ['user']; // Careful: โหลดทุกครั้ง
}

// ดีกว่า: ใช้ withDefault()
class Order extends Model
{
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class)->withDefault([
            'name' => 'Unknown User',
            'email' => '',
        ]);
    }
}
```

### Step 3: Cache Heavy Computations

```php
<?php

class DashboardService
{
    public function getStats(): array
    {
        return Cache::remember('dashboard:stats', 300, function () {
            return [
                'total_users' => User::count(),
                'active_users' => User::where('last_login_at', '>', now()->subDays(30))->count(),
                'total_orders' => Order::count(),
                'pending_orders' => Order::where('status', 'pending')->count(),
                'monthly_revenue' => Order::where('status', 'completed')
                    ->whereMonth('created_at', now()->month)
                    ->sum('total'),
                'top_products' => Product::withCount(['orderItems as sold'])
                    ->orderByDesc('sold')
                    ->limit(5)
                    ->get(['id', 'name', 'sold'])
                    ->toArray(),
            ];
        });
    }
}
```

---

## สรุป Performance Checklist

```
✅ OPcache เปิดใช้งาน
✅ JIT เปิดใช้งาน (PHP 8.0+)
✅ Composer Autoloader Optimized (composer dump-autoload -o)
✅ Config/Route/View Cached (php artisan optimize)
✅ Eager Loading แก้ N+1
✅ Database Indexes อยู่ที่ Column ที่ Filter/Sort บ่อย
✅ Redis Cache สำหรับ Expensive Queries
✅ Queue สำหรับ Heavy Background Tasks
✅ CDN สำหรับ Static Assets
✅ HTTP/2 เปิดใช้งาน
✅ Response Compression (gzip/brotli)
✅ Connection Pooling สำหรับ DB
✅ Horizontal Scaling พร้อมใช้
```

---

*Profile before you optimize. อย่า Guess ว่าช้าตรงไหน - วัดก่อนเสมอ*
