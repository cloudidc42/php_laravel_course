# Part 003: PHP Operators และ Expressions
## ระดับ: พื้นฐาน | ขั้นตอนที่ 51-80

---

## 🎯 เป้าหมายของ Part นี้

เมื่อเรียนจบ Part นี้คุณจะสามารถ:
- ใช้ Arithmetic Operators ได้ครบทุกตัว
- เปรียบเทียบค่าด้วย Comparison Operators อย่างถูกต้อง
- ใช้ Logical Operators ในเงื่อนไขซับซ้อน
- เข้าใจ Operator Precedence
- ใช้ Ternary, Null Coalescing, Spaceship Operators

---

## 📖 เนื้อหา

### 1. Arithmetic Operators

```php
<?php
$a = 10;
$b = 3;

// พื้นฐาน
echo $a + $b . "<br>";  // 13 - บวก
echo $a - $b . "<br>";  // 7  - ลบ
echo $a * $b . "<br>";  // 30 - คูณ
echo $a / $b . "<br>";  // 3.333... - หาร
echo $a % $b . "<br>";  // 1  - เศษจากการหาร (Modulo)
echo $a ** $b . "<br>"; // 1000 - ยกกำลัง (PHP 5.6+)

// Division ที่ควรระวัง
echo 10 / 3 . "<br>";   // 3.3333333333333 (float)
echo intdiv(10, 3) . "<br>"; // 3 (integer division)
echo fmod(10.5, 3.5) . "<br>"; // 0 (float modulo)

// Increment/Decrement
$x = 5;
echo $x++ . "<br>"; // 5 (แสดงค่าก่อน แล้วค่อยเพิ่ม)
echo $x . "<br>";   // 6
echo ++$x . "<br>"; // 7 (เพิ่มก่อน แล้วค่อยแสดง)
echo $x-- . "<br>"; // 7 (แสดงค่าก่อน แล้วค่อยลด)
echo $x . "<br>";   // 6
echo --$x . "<br>"; // 5 (ลดก่อน แล้วค่อยแสดง)

// ตัวอย่างการใช้งานจริง
$price = 100;
$discount = 15; // 15%
$discountAmount = $price * ($discount / 100);
$finalPrice = $price - $discountAmount;
echo "ราคาหลังลด: " . $finalPrice . " บาท<br>"; // 85 บาท

// ตัวอย่าง: คำนวณ fibonacci
function fibonacci(int $n): int
{
    if ($n <= 1) return $n;
    return fibonacci($n - 1) + fibonacci($n - 2);
}

for ($i = 0; $i <= 10; $i++) {
    echo fibonacci($i) . " ";
}
echo "<br>"; // 0 1 1 2 3 5 8 13 21 34 55

// ตัวอย่าง: คำนวณ factorial
function factorial(int $n): int
{
    if ($n <= 1) return 1;
    return $n * factorial($n - 1);
}

echo factorial(5) . "<br>"; // 120 (5! = 5*4*3*2*1)
echo factorial(10) . "<br>"; // 3628800
```

---

### 2. Assignment Operators

```php
<?php
$x = 10;

// Compound Assignment Operators
$x += 5;   // $x = $x + 5 = 15
echo $x . "<br>";

$x -= 3;   // $x = $x - 3 = 12
echo $x . "<br>";

$x *= 2;   // $x = $x * 2 = 24
echo $x . "<br>";

$x /= 4;   // $x = $x / 4 = 6
echo $x . "<br>";

$x **= 2;  // $x = $x ** 2 = 36
echo $x . "<br>";

$x %= 7;   // $x = $x % 7 = 1
echo $x . "<br>";

// String Assignment
$str = "Hello";
$str .= " World"; // $str = $str . " World"
echo $str . "<br>"; // Hello World

// Null Coalescing Assignment (PHP 7.4+)
$config = [];
$config['debug'] ??= false;  // กำหนดค่าเฉพาะถ้าเป็น null/ไม่มีค่า
echo $config['debug'] ? "debug on" : "debug off"; // debug off

// Multiple Assignment
$a = $b = $c = 0;
echo "$a $b $c<br>"; // 0 0 0

// Destructuring Assignment
[$first, $second, $third] = [1, 2, 3];
echo "$first $second $third<br>"; // 1 2 3

// Named destructuring
['name' => $name, 'age' => $age] = ['name' => 'สมชาย', 'age' => 25];
echo "$name is $age years old<br>"; // สมชาย is 25 years old

// list() แบบเก่า (PHP 5)
list($a, $b, $c) = [10, 20, 30];
echo "$a $b $c<br>"; // 10 20 30

// ข้ามค่าบางตัว
[, $second, , $fourth] = [1, 2, 3, 4];
echo "$second $fourth<br>"; // 2 4

// Swap variables
$x = "apple";
$y = "banana";
[$x, $y] = [$y, $x]; // swap
echo "$x $y<br>"; // banana apple
```

