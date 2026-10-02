# ⚡ Part 24: PHP Performance - การ Optimize ประสิทธิภาพ PHP

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- Profiling โปรแกรมด้วย Xdebug และ Blackfire ได้
- ตั้งค่า OPcache เพื่อเร่งความเร็วได้
- ใช้ Generators และเทคนิค Memory optimization ได้
- เข้าใจ Algorithm Complexity (Big O) และเลือกใช้ได้ถูกต้อง
- Optimize Database queries ได้
- เขียน Benchmark และวัดผลก่อน/หลัง optimize ได้

---

## 📌 1. Profiling ด้วย Xdebug

### 1.1 ติดตั้งและเปิดใช้ Xdebug Profiler

```ini
; php.ini หรือ xdebug.ini
[xdebug]
zend_extension=xdebug.so

; Profiling
xdebug.mode=profile
xdebug.start_with_request=trigger
xdebug.output_dir=/tmp/xdebug
xdebug.profiler_output_name=cachegrind.out.%p.%t

; Coverage (สำหรับ test coverage)
; xdebug.mode=coverage
```

### 1.2 Trigger Profiling

```php
<?php
// วิธีที่ 1: ผ่าน Query String
// URL: http://example.com/page.php?XDEBUG_PROFILE=1

// วิธีที่ 2: ผ่าน Cookie
// setcookie('XDEBUG_PROFILE', '1')

// วิธีที่ 3: Programmatic
xdebug_start_trace('/tmp/trace');
// ... โค้ดที่ต้องการ trace ...
xdebug_stop_trace();
```

### 1.3 อ่านผล Profiling ด้วย KCachegrind / Webgrind

```bash
# ดู output file
ls /tmp/xdebug/cachegrind.out.*

# เปิดด้วย KCachegrind (Linux)
kcachegrind /tmp/xdebug/cachegrind.out.12345.1234567890

# หรือใช้ qcachegrind (macOS)
qcachegrind /tmp/xdebug/cachegrind.out.12345.1234567890
```

### 1.4 Simple PHP Profiler (ไม่ต้องติดตั้งอะไรเพิ่ม)

```php
<?php
class SimpleProfiler {
    private static array $timers  = [];
    private static array $memory  = [];
    private static array $counts  = [];

    public static function start(string $label): void {
        self::$timers[$label]  = microtime(true);
        self::$memory[$label]  = memory_get_usage(true);
        self::$counts[$label]  = (self::$counts[$label] ?? 0) + 1;
    }

    public static function end(string $label): array {
        $elapsed = microtime(true) - (self::$timers[$label] ?? 0);
        $memDiff = memory_get_usage(true) - (self::$memory[$label] ?? 0);

        return [
            'label'   => $label,
            'time_ms' => round($elapsed * 1000, 3),
            'mem_kb'  => round($memDiff / 1024, 2),
            'calls'   => self::$counts[$label],
        ];
    }

    public static function measure(string $label, callable $fn): mixed {
        self::start($label);
        $result = $fn();
        $stats  = self::end($label);
        printf(
            "[%s] เวลา: %s ms | หน่วยความจำ: %s KB\n",
            $stats['label'], $stats['time_ms'], $stats['mem_kb']
        );
        return $result;
    }
}

// ใช้งาน
SimpleProfiler::measure('array_operation', function () {
    $data = range(1, 100000);
    return array_sum($data);
});

SimpleProfiler::measure('string_build', function () {
    $result = '';
    for ($i = 0; $i < 10000; $i++) {
        $result .= "item_{$i} ";
    }
    return strlen($result);
});
```

---

## 📌 2. OPcache

### 2.1 OPcache คืออะไร?

OPcache เก็บ "bytecode" ที่ compile แล้วของ PHP files ไว้ใน shared memory ทำให้ไม่ต้อง parse + compile ซ้ำทุก request

```
ไม่มี OPcache:
Request → อ่านไฟล์ PHP → Parse → Compile → Execute → Response

มี OPcache:
Request ครั้งแรก → อ่านไฟล์ → Parse → Compile → [บันทึก bytecode] → Execute → Response
Request ต่อมา  → [โหลด bytecode จาก cache] → Execute → Response (เร็วขึ้น ~3-10x!)
```

