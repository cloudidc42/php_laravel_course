# Part 86: Design Patterns ใน PHP

## บทนำ

Design Patterns คือแนวทางการแก้ปัญหาที่พิสูจน์แล้วว่าได้ผลสำหรับปัญหาที่พบบ่อยในการออกแบบซอฟต์แวร์ ถูกรวบรวมโดย Gang of Four (GoF) ในหนังสือ "Design Patterns: Elements of Reusable Object-Oriented Software" ในปี 1994

Patterns แบ่งออกเป็น 3 กลุ่ม:
- **Creational**: เกี่ยวกับการสร้าง Object
- **Structural**: เกี่ยวกับโครงสร้างของ Class
- **Behavioral**: เกี่ยวกับพฤติกรรมและการสื่อสารระหว่าง Object

---

## Creational Patterns

### 1. Singleton Pattern

Singleton รับประกันว่า Class จะมี Instance เพียงหนึ่งเดียว และมี Global Access Point

**ปัญหาที่แก้ได้**: เมื่อต้องการ Resource ที่แชร์กัน เช่น Database Connection, Configuration, Logger

```php
<?php

class DatabaseConnection
{
    private static ?DatabaseConnection $instance = null;
    private \PDO $connection;
    
    // Private constructor ป้องกันการ new
    private function __construct(
        private string $dsn,
        private string $username,
        private string $password
    ) {
        $this->connection = new \PDO($dsn, $username, $password);
        $this->connection->setAttribute(\PDO::ATTR_ERRMODE, \PDO::ERRMODE_EXCEPTION);
    }
    
    // ป้องกัน Clone
    private function __clone() {}
    
    // ป้องกัน Unserialize
    public function __wakeup()
    {
        throw new \Exception("Cannot unserialize singleton");
    }
    
    public static function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self(
                dsn: 'mysql:host=localhost;dbname=mydb',
                username: 'root',
                password: 'secret'
            );
        }
        
        return self::$instance;
    }
    
    public function getConnection(): \PDO
    {
        return $this->connection;
    }
    
    public function query(string $sql, array $params = []): \PDOStatement
    {
        $stmt = $this->connection->prepare($sql);
        $stmt->execute($params);
        return $stmt;
    }
}

// การใช้งาน
$db1 = DatabaseConnection::getInstance();
$db2 = DatabaseConnection::getInstance();

var_dump($db1 === $db2); // true - เป็น instance เดียวกัน

$users = $db1->query("SELECT * FROM users WHERE active = ?", [1]);
```

**Thread-Safe Singleton** (สำหรับ PHP-FPM หรือ Swoole):

```php
<?php

class Registry
{
    private static array $instances = [];
    private static array $locks = [];
    
    public static function getInstance(string $class): object
    {
        if (!isset(self::$instances[$class])) {
            // Double-checked locking
            if (!isset(self::$locks[$class])) {
                self::$locks[$class] = true;
                self::$instances[$class] = new $class();
                unset(self::$locks[$class]);
            }
        }
        
        return self::$instances[$class];
    }
}
```

---

### 2. Factory Pattern

Factory Pattern สร้าง Object โดยไม่ระบุ Class ที่แน่นอนที่จะถูกสร้าง

**ปัญหาที่แก้ได้**: เมื่อต้องสร้าง Object ที่ขึ้นกับ Context หรือ Configuration

```php
<?php

// Abstract Product
interface Logger
{
    public function log(string $level, string $message, array $context = []): void;
}

// Concrete Products
class FileLogger implements Logger
{
    public function __construct(private string $path) {}
    
    public function log(string $level, string $message, array $context = []): void
    {
        $formatted = sprintf(
            "[%s] %s: %s %s\n",
            date('Y-m-d H:i:s'),
            strtoupper($level),
            $message,
            !empty($context) ? json_encode($context) : ''
        );
        
        file_put_contents($this->path, $formatted, FILE_APPEND);
    }
}

class DatabaseLogger implements Logger
{
    public function __construct(private \PDO $db) {}
    
    public function log(string $level, string $message, array $context = []): void
    {
        $stmt = $this->db->prepare(
            "INSERT INTO logs (level, message, context, created_at) VALUES (?, ?, ?, ?)"
        );
        $stmt->execute([$level, $message, json_encode($context), date('Y-m-d H:i:s')]);
    }
}

class SlackLogger implements Logger
{
    public function __construct(
        private string $webhookUrl,
        private string $channel = '#alerts'
    ) {}
    
    public function log(string $level, string $message, array $context = []): void
    {
        $payload = [
            'channel' => $this->channel,
            'text' => sprintf("[%s] %s", strtoupper($level), $message),
            'attachments' => [
                ['text' => json_encode($context)]
            ]
        ];
        
        // Send to Slack
        $ch = curl_init($this->webhookUrl);
        curl_setopt($ch, CURLOPT_POST, 1);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
        curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
        curl_exec($ch);
        curl_close($ch);
    }
}

// Factory
class LoggerFactory
{
    public static function create(string $type, array $config = []): Logger
    {
        return match($type) {
            'file' => new FileLogger($config['path'] ?? '/var/log/app.log'),
            'database' => new DatabaseLogger($config['db']),
            'slack' => new SlackLogger($config['webhook_url'], $config['channel'] ?? '#alerts'),
            default => throw new \InvalidArgumentException("Unknown logger type: {$type}")
        };
    }
    
    public static function createFromConfig(array $config): Logger
    {
        $type = $config['driver'] ?? 'file';
        return self::create($type, $config);
    }
}

// การใช้งาน
$fileLogger = LoggerFactory::create('file', ['path' => '/var/log/myapp.log']);
$fileLogger->log('info', 'Application started');

// จาก Config
$config = [
    'driver' => 'slack',
    'webhook_url' => 'https://hooks.slack.com/...',
    'channel' => '#production-alerts'
];
$logger = LoggerFactory::createFromConfig($config);
$logger->log('error', 'Critical error occurred', ['exception' => 'OutOfMemoryException']);
```

