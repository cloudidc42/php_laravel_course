# ⚙️ Part 006: PHP Functions — ฟังก์ชัน

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ประกาศและใช้งานฟังก์ชันได้อย่างถูกต้อง
- ใช้ parameters, default values, type hints ได้
- เขียน anonymous functions, closures, arrow functions
- ใช้ recursive functions และ first-class callables
- สร้าง utility functions library ที่ใช้งานได้จริง

---

## 📖 1. Function Declaration

### ฟังก์ชันพื้นฐาน

```php
<?php
// รูปแบบ: function functionName(parameters): returnType { body }
function greet(string $name): string
{
    return "สวัสดี, $name!";
}

echo greet("สมชาย"); // สวัสดี, สมชาย!
```

### Naming Convention

```php
<?php
// ✅ ใช้ camelCase สำหรับ function names
function getUserById(int $id): array { /* ... */ }
function calculateTotalPrice(array $items): float { /* ... */ }
function isValidEmail(string $email): bool { /* ... */ }

// ❌ หลีกเลี่ยง
function get_user_by_id() {} // snake_case (PHP อนุญาต แต่ไม่ใช่ PSR standard)
function GetUser() {}         // PascalCase (สำหรับ class methods)
function u() {}               // ชื่อสั้นเกิน ไม่สื่อความหมาย
```

### ฟังก์ชันไม่ส่งคืนค่า (void)

```php
<?php
function printSeparator(int $length = 40, string $char = '-'): void
{
    echo str_repeat($char, $length) . PHP_EOL;
}

printSeparator();        // ----------------------------------------
printSeparator(20);      // --------------------
printSeparator(10, '='); // ==========
```

---

## 📖 2. Parameters & Default Values

### Parameters พื้นฐาน

```php
<?php
// Required parameters
function add(int $a, int $b): int
{
    return $a + $b;
}

echo add(3, 5); // 8
// add(3);      // Error! ขาด argument
```

### Default Values

```php
<?php
// Default values — parameters ที่มี default ต้องอยู่หลัง required
function createUserProfile(
    string $name,
    string $role    = 'user',
    bool   $active  = true,
    int    $age     = 0
): array {
    return [
        'name'   => $name,
        'role'   => $role,
        'active' => $active,
        'age'    => $age,
    ];
}

print_r(createUserProfile("สมชาย"));
print_r(createUserProfile("สมหญิง", "admin"));
print_r(createUserProfile("สมศรี", "moderator", false, 25));
```

### Named Arguments (PHP 8.0+)

```php
<?php
function createButton(
    string $text,
    string $type    = 'button',
    string $color   = 'blue',
    bool   $disabled = false
): string {
    $disabledAttr = $disabled ? ' disabled' : '';
    return "<button type=\"$type\" class=\"btn btn-$color\"$disabledAttr>$text</button>";
}

// ใช้ Named Arguments — ไม่ต้องเรียงลำดับ
echo createButton(
    text: "บันทึก",
    color: "green",
    type: "submit"
);

echo createButton(
    text: "ปิด",
    disabled: true,
    color: "red"
);
```

### Variadic Functions (รับ arguments ไม่จำกัด)

```php
<?php
// รูปแบบ: function name(type ...$params)
function sum(float ...$numbers): float
{
    return array_sum($numbers);
}

echo sum(1, 2, 3);          // 6
echo sum(1.5, 2.5, 3.5);    // 7.5
echo sum(10, 20, 30, 40);   // 100

// Variadic กับ parameters ปกติ
function logMessage(string $level, string ...$messages): void
{
    $timestamp = date('Y-m-d H:i:s');
    foreach ($messages as $msg) {
        echo "[$timestamp][$level] $msg\n";
    }
}

logMessage('INFO', 'เริ่มต้นระบบ', 'โหลดการตั้งค่า', 'เชื่อมต่อฐานข้อมูล');
```

### Spread Operator

```php
<?php
function multiply(int $a, int $b, int $c): int
{
    return $a * $b * $c;
}

$args = [2, 3, 4];
echo multiply(...$args); // 24 — เหมือน multiply(2, 3, 4)

// Spread กับ array รวม
$first  = [1, 2, 3];
$second = [4, 5, 6];
$merged = [...$first, ...$second]; // [1, 2, 3, 4, 5, 6]
print_r($merged);
```

