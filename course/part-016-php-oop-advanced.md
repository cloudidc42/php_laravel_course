# Part 016: PHP OOP ขั้นสูง (Advanced OOP)

## ระดับ: Intermediate to Advanced
## เวลาเรียน: 4-5 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:
- เข้าใจและใช้งาน Abstract Classes ได้อย่างถูกต้อง
- สร้างและ implement Interfaces ได้
- ใช้ Traits เพื่อ reuse code
- เข้าใจ Late Static Binding
- สร้าง Anonymous Classes
- ออกแบบ Plugin System ด้วย Interfaces

---

## 1. Abstract Classes (คลาสนามธรรม)

Abstract Class คือคลาสที่ไม่สามารถสร้าง object ได้โดยตรง มีไว้เพื่อเป็น "แม่แบบ" ให้คลาสลูกสืบทอด

### ทำไมต้องใช้ Abstract Class?

```php
<?php
// ปัญหา: ถ้าเราสร้าง Shape โดยตรง มันไม่มีความหมาย
class Shape {
    public function area(): float {
        // จะคำนวณยังไง? ไม่รู้ว่าเป็นรูปทรงอะไร
        return 0;
    }
}

$shape = new Shape(); // ไม่มีความหมาย!
```

```php
<?php
// วิธีแก้: ใช้ Abstract Class
abstract class Shape {
    // Abstract method = ต้องให้คลาสลูก implement
    abstract public function area(): float;
    abstract public function perimeter(): float;
    
    // Concrete method = ใช้ได้ทันทีโดยไม่ต้อง override
    public function describe(): string {
        return sprintf(
            "รูปทรง %s มีพื้นที่ %.2f และรอบรูป %.2f",
            get_class($this),
            $this->area(),
            $this->perimeter()
        );
    }
    
    public function isLargerThan(Shape $other): bool {
        return $this->area() > $other->area();
    }
}

class Circle extends Shape {
    public function __construct(
        private float $radius
    ) {}
    
    public function area(): float {
        return M_PI * $this->radius ** 2;
    }
    
    public function perimeter(): float {
        return 2 * M_PI * $this->radius;
    }
}

class Rectangle extends Shape {
    public function __construct(
        private float $width,
        private float $height
    ) {}
    
    public function area(): float {
        return $this->width * $this->height;
    }
    
    public function perimeter(): float {
        return 2 * ($this->width + $this->height);
    }
}

class Triangle extends Shape {
    public function __construct(
        private float $a,
        private float $b,
        private float $c
    ) {}
    
    public function area(): float {
        // Heron's formula
        $s = ($this->a + $this->b + $this->c) / 2;
        return sqrt($s * ($s - $this->a) * ($s - $this->b) * ($s - $this->c));
    }
    
    public function perimeter(): float {
        return $this->a + $this->b + $this->c;
    }
}

// ทดสอบ
$circle = new Circle(5);
$rect = new Rectangle(4, 6);
$triangle = new Triangle(3, 4, 5);

echo $circle->describe() . "\n";
echo $rect->describe() . "\n";
echo $triangle->describe() . "\n";

echo "\nวงกลมใหญ่กว่าสี่เหลี่ยมไหม? ";
echo $circle->isLargerThan($rect) ? "ใช่" : "ไม่ใช่";

// $shape = new Shape(); // Error! Cannot instantiate abstract class
```

### Abstract Class กับ Constructor

```php
<?php
abstract class Animal {
    private static int $count = 0;
    
    public function __construct(
        protected string $name,
        protected int $age
    ) {
        self::$count++;
        echo "สร้างสัตว์ชื่อ {$this->name}\n";
    }
    
    abstract public function makeSound(): string;
    abstract public function move(): string;
    
    public function getInfo(): string {
        return "{$this->name} อายุ {$this->age} ปี";
    }
    
    public static function getCount(): int {
        return self::$count;
    }
}

class Dog extends Animal {
    public function makeSound(): string {
        return "โฮ่ง!";
    }
    
    public function move(): string {
        return "{$this->name} วิ่ง";
    }
    
    public function fetch(): string {
        return "{$this->name} ไปเอาลูกบอลมา";
    }
}

class Bird extends Animal {
    public function __construct(
        string $name,
        int $age,
        private float $wingspan
    ) {
        parent::__construct($name, $age);
    }
    
    public function makeSound(): string {
        return "จ๊อก จ๊อก!";
    }
    
    public function move(): string {
        return "{$this->name} บิน";
    }
}

$dog = new Dog("บุญมา", 3);
$bird = new Bird("น้ำหวาน", 2, 0.5);

echo $dog->getInfo() . " - " . $dog->makeSound() . "\n";
echo $bird->getInfo() . " - " . $bird->makeSound() . "\n";
echo "จำนวนสัตว์ที่สร้าง: " . Animal::getCount() . "\n";
```