---

### 3. Comparison Operators

```php
<?php
// Comparison Operators - ระวัง loose vs strict

$a = 5;
$b = "5";
$c = 5;

// == Loose equality (เปรียบเทียบค่า แปลง type อัตโนมัติ)
var_dump($a == $b);   // bool(true) - 5 == "5"
var_dump($a == $c);   // bool(true)
var_dump(0 == "a");   // bool(false) PHP 8, bool(true) PHP 7 !!

// === Strict equality (เปรียบเทียบทั้งค่าและ type)
var_dump($a === $b);  // bool(false) - int vs string
var_dump($a === $c);  // bool(true) - int == int

// != หรือ <> Loose inequality
var_dump($a != $b);   // bool(false) - เพราะ 5 == "5"
var_dump($a <> "6");  // bool(true)

// !== Strict inequality
var_dump($a !== $b);  // bool(true) - ต่าง type

// Comparison
var_dump(5 > 3);    // bool(true)
var_dump(5 >= 5);   // bool(true)
var_dump(3 < 5);    // bool(true)
var_dump(5 <= 5);   // bool(true)

// Spaceship Operator <=> (PHP 7+) - สำหรับ sorting
echo (1 <=> 2) . "<br>";   // -1 (น้อยกว่า)
echo (2 <=> 2) . "<br>";   // 0  (เท่ากัน)
echo (3 <=> 2) . "<br>";   // 1  (มากกว่า)
echo ("a" <=> "b") . "<br>"; // -1

// ใช้ <=> กับ usort
$names = ["Charlie", "Alice", "Bob"];
usort($names, fn($a, $b) => $a <=> $b);
echo implode(", ", $names) . "<br>"; // Alice, Bob, Charlie

// Comparison ที่น่าสับสน !! อ่านให้ดี !!
// PHP 8 เปลี่ยนพฤติกรรมหลายอย่าง

// PHP 8: ตัวเลขกับ non-numeric string
var_dump(0 == "foo");    // PHP 8: false (PHP 7: true !!)
var_dump(0 == "");       // PHP 8: false (PHP 7: true !!)
var_dump(42 == "42");    // true ทั้ง PHP 7 และ 8
var_dump(42 == "42abc"); // PHP 8: false (PHP 7: true !!)

// Comparison ระหว่าง null
var_dump(null == false);  // true
var_dump(null === false); // false
var_dump(null == 0);      // true
var_dump(null < 1);       // true
var_dump(null > -1);      // false ... ??? ระวัง!

// Comparison ตาราง (เฉพาะที่ควรจำ)
// null == 0 == "" == false (loose)
// แต่ null !== 0 !== "" !== false (strict)
echo "<br>";
echo "=== ควรใช้เสมอเพื่อความปลอดภัย ===<br>";
```

---

### 4. Logical Operators

