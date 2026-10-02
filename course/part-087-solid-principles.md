# Part 87: SOLID Principles ใน PHP

## บทนำ

SOLID เป็นชุดหลักการออกแบบ Object-Oriented ที่ถูกแนะนำโดย Robert C. Martin (Uncle Bob) เพื่อทำให้ Code มีความยืดหยุ่น บำรุงรักษาง่าย และทดสอบได้

- **S** - Single Responsibility Principle
- **O** - Open/Closed Principle  
- **L** - Liskov Substitution Principle
- **I** - Interface Segregation Principle
- **D** - Dependency Inversion Principle

---

## S - Single Responsibility Principle (SRP)

**หลักการ**: Class ควรมีเหตุผลในการเปลี่ยนแปลงเพียงเหตุผลเดียว

### ❌ ละเมิด SRP

```php
<?php

class User
{
    private int $id;
    private string $name;
    private string $email;
    private string $password;
    
    // User logic
    public function __construct(string $name, string $email, string $password)
    {
        $this->name = $name;
        $this->email = $email;
        $this->password = password_hash($password, PASSWORD_BCRYPT);
    }
    
    // Database logic - ไม่ควรอยู่ใน User class
    public function save(): bool
    {
        $db = DatabaseConnection::getInstance();
        $stmt = $db->prepare("INSERT INTO users (name, email, password) VALUES (?, ?, ?)");
        return $stmt->execute([$this->name, $this->email, $this->password]);
    }
    
    public function findById(int $id): ?array
    {
        $db = DatabaseConnection::getInstance();
        $stmt = $db->prepare("SELECT * FROM users WHERE id = ?");
        $stmt->execute([$id]);
        return $stmt->fetch() ?: null;
    }
    
    // Email logic - ไม่ควรอยู่ใน User class
    public function sendWelcomeEmail(): void
    {
        mail($this->email, 'Welcome!', "Dear {$this->name}, Welcome to our platform!");
    }
    
    public function sendPasswordResetEmail(string $token): void
    {
        $link = "https://myapp.com/reset-password?token={$token}";
        mail($this->email, 'Password Reset', "Click here to reset: {$link}");
    }
    
    // Validation logic - ไม่ควรอยู่ใน User class
    public function validateEmail(): bool
    {
        return filter_var($this->email, FILTER_VALIDATE_EMAIL) !== false;
    }
    
    public function validatePassword(string $password): bool
    {
        return strlen($password) >= 8
            && preg_match('/[A-Z]/', $password)
            && preg_match('/[0-9]/', $password);
    }
}
```

### ✅ ปฏิบัติตาม SRP

