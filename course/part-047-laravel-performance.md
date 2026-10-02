# Part 047: Laravel Performance — Optimization เพื่อ Scale

**ระดับ: ระดับโลก (World-class)**
**เวลาเรียน: 6-8 ชั่วโมง**

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- Optimize Database ด้วย Indexes และ Eager Loading
- เขียน Query ที่มีประสิทธิภาพ
- ใช้ Laravel Horizon สำหรับ Queue Monitoring
- ใช้ Laravel Telescope สำหรับ Debugging
- ตั้งค่า Laravel Octane ด้วย Swoole/RoadRunner
- ทำ Performance Profiling และ Optimization

---

## 1. Database Optimization

### 1.1 Database Indexes

Index ทำให้ Query เร็วขึ้นหลายเท่า แต่ช้าลง Insert/Update/Delete

```php
// database/migrations/optimize_products_table.php
Schema::table('products', function (Blueprint $table) {
    // Single column index
    $table->index('category_id');
    $table->index('status');
    $table->index('created_at');
    
    // Composite index (column order สำคัญมาก)
    $table->index(['category_id', 'status', 'created_at']);
    
    // Unique index
    $table->unique('slug');
    
    // Full-text index (MySQL)
    $table->fullText(['name', 'description']);
});
```

**กฎของ Composite Index:**
```sql
-- Composite index: (category_id, status, created_at)
-- ใช้ index ได้:
SELECT * FROM products WHERE category_id = 1
SELECT * FROM products WHERE category_id = 1 AND status = 'active'
SELECT * FROM products WHERE category_id = 1 AND status = 'active' AND created_at > '2024-01-01'

-- ไม่ใช้ index (ข้ามคอลัมน์แรก):
SELECT * FROM products WHERE status = 'active'
SELECT * FROM products WHERE created_at > '2024-01-01'
```

**วิเคราะห์ Query ด้วย EXPLAIN:**
```php
// ดู Execution Plan
$products = DB::select('EXPLAIN SELECT * FROM products WHERE category_id = 1');
dd($products);

// ใน Laravel Query Builder
$query = Product::where('category_id', 1)->toSql();
$bindings = Product::where('category_id', 1)->getBindings();

$result = DB::select('EXPLAIN ' . $query, $bindings);
```

### 1.2 Eager Loading ป้องกัน N+1 Problem

```php
// BAD: N+1 Problem - 1 query + N queries
$orders = Order::all(); // 1 query

foreach ($orders as $order) {
    echo $order->user->name;    // N queries
    echo $order->product->name; // N queries
}

// GOOD: Eager Loading - 3 queries total
$orders = Order::with(['user', 'product'])->get();

foreach ($orders as $order) {
    echo $order->user->name;    // No additional query
    echo $order->product->name; // No additional query
}
```

**Nested Eager Loading:**
```php
// Load relationships หลายระดับ
$orders = Order::with([
    'user.profile',              // User + Profile
    'user.addresses',            // User + Addresses
    'items.product.category',    // Items + Product + Category
    'items.product.images',      // Items + Product + Images
])->get();

// Conditional Eager Loading
$orders = Order::with([
    'items' => function ($query) {
        $query->where('status', '!=', 'cancelled')
              ->orderBy('created_at');
    },
    'user:id,name,email',        // Load เฉพาะบาง columns
])->get();
```

**Lazy Eager Loading:**
```php
// โหลดทีหลังเมื่อต้องการ
$orders = Order::all();

// ตรวจสอบก่อนว่าต้องการ load หรือไม่
if ($needsDetails) {
    $orders->load(['user', 'items']);
}
```

### 1.3 Query Optimization