---

## 2. Interfaces

Interface คือ "สัญญา" ที่กำหนดว่าคลาสที่ implement ต้องมี method อะไรบ้าง

### ความแตกต่างระหว่าง Abstract Class และ Interface

| คุณสมบัติ | Abstract Class | Interface |
|-----------|---------------|-----------|
| สืบทอดได้ | 1 คลาส | หลาย interface |
| มี properties | ได้ | ไม่ได้ (มีแค่ constants) |
| Constructor | มีได้ | ไม่มี |
| Method implementation | บางส่วนได้ | PHP 8+ มี default ได้ |
| Access modifier | ทุกอย่าง | public เท่านั้น |

```php
<?php
// Interface พื้นฐาน
interface Printable {
    public function print(): void;
    public function getPrintPreview(): string;
}

interface Exportable {
    public function exportToPdf(): string;
    public function exportToCsv(): string;
}

interface Saveable {
    public function save(): bool;
    public function load(int $id): static;
    public function delete(): bool;
}

// Interface สามารถ extend interface อื่นได้
interface DocumentInterface extends Printable, Exportable, Saveable {
    public function getTitle(): string;
    public function setTitle(string $title): void;
}

class Invoice implements DocumentInterface {
    private string $title = '';
    
    public function __construct(
        private int $invoiceNumber,
        private array $items = []
    ) {
        $this->title = "Invoice #{$invoiceNumber}";
    }
    
    public function getTitle(): string {
        return $this->title;
    }
    
    public function setTitle(string $title): void {
        $this->title = $title;
    }
    
    public function print(): void {
        echo $this->getPrintPreview();
    }
    
    public function getPrintPreview(): string {
        $output = "=== {$this->title} ===\n";
        foreach ($this->items as $item) {
            $output .= "- {$item['name']}: {$item['price']} บาท\n";
        }
        return $output;
    }
    
    public function exportToPdf(): string {
        return "PDF exported for {$this->title}";
    }
    
    public function exportToCsv(): string {
        $csv = "name,price\n";
        foreach ($this->items as $item) {
            $csv .= "{$item['name']},{$item['price']}\n";
        }
        return $csv;
    }
    
    public function save(): bool {
        echo "Saving {$this->title} to database\n";
        return true;
    }
    
    public function load(int $id): static {
        // Load from database
        return new static($id);
    }
    
    public function delete(): bool {
        echo "Deleting {$this->title}\n";
        return true;
    }
    
    public function addItem(string $name, float $price): void {
        $this->items[] = ['name' => $name, 'price' => $price];
    }
    
    public function getTotal(): float {
        return array_sum(array_column($this->items, 'price'));
    }
}

// ใช้งาน
$invoice = new Invoice(1001);
$invoice->addItem("PHP Book", 450);
$invoice->addItem("Laravel Course", 1200);
$invoice->addItem("VSCode License", 0);

$invoice->print();
echo "ยอดรวม: " . $invoice->getTotal() . " บาท\n";
echo $invoice->exportToCsv();
```

### Interface Constants

```php
<?php
interface Status {
    const ACTIVE = 'active';
    const INACTIVE = 'inactive';
    const PENDING = 'pending';
    const DELETED = 'deleted';
    
    public function getStatus(): string;
    public function setStatus(string $status): void;
}

interface Priority {
    const LOW = 1;
    const MEDIUM = 2;
    const HIGH = 3;
    const CRITICAL = 4;
    
    public function getPriority(): int;
}

class Task implements Status, Priority {
    private string $status = Status::PENDING;
    private int $priority = Priority::MEDIUM;
    
    public function __construct(
        private string $name
    ) {}
    
    public function getStatus(): string {
        return $this->status;
    }
    
    public function setStatus(string $status): void {
        $this->status = $status;
    }
    
    public function getPriority(): int {
        return $this->priority;
    }
    
    public function setPriority(int $priority): void {
        $this->priority = $priority;
    }
    
    public function __toString(): string {
        $priorities = [
            Priority::LOW => 'Low',
            Priority::MEDIUM => 'Medium',
            Priority::HIGH => 'High',
            Priority::CRITICAL => 'Critical',
        ];
        
        return "[{$priorities[$this->priority]}] {$this->name} ({$this->status})";
    }
}

$task = new Task("แก้ bug หน้า login");
$task->setStatus(Status::ACTIVE);
$task->setPriority(Priority::HIGH);

echo $task . "\n";
echo "Status: " . $task->getStatus() . "\n";
```

