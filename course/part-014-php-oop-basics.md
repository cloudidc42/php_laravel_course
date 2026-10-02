# 🏗️ Part 14: PHP OOP Basics - Object-Oriented Programming พื้นฐาน

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้างและใช้งาน Class & Object ได้
- กำหนด Properties & Methods พร้อม Access Modifiers ได้
- เขียน Constructor & Destructor ได้
- ใช้ Static properties & methods ได้
- ใช้งาน Magic Methods ต่างๆ ได้
- Clone Object ได้อย่างถูกต้อง
- สร้าง BankAccount class ที่ครบฟีเจอร์ได้

---

## 📌 1. Classes & Objects

### 1.1 Class คืออะไร?

Class คือ "แบบพิมพ์เขียว" (blueprint) ที่ใช้สร้าง Object โดย Object คือ instance ของ Class

```php
<?php
// นิยาม Class
class Car {
    // Properties (ตัวแปรของ Class)
    public string $brand;
    public string $color;
    public int $year;

    // Method (ฟังก์ชันของ Class)
    public function drive(): string {
        return "{$this->brand} กำลังวิ่ง!";
    }

    public function getInfo(): string {
        return "{$this->year} {$this->brand} สี{$this->color}";
    }
}

// สร้าง Object จาก Class
$myCar = new Car();
$myCar->brand = 'Toyota';
$myCar->color = 'แดง';
$myCar->year  = 2024;

echo $myCar->drive();   // Toyota กำลังวิ่ง!
echo $myCar->getInfo(); // 2024 Toyota สีแดง
```

### 1.2 $this Keyword

`$this` ใช้อ้างถึง Object ปัจจุบันภายใน Class

```php
<?php
class Person {
    public string $name;
    public int $age;

    public function introduce(): string {
        // $this อ้างถึง object ที่เรียก method นี้
        return "สวัสดี ผมชื่อ {$this->name} อายุ {$this->age} ปี";
    }

    public function isAdult(): bool {
        return $this->age >= 18;
    }
}

$person = new Person();
$person->name = 'สมชาย';
$person->age  = 25;

echo $person->introduce(); // สวัสดี ผมชื่อ สมชาย อายุ 25 ปี
var_dump($person->isAdult()); // bool(true)
```

---

## 📌 2. Properties & Methods

### 2.1 Property Types

```php
<?php
class Product {
    // Typed properties (PHP 7.4+)
    public string $name;
    public float $price;
    public int $stock = 0;       // มี default value
    public bool $isActive = true;
    public ?string $description = null; // nullable

    // PHP 8.0+ property types
    public int|string $id;       // Union type
}
```

### 2.2 Method Return Types

```php
<?php
class Calculator {
    public function add(int $a, int $b): int {
        return $a + $b;
    }

    public function divide(float $a, float $b): float|false {
        if ($b === 0.0) {
            return false;
        }
        return $a / $b;
    }

    // void = ไม่ return อะไร
    public function printResult(int $result): void {
        echo "ผลลัพธ์: {$result}";
    }
}

$calc = new Calculator();
echo $calc->add(5, 3);        // 8
var_dump($calc->divide(10, 0)); // bool(false)
```

### 2.3 Fluent Interface (Method Chaining)

```php
<?php
class QueryBuilder {
    private string $table = '';
    private array $conditions = [];
    private ?int $limitValue = null;

    public function from(string $table): static {
        $this->table = $table;
        return $this; // คืนตัวเองเพื่อ chain
    }

    public function where(string $condition): static {
        $this->conditions[] = $condition;
        return $this;
    }

    public function limit(int $n): static {
        $this->limitValue = $n;
        return $this;
    }

    public function build(): string {
        $sql = "SELECT * FROM {$this->table}";
        if (!empty($this->conditions)) {
            $sql .= " WHERE " . implode(' AND ', $this->conditions);
        }
        if ($this->limitValue !== null) {
            $sql .= " LIMIT {$this->limitValue}";
        }
        return $sql;
    }
}

// Method Chaining
$query = (new QueryBuilder())
    ->from('users')
    ->where('age > 18')
    ->where('active = 1')
    ->limit(10)
    ->build();

echo $query;
// SELECT * FROM users WHERE age > 18 AND active = 1 LIMIT 10
```