---

### 3. Builder Pattern

Builder Pattern สร้าง Complex Object ทีละขั้นตอน

**ปัญหาที่แก้ได้**: เมื่อ Object มี Constructor ที่ซับซ้อนมาก หรือมีหลาย Combination ของ Attributes

```php
<?php

class QueryBuilder
{
    private string $table = '';
    private array $columns = ['*'];
    private array $conditions = [];
    private array $orderBy = [];
    private ?int $limit = null;
    private ?int $offset = null;
    private array $joins = [];
    private array $bindings = [];
    
    public function table(string $table): self
    {
        $clone = clone $this;
        $clone->table = $table;
        return $clone;
    }
    
    public function select(string ...$columns): self
    {
        $clone = clone $this;
        $clone->columns = $columns;
        return $clone;
    }
    
    public function where(string $column, string $operator, mixed $value): self
    {
        $clone = clone $this;
        $clone->conditions[] = "{$column} {$operator} ?";
        $clone->bindings[] = $value;
        return $clone;
    }
    
    public function whereIn(string $column, array $values): self
    {
        $clone = clone $this;
        $placeholders = implode(', ', array_fill(0, count($values), '?'));
        $clone->conditions[] = "{$column} IN ({$placeholders})";
        $clone->bindings = array_merge($clone->bindings, $values);
        return $clone;
    }
    
    public function join(string $table, string $first, string $operator, string $second): self
    {
        $clone = clone $this;
        $clone->joins[] = "JOIN {$table} ON {$first} {$operator} {$second}";
        return $clone;
    }
    
    public function leftJoin(string $table, string $first, string $operator, string $second): self
    {
        $clone = clone $this;
        $clone->joins[] = "LEFT JOIN {$table} ON {$first} {$operator} {$second}";
        return $clone;
    }
    
    public function orderBy(string $column, string $direction = 'ASC'): self
    {
        $clone = clone $this;
        $clone->orderBy[] = "{$column} {$direction}";
        return $clone;
    }
    
    public function limit(int $limit): self
    {
        $clone = clone $this;
        $clone->limit = $limit;
        return $clone;
    }
    
    public function offset(int $offset): self
    {
        $clone = clone $this;
        $clone->offset = $offset;
        return $clone;
    }
    
    public function build(): array
    {
        $sql = sprintf(
            "SELECT %s FROM %s",
            implode(', ', $this->columns),
            $this->table
        );
        
        if (!empty($this->joins)) {
            $sql .= ' ' . implode(' ', $this->joins);
        }
        
        if (!empty($this->conditions)) {
            $sql .= ' WHERE ' . implode(' AND ', $this->conditions);
        }
        
        if (!empty($this->orderBy)) {
            $sql .= ' ORDER BY ' . implode(', ', $this->orderBy);
        }
        
        if ($this->limit !== null) {
            $sql .= " LIMIT {$this->limit}";
        }
        
        if ($this->offset !== null) {
            $sql .= " OFFSET {$this->offset}";
        }
        
        return ['sql' => $sql, 'bindings' => $this->bindings];
    }
}

// การใช้งาน
$query = (new QueryBuilder())
    ->table('users')
    ->select('users.id', 'users.name', 'users.email', 'roles.name as role')
    ->leftJoin('user_roles', 'users.id', '=', 'user_roles.user_id')
    ->leftJoin('roles', 'user_roles.role_id', '=', 'roles.id')
    ->where('users.active', '=', 1)
    ->whereIn('roles.name', ['admin', 'moderator'])
    ->orderBy('users.name', 'ASC')
    ->limit(20)
    ->offset(0)
    ->build();

echo $query['sql'];
// SELECT users.id, users.name, users.email, roles.name as role
// FROM users
// LEFT JOIN user_roles ON users.id = user_roles.user_id
// LEFT JOIN roles ON user_roles.role_id = roles.id
// WHERE users.active = ? AND roles.name IN (?, ?)
// ORDER BY users.name ASC LIMIT 20 OFFSET 0
```

---

### 4. Prototype Pattern

Prototype Pattern สร้าง Object ใหม่โดยการ Clone Object ที่มีอยู่แล้ว

```php
<?php

abstract class Shape
{
    protected string $color = 'black';
    protected int $x = 0;
    protected int $y = 0;
    
    abstract public function area(): float;
    
    public function setColor(string $color): self
    {
        $clone = clone $this;
        $clone->color = $color;
        return $clone;
    }
    
    public function moveTo(int $x, int $y): self
    {
        $clone = clone $this;
        $clone->x = $x;
        $clone->y = $y;
        return $clone;
    }
    
    public function clone(): static
    {
        return clone $this;
    }
}

class Circle extends Shape
{
    public function __construct(private float $radius) {}
    
    public function area(): float
    {
        return M_PI * $this->radius ** 2;
    }
    
    public function withRadius(float $radius): self
    {
        $clone = clone $this;
        $clone->radius = $radius;
        return $clone;
    }
}

class Rectangle extends Shape
{
    public function __construct(
        private float $width,
        private float $height
    ) {}
    
    public function area(): float
    {
        return $this->width * $this->height;
    }
}

// ShapeRegistry เป็น Prototype Registry
class ShapeRegistry
{
    private array $prototypes = [];
    
    public function register(string $name, Shape $shape): void
    {
        $this->prototypes[$name] = $shape;
    }
    
    public function create(string $name): Shape
    {
        if (!isset($this->prototypes[$name])) {
            throw new \InvalidArgumentException("Unknown shape: {$name}");
        }
        
        return clone $this->prototypes[$name];
    }
}

// การใช้งาน
$registry = new ShapeRegistry();
$registry->register('circle', new Circle(5));
$registry->register('big-circle', new Circle(50));
$registry->register('square', new Rectangle(10, 10));

$c1 = $registry->create('circle');
$c2 = $registry->create('circle')->setColor('red');
$c3 = (new Circle(5))->withRadius(10)->setColor('blue');
```

