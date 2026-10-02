# Part 025: PHP Modern Features (PHP 8.0 - 8.3)

## ระดับ: Intermediate to Advanced
## เวลาเรียน: 5-6 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:
- ใช้ PHP 8.0 features: Named arguments, Match expression, Nullsafe operator, Attributes
- ใช้ PHP 8.1 features: Enums, Fibers, Intersection types, readonly properties
- ใช้ PHP 8.2 features: readonly classes, DNF types, Constants in traits
- ใช้ PHP 8.3 features: Typed class constants, Override attribute, json_validate()
- Modernize Legacy PHP Code

---

## 1. PHP 8.0 Features

### Named Arguments

```php
<?php
// ก่อน PHP 8.0: ต้องจำลำดับ arguments
$result = array_slice($array, 1, 5, true);

// PHP 8.0+: Named arguments - จำชื่อแทนลำดับ
$result = array_slice(array: $array, offset: 1, length: 5, preserve_keys: true);

// ข้ามค่าที่มี default
function createUser(
    string $name,
    string $email,
    string $role = 'user',
    bool $active = true,
    ?string $phone = null,
    ?string $avatar = null,
): array {
    return compact('name', 'email', 'role', 'active', 'phone', 'avatar');
}

// ก่อน PHP 8.0: ต้องใส่ทุก argument จนถึง argument ที่ต้องการ
$user = createUser('สมชาย', 'somchai@example.com', 'user', true, null, 'avatar.jpg');

// PHP 8.0+: ข้าม arguments ที่ไม่ต้องการ
$user = createUser(
    name: 'สมชาย',
    email: 'somchai@example.com',
    avatar: 'avatar.jpg' // ข้าม role, active, phone
);

// ประโยชน์: HTML functions ที่มี arguments เยอะ
$imgTag = htmlspecialchars(
    string: '<img src="test.jpg">',
    flags: ENT_QUOTES | ENT_HTML5,
    encoding: 'UTF-8',
    double_encode: false
);

// Named arguments กับ spread operator
function sum(int ...$nums): int {
    return array_sum($nums);
}

$args = ['a' => 1, 'b' => 2, 'c' => 3];
$result = sum(...$args); // spread named args
```

### Match Expression

```php
<?php
// ก่อน PHP 8.0: switch statement (verbose, ไม่ return value)
function getStatusLabelSwitch(string $status): string {
    switch ($status) {
        case 'active':
            $label = 'กำลังใช้งาน';
            break;
        case 'inactive':
            $label = 'ไม่ได้ใช้งาน';
            break;
        case 'pending':
            $label = 'รอดำเนินการ';
            break;
        default:
            $label = 'ไม่ทราบสถานะ';
    }
    return $label;
}

// PHP 8.0+: match expression (concise, strict comparison, returns value)
function getStatusLabel(string $status): string {
    return match($status) {
        'active'   => 'กำลังใช้งาน',
        'inactive' => 'ไม่ได้ใช้งาน',
        'pending'  => 'รอดำเนินการ',
        'deleted'  => 'ถูกลบแล้ว',
        default    => 'ไม่ทราบสถานะ',
    };
}

// Match กับ หลายค่า
function classify(int $score): string {
    return match(true) {
        $score >= 90 => 'A',
        $score >= 80 => 'B',
        $score >= 70 => 'C',
        $score >= 60 => 'D',
        default      => 'F',
    };
}

// Match กับ multiple conditions
function getIcon(string $type): string {
    return match($type) {
        'error', 'danger'           => '🔴',
        'warning', 'caution'        => '🟡',
        'success', 'ok', 'done'     => '🟢',
        'info', 'notice'            => '🔵',
        default                     => '⚪',
    };
}

// Match throw exception
function divide(float $a, float $b): float {
    return match(true) {
        $b == 0  => throw new \DivisionByZeroError('Cannot divide by zero'),
        default  => $a / $b,
    };
}

// Switch vs Match: ความแตกต่างหลัก
// switch: loose comparison (==), match: strict comparison (===)
$value = "0";
switch ($value) {
    case 0: echo "Switch: matched zero\n"; break; // จะ match! (0 == "0")
}

echo match($value) {
    0 => "Match: matched zero",
    "0" => "Match: matched string zero", // จะ match นี้! ("0" === "0")
    default => "no match",
};
```

### Nullsafe Operator

