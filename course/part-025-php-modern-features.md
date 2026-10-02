# 🚀 Part 25: PHP Modern Features - PHP 8.0 ถึง 8.3

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ฟีเจอร์ใหม่ใน PHP 8.0 (Named Arguments, Match, Nullsafe, Union Types, Attributes, Constructor Promotion) ได้
- ใช้ PHP 8.1 (Enums, Fibers, Intersection Types, readonly, never) ได้
- ใช้ PHP 8.2 (readonly classes, DNF Types, Constants in Traits) ได้
- ใช้ PHP 8.3 (Typed Class Constants, json_validate, Override attribute) ได้
- Modernize โค้ด PHP 7 ให้เป็น PHP 8.3 ได้

---

## 📌 1. PHP 8.0 Features

### 1.1 Named Arguments

```php
<?php
// แบบเดิม: ต้องจำลำดับ arguments
$result = array_slice($array, 1, 5, true);

// Named Arguments: เรียกด้วยชื่อ ไม่ต้องสนใจลำดับ
$result = array_slice(array: $array, offset: 1, length: 5, preserve_keys: true);

// ประโยชน์ชัดเจนกับ built-in functions ที่มี optional params มาก
function createUser(
    string $name,
    string $email,
    int $age = 0,
    bool $isActive = true,
    string $role = 'user',
    ?string $phone = null
): array {
    return compact('name', 'email', 'age', 'isActive', 'role', 'phone');
}

// แบบเดิม: ต้องส่ง default values ที่ไม่ต้องการ
$user = createUser('สมชาย', 'test@example.com', 0, true, 'admin');

// Named Arguments: ข้ามค่าที่ไม่ต้องการ
$user = createUser(
    name:  'สมชาย',
    email: 'test@example.com',
    role:  'admin'          // ข้าม age, isActive ที่มี default
);

// ผสม positional + named (positional ต้องมาก่อน)
$user = createUser('สมชาย', 'test@example.com', role: 'admin', phone: '0812345678');
```

### 1.2 Match Expression

```php
<?php
// switch แบบเดิม: loose comparison, ต้อง break
$status = 2;
switch ($status) {
    case 1:
        $label = 'รอดำเนินการ';
        break;
    case 2:
    case 3:
        $label = 'กำลังดำเนินการ';
        break;
    default:
        $label = 'ไม่ทราบสถานะ';
}

// match: strict comparison (===), return value, exception ถ้าไม่ match
$label = match($status) {
    1       => 'รอดำเนินการ',
    2, 3    => 'กำลังดำเนินการ', // comma-separated arms
    4       => 'เสร็จสิ้น',
    5       => 'ยกเลิก',
    default => throw new \UnexpectedValueException("สถานะไม่ถูกต้อง: {$status}"),
};

// match กับ expression ที่ซับซ้อน
$score = 75;
$grade = match(true) {
    $score >= 90 => 'A',
    $score >= 80 => 'B',
    $score >= 70 => 'C',
    $score >= 60 => 'D',
    default      => 'F',
};

echo $grade; // C

// match กับ no-match → UnhandledMatchError
try {
    $x = match(99) {
        1 => 'one',
        2 => 'two',
    };
} catch (\UnhandledMatchError $e) {
    echo "ไม่มี arm ที่ตรง: " . $e->getMessage() . "\n";
}
```

### 1.3 Nullsafe Operator (?->)