---

## Structural Patterns

### 5. Adapter Pattern

Adapter ทำให้ Interface ที่ไม่เข้ากันสามารถทำงานร่วมกันได้

```php
<?php

// Target Interface - ที่ระบบของเราคาดหวัง
interface PaymentGateway
{
    public function charge(float $amount, string $currency, array $cardDetails): PaymentResult;
    public function refund(string $transactionId, float $amount): RefundResult;
}

class PaymentResult
{
    public function __construct(
        public readonly bool $success,
        public readonly string $transactionId,
        public readonly string $message
    ) {}
}

class RefundResult
{
    public function __construct(
        public readonly bool $success,
        public readonly string $refundId,
        public readonly string $message
    ) {}
}

// Adaptee - Library ของ Third-party ที่มี Interface ต่างออกไป
class StripeClient
{
    public function createCharge(array $params): array
    {
        // Stripe API call
        return [
            'id' => 'ch_' . uniqid(),
            'status' => 'succeeded',
            'amount' => $params['amount'],
            'currency' => $params['currency']
        ];
    }
    
    public function createRefund(string $chargeId, int $amount): array
    {
        return [
            'id' => 're_' . uniqid(),
            'status' => 'succeeded',
            'charge' => $chargeId
        ];
    }
}

// Adapter
class StripePaymentAdapter implements PaymentGateway
{
    public function __construct(private StripeClient $stripe) {}
    
    public function charge(float $amount, string $currency, array $cardDetails): PaymentResult
    {
        try {
            $result = $this->stripe->createCharge([
                'amount' => (int)($amount * 100), // Stripe ใช้ cents
                'currency' => strtolower($currency),
                'source' => $cardDetails['token'] ?? null,
                'description' => $cardDetails['description'] ?? 'Payment'
            ]);
            
            return new PaymentResult(
                success: $result['status'] === 'succeeded',
                transactionId: $result['id'],
                message: 'Payment successful'
            );
        } catch (\Exception $e) {
            return new PaymentResult(
                success: false,
                transactionId: '',
                message: $e->getMessage()
            );
        }
    }
    
    public function refund(string $transactionId, float $amount): RefundResult
    {
        try {
            $result = $this->stripe->createRefund($transactionId, (int)($amount * 100));
            
            return new RefundResult(
                success: $result['status'] === 'succeeded',
                refundId: $result['id'],
                message: 'Refund successful'
            );
        } catch (\Exception $e) {
            return new RefundResult(
                success: false,
                refundId: '',
                message: $e->getMessage()
            );
        }
    }
}

// PayPal Adapter
class PayPalClient
{
    public function executePayment(float $total, string $currency): string
    {
        // Returns transaction ID
        return 'PAY-' . strtoupper(uniqid());
    }
    
    public function processRefund(string $saleId): bool
    {
        return true;
    }
}

class PayPalPaymentAdapter implements PaymentGateway
{
    public function __construct(private PayPalClient $paypal) {}
    
    public function charge(float $amount, string $currency, array $cardDetails): PaymentResult
    {
        $transactionId = $this->paypal->executePayment($amount, $currency);
        
        return new PaymentResult(
            success: true,
            transactionId: $transactionId,
            message: 'PayPal payment successful'
        );
    }
    
    public function refund(string $transactionId, float $amount): RefundResult
    {
        $success = $this->paypal->processRefund($transactionId);
        
        return new RefundResult(
            success: $success,
            refundId: 'REF-' . uniqid(),
            message: $success ? 'Refund processed' : 'Refund failed'
        );
    }
}

// การใช้งาน - ระบบเราไม่ต้องรู้ว่าใช้ Payment Gateway ไหน
class OrderService
{
    public function __construct(private PaymentGateway $gateway) {}
    
    public function processPayment(float $amount, array $cardDetails): PaymentResult
    {
        return $this->gateway->charge($amount, 'THB', $cardDetails);
    }
}

$stripeService = new OrderService(new StripePaymentAdapter(new StripeClient()));
$paypalService = new OrderService(new PayPalPaymentAdapter(new PayPalClient()));
```

---

### 6. Decorator Pattern

Decorator เพิ่ม Functionality ให้ Object โดยการ Wrap มัน

