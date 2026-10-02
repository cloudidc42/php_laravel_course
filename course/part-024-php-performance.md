# Part 024: PHP Performance Optimization

## ระดับ: Advanced
## เวลาเรียน: 4-5 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:
- Profiling ด้วย Xdebug และ Blackfire
- ตั้งค่าและใช้ OPcache
- เขียน PHP ที่ประหยัด Memory
- เข้าใจ String Performance
- วิเคราะห์ Algorithm Complexity (Big O)
- Optimize script ที่ช้า

---

## 1. Profiling

### Xdebug Profiling

```bash
# ติดตั้ง Xdebug
pecl install xdebug

# php.ini configuration
[xdebug]
zend_extension=xdebug.so
xdebug.mode=profile
xdebug.start_with_request=trigger
xdebug.output_dir=/tmp/xdebug
xdebug.profiler_output_name=cachegrind.out.%p.%t
```

```php
<?php
// เปิด profiling ด้วย trigger parameter
// URL: http://localhost/script.php?XDEBUG_PROFILE=1

// หรือ programmatically
xdebug_start_profiling();

// ... code ที่ต้องการ profile ...

xdebug_stop_profiling();
// ไฟล์ cachegrind จะถูกสร้างใน output_dir
// เปิดด้วย KCacheGrind, QCacheGrind, หรือ Webgrind
```

### PHP Benchmark Function

```php
<?php
class Benchmark {
    private float $startTime;
    private float $startMemory;
    private array $checkpoints = [];
    
    public function start(): void {
        $this->startTime = microtime(true);
        $this->startMemory = memory_get_usage(true);
    }
    
    public function checkpoint(string $name): void {
        $this->checkpoints[$name] = [
            'time' => microtime(true) - $this->startTime,
            'memory' => memory_get_usage(true) - $this->startMemory,
        ];
    }
    
    public function end(): void {
        $this->checkpoint('end');
        $this->report();
    }
    
    public function report(): void {
        echo "\n=== Benchmark Report ===\n";
        foreach ($this->checkpoints as $name => $data) {
            $time = round($data['time'] * 1000, 2);
            $memory = round($data['memory'] / 1024 / 1024, 2);
            echo "{$name}: {$time}ms, {$memory}MB\n";
        }
    }
    
    public static function run(callable $fn, int $iterations = 1000): float {
        $start = microtime(true);
        for ($i = 0; $i < $iterations; $i++) {
            $fn();
        }
        return (microtime(true) - $start) * 1000; // milliseconds
    }
    
    public static function compare(array $functions, int $iterations = 1000): void {
        $results = [];
        
        foreach ($functions as $name => $fn) {
            $results[$name] = self::run($fn, $iterations);
        }
        
        asort($results);
        
        echo "\n=== Performance Comparison ({$iterations} iterations) ===\n";
        $fastest = reset($results);
        
        foreach ($results as $name => $time) {
            $ratio = round($time / $fastest, 2);
            $bar = str_repeat('█', (int) ($ratio * 20));
            echo sprintf("%-30s %8.2fms  %sx  %s\n", $name, $time, $ratio, $bar);
        }
    }
}

// ตัวอย่าง: เปรียบเทียบวิธีนับ array
$arr = range(1, 10000);

Benchmark::compare([
    'count($arr)' => fn() => count($arr),
    'sizeof($arr)' => fn() => sizeof($arr),
    'foreach + counter' => function() use ($arr) {
        $c = 0;
        foreach ($arr as $_) { $c++; }
        return $c;
    },
], 10000);
```

### Simple Profiling Functions

```php
<?php
// Profiling แบบง่ายๆ สำหรับ development
function profile(string $label, callable $fn): mixed {
    $start = microtime(true);
    $memBefore = memory_get_usage();
    
    $result = $fn();
    
    $time = (microtime(true) - $start) * 1000;
    $memory = memory_get_usage() - $memBefore;
    
    echo sprintf(
        "[PROFILE] %s: %.2fms, %+d bytes\n",
        $label,
        $time,
        $memory
    );
    
    return $result;
}

// ใช้งาน
$users = profile('Load users from DB', fn() => loadUsersFromDatabase());
$filtered = profile('Filter active users', fn() => array_filter($users, fn($u) => $u['active']));
$sorted = profile('Sort by name', function() use ($filtered) {
    usort($filtered, fn($a, $b) => strcmp($a['name'], $b['name']));
    return $filtered;
});
```

