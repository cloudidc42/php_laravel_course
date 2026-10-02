# Part 023: PHP Testing ด้วย PHPUnit

## ระดับ: Intermediate to Advanced
## เวลาเรียน: 4-5 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:
- ตั้งค่าและใช้งาน PHPUnit
- เขียน Unit Tests และ Integration Tests
- ใช้ Data Providers
- Mock dependencies ด้วย Mockery
- วัด Code Coverage
- สร้าง Test ครบวงจรสำหรับ Shopping Cart

---

## 1. PHPUnit Setup

### ติดตั้ง PHPUnit

```bash
composer require --dev phpunit/phpunit:^11.0

# ตรวจสอบ version
./vendor/bin/phpunit --version
```

### phpunit.xml Configuration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         cacheDirectory=".phpunit.cache"
         colors="true"
         stopOnFailure="false"
         displayDetailsOnTestsThatTriggerDeprecations="true"
         displayDetailsOnTestsThatTriggerErrors="true"
         displayDetailsOnTestsThatTriggerNotices="true"
         displayDetailsOnTestsThatTriggerWarnings="true">

    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="Integration">
            <directory>tests/Integration</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory>tests/Feature</directory>
        </testsuite>
    </testsuites>

    <source>
        <include>
            <directory suffix=".php">src</directory>
        </include>
        <exclude>
            <directory>src/Legacy</directory>
        </exclude>
    </source>

    <coverage>
        <report>
            <html outputDirectory="coverage/html"/>
            <text outputFile="coverage/coverage.txt"/>
            <clover outputFile="coverage/coverage.xml"/>
        </report>
    </coverage>

    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="DB_DATABASE" value=":memory:"/>
        <ini name="error_reporting" value="-1"/>
    </php>
</phpunit>
```

### โครงสร้าง Test Directory

```
tests/
├── Unit/
│   ├── Models/
│   │   └── UserTest.php
│   ├── Services/
│   │   └── PaymentServiceTest.php
│   └── Helpers/
│       └── StringHelperTest.php
├── Integration/
│   ├── Database/
│   │   └── UserRepositoryTest.php
│   └── API/
│       └── UserApiTest.php
├── Feature/
│   └── CheckoutTest.php
└── TestCase.php (base class)
```

---

## 2. Unit Tests พื้นฐาน

### Test Case พื้นฐาน

```php
<?php
namespace Tests\Unit;

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\Attributes\Depends;

class CalculatorTest extends TestCase {
    
    private Calculator $calc;
    
    // ทำงานก่อนแต่ละ test method
    protected function setUp(): void {
        parent::setUp();
        $this->calc = new Calculator();
    }
    
    // ทำงานหลังแต่ละ test method
    protected function tearDown(): void {
        parent::tearDown();
        // cleanup ถ้าจำเป็น
    }
    
    // ทำงานครั้งเดียวก่อน test ทั้งหมดใน class
    public static function setUpBeforeClass(): void {
        parent::setUpBeforeClass();
        echo "Starting Calculator tests...\n";
    }
    
    // Test method ต้องขึ้นต้นด้วย test หรือใส่ #[Test]
    public function testAdd(): void {
        $result = $this->calc->add(2, 3);
        $this->assertEquals(5, $result);
    }
    
    #[Test]
    public function subtract(): void {
        $this->assertEquals(3, $this->calc->subtract(10, 7));
    }
    
    public function testDivide(): void {
        $this->assertEquals(4.0, $this->calc->divide(20, 5));
    }
    
    public function testDivideByZeroThrows(): void {
        $this->expectException(\DivisionByZeroError::class);
        $this->expectExceptionMessage('Division by zero');
        
        $this->calc->divide(10, 0);
    }
    
    // Test ที่ depends กัน
    public function testCreateUser(): int {
        $userId = $this->calc->someMethod();
        $this->assertGreaterThan(0, $userId);
        return $userId; // ส่งค่าไปให้ test ถัดไป
    }
    
    #[Depends('testCreateUser')]
    public function testUpdateUser(int $userId): void {
        // userId มาจาก testCreateUser
        $this->assertIsInt($userId);
    }
}

class Calculator {
    public function add(float $a, float $b): float {
        return $a + $b;
    }
    
    public function subtract(float $a, float $b): float {
        return $a - $b;
    }
    
    public function multiply(float $a, float $b): float {
        return $a * $b;
    }
    
    public function divide(float $a, float $b): float {
        if ($b == 0) throw new \DivisionByZeroError('Division by zero');
        return $a / $b;
    }
    
    public function someMethod(): int {
        return 42;
    }
}
```

### Assertions ที่สำคัญ

```php
<?php
use PHPUnit\Framework\TestCase;

class AssertionsDemo extends TestCase {
    