```php
<?php

interface TextProcessor
{
    public function process(string $text): string;
}

class PlainTextProcessor implements TextProcessor
{
    public function process(string $text): string
    {
        return $text;
    }
}

abstract class TextProcessorDecorator implements TextProcessor
{
    public function __construct(protected TextProcessor $processor) {}
}

class UpperCaseDecorator extends TextProcessorDecorator
{
    public function process(string $text): string
    {
        return strtoupper($this->processor->process($text));
    }
}

class TrimDecorator extends TextProcessorDecorator
{
    public function process(string $text): string
    {
        return trim($this->processor->process($text));
    }
}

class HtmlEncodeDecorator extends TextProcessorDecorator
{
    public function process(string $text): string
    {
        return htmlspecialchars($this->processor->process($text), ENT_QUOTES, 'UTF-8');
    }
}

class MarkdownDecorator extends TextProcessorDecorator
{
    private array $patterns = [
        '/\*\*(.*?)\*\*/s' => '<strong>$1</strong>',
        '/\*(.*?)\*/s' => '<em>$1</em>',
        '/`(.*?)`/' => '<code>$1</code>',
        '/\[([^\]]+)\]\(([^\)]+)\)/' => '<a href="$2">$1</a>',
    ];
    
    public function process(string $text): string
    {
        $processed = $this->processor->process($text);
        
        foreach ($this->patterns as $pattern => $replacement) {
            $processed = preg_replace($pattern, $replacement, $processed);
        }
        
        return $processed;
    }
}

class LoggingDecorator extends TextProcessorDecorator
{
    private array $log = [];
    
    public function process(string $text): string
    {
        $start = microtime(true);
        $result = $this->processor->process($text);
        $end = microtime(true);
        
        $this->log[] = [
            'input_length' => strlen($text),
            'output_length' => strlen($result),
            'time_ms' => ($end - $start) * 1000
        ];
        
        return $result;
    }
    
    public function getLog(): array
    {
        return $this->log;
    }
}

// การใช้งาน
$processor = new LoggingDecorator(
    new HtmlEncodeDecorator(
        new TrimDecorator(
            new PlainTextProcessor()
        )
    )
);

$result = $processor->process("  <script>alert('xss')</script>  ");
echo $result; // &lt;script&gt;alert(&#039;xss&#039;)&lt;/script&gt;
```

---

### 7. Facade Pattern

Facade ให้ Interface ที่เรียบง่ายสำหรับ Subsystem ที่ซับซ้อน

```php
<?php

// Subsystems ที่ซับซ้อน
class Inventory
{
    public function checkAvailability(int $productId, int $quantity): bool
    {
        // ตรวจสอบสต็อก
        return true;
    }
    
    public function reserve(int $productId, int $quantity): string
    {
        // จองสินค้า
        return 'RESERVE-' . uniqid();
    }
    
    public function release(string $reservationId): void
    {
        // คืนการจอง
    }
}

class ShippingCalculator
{
    public function calculateCost(array $address, float $weight): float
    {
        // คำนวณค่าส่ง
        return 50.0;
    }
    
    public function estimateDeliveryDate(array $address): \DateTime
    {
        return new \DateTime('+3 days');
    }
}

class TaxCalculator
{
    public function calculate(float $subtotal, string $countryCode): float
    {
        $rates = ['TH' => 0.07, 'US' => 0.08, 'UK' => 0.20];
        return $subtotal * ($rates[$countryCode] ?? 0);
    }
}

class EmailNotificationService
{
    public function sendOrderConfirmation(array $order): void
    {
        // ส่ง Email ยืนยัน
    }
    
    public function sendShippingNotification(array $order, string $trackingNumber): void
    {
        // ส่ง Email แจ้ง Tracking
    }
}

// Facade
class OrderFacade
{
    private Inventory $inventory;
    private ShippingCalculator $shipping;
    private TaxCalculator $tax;
    private EmailNotificationService $email;
    
    public function __construct()
    {
        $this->inventory = new Inventory();
        $this->shipping = new ShippingCalculator();
        $this->tax = new TaxCalculator();
        $this->email = new EmailNotificationService();
    }
    
    public function placeOrder(array $items, array $shippingAddress, string $countryCode): array
    {
        // ตรวจสอบสต็อก
        foreach ($items as $item) {
            if (!$this->inventory->checkAvailability($item['product_id'], $item['quantity'])) {
                throw new \RuntimeException("Product {$item['product_id']} is out of stock");
            }
        }
        
        // จองสินค้า
        $reservations = [];
        foreach ($items as $item) {
            $reservations[] = $this->inventory->reserve($item['product_id'], $item['quantity']);
        }
        
        // คำนวณราคา
        $subtotal = array_sum(array_column($items, 'total'));
        $totalWeight = array_sum(array_column($items, 'weight'));
        $shippingCost = $this->shipping->calculateCost($shippingAddress, $totalWeight);
        $tax = $this->tax->calculate($subtotal, $countryCode);
        $total = $subtotal + $shippingCost + $tax;
        
        $order = [
            'id' => 'ORD-' . uniqid(),
            'items' => $items,
            'subtotal' => $subtotal,
            'shipping_cost' => $shippingCost,
            'tax' => $tax,
            'total' => $total,
            'shipping_address' => $shippingAddress,
            'estimated_delivery' => $this->shipping->estimateDeliveryDate($shippingAddress)->format('Y-m-d'),
            'reservations' => $reservations
        ];
        
        // ส่ง Email
        $this->email->sendOrderConfirmation($order);
        
        return $order;
    }
}

// การใช้งาน - เรียบง่ายมาก
$orderFacade = new OrderFacade();

try {
    $order = $orderFacade->placeOrder(
        items: [
            ['product_id' => 1, 'quantity' => 2, 'total' => 400.0, 'weight' => 0.5],
            ['product_id' => 5, 'quantity' => 1, 'total' => 150.0, 'weight' => 0.2],
        ],
        shippingAddress: [
            'name' => 'สมชาย ใจดี',
            'address' => '123 ถนนสุขุมวิท',
            'city' => 'กรุงเทพ',
            'postal_code' => '10110'
        ],
        countryCode: 'TH'
    );
    
    echo "Order placed: " . $order['id'] . "\n";
    echo "Total: " . number_format($order['total'], 2) . " THB\n";
} catch (\Exception $e) {
    echo "Order failed: " . $e->getMessage();
}
```

