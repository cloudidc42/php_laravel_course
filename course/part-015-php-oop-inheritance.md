# 🧬 Part 15: PHP OOP Inheritance - การสืบทอด Polymorphism & Interfaces

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ใช้ Inheritance (extends) และ Method Overriding ได้
- สร้าง Abstract Classes & Interfaces ได้
- ใช้ Traits รวมถึงแก้ conflict ระหว่าง Traits ได้
- กำหนด Final classes & methods ได้
- เข้าใจ Covariant return types ได้
- สร้าง Animal hierarchy และ Payment system ได้

---

## 📌 1. Inheritance (extends)

### 1.1 การสืบทอด Class พื้นฐาน

```php
<?php
class Animal {
    protected string $name;
    protected int $age;

    public function __construct(string $name, int $age) {
        $this->name = $name;
        $this->age  = $age;
    }

    public function breathe(): string {
        return "{$this->name} กำลังหายใจ";
    }

    public function getInfo(): string {
        return "{$this->name} อายุ {$this->age} ปี";
    }

    public function makeSound(): string {
        return "...";
    }
}

class Dog extends Animal {
    private string $breed;

    public function __construct(string $name, int $age, string $breed) {
        parent::__construct($name, $age); // เรียก constructor ของ parent
        $this->breed = $breed;
    }

    // Override method
    public function makeSound(): string {
        return "{$this->name} พูดว่า: โฮ่ง โฮ่ง!";
    }

    public function fetch(): string {
        return "{$this->name} วิ่งไปเอาของ!";
    }

    public function getInfo(): string {
        return parent::getInfo() . " พันธุ์: {$this->breed}";
    }
}

class Cat extends Animal {
    public function makeSound(): string {
        return "{$this->name} พูดว่า: เมี้ยว~";
    }

    public function purr(): string {
        return "{$this->name} กำลังครางเพลิน...";
    }
}

$dog = new Dog('บักหมา', 3, 'Golden Retriever');
$cat = new Cat('มิ้ว', 2);

echo $dog->makeSound(); // บักหมา พูดว่า: โฮ่ง โฮ่ง!
echo $dog->breathe();   // บักหมา กำลังหายใจ (inherited)
echo $dog->getInfo();   // บักหมา อายุ 3 ปี พันธุ์: Golden Retriever
echo $cat->makeSound(); // มิ้ว พูดว่า: เมี้ยว~
```

### 1.2 Polymorphism

```php
<?php
// Polymorphism: object ต่างๆ ตอบสนองต่อ interface เดิมต่างกัน
$animals = [
    new Dog('หมา1', 2, 'Labrador'),
    new Cat('แมว1', 1),
    new Dog('หมา2', 4, 'Poodle'),
    new Cat('แมว2', 3),
];

foreach ($animals as $animal) {
    // เรียก makeSound() เหมือนกัน แต่พฤติกรรมต่างกัน
    echo $animal->makeSound() . "\n";
}
```

### 1.3 instanceof operator

```php
<?php
$dog = new Dog('เจ้าบุญ', 2, 'Thai Ridgeback');

var_dump($dog instanceof Dog);    // true
var_dump($dog instanceof Animal); // true (parent class)
var_dump($dog instanceof Cat);    // false

// Type checking ก่อนใช้ method เฉพาะ
foreach ($animals as $animal) {
    if ($animal instanceof Dog) {
        echo $animal->fetch() . "\n";
    }
}
```

---

## 📌 2. Method Overriding & parent::

### 2.1 ใช้ parent:: เรียก method ของ parent

