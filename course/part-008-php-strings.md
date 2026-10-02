# 🔤 Part 008: PHP Strings — สตริง

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ string functions ที่สำคัญทั้งหมดได้
- เขียน Regular Expressions ใน PHP ได้
- จัดการ Multibyte strings (ภาษาไทย, ภาษาจีน) ได้ถูกต้อง
- Escape ข้อมูลเพื่อความปลอดภัยได้
- สร้าง Template engine และ Text processor ได้

---

## 📖 1. String Functions ทั้งหมด

### การตรวจสอบความยาว

```php
<?php
$str = "Hello, World!";
$thai = "สวัสดีชาวโลก";

// strlen — นับจำนวน bytes (ไม่ใช่ตัวอักษรสำหรับ Unicode)
echo strlen($str);    // 13
echo strlen($thai);   // 39 (ภาษาไทย UTF-8 ใช้ 3 bytes/ตัว)

// mb_strlen — นับจำนวนตัวอักษรจริง (Multibyte-safe)
echo mb_strlen($str, 'UTF-8');   // 13
echo mb_strlen($thai, 'UTF-8');  // 13
```

### การแปลง Case

```php
<?php
$text = "Hello World from PHP";

echo strtoupper($text);   // HELLO WORLD FROM PHP
echo strtolower($text);   // hello world from php
echo ucfirst($text);      // Hello World from PHP (ตัวแรก)
echo lcfirst($text);      // hELLO WORLD FROM PHP (ตัวแรก lowercase)
echo ucwords($text);      // Hello World From Php (ทุกคำ)

// mb_ versions สำหรับ Unicode
$thai = "สวัสดีชาวโลก";
echo mb_strtoupper($thai, 'UTF-8'); // ภาษาไทยไม่มี uppercase แต่ใช้ได้กับภาษาอื่น
```

### การค้นหาใน String

```php
<?php
$text = "The quick brown fox jumps over the lazy dog";

// strpos — ค้นหาตำแหน่ง (case-sensitive)
$pos = strpos($text, "fox");
echo $pos; // 16

// ค้นหาจากตำแหน่งที่กำหนด
$pos2 = strpos($text, "the", 10); // เริ่มค้นหาที่ index 10
echo $pos2; // 31

// stripos — ค้นหาโดยไม่สนใจ case
$pos3 = stripos($text, "THE");
echo $pos3; // 0

// strrpos — ค้นหาตำแหน่งสุดท้าย
$pos4 = strrpos($text, "the");
echo $pos4; // 31

// ตรวจสอบว่ามีหรือไม่ (ระวัง! ใช้ !== false)
if (strpos($text, "fox") !== false) {
    echo "พบคำว่า fox\n";
}

// PHP 8.0+: str_contains, str_starts_with, str_ends_with
var_dump(str_contains($text, "fox"));     // bool(true)
var_dump(str_starts_with($text, "The")); // bool(true)
var_dump(str_ends_with($text, "dog"));   // bool(true)
```

### การตัดและแทน

```php
<?php
$text = "Hello, World!";

// substr — ดึง substring
echo substr($text, 7);      // World!
echo substr($text, 7, 5);   // World
echo substr($text, -6);     // orld!
echo substr($text, -6, 5);  // orld

// str_replace — แทนที่ string
$result = str_replace("World", "PHP", $text);
echo $result; // Hello, PHP!

// แทนที่หลายค่า
$search  = ['Hello', 'World'];
$replace = ['Sawadee', 'Thailand'];
$result  = str_replace($search, $replace, $text);
echo $result; // Sawadee, Thailand!

// str_ireplace — แทนที่โดยไม่สนใจ case
$result = str_ireplace("hello", "Hi", "Hello World HELLO");
echo $result; // Hi World Hi

// substr_replace — แทนที่ที่ตำแหน่งที่กำหนด
$result = substr_replace($text, "PHP", 7, 5);
echo $result; // Hello, PHP!
```

### Trim Functions

```php
<?php
$messy = "   Hello World   ";
$withNewlines = "\n\tHello\n\t";

echo trim($messy);          // "Hello World"   — ตัดทั้งหัวท้าย
echo ltrim($messy);         // "Hello World   " — ตัดหัว
echo rtrim($messy);         // "   Hello World" — ตัดท้าย
echo trim($withNewlines);   // "Hello"          — ตัด whitespace ทุกชนิด

// ระบุ characters ที่จะตัด
$url = "***https://example.com***";
echo trim($url, '*'); // https://example.com

$csv = ",,,apple,banana,";
echo trim($csv, ','); // apple,banana
```

### Split & Join

```php
<?php
// explode — แยก string เป็น array
$csv = "apple,banana,cherry,date";
$fruits = explode(",", $csv);
print_r($fruits); // ['apple', 'banana', 'cherry', 'date']

// จำกัดจำนวน pieces
$parts = explode(",", $csv, 3);
print_r($parts); // ['apple', 'banana', 'cherry,date']

// implode/join — รวม array เป็น string
$joined = implode(" | ", $fruits);
echo $joined; // apple | banana | cherry | date

// str_split — แยกเป็น array ของตัวอักษร
$chars = str_split("Hello");
print_r($chars); // ['H', 'e', 'l', 'l', 'o']

// แยกทีละ N ตัว
$chunks = str_split("Hello World", 3);
print_r($chunks); // ['Hel', 'lo ', 'Wor', 'ld']
```

