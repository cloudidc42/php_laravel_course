# 🔀 Part 004: PHP Control Structures — การควบคุมการทำงานของโปรแกรม

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ `if/elseif/else` ควบคุมเงื่อนไขได้อย่างมืออาชีพ
- เลือกใช้ `switch/case` และ `match` expression ได้อย่างถูกต้อง
- เขียน ternary conditions และ null coalescing ได้อย่างกระชับ
- ใช้ Guard clauses และ Early return pattern เพื่อเขียนโค้ดที่อ่านง่าย
- สร้างระบบ grading และ shipping calculator จริงๆ ได้

---

## 📖 1. if / elseif / else

### หลักการพื้นฐาน

`if` คือหัวใจของการตัดสินใจในโปรแกรม เราใช้มันเพื่อบอกว่า "ถ้า...ให้ทำ...มิเช่นนั้น...ให้ทำ..."

```php
<?php
// โครงสร้างพื้นฐานของ if statement
$age = 20;

if ($age >= 18) {
    echo "คุณเป็นผู้ใหญ่แล้ว สามารถลงคะแนนเสียงได้";
}
```

```php
<?php
// if...else
$score = 75;

if ($score >= 50) {
    echo "สอบผ่าน ✅";
} else {
    echo "สอบตก ❌";
}
```

```php
<?php
// if...elseif...else
$score = 85;

if ($score >= 90) {
    echo "เกรด A";
} elseif ($score >= 80) {
    echo "เกรด B";
} elseif ($score >= 70) {
    echo "เกรด C";
} elseif ($score >= 60) {
    echo "เกรด D";
} else {
    echo "เกรด F";
}
// Output: เกรด B
```

### การเปรียบเทียบ == vs ===

```php
<?php
$value = "5";

// == เปรียบเทียบค่า (loose comparison)
if ($value == 5) {
    echo "== ผ่าน: PHP แปลง string '5' เป็น int 5 ให้อัตโนมัติ";
}

// === เปรียบเทียบทั้งค่าและชนิดข้อมูล (strict comparison)
if ($value === 5) {
    echo "ไม่แสดง เพราะ string ไม่เท่ากับ int";
} else {
    echo "=== ไม่ผ่าน: '5' (string) ไม่เหมือน 5 (int)";
}
```

> **⚠️ คำแนะนำ:** ใช้ `===` เสมอเพื่อหลีกเลี่ยง bug ที่เกิดจาก type coercion

### Truthy และ Falsy Values

```php
<?php
// ค่าที่ PHP ถือว่าเป็น false (falsy):
$falsy_values = [
    false,      // boolean false
    0,          // integer 0
    0.0,        // float 0.0
    "0",        // string "0"
    "",         // empty string
    [],         // empty array
    null,       // null
];

// ตัวอย่างการใช้งาน
$username = "";

if ($username) {
    echo "มีชื่อผู้ใช้: $username";
} else {
    echo "ยังไม่ได้กรอกชื่อผู้ใช้";
}
// Output: ยังไม่ได้กรอกชื่อผู้ใช้

// วิธีที่ดีกว่า — ตรวจสอบชัดเจน
if (!empty($username)) {
    echo "มีชื่อผู้ใช้: $username";
} else {
    echo "ยังไม่ได้กรอกชื่อผู้ใช้";
}
```

### Nested if (if ซ้อน if)

```php
<?php
$isLoggedIn = true;
$isAdmin = false;
$hasPermission = true;

if ($isLoggedIn) {
    if ($isAdmin) {
        echo "ยินดีต้อนรับ Admin! คุณมีสิทธิ์ทุกอย่าง";
    } elseif ($hasPermission) {
        echo "ยินดีต้อนรับ! คุณมีสิทธิ์เข้าถึงส่วนนี้";
    } else {
        echo "คุณไม่มีสิทธิ์เข้าถึงส่วนนี้";
    }
} else {
    echo "กรุณาเข้าสู่ระบบก่อน";
}
// Output: ยินดีต้อนรับ! คุณมีสิทธิ์เข้าถึงส่วนนี้
```

> **💡 เทคนิค:** Nested if ที่ลึกเกิน 3 ชั้นมักเป็นสัญญาณว่าควรใช้ Guard Clauses แทน

---

## 📖 2. switch / case

`switch` ใช้เมื่อต้องเปรียบเทียบตัวแปรหนึ่งกับหลายๆ ค่า