```php
<?php
// ก่อน PHP 8.0: ตรวจ null ทุกขั้น (verbose มาก)
$country = null;
if ($user !== null) {
    $address = $user->getAddress();
    if ($address !== null) {
        $city = $address->getCity();
        if ($city !== null) {
            $country = $city->getCountry();
        }
    }
}

// PHP 8.0+: Nullsafe operator ?->
$country = $user?->getAddress()?->getCity()?->getCountry();

// ตัวอย่างจริง
class User {
    private ?Address $address;
    private ?Profile $profile;
    
    public function __construct(
        public readonly string $name,
        public readonly string $email
    ) {}
    
    public function getAddress(): ?Address { return $this->address; }
    public function setAddress(?Address $address): void { $this->address = $address; }
    public function getProfile(): ?Profile { return $this->profile; }
}

class Address {
    public function __construct(
        public readonly string $street,
        public readonly ?City $city = null
    ) {}
    
    public function getCity(): ?City { return $this->city; }
}

class City {
    public function __construct(
        public readonly string $name,
        public readonly string $country = 'Thailand'
    ) {}
    
    public function getCountry(): string { return $this->country; }
}

class Profile {
    public function __construct(
        public readonly ?string $bio = null,
        public readonly ?string $avatar = null
    ) {}
    
    public function getAvatar(): ?string { return $this->avatar; }
}

// ใช้งาน
$user1 = new User("สมชาย", "somchai@example.com");
$user1->setAddress(new Address("123 Main St", new City("Bangkok")));

$user2 = new User("สมหญิง", "somying@example.com");
// ไม่มี address

echo $user1?->getAddress()?->getCity()?->getCountry() . "\n"; // Thailand
echo $user2?->getAddress()?->getCity()?->getCountry() . "\n"; // null (ไม่ throw!)

// Nullsafe กับ method calls
$avatar = $user2?->getProfile()?->getAvatar() ?? 'default.jpg';
echo "Avatar: {$avatar}\n"; // default.jpg

// ไม่สามารถใช้กับ assignment
// $user?->address = "new"; // Error! ใช้ไม่ได้
```

### Attributes (PHP 8.0)

```php
<?php
// Attributes = metadata ที่ attach กับ code (แทน DocBlock annotations)
use Attribute;

// สร้าง Attribute ของตัวเอง
#[Attribute(Attribute::TARGET_METHOD)]
class Route {
    public function __construct(
        public readonly string $path,
        public readonly string $method = 'GET'
    ) {}
}

#[Attribute(Attribute::TARGET_METHOD)]
class Middleware {
    public function __construct(
        public readonly string ...$middleware
    ) {}
}

#[Attribute(Attribute::TARGET_CLASS | Attribute::TARGET_METHOD)]
class Cache {
    public function __construct(
        public readonly int $ttl = 300,
        public readonly string $key = ''
    ) {}
}

#[Attribute(Attribute::TARGET_PROPERTY)]
class Column {
    public function __construct(
        public readonly string $name,
        public readonly string $type = 'varchar',
        public readonly int $length = 255,
        public readonly bool $nullable = false
    ) {}
}

// ใช้ Attributes
#[Cache(ttl: 60)]
class UserController {
    #[Route('/users', 'GET')]
    #[Middleware('auth', 'throttle:60,1')]
    #[Cache(ttl: 300, key: 'users_list')]
    public function index(): array {
        return [];
    }
    
    #[Route('/users/{id}', 'GET')]
    #[Middleware('auth')]
    public function show(int $id): array {
        return [];
    }
    
    #[Route('/users', 'POST')]
    #[Middleware('auth', 'admin')]
    public function store(array $data): array {
        return [];
    }
}

class User {
    #[Column('id', 'int')]
    public int $id;
    
    #[Column('username', 'varchar', 50)]
    public string $username;
    
    #[Column('email', 'varchar', 255, false)]
    public string $email;
    
    #[Column('bio', 'text', 65535, true)]
    public ?string $bio = null;
}

// อ่าน Attributes ด้วย Reflection
function getRoutes(string $className): array {
    $reflection = new ReflectionClass($className);
    $routes = [];
    
    foreach ($reflection->getMethods() as $method) {
        $routeAttrs = $method->getAttributes(Route::class);
        $middlewareAttrs = $method->getAttributes(Middleware::class);
        
        if (empty($routeAttrs)) continue;
        
        $route = $routeAttrs[0]->newInstance();
        $middlewares = array_map(
            fn($attr) => $attr->newInstance()->middleware,
            $middlewareAttrs
        );
        
        $routes[] = [
            'method' => $route->method,
            'path' => $route->path,
            'handler' => "{$className}::{$method->getName()}",
            'middleware' => array_merge(...$middlewares),
        ];
    }
    
    return $routes;
}

$routes = getRoutes(UserController::class);
foreach ($routes as $route) {
    echo "[{$route['method']}] {$route['path']} -> {$route['handler']}\n";
    if (!empty($route['middleware'])) {
        echo "  Middleware: " . implode(', ', $route['middleware']) . "\n";
    }
}
```

### Union Types และ Mixed Type (PHP 8.0)