### 2.2 ตั้งค่า OPcache ที่แนะนำ

```ini
; php.ini
[opcache]
opcache.enable=1
opcache.enable_cli=1             ; เปิดสำหรับ CLI ด้วย

; Memory
opcache.memory_consumption=256   ; MB
opcache.interned_strings_buffer=16 ; MB สำหรับ string interning
opcache.max_accelerated_files=20000 ; จำนวน files สูงสุด

; Revalidation (Production: ปิด, Dev: เปิด)
opcache.revalidate_freq=0        ; 0 = ไม่ตรวจ (Production)
opcache.validate_timestamps=0    ; 0 = เชื่อ cache เสมอ (Production)

; Performance
opcache.save_comments=1          ; เก็บ comments (จำเป็นสำหรับ Annotations/Attributes)
opcache.fast_shutdown=1
opcache.enable_file_override=0

; JIT (PHP 8.0+)
opcache.jit_buffer_size=100M
opcache.jit=tracing              ; tracing = ดีที่สุดสำหรับ web
```

### 2.3 ตรวจสอบสถานะ OPcache

```php
<?php
function showOpcacheStatus(): void {
    if (!function_exists('opcache_get_status')) {
        echo "OPcache ไม่ได้ติดตั้ง\n";
        return;
    }

    $status = opcache_get_status();

    echo "=== OPcache Status ===\n";
    echo "Enabled: " . ($status['opcache_enabled'] ? 'Yes' : 'No') . "\n";
    echo "Cache Full: " . ($status['cache_full'] ? 'Yes' : 'No') . "\n";

    $mem = $status['memory_usage'];
    $used  = $mem['used_memory'] / 1024 / 1024;
    $free  = $mem['free_memory'] / 1024 / 1024;
    $total = $used + $free;

    printf("Memory: %.1fMB / %.1fMB (%.1f%%)\n", $used, $total, ($used / $total) * 100);

    $hits    = $status['opcache_statistics']['hits'];
    $misses  = $status['opcache_statistics']['misses'];
    $hitRate = $hits / max(1, $hits + $misses) * 100;
    printf("Hit Rate: %.2f%% (%d hits, %d misses)\n", $hitRate, $hits, $misses);
}

// Clear cache เมื่อ deploy ใหม่
function clearOpcache(): void {
    if (function_exists('opcache_reset')) {
        opcache_reset();
        echo "OPcache cleared!\n";
    }
}
```

---

## 📌 3. Memory Optimization

### 3.1 Generators ประหยัด Memory

```php
<?php
// แบบเดิม: โหลดทั้งหมดเข้า array → ใช้ memory มาก
function readLargeFileArray(string $filename): array {
    return file($filename, FILE_IGNORE_NEW_LINES); // โหลดทุก line เข้า array
}

// แบบ Generator: โหลดทีละ line → memory คงที่
function readLargeFileGenerator(string $filename): \Generator {
    $handle = fopen($filename, 'r');
    if ($handle === false) {
        throw new \RuntimeException("ไม่สามารถเปิดไฟล์: {$filename}");
    }

    try {
        while (($line = fgets($handle)) !== false) {
            yield trim($line); // ส่งออกทีละ line
        }
    } finally {
        fclose($handle);
    }
}

// ทดสอบ memory
echo "ก่อน: " . number_format(memory_get_usage()) . " bytes\n";

// แบบ Array (ใช้ memory มาก)
// $lines = readLargeFileArray('large_file.txt');

// แบบ Generator (memory คงที่ ~ไม่เปลี่ยน)
foreach (readLargeFileGenerator('/var/log/syslog') as $line) {
    if (str_contains($line, 'ERROR')) {
        echo $line . "\n";
        // หยุดหลัง 10 บรรทัด
        break;
    }
}

echo "หลัง: " . number_format(memory_get_usage()) . " bytes\n";
```

### 3.2 Generator Pipeline