```php
// Select เฉพาะ columns ที่ต้องการ
// BAD: SELECT *
$users = User::all();

// GOOD: SELECT เฉพาะที่ต้องการ
$users = User::select('id', 'name', 'email')->get();

// BAD: Count ด้วย collection
$count = User::all()->count();

// GOOD: Count ด้วย SQL
$count = User::count();

// BAD: Exists check ด้วย first()
$exists = User::where('email', $email)->first() !== null;

// GOOD: exists()
$exists = User::where('email', $email)->exists();

// Chunking สำหรับ large datasets
User::chunk(1000, function ($users) {
    foreach ($users as $user) {
        // process...
    }
});

// LazyCollection - Memory efficient
User::lazy()->each(function ($user) {
    // process one at a time
});

// Cursor - ใช้ Server-side cursor
foreach (User::cursor() as $user) {
    // process...
}
```

**Subqueries:**
```php
// BAD: N+1
$users = User::all();
foreach ($users as $user) {
    $user->latest_order = $user->orders()->latest()->first();
}

// GOOD: Subquery
$users = User::addSelect([
    'latest_order_id' => Order::select('id')
        ->whereColumn('user_id', 'users.id')
        ->latest()
        ->limit(1),
])->with('latestOrder')->get();
```

---

## 2. Caching ระดับ Database

### 2.1 Query Caching

```php
class ProductRepository
{
    public function getPopular(int $limit = 10): Collection
    {
        return Cache::remember(
            'products:popular:' . $limit,
            now()->addHour(),
            fn() => Product::withCount('orders')
                ->orderByDesc('orders_count')
                ->limit($limit)
                ->get()
        );
    }
    
    public function search(string $query, array $filters = []): LengthAwarePaginator
    {
        $cacheKey = 'products:search:' . md5($query . serialize($filters));
        
        // ไม่ Cache search results (เปลี่ยนบ่อย)
        // แต่ Cache autocomplete suggestions
        $suggestions = Cache::remember(
            'products:suggestions:' . $query,
            now()->addMinutes(5),
            fn() => Product::where('name', 'like', $query . '%')->take(10)->pluck('name')
        );
        
        return Product::search($query)->paginate(20);
    }
}
```

### 2.2 Raw Query Optimization

```php
// ใช้ Raw Query เมื่อ Eloquent ไม่เพียงพอ
$stats = DB::select(
    'SELECT 
        DATE(created_at) as date,
        COUNT(*) as orders,
        SUM(total) as revenue,
        AVG(total) as avg_order_value
    FROM orders
    WHERE created_at >= ? AND status = ?
    GROUP BY DATE(created_at)
    ORDER BY date DESC
    LIMIT ?',
    [now()->subDays(30), 'completed', 30]
);

// Window Functions (MySQL 8+)
$topProducts = DB::select(
    'SELECT 
        product_id,
        total_sold,
        RANK() OVER (ORDER BY total_sold DESC) as rank
    FROM (
        SELECT product_id, SUM(quantity) as total_sold
        FROM order_items
        GROUP BY product_id
    ) t
    LIMIT 10'
);
```

---

## 3. Laravel Horizon

Horizon เป็น Dashboard สำหรับ Monitor และจัดการ Queue Workers

### 3.1 ติดตั้ง Horizon

```bash
composer require laravel/horizon
php artisan horizon:install
```

### 3.2 ตั้งค่า Horizon