---

## 3. Traits

Trait คือกลไกสำหรับ reuse code ในภาษาที่ไม่รองรับ multiple inheritance

### Trait พื้นฐาน

```php
<?php
trait Timestampable {
    private ?DateTime $createdAt = null;
    private ?DateTime $updatedAt = null;
    
    public function setCreatedAt(): void {
        $this->createdAt = new DateTime();
    }
    
    public function setUpdatedAt(): void {
        $this->updatedAt = new DateTime();
    }
    
    public function getCreatedAt(): ?DateTime {
        return $this->createdAt;
    }
    
    public function getUpdatedAt(): ?DateTime {
        return $this->updatedAt;
    }
    
    public function getTimestampInfo(): string {
        $created = $this->createdAt?->format('Y-m-d H:i:s') ?? 'ยังไม่ได้บันทึก';
        $updated = $this->updatedAt?->format('Y-m-d H:i:s') ?? 'ยังไม่ได้อัปเดต';
        return "Created: {$created}, Updated: {$updated}";
    }
}

trait SoftDeletable {
    private ?DateTime $deletedAt = null;
    
    public function softDelete(): void {
        $this->deletedAt = new DateTime();
    }
    
    public function restore(): void {
        $this->deletedAt = null;
    }
    
    public function isDeleted(): bool {
        return $this->deletedAt !== null;
    }
    
    public function getDeletedAt(): ?DateTime {
        return $this->deletedAt;
    }
}

trait Loggable {
    private array $logs = [];
    
    public function log(string $message, string $level = 'info'): void {
        $this->logs[] = [
            'level' => $level,
            'message' => $message,
            'time' => date('Y-m-d H:i:s'),
        ];
    }
    
    public function getLogs(): array {
        return $this->logs;
    }
    
    public function clearLogs(): void {
        $this->logs = [];
    }
}

class User {
    use Timestampable, SoftDeletable, Loggable;
    
    public function __construct(
        private string $name,
        private string $email
    ) {
        $this->setCreatedAt();
        $this->log("User created: {$name}");
    }
    
    public function update(string $name): void {
        $this->name = $name;
        $this->setUpdatedAt();
        $this->log("User updated: {$name}", 'update');
    }
    
    public function getName(): string {
        return $this->name;
    }
}

$user = new User("สมชาย", "somchai@example.com");
sleep(1); // รอสักครู่
$user->update("สมชาย ใจดี");
$user->softDelete();
$user->log("User deleted", 'warning');

echo "ชื่อ: " . $user->getName() . "\n";
echo $user->getTimestampInfo() . "\n";
echo "ถูกลบแล้ว: " . ($user->isDeleted() ? "ใช่" : "ไม่ใช่") . "\n";
echo "\n--- Logs ---\n";
foreach ($user->getLogs() as $log) {
    echo "[{$log['level']}] {$log['message']}\n";
}
```

### Multiple Traits และการแก้ไข Conflict

```php
<?php
trait Hello {
    public function greet(): string {
        return "สวัสดี!";
    }
    
    public function sayName(): string {
        return "ฉันคือ Hello Trait";
    }
}

trait Hi {
    public function greet(): string {
        return "หวัดดี!";
    }
    
    public function sayName(): string {
        return "ฉันคือ Hi Trait";
    }
}

class Greeter {
    use Hello, Hi {
        // การแก้ไข conflict
        Hello::greet insteadof Hi;    // ใช้ Hello::greet แทน Hi::greet
        Hi::greet as hiGreet;         // สร้าง alias สำหรับ Hi::greet
        
        Hello::sayName insteadof Hi;  // ใช้ Hello::sayName
        Hi::sayName as hiSayName;     // alias สำหรับ Hi::sayName
        
        // เปลี่ยน visibility
        hiGreet as protected;         // เปลี่ยนเป็น protected
    }
}

$greeter = new Greeter();
echo $greeter->greet() . "\n";      // สวัสดี! (จาก Hello)
// echo $greeter->hiGreet() . "\n"; // Error! protected
echo $greeter->sayName() . "\n";    // ฉันคือ Hello Trait
echo $greeter->hiSayName() . "\n";  // ฉันคือ Hi Trait
```

### Trait กับ Abstract Methods

