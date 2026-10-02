# Part 002: PHP Syntax พื้นฐาน - Variables & Data Types
## ระดับ: พื้นฐาน | ขั้นตอนที่ 21-50

---

## 🎯 เป้าหมายของ Part นี้

เมื่อเรียนจบ Part นี้คุณจะสามารถ:
- เข้าใจและใช้งาน Variables ใน PHP
- รู้จัก Data Types ทั้งหมดใน PHP
- ใช้ Type Casting และ Type Juggling
- สร้างและใช้งาน Constants
- เข้าใจ Variable Scope

---

## 📖 เนื้อหา

### 1. Variables ใน PHP

Variable คือ container สำหรับเก็บข้อมูล ใน PHP เริ่มต้นด้วยเครื่องหมาย `$`

#### กฎการตั้งชื่อ Variable:
```php
<?php
// ✅ ถูกต้อง
$name = "สมชาย";
$age = 25;
$_privateVar = "private";
$camelCase = "ใช้บ่อยใน PHP";
$snake_case = "ก็ใช้ได้เช่นกัน";
$userAge = 18;
$user123 = "มีตัวเลขได้";

// ❌ ผิด
// $123abc = "ขึ้นต้นด้วยตัวเลขไม่ได้";
// $my-var = "มีขีดกลางไม่ได้";
// $my var = "มีช่องว่างไม่ได้";

// PHP เป็น case-sensitive สำหรับ variables
$Name = "สมหญิง";  // ต่างจาก $name
$NAME = "สมศักดิ์"; // ต่างจากทั้ง $name และ $Name

echo $name . "<br>";   // สมชาย
echo $Name . "<br>";   // สมหญิง
echo $NAME . "<br>";   // สมศักดิ์
```

#### PHP Naming Conventions:
```php
<?php
// camelCase - สำหรับ local variables และ methods
$firstName = "สมชาย";
$lastName = "ใจดี";
$totalPrice = 1500.50;

// PascalCase - สำหรับ Classes
// class UserController { ... }

// UPPER_CASE - สำหรับ Constants
define('MAX_SIZE', 100);
const DB_HOST = 'localhost';

// snake_case - ก็ใช้ได้ใน PHP (บางทีม)
$first_name = "สมชาย";

// แนะนำตาม PSR-12 Standard
// Variables & Functions: camelCase
// Classes: PascalCase  
// Constants: UPPER_CASE
```

---

### 2. Data Types ใน PHP

PHP มี 8 primitive types และ 2 special types

```
Scalar Types:
├── boolean (bool)
├── integer (int)
├── float (double)
└── string

Compound Types:
├── array
└── object

Special Types:
├── null
└── resource
```

#### 2.1 Boolean

```php
<?php
$isTrue = true;
$isFalse = false;

// Boolean เป็น case-insensitive
$a = TRUE;
$b = True;
$c = true; // ทั้งหมดเท่ากัน

// ค่าที่ถือว่าเป็น false (falsy values)
$falsy_values = [
    false,          // false ตรงๆ
    0,              // integer 0
    0.0,            // float 0.0
    "",             // empty string
    "0",            // string "0"
    [],             // empty array
    null,           // null
];

// ค่าที่ถือว่าเป็น true (truthy values)
$truthy_values = [
    true,
    1,
    -1,             // ตัวเลขที่ไม่ใช่ 0
    "hello",
    "false",        // string ที่ไม่ใช่ "" หรือ "0"
    [0],            // array ที่มีค่า
];

// การตรวจสอบ boolean
var_dump($isTrue);   // bool(true)
var_dump($isFalse);  // bool(false)

// การแปลงเป็น boolean
var_dump((bool) 0);      // bool(false)
var_dump((bool) 1);      // bool(true)
var_dump((bool) "");     // bool(false)
var_dump((bool) "hello"); // bool(true)
```

#### 2.2 Integer