---

## 2. OPcache

OPcache เก็บ compiled PHP bytecode ใน shared memory ทำให้ PHP ไม่ต้อง parse/compile ทุกครั้ง

### ตั้งค่า OPcache

```ini
; php.ini
[opcache]
opcache.enable=1
opcache.enable_cli=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
opcache.max_wasted_percentage=10
opcache.validate_timestamps=1      ; 0 ใน production เร็วกว่า
opcache.revalidate_freq=60         ; seconds (ถ้า validate_timestamps=1)
opcache.fast_shutdown=1
opcache.jit_buffer_size=100M       ; PHP 8+ JIT
opcache.jit=1255                   ; PHP 8+ JIT mode
```

### ตรวจสอบ OPcache Status

```php
<?php
function getOpcacheStatus(): array {
    if (!function_exists('opcache_get_status')) {
        return ['enabled' => false, 'message' => 'OPcache not installed'];
    }
    
    $status = opcache_get_status(false);
    $config = opcache_get_configuration();
    
    if ($status === false) {
        return ['enabled' => false];
    }
    
    $memory = $status['memory_usage'];
    $total = $memory['used_memory'] + $memory['free_memory'] + $memory['wasted_memory'];
    
    return [
        'enabled' => true,
        'memory' => [
            'total_mb' => round($total / 1024 / 1024, 2),
            'used_mb' => round($memory['used_memory'] / 1024 / 1024, 2),
            'free_mb' => round($memory['free_memory'] / 1024 / 1024, 2),
            'wasted_mb' => round($memory['wasted_memory'] / 1024 / 1024, 2),
            'usage_percent' => round($memory['used_memory'] / $total * 100, 1),
        ],
        'scripts' => [
            'cached' => $status['opcache_statistics']['num_cached_scripts'],
            'hits' => $status['opcache_statistics']['hits'],
            'misses' => $status['opcache_statistics']['misses'],
            'hit_rate' => round($status['opcache_statistics']['opcache_hit_rate'], 2),
        ],
        'jit' => $status['jit'] ?? ['enabled' => false],
    ];
}

$opcacheInfo = getOpcacheStatus();
echo json_encode($opcacheInfo, JSON_PRETTY_PRINT) . "\n";

// Force clear cache (ในขณะ deploy)
function clearOpcache(): void {
    if (function_exists('opcache_reset')) {
        opcache_reset();
        echo "OPcache cleared!\n";
    }
}

// Clear specific file
function invalidateOpcacheFile(string $filepath): void {
    if (function_exists('opcache_invalidate')) {
        opcache_invalidate($filepath, force: true);
    }
}
```

### JIT Compilation (PHP 8.0+)

```php
<?php
// JIT ช่วยงาน CPU-intensive มาก (เช่น numeric computation)
// แต่ไม่ช่วยงาน I/O bound (database, file, network)

// ตัวอย่างที่ JIT ช่วยได้มาก
function fibonacci(int $n): int {
    if ($n <= 1) return $n;
    return fibonacci($n - 1) + fibonacci($n - 2);
}

// JIT modes ใน opcache.jit:
// 0    = disabled
// 1    = minimal (ใช้ทรัพยากรน้อย)
// 1205 = tracing (แนะนำสำหรับ web)
// 1235 = tracing + function
// 1255 = tracing + function (aggressive)

// ตรวจสอบ JIT status
$status = opcache_get_status();
$jit = $status['jit'] ?? null;
if ($jit) {
    echo "JIT enabled: " . ($jit['enabled'] ? 'yes' : 'no') . "\n";
    echo "JIT kind: " . ($jit['kind'] ?? 'N/A') . "\n";
}
```

---

## 3. Memory Optimization