```php
<?php
// Logical Operators

// AND operators
var_dump(true && true);   // bool(true)
var_dump(true && false);  // bool(false)
var_dump(false && true);  // bool(false)
var_dump(false && false); // bool(false)

// OR operators
var_dump(true || true);   // bool(true)
var_dump(true || false);  // bool(true)
var_dump(false || true);  // bool(true)
var_dump(false || false); // bool(false)

// NOT operator
var_dump(!true);  // bool(false)
var_dump(!false); // bool(true)
var_dump(!0);     // bool(true)
var_dump(!"");    // bool(true)
var_dump(!"abc"); // bool(false)

// XOR operator (exclusive or)
var_dump(true xor true);   // bool(false)
var_dump(true xor false);  // bool(true)
var_dump(false xor false); // bool(false)

// Short-circuit evaluation
function sideEffect(string $msg): bool
{
    echo "Checking: $msg<br>";
    return true;
}

// && หยุดเช็คเมื่อพบ false
$result = false && sideEffect("สิ่งนี้จะไม่ทำงาน");
// sideEffect ไม่ถูกเรียก เพราะ false && anything = false

// || หยุดเช็คเมื่อพบ true  
$result = true || sideEffect("สิ่งนี้จะไม่ทำงาน");
// sideEffect ไม่ถูกเรียก เพราะ true || anything = true

// ความแตกต่างระหว่าง && กับ and, || กับ or (ลำดับความสำคัญ!)
$a = true;
$b = false;

// && มี precedence สูงกว่า =
$result1 = $a && $b;    // $result1 = (true && false) = false ✅

// and มี precedence ต่ำกว่า =
$result2 = $a and $b;   // ($result2 = $a) and $b = true !! ระวัง !!
var_dump($result1); // bool(false)
var_dump($result2); // bool(true) !! ไม่ตรงกับที่คาดไว้

// แนะนำ: ใช้ && และ || เสมอ (ไม่ใช้ and, or)

// ตัวอย่างการใช้งานจริง
function isValidAge(int $age): bool
{
    return $age >= 0 && $age <= 150;
}

function canVote(int $age, string $nationality): bool
{
    return isValidAge($age) && $age >= 18 && $nationality === 'Thai';
}

echo canVote(25, 'Thai') ? "สามารถลงคะแนนได้" : "ไม่สามารถลงคะแนนได้";
echo "<br>";
echo canVote(16, 'Thai') ? "สามารถลงคะแนนได้" : "ไม่สามารถลงคะแนนได้";
echo "<br>";

// Null coalescing as logical
$username = $_GET['user'] ?? $_SESSION['user'] ?? 'Guest';
echo "Username: $username<br>";
```

---

### 5. String Operators

```php
<?php
// String Operators

// Concatenation (.)
$firstName = "สมชาย";
$lastName = "ใจดี";
$fullName = $firstName . " " . $lastName;
echo $fullName . "<br>"; // สมชาย ใจดี

// Concatenation Assignment (.=)
$greeting = "สวัสดี";
$greeting .= " ครับ/ค่ะ";
echo $greeting . "<br>"; // สวัสดี ครับ/ค่ะ

// String building ประสิทธิภาพสูง
// ไม่ดี (สร้าง string ใหม่ทุกครั้ง)
$result = "";
for ($i = 0; $i < 1000; $i++) {
    $result .= $i . ", "; // ช้า
}

// ดีกว่า (ใช้ array แล้ว implode ตอนสุดท้าย)
$parts = [];
for ($i = 0; $i < 1000; $i++) {
    $parts[] = $i;
}
$result = implode(", ", $parts); // เร็วกว่า

// Heredoc for multiline
$html = <<<HTML
    <div class="card">
        <h2>$fullName</h2>
        <p>ID: {$_SERVER['REMOTE_ADDR']}</p>
    </div>
HTML;
echo $html;

// String comparison
echo strcmp("apple", "banana") . "<br>";  // < 0 (apple มาก่อน)
echo strcmp("banana", "apple") . "<br>"; // > 0
echo strcmp("apple", "apple") . "<br>";  // 0

// Case-insensitive comparison
echo strcasecmp("Hello", "hello") . "<br>"; // 0
echo strcasecmp("Hello", "HELLO") . "<br>"; // 0
```

---

### 6. Bitwise Operators