```php
<?php
$day = "Monday";

switch ($day) {
    case "Monday":
        echo "วันจันทร์ — เริ่มสัปดาห์ใหม่!";
        break;
    case "Tuesday":
        echo "วันอังคาร";
        break;
    case "Wednesday":
        echo "วันพุธ — กลางสัปดาห์แล้ว!";
        break;
    case "Thursday":
        echo "วันพฤหัสบดี";
        break;
    case "Friday":
        echo "วันศุกร์ — ใกล้หยุดแล้ว! 🎉";
        break;
    case "Saturday":
    case "Sunday":
        echo "วันหยุดสุดสัปดาห์! 😴";
        break;
    default:
        echo "ไม่รู้จักวันนี้";
}
// Output: วันจันทร์ — เริ่มสัปดาห์ใหม่!
```

### Fall-through (ไม่มี break)

```php
<?php
// Fall-through — ทำงานต่อเนื่องหากไม่มี break
$grade = "B";

switch ($grade) {
    case "A":
    case "B":
    case "C":
        echo "สอบผ่าน";
        break;
    case "D":
    case "F":
        echo "สอบตก";
        break;
    default:
        echo "เกรดไม่ถูกต้อง";
}
// Output: สอบผ่าน
```

### switch กับ return (ใน function)

```php
<?php
function getDayType(string $day): string
{
    switch ($day) {
        case "Saturday":
        case "Sunday":
            return "วันหยุดสุดสัปดาห์";
        case "Monday":
        case "Tuesday":
        case "Wednesday":
        case "Thursday":
        case "Friday":
            return "วันทำงาน";
        default:
            return "ไม่รู้จัก";
    }
}

echo getDayType("Saturday"); // วันหยุดสุดสัปดาห์
echo PHP_EOL;
echo getDayType("Tuesday");  // วันทำงาน
```

### ข้อควรระวัง: switch ใช้ loose comparison (==)

```php
<?php
$value = 0;

switch ($value) {
    case false:
        echo "เป็น false"; // จะแสดงข้อความนี้!
        break;
    case 0:
        echo "เป็น 0";
        break;
}
// Output: เป็น false — ระวัง! switch ใช้ == ไม่ใช่ ===
```

---

## 📖 3. Match Expression (PHP 8.0+)

`match` คือเวอร์ชันที่ดีกว่าของ `switch` ที่ใช้ strict comparison (===)

```php
<?php
$status = 2;

$message = match($status) {
    1 => "กำลังดำเนินการ",
    2 => "สำเร็จแล้ว",
    3 => "ยกเลิก",
    default => "ไม่รู้จัก status",
};

echo $message; // สำเร็จแล้ว
```

### match vs switch — ความแตกต่างสำคัญ

```php
<?php
// 1. match ส่งคืนค่า
$discount = match(true) {
    $age < 12  => 50,  // เด็ก — ลด 50%
    $age >= 65 => 30,  // ผู้สูงอายุ — ลด 30%
    default    => 0,   // ปกติ — ไม่ลด
};

// 2. match ใช้ === (strict)
$value = "1";
$result = match($value) {
    1     => "integer 1",
    "1"   => "string '1'",  // จะถูกเลือกข้อนี้
    true  => "boolean true",
};
echo $result; // string '1'

// 3. match โยน UnhandledMatchError ถ้าไม่มี arm ที่ตรง
try {
    $x = 99;
    $result = match($x) {
        1 => "หนึ่ง",
        2 => "สอง",
        // ไม่มี default — ถ้า $x ไม่ตรงกับ 1 หรือ 2 จะ throw error
    };
} catch (\UnhandledMatchError $e) {
    echo "ไม่มี arm ที่ตรงกับ 99";
}
```

### match กับหลาย conditions

```php
<?php
$lang = "th";

$greeting = match($lang) {
    "en", "en-US", "en-GB" => "Hello!",
    "th", "th-TH"          => "สวัสดี!",
    "ja", "ja-JP"          => "こんにちは!",
    "zh", "zh-CN", "zh-TW" => "你好!",
    default                 => "Hi!",
};

echo $greeting; // สวัสดี!
```

### match ใน real-world scenario