```php
<?php
class Country {
    public function __construct(public string $name, public string $code) {}
}

class Address {
    public function __construct(
        public string $street,
        public ?Country $country = null
    ) {}

    public function getCountry(): ?Country {
        return $this->country;
    }
}

class User {
    public function __construct(
        public string $name,
        public ?Address $address = null
    ) {}

    public function getAddress(): ?Address {
        return $this->address;
    }
}

$user = new User('สมชาย', new Address('ถนนสีลม', new Country('ไทย', 'TH')));
$userNoAddr = new User('สมหญิง'); // ไม่มี address

// แบบเดิม: ต้องตรวจ null ทุกขั้น
if ($user->getAddress() !== null) {
    $addr = $user->getAddress();
    if ($addr->getCountry() !== null) {
        echo $addr->getCountry()->name;
    }
}

// Nullsafe (?->): ถ้า null จะหยุดทันที คืน null
$countryName = $user->getAddress()?->getCountry()?->name;
echo $countryName ?? 'ไม่ระบุ'; // ไทย

$countryName = $userNoAddr->getAddress()?->getCountry()?->name;
echo $countryName ?? 'ไม่ระบุ'; // ไม่ระบุ (ไม่ throw error)

// ผสมกับ null coalescing
$code = $user->getAddress()?->getCountry()?->code ?? 'UNKNOWN';
echo $code; // TH
```

### 1.4 Union Types

```php
<?php
// PHP 8.0: Union types ใน parameter, return type, property
function processInput(int|string|array $input): string|false {
    if (is_array($input)) {
        return implode(', ', $input);
    }
    if (is_int($input)) {
        return "Number: {$input}";
    }
    return strlen($input) > 0 ? $input : false;
}

echo processInput(42);             // Number: 42
echo processInput("hello");        // hello
echo processInput(['a', 'b', 'c']); // a, b, c
var_dump(processInput(''));         // bool(false)

class DataStore {
    private int|string|null $id = null; // Union type property

    public function setId(int|string $id): void {
        $this->id = $id;
    }
}
```

### 1.5 Attributes (Annotations ใหม่)

```php
<?php
// กำหนด custom Attribute
#[\Attribute(\Attribute::TARGET_CLASS | \Attribute::TARGET_METHOD)]
class Route {
    public function __construct(
        public string $path,
        public string $method = 'GET',
        public array $middleware = []
    ) {}
}

#[\Attribute(\Attribute::TARGET_PROPERTY)]
class Validate {
    public function __construct(
        public string $rule,
        public string $message = ''
    ) {}
}

// ใช้ Attribute
#[Route('/users', 'GET', ['auth'])]
class UserController {
    #[Validate('required')]
    public string $name = '';

    #[Route('/users/{id}', 'GET')]
    public function show(int $id): array {
        return ['id' => $id];
    }

    #[Route('/users', 'POST', ['auth', 'admin'])]
    public function store(): array {
        return ['created' => true];
    }
}

// อ่าน Attributes ด้วย Reflection
$reflection = new \ReflectionClass(UserController::class);
$classAttrs = $reflection->getAttributes(Route::class);

foreach ($classAttrs as $attr) {
    $route = $attr->newInstance();
    echo "Class Route: {$route->method} {$route->path}\n";
}

foreach ($reflection->getMethods() as $method) {
    $methodAttrs = $method->getAttributes(Route::class);
    foreach ($methodAttrs as $attr) {
        $route = $attr->newInstance();
        echo "Method {$method->getName()}: {$route->method} {$route->path}\n";
        if (!empty($route->middleware)) {
            echo "  Middleware: " . implode(', ', $route->middleware) . "\n";
        }
    }
}
```

### 1.6 Constructor Promotion (ทบทวน)

```php
<?php
// PHP 8.0 Constructor Promotion
class Point {
    public function __construct(
        public readonly float $x,
        public readonly float $y,
        public readonly float $z = 0.0
    ) {}

    public function distanceTo(Point $other): float {
        return sqrt(
            ($this->x - $other->x) ** 2 +
            ($this->y - $other->y) ** 2 +
            ($this->z - $other->z) ** 2
        );
    }
}

$p1 = new Point(1.0, 2.0, 3.0);
$p2 = new Point(4.0, 6.0, 3.0);
echo $p1->distanceTo($p2); // 5
```

---

## 📌 2. PHP 8.1 Features

### 2.1 Enums