```php
<?php
trait Validatable {
    // Trait สามารถกำหนด abstract method ที่คลาสต้องมี
    abstract protected function getValidationRules(): array;
    
    public function validate(array $data): array {
        $errors = [];
        $rules = $this->getValidationRules();
        
        foreach ($rules as $field => $rule) {
            if ($rule === 'required' && empty($data[$field])) {
                $errors[$field] = "{$field} จำเป็นต้องกรอก";
            }
            
            if (str_starts_with($rule, 'min:')) {
                $min = (int) substr($rule, 4);
                if (strlen($data[$field] ?? '') < $min) {
                    $errors[$field] = "{$field} ต้องมีความยาวอย่างน้อย {$min} ตัวอักษร";
                }
            }
            
            if ($rule === 'email') {
                if (!filter_var($data[$field] ?? '', FILTER_VALIDATE_EMAIL)) {
                    $errors[$field] = "{$field} รูปแบบ email ไม่ถูกต้อง";
                }
            }
        }
        
        return $errors;
    }
    
    public function isValid(array $data): bool {
        return empty($this->validate($data));
    }
}

class RegistrationForm {
    use Validatable;
    
    protected function getValidationRules(): array {
        return [
            'username' => 'required',
            'email' => 'email',
            'password' => 'min:8',
        ];
    }
}

class LoginForm {
    use Validatable;
    
    protected function getValidationRules(): array {
        return [
            'email' => 'email',
            'password' => 'required',
        ];
    }
}

$regForm = new RegistrationForm();
$errors = $regForm->validate([
    'username' => '',
    'email' => 'invalid-email',
    'password' => 'short',
]);

echo "Validation Errors:\n";
foreach ($errors as $field => $error) {
    echo "- {$error}\n";
}
```

### Trait Properties และ Static Methods

```php
<?php
trait Counter {
    private static int $instanceCount = 0;
    private int $id;
    
    public static function getInstanceCount(): int {
        return static::$instanceCount;
    }
    
    protected function initCounter(): void {
        static::$instanceCount++;
        $this->id = static::$instanceCount;
    }
    
    public function getId(): int {
        return $this->id;
    }
}

class Product {
    use Counter;
    
    public function __construct(private string $name) {
        $this->initCounter();
    }
}

class Category {
    use Counter;
    
    public function __construct(private string $name) {
        $this->initCounter();
    }
}

$p1 = new Product("PHP Book");
$p2 = new Product("Laravel Book");
$p3 = new Product("MySQL Guide");

$c1 = new Category("Programming");
$c2 = new Category("Database");

echo "Products: " . Product::getInstanceCount() . "\n"; // 3
echo "Categories: " . Category::getInstanceCount() . "\n"; // 2

echo "Product IDs: {$p1->getId()}, {$p2->getId()}, {$p3->getId()}\n";
```

---

## 4. Late Static Binding (LSB)

LSB ทำให้ `static::` อ้างอิงถึงคลาสที่ถูกเรียกจริงๆ แทนที่จะเป็นคลาสที่ method ถูกนิยาม

```php
<?php
class Base {
    protected static string $type = 'Base';
    
    // ใช้ self:: - อ้างถึงคลาสที่ method ถูกนิยาม (Base)
    public static function createWithSelf(): static {
        $instance = new self(); // จะสร้าง Base เสมอ!
        return $instance;
    }
    
    // ใช้ static:: - อ้างถึงคลาสที่ถูกเรียกจริง (LSB)
    public static function createWithStatic(): static {
        $instance = new static(); // สร้างคลาสที่เรียก method นี้
        return $instance;
    }
    
    public static function getType(): string {
        return static::$type; // LSB - อ้างถึง property ของคลาสที่เรียก
    }
    
    public function getClass(): string {
        return static::class; // LSB
    }
}

class Child extends Base {
    protected static string $type = 'Child';
}

class GrandChild extends Child {
    protected static string $type = 'GrandChild';
}

// ทดสอบ self:: vs static::
$fromSelf = Child::createWithSelf();
$fromStatic = Child::createWithStatic();

echo get_class($fromSelf) . "\n";   // Base (ใช้ self::)
echo get_class($fromStatic) . "\n"; // Child (ใช้ static::)

echo Base::getType() . "\n";        // Base
echo Child::getType() . "\n";       // Child
echo GrandChild::getType() . "\n";  // GrandChild
```

### ประโยชน์จริงของ LSB: ActiveRecord Pattern