```php
<?php
// PHP 8.0: Union types
function formatId(int|string $id): string {
    return is_int($id) ? "#{$id}" : $id;
}

echo formatId(42) . "\n";      // #42
echo formatId("abc-123") . "\n"; // abc-123

// Mixed type = any type
function process(mixed $value): mixed {
    return $value;
}

// Never return type = function ไม่ return หรือ throw เสมอ
function fail(string $message): never {
    throw new \RuntimeException($message);
}

function redirect(string $url): never {
    header("Location: {$url}");
    exit;
}

// PHP 8.0: Fibers (ใน 8.1 แต่แนะนำใน context นี้)
// str_contains, str_starts_with, str_ends_with
$str = "Hello World PHP";
echo str_contains($str, "World") ? "yes\n" : "no\n"; // yes
echo str_starts_with($str, "Hello") ? "yes\n" : "no\n"; // yes
echo str_ends_with($str, "PHP") ? "yes\n" : "no\n"; // yes
```

---

## 2. PHP 8.1 Features

### Enums

```php
<?php
// Pure Enum (ไม่มี value)
enum Status {
    case Active;
    case Inactive;
    case Pending;
    case Deleted;
    
    // Method ใน enum ได้
    public function label(): string {
        return match($this) {
            Status::Active   => 'กำลังใช้งาน',
            Status::Inactive => 'ไม่ได้ใช้งาน',
            Status::Pending  => 'รอดำเนินการ',
            Status::Deleted  => 'ถูกลบแล้ว',
        };
    }
    
    public function isActive(): bool {
        return $this === Status::Active;
    }
    
    public function color(): string {
        return match($this) {
            Status::Active   => 'green',
            Status::Inactive => 'gray',
            Status::Pending  => 'yellow',
            Status::Deleted  => 'red',
        };
    }
}

// Backed Enum (มี value - string หรือ int)
enum OrderStatus: string {
    case Pending   = 'pending';
    case Processing = 'processing';
    case Shipped   = 'shipped';
    case Delivered = 'delivered';
    case Cancelled = 'cancelled';
    
    // จาก value ไปหา enum
    public static function fromLabel(string $label): self {
        return match(strtolower($label)) {
            'รอดำเนินการ', 'pending'        => self::Pending,
            'กำลังประมวลผล', 'processing'   => self::Processing,
            'จัดส่งแล้ว', 'shipped'         => self::Shipped,
            'ได้รับแล้ว', 'delivered'       => self::Delivered,
            'ยกเลิก', 'cancelled'           => self::Cancelled,
            default => throw new \ValueError("Unknown status: {$label}"),
        };
    }
    
    public function isTerminal(): bool {
        return in_array($this, [self::Delivered, self::Cancelled]);
    }
    
    public function canTransitionTo(self $next): bool {
        return match($this) {
            self::Pending     => in_array($next, [self::Processing, self::Cancelled]),
            self::Processing  => in_array($next, [self::Shipped, self::Cancelled]),
            self::Shipped     => in_array($next, [self::Delivered]),
            self::Delivered,
            self::Cancelled   => false,
        };
    }
    
    public function label(): string {
        return match($this) {
            self::Pending    => 'รอดำเนินการ',
            self::Processing => 'กำลังประมวลผล',
            self::Shipped    => 'จัดส่งแล้ว',
            self::Delivered  => 'ได้รับแล้ว',
            self::Cancelled  => 'ยกเลิก',
        };
    }
}

// Int backed enum
enum Priority: int {
    case Low      = 1;
    case Medium   = 2;
    case High     = 3;
    case Critical = 4;
    
    public function isHigherThan(self $other): bool {
        return $this->value > $other->value;
    }
}

// Enum implements Interface
interface HasColor {
    public function color(): string;
}

enum Suit: string implements HasColor {
    case Hearts   = 'H';
    case Diamonds = 'D';
    case Clubs    = 'C';
    case Spades   = 'S';
    
    public function color(): string {
        return match($this) {
            Suit::Hearts, Suit::Diamonds => 'red',
            Suit::Clubs, Suit::Spades    => 'black',
        };
    }
    
    public function symbol(): string {
        return match($this) {
            Suit::Hearts   => '♥',
            Suit::Diamonds => '♦',
            Suit::Clubs    => '♣',
            Suit::Spades   => '♠',
        };
    }
}

// ใช้งาน
$status = Status::Active;
echo $status->label() . "\n"; // กำลังใช้งาน
echo $status->name . "\n";    // Active

$orderStatus = OrderStatus::Pending;
echo $orderStatus->value . "\n"; // pending

// From value
$fromDb = OrderStatus::from('shipped'); // throw ถ้าไม่เจอ
$fromDb = OrderStatus::tryFrom('invalid'); // return null ถ้าไม่เจอ

// Transition
if ($orderStatus->canTransitionTo(OrderStatus::Processing)) {
    $orderStatus = OrderStatus::Processing;
    echo "สถานะเปลี่ยนเป็น: " . $orderStatus->label() . "\n";
}

// All cases
foreach (OrderStatus::cases() as $case) {
    echo "{$case->name}: {$case->value}\n";
}

// Enum ใน match
$priority = Priority::High;
$message = match($priority) {
    Priority::Low, Priority::Medium => 'ไม่เร่งด่วน',
    Priority::High    => 'ต้องทำวันนี้',
    Priority::Critical => 'ต้องทำทันที!',
};
echo $message . "\n";
```