```php
<?php
// Bitwise Operators - ทำงานกับ bits โดยตรง

$a = 0b1010; // 10 ในทศนิยม
$b = 0b1100; // 12 ในทศนิยม

// AND (&) - 1 เมื่อทั้งคู่เป็น 1
echo ($a & $b) . "<br>";  // 8 (0b1000)

// OR (|) - 1 เมื่ออย่างน้อยหนึ่งตัวเป็น 1
echo ($a | $b) . "<br>";  // 14 (0b1110)

// XOR (^) - 1 เมื่อต่างกัน
echo ($a ^ $b) . "<br>";  // 6 (0b0110)

// NOT (~) - กลับทุก bit
echo (~$a) . "<br>";       // -11

// Left shift (<<) - เลื่อน bits ไปซ้าย
echo ($a << 1) . "<br>";   // 20 (10 * 2^1)
echo ($a << 2) . "<br>";   // 40 (10 * 2^2)

// Right shift (>>) - เลื่อน bits ไปขวา
echo ($a >> 1) . "<br>";   // 5 (10 / 2^1)
echo ($a >> 2) . "<br>";   // 2 (10 / 2^2)

// ตัวอย่างการใช้งานจริง: Permission flags
const PERM_READ    = 0b0001; // 1
const PERM_WRITE   = 0b0010; // 2
const PERM_EXECUTE = 0b0100; // 4
const PERM_ADMIN   = 0b1000; // 8

// กำหนด permission
$userPermissions = PERM_READ | PERM_WRITE; // 3

// ตรวจสอบ permission
function hasPermission(int $userPerms, int $requiredPerm): bool
{
    return ($userPerms & $requiredPerm) === $requiredPerm;
}

echo hasPermission($userPermissions, PERM_READ) ? "✅ Read" : "❌ Read";
echo "<br>";
echo hasPermission($userPermissions, PERM_WRITE) ? "✅ Write" : "❌ Write";
echo "<br>";
echo hasPermission($userPermissions, PERM_EXECUTE) ? "✅ Execute" : "❌ Execute";
echo "<br>";
echo hasPermission($userPermissions, PERM_ADMIN) ? "✅ Admin" : "❌ Admin";
echo "<br>";

// เพิ่ม permission
$userPermissions |= PERM_EXECUTE;
echo hasPermission($userPermissions, PERM_EXECUTE) ? "✅ Execute added" : "❌"; 
echo "<br>";

// ลบ permission
$userPermissions &= ~PERM_WRITE;
echo hasPermission($userPermissions, PERM_WRITE) ? "✅ Write" : "❌ Write removed";
echo "<br>";
```

---

### 7. Ternary และ Special Operators

```php
<?php
// Ternary Operator (?:)
$age = 20;
$status = ($age >= 18) ? "ผู้ใหญ่" : "เยาวชน";
echo $status . "<br>"; // ผู้ใหญ่

// Short ternary / Elvis operator (?:)
$name = "";
$displayName = $name ?: "ไม่ระบุชื่อ"; // ถ้า $name เป็น falsy ใช้ค่าขวา
echo $displayName . "<br>"; // ไม่ระบุชื่อ

// Null Coalescing (??)
$user = null;
$username = $user ?? "Guest"; // ถ้า $user เป็น null ใช้ "Guest"
echo $username . "<br>"; // Guest

// ความต่างระหว่าง ?: และ ??
$value = 0;
echo ($value ?: "default") . "<br>"; // default (0 เป็น falsy)
echo ($value ?? "default") . "<br>"; // 0 (0 ไม่ใช่ null)

$arr = ['key' => null];
echo ($arr['key'] ?: "Elvis") . "<br>";    // Elvis (null เป็น falsy)
echo ($arr['key'] ?? "Coalescing") . "<br>"; // Coalescing (null)
echo ($arr['missing'] ?: "Elvis") . "<br>"; // Elvis + Notice
echo ($arr['missing'] ?? "Coalescing") . "<br>"; // Coalescing (ไม่มี Notice)

// Null Coalescing Assignment (??=) PHP 7.4+
$settings = [];
$settings['theme'] ??= 'light';  // กำหนดเฉพาะถ้าไม่มีค่าหรือเป็น null
$settings['theme'] ??= 'dark';   // ไม่เปลี่ยน เพราะมีค่าแล้ว
echo $settings['theme'] . "<br>"; // light

// Chained null coalescing
$a = null;
$b = null;
$c = "found";
echo ($a ?? $b ?? $c ?? "default") . "<br>"; // found

// Nested ternary (PHP 8 เปลี่ยน behavior - ไม่แนะนำ)
// $result = $a ? $b : $c ? $d : $e; // Deprecated PHP 7.4, Error PHP 8

// ใช้ match() แทน nested ternary ใน PHP 8
$score = 85;
$grade = match(true) {
    $score >= 90 => 'A',
    $score >= 80 => 'B',
    $score >= 70 => 'C',
    $score >= 60 => 'D',
    default => 'F'
};
echo "Grade: $grade<br>"; // B

// Instanceof Operator
class Animal {}
class Dog extends Animal {}
class Cat extends Animal {}

$dog = new Dog();
echo ($dog instanceof Dog) ? "เป็น Dog" : "ไม่ใช่ Dog"; 
echo "<br>";
echo ($dog instanceof Animal) ? "เป็น Animal" : "ไม่ใช่ Animal";
echo "<br>";
echo ($dog instanceof Cat) ? "เป็น Cat" : "ไม่ใช่ Cat";
echo "<br>";
```

---