    public function testAllAssertions(): void {
        // Basic equality
        $this->assertEquals(5, 2 + 3);           // == (loose)
        $this->assertSame(5, 2 + 3);              // === (strict)
        $this->assertNotEquals(6, 2 + 3);
        $this->assertNotSame('5', 5);             // string '5' !== int 5
        
        // Types
        $this->assertIsInt(42);
        $this->assertIsString("hello");
        $this->assertIsArray([1, 2, 3]);
        $this->assertIsFloat(3.14);
        $this->assertIsBool(true);
        $this->assertIsNull(null);
        $this->assertIsObject(new \stdClass());
        $this->assertInstanceOf(\DateTime::class, new \DateTime());
        
        // Boolean
        $this->assertTrue(1 === 1);
        $this->assertFalse(1 === 2);
        $this->assertNull(null);
        $this->assertNotNull("not null");
        
        // Numbers
        $this->assertGreaterThan(3, 5);
        $this->assertGreaterThanOrEqual(5, 5);
        $this->assertLessThan(10, 5);
        $this->assertLessThanOrEqual(5, 5);
        $this->assertNan(sqrt(-1));
        $this->assertInfinite(INF);
        $this->assertFinite(1.5);
        
        // Strings
        $this->assertStringContainsString('world', 'hello world');
        $this->assertStringNotContainsString('PHP', 'Python');
        $this->assertStringStartsWith('Hello', 'Hello World');
        $this->assertStringEndsWith('World', 'Hello World');
        $this->assertMatchesRegularExpression('/^\d+$/', '12345');
        $this->assertStringEqualsIgnoringCase('HELLO', 'hello');
        
        // Arrays
        $arr = [1, 2, 3, 4, 5];
        $this->assertCount(5, $arr);
        $this->assertContains(3, $arr);
        $this->assertNotContains(6, $arr);
        $this->assertEmpty([]);
        $this->assertNotEmpty($arr);
        $this->assertArrayHasKey('key', ['key' => 'value']);
        $this->assertArrayNotHasKey('missing', ['key' => 'value']);
        
        // Files/Directories
        // $this->assertFileExists('/path/to/file');
        // $this->assertDirectoryExists('/path/to/dir');
        
        // JSON
        $json = '{"name":"PHP","version":"8.2"}';
        $this->assertJson($json);
        $this->assertJsonStringEqualsJsonString($json, '{"version":"8.2","name":"PHP"}');
        
        // Exception
        $this->expectException(\InvalidArgumentException::class);
        throw new \InvalidArgumentException("test");
    }
}
```

---

## 3. Data Providers

```php
<?php
use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\DataProvider;

class DataProviderTest extends TestCase {
    
    // Data Provider: return array of test cases
    public static function additionProvider(): array {
        return [
            'positive numbers'   => [1, 2, 3],
            'negative numbers'   => [-1, -2, -3],
            'mixed'              => [-5, 10, 5],
            'zeros'              => [0, 0, 0],
            'floats'             => [1.5, 2.5, 4.0],
        ];
    }
    
    #[DataProvider('additionProvider')]
    public function testAdd(float $a, float $b, float $expected): void {
        $calc = new Calculator();
        $this->assertEquals($expected, $calc->add($a, $b));
    }
    
    // Data Provider สำหรับ validation
    public static function emailProvider(): \Generator {
        yield 'valid simple' => ['user@example.com', true];
        yield 'valid with dots' => ['user.name@example.co.th', true];
        yield 'valid with plus' => ['user+tag@example.com', true];
        yield 'invalid no @' => ['userexample.com', false];
        yield 'invalid no domain' => ['user@', false];
        yield 'invalid empty' => ['', false];
        yield 'invalid spaces' => ['user @example.com', false];
    }
    
    #[DataProvider('emailProvider')]
    public function testEmailValidation(string $email, bool $expected): void {
        $isValid = (bool) filter_var($email, FILTER_VALIDATE_EMAIL);
        $this->assertEquals($expected, $isValid);
    }
    
    // Data Provider สำหรับ exception testing
    public static function invalidDivisionProvider(): array {
        return [
            'divide by zero' => [10, 0, \DivisionByZeroError::class],
        ];
    }
    
    #[DataProvider('invalidDivisionProvider')]
    public function testDivisionExceptions(
        float $a,
        float $b,
        string $exceptionClass
    ): void {
        $this->expectException($exceptionClass);
        (new Calculator())->divide($a, $b);
    }
    
    // Generator data provider (ประหยัด memory)
    public static function largeDataProvider(): \Generator {
        for ($i = 1; $i <= 100; $i++) {
            yield "test_{$i}" => [$i, $i * 2, $i + $i * 2];
        }
    }
    
    #[DataProvider('largeDataProvider')]
    public function testWithLargeData(int $a, int $b, int $expected): void {
        $this->assertEquals($expected, $a + $b);
    }
}
```

---

## 4. Mocking ด้วย Mockery

```bash
composer require --dev mockery/mockery
```

### PHPUnit Built-in Mock

```php
<?php
use PHPUnit\Framework\TestCase;