```php
<?php
class Vehicle {
    protected float $speed = 0;
    protected float $fuel;

    public function __construct(float $initialFuel = 100.0) {
        $this->fuel = $initialFuel;
    }

    public function accelerate(float $amount): void {
        $this->speed += $amount;
        $this->fuel  -= $amount * 0.1;
        echo "เร็วขึ้น {$amount} km/h | ความเร็ว: {$this->speed} km/h\n";
    }

    public function brake(float $amount): void {
        $this->speed = max(0, $this->speed - $amount);
        echo "เบรก | ความเร็ว: {$this->speed} km/h\n";
    }

    public function getStatus(): string {
        return "ความเร็ว: {$this->speed} km/h | เชื้อเพลิง: {$this->fuel}%";
    }
}

class ElectricCar extends Vehicle {
    private float $battery;

    public function __construct(float $initialFuel = 0.0, float $battery = 100.0) {
        parent::__construct($initialFuel);
        $this->battery = $battery;
    }

    // Override: electric car ใช้ battery แทน fuel
    public function accelerate(float $amount): void {
        $this->speed   += $amount;
        $this->battery -= $amount * 0.05; // ประหยัดกว่า
        echo "[EV] เร็วขึ้น {$amount} km/h | ความเร็ว: {$this->speed} km/h\n";
    }

    // Extend parent method
    public function getStatus(): string {
        return parent::getStatus() . " | Battery: {$this->battery}%";
    }

    public function chargeBattery(float $amount): void {
        $this->battery = min(100, $this->battery + $amount);
        echo "ชาร์จแบตเตอรี่ | Battery: {$this->battery}%\n";
    }
}

$ev = new ElectricCar();
$ev->accelerate(60);
$ev->accelerate(40);
$ev->brake(30);
echo $ev->getStatus() . "\n";
```

---

## 📌 3. Abstract Classes & Abstract Methods

### 3.1 Abstract class

Abstract class ไม่สามารถสร้าง instance ได้โดยตรง และ abstract method ต้อง implement ใน subclass

```php
<?php
abstract class Shape {
    protected string $color;

    public function __construct(string $color = 'ดำ') {
        $this->color = $color;
    }

    // Abstract method: subclass ต้อง implement
    abstract public function area(): float;
    abstract public function perimeter(): float;

    // Concrete method: ทุก subclass ใช้ร่วมกัน
    public function describe(): string {
        return sprintf(
            "%s สี%s | พื้นที่: %.2f | เส้นรอบรูป: %.2f",
            static::class,
            $this->color,
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
        private float $radius,
        string $color = 'แดง'
    ) {
        parent::__construct($color);
    }

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
        private float $height,
        string $color = 'น้ำเงิน'
    ) {
        parent::__construct($color);
    }

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
        private float $c,
        string $color = 'เขียว'
    ) {
        parent::__construct($color);
    }

    public function area(): float {
        $s = $this->perimeter() / 2; // semi-perimeter
        return sqrt($s * ($s - $this->a) * ($s - $this->b) * ($s - $this->c));
    }

    public function perimeter(): float {
        return $this->a + $this->b + $this->c;
    }
}

$shapes = [
    new Circle(5),
    new Rectangle(4, 6),
    new Triangle(3, 4, 5),
];

foreach ($shapes as $shape) {
    echo $shape->describe() . "\n";
}

// เปรียบเทียบขนาด
echo $shapes[0]->isLargerThan($shapes[1]) ? "วงกลมใหญ่กว่า" : "สี่เหลี่ยมใหญ่กว่า";
```

---

## 📌 4. Interfaces

### 4.1 Interface พื้นฐาน

```php
<?php
interface Printable {
    public function print(): void;
    public function toPDF(): string;
}

interface Exportable {
    public function exportToCSV(): string;
    public function exportToJSON(): string;
}

interface Serializable {
    public function serialize(): string;
    public static function deserialize(string $data): static;
}
```

### 4.2 Implement หลาย Interfaces

