# 📦 Part 007: PHP Arrays — อาร์เรย์

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้างและใช้งาน Indexed, Associative, Multidimensional arrays ได้
- ใช้ array functions สำคัญๆ ได้ทั้งหมด
- ใช้ Spread operator และ Array destructuring
- สร้าง data processing pipeline ได้

---

## 📖 1. Indexed Arrays

```php
<?php
// สร้าง array แบบต่างๆ
$fruits = ['แอปเปิ้ล', 'กล้วย', 'ส้ม', 'มะม่วง'];
$numbers = [1, 2, 3, 4, 5];
$mixed   = [1, "hello", true, null, 3.14];

// เข้าถึงด้วย index (เริ่มที่ 0)
echo $fruits[0]; // แอปเปิ้ล
echo $fruits[3]; // มะม่วง

// แก้ไขค่า
$fruits[1] = 'สับปะรด';
echo $fruits[1]; // สับปะรด

// เพิ่มต่อท้าย
$fruits[] = 'ลิ้นจี่';
$fruits[] = 'ทุเรียน';

// นับจำนวน
echo count($fruits); // 6
```

### การสร้าง array ด้วย range()

```php
<?php
$numbers = range(1, 10);
print_r($numbers); // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

$evens = range(0, 20, 2);
print_r($evens); // [0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

$letters = range('a', 'z');
echo implode('', $letters); // abcdefghijklmnopqrstuvwxyz
```

---

## 📖 2. Associative Arrays

```php
<?php
// Key => Value pairs
$person = [
    'name'     => 'สมชาย ใจดี',
    'age'      => 30,
    'email'    => 'somchai@example.com',
    'city'     => 'กรุงเทพมหานคร',
    'active'   => true,
];

// เข้าถึงด้วย key
echo $person['name'];  // สมชาย ใจดี
echo $person['age'];   // 30

// ตรวจสอบว่า key มีอยู่หรือไม่
if (isset($person['email'])) {
    echo "มี email: " . $person['email'];
}

// array_key_exists ต่างจาก isset — คืน true แม้ value เป็น null
$data = ['key' => null];
var_dump(isset($data['key']));           // bool(false) — null ถือว่าไม่ set!
var_dump(array_key_exists('key', $data)); // bool(true)  — key มีอยู่จริง

// เพิ่ม/แก้ไข key
$person['phone']  = '081-234-5678';
$person['age']    = 31;

// ลบ key
unset($person['active']);

print_r($person);
```

### Nested Associative Array

```php
<?php
$config = [
    'database' => [
        'host'     => 'localhost',
        'port'     => 3306,
        'name'     => 'myapp',
        'username' => 'root',
        'password' => 'secret',
    ],
    'cache' => [
        'driver'  => 'redis',
        'host'    => '127.0.0.1',
        'port'    => 6379,
        'ttl'     => 3600,
    ],
    'mail' => [
        'driver' => 'smtp',
        'host'   => 'smtp.gmail.com',
        'port'   => 587,
    ],
];

// เข้าถึงแบบ nested
echo $config['database']['host'];  // localhost
echo $config['cache']['driver'];   // redis

// ใช้ null coalescing สำหรับค่าที่อาจไม่มี
$timezone = $config['app']['timezone'] ?? 'Asia/Bangkok';
echo $timezone; // Asia/Bangkok
```

---

## 📖 3. Multidimensional Arrays