```php
<?php
// Pure Enum (ไม่มี value)
enum Status {
    case Pending;
    case Active;
    case Inactive;
    case Banned;

    public function label(): string {
        return match($this) {
            Status::Pending  => 'รอดำเนินการ',
            Status::Active   => 'ใช้งาน',
            Status::Inactive => 'ไม่ใช้งาน',
            Status::Banned   => 'ถูกแบน',
        };
    }

    public function canLogin(): bool {
        return $this === Status::Active;
    }
}

// Backed Enum (มี value เป็น string หรือ int)
enum Color: string {
    case Red   = 'red';
    case Green = 'green';
    case Blue  = 'blue';

    public static function fromLabel(string $label): self {
        return match(strtolower($label)) {
            'แดง'   => self::Red,
            'เขียว' => self::Green,
            'น้ำเงิน' => self::Blue,
            default => throw new \ValueError("ไม่รู้จักสี: {$label}"),
        };
    }
}

enum HttpStatus: int {
    case OK        = 200;
    case Created   = 201;
    case NotFound  = 404;
    case ServerError = 500;

    public function isSuccess(): bool {
        return $this->value >= 200 && $this->value < 300;
    }

    public function message(): string {
        return match($this) {
            self::OK          => 'OK',
            self::Created     => 'Created',
            self::NotFound    => 'Not Found',
            self::ServerError => 'Internal Server Error',
        };
    }
}

// ใช้งาน
$status = Status::Active;
echo $status->label();      // ใช้งาน
echo $status->canLogin() ? 'เข้าได้' : 'เข้าไม่ได้'; // เข้าได้

$color = Color::Red;
echo $color->value;         // red
echo Color::from('green')->name; // Green (from() คืน enum จาก value)
echo Color::tryFrom('purple') === null ? 'ไม่พบ' : 'พบ'; // ไม่พบ

$http = HttpStatus::from(404);
echo $http->message();      // Not Found
echo $http->isSuccess() ? 'สำเร็จ' : 'ล้มเหลว'; // ล้มเหลว

// Enum ใน match
function handleStatus(Status $s): string {
    return match($s) {
        Status::Active   => "ยินดีต้อนรับ!",
        Status::Pending  => "กรุณารอการอนุมัติ",
        Status::Inactive => "บัญชีถูกปิดใช้งาน",
        Status::Banned   => "บัญชีถูกแบน",
    };
}

// Enum implements Interface
interface HasDescription {
    public function description(): string;
}

enum Planet: int implements HasDescription {
    case Mercury = 1;
    case Venus   = 2;
    case Earth   = 3;
    case Mars    = 4;

    public function description(): string {
        return match($this) {
            self::Earth => "บ้านของเรา",
            self::Mars  => "ดาวแดง",
            default     => $this->name,
        };
    }
}
```

### 2.2 Readonly Properties

```php
<?php
class User {
    public readonly string $name;
    public readonly string $email;

    public function __construct(string $name, string $email) {
        $this->name  = $name;  // กำหนดได้ครั้งแรกใน constructor
        $this->email = $email;
    }
}

$user = new User('สมชาย', 'somchai@example.com');
echo $user->name;  // สมชาย

// $user->name = 'อื่น'; // Error: Cannot modify readonly property
```

### 2.3 Intersection Types

```php
<?php
interface Stringable {
    public function __toString(): string;
}

interface Countable {
    public function count(): int;
}

interface JsonSerializable {
    public function jsonSerialize(): mixed;
}

// Intersection type: ต้อง implement ทุก interface
function processCollection(Countable&\Stringable $collection): void {
    echo "จำนวน: " . $collection->count() . "\n";
    echo "ข้อมูล: " . $collection . "\n";
}

class StringCollection implements Countable, \Stringable {
    private array $items;

    public function __construct(array $items) {
        $this->items = $items;
    }

    public function count(): int { return count($this->items); }
    public function __toString(): string { return implode(', ', $this->items); }
}

$col = new StringCollection(['PHP', 'Laravel', 'MySQL']);
processCollection($col); // ทำงานได้
```

### 2.4 Fibers