```php
<?php
// Integer - จำนวนเต็ม
$decimal = 42;           // ทศนิยม (Decimal)
$negative = -17;         // ลบ
$hex = 0x1A;             // เลขฐาน 16 (Hexadecimal) = 26
$octal = 0777;           // เลขฐาน 8 (Octal) = 511
$binary = 0b1111011;     // เลขฐาน 2 (Binary) = 123

echo $decimal . "<br>"; // 42
echo $hex . "<br>";      // 26
echo $octal . "<br>";    // 511
echo $binary . "<br>";   // 123

// PHP_INT_MAX - ค่า integer สูงสุด
echo PHP_INT_MAX . "<br>";  // 9223372036854775807 (64-bit)
echo PHP_INT_MIN . "<br>";  // -9223372036854775808
echo PHP_INT_SIZE . "<br>"; // 8 (bytes)

// Underscore separator (PHP 7.4+)
$million = 1_000_000;      // อ่านง่ายกว่า 1000000
$hex2 = 0xFF_CC_00;        // Color code

// ตรวจสอบว่าเป็น integer
var_dump(is_int($decimal));   // bool(true)
var_dump(is_integer(42.5));   // bool(false)

// ถ้าเกิน PHP_INT_MAX จะแปลงเป็น float อัตโนมัติ
$big = PHP_INT_MAX + 1;
var_dump($big); // float(9.2233720368548E+18)
```

#### 2.3 Float

```php
<?php
// Float (Double) - ทศนิยม
$price = 99.99;
$pi = 3.14159265358979;
$scientific = 1.2e3;    // 1200
$small = 7E-10;          // 0.0000000007

echo $price . "<br>";       // 99.99
echo $scientific . "<br>"; // 1200
echo $small . "<br>";       // 7.0E-10

// ความแม่นยำ Float
$a = 0.1 + 0.2;
echo $a . "<br>";              // 0.3 (แต่จริงๆ คือ 0.30000000000000004)
var_dump($a == 0.3);           // bool(false) !! ระวัง !!

// วิธีที่ถูกต้องในการเปรียบเทียบ float
$epsilon = PHP_FLOAT_EPSILON;
var_dump(abs($a - 0.3) < $epsilon); // bool(true)

// หรือใช้ round()
echo round($a, 1) == 0.3 ? "equal" : "not equal"; // equal

// PHP Float constants
echo PHP_FLOAT_MAX . "<br>";     // ค่า float สูงสุด
echo PHP_FLOAT_MIN . "<br>";     // ค่า float บวกที่เล็กสุด
echo PHP_FLOAT_EPSILON . "<br>"; // ค่าต่างเล็กสุด

// ค่าพิเศษ
$inf = INF;
$neg_inf = -INF;
$nan = NAN;

var_dump(is_infinite($inf));   // bool(true)
var_dump(is_nan($nan));        // bool(true)
var_dump(is_finite(99.9));     // bool(true)
```

#### 2.4 String

```php
<?php
// String - ข้อความ

// Single quotes - ข้อความตามตัว ไม่ parse variables
$name = "สมชาย";
$singleQuote = 'สวัสดี $name';  // แสดง: สวัสดี $name
echo $singleQuote . "<br>";

// Double quotes - parse variables และ escape sequences
$doubleQuote = "สวัสดี $name";  // แสดง: สวัสดี สมชาย
echo $doubleQuote . "<br>";

// Complex variables ใน string
$user = ['name' => 'สมชาย', 'age' => 25];
echo "ชื่อ: {$user['name']}, อายุ: {$user['age']}<br>";

// Escape Sequences
$escape = "บรรทัดใหม่\nTabulator\tBackslash\\Double quote\"";
echo nl2br($escape) . "<br>"; // nl2br แปลง \n เป็น <br>

// Heredoc - สำหรับข้อความหลายบรรทัด
$heredoc = <<<EOT
    นี่คือ Heredoc
    สามารถมี $name ข้างใน
    และมีหลายบรรทัดได้
    ไม่ต้องใส่ quotes
EOT;
echo $heredoc . "<br>";

// Nowdoc - เหมือน single quote แต่หลายบรรทัด
$nowdoc = <<<'EOT'
    นี่คือ Nowdoc
    ไม่ parse $name
    ข้อความตามตัว
EOT;
echo $nowdoc . "<br>";

// String Functions ที่ใช้บ่อย
$str = "Hello, World!";
echo strlen($str) . "<br>";          // 13 - ความยาว
echo strtoupper($str) . "<br>";      // HELLO, WORLD!
echo strtolower($str) . "<br>";      // hello, world!
echo str_replace("World", "PHP", $str) . "<br>"; // Hello, PHP!
echo substr($str, 0, 5) . "<br>";    // Hello
echo strpos($str, "World") . "<br>"; // 7
echo trim("  hello  ") . "<br>";     // hello
echo str_repeat("PHP ", 3) . "<br>"; // PHP PHP PHP
echo implode(", ", ["a", "b", "c"]) . "<br>"; // a, b, c

// String interpolation แบบต่างๆ
$product = "iPhone";
$price = 45000;

// วิธี 1: Concatenation
echo $product . " ราคา " . $price . " บาท<br>";

// วิธี 2: Double quote interpolation
echo "$product ราคา $price บาท<br>";

// วิธี 3: sprintf/printf
echo sprintf("%s ราคา %d บาท<br>", $product, $price);

// วิธี 4: number_format
echo number_format($price, 2, '.', ',') . " บาท<br>"; // 45,000.00 บาท
```