### Readonly Properties (PHP 8.1)

```php
<?php
class User {
    public readonly string $name;
    public readonly string $email;
    public readonly string $createdAt;
    
    public function __construct(string $name, string $email) {
        $this->name = $name;         // set ได้ครั้งเดียว
        $this->email = $email;
        $this->createdAt = date('Y-m-d H:i:s');
    }
    
    // แก้ไขจะ throw Error
    public function tryToChange(): void {
        // $this->name = "new name"; // Error! Cannot modify readonly property
    }
}

// Constructor Promotion กับ readonly (PHP 8.0 + 8.1)
class Product {
    public function __construct(
        public readonly int $id,
        public readonly string $name,
        public readonly float $price,
        public readonly string $sku,
        public readonly \DateTimeImmutable $createdAt = new \DateTimeImmutable()
    ) {}
    
    // clone กับ modified values
    public function withPrice(float $newPrice): static {
        // clone ไม่ได้เปลี่ยน readonly
        // ต้องสร้าง object ใหม่
        return new static(
            id: $this->id,
            name: $this->name,
            price: $newPrice, // เปลี่ยน price
            sku: $this->sku,
            createdAt: $this->createdAt
        );
    }
}

$product = new Product(1, "PHP Book", 450.0, "PHP-001");
$expensive = $product->withPrice(500.0);

echo "Original: {$product->price}\n"; // 450
echo "Updated: {$expensive->price}\n";  // 500
```

### Intersection Types (PHP 8.1)

```php
<?php
interface Countable {
    public function count(): int;
}

interface Stringable {
    public function __toString(): string;
}

interface Serializable {
    public function serialize(): string;
}

// Intersection type: ต้อง implement ทุก interface
function process(Countable&Stringable $obj): string {
    return "Count: {$obj->count()}, String: {$obj}";
}

class Collection implements Countable, Stringable {
    private array $items;
    
    public function __construct(array $items = []) {
        $this->items = $items;
    }
    
    public function count(): int {
        return count($this->items);
    }
    
    public function __toString(): string {
        return implode(', ', $this->items);
    }
}

$col = new Collection([1, 2, 3]);
echo process($col) . "\n"; // Count: 3, String: 1, 2, 3

// Intersection กับ Nullable ไม่ได้
// function fn(?A&B $x) {} // Error!
// ใช้ Union แทน
function process2(Countable&Stringable|null $obj): ?string {
    return $obj ? process($obj) : null;
}
```

### Fibers (PHP 8.1)

```php
<?php
// Fiber = lightweight cooperative multitasking
// คล้าย coroutines ใน Python หรือ goroutines ใน Go

$fiber = new Fiber(function(): void {
    $value = Fiber::suspend('fiber started'); // pause, ส่งค่าออก
    echo "Fiber resumed with: {$value}\n";
    
    Fiber::suspend('fiber paused again'); // pause อีกครั้ง
    echo "Fiber finishing...\n";
});

// เริ่ม fiber
$result = $fiber->start(); // return 'fiber started'
echo "Main: got '{$result}'\n";

// Resume fiber
$result = $fiber->resume('hello from main'); // return 'fiber paused again'
echo "Main: got '{$result}'\n";

// Resume อีกครั้ง
$fiber->resume();
echo "Main: fiber finished\n";

// Fiber สำหรับ event loop (simplified)
class EventLoop {
    private array $pending = [];
    
    public function add(callable $task): void {
        $this->pending[] = new Fiber($task);
    }
    
    public function run(): void {
        while (!empty($this->pending)) {
            $tasks = $this->pending;
            $this->pending = [];
            
            foreach ($tasks as $fiber) {
                if (!$fiber->isStarted()) {
                    $fiber->start();
                } elseif ($fiber->isSuspended()) {
                    $fiber->resume();
                }
                
                if ($fiber->isSuspended()) {
                    $this->pending[] = $fiber; // ยังไม่เสร็จ
                }
            }
        }
    }
}

$loop = new EventLoop();

$loop->add(function() {
    echo "Task 1: step 1\n";
    Fiber::suspend();
    echo "Task 1: step 2\n";
    Fiber::suspend();
    echo "Task 1: step 3 (done)\n";
});

$loop->add(function() {
    echo "Task 2: step A\n";
    Fiber::suspend();
    echo "Task 2: step B (done)\n";
});

$loop->run();
// Task 1: step 1
// Task 2: step A
// Task 1: step 2
// Task 2: step B (done)
// Task 1: step 3 (done)
```