```php
<?php
abstract class Model {
    protected static string $table = '';
    protected array $attributes = [];
    protected bool $exists = false;
    
    public function __construct(array $attributes = []) {
        $this->attributes = $attributes;
    }
    
    // Factory method ด้วย LSB
    public static function create(array $data): static {
        $model = new static($data);
        $model->save();
        return $model;
    }
    
    public static function find(int $id): ?static {
        // จำลองการ query database
        $fakeData = static::getFakeData($id);
        if ($fakeData === null) return null;
        
        $model = new static($fakeData);
        $model->exists = true;
        return $model;
    }
    
    // แต่ละ model จะ override method นี้
    protected static function getFakeData(int $id): ?array {
        return null;
    }
    
    public static function getTableName(): string {
        return static::$table;
    }
    
    public function save(): bool {
        if ($this->exists) {
            echo "UPDATE " . static::$table . " SET ...\n";
        } else {
            echo "INSERT INTO " . static::$table . " VALUES ...\n";
            $this->exists = true;
        }
        return true;
    }
    
    public function __get(string $key): mixed {
        return $this->attributes[$key] ?? null;
    }
    
    public function __set(string $key, mixed $value): void {
        $this->attributes[$key] = $value;
    }
}

class UserModel extends Model {
    protected static string $table = 'users';
    
    protected static function getFakeData(int $id): ?array {
        $data = [
            1 => ['id' => 1, 'name' => 'สมชาย', 'email' => 'somchai@example.com'],
            2 => ['id' => 2, 'name' => 'สมหญิง', 'email' => 'somying@example.com'],
        ];
        return $data[$id] ?? null;
    }
}

class PostModel extends Model {
    protected static string $table = 'posts';
    
    protected static function getFakeData(int $id): ?array {
        $data = [
            1 => ['id' => 1, 'title' => 'Hello World', 'user_id' => 1],
        ];
        return $data[$id] ?? null;
    }
}

// ทดสอบ
$user = UserModel::create(['name' => 'วิชัย', 'email' => 'wichai@example.com']);
echo get_class($user) . "\n"; // UserModel

$existingUser = UserModel::find(1);
echo "ชื่อ: {$existingUser->name}\n";
echo "Table: " . UserModel::getTableName() . "\n";

$post = PostModel::find(1);
echo "Post: {$post->title}\n";
echo "Post Table: " . PostModel::getTableName() . "\n";
```

---

## 5. Anonymous Classes

Anonymous Class คือคลาสที่ไม่มีชื่อ สร้างและใช้งานในที่เดียวกัน

```php
<?php
// Anonymous Class พื้นฐาน
$obj = new class("สวัสดี") {
    public function __construct(private string $message) {}
    
    public function getMessage(): string {
        return $this->message;
    }
};

echo $obj->getMessage() . "\n";

// Anonymous Class implement Interface
interface Logger {
    public function log(string $message): void;
}

function doSomething(Logger $logger): void {
    $logger->log("เริ่มทำงาน...");
    // ... ทำงานบางอย่าง
    $logger->log("เสร็จสิ้น!");
}

// ส่ง anonymous class แทนที่จะต้องสร้างคลาสใหม่
doSomething(new class implements Logger {
    public function log(string $message): void {
        echo "[LOG] {$message}\n";
    }
});

// Anonymous Class สืบทอดจาก abstract class
abstract class BaseHandler {
    abstract public function handle(mixed $data): mixed;
    
    public function process(mixed $data): mixed {
        echo "Before processing\n";
        $result = $this->handle($data);
        echo "After processing\n";
        return $result;
    }
}

$handler = new class extends BaseHandler {
    public function handle(mixed $data): mixed {
        return strtoupper($data);
    }
};

$result = $handler->process("hello world");
echo "Result: {$result}\n";
```

### Anonymous Class ใน Test/Mock

```php
<?php
interface DatabaseConnection {
    public function query(string $sql): array;
    public function execute(string $sql, array $params = []): bool;
}

class UserRepository {
    public function __construct(
        private DatabaseConnection $db
    ) {}
    
    public function findById(int $id): ?array {
        $results = $this->db->query("SELECT * FROM users WHERE id = {$id}");
        return $results[0] ?? null;
    }
    
    public function create(array $data): bool {
        return $this->db->execute(
            "INSERT INTO users (name, email) VALUES (?, ?)",
            [$data['name'], $data['email']]
        );
    }
}

// สร้าง Mock ด้วย Anonymous Class
$mockDb = new class implements DatabaseConnection {
    private array $data = [
        ['id' => 1, 'name' => 'สมชาย', 'email' => 'somchai@test.com'],
        ['id' => 2, 'name' => 'สมหญิง', 'email' => 'somying@test.com'],
    ];
    
    public function query(string $sql): array {
        echo "[Mock DB] Query: {$sql}\n";
        // คืนข้อมูล fake
        if (str_contains($sql, 'WHERE id = 1')) {
            return [$this->data[0]];
        }
        return $this->data;
    }
    
    public function execute(string $sql, array $params = []): bool {
        echo "[Mock DB] Execute: {$sql}\n";
        return true;
    }
};

$repo = new UserRepository($mockDb);
$user = $repo->findById(1);
echo "พบผู้ใช้: {$user['name']}\n";
$repo->create(['name' => 'วิชัย', 'email' => 'wichai@test.com']);
```