---

## 📌 3. Constructors & Destructors

### 3.1 __construct()

Constructor ทำงานอัตโนมัติเมื่อสร้าง Object ใหม่

```php
<?php
class DatabaseConnection {
    private \PDO $pdo;
    private static int $connectionCount = 0;

    public function __construct(
        private string $host,
        private string $dbname,
        private string $username,
        private string $password
    ) {
        $this->connect();
        self::$connectionCount++;
        echo "เปิดการเชื่อมต่อ #{" . self::$connectionCount . "}\n";
    }

    private function connect(): void {
        $dsn = "mysql:host={$this->host};dbname={$this->dbname};charset=utf8mb4";
        $this->pdo = new \PDO($dsn, $this->username, $this->password, [
            \PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION,
        ]);
    }

    public function getPdo(): \PDO {
        return $this->pdo;
    }

    public static function getConnectionCount(): int {
        return self::$connectionCount;
    }
}
```

### 3.2 __destruct()

Destructor ทำงานอัตโนมัติเมื่อ Object ถูก destroy (หมด scope หรือ unset)

```php
<?php
class FileLogger {
    private $fileHandle;
    private string $filename;

    public function __construct(string $filename) {
        $this->filename   = $filename;
        $this->fileHandle = fopen($filename, 'a');
        $this->log("Logger เริ่มทำงาน");
    }

    public function log(string $message): void {
        $timestamp = date('Y-m-d H:i:s');
        fwrite($this->fileHandle, "[{$timestamp}] {$message}\n");
    }

    public function __destruct() {
        $this->log("Logger หยุดทำงาน");
        if (is_resource($this->fileHandle)) {
            fclose($this->fileHandle); // ปิดไฟล์อัตโนมัติ
        }
        echo "FileLogger ถูก destroy แล้ว\n";
    }
}

{
    $logger = new FileLogger('/tmp/app.log');
    $logger->log("บันทึกข้อมูล...");
    // เมื่อออกจาก block นี้ $logger ถูก destroy → __destruct() ทำงาน
}
// ที่นี่ $logger ไม่มีอีกแล้ว
```

### 3.3 Constructor Promotion (PHP 8.0+)

```php
<?php
// แบบเดิม (PHP 7)
class UserOld {
    private string $name;
    private string $email;
    private int $age;

    public function __construct(string $name, string $email, int $age) {
        $this->name  = $name;
        $this->email = $email;
        $this->age   = $age;
    }
}

// แบบใหม่ (PHP 8.0) - Constructor Promotion
class User {
    public function __construct(
        private string $name,
        private string $email,
        private int $age,
        private bool $isActive = true
    ) {
        // Properties ถูกกำหนดอัตโนมัติ ไม่ต้องเขียนซ้ำ
    }

    public function getName(): string { return $this->name; }
    public function getEmail(): string { return $this->email; }
}

$user = new User('สมหญิง', 'somying@example.com', 28);
echo $user->getName(); // สมหญิง
```

---

## 📌 4. Access Modifiers

### 4.1 public, protected, private

| Modifier | ภายใน Class | Subclass | นอก Class |
|----------|------------|----------|-----------|
| `public` | ✅ | ✅ | ✅ |
| `protected` | ✅ | ✅ | ❌ |
| `private` | ✅ | ❌ | ❌ |

```php
<?php
class Employee {
    public string $name;           // ทุกคนเข้าถึงได้
    protected float $baseSalary;   // เฉพาะ class นี้และ subclass
    private string $taxId;         // เฉพาะ class นี้เท่านั้น

    public function __construct(string $name, float $salary, string $taxId) {
        $this->name       = $name;
        $this->baseSalary = $salary;
        $this->taxId      = $taxId;
    }

    public function getSalary(): float {
        return $this->baseSalary; // เข้าถึง protected ได้
    }

    private function getTaxId(): string {
        return $this->taxId; // เข้าถึง private ได้
    }

    public function getInfo(): string {
        return "{$this->name} | เลขที่: " . $this->getTaxId();
    }
}

class Manager extends Employee {
    public function getBonus(): float {
        return $this->baseSalary * 0.2; // ✅ เข้าถึง protected ได้
    }

    public function showTax(): void {
        // echo $this->taxId; // ❌ Error: Cannot access private property
    }
}

$emp = new Employee('สมชาย', 50000, 'TAX001');
echo $emp->name;        // ✅ สมชาย
// echo $emp->baseSalary; // ❌ Error
// echo $emp->taxId;      // ❌ Error
```