```php
<?php
// 2D Array — เหมาะสำหรับ table data
$students = [
    ['id' => 1, 'name' => 'สมชาย',   'score' => 85, 'grade' => 'B'],
    ['id' => 2, 'name' => 'สมหญิง',  'score' => 92, 'grade' => 'A'],
    ['id' => 3, 'name' => 'สมศรี',   'score' => 67, 'grade' => 'C+'],
    ['id' => 4, 'name' => 'สมปอง',   'score' => 78, 'grade' => 'B+'],
];

// เข้าถึง element
echo $students[0]['name'];  // สมชาย
echo $students[1]['score']; // 92

// วนเข้าถึงทั้งหมด
foreach ($students as $student) {
    echo "{$student['name']}: {$student['score']} ({$student['grade']})\n";
}

// 3D Array — เช่น data แบ่งตาม semester/subject
$grades = [
    'semester1' => [
        'math'    => [85, 90, 78, 92],
        'english' => [70, 75, 80, 68],
    ],
    'semester2' => [
        'math'    => [88, 92, 95, 85],
        'english' => [75, 80, 78, 82],
    ],
];

foreach ($grades as $semester => $subjects) {
    echo "\n$semester:\n";
    foreach ($subjects as $subject => $scores) {
        $avg = array_sum($scores) / count($scores);
        echo "  $subject: " . implode(', ', $scores) . " (avg: " . round($avg, 1) . ")\n";
    }
}
```

---

## 📖 4. Array Functions ทั้งหมด

### Sorting Functions

```php
<?php
// sort() — เรียงจากน้อยไปมาก (index reset)
$numbers = [3, 1, 4, 1, 5, 9, 2, 6];
sort($numbers);
print_r($numbers); // [1, 1, 2, 3, 4, 5, 6, 9]

// rsort() — เรียงจากมากไปน้อย
rsort($numbers);
print_r($numbers); // [9, 6, 5, 4, 3, 2, 1, 1]

// asort() — เรียงค่า แต่คง key ไว้
$scores = ['สมชาย' => 85, 'สมหญิง' => 92, 'สมศรี' => 78];
asort($scores);
print_r($scores); // เรียงตามคะแนน แต่ชื่อยังเป็น key

// arsort() — เรียงค่าจากมากไปน้อย คง key
arsort($scores);

// ksort() — เรียงตาม key
$data = ['banana' => 2, 'apple' => 5, 'cherry' => 1];
ksort($data);
print_r($data); // ['apple' => 5, 'banana' => 2, 'cherry' => 1]

// krsort() — เรียงตาม key จากมากไปน้อย
krsort($data);

// usort() — เรียงด้วย custom comparison function
$people = [
    ['name' => 'สมชาย', 'age' => 30],
    ['name' => 'สมหญิง', 'age' => 25],
    ['name' => 'สมศรี', 'age' => 35],
];

usort($people, fn($a, $b) => $a['age'] <=> $b['age']);
foreach ($people as $p) {
    echo "{$p['name']}: {$p['age']}\n";
}
// สมหญิง: 25
// สมชาย: 30
// สมศรี: 35

// uasort() — usort แต่คง key
// uksort() — เรียง key ด้วย custom function
```

### Searching Functions

```php
<?php
$fruits = ['apple', 'banana', 'cherry', 'date', 'elderberry'];

// in_array() — ค้นหาค่า
var_dump(in_array('banana', $fruits));      // bool(true)
var_dump(in_array('mango', $fruits));       // bool(false)
var_dump(in_array('banana', $fruits, true)); // strict mode

// array_search() — ค้นหาค่า ส่งคืน key
$key = array_search('cherry', $fruits);
var_dump($key); // int(2)

$notFound = array_search('mango', $fruits);
var_dump($notFound); // bool(false)

// array_key_exists()
$person = ['name' => 'สมชาย', 'age' => null];
var_dump(array_key_exists('age', $person)); // bool(true)
var_dump(isset($person['age']));            // bool(false) — null!
```

### Filtering Functions

```php
<?php
$numbers = range(1, 20);

// array_filter() — กรองด้วย callback
$evens = array_filter($numbers, fn($n) => $n % 2 === 0);
print_r(array_values($evens)); // [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

// กรองค่า falsy (ไม่ระบุ callback)
$mixed = [0, 1, '', 'hello', null, false, true, [], [1,2]];
$truthy = array_filter($mixed);
print_r(array_values($truthy)); // [1, 'hello', true, [1,2]]

// array_unique() — เอาเฉพาะค่าไม่ซ้ำ
$tags = ['php', 'laravel', 'php', 'mysql', 'laravel', 'redis'];
$uniqueTags = array_unique($tags);
print_r(array_values($uniqueTags)); // ['php', 'laravel', 'mysql', 'redis']
```