class MockTest extends TestCase {
    
    public function testWithBuiltinMock(): void {
        // สร้าง mock object
        $userRepo = $this->createMock(UserRepository::class);
        
        // กำหนด behavior
        $userRepo->method('find')
            ->with(1)
            ->willReturn(['id' => 1, 'name' => 'สมชาย', 'email' => 'somchai@example.com']);
        
        $userRepo->method('findAll')
            ->willReturn([
                ['id' => 1, 'name' => 'สมชาย'],
                ['id' => 2, 'name' => 'สมหญิง'],
            ]);
        
        // ตรวจสอบว่าถูกเรียก
        $userRepo->expects($this->once())
            ->method('find')
            ->with(1);
        
        // ทดสอบ service ที่ใช้ mock
        $service = new UserService($userRepo);
        $user = $service->getUser(1);
        
        $this->assertEquals('สมชาย', $user['name']);
    }
    
    public function testThrowException(): void {
        $repo = $this->createMock(UserRepository::class);
        
        $repo->method('find')
            ->willThrowException(new \RuntimeException("Not found", 404));
        
        $this->expectException(\RuntimeException::class);
        $this->expectExceptionCode(404);
        
        $repo->find(999);
    }
    
    public function testConsecutiveReturns(): void {
        $repo = $this->createMock(UserRepository::class);
        
        $repo->method('find')
            ->willReturnOnConsecutiveCalls(
                ['id' => 1, 'name' => 'First'],
                ['id' => 2, 'name' => 'Second'],
                null,
            );
        
        $this->assertEquals('First', $repo->find(1)['name']);
        $this->assertEquals('Second', $repo->find(2)['name']);
        $this->assertNull($repo->find(3));
    }
}
```

### Mockery (ยืดหยุ่นกว่า)

```php
<?php
use PHPUnit\Framework\TestCase;
use Mockery;
use Mockery\MockInterface;

class MockeryTest extends TestCase {
    
    protected function tearDown(): void {
        parent::tearDown();
        Mockery::close(); // สำคัญ! ต้อง call ทุกครั้ง
    }
    
    public function testMockery(): void {
        // สร้าง mock
        $emailService = Mockery::mock(EmailService::class);
        
        // กำหนด expectation
        $emailService->shouldReceive('send')
            ->once()
            ->with(
                Mockery::on(fn($to) => str_contains($to, '@')), // custom matcher
                Mockery::type('string'),
                Mockery::any()
            )
            ->andReturn(true);
        
        $userService = new UserService(
            new InMemoryUserRepository(),
            $emailService
        );
        
        $userService->register([
            'username' => 'newuser',
            'email' => 'new@example.com',
            'password' => 'Pass123!',
        ]);
    }
    
    public function testSpy(): void {
        // Spy - track calls แต่ไม่ mock behavior
        $logger = Mockery::spy(Logger::class);
        
        // Call ได้โดยไม่ต้อง setup (spy passthrough)
        $service = new UserService(new InMemoryUserRepository(), null, $logger);
        $service->getUser(1);
        
        // ตรวจสอบหลังจากนั้น
        $logger->shouldHaveReceived('info')
            ->once()
            ->with(Mockery::pattern('/User \d+ loaded/'));
    }
    
    public function testPartialMock(): void {
        // Mock บาง methods เท่านั้น
        $service = Mockery::mock(UserService::class)->makePartial();
        
        $service->shouldReceive('sendEmail')
            ->once()
            ->andReturn(true);
        
        // Methods อื่นทำงานจริง
        $result = $service->register(['email' => 'test@example.com']);
        $this->assertTrue($result['success']);
    }
    
    public function testMockInterface(): void {
        // Mock interface ก็ได้
        /** @var UserRepository&MockInterface $repo */
        $repo = Mockery::mock(UserRepository::class);
        
        $repo->shouldReceive('findByEmail')
            ->with('existing@example.com')
            ->andReturn(['id' => 1, 'email' => 'existing@example.com']);
        
        $repo->shouldReceive('findByEmail')
            ->with(Mockery::not('existing@example.com'))
            ->andReturn(null);
        
        $this->assertNotNull($repo->findByEmail('existing@example.com'));
        $this->assertNull($repo->findByEmail('new@example.com'));
    }
}
```

---

## 5. Integration Tests

```php
<?php
namespace Tests\Integration;

use PHPUnit\Framework\TestCase;

class UserRepositoryTest extends TestCase {
    private \PDO $db;
    private UserRepository $repo;
    