```php
<?php
// Fiber: lightweight concurrency (cooperative multitasking)
$fiber = new Fiber(function (): void {
    echo "Fiber: เริ่มทำงาน\n";

    $value = Fiber::suspend('ค่าแรก'); // หยุดชั่วคราว ส่งค่ากลับ
    echo "Fiber: ได้รับ '{$value}' หลัง resume\n";

    $value2 = Fiber::suspend('ค่าสอง');
    echo "Fiber: ได้รับ '{$value2}' หลัง resume ครั้งที่ 2\n";

    echo "Fiber: จบการทำงาน\n";
});

$result1 = $fiber->start();         // เริ่ม fiber
echo "Main: ได้รับ '{$result1}'\n";  // ค่าแรก

$result2 = $fiber->resume('hello'); // resume ส่ง 'hello'
echo "Main: ได้รับ '{$result2}'\n";  // ค่าสอง

$fiber->resume('world');            // resume ส่ง 'world', fiber จบ
echo "Fiber status: " . ($fiber->isTerminated() ? 'จบแล้ว' : 'ยังทำงาน') . "\n";

// ตัวอย่าง Simple Event Loop ด้วย Fibers
class SimpleEventLoop {
    private array $fibers = [];

    public function add(Fiber $fiber): void {
        $this->fibers[] = $fiber;
    }

    public function run(): void {
        while (!empty($this->fibers)) {
            foreach ($this->fibers as $key => $fiber) {
                if ($fiber->isSuspended()) {
                    $fiber->resume();
                } elseif (!$fiber->isStarted()) {
                    $fiber->start();
                }

                if ($fiber->isTerminated()) {
                    unset($this->fibers[$key]);
                }
            }
        }
    }
}
```

### 2.5 never Return Type

```php
<?php
// never = function ไม่ return ค่าเลย (throw exception หรือ exit)
function throwError(string $message): never {
    throw new \RuntimeException($message);
}

function redirect(string $url): never {
    header("Location: {$url}");
    exit(0); // never returns
}

function abort(int $code, string $message): never {
    http_response_code($code);
    echo json_encode(['error' => $message]);
    exit(1);
}

// never ช่วย type checker รู้ว่าโค้ดหลัง call นี้ไม่ถูก execute
function findUserOrFail(int $id): array {
    $user = ['id' => 1, 'name' => 'test']; // simulate DB
    if (!$user) {
        throwError("User {$id} not found"); // never returns
    }
    return $user; // type checker รู้ว่า $user ไม่ null ที่นี่
}
```

### 2.6 array_is_list()

```php
<?php
// ตรวจสอบว่า array เป็น list (sequential integer keys 0,1,2,...)
var_dump(array_is_list([1, 2, 3]));           // true
var_dump(array_is_list(['a', 'b', 'c']));     // true
var_dump(array_is_list([]));                  // true (empty = list)
var_dump(array_is_list(['a' => 1, 'b' => 2])); // false (string keys)
var_dump(array_is_list([0 => 'a', 2 => 'b'])); // false (gap in keys)
var_dump(array_is_list([1 => 'a', 0 => 'b'])); // false (wrong order)

function jsonEncodeSafe(mixed $data): string {
    if (is_array($data) && !array_is_list($data)) {
        // Associative array → JSON object
        return json_encode((object) $data);
    }
    return json_encode($data);
}
```

---

## 📌 3. PHP 8.2 Features

### 3.1 Readonly Classes

```php
<?php
// PHP 8.2: readonly class = ทุก property เป็น readonly อัตโนมัติ
readonly class Money {
    public function __construct(
        public float $amount,
        public string $currency
    ) {}

    public function add(Money $other): self {
        if ($this->currency !== $other->currency) {
            throw new \InvalidArgumentException("สกุลเงินต่างกัน");
        }
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function multiply(float $factor): self {
        return new self($this->amount * $factor, $this->currency);
    }

    public function __toString(): string {
        return number_format($this->amount, 2) . ' ' . $this->currency;
    }
}

$price     = new Money(100.0, 'THB');
$tax       = new Money(7.0, 'THB');
$total     = $price->add($tax);
$withPromo = $total->multiply(0.9); // 10% discount

echo $price;     // 100.00 THB
echo $total;     // 107.00 THB
echo $withPromo; // 96.30 THB

// $price->amount = 200; // Error: readonly
```