---

## 3. PHP 8.2 Features

### Readonly Classes

```php
<?php
// PHP 8.2: readonly class = ทุก property เป็น readonly อัตโนมัติ
readonly class Point {
    public function __construct(
        public float $x,
        public float $y,
        public float $z = 0.0
    ) {}
    
    public function distanceTo(Point $other): float {
        return sqrt(
            ($this->x - $other->x) ** 2 +
            ($this->y - $other->y) ** 2 +
            ($this->z - $other->z) ** 2
        );
    }
    
    public function translate(float $dx, float $dy, float $dz = 0): static {
        return new static($this->x + $dx, $this->y + $dy, $this->z + $dz);
    }
    
    public function __toString(): string {
        return "({$this->x}, {$this->y}, {$this->z})";
    }
}

$p1 = new Point(0, 0);
$p2 = new Point(3, 4);

echo "Distance: " . $p1->distanceTo($p2) . "\n"; // 5
echo "Translated: " . $p1->translate(1, 2) . "\n"; // (1, 2, 0)

// readonly class ไม่สามารถ extend ได้โดย non-readonly class
// readonly class ChildPoint extends Point {} // OK
// class MutablePoint extends Point {} // Error!

// DTO pattern ด้วย readonly class
readonly class CreateUserDto {
    public function __construct(
        public string $username,
        public string $email,
        public string $password,
        public string $role = 'user'
    ) {}
    
    public static function fromRequest(array $data): self {
        return new self(
            username: trim($data['username'] ?? ''),
            email: strtolower(trim($data['email'] ?? '')),
            password: $data['password'] ?? '',
            role: $data['role'] ?? 'user'
        );
    }
}

readonly class UserResponseDto {
    public function __construct(
        public int $id,
        public string $username,
        public string $email,
        public string $role,
        public string $createdAt
    ) {}
    
    public static function fromModel(array $user): self {
        return new self(
            id: $user['id'],
            username: $user['username'],
            email: $user['email'],
            role: $user['role'] ?? 'user',
            createdAt: $user['created_at'] ?? ''
        );
    }
    
    public function toArray(): array {
        return get_object_vars($this);
    }
    
    public function toJson(): string {
        return json_encode($this->toArray(), JSON_UNESCAPED_UNICODE);
    }
}
```

### DNF Types (Disjunctive Normal Form)

```php
<?php
// PHP 8.2: DNF Types = Intersection types ใน Union types
interface Stringable {
    public function __toString(): string;
}

interface Countable {
    public function count(): int;
}

interface Nullable {
    public function isNull(): bool;
}

// DNF: (A&B) | null - ต้อง implement ทั้ง A และ B หรือเป็น null
function process((Stringable&Countable)|null $obj): string {
    if ($obj === null) return 'null';
    return "count={$obj->count()}, str={$obj}";
}

// (A&B) | (C&D) | null
function complexType((Stringable&Countable)|(Stringable&Nullable)|null $obj): string {
    if ($obj === null) return 'null';
    if ($obj instanceof Countable) return "countable: {$obj->count()}";
    return "stringable: {$obj}";
}

// Constants in Traits (PHP 8.2)
trait HasVersion {
    public const VERSION = '1.0.0'; // PHP 8.2+
    
    public static function getVersion(): string {
        return static::VERSION; // LSB
    }
}

class MyLibrary {
    use HasVersion;
    
    public const VERSION = '2.0.0'; // override trait constant
}

echo MyLibrary::getVersion() . "\n"; // 2.0.0
```

---

## 4. PHP 8.3 Features

### Typed Class Constants

```php
<?php
// PHP 8.3: Type declarations สำหรับ class constants
class Config {
    public const string APP_NAME = 'My Application';
    public const int MAX_RETRIES = 3;
    public const float TAX_RATE = 0.07;
    public const bool DEBUG = false;
    public const array SUPPORTED_LOCALES = ['th', 'en', 'ja'];
    
    // Interface/class type
    // public const Status DEFAULT_STATUS = Status::Active; // PHP 8.3+
}

// ใน Interfaces
interface Colorable {
    public const string RED = '#FF0000';
    public const string GREEN = '#00FF00';
    public const string BLUE = '#0000FF';
    
    public function getColor(): string;
}

// Override attribute (PHP 8.3)
class ParentClass {
    public function doSomething(): string {
        return 'parent';
    }
    
    public function onlyInParent(): string {
        return 'parent only';
    }
}

class ChildClass extends ParentClass {
    #[\Override]
    public function doSomething(): string { // บอก PHP ว่า method นี้ override parent
        return 'child';
    }
    
    // #[\Override]
    // public function doTypo(): string {} // Error! ไม่มี method นี้ใน parent
}

// json_validate() (PHP 8.3)
$validJson = '{"name": "PHP", "version": 8.3}';
$invalidJson = '{"name": "PHP", version: 8.3}'; // missing quotes

echo json_validate($validJson) ? "Valid JSON\n" : "Invalid JSON\n";   // Valid JSON
echo json_validate($invalidJson) ? "Valid JSON\n" : "Invalid JSON\n"; // Invalid JSON

// เร็วกว่า json_decode ถ้าแค่ต้องการตรวจสอบ
function isValidJson(string $str): bool {
    // ก่อน PHP 8.3
    json_decode($str);
    return json_last_error() === JSON_ERROR_NONE; // ต้อง decode ทั้งหมด!
    
    // PHP 8.3+
    // return json_validate($str); // เร็วกว่า!
}

// Readonly property clone (PHP 8.3)
readonly class Address {
    public function __construct(
        public string $street,
        public string $city,
        public string $country
    ) {}
}

$addr = new Address('123 Main St', 'Bangkok', 'Thailand');

// PHP 8.3: clone with modified readonly properties
$newAddr = clone $addr;
// $newAddr->city = 'Chiang Mai'; // Error ใน PHP 8.2

// PHP 8.3 clone syntax (ถ้า supported)
// $newAddr = clone($addr) {city: 'Chiang Mai'}; // Experimental
```