### Transform Functions

```php
<?php
$numbers = range(1, 5);

// array_map() — แปลงทุก element
$squared = array_map(fn($n) => $n ** 2, $numbers);
print_r($squared); // [1, 4, 9, 16, 25]

// array_map กับหลาย arrays
$a = [1, 2, 3];
$b = [10, 20, 30];
$sums = array_map(fn($x, $y) => $x + $y, $a, $b);
print_r($sums); // [11, 22, 33]

// array_reduce() — ลดเป็นค่าเดียว
$sum = array_reduce($numbers, fn($carry, $item) => $carry + $item, 0);
echo $sum; // 15

$product = array_reduce($numbers, fn($carry, $item) => $carry * $item, 1);
echo $product; // 120

// array_walk() — แก้ไข array ใน place
$prices = ['apple' => 100, 'banana' => 50, 'cherry' => 150];
array_walk($prices, function(&$price, $fruit) {
    $price = "$fruit: ฿$price";
});
print_r($prices);
```

### Slice & Combine Functions

```php
<?php
$arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// array_slice() — ตัด array
$slice = array_slice($arr, 2, 5); // เริ่มที่ index 2, ยาว 5
print_r($slice); // [3, 4, 5, 6, 7]

$last3 = array_slice($arr, -3); // 3 ตัวสุดท้าย
print_r($last3); // [8, 9, 10]

// array_splice() — ตัดและ/หรือแทรก
$colors = ['red', 'green', 'blue', 'yellow'];
$removed = array_splice($colors, 1, 2, ['purple', 'orange', 'pink']);
print_r($colors);   // ['red', 'purple', 'orange', 'pink', 'yellow']
print_r($removed);  // ['green', 'blue']

// array_chunk() — แบ่งเป็น chunk
$items = range(1, 10);
$chunks = array_chunk($items, 3);
print_r($chunks); // [[1,2,3], [4,5,6], [7,8,9], [10]]

// array_combine() — รวม key array กับ value array
$keys   = ['name', 'age', 'city'];
$values = ['สมชาย', 30, 'กรุงเทพ'];
$person = array_combine($keys, $values);
print_r($person);
// ['name' => 'สมชาย', 'age' => 30, 'city' => 'กรุงเทพ']

// array_merge() — รวม arrays
$arr1 = ['a' => 1, 'b' => 2];
$arr2 = ['b' => 3, 'c' => 4]; // 'b' จะถูก override
$merged = array_merge($arr1, $arr2);
print_r($merged); // ['a' => 1, 'b' => 3, 'c' => 4]

// array_merge กับ indexed arrays
$merged2 = array_merge([1, 2, 3], [4, 5, 6]);
print_r($merged2); // [1, 2, 3, 4, 5, 6]
```

### Set Operations

```php
<?php
$a = [1, 2, 3, 4, 5];
$b = [3, 4, 5, 6, 7];

// array_intersect() — ค่าที่มีใน a และ b
$intersection = array_intersect($a, $b);
print_r(array_values($intersection)); // [3, 4, 5]

// array_diff() — ค่าที่มีใน a แต่ไม่มีใน b
$difference = array_diff($a, $b);
print_r(array_values($difference)); // [1, 2]

// array_intersect_key() / array_diff_key() — ใช้ key แทนค่า
$user1 = ['name' => 'สมชาย', 'age' => 30, 'email' => 'a@b.com'];
$user2 = ['name' => 'สมหญิง', 'phone' => '081', 'email' => 'c@d.com'];

$commonKeys = array_intersect_key($user1, $user2);
print_r($commonKeys); // ['name' => 'สมชาย', 'email' => 'a@b.com']
```