```php
<?php
function csvLines(string $filename): \Generator {
    $handle = fopen($filename, 'r');
    $headers = null;

    while (($row = fgetcsv($handle)) !== false) {
        if ($headers === null) {
            $headers = $row;
            continue;
        }
        yield array_combine($headers, $row);
    }
    fclose($handle);
}

function filterActive(iterable $records): \Generator {
    foreach ($records as $record) {
        if (($record['status'] ?? '') === 'active') {
            yield $record;
        }
    }
}

function transformRecord(iterable $records): \Generator {
    foreach ($records as $record) {
        yield [
            'id'    => (int) $record['id'],
            'name'  => strtoupper($record['name']),
            'email' => strtolower($record['email']),
        ];
    }
}

// Pipeline: ไม่โหลดทั้งหมดเข้า memory
$pipeline = transformRecord(
    filterActive(
        csvLines('users.csv')
    )
);

foreach ($pipeline as $user) {
    echo "{$user['id']}: {$user['name']} ({$user['email']})\n";
}
```

### 3.3 Array vs SplFixedArray

```php
<?php
$size = 1_000_000;

// Array ปกติ
$before = memory_get_usage();
$array  = range(0, $size - 1);
$afterArray = memory_get_usage();
echo "PHP Array: " . number_format(($afterArray - $before) / 1024 / 1024, 2) . " MB\n";
unset($array);

// SplFixedArray (fixed size, integer index only)
$before         = memory_get_usage();
$fixed          = new \SplFixedArray($size);
for ($i = 0; $i < $size; $i++) {
    $fixed[$i] = $i;
}
$afterFixed = memory_get_usage();
echo "SplFixedArray: " . number_format(($afterFixed - $before) / 1024 / 1024, 2) . " MB\n";
// SplFixedArray ใช้ memory น้อยกว่าประมาณ 30-50%
```

### 3.4 Weak References

```php
<?php
// WeakReference: ไม่ป้องกัน GC จาก collecting object
class Cache {
    private array $storage = [];

    public function set(string $key, object $value): void {
        // WeakReference = ถ้า object ถูก GC ก็ไม่เป็นไร
        $this->storage[$key] = \WeakReference::create($value);
    }

    public function get(string $key): ?object {
        $ref = $this->storage[$key] ?? null;
        return $ref?->get(); // คืน null ถ้า object ถูก GC ไปแล้ว
    }
}

$cache = new Cache();
$obj   = new \stdClass();
$obj->data = "important data";

$cache->set('myobj', $obj);
echo $cache->get('myobj')->data . "\n"; // important data

unset($obj); // object ถูก destroy
echo var_export($cache->get('myobj'), true) . "\n"; // NULL
```

---

## 📌 4. String Performance

### 4.1 StringBuilder Pattern

```php
<?php
// แย่: string concatenation ใน loop สร้าง string ใหม่ทุกครั้ง
function buildStringBad(int $n): string {
    $result = '';
    for ($i = 0; $i < $n; $i++) {
        $result .= "item_{$i}, "; // O(n²) memory allocation!
    }
    return rtrim($result, ', ');
}

// ดี: ใช้ array + implode
function buildStringGood(int $n): string {
    $parts = [];
    for ($i = 0; $i < $n; $i++) {
        $parts[] = "item_{$i}";
    }
    return implode(', ', $parts); // O(n) allocation
}

// ดีที่สุดสำหรับ HTML ขนาดใหญ่: output buffering
function buildHtml(array $items): string {
    ob_start();
    foreach ($items as $item) {
        echo "<li>{$item}</li>\n";
    }
    return ob_get_clean();
}

// Benchmark
$n = 100_000;

$t1 = microtime(true);
buildStringBad($n);
echo "Bad: " . round((microtime(true) - $t1) * 1000, 2) . " ms\n";

$t2 = microtime(true);
buildStringGood($n);
echo "Good: " . round((microtime(true) - $t2) * 1000, 2) . " ms\n";
```

### 4.2 String Function Comparison