```php
<?php
function getHttpStatusMessage(int $code): string
{
    return match($code) {
        200 => "OK",
        201 => "Created",
        204 => "No Content",
        301 => "Moved Permanently",
        302 => "Found",
        400 => "Bad Request",
        401 => "Unauthorized",
        403 => "Forbidden",
        404 => "Not Found",
        405 => "Method Not Allowed",
        422 => "Unprocessable Entity",
        429 => "Too Many Requests",
        500 => "Internal Server Error",
        502 => "Bad Gateway",
        503 => "Service Unavailable",
        default => "Unknown Status Code",
    };
}

echo getHttpStatusMessage(404); // Not Found
echo PHP_EOL;
echo getHttpStatusMessage(200); // OK
```

---

## 📖 4. Ternary Conditions

### Ternary Operator (?:)

```php
<?php
// รูปแบบ: condition ? value_if_true : value_if_false
$age = 20;
$status = ($age >= 18) ? "ผู้ใหญ่" : "เยาวชน";
echo $status; // ผู้ใหญ่

// เทียบเท่ากับ:
if ($age >= 18) {
    $status = "ผู้ใหญ่";
} else {
    $status = "เยาวชน";
}
```

### Elvis Operator (?:) — Short Ternary

```php
<?php
// รูปแบบ: value ?: fallback
// ถ้า value เป็น truthy ใช้ value, ถ้าไม่ก็ใช้ fallback

$name = $_GET['name'] ?? null; // จาก query string
$displayName = $name ?: "ผู้ใช้ไม่ระบุชื่อ";
echo $displayName; // ผู้ใช้ไม่ระบุชื่อ (ถ้าไม่มี name ใน query)

// เทียบเท่ากับ:
$displayName = $name ? $name : "ผู้ใช้ไม่ระบุชื่อ";
```

### Null Coalescing Operator (??)

```php
<?php
// รูปแบบ: value ?? fallback
// ถ้า value ไม่ใช่ null ใช้ value, ถ้า null ใช้ fallback

$config = [
    'timezone' => 'Asia/Bangkok',
    'language' => 'th',
];

$timezone = $config['timezone'] ?? 'UTC';
$currency = $config['currency'] ?? 'THB'; // key ไม่มีใน array
$debug    = $config['debug'] ?? false;

echo $timezone; // Asia/Bangkok
echo $currency; // THB
var_dump($debug); // bool(false)
```

### Null Coalescing Assignment (??=) — PHP 7.4+

```php
<?php
$settings = [];

// ถ้า $settings['theme'] เป็น null หรือไม่มี key — set ค่า default
$settings['theme'] ??= 'dark';
$settings['lang']  ??= 'th';

// เทียบเท่ากับ:
$settings['theme'] = $settings['theme'] ?? 'dark';
$settings['lang']  = $settings['lang']  ?? 'th';

print_r($settings);
// Array ( [theme] => dark [lang] => th )
```

### Chaining Null Coalescing

```php
<?php
$user = null;
$cachedUser = null;
$defaultUser = ['name' => 'Guest', 'role' => 'visitor'];

// ลองใช้ค่าตามลำดับ — ใช้ค่าแรกที่ไม่เป็น null
$activeUser = $user ?? $cachedUser ?? $defaultUser;
echo $activeUser['name']; // Guest
```

---

## 📖 5. Guard Clauses / Early Return Pattern

Guard clauses คือเทคนิคการเขียนโค้ดที่ช่วยลด nesting และทำให้อ่านง่ายขึ้น

### ปัญหา: Deeply Nested Code (Arrow Anti-pattern)

```php
<?php
// ❌ แบบที่ไม่ดี — nested if ลึกมาก (Arrow Code)
function processOrder_BAD(array $order): string
{
    if ($order !== null) {
        if (!empty($order['items'])) {
            if ($order['total'] > 0) {
                if ($order['payment'] === 'paid') {
                    if ($order['stock_available']) {
                        // ทำงานจริงๆ อยู่ตรงนี้ ลึกมาก!
                        return "กำลังประมวลผลคำสั่งซื้อ: " . $order['id'];
                    } else {
                        return "สินค้าหมด";
                    }
                } else {
                    return "ยังไม่ได้ชำระเงิน";
                }
            } else {
                return "ยอดรวมไม่ถูกต้อง";
            }
        } else {
            return "ไม่มีสินค้าในคำสั่งซื้อ";
        }
    } else {
        return "คำสั่งซื้อไม่ถูกต้อง";
    }
}
```

### วิธีแก้: Guard Clauses (Early Return)

