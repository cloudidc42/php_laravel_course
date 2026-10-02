# 🔄 Part 005: PHP Loops — การวนซ้ำ

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ `for`, `while`, `do-while`, `foreach` ได้อย่างถูกกับงาน
- ควบคุมการวนซ้ำด้วย `break`, `continue`, `return`
- เขียน Nested loops และรู้จักปัญหาด้านประสิทธิภาพ
- สร้าง FizzBuzz, Fibonacci sequence, และ Matrix operations ได้

---

## 📖 1. for Loop

`for` ใช้เมื่อรู้จำนวนรอบที่ต้องการวนล่วงหน้า

```php
<?php
// รูปแบบ: for (init; condition; increment/decrement)
for ($i = 1; $i <= 5; $i++) {
    echo "รอบที่ $i\n";
}
// รอบที่ 1
// รอบที่ 2
// รอบที่ 3
// รอบที่ 4
// รอบที่ 5
```

### นับถอยหลัง

```php
<?php
for ($i = 10; $i >= 1; $i--) {
    echo "$i... ";
}
echo "🚀 ยิง!\n";
// 10... 9... 8... 7... 6... 5... 4... 3... 2... 1... 🚀 ยิง!
```

### เพิ่มทีละหลายตัว

```php
<?php
// นับทีละ 2 (เลขคู่)
for ($i = 0; $i <= 20; $i += 2) {
    echo "$i ";
}
// 0 2 4 6 8 10 12 14 16 18 20

echo PHP_EOL;

// นับทีละ 5
for ($i = 0; $i <= 50; $i += 5) {
    echo "$i ";
}
// 0 5 10 15 20 25 30 35 40 45 50
```

### for กับ array

```php
<?php
$fruits = ['แอปเปิ้ล', 'กล้วย', 'ส้ม', 'มะม่วง'];
$count = count($fruits);

for ($i = 0; $i < $count; $i++) {
    echo ($i + 1) . ". {$fruits[$i]}\n";
}
// 1. แอปเปิ้ล
// 2. กล้วย
// 3. ส้ม
// 4. มะม่วง
```

### หลาย expressions ใน for

```php
<?php
// for สามารถมีหลาย expression ในแต่ละส่วนได้
for ($i = 0, $j = 10; $i < 5; $i++, $j--) {
    echo "i=$i, j=$j\n";
}
// i=0, j=10
// i=1, j=9
// i=2, j=8
// i=3, j=7
// i=4, j=6
```

---

## 📖 2. while Loop

`while` ใช้เมื่อไม่รู้จำนวนรอบล่วงหน้า — วนจนกว่าเงื่อนไขจะเป็น false

```php
<?php
// รูปแบบ: while (condition) { ... }
$number = 1;

while ($number <= 5) {
    echo "เลข: $number\n";
    $number++;
}
```

### while กับ user input simulation

```php
<?php
// จำลองการรับ input จนกว่าจะถูกต้อง
$attempts = 0;
$maxAttempts = 3;
$correctPassword = "secret123";
$passwords = ["wrong1", "wrong2", "secret123"]; // จำลอง input

$index = 0;
while ($attempts < $maxAttempts) {
    $input = $passwords[$index++] ?? "";
    $attempts++;

    if ($input === $correctPassword) {
        echo "✅ เข้าสู่ระบบสำเร็จ (ครั้งที่ $attempts)\n";
        break;
    }

    echo "❌ รหัสผ่านผิด (ครั้งที่ $attempts/$maxAttempts)\n";
}

if ($attempts >= $maxAttempts && $input !== $correctPassword) {
    echo "🔒 บัญชีถูกล็อก\n";
}
```

### while กับ file reading simulation

```php
<?php
// จำลองการอ่านข้อมูลทีละ chunk
$data = "line1\nline2\nline3\nline4\nline5";
$lines = explode("\n", $data);
$index = 0;
$total = count($lines);

while ($index < $total) {
    $line = $lines[$index];
    echo "ประมวลผล: $line\n";
    $index++;
}
```

### ระวัง Infinite Loop!