### gettype() improvements และ new functions

```php
<?php
// PHP 8.3: str_pad() ใช้ multibyte string ได้
$str = "สวัสดี"; // 6 chars Thai
$padded = mb_str_pad($str, 10, " "); // PHP 8.3+
echo strlen($padded) . "\n"; // pad ถูกต้อง

// Array functions improvements
$arr = [3, 1, 4, 1, 5, 9, 2, 6];

// array_find (PHP 8.4 แต่มักพูดถึงในบริบท 8.3+)
// $first = array_find($arr, fn($x) => $x > 4); // 5

// Static properties in anonymous classes
$obj = new class {
    public static int $count = 0;
    
    public function __construct() {
        static::$count++;
    }
    
    public static function getCount(): int {
        return static::$count;
    }
};
```

---

## 5. Workshop: Modernize Legacy PHP Code

### โจทย์: แปลง Legacy Code เป็น Modern PHP

```php
<?php
// ============================================================
// Legacy Code (PHP 5.x style)
// ============================================================

class LegacyUserManager {
    private $db;
    private $cache = array();
    
    public function __construct($dsn, $username, $password) {
        $this->db = new PDO($dsn, $username, $password);
    }
    
    public function getUser($id) {
        if (isset($this->cache[$id])) {
            return $this->cache[$id];
        }
        
        $stmt = $this->db->prepare("SELECT * FROM users WHERE id = ?");
        $stmt->execute(array($id));
        $user = $stmt->fetch(PDO::FETCH_ASSOC);
        
        if ($user === false) {
            return null;
        }
        
        $this->cache[$id] = $user;
        return $user;
    }
    
    public function updateStatus($id, $status) {
        $validStatuses = array('active', 'inactive', 'pending');
        if (!in_array($status, $validStatuses)) {
            throw new Exception("Invalid status: " . $status);
        }
        
        $stmt = $this->db->prepare("UPDATE users SET status = ? WHERE id = ?");
        return $stmt->execute(array($status, $id));
    }
    
    public function formatUser($user) {
        if ($user == null) {
            return "ไม่พบผู้ใช้";
        }
        
        $name = $user['first_name'] . ' ' . $user['last_name'];
        $status = '';
        switch ($user['status']) {
            case 'active':
                $status = 'กำลังใช้งาน';
                break;
            case 'inactive':
                $status = 'ไม่ได้ใช้งาน';
                break;
            case 'pending':
                $status = 'รอดำเนินการ';
                break;
            default:
                $status = 'ไม่ทราบ';
        }
        
        return $name . ' (' . $status . ')';
    }
    
    public function getUsersByRole($role) {
        $stmt = $this->db->prepare("SELECT * FROM users WHERE role = ?");
        $stmt->execute(array($role));
        $users = array();
        while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
            $users[] = $row;
        }
        return $users;
    }
}

// ============================================================
// Modern Code (PHP 8.3 style)
// ============================================================

// 1. Enum แทน string constants
enum UserStatus: string {
    case Active   = 'active';
    case Inactive = 'inactive';
    case Pending  = 'pending';
    
    public function label(): string {
        return match($this) {
            self::Active   => 'กำลังใช้งาน',
            self::Inactive => 'ไม่ได้ใช้งาน',
            self::Pending  => 'รอดำเนินการ',
        };
    }
}

// 2. Readonly class สำหรับ DTO
readonly class UserDto {
    public function __construct(
        public int $id,
        public string $firstName,
        public string $lastName,
        public string $email,
        public UserStatus $status,
        public string $role
    ) {}
    
    public function getFullName(): string {
        return "{$this->firstName} {$this->lastName}";
    }
    
    public function format(): string {
        return "{$this->getFullName()} ({$this->status->label()})";
    }
    
    public static function fromArray(array $data): self {
        return new self(
            id: (int) $data['id'],
            firstName: $data['first_name'],
            lastName: $data['last_name'],
            email: $data['email'],
            status: UserStatus::from($data['status']),
            role: $data['role'] ?? 'user'
        );
    }
    
    public function toArray(): array {
        return [
            'id' => $this->id,
            'first_name' => $this->firstName,
            'last_name' => $this->lastName,
            'email' => $this->email,
            'status' => $this->status->value,
            'role' => $this->role,
        ];
    }
}

// 3. Modern UserManager
class ModernUserManager {
    private array $cache = [];
    
    public function __construct(
        private readonly PDO $db
    ) {}
    
    public function getUser(int $id): ?UserDto {
        if (isset($this->cache[$id])) {
            return $this->cache[$id];
        }
        
        $stmt = $this->db->prepare(
            'SELECT id, first_name, last_name, email, status, role 
             FROM users WHERE id = :id'
        );
        $stmt->execute([':id' => $id]);
        $row = $stmt->fetch();
        
        if (!$row) return null;
        
        return $this->cache[$id] = UserDto::fromArray($row);
    }
    
    public function updateStatus(int $id, UserStatus $status): bool {
        // Enum ทำให้ไม่ต้อง validate string แล้ว
        $stmt = $this->db->prepare(
            'UPDATE users SET status = :status WHERE id = :id'
        );
        return $stmt->execute([':status' => $status->value, ':id' => $id]);
    }
    
    public function getUsersByRole(string $role): array {
        $stmt = $this->db->prepare(
            'SELECT id, first_name, last_name, email, status, role 
             FROM users WHERE role = :role'
        );
        $stmt->execute([':role' => $role]);
        
        return array_map(
            fn(array $row) => UserDto::fromArray($row),
            $stmt->fetchAll()
        );
    }
    
    public function formatUser(?UserDto $user): string {
        return $user?->format() ?? 'ไม่พบผู้ใช้';
    }
}

// ============================================================
// Migration Checklist
// ============================================================

class MigrationHelper {
    
    public static function demonstrateModernFeatures(): void {
        echo "=== Modern PHP Features Demo ===\n\n";
        
        // 1. Named Arguments
        echo "1. Named Arguments:\n";
        $result = implode(separator: ', ', array: ['PHP', 'Laravel', 'MySQL']);
        echo "   {$result}\n\n";
        
        // 2. Match
        echo "2. Match Expression:\n";
        $status = UserStatus::Active;
        $icon = match($status) {
            UserStatus::Active   => '🟢',
            UserStatus::Inactive => '🔴',
            UserStatus::Pending  => '🟡',
        };
        echo "   Status icon: {$icon}\n\n";
        
        // 3. Nullsafe
        echo "3. Nullsafe Operator:\n";
        $user = null;
        $name = $user?->getFullName() ?? 'Anonymous';
        echo "   Name: {$name}\n\n";
        
        // 4. Enum
        echo "4. Enums:\n";
        foreach (UserStatus::cases() as $case) {
            echo "   {$case->value}: {$case->label()}\n";
        }
        echo "\n";
        
        // 5. Readonly
        echo "5. Readonly Class:\n";
        $dto = new UserDto(1, 'สมชาย', 'ใจดี', 'somchai@example.com', UserStatus::Active, 'user');
        echo "   {$dto->format()}\n\n";
        
        // 6. Fibers
        echo "6. Fibers:\n";
        $fiber = new Fiber(function() {
            echo "   Fiber: step 1\n";
            Fiber::suspend();
            echo "   Fiber: step 2\n";
        });
        $fiber->start();
        echo "   Main: between fiber steps\n";
        $fiber->resume();
        echo "\n";
        
        // 7. Typed Constants (8.3)
        echo "7. Typed Constants (8.3):\n";
        echo "   App: " . Config::APP_NAME . "\n";
        echo "   Max retries: " . Config::MAX_RETRIES . "\n\n";
        
        // 8. json_validate (8.3)
        echo "8. json_validate (8.3):\n";
        $valid = '{"name":"PHP","version":8.3}';
        $invalid = '{name: PHP}';
        echo "   Valid: " . (json_validate($valid) ? 'yes' : 'no') . "\n";
        echo "   Invalid: " . (json_validate($invalid) ? 'yes' : 'no') . "\n";
    }
}

// ============================================================
// Type Safety Improvements ตลอด PHP 8.x
// ============================================================

// Strict mode
declare(strict_types=1);

// Union types
function parseId(int|string $id): int {
    return is_string($id) ? (int) ltrim($id, '#') : $id;
}

// First-class callables (PHP 8.1)
$strlen = strlen(...);
$result = array_map($strlen, ['hello', 'world', 'php']); // [5, 5, 3]

$arr = [3, 1, 4, 1, 5];
$sorted = array_filter($arr, is_int(...)); // first-class callable

// Array unpacking กับ string keys (PHP 8.1)
$arr1 = ['a' => 1, 'b' => 2];
$arr2 = ['b' => 3, 'c' => 4];
$merged = [...$arr1, ...$arr2]; // ['a' => 1, 'b' => 3, 'c' => 4]

// Never type
function throwError(string $msg): never {
    throw new \RuntimeException($msg);
}

// Intersection types (PHP 8.1)
function serialize(Countable&Stringable $obj): string {
    return "count:{$obj->count()},str:{$obj}";
}

// Disjunctive Normal Form types (PHP 8.2)
function handle((Countable&Stringable)|null $obj): string {
    return $obj?->__toString() ?? 'null';
}

// Run demo
MigrationHelper::demonstrateModernFeatures();
```