```php
<?php
// ✅ แบบที่ดี — Guard Clauses
function processOrder(array $order): string
{
    // Guard: ตรวจสอบเงื่อนไขผิดพลาดก่อน แล้ว return ออกทันที
    if ($order === null) {
        return "คำสั่งซื้อไม่ถูกต้อง";
    }

    if (empty($order['items'])) {
        return "ไม่มีสินค้าในคำสั่งซื้อ";
    }

    if ($order['total'] <= 0) {
        return "ยอดรวมไม่ถูกต้อง";
    }

    if ($order['payment'] !== 'paid') {
        return "ยังไม่ได้ชำระเงิน";
    }

    if (!$order['stock_available']) {
        return "สินค้าหมด";
    }

    // Happy path — อ่านง่ายขึ้นมาก!
    return "กำลังประมวลผลคำสั่งซื้อ: " . $order['id'];
}

// ทดสอบ
$order = [
    'id'              => 'ORD-001',
    'items'           => ['item1', 'item2'],
    'total'           => 1500,
    'payment'         => 'paid',
    'stock_available' => true,
];

echo processOrder($order); // กำลังประมวลผลคำสั่งซื้อ: ORD-001
```

### Guard Clauses กับ Validation

```php
<?php
function createUser(string $name, string $email, int $age): array|string
{
    // Guards — validate inputs
    if (empty(trim($name))) {
        return "ชื่อห้ามว่าง";
    }

    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        return "รูปแบบ email ไม่ถูกต้อง";
    }

    if ($age < 18) {
        return "ต้องมีอายุ 18 ปีขึ้นไป";
    }

    if ($age > 120) {
        return "อายุไม่สมเหตุสมผล";
    }

    // All validations passed
    return [
        'name'       => trim($name),
        'email'      => strtolower($email),
        'age'        => $age,
        'created_at' => date('Y-m-d H:i:s'),
    ];
}

// ทดสอบ
var_dump(createUser("", "test@example.com", 25));
// string(12) "ชื่อห้ามว่าง"

var_dump(createUser("สมชาย", "invalid-email", 25));
// string(30) "รูปแบบ email ไม่ถูกต้อง"

var_dump(createUser("สมชาย", "somchai@example.com", 15));
// string(27) "ต้องมีอายุ 18 ปีขึ้นไป"

$result = createUser("สมชาย ใจดี", "somchai@example.com", 30);
print_r($result);
```

---

## 🔧 Workshop 1: ระบบ Grading

สร้างระบบให้เกรดนักเรียนที่ครบครัน