```php
<?php
class Invoice implements Printable, Exportable {
    private array $items = [];

    public function __construct(
        private int $invoiceNumber,
        private string $customerName,
        private \DateTimeImmutable $date
    ) {}

    public function addItem(string $name, float $price, int $qty): static {
        $this->items[] = compact('name', 'price', 'qty');
        return $this;
    }

    public function getTotal(): float {
        return array_sum(array_map(
            fn($item) => $item['price'] * $item['qty'],
            $this->items
        ));
    }

    // Implement Printable
    public function print(): void {
        echo "═══ ใบแจ้งหนี้ #{$this->invoiceNumber} ═══\n";
        echo "ลูกค้า: {$this->customerName}\n";
        echo "วันที่: " . $this->date->format('d/m/Y') . "\n";
        foreach ($this->items as $item) {
            printf("  %-20s %d x %.2f = %.2f\n",
                $item['name'], $item['qty'], $item['price'],
                $item['price'] * $item['qty']
            );
        }
        printf("ยอดรวม: %.2f THB\n", $this->getTotal());
    }

    public function toPDF(): string {
        return "PDF_CONTENT_FOR_INVOICE_{$this->invoiceNumber}";
    }

    // Implement Exportable
    public function exportToCSV(): string {
        $lines = ["Name,Price,Qty,Total"];
        foreach ($this->items as $item) {
            $lines[] = implode(',', [
                $item['name'],
                $item['price'],
                $item['qty'],
                $item['price'] * $item['qty'],
            ]);
        }
        return implode("\n", $lines);
    }

    public function exportToJSON(): string {
        return json_encode([
            'invoice_number' => $this->invoiceNumber,
            'customer'       => $this->customerName,
            'items'          => $this->items,
            'total'          => $this->getTotal(),
        ], JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT);
    }
}

$invoice = new Invoice(1001, 'บริษัท ทดสอบ จำกัด', new \DateTimeImmutable());
$invoice
    ->addItem('สินค้า A', 250.00, 3)
    ->addItem('สินค้า B', 1200.00, 1)
    ->addItem('สินค้า C', 85.50, 5);

$invoice->print();
echo "\nJSON Export:\n" . $invoice->exportToJSON();
```

### 4.3 Interface Inheritance

```php
<?php
interface Readable {
    public function read(): string;
}

interface Writable {
    public function write(string $data): void;
}

// Interface สืบทอด Interfaces อื่น
interface ReadWritable extends Readable, Writable {
    public function seek(int $position): void;
}

class FileStream implements ReadWritable {
    private int $position = 0;
    private string $buffer = '';

    public function read(): string {
        return substr($this->buffer, $this->position);
    }

    public function write(string $data): void {
        $this->buffer .= $data;
    }

    public function seek(int $position): void {
        $this->position = $position;
    }
}
```

---

## 📌 5. Traits

### 5.1 Trait พื้นฐาน

```php
<?php
trait Timestampable {
    private ?\DateTimeImmutable $createdAt = null;
    private ?\DateTimeImmutable $updatedAt = null;

    public function setCreatedAt(): void {
        $this->createdAt = new \DateTimeImmutable();
    }

    public function setUpdatedAt(): void {
        $this->updatedAt = new \DateTimeImmutable();
    }

    public function getCreatedAt(): ?\DateTimeImmutable { return $this->createdAt; }
    public function getUpdatedAt(): ?\DateTimeImmutable { return $this->updatedAt; }
}

trait SoftDeletable {
    private ?\DateTimeImmutable $deletedAt = null;

    public function softDelete(): void {
        $this->deletedAt = new \DateTimeImmutable();
    }

    public function restore(): void {
        $this->deletedAt = null;
    }

    public function isDeleted(): bool {
        return $this->deletedAt !== null;
    }

    public function getDeletedAt(): ?\DateTimeImmutable {
        return $this->deletedAt;
    }
}

trait Loggable {
    private array $logs = [];

    public function log(string $action, array $context = []): void {
        $this->logs[] = [
            'action'    => $action,
            'context'   => $context,
            'timestamp' => new \DateTimeImmutable(),
        ];
    }

    public function getLogs(): array {
        return $this->logs;
    }
}
```

### 5.2 ใช้หลาย Traits

```php
<?php
class Post {
    use Timestampable, SoftDeletable, Loggable;

    public function __construct(
        private string $title,
        private string $content
    ) {
        $this->setCreatedAt();
        $this->log('created', ['title' => $title]);
    }

    public function update(string $title, string $content): void {
        $this->title   = $title;
        $this->content = $content;
        $this->setUpdatedAt();
        $this->log('updated', ['title' => $title]);
    }

    public function delete(): void {
        $this->softDelete();
        $this->log('deleted');
    }
}

$post = new Post('สวัสดี PHP OOP', 'เนื้อหาบทความ...');
$post->update('สวัสดี PHP OOP - แก้ไข', 'เนื้อหาใหม่...');
$post->delete();

echo "ลบแล้ว: " . ($post->isDeleted() ? 'ใช่' : 'ไม่') . "\n"; // ใช่
echo "Logs: " . count($post->getLogs()) . " รายการ\n";           // 3
```

### 5.3 Trait Conflict Resolution