### Padding & Repeating

```php
<?php
// str_pad — เติม padding
$num = "42";
echo str_pad($num, 5);           // "42   "  — เติมขวา (default)
echo str_pad($num, 5, "0", STR_PAD_LEFT);  // "00042" — เติมซ้าย
echo str_pad($num, 5, "0", STR_PAD_BOTH);  // "0420 " — เติมสองข้าง
echo str_pad($num, 10, "-+", STR_PAD_BOTH); // "-+-+42-+-+"

// str_repeat — ทำซ้ำ
echo str_repeat("*", 20);  // ********************
echo str_repeat("ab", 5);  // ababababab

// ตัวอย่างการใช้งาน: แสดง progress bar
function progressBar(int $current, int $total, int $width = 30): string
{
    $percent = $current / $total;
    $filled  = (int)($percent * $width);
    $empty   = $width - $filled;

    $bar = str_repeat('█', $filled) . str_repeat('░', $empty);
    $pct = str_pad(round($percent * 100), 3, ' ', STR_PAD_LEFT);

    return "[$bar] $pct%";
}

echo progressBar(35, 100) . "\n"; // [██████████░░░░░░░░░░░░░░░░░░░░]  35%
echo progressBar(75, 100) . "\n"; // [██████████████████████░░░░░░░░]  75%
```

### Comparison Functions

```php
<?php
// strcmp — เปรียบเทียบ string (case-sensitive)
// คืน: < 0 ถ้า str1 < str2, 0 ถ้าเท่ากัน, > 0 ถ้า str1 > str2
echo strcmp("apple", "banana");  // negative (a < b)
echo strcmp("banana", "apple");  // positive
echo strcmp("apple", "apple");   // 0

// strcasecmp — เปรียบเทียบโดยไม่สนใจ case
echo strcasecmp("Hello", "hello"); // 0

// similar_text — ความคล้ายคลึง
similar_text("Hello", "World", $percent);
echo "คล้ายกัน: $percent%\n"; // คล้ายกัน: 40%

// levenshtein — Levenshtein distance (จำนวน edit ที่ต้องทำ)
echo levenshtein("kitten", "sitting"); // 3
echo levenshtein("hello", "hello");    // 0

// soundex / metaphone — เสียงคล้ายกัน
echo soundex("Thompson"); // T512
echo metaphone("Smith");  // SM0
```

### Format Functions

```php
<?php
// sprintf — format string
$name  = "สมชาย";
$score = 95.5;
$rank  = 3;

$formatted = sprintf("ชื่อ: %s | คะแนน: %.1f | อันดับ: %d", $name, $score, $rank);
echo $formatted; // ชื่อ: สมชาย | คะแนน: 95.5 | อันดับ: 3

// รูปแบบ format ที่สำคัญ
echo sprintf("%d",   42);         // 42        (integer)
echo sprintf("%05d", 42);         // 00042     (zero-padded)
echo sprintf("%.2f", 3.14159);    // 3.14      (float 2 decimal)
echo sprintf("%e",   12345.678);  // 1.234568e+4 (scientific)
echo sprintf("%s",   "hello");    // hello     (string)
echo sprintf("%10s", "hello");    // "     hello" (right-aligned)
echo sprintf("%-10s", "hello");   // "hello     " (left-aligned)
echo sprintf("%x",   255);        // ff         (hex)
echo sprintf("%o",   8);          // 10         (octal)
echo sprintf("%b",   10);         // 1010       (binary)

// number_format — จัดรูปแบบตัวเลข
echo number_format(1234567.891, 2, '.', ','); // 1,234,567.89
echo number_format(9999.5, 0, '.', ',');      // 10,000
```

### Miscellaneous Functions

```php
<?php
// strrev — กลับ string
echo strrev("Hello"); // olleH

// str_word_count — นับคำ
echo str_word_count("Hello World PHP"); // 3

// wordwrap — ตัดบรรทัดตามความยาว
$long = "The quick brown fox jumped over the lazy dog";
echo wordwrap($long, 15, "\n", true);
// The quick brown
// fox jumped over
// the lazy dog

// chunk_split — แบ่ง string ใส่ separator
echo chunk_split("AAABBBCCC", 3, "-"); // AAA-BBB-CCC-

// nl2br — แปลง newline เป็น <br>
$text = "บรรทัดที่ 1\nบรรทัดที่ 2";
echo nl2br($text);
// บรรทัดที่ 1<br />
// บรรทัดที่ 2

// quoted_printable_encode / decode
// base64_encode / decode
$data = "สวัสดี PHP";
$encoded = base64_encode($data);
echo $encoded; // สตริง base64
$decoded = base64_decode($encoded);
echo $decoded; // สวัสดี PHP

// md5 / sha1 / hash
echo md5("password");                    // 5f4dcc3b5aa765d61d8327deb882cf99
echo sha1("password");                   // 5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8
echo hash('sha256', "password");         // 5e884898da28047151d0e56f8dc6292773603d0d...
echo password_hash("password", PASSWORD_BCRYPT); // $2y$10$...
```