```php
<?php
// ❌ อย่าทำ! — Infinite loop
// $i = 0;
// while ($i < 10) {
//     echo $i;
//     // ลืม $i++ ทำให้วนไม่รู้จบ!
// }

// ✅ ถ้าไม่แน่ใจ ใส่ safety counter
$i = 0;
$safetyLimit = 1000;
$loopCount   = 0;

while ($i < 10 && $loopCount < $safetyLimit) {
    echo "$i ";
    $i++;
    $loopCount++;
}
// 0 1 2 3 4 5 6 7 8 9
```

---

## 📖 3. do-while Loop

`do-while` ทำงานอย่างน้อย 1 ครั้งก่อนตรวจเงื่อนไข

```php
<?php
// รูปแบบ: do { ... } while (condition);
$i = 1;

do {
    echo "ทำงานครั้งที่ $i\n";
    $i++;
} while ($i <= 3);
// ทำงานครั้งที่ 1
// ทำงานครั้งที่ 2
// ทำงานครั้งที่ 3
```

### do-while vs while — ความต่าง

```php
<?php
// while — อาจไม่ทำเลยถ้าเงื่อนไขเป็น false ตั้งแต่ต้น
$count = 10;

while ($count < 5) {
    echo "while: ไม่แสดง\n";
    $count++;
}
echo "while: ผ่านไปเลยโดยไม่ทำงาน\n";

// do-while — ทำงาน 1 ครั้งเสมอ แม้เงื่อนไขเป็น false
$count = 10;

do {
    echo "do-while: แสดงครั้งนี้ ($count)\n";
    $count++;
} while ($count < 5);
// do-while: แสดงครั้งนี้ (10)
```

### ตัวอย่าง: Menu System

```php
<?php
// จำลองระบบเมนู — แสดงเมนูอย่างน้อย 1 ครั้ง
$choices = [1, 2, 3, 4]; // จำลอง user input
$choiceIndex = 0;

do {
    $userChoice = $choices[$choiceIndex++] ?? 4;

    echo "\n=== เมนูหลัก ===\n";
    echo "1. ดูสินค้า\n";
    echo "2. เพิ่มสินค้าลงตะกร้า\n";
    echo "3. ชำระเงิน\n";
    echo "4. ออกจากระบบ\n";
    echo "เลือก: $userChoice\n";

    switch ($userChoice) {
        case 1:
            echo "→ แสดงรายการสินค้า...\n";
            break;
        case 2:
            echo "→ เพิ่มสินค้า...\n";
            break;
        case 3:
            echo "→ ชำระเงิน...\n";
            break;
        case 4:
            echo "→ ออกจากระบบ\n";
            break;
        default:
            echo "→ ตัวเลือกไม่ถูกต้อง\n";
    }
} while ($userChoice !== 4 && $choiceIndex < count($choices));
```

---

## 📖 4. foreach Loop

`foreach` ออกแบบมาสำหรับ array และ iterable objects โดยเฉพาะ

```php
<?php
// foreach กับ indexed array
$colors = ['แดง', 'เขียว', 'น้ำเงิน', 'เหลือง'];

foreach ($colors as $color) {
    echo "สี: $color\n";
}

// foreach กับ key => value
foreach ($colors as $index => $color) {
    echo "[$index] $color\n";
}
```

### foreach กับ associative array

```php
<?php
$person = [
    'name'  => 'สมชาย',
    'age'   => 30,
    'email' => 'somchai@example.com',
    'city'  => 'กรุงเทพ',
];

foreach ($person as $key => $value) {
    echo "$key: $value\n";
}
// name: สมชาย
// age: 30
// email: somchai@example.com
// city: กรุงเทพ
```

### foreach กับ array ซ้อนกัน

```php
<?php
$students = [
    ['name' => 'สมชาย',  'score' => 85, 'grade' => 'B'],
    ['name' => 'สมหญิง', 'score' => 92, 'grade' => 'A'],
    ['name' => 'สมศรี',  'score' => 67, 'grade' => 'C+'],
];

echo "รายชื่อนักเรียน:\n";
echo str_repeat('-', 40) . "\n";

foreach ($students as $index => $student) {
    $no = $index + 1;
    echo "$no. {$student['name']}\t คะแนน: {$student['score']}\t เกรด: {$student['grade']}\n";
}
```