```php
<?php
trait A {
    public function hello(): string {
        return "Hello from A";
    }

    public function greet(): string {
        return "A greet";
    }
}

trait B {
    public function hello(): string {
        return "Hello from B";
    }

    public function greet(): string {
        return "B greet";
    }
}

class MyClass {
    use A, B {
        A::hello insteadof B; // ใช้ hello จาก A แทน B
        B::hello as helloFromB; // แต่ยัง access hello ของ B ได้ด้วยชื่อใหม่
        B::greet insteadof A;   // ใช้ greet จาก B แทน A
        A::greet as greetFromA; // alias
    }
}

$obj = new MyClass();
echo $obj->hello();      // Hello from A
echo $obj->helloFromB(); // Hello from B
echo $obj->greet();      // B greet
echo $obj->greetFromA(); // A greet
```

### 5.4 Abstract Method ใน Trait

```php
<?php
trait Validatable {
    // abstract method ใน trait: class ที่ use ต้อง implement
    abstract protected function rules(): array;

    public function validate(array $data): array {
        $errors = [];
        foreach ($this->rules() as $field => $rule) {
            if ($rule === 'required' && empty($data[$field])) {
                $errors[$field] = "{$field} จำเป็นต้องกรอก";
            }
        }
        return $errors;
    }
}

class RegistrationForm {
    use Validatable;

    protected function rules(): array {
        return [
            'name'     => 'required',
            'email'    => 'required',
            'password' => 'required',
        ];
    }
}

$form   = new RegistrationForm();
$errors = $form->validate(['name' => 'สมชาย', 'email' => '']);
print_r($errors);
```

---

## 📌 6. Final Classes & Methods

```php
<?php
class Base {
    public function normalMethod(): string {
        return "override ได้";
    }

    // final method: ห้าม subclass override
    final public function secureMethod(): string {
        return "ห้าม override!";
    }
}

class Child extends Base {
    public function normalMethod(): string {
        return "overridden!"; // ✅ ทำได้
    }

    // public function secureMethod(): string {} // ❌ Fatal error
}

// final class: ห้าม extend
final class Singleton {
    private static ?self $instance = null;

    private function __construct() {}

    public static function getInstance(): static {
        return self::$instance ??= new self();
    }
}

// class ExtendSingleton extends Singleton {} // ❌ Fatal error
```

---

## 📌 7. Covariant Return Types

```php
<?php
class Collection {
    protected array $items = [];

    public function add(mixed $item): static { // static = Covariant
        $this->items[] = $item;
        return $this;
    }

    public function all(): array {
        return $this->items;
    }
}

class NumberCollection extends Collection {
    public function sum(): float {
        return array_sum($this->items);
    }

    public function average(): float {
        return count($this->items) > 0
            ? $this->sum() / count($this->items)
            : 0.0;
    }
}

$nums = new NumberCollection();
$nums->add(10)->add(20)->add(30); // return type is NumberCollection, not Collection

echo $nums->sum();     // 60
echo $nums->average(); // 20
```

---

## 🛠️ Workshop Part A: Animal Hierarchy