---

## 📖 3. Return Values & Type Hints

### Type Hints ที่รองรับ

```php
<?php
// Scalar types
function toInt(string $s): int       { return (int)$s; }
function toFloat(string $s): float   { return (float)$s; }
function toString(int $n): string    { return (string)$n; }
function isEven(int $n): bool        { return $n % 2 === 0; }

// Compound types
function getNames(): array           { return ['สมชาย', 'สมหญิง']; }

// Nullable types (PHP 7.1+)
function findUser(int $id): ?array
{
    $users = [1 => ['name' => 'สมชาย'], 2 => ['name' => 'สมหญิง']];
    return $users[$id] ?? null;
}

$user = findUser(1);
$missing = findUser(99);
var_dump($user);   // array(1) { ["name"]=> string(12) "สมชาย" }
var_dump($missing); // NULL
```

### Union Types (PHP 8.0+)

```php
<?php
// รับได้หลาย type
function processInput(int|string $value): string
{
    if (is_int($value)) {
        return "Integer: $value";
    }
    return "String: $value";
}

echo processInput(42);      // Integer: 42
echo processInput("hello"); // String: hello

// Return หลาย type
function divide(int $a, int $b): int|float|false
{
    if ($b === 0) return false;
    $result = $a / $b;
    return is_int($result) ? $result : (float)$result;
}

var_dump(divide(10, 2));  // int(5)
var_dump(divide(10, 3));  // float(3.333...)
var_dump(divide(10, 0));  // bool(false)
```

### Intersection Types (PHP 8.1+)

```php
<?php
interface Countable2
{
    public function count(): int;
}

interface Stringable2
{
    public function __toString(): string;
}

// ต้อง implement ทั้ง Countable2 และ Stringable2
function processCollection(Countable2&Stringable2 $collection): void
{
    echo "Count: " . $collection->count() . "\n";
    echo "String: " . $collection . "\n";
}
```

### Never Return Type (PHP 8.1+)

```php
<?php
// ฟังก์ชันที่ไม่ return เลย — throw exception เสมอ
function throwError(string $message): never
{
    throw new \RuntimeException($message);
}

// หรือ exit เสมอ
function exitWithCode(int $code): never
{
    exit($code);
}
```

### Multiple Return Values (ใช้ array/tuple)

```php
<?php
function divmod(int $a, int $b): array
{
    return [
        'quotient'  => intdiv($a, $b),
        'remainder' => $a % $b,
    ];
}

['quotient' => $q, 'remainder' => $r] = divmod(17, 5);
echo "17 ÷ 5 = $q เศษ $r\n"; // 17 ÷ 5 = 3 เศษ 2
```

---

## 📖 4. Variable Functions

```php
<?php
// เก็บชื่อ function ลงตัวแปร แล้วเรียกผ่านตัวแปร
$funcName = 'strtoupper';
echo $funcName("hello world"); // HELLO WORLD

// ตัวอย่างใช้งานจริง
function formatCurrency(float $amount): string
{
    return "฿" . number_format($amount, 2);
}

function formatPercent(float $value): string
{
    return number_format($value * 100, 1) . "%";
}

$formatter = 'formatCurrency';
echo $formatter(1500.50); // ฿1,500.50

$formatter = 'formatPercent';
echo $formatter(0.156);   // 15.6%

// ตรวจสอบก่อนเรียก
$action = 'nonExistentFunction';
if (function_exists($action)) {
    $action();
} else {
    echo "ไม่พบฟังก์ชัน: $action\n";
}
```

---

## 📖 5. Anonymous Functions (Closures)

### Closure พื้นฐาน

```php
<?php
// Anonymous function — ไม่มีชื่อ, เก็บในตัวแปร
$greet = function(string $name): string {
    return "สวัสดี, $name!";
};

echo $greet("สมชาย"); // สวัสดี, สมชาย!
```

### Closure กับ use — capture variables