---

## 📖 2. Regular Expressions ใน PHP

### พื้นฐาน Regex

```php
<?php
// preg_match — ค้นหา pattern (ส่งคืน 1 ถ้าพบ)
$email = "user@example.com";
$pattern = '/^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/';

if (preg_match($pattern, $email)) {
    echo "Email ถูกต้อง\n";
}

// เก็บ match ไว้ใน variable
$text = "วันที่ 25/12/2024 เป็นวันคริสต์มาส";
if (preg_match('/(\d{2})\/(\d{2})\/(\d{4})/', $text, $matches)) {
    echo "พบวันที่: {$matches[0]}\n";  // 25/12/2024
    echo "วัน: {$matches[1]}\n";       // 25
    echo "เดือน: {$matches[2]}\n";     // 12
    echo "ปี: {$matches[3]}\n";        // 2024
}
```

### Pattern Syntax พื้นฐาน

```php
<?php
// Character classes
// .  — ตัวอักษรใดก็ได้ (ยกเว้น newline)
// \d — digit [0-9]
// \D — ไม่ใช่ digit
// \w — word character [a-zA-Z0-9_]
// \W — ไม่ใช่ word character
// \s — whitespace (space, tab, newline)
// \S — ไม่ใช่ whitespace

// Quantifiers
// *  — 0 ครั้งหรือมากกว่า
// +  — 1 ครั้งหรือมากกว่า
// ?  — 0 หรือ 1 ครั้ง
// {n} — n ครั้ง
// {n,m} — n ถึง m ครั้ง

// Anchors
// ^  — ต้นบรรทัด
// $  — ท้ายบรรทัด
// \b — word boundary

// Examples
$patterns = [
    '/^\d{10}$/'      => "เบอร์โทรศัพท์ 10 หลัก",
    '/^[A-Z]{2}\d{6}$/' => "รหัสสินค้า เช่น TH123456",
    '/^\d{4}-\d{2}-\d{2}$/' => "วันที่ format YYYY-MM-DD",
];

$tests = [
    "0891234567",
    "TH123456",
    "2024-12-25",
    "invalid",
];

foreach ($tests as $test) {
    foreach ($patterns as $pattern => $name) {
        if (preg_match($pattern, $test)) {
            echo "'$test' ตรงกับ '$name'\n";
        }
    }
}
```

### preg_match_all

```php
<?php
// preg_match_all — หาทุก match
$html = '<a href="https://php.net">PHP</a> and <a href="https://laravel.com">Laravel</a>';

preg_match_all('/<a href="([^"]+)">([^<]+)<\/a>/', $html, $matches);

echo "URLs พบ: " . count($matches[0]) . "\n";
for ($i = 0; $i < count($matches[0]); $i++) {
    echo "Link: {$matches[2][$i]} → {$matches[1][$i]}\n";
}
// Link: PHP → https://php.net
// Link: Laravel → https://laravel.com
```

### preg_replace

```php
<?php
// preg_replace — แทนที่ด้วย regex
$text = "โทร: 081-234-5678 หรือ 089-876-5432";

// ลบเครื่องหมาย - จากเบอร์โทร
$result = preg_replace('/(\d{3})-(\d{3})-(\d{4})/', '$1$2$3', $text);
echo $result; // โทร: 0812345678 หรือ 0898765432

// ลบ HTML tags
$html = "<p>Hello <b>World</b>!</p>";
$plain = preg_replace('/<[^>]+>/', '', $html);
echo $plain; // Hello World!

// Callback version
$result = preg_replace_callback(
    '/\d+/',
    fn($matches) => $matches[0] * 2,
    "I have 3 cats and 5 dogs"
);
echo $result; // I have 6 cats and 10 dogs
```

### preg_split

```php
<?php
// preg_split — แบ่ง string ด้วย regex
$text = "one,two;three|four five";
$parts = preg_split('/[,;| ]+/', $text);
print_r($parts); // ['one', 'two', 'three', 'four', 'five']

// Named captures (PHP 7.0+)
$date = "2024-12-25";
preg_match('/(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})/', $date, $m);
echo "ปี: {$m['year']}, เดือน: {$m['month']}, วัน: {$m['day']}\n";
// ปี: 2024, เดือน: 12, วัน: 25
```

### Regex Patterns ที่ใช้บ่อย