```php
<?php
declare(strict_types=1);

interface Domesticated {
    public function trainCommand(string $command): void;
    public function respondToCommand(string $command): string;
}

interface WildAnimal {
    public function hunt(): string;
    public function territory(): float; // km²
}

abstract class Animal {
    protected array $attributes = [];

    public function __construct(
        protected string $name,
        protected string $species,
        protected int $age,
        protected float $weight // kg
    ) {}

    abstract public function makeSound(): string;
    abstract public function move(): string;

    public function eat(string $food): string {
        return "{$this->name} กำลังกิน {$food}";
    }

    public function sleep(): string {
        return "{$this->name} กำลังนอนหลับ";
    }

    public function __toString(): string {
        return "[{$this->species}] {$this->name} ({$this->age} ปี, {$this->weight} kg)";
    }
}

class Dog extends Animal implements Domesticated {
    private array $learnedCommands = [];

    public function __construct(
        string $name,
        int $age,
        float $weight,
        private string $breed
    ) {
        parent::__construct($name, 'Canis lupus familiaris', $age, $weight);
    }

    public function makeSound(): string { return "โฮ่ง! โฮ่ง!"; }
    public function move(): string { return "{$this->name} วิ่งด้วยสี่ขา"; }

    public function trainCommand(string $command): void {
        $this->learnedCommands[] = $command;
        echo "{$this->name} เรียนรู้คำสั่ง: {$command}\n";
    }

    public function respondToCommand(string $command): string {
        if (in_array($command, $this->learnedCommands)) {
            return "{$this->name} ทำตามคำสั่ง '{$command}'!";
        }
        return "{$this->name} งงกับคำสั่ง '{$command}'";
    }
}

class Lion extends Animal implements WildAnimal {
    public function __construct(string $name, int $age, float $weight) {
        parent::__construct($name, 'Panthera leo', $age, $weight);
    }

    public function makeSound(): string { return "ROARRR!!!"; }
    public function move(): string { return "{$this->name} เดินย่องเงียบ"; }

    public function hunt(): string {
        return "{$this->name} ไล่ล่าเหยื่อ!";
    }

    public function territory(): float {
        return 250.0; // 250 km²
    }
}

// ทดสอบ
$dog = new Dog('บักหมา', 3, 15.0, 'German Shepherd');
$dog->trainCommand('นั่ง');
$dog->trainCommand('หมอบ');
echo $dog->respondToCommand('นั่ง') . "\n";
echo $dog->makeSound() . "\n";
echo $dog . "\n";

$lion = new Lion('ราชา', 5, 190.0);
echo $lion->hunt() . "\n";
echo $lion->makeSound() . "\n";
echo "อาณาเขต: " . $lion->territory() . " km²\n";
```

## 🛠️ Workshop Part B: Payment System