```php
<?php
// ============================================================
// 1. ใช้ Generator แทน Array สำหรับข้อมูลจำนวนมาก
// ============================================================

// แย่: โหลดทั้งหมดเข้า memory
function getAllLogs(): array {
    $logs = [];
    $handle = fopen('/var/log/app.log', 'r');
    while (($line = fgets($handle)) !== false) {
        $logs[] = trim($line); // ทุก line อยู่ใน memory!
    }
    fclose($handle);
    return $logs;
}

// ดี: ใช้ Generator
function getLogs(): \Generator {
    $handle = fopen('/var/log/app.log', 'r');
    while (($line = fgets($handle)) !== false) {
        yield trim($line); // ทีละ line
    }
    fclose($handle);
}

// ใช้งาน
foreach (getLogs() as $line) {
    if (str_contains($line, 'ERROR')) {
        processErrorLog($line);
    }
    // memory ไม่เพิ่มขึ้น!
}

// ============================================================
// 2. Chunking สำหรับ batch processing
// ============================================================

function processLargeDataset(array $ids, callable $processor, int $chunkSize = 100): void {
    $chunks = array_chunk($ids, $chunkSize);
    
    foreach ($chunks as $i => $chunk) {
        $data = fetchFromDatabase($chunk); // fetch แค่ chunk ละครั้ง
        $processor($data);
        
        // Free memory explicitly
        unset($data);
        
        // Force garbage collection ถ้าจำเป็น
        if ($i % 10 === 0) {
            gc_collect_cycles();
        }
    }
}

// ============================================================
// 3. ลดการ Copy ด้วย References
// ============================================================

// PHP Copy-on-Write (CoW): array ถูก copy เมื่อ modify
$arr = range(1, 1000000);
$arr2 = $arr;         // ยังไม่ copy (CoW)
$arr2[0] = 999;       // ตอนนี้ copy! = 2 copies ใน memory

// ใช้ reference เพื่อหลีกเลี่ยง copy
function processArray(array &$arr): void { // reference parameter
    foreach ($arr as &$item) { // reference in foreach
        $item *= 2;
    }
    unset($item); // สำคัญ! ลบ reference หลัง foreach
}

// ============================================================
// 4. String memory optimization
// ============================================================

// แย่: string concatenation ใน loop
function buildHtmlBad(array $items): string {
    $html = '';
    foreach ($items as $item) {
        $html .= "<li>{$item}</li>"; // string grow ทุก iteration
    }
    return $html;
}

// ดี: ใช้ array + implode
function buildHtmlGood(array $items): string {
    $parts = [];
    foreach ($items as $item) {
        $parts[] = "<li>{$item}</li>";
    }
    return implode('', $parts); // join ครั้งเดียว
}

// ดีมาก: ใช้ array_map + implode
function buildHtmlBest(array $items): string {
    return implode('', array_map(fn($item) => "<li>{$item}</li>", $items));
}

// ============================================================
// 5. Lazy Loading / Deferred Computation
// ============================================================

class LazyLoader {
    private array $resolved = [];
    private array $resolvers = [];
    
    public function register(string $key, callable $resolver): void {
        $this->resolvers[$key] = $resolver;
    }
    
    public function get(string $key): mixed {
        if (!isset($this->resolved[$key])) {
            if (!isset($this->resolvers[$key])) {
                throw new \RuntimeException("Key '{$key}' not registered");
            }
            $this->resolved[$key] = ($this->resolvers[$key])();
        }
        return $this->resolved[$key];
    }
}

$lazy = new LazyLoader();

// Register ก่อน แต่ยังไม่ execute
$lazy->register('config', fn() => loadConfig()); // ไม่ทำงานตอนนี้
$lazy->register('users', fn() => fetchAllUsers()); // ไม่ทำงานตอนนี้

// Execute เฉพาะที่ต้องการ
if ($needsConfig) {
    $config = $lazy->get('config'); // โหลด config
}
// 'users' ไม่ถูกโหลดเลยถ้าไม่ต้องการ
```

---

## 4. String Performance