```php
<?php

// User Entity - รับผิดชอบแค่ User data และ Business logic
class User
{
    private int $id;
    private string $passwordHash;
    
    public function __construct(
        private string $name,
        private string $email,
        string $password
    ) {
        $this->passwordHash = password_hash($password, PASSWORD_BCRYPT);
    }
    
    public function getId(): int { return $this->id; }
    public function getName(): string { return $this->name; }
    public function getEmail(): string { return $this->email; }
    public function getPasswordHash(): string { return $this->passwordHash; }
    
    public function verifyPassword(string $password): bool
    {
        return password_verify($password, $this->passwordHash);
    }
    
    public function changeName(string $newName): void
    {
        if (empty(trim($newName))) {
            throw new \InvalidArgumentException("Name cannot be empty");
        }
        $this->name = $newName;
    }
}

// UserRepository - รับผิดชอบการ Persist User
interface UserRepository
{
    public function save(User $user): bool;
    public function findById(int $id): ?User;
    public function findByEmail(string $email): ?User;
    public function delete(int $id): bool;
}

class MySQLUserRepository implements UserRepository
{
    public function __construct(private \PDO $db) {}
    
    public function save(User $user): bool
    {
        $stmt = $this->db->prepare(
            "INSERT INTO users (name, email, password_hash) VALUES (?, ?, ?)
             ON DUPLICATE KEY UPDATE name = VALUES(name)"
        );
        return $stmt->execute([
            $user->getName(),
            $user->getEmail(),
            $user->getPasswordHash()
        ]);
    }
    
    public function findById(int $id): ?User
    {
        $stmt = $this->db->prepare("SELECT * FROM users WHERE id = ?");
        $stmt->execute([$id]);
        $data = $stmt->fetch(\PDO::FETCH_ASSOC);
        return $data ? $this->hydrateUser($data) : null;
    }
    
    public function findByEmail(string $email): ?User
    {
        $stmt = $this->db->prepare("SELECT * FROM users WHERE email = ?");
        $stmt->execute([$email]);
        $data = $stmt->fetch(\PDO::FETCH_ASSOC);
        return $data ? $this->hydrateUser($data) : null;
    }
    
    public function delete(int $id): bool
    {
        $stmt = $this->db->prepare("DELETE FROM users WHERE id = ?");
        return $stmt->execute([$id]);
    }
    
    private function hydrateUser(array $data): User
    {
        // ... สร้าง User object จาก data
        return new User($data['name'], $data['email'], '');
    }
}

// UserMailer - รับผิดชอบการส่ง Email ที่เกี่ยวกับ User
class UserMailer
{
    public function __construct(private MailerInterface $mailer) {}
    
    public function sendWelcomeEmail(User $user): void
    {
        $this->mailer->send(
            to: $user->getEmail(),
            subject: 'ยินดีต้อนรับ!',
            body: "สวัสดี {$user->getName()}, ขอบคุณที่สมัครสมาชิก!"
        );
    }
    
    public function sendPasswordResetEmail(User $user, string $token): void
    {
        $link = "https://myapp.com/reset-password?token={$token}";
        $this->mailer->send(
            to: $user->getEmail(),
            subject: 'รีเซ็ตรหัสผ่าน',
            body: "คลิกที่ลิ้งนี้เพื่อรีเซ็ตรหัสผ่าน: {$link}"
        );
    }
}

// UserValidator - รับผิดชอบการ Validate User data
class UserValidator
{
    public function validateRegistration(array $data): array
    {
        $errors = [];
        
        if (empty($data['name'])) {
            $errors['name'] = 'กรุณาระบุชื่อ';
        }
        
        if (!filter_var($data['email'] ?? '', FILTER_VALIDATE_EMAIL)) {
            $errors['email'] = 'Email ไม่ถูกต้อง';
        }
        
        if (!$this->isStrongPassword($data['password'] ?? '')) {
            $errors['password'] = 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร ตัวพิมพ์ใหญ่ และตัวเลข';
        }
        
        if ($data['password'] !== ($data['password_confirmation'] ?? '')) {
            $errors['password_confirmation'] = 'รหัสผ่านไม่ตรงกัน';
        }
        
        return $errors;
    }
    
    private function isStrongPassword(string $password): bool
    {
        return strlen($password) >= 8
            && preg_match('/[A-Z]/', $password)
            && preg_match('/[0-9]/', $password);
    }
}
```

---

## O - Open/Closed Principle (OCP)

**หลักการ**: Software entities ควร Open สำหรับการ Extension แต่ Closed สำหรับการ Modification

### ❌ ละเมิด OCP

```php
<?php

class DiscountCalculator
{
    public function calculate(string $type, float $price): float
    {
        // ทุกครั้งที่เพิ่ม discount type ใหม่ ต้อง Modify class นี้
        if ($type === 'percentage') {
            return $price * 0.9; // 10% off
        } elseif ($type === 'fixed') {
            return $price - 50; // 50 THB off
        } elseif ($type === 'buy_two_get_one') {
            // ...
        } elseif ($type === 'flash_sale') {
            // ...
        }
        // เพิ่ม type ใหม่ต้อง modify here...
        
        return $price;
    }
}
```

### ✅ ปฏิบัติตาม OCP