```php
<?php
declare(strict_types=1);

interface PaymentGateway {
    public function charge(float $amount, string $currency = 'THB'): PaymentResult;
    public function refund(string $transactionId, float $amount): RefundResult;
    public function supports(string $currency): bool;
}

interface Installable {
    public function chargeWithInstallment(float $amount, int $months): PaymentResult;
    public function getInstallmentRate(int $months): float;
}

class PaymentResult {
    public function __construct(
        public readonly bool $success,
        public readonly string $transactionId,
        public readonly float $amount,
        public readonly string $message,
        public readonly array $metadata = []
    ) {}

    public function __toString(): string {
        $status = $this->success ? '✅ สำเร็จ' : '❌ ล้มเหลว';
        return "{$status} | TXN: {$this->transactionId} | {$this->amount} THB | {$this->message}";
    }
}

class RefundResult {
    public function __construct(
        public readonly bool $success,
        public readonly float $refundedAmount,
        public readonly string $message
    ) {}
}

abstract class BasePayment implements PaymentGateway {
    protected array $transactionLog = [];

    protected function generateTransactionId(string $prefix): string {
        return $prefix . '_' . strtoupper(bin2hex(random_bytes(8)));
    }

    protected function logTransaction(PaymentResult $result): void {
        $this->transactionLog[] = [
            'result'    => $result,
            'timestamp' => new \DateTimeImmutable(),
        ];
    }

    public function getTransactionLog(): array {
        return $this->transactionLog;
    }

    public function supports(string $currency): bool {
        return in_array(strtoupper($currency), $this->supportedCurrencies());
    }

    abstract protected function supportedCurrencies(): array;
}

class CreditCardPayment extends BasePayment implements Installable {
    private array $installmentRates = [
        3  => 0.0,    // 0% 3 เดือน
        6  => 0.5,    // 0.5% ต่อเดือน
        10 => 0.64,   // 0.64% ต่อเดือน
        12 => 0.64,
    ];

    public function __construct(
        private string $cardNumber,  // masked
        private string $cardHolderName,
        private string $bankName
    ) {}

    public function charge(float $amount, string $currency = 'THB'): PaymentResult {
        if (!$this->supports($currency)) {
            return new PaymentResult(false, '', $amount, "ไม่รองรับสกุลเงิน {$currency}");
        }

        // จำลองการชำระเงิน
        $txnId  = $this->generateTransactionId('CC');
        $result = new PaymentResult(
            success:       true,
            transactionId: $txnId,
            amount:        $amount,
            message:       "ชำระด้วยบัตร {$this->bankName} ({$this->cardNumber})",
            metadata:      ['cardholder' => $this->cardHolderName]
        );

        $this->logTransaction($result);
        return $result;
    }

    public function refund(string $transactionId, float $amount): RefundResult {
        return new RefundResult(true, $amount, "คืนเงิน {$amount} THB สู่บัตรเครดิต");
    }

    public function chargeWithInstallment(float $amount, int $months): PaymentResult {
        $rate    = $this->getInstallmentRate($months);
        $monthly = ($amount * (1 + $rate * $months / 100)) / $months;

        $txnId  = $this->generateTransactionId('CC_INS');
        $result = new PaymentResult(
            success:       true,
            transactionId: $txnId,
            amount:        $amount,
            message:       "ผ่อน {$months} เดือน | เดือนละ " . number_format($monthly, 2) . " THB",
            metadata:      ['months' => $months, 'monthly_payment' => $monthly]
        );

        $this->logTransaction($result);
        return $result;
    }

    public function getInstallmentRate(int $months): float {
        return $this->installmentRates[$months] ?? 0.8;
    }

    protected function supportedCurrencies(): array {
        return ['THB', 'USD', 'EUR', 'JPY', 'CNY'];
    }
}

class BankTransferPayment extends BasePayment {
    public function __construct(
        private string $bankCode,
        private string $accountName,
        private string $accountNumber
    ) {}

    public function charge(float $amount, string $currency = 'THB'): PaymentResult {
        if (!$this->supports($currency)) {
            return new PaymentResult(false, '', $amount, "ไม่รองรับสกุลเงิน {$currency}");
        }

        $txnId  = $this->generateTransactionId('BT');
        $result = new PaymentResult(
            success:       true,
            transactionId: $txnId,
            amount:        $amount,
            message:       "โอนเงินไปยัง {$this->bankCode} เลขบัญชี {$this->accountNumber}",
            metadata:      ['bank' => $this->bankCode, 'account_name' => $this->accountName]
        );

        $this->logTransaction($result);
        return $result;
    }

    public function refund(string $transactionId, float $amount): RefundResult {
        return new RefundResult(true, $amount, "คืนเงินผ่านการโอนธนาคาร ใช้เวลา 1-3 วันทำการ");
    }

    protected function supportedCurrencies(): array {
        return ['THB'];
    }
}

class PromptPayPayment extends BasePayment {
    public function __construct(
        private string $promptPayId, // เบอร์โทร, เลขบัตร, หรือ tax ID
        private string $displayName
    ) {}

    public function charge(float $amount, string $currency = 'THB'): PaymentResult {
        if (!$this->supports($currency)) {
            return new PaymentResult(false, '', $amount, "PromptPay รองรับเฉพาะ THB");
        }

        if ($amount > 300000) {
            return new PaymentResult(
                false, '', $amount,
                "ยอดชำระเกิน 300,000 THB สำหรับ PromptPay"
            );
        }

        $txnId  = $this->generateTransactionId('PP');
        $result = new PaymentResult(
            success:       true,
            transactionId: $txnId,
            amount:        $amount,
            message:       "PromptPay: {$this->promptPayId} ({$this->displayName})",
            metadata:      ['qr_code' => "QR_DATA_{$txnId}"]
        );

        $this->logTransaction($result);
        return $result;
    }

    public function refund(string $transactionId, float $amount): RefundResult {
        return new RefundResult(true, $amount, "คืนเงินผ่าน PromptPay ทันที");
    }

    protected function supportedCurrencies(): array {
        return ['THB'];
    }
}

// ======= Payment Processor =======
class PaymentProcessor {
    public function __construct(
        private PaymentGateway $gateway
    ) {}

    public function processPayment(float $amount): void {
        echo "กำลังประมวลผล: " . number_format($amount, 2) . " THB\n";
        $result = $this->gateway->charge($amount);
        echo $result . "\n";

        if (!$result->success) {
            throw new \RuntimeException("การชำระเงินล้มเหลว: " . $result->message);
        }
    }
}

// ======= ทดสอบ =======
echo "=== ทดสอบระบบชำระเงิน ===\n\n";

$creditCard  = new CreditCardPayment('**** **** **** 1234', 'สมชาย ใจดี', 'กสิกรไทย');
$bankTransfer = new BankTransferPayment('SCB', 'บริษัท ทดสอบ', '123-456-789-0');
$promptPay   = new PromptPayPayment('0812345678', 'สมชาย ใจดี');

// ชำระปกติ
echo "--- Credit Card ---\n";
echo $creditCard->charge(1500.00) . "\n";

echo "\n--- ผ่อนชำระ ---\n";
echo $creditCard->chargeWithInstallment(12000, 6) . "\n";

echo "\n--- Bank Transfer ---\n";
echo $bankTransfer->charge(50000) . "\n";

echo "\n--- PromptPay ---\n";
echo $promptPay->charge(350) . "\n";
echo $promptPay->charge(400000) . "\n"; // เกิน limit

// ใช้ PaymentProcessor
echo "\n--- PaymentProcessor ---\n";
$processor = new PaymentProcessor($promptPay);
$processor->processPayment(999);
```