```php
// config/horizon.php
return [
    'domain' => env('HORIZON_DOMAIN'),
    'path' => env('HORIZON_PATH', 'horizon'),
    
    'driver' => env('QUEUE_CONNECTION', 'redis'),
    
    'memory_limit' => 64, // MB
    
    // Authentication
    'middleware' => ['web'],
    
    'waits' => [
        'redis:default' => 60,
    ],
    
    'trim' => [
        'recent' => 60,    // นาที
        'pending' => 60,
        'completed' => 60,
        'recent_failed' => 10080, // 7 วัน
        'failed' => 10080,
        'monitored' => 10080,
    ],
    
    'silenced' => [
        // Jobs ที่ไม่ต้องการ Monitor
    ],
    
    'metrics' => [
        'trim_snapshots' => [
            'job' => 24,   // ชั่วโมง
            'queue' => 24,
        ],
    ],
    
    'fast_termination' => false,
    
    'timeout' => 60,
    
    // กำหนด Environments และ Workers
    'environments' => [
        'production' => [
            'supervisor-1' => [
                'connection' => 'redis',
                'queue' => ['default'],
                'balance' => 'auto',
                'autoScalingStrategy' => 'time',
                'minProcesses' => 1,
                'maxProcesses' => 10,
                'balanceMaxShift' => 1,
                'balanceCooldown' => 3,
                'tries' => 3,
                'timeout' => 60,
            ],
            
            'supervisor-notifications' => [
                'connection' => 'redis',
                'queue' => ['notifications'],
                'balance' => 'simple',
                'processes' => 3,
                'tries' => 3,
            ],
            
            'supervisor-heavy' => [
                'connection' => 'redis',
                'queue' => ['reports', 'exports'],
                'balance' => 'simple',
                'processes' => 1,
                'tries' => 1,
                'timeout' => 300, // 5 นาที
            ],
        ],
        
        'local' => [
            'supervisor-1' => [
                'connection' => 'redis',
                'queue' => ['default', 'notifications', 'reports'],
                'balance' => 'simple',
                'processes' => 3,
                'tries' => 3,
            ],
        ],
    ],
];
```

### 3.3 ป้องกัน Horizon Dashboard

```php
// app/Providers/HorizonServiceProvider.php
namespace App\Providers;

use Illuminate\Support\Facades\Gate;
use Laravel\Horizon\Horizon;
use Laravel\Horizon\HorizonApplicationServiceProvider;

class HorizonServiceProvider extends HorizonApplicationServiceProvider
{
    public function boot(): void
    {
        parent::boot();
        
        // เพิ่ม Badge แสดงจำนวน Failed Jobs
        Horizon::routeSmsNotificationsTo('0812345678');
        Horizon::routeMailNotificationsTo('admin@example.com');
        Horizon::routeSlackNotificationsTo(env('HORIZON_SLACK_WEBHOOK'));
        
        // Auto-tag Jobs
        Horizon::tag(function ($job) {
            if (isset($job->user)) {
                return ['user:' . $job->user->id];
            }
            return [];
        });
    }
    
    protected function gate(): void
    {
        Gate::define('viewHorizon', function ($user) {
            return in_array($user->email, [
                'admin@example.com',
            ]);
        });
    }
}
```

### 3.4 รัน Horizon

```bash
# Start Horizon
php artisan horizon

# Stop Horizon gracefully
php artisan horizon:terminate

# Check status
php artisan horizon:status

# Pause/Continue processing
php artisan horizon:pause
php artisan horizon:continue

# ตรวจสอบ Metrics
php artisan horizon:snapshot
```

---

## 4. Laravel Telescope

Telescope เป็น Debug Assistant สำหรับ Development

### 4.1 ติดตั้ง Telescope

```bash
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate
```

### 4.2 ตั้งค่า Telescope