#### 2.5 Array

```php
<?php
// Array - เก็บข้อมูลหลายค่า

// Indexed Array
$fruits = ["มะม่วง", "กล้วย", "ส้ม", "แอปเปิ้ล"];
echo $fruits[0] . "<br>"; // มะม่วง
echo $fruits[3] . "<br>"; // แอปเปิ้ล

// Alternative syntax (เก่า)
$vegetables = array("แครอท", "บร็อคโคลี", "ผักชี");

// Associative Array (key => value)
$person = [
    'name' => 'สมชาย',
    'age' => 25,
    'email' => 'somchai@example.com',
    'city' => 'กรุงเทพ'
];

echo $person['name'] . "<br>";  // สมชาย
echo $person['age'] . "<br>";   // 25

// Multidimensional Array
$students = [
    ['name' => 'สมชาย', 'score' => 85],
    ['name' => 'สมหญิง', 'score' => 92],
    ['name' => 'สมศักดิ์', 'score' => 78],
];

foreach ($students as $student) {
    echo $student['name'] . ": " . $student['score'] . "<br>";
}

// Array Functions ที่ใช้บ่อย
$numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3];

echo count($numbers) . "<br>";           // 10
sort($numbers);                          // เรียงน้อยไปมาก
echo implode(", ", $numbers) . "<br>";

$unique = array_unique($numbers);        // ลบซ้ำ
echo implode(", ", $unique) . "<br>";

$sum = array_sum($numbers);             // รวมทั้งหมด
echo $sum . "<br>";

$slice = array_slice($numbers, 0, 3);   // ตัด 3 ค่าแรก
echo implode(", ", $slice) . "<br>";

// array_map - ทำกับทุก element
$doubled = array_map(fn($n) => $n * 2, [1, 2, 3, 4, 5]);
echo implode(", ", $doubled) . "<br>"; // 2, 4, 6, 8, 10

// array_filter - กรอง elements
$evens = array_filter([1, 2, 3, 4, 5, 6], fn($n) => $n % 2 === 0);
echo implode(", ", $evens) . "<br>"; // 2, 4, 6

// array_reduce - รวมเป็นค่าเดียว
$total = array_reduce([1, 2, 3, 4, 5], fn($carry, $item) => $carry + $item, 0);
echo $total . "<br>"; // 15
```

#### 2.6 Object

```php
<?php
// Object - instance ของ Class

class Car {
    public string $brand;
    public string $model;
    public int $year;
    
    public function __construct(string $brand, string $model, int $year)
    {
        $this->brand = $brand;
        $this->model = $model;
        $this->year = $year;
    }
    
    public function getInfo(): string
    {
        return "{$this->year} {$this->brand} {$this->model}";
    }
}

$myCar = new Car("Toyota", "Camry", 2023);
echo $myCar->getInfo() . "<br>"; // 2023 Toyota Camry

// stdClass - Generic Object
$obj = new stdClass();
$obj->name = "สมชาย";
$obj->age = 25;
echo $obj->name . "<br>"; // สมชาย

// แปลง array เป็น object
$data = ['name' => 'สมหญิง', 'age' => 22];
$obj2 = (object) $data;
echo $obj2->name . "<br>"; // สมหญิง

// แปลง object เป็น array
$arr = (array) $obj2;
echo $arr['name'] . "<br>"; // สมหญิง
```

#### 2.7 NULL

```php
<?php
// NULL - ไม่มีค่า

$nothing = null;
$undefined; // ก็เป็น null แต่จะเกิด notice

// isset() - ตรวจสอบว่ากำหนดค่าแล้วและไม่ใช่ null
var_dump(isset($nothing));   // bool(false) - เพราะ null
var_dump(isset($name));      // bool(true) - ถ้ากำหนดค่าแล้ว

// is_null() - ตรวจสอบว่าเป็น null
var_dump(is_null($nothing)); // bool(true)
var_dump(is_null(0));        // bool(false)
var_dump(is_null(""));       // bool(false)

// Null Coalescing Operator (??)
$username = $_GET['user'] ?? 'Guest';  // ถ้า $_GET['user'] ไม่มีหรือ null ใช้ 'Guest'

// Nullsafe Operator (?->)  PHP 8.0+
$user = null;
$city = $user?->address?->city; // ไม่ error แม้ $user เป็น null - คืน null
echo $city ?? "ไม่ระบุ"; // ไม่ระบุ
```