```php
<?php

interface DiscountStrategy
{
    public function apply(float $price): float;
    public function getDescription(): string;
}

class PercentageDiscount implements DiscountStrategy
{
    public function __construct(private float $percentage) {}
    
    public function apply(float $price): float
    {
        return $price * (1 - $this->percentage / 100);
    }
    
    public function getDescription(): string
    {
        return "ลด {$this->percentage}%";
    }
}

class FixedAmountDiscount implements DiscountStrategy
{
    public function __construct(private float $amount) {}
    
    public function apply(float $price): float
    {
        return max(0, $price - $this->amount);
    }
    
    public function getDescription(): string
    {
        return "ลด " . number_format($this->amount, 2) . " บาท";
    }
}

class BuyXGetYFreeDiscount implements DiscountStrategy
{
    public function __construct(
        private int $buyQuantity,
        private int $freeQuantity,
        private float $unitPrice
    ) {}
    
    public function apply(float $totalPrice): float
    {
        $quantity = (int)round($totalPrice / $this->unitPrice);
        $cycles = floor($quantity / ($this->buyQuantity + $this->freeQuantity));
        $discount = $cycles * $this->freeQuantity * $this->unitPrice;
        return $totalPrice - $discount;
    }
    
    public function getDescription(): string
    {
        return "ซื้อ {$this->buyQuantity} แถม {$this->freeQuantity}";
    }
}

class FlashSaleDiscount implements DiscountStrategy
{
    public function __construct(
        private float $percentage,
        private \DateTime $endTime
    ) {}
    
    public function apply(float $price): float
    {
        if (new \DateTime() > $this->endTime) {
            return $price; // หมดเวลา Flash sale
        }
        return $price * (1 - $this->percentage / 100);
    }
    
    public function getDescription(): string
    {
        return "Flash Sale ลด {$this->percentage}% (หมดเวลา: {$this->endTime->format('H:i')})";
    }
}

// เพิ่ม Discount ใหม่ไม่ต้อง modify class นี้
class DiscountCalculator
{
    /** @var DiscountStrategy[] */
    private array $discounts = [];
    
    public function addDiscount(DiscountStrategy $discount): self
    {
        $this->discounts[] = $discount;
        return $this;
    }
    
    public function calculate(float $price): float
    {
        foreach ($this->discounts as $discount) {
            $price = $discount->apply($price);
        }
        return $price;
    }
    
    public function getSummary(): array
    {
        return array_map(fn($d) => $d->getDescription(), $this->discounts);
    }
}

// การใช้งาน
$calculator = new DiscountCalculator();
$calculator
    ->addDiscount(new PercentageDiscount(10))
    ->addDiscount(new FixedAmountDiscount(50));

$finalPrice = $calculator->calculate(500.0);
echo "Final price: " . number_format($finalPrice, 2); // 400.00
echo "\nDiscounts applied:\n";
foreach ($calculator->getSummary() as $desc) {
    echo "- {$desc}\n";
}
```

---

## L - Liskov Substitution Principle (LSP)

**หลักการ**: Objects ของ Subclass ควรใช้แทน Objects ของ Parent class ได้โดยไม่ทำให้โปรแกรมผิดพลาด

### ❌ ละเมิด LSP

```php
<?php

class Rectangle
{
    protected float $width;
    protected float $height;
    
    public function setWidth(float $width): void
    {
        $this->width = $width;
    }
    
    public function setHeight(float $height): void
    {
        $this->height = $height;
    }
    
    public function area(): float
    {
        return $this->width * $this->height;
    }
}

// Square ละเมิด LSP เพราะเปลี่ยนพฤติกรรมของ Parent
class Square extends Rectangle
{
    public function setWidth(float $width): void
    {
        $this->width = $width;
        $this->height = $width; // Override behavior!
    }
    
    public function setHeight(float $height): void
    {
        $this->height = $height;
        $this->width = $height; // Override behavior!
    }
}

// Test ที่ทำงานกับ Rectangle แต่ไม่ทำงานกับ Square
function testRectangle(Rectangle $rect): void
{
    $rect->setWidth(5);
    $rect->setHeight(4);
    
    assert($rect->area() === 20.0, "Expected area 20, got " . $rect->area());
    // Square จะ fail เพราะ area = 4*4 = 16, ไม่ใช่ 20
}
```

### ✅ ปฏิบัติตาม LSP

```php
<?php

// ใช้ Interface แทน Inheritance
interface Shape
{
    public function area(): float;
    public function perimeter(): float;
}

class Rectangle implements Shape
{
    public function __construct(
        private float $width,
        private float $height
    ) {}
    
    public function area(): float
    {
        return $this->width * $this->height;
    }
    
    public function perimeter(): float
    {
        return 2 * ($this->width + $this->height);
    }
}

class Square implements Shape
{
    public function __construct(private float $side) {}
    
    public function area(): float
    {
        return $this->side ** 2;
    }
    
    public function perimeter(): float
    {
        return 4 * $this->side;
    }
}

class Circle implements Shape
{
    public function __construct(private float $radius) {}
    
    public function area(): float
    {
        return M_PI * $this->radius ** 2;
    }
    
    public function perimeter(): float
    {
        return 2 * M_PI * $this->radius;
    }
}

// ทำงานได้กับทุก Shape โดยไม่ต้องรู้ว่าเป็น Shape อะไร
function calculateTotalArea(Shape ...$shapes): float
{
    return array_sum(array_map(fn($s) => $s->area(), $shapes));
}

$shapes = [
    new Rectangle(5, 4),
    new Square(3),
    new Circle(2)
];

echo "Total area: " . number_format(calculateTotalArea(...$shapes), 2);
```