```php
<?php
// ============================================================
// String Operations Benchmark
// ============================================================

$str = str_repeat('Hello World PHP! ', 1000); // 17000 chars

Benchmark::compare([
    // Checking if string contains something
    'strpos != false'           => fn() => strpos($str, 'PHP') !== false,
    'str_contains'              => fn() => str_contains($str, 'PHP'),
    'preg_match'                => fn() => preg_match('/PHP/', $str),
    
    // String replacement
    'str_replace single'        => fn() => str_replace('Hello', 'Hi', $str),
    'preg_replace simple'       => fn() => preg_replace('/Hello/', 'Hi', $str),
    
    // String splitting
    'explode'                   => fn() => explode(' ', $str),
    'preg_split'                => fn() => preg_split('/\s+/', $str),
    'str_split by char'         => fn() => str_split($str),
], 1000);

// ============================================================
// Sprintf vs String Concatenation vs Interpolation
// ============================================================

$name = "สมชาย";
$age = 30;
$city = "กรุงเทพ";

Benchmark::compare([
    'concatenation'  => fn() => "ชื่อ: " . $name . " อายุ " . $age . " ปี จาก " . $city,
    'interpolation'  => fn() => "ชื่อ: {$name} อายุ {$age} ปี จาก {$city}",
    'sprintf'        => fn() => sprintf("ชื่อ: %s อายุ %d ปี จาก %s", $name, $age, $city),
    'printf to buf'  => fn() => vsprintf("ชื่อ: %s อายุ %d ปี จาก %s", [$name, $age, $city]),
], 100000);

// ============================================================
// การค้นหาใน String
// ============================================================

$haystack = str_repeat('abcdefghijklmnopqrstuvwxyz', 1000);
$needle = 'z';

Benchmark::compare([
    'strpos'              => fn() => strpos($haystack, $needle),
    'str_contains'        => fn() => str_contains($haystack, $needle),
    'stripos'             => fn() => stripos($haystack, $needle), // case-insensitive
    'preg_match'          => fn() => preg_match('/' . $needle . '/', $haystack),
    'substr_count'        => fn() => substr_count($haystack, $needle) > 0,
], 10000);

// ============================================================
// Regex: compile vs non-compile
// ============================================================

$emails = array_fill(0, 100, 'user@example.com');

// แย่: compile pattern ทุกครั้ง (ใน loop)
$count = 0;
foreach ($emails as $email) {
    if (preg_match('/^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$/i', $email)) {
        $count++;
    }
}

// ดี: เก็บ pattern ใน variable (PHP cache compiled patterns)
$pattern = '/^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$/i';
$count = 0;
foreach ($emails as $email) {
    if (preg_match($pattern, $email)) {
        $count++;
    }
}

// ดีมาก: ใช้ preg_match_all กับ array
$count = preg_match_all($pattern, implode("\n", $emails));
```

---

## 5. Algorithm Complexity (Big O)

```php
<?php
// ============================================================
// Big O Examples
// ============================================================

// O(1) - Constant time
function getArrayElement(array $arr, int $index): mixed {
    return $arr[$index]; // เสมอ O(1)
}

function hashTableLookup(array $map, string $key): mixed {
    return $map[$key] ?? null; // PHP array เป็น hash map = O(1)
}

// O(log n) - Logarithmic (Binary Search)
function binarySearch(array $sortedArr, int $target): int {
    $low = 0;
    $high = count($sortedArr) - 1;
    
    while ($low <= $high) {
        $mid = intdiv($low + $high, 2);
        
        if ($sortedArr[$mid] === $target) return $mid;
        elseif ($sortedArr[$mid] < $target) $low = $mid + 1;
        else $high = $mid - 1;
    }
    
    return -1; // ไม่พบ
}

// O(n) - Linear
function findMax(array $arr): mixed {
    $max = $arr[0];
    foreach ($arr as $item) { // ทำ n ครั้ง
        if ($item > $max) $max = $item;
    }
    return $max;
}

function hasValue(array $arr, mixed $value): bool {
    return in_array($value, $arr); // O(n) - sequential search
}

// O(n log n) - Linearithmic (Merge Sort)
function mergeSort(array $arr): array {
    $n = count($arr);
    if ($n <= 1) return $arr;
    
    $mid = intdiv($n, 2);
    $left = mergeSort(array_slice($arr, 0, $mid));
    $right = mergeSort(array_slice($arr, $mid));
    
    return merge($left, $right);
}

function merge(array $left, array $right): array {
    $result = [];
    $i = $j = 0;
    
    while ($i < count($left) && $j < count($right)) {
        if ($left[$i] <= $right[$j]) {
            $result[] = $left[$i++];
        } else {
            $result[] = $right[$j++];
        }
    }
    
    return array_merge($result, array_slice($left, $i), array_slice($right, $j));
}

// O(n²) - Quadratic (Bubble Sort - หลีกเลี่ยง!)
function bubbleSort(array $arr): array {
    $n = count($arr);
    for ($i = 0; $i < $n - 1; $i++) { // O(n)
        for ($j = 0; $j < $n - $i - 1; $j++) { // O(n)
            if ($arr[$j] > $arr[$j + 1]) {
                [$arr[$j], $arr[$j + 1]] = [$arr[$j + 1], $arr[$j]];
            }
        }
    }
    return $arr;
}

// ============================================================
// ปรับปรุง Performance ด้วย Data Structures
// ============================================================

// แย่: O(n) lookup ทุกครั้ง
function findDuplicatesSlow(array $arr): array {
    $duplicates = [];
    $n = count($arr);
    
    for ($i = 0; $i < $n; $i++) {
        for ($j = $i + 1; $j < $n; $j++) { // O(n²)!
            if ($arr[$i] === $arr[$j] && !in_array($arr[$i], $duplicates)) {
                $duplicates[] = $arr[$i];
            }
        }
    }
    
    return $duplicates;
}

// ดี: O(n) ด้วย hash map
function findDuplicatesFast(array $arr): array {
    $seen = [];
    $duplicates = [];
    
    foreach ($arr as $item) { // O(n)
        if (isset($seen[$item])) { // O(1) hash lookup
            if (!isset($seen[$item . '_dup'])) {
                $duplicates[] = $item;
                $seen[$item . '_dup'] = true;
            }
        } else {
            $seen[$item] = true;
        }
    }
    
    return $duplicates;
}

// ============================================================
// Caching Expensive Operations
// ============================================================

class MemoizedFibonacci {
    private array $cache = [];
    
    public function calculate(int $n): int {
        if (isset($this->cache[$n])) {
            return $this->cache[$n]; // O(1) cache hit
        }
        
        if ($n <= 1) {
            return $this->cache[$n] = $n;
        }
        
        return $this->cache[$n] = $this->calculate($n - 1) + $this->calculate($n - 2);
    }
}

// ไม่มี cache: O(2^n)
// มี cache: O(n)
$fib = new MemoizedFibonacci();
echo $fib->calculate(40) . "\n"; // เร็วมาก!
```

