# Part 020: PHP Regular Expressions

## ระดับ: Intermediate
## เวลาเรียน: 3-4 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:
- เข้าใจ Regular Expression syntax พื้นฐานถึงขั้นสูง
- ใช้ PCRE functions ใน PHP ได้ (preg_match, preg_replace, preg_split, ฯลฯ)
- ใช้ Named captures
- ใช้ Lookahead และ Lookbehind
- สร้าง Form Validator ด้วย Regex

---

## 1. Regular Expression Syntax

### Metacharacters พื้นฐาน

```
.     - ตรงกับทุก character ยกเว้น newline
^     - จุดเริ่มต้นของ string/line
$     - จุดสิ้นสุดของ string/line
*     - 0 หรือมากกว่า
+     - 1 หรือมากกว่า
?     - 0 หรือ 1 ครั้ง (optional)
{n}   - n ครั้งพอดี
{n,}  - อย่างน้อย n ครั้ง
{n,m} - ระหว่าง n ถึง m ครั้ง
[...]  - Character class
[^...] - Negated character class
|     - OR
()    - Grouping / Capturing group
\     - Escape character
```

### Character Classes

```php
<?php
// Character Classes
$patterns = [
    '\d'   => 'ตัวเลข [0-9]',
    '\D'   => 'ไม่ใช่ตัวเลข',
    '\w'   => 'word character [a-zA-Z0-9_]',
    '\W'   => 'ไม่ใช่ word character',
    '\s'   => 'whitespace [ \t\n\r\f\v]',
    '\S'   => 'ไม่ใช่ whitespace',
    '\b'   => 'word boundary',
    '\B'   => 'ไม่ใช่ word boundary',
];

// ตัวอย่าง
$text = "สวัสดี Hello 123!";

// หาตัวเลข
preg_match_all('/\d+/', $text, $matches);
print_r($matches[0]); // ['123']

// หา word characters
preg_match_all('/\w+/', $text, $matches);
print_r($matches[0]); // ['Hello', '123']

// Custom character class
preg_match_all('/[aeiou]/', 'Hello World', $matches);
print_r($matches[0]); // ['e', 'o', 'o']

// Range
preg_match_all('/[a-z]/', 'Hello World', $matches);
print_r($matches[0]); // ['e', 'l', 'l', 'o', 'o', 'r', 'l', 'd']

// Negated
preg_match_all('/[^0-9]/', 'abc123', $matches);
print_r($matches[0]); // ['a', 'b', 'c']
```

### Quantifiers

```php
<?php
$examples = [
    // Greedy (ตรงกับมากที่สุดที่เป็นไปได้)
    'greedy_star'   => '/a.*b/',    // a + อะไรก็ได้ + b (ยาวที่สุด)
    'greedy_plus'   => '/a.+b/',    // a + อย่างน้อย 1 + b
    'greedy_q'      => '/colou?r/', // colour หรือ color
    'greedy_exact'  => '/\d{4}/',   // 4 ตัวเลขพอดี
    'greedy_range'  => '/\d{2,4}/', // 2-4 ตัวเลข
    
    // Lazy (ตรงกับน้อยที่สุดที่เป็นไปได้ - เติม ? หลัง quantifier)
    'lazy_star'     => '/a.*?b/',   // a + น้อยที่สุด + b
    'lazy_plus'     => '/a.+?b/',   // a + อย่างน้อย 1 (น้อยที่สุด) + b
];

// Greedy vs Lazy
$html = '<b>Bold</b> and <b>more bold</b>';

preg_match('/<b>.*<\/b>/', $html, $greedyMatch);
echo "Greedy: " . $greedyMatch[0] . "\n";
// <b>Bold</b> and <b>more bold</b> (ตรงกับทั้งหมด!)

preg_match('/<b>.*?<\/b>/', $html, $lazyMatch);
echo "Lazy: " . $lazyMatch[0] . "\n";
// <b>Bold</b> (ตรงกับน้อยที่สุด)

preg_match_all('/<b>.*?<\/b>/', $html, $allMatches);
print_r($allMatches[0]); // ['<b>Bold</b>', '<b>more bold</b>']
```

### Groups และ Alternation