### Stack & Queue Functions

```php
<?php
// Stack (LIFO) — array_push / array_pop
$stack = [];
array_push($stack, 'first', 'second', 'third');
// หรือ
$stack[] = 'fourth';

$last = array_pop($stack); // ดึงจากด้านหลัง
echo $last; // fourth

// Queue (FIFO) — array_push / array_shift
$queue = ['task1', 'task2', 'task3'];
$first = array_shift($queue); // ดึงจากด้านหน้า
echo $first; // task1
print_r($queue); // ['task2', 'task3']

// array_unshift — เพิ่มที่ด้านหน้า
array_unshift($queue, 'urgent_task');
print_r($queue); // ['urgent_task', 'task2', 'task3']
```

### Column & Flip Functions

```php
<?php
$records = [
    ['id' => 1, 'name' => 'สมชาย',  'dept' => 'IT'],
    ['id' => 2, 'name' => 'สมหญิง', 'dept' => 'HR'],
    ['id' => 3, 'name' => 'สมศรี',  'dept' => 'IT'],
];

// array_column() — ดึงคอลัมน์ออกมา
$names = array_column($records, 'name');
print_r($names); // ['สมชาย', 'สมหญิง', 'สมศรี']

// array_column กับ index key
$byId = array_column($records, null, 'id');
print_r($byId); // ['1' => [...], '2' => [...], '3' => [...]]

// array_flip() — สลับ key กับ value
$fruits = ['apple', 'banana', 'cherry'];
$flipped = array_flip($fruits);
print_r($flipped);
// ['apple' => 0, 'banana' => 1, 'cherry' => 2]

// ประโยชน์: lookup O(1) แทน in_array O(n)
$allowedUsers = ['admin', 'editor', 'viewer'];
$lookup = array_flip($allowedUsers);
$role = 'editor';
if (isset($lookup[$role])) {
    echo "มีสิทธิ์เข้าถึง\n";
}
```

### Math Functions

```php
<?php
$numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5];

echo array_sum($numbers) . "\n";     // 44 — ผลรวม
echo array_product([1,2,3,4,5]) . "\n"; // 120 — ผลคูณ
echo min($numbers) . "\n";           // 1 — ค่าต่ำสุด
echo max($numbers) . "\n";           // 9 — ค่าสูงสุด

// array_count_values() — นับจำนวนแต่ละค่า
$counted = array_count_values($numbers);
arsort($counted);
foreach ($counted as $value => $count) {
    echo "$value: $count ครั้ง\n";
}
```

---

## 📖 5. Spread Operator

```php
<?php
// Spread ใน function call
function sum3(int $a, int $b, int $c): int
{
    return $a + $b + $c;
}

$args = [1, 2, 3];
echo sum3(...$args); // 6

// Spread ใน array creation
$first  = [1, 2, 3];
$second = [4, 5, 6];
$all    = [...$first, ...$second, 7, 8];
print_r($all); // [1, 2, 3, 4, 5, 6, 7, 8]

// Spread กับ associative (PHP 8.1+)
$defaults = ['color' => 'red', 'size' => 'M', 'stock' => 100];
$custom   = ['color' => 'blue', 'price' => 299];
$merged   = [...$defaults, ...$custom];
print_r($merged);
// ['color' => 'blue', 'size' => 'M', 'stock' => 100, 'price' => 299]

// Clone array ด้วย spread
$original = [1, 2, 3, 4, 5];
$copy     = [...$original]; // shallow copy
$copy[] = 6;
print_r($original); // [1, 2, 3, 4, 5] — ไม่เปลี่ยน
print_r($copy);     // [1, 2, 3, 4, 5, 6]
```

---

## 📖 6. Array Destructuring