### Anonymous Class สำหรับ One-time Callbacks

```php
<?php
function processItems(array $items, object $processor): array {
    return array_map(
        fn($item) => $processor->process($item),
        $items
    );
}

$numbers = [1, 2, 3, 4, 5];

// สร้าง processor แบบ one-time
$doubled = processItems($numbers, new class {
    public function process(mixed $item): mixed {
        return $item * 2;
    }
});

print_r($doubled);
```

---

## 6. Workshop: สร้าง Plugin System ด้วย Interfaces

### โจทย์: สร้าง Plugin System สำหรับ CMS

```php
<?php

// ============================================================
// Plugin Interfaces
// ============================================================

interface PluginInterface {
    public function getName(): string;
    public function getVersion(): string;
    public function getDescription(): string;
    public function activate(): void;
    public function deactivate(): void;
    public function isActive(): bool;
}

interface ContentFilterPlugin extends PluginInterface {
    public function filter(string $content): string;
    public function getPriority(): int; // ลำดับความสำคัญ (ต่ำ = ทำงานก่อน)
}

interface StoragePlugin extends PluginInterface {
    public function store(string $key, mixed $data): bool;
    public function retrieve(string $key): mixed;
    public function delete(string $key): bool;
}

interface AuthPlugin extends PluginInterface {
    public function authenticate(array $credentials): bool;
    public function getToken(array $credentials): string;
}

// ============================================================
// Plugin Manager
// ============================================================

class PluginManager {
    private array $plugins = [];
    private array $activePlugins = [];
    
    public function register(PluginInterface $plugin): void {
        $name = $plugin->getName();
        $this->plugins[$name] = $plugin;
        echo "Registered plugin: {$name} v{$plugin->getVersion()}\n";
    }
    
    public function activate(string $name): void {
        if (!isset($this->plugins[$name])) {
            throw new \RuntimeException("Plugin '{$name}' not found");
        }
        
        $plugin = $this->plugins[$name];
        $plugin->activate();
        $this->activePlugins[$name] = $plugin;
        echo "Activated: {$name}\n";
    }
    
    public function deactivate(string $name): void {
        if (isset($this->activePlugins[$name])) {
            $this->activePlugins[$name]->deactivate();
            unset($this->activePlugins[$name]);
            echo "Deactivated: {$name}\n";
        }
    }
    
    public function getContentFilters(): array {
        $filters = array_filter(
            $this->activePlugins,
            fn($p) => $p instanceof ContentFilterPlugin
        );
        
        // เรียงตาม priority
        usort($filters, fn($a, $b) => $a->getPriority() - $b->getPriority());
        
        return $filters;
    }
    
    public function applyContentFilters(string $content): string {
        foreach ($this->getContentFilters() as $filter) {
            $content = $filter->filter($content);
        }
        return $content;
    }
    
    public function getPlugin(string $name): ?PluginInterface {
        return $this->activePlugins[$name] ?? null;
    }
    
    public function listPlugins(): void {
        echo "\n=== Installed Plugins ===\n";
        foreach ($this->plugins as $name => $plugin) {
            $status = $plugin->isActive() ? '✓' : '✗';
            echo "[{$status}] {$name} v{$plugin->getVersion()} - {$plugin->getDescription()}\n";
        }
    }
}

// ============================================================
// Concrete Plugin Implementations
// ============================================================

abstract class BasePlugin implements PluginInterface {
    private bool $active = false;
    
    public function activate(): void {
        $this->active = true;
    }
    
    public function deactivate(): void {
        $this->active = false;
    }
    
    public function isActive(): bool {
        return $this->active;
    }
}

// Plugin 1: Markdown Filter
class MarkdownPlugin extends BasePlugin implements ContentFilterPlugin {
    public function getName(): string { return 'markdown'; }
    public function getVersion(): string { return '1.0.0'; }
    public function getDescription(): string { return 'แปลง Markdown เป็น HTML'; }
    public function getPriority(): int { return 10; }
    
    public function filter(string $content): string {
        // Simple markdown conversion
        $patterns = [
            '/^# (.+)$/m'       => '<h1>$1</h1>',
            '/^## (.+)$/m'      => '<h2>$1</h2>',
            '/^### (.+)$/m'     => '<h3>$1</h3>',
            '/\*\*(.+?)\*\*/'   => '<strong>$1</strong>',
            '/\*(.+?)\*/'       => '<em>$1</em>',
            '/`(.+?)`/'         => '<code>$1</code>',
            '/\n/'              => '<br>',
        ];
        
        return preg_replace(
            array_keys($patterns),
            array_values($patterns),
            $content
        );
    }
}

// Plugin 2: Spam Filter
class SpamFilterPlugin extends BasePlugin implements ContentFilterPlugin {
    private array $bannedWords = ['spam', 'casino', 'cheap medicine'];
    
    public function getName(): string { return 'spam-filter'; }
    public function getVersion(): string { return '2.1.0'; }
    public function getDescription(): string { return 'กรอง spam และคำไม่เหมาะสม'; }
    public function getPriority(): int { return 5; } // ทำงานก่อน markdown
    
    public function filter(string $content): string {
        foreach ($this->bannedWords as $word) {
            $content = str_ireplace($word, str_repeat('*', strlen($word)), $content);
        }
        return $content;
    }
    
    public function addBannedWord(string $word): void {
        $this->bannedWords[] = $word;
    }
}

// Plugin 3: Cache Storage
class FileCachePlugin extends BasePlugin implements StoragePlugin {
    private array $cache = [];
    
    public function getName(): string { return 'file-cache'; }
    public function getVersion(): string { return '1.5.0'; }
    public function getDescription(): string { return 'Cache ข้อมูลด้วย File System'; }
    
    public function store(string $key, mixed $data): bool {
        $this->cache[$key] = [
            'data' => $data,
            'stored_at' => time(),
        ];
        echo "[Cache] Stored: {$key}\n";
        return true;
    }
    
    public function retrieve(string $key): mixed {
        if (!isset($this->cache[$key])) {
            echo "[Cache] Miss: {$key}\n";
            return null;
        }
        echo "[Cache] Hit: {$key}\n";
        return $this->cache[$key]['data'];
    }
    
    public function delete(string $key): bool {
        unset($this->cache[$key]);
        return true;
    }
}

// Plugin 4: JWT Auth
class JwtAuthPlugin extends BasePlugin implements AuthPlugin {
    private string $secret = 'my-secret-key';
    private array $users = [
        'admin' => 'password123',
        'user1' => 'pass456',
    ];
    
    public function getName(): string { return 'jwt-auth'; }
    public function getVersion(): string { return '3.0.0'; }
    public function getDescription(): string { return 'Authentication ด้วย JWT Token'; }
    
    public function authenticate(array $credentials): bool {
        $username = $credentials['username'] ?? '';
        $password = $credentials['password'] ?? '';
        
        return isset($this->users[$username]) 
            && $this->users[$username] === $password;
    }
    
    public function getToken(array $credentials): string {
        if (!$this->authenticate($credentials)) {
            throw new \RuntimeException("Invalid credentials");
        }
        
        // Mock JWT token
        $payload = base64_encode(json_encode([
            'user' => $credentials['username'],
            'exp' => time() + 3600,
        ]));
        
        return "header.{$payload}.signature";
    }
}

// ============================================================
// CMS Application
// ============================================================

class CMS {
    public function __construct(
        private PluginManager $plugins
    ) {}
    
    public function publishPost(string $content): string {
        echo "\n--- Publishing Post ---\n";
        echo "Original:\n{$content}\n\n";
        
        $filtered = $this->plugins->applyContentFilters($content);
        
        echo "Processed:\n{$filtered}\n";
        return $filtered;
    }
    
    public function cacheData(string $key, mixed $data): void {
        $cache = $this->plugins->getPlugin('file-cache');
        if ($cache instanceof StoragePlugin) {
            $cache->store($key, $data);
        }
    }
    
    public function login(string $username, string $password): string {
        $auth = $this->plugins->getPlugin('jwt-auth');
        if (!$auth instanceof AuthPlugin) {
            throw new \RuntimeException("No auth plugin active");
        }
        
        return $auth->getToken([
            'username' => $username,
            'password' => $password,
        ]);
    }
}

// ============================================================
// Main: ทดสอบ Plugin System
// ============================================================

$manager = new PluginManager();

// Register plugins
$manager->register(new MarkdownPlugin());
$manager->register(new SpamFilterPlugin());
$manager->register(new FileCachePlugin());
$manager->register(new JwtAuthPlugin());

// Activate plugins
$manager->activate('markdown');
$manager->activate('spam-filter');
$manager->activate('file-cache');
$manager->activate('jwt-auth');

$manager->listPlugins();

// สร้าง CMS
$cms = new CMS($manager);

// ทดสอบ content filters
$post = "# Welcome to My Blog\n\n**PHP** is *awesome*!\n\nCheck out this `code` example.\n\nAvoid spam content here.";
$cms->publishPost($post);

// ทดสอบ cache
$cms->cacheData('homepage', ['title' => 'Home', 'content' => 'Welcome!']);
$cms->cacheData('homepage', ['title' => 'Home Updated']);

// ทดสอบ auth
try {
    $token = $cms->login('admin', 'password123');
    echo "\nLogin successful! Token: {$token}\n";
} catch (\RuntimeException $e) {
    echo "Login failed: " . $e->getMessage() . "\n";
}

// ปิด plugin
$manager->deactivate('spam-filter');
$manager->listPlugins();
```