---

### 8. Proxy Pattern

Proxy เป็น Wrapper ที่ควบคุมการเข้าถึง Object

```php
<?php

interface UserRepository
{
    public function findById(int $id): ?array;
    public function findAll(): array;
    public function save(array $user): bool;
}

class DatabaseUserRepository implements UserRepository
{
    public function __construct(private \PDO $db) {}
    
    public function findById(int $id): ?array
    {
        $stmt = $this->db->prepare("SELECT * FROM users WHERE id = ?");
        $stmt->execute([$id]);
        return $stmt->fetch(\PDO::FETCH_ASSOC) ?: null;
    }
    
    public function findAll(): array
    {
        return $this->db->query("SELECT * FROM users")->fetchAll(\PDO::FETCH_ASSOC);
    }
    
    public function save(array $user): bool
    {
        // ...
        return true;
    }
}

// Caching Proxy
class CachedUserRepository implements UserRepository
{
    private array $cache = [];
    
    public function __construct(
        private UserRepository $repository,
        private int $ttl = 300 // 5 minutes
    ) {}
    
    public function findById(int $id): ?array
    {
        $cacheKey = "user_{$id}";
        
        if (isset($this->cache[$cacheKey])) {
            $cached = $this->cache[$cacheKey];
            if (time() - $cached['time'] < $this->ttl) {
                return $cached['data'];
            }
        }
        
        $user = $this->repository->findById($id);
        
        if ($user !== null) {
            $this->cache[$cacheKey] = [
                'data' => $user,
                'time' => time()
            ];
        }
        
        return $user;
    }
    
    public function findAll(): array
    {
        $cacheKey = 'all_users';
        
        if (isset($this->cache[$cacheKey])) {
            $cached = $this->cache[$cacheKey];
            if (time() - $cached['time'] < $this->ttl) {
                return $cached['data'];
            }
        }
        
        $users = $this->repository->findAll();
        $this->cache[$cacheKey] = ['data' => $users, 'time' => time()];
        
        return $users;
    }
    
    public function save(array $user): bool
    {
        // Clear cache on save
        $this->cache = [];
        return $this->repository->save($user);
    }
}

// Authorization Proxy
class AuthorizedUserRepository implements UserRepository
{
    public function __construct(
        private UserRepository $repository,
        private string $currentUserRole
    ) {}
    
    public function findById(int $id): ?array
    {
        return $this->repository->findById($id);
    }
    
    public function findAll(): array
    {
        if ($this->currentUserRole !== 'admin') {
            throw new \UnauthorizedException("Only admins can list all users");
        }
        
        return $this->repository->findAll();
    }
    
    public function save(array $user): bool
    {
        if (!in_array($this->currentUserRole, ['admin', 'manager'])) {
            throw new \UnauthorizedException("Insufficient permissions to save user");
        }
        
        return $this->repository->save($user);
    }
}
```

---

## Behavioral Patterns

### 9. Observer Pattern

Observer Pattern ให้ Object แจ้ง Object อื่นๆ เมื่อ State เปลี่ยน

```php
<?php

// PHP SplObserver/SplSubject interfaces
interface EventInterface
{
    public function getName(): string;
    public function getData(): array;
}

interface EventListenerInterface
{
    public function handle(EventInterface $event): void;
}

class Event implements EventInterface
{
    public function __construct(
        private string $name,
        private array $data = []
    ) {}
    
    public function getName(): string { return $this->name; }
    public function getData(): array { return $this->data; }
}

class EventDispatcher
{
    private array $listeners = [];
    
    public function listen(string $eventName, EventListenerInterface|callable $listener): void
    {
        $this->listeners[$eventName][] = $listener;
    }
    
    public function dispatch(EventInterface $event): void
    {
        $listeners = $this->listeners[$event->getName()] ?? [];
        
        foreach ($listeners as $listener) {
            if ($listener instanceof EventListenerInterface) {
                $listener->handle($event);
            } else {
                $listener($event);
            }
        }
    }
}

// Events
class UserRegisteredEvent extends Event
{
    public function __construct(
        public readonly array $user
    ) {
        parent::__construct('user.registered', ['user' => $user]);
    }
}

class OrderPlacedEvent extends Event
{
    public function __construct(
        public readonly array $order,
        public readonly array $customer
    ) {
        parent::__construct('order.placed', [
            'order' => $order,
            'customer' => $customer
        ]);
    }
}

// Listeners
class SendWelcomeEmailListener implements EventListenerInterface
{
    public function handle(EventInterface $event): void
    {
        $user = $event->getData()['user'];
        echo "Sending welcome email to: {$user['email']}\n";
        // ส่ง Email
    }
}

class CreateDefaultSettingsListener implements EventListenerInterface
{
    public function handle(EventInterface $event): void
    {
        $user = $event->getData()['user'];
        echo "Creating default settings for user: {$user['id']}\n";
        // สร้าง Settings default
    }
}

class NotifyAdminListener implements EventListenerInterface
{
    public function handle(EventInterface $event): void
    {
        echo "Notifying admin about new user registration\n";
    }
}

// การใช้งาน
$dispatcher = new EventDispatcher();

$dispatcher->listen('user.registered', new SendWelcomeEmailListener());
$dispatcher->listen('user.registered', new CreateDefaultSettingsListener());
$dispatcher->listen('user.registered', new NotifyAdminListener());

// เมื่อ User ลงทะเบียน
$user = ['id' => 123, 'email' => 'user@example.com', 'name' => 'John'];
$dispatcher->dispatch(new UserRegisteredEvent($user));
```