```php
<?php
$prefix = "คุณ";
$suffix = " ครับ/ค่ะ";

$formatName = function(string $name) use ($prefix, $suffix): string {
    return $prefix . $name . $suffix;
};

echo $formatName("สมชาย"); // คุณสมชาย ครับ/ค่ะ

// use by reference — แก้ไขตัวแปรภายนอก
$counter = 0;
$increment = function(int $by = 1) use (&$counter): void {
    $counter += $by;
};

$increment();
$increment();
$increment(5);
echo $counter; // 7
```

### Closure ใน array functions

```php
<?php
$products = [
    ['name' => 'หมวก',    'price' => 299, 'category' => 'เสื้อผ้า'],
    ['name' => 'โทรศัพท์', 'price' => 15000, 'category' => 'อิเล็กทรอนิกส์'],
    ['name' => 'เสื้อ',   'price' => 599, 'category' => 'เสื้อผ้า'],
    ['name' => 'คอมพิวเตอร์', 'price' => 35000, 'category' => 'อิเล็กทรอนิกส์'],
    ['name' => 'กางเกง',  'price' => 799, 'category' => 'เสื้อผ้า'],
];

// กรองเฉพาะสินค้าราคาไม่เกิน 1000
$affordable = array_filter($products, function($p) {
    return $p['price'] <= 1000;
});

// แปลงเป็น name => price
$namePrice = array_map(function($p) {
    return "{$p['name']}: ฿{$p['price']}";
}, $affordable);

print_r(array_values($namePrice));

// Closure ที่รับ external variable
$maxPrice = 1000;
$filtered = array_filter($products, function($p) use ($maxPrice) {
    return $p['price'] <= $maxPrice;
});
echo "สินค้าราคาไม่เกิน ฿$maxPrice: " . count($filtered) . " รายการ\n";
```

### Closure as Callback

```php
<?php
function applyDiscount(array $products, \Closure $discountFn): array
{
    return array_map(function($product) use ($discountFn) {
        $product['discounted_price'] = $discountFn($product['price']);
        return $product;
    }, $products);
}

$products = [
    ['name' => 'หมวก', 'price' => 300],
    ['name' => 'เสื้อ', 'price' => 600],
];

// ส่ง closure เป็น argument
$discounted = applyDiscount($products, function(float $price): float {
    return $price * 0.9; // ลด 10%
});

foreach ($discounted as $p) {
    echo "{$p['name']}: ฿{$p['price']} → ฿{$p['discounted_price']}\n";
}
```

---

## 📖 6. Arrow Functions (PHP 7.4+)

Arrow functions คือ syntax สั้นของ closure ที่ capture scope โดยอัตโนมัติ

```php
<?php
// Closure ปกติ
$multiply = function(int $x) use ($multiplier): int {
    return $x * $multiplier;
};

// Arrow function — สั้นกว่า และ capture ตัวแปรอัตโนมัติ
$multiplier = 3;
$multiply = fn(int $x): int => $x * $multiplier;

echo $multiply(5);  // 15
echo $multiply(10); // 30
```

### Arrow function ใน array functions

```php
<?php
$numbers = range(1, 10);
$threshold = 5;

// array_filter กับ arrow function
$above = array_filter($numbers, fn($n) => $n > $threshold);
echo implode(', ', $above); // 6, 7, 8, 9, 10

// array_map
$doubled = array_map(fn($n) => $n * 2, $numbers);
echo implode(', ', $doubled); // 2, 4, 6, 8, 10, 12, 14, 16, 18, 20

// Chained
$result = array_map(
    fn($n) => $n ** 2,
    array_filter($numbers, fn($n) => $n % 2 === 0)
);
print_r(array_values($result));
// [4, 16, 36, 64, 100] — เลขคู่ยกกำลัง 2
```

### Arrow function ซ้อนกัน

```php
<?php
// Currying ด้วย arrow functions
$add = fn($x) => fn($y) => $x + $y;
$add5 = $add(5);

echo $add5(3);  // 8
echo $add5(10); // 15

// ตัวอย่างจริง: สร้าง validator factory
$minLength = fn($min) => fn($str) => strlen($str) >= $min;
$maxLength = fn($max) => fn($str) => strlen($str) <= $max;

$validatePassword = fn($pass) =>
    $minLength(8)($pass) && $maxLength(64)($pass);

var_dump($validatePassword("abc"));      // bool(false) — สั้นเกิน
var_dump($validatePassword("password")); // bool(true)
var_dump($validatePassword(str_repeat("x", 100))); // bool(false) — ยาวเกิน
```