```php
<?php
$haystack = str_repeat('abcdefghij', 10000); // string ยาว

// ค้นหา prefix - strpos vs str_starts_with (PHP 8)
$t = microtime(true);
for ($i = 0; $i < 100000; $i++) {
    strpos($haystack, 'abc') === 0;
}
printf("strpos: %.3f ms\n", (microtime(true) - $t) * 1000);

$t = microtime(true);
for ($i = 0; $i < 100000; $i++) {
    str_starts_with($haystack, 'abc');
}
printf("str_starts_with: %.3f ms\n", (microtime(true) - $t) * 1000);
// str_starts_with เร็วกว่า เพราะไม่ต้องหาตำแหน่ง

// sprintf vs string interpolation
$name  = 'สมชาย';
$score = 9999;

$t = microtime(true);
for ($i = 0; $i < 1000000; $i++) {
    $s = "ชื่อ: {$name} คะแนน: {$score}";
}
printf("Interpolation: %.3f ms\n", (microtime(true) - $t) * 1000);

$t = microtime(true);
for ($i = 0; $i < 1000000; $i++) {
    $s = sprintf("ชื่อ: %s คะแนน: %d", $name, $score);
}
printf("sprintf: %.3f ms\n", (microtime(true) - $t) * 1000);
```

---

## 📌 5. Algorithm Complexity (Big O)

### 5.1 Big O ในตัวอย่าง PHP จริง

```php
<?php
// O(1) - Constant: เวลาคงที่ไม่ว่า input จะใหญ่แค่ไหน
function getFirstElement(array $arr): mixed {
    return $arr[0] ?? null; // เสมอ O(1)
}

// array_key_exists = O(1)
$map = array_fill_keys(range(1, 1000000), true);
$start = microtime(true);
$exists = array_key_exists(500000, $map);
printf("array_key_exists: %.6f ms\n", (microtime(true) - $start) * 1000);

// O(n) - Linear: เวลาเพิ่มตาม input
function linearSearch(array $arr, mixed $target): int {
    foreach ($arr as $i => $value) {
        if ($value === $target) return $i;
    }
    return -1;
}

// O(log n) - Logarithmic: เร็วมาก (Binary Search)
function binarySearch(array $sortedArr, int $target): int {
    $left  = 0;
    $right = count($sortedArr) - 1;

    while ($left <= $right) {
        $mid = intdiv($left + $right, 2);
        if ($sortedArr[$mid] === $target) return $mid;
        if ($sortedArr[$mid] < $target) $left = $mid + 1;
        else $right = $mid - 1;
    }
    return -1;
}

// O(n²) - Quadratic: ช้ามาก
function bubbleSort(array $arr): array {
    $n = count($arr);
    for ($i = 0; $i < $n - 1; $i++) {
        for ($j = 0; $j < $n - $i - 1; $j++) {
            if ($arr[$j] > $arr[$j + 1]) {
                [$arr[$j], $arr[$j + 1]] = [$arr[$j + 1], $arr[$j]];
            }
        }
    }
    return $arr;
}

// Benchmark: linear vs binary search
$n    = 100000;
$data = range(1, $n);
$target = 99999;

$t = microtime(true);
linearSearch($data, $target);
printf("Linear search O(n): %.3f ms\n", (microtime(true) - $t) * 1000);

$t = microtime(true);
binarySearch($data, $target);
printf("Binary search O(log n): %.6f ms\n", (microtime(true) - $t) * 1000);
```

### 5.2 PHP Built-in ที่ควรรู้ Complexity

```php
<?php
// O(1): array_push, array_pop, count (cached), array_key_exists, isset
// O(n): array_search, in_array, array_values, array_reverse
// O(n log n): sort, usort, array_unique
// O(n²): bubble sort, nested loops

// ตัวอย่าง: in_array O(n) vs array_key_exists O(1)
$haystack = array_fill(0, 1000000, false);
$haystack[999999] = true;

$lookup = array_flip($haystack); // แปลงเป็น key สำหรับ O(1) lookup

// แย่: O(n) ทุกครั้ง
$t = microtime(true);
for ($i = 0; $i < 1000; $i++) {
    in_array(true, $haystack);
}
printf("in_array (O(n)): %.3f ms\n", (microtime(true) - $t) * 1000);

// ดี: O(1) ด้วย hash map
$set = array_flip([1, 2, 3, 4, 5]); // [1=>0, 2=>1, ...]
$t   = microtime(true);
for ($i = 0; $i < 1000; $i++) {
    isset($set[3]); // O(1)
}
printf("isset hash (O(1)): %.3f ms\n", (microtime(true) - $t) * 1000);
```

---

## 📌 6. Database Query Optimization

### 6.1 N+1 Problem