### LSP กับ Exception Handling

```php
<?php

abstract class FileReader
{
    // Contract: ต้อง return string หรือ throw FileNotFoundException
    abstract public function read(string $path): string;
}

class LocalFileReader extends FileReader
{
    public function read(string $path): string
    {
        if (!file_exists($path)) {
            throw new \FileNotFoundException("File not found: {$path}");
        }
        return file_get_contents($path);
    }
}

class S3FileReader extends FileReader
{
    public function read(string $path): string
    {
        // ไม่ควร throw Exception ชนิดใหม่ที่ไม่ได้อยู่ใน Contract
        // หาก S3 ไม่พบ ควร throw FileNotFoundException เหมือนกัน
        try {
            return $this->s3->getObject($path);
        } catch (\S3Exception $e) {
            // แปลง Exception ให้ตรงกับ Contract
            throw new \FileNotFoundException("File not found in S3: {$path}");
        }
    }
}
```

---

## I - Interface Segregation Principle (ISP)

**หลักการ**: Client ไม่ควรถูกบังคับให้ Implement Interface ที่ไม่ได้ใช้

### ❌ ละเมิด ISP

```php
<?php

// "Fat Interface" - มีทุกอย่างใน Interface เดียว
interface Animal
{
    public function eat(): void;
    public function sleep(): void;
    public function run(): void;
    public function fly(): void;     // ไม่ใช่ทุก Animal บิน
    public function swim(): void;    // ไม่ใช่ทุก Animal ว่ายน้ำ
    public function makeSound(): void;
    public function breathe(): void;
    public function layEggs(): void; // ไม่ใช่ทุก Animal วางไข่
}

class Dog implements Animal
{
    public function eat(): void { /* ... */ }
    public function sleep(): void { /* ... */ }
    public function run(): void { /* ... */ }
    public function fly(): void
    {
        throw new \Exception("Dogs cannot fly!"); // บังคับ implement แต่ไม่ได้ใช้
    }
    public function swim(): void { /* some dogs swim */ }
    public function makeSound(): void { /* ... */ }
    public function breathe(): void { /* ... */ }
    public function layEggs(): void
    {
        throw new \Exception("Dogs don't lay eggs!"); // ไม่สมเหตุสมผล
    }
}
```

### ✅ ปฏิบัติตาม ISP

```php
<?php

// แบ่ง Interface ให้เล็กลง ตาม Capability
interface CanEat
{
    public function eat(string $food): void;
}

interface CanSleep
{
    public function sleep(int $hours): void;
}

interface CanRun
{
    public function run(float $speed): void;
}

interface CanFly
{
    public function fly(float $altitude): void;
    public function land(): void;
}

interface CanSwim
{
    public function swim(float $depth): void;
}

interface MakesSound
{
    public function makeSound(): string;
}

interface LaysEggs
{
    public function layEggs(int $count): void;
}

// Implement เฉพาะ Interface ที่เกี่ยวข้อง
class Dog implements CanEat, CanSleep, CanRun, CanSwim, MakesSound
{
    public function eat(string $food): void
    {
        echo "Dog is eating {$food}\n";
    }
    
    public function sleep(int $hours): void
    {
        echo "Dog is sleeping for {$hours} hours\n";
    }
    
    public function run(float $speed): void
    {
        echo "Dog is running at {$speed} km/h\n";
    }
    
    public function swim(float $depth): void
    {
        echo "Dog is swimming at {$depth}m depth\n";
    }
    
    public function makeSound(): string
    {
        return "Woof!";
    }
}

class Eagle implements CanEat, CanFly, CanRun, MakesSound, LaysEggs
{
    public function eat(string $food): void
    {
        echo "Eagle is eating {$food}\n";
    }
    
    public function fly(float $altitude): void
    {
        echo "Eagle is flying at {$altitude}m\n";
    }
    
    public function land(): void
    {
        echo "Eagle is landing\n";
    }
    
    public function run(float $speed): void
    {
        echo "Eagle is running at {$speed} km/h\n";
    }
    
    public function makeSound(): string
    {
        return "Screech!";
    }
    
    public function layEggs(int $count): void
    {
        echo "Eagle lays {$count} eggs\n";
    }
}

// ใช้ Type Hint ที่ Specific
function makeAnimalFly(CanFly $animal, float $altitude): void
{
    $animal->fly($altitude);
}

function feedAnimal(CanEat $animal, string $food): void
{
    $animal->eat($food);
}
```