#### 2.8 Resource

```php
<?php
// Resource - reference ไปยัง external resource
// ใช้กับ file handle, database connection, image, etc.

// File resource
$file = fopen("test.txt", "w");
var_dump(get_resource_type($file)); // string(6) "stream"

// ต้องปิด resource เมื่อเสร็จแล้ว
fwrite($file, "Hello World");
fclose($file);
var_dump(is_resource($file)); // bool(false) หลัง close

// Database resource (MySQLi แบบเก่า)
// $conn = mysql_connect(...); // deprecated แล้ว
// ปัจจุบันใช้ PDO แทน
```

---

### 3. Type Juggling (การแปลง Type อัตโนมัติ)

PHP เป็น loosely typed language - แปลง type อัตโนมัติตามบริบท

```php
<?php
// PHP แปลง type อัตโนมัติ
$result = "10" + 5;   // string + int = int: 15
echo gettype($result); // integer
echo $result;         // 15

// String ที่มีตัวเลขข้างหน้า
$result2 = "10 apples" + 5; // 15 (อ่านแค่ตัวเลขข้างหน้า)
echo $result2;              // 15

// Comparison แบบ loose (==)
var_dump(0 == "a");   // bool(true) ใน PHP 7, bool(false) ใน PHP 8 !!!
var_dump(0 == "");    // bool(true) ใน PHP 7, bool(false) ใน PHP 8
var_dump("1" == 1);   // bool(true)
var_dump("01" == 1);  // bool(true)
var_dump("10" == "1e1"); // bool(true) !! ระวัง !!
var_dump(100 == "1e2");  // bool(true) !! ระวัง !!

// Comparison แบบ strict (===) - ตรวจสอบทั้งค่าและ type
var_dump("1" === 1);   // bool(false) - ต่าง type
var_dump(1 === 1);     // bool(true)
var_dump(0 === false); // bool(false) - ต่าง type

// ⚠️ แนะนำให้ใช้ === เสมอ เพื่อความปลอดภัย

// Boolean conversion ที่ควรระวัง
var_dump((bool) "false"); // bool(true) !! "false" เป็น string ที่ไม่ว่างเปล่า !!
var_dump((bool) "0");     // bool(false)
var_dump((bool) "");      // bool(false)
var_dump((bool) []);      // bool(false) - array ว่าง
var_dump((bool) [0]);     // bool(true) - array ที่มีค่า

// PHP 8 เปลี่ยนพฤติกรรมบางอย่าง
// var_dump(0 == "a"); // PHP 8: false, PHP 7: true
// แนะนำให้ upgrade เป็น PHP 8+
```

---

### 4. Type Casting (การแปลง Type แบบ explicit)

```php
<?php
// Type Casting - แปลง type อย่างชัดเจน

$str = "42.5abc";
$num = 42;
$float = 3.14;
$bool = true;

// แปลงเป็น Integer
echo (int) $str . "<br>";     // 42 - อ่านตัวเลขจนถึงอักขระที่ไม่ใช่ตัวเลข
echo (int) $float . "<br>";   // 3 - ตัดทศนิยม
echo (int) $bool . "<br>";    // 1
echo (int) "abc" . "<br>";    // 0 - ไม่มีตัวเลขเริ่มต้น

// แปลงเป็น Float
echo (float) "3.14abc" . "<br>"; // 3.14
echo (float) "abc" . "<br>";     // 0

// แปลงเป็น String
echo (string) 42 . "<br>";       // "42"
echo (string) 3.14 . "<br>";     // "3.14"
echo (string) true . "<br>";     // "1"
echo (string) false . "<br>";    // "" (empty string)
echo (string) null . "<br>";     // ""

// แปลงเป็น Boolean
echo var_export((bool) 1, true) . "<br>";     // true
echo var_export((bool) 0, true) . "<br>";     // false
echo var_export((bool) "hello", true) . "<br>"; // true
echo var_export((bool) "", true) . "<br>";    // false

// แปลงเป็น Array
$arr = (array) "hello";  // ["hello"]
$arr2 = (array) 42;      // [42]
$arr3 = (array) null;    // []

// แปลงเป็น Object
$obj = (object) ['name' => 'PHP', 'version' => 8];
echo $obj->name . "<br>"; // PHP

// Functions สำหรับแปลง type
$value = "100";
echo intval($value) . "<br>";       // 100
echo floatval("3.14") . "<br>";     // 3.14
echo strval(42) . "<br>";           // "42"
echo boolval(1) . "<br>";           // 1
echo settype($value, "integer");    // แปลง $value เป็น integer ใน place

// Type checking functions
var_dump(is_int(42));           // bool(true)
var_dump(is_float(3.14));       // bool(true)
var_dump(is_string("hello"));   // bool(true)
var_dump(is_bool(true));        // bool(true)
var_dump(is_array([]));         // bool(true)
var_dump(is_object(new stdClass())); // bool(true)
var_dump(is_null(null));        // bool(true)
var_dump(is_numeric("42"));     // bool(true) - string ที่เป็นตัวเลขก็ผ่าน
var_dump(is_numeric("42abc"));  // bool(false)
```