    protected function setUp(): void {
        // ใช้ SQLite in-memory สำหรับ tests
        $this->db = new \PDO('sqlite::memory:');
        $this->db->setAttribute(\PDO::ATTR_ERRMODE, \PDO::ERRMODE_EXCEPTION);
        
        // สร้าง schema
        $this->db->exec("
            CREATE TABLE users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                username TEXT UNIQUE NOT NULL,
                email TEXT UNIQUE NOT NULL,
                password_hash TEXT NOT NULL,
                active INTEGER DEFAULT 1,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        ");
        
        $this->repo = new UserRepository($this->db);
    }
    
    public function testCreateAndFind(): void {
        $userId = $this->repo->create([
            'username' => 'testuser',
            'email' => 'test@example.com',
            'password' => 'Pass123!',
        ]);
        
        $this->assertGreaterThan(0, $userId);
        
        $user = $this->repo->find($userId);
        $this->assertNotNull($user);
        $this->assertEquals('testuser', $user['username']);
        $this->assertEquals('test@example.com', $user['email']);
        $this->assertArrayNotHasKey('password_hash', $user); // ไม่ควรเห็น hash
    }
    
    public function testFindByEmail(): void {
        $this->repo->create([
            'username' => 'user1',
            'email' => 'user1@example.com',
            'password' => 'pass',
        ]);
        
        $found = $this->repo->findByEmail('user1@example.com');
        $this->assertNotNull($found);
        $this->assertEquals('user1', $found['username']);
        
        $notFound = $this->repo->findByEmail('notexist@example.com');
        $this->assertNull($notFound);
    }
    
    public function testUniqueConstraint(): void {
        $this->repo->create([
            'username' => 'unique',
            'email' => 'unique@example.com',
            'password' => 'pass',
        ]);
        
        $this->expectException(\RuntimeException::class);
        
        // Duplicate email ต้อง throw
        $this->repo->create([
            'username' => 'other',
            'email' => 'unique@example.com', // duplicate!
            'password' => 'pass',
        ]);
    }
    
    public function testPagination(): void {
        // สร้าง users 15 คน
        for ($i = 1; $i <= 15; $i++) {
            $this->repo->create([
                'username' => "user{$i}",
                'email' => "user{$i}@example.com",
                'password' => 'pass',
            ]);
        }
        
        // หน้า 1: 10 รายการ
        [$users, $total] = $this->repo->paginate(1, 10);
        $this->assertCount(10, $users);
        $this->assertEquals(15, $total);
        
        // หน้า 2: 5 รายการ
        [$users, $total] = $this->repo->paginate(2, 10);
        $this->assertCount(5, $users);
        $this->assertEquals(15, $total);
    }
    
    public function testUpdate(): void {
        $id = $this->repo->create([
            'username' => 'updatetest',
            'email' => 'update@example.com',
            'password' => 'pass',
        ]);
        
        $updated = $this->repo->update($id, ['username' => 'updated_name']);
        $this->assertEquals('updated_name', $updated['username']);
        $this->assertEquals('update@example.com', $updated['email']); // email ไม่เปลี่ยน
    }
    
    public function testSoftDelete(): void {
        $id = $this->repo->create([
            'username' => 'deletetest',
            'email' => 'delete@example.com',
            'password' => 'pass',
        ]);
        
        $this->assertTrue($this->repo->delete($id));
        
        $user = $this->repo->find($id);
        $this->assertEquals(0, $user['active']); // soft delete
        
        $activeUsers = $this->repo->findAll(['active' => true]);
        $ids = array_column($activeUsers, 'id');
        $this->assertNotContains($id, $ids);
    }
}
```

---

## 6. Workshop: Test ครบวงจรสำหรับ Shopping Cart

### Code ที่จะ Test: Shopping Cart

```php
<?php
namespace App\Cart;

class Product {
    public function __construct(
        public readonly string $id,
        public readonly string $name,
        public readonly float $price,
        private int $stock
    ) {}
    
    public function getStock(): int { return $this->stock; }
    
    public function decreaseStock(int $qty): void {
        if ($qty > $this->stock) {
            throw new \RuntimeException("Insufficient stock for {$this->name}");
        }
        $this->stock -= $qty;
    }
    
    public function isAvailable(int $qty = 1): bool {
        return $this->stock >= $qty;
    }
}

class CartItem {
    private int $quantity;
    
    public function __construct(
        public readonly Product $product,
        int $quantity
    ) {
        $this->setQuantity($quantity);
    }
    
    public function getQuantity(): int { return $this->quantity; }
    
    public function setQuantity(int $qty): void {
        if ($qty <= 0) throw new \InvalidArgumentException("Quantity must be positive");
        if (!$this->product->isAvailable($qty)) {
            throw new \RuntimeException("Not enough stock");
        }
        $this->quantity = $qty;
    }
    
    public function getSubtotal(): float {
        return $this->product->price * $this->quantity;
    }
    
    public function increaseQuantity(int $by = 1): void {
        $this->setQuantity($this->quantity + $by);
    }
}

interface DiscountInterface {
    public function apply(Cart $cart): float;
    public function getDescription(): string;
}

class PercentageDiscount implements DiscountInterface {
    public function __construct(
        private float $percentage,
        private ?float $minAmount = null
    ) {}
    