```php
<?php
class Validator
{
    private static array $patterns = [
        'email'    => '/^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/',
        'phone_th' => '/^(0[689]\d{8}|0[2-5]\d{7,8})$/', // เบอร์ไทย
        'url'      => '/^https?:\/\/([\da-z\.-]+)\.([a-z\.]{2,6})([\/\w \.-]*)*\/?$/',
        'ip'       => '/^(\d{1,3}\.){3}\d{1,3}$/',
        'date'     => '/^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$/',
        'thai_id'  => '/^\d{13}$/', // เลขบัตรประชาชน
        'password' => '/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)[a-zA-Z\d@$!%*?&]{8,}$/',
        'slug'     => '/^[a-z0-9]+(?:-[a-z0-9]+)*$/',
        'hex_color' => '/^#([a-fA-F0-9]{6}|[a-fA-F0-9]{3})$/',
    ];

    public static function validate(string $value, string $rule): bool
    {
        $pattern = self::$patterns[$rule] ?? null;
        if (!$pattern) return false;
        return (bool)preg_match($pattern, $value);
    }
}

// ทดสอบ
$tests = [
    ['user@example.com', 'email'],
    ['invalid-email', 'email'],
    ['0891234567', 'phone_th'],
    ['2024-12-25', 'date'],
    ['#FF5733', 'hex_color'],
    ['my-slug-here', 'slug'],
];

foreach ($tests as [$value, $rule]) {
    $valid = Validator::validate($value, $rule) ? "✅" : "❌";
    echo "$valid [$rule] '$value'\n";
}
```

---

## 📖 3. Multibyte String (mb_*)

```php
<?php
// ตั้งค่า default encoding
mb_internal_encoding('UTF-8');
mb_language('uni');

// mb_strlen vs strlen
$thai = "สวัสดีชาวโลก";
echo strlen($thai);    // 39 — นับ bytes
echo mb_strlen($thai); // 13 — นับตัวอักษรจริง

// mb_substr — ตัด substring (Multibyte-safe)
echo mb_substr($thai, 0, 6);   // สวัสดี
echo mb_substr($thai, 6);      // ชาวโลก
echo mb_substr($thai, -3);     // โลก

// mb_strpos — หาตำแหน่ง
echo mb_strpos($thai, "ชาว");  // 6

// mb_strtolower / mb_strtoupper (สำหรับภาษาที่มี case)
$german = "Über den Wolken";
echo mb_strtolower($german, 'UTF-8'); // über den wolken
echo mb_strtoupper($german, 'UTF-8'); // ÜBER DEN WOLKEN

// mb_convert_encoding — แปลง encoding
$sjis = mb_convert_encoding($thai, 'SJIS', 'UTF-8');
$utf8 = mb_convert_encoding($sjis, 'UTF-8', 'SJIS');

// mb_detect_encoding — ตรวจสอบ encoding
$encoding = mb_detect_encoding($thai, ['UTF-8', 'TIS-620', 'SJIS']);
echo "Encoding: $encoding\n"; // UTF-8
```

### การจัดการ Thai Text

```php
<?php
/**
 * ฟังก์ชันสำหรับจัดการข้อความภาษาไทย
 */

/**
 * ตัดข้อความภาษาไทยอย่างถูกต้อง
 */
function truncateThai(string $text, int $maxChars, string $suffix = '...'): string
{
    if (mb_strlen($text, 'UTF-8') <= $maxChars) {
        return $text;
    }

    $truncated = mb_substr($text, 0, $maxChars - mb_strlen($suffix, 'UTF-8'), 'UTF-8');
    return $truncated . $suffix;
}

/**
 * นับคำในภาษาไทย (ประมาณ)
 */
function countThaiWords(string $text): int
{
    // วิธีประมาณ: นับตามช่องว่างและเครื่องหมาย
    $words = preg_split('/[\s,\.!?]+/u', trim($text), -1, PREG_SPLIT_NO_EMPTY);
    return count($words);
}

/**
 * Pad string โดยนับตัวอักษรจริง
 */
function mbStrPad(string $str, int $padLength, string $padString = ' ', int $padType = STR_PAD_RIGHT, string $encoding = 'UTF-8'): string
{
    $strLen    = mb_strlen($str, $encoding);
    $padStrLen = mb_strlen($padString, $encoding);

    if ($strLen >= $padLength) {
        return $str;
    }

    $needed = $padLength - $strLen;

    $padding = str_repeat($padString, (int)ceil($needed / $padStrLen));
    $padding = mb_substr($padding, 0, $needed, $encoding);

    return match($padType) {
        STR_PAD_LEFT  => $padding . $str,
        STR_PAD_BOTH  => mb_substr($padding, 0, (int)floor($needed / 2), $encoding) . $str . mb_substr($padding, 0, (int)ceil($needed / 2), $encoding),
        default        => $str . $padding,
    };
}

// ทดสอบ
echo truncateThai("สวัสดีชาวโลก ยินดีต้อนรับสู่ PHP", 10) . "\n"; // สวัสดีชาวโ...
echo countThaiWords("สวัสดี ชาวโลก ยินดีต้อนรับ") . "\n";          // 3

// ตาราง aligned
$items = ["สมชาย ใจดี", "สมหญิง", "สมศรี เจริญ", "ส"];
foreach ($items as $name) {
    echo mbStrPad($name, 15) . " | ข้อมูลเพิ่มเติม\n";
}
```

---

## 📖 4. String Formatting