---

### 10. Strategy Pattern

Strategy Pattern กำหนด Algorithm Family และทำให้ Interchangeable

```php
<?php

interface SortingStrategy
{
    public function sort(array $data): array;
}

class BubbleSortStrategy implements SortingStrategy
{
    public function sort(array $data): array
    {
        $n = count($data);
        for ($i = 0; $i < $n - 1; $i++) {
            for ($j = 0; $j < $n - $i - 1; $j++) {
                if ($data[$j] > $data[$j + 1]) {
                    [$data[$j], $data[$j + 1]] = [$data[$j + 1], $data[$j]];
                }
            }
        }
        return $data;
    }
}

class QuickSortStrategy implements SortingStrategy
{
    public function sort(array $data): array
    {
        if (count($data) <= 1) return $data;
        
        $pivot = $data[0];
        $left = $right = [];
        
        for ($i = 1; $i < count($data); $i++) {
            if ($data[$i] <= $pivot) {
                $left[] = $data[$i];
            } else {
                $right[] = $data[$i];
            }
        }
        
        return array_merge($this->sort($left), [$pivot], $this->sort($right));
    }
}

class PhpNativeSortStrategy implements SortingStrategy
{
    public function sort(array $data): array
    {
        sort($data);
        return $data;
    }
}

class Sorter
{
    public function __construct(private SortingStrategy $strategy) {}
    
    public function setStrategy(SortingStrategy $strategy): void
    {
        $this->strategy = $strategy;
    }
    
    public function sort(array $data): array
    {
        return $this->strategy->sort($data);
    }
}

// Practical: Pricing Strategy
interface PricingStrategy
{
    public function calculate(float $basePrice, array $context): float;
}

class RegularPricingStrategy implements PricingStrategy
{
    public function calculate(float $basePrice, array $context): float
    {
        return $basePrice;
    }
}

class MemberDiscountStrategy implements PricingStrategy
{
    public function calculate(float $basePrice, array $context): float
    {
        $discount = match($context['tier'] ?? 'bronze') {
            'gold' => 0.15,
            'silver' => 0.10,
            'bronze' => 0.05,
            default => 0
        };
        
        return $basePrice * (1 - $discount);
    }
}

class SeasonalPricingStrategy implements PricingStrategy
{
    public function calculate(float $basePrice, array $context): float
    {
        $month = (int)date('n');
        
        // ลดราคา 20% ในช่วง New Year / Songkran / Year End
        if (in_array($month, [1, 4, 12])) {
            return $basePrice * 0.8;
        }
        
        return $basePrice;
    }
}

class BulkPricingStrategy implements PricingStrategy
{
    public function calculate(float $basePrice, array $context): float
    {
        $quantity = $context['quantity'] ?? 1;
        
        $discount = match(true) {
            $quantity >= 100 => 0.20,
            $quantity >= 50 => 0.15,
            $quantity >= 20 => 0.10,
            $quantity >= 10 => 0.05,
            default => 0
        };
        
        return $basePrice * (1 - $discount);
    }
}
```

---

### 11. Command Pattern

Command Pattern แปลง Request เป็น Object ที่ประกอบด้วยข้อมูลทั้งหมดของ Request

```php
<?php

interface Command
{
    public function execute(): void;
    public function undo(): void;
}

class TextEditor
{
    private string $content = '';
    private array $history = [];
    
    public function getContent(): string
    {
        return $this->content;
    }
    
    public function setContent(string $content): void
    {
        $this->content = $content;
    }
    
    public function execute(Command $command): void
    {
        $command->execute();
        $this->history[] = $command;
    }
    
    public function undo(): void
    {
        if (empty($this->history)) return;
        
        $command = array_pop($this->history);
        $command->undo();
    }
    
    public function undoAll(): void
    {
        while (!empty($this->history)) {
            $this->undo();
        }
    }
}

class InsertTextCommand implements Command
{
    private string $previousContent;
    
    public function __construct(
        private TextEditor $editor,
        private string $text,
        private int $position
    ) {}
    
    public function execute(): void
    {
        $this->previousContent = $this->editor->getContent();
        $content = $this->editor->getContent();
        $newContent = substr($content, 0, $this->position)
            . $this->text
            . substr($content, $this->position);
        $this->editor->setContent($newContent);
    }
    
    public function undo(): void
    {
        $this->editor->setContent($this->previousContent);
    }
}

class DeleteTextCommand implements Command
{
    private string $previousContent;
    
    public function __construct(
        private TextEditor $editor,
        private int $start,
        private int $length
    ) {}
    
    public function execute(): void
    {
        $this->previousContent = $this->editor->getContent();
        $content = $this->editor->getContent();
        $newContent = substr($content, 0, $this->start)
            . substr($content, $this->start + $this->length);
        $this->editor->setContent($newContent);
    }
    
    public function undo(): void
    {
        $this->editor->setContent($this->previousContent);
    }
}

// การใช้งาน
$editor = new TextEditor();

$editor->execute(new InsertTextCommand($editor, 'Hello', 0));
echo $editor->getContent(); // "Hello"

$editor->execute(new InsertTextCommand($editor, ' World', 5));
echo $editor->getContent(); // "Hello World"

$editor->undo();
echo $editor->getContent(); // "Hello"

$editor->undo();
echo $editor->getContent(); // ""
```

---

### 12. Template Method Pattern

Template Method กำหนด Skeleton ของ Algorithm ใน Base Class และให้ Subclass Override บางขั้นตอน