### ISP ในระบบจริง - Worker Interface

```php
<?php

// ❌ Fat Interface
interface Worker
{
    public function work(): void;
    public function eat(): void;
    public function sleep(): void;
    public function takeBreak(): void;
    public function requestSickLeave(): void;
    public function claimOvertime(): void;
}

// ✅ Segregated Interfaces
interface Workable
{
    public function work(): void;
}

interface Eatable
{
    public function eat(): void;
}

interface Sleepable
{
    public function sleep(): void;
}

interface HumanWorker extends Workable, Eatable, Sleepable
{
    public function takeBreak(): void;
    public function requestSickLeave(): void;
}

class Employee implements HumanWorker
{
    public function work(): void { echo "Employee working\n"; }
    public function eat(): void { echo "Employee eating\n"; }
    public function sleep(): void { echo "Employee sleeping\n"; }
    public function takeBreak(): void { echo "Taking break\n"; }
    public function requestSickLeave(): void { echo "Requesting sick leave\n"; }
}

// Robot ไม่ต้องกิน/นอน
class Robot implements Workable
{
    public function work(): void { echo "Robot working 24/7\n"; }
}
```

---

## D - Dependency Inversion Principle (DIP)

**หลักการ**:
1. High-level modules ไม่ควร depend on Low-level modules แต่ทั้งคู่ควร depend on Abstractions
2. Abstractions ไม่ควร depend on Details แต่ Details ควร depend on Abstractions

### ❌ ละเมิด DIP

```php
<?php

// Low-level module
class MySQLDatabase
{
    public function query(string $sql): array
    {
        // MySQL specific implementation
        return [];
    }
}

// High-level module depend on Low-level module โดยตรง
class UserService
{
    private MySQLDatabase $db; // ❌ Depend on concrete class
    
    public function __construct()
    {
        $this->db = new MySQLDatabase(); // ❌ Direct instantiation
    }
    
    public function getUser(int $id): ?array
    {
        $result = $this->db->query("SELECT * FROM users WHERE id = {$id}");
        return $result[0] ?? null;
    }
}
```

### ✅ ปฏิบัติตาม DIP

```php
<?php

// Abstraction - Interface
interface DatabaseInterface
{
    public function query(string $sql, array $params = []): array;
    public function execute(string $sql, array $params = []): bool;
    public function lastInsertId(): string;
}

// Low-level module ที่ Implement Abstraction
class MySQLDatabase implements DatabaseInterface
{
    private \PDO $pdo;
    
    public function __construct(string $dsn, string $user, string $password)
    {
        $this->pdo = new \PDO($dsn, $user, $password);
    }
    
    public function query(string $sql, array $params = []): array
    {
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($params);
        return $stmt->fetchAll(\PDO::FETCH_ASSOC);
    }
    
    public function execute(string $sql, array $params = []): bool
    {
        $stmt = $this->pdo->prepare($sql);
        return $stmt->execute($params);
    }
    
    public function lastInsertId(): string
    {
        return $this->pdo->lastInsertId();
    }
}

class PostgreSQLDatabase implements DatabaseInterface
{
    // PostgreSQL implementation
    public function query(string $sql, array $params = []): array
    {
        // PostgreSQL specific
        return [];
    }
    
    public function execute(string $sql, array $params = []): bool
    {
        return true;
    }
    
    public function lastInsertId(): string
    {
        return '';
    }
}

class InMemoryDatabase implements DatabaseInterface
{
    private array $data = [];
    
    public function query(string $sql, array $params = []): array
    {
        // Simple in-memory implementation for testing
        return $this->data;
    }
    
    public function execute(string $sql, array $params = []): bool
    {
        return true;
    }
    
    public function lastInsertId(): string
    {
        return (string)count($this->data);
    }
}

// High-level module depend on Abstraction
class UserService
{
    // ✅ Depend on Abstraction
    public function __construct(private DatabaseInterface $db) {}
    
    public function getUser(int $id): ?array
    {
        $result = $this->db->query("SELECT * FROM users WHERE id = ?", [$id]);
        return $result[0] ?? null;
    }
    
    public function createUser(array $data): int
    {
        $this->db->execute(
            "INSERT INTO users (name, email) VALUES (?, ?)",
            [$data['name'], $data['email']]
        );
        return (int)$this->db->lastInsertId();
    }
}

// การใช้งาน - Inject dependency จากภายนอก
$mysql = new MySQLDatabase('mysql:host=localhost;dbname=mydb', 'root', 'secret');
$userService = new UserService($mysql);

// Testing - inject InMemoryDatabase
$testDb = new InMemoryDatabase();
$testUserService = new UserService($testDb);

// Laravel Container จัดการ DI ให้
// App::bind(DatabaseInterface::class, MySQLDatabase::class);
```