### foreach by reference

```php
<?php
$prices = [100, 200, 300, 400, 500];

// เพิ่ม VAT 7% ให้ทุกราคา — แก้ไข array โดยตรงด้วย &
foreach ($prices as &$price) {
    $price = $price * 1.07;
}
unset($price); // สำคัญมาก! ต้อง unset reference หลัง foreach

print_r($prices);
// Array ( [0] => 107 [1] => 214 [2] => 321 [3] => 428 [4] => 535 )
```

> **⚠️ คำเตือน:** ต้อง `unset($variable)` หลัง `foreach` by reference เสมอ มิเช่นนั้น `$price` ยังคง reference ไปยัง element สุดท้าย

### foreach กับ object

```php
<?php
class Product
{
    public function __construct(
        public string $name,
        public float  $price,
        public int    $stock,
    ) {}
}

$products = [
    new Product('หมวก', 299.0, 50),
    new Product('เสื้อ', 599.0, 30),
    new Product('กางเกง', 799.0, 20),
];

foreach ($products as $product) {
    $total = $product->price * $product->stock;
    echo "{$product->name}: {$product->price} บาท × {$product->stock} ชิ้น = " . number_format($total, 2) . " บาท\n";
}
```

---

## 📖 5. break / continue / return ใน Loops

### break — หยุดวนซ้ำ

```php
<?php
// หยุดเมื่อเจอค่าที่ต้องการ
$numbers = [3, 7, 1, 9, 4, 6, 2, 8];
$target = 9;
$found = false;

foreach ($numbers as $index => $number) {
    if ($number === $target) {
        echo "พบ $target ที่ตำแหน่ง $index\n";
        $found = true;
        break; // หยุดทันที ไม่ต้องวนต่อ
    }
}

if (!$found) {
    echo "ไม่พบ $target\n";
}
```

### break กับตัวเลข (Break Levels)

```php
<?php
// break 2 — หยุดออกจาก 2 ชั้น loop
for ($i = 0; $i < 3; $i++) {
    for ($j = 0; $j < 3; $j++) {
        if ($i === 1 && $j === 1) {
            echo "หยุดทั้งหมดที่ i=$i, j=$j\n";
            break 2; // ออกจาก loop ทั้ง 2 ชั้น
        }
        echo "i=$i, j=$j\n";
    }
}
// i=0, j=0
// i=0, j=1
// i=0, j=2
// i=1, j=0
// หยุดทั้งหมดที่ i=1, j=1
```

### continue — ข้ามรอบปัจจุบัน

```php
<?php
// ข้ามเลขคู่
for ($i = 1; $i <= 10; $i++) {
    if ($i % 2 === 0) {
        continue; // ข้ามรอบนี้ไป
    }
    echo "$i "; // แสดงเฉพาะเลขคี่
}
// 1 3 5 7 9

echo PHP_EOL;

// กรองค่าที่ไม่ต้องการ
$data = [1, -5, 3, 0, -2, 8, -1, 6];

echo "เลขบวกเท่านั้น: ";
foreach ($data as $num) {
    if ($num <= 0) {
        continue;
    }
    echo "$num ";
}
// เลขบวกเท่านั้น: 1 3 8 6
```

### continue กับตัวเลข

```php
<?php
// continue 2 — ข้ามไปยัง loop ชั้นนอก
for ($i = 1; $i <= 3; $i++) {
    for ($j = 1; $j <= 3; $j++) {
        if ($j === 2) {
            continue 2; // ข้ามไปยัง loop ชั้นนอก (i++)
        }
        echo "i=$i, j=$j\n";
    }
    echo "จบ i=$i\n"; // บรรทัดนี้จะไม่แสดงเพราะ continue 2
}
// i=1, j=1
// i=2, j=1
// i=3, j=1
```

### return ใน loop