---

### 5. Constants

```php
<?php
// Constants - ค่าคงที่ที่ไม่เปลี่ยนแปลง

// กำหนดด้วย define()
define('APP_NAME', 'My PHP App');
define('MAX_UPLOAD_SIZE', 5 * 1024 * 1024); // 5MB
define('SUPPORTED_TYPES', ['jpg', 'png', 'gif', 'pdf']);

echo APP_NAME . "<br>";         // My PHP App
echo MAX_UPLOAD_SIZE . "<br>"; // 5242880

// กำหนดด้วย const (ใน global scope)
const VERSION = '1.0.0';
const DEBUG = true;

echo VERSION . "<br>"; // 1.0.0

// ความแตกต่าง define() vs const
// const: กำหนดได้เฉพาะ top-level scope (ไม่อยู่ใน if, function)
// define(): กำหนดได้ทุกที่รวมถึงใน conditions

if (DEBUG) {
    define('LOG_LEVEL', 'verbose');  // ✅ ได้
    // const LOG_LEVEL = 'verbose';  // ❌ Error ใน PHP 7
}

// PHP Constants ที่มีอยู่แล้ว
echo PHP_VERSION . "<br>";     // 8.3.x
echo PHP_EOL;                  // newline character
echo PHP_INT_MAX . "<br>";     // ค่า int สูงสุด
echo PHP_FLOAT_EPSILON . "<br>"; // ค่า float เล็กสุด
echo DIRECTORY_SEPARATOR . "<br>"; // / หรือ \
echo PATH_SEPARATOR . "<br>";  // : หรือ ;

// Magic Constants
echo __FILE__ . "<br>";       // path ของไฟล์นี้
echo __DIR__ . "<br>";        // directory ของไฟล์นี้
echo __LINE__ . "<br>";       // หมายเลขบรรทัดปัจจุบัน
echo __FUNCTION__ . "<br>";   // ชื่อ function ปัจจุบัน (ว่างถ้าอยู่นอก function)
echo __CLASS__ . "<br>";      // ชื่อ class ปัจจุบัน
echo __NAMESPACE__ . "<br>"; // namespace ปัจจุบัน

// Enum Constants (PHP 8.1+)
enum Status {
    case Active;
    case Inactive;
    case Pending;
}

$status = Status::Active;
echo $status->name . "<br>"; // Active

// Backed Enum
enum Color: string {
    case Red = 'red';
    case Green = 'green';
    case Blue = 'blue';
}

$color = Color::Red;
echo $color->value . "<br>"; // red
echo Color::from('green')->name . "<br>"; // Green
```

---

### 6. Variable Scope

```php
<?php
// Variable Scope - ขอบเขตของ variable

// Global scope
$globalVar = "ฉันเป็น global variable";

function testScope(): void
{
    // ภายใน function ไม่เห็น global variable
    // echo $globalVar; // Notice: Undefined variable
    
    // ต้องใช้ global keyword
    global $globalVar;
    echo $globalVar . "<br>"; // ฉันเป็น global variable
    
    // หรือใช้ $GLOBALS superglobal
    echo $GLOBALS['globalVar'] . "<br>"; // ฉันเป็น global variable
    
    // Local variable
    $localVar = "ฉันอยู่ใน function";
    echo $localVar . "<br>";
}

testScope();
// echo $localVar; // Notice: Undefined variable - ไม่เห็นนอก function

// Static Variables
function counter(): int
{
    static $count = 0; // สร้างครั้งเดียว ค่าอยู่ระหว่างการเรียก
    $count++;
    return $count;
}

echo counter() . "<br>"; // 1
echo counter() . "<br>"; // 2
echo counter() . "<br>"; // 3

// Variable Variables
$varName = 'hello';
$$varName = 'World'; // สร้าง variable ชื่อ $hello
echo $hello . "<br>"; // World
echo $$varName . "<br>"; // World (เหมือนกัน)

// Superglobals - accessible ทุกที่
// $_GET - HTTP GET parameters
// $_POST - HTTP POST data
// $_SESSION - Session data
// $_COOKIE - Cookie data
// $_SERVER - Server information
// $_ENV - Environment variables
// $_FILES - Uploaded files
// $_REQUEST - GET + POST + COOKIE
// $GLOBALS - ทุก global variables

echo $_SERVER['PHP_SELF'] . "<br>";    // ชื่อ script ปัจจุบัน
echo $_SERVER['SERVER_NAME'] . "<br>"; // ชื่อ server
echo $_SERVER['HTTP_HOST'] . "<br>";   // HTTP Host header
```