### Dependency Injection Container

```php
<?php

class Container
{
    private array $bindings = [];
    private array $instances = [];
    
    public function bind(string $abstract, callable|string $concrete): void
    {
        $this->bindings[$abstract] = $concrete;
    }
    
    public function singleton(string $abstract, callable|string $concrete): void
    {
        $this->bindings[$abstract] = function() use ($abstract, $concrete) {
            if (!isset($this->instances[$abstract])) {
                $this->instances[$abstract] = $this->resolve($concrete);
            }
            return $this->instances[$abstract];
        };
    }
    
    public function make(string $abstract): mixed
    {
        if (isset($this->bindings[$abstract])) {
            $binding = $this->bindings[$abstract];
            if (is_callable($binding)) {
                return $binding($this);
            }
            return $this->resolve($binding);
        }
        
        return $this->resolve($abstract);
    }
    
    private function resolve(string $class): mixed
    {
        $reflection = new \ReflectionClass($class);
        $constructor = $reflection->getConstructor();
        
        if ($constructor === null) {
            return new $class();
        }
        
        $params = [];
        foreach ($constructor->getParameters() as $param) {
            $type = $param->getType();
            if ($type instanceof \ReflectionNamedType && !$type->isBuiltin()) {
                $params[] = $this->make($type->getName());
            } elseif ($param->isDefaultValueAvailable()) {
                $params[] = $param->getDefaultValue();
            }
        }
        
        return $reflection->newInstanceArgs($params);
    }
}

// การใช้งาน
$container = new Container();

$container->singleton(DatabaseInterface::class, function() {
    return new MySQLDatabase('mysql:host=localhost;dbname=mydb', 'root', 'secret');
});

$container->bind(UserService::class, UserService::class);

$userService = $container->make(UserService::class);
```

---

## การ Refactor Code ที่ละเมิด SOLID

### ตัวอย่าง: Legacy Order Processing

```php
<?php

// ❌ Legacy Code ที่ละเมิดทุก SOLID Principle
class OrderProcessor
{
    public function process(array $orderData): array
    {
        // Validation (SRP violation)
        if (empty($orderData['items'])) {
            throw new \Exception("No items");
        }
        if ($orderData['total'] <= 0) {
            throw new \Exception("Invalid total");
        }
        
        // Database (DIP violation)
        $db = new \PDO('mysql:host=localhost;dbname=shop', 'root', '');
        
        // Calculate discount (OCP violation - ต้อง modify เพื่อเพิ่ม discount type)
        $discount = 0;
        if ($orderData['coupon'] === 'SAVE10') {
            $discount = $orderData['total'] * 0.1;
        } elseif ($orderData['coupon'] === 'FLAT50') {
            $discount = 50;
        }
        
        $finalTotal = $orderData['total'] - $discount;
        
        // Payment (DIP violation)
        if ($orderData['payment_method'] === 'stripe') {
            // Stripe specific code
            $charge = \Stripe\Charge::create([
                'amount' => $finalTotal * 100,
                'currency' => 'thb',
                'source' => $orderData['token']
            ]);
            $transactionId = $charge->id;
        } elseif ($orderData['payment_method'] === 'paypal') {
            // PayPal specific code
            // ...
            $transactionId = 'PP-' . uniqid();
        }
        
        // Save to database
        $stmt = $db->prepare("INSERT INTO orders ...");
        $stmt->execute([...]);
        
        // Send email (SRP violation)
        mail($orderData['email'], 'Order Confirmation', '...');
        
        return ['order_id' => $db->lastInsertId(), 'total' => $finalTotal];
    }
}
```

### ✅ Refactored Code