```php
<?php
// return ออกจากทั้ง loop และ function
function findFirstPositive(array $numbers): ?int
{
    foreach ($numbers as $num) {
        if ($num > 0) {
            return $num; // ออกจาก function ทันที
        }
    }
    return null; // ถ้าไม่พบ
}

echo findFirstPositive([-3, -1, 5, 2, -4]); // 5
echo PHP_EOL;
echo findFirstPositive([-3, -1, -5])        ?? "ไม่พบ"; // ไม่พบ
```

---

## 📖 6. Nested Loops

```php
<?php
// สร้างตาราง multiplication
echo "ตารางสูตรคูณ:\n";
echo str_repeat('-', 50) . "\n";

for ($i = 1; $i <= 5; $i++) {
    for ($j = 1; $j <= 5; $j++) {
        printf("%4d", $i * $j);
    }
    echo PHP_EOL;
}
//    1   2   3   4   5
//    2   4   6   8  10
//    3   6   9  12  15
//    4   8  12  16  20
//    5  10  15  20  25
```

### Nested foreach

```php
<?php
$categories = [
    'ผลไม้'   => ['แอปเปิ้ล', 'กล้วย', 'ส้ม'],
    'ผัก'     => ['แครอท', 'บรอคโคลี', 'ผักโขม'],
    'เนื้อสัตว์' => ['ไก่', 'หมู', 'วัว'],
];

foreach ($categories as $category => $items) {
    echo "\n📂 $category:\n";
    foreach ($items as $index => $item) {
        $no = $index + 1;
        echo "  $no. $item\n";
    }
}
```

### ระวัง O(n²) Complexity

```php
<?php
// ❌ แบบที่ช้า — O(n²): nested loop หาค่าซ้ำ
function findDuplicates_SLOW(array $arr): array
{
    $duplicates = [];
    $n = count($arr);

    for ($i = 0; $i < $n; $i++) {
        for ($j = $i + 1; $j < $n; $j++) {
            if ($arr[$i] === $arr[$j] && !in_array($arr[$i], $duplicates)) {
                $duplicates[] = $arr[$i];
            }
        }
    }

    return $duplicates;
}

// ✅ แบบที่เร็วกว่า — O(n): ใช้ hash map
function findDuplicates_FAST(array $arr): array
{
    $seen = [];
    $duplicates = [];

    foreach ($arr as $item) {
        if (isset($seen[$item])) {
            $seen[$item]++;
            if ($seen[$item] === 2) {
                $duplicates[] = $item;
            }
        } else {
            $seen[$item] = 1;
        }
    }

    return $duplicates;
}

$data = [1, 3, 5, 3, 7, 1, 9, 5];
print_r(findDuplicates_FAST($data));
// Array ( [0] => 3 [1] => 1 [2] => 5 )
```

---

## 📖 7. Loop Performance Tips

### Tip 1: คำนวณ count() ก่อน loop

```php
<?php
$items = range(1, 10000);

// ❌ ช้ากว่า — นับ count ทุกรอบ (แม้ PHP จะ optimize บางส่วน)
for ($i = 0; $i < count($items); $i++) {
    // ...
}

// ✅ เร็วกว่า — คำนวณครั้งเดียว
$count = count($items);
for ($i = 0; $i < $count; $i++) {
    // ...
}
```

### Tip 2: ใช้ array functions แทน loop

```php
<?php
$numbers = range(1, 100);

// ❌ ใช้ loop
$sum = 0;
foreach ($numbers as $n) {
    $sum += $n;
}

// ✅ ใช้ built-in function — เร็วกว่าและอ่านง่ายกว่า
$sum = array_sum($numbers);
echo $sum; // 5050

// ❌ ใช้ loop กรอง
$evens = [];
foreach ($numbers as $n) {
    if ($n % 2 === 0) {
        $evens[] = $n;
    }
}

// ✅ ใช้ array_filter
$evens = array_filter($numbers, fn($n) => $n % 2 === 0);
```

### Tip 3: Early break เมื่อพบสิ่งที่ต้องการ