```php
<?php
// รูปแบบพื้นฐาน
[$a, $b, $c] = [1, 2, 3];
echo "$a, $b, $c"; // 1, 2, 3

// ข้ามบาง index
[, $second, , $fourth] = [10, 20, 30, 40];
echo "$second, $fourth"; // 20, 40

// Destructuring กับ associative array
['name' => $name, 'age' => $age] = ['name' => 'สมชาย', 'age' => 30, 'city' => 'กรุงเทพ'];
echo "$name อายุ $age ปี"; // สมชาย อายุ 30 ปี

// ใช้กับ foreach
$points = [[1, 2], [3, 4], [5, 6]];
foreach ($points as [$x, $y]) {
    echo "($x, $y) ";
}
// (1, 2) (3, 4) (5, 6)

// list() — รูปแบบเก่า (ก่อน PHP 7.1)
list($first, $second) = ['A', 'B'];
echo "$first, $second"; // A, B

// Swap variables ด้วย array destructuring
$x = 1;
$y = 2;
[$x, $y] = [$y, $x];
echo "$x, $y"; // 2, 1
```

---

## 🔧 Workshop: Data Processing Pipeline

สร้าง pipeline สำหรับประมวลผลข้อมูลสินค้า

```php
<?php
/**
 * Data Processing Pipeline
 * Workshop: PHP Arrays
 */

// ข้อมูล raw สินค้า
$rawProducts = [
    ['id' => 1,  'name' => 'iPhone 15',          'category' => 'Electronics', 'price' => 32900, 'stock' => 50,  'rating' => 4.8, 'sold' => 1200],
    ['id' => 2,  'name' => 'Samsung Galaxy S24',  'category' => 'Electronics', 'price' => 28900, 'stock' => 30,  'rating' => 4.6, 'sold' => 980],
    ['id' => 3,  'name' => 'MacBook Pro M3',       'category' => 'Computers',   'price' => 79900, 'stock' => 10,  'rating' => 4.9, 'sold' => 350],
    ['id' => 4,  'name' => 'AirPods Pro',          'category' => 'Electronics', 'price' => 8900,  'stock' => 100, 'rating' => 4.7, 'sold' => 2500],
    ['id' => 5,  'name' => 'Dell XPS 15',          'category' => 'Computers',   'price' => 55900, 'stock' => 15,  'rating' => 4.5, 'sold' => 420],
    ['id' => 6,  'name' => 'Logitech MX Master 3', 'category' => 'Accessories', 'price' => 3200,  'stock' => 80,  'rating' => 4.8, 'sold' => 3200],
    ['id' => 7,  'name' => 'Mechanical Keyboard',  'category' => 'Accessories', 'price' => 4500,  'stock' => 0,   'rating' => 4.6, 'sold' => 1800],
    ['id' => 8,  'name' => 'iPad Pro 12.9"',       'category' => 'Electronics', 'price' => 41900, 'stock' => 25,  'rating' => 4.7, 'sold' => 680],
    ['id' => 9,  'name' => 'USB-C Hub',            'category' => 'Accessories', 'price' => 1290,  'stock' => 200, 'rating' => 4.2, 'sold' => 5600],
    ['id' => 10, 'name' => 'Monitor 4K 27"',       'category' => 'Computers',   'price' => 18900, 'stock' => 20,  'rating' => 4.5, 'sold' => 890],
];

// ============================================================
// PIPELINE STEP 1: Filter — กรองสินค้าที่มีใน stock
// ============================================================

$inStock = array_filter($rawProducts, fn($p) => $p['stock'] > 0);
echo "สินค้าที่มีใน stock: " . count($inStock) . " รายการ\n";

// ============================================================
// PIPELINE STEP 2: Enrich — เพิ่มข้อมูลที่คำนวณได้
// ============================================================

$enriched = array_map(function($product) {
    $product['revenue']         = $product['price'] * $product['sold'];
    $product['stock_value']     = $product['price'] * $product['stock'];
    $product['popularity_score'] = ($product['rating'] * 20) + log($product['sold'] + 1, 10) * 10;

    $product['stock_status'] = match(true) {
        $product['stock'] === 0  => 'out_of_stock',
        $product['stock'] <= 10  => 'low_stock',
        $product['stock'] <= 50  => 'normal',
        default                   => 'high_stock',
    };

    return $product;
}, $inStock);

// ============================================================
// PIPELINE STEP 3: Sort — เรียงตาม revenue
// ============================================================

usort($enriched, fn($a, $b) => $b['revenue'] <=> $a['revenue']);

// ============================================================
// PIPELINE STEP 4: Group — จัดกลุ่มตาม category
// ============================================================

$byCategory = [];
foreach ($enriched as $product) {
    $byCategory[$product['category']][] = $product;
}

// ============================================================
// PIPELINE STEP 5: Aggregate — สรุปข้อมูล
// ============================================================

$categoryStats = [];
foreach ($byCategory as $category => $products) {
    $prices   = array_column($products, 'price');
    $revenues = array_column($products, 'revenue');
    $ratings  = array_column($products, 'rating');

    $categoryStats[$category] = [
        'count'        => count($products),
        'avg_price'    => round(array_sum($prices) / count($prices), 2),
        'total_revenue' => array_sum($revenues),
        'avg_rating'   => round(array_sum($ratings) / count($ratings), 2),
        'best_seller'  => $products[0]['name'], // อยู่ด้านบนเพราะ sort ตาม revenue
    ];
}

// ============================================================
// OUTPUT: แสดงผล
// ============================================================

echo "\n" . str_repeat('=', 60) . "\n";
echo "       รายงานสินค้า Top 5 (เรียงตาม Revenue)\n";
echo str_repeat('=', 60) . "\n";

$top5 = array_slice(array_values($enriched), 0, 5);
foreach ($top5 as $i => $product) {
    $rank = $i + 1;
    echo "\n#{$rank} {$product['name']}\n";
    echo "   ราคา:     ฿" . number_format($product['price']) . "\n";
    echo "   ขายได้:   " . number_format($product['sold']) . " ชิ้น\n";
    echo "   Revenue:  ฿" . number_format($product['revenue']) . "\n";
    echo "   Rating:   {$product['rating']}/5.0\n";
    echo "   Stock:    {$product['stock']} ({$product['stock_status']})\n";
}

echo "\n" . str_repeat('=', 60) . "\n";
echo "       สถิติตาม Category\n";
echo str_repeat('=', 60) . "\n";

foreach ($categoryStats as $category => $stats) {
    echo "\n📂 $category ({$stats['count']} รายการ)\n";
    echo "   ราคาเฉลี่ย:    ฿" . number_format($stats['avg_price'], 2) . "\n";
    echo "   รวม Revenue:   ฿" . number_format($stats['total_revenue']) . "\n";
    echo "   Rating เฉลี่ย: {$stats['avg_rating']}/5.0\n";
    echo "   Best Seller:   {$stats['best_seller']}\n";
}

// ============================================================
// BONUS: หา product ที่ต้องเติม stock
// ============================================================

$needRestock = array_filter(
    $rawProducts,
    fn($p) => $p['stock'] <= 15 && $p['sold'] > 300
);

echo "\n" . str_repeat('=', 60) . "\n";
echo "       สินค้าที่ต้องเติม Stock\n";
echo str_repeat('=', 60) . "\n";

foreach ($needRestock as $product) {
    $urgency = $product['stock'] === 0 ? "⚠️  หมดแล้ว!" : "📉 เหลือน้อย";
    echo "{$product['name']}: stock={$product['stock']} $urgency\n";
}
```