---

## Quiz

### คำถาม 1
Match expression ต่างจาก Switch อย่างไร? (เลือกทั้งหมดที่ถูก)
- A. Match ใช้ strict comparison (===)
- B. Match return value ได้
- C. Match ไม่มี fallthrough
- D. Match ช้ากว่า Switch

**เฉลย: A, B, C** - Match: strict (===), return value, no fallthrough, throw UnhandledMatchError ถ้าไม่มี default

### คำถาม 2
Enum ใน PHP 8.1 ต่างจาก class constants อย่างไร?
- A. Enum ทำ type-safe ได้ (enum type สำหรับ parameter)
- B. Enum มี methods ได้
- C. Enum มี list ของ cases ได้ (`cases()`)
- D. ถูกทั้ง A, B, C

**เฉลย: D** - Enums ให้ประโยชน์ครบทั้ง type-safety, methods, และ enumerable cases

### คำถาม 3
readonly class (PHP 8.2) ต่างจาก readonly properties (PHP 8.1) อย่างไร?

**เฉลย:** 
- `readonly` property: ต้องใส่ `readonly` ทีละ property
- `readonly class`: ทุก property ใน class เป็น readonly อัตโนมัติ ไม่ต้องใส่ทีละตัว

### คำถาม 4
Fiber ใน PHP 8.1 คืออะไร และต่างจาก Thread อย่างไร?