---

## ❓ Quiz

### ข้อที่ 1
Abstract class ต่างจาก Interface อย่างไร?

- A. Abstract class สามารถมี concrete methods ได้ แต่ Interface ไม่ได้
- B. ทั้งสองเหมือนกันทุกประการ
- C. Interface รองรับ multiple inheritance แต่ Abstract class ไม่รองรับ
- D. ทั้ง A และ C ถูก

**เฉลย: D**
Abstract class สามารถมี concrete methods, constructor, properties ที่มี visibility ได้ แต่ Interface (ก่อน PHP 8) มีเฉพาะ method signatures และ constants. Interface รองรับ multiple implementation แต่ class extends ได้แค่ class เดียว

---

### ข้อที่ 2
เมื่อ Trait สองตัวมี method ชื่อเดียวกัน วิธีแก้ conflict คืออะไร?

- A. PHP จะใช้ Trait ที่ประกาศก่อนเสมอ
- B. ใช้ `insteadof` เพื่อเลือก Trait ที่ต้องการ และ `as` เพื่อสร้าง alias
- C. ต้อง rename method ใน Trait ก่อน
- D. PHP จะ throw error เสมอ

**เฉลย: B**
ใช้ syntax `TraitA::method insteadof TraitB` เพื่อเลือก และ `TraitB::method as newName` เพื่อสร้าง alias ให้ยังเข้าถึงได้

---

### ข้อที่ 3
`parent::__construct()` ใน subclass จำเป็นต้องเรียกเสมอหรือไม่?

- A. ใช่ PHP บังคับ
- B. ไม่ แต่ถ้า parent constructor ทำงานสำคัญ ควรเรียก
- C. ใช่ ถ้า parent เป็น abstract class
- D. ไม่ต้องเรียกเลย เพราะ PHP เรียกให้อัตโนมัติ

**เฉลย: B**
PHP ไม่บังคับ แต่ถ้าไม่เรียก initialization ใน parent constructor จะไม่ทำงาน ทำให้ object ของ parent อาจอยู่ในสถานะไม่ถูกต้อง

---

### ข้อที่ 4
Final class แตกต่างจาก class ปกติอย่างไร?

- A. Final class สร้าง instance ไม่ได้
- B. Final class ไม่สามารถถูก extend โดย class อื่น
- C. Final class ไม่มี method
- D. Final class implement interface ไม่ได้

**เฉลย: B**
`final class` ป้องกันการ extend แต่ยังสร้าง instance และ implement interface ได้ตามปกติ ใช้เพื่อป้องกันการ override พฤติกรรมสำคัญ เช่น Singleton

---

### ข้อที่ 5
Covariant return type คืออะไรและใช้ `static` อย่างไร?

- A. Return type ที่ strict มากขึ้นใน subclass และ `static` หมายถึง class ที่ถูกเรียก
- B. Return type ที่กว้างขึ้นใน subclass
- C. ใช้ได้เฉพาะกับ abstract method
- D. `static` ใน return type หมายถึง static method เท่านั้น

**เฉลย: A**
Covariant return types ช่วยให้ subclass return type ที่ specific มากขึ้นได้ ในขณะที่ `static` เป็น special return type ที่บอกว่า method คืน instance ของ class ที่ถูกเรียก (ไม่ใช่ class ที่ประกาศ)

---

## 🔗 ไปต่อ

➡️ **[Part 16: PHP OOP Advanced - Design Patterns & Best Practices](part-016-php-oop-advanced.md)**

เรียนรู้ Design Patterns (Singleton, Factory, Observer, Strategy) และหลักการ SOLID ที่ทำให้โค้ด PHP ของคุณมีคุณภาพระดับ Production