### 4.2 Getters & Setters (Encapsulation)

```php
<?php
class Temperature {
    private float $celsius;

    public function __construct(float $celsius) {
        $this->setCelsius($celsius);
    }

    public function getCelsius(): float {
        return $this->celsius;
    }

    public function setCelsius(float $value): void {
        if ($value < -273.15) {
            throw new \InvalidArgumentException(
                "อุณหภูมิต้องไม่ต่ำกว่า -273.15°C (absolute zero)"
            );
        }
        $this->celsius = $value;
    }

    public function getFahrenheit(): float {
        return ($this->celsius * 9 / 5) + 32;
    }

    public function getKelvin(): float {
        return $this->celsius + 273.15;
    }
}

$temp = new Temperature(100);
echo $temp->getCelsius();    // 100
echo $temp->getFahrenheit(); // 212
echo $temp->getKelvin();     // 373.15

$temp->setCelsius(-300); // Throws InvalidArgumentException
```

---

## 📌 5. Static Properties & Methods

### 5.1 static keyword

```php
<?php
class Counter {
    private static int $count = 0;    // shared ระหว่างทุก instance

    public function __construct() {
        self::$count++;
    }

    public static function getCount(): int {
        return self::$count;
    }

    public static function reset(): void {
        self::$count = 0;
    }
}

$a = new Counter();
$b = new Counter();
$c = new Counter();

echo Counter::getCount(); // 3 (static เข้าถึงผ่าน ClassName::)
Counter::reset();
echo Counter::getCount(); // 0
```

### 5.2 Singleton Pattern ด้วย Static

```php
<?php
class Config {
    private static ?Config $instance = null;
    private array $data = [];

    private function __construct() {
        // private constructor = ป้องกันการสร้าง instance ตรงๆ
        $this->data = [
            'app_name' => 'MyApp',
            'version'  => '1.0.0',
            'debug'    => false,
        ];
    }

    public static function getInstance(): static {
        if (self::$instance === null) {
            self::$instance = new static();
        }
        return self::$instance;
    }

    public function get(string $key, mixed $default = null): mixed {
        return $this->data[$key] ?? $default;
    }

    public function set(string $key, mixed $value): void {
        $this->data[$key] = $value;
    }
}

// ได้ instance เดิมเสมอ
$config1 = Config::getInstance();
$config2 = Config::getInstance();

var_dump($config1 === $config2); // bool(true)

$config1->set('debug', true);
echo $config2->get('debug') ? 'true' : 'false'; // true (object เดียวกัน)
```

### 5.3 self vs static (Late Static Binding)

```php
<?php
class Base {
    public static function create(): static {
        return new static(); // static = class ที่ถูกเรียก (late binding)
    }

    public static function className(): string {
        return static::class; // ชื่อ class ที่เรียก
    }
}

class Child extends Base {
    // ไม่ต้อง override create()
}

$base  = Base::create();   // instance ของ Base
$child = Child::create();  // instance ของ Child (ไม่ใช่ Base!)

echo Base::className();  // Base
echo Child::className(); // Child
```

---

## 📌 6. Magic Methods

Magic methods เป็น method พิเศษที่ PHP เรียกอัตโนมัติในสถานการณ์ต่างๆ

### 6.1 __toString()