    public function apply(Cart $cart): float {
        $total = $cart->getSubtotal();
        if ($this->minAmount !== null && $total < $this->minAmount) {
            return 0;
        }
        return $total * ($this->percentage / 100);
    }
    
    public function getDescription(): string {
        $min = $this->minAmount ? " (ขั้นต่ำ {$this->minAmount} บาท)" : "";
        return "ส่วนลด {$this->percentage}%{$min}";
    }
}

class FixedDiscount implements DiscountInterface {
    public function __construct(private float $amount) {}
    
    public function apply(Cart $cart): float {
        return min($this->amount, $cart->getSubtotal());
    }
    
    public function getDescription(): string {
        return "ส่วนลดคงที่ {$this->amount} บาท";
    }
}

class Cart {
    private array $items = [];
    private array $discounts = [];
    private float $shippingCost = 0;
    
    public function addProduct(Product $product, int $quantity = 1): void {
        $id = $product->id;
        
        if (isset($this->items[$id])) {
            $this->items[$id]->increaseQuantity($quantity);
        } else {
            $this->items[$id] = new CartItem($product, $quantity);
        }
    }
    
    public function removeProduct(string $productId): void {
        unset($this->items[$productId]);
    }
    
    public function updateQuantity(string $productId, int $quantity): void {
        if (!isset($this->items[$productId])) {
            throw new \RuntimeException("Product not in cart");
        }
        
        if ($quantity === 0) {
            $this->removeProduct($productId);
            return;
        }
        
        $this->items[$productId]->setQuantity($quantity);
    }
    
    public function addDiscount(DiscountInterface $discount): void {
        $this->discounts[] = $discount;
    }
    
    public function clearDiscounts(): void {
        $this->discounts = [];
    }
    
    public function setShipping(float $cost): void {
        $this->shippingCost = max(0, $cost);
    }
    
    public function getItems(): array {
        return $this->items;
    }
    
    public function getSubtotal(): float {
        return array_sum(array_map(fn($item) => $item->getSubtotal(), $this->items));
    }
    
    public function getTotalDiscount(): float {
        return array_sum(array_map(fn($d) => $d->apply($this), $this->discounts));
    }
    
    public function getTotal(): float {
        return max(0, $this->getSubtotal() - $this->getTotalDiscount() + $this->shippingCost);
    }
    
    public function getItemCount(): int {
        return array_sum(array_map(fn($item) => $item->getQuantity(), $this->items));
    }
    
    public function isEmpty(): bool {
        return empty($this->items);
    }
    
    public function clear(): void {
        $this->items = [];
        $this->discounts = [];
        $this->shippingCost = 0;
    }
    
    public function toArray(): array {
        return [
            'items' => array_map(fn($item) => [
                'product_id' => $item->product->id,
                'name' => $item->product->name,
                'price' => $item->product->price,
                'quantity' => $item->getQuantity(),
                'subtotal' => $item->getSubtotal(),
            ], $this->items),
            'subtotal' => $this->getSubtotal(),
            'discount' => $this->getTotalDiscount(),
            'shipping' => $this->shippingCost,
            'total' => $this->getTotal(),
        ];
    }
}
```

### Test Suite ครบวงจร

```php
<?php
namespace Tests\Unit\Cart;

use App\Cart\{Cart, CartItem, Product, PercentageDiscount, FixedDiscount};
use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\{Test, DataProvider, Group};

#[Group('cart')]
class CartTest extends TestCase {
    
    private Cart $cart;
    private Product $phpBook;
    private Product $laravelCourse;
    private Product $mysqlGuide;
    
    protected function setUp(): void {
        $this->cart = new Cart();
        $this->phpBook = new Product('php-book', 'PHP Book', 450.0, 10);
        $this->laravelCourse = new Product('laravel-course', 'Laravel Course', 1200.0, 5);
        $this->mysqlGuide = new Product('mysql-guide', 'MySQL Guide', 350.0, 0); // out of stock
    }
    
    // ===== Product Tests =====
    
    #[Test]
    public function productHasCorrectProperties(): void {
        $this->assertEquals('php-book', $this->phpBook->id);
        $this->assertEquals('PHP Book', $this->phpBook->name);
        $this->assertEquals(450.0, $this->phpBook->price);
        $this->assertEquals(10, $this->phpBook->getStock());
    }
    
    #[Test]
    public function productIsAvailableWhenHasStock(): void {
        $this->assertTrue($this->phpBook->isAvailable());
        $this->assertTrue($this->phpBook->isAvailable(5));
        $this->assertTrue($this->phpBook->isAvailable(10));
        $this->assertFalse($this->phpBook->isAvailable(11)); // เกิน stock
    }
    