```php
<?php
// Heredoc — multiline string
$name = "สมชาย";
$age  = 30;

$heredoc = <<<EOT
ชื่อ: $name
อายุ: $age ปี
ที่อยู่: กรุงเทพมหานคร
EOT;

echo $heredoc;

// Nowdoc — เหมือน single quote (ไม่ interpolate)
$nowdoc = <<<'EOT'
ชื่อ: $name
อายุ: $age ปี
EOT;

echo $nowdoc; // แสดง $name และ $age ตามตัว

// printf — format โดยตรง (ไม่ต้อง echo)
printf("%-20s %5d บาท\n", "หมวก", 299);
printf("%-20s %5d บาท\n", "เสื้อผ้า", 599);
printf("%-20s %5d บาท\n", "กางเกง", 799);

// vsprintf — format จาก array
$data = ['สมชาย', 30, 'IT'];
$formatted = vsprintf("ชื่อ: %s | อายุ: %d | แผนก: %s", $data);
echo $formatted;
```

---

## 📖 5. String Security

### htmlspecialchars — ป้องกัน XSS

```php
<?php
// ❌ อันตราย — XSS vulnerability
$userInput = '<script>alert("hacked!");</script>';
echo $userInput; // จะรัน JavaScript!

// ✅ ถูกต้อง — escape HTML entities
echo htmlspecialchars($userInput, ENT_QUOTES | ENT_HTML5, 'UTF-8');
// &lt;script&gt;alert(&quot;hacked!&quot;);&lt;/script&gt;

// htmlentities — เข้ารหัส characters มากกว่า
echo htmlentities("<b>Hello & \"World\"</b>", ENT_QUOTES, 'UTF-8');
// &lt;b&gt;Hello &amp; &quot;World&quot;&lt;/b&gt;

// html_entity_decode — ถอดรหัสกลับ
$encoded = "&lt;p&gt;Hello &amp; World&lt;/p&gt;";
echo html_entity_decode($encoded, ENT_QUOTES, 'UTF-8');
// <p>Hello & World</p>

// strip_tags — ลบ HTML tags
$dirty = "<p>Hello <b>World</b>! <script>evil()</script></p>";
echo strip_tags($dirty);              // Hello World! evil()
echo strip_tags($dirty, ['p', 'b']); // <p>Hello <b>World</b>! evil()</p>
```

### SQL Injection Prevention

```php
<?php
// ❌ อันตราย — SQL Injection
$username = "'; DROP TABLE users; --";
$query_bad = "SELECT * FROM users WHERE username = '$username'";
// ได้ SQL: SELECT * FROM users WHERE username = ''; DROP TABLE users; --'

// ✅ ใช้ Prepared Statements (วิธีที่ดีที่สุด)
$pdo = new \PDO("mysql:host=localhost;dbname=test", "user", "pass");
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ?");
$stmt->execute([$username]);

// ✅ หรือ addslashes (ทางเลือกแต่ไม่แนะนำ)
$escaped = addslashes($username);
// '; DROP TABLE users; -- → \'; DROP TABLE users; --

// ✅ PDO::quote — escape สำหรับ query string
$quoted = $pdo->quote($username);
```

### Password Hashing

```php
<?php
// ✅ ใช้ password_hash — ปลอดภัย
$password = "mySecretPassword123";
$hash = password_hash($password, PASSWORD_BCRYPT);
echo $hash; // $2y$10$...

// ตรวจสอบ password
$isCorrect = password_verify($password, $hash);
var_dump($isCorrect); // bool(true)

// PHP 8.0+ — PASSWORD_ARGON2ID (แนะนำกว่า Bcrypt)
$hash2 = password_hash($password, PASSWORD_ARGON2ID, [
    'memory_cost' => 65536,
    'time_cost'   => 4,
    'threads'     => 3,
]);

// ❌ อย่าใช้ md5 หรือ sha1 สำหรับ password!
// echo md5($password); // ไม่ปลอดภัย!
```

### Sanitization Functions

```php
<?php
// filter_var — validate และ sanitize
$email = "  USER@EXAMPLE.COM  ";

// Sanitize — ทำให้สะอาด
$cleanEmail = filter_var(trim($email), FILTER_SANITIZE_EMAIL);
echo $cleanEmail; // USER@EXAMPLE.COM

// Validate
$validEmail = filter_var($cleanEmail, FILTER_VALIDATE_EMAIL);
var_dump($validEmail); // string("USER@EXAMPLE.COM") หรือ false

// ตัวอย่าง sanitization functions ที่ใช้บ่อย
$inputs = [
    'name'   => "<b>สมชาย</b> O'Brien",
    'age'    => "25abc",
    'price'  => "1,234.56",
    'url'    => "  https://example.com/path?q=1  ",
    'email'  => "user@example.com",
];

// Clean name — ลบ HTML, แต่อนุญาต apostrophe
$cleanName = htmlspecialchars(strip_tags($inputs['name']), ENT_QUOTES, 'UTF-8');
echo "Name: $cleanName\n"; // Name: สมชาย O&#039;Brien

// ดึงตัวเลขจาก string
$age = filter_var($inputs['age'], FILTER_SANITIZE_NUMBER_INT);
echo "Age: $age\n"; // Age: 25

// ดึง float
$price = filter_var($inputs['price'], FILTER_SANITIZE_NUMBER_FLOAT, FILTER_FLAG_ALLOW_FRACTION);
echo "Price: $price\n"; // Price: 1234.56 (ลบ comma)
```

---

## 🔧 Workshop 1: Simple Template Engine