```php
<?php

abstract class ReportGenerator
{
    // Template Method
    final public function generate(array $data): string
    {
        $this->validateData($data);
        $processedData = $this->processData($data);
        $header = $this->generateHeader();
        $body = $this->generateBody($processedData);
        $footer = $this->generateFooter();
        
        return $this->assemble($header, $body, $footer);
    }
    
    protected function validateData(array $data): void
    {
        if (empty($data)) {
            throw new \InvalidArgumentException("Data cannot be empty");
        }
    }
    
    protected function processData(array $data): array
    {
        return $data; // Default: no processing
    }
    
    abstract protected function generateHeader(): string;
    abstract protected function generateBody(array $data): string;
    
    protected function generateFooter(): string
    {
        return ''; // Default: no footer
    }
    
    private function assemble(string $header, string $body, string $footer): string
    {
        return $header . $body . $footer;
    }
}

class HtmlReportGenerator extends ReportGenerator
{
    protected function generateHeader(): string
    {
        return "<!DOCTYPE html><html><body><h1>Report</h1>";
    }
    
    protected function generateBody(array $data): string
    {
        $rows = '';
        foreach ($data as $item) {
            $rows .= "<tr><td>{$item['name']}</td><td>{$item['value']}</td></tr>";
        }
        return "<table>{$rows}</table>";
    }
    
    protected function generateFooter(): string
    {
        return "<p>Generated: " . date('Y-m-d H:i:s') . "</p></body></html>";
    }
}

class CsvReportGenerator extends ReportGenerator
{
    protected function generateHeader(): string
    {
        return "Name,Value\n";
    }
    
    protected function generateBody(array $data): string
    {
        $lines = [];
        foreach ($data as $item) {
            $lines[] = "\"{$item['name']}\",\"{$item['value']}\"";
        }
        return implode("\n", $lines);
    }
}

// การใช้งาน
$data = [
    ['name' => 'Product A', 'value' => 1500],
    ['name' => 'Product B', 'value' => 2300],
];

$htmlReport = new HtmlReportGenerator();
$csvReport = new CsvReportGenerator();

echo $htmlReport->generate($data);
echo $csvReport->generate($data);
```

---

## PHP-Specific Patterns

### Magic Methods Pattern

```php
<?php

class DynamicModel
{
    private array $attributes = [];
    private array $dirty = [];
    
    public function __set(string $name, mixed $value): void
    {
        if (!isset($this->attributes[$name]) || $this->attributes[$name] !== $value) {
            $this->dirty[] = $name;
        }
        $this->attributes[$name] = $value;
    }
    
    public function __get(string $name): mixed
    {
        return $this->attributes[$name] ?? null;
    }
    
    public function __isset(string $name): bool
    {
        return isset($this->attributes[$name]);
    }
    
    public function __unset(string $name): void
    {
        unset($this->attributes[$name]);
    }
    
    public function isDirty(string $field = null): bool
    {
        if ($field === null) {
            return !empty($this->dirty);
        }
        return in_array($field, $this->dirty);
    }
    
    public function toArray(): array
    {
        return $this->attributes;
    }
}
```

### Fluent Interface Pattern

```php
<?php

class EmailBuilder
{
    private array $to = [];
    private array $cc = [];
    private string $subject = '';
    private string $body = '';
    private array $attachments = [];
    
    public function to(string ...$addresses): self
    {
        $this->to = array_merge($this->to, $addresses);
        return $this;
    }
    
    public function cc(string ...$addresses): self
    {
        $this->cc = array_merge($this->cc, $addresses);
        return $this;
    }
    
    public function subject(string $subject): self
    {
        $this->subject = $subject;
        return $this;
    }
    
    public function body(string $body): self
    {
        $this->body = $body;
        return $this;
    }
    
    public function attach(string $filePath): self
    {
        $this->attachments[] = $filePath;
        return $this;
    }
    
    public function send(): bool
    {
        // ส่ง Email
        echo "Sending email to: " . implode(', ', $this->to) . "\n";
        echo "Subject: {$this->subject}\n";
        return true;
    }
}

// การใช้งาน
(new EmailBuilder())
    ->to('manager@company.com', 'hr@company.com')
    ->cc('ceo@company.com')
    ->subject('Monthly Report - ' . date('F Y'))
    ->body('Please find the monthly report attached.')
    ->attach('/reports/monthly-' . date('Y-m') . '.pdf')
    ->send();
```

---

## Workshop: E-commerce System

สร้างระบบ E-commerce ที่ใช้หลาย Design Patterns รวมกัน