### 3.2 DNF Types (Disjunctive Normal Form)

```php
<?php
interface Serializable { public function serialize(): string; }
interface Loggable { public function log(): void; }
interface Cacheable { public function cache(): void; }

// DNF: (A&B)|C = (Serializable AND Loggable) OR Cacheable
function processItem((Serializable&Loggable)|Cacheable $item): void {
    if ($item instanceof Cacheable) {
        $item->cache();
    } else {
        // $item เป็นทั้ง Serializable และ Loggable แน่ๆ ตรงนี้
        $item->serialize();
        $item->log();
    }
}
```

### 3.3 Constants in Traits

```php
<?php
// PHP 8.2: Traits สามารถมี constants ได้
trait HasVersion {
    public const VERSION = '1.0.0'; // ใน PHP 8.2+

    public function getVersion(): string {
        return self::VERSION;
    }
}

class App {
    use HasVersion;
}

echo App::VERSION;          // 1.0.0
echo (new App)->getVersion(); // 1.0.0
```

---

## 📌 4. PHP 8.3 Features

### 4.1 Typed Class Constants

```php
<?php
// PHP 8.3: กำหนด type ให้ class constant ได้
class Config {
    public const string VERSION     = '2.0.0';
    public const int    MAX_RETRIES = 3;
    public const float  TIMEOUT     = 30.5;
    public const bool   DEBUG       = false;
    public const array  DRIVERS     = ['mysql', 'pgsql', 'sqlite'];
}

interface HasVersion {
    public const string VERSION = '1.0'; // typed constant ใน interface
}

class App implements HasVersion {
    public const string VERSION = '2.0'; // override ต้องใช้ type เดิม
}

echo Config::VERSION;     // 2.0.0
echo Config::MAX_RETRIES; // 3
```

### 4.2 json_validate()

```php
<?php
// PHP 8.3: ตรวจสอบ JSON โดยไม่ต้อง decode
$validJson   = '{"name": "สมชาย", "age": 25}';
$invalidJson = '{"name": "สมชาย", age: 25}'; // invalid (key ไม่มี quotes)
$arrayJson   = '[1, 2, 3]';

var_dump(json_validate($validJson));   // true
var_dump(json_validate($invalidJson)); // false
var_dump(json_validate($arrayJson));   // true

// ก่อน PHP 8.3 ต้องทำแบบนี้
function jsonValidateOld(string $json): bool {
    json_decode($json);
    return json_last_error() === JSON_ERROR_NONE;
}

// json_validate() เร็วกว่าเพราะไม่ต้อง parse/build ทั้ง tree
function validateApiPayload(string $body): array {
    if (!json_validate($body)) {
        throw new \InvalidArgumentException("Invalid JSON payload");
    }
    return json_decode($body, true); // decode ครั้งเดียวหลังจาก validate ผ่านแล้ว
}
```

### 4.3 Override Attribute

```php
<?php
// PHP 8.3: #[Override] บอกชัดว่า method นี้ override parent
class ParentClass {
    public function calculate(): int {
        return 1;
    }
}

class ChildClass extends ParentClass {
    #[\Override] // PHP จะ error ถ้า parent ไม่มี method นี้
    public function calculate(): int {
        return parent::calculate() * 2;
    }

    // #[\Override]
    // public function calculateX(): int { // Error: ไม่มีใน parent
    //     return 0;
    // }
}
```

---

## 🛠️ Workshop: Modernize Legacy PHP 7 → PHP 8.3