```php
<?php
/**
 * ระบบให้เกรดนักเรียน
 * Workshop: PHP Control Structures
 */

/**
 * คำนวณเกรดจากคะแนน
 */
function calculateGrade(float $score): array
{
    // Guard clauses
    if ($score < 0) {
        return ['error' => "คะแนนต้องไม่ต่ำกว่า 0"];
    }

    if ($score > 100) {
        return ['error' => "คะแนนต้องไม่เกิน 100"];
    }

    // คำนวณเกรด
    [$grade, $gpa, $status] = match(true) {
        $score >= 80 => ['A', 4.0, 'ดีเยี่ยม'],
        $score >= 75 => ['B+', 3.5, 'ดีมาก'],
        $score >= 70 => ['B', 3.0, 'ดี'],
        $score >= 65 => ['C+', 2.5, 'ค่อนข้างดี'],
        $score >= 60 => ['C', 2.0, 'พอใช้'],
        $score >= 55 => ['D+', 1.5, 'อ่อน'],
        $score >= 50 => ['D', 1.0, 'อ่อนมาก'],
        default      => ['F', 0.0, 'ตก'],
    };

    $passed = $grade !== 'F';

    return [
        'score'   => $score,
        'grade'   => $grade,
        'gpa'     => $gpa,
        'status'  => $status,
        'passed'  => $passed,
        'message' => $passed
            ? "ผ่าน: ได้เกรด $grade ($status)"
            : "ไม่ผ่าน: ได้เกรด F (ตก)",
    ];
}

/**
 * คำนวณสถิติของนักเรียนทั้งห้อง
 */
function calculateClassStats(array $students): array
{
    if (empty($students)) {
        return ['error' => "ไม่มีข้อมูลนักเรียน"];
    }

    $scores = array_column($students, 'score');
    $total  = count($scores);

    // นับจำนวนผ่าน/ตก
    $passed = 0;
    $failed = 0;
    $gradeCount = ['A' => 0, 'B+' => 0, 'B' => 0, 'C+' => 0, 'C' => 0, 'D+' => 0, 'D' => 0, 'F' => 0];

    foreach ($students as $student) {
        $result = calculateGrade($student['score']);
        if (isset($result['error'])) continue;

        if ($result['passed']) {
            $passed++;
        } else {
            $failed++;
        }

        if (isset($gradeCount[$result['grade']])) {
            $gradeCount[$result['grade']]++;
        }
    }

    return [
        'total_students' => $total,
        'passed'         => $passed,
        'failed'         => $failed,
        'pass_rate'      => round(($passed / $total) * 100, 2),
        'average'        => round(array_sum($scores) / $total, 2),
        'highest'        => max($scores),
        'lowest'         => min($scores),
        'grade_count'    => $gradeCount,
    ];
}

// ===== ทดสอบระบบ =====

echo "=== ทดสอบการให้เกรดเดี่ยว ===\n";
$testScores = [95, 82, 73, 67, 55, 45];

foreach ($testScores as $score) {
    $result = calculateGrade($score);
    echo "คะแนน {$result['score']}: {$result['message']}\n";
}

echo "\n=== ทดสอบข้อผิดพลาด ===\n";
$errorTests = [-5, 105];
foreach ($errorTests as $score) {
    $result = calculateGrade($score);
    echo "คะแนน $score: " . ($result['error'] ?? "ปกติ") . "\n";
}

echo "\n=== สถิติห้องเรียน ===\n";
$students = [
    ['name' => 'สมชาย',   'score' => 92],
    ['name' => 'สมหญิง',  'score' => 78],
    ['name' => 'สมศรี',   'score' => 65],
    ['name' => 'สมปอง',   'score' => 55],
    ['name' => 'สมใจ',    'score' => 43],
    ['name' => 'สมบัติ',  'score' => 88],
    ['name' => 'สมนึก',   'score' => 71],
    ['name' => 'สมพร',    'score' => 60],
];

$stats = calculateClassStats($students);
echo "นักเรียนทั้งหมด: {$stats['total_students']} คน\n";
echo "ผ่าน: {$stats['passed']} คน | ตก: {$stats['failed']} คน\n";
echo "อัตราผ่าน: {$stats['pass_rate']}%\n";
echo "คะแนนเฉลี่ย: {$stats['average']}\n";
echo "คะแนนสูงสุด: {$stats['highest']} | ต่ำสุด: {$stats['lowest']}\n";
echo "\nการกระจายเกรด:\n";
foreach ($stats['grade_count'] as $grade => $count) {
    if ($count > 0) {
        $bar = str_repeat('█', $count);
        echo "  เกรด $grade: $bar ($count คน)\n";
    }
}
```

---

## 🔧 Workshop 2: Shipping Calculator