---

## Quiz

### คำถาม 1
Abstract Class และ Interface ต่างกันอย่างไร? (เลือกข้อที่ถูกต้องทั้งหมด)
- A. Abstract Class สืบทอดได้หลายคลาส แต่ Interface สืบทอดได้คลาสเดียว
- B. Interface ไม่สามารถมี properties ได้ แต่ Abstract Class มีได้
- C. คลาสหนึ่งสามารถ implement หลาย Interface ได้
- D. Abstract Class สามารถมี method ที่มี implementation ได้

**เฉลย: B, C, D**
- A ผิด (ตรงกันข้าม - PHP ไม่รองรับ multiple inheritance สำหรับ class)

### คำถาม 2
Trait และ Interface ต่างกันอย่างไร?
- A. Trait มี code implementation ได้ Interface ไม่ได้
- B. Interface กำหนด "สัญญา" ส่วน Trait ให้ "พฤติกรรม"
- C. Trait ใช้ `use` ส่วน Interface ใช้ `implements`
- D. ถูกทั้ง A, B, C

**เฉลย: D**

### คำถาม 3
ผลลัพธ์ของโค้ดนี้คืออะไร?
```php
class Base {
    public static function create(): static {
        return new static();
    }
    public function getClass(): string {
        return static::class;
    }
}
class Child extends Base {}

$obj = Child::create();
echo $obj->getClass();
```
- A. `Base`
- B. `Child`
- C. Error
- D. `static`