    #[Test]
    public function outOfStockProductIsUnavailable(): void {
        $this->assertFalse($this->mysqlGuide->isAvailable());
    }
    
    // ===== Cart Item Tests =====
    
    #[Test]
    public function cartItemCalculatesSubtotal(): void {
        $item = new CartItem($this->phpBook, 3);
        $this->assertEquals(1350.0, $item->getSubtotal()); // 450 * 3
    }
    
    #[Test]
    public function cartItemThrowsForZeroQuantity(): void {
        $this->expectException(\InvalidArgumentException::class);
        new CartItem($this->phpBook, 0);
    }
    
    #[Test]
    public function cartItemThrowsForNegativeQuantity(): void {
        $this->expectException(\InvalidArgumentException::class);
        new CartItem($this->phpBook, -1);
    }
    
    #[Test]
    public function cartItemThrowsWhenExceedingStock(): void {
        $this->expectException(\RuntimeException::class);
        $this->expectExceptionMessage('Not enough stock');
        
        new CartItem($this->phpBook, 11); // stock มี 10
    }
    
    // ===== Cart Add/Remove Tests =====
    
    #[Test]
    public function newCartIsEmpty(): void {
        $this->assertTrue($this->cart->isEmpty());
        $this->assertEquals(0, $this->cart->getItemCount());
        $this->assertEquals(0.0, $this->cart->getSubtotal());
    }
    
    #[Test]
    public function canAddProductToCart(): void {
        $this->cart->addProduct($this->phpBook, 2);
        
        $this->assertFalse($this->cart->isEmpty());
        $this->assertEquals(2, $this->cart->getItemCount());
        $this->assertEquals(900.0, $this->cart->getSubtotal()); // 450 * 2
    }
    
    #[Test]
    public function addingSameProductIncreasesQuantity(): void {
        $this->cart->addProduct($this->phpBook, 2);
        $this->cart->addProduct($this->phpBook, 3);
        
        $this->assertEquals(5, $this->cart->getItemCount());
        $this->assertEquals(2250.0, $this->cart->getSubtotal()); // 450 * 5
    }
    
    #[Test]
    public function canAddMultipleProducts(): void {
        $this->cart->addProduct($this->phpBook, 1);
        $this->cart->addProduct($this->laravelCourse, 1);
        
        $this->assertEquals(2, count($this->cart->getItems()));
        $this->assertEquals(1650.0, $this->cart->getSubtotal()); // 450 + 1200
    }
    
    #[Test]
    public function canRemoveProductFromCart(): void {
        $this->cart->addProduct($this->phpBook, 2);
        $this->cart->addProduct($this->laravelCourse, 1);
        
        $this->cart->removeProduct('php-book');
        
        $this->assertEquals(1, count($this->cart->getItems()));
        $this->assertEquals(1200.0, $this->cart->getSubtotal());
    }
    
    #[Test]
    public function canUpdateQuantity(): void {
        $this->cart->addProduct($this->phpBook, 2);
        $this->cart->updateQuantity('php-book', 5);
        
        $this->assertEquals(5, $this->cart->getItemCount());
    }
    
    #[Test]
    public function updateQuantityToZeroRemovesProduct(): void {
        $this->cart->addProduct($this->phpBook, 2);
        $this->cart->updateQuantity('php-book', 0);
        
        $this->assertTrue($this->cart->isEmpty());
    }
    
    #[Test]
    public function updateNonExistentProductThrows(): void {
        $this->expectException(\RuntimeException::class);
        $this->cart->updateQuantity('non-existent', 1);
    }
    
    #[Test]
    public function clearCartRemovesEverything(): void {
        $this->cart->addProduct($this->phpBook, 2);
        $this->cart->addDiscount(new PercentageDiscount(10));
        $this->cart->setShipping(50);
        
        $this->cart->clear();
        
        $this->assertTrue($this->cart->isEmpty());
        $this->assertEquals(0.0, $this->cart->getTotal());
    }
    
    // ===== Discount Tests =====
    
    #[Test]
    public function percentageDiscountCalculatesCorrectly(): void {
        $this->cart->addProduct($this->phpBook, 2); // 900 บาท
        $this->cart->addDiscount(new PercentageDiscount(10));
        
        $this->assertEquals(90.0, $this->cart->getTotalDiscount()); // 10% of 900
        $this->assertEquals(810.0, $this->cart->getTotal()); // 900 - 90
    }
    
    #[Test]
    public function percentageDiscountWithMinAmountNotApplied(): void {
        $this->cart->addProduct($this->phpBook, 1); // 450 บาท
        $this->cart->addDiscount(new PercentageDiscount(10, 500)); // ขั้นต่ำ 500
        
        $this->assertEquals(0.0, $this->cart->getTotalDiscount()); // ไม่ถึงขั้นต่ำ
    }
    