```php
<?php
/**
 * ระบบคำนวณค่าจัดส่ง
 * Workshop: PHP Control Structures
 */

/**
 * คำนวณค่าจัดส่งตามโซน และน้ำหนัก
 */
function calculateShipping(
    float  $weight,
    string $zone,
    string $type = 'standard',
    bool   $insurance = false
): array {
    // Guard clauses
    if ($weight <= 0) {
        return ['error' => "น้ำหนักต้องมากกว่า 0"];
    }

    if ($weight > 30) {
        return ['error' => "น้ำหนักเกิน 30 กิโลกรัม กรุณาติดต่อเราโดยตรง"];
    }

    $validZones = ['bangkok', 'central', 'north', 'south', 'northeast', 'east'];
    if (!in_array(strtolower($zone), $validZones)) {
        return ['error' => "โซนไม่ถูกต้อง: $zone"];
    }

    $validTypes = ['standard', 'express', 'overnight'];
    if (!in_array($type, $validTypes)) {
        return ['error' => "ประเภทการจัดส่งไม่ถูกต้อง"];
    }

    // คำนวณราคาฐานตามโซน (ต่อกิโลกรัม)
    $baseRatePerKg = match(strtolower($zone)) {
        'bangkok'   => 20,
        'central'   => 25,
        'north'     => 35,
        'south'     => 35,
        'northeast' => 40,
        'east'      => 30,
    };

    // ราคาขั้นต่ำตามโซน
    $minimumFee = match(strtolower($zone)) {
        'bangkok'   => 50,
        'central'   => 60,
        'north'     => 80,
        'south'     => 80,
        'northeast' => 90,
        'east'      => 70,
    };

    // คำนวณราคาตามน้ำหนัก
    $weightFee = match(true) {
        $weight <= 1  => $minimumFee,
        $weight <= 5  => $minimumFee + (($weight - 1) * $baseRatePerKg),
        $weight <= 10 => $minimumFee + (4 * $baseRatePerKg) + (($weight - 5) * $baseRatePerKg * 0.9),
        default       => $minimumFee + (4 * $baseRatePerKg) + (5 * $baseRatePerKg * 0.9) + (($weight - 10) * $baseRatePerKg * 0.8),
    };

    // ค่าบวกเพิ่มตามประเภท
    $typeSurcharge = match($type) {
        'standard'  => 0,
        'express'   => $weightFee * 0.5,   // +50%
        'overnight' => $weightFee * 1.0,   // +100%
    };

    // ค่าประกัน
    $insuranceFee = $insurance ? 50 : 0;

    // ยอดรวม
    $subtotal = $weightFee + $typeSurcharge;
    $total    = $subtotal + $insuranceFee;

    // เวลาจัดส่ง
    $deliveryDays = match($type) {
        'overnight' => "1 วันทำการ",
        'express'   => match(strtolower($zone)) {
            'bangkok' => "1-2 วันทำการ",
            default   => "2-3 วันทำการ",
        },
        'standard' => match(strtolower($zone)) {
            'bangkok'   => "2-3 วันทำการ",
            'central'   => "3-4 วันทำการ",
            default     => "4-7 วันทำการ",
        },
    };

    return [
        'weight'         => $weight,
        'zone'           => $zone,
        'type'           => $type,
        'insurance'      => $insurance,
        'weight_fee'     => round($weightFee, 2),
        'type_surcharge' => round($typeSurcharge, 2),
        'insurance_fee'  => $insuranceFee,
        'subtotal'       => round($subtotal, 2),
        'total'          => round($total, 2),
        'delivery_days'  => $deliveryDays,
    ];
}

/**
 * แสดงผลใบเสร็จค่าจัดส่ง
 */
function printShippingReceipt(array $result): void
{
    if (isset($result['error'])) {
        echo "❌ ข้อผิดพลาด: {$result['error']}\n";
        return;
    }

    $typeName = match($result['type']) {
        'standard'  => 'มาตรฐาน',
        'express'   => 'ด่วนพิเศษ',
        'overnight' => 'ถึงวันรุ่งขึ้น',
    };

    echo "========================================\n";
    echo "       ใบเสร็จค่าจัดส่ง\n";
    echo "========================================\n";
    echo "น้ำหนัก      : {$result['weight']} กก.\n";
    echo "ปลายทาง     : {$result['zone']}\n";
    echo "ประเภท       : $typeName\n";
    echo "ประกัน       : " . ($result['insurance'] ? "มี" : "ไม่มี") . "\n";
    echo "----------------------------------------\n";
    echo "ค่าน้ำหนัก   : " . number_format($result['weight_fee'], 2) . " บาท\n";

    if ($result['type_surcharge'] > 0) {
        echo "ค่าบริการเร่งด่วน: " . number_format($result['type_surcharge'], 2) . " บาท\n";
    }

    if ($result['insurance_fee'] > 0) {
        echo "ค่าประกัน    : " . number_format($result['insurance_fee'], 2) . " บาท\n";
    }

    echo "----------------------------------------\n";
    echo "รวมทั้งหมด   : " . number_format($result['total'], 2) . " บาท\n";
    echo "ระยะเวลาจัดส่ง: {$result['delivery_days']}\n";
    echo "========================================\n\n";
}

// ===== ทดสอบระบบ =====

echo "=== ทดสอบคำนวณค่าจัดส่ง ===\n\n";

// ทดสอบ 1: ส่งมาตรฐาน กรุงเทพ
$r1 = calculateShipping(2.5, 'bangkok', 'standard');
printShippingReceipt($r1);

// ทดสอบ 2: ส่งด่วน เหนือ พร้อมประกัน
$r2 = calculateShipping(5, 'north', 'express', true);
printShippingReceipt($r2);

// ทดสอบ 3: ส่งถึงพรุ่งนี้ ใต้
$r3 = calculateShipping(1, 'south', 'overnight');
printShippingReceipt($r3);

// ทดสอบ 4: Error cases
echo "=== ทดสอบ error cases ===\n";
printShippingReceipt(calculateShipping(-1, 'bangkok'));
printShippingReceipt(calculateShipping(35, 'bangkok'));
printShippingReceipt(calculateShipping(5, 'invalid_zone'));
```