```php
// config/telescope.php
return [
    'driver' => env('TELESCOPE_DRIVER', 'database'),
    
    'storage' => [
        'database' => [
            'connection' => env('DB_CONNECTION', 'mysql'),
            'chunk' => 1000,
        ],
    ],
    
    'enabled' => env('TELESCOPE_ENABLED', true),
    
    'domain' => env('TELESCOPE_DOMAIN'),
    
    'path' => env('TELESCOPE_PATH', 'telescope'),
    
    'middleware' => [
        'web',
        Authorize::class,
    ],
    
    'only_paths' => [
        // 'api/*'
    ],
    
    'ignore_paths' => [
        'nova-api*',
        'telescope*',
    ],
    
    'ignore_commands' => [
        'schedule:run',
    ],
    
    'watchers' => [
        Watchers\CacheWatcher::class => [
            'enabled' => env('TELESCOPE_CACHE_WATCHER', true),
            'hidden' => [],
        ],
        Watchers\CommandWatcher::class => [
            'enabled' => env('TELESCOPE_COMMAND_WATCHER', true),
            'ignore' => [],
        ],
        Watchers\DumpWatcher::class => [
            'enabled' => env('TELESCOPE_DUMP_WATCHER', true),
            'always' => env('TELESCOPE_DUMP_WATCHER_ALWAYS', false),
        ],
        Watchers\EventWatcher::class => [
            'enabled' => env('TELESCOPE_EVENT_WATCHER', true),
            'ignore' => [],
        ],
        Watchers\ExceptionWatcher::class => [
            'enabled' => env('TELESCOPE_EXCEPTION_WATCHER', true),
            'ignore' => [],
        ],
        Watchers\JobWatcher::class => [
            'enabled' => env('TELESCOPE_JOB_WATCHER', true),
            'ignore' => [],
        ],
        Watchers\LogWatcher::class => [
            'enabled' => env('TELESCOPE_LOG_WATCHER', true),
            'level' => 'error',
        ],
        Watchers\MailWatcher::class => [
            'enabled' => env('TELESCOPE_MAIL_WATCHER', true),
        ],
        Watchers\ModelWatcher::class => [
            'enabled' => env('TELESCOPE_MODEL_WATCHER', true),
            'events' => ['eloquent.created*', 'eloquent.updated*', 'eloquent.deleted*'],
            'hydrations' => false,
        ],
        Watchers\NotificationWatcher::class => [
            'enabled' => env('TELESCOPE_NOTIFICATION_WATCHER', true),
        ],
        Watchers\QueryWatcher::class => [
            'enabled' => env('TELESCOPE_QUERY_WATCHER', true),
            'ignore_packages' => true,
            'slow' => 100, // milliseconds
        ],
        Watchers\RedisWatcher::class => [
            'enabled' => env('TELESCOPE_REDIS_WATCHER', true),
        ],
        Watchers\RequestWatcher::class => [
            'enabled' => env('TELESCOPE_REQUEST_WATCHER', true),
            'size_limit' => env('TELESCOPE_RESPONSE_SIZE_LIMIT', 64),
            'ignore_http_methods' => [],
            'ignore_status_codes' => [],
        ],
        Watchers\ScheduleWatcher::class => [
            'enabled' => env('TELESCOPE_SCHEDULE_WATCHER', true),
        ],
        Watchers\ViewWatcher::class => [
            'enabled' => env('TELESCOPE_VIEW_WATCHER', true),
        ],
    ],
];
```

### 4.3 Telescope Authorization

```php
// app/Providers/TelescopeServiceProvider.php
namespace App\Providers;

use App\Models\User;
use Illuminate\Support\Facades\Gate;
use Laravel\Telescope\IncomingEntry;
use Laravel\Telescope\Telescope;
use Laravel\Telescope\TelescopeApplicationServiceProvider;

class TelescopeServiceProvider extends TelescopeApplicationServiceProvider
{
    public function register(): void
    {
        Telescope::night(); // Dark mode
        
        // Filter entries ใน Production
        $this->hideSensitiveRequestDetails();
        
        $isLocal = $this->app->environment('local');
        
        Telescope::filter(function (IncomingEntry $entry) use ($isLocal) {
            if ($isLocal) {
                return true;
            }
            
            return $entry->isReportableException() ||
                   $entry->isFailedRequest() ||
                   $entry->isFailedJob() ||
                   $entry->isScheduledTask() ||
                   $entry->hasMonitoredTag();
        });
    }
    
    private function hideSensitiveRequestDetails(): void
    {
        if ($this->app->environment('local')) {
            return;
        }
        
        Telescope::hideRequestParameters([
            'password',
            'password_confirmation',
            'credit_card_number',
            'cvv',
        ]);
        
        Telescope::hideRequestHeaders([
            'cookie',
            'x-csrf-token',
            'x-xsrf-token',
        ]);
    }
    
    protected function gate(): void
    {
        Gate::define('viewTelescope', function (User $user) {
            return $user->hasRole('developer');
        });
    }
}
```