```php
<?php
$users = range(1, 100000);

// ❌ วนครบทุกตัวทุกครั้ง
function findUser_SLOW(array $users, int $targetId): ?int
{
    $found = null;
    foreach ($users as $id) {
        if ($id === $targetId) {
            $found = $id;
        }
    }
    return $found;
}

// ✅ หยุดทันทีเมื่อพบ
function findUser_FAST(array $users, int $targetId): ?int
{
    foreach ($users as $id) {
        if ($id === $targetId) {
            return $id; // early return
        }
    }
    return null;
}
```

### Tip 4: ใช้ Generator สำหรับ data ขนาดใหญ่

```php
<?php
// ❌ โหลดทั้งหมดเข้า memory
function getAllNumbers_SLOW(int $max): array
{
    $result = [];
    for ($i = 1; $i <= $max; $i++) {
        $result[] = $i;
    }
    return $result;
}

// ✅ ใช้ Generator — ประหยัด memory
function getAllNumbers_FAST(int $max): \Generator
{
    for ($i = 1; $i <= $max; $i++) {
        yield $i;
    }
}

// ใช้งาน
$sum = 0;
foreach (getAllNumbers_FAST(1000000) as $number) {
    $sum += $number;
}
echo $sum; // 500000500000
```

---

## 🔧 Workshop 1: FizzBuzz

FizzBuzz เป็นโจทย์คลาสสิก:
- หาร 3 ลงตัว → พิมพ์ "Fizz"
- หาร 5 ลงตัว → พิมพ์ "Buzz"
- หาร 15 ลงตัว → พิมพ์ "FizzBuzz"
- อื่นๆ → พิมพ์ตัวเลขนั้น

```php
<?php
/**
 * FizzBuzz — หลายวิธี
 */

// วิธีที่ 1: แบบพื้นฐาน
function fizzBuzz_basic(int $max): array
{
    $result = [];

    for ($i = 1; $i <= $max; $i++) {
        if ($i % 15 === 0) {
            $result[] = 'FizzBuzz';
        } elseif ($i % 3 === 0) {
            $result[] = 'Fizz';
        } elseif ($i % 5 === 0) {
            $result[] = 'Buzz';
        } else {
            $result[] = (string)$i;
        }
    }

    return $result;
}

// วิธีที่ 2: ใช้ string concatenation (elegant)
function fizzBuzz_elegant(int $max): array
{
    $result = [];

    for ($i = 1; $i <= $max; $i++) {
        $output = '';
        $output .= ($i % 3 === 0) ? 'Fizz' : '';
        $output .= ($i % 5 === 0) ? 'Buzz' : '';
        $result[] = $output ?: (string)$i;
    }

    return $result;
}

// วิธีที่ 3: ใช้ match
function fizzBuzz_match(int $max): array
{
    $result = [];

    for ($i = 1; $i <= $max; $i++) {
        $result[] = match(true) {
            $i % 15 === 0 => 'FizzBuzz',
            $i % 3  === 0 => 'Fizz',
            $i % 5  === 0 => 'Buzz',
            default       => (string)$i,
        };
    }

    return $result;
}

// ทดสอบ
echo "FizzBuzz 1-20:\n";
echo implode(', ', fizzBuzz_elegant(20)) . "\n";
// 1, 2, Fizz, 4, Buzz, Fizz, 7, 8, Fizz, Buzz, 11, Fizz, 13, 14, FizzBuzz, 16, 17, Fizz, 19, Buzz

// ขยายเป็น FizzBuzzWoof (หาร 7 → Woof)
function fizzBuzzWoof(int $max): array
{
    $result = [];

    for ($i = 1; $i <= $max; $i++) {
        $output = '';
        $output .= ($i % 3 === 0) ? 'Fizz' : '';
        $output .= ($i % 5 === 0) ? 'Buzz' : '';
        $output .= ($i % 7 === 0) ? 'Woof' : '';
        $result[] = $output ?: (string)$i;
    }

    return $result;
}

echo "\nFizzBuzzWoof 1-30:\n";
echo implode(', ', fizzBuzzWoof(30)) . "\n";
```

---

## 🔧 Workshop 2: Fibonacci Sequence