---

## 📖 7. Recursive Functions

Recursion คือฟังก์ชันที่เรียกตัวเองซ้ำๆ

```php
<?php
// Factorial: n! = n × (n-1) × ... × 1
function factorial(int $n): int
{
    // Base case — หยุดเมื่อถึงเงื่อนไขนี้
    if ($n <= 1) {
        return 1;
    }

    // Recursive case — เรียกตัวเองด้วย n ที่เล็กลง
    return $n * factorial($n - 1);
}

echo factorial(5);  // 120  (5 × 4 × 3 × 2 × 1)
echo factorial(10); // 3628800
echo factorial(0);  // 1
```

### Fibonacci แบบ Recursive

```php
<?php
// ⚠️ Naive recursive Fibonacci — ช้ามาก O(2^n)
function fibonacci_naive(int $n): int
{
    if ($n <= 1) return $n;
    return fibonacci_naive($n - 1) + fibonacci_naive($n - 2);
}

// ✅ Memoized Fibonacci — เร็วขึ้นมาก O(n)
function fibonacci_memo(int $n, array &$memo = []): int
{
    if ($n <= 1) return $n;

    if (isset($memo[$n])) {
        return $memo[$n];
    }

    $memo[$n] = fibonacci_memo($n - 1, $memo) + fibonacci_memo($n - 2, $memo);
    return $memo[$n];
}

// ทดสอบ
echo fibonacci_memo(30);  // 832040
echo PHP_EOL;
echo fibonacci_memo(40);  // 102334155
```

### Tree Traversal (เดินผ่านโครงสร้าง tree)

```php
<?php
$fileSystem = [
    'name'     => 'root',
    'type'     => 'dir',
    'children' => [
        [
            'name'     => 'src',
            'type'     => 'dir',
            'children' => [
                ['name' => 'index.php',  'type' => 'file', 'size' => 1024],
                ['name' => 'config.php', 'type' => 'file', 'size' => 512],
            ],
        ],
        [
            'name'     => 'tests',
            'type'     => 'dir',
            'children' => [
                ['name' => 'UnitTest.php', 'type' => 'file', 'size' => 2048],
            ],
        ],
        ['name' => 'README.md', 'type' => 'file', 'size' => 768],
    ],
];

function printTree(array $node, string $indent = "", bool $isLast = true): void
{
    $connector = $isLast ? "└── " : "├── ";
    $icon = $node['type'] === 'dir' ? "📁" : "📄";

    if ($indent === "") {
        echo "$icon {$node['name']}\n";
    } else {
        echo $indent . $connector . "$icon {$node['name']}";
        if ($node['type'] === 'file') {
            echo " ({$node['size']} bytes)";
        }
        echo "\n";
    }

    if (isset($node['children'])) {
        $children = $node['children'];
        $count = count($children);
        foreach ($children as $i => $child) {
            $newIndent = $indent . ($isLast ? "    " : "│   ");
            $isLastChild = ($i === $count - 1);
            printTree($child, $newIndent, $isLastChild);
        }
    }
}

printTree($fileSystem);
```

### Tail Call Recursion

```php
<?php
// PHP ไม่ optimize tail calls แต่เข้าใจ concept ไว้
function sumTail(int $n, int $acc = 0): int
{
    if ($n <= 0) return $acc;
    return sumTail($n - 1, $acc + $n); // tail call
}

echo sumTail(100); // 5050
```

---

## 📖 8. First-class Callables (PHP 8.1+)