```php
<?php
/**
 * Simple Template Engine
 * Workshop: PHP Strings
 */
class TemplateEngine
{
    private array $data = [];
    private array $filters = [];

    public function __construct()
    {
        // Register built-in filters
        $this->registerFilter('upper',    'strtoupper');
        $this->registerFilter('lower',    'strtolower');
        $this->registerFilter('trim',     'trim');
        $this->registerFilter('nl2br',    'nl2br');
        $this->registerFilter('escape',   fn($v) => htmlspecialchars((string)$v, ENT_QUOTES | ENT_HTML5, 'UTF-8'));
        $this->registerFilter('url',      'urlencode');
        $this->registerFilter('currency', fn($v) => '฿' . number_format((float)$v, 2));
        $this->registerFilter('date',     fn($v) => date('d/m/Y', strtotime($v)));
        $this->registerFilter('truncate', fn($v, $len = 50) => mb_strlen($v) > $len
            ? mb_substr($v, 0, $len) . '...' : $v);
    }

    public function registerFilter(string $name, callable $fn): void
    {
        $this->filters[$name] = $fn;
    }

    public function assign(string|array $key, mixed $value = null): self
    {
        if (is_array($key)) {
            $this->data = array_merge($this->data, $key);
        } else {
            $this->data[$key] = $value;
        }
        return $this;
    }

    public function render(string $template): string
    {
        // แทนที่ {{ variable }} และ {{ variable|filter }}
        $result = preg_replace_callback(
            '/\{\{\s*([a-zA-Z0-9_.]+)(?:\|([a-zA-Z0-9_]+)(?::([^}]*))?)?\s*\}\}/',
            function($matches) {
                $varName    = $matches[1];
                $filterName = $matches[2] ?? null;
                $filterArg  = $matches[3] ?? null;

                // ดึงค่าจาก dot notation (e.g., user.name)
                $value = $this->getValue($varName);

                // Apply filter
                if ($filterName && isset($this->filters[$filterName])) {
                    $fn = $this->filters[$filterName];
                    $value = $filterArg !== null ? $fn($value, $filterArg) : $fn($value);
                }

                return htmlspecialchars((string)($value ?? ''), ENT_QUOTES, 'UTF-8');
            },
            $template
        );

        // แทนที่ {{{ unescaped }}} (raw output)
        $result = preg_replace_callback(
            '/\{\{\{\s*([a-zA-Z0-9_.]+)\s*\}\}\}/',
            function($matches) {
                return (string)($this->getValue($matches[1]) ?? '');
            },
            $result
        );

        // จัดการ @if ... @endif
        $result = $this->processConditionals($result);

        // จัดการ @foreach ... @endforeach
        $result = $this->processLoops($result);

        return $result;
    }

    private function getValue(string $path): mixed
    {
        $parts = explode('.', $path);
        $value = $this->data;

        foreach ($parts as $part) {
            if (is_array($value) && isset($value[$part])) {
                $value = $value[$part];
            } elseif (is_object($value) && isset($value->$part)) {
                $value = $value->$part;
            } else {
                return null;
            }
        }

        return $value;
    }

    private function processConditionals(string $template): string
    {
        return preg_replace_callback(
            '/@if\(([^)]+)\)(.*?)(?:@else(.*?))?@endif/s',
            function($matches) {
                $condition = $matches[1];
                $ifBlock   = $matches[2];
                $elseBlock = $matches[3] ?? '';

                // ตรวจสอบเงื่อนไขอย่างง่าย
                $varName = trim($condition);
                $value   = $this->getValue($varName);

                return $value ? $ifBlock : $elseBlock;
            },
            $template
        );
    }

    private function processLoops(string $template): string
    {
        return preg_replace_callback(
            '/@foreach\(([a-zA-Z0-9_.]+)\s+as\s+\$([a-zA-Z_]+)\)(.*?)@endforeach/s',
            function($matches) {
                $arrayName = $matches[1];
                $itemName  = $matches[2];
                $body      = $matches[3];

                $items = $this->getValue($arrayName);
                if (!is_array($items)) return '';

                $result = '';
                foreach ($items as $index => $item) {
                    $this->data[$itemName]         = $item;
                    $this->data[$itemName . '_index'] = $index;
                    $result .= $this->render($body);
                }

                unset($this->data[$itemName], $this->data[$itemName . '_index']);
                return $result;
            },
            $template
        );
    }
}

// ===== ทดสอบ Template Engine =====

$engine = new TemplateEngine();

$engine->assign([
    'title'        => 'รายงานยอดขาย',
    'company'      => 'Tech Corp',
    'report_date'  => '2024-12-25',
    'total_revenue' => 152500.75,
    'is_profit'    => true,
    'products'     => [
        ['name' => 'iPhone 15',   'price' => 32900, 'sold' => 50],
        ['name' => 'MacBook Pro', 'price' => 79900, 'sold' => 15],
        ['name' => 'AirPods',     'price' => 8900,  'sold' => 120],
    ],
    'note'         => "ยอดขายเพิ่มขึ้นจากเดือนที่แล้ว\nโปรดดูรายละเอียดเพิ่มเติม",
]);

$template = <<<'TEMPLATE'
===========================================
{{ title|upper }}
{{ company }} | วันที่: {{ report_date|date }}
===========================================

รายการสินค้า:
@foreach(products as $product)
  - {{ product.name }}: {{ product.price|currency }} × {{ product.sold }} ชิ้น
@endforeach

รวมรายได้ทั้งหมด: {{ total_revenue|currency }}

@if(is_profit)
สถานะ: กำไร ✅
@else
สถานะ: ขาดทุน ❌
@endif

หมายเหตุ:
{{{ note }}}

===========================================
TEMPLATE;

echo $engine->render($template);
```