### 8. Operator Precedence (ลำดับความสำคัญ)

```php
<?php
// Operator Precedence - สูงไปต่ำ
// 1. clone, new
// 2. **
// 3. ~, (int), (float), (string), (array), (object), (bool), @
// 4. instanceof
// 5. !
// 6. *, /, %
// 7. +, -, .
// 8. <<, >>
// 9. <, <=, >=, >
// 10. ==, !=, ===, !==, <=>
// 11. &
// 12. ^
// 13. |
// 14. &&
// 15. ||
// 16. ??
// 17. ?:
// 18. =, +=, -=, etc., ??=
// 19. yield, yield from
// 20. print
// 21. and
// 22. xor
// 23. or

// ตัวอย่างที่อาจสับสน
echo 2 + 3 * 4 . "<br>";    // 14 (คูณก่อน)
echo (2 + 3) * 4 . "<br>";  // 20 (ใน () ก่อน)

echo 2 ** 3 ** 2 . "<br>";  // 512 (** เป็น right-associative: 2**(3**2) = 2**9 = 512)
echo (2 ** 3) ** 2 . "<br>"; // 64

// && กับ ||
$a = true || false && false; // true || (false && false) = true || false = true
var_dump($a); // bool(true)

// การใช้วงเล็บเพื่อความชัดเจน
$x = 5;
$y = 3;
$z = 2;

// ไม่ชัดเจน
$result1 = $x > $y && $y > $z || $z == 2;

// ชัดเจนกว่า
$result2 = (($x > $y) && ($y > $z)) || ($z == 2);

// ทั้งคู่ได้ผลเหมือนกัน แต่แบบที่ 2 อ่านง่ายกว่า

// ตัวอย่างที่ต้องระวัง
$val = null;
echo $val ?? "default" . "!"; // null ?? ("default" . "!") = "default!"
echo "<br>";
// เพราะ . มี precedence สูงกว่า ??

// แก้ด้วย ()
echo ($val ?? "default") . "!"; // "default!"
echo "<br>";
```

---

### 9. PHP 8 Match Expression

```php
<?php
// Match Expression - เหมือน switch แต่ดีกว่า

$status = 2;

// switch แบบเก่า
switch ($status) {
    case 1:
        echo "Active<br>";
        break;
    case 2:
    case 3:
        echo "Pending<br>";
        break;
    default:
        echo "Unknown<br>";
}

// match แบบใหม่ (PHP 8.0+)
$label = match($status) {
    1 => "Active",
    2, 3 => "Pending",   // หลาย conditions
    4 => "Inactive",
    default => "Unknown"
};
echo $label . "<br>";

// ความแตกต่าง switch vs match:
// 1. match ใช้ strict comparison (===) - switch ใช้ loose (==)
// 2. match ต้องครอบคลุมทุก case (ถ้าไม่มี default จะ throw exception)
// 3. match คืนค่า (expression) - switch เป็น statement
// 4. match ไม่มี fall-through
// 5. match expressions ต้องเป็น single expression

// Match กับ complex conditions
$x = 10;
$result = match(true) {
    $x < 0 => "ลบ",
    $x === 0 => "ศูนย์",
    $x < 10 => "น้อยกว่า 10",
    $x === 10 => "สิบ",
    $x <= 100 => "น้อยกว่าหรือเท่ากับ 100",
    default => "มากกว่า 100"
};
echo $result . "<br>"; // สิบ

// Match กับ no-default (throw exception)
try {
    $value = match(999) {
        1 => "one",
        2 => "two",
        // ไม่มี default
    };
} catch (\UnhandledMatchError $e) {
    echo "UnhandledMatchError: " . $e->getMessage() . "<br>";
}

// Match กับ enum (PHP 8.1+)
enum Color {
    case Red;
    case Green;
    case Blue;
}

$color = Color::Green;
$hex = match($color) {
    Color::Red => '#FF0000',
    Color::Green => '#00FF00',
    Color::Blue => '#0000FF',
};
echo "Color: $hex<br>"; // #00FF00
```

---

### 10. Error Suppression Operator (@)