```php
<?php
// ก่อน PHP 8.1 — ต้องใช้ Closure::fromCallable() หรือ string
$strlen_old = Closure::fromCallable('strlen');
$strlen_old2 = fn($s) => strlen($s);

// PHP 8.1+ — syntax สั้นกว่า
$strlen = strlen(...);
$strtoupper = strtoupper(...);
$array_sum = array_sum(...);

echo $strlen("hello");        // 5
echo $strtoupper("hello");    // HELLO

// ใช้กับ array functions
$words = ['apple', 'banana', 'cherry'];
$lengths = array_map(strlen(...), $words);
print_r($lengths); // [5, 6, 6]

// ใช้กับ method ของ object
class Calculator
{
    public function square(int $n): int
    {
        return $n ** 2;
    }

    public static function cube(int $n): int
    {
        return $n ** 3;
    }
}

$calc = new Calculator();
$squareFn = $calc->square(...);
$cubeFn = Calculator::cube(...);

$numbers = [1, 2, 3, 4, 5];
$squares = array_map($squareFn, $numbers);
$cubes   = array_map($cubeFn, $numbers);

echo "Squares: " . implode(', ', $squares) . "\n"; // 1, 4, 9, 16, 25
echo "Cubes:   " . implode(', ', $cubes)   . "\n"; // 1, 8, 27, 64, 125
```

---

## 🔧 Workshop: Utility Functions Library

สร้าง utility library ที่ใช้งานได้จริงในโปรเจกต์