```php
<?php
// Capturing Groups
$text = "2024-01-15";
preg_match('/(\d{4})-(\d{2})-(\d{2})/', $text, $matches);
echo "Full match: {$matches[0]}\n";  // 2024-01-15
echo "Year: {$matches[1]}\n";         // 2024
echo "Month: {$matches[2]}\n";        // 01
echo "Day: {$matches[3]}\n";          // 15

// Non-capturing Groups (?:...)
preg_match('/(?:\d{4})-(\d{2})-(\d{2})/', $text, $matches);
// Group 1 จะเป็น month แทน year

// Alternation
preg_match_all('/cat|dog|bird/', 'I have a cat and a dog', $matches);
print_r($matches[0]); // ['cat', 'dog']

// Alternation ใน group
preg_match_all('/(ph|f)one/', 'phone and fone are same', $matches);
print_r($matches[0]); // ['phone', 'fone']

// Nested Groups
$dateTime = "2024-01-15 10:30:00";
preg_match('/(\d{4}-(\d{2})-(\d{2})) (\d{2}:\d{2}:\d{2})/', $dateTime, $matches);
echo "Full date: {$matches[1]}\n";  // 2024-01-15
echo "Month: {$matches[2]}\n";       // 01
echo "Time: {$matches[4]}\n";        // 10:30:00
```

---

## 2. PCRE Functions ใน PHP

### preg_match()

```php
<?php
// preg_match - หาการ match ครั้งแรก
// คืน 1 ถ้าเจอ, 0 ถ้าไม่เจอ, false ถ้าเกิด error

$email = "user@example.com";
$pattern = '/^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/';

if (preg_match($pattern, $email, $matches)) {
    echo "Valid email: {$matches[0]}\n";
} else {
    echo "Invalid email\n";
}

// ดึงข้อมูลจาก URL
$url = "https://www.example.com:8080/path/to/page?q=test#section";
$urlPattern = '/^(https?):\/\/([^:\/]+)(?::(\d+))?(\/[^?#]*)(?:\?([^#]*))?(?:#(.*))?$/';

if (preg_match($urlPattern, $url, $parts)) {
    echo "Protocol: {$parts[1]}\n";   // https
    echo "Host: {$parts[2]}\n";       // www.example.com
    echo "Port: {$parts[3]}\n";       // 8080
    echo "Path: {$parts[4]}\n";       // /path/to/page
    echo "Query: {$parts[5]}\n";      // q=test
    echo "Fragment: {$parts[6]}\n";   // section
}
```

### preg_match_all()

```php
<?php
// preg_match_all - หาทุก match
// คืนจำนวน matches ที่เจอ

$html = '<a href="https://google.com">Google</a> and <a href="https://php.net">PHP</a>';

// ดึง href จาก links
$count = preg_match_all('/<a href="([^"]+)">([^<]+)<\/a>/', $html, $matches);

echo "Found {$count} links:\n";
for ($i = 0; $i < $count; $i++) {
    echo "URL: {$matches[1][$i]}, Text: {$matches[2][$i]}\n";
}

// PREG_SET_ORDER - จัดกลุ่ม match แบบ set
preg_match_all('/<a href="([^"]+)">([^<]+)<\/a>/', $html, $matches, PREG_SET_ORDER);
foreach ($matches as $match) {
    echo "URL: {$match[1]}, Text: {$match[2]}\n";
}

// ดึงทุก IP address
$log = "192.168.1.1 - 10.0.0.1 connected from 172.16.0.254";
preg_match_all('/\b(?:\d{1,3}\.){3}\d{1,3}\b/', $log, $ips);
print_r($ips[0]); // ['192.168.1.1', '10.0.0.1', '172.16.0.254']
```

### preg_replace()

```php
<?php
// preg_replace - แทนที่ด้วย string
$text = "Hello   World   PHP";
$cleaned = preg_replace('/\s+/', ' ', $text);
echo $cleaned . "\n"; // "Hello World PHP"

// แทนที่หลาย pattern พร้อมกัน
$dirty = '<script>alert("xss")</script> Hello <b>World</b>';
$clean = preg_replace(
    ['/\<script[^>]*\>.*?\<\/script\>/is', '/<[^>]+>/'],
    ['', ''],
    $dirty
);
echo $clean . "\n"; // " Hello World"

// ใช้ backreference ใน replacement
$date = "2024-01-15";
$formatted = preg_replace('/(\d{4})-(\d{2})-(\d{2})/', '$3/$2/$1', $date);
echo $formatted . "\n"; // 15/01/2024

// หมายเลขโทรศัพท์
$phone = "0812345678";
$formatted = preg_replace('/^(\d{3})(\d{3})(\d{4})$/', '$1-$2-$3', $phone);
echo $formatted . "\n"; // 081-234-5678

// preg_replace_callback - ใช้ function สำหรับ replacement ที่ซับซ้อน
$template = "สวัสดี [name]! คุณมี [count] ข้อความ";
$vars = ['name' => 'สมชาย', 'count' => '5'];

$result = preg_replace_callback('/\[(\w+)\]/', function($matches) use ($vars) {
    return $vars[$matches[1]] ?? $matches[0];
}, $template);

echo $result . "\n"; // สวัสดี สมชาย! คุณมี 5 ข้อความ
```