---

## 6. Workshop: Optimize Slow Script

### โจทย์: ระบบประมวลผล Product Report

```php
<?php
// ============================================================
// Version 1: โค้ดช้า (ปัญหาหลายจุด)
// ============================================================

class SlowProductReport {
    private array $products;
    private array $orders;
    
    public function __construct(int $productCount = 10000, int $orderCount = 100000) {
        // สร้างข้อมูล test
        $this->products = $this->generateProducts($productCount);
        $this->orders = $this->generateOrders($orderCount, $productCount);
    }
    
    private function generateProducts(int $count): array {
        $products = [];
        $categories = ['Electronics', 'Books', 'Clothing', 'Food', 'Sports'];
        
        for ($i = 1; $i <= $count; $i++) {
            $products[] = [
                'id' => $i,
                'name' => "Product {$i}",
                'price' => rand(10, 10000) / 10,
                'category' => $categories[array_rand($categories)],
                'stock' => rand(0, 1000),
            ];
        }
        return $products;
    }
    
    private function generateOrders(int $count, int $productCount): array {
        $orders = [];
        for ($i = 1; $i <= $count; $i++) {
            $orders[] = [
                'id' => $i,
                'product_id' => rand(1, $productCount),
                'quantity' => rand(1, 10),
                'date' => date('Y-m-d', strtotime("-" . rand(0, 365) . " days")),
                'status' => ['pending', 'completed', 'cancelled'][rand(0, 2)],
            ];
        }
        return $orders;
    }
    
    // SLOW: O(n*m) nested loop
    public function getProductSalesSlow(): array {
        $report = [];
        
        foreach ($this->products as $product) { // O(n)
            $totalSales = 0;
            $totalQuantity = 0;
            
            foreach ($this->orders as $order) { // O(m) ต่อ product = O(n*m)!
                if ($order['product_id'] === $product['id'] 
                    && $order['status'] === 'completed') {
                    $totalSales += $product['price'] * $order['quantity'];
                    $totalQuantity += $order['quantity'];
                }
            }
            
            $report[] = [
                'product_id' => $product['id'],
                'name' => $product['name'],
                'total_sales' => $totalSales,
                'total_quantity' => $totalQuantity,
            ];
        }
        
        return $report;
    }
    
    // SLOW: String concat ใน loop
    public function generateHTMLReportSlow(array $report): string {
        $html = '<table><tr><th>Product</th><th>Sales</th><th>Qty</th></tr>';
        
        foreach ($report as $item) {
            $html .= '<tr>';  // string grows every iteration!
            $html .= "<td>{$item['name']}</td>";
            $html .= "<td>" . number_format($item['total_sales'], 2) . "</td>";
            $html .= "<td>{$item['total_quantity']}</td>";
            $html .= '</tr>';
        }
        
        $html .= '</table>';
        return $html;
    }
    
    // SLOW: Sorting ทุกครั้งที่เรียก
    public function getTopProductsSlow(array $report, int $top = 10): array {
        usort($report, fn($a, $b) => $b['total_sales'] <=> $a['total_sales']); // sort ทั้ง array
        return array_slice($report, 0, $top);
    }
}

// ============================================================
// Version 2: โค้ดที่ Optimized
// ============================================================

class FastProductReport {
    private array $products;
    private array $orders;
    private ?array $productIndex = null; // lazy-built index
    private ?array $ordersByProduct = null; // pre-grouped
    
    public function __construct(
        private SlowProductReport $slow // reuse data
    ) {
        // reflection hack สำหรับ demo - ใน production ใช้ dependency injection
        $this->products = (fn() => $this->products)->bindTo($slow, $slow)();
        $this->orders = (fn() => $this->orders)->bindTo($slow, $slow)();
    }
    
    // FAST: O(m) pre-process + O(n) report = O(n+m)
    public function getProductSalesFast(): array {
        // Step 1: Group orders by product (O(m))
        $ordersByProduct = $this->getOrdersByProduct();
        
        // Step 2: Build product index (O(n))
        $productIndex = $this->getProductIndex();
        
        // Step 3: Calculate report (O(n))
        $report = [];
        foreach ($productIndex as $productId => $product) {
            $orders = $ordersByProduct[$productId] ?? [];
            
            $totalSales = 0;
            $totalQuantity = 0;
            
            foreach ($orders as $order) { // O(orders per product) ไม่ใช่ O(m)!
                $totalSales += $product['price'] * $order['quantity'];
                $totalQuantity += $order['quantity'];
            }
            
            $report[$productId] = [
                'product_id' => $productId,
                'name' => $product['name'],
                'total_sales' => $totalSales,
                'total_quantity' => $totalQuantity,
            ];
        }
        
        return array_values($report);
    }
    
    // Cache: build เฉพาะครั้งแรก
    private function getProductIndex(): array {
        if ($this->productIndex === null) {
            $this->productIndex = array_column($this->products, null, 'id');
        }
        return $this->productIndex;
    }
    
    private function getOrdersByProduct(): array {
        if ($this->ordersByProduct === null) {
            $this->ordersByProduct = [];
            
            foreach ($this->orders as $order) {
                if ($order['status'] === 'completed') {
                    $this->ordersByProduct[$order['product_id']][] = $order;
                }
            }
        }
        return $this->ordersByProduct;
    }
    
    // FAST: Array + implode
    public function generateHTMLReportFast(array $report): string {
        $rows = array_map(function($item) {
            return sprintf(
                '<tr><td>%s</td><td>%s</td><td>%d</td></tr>',
                htmlspecialchars($item['name']),
                number_format($item['total_sales'], 2),
                $item['total_quantity']
            );
        }, $report);
        
        return '<table><tr><th>Product</th><th>Sales</th><th>Qty</th></tr>'
            . implode('', $rows)
            . '</table>';
    }
    
    // FAST: Partial sort ด้วย heap (top-k problem)
    public function getTopProductsFast(array $report, int $top = 10): array {
        // สำหรับ top-k ไม่ต้อง sort ทั้ง array
        $heap = new \SplMaxHeap();
        
        foreach ($report as $item) {
            $heap->insert([$item['total_sales'], $item]);
        }
        
        $result = [];
        for ($i = 0; $i < $top && !$heap->isEmpty(); $i++) {
            [, $item] = $heap->extract();
            $result[] = $item;
        }
        
        return $result;
    }
    
    // FAST: Streaming/Generator approach
    public function streamReport(): \Generator {
        $ordersByProduct = $this->getOrdersByProduct();
        $productIndex = $this->getProductIndex();
        
        foreach ($productIndex as $productId => $product) {
            $orders = $ordersByProduct[$productId] ?? [];
            
            yield [
                'product_id' => $productId,
                'name' => $product['name'],
                'total_sales' => array_sum(array_map(
                    fn($o) => $product['price'] * $o['quantity'],
                    $orders
                )),
                'total_quantity' => array_sum(array_column($orders, 'quantity')),
            ];
        }
    }
}

// ============================================================
// Benchmark Comparison
// ============================================================

function runBenchmark(): void {
    echo "กำลังสร้างข้อมูลทดสอบ...\n";
    
    $slow = new SlowProductReport(1000, 10000); // 1000 products, 10000 orders
    $fast = new FastProductReport($slow);
    
    // Test 1: Report generation
    echo "\n=== Report Generation ===\n";
    
    $b = new Benchmark();
    $b->start();
    
    // Slow approach
    $start = microtime(true);
    $slowReport = $slow->getProductSalesSlow();
    $slowTime = (microtime(true) - $start) * 1000;
    echo "Slow (O(n*m)): " . round($slowTime, 2) . "ms\n";
    
    // Fast approach
    $start = microtime(true);
    $fastReport = $fast->getProductSalesFast();
    $fastTime = (microtime(true) - $start) * 1000;
    echo "Fast (O(n+m)): " . round($fastTime, 2) . "ms\n";
    
    echo "Speedup: " . round($slowTime / $fastTime, 1) . "x\n";
    
    // Test 2: HTML generation
    echo "\n=== HTML Generation (1000 rows) ===\n";
    
    $sample = array_slice($fastReport, 0, 1000);
    
    Benchmark::compare([
        'Slow (concat)'      => fn() => $slow->generateHTMLReportSlow($sample),
        'Fast (array+join)'  => fn() => $fast->generateHTMLReportFast($sample),
    ], 100);
    
    // Test 3: Top-k
    echo "\n=== Top-10 Products ===\n";
    
    Benchmark::compare([
        'Slow (full sort)'   => fn() => $slow->getTopProductsSlow($fastReport, 10),
        'Fast (heap)'        => fn() => $fast->getTopProductsFast($fastReport, 10),
    ], 100);
    
    // Test 4: Memory
    echo "\n=== Memory Usage ===\n";
    
    $before = memory_get_usage(true);
    $allReport = $fast->getProductSalesFast();
    $afterArray = memory_get_usage(true);
    echo "Array report: " . round(($afterArray - $before) / 1024, 2) . "KB\n";
    
    unset($allReport);
    gc_collect_cycles();
    
    $before = memory_get_usage(true);
    $count = 0;
    foreach ($fast->streamReport() as $item) {
        $count++;
    }
    $afterGenerator = memory_get_usage(true);
    echo "Generator stream: " . round(($afterGenerator - $before) / 1024, 2) . "KB (processed {$count} items)\n";
}

runBenchmark();
```