```php
<?php
class Money {
    public function __construct(
        private float $amount,
        private string $currency = 'THB'
    ) {}

    public function __toString(): string {
        return number_format($this->amount, 2) . ' ' . $this->currency;
    }
}

$price = new Money(1500.50);
echo $price;                    // 1,500.50 THB
echo "ราคา: " . $price;         // ราคา: 1,500.50 THB
echo sprintf("ชำระ: %s", $price); // ชำระ: 1,500.50 THB
```

### 6.2 __get(), __set(), __isset(), __unset()

```php
<?php
class DynamicObject {
    private array $data = [];
    private array $allowedKeys = ['name', 'email', 'phone'];

    public function __get(string $name): mixed {
        echo "__get called for: {$name}\n";
        return $this->data[$name] ?? null;
    }

    public function __set(string $name, mixed $value): void {
        echo "__set called for: {$name}\n";
        if (!in_array($name, $this->allowedKeys)) {
            throw new \InvalidArgumentException("ไม่อนุญาต key: {$name}");
        }
        $this->data[$name] = $value;
    }

    public function __isset(string $name): bool {
        echo "__isset called for: {$name}\n";
        return isset($this->data[$name]);
    }

    public function __unset(string $name): void {
        echo "__unset called for: {$name}\n";
        unset($this->data[$name]);
    }
}

$obj = new DynamicObject();
$obj->name  = 'สมชาย';   // __set called
echo $obj->name;          // __get called → สมชาย
var_dump(isset($obj->name)); // __isset called → true
unset($obj->name);           // __unset called
var_dump(isset($obj->name)); // __isset called → false
```

### 6.3 __call() และ __callStatic()

```php
<?php
class Router {
    private array $routes = [];

    // เรียกเมื่อ method ไม่มีอยู่บน instance
    public function __call(string $method, array $args): mixed {
        $httpMethods = ['get', 'post', 'put', 'patch', 'delete'];

        if (in_array(strtolower($method), $httpMethods)) {
            [$path, $handler] = $args;
            $this->routes[] = [
                'method'  => strtoupper($method),
                'path'    => $path,
                'handler' => $handler,
            ];
            echo "ลงทะเบียน route: " . strtoupper($method) . " {$path}\n";
            return $this;
        }

        throw new \BadMethodCallException("Method ไม่รองรับ: {$method}");
    }

    // เรียกเมื่อ static method ไม่มีอยู่
    public static function __callStatic(string $method, array $args): void {
        echo "เรียก static method: {$method} ด้วย args: " . implode(', ', $args) . "\n";
    }

    public function getRoutes(): array {
        return $this->routes;
    }
}

$router = new Router();
$router->get('/users', 'UserController@index');
$router->post('/users', 'UserController@store');
$router->delete('/users/{id}', 'UserController@destroy');

Router::someStaticMethod('a', 'b'); // __callStatic called
```

### 6.4 __invoke()

```php
<?php
class Multiplier {
    public function __construct(private int $factor) {}

    // เรียกเมื่อใช้ object เหมือน function
    public function __invoke(int $value): int {
        return $value * $this->factor;
    }
}

$double = new Multiplier(2);
$triple = new Multiplier(3);

echo $double(5);  // 10
echo $triple(5);  // 15

// ใช้เป็น callback ได้เลย
$numbers = [1, 2, 3, 4, 5];
$doubled = array_map($double, $numbers);
print_r($doubled); // [2, 4, 6, 8, 10]

// ตรวจสอบว่า object เป็น callable
var_dump(is_callable($double)); // bool(true)
```

### 6.5 __clone()

```php
<?php
class DeepCopyExample {
    public array $tags = [];
    public \DateTime $createdAt;

    public function __construct(array $tags) {
        $this->tags      = $tags;
        $this->createdAt = new \DateTime();
    }

    public function __clone() {
        // Deep clone: สร้าง DateTime ใหม่
        $this->createdAt = clone $this->createdAt;
        echo "Object ถูก clone แล้ว\n";
    }
}

$original = new DeepCopyExample(['php', 'oop']);
$cloned   = clone $original; // __clone() ถูกเรียก

// แก้ไข clone ไม่กระทบ original
$cloned->tags[] = 'laravel';
$cloned->createdAt->modify('+1 day');

echo count($original->tags); // 2 (ไม่เปลี่ยน)
echo count($cloned->tags);   // 3
```