```php
<?php
// ======= โค้ดเดิม PHP 7 (Legacy) =======

class OrderLegacy {
    /** @var int */
    private $id;
    /** @var string */
    private $status;
    /** @var float */
    private $total;
    /** @var string|null */
    private $customerEmail;
    /** @var \DateTime */
    private $createdAt;

    public function __construct($id, $status, $total, $customerEmail = null) {
        $this->id            = $id;
        $this->status        = $status;
        $this->total         = $total;
        $this->customerEmail = $customerEmail;
        $this->createdAt     = new \DateTime();
    }

    public function getStatus() {
        return $this->status;
    }

    public function setStatus($status) {
        $allowed = ['pending', 'processing', 'shipped', 'delivered', 'cancelled'];
        if (!in_array($status, $allowed)) {
            throw new \InvalidArgumentException("สถานะไม่ถูกต้อง: " . $status);
        }
        $this->status = $status;
    }

    public function getTotal() {
        return $this->total;
    }

    public function getCustomerEmail() {
        return $this->customerEmail;
    }

    public function getStatusLabel() {
        switch ($this->status) {
            case 'pending': return 'รอดำเนินการ';
            case 'processing': return 'กำลังดำเนินการ';
            case 'shipped': return 'จัดส่งแล้ว';
            case 'delivered': return 'ส่งถึงแล้ว';
            case 'cancelled': return 'ยกเลิก';
            default: return 'ไม่ทราบ';
        }
    }

    public function canCancel() {
        return in_array($this->status, ['pending', 'processing']);
    }
}

function processOrderLegacy($order, $discountCode = null) {
    $total = $order->getTotal();

    if ($discountCode !== null) {
        if ($discountCode === 'SAVE10') {
            $total = $total * 0.9;
        } elseif ($discountCode === 'SAVE20') {
            $total = $total * 0.8;
        }
    }

    $email = $order->getCustomerEmail();
    if ($email !== null) {
        $emailDomain = explode('@', $email);
        if (isset($emailDomain[1])) {
            $domain = $emailDomain[1];
        } else {
            $domain = null;
        }
    } else {
        $domain = null;
    }

    return [
        'total'  => $total,
        'domain' => $domain,
        'status' => $order->getStatusLabel(),
    ];
}
```