### preg_split()

```php
<?php
// preg_split - แยก string ด้วย regex

// แยกด้วย whitespace
$words = preg_split('/\s+/', "Hello   World   PHP");
print_r($words); // ['Hello', 'World', 'PHP']

// แยกด้วย delimiter หลายตัว
$csv = "apple, banana;cherry|date";
$fruits = preg_split('/[\s,;|]+/', $csv);
print_r($fruits); // ['apple', 'banana', 'cherry', 'date']

// PREG_SPLIT_DELIM_CAPTURE - เก็บ delimiter ด้วย
$text = "Hello World PHP";
$parts = preg_split('/(\s+)/', $text, -1, PREG_SPLIT_DELIM_CAPTURE);
print_r($parts); // ['Hello', ' ', 'World', ' ', 'PHP']

// PREG_SPLIT_NO_EMPTY - ข้าม empty strings
$text = "a,,b,,c";
$parts = preg_split('/,/', $text, -1, PREG_SPLIT_NO_EMPTY);
print_r($parts); // ['a', 'b', 'c']

// แยก camelCase เป็นคำ
$str = "helloWorldFooBar";
$words = preg_split('/(?=[A-Z])/', $str);
print_r($words); // ['hello', 'World', 'Foo', 'Bar']
```

### preg_grep() และ preg_quote()

```php
<?php
// preg_grep - กรอง array
$items = ["apple", "BANANA", "cherry", "Apricot", "blueberry"];

$aFruits = preg_grep('/^a/i', $items); // ขึ้นต้นด้วย a (case-insensitive)
print_r($aFruits); // ['apple', 'Apricot']

$notApple = preg_grep('/apple/i', $items, PREG_GREP_INVERT); // invert
print_r($notApple); // ['BANANA', 'cherry', 'Apricot', 'blueberry']

// preg_quote - escape special characters
$userInput = "Hello (World) + PHP.";
$escaped = preg_quote($userInput, '/'); // escape สำหรับ delimiter /
echo $escaped . "\n"; // Hello \(World\) \+ PHP\.

// ใช้งานจริง: ค้นหา user input ใน string
$searchTerm = "PHP.net";
$text = "Visit PHP.net for PHP documentation";
$pattern = '/' . preg_quote($searchTerm, '/') . '/i';
preg_match_all($pattern, $text, $matches);
echo "Found: " . count($matches[0]) . " times\n";
```

---

## 3. Named Captures

```php
<?php
// Named capture groups: (?P<name>pattern) หรือ (?<name>pattern)

// ดึงข้อมูลวันที่
$date = "2024-01-15";
preg_match('/(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})/', $date, $match);

echo "Year: {$match['year']}\n";   // 2024
echo "Month: {$match['month']}\n"; // 01
echo "Day: {$match['day']}\n";     // 15

// Named captures ทำให้โค้ดอ่านง่ายกว่า index
$pattern = '/^(?P<protocol>https?):\/\/(?P<host>[^\/]+)(?P<path>\/.*)?$/';
$url = "https://example.com/path/to/page";

if (preg_match($pattern, $url, $m)) {
    echo "Protocol: {$m['protocol']}\n";
    echo "Host: {$m['host']}\n";
    echo "Path: {$m['path']}\n";
}

// ใช้ named backreference ใน replacement
$text = "ชื่อ: สมชาย, นามสกุล: ใจดี";
$result = preg_replace(
    '/ชื่อ: (?P<first>[ก-ฮ]+), นามสกุล: (?P<last>[ก-ฮ]+)/',
    'ชื่อเต็ม: ${first} ${last}',
    $text
);
echo $result . "\n"; // ชื่อเต็ม: สมชาย ใจดี

// Named captures ใน preg_match_all
$log = "
2024-01-15 10:30:00 ERROR Database connection failed
2024-01-15 10:31:00 INFO User logged in
2024-01-15 10:32:00 WARNING Slow query detected
";

$pattern = '/(?P<date>\d{4}-\d{2}-\d{2}) (?P<time>\d{2}:\d{2}:\d{2}) (?P<level>\w+) (?P<message>.+)/';
preg_match_all($pattern, $log, $matches, PREG_SET_ORDER);

foreach ($matches as $entry) {
    echo "[{$entry['level']}] {$entry['date']} {$entry['time']}: {$entry['message']}\n";
}
```

---

## 4. Lookahead และ Lookbehind

### Lookahead