---

## 📌 7. Object Cloning

### 7.1 Shallow Clone vs Deep Clone

```php
<?php
class Address {
    public function __construct(
        public string $city,
        public string $country
    ) {}
}

class UserProfile {
    public function __construct(
        public string $name,
        public Address $address // object reference
    ) {}
}

$user1 = new UserProfile('สมชาย', new Address('กรุงเทพ', 'ไทย'));
$user2 = clone $user1; // Shallow clone

// Shallow clone: $address ยัง share กัน!
$user2->name = 'สมหญิง';
$user2->address->city = 'เชียงใหม่';

echo $user1->name;          // สมชาย (ไม่เปลี่ยน)
echo $user1->address->city; // เชียงใหม่ (เปลี่ยนตาม!)

// Deep clone ด้วย __clone
class UserProfileDeep {
    public function __construct(
        public string $name,
        public Address $address
    ) {}

    public function __clone() {
        $this->address = clone $this->address; // deep clone address ด้วย
    }
}

$user3 = new UserProfileDeep('สมชาย', new Address('กรุงเทพ', 'ไทย'));
$user4 = clone $user3;

$user4->address->city = 'เชียงใหม่';
echo $user3->address->city; // กรุงเทพ (ไม่เปลี่ยน!)
```

---

## 🛠️ Workshop: BankAccount Class

สร้าง `BankAccount` class ที่มีฟีเจอร์ครบถ้วน