```php
<?php
// @ operator - ยับยั้ง error messages
// ⚠️ ไม่แนะนำให้ใช้ใน code สมัยใหม่

// แบบเก่า (ไม่แนะนำ)
$result = @file_get_contents("nonexistent.txt");
if ($result === false) {
    echo "ไม่สามารถอ่านไฟล์ได้<br>";
}

// แบบที่ดีกว่า - ใช้ try/catch หรือตรวจสอบก่อน
if (file_exists("nonexistent.txt")) {
    $result = file_get_contents("nonexistent.txt");
} else {
    echo "ไฟล์ไม่มีอยู่<br>";
}

// หรือใช้ try/catch
try {
    $conn = new PDO("mysql:host=invalid", "user", "pass");
} catch (PDOException $e) {
    echo "DB Error: " . $e->getMessage() . "<br>";
}

// PHP 8 Nullsafe operator (?->) ดีกว่า @ มาก
class User {
    public ?Address $address = null;
}
class Address {
    public ?City $city = null;
}
class City {
    public string $name = "Bangkok";
}

$user = new User();
// แบบเก่า (น่าเกลียด)
$cityName = isset($user) && isset($user->address) && isset($user->address->city) 
    ? $user->address->city->name 
    : "Unknown";

// แบบใหม่ PHP 8
$cityName = $user?->address?->city?->name ?? "Unknown";
echo $cityName . "<br>"; // Unknown
```

---

### 11. Expressions คืออะไร

```php
<?php
// Expression - สิ่งที่ evaluate แล้วได้ค่า

// Literals เป็น expressions
42;          // integer expression
3.14;        // float expression
"hello";     // string expression
true;        // bool expression
null;        // null expression

// Variable references
$x = 5;
$x;  // variable expression (ค่าคือ 5)

// Function calls
strlen("hello"); // function call expression (ค่าคือ 5)
abs(-42);        // ค่าคือ 42

// Assignments เป็น expressions (คืนค่าที่ assign)
$y = ($x = 10); // $x = 10, $y = 10
echo $y . "<br>"; // 10

// ตัวอย่าง: while loop ที่ใช้ assignment เป็น expression
$file = fopen("test.txt", "w");
fwrite($file, "Line 1\nLine 2\nLine 3\n");
fclose($file);

$file = fopen("test.txt", "r");
while (($line = fgets($file)) !== false) { // assignment เป็น expression
    echo trim($line) . "<br>";
}
fclose($file);

// Conditional Expressions
$age = 25;
$category = $age < 18 ? "minor" : ($age < 65 ? "adult" : "senior");
echo $category . "<br>"; // adult

// Complex expressions
$numbers = [5, 3, 8, 1, 9, 2, 7, 4, 6];
$sum = array_sum(array_filter($numbers, fn($n) => $n > 5));
echo "Sum of numbers > 5: $sum<br>"; // 8+9+7+6 = 30

// Expression ใน interpolation
$items = ['apple', 'banana', 'cherry'];
echo "Total items: " . count($items) . "<br>"; // 3
echo "First: {$items[0]}<br>"; // apple
```

---

### 12. Bitwise Tricks และ Practical Examples

```php
<?php
// Practical Bitwise Operations

// 1. ตรวจสอบว่าเลขคู่หรือคี่
function isEven(int $n): bool
{
    return ($n & 1) === 0; // bit สุดท้ายเป็น 0 = คู่
}

function isOdd(int $n): bool
{
    return ($n & 1) === 1; // bit สุดท้ายเป็น 1 = คี่
}

echo isEven(4) ? "4 เป็นเลขคู่" : "4 เป็นเลขคี่";
echo "<br>";
echo isOdd(7) ? "7 เป็นเลขคี่" : "7 เป็นเลขคู่";
echo "<br>";

// 2. คูณและหารด้วย 2 อย่างรวดเร็ว
$x = 16;
echo ($x << 1) . "<br>"; // 32 (คูณ 2)
echo ($x >> 1) . "<br>"; // 8  (หาร 2)

// 3. ค่า absolute โดยไม่ใช้ abs()
function absoluteValue(int $n): int
{
    $mask = $n >> 31; // 0 ถ้าบวก, -1 ถ้าลบ
    return ($n + $mask) ^ $mask;
}

echo absoluteValue(-15) . "<br>"; // 15
echo absoluteValue(7) . "<br>";   // 7

// 4. Feature Flags
class FeatureFlags
{
    private int $flags = 0;
    
    const DARK_MODE       = 1 << 0; // 1
    const NOTIFICATIONS   = 1 << 1; // 2
    const BETA_FEATURES   = 1 << 2; // 4
    const ADVANCED_SEARCH = 1 << 3; // 8
    
    public function enable(int $flag): void
    {
        $this->flags |= $flag;
    }
    
    public function disable(int $flag): void
    {
        $this->flags &= ~$flag;
    }
    
    public function toggle(int $flag): void
    {
        $this->flags ^= $flag;
    }
    
    public function isEnabled(int $flag): bool
    {
        return ($this->flags & $flag) !== 0;
    }
    
    public function getFlags(): int
    {
        return $this->flags;
    }
}

$features = new FeatureFlags();
$features->enable(FeatureFlags::DARK_MODE);
$features->enable(FeatureFlags::NOTIFICATIONS);

echo "Dark Mode: " . ($features->isEnabled(FeatureFlags::DARK_MODE) ? "On" : "Off") . "<br>";
echo "Beta: " . ($features->isEnabled(FeatureFlags::BETA_FEATURES) ? "On" : "Off") . "<br>";

$features->toggle(FeatureFlags::DARK_MODE);
echo "Dark Mode after toggle: " . ($features->isEnabled(FeatureFlags::DARK_MODE) ? "On" : "Off") . "<br>";
```