```php
<?php
/**
 * Utility Functions Library
 * Workshop: PHP Functions
 */

// ============================================================
// STRING UTILITIES
// ============================================================

/**
 * แปลง string เป็น slug (สำหรับ URL)
 */
function slugify(string $text, string $separator = '-'): string
{
    // แปลง unicode เป็น ASCII (สำหรับภาษาอังกฤษ)
    $text = transliterator_transliterate('Any-Latin; Latin-ASCII; Lower()', $text)
        ?? strtolower($text);

    // แทนที่อักขระพิเศษด้วย separator
    $text = preg_replace('/[^a-z0-9]+/', $separator, $text);

    // ลบ separator ที่หัวและท้าย
    return trim($text, $separator);
}

/**
 * ตัดข้อความยาวและเพิ่ม ...
 */
function truncate(string $text, int $maxLength, string $suffix = '...'): string
{
    if (mb_strlen($text) <= $maxLength) {
        return $text;
    }

    return mb_substr($text, 0, $maxLength - mb_strlen($suffix)) . $suffix;
}

/**
 * แปลง camelCase เป็น snake_case
 */
function camelToSnake(string $str): string
{
    return strtolower(preg_replace('/[A-Z]/', '_$0', lcfirst($str)));
}

/**
 * แปลง snake_case เป็น camelCase
 */
function snakeToCamel(string $str): string
{
    return lcfirst(str_replace('_', '', ucwords($str, '_')));
}

/**
 * สร้าง random string
 */
function randomString(int $length = 16, string $charset = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'): string
{
    $result = '';
    $charsetLength = strlen($charset);

    for ($i = 0; $i < $length; $i++) {
        $result .= $charset[random_int(0, $charsetLength - 1)];
    }

    return $result;
}

// ============================================================
// ARRAY UTILITIES
// ============================================================

/**
 * Flatten nested array (แบบกำหนด depth ได้)
 */
function flattenArray(array $array, int $depth = PHP_INT_MAX): array
{
    $result = [];

    foreach ($array as $item) {
        if (is_array($item) && $depth > 0) {
            $result = array_merge($result, flattenArray($item, $depth - 1));
        } else {
            $result[] = $item;
        }
    }

    return $result;
}

/**
 * จัดกลุ่ม array ตาม key
 */
function groupBy(array $items, string|callable $key): array
{
    $groups = [];

    foreach ($items as $item) {
        $groupKey = is_callable($key) ? $key($item) : $item[$key];
        $groups[$groupKey][] = $item;
    }

    return $groups;
}

/**
 * หา unique values จาก array of arrays
 */
function uniqueBy(array $items, string $key): array
{
    $seen = [];
    $result = [];

    foreach ($items as $item) {
        $value = $item[$key];
        if (!in_array($value, $seen, true)) {
            $seen[]   = $value;
            $result[] = $item;
        }
    }

    return $result;
}

/**
 * คำนวณ sum จาก array of arrays
 */
function sumBy(array $items, string $key): float
{
    return array_sum(array_column($items, $key));
}

/**
 * แบ่ง array เป็น chunks ตาม predicate
 */
function partition(array $items, callable $predicate): array
{
    $pass = [];
    $fail = [];

    foreach ($items as $key => $item) {
        if ($predicate($item)) {
            $pass[$key] = $item;
        } else {
            $fail[$key] = $item;
        }
    }

    return [$pass, $fail];
}

// ============================================================
// NUMBER UTILITIES
// ============================================================

/**
 * จำกัดค่าให้อยู่ในช่วงที่กำหนด
 */
function clamp(float $value, float $min, float $max): float
{
    return max($min, min($max, $value));
}

/**
 * แปลงเงิน format ไทย
 */
function formatThaiCurrency(float $amount, bool $showSymbol = true): string
{
    $formatted = number_format($amount, 2, '.', ',');
    return $showSymbol ? "฿{$formatted}" : $formatted;
}

/**
 * คำนวณเปอร์เซ็นต์
 */
function percentage(float $value, float $total, int $decimals = 1): float
{
    if ($total == 0) return 0.0;
    return round(($value / $total) * 100, $decimals);
}

// ============================================================
// DATE UTILITIES
// ============================================================

/**
 * แปลงวันที่เป็น relative time ("3 นาทีที่แล้ว")
 */
function timeAgo(\DateTimeInterface $date): string
{
    $now  = new \DateTime();
    $diff = $now->getTimestamp() - $date->getTimestamp();

    return match(true) {
        $diff < 60        => "เมื่อสักครู่",
        $diff < 3600      => intdiv($diff, 60) . " นาทีที่แล้ว",
        $diff < 86400     => intdiv($diff, 3600) . " ชั่วโมงที่แล้ว",
        $diff < 604800    => intdiv($diff, 86400) . " วันที่แล้ว",
        $diff < 2592000   => intdiv($diff, 604800) . " สัปดาห์ที่แล้ว",
        $diff < 31536000  => intdiv($diff, 2592000) . " เดือนที่แล้ว",
        default            => intdiv($diff, 31536000) . " ปีที่แล้ว",
    };
}

// ============================================================
// ทดสอบ Library
// ============================================================

echo "=== String Utilities ===\n";
echo slugify("Hello World! This is PHP") . "\n";     // hello-world-this-is-php
echo truncate("สวัสดีชาวโลก ยินดีต้อนรับ", 10) . "\n"; // สวัสดีชาวโ...
echo camelToSnake("getUserById") . "\n";             // get_user_by_id
echo snakeToCamel("get_user_by_id") . "\n";          // getUserById
echo randomString(12) . "\n";                         // สุ่ม 12 ตัวอักษร

echo "\n=== Array Utilities ===\n";
$nested = [1, [2, 3], [4, [5, 6]], 7];
print_r(flattenArray($nested));
// [1, 2, 3, 4, 5, 6, 7]

$orders = [
    ['id' => 1, 'status' => 'paid',    'amount' => 500],
    ['id' => 2, 'status' => 'pending', 'amount' => 300],
    ['id' => 3, 'status' => 'paid',    'amount' => 700],
    ['id' => 4, 'status' => 'pending', 'amount' => 200],
];

$grouped = groupBy($orders, 'status');
echo "Paid orders:    " . count($grouped['paid'])    . "\n"; // 2
echo "Pending orders: " . count($grouped['pending']) . "\n"; // 2

[$paidOrders, $pendingOrders] = partition($orders, fn($o) => $o['status'] === 'paid');
echo "Total paid:    ฿" . sumBy($paidOrders, 'amount') . "\n";   // ฿1200
echo "Total pending: ฿" . sumBy($pendingOrders, 'amount') . "\n"; // ฿500

echo "\n=== Number Utilities ===\n";
echo clamp(150, 0, 100) . "\n";          // 100
echo clamp(-10, 0, 100) . "\n";          // 0
echo clamp(50, 0, 100) . "\n";           // 50
echo formatThaiCurrency(12500.75) . "\n"; // ฿12,500.75
echo percentage(75, 300) . "%\n";         // 25.0%

echo "\n=== Date Utilities ===\n";
$fiveMinutesAgo  = new \DateTime('-5 minutes');
$twoHoursAgo     = new \DateTime('-2 hours');
$threeDaysAgo    = new \DateTime('-3 days');

echo timeAgo($fiveMinutesAgo)  . "\n"; // 5 นาทีที่แล้ว
echo timeAgo($twoHoursAgo)     . "\n"; // 2 ชั่วโมงที่แล้ว
echo timeAgo($threeDaysAgo)    . "\n"; // 3 วันที่แล้ว
```