```php
<?php

// Step 1: แยก Validation
class OrderValidator
{
    public function validate(array $orderData): void
    {
        if (empty($orderData['items'])) {
            throw new \InvalidArgumentException("Order must have at least one item");
        }
        
        if ($orderData['total'] <= 0) {
            throw new \InvalidArgumentException("Order total must be positive");
        }
        
        foreach ($orderData['items'] as $item) {
            if ($item['quantity'] <= 0) {
                throw new \InvalidArgumentException("Item quantity must be positive");
            }
        }
    }
}

// Step 2: แยก Discount Calculation (OCP)
interface DiscountRule
{
    public function isApplicable(array $orderData): bool;
    public function getDiscount(float $total): float;
}

class CouponDiscount implements DiscountRule
{
    private array $coupons = [
        'SAVE10' => ['type' => 'percentage', 'value' => 10],
        'FLAT50' => ['type' => 'fixed', 'value' => 50],
        'VIP20' => ['type' => 'percentage', 'value' => 20],
    ];
    
    public function isApplicable(array $orderData): bool
    {
        return isset($this->coupons[$orderData['coupon'] ?? '']);
    }
    
    public function getDiscount(float $total): float
    {
        // ต้องส่ง coupon code มาด้วย แต่ทำ simple version ก่อน
        return 0;
    }
}

// Step 3: แยก Payment Gateway (DIP)
interface PaymentProcessor
{
    public function charge(float $amount, array $details): PaymentResult;
}

class StripePaymentProcessor implements PaymentProcessor
{
    public function charge(float $amount, array $details): PaymentResult
    {
        // Stripe implementation
        return new PaymentResult(true, 'ch_' . uniqid(), 'Stripe payment successful');
    }
}

// Step 4: แยก Order Repository
interface OrderRepository
{
    public function save(array $order): string;
}

class MySQLOrderRepository implements OrderRepository
{
    public function __construct(private \PDO $db) {}
    
    public function save(array $order): string
    {
        $stmt = $this->db->prepare(
            "INSERT INTO orders (customer_email, total, status) VALUES (?, ?, ?)"
        );
        $stmt->execute([$order['email'], $order['total'], $order['status']]);
        return $this->db->lastInsertId();
    }
}

// Step 5: แยก Notification
interface OrderNotifier
{
    public function notify(array $order): void;
}

class EmailOrderNotifier implements OrderNotifier
{
    public function notify(array $order): void
    {
        mail(
            $order['email'],
            'ยืนยันคำสั่งซื้อ #' . $order['id'],
            "ขอบคุณสำหรับการสั่งซื้อ ยอดรวม: " . number_format($order['total'], 2)
        );
    }
}

// Step 6: Refactored OrderProcessor (High-level module)
class OrderProcessor
{
    public function __construct(
        private OrderValidator $validator,
        private PaymentProcessor $paymentProcessor,
        private OrderRepository $repository,
        private OrderNotifier $notifier,
        /** @var DiscountRule[] */
        private array $discountRules = []
    ) {}
    
    public function process(array $orderData): array
    {
        // Validate
        $this->validator->validate($orderData);
        
        // Calculate discount
        $discount = 0;
        foreach ($this->discountRules as $rule) {
            if ($rule->isApplicable($orderData)) {
                $discount += $rule->getDiscount($orderData['total']);
            }
        }
        
        $finalTotal = $orderData['total'] - $discount;
        
        // Process payment
        $paymentResult = $this->paymentProcessor->charge(
            $finalTotal,
            $orderData['payment_details']
        );
        
        if (!$paymentResult->success) {
            throw new \RuntimeException("Payment failed: {$paymentResult->message}");
        }
        
        // Save order
        $orderId = $this->repository->save([
            'email' => $orderData['email'],
            'total' => $finalTotal,
            'status' => 'confirmed',
            'transaction_id' => $paymentResult->transactionId
        ]);
        
        $order = array_merge($orderData, [
            'id' => $orderId,
            'total' => $finalTotal,
            'discount' => $discount,
            'transaction_id' => $paymentResult->transactionId
        ]);
        
        // Notify
        $this->notifier->notify($order);
        
        return $order;
    }
}

// การใช้งาน
$processor = new OrderProcessor(
    validator: new OrderValidator(),
    paymentProcessor: new StripePaymentProcessor(),
    repository: new MySQLOrderRepository($pdo),
    notifier: new EmailOrderNotifier(),
    discountRules: [new CouponDiscount()]
);
```

---

## Workshop: Refactor Legacy Code