**เฉลย: B** - `static::` ใช้ Late Static Binding อ้างถึงคลาสที่เรียกจริง (`Child`)

### คำถาม 4
เมื่อใช้ Traits สองตัวที่มี method ชื่อเดียวกัน จะต้องทำอย่างไร?
```php
trait A { public function hello(): string { return "A"; } }
trait B { public function hello(): string { return "B"; } }
```

**เฉลย:** ใช้ `insteadof` เพื่อเลือกว่าจะใช้ method จาก Trait ไหน และ `as` เพื่อสร้าง alias สำหรับ method ที่ไม่ได้เลือก:
```php
class C {
    use A, B {
        A::hello insteadof B;
        B::hello as helloB;
    }
}
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- **Abstract Classes** - แม่แบบที่บังคับให้คลาสลูก implement method ที่กำหนด
- **Interfaces** - สัญญาที่ define พฤติกรรมโดยไม่มี implementation
- **Traits** - กลไก code reuse แบบ horizontal ที่แก้ปัญหา multiple inheritance
- **Late Static Binding** - `static::` สำหรับ polymorphism ใน static context
- **Anonymous Classes** - สร้างคลาสแบบ inline สำหรับการใช้งานครั้งเดียว
- **Plugin System** - การนำ Interfaces ไปใช้งานจริงในการออกแบบระบบ

---

## แหล่งข้อมูลเพิ่มเติม

- [PHP Manual: Abstract Classes](https://www.php.net/manual/en/language.oop5.abstract.php)
- [PHP Manual: Interfaces](https://www.php.net/manual/en/language.oop5.interfaces.php)
- [PHP Manual: Traits](https://www.php.net/manual/en/language.oop5.traits.php)
- [PHP Manual: Late Static Bindings](https://www.php.net/manual/en/language.oop5.late-static-bindings.php)

---

## ➡️ Part ถัดไป

[Part 017: PHP Namespaces และ Autoloading](./part-017-php-namespaces.md)