```php
<?php
// แย่: N+1 queries
function getUsersWithPostsBad(PDO $pdo): array {
    $users = $pdo->query("SELECT * FROM users")->fetchAll(); // 1 query
    foreach ($users as &$user) {
        // N queries! (1 per user)
        $stmt = $pdo->prepare("SELECT * FROM posts WHERE user_id = ?");
        $stmt->execute([$user['id']]);
        $user['posts'] = $stmt->fetchAll();
    }
    return $users;
}

// ดี: 2 queries แทน N+1
function getUsersWithPostsGood(PDO $pdo): array {
    $users = $pdo->query("SELECT * FROM users")->fetchAll(\PDO::FETCH_ASSOC);

    if (empty($users)) return [];

    $userIds      = array_column($users, 'id');
    $placeholders = implode(',', array_fill(0, count($userIds), '?'));

    $stmt = $pdo->prepare("SELECT * FROM posts WHERE user_id IN ({$placeholders})");
    $stmt->execute($userIds);
    $posts = $stmt->fetchAll(\PDO::FETCH_ASSOC);

    // Group posts by user_id
    $postsByUser = [];
    foreach ($posts as $post) {
        $postsByUser[$post['user_id']][] = $post;
    }

    // Attach posts to users
    foreach ($users as &$user) {
        $user['posts'] = $postsByUser[$user['id']] ?? [];
    }

    return $users;
}
```

### 6.2 Query Cache & Prepared Statements

```php
<?php
class QueryCache {
    private array $cache = [];

    public function __construct(private \PDO $pdo) {}

    public function query(string $sql, array $params = [], int $ttl = 60): array {
        $key = md5($sql . serialize($params));

        if (isset($this->cache[$key]) && $this->cache[$key]['expires'] > time()) {
            echo "[CACHE HIT] {$sql}\n";
            return $this->cache[$key]['data'];
        }

        echo "[CACHE MISS] {$sql}\n";
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($params);
        $data = $stmt->fetchAll(\PDO::FETCH_ASSOC);

        $this->cache[$key] = [
            'data'    => $data,
            'expires' => time() + $ttl,
        ];

        return $data;
    }

    public function invalidate(string $pattern = ''): void {
        if ($pattern === '') {
            $this->cache = [];
        } else {
            foreach (array_keys($this->cache) as $key) {
                if (str_contains($key, $pattern)) {
                    unset($this->cache[$key]);
                }
            }
        }
    }
}
```

### 6.3 EXPLAIN และ Index Strategy

```php
<?php
// ตรวจสอบ query plan ก่อน
function explainQuery(PDO $pdo, string $sql, array $params = []): void {
    $stmt = $pdo->prepare("EXPLAIN FORMAT=JSON " . $sql);
    $stmt->execute($params);
    $result = $stmt->fetch(\PDO::FETCH_ASSOC);

    $plan = json_decode($result['EXPLAIN'] ?? $result[0], true);
    echo "Query Plan:\n";
    echo json_encode($plan, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE) . "\n";
}

// SQL ที่ควรมี Index
$slowQuery  = "SELECT * FROM orders WHERE customer_email = ? AND status = ? ORDER BY created_at DESC";
$fastQuery  = "SELECT id, total, created_at FROM orders WHERE customer_id = ? AND status = ? ORDER BY created_at DESC LIMIT 20";

// Migration สร้าง Index
$createIndexSQL = "
    CREATE INDEX idx_orders_customer_status_date
    ON orders (customer_id, status, created_at DESC);
";
```

---

## 🛠️ Workshop: ก่อน/หลัง Optimize พร้อม Benchmark