---

## ❓ Quiz

### คำถามที่ 1
ข้อใดต่อไปนี้เป็น syntax ที่ถูกต้องสำหรับ Arrow function?

**ตัวเลือก:**
- a) `$fn = fn($x) { return $x * 2; }`
- b) `$fn = fn($x) => $x * 2;`
- c) `$fn = arrow($x) => $x * 2;`
- d) `$fn = function($x) => $x * 2;`

---

### คำถามที่ 2
ความแตกต่างระหว่าง closure ที่ใช้ `use ($var)` กับ `use (&$var)` คืออะไร?

**ตัวเลือก:**
- a) ไม่มีความต่าง
- b) `use ($var)` คัดลอกค่า ส่วน `use (&$var)` ส่ง reference — แก้ใน closure จะเปลี่ยนค่าภายนอกด้วย
- c) `use (&$var)` จะทำให้ closure ทำงานเร็วขึ้น
- d) `use ($var)` ใช้ได้เฉพาะกับ scalar types

---

### คำถามที่ 3
อะไรคือปัญหาของ recursive Fibonacci แบบ naive (ไม่มี memoization)?

**ตัวเลือก:**
- a) ไม่มีปัญหา
- b) Time complexity O(2^n) — คำนวณซ้ำค่าเดิมหลายรอบ
- c) ใช้ memory O(n²)
- d) PHP ไม่รองรับ recursion

---

### คำถามที่ 4
First-class callables (PHP 8.1+) คืออะไร?

**ตัวเลือก:**
- a) ฟังก์ชันที่มีประสิทธิภาพสูง
- b) Syntax `strlen(...)` ที่แปลง function/method เป็น Closure object ได้ง่ายขึ้น
- c) ฟังก์ชันที่เรียกก่อนใคว
- d) ฟังก์ชัน built-in ของ PHP

---

## ✅ เฉลย Quiz

**ข้อ 1: b) `$fn = fn($x) => $x * 2;`**
> Arrow function ใช้ `fn` keyword และ `=>` โดยไม่ต้องมี `return` keyword และ `{}`

**ข้อ 2: b) `use ($var)` คัดลอกค่า ส่วน `use (&$var)` ส่ง reference**
> - `use ($var)`: capture ค่า ณ ตอนที่สร้าง closure (copy by value)
> - `use (&$var)`: capture reference — แก้ค่าใน closure กระทบตัวแปรภายนอกด้วย

**ข้อ 3: b) Time complexity O(2^n)**
> naive fibonacci คำนวณ `fib(n-1)` และ `fib(n-2)` ซ้ำๆ
> `fib(40)` ต้องคำนวณประมาณ 300 ล้านครั้ง!
> แก้ด้วย memoization จะลดเหลือ O(n)

**ข้อ 4: b) Syntax `strlen(...)` ที่แปลง function เป็น Closure**
> `strlen(...)` คือ first-class callable syntax (PHP 8.1+)
> เทียบเท่ากับ `Closure::fromCallable('strlen')` แต่สั้นกว่า

---

## 🔗 สรุป

| Concept | รูปแบบ | ใช้เมื่อ |
|---------|--------|---------|
| Regular function | `function name() {}` | ทั่วไป, reusable |
| Anonymous/Closure | `function() {}` | callback, one-time use |
| Arrow function | `fn() => expr` | one-liner, auto-capture |
| Recursive | เรียกตัวเอง | tree traversal, divide & conquer |
| Variadic | `...$args` | รับ arguments ไม่จำกัด |
| First-class callable | `fn(...)` | convert ฟังก์ชันเป็น object |

---

## ➡️ Part ถัดไป

👉 **[Part 007: PHP Arrays — อาร์เรย์](./part-007-php-arrays.md)**

ใน Part หน้า เราจะเรียนรู้:
- Indexed, Associative, Multidimensional arrays
- Array functions ทั้งหมด
- Spread operator และ Array destructuring
- Workshop: Data processing pipeline