---

## 5. Laravel Octane

Octane เป็น High-performance HTTP server สำหรับ Laravel ที่ใช้ Swoole หรือ RoadRunner

### 5.1 ติดตั้ง Octane

```bash
composer require laravel/octane

# สำหรับ Swoole
pecl install swoole

# สำหรับ RoadRunner
./vendor/bin/rr get

php artisan octane:install
```

### 5.2 ตั้งค่า Octane

```php
// config/octane.php
return [
    'server' => env('OCTANE_SERVER', 'swoole'),
    
    'https' => env('OCTANE_HTTPS', false),
    
    'listeners' => [
        WorkerStarting::class => [
            EnsureUploadedFilesAreValid::class,
            EnsureEnvironmentVariablesAreValid::class,
        ],
        
        RequestReceived::class => [
            ...Octane::prepareApplicationForNextOperation(),
            ...Octane::prepareApplicationForNextRequest(),
        ],
        
        TaskReceived::class => [
            ...Octane::prepareApplicationForNextOperation(),
        ],
        
        TickReceived::class => [
            ...Octane::prepareApplicationForNextOperation(),
        ],
        
        RequestHandled::class => [],
        
        RequestTerminated::class => [],
    ],
    
    'warm' => [
        ...Octane::defaultServicesToWarm(),
    ],
    
    'flush' => [
        // Services ที่ต้อง flush ทุก request
    ],
    
    'garbage' => 50, // Collect garbage every 50 requests
    
    'max_execution_time' => 30,
    
    'tick_interval' => 1, // Seconds between ticks
    
    'swoole' => [
        'options' => [
            'log_file' => storage_path('logs/swoole_http.log'),
            'worker_num' => swoole_cpu_num(),
            'task_worker_num' => swoole_cpu_num() * 2,
            'package_max_length' => 10 * 1024 * 1024,
        ],
    ],
];
```

### 5.3 Octane-Safe Code

Octane ทำงานแบบ Long-running process ต้องระวัง Memory Leaks

```php
// BAD: Static properties จะ persist ระหว่าง requests
class UserService
{
    private static array $cache = []; // ปัญหา! ข้ามไปทุก request
    
    public static function getUser(int $id): User
    {
        if (!isset(static::$cache[$id])) {
            static::$cache[$id] = User::find($id);
        }
        return static::$cache[$id];
    }
}

// GOOD: ใช้ per-request cache
class UserService
{
    public function getUser(int $id): User
    {
        return cache()->remember('user:' . $id, 60, fn() => User::find($id));
    }
}
```

```php
// BAD: Global state
app()->singleton('current-user', function () {
    return auth()->user(); // ค่านี้จะ persist ข้าม requests
});

// GOOD: Request-scoped bindings
app()->scoped('current-user', function () {
    return auth()->user();
});
```

### 5.4 Octane Tasks (Concurrent)

```php
use Laravel\Octane\Facades\Octane;

class DashboardController extends Controller
{
    public function index()
    {
        // รัน Tasks แบบ Concurrent
        [$users, $revenue, $orders] = Octane::concurrently([
            fn() => User::count(),
            fn() => Order::where('status', 'completed')->sum('total'),
            fn() => Order::where('created_at', '>=', now()->startOfDay())->count(),
        ]);
        
        return response()->json(compact('users', 'revenue', 'orders'));
    }
}
```

### 5.5 รัน Octane

```bash
# Start Octane
php artisan octane:start --workers=4 --task-workers=6

# Hot reload (development)
php artisan octane:start --watch

# Production
php artisan octane:start --server=swoole --host=0.0.0.0 --port=8000 --workers=auto
```

---

## 6. Response Optimization

### 6.1 Response Compression