---

## ❓ Quiz

### คำถามที่ 1
`array_merge(['a' => 1, 'b' => 2], ['b' => 3, 'c' => 4])` จะได้ผลลัพธ์อะไร?

**ตัวเลือก:**
- a) `['a' => 1, 'b' => 2, 'c' => 4]`
- b) `['a' => 1, 'b' => 3, 'c' => 4]`
- c) Error เพราะ key ซ้ำ
- d) `['a' => 1, 'b' => 2, 'b' => 3, 'c' => 4]`

---

### คำถามที่ 2
`array_filter` กับ `array_map` ต่างกันอย่างไร?

**ตัวเลือก:**
- a) `array_filter` กรอง elements, `array_map` แปลงค่าทุก element
- b) `array_filter` เร็วกว่า `array_map`
- c) `array_map` สามารถลบ element ได้ แต่ `array_filter` ไม่ได้
- d) ไม่มีความต่าง

---

### คำถามที่ 3
ข้อใดต่อไปนี้จะทำให้ `$data[4]` มีค่าผิดพลาด?

```php
<?php
$data = [1, 2, 3, 4, 5];
foreach ($data as &$v) { $v *= 2; }
// ลืม unset($v);
foreach ([10, 20, 30] as $v) { }
echo $data[4];
```

**ตัวเลือก:**
- a) แสดง 10
- b) แสดง 30
- c) แสดง 10 (ถูก)
- d) แสดง 5 (ถูก)