```php
<?php
declare(strict_types=1);

class Benchmark {
    private array $results = [];

    public function run(string $label, callable $fn, int $iterations = 1000): array {
        $memBefore = memory_get_usage(true);
        $times     = [];

        for ($i = 0; $i < $iterations; $i++) {
            $start   = hrtime(true); // nanoseconds precision
            $fn();
            $times[] = hrtime(true) - $start;
        }

        $memAfter = memory_get_usage(true);
        $avgNs    = array_sum($times) / count($times);
        $minNs    = min($times);
        $maxNs    = max($times);

        $result = [
            'label'      => $label,
            'iterations' => $iterations,
            'avg_ms'     => round($avgNs / 1_000_000, 4),
            'min_ms'     => round($minNs / 1_000_000, 4),
            'max_ms'     => round($maxNs / 1_000_000, 4),
            'mem_kb'     => round(($memAfter - $memBefore) / 1024, 2),
        ];

        $this->results[$label] = $result;
        return $result;
    }

    public function compare(string $label1, string $label2): void {
        $r1 = $this->results[$label1] ?? null;
        $r2 = $this->results[$label2] ?? null;

        if (!$r1 || !$r2) {
            echo "ไม่พบผล benchmark\n";
            return;
        }

        $ratio = $r1['avg_ms'] > 0 ? round($r2['avg_ms'] / $r1['avg_ms'], 2) : 0;
        echo "\n=== เปรียบเทียบ ===\n";
        printf("%-30s avg: %8.4f ms\n", $label1, $r1['avg_ms']);
        printf("%-30s avg: %8.4f ms\n", $label2, $r2['avg_ms']);

        if ($ratio > 1) {
            printf("%s เร็วกว่า %s ประมาณ %.1fx\n", $label1, $label2, $ratio);
        } else {
            printf("%s เร็วกว่า %s ประมาณ %.1fx\n", $label2, $label1, 1 / $ratio);
        }
    }

    public function printAll(): void {
        echo "\n╔══════════════════════════════════════════════════════════╗\n";
        echo "║                   Benchmark Results                     ║\n";
        echo "╠══════════════════╦════════╦════════╦════════╦══════════╣\n";
        echo "║ Label            ║ avg ms ║ min ms ║ max ms ║ mem KB   ║\n";
        echo "╠══════════════════╬════════╬════════╬════════╬══════════╣\n";

        foreach ($this->results as $r) {
            printf(
                "║ %-16s ║ %6.4f ║ %6.4f ║ %6.4f ║ %8.2f ║\n",
                substr($r['label'], 0, 16),
                $r['avg_ms'], $r['min_ms'], $r['max_ms'], $r['mem_kb']
            );
        }
        echo "╚══════════════════╩════════╩════════╩════════╩══════════╝\n";
    }
}

// ======= TEST 1: Array Building =======
$bench = new Benchmark();
$n     = 5000;

$bench->run("concat (แย่)", function () use ($n) {
    $str = '';
    for ($i = 0; $i < $n; $i++) $str .= "item_$i,";
    return $str;
});

$bench->run("implode (ดี)", function () use ($n) {
    $arr = [];
    for ($i = 0; $i < $n; $i++) $arr[] = "item_$i";
    return implode(',', $arr);
});

$bench->run("ob_start (ดีที่สุด)", function () use ($n) {
    ob_start();
    for ($i = 0; $i < $n; $i++) echo "item_$i,";
    return ob_get_clean();
});

$bench->compare("implode (ดี)", "concat (แย่)");

// ======= TEST 2: Array Search =======
$haystack  = range(1, 10000);
$searchVal = 9999;

$bench->run("in_array O(n)", function () use ($haystack, $searchVal) {
    return in_array($searchVal, $haystack);
});

$hashMap = array_flip($haystack);
$bench->run("isset O(1)", function () use ($hashMap, $searchVal) {
    return isset($hashMap[$searchVal]);
});

$bench->compare("isset O(1)", "in_array O(n)");

// ======= TEST 3: Loop Optimization =======
$data = range(1, 1000);

$bench->run("count() ใน loop (แย่)", function () use ($data) {
    $sum = 0;
    for ($i = 0; $i < count($data); $i++) { // count() เรียกทุก iteration
        $sum += $data[$i];
    }
    return $sum;
});

$bench->run("count() นอก loop (ดี)", function () use ($data) {
    $sum = 0;
    $n   = count($data); // เรียกครั้งเดียว
    for ($i = 0; $i < $n; $i++) {
        $sum += $data[$i];
    }
    return $sum;
});

$bench->run("array_sum builtin", function () use ($data) {
    return array_sum($data); // C implementation, เร็วที่สุด
});

// ======= TEST 4: Memory - Generator vs Array =======
$genBench = new Benchmark();

$genBench->run("array 10k items", function () {
    $data = [];
    for ($i = 0; $i < 10000; $i++) {
        $data[] = ['id' => $i, 'value' => rand()];
    }
    $sum = 0;
    foreach ($data as $item) $sum += $item['value'];
    return $sum;
}, 100);

$genBench->run("generator 10k items", function () {
    $gen = (function () {
        for ($i = 0; $i < 10000; $i++) {
            yield ['id' => $i, 'value' => rand()];
        }
    })();
    $sum = 0;
    foreach ($gen as $item) $sum += $item['value'];
    return $sum;
}, 100);

echo "\n=== Array vs Generator ===\n";
$genBench->compare("generator 10k items", "array 10k items");

$bench->printAll();
$genBench->printAll();
```