---

## 🔧 Workshop 2: Text Processor

```php
<?php
/**
 * Text Processor
 * Workshop: PHP Strings
 */
class TextProcessor
{
    /**
     * สรุปข้อความ (ตัดเหลือ N ประโยค)
     */
    public static function summarize(string $text, int $sentences = 3): string
    {
        // แยกประโยค
        $sentencePattern = '/(?<=[.!?])\s+(?=[A-Z฀-๿])/u';
        $parts = preg_split($sentencePattern, trim($text), -1, PREG_SPLIT_NO_EMPTY);

        if (count($parts) <= $sentences) {
            return $text;
        }

        return implode(' ', array_slice($parts, 0, $sentences));
    }

    /**
     * Extract URLs จากข้อความ
     */
    public static function extractUrls(string $text): array
    {
        $pattern = '/https?:\/\/([\da-z\.-]+)\.([a-z\.]{2,6})([\/\w \.-]*\/?)?/i';
        preg_match_all($pattern, $text, $matches);
        return array_unique($matches[0]);
    }

    /**
     * Extract email addresses
     */
    public static function extractEmails(string $text): array
    {
        $pattern = '/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/';
        preg_match_all($pattern, $text, $matches);
        return array_unique($matches[0]);
    }

    /**
     * Highlight คำใน HTML
     */
    public static function highlight(string $text, array $words, string $tag = 'mark'): string
    {
        foreach ($words as $word) {
            $escaped = preg_quote($word, '/');
            $text = preg_replace(
                "/($escaped)/iu",
                "<$tag>$1</$tag>",
                $text
            );
        }
        return $text;
    }

    /**
     * นับความถี่คำ
     */
    public static function wordFrequency(string $text, int $minLength = 3): array
    {
        // แยกคำ (รองรับทั้งไทยและอังกฤษ)
        preg_match_all('/[\w\x{0E00}-\x{0E7F}]+/u', mb_strtolower($text), $matches);
        $words = $matches[0];

        // กรองคำสั้น
        $words = array_filter($words, fn($w) => mb_strlen($w) >= $minLength);

        // นับความถี่
        $freq = array_count_values($words);

        // เรียงจากมากไปน้อย
        arsort($freq);

        return $freq;
    }

    /**
     * แปลง Markdown เป็น HTML (อย่างง่าย)
     */
    public static function markdownToHtml(string $markdown): string
    {
        $html = htmlspecialchars($markdown, ENT_QUOTES, 'UTF-8');

        // Headers
        $html = preg_replace('/^### (.+)$/m', '<h3>$1</h3>', $html);
        $html = preg_replace('/^## (.+)$/m',  '<h2>$1</h2>', $html);
        $html = preg_replace('/^# (.+)$/m',   '<h1>$1</h1>', $html);

        // Bold และ Italic
        $html = preg_replace('/\*\*(.+?)\*\*/s', '<strong>$1</strong>', $html);
        $html = preg_replace('/\*(.+?)\*/s',     '<em>$1</em>', $html);
        $html = preg_replace('/`(.+?)`/s',        '<code>$1</code>', $html);

        // Links
        $html = preg_replace('/\[(.+?)\]\((.+?)\)/', '<a href="$2">$1</a>', $html);

        // Paragraphs
        $html = preg_replace('/\n{2,}/', '</p><p>', $html);
        $html = "<p>$html</p>";

        // Line breaks
        $html = preg_replace('/(?<!\>)\n(?!<)/', '<br>', $html);

        return $html;
    }

    /**
     * Mask ข้อมูลส่วนตัว
     */
    public static function maskSensitiveData(string $text): string
    {
        // Mask email (user@domain.com → u***@domain.com)
        $text = preg_replace_callback(
            '/([a-zA-Z0-9._%+-]+)@([a-zA-Z0-9.-]+\.[a-zA-Z]{2,})/',
            function($m) {
                $user = mb_substr($m[1], 0, 1) . str_repeat('*', mb_strlen($m[1]) - 1);
                return "$user@{$m[2]}";
            },
            $text
        );

        // Mask เบอร์โทร (0891234567 → 089****567)
        $text = preg_replace('/(\d{3})\d{4}(\d{3})/', '$1****$2', $text);

        // Mask บัตรเครดิต
        $text = preg_replace('/\b\d{4}[ -]?\d{4}[ -]?\d{4}[ -]?(\d{4})\b/', '****-****-****-$1', $text);

        return $text;
    }
}

// ===== ทดสอบ =====

echo "=== Extract URLs & Emails ===\n";
$text = "ติดต่อ admin@company.com หรือ support@help.org เว็บ https://company.com และ http://www.help.org/support";
echo "URLs: " . implode(', ', TextProcessor::extractUrls($text)) . "\n";
echo "Emails: " . implode(', ', TextProcessor::extractEmails($text)) . "\n";

echo "\n=== Word Frequency ===\n";
$article = "PHP is a popular scripting language. PHP is used for web development. PHP makes web development easy and fun.";
$freq = TextProcessor::wordFrequency($article);
$top5 = array_slice($freq, 0, 5, true);
foreach ($top5 as $word => $count) {
    echo "  '$word': $count ครั้ง\n";
}

echo "\n=== Markdown to HTML ===\n";
$markdown = "# หัวข้อหลัก\n\nนี่คือ **ข้อความหนา** และ *ตัวเอียง*\n\n## หัวข้อย่อย\n\nโค้ด `echo \"Hello\"`\n\n[คลิกที่นี่](https://example.com)";
echo TextProcessor::markdownToHtml($markdown);