```php
<?php
// Positive Lookahead: (?=...) - ตรงกับ X ที่ตามด้วย Y
// Negative Lookahead: (?!...) - ตรงกับ X ที่ไม่ตามด้วย Y

$prices = "apple: $10, banana: $5, cherry: $15 (sale: $12)";

// ดึงตัวเลขที่ตามด้วย $ (ตัวเลขของราคา)
// รูปแบบ: ตัวเลขตามด้วย อาจมี .xx แต่ lookahead ไม่รวม $ ใน match
preg_match_all('/\d+(?=\b)/', $prices, $matches);
print_r($matches[0]);

// หาคำที่ตามด้วย "ing"
preg_match_all('/\w+(?=ing\b)/', 'running jumping coding', $matches);
print_r($matches[0]); // ['runn', 'jump', 'cod']

// Negative Lookahead: หา "is" ที่ไม่ตามด้วย "not"
$text = "PHP is great. PHP is not perfect. JavaScript is fun.";
preg_match_all('/is(?! not)/', $text, $matches, PREG_OFFSET_CAPTURE);
echo count($matches[0]) . " matches found\n"; // ได้ 'is great' และ 'is fun'
```

### Lookbehind

```php
<?php
// Positive Lookbehind: (?<=...) - ตรงกับ X ที่มี Y นำหน้า
// Negative Lookbehind: (?<!...) - ตรงกับ X ที่ไม่มี Y นำหน้า

$text = "Price: $100, Discount: 20%, Total: $80";

// ดึงตัวเลขที่มี $ นำหน้า
preg_match_all('/(?<=\$)\d+/', $text, $matches);
print_r($matches[0]); // ['100', '80']

// ดึงตัวเลขที่ไม่มี $ นำหน้า
preg_match_all('/(?<!\$)\b\d+/', $text, $matches);
print_r($matches[0]); // ['20']

// Lookbehind ใน replacement
$code = "function hello() { return 'world'; }";

// เพิ่ม semicolons หลัง string literals ที่อยู่ก่อน }
$result = preg_replace("/(?<='[^']*')\s*(?=})/", '; ', $code);

// Password validation: ต้องมีทั้ง uppercase, lowercase, และ digit
function validatePassword(string $password): bool {
    return preg_match('/(?=.*[A-Z])(?=.*[a-z])(?=.*\d).{8,}/', $password) === 1;
}

echo validatePassword("password") ? "Valid\n" : "Invalid\n";    // Invalid
echo validatePassword("Password1") ? "Valid\n" : "Invalid\n";   // Valid
echo validatePassword("p@ssW0rd") ? "Valid\n" : "Invalid\n";    // Valid
```

### ตัวอย่างขั้นสูง

```php
<?php
// ดึง credit card numbers (mask หลายรูปแบบ)
$texts = [
    "Card: 4111111111111111",
    "Card: 4111-1111-1111-1111",
    "Card: 4111 1111 1111 1111",
];

$pattern = '/\b(?:\d[ -]?){16}\b/';
foreach ($texts as $text) {
    preg_match($pattern, $text, $m);
    if ($m) {
        $card = preg_replace('/[\s-]/', '', $m[0]);
        echo "Found: " . substr($card, 0, 4) . " **** **** " . substr($card, -4) . "\n";
    }
}

// Extract variables จาก template engine
$template = "Hello {name}! You have {count} messages. Your email is {email}.";
preg_match_all('/\{(?P<var>[a-z_]+)\}/', $template, $matches, PREG_SET_ORDER);
foreach ($matches as $m) {
    echo "Variable: {$m['var']}\n";
}

// Semantic version parsing
$versions = ["1.2.3", "2.0.0-beta.1", "3.0.0-rc.2+build.456", "v4.1.0"];
$semverPattern = '/^v?(?P<major>\d+)\.(?P<minor>\d+)\.(?P<patch>\d+)(?:-(?P<pre>[a-zA-Z0-9.]+))?(?:\+(?P<build>[a-zA-Z0-9.]+))?$/';

foreach ($versions as $version) {
    if (preg_match($semverPattern, $version, $m)) {
        echo "Version: {$m[0]}\n";
        echo "  Major: {$m['major']}, Minor: {$m['minor']}, Patch: {$m['patch']}\n";
        if (!empty($m['pre'])) echo "  Pre-release: {$m['pre']}\n";
        if (!empty($m['build'])) echo "  Build: {$m['build']}\n";
    }
}
```

---

## 5. Regex Flags/Modifiers