```php
<?php
/**
 * Fibonacci Sequence: 0, 1, 1, 2, 3, 5, 8, 13, 21, ...
 * F(n) = F(n-1) + F(n-2)
 */

// วิธีที่ 1: Iterative (วนซ้ำ) — แนะนำ
function fibonacci_iterative(int $n): array
{
    if ($n <= 0) return [];
    if ($n === 1) return [0];

    $sequence = [0, 1];

    for ($i = 2; $i < $n; $i++) {
        $sequence[] = $sequence[$i - 1] + $sequence[$i - 2];
    }

    return $sequence;
}

// วิธีที่ 2: ใช้ while loop
function fibonacci_while(int $count): array
{
    $sequence = [];
    $a = 0;
    $b = 1;
    $i = 0;

    while ($i < $count) {
        $sequence[] = $a;
        [$a, $b] = [$b, $a + $b]; // swap ด้วย destructuring
        $i++;
    }

    return $sequence;
}

// วิธีที่ 3: Generator (ประหยัด memory สำหรับ sequence ใหญ่)
function fibonacci_generator(): \Generator
{
    [$a, $b] = [0, 1];

    while (true) {
        yield $a;
        [$a, $b] = [$b, $a + $b];
    }
}

// ทดสอบ
echo "Fibonacci 10 ตัว (iterative):\n";
echo implode(', ', fibonacci_iterative(10)) . "\n";
// 0, 1, 1, 2, 3, 5, 8, 13, 21, 34

echo "\nFibonacci 10 ตัว (while):\n";
echo implode(', ', fibonacci_while(10)) . "\n";

echo "\nFibonacci 10 ตัว (generator):\n";
$gen = fibonacci_generator();
$results = [];
for ($i = 0; $i < 10; $i++) {
    $results[] = $gen->current();
    $gen->next();
}
echo implode(', ', $results) . "\n";

// ตรวจสอบว่าเป็น Fibonacci number หรือไม่
function isFibonacci(int $n): bool
{
    // n เป็น Fibonacci ถ้า 5n² ± 4 เป็นกำลังสองสมบูรณ์
    $check1 = 5 * $n * $n + 4;
    $check2 = 5 * $n * $n - 4;

    $sqrt1 = (int)sqrt($check1);
    $sqrt2 = (int)sqrt($check2);

    return ($sqrt1 * $sqrt1 === $check1) || ($sqrt2 * $sqrt2 === $check2);
}

echo "\nตรวจสอบ Fibonacci numbers:\n";
$testNumbers = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 13, 21, 34, 50];
foreach ($testNumbers as $num) {
    $isFib = isFibonacci($num) ? "✅ เป็น" : "❌ ไม่เป็น";
    echo "$num: $isFib Fibonacci\n";
}
```

---

## 🔧 Workshop 3: Matrix Operations