---

## ❓ Quiz

### ข้อที่ 1
Generator ช่วย optimize อะไรเป็นหลัก?

- A. CPU Speed
- B. Memory Usage โดยไม่โหลดทั้งหมดเข้า memory พร้อมกัน
- C. Database query speed
- D. Network latency

**เฉลย: B**
Generator ใช้ `yield` แทน `return` เพื่อส่งค่าออกทีละรายการโดยไม่ต้องสร้าง array ขนาดใหญ่ในหน่วยความจำ เหมาะกับการประมวลผลไฟล์ขนาดใหญ่หรือ dataset จำนวนมาก

---

### ข้อที่ 2
OPcache ช่วยประสิทธิภาพอย่างไร?

- A. ลด memory การใช้ตัวแปร
- B. Cache ผลลัพธ์ SQL queries
- C. เก็บ compiled bytecode ใน memory ลด parse + compile overhead
- D. Compress HTTP response

**เฉลย: C**
OPcache เก็บ bytecode ที่ PHP compile แล้วไว้ใน shared memory ทำให้ request ต่อๆ มาข้ามขั้นตอน parse และ compile ไปได้ ลด latency ได้ 2-10x

---

### ข้อที่ 3
`in_array()` มี Time Complexity เท่าไร และจะ optimize ได้อย่างไร?

- A. O(1) และ optimize ด้วย sort ก่อน
- B. O(n) และ optimize โดยแปลงเป็น hash map แล้วใช้ `isset()`
- C. O(log n) อยู่แล้ว ไม่ต้อง optimize
- D. O(n²) และใช้ binary search แทน

**เฉลย: B**
`in_array()` ต้องวน loop ทุก element จึง O(n). การใช้ `array_flip()` แล้ว `isset()` หรือ `array_key_exists()` ทำให้ lookup เป็น O(1) เพราะ PHP array ใช้ hash table ภายใน

---

### ข้อที่ 4
N+1 Problem คืออะไร?

- A. Query ที่มี JOIN มากกว่า N ตาราง
- B. การ query 1 ครั้งเพื่อดึง N records แล้ว query อีก N ครั้งสำหรับแต่ละ record
- C. Query ที่ไม่มี INDEX ทำให้ Full Table Scan
- D. Query ที่มี subquery ซ้อนกัน

**เฉลย: B**
N+1 Problem เกิดเมื่อดึง N users แล้ว loop query posts ของแต่ละ user รวมเป็น N+1 queries แก้ได้ด้วย JOIN หรือ Eager Loading (IN clause) เพื่อลดเหลือ 1-2 queries

---

### ข้อที่ 5
`SplFixedArray` ต่างจาก PHP array ปกติอย่างไร?

- A. SplFixedArray เร็วกว่าสำหรับ string keys
- B. SplFixedArray มีขนาดคงที่และรองรับเฉพาะ integer index ทำให้ประหยัด memory กว่า
- C. SplFixedArray รองรับ multidimensional array ดีกว่า
- D. SplFixedArray automatic resize ได้

**เฉลย: B**
`SplFixedArray` ใช้ memory น้อยกว่า PHP array ปกติ 30-50% เพราะไม่มี overhead ของ hash table รองรับเฉพาะ integer index ขนาดคงที่ เหมาะกับ numerical data ขนาดใหญ่

---

## 🔗 ไปต่อ

➡️ **[Part 25: PHP Modern Features - PHP 8.0 ถึง 8.3](part-025-php-modern-features.md)**

เรียนรู้ฟีเจอร์ใหม่ๆ ของ PHP 8.x ที่ทำให้โค้ดสั้นลง อ่านง่ายขึ้น และปลอดภัยมากขึ้น