    #[Test]
    public function percentageDiscountWithMinAmountApplied(): void {
        $this->cart->addProduct($this->phpBook, 2); // 900 บาท
        $this->cart->addDiscount(new PercentageDiscount(10, 500)); // ขั้นต่ำ 500
        
        $this->assertEquals(90.0, $this->cart->getTotalDiscount()); // ถึงขั้นต่ำ
    }
    
    #[Test]
    public function fixedDiscountCalculatesCorrectly(): void {
        $this->cart->addProduct($this->phpBook, 2); // 900 บาท
        $this->cart->addDiscount(new FixedDiscount(100));
        
        $this->assertEquals(100.0, $this->cart->getTotalDiscount());
        $this->assertEquals(800.0, $this->cart->getTotal());
    }
    
    #[Test]
    public function fixedDiscountCannotExceedTotal(): void {
        $this->cart->addProduct($this->phpBook, 1); // 450 บาท
        $this->cart->addDiscount(new FixedDiscount(1000)); // ส่วนลดมากกว่าราคา
        
        $this->assertEquals(450.0, $this->cart->getTotalDiscount()); // max = subtotal
        $this->assertEquals(0.0, $this->cart->getTotal()); // ไม่ติดลบ
    }
    
    #[Test]
    public function multipleDiscountsStackCorrectly(): void {
        $this->cart->addProduct($this->phpBook, 2); // 900 บาท
        $this->cart->addDiscount(new PercentageDiscount(10)); // -90
        $this->cart->addDiscount(new FixedDiscount(50));       // -50
        
        $this->assertEquals(140.0, $this->cart->getTotalDiscount());
        $this->assertEquals(760.0, $this->cart->getTotal());
    }
    
    // ===== Shipping Tests =====
    
    #[Test]
    public function shippingCostAdded(): void {
        $this->cart->addProduct($this->phpBook, 1);
        $this->cart->setShipping(50);
        
        $this->assertEquals(500.0, $this->cart->getTotal()); // 450 + 50
    }
    
    #[Test]
    public function shippingCannotBeNegative(): void {
        $this->cart->addProduct($this->phpBook, 1);
        $this->cart->setShipping(-100); // ค่าลบถูก ignore
        
        $this->assertEquals(450.0, $this->cart->getTotal()); // ไม่มี shipping
    }
    
    // ===== Data Provider Tests =====
    
    public static function discountProvider(): array {
        return [
            '10% off'         => [10, 0, 900, 90, 810],
            '20% off'         => [20, 0, 900, 180, 720],
            '50% with min'    => [50, 1000, 900, 0, 900], // ไม่ถึงขั้นต่ำ
            '50% above min'   => [50, 500, 900, 450, 450], // ถึงขั้นต่ำ
        ];
    }
    
    #[DataProvider('discountProvider')]
    public function testDiscountScenarios(
        float $percentage,
        float $minAmount,
        float $cartTotal,
        float $expectedDiscount,
        float $expectedTotal
    ): void {
        // สร้าง product ที่มีราคาตามต้องการ
        $product = new Product('test', 'Test Product', $cartTotal, 100);
        $this->cart->addProduct($product, 1);
        $this->cart->addDiscount(new PercentageDiscount($percentage, $minAmount ?: null));
        
        $this->assertEquals($expectedDiscount, $this->cart->getTotalDiscount(), "Discount mismatch", 0.01);
        $this->assertEquals($expectedTotal, $this->cart->getTotal(), "Total mismatch", 0.01);
    }
    
    // ===== Complete Flow Test =====
    
    #[Test]
    public function completeShoppingScenario(): void {
        // ลูกค้าเพิ่มสินค้า
        $this->cart->addProduct($this->phpBook, 2);      // 900
        $this->cart->addProduct($this->laravelCourse, 1); // 1200
        
        // ใส่ coupon
        $this->cart->addDiscount(new PercentageDiscount(15, 1000)); // 15% ถ้า >= 1000
        
        // ค่าส่ง
        $this->cart->setShipping(80);
        
        $subtotal = $this->cart->getSubtotal(); // 2100
        $discount = $this->cart->getTotalDiscount(); // 315
        $total = $this->cart->getTotal(); // 2100 - 315 + 80 = 1865
        
        $this->assertEquals(2100.0, $subtotal);
        $this->assertEquals(315.0, $discount); // 15% of 2100
        $this->assertEquals(1865.0, $total);
        
        // Convert to array
        $data = $this->cart->toArray();
        $this->assertArrayHasKey('items', $data);
        $this->assertArrayHasKey('subtotal', $data);
        $this->assertArrayHasKey('discount', $data);
        $this->assertArrayHasKey('shipping', $data);
        $this->assertArrayHasKey('total', $data);
        $this->assertCount(2, $data['items']);
        $this->assertEquals(1865.0, $data['total']);
    }
    