```php
<?php
/**
 * Matrix Operations — การทำงานกับเมทริกซ์
 */

/**
 * สร้างเมทริกซ์ขนาด rows x cols
 */
function createMatrix(int $rows, int $cols, int $defaultValue = 0): array
{
    $matrix = [];
    for ($i = 0; $i < $rows; $i++) {
        for ($j = 0; $j < $cols; $j++) {
            $matrix[$i][$j] = $defaultValue;
        }
    }
    return $matrix;
}

/**
 * แสดงเมทริกซ์ในรูปแบบสวยงาม
 */
function printMatrix(array $matrix, string $title = ""): void
{
    if ($title) {
        echo "\n$title\n";
    }
    echo str_repeat('-', count($matrix[0]) * 6 + 1) . "\n";

    foreach ($matrix as $row) {
        echo "| ";
        foreach ($row as $cell) {
            printf("%4d ", $cell);
        }
        echo "|\n";
    }
    echo str_repeat('-', count($matrix[0]) * 6 + 1) . "\n";
}

/**
 * บวกเมทริกซ์ 2 ตัว
 */
function addMatrices(array $a, array $b): array|string
{
    $rowsA = count($a);
    $colsA = count($a[0]);
    $rowsB = count($b);
    $colsB = count($b[0]);

    if ($rowsA !== $rowsB || $colsA !== $colsB) {
        return "ขนาดเมทริกซ์ไม่เท่ากัน";
    }

    $result = createMatrix($rowsA, $colsA);

    for ($i = 0; $i < $rowsA; $i++) {
        for ($j = 0; $j < $colsA; $j++) {
            $result[$i][$j] = $a[$i][$j] + $b[$i][$j];
        }
    }

    return $result;
}

/**
 * คูณเมทริกซ์ 2 ตัว (Matrix Multiplication)
 */
function multiplyMatrices(array $a, array $b): array|string
{
    $rowsA = count($a);
    $colsA = count($a[0]);
    $rowsB = count($b);
    $colsB = count($b[0]);

    if ($colsA !== $rowsB) {
        return "ขนาดเมทริกซ์ไม่รองรับการคูณ (colsA ต้องเท่ากับ rowsB)";
    }

    $result = createMatrix($rowsA, $colsB);

    for ($i = 0; $i < $rowsA; $i++) {
        for ($j = 0; $j < $colsB; $j++) {
            for ($k = 0; $k < $colsA; $k++) {
                $result[$i][$j] += $a[$i][$k] * $b[$k][$j];
            }
        }
    }

    return $result;
}

/**
 * Transpose เมทริกซ์ (สลับแถวและคอลัมน์)
 */
function transposeMatrix(array $matrix): array
{
    $rows = count($matrix);
    $cols = count($matrix[0]);
    $result = createMatrix($cols, $rows);

    for ($i = 0; $i < $rows; $i++) {
        for ($j = 0; $j < $cols; $j++) {
            $result[$j][$i] = $matrix[$i][$j];
        }
    }

    return $result;
}

/**
 * หา diagonal sum ของ square matrix
 */
function diagonalSum(array $matrix): array
{
    $n = count($matrix);
    $primarySum = 0;
    $secondarySum = 0;

    for ($i = 0; $i < $n; $i++) {
        $primarySum   += $matrix[$i][$i];
        $secondarySum += $matrix[$i][$n - 1 - $i];
    }

    return [
        'primary'   => $primarySum,
        'secondary' => $secondarySum,
    ];
}

// ===== ทดสอบ =====

$matrixA = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9],
];

$matrixB = [
    [9, 8, 7],
    [6, 5, 4],
    [3, 2, 1],
];

printMatrix($matrixA, "เมทริกซ์ A:");
printMatrix($matrixB, "เมทริกซ์ B:");

$sum = addMatrices($matrixA, $matrixB);
printMatrix($sum, "A + B:");

$product = multiplyMatrices($matrixA, $matrixB);
printMatrix($product, "A × B:");

$transposed = transposeMatrix($matrixA);
printMatrix($transposed, "Transpose A:");

$diag = diagonalSum($matrixA);
echo "\nDiagonal sums ของ A:\n";
echo "Primary diagonal (↘):   {$diag['primary']}\n";
echo "Secondary diagonal (↙): {$diag['secondary']}\n";

// Pascal's Triangle
echo "\nPascal's Triangle (5 แถว):\n";
$pascal = [[1]];
for ($i = 1; $i < 5; $i++) {
    $row = [1];
    for ($j = 1; $j < $i; $j++) {
        $row[] = $pascal[$i - 1][$j - 1] + $pascal[$i - 1][$j];
    }
    $row[] = 1;
    $pascal[] = $row;
}

foreach ($pascal as $i => $row) {
    $spaces = str_repeat("  ", 4 - $i);
    echo $spaces;
    foreach ($row as $val) {
        printf("%4d", $val);
    }
    echo "\n";
}
```

---

## ❓ Quiz

### คำถามที่ 1
โค้ดต่อไปนี้จะแสดงผลอะไร?

```php
<?php
for ($i = 0; $i < 5; $i++) {
    if ($i === 3) {
        continue;
    }
    echo $i . " ";
}
```

**ตัวเลือก:**
- a) 0 1 2 3 4
- b) 0 1 2 4
- c) 0 1 2
- d) 1 2 4 5

---