### Scenario: Blog Management System

```php
<?php

// Legacy Code
class BlogPost
{
    public function publishPost(
        string $title,
        string $content,
        int $authorId,
        array $tags,
        bool $sendNewsletter = false
    ): array {
        // 1. Validate
        if (strlen($title) < 10) throw new \Exception("Title too short");
        if (strlen($content) < 100) throw new \Exception("Content too short");
        
        // 2. Save to DB
        $db = new \mysqli('localhost', 'root', '', 'blog');
        $stmt = $db->prepare("INSERT INTO posts (title, content, author_id) VALUES (?, ?, ?)");
        $stmt->bind_param('ssi', $title, $content, $authorId);
        $stmt->execute();
        $postId = $db->insert_id;
        
        // 3. Handle tags
        foreach ($tags as $tag) {
            $tagStmt = $db->prepare("INSERT IGNORE INTO tags (name) VALUES (?)");
            $tagStmt->bind_param('s', $tag);
            $tagStmt->execute();
            $tagId = $db->insert_id;
            
            $pivotStmt = $db->prepare("INSERT INTO post_tags (post_id, tag_id) VALUES (?, ?)");
            $pivotStmt->bind_param('ii', $postId, $tagId);
            $pivotStmt->execute();
        }
        
        // 4. Send newsletter
        if ($sendNewsletter) {
            $subscribers = $db->query("SELECT email FROM subscribers");
            while ($subscriber = $subscribers->fetch_assoc()) {
                mail($subscriber['email'], "New Post: {$title}", $content);
            }
        }
        
        // 5. Clear cache
        unlink('/tmp/posts_cache.json');
        
        return ['id' => $postId, 'title' => $title, 'status' => 'published'];
    }
}

// ✅ Refactored Version

interface PostValidator
{
    public function validate(array $data): void;
}

class BlogPostValidator implements PostValidator
{
    public function validate(array $data): void
    {
        $errors = [];
        
        if (strlen($data['title'] ?? '') < 10) {
            $errors['title'] = "Title must be at least 10 characters";
        }
        
        if (strlen($data['content'] ?? '') < 100) {
            $errors['content'] = "Content must be at least 100 characters";
        }
        
        if (!empty($errors)) {
            throw new ValidationException($errors);
        }
    }
}

interface PostRepository
{
    public function create(array $data): int;
    public function attachTags(int $postId, array $tags): void;
    public function findAll(): array;
}

interface NewsletterService
{
    public function sendNewPostNotification(array $post): void;
}

interface CacheService
{
    public function invalidate(string $key): void;
}

class PostPublisher
{
    public function __construct(
        private PostValidator $validator,
        private PostRepository $repository,
        private NewsletterService $newsletter,
        private CacheService $cache
    ) {}
    
    public function publish(array $data): array
    {
        $this->validator->validate($data);
        
        $postId = $this->repository->create([
            'title' => $data['title'],
            'content' => $data['content'],
            'author_id' => $data['author_id'],
            'status' => 'published',
            'published_at' => date('Y-m-d H:i:s')
        ]);
        
        if (!empty($data['tags'])) {
            $this->repository->attachTags($postId, $data['tags']);
        }
        
        $post = array_merge($data, ['id' => $postId]);
        
        if ($data['send_newsletter'] ?? false) {
            $this->newsletter->sendNewPostNotification($post);
        }
        
        $this->cache->invalidate('posts_list');
        
        return $post;
    }
}
```

---

## สรุป SOLID

| Principle | ย่อ | ใจความ | ประโยชน์ |
|-----------|-----|--------|---------|
| Single Responsibility | SRP | 1 class 1 หน้าที่ | ง่ายต่อการแก้ไขและทดสอบ |
| Open/Closed | OCP | Extend ได้ Modify ไม่ได้ | รองรับการเปลี่ยนแปลงในอนาคต |
| Liskov Substitution | LSP | Subclass ใช้แทน Superclass ได้ | Polymorphism ที่ถูกต้อง |
| Interface Segregation | ISP | Interface เล็กและ Specific | ลด Dependencies ที่ไม่จำเป็น |
| Dependency Inversion | DIP | Depend on Abstractions | Loosely coupled, Testable |

---

*SOLID ไม่ใช่กฎที่ต้องทำตามตลอดเวลา แต่เป็นหลักการที่ช่วยให้ตัดสินใจออกแบบได้ดีขึ้น*