```php
// app/Http/Kernel.php
protected $middleware = [
    \Illuminate\Http\Middleware\HandleCors::class,
    \App\Http\Middleware\CompressResponse::class,
];
```

```php
// app/Http/Middleware/CompressResponse.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class CompressResponse
{
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);
        
        if (!$request->acceptsHtml() && str_contains($request->header('Accept-Encoding', ''), 'gzip')) {
            $content = $response->getContent();
            
            if (strlen($content) > 1024) { // 1KB minimum
                $response->setContent(gzencode($content, 6));
                $response->header('Content-Encoding', 'gzip');
                $response->header('Content-Length', strlen($response->getContent()));
            }
        }
        
        return $response;
    }
}
```

### 6.2 HTTP Caching Headers

```php
class ProductController extends Controller
{
    public function show(Product $product)
    {
        $response = new ProductResource($product);
        
        return $response
            ->response()
            ->setLastModified($product->updated_at)
            ->setEtag(md5($product->updated_at->timestamp))
            ->setPublic()
            ->setMaxAge(3600)
            ->setSharedMaxAge(3600);
    }
}
```

---

## 7. Workshop: Performance Profiling และ Optimization

### Step 1: สร้าง Profiling Service

```php
// app/Services/PerformanceProfiler.php
namespace App\Services;

use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class PerformanceProfiler
{
    private array $queries = [];
    private float $startTime;
    private int $startMemory;
    
    public function start(): void
    {
        $this->startTime = microtime(true);
        $this->startMemory = memory_get_usage(true);
        $this->queries = [];
        
        DB::listen(function ($query) {
            $this->queries[] = [
                'sql' => $query->sql,
                'bindings' => $query->bindings,
                'time' => $query->time,
            ];
        });
    }
    
    public function report(string $label = 'Performance'): array
    {
        $endTime = microtime(true);
        $endMemory = memory_get_usage(true);
        
        $report = [
            'label' => $label,
            'execution_time_ms' => round(($endTime - $this->startTime) * 1000, 2),
            'memory_usage_mb' => round(($endMemory - $this->startMemory) / 1024 / 1024, 2),
            'query_count' => count($this->queries),
            'slow_queries' => array_filter($this->queries, fn($q) => $q['time'] > 100),
            'total_query_time_ms' => round(array_sum(array_column($this->queries, 'time')), 2),
        ];
        
        if (app()->environment('local')) {
            Log::channel('performance')->info($label, $report);
        }
        
        return $report;
    }
    
    public function benchmark(string $label, callable $callback, int $iterations = 100): array
    {
        $times = [];
        $memoriesUsed = [];
        
        for ($i = 0; $i < $iterations; $i++) {
            $start = microtime(true);
            $memStart = memory_get_usage(true);
            
            $callback();
            
            $times[] = (microtime(true) - $start) * 1000;
            $memoriesUsed[] = memory_get_usage(true) - $memStart;
        }
        
        sort($times);
        
        return [
            'label' => $label,
            'iterations' => $iterations,
            'min_ms' => round(min($times), 4),
            'max_ms' => round(max($times), 4),
            'avg_ms' => round(array_sum($times) / count($times), 4),
            'median_ms' => round($times[floor($iterations / 2)], 4),
            'p95_ms' => round($times[floor($iterations * 0.95)], 4),
            'avg_memory_kb' => round(array_sum($memoriesUsed) / count($memoriesUsed) / 1024, 2),
        ];
    }
}
```

### Step 2: สร้าง Slow Query Monitor

```php
// app/Http/Middleware/SlowQueryMonitor.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class SlowQueryMonitor
{
    private const SLOW_THRESHOLD = 200; // ms
    
    public function handle(Request $request, Closure $next)
    {
        $queries = [];
        
        DB::listen(function ($query) use (&$queries) {
            if ($query->time > self::SLOW_THRESHOLD) {
                $queries[] = [
                    'sql' => $query->sql,
                    'time' => $query->time,
                    'connection' => $query->connection->getDatabaseName(),
                ];
            }
        });
        
        $response = $next($request);
        
        if (!empty($queries)) {
            Log::warning('Slow queries detected', [
                'url' => $request->fullUrl(),
                'method' => $request->method(),
                'queries' => $queries,
                'total_slow_queries' => count($queries),
            ]);
        }
        
        return $response;
    }
}
```