echo "\n\n=== Mask Sensitive Data ===\n";
$sensitive = "อีเมล: somchai.thainame@company.com โทร: 0891234567 บัตร: 4532-1234-5678-9012";
echo TextProcessor::maskSensitiveData($sensitive);
```

---

## ❓ Quiz

### คำถามที่ 1
`strlen("สวัสดี")` ใน PHP จะได้ผลลัพธ์อะไร?

**ตัวเลือก:**
- a) 6
- b) 7
- c) 18
- d) ขึ้นอยู่กับ encoding

---

### คำถามที่ 2
ข้อใดเป็น Regex ที่ถูกต้องสำหรับตรวจสอบ email?

**ตัวเลือก:**
- a) `/email@domain/`
- b) `/^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/`
- c) `/\w+@\w+/`
- d) `/[email]/`

---

### คำถามที่ 3
ทำไมต้องใช้ `htmlspecialchars()` ก่อน output?

**ตัวเลือก:**
- a) ทำให้ HTML อ่านง่ายขึ้น
- b) ป้องกัน XSS attack โดยแปลงอักขระพิเศษ HTML เป็น entities
- c) บีบอัด HTML ให้เล็กลง
- d) แปลงภาษาไทยให้ถูกต้อง

---

### คำถามที่ 4
`str_contains`, `str_starts_with`, `str_ends_with` มีใน PHP เวอร์ชันใดขึ้นไป?

**ตัวเลือก:**
- a) PHP 7.0+
- b) PHP 7.4+
- c) PHP 8.0+
- d) PHP 8.1+

---

## ✅ เฉลย Quiz

**ข้อ 1: c) 18**
> ภาษาไทยใน UTF-8 ใช้ 3 bytes ต่อตัวอักษร
> "สวัสดี" = 6 ตัวอักษร × 3 bytes = 18 bytes
> ใช้ `mb_strlen("สวัสดี", 'UTF-8')` จะได้ 6 (ถูกต้อง)

**ข้อ 2: b) `/^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/`**
> - `^` ต้นบรรทัด, `$` ท้ายบรรทัด
> - `[a-zA-Z0-9._%+-]+` ส่วน username
> - `@` เครื่องหมาย @
> - `[a-zA-Z0-9.-]+` domain name
> - `\.[a-zA-Z]{2,}` TLD (2+ ตัวอักษร)

**ข้อ 3: b) ป้องกัน XSS attack**
> XSS (Cross-Site Scripting) คือการโจมตีที่ attacker ฝัง JavaScript ลงในหน้าเว็บ
> `htmlspecialchars()` แปลง `<script>` → `&lt;script&gt;` ทำให้ browser ไม่รัน JS

**ข้อ 4: c) PHP 8.0+**
> `str_contains()`, `str_starts_with()`, `str_ends_with()` เพิ่มใน PHP 8.0
> ก่อนหน้านั้นต้องใช้ `strpos() !== false` แทน `str_contains()`

---

## 🔗 สรุป String Functions

| ฟังก์ชัน | หน้าที่ |
|---------|---------|
| `strlen/mb_strlen` | นับความยาว |
| `strtoupper/lower` | แปลง case |
| `strpos/mb_strpos` | ค้นหาตำแหน่ง |
| `str_contains/starts_with/ends_with` | ตรวจสอบ (PHP 8.0+) |
| `substr/mb_substr` | ตัด substring |
| `str_replace/preg_replace` | แทนที่ |
| `trim/ltrim/rtrim` | ตัด whitespace |
| `explode/implode` | แยก/รวม |
| `sprintf/printf` | จัดรูปแบบ |
| `htmlspecialchars` | escape HTML (ความปลอดภัย) |
| `preg_match/match_all` | Regex ค้นหา |
| `preg_replace` | Regex แทนที่ |

---

## ➡️ Part ถัดไป

👉 **[Part 009: PHP OOP — Object-Oriented Programming](./part-009-php-oop.md)**

ใน Part หน้า เราจะเรียนรู้:
- Classes และ Objects
- Properties และ Methods
- Inheritance และ Polymorphism
- Interfaces และ Abstract Classes
- Traits
- Workshop: สร้าง E-commerce domain model