```php
<?php
// Modifiers
// i - Case insensitive
// m - Multiline (^ และ $ match ทุก line)
// s - DOTALL (. match ทั้ง newline)
// x - Extended (whitespace ถูก ignore, # comments ได้)
// u - Unicode
// g - (ไม่มีใน PHP PCRE, ใช้ preg_match_all แทน)

// i - Case insensitive
preg_match_all('/php/i', 'PHP is PHP and php', $m);
echo count($m[0]) . " matches\n"; // 3

// m - Multiline
$text = "first line\nsecond line\nthird line";
preg_match_all('/^\w+/m', $text, $m);
print_r($m[0]); // ['first', 'second', 'third']

// s - DOTALL
$html = "<div>\nHello\nWorld\n</div>";
preg_match('/<div>(.*?)<\/div>/s', $html, $m);
echo $m[1] . "\n"; // \nHello\nWorld\n

// x - Extended (readable regex)
$pattern = '/
    ^               # Start of string
    (?P<year>\d{4}) # 4-digit year
    -               # Separator
    (?P<month>\d{2}) # 2-digit month
    -               # Separator
    (?P<day>\d{2})   # 2-digit day
    $               # End of string
/x';

preg_match($pattern, '2024-01-15', $m);
echo "Year: {$m['year']}, Month: {$m['month']}, Day: {$m['day']}\n";

// u - Unicode (สำหรับ UTF-8 strings)
$thai = "สวัสดี ชาวโลก";
preg_match_all('/\p{Thai}+/u', $thai, $m); // \p{Thai} = Thai Unicode block
print_r($m[0]);

// หาคำภาษาไทย
preg_match_all('/[\x{0E00}-\x{0E7F}]+/u', $thai, $m);
print_r($m[0]);
```

---

## 6. Workshop: สร้าง Form Validator ด้วย Regex

### โจทย์: Validator ที่สมบูรณ์