### Step 3: Artisan Command สำหรับ Performance Analysis

```php
// app/Console/Commands/AnalyzePerformance.php
namespace App\Console\Commands;

use App\Services\PerformanceProfiler;
use Illuminate\Console\Command;

class AnalyzePerformance extends Command
{
    protected $signature = 'performance:analyze 
                            {--route= : Specific route to analyze}
                            {--iterations=10 : Number of iterations}';
    
    protected $description = 'Analyze application performance';
    
    public function handle(PerformanceProfiler $profiler): int
    {
        $this->info('Running performance analysis...');
        
        $benchmarks = $this->runBenchmarks($profiler);
        
        $this->table(
            ['Test', 'Min (ms)', 'Avg (ms)', 'Max (ms)', 'P95 (ms)', 'Memory (KB)'],
            array_map(fn($b) => [
                $b['label'],
                $b['min_ms'],
                $b['avg_ms'],
                $b['max_ms'],
                $b['p95_ms'],
                $b['avg_memory_kb'],
            ], $benchmarks)
        );
        
        $this->checkForIssues($benchmarks);
        
        return 0;
    }
    
    private function runBenchmarks(PerformanceProfiler $profiler): array
    {
        return [
            $profiler->benchmark('Products Query (No Index)', function () {
                \App\Models\Product::where('status', 'active')->count();
            }),
            
            $profiler->benchmark('Products with Eager Loading', function () {
                \App\Models\Product::with(['category', 'images'])
                    ->where('status', 'active')
                    ->take(20)
                    ->get();
            }),
            
            $profiler->benchmark('Cache Read', function () {
                cache()->get('test:key', 'default');
            }),
            
            $profiler->benchmark('Cache Write', function () {
                cache()->put('test:key', 'value', 60);
            }),
        ];
    }
    
    private function checkForIssues(array $benchmarks): void
    {
        foreach ($benchmarks as $benchmark) {
            if ($benchmark['avg_ms'] > 100) {
                $this->warn("SLOW: {$benchmark['label']} avg {$benchmark['avg_ms']}ms");
            }
            
            if ($benchmark['p95_ms'] > 500) {
                $this->error("CRITICAL: {$benchmark['label']} P95 {$benchmark['p95_ms']}ms");
            }
        }
    }
}
```

### Step 4: Configuration Optimization

```bash
# Cache Config
php artisan config:cache

# Cache Routes
php artisan route:cache

# Cache Views
php artisan view:cache

# Cache Events/Listeners
php artisan event:cache

# Optimize Autoloader
composer dump-autoload --optimize --classmap-authoritative

# All at once (Production)
php artisan optimize
```

---

## 8. Monitoring และ Alerts