**เฉลย:** Fiber คือ cooperative coroutine ที่สามารถ pause/resume ได้ แต่ยังทำงานบน single thread เหมือนเดิม (ไม่ใช่ parallel execution) ต่างจาก Thread ที่ทำงานบน multiple cores พร้อมกัน Fiber เหมาะสำหรับ async I/O (เช่น event loop) ที่ต้องการ interleaving ไม่ใช่ parallelism

---

## สรุป Part 025

ใน Part นี้เราได้เรียนรู้ PHP 8.x modern features ที่สำคัญ:

### PHP 8.0
- **Named Arguments** - เรียก function โดยระบุชื่อ parameter
- **Match Expression** - เปลี่ยน switch เป็น expression
- **Nullsafe Operator** - `?->` หลีกเลี่ยง null checks
- **Attributes** - metadata annotations ที่เป็นทางการ
- **Union Types** - `int|string`

### PHP 8.1
- **Enums** - type-safe enumeration
- **Readonly Properties** - immutable object properties
- **Intersection Types** - `A&B`
- **Fibers** - cooperative coroutines
- **First-class Callables** - `strlen(...)`

### PHP 8.2
- **Readonly Classes** - ทุก property เป็น readonly
- **DNF Types** - `(A&B)|null`
- **Constants in Traits**

### PHP 8.3
- **Typed Class Constants** - `public const string NAME = 'value'`
- **#[Override] Attribute**
- **json_validate()**

---

## สรุปหลักสูตร PHP OOP ขั้นสูง

คุณได้เรียนรู้ครบทั้ง 10 Part:

| Part | หัวข้อ |
|------|--------|
| 016 | OOP ขั้นสูง (Abstract, Interface, Traits, LSB) |
| 017 | Namespaces & Autoloading (PSR-4, Composer) |
| 018 | Composer ขั้นสูง (SemVer, Packages, Satis) |
| 019 | Error Handling & Logging (Monolog) |
| 020 | Regular Expressions (PCRE, Regex) |
| 021 | JSON & REST API (cURL, Guzzle) |
| 022 | PHP Security (XSS, SQLi, CSRF, Password) |
| 023 | Testing (PHPUnit, Mockery, Coverage) |
| 024 | Performance (OPcache, Big O, Memory) |
| 025 | PHP Modern (8.0-8.3 features) |

ขั้นตอนต่อไปคือเรียน Laravel Framework!

---

## ➡️ Part ถัดไป

[Part 026: เริ่มต้นกับ Laravel](./part-026-laravel-intro.md)