---

### 7. Type Declaration (PHP 7+)

```php
<?php
declare(strict_types=1); // บังคับ strict type checking

// Type Hints สำหรับ Function Parameters
function add(int $a, int $b): int
{
    return $a + $b;
}

echo add(5, 3) . "<br>";  // 8
// echo add(5.5, 3);      // TypeError ถ้าใช้ strict_types=1

// Nullable Types (PHP 7.1+)
function findUser(?int $id): ?string
{
    if ($id === null) {
        return null;
    }
    return "User #$id";
}

echo findUser(5) . "<br>";   // User #5
echo findUser(null) ?? "ไม่พบ"; // ไม่พบ

// Union Types (PHP 8.0+)
function processInput(int|string $input): string
{
    if (is_int($input)) {
        return "Number: $input";
    }
    return "String: $input";
}

echo processInput(42) . "<br>";     // Number: 42
echo processInput("hello") . "<br>"; // String: hello

// Intersection Types (PHP 8.1+)
interface Countable2 { public function count(): int; }
interface Stringable2 { public function __toString(): string; }

// function process(Countable2&Stringable2 $obj): void { ... }

// mixed type - รับทุก type
function anything(mixed $value): mixed
{
    return $value;
}

// never type (PHP 8.1+) - function ไม่ return
function throwError(string $message): never
{
    throw new RuntimeException($message);
}

// Readonly Properties (PHP 8.1+)
class User {
    public function __construct(
        public readonly int $id,
        public readonly string $name,
        public string $email, // ไม่ readonly - เปลี่ยนได้
    ) {}
}

$user = new User(1, "สมชาย", "somchai@example.com");
echo $user->name . "<br>"; // สมชาย
$user->email = "new@example.com"; // ✅ OK
// $user->name = "other"; // ❌ Error - readonly

// Fibers (PHP 8.1+) - cooperative multitasking
$fiber = new Fiber(function(): void {
    $value = Fiber::suspend('first');
    echo "Resumed with: $value\n";
});

$value = $fiber->start();
echo $value . "\n"; // first
$fiber->resume('hello');
```

---

### 8. String Functions เพิ่มเติม

```php
<?php
// String functions ที่ใช้บ่อย

$text = "  Hello, World! PHP is awesome!  ";

// Trimming
echo trim($text) . "<br>";       // "Hello, World! PHP is awesome!"
echo ltrim($text) . "<br>";      // "Hello, World! PHP is awesome!  "
echo rtrim($text) . "<br>";      // "  Hello, World! PHP is awesome!"

// Case
$str = "hello world";
echo ucfirst($str) . "<br>";     // Hello world
echo ucwords($str) . "<br>";     // Hello World
echo strtoupper($str) . "<br>";  // HELLO WORLD

// Padding
echo str_pad("42", 5, "0", STR_PAD_LEFT) . "<br>";  // 00042
echo str_pad("Hi", 10, "-", STR_PAD_BOTH) . "<br>"; // ----Hi----

// Search & Replace
$html = "<p>Hello <b>World</b></p>";
echo strip_tags($html) . "<br>";         // Hello World
echo htmlspecialchars("<script>alert('xss')</script>") . "<br>"; // Escaped
echo htmlspecialchars_decode("&lt;b&gt;bold&lt;/b&gt;") . "<br>"; // <b>bold</b>

// Splitting
$csv = "John,Jane,Bob,Alice";
$names = explode(",", $csv);
print_r($names); // Array ( [0] => John [1] => Jane ... )

// Joining
$joined = implode(" | ", $names);
echo $joined . "<br>"; // John | Jane | Bob | Alice

// Checking
echo str_contains("Hello World", "World") ? "yes" : "no"; // yes (PHP 8)
echo str_starts_with("Hello World", "Hello") ? "yes" : "no"; // yes (PHP 8)
echo str_ends_with("Hello World", "World") ? "yes" : "no"; // yes (PHP 8)

// Formatting
printf("%.2f บาท<br>", 1234.5);     // 1234.50 บาท
printf("%05d<br>", 42);              // 00042
printf("%-10s|<br>", "left");        // left      |
printf("%10s|<br>", "right");        //      right|

// sprintf
$formatted = sprintf("Order #%06d - Total: %.2f", 42, 1234.5);
echo $formatted . "<br>"; // Order #000042 - Total: 1234.50

// Multibyte String (สำหรับภาษาไทย)
$thai = "สวัสดีครับ";
echo strlen($thai) . "<br>";     // bytes (อาจมากกว่า characters)
echo mb_strlen($thai) . "<br>";  // characters จริงๆ: 10
echo mb_strtoupper("hello") . "<br>"; // HELLO
echo mb_substr($thai, 0, 4) . "<br>"; // สวัสดี

// String to Array
$chars = str_split("Hello", 2);
print_r($chars); // ["He", "ll", "o"]

// Counting
echo substr_count("hello world hello", "hello") . "<br>"; // 2
echo str_word_count("Hello World PHP") . "<br>"; // 3
```