```php
<?php
declare(strict_types=1);

class InsufficientFundsException extends \RuntimeException {}
class InvalidAmountException extends \InvalidArgumentException {}

class Transaction {
    private string $id;
    private \DateTimeImmutable $timestamp;

    public function __construct(
        private string $type,       // 'deposit', 'withdraw', 'transfer_in', 'transfer_out'
        private float $amount,
        private float $balanceAfter,
        private string $description = ''
    ) {
        $this->id        = uniqid('TXN_', true);
        $this->timestamp = new \DateTimeImmutable();
    }

    public function getId(): string { return $this->id; }
    public function getType(): string { return $this->type; }
    public function getAmount(): float { return $this->amount; }
    public function getBalanceAfter(): float { return $this->balanceAfter; }
    public function getDescription(): string { return $this->description; }
    public function getTimestamp(): \DateTimeImmutable { return $this->timestamp; }

    public function __toString(): string {
        $sign = in_array($this->type, ['deposit', 'transfer_in']) ? '+' : '-';
        return sprintf(
            "[%s] %s %s%.2f THB | คงเหลือ: %.2f THB | %s",
            $this->timestamp->format('Y-m-d H:i:s'),
            strtoupper($this->type),
            $sign,
            $this->amount,
            $this->balanceAfter,
            $this->description ?: '-'
        );
    }
}

class BankAccount {
    private string $accountNumber;
    private float $balance;
    private array $transactions = [];
    private static int $accountCounter = 1000;
    private bool $isFrozen = false;

    public function __construct(
        private string $ownerName,
        float $initialDeposit = 0.0,
        private float $interestRate = 0.015 // 1.5% ต่อปี
    ) {
        self::$accountCounter++;
        $this->accountNumber = 'TH' . str_pad((string) self::$accountCounter, 10, '0', STR_PAD_LEFT);
        $this->balance       = 0.0;

        if ($initialDeposit > 0) {
            $this->deposit($initialDeposit, 'ยอดเงินเปิดบัญชี');
        }
    }

    // ฝากเงิน
    public function deposit(float $amount, string $description = ''): static {
        $this->validateAmount($amount);
        $this->checkNotFrozen();

        $this->balance += $amount;
        $this->recordTransaction('deposit', $amount, $description ?: 'ฝากเงิน');

        return $this;
    }

    // ถอนเงิน
    public function withdraw(float $amount, string $description = ''): static {
        $this->validateAmount($amount);
        $this->checkNotFrozen();

        if ($amount > $this->balance) {
            throw new InsufficientFundsException(
                "ยอดเงินไม่เพียงพอ: มี {$this->balance} THB แต่ต้องการ {$amount} THB"
            );
        }

        $this->balance -= $amount;
        $this->recordTransaction('withdraw', $amount, $description ?: 'ถอนเงิน');

        return $this;
    }

    // โอนเงิน
    public function transfer(BankAccount $target, float $amount, string $description = ''): static {
        $this->validateAmount($amount);
        $this->checkNotFrozen();
        $target->checkNotFrozen();

        if ($amount > $this->balance) {
            throw new InsufficientFundsException(
                "ยอดเงินไม่เพียงพอสำหรับการโอน: {$amount} THB"
            );
        }

        $desc = $description ?: "โอนไปยัง {$target->ownerName}";

        $this->balance -= $amount;
        $this->recordTransaction('transfer_out', $amount, $desc);

        $target->balance += $amount;
        $target->recordTransaction('transfer_in', $amount, "รับโอนจาก {$this->ownerName}");

        return $this;
    }

    // ดอกเบี้ย
    public function applyInterest(): static {
        $interest = round($this->balance * $this->interestRate, 2);
        if ($interest > 0) {
            $this->deposit($interest, 'ดอกเบี้ยรายปี');
        }
        return $this;
    }

    // อายัดบัญชี
    public function freeze(): static {
        $this->isFrozen = true;
        return $this;
    }

    public function unfreeze(): static {
        $this->isFrozen = false;
        return $this;
    }

    // ประวัติรายการ
    public function getHistory(int $limit = 0): array {
        $txns = array_reverse($this->transactions); // ล่าสุดก่อน
        return $limit > 0 ? array_slice($txns, 0, $limit) : $txns;
    }

    public function printStatement(): void {
        echo "════════════════════════════════════════\n";
        echo "  บัญชีธนาคาร: {$this->accountNumber}\n";
        echo "  เจ้าของ: {$this->ownerName}\n";
        echo "  ยอดคงเหลือ: " . number_format($this->balance, 2) . " THB\n";
        echo "  สถานะ: " . ($this->isFrozen ? '🔒 อายัด' : '✅ ปกติ') . "\n";
        echo "════════════════════════════════════════\n";
        echo "  ประวัติรายการ:\n";
        foreach ($this->transactions as $txn) {
            echo "  " . $txn . "\n";
        }
        echo "════════════════════════════════════════\n";
    }

    // Getters
    public function getBalance(): float { return $this->balance; }
    public function getAccountNumber(): string { return $this->accountNumber; }
    public function getOwnerName(): string { return $this->ownerName; }
    public function isFrozen(): bool { return $this->isFrozen; }

    public function __toString(): string {
        return "[{$this->accountNumber}] {$this->ownerName}: " .
               number_format($this->balance, 2) . " THB";
    }

    // Magic method สำหรับ debug
    public function __debugInfo(): array {
        return [
            'accountNumber'    => $this->accountNumber,
            'owner'            => $this->ownerName,
            'balance'          => $this->balance,
            'transactionCount' => count($this->transactions),
            'isFrozen'         => $this->isFrozen,
        ];
    }

    // Private helpers
    private function validateAmount(float $amount): void {
        if ($amount <= 0) {
            throw new InvalidAmountException("จำนวนเงินต้องมากกว่า 0");
        }
    }

    private function checkNotFrozen(): void {
        if ($this->isFrozen) {
            throw new \RuntimeException("บัญชีถูกอายัด ไม่สามารถทำรายการได้");
        }
    }

    private function recordTransaction(string $type, float $amount, string $desc): void {
        $this->transactions[] = new Transaction($type, $amount, $this->balance, $desc);
    }
}

// ======= ทดสอบ =======

$alice = new BankAccount('Alice', 10000.00);
$bob   = new BankAccount('Bob', 5000.00);

// ฝาก
$alice->deposit(2000, 'รับเงินเดือน');

// ถอน
$alice->withdraw(500, 'ค่าอาหาร');

// โอน (method chaining)
$alice->transfer($bob, 3000, 'ชำระหนี้');

// ดอกเบี้ย
$alice->applyInterest();

// แสดงใบแจ้งยอด
$alice->printStatement();
$bob->printStatement();

// ทดสอบ exception
try {
    $alice->withdraw(999999);
} catch (InsufficientFundsException $e) {
    echo "❌ Error: " . $e->getMessage() . "\n";
}

// ทดสอบอายัด
$alice->freeze();
try {
    $alice->deposit(100);
} catch (\RuntimeException $e) {
    echo "❌ Error: " . $e->getMessage() . "\n";
}
```