---

### 13. Workshop: Calculator

```php
<?php
declare(strict_types=1);

class Calculator
{
    private float $result = 0;
    private array $history = [];
    
    public function __construct(float $initial = 0)
    {
        $this->result = $initial;
    }
    
    public function add(float $value): static
    {
        $this->history[] = "+ $value";
        $this->result += $value;
        return $this;
    }
    
    public function subtract(float $value): static
    {
        $this->history[] = "- $value";
        $this->result -= $value;
        return $this;
    }
    
    public function multiply(float $value): static
    {
        $this->history[] = "× $value";
        $this->result *= $value;
        return $this;
    }
    
    public function divide(float $value): static
    {
        if ($value == 0) {
            throw new \DivisionByZeroError("ไม่สามารถหารด้วย 0 ได้");
        }
        $this->history[] = "÷ $value";
        $this->result /= $value;
        return $this;
    }
    
    public function power(float $exponent): static
    {
        $this->history[] = "^ $exponent";
        $this->result **= $exponent;
        return $this;
    }
    
    public function modulo(float $divisor): static
    {
        $this->history[] = "% $divisor";
        $this->result = fmod($this->result, $divisor);
        return $this;
    }
    
    public function sqrt(): static
    {
        if ($this->result < 0) {
            throw new \InvalidArgumentException("ไม่สามารถหารากที่สองของจำนวนลบได้");
        }
        $this->history[] = "√";
        $this->result = sqrt($this->result);
        return $this;
    }
    
    public function percentage(): static
    {
        $this->history[] = "%";
        $this->result /= 100;
        return $this;
    }
    
    public function reset(float $value = 0): static
    {
        $this->result = $value;
        $this->history = [];
        return $this;
    }
    
    public function getResult(): float
    {
        return $this->result;
    }
    
    public function getHistory(): array
    {
        return $this->history;
    }
    
    public function display(): string
    {
        return "Result: " . number_format($this->result, 4, '.', ',');
    }
}

// ใช้งาน Calculator
try {
    $calc = new Calculator(100);
    
    $result = $calc
        ->add(50)
        ->multiply(2)
        ->subtract(30)
        ->divide(4)
        ->power(2);
    
    echo $result->display() . "<br>"; // Result: 2025.0000
    echo "History: " . implode(", ", $result->getHistory()) . "<br>";
    
    // Calculate discount
    $price = new Calculator(1000);
    $finalPrice = $price->multiply(0.85); // 15% discount
    echo "Final Price: " . number_format($finalPrice->getResult(), 2) . " บาท<br>";
    
    // Test division by zero
    $calc2 = new Calculator(10);
    $calc2->divide(0);
    
} catch (\DivisionByZeroError $e) {
    echo "Error: " . $e->getMessage() . "<br>";
} catch (\InvalidArgumentException $e) {
    echo "Error: " . $e->getMessage() . "<br>";
}

// ตัวอย่าง: คำนวณภาษี
function calculateVAT(float $price, float $vatRate = 7.0): array
{
    $vatAmount = $price * ($vatRate / 100);
    $total = $price + $vatAmount;
    
    return [
        'price' => $price,
        'vat_rate' => $vatRate,
        'vat_amount' => $vatAmount,
        'total' => $total,
        'formatted' => [
            'price' => number_format($price, 2),
            'vat_amount' => number_format($vatAmount, 2),
            'total' => number_format($total, 2),
        ]
    ];
}

$invoice = calculateVAT(10000);
echo "ราคาก่อน VAT: " . $invoice['formatted']['price'] . " บาท<br>";
echo "VAT {$invoice['vat_rate']}%: " . $invoice['formatted']['vat_amount'] . " บาท<br>";
echo "ราคารวม VAT: " . $invoice['formatted']['total'] . " บาท<br>";
```