---

### 9. PHP 8 Type System Features

```php
<?php
declare(strict_types=1);

// Named Arguments (PHP 8.0+)
function createUser(
    string $name,
    int $age = 25,
    string $role = 'user',
    bool $active = true
): array {
    return compact('name', 'age', 'role', 'active');
}

// แบบเดิม
$user1 = createUser('สมชาย', 30, 'admin', false);

// Named Arguments - ไม่ต้องสนใจลำดับ
$user2 = createUser(
    name: 'สมหญิง',
    role: 'editor',
    age: 28,
    active: true
);

// Match Expression (PHP 8.0+) - เหมือน switch แต่ strict และ return ค่า
$status = 2;
$label = match($status) {
    1 => 'Active',
    2, 3 => 'Pending',  // หลาย conditions ได้
    4 => 'Inactive',
    default => 'Unknown'
};
echo $label . "<br>"; // Pending

// Nullsafe Operator (PHP 8.0+)
class Order {
    public function getCustomer(): ?Customer { return null; }
}
class Customer {
    public function getAddress(): ?Address { return null; }
}
class Address {
    public string $city = "Bangkok";
}

$order = new Order();
$city = $order->getCustomer()?->getAddress()?->city;
echo $city ?? "No city"; // No city

// str_contains, str_starts_with, str_ends_with (PHP 8.0+)
$haystack = "Hello World PHP";
echo str_contains($haystack, "World") ? "✅" : "❌"; // ✅
echo str_starts_with($haystack, "Hello") ? "✅" : "❌"; // ✅
echo str_ends_with($haystack, "PHP") ? "✅" : "❌"; // ✅

// First class callables (PHP 8.1+)
function double(int $x): int { return $x * 2; }
$fn = double(...); // First class callable
echo $fn(5) . "<br>"; // 10

$arr = [1, 2, 3, 4, 5];
$doubled = array_map(double(...), $arr);
echo implode(", ", $doubled) . "<br>"; // 2, 4, 6, 8, 10

// Readonly Classes (PHP 8.2+)
readonly class Point {
    public function __construct(
        public float $x,
        public float $y,
        public float $z = 0.0,
    ) {}
}

$point = new Point(1.5, 2.5);
echo $point->x . "<br>"; // 1.5
// $point->x = 5.0; // ❌ Error - readonly

// Disjunctive Normal Form (DNF) Types (PHP 8.2+)
// function process((Iterator&Countable)|string $param): void { ... }
```

---

### 10. Workshop: Data Types

#### Exercise 1: Type Conversion Calculator

```php
<?php
declare(strict_types=1);

function convertAndDisplay(mixed $value): void
{
    echo "Original: ";
    var_dump($value);
    echo "As int: " . (int)$value . "<br>";
    echo "As float: " . (float)$value . "<br>";
    echo "As string: '" . (string)$value . "'<br>";
    echo "As bool: " . var_export((bool)$value, true) . "<br>";
    echo "gettype: " . gettype($value) . "<br>";
    echo "---<br>";
}

$testValues = [0, 1, -5, 3.14, "42", "hello", "", "0", null, true, false, [], [1,2,3]];

foreach ($testValues as $val) {
    convertAndDisplay($val);
}
```

#### Exercise 2: String Manipulation