---

## ❓ Quiz

### ข้อที่ 1
ข้อใดคือความแตกต่างหลักระหว่าง `self::` และ `static::`?

- A. `self::` ใช้ใน static method เท่านั้น
- B. `static::` รองรับ Late Static Binding แต่ `self::` อ้างถึง class ที่ประกาศ method
- C. ทั้งสองใช้แทนกันได้เสมอ
- D. `self::` ใช้กับ property และ `static::` ใช้กับ method

**เฉลย: B**
`self::` อ้างถึง class ที่ **ประกาศ** method นั้น (compile-time) ส่วน `static::` อ้างถึง class ที่ **ถูกเรียก** จริงๆ (runtime) ซึ่งสำคัญมากใน inheritance

---

### ข้อที่ 2
Magic method ใดถูกเรียกเมื่อพยายามเรียก method ที่ไม่มีอยู่บน instance?

- A. `__get()`
- B. `__invoke()`
- C. `__call()`
- D. `__missing()`

**เฉลย: C**
`__call($name, $args)` ถูกเรียกเมื่อเรียก method ที่ไม่มีอยู่บน instance ส่วน `__callStatic()` ใช้กับ static call

---

### ข้อที่ 3
ผลลัพธ์ของโค้ดต่อไปนี้คืออะไร?

```php
class A {
    private int $x = 10;
    public function getX(): int { return $this->x; }
}
class B extends A {
    private int $x = 20;
}
$b = new B();
echo $b->getX();
```

- A. 10
- B. 20
- C. Error
- D. null

**เฉลย: A**
`getX()` ถูกประกาศใน class A และ `$this->x` ใน method นั้นอ้างถึง private `$x` ของ class A (= 10) ไม่ใช่ `$x` ของ class B เพราะ private ไม่ถูก override

---

### ข้อที่ 4
`__destruct()` ถูกเรียกในสถานการณ์ใดบ้าง?

- A. เมื่อเรียก `delete $obj`
- B. เมื่อตัวแปรหมด scope หรือเรียก `unset($obj)` หรือเมื่อ script จบ
- C. เฉพาะเมื่อเรียก `unset($obj)` เท่านั้น
- D. ต้องเรียกเอง เช่น `$obj->__destruct()`

**เฉลย: B**
PHP จะเรียก `__destruct()` เมื่อ reference count ของ object เป็น 0 ซึ่งเกิดได้จาก: หมด scope, `unset()`, หรือ script สิ้นสุด

---

### ข้อที่ 5
Shallow clone ต่างจาก Deep clone อย่างไร?

- A. Shallow clone คัดลอก method แต่ Deep clone คัดลอก property
- B. Shallow clone คัดลอก primitive values แต่ object properties ยัง share reference กัน ส่วน Deep clone คัดลอกทุกอย่างรวมถึง nested objects
- C. ทั้งสองเหมือนกันใน PHP
- D. Deep clone ใช้ได้เฉพาะกับ final class

**เฉลย: B**
`clone` ใน PHP ทำ shallow clone โดย default: primitive properties ถูกคัดลอก แต่ object properties ยัง point ไปที่ object เดิม ต้องใช้ `__clone()` เพื่อทำ deep clone ด้วยตัวเอง

---

## 🔗 ไปต่อ

➡️ **[Part 15: PHP OOP Inheritance - การสืบทอดและ Polymorphism](part-015-php-oop-inheritance.md)**

เรียนรู้เรื่อง Inheritance, Abstract Classes, Interfaces, และ Traits ที่ทำให้โค้ดยืดหยุ่นและ reuse ได้มากขึ้น