```php
<?php
// ======= โค้ดใหม่ PHP 8.3 (Modernized) =======
declare(strict_types=1);

// Enum แทน string constants
enum OrderStatus: string {
    case Pending    = 'pending';
    case Processing = 'processing';
    case Shipped    = 'shipped';
    case Delivered  = 'delivered';
    case Cancelled  = 'cancelled';

    public function label(): string {
        return match($this) {
            self::Pending    => 'รอดำเนินการ',
            self::Processing => 'กำลังดำเนินการ',
            self::Shipped    => 'จัดส่งแล้ว',
            self::Delivered  => 'ส่งถึงแล้ว',
            self::Cancelled  => 'ยกเลิก',
        };
    }

    public function canCancel(): bool {
        return in_array($this, [self::Pending, self::Processing]);
    }

    public function canTransitionTo(OrderStatus $newStatus): bool {
        return match($this) {
            self::Pending    => in_array($newStatus, [self::Processing, self::Cancelled]),
            self::Processing => in_array($newStatus, [self::Shipped, self::Cancelled]),
            self::Shipped    => $newStatus === self::Delivered,
            default          => false,
        };
    }
}

enum DiscountCode: string {
    case Save10 = 'SAVE10';
    case Save20 = 'SAVE20';
    case VIP30  = 'VIP30';

    public function multiplier(): float {
        return match($this) {
            self::Save10 => 0.9,
            self::Save20 => 0.8,
            self::VIP30  => 0.7,
        };
    }
}

// readonly class สำหรับ Value Objects ที่ immutable
readonly class OrderId {
    public string $value;

    public function __construct(int|string $id) {
        $this->value = (string) $id;
    }

    public function __toString(): string {
        return $this->value;
    }
}

// Typed Class Constants (PHP 8.3)
class OrderConfig {
    public const int    MAX_ITEMS    = 100;
    public const float  MIN_AMOUNT   = 1.0;
    public const string DATE_FORMAT  = 'Y-m-d H:i:s';
}

// Modern Order class
class Order {
    private OrderStatus $status;
    private \DateTimeImmutable $createdAt;

    public function __construct(
        private readonly OrderId $id,          // readonly property
        OrderStatus $status,
        private float $total,
        private readonly ?string $customerEmail = null
    ) {
        if ($total < OrderConfig::MIN_AMOUNT) {
            throw new \InvalidArgumentException(
                "ยอดสั่งซื้อต้องมากกว่า " . OrderConfig::MIN_AMOUNT . " THB"
            );
        }

        $this->status    = $status;
        $this->createdAt = new \DateTimeImmutable();
    }

    public function getId(): OrderId { return $this->id; }
    public function getStatus(): OrderStatus { return $this->status; }
    public function getTotal(): float { return $this->total; }
    public function getCustomerEmail(): ?string { return $this->customerEmail; }

    public function transitionTo(OrderStatus $newStatus): static {
        if (!$this->status->canTransitionTo($newStatus)) {
            throw new \LogicException(
                "ไม่สามารถเปลี่ยนจาก {$this->status->label()} เป็น {$newStatus->label()}"
            );
        }
        $new = clone $this;
        $new->status = $newStatus;
        return $new; // immutable style: คืน object ใหม่
    }

    public function applyDiscount(DiscountCode $code): static {
        $new        = clone $this;
        $new->total = round($this->total * $code->multiplier(), 2);
        return $new;
    }
}

// Modern function ด้วย Named Args + Nullsafe + Match
function processOrder(
    Order $order,
    ?string $discountCodeStr = null
): array {
    $order = $discountCodeStr !== null
        ? $order->applyDiscount(
            DiscountCode::tryFrom($discountCodeStr)
            ?? throw new \InvalidArgumentException("โค้ดส่วนลดไม่ถูกต้อง")
          )
        : $order;

    // Nullsafe + null coalescing
    $emailDomain = $order->getCustomerEmail() !== null
        ? explode('@', $order->getCustomerEmail())[1] ?? null
        : null;

    return [
        'order_id' => (string) $order->getId(),
        'total'    => $order->getTotal(),
        'domain'   => $emailDomain,
        'status'   => $order->getStatus()->label(),
    ];
}

// ======= ทดสอบ =======
$order = new Order(
    id:            new OrderId(1001),
    status:        OrderStatus::Pending,
    total:         1500.00,
    customerEmail: 'customer@gmail.com'
);

// State machine
$order2 = $order->transitionTo(OrderStatus::Processing);
$order3 = $order2->transitionTo(OrderStatus::Shipped);

echo $order->getStatus()->label();  // รอดำเนินการ
echo $order3->getStatus()->label(); // จัดส่งแล้ว

// ลอง transition ที่ไม่ถูกต้อง
try {
    $order->transitionTo(OrderStatus::Delivered); // ข้ามขั้น
} catch (\LogicException $e) {
    echo "Error: " . $e->getMessage() . "\n";
}

// ใช้ discount
$discounted = $order->applyDiscount(DiscountCode::Save20);
echo $order->getTotal();      // 1500 (ไม่เปลี่ยน)
echo $discounted->getTotal(); // 1200 (ลด 20%)

// json_validate (PHP 8.3)
$payload = json_encode(['order_id' => 1001, 'action' => 'confirm']);
if (json_validate($payload)) {
    $data = json_decode($payload, true);
    echo "Valid JSON: order_id = " . $data['order_id'] . "\n";
}

// processOrder
$result = processOrder($discounted, 'SAVE10');
print_r($result);
```

---

## ❓ Quiz

### ข้อที่ 1
Match expression ต่างจาก switch statement อย่างไรในด้าน type safety?