```php
<?php

// === Product Catalog (Factory + Prototype) ===

interface Product
{
    public function getId(): int;
    public function getName(): string;
    public function getPrice(): float;
    public function clone(): static;
}

class PhysicalProduct implements Product
{
    public function __construct(
        private int $id,
        private string $name,
        private float $price,
        private float $weight,
        private array $dimensions
    ) {}
    
    public function getId(): int { return $this->id; }
    public function getName(): string { return $this->name; }
    public function getPrice(): float { return $this->price; }
    public function getWeight(): float { return $this->weight; }
    
    public function clone(): static
    {
        return clone $this;
    }
}

class DigitalProduct implements Product
{
    public function __construct(
        private int $id,
        private string $name,
        private float $price,
        private string $downloadUrl
    ) {}
    
    public function getId(): int { return $this->id; }
    public function getName(): string { return $this->name; }
    public function getPrice(): float { return $this->price; }
    public function getDownloadUrl(): string { return $this->downloadUrl; }
    
    public function clone(): static
    {
        return clone $this;
    }
}

// === Shopping Cart (Observer + Command) ===

class CartItem
{
    public function __construct(
        public readonly Product $product,
        public int $quantity
    ) {}
    
    public function getTotal(): float
    {
        return $this->product->getPrice() * $this->quantity;
    }
}

interface CartObserver
{
    public function onItemAdded(CartItem $item): void;
    public function onItemRemoved(CartItem $item): void;
    public function onCartCleared(): void;
}

class Cart
{
    private array $items = [];
    private array $observers = [];
    
    public function addObserver(CartObserver $observer): void
    {
        $this->observers[] = $observer;
    }
    
    public function addItem(Product $product, int $quantity = 1): void
    {
        $productId = $product->getId();
        
        if (isset($this->items[$productId])) {
            $this->items[$productId]->quantity += $quantity;
        } else {
            $this->items[$productId] = new CartItem($product, $quantity);
        }
        
        foreach ($this->observers as $observer) {
            $observer->onItemAdded($this->items[$productId]);
        }
    }
    
    public function getItems(): array
    {
        return $this->items;
    }
    
    public function getTotal(): float
    {
        return array_sum(array_map(fn($item) => $item->getTotal(), $this->items));
    }
}

class InventoryCheckObserver implements CartObserver
{
    public function onItemAdded(CartItem $item): void
    {
        echo "Checking inventory for: {$item->product->getName()}\n";
    }
    
    public function onItemRemoved(CartItem $item): void {}
    public function onCartCleared(): void {}
}

class PriceCalculatorObserver implements CartObserver
{
    public function onItemAdded(CartItem $item): void
    {
        echo "Recalculating cart total...\n";
    }
    
    public function onItemRemoved(CartItem $item): void
    {
        echo "Recalculating cart total...\n";
    }
    
    public function onCartCleared(): void
    {
        echo "Cart cleared, total is now 0\n";
    }
}

// === Checkout Process (Facade + Strategy) ===

class CheckoutFacade
{
    private Cart $cart;
    private PricingStrategy $pricingStrategy;
    
    public function __construct(Cart $cart, PricingStrategy $pricingStrategy)
    {
        $this->cart = $cart;
        $this->pricingStrategy = $pricingStrategy;
    }
    
    public function checkout(array $customer, array $paymentDetails): array
    {
        $items = $this->cart->getItems();
        $subtotal = $this->cart->getTotal();
        
        // Apply pricing strategy
        $finalPrice = $this->pricingStrategy->calculate($subtotal, [
            'tier' => $customer['tier'] ?? 'bronze',
            'quantity' => count($items)
        ]);
        
        return [
            'order_id' => 'ORD-' . uniqid(),
            'customer' => $customer,
            'items' => array_values(array_map(fn($item) => [
                'product' => $item->product->getName(),
                'quantity' => $item->quantity,
                'price' => $item->product->getPrice(),
                'total' => $item->getTotal()
            ], $items)),
            'subtotal' => $subtotal,
            'discount' => $subtotal - $finalPrice,
            'total' => $finalPrice,
            'status' => 'confirmed'
        ];
    }
}

// === การใช้งานทั้งหมด ===

// สร้าง Cart พร้อม Observers
$cart = new Cart();
$cart->addObserver(new InventoryCheckObserver());
$cart->addObserver(new PriceCalculatorObserver());

// เพิ่มสินค้า
$laptop = new PhysicalProduct(1, 'MacBook Pro', 89900.0, 1.4, ['30cm', '20cm', '1.5cm']);
$software = new DigitalProduct(2, 'Adobe Photoshop', 1900.0, 'https://download.adobe.com/...');

$cart->addItem($laptop, 1);
$cart->addItem($software, 1);

echo "Cart total: " . number_format($cart->getTotal(), 2) . " THB\n";

// Checkout ด้วย Member Discount
$checkout = new CheckoutFacade($cart, new MemberDiscountStrategy());

$order = $checkout->checkout(
    customer: ['id' => 42, 'name' => 'สมชาย', 'tier' => 'gold'],
    paymentDetails: ['method' => 'credit_card', 'token' => 'tok_visa']
);

echo "\nOrder confirmed: {$order['order_id']}\n";
echo "Subtotal: " . number_format($order['subtotal'], 2) . " THB\n";
echo "Discount: " . number_format($order['discount'], 2) . " THB\n";
echo "Total: " . number_format($order['total'], 2) . " THB\n";
```

---

## สรุป

| Pattern | ประเภท | ใช้เมื่อ |
|---------|--------|---------|
| Singleton | Creational | ต้องการ Instance เดียว (DB, Config) |
| Factory | Creational | สร้าง Object ที่หลากหลายชนิด |
| Builder | Creational | Object ที่ซับซ้อน มี Optional params เยอะ |
| Prototype | Creational | Copy Object ที่ Cost สูงในการสร้าง |
| Adapter | Structural | ใช้ Third-party Library ที่ Interface ไม่ตรง |
| Decorator | Structural | เพิ่ม Behavior แบบ Dynamic |
| Facade | Structural | ซ่อน Complexity ของ Subsystem |
| Proxy | Structural | Control Access, Caching, Logging |
| Observer | Behavioral | Event-driven, Loose coupling |
| Strategy | Behavioral | Algorithm ที่เปลี่ยนได้ตาม Context |
| Command | Behavioral | Undo/Redo, Queue operations |
| Template Method | Behavioral | Algorithm skeleton ที่ Subclass Override ได้ |

---

*Design Patterns ไม่ใช่ Silver Bullet - ใช้เมื่อเหมาะสม อย่า Overengineer*