---

### 14. ตัวอย่างการใช้ Operators จริงๆ

```php
<?php
declare(strict_types=1);

// E-commerce Price Calculation
class Price
{
    public function __construct(
        private float $amount,
        private string $currency = 'THB'
    ) {}
    
    public function withDiscount(float $percent): static
    {
        return new static(
            $this->amount * (1 - $percent / 100),
            $this->currency
        );
    }
    
    public function withVAT(float $vatPercent = 7.0): static
    {
        return new static(
            $this->amount * (1 + $vatPercent / 100),
            $this->currency
        );
    }
    
    public function add(Price $other): static
    {
        if ($this->currency !== $other->currency) {
            throw new \InvalidArgumentException("สกุลเงินไม่ตรงกัน");
        }
        return new static($this->amount + $other->amount, $this->currency);
    }
    
    public function isGreaterThan(Price $other): bool
    {
        return $this->amount > $other->amount;
    }
    
    public function format(): string
    {
        return number_format($this->amount, 2) . ' ' . $this->currency;
    }
    
    public function getAmount(): float
    {
        return $this->amount;
    }
}

// ใช้งาน
$originalPrice = new Price(1000);
$salePrice = $originalPrice->withDiscount(20); // ลด 20%
$finalPrice = $salePrice->withVAT();          // บวก VAT 7%

echo "ราคาเดิม: " . $originalPrice->format() . "<br>";      // 1,000.00 THB
echo "ราคาหลังลด 20%: " . $salePrice->format() . "<br>";    // 800.00 THB
echo "ราคารวม VAT: " . $finalPrice->format() . "<br>";       // 856.00 THB

// Shopping cart
$cart = [
    new Price(500),
    new Price(300),
    new Price(750),
];

$total = array_reduce(
    $cart,
    fn(Price $carry, Price $item) => $carry->add($item),
    new Price(0)
);

echo "ยอดรวม: " . $total->format() . "<br>"; // 1,550.00 THB

// Comparison
$threshold = new Price(1000);
echo $total->isGreaterThan($threshold) 
    ? "ได้รับการจัดส่งฟรี!" 
    : "ต้องการยอดซื้อเพิ่ม " . ($threshold->getAmount() - $total->getAmount()) . " บาท";
```

---

## 📝 Quiz

1. `0.1 + 0.2 === 0.3` ได้ผลลัพธ์อะไรและทำไม?
2. ความแตกต่างระหว่าง `??` และ `?:` คืออะไร?
3. `$a = true || false && false` - `$a` มีค่าเท่าไร?
4. Spaceship operator (`<=>`) ใช้ทำอะไร?
5. ทำไมถึงควรใช้ `&&` แทน `and`?
6. Bitwise `&` ใช้ประโยชน์อะไรได้บ้างในการเขียน code จริงๆ?
7. Match expression ต่างจาก switch อย่างไร?

**เฉลย:**
1. `false` - เพราะ floating point precision (0.30000000000000004)
2. `??` เช็คเฉพาะ null; `?:` เช็ค falsy ทั้งหมด (null, 0, "", false, [])
3. `true` - เพราะ `&&` มี precedence สูงกว่า `||`: `true || (false && false)` = `true || false` = `true`
4. เปรียบเทียบสองค่า คืน -1, 0, หรือ 1 - ใช้ใน sorting
5. `and` มี precedence ต่ำกว่า `=` ทำให้เกิด bug ได้ง่าย
6. Permission flags, feature toggles, color values, packed data
7. Match ใช้ `===`, คืนค่า, ไม่มี fall-through, ต้องครอบคลุมทุก case

---

## ⏭️ Part ถัดไป

**Part 004: PHP Control Structures - if/else/switch**

เราจะเรียนรู้:
- if, elseif, else
- switch/case
- Match expression (ขั้นสูง)
- Nested conditions
- Best practices สำหรับ conditionals

---

*Part 003 | ระดับพื้นฐาน | PHP Operators*