---

## Quiz

### คำถาม 1
OPcache ช่วยอะไรหลักๆ?
- A. ลด database queries
- B. Cache compiled PHP bytecode ทำให้ไม่ต้อง parse/compile ทุก request
- C. Cache HTTP responses
- D. ลด network latency

**เฉลย: B** - OPcache เก็บ compiled bytecode ใน shared memory ลด overhead ของ parsing

### คำถาม 2
ทำไม `in_array()` บน array ขนาดใหญ่จึงช้า?

**เฉลย:** `in_array()` ทำ sequential search = O(n) เพราะต้องตรวจสอบทุก element วิธีที่เร็วกว่าคือใช้ key lookup (`isset($arr[$key])`) ซึ่งเป็น O(1) เพราะ PHP array ใช้ hash table

### คำถาม 3
Generator ช่วยเรื่องอะไร?

**เฉลย:** Generator ช่วยลดการใช้ memory เพราะ yield ค่าทีละตัว แทนที่จะสร้าง array ขนาดใหญ่ทั้งหมดก่อน เหมาะกับการประมวลผล dataset ขนาดใหญ่ที่ไม่จำเป็นต้องโหลดทั้งหมดพร้อมกัน

### คำถาม 4
Big O ของการค้นหาใน PHP associative array (`$arr[$key]`) คืออะไร?

**เฉลย: O(1)** - PHP array ใช้ hash table โดยทั้ง read และ write เป็น O(1) average case (อาจ O(n) worst case เมื่อ hash collision สูง แต่ปกติ PHP จัดการได้ดี)

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- **Profiling** - Xdebug, Blackfire, custom benchmark functions
- **OPcache** - configuration, JIT, cache management
- **Memory Optimization** - Generators, chunking, references, lazy loading
- **String Performance** - str_contains vs strpos, array+implode vs concat
- **Algorithm Complexity** - Big O analysis, O(1)/O(n)/O(n²) examples
- **Optimization Workshop** - ปรับปรุง slow script ให้เร็วขึ้น 10x+

---

## ➡️ Part ถัดไป

[Part 025: PHP Modern Features (8.0-8.3)](./part-025-php-modern.md)