```php
<?php
// สร้าง function สำหรับ validate และ format ข้อมูล

function formatPhoneNumber(string $phone): string
{
    // ลบ characters ที่ไม่ใช่ตัวเลข
    $digits = preg_replace('/[^0-9]/', '', $phone);
    
    // ตรวจสอบความยาว
    if (strlen($digits) !== 10) {
        return "Invalid phone number";
    }
    
    // Format: 0XX-XXX-XXXX
    return substr($digits, 0, 3) . '-' . 
           substr($digits, 3, 3) . '-' . 
           substr($digits, 6, 4);
}

$phones = ['0812345678', '081-234-5678', '(081) 234-5678', '0991234'];
foreach ($phones as $phone) {
    echo formatPhoneNumber($phone) . "<br>";
}
// 081-234-567
// 081-234-567
// 081-234-567
// Invalid phone number

function validateEmail(string $email): bool
{
    return filter_var($email, FILTER_VALIDATE_EMAIL) !== false;
}

$emails = ['valid@example.com', 'invalid@', '@example.com', 'test@test.co.th'];
foreach ($emails as $email) {
    $valid = validateEmail($email) ? "✅ Valid" : "❌ Invalid";
    echo "$email: $valid<br>";
}
```

#### Exercise 3: Array Manipulation

```php
<?php
declare(strict_types=1);

// สร้างระบบจัดการคะแนนนักเรียน
$scores = [
    ['name' => 'สมชาย', 'math' => 85, 'science' => 90, 'english' => 75],
    ['name' => 'สมหญิง', 'math' => 92, 'science' => 88, 'english' => 95],
    ['name' => 'สมศักดิ์', 'math' => 70, 'science' => 75, 'english' => 80],
    ['name' => 'มาลี', 'math' => 95, 'science' => 92, 'english' => 88],
    ['name' => 'วิชัย', 'math' => 60, 'science' => 65, 'english' => 70],
];

// คำนวณ average ของแต่ละคน
$students = array_map(function($student) {
    $subjects = ['math', 'science', 'english'];
    $total = array_sum(array_map(fn($s) => $student[$s], $subjects));
    $avg = $total / count($subjects);
    
    return array_merge($student, [
        'total' => $total,
        'average' => round($avg, 2),
        'grade' => match(true) {
            $avg >= 90 => 'A',
            $avg >= 80 => 'B',
            $avg >= 70 => 'C',
            $avg >= 60 => 'D',
            default => 'F'
        }
    ]);
}, $scores);

// เรียงตาม average
usort($students, fn($a, $b) => $b['average'] <=> $a['average']);

// แสดงผล
echo "<table border='1'>";
echo "<tr><th>อันดับ</th><th>ชื่อ</th><th>คณิต</th><th>วิทย์</th><th>อังกฤษ</th><th>เฉลี่ย</th><th>เกรด</th></tr>";

foreach ($students as $rank => $student) {
    echo "<tr>";
    echo "<td>" . ($rank + 1) . "</td>";
    echo "<td>{$student['name']}</td>";
    echo "<td>{$student['math']}</td>";
    echo "<td>{$student['science']}</td>";
    echo "<td>{$student['english']}</td>";
    echo "<td>{$student['average']}</td>";
    echo "<td>{$student['grade']}</td>";
    echo "</tr>";
}
echo "</table>";

// สถิติ
$averages = array_column($students, 'average');
echo "<br>สูงสุด: " . max($averages) . "<br>";
echo "ต่ำสุด: " . min($averages) . "<br>";
echo "เฉลี่ย: " . round(array_sum($averages) / count($averages), 2) . "<br>";
```

---

## 📝 Quiz

1. ความแตกต่างระหว่าง `==` และ `===` คืออะไร?
2. ทำไม `(bool) "false"` ถึงเป็น `true`?
3. `const` กับ `define()` ต่างกันอย่างไร?
4. `static` variable ในฟังก์ชันทำงานอย่างไร?
5. PHP 8.0 เพิ่ม feature อะไรเกี่ยวกับ type system?
6. `??` operator ทำงานอย่างไร?
7. ทำไม `0.1 + 0.2 !== 0.3`?

---

## ⏭️ Part ถัดไป

**Part 003: PHP Operators และ Expressions**

เราจะเรียนรู้:
- Arithmetic Operators
- Comparison Operators
- Logical Operators
- Bitwise Operators
- Assignment Operators
- String Operators
- Ternary และ Null Coalescing

---

*Part 002 | ระดับพื้นฐาน | Variables & Data Types*