```php
// app/Console/Commands/HealthCheck.php
namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Redis;

class HealthCheck extends Command
{
    protected $signature = 'health:check';
    protected $description = 'Check application health';
    
    public function handle(): int
    {
        $this->info('Running health checks...');
        
        $checks = [
            'Database' => $this->checkDatabase(),
            'Redis' => $this->checkRedis(),
            'Cache' => $this->checkCache(),
            'Queue' => $this->checkQueue(),
            'Disk Space' => $this->checkDiskSpace(),
        ];
        
        $failed = array_filter($checks, fn($status) => !$status['ok']);
        
        foreach ($checks as $name => $check) {
            $status = $check['ok'] ? '<fg=green>OK</>' : '<fg=red>FAIL</>';
            $this->line("{$name}: {$status} - {$check['message']}");
        }
        
        if (!empty($failed)) {
            $this->error('Some health checks failed!');
            return 1;
        }
        
        $this->info('All health checks passed!');
        return 0;
    }
    
    private function checkDatabase(): array
    {
        try {
            $start = microtime(true);
            DB::select('SELECT 1');
            $time = round((microtime(true) - $start) * 1000, 2);
            
            return ['ok' => true, 'message' => "Connected ({$time}ms)"];
        } catch (\Exception $e) {
            return ['ok' => false, 'message' => $e->getMessage()];
        }
    }
    
    private function checkRedis(): array
    {
        try {
            $start = microtime(true);
            Redis::ping();
            $time = round((microtime(true) - $start) * 1000, 2);
            
            return ['ok' => true, 'message' => "Connected ({$time}ms)"];
        } catch (\Exception $e) {
            return ['ok' => false, 'message' => $e->getMessage()];
        }
    }
    
    private function checkCache(): array
    {
        try {
            Cache::put('health:test', 'ok', 10);
            $value = Cache::get('health:test');
            
            if ($value !== 'ok') {
                return ['ok' => false, 'message' => 'Cache read/write mismatch'];
            }
            
            return ['ok' => true, 'message' => 'Working'];
        } catch (\Exception $e) {
            return ['ok' => false, 'message' => $e->getMessage()];
        }
    }
    
    private function checkQueue(): array
    {
        $failedCount = DB::table('failed_jobs')->count();
        
        if ($failedCount > 100) {
            return ['ok' => false, 'message' => "{$failedCount} failed jobs"];
        }
        
        return ['ok' => true, 'message' => "{$failedCount} failed jobs"];
    }
    
    private function checkDiskSpace(): array
    {
        $freeBytes = disk_free_space('/');
        $freeMB = round($freeBytes / 1024 / 1024, 2);
        
        if ($freeMB < 500) {
            return ['ok' => false, 'message' => "Only {$freeMB}MB free"];
        }
        
        return ['ok' => true, 'message' => "{$freeMB}MB free"];
    }
}
```

---

## Quiz

**ข้อ 1:** N+1 Problem คืออะไรและแก้อย่างไร?

a) มี 1 Query เกินในทุก Page  
b) ทำ 1 Query เพื่อดึง records แล้วทำอีก N queries สำหรับ related records ของแต่ละ record  
c) Query ที่มีเงื่อนไข N+1 ข้อ  
d) Database connection limit ที่ N+1

**เฉลย:** b) แก้ด้วย Eager Loading (`with()`)

---

**ข้อ 2:** Laravel Octane ทำงานต่างจาก php-fpm อย่างไร?

a) Octane เร็วกว่าเพราะใช้ Redis  
b) Octane เป็น Long-running process ไม่ต้อง boot Application ทุก request  
c) Octane ใช้ Async I/O เท่านั้น  
d) ไม่มีความแตกต่าง

**เฉลย:** b) Octane boot Application ครั้งเดียวและ keep ไว้ใน memory ทำให้ไม่ต้องทำ bootstrap ซ้ำทุก request

---

**ข้อ 3:** Composite Index `(category_id, status, price)` จะใช้กับ Query ใด?

a) `WHERE status = 'active'`  
b) `WHERE price < 100`  
c) `WHERE category_id = 1 AND status = 'active'`  
d) `WHERE status = 'active' AND price < 100`

**เฉลย:** c) Composite Index ใช้ได้เมื่อ Query เริ่มต้นด้วย column แรก (category_id) ตาม "Leftmost Prefix Rule"

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- **Database:** Indexes, Eager Loading, Query Optimization
- **Horizon:** Queue Monitoring และ Auto-scaling
- **Telescope:** Debug ด้วย Dashboard
- **Octane:** High-performance server ด้วย Swoole/RoadRunner
- **Workshop:** Profiling และ Optimization Tools

---

## ไปต่อ

➡️ [Part 048: Laravel Deployment — Deploy สู่ Production](./part-048-laravel-deployment.md)