    #[Test]
    public function cartToArrayHasCorrectStructure(): void {
        $this->cart->addProduct($this->phpBook, 1);
        
        $data = $this->cart->toArray();
        
        $this->assertIsArray($data['items']);
        $item = $data['items'][0];
        
        $this->assertArrayHasKey('product_id', $item);
        $this->assertArrayHasKey('name', $item);
        $this->assertArrayHasKey('price', $item);
        $this->assertArrayHasKey('quantity', $item);
        $this->assertArrayHasKey('subtotal', $item);
        
        $this->assertEquals('php-book', $item['product_id']);
        $this->assertEquals(450.0, $item['price']);
        $this->assertEquals(1, $item['quantity']);
        $this->assertEquals(450.0, $item['subtotal']);
    }
}
```

### รัน Tests

```bash
# รัน tests ทั้งหมด
./vendor/bin/phpunit

# รัน specific test file
./vendor/bin/phpunit tests/Unit/Cart/CartTest.php

# รัน specific test method
./vendor/bin/phpunit --filter testCompleteShoppingScenario

# รัน tests ใน group
./vendor/bin/phpunit --group cart

# รัน พร้อม coverage
./vendor/bin/phpunit --coverage-html coverage/html

# รัน verbose
./vendor/bin/phpunit --testdox

# รัน specific testsuite
./vendor/bin/phpunit --testsuite Unit
```

---

## 7. Code Coverage

```bash
# ต้องมี Xdebug หรือ PCOV
pecl install xdebug

# หรือ PCOV (เร็วกว่า Xdebug)
pecl install pcov
```

```xml
<!-- phpunit.xml -->
<source>
    <include>
        <directory suffix=".php">src</directory>
    </include>
</source>

<coverage>
    <report>
        <html outputDirectory="coverage/html"/>
        <text outputFile="php://stdout" showUncoveredFiles="true"/>
    </report>
</coverage>
```

```php
<?php
// Attribute สำหรับ skip coverage (code ที่ไม่สมเหตุสมผลที่จะ test)
use PHPUnit\Framework\Attributes\CoversClass;
use PHPUnit\Framework\Attributes\CoversMethod;
use PHPUnit\Framework\Attributes\ExcludeGlobalVariableFromBackup;

#[CoversClass(Cart::class)]
class CartTest extends TestCase {
    // ทุก test ใน class นี้ count เข้า coverage ของ Cart::class
}

// @codeCoverageIgnore ใน source code
class Config {
    // @codeCoverageIgnoreStart
    private static function legacyMethod(): void {
        // โค้ดเก่าที่ไม่ test
    }
    // @codeCoverageIgnoreEnd
}
```

---

## Quiz

### คำถาม 1
`assertEquals` กับ `assertSame` ต่างกันอย่างไร?
- A. ไม่ต่างกัน
- B. assertEquals ใช้ == (loose), assertSame ใช้ === (strict)
- C. assertSame ช้ากว่า
- D. assertEquals ใช้ได้กับ objects เท่านั้น

**เฉลย: B** - `assertEquals(0, false)` passes แต่ `assertSame(0, false)` fails

### คำถาม 2
Data Provider ต้องมี signature แบบไหน?
- A. `private function provider(): array`
- B. `public static function provider(): array|Generator`
- C. `protected function provider(): iterable`
- D. ทุกแบบใช้ได้

**เฉลย: B** - ต้องเป็น `public static` method และ return array หรือ Generator

### คำถาม 3
ทำไมต้องเรียก `Mockery::close()` ใน `tearDown()`?
- A. เพื่อ free memory
- B. เพื่อ verify expectations ที่ตั้งไว้
- C. เพื่อ reset state
- D. ถูกทั้ง A, B และ C

**เฉลย: D** - Mockery::close() verify ว่า expectations ถูกต้อง, ทำความสะอาด, และ free memory

### คำถาม 4
Unit Test กับ Integration Test ต่างกันอย่างไร?

**เฉลย:**
- **Unit Test**: ทดสอบ class/method เดียวแยกออกจากสิ่งอื่น ใช้ mock แทน dependencies ที่แท้จริง เร็วมาก
- **Integration Test**: ทดสอบหลาย components ทำงานร่วมกัน เช่น Repository + Database อาจช้ากว่า unit test แต่ตรวจสอบการทำงานจริงได้

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- **PHPUnit Setup** - การตั้งค่า phpunit.xml, directory structure
- **Unit Tests** - assertions, setUp/tearDown, test lifecycle
- **Data Providers** - testing หลาย scenarios อย่างมีประสิทธิภาพ
- **Mocking** - PHPUnit built-in mocks และ Mockery
- **Integration Tests** - ทดสอบกับ database จริง (SQLite in-memory)
- **Shopping Cart Tests** - test suite ครบวงจร 25+ test cases

---

## ➡️ Part ถัดไป

[Part 024: PHP Performance Optimization](./part-024-php-performance.md)