```php
<?php
class RegexValidator {
    private array $rules = [];
    private array $errors = [];
    private array $customMessages = [];
    
    // Regex Patterns
    private static array $patterns = [
        'email' => '/^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/',
        
        'thai_phone' => '/^(?:\+66|66|0)([89]\d{8}|[2-8]\d{7})$/',
        
        'url' => '/^https?:\/\/(?:[-\w]+\.)+[\w]{2,}(?:\/[-\w@.%+~,#?&=\/]*)?$/',
        
        'thai_id' => '/^\d{13}$/',
        
        'password_medium' => '/^(?=.*[A-Z])(?=.*[a-z])(?=.*\d).{8,}$/',
        
        'password_strong' => '/^(?=.*[A-Z])(?=.*[a-z])(?=.*\d)(?=.*[@$!%*?&]).{8,}$/',
        
        'credit_card' => '/^\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}$/',
        
        'ip_v4' => '/^(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/',
        
        'thai_text' => '/^[ก-ฮ\s]+$/',
        
        'username' => '/^[a-zA-Z][a-zA-Z0-9_]{2,19}$/',
        
        'slug' => '/^[a-z0-9]+(?:-[a-z0-9]+)*$/',
        
        'hex_color' => '/^#(?:[0-9a-fA-F]{3}){1,2}$/',
        
        'date_ymd' => '/^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$/',
        
        'time_hms' => '/^([01]\d|2[0-3]):[0-5]\d(:[0-5]\d)?$/',
        
        'postal_th' => '/^\d{5}$/',
    ];
    
    public function make(array $data, array $rules, array $messages = []): self {
        $validator = new self();
        $validator->data = $data;
        $validator->rules = $rules;
        $validator->customMessages = $messages;
        $validator->validate();
        return $validator;
    }
    
    private array $data = [];
    
    private function validate(): void {
        $this->errors = [];
        
        foreach ($this->rules as $field => $fieldRules) {
            $rules = is_string($fieldRules) 
                ? explode('|', $fieldRules) 
                : $fieldRules;
            
            $value = $this->data[$field] ?? null;
            
            foreach ($rules as $rule) {
                $this->applyRule($field, $value, $rule);
            }
        }
    }
    
    private function applyRule(string $field, mixed $value, string $rule): void {
        [$ruleName, $ruleParam] = array_pad(explode(':', $rule, 2), 2, null);
        
        $error = match($ruleName) {
            'required'    => $this->validateRequired($value),
            'email'       => $this->validateEmail($value),
            'phone'       => $this->validatePhone($value),
            'url'         => $this->validateUrl($value),
            'min'         => $this->validateMin($value, (int) $ruleParam),
            'max'         => $this->validateMax($value, (int) $ruleParam),
            'between'     => $this->validateBetween($value, $ruleParam),
            'min_length'  => $this->validateMinLength($value, (int) $ruleParam),
            'max_length'  => $this->validateMaxLength($value, (int) $ruleParam),
            'regex'       => $this->validateRegex($value, $ruleParam),
            'not_regex'   => $this->validateNotRegex($value, $ruleParam),
            'password'    => $this->validatePassword($value, $ruleParam ?? 'medium'),
            'credit_card' => $this->validateCreditCard($value),
            'thai_id'     => $this->validateThaiId($value),
            'ip'          => $this->validateIp($value),
            'date'        => $this->validateDate($value),
            'username'    => $this->validateUsername($value),
            'slug'        => $this->validateSlug($value),
            'alpha'       => $this->validateAlpha($value),
            'alpha_num'   => $this->validateAlphaNum($value),
            'numeric'     => $this->validateNumeric($value),
            'integer'     => $this->validateInteger($value),
            'in'          => $this->validateIn($value, $ruleParam),
            'not_in'      => $this->validateNotIn($value, $ruleParam),
            'same'        => $this->validateSame($value, $ruleParam),
            default       => null,
        };
        
        if ($error !== null) {
            $key = "{$field}.{$ruleName}";
            $this->errors[$field][] = $this->customMessages[$key] ?? $error;
        }
    }
    
    private function validateRequired(mixed $value): ?string {
        if (empty($value) && $value !== '0' && $value !== 0) {
            return "ข้อมูลนี้จำเป็นต้องกรอก";
        }
        return null;
    }
    
    private function validateEmail(mixed $value): ?string {
        if (empty($value)) return null;
        if (!preg_match(self::$patterns['email'], $value)) {
            return "รูปแบบ email ไม่ถูกต้อง";
        }
        return null;
    }
    
    private function validatePhone(mixed $value): ?string {
        if (empty($value)) return null;
        $cleaned = preg_replace('/[\s\-\(\)]/', '', $value);
        if (!preg_match(self::$patterns['thai_phone'], $cleaned)) {
            return "เบอร์โทรศัพท์ไม่ถูกต้อง (ต้องเป็นเบอร์ไทย)";
        }
        return null;
    }
    
    private function validateUrl(mixed $value): ?string {
        if (empty($value)) return null;
        if (!preg_match(self::$patterns['url'], $value)) {
            return "รูปแบบ URL ไม่ถูกต้อง";
        }
        return null;
    }
    
    private function validateMin(mixed $value, int $min): ?string {
        if (empty($value)) return null;
        if (is_numeric($value) && (float) $value < $min) {
            return "ค่าต้องมากกว่าหรือเท่ากับ {$min}";
        }
        return null;
    }
    
    private function validateMax(mixed $value, int $max): ?string {
        if (empty($value)) return null;
        if (is_numeric($value) && (float) $value > $max) {
            return "ค่าต้องน้อยกว่าหรือเท่ากับ {$max}";
        }
        return null;
    }
    
    private function validateBetween(mixed $value, string $params): ?string {
        [$min, $max] = explode(',', $params);
        if (!is_numeric($value)) return null;
        $v = (float) $value;
        if ($v < (float) $min || $v > (float) $max) {
            return "ค่าต้องอยู่ระหว่าง {$min} ถึง {$max}";
        }
        return null;
    }
    
    private function validateMinLength(mixed $value, int $min): ?string {
        if (empty($value)) return null;
        if (mb_strlen((string) $value) < $min) {
            return "ต้องมีความยาวอย่างน้อย {$min} ตัวอักษร";
        }
        return null;
    }
    
    private function validateMaxLength(mixed $value, int $max): ?string {
        if (empty($value)) return null;
        if (mb_strlen((string) $value) > $max) {
            return "ต้องมีความยาวไม่เกิน {$max} ตัวอักษร";
        }
        return null;
    }
    
    private function validateRegex(mixed $value, string $pattern): ?string {
        if (empty($value)) return null;
        if (!preg_match($pattern, (string) $value)) {
            return "รูปแบบข้อมูลไม่ถูกต้อง";
        }
        return null;
    }
    
    private function validateNotRegex(mixed $value, string $pattern): ?string {
        if (empty($value)) return null;
        if (preg_match($pattern, (string) $value)) {
            return "ข้อมูลมีรูปแบบที่ไม่อนุญาต";
        }
        return null;
    }
    
    private function validatePassword(mixed $value, string $strength): ?string {
        if (empty($value)) return null;
        $pattern = match($strength) {
            'strong' => self::$patterns['password_strong'],
            default  => self::$patterns['password_medium'],
        };
        
        if (!preg_match($pattern, $value)) {
            return $strength === 'strong'
                ? "รหัสผ่านต้องมีอย่างน้อย 8 ตัว มี A-Z, a-z, 0-9 และอักขระพิเศษ"
                : "รหัสผ่านต้องมีอย่างน้อย 8 ตัว มี A-Z, a-z และ 0-9";
        }
        return null;
    }
    
    private function validateCreditCard(mixed $value): ?string {
        if (empty($value)) return null;
        $cleaned = preg_replace('/[\s\-]/', '', $value);
        if (!preg_match('/^\d{16}$/', $cleaned)) {
            return "หมายเลขบัตรเครดิตไม่ถูกต้อง";
        }
        // Luhn algorithm
        if (!$this->luhnCheck($cleaned)) {
            return "หมายเลขบัตรเครดิตไม่ถูกต้อง (checksum failed)";
        }
        return null;
    }
    
    private function luhnCheck(string $number): bool {
        $sum = 0;
        $alt = false;
        for ($i = strlen($number) - 1; $i >= 0; $i--) {
            $n = (int) $number[$i];
            if ($alt) {
                $n *= 2;
                if ($n > 9) $n -= 9;
            }
            $sum += $n;
            $alt = !$alt;
        }
        return $sum % 10 === 0;
    }
    
    private function validateThaiId(mixed $value): ?string {
        if (empty($value)) return null;
        $id = preg_replace('/[\s\-]/', '', $value);
        if (!preg_match('/^\d{13}$/', $id)) {
            return "เลขบัตรประชาชนต้องมี 13 หลัก";
        }
        // Thai national ID checksum
        $sum = 0;
        for ($i = 0; $i < 12; $i++) {
            $sum += (int)$id[$i] * (13 - $i);
        }
        $check = (11 - ($sum % 11)) % 10;
        if ($check !== (int)$id[12]) {
            return "เลขบัตรประชาชนไม่ถูกต้อง";
        }
        return null;
    }
    
    private function validateIp(mixed $value): ?string {
        if (empty($value)) return null;
        if (!preg_match(self::$patterns['ip_v4'], $value)) {
            return "IP address ไม่ถูกต้อง";
        }
        return null;
    }
    
    private function validateDate(mixed $value): ?string {
        if (empty($value)) return null;
        if (!preg_match(self::$patterns['date_ymd'], $value)) {
            return "รูปแบบวันที่ไม่ถูกต้อง (YYYY-MM-DD)";
        }
        // ตรวจสอบวันที่จริง
        [$y, $m, $d] = explode('-', $value);
        if (!checkdate((int) $m, (int) $d, (int) $y)) {
            return "วันที่ไม่มีอยู่จริง";
        }
        return null;
    }
    
    private function validateUsername(mixed $value): ?string {
        if (empty($value)) return null;
        if (!preg_match(self::$patterns['username'], $value)) {
            return "Username ต้องขึ้นต้นด้วยตัวอักษร ความยาว 3-20 ตัว ใช้ได้แค่ a-z, 0-9, _";
        }
        return null;
    }
    
    private function validateSlug(mixed $value): ?string {
        if (empty($value)) return null;
        if (!preg_match(self::$patterns['slug'], $value)) {
            return "Slug ต้องเป็นตัวเล็กทั้งหมด คั่นด้วย - เท่านั้น";
        }
        return null;
    }
    
    private function validateAlpha(mixed $value): ?string {
        if (empty($value)) return null;
        if (!preg_match('/^[a-zA-Z]+$/', $value)) {
            return "ต้องเป็นตัวอักษรเท่านั้น";
        }
        return null;
    }
    
    private function validateAlphaNum(mixed $value): ?string {
        if (empty($value)) return null;
        if (!preg_match('/^[a-zA-Z0-9]+$/', $value)) {
            return "ต้องเป็นตัวอักษรและตัวเลขเท่านั้น";
        }
        return null;
    }
    
    private function validateNumeric(mixed $value): ?string {
        if (empty($value)) return null;
        if (!is_numeric($value)) {
            return "ต้องเป็นตัวเลขเท่านั้น";
        }
        return null;
    }
    
    private function validateInteger(mixed $value): ?string {
        if (empty($value)) return null;
        if (!preg_match('/^-?\d+$/', (string) $value)) {
            return "ต้องเป็นจำนวนเต็มเท่านั้น";
        }
        return null;
    }
    
    private function validateIn(mixed $value, string $list): ?string {
        if (empty($value)) return null;
        $allowed = explode(',', $list);
        if (!in_array($value, $allowed, true)) {
            return "ค่าต้องเป็นหนึ่งใน: {$list}";
        }
        return null;
    }
    
    private function validateNotIn(mixed $value, string $list): ?string {
        if (empty($value)) return null;
        $forbidden = explode(',', $list);
        if (in_array($value, $forbidden, true)) {
            return "ค่านี้ไม่ได้รับอนุญาต";
        }
        return null;
    }
    
    private function validateSame(mixed $value, string $field): ?string {
        if ($value !== ($this->data[$field] ?? null)) {
            return "ค่าต้องตรงกับ {$field}";
        }
        return null;
    }
    
    public function passes(): bool {
        return empty($this->errors);
    }
    
    public function fails(): bool {
        return !empty($this->errors);
    }
    
    public function getErrors(): array {
        return $this->errors;
    }
    
    public function getFirstError(string $field = ''): ?string {
        if ($field) {
            return $this->errors[$field][0] ?? null;
        }
        $first = reset($this->errors);
        return is_array($first) ? $first[0] : null;
    }
}

// ============================================================
// ทดสอบ Validator
// ============================================================

$validator = new RegexValidator();

// ทดสอบ Registration Form
$registerData = [
    'username'         => 'john_doe',
    'email'            => 'john@example.com',
    'password'         => 'MyPass123',
    'confirm_password' => 'MyPass123',
    'phone'            => '0812345678',
    'birth_date'       => '1990-05-15',
];

$v = $validator->make($registerData, [
    'username'         => 'required|username',
    'email'            => 'required|email',
    'password'         => 'required|password:medium|min_length:8',
    'confirm_password' => 'required|same:password',
    'phone'            => 'phone',
    'birth_date'       => 'required|date',
]);

if ($v->passes()) {
    echo "✓ Registration data is valid!\n";
} else {
    echo "✗ Validation failed:\n";
    foreach ($v->getErrors() as $field => $errors) {
        foreach ($errors as $error) {
            echo "  - {$field}: {$error}\n";
        }
    }
}

// ทดสอบ Invalid Data
echo "\n--- Testing Invalid Data ---\n";
$invalidData = [
    'username'         => '1invalid',      // ขึ้นต้นด้วยตัวเลข
    'email'            => 'not-an-email',
    'password'         => 'weak',           // สั้นและอ่อน
    'confirm_password' => 'different',
    'phone'            => '123',
    'birth_date'       => '2024-13-45',    // เดือนและวันไม่ถูก
];

$v2 = $validator->make($invalidData, [
    'username'         => 'required|username',
    'email'            => 'required|email',
    'password'         => 'required|password:medium',
    'confirm_password' => 'required|same:password',
    'phone'            => 'phone',
    'birth_date'       => 'required|date',
]);

foreach ($v2->getErrors() as $field => $errors) {
    foreach ($errors as $error) {
        echo "  - {$field}: {$error}\n";
    }
}
```