- A. Match ใช้ loose comparison (==) เหมือน switch
- B. Match ใช้ strict comparison (===) ทำให้ไม่เกิด type coercion และ throw UnhandledMatchError ถ้าไม่มี arm ตรง
- C. Match รองรับ fall-through เหมือน switch
- D. Match ต้องการ `break` เหมือน switch

**เฉลย: B**
Match ใช้ `===` ทำให้ `match("1")` ไม่ match กับ `1` (ต่างจาก switch ที่ใช้ `==`). ถ้าไม่มี arm ตรงและไม่มี `default` จะ throw `UnhandledMatchError` ต่างจาก switch ที่เงียบๆ

---

### ข้อที่ 2
Backed Enum ต่างจาก Pure Enum อย่างไร?

- A. Pure Enum มี value เป็น int/string แต่ Backed Enum ไม่มี
- B. Backed Enum มีค่าที่กำหนดได้ (int หรือ string) และใช้ from()/tryFrom() ได้ แต่ Pure Enum ไม่มีค่า
- C. ทั้งสองใช้แทนกันได้
- D. Backed Enum ใช้ interface ไม่ได้

**เฉลย: B**
Pure Enum (`enum Status { case Active; }`) ไม่มีค่า. Backed Enum (`enum Color: string { case Red = 'red'; }`) มีค่า int/string และเปิดใช้ `from()` / `tryFrom()` สำหรับ convert จากค่า

---

### ข้อที่ 3
Nullsafe Operator (?->) ต่างจาก ternary (?:) อย่างไร?

- A. เหมือนกันทุกประการ
- B. ?-> ใช้กับ method/property chaining หยุดทันทีที่ตัวซ้ายเป็น null คืน null แทน error ส่วน ?: คือ null coalescing
- C. ?-> ใช้ได้เฉพาะกับ property ไม่ใช้กับ method
- D. Nullsafe operator มีใน PHP 7.4 แล้ว

**เฉลย: B**
`$a?->b()?->c` หยุดที่ `$a` ถ้า null คืน null ทั้งสาย ไม่ throw error. `?:` คือ null coalescing ที่ใช้สำหรับ `$a ?? $default`

---

### ข้อที่ 4
readonly class ใน PHP 8.2 มีข้อจำกัดอะไรบ้าง?

- A. ไม่มีข้อจำกัด ใช้แทน class ปกติได้ทุกอย่าง
- B. properties ต้องมี type declaration ทุกตัว และไม่สามารถมี untyped/static properties
- C. readonly class extend class อื่นไม่ได้
- D. readonly class implement interface ไม่ได้

**เฉลย: B**
`readonly class` กำหนดให้ทุก property ต้องมี type declaration และไม่อนุญาต untyped properties หรือ static properties. readonly class extend class อื่นได้ถ้า parent ก็เป็น readonly class และ implement interface ได้ตามปกติ

---

### ข้อที่ 5
`json_validate()` ใน PHP 8.3 ให้ประโยชน์อะไรเพิ่มเติมจาก `json_decode()` + `json_last_error()`?

- A. ไม่มีความแตกต่าง ใช้วิธีเดิมดีกว่า
- B. `json_validate()` เร็วกว่าเพราะไม่ต้อง parse/build AST เต็มรูปแบบ และ semantics ชัดเจนกว่า
- C. `json_validate()` ตรวจ schema ด้วย
- D. `json_validate()` รองรับ JSON5 ด้วย

**เฉลย: B**
`json_validate()` เร็วกว่า `json_decode()` + `json_last_error()` เพราะทำแค่ validate syntax ไม่สร้าง PHP value tree ทำให้ประหยัด memory และ CPU โดยเฉพาะกับ JSON ขนาดใหญ่ที่ validate แล้วไม่ต้องใช้ทันที

---

## 🔗 ไปต่อ

➡️ **[Part 26: Laravel Installation - ติดตั้งและเริ่มต้น Laravel](part-026-laravel-installation.md)**

เรียนรู้การติดตั้ง Laravel ด้วย Composer, โครงสร้างโปรเจกต์, Artisan CLI และการ configure environment สำหรับการพัฒนา