---

### คำถามที่ 4
`array_column($records, 'name', 'id')` จะได้ผลลัพธ์แบบใด?

**ตัวเลือก:**
- a) array ของ name ทั้งหมด
- b) array ของ id ทั้งหมด
- c) associative array ที่ key คือ id และ value คือ name
- d) จำนวน records ที่มี column 'name' และ 'id'

---

## ✅ เฉลย Quiz

**ข้อ 1: b) `['a' => 1, 'b' => 3, 'c' => 4]`**
> `array_merge` กับ associative keys: ถ้า key ซ้ำ ค่าหลังจะ override ค่าก่อน
> ดังนั้น `'b' => 2` จะถูก override ด้วย `'b' => 3`

**ข้อ 2: a) array_filter กรอง, array_map แปลงค่า**
> - `array_filter(arr, fn)` — คืน elements ที่ callback ส่งคืน true
> - `array_map(fn, arr)` — คืน array ใหม่ที่ทุก element ถูกแปลงด้วย callback

**ข้อ 3: b) แสดง 30**
> หลัง `foreach ($data as &$v)` โดยไม่ `unset($v)` — `$v` ยัง reference ไปยัง `$data[4]`
> foreach loop ถัดมา `foreach ([10, 20, 30] as $v)` กำหนดค่า 10, 20, 30 ให้ `$v`
> ค่าสุดท้ายคือ 30 ซึ่งเขียนลงใน `$data[4]` ด้วย!

**ข้อ 4: c) associative array ที่ key คือ id และ value คือ name**
> `array_column($records, 'name', 'id')` → คอลัมน์ 'name' เป็น value, คอลัมน์ 'id' เป็น key
> ผลลัพธ์: `[1 => 'สมชาย', 2 => 'สมหญิง', ...]`

---

## 🔗 สรุป Array Functions

| ฟังก์ชัน | หน้าที่ |
|---------|---------|
| `sort/rsort/usort` | เรียงลำดับ |
| `array_filter` | กรอง elements |
| `array_map` | แปลงทุก element |
| `array_reduce` | รวมเป็นค่าเดียว |
| `array_search/in_array` | ค้นหา |
| `array_slice/splice` | ตัดและแทรก |
| `array_merge/+` | รวม arrays |
| `array_unique` | ลบซ้ำ |
| `array_column` | ดึงคอลัมน์ |
| `array_flip` | สลับ key-value |
| `array_chunk` | แบ่ง chunks |
| `array_intersect/diff` | set operations |

---

## ➡️ Part ถัดไป

👉 **[Part 008: PHP Strings — สตริง](./part-008-php-strings.md)**

ใน Part หน้า เราจะเรียนรู้:
- String functions ทั้งหมด
- Regular Expressions ใน PHP
- Multibyte string (mb_*)
- String Security
- Workshop: Template engine, Text processor