---

## Quiz

### คำถาม 1
Regex `/^Hello$/m` กับ `/^Hello$/` ต่างกันอย่างไร?
- A. ไม่ต่างกัน
- B. `/m` ทำให้ ^ และ $ match แต่ละ line แทนที่จะเป็น start/end ของ string ทั้งหมด
- C. `/m` ทำให้ case-insensitive
- D. `/m` ทำให้ . match newline ด้วย

**เฉลย: B** - `m` flag (multiline) เปลี่ยนพฤติกรรมของ `^` และ `$`

### คำถาม 2
`(?:...)` ต่างจาก `(...)` อย่างไร?
- A. `(?:...)` ทำงานเร็วกว่า เพราะไม่ capture
- B. `(?:...)` ไม่สร้าง backreference
- C. ถูกทั้ง A และ B
- D. ไม่ต่างกัน

**เฉลย: C** - Non-capturing group ทำงานเหมือน group แต่ไม่บันทึก match ลงใน `$matches`

### คำถาม 3
`/\bcat\b/` จะ match อะไรในประโยค "the cat sat on a catfish"?
- A. "cat" ทั้งสองตำแหน่ง
- B. แค่ "cat" ตัวแรก (แยกคำ)
- C. "catfish" ด้วย
- D. ไม่ match อะไร

**เฉลย: B** - `\b` คือ word boundary ดังนั้น "catfish" จะไม่ match

### คำถาม 4
Lookahead `(?=...)` กับ Lookbehind `(?<=...)` ต่างกันอย่างไร?

**เฉลย:**
- Lookahead `(?=Y)` - ตรวจสอบว่าข้างหน้า (ที่จะตามมา) คือ Y
- Lookbehind `(?<=Y)` - ตรวจสอบว่าข้างหลัง (ที่เพิ่งผ่านมา) คือ Y
- ทั้งคู่เป็น zero-width assertion (ไม่ consume characters)

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- **Regex Syntax** - metacharacters, quantifiers, character classes, groups
- **PCRE Functions** - preg_match, preg_match_all, preg_replace, preg_split, preg_grep
- **Named Captures** - `(?P<name>...)` ทำให้โค้ดอ่านง่าย
- **Lookahead/Lookbehind** - zero-width assertions สำหรับ context-dependent matching
- **Form Validator** - นำ regex ไปสร้าง validation system ที่ครอบคลุม

---

## ➡️ Part ถัดไป

[Part 021: PHP JSON และ REST API](./part-021-php-json-api.md)