### คำถามที่ 2
อะไรคือความแตกต่างระหว่าง `while` และ `do-while`?

**ตัวเลือก:**
- a) `do-while` เร็วกว่า `while`
- b) `do-while` ทำงานอย่างน้อย 1 ครั้งเสมอ แม้เงื่อนไขเป็น false ตั้งแต่ต้น
- c) `while` รองรับ `break` แต่ `do-while` ไม่รองรับ
- d) ไม่มีความแตกต่าง

---

### คำถามที่ 3
โค้ดต่อไปนี้มีปัญหาอะไร?

```php
<?php
$data = [1, 2, 3, 4, 5];
foreach ($data as &$item) {
    $item *= 2;
}
echo $data[4]; // ยังไม่มีปัญหาตรงนี้

$other = [10, 20, 30];
foreach ($other as $item) {
    echo $item . " ";
}
echo $data[4]; // ค่านี้อาจผิดพลาด!
```

**ตัวเลือก:**
- a) ไม่มีปัญหา
- b) ลืม `unset($item)` หลัง foreach by reference ทำให้ `$item` ยัง reference ไปยัง `$data[4]`
- c) `$data` และ `$other` ใช้ชื่อ `$item` เหมือนกัน
- d) ข้อ b และ c ถูกทั้งคู่

---

### คำถามที่ 4
อะไรคือปัญหาของการใช้ Nested Loops กับ array ขนาดใหญ่?

**ตัวเลือก:**
- a) PHP ไม่รองรับ nested loops
- b) Time complexity เป็น O(n²) หรือมากกว่า ทำให้ช้ามากเมื่อข้อมูลมาก
- c) ใช้ memory มากขึ้น 2 เท่า
- d) ไม่มีปัญหา

---

## ✅ เฉลย Quiz

**ข้อ 1: b) 0 1 2 4**
> `continue` ข้ามรอบที่ `$i === 3` ทำให้ไม่พิมพ์ 3 แต่วนต่อไปถึง 4

**ข้อ 2: b) `do-while` ทำงานอย่างน้อย 1 ครั้งเสมอ**
> `while` ตรวจเงื่อนไขก่อน ถ้า false ตั้งแต่ต้นก็ไม่ทำงานเลย แต่ `do-while` ทำงานก่อนแล้วค่อยตรวจเงื่อนไข

**ข้อ 3: d) ข้อ b และ c ถูกทั้งคู่**
> หลัง `foreach ($data as &$item)` โดยไม่ `unset($item)` — `$item` ยัง reference ไปยัง `$data[4]`
> ดังนั้น `foreach ($other as $item)` จะเขียนค่าลงใน `$data[4]` ด้วย!
> ค่า `$data[4]` จะเปลี่ยนเป็นค่าสุดท้ายของ `$other` (คือ 30)

**ข้อ 4: b) Time complexity เป็น O(n²)**
> Nested loop 2 ชั้นกับ n elements แต่ละชั้น = n × n = n² operations
> ถ้า n = 1,000 → ทำงาน 1,000,000 ครั้ง
> ถ้า n = 10,000 → ทำงาน 100,000,000 ครั้ง (ช้ามาก!)

---

## 🔗 สรุปเปรียบเทียบ Loops

| Loop | ใช้เมื่อ | ตรวจเงื่อนไข |
|------|---------|--------------|
| `for` | รู้จำนวนรอบล่วงหน้า | ก่อนแต่ละรอบ |
| `while` | ไม่รู้จำนวนรอบ | ก่อนแต่ละรอบ |
| `do-while` | ต้องทำอย่างน้อย 1 ครั้ง | หลังแต่ละรอบ |
| `foreach` | วนผ่าน array/iterable | อัตโนมัติ |

---

## ➡️ Part ถัดไป

👉 **[Part 006: PHP Functions — ฟังก์ชัน](./part-006-php-functions.md)**

ใน Part หน้า เราจะเรียนรู้:
- Function declaration และ parameters
- Return values และ type hints
- Anonymous functions (closures) และ Arrow functions
- Recursive functions
- Workshop: สร้าง utility functions library