---

## ❓ Quiz

### คำถามที่ 1
โค้ดต่อไปนี้จะแสดงผลอะไร?

```php
<?php
$x = 0;
switch ($x) {
    case false:
        echo "A";
        break;
    case 0:
        echo "B";
        break;
    case null:
        echo "C";
        break;
    default:
        echo "D";
}
```

**ตัวเลือก:**
- a) A
- b) B
- c) C
- d) D

---

### คำถามที่ 2
โค้ดต่อไปนี้จะทำงานอย่างไร?

```php
<?php
$a = null;
$b = "";
$c = "hello";

echo $a ?? $b ?? $c;
```

**ตัวเลือก:**
- a) null
- b) "" (empty string)
- c) hello
- d) Error

---

### คำถามที่ 3
Guard Clauses ช่วยแก้ปัญหาอะไร?

**ตัวเลือก:**
- a) ทำให้โค้ดทำงานเร็วขึ้น
- b) ลด nesting ของ if ซ้อนกัน ทำให้อ่านง่าย
- c) ป้องกัน SQL Injection
- d) ลดการใช้หน่วยความจำ

---

### คำถามที่ 4
`match` ต่างจาก `switch` อย่างไร? (เลือกทุกข้อที่ถูก)

- a) match ใช้ === (strict comparison)
- b) match โยน error ถ้าไม่มี arm ที่ตรง
- c) match ส่งคืนค่าได้
- d) match รองรับ fall-through เหมือน switch

---

## ✅ เฉลย Quiz

**ข้อ 1: a) A**
> `switch` ใช้ loose comparison (==) ดังนั้น `0 == false` เป็น true จะ match กับ `case false:` ก่อน

**ข้อ 2: b) "" (empty string)**
> `??` ตรวจว่าเป็น null หรือเปล่า
> - `$a` เป็น null → ข้ามไป
> - `$b` เป็น "" (ไม่ใช่ null) → ใช้ค่านี้
> ผลลัพธ์คือ "" (empty string)
> หมายเหตุ: `??` ต่างจาก `?:` ตรงที่ `??` ตรวจเฉพาะ null ส่วน `?:` ตรวจ falsy

**ข้อ 3: b) ลด nesting ของ if ซ้อนกัน ทำให้อ่านง่าย**
> Guard Clauses ช่วยลด Arrow Code (Deep Nesting) โดยการ return ออกเร็วเมื่อพบเงื่อนไขผิดพลาด

**ข้อ 4: a, b, c**
> - a ✅ match ใช้ === (strict comparison)
> - b ✅ match โยน `UnhandledMatchError` ถ้าไม่มี arm ที่ตรง
> - c ✅ match เป็น expression ที่ส่งคืนค่าได้
> - d ❌ match ไม่รองรับ fall-through (แต่ละ arm ต้องแยกกัน)

---

## 🔗 สรุป

| Concept | เมื่อใช้ |
|---------|---------|
| `if/elseif/else` | เงื่อนไขทั่วไป หลายเงื่อนไขที่ซับซ้อน |
| `switch/case` | เปรียบเทียบตัวแปรเดียวกับหลายค่า (PHP < 8.0) |
| `match` | เหมือน switch แต่ strict และส่งคืนค่าได้ (PHP 8.0+) |
| `?:` (ternary) | เลือกค่าสั้นๆ จาก 2 ตัวเลือก |
| `??` (null coalescing) | ค่า default เมื่อเป็น null |
| Guard Clauses | ลด nesting, validate inputs ต้นฟังก์ชัน |

---

## ➡️ Part ถัดไป

👉 **[Part 005: PHP Loops — การวนซ้ำ](./part-005-php-loops.md)**

ใน Part หน้า เราจะเรียนรู้:
- `for`, `while`, `do-while`, `foreach` loops
- `break`, `continue`, `return` ใน loops
- Nested loops และ performance tips
- Workshop: FizzBuzz, Fibonacci, Matrix operations
