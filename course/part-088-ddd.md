# Part 88: Domain-Driven Design (DDD) ใน PHP

## บทนำ

Domain-Driven Design (DDD) คือแนวทางการพัฒนาซอฟต์แวร์ที่เน้นการสร้าง Model ที่สะท้อน Business Domain อย่างแม่นยำ พัฒนาโดย Eric Evans ในหนังสือ "Domain-Driven Design: Tackling Complexity in the Heart of Software" (2003)

**แนวคิดหลัก**:
- Code ควรสื่อสาร Business Logic อย่างชัดเจน
- ใช้ Ubiquitous Language (ภาษากลางระหว่าง Developer และ Domain Expert)
- แยก Business Logic ออกจาก Technical Concerns

---

## Building Blocks ของ DDD

### 1. Entities

Entity คือ Object ที่มี Identity เป็นของตัวเอง และ Identity ไม่เปลี่ยนแปลงตลอดชีวิต

```php
<?php

namespace App\Domain\Order;

use App\Domain\Shared\AggregateRoot;
use App\Domain\Shared\DomainEvent;

class OrderId
{
    public function __construct(private readonly string $value)
    {
        if (empty($value)) {
            throw new \InvalidArgumentException("OrderId cannot be empty");
        }
    }
    
    public static function generate(): self
    {
        return new self('ORD-' . strtoupper(uniqid()));
    }
    
    public static function fromString(string $value): self
    {
        return new self($value);
    }
    
    public function value(): string
    {
        return $this->value;
    }
    
    public function equals(OrderId $other): bool
    {
        return $this->value === $other->value;
    }
    
    public function __toString(): string
    {
        return $this->value;
    }
}

class Order // Entity
{
    private OrderId $id;
    private array $lineItems = [];
    private OrderStatus $status;
    private \DateTimeImmutable $createdAt;
    private ?\DateTimeImmutable $confirmedAt = null;
    private array $domainEvents = [];
    
    public function __construct(
        OrderId $id,
        private CustomerId $customerId,
        private Money $subtotal
    ) {
        $this->id = $id;
        $this->status = OrderStatus::DRAFT;
        $this->createdAt = new \DateTimeImmutable();
    }
    
    public static function create(CustomerId $customerId): self
    {
        $order = new self(
            id: OrderId::generate(),
            customerId: $customerId,
            subtotal: Money::zero('THB')
        );
        
        $order->recordEvent(new OrderCreatedEvent($order->id, $customerId));
        
        return $order;
    }
    
    public function addItem(ProductId $productId, Quantity $quantity, Money $unitPrice): void
    {
        $this->ensureNotConfirmed();
        
        $lineItem = new OrderLineItem($productId, $quantity, $unitPrice);
        $this->lineItems[] = $lineItem;
        $this->recalculateSubtotal();
        
        $this->recordEvent(new OrderItemAddedEvent($this->id, $productId, $quantity));
    }
    
    public function confirm(): void
    {
        if ($this->status !== OrderStatus::DRAFT) {
            throw new \DomainException("Only draft orders can be confirmed");
        }
        
        if (empty($this->lineItems)) {
            throw new \DomainException("Cannot confirm empty order");
        }
        
        $this->status = OrderStatus::CONFIRMED;
        $this->confirmedAt = new \DateTimeImmutable();
        
        $this->recordEvent(new OrderConfirmedEvent($this->id, $this->subtotal));
    }
    
    public function cancel(string $reason): void
    {
        if ($this->status === OrderStatus::SHIPPED) {
            throw new \DomainException("Cannot cancel a shipped order");
        }
        
        if ($this->status === OrderStatus::CANCELLED) {
            throw new \DomainException("Order is already cancelled");
        }
        
        $this->status = OrderStatus::CANCELLED;
        $this->recordEvent(new OrderCancelledEvent($this->id, $reason));
    }
    
    private function ensureNotConfirmed(): void
    {
        if ($this->status !== OrderStatus::DRAFT) {
            throw new \DomainException("Cannot modify a confirmed order");
        }
    }
    
    private function recalculateSubtotal(): void
    {
        $this->subtotal = array_reduce(
            $this->lineItems,
            fn(Money $carry, OrderLineItem $item) => $carry->add($item->getTotal()),
            Money::zero('THB')
        );
    }
    
    private function recordEvent(DomainEvent $event): void
    {
        $this->domainEvents[] = $event;
    }
    
    public function pullDomainEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }
    
    // Getters
    public function getId(): OrderId { return $this->id; }
    public function getStatus(): OrderStatus { return $this->status; }
    public function getSubtotal(): Money { return $this->subtotal; }
    public function getLineItems(): array { return $this->lineItems; }
    
    // Entity identity based on ID, not data
    public function equals(Order $other): bool
    {
        return $this->id->equals($other->id);
    }
}
```

---

### 2. Value Objects

Value Object คือ Object ที่กำหนดด้วยค่าของมัน ไม่มี Identity และ Immutable

```php
<?php

namespace App\Domain\Shared;

final class Money
{
    private function __construct(
        private readonly int $amount, // ใช้ int เพื่อหลีกเลี่ยง floating point issues
        private readonly string $currency
    ) {
        if ($amount < 0) {
            throw new \InvalidArgumentException("Amount cannot be negative");
        }
        
        if (!in_array($currency, ['THB', 'USD', 'EUR', 'JPY'])) {
            throw new \InvalidArgumentException("Unsupported currency: {$currency}");
        }
    }
    
    public static function of(float $amount, string $currency): self
    {
        return new self((int)round($amount * 100), $currency);
    }
    
    public static function zero(string $currency): self
    {
        return new self(0, $currency);
    }
    
    public function add(Money $other): self
    {
        $this->ensureSameCurrency($other);
        return new self($this->amount + $other->amount, $this->currency);
    }
    
    public function subtract(Money $other): self
    {
        $this->ensureSameCurrency($other);
        $result = $this->amount - $other->amount;
        
        if ($result < 0) {
            throw new \DomainException("Insufficient funds");
        }
        
        return new self($result, $this->currency);
    }
    
    public function multiply(float $factor): self
    {
        return new self((int)round($this->amount * $factor), $this->currency);
    }
    
    public function percentage(float $percent): self
    {
        return $this->multiply($percent / 100);
    }
    
    public function isGreaterThan(Money $other): bool
    {
        $this->ensureSameCurrency($other);
        return $this->amount > $other->amount;
    }
    
    public function isLessThan(Money $other): bool
    {
        $this->ensureSameCurrency($other);
        return $this->amount < $other->amount;
    }
    
    public function equals(Money $other): bool
    {
        return $this->amount === $other->amount
            && $this->currency === $other->currency;
    }
    
    public function getAmount(): float
    {
        return $this->amount / 100;
    }
    
    public function getCurrency(): string
    {
        return $this->currency;
    }
    
    public function format(): string
    {
        return number_format($this->getAmount(), 2) . ' ' . $this->currency;
    }
    
    private function ensureSameCurrency(Money $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new \DomainException(
                "Cannot operate on different currencies: {$this->currency} and {$other->currency}"
            );
        }
    }
}

// Email Value Object
final class Email
{
    private readonly string $value;
    
    public function __construct(string $email)
    {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("Invalid email: {$email}");
        }
        $this->value = strtolower(trim($email));
    }
    
    public function value(): string { return $this->value; }
    
    public function getDomain(): string
    {
        return substr($this->value, strpos($this->value, '@') + 1);
    }
    
    public function equals(Email $other): bool
    {
        return $this->value === $other->value;
    }
    
    public function __toString(): string
    {
        return $this->value;
    }
}

// Address Value Object
final class Address
{
    public function __construct(
        private readonly string $street,
        private readonly string $city,
        private readonly string $postalCode,
        private readonly string $country
    ) {
        if (empty(trim($street))) throw new \InvalidArgumentException("Street cannot be empty");
        if (empty(trim($city))) throw new \InvalidArgumentException("City cannot be empty");
        if (!preg_match('/^\d{5}$/', $postalCode)) {
            throw new \InvalidArgumentException("Invalid postal code: {$postalCode}");
        }
    }
    
    public function equals(Address $other): bool
    {
        return $this->street === $other->street
            && $this->city === $other->city
            && $this->postalCode === $other->postalCode
            && $this->country === $other->country;
    }
    
    public function format(): string
    {
        return "{$this->street}, {$this->city} {$this->postalCode}, {$this->country}";
    }
    
    public function withStreet(string $street): self
    {
        return new self($street, $this->city, $this->postalCode, $this->country);
    }
}

// Quantity Value Object
final class Quantity
{
    public function __construct(private readonly int $value)
    {
        if ($value <= 0) {
            throw new \InvalidArgumentException("Quantity must be positive, got: {$value}");
        }
    }
    
    public function value(): int { return $this->value; }
    
    public function add(Quantity $other): self
    {
        return new self($this->value + $other->value);
    }
    
    public function subtract(Quantity $other): self
    {
        return new self($this->value - $other->value);
    }
    
    public function isGreaterThan(Quantity $other): bool
    {
        return $this->value > $other->value;
    }
}
```

---

### 3. Aggregates

Aggregate คือกลุ่มของ Entities และ Value Objects ที่เกี่ยวข้องกัน มี Aggregate Root ที่ควบคุมการเข้าถึง

```php
<?php

namespace App\Domain\Product;

class Product // Aggregate Root
{
    private array $variants = [];
    private array $reviews = [];
    private array $domainEvents = [];
    
    private function __construct(
        private ProductId $id,
        private ProductName $name,
        private Money $basePrice,
        private ProductStatus $status,
        private CategoryId $categoryId
    ) {}
    
    public static function create(
        ProductName $name,
        Money $basePrice,
        CategoryId $categoryId
    ): self {
        $product = new self(
            id: ProductId::generate(),
            name: $name,
            basePrice: $basePrice,
            status: ProductStatus::DRAFT,
            categoryId: $categoryId
        );
        
        $product->recordEvent(new ProductCreatedEvent($product->id, $name, $basePrice));
        
        return $product;
    }
    
    public function addVariant(ProductVariant $variant): void
    {
        $this->ensureActive();
        
        // Check for duplicate variant
        foreach ($this->variants as $existing) {
            if ($existing->hasSameAttributes($variant)) {
                throw new \DomainException("Duplicate variant attributes");
            }
        }
        
        $this->variants[] = $variant;
    }
    
    public function addReview(
        CustomerId $customerId,
        Rating $rating,
        string $comment
    ): void {
        // ตรวจสอบว่า Customer ซื้อสินค้านี้ไปแล้ว (Domain Rule)
        // ตรวจสอบว่ายังไม่เคย Review
        foreach ($this->reviews as $review) {
            if ($review->getCustomerId()->equals($customerId)) {
                throw new \DomainException("Customer has already reviewed this product");
            }
        }
        
        $review = new ProductReview($customerId, $rating, $comment);
        $this->reviews[] = $review;
        
        $this->recordEvent(new ProductReviewedEvent(
            $this->id,
            $customerId,
            $rating
        ));
    }
    
    public function publish(): void
    {
        if ($this->status !== ProductStatus::DRAFT) {
            throw new \DomainException("Only draft products can be published");
        }
        
        if (empty($this->variants)) {
            throw new \DomainException("Product must have at least one variant");
        }
        
        $this->status = ProductStatus::ACTIVE;
        $this->recordEvent(new ProductPublishedEvent($this->id));
    }
    
    public function discontinue(string $reason): void
    {
        if ($this->status === ProductStatus::DISCONTINUED) {
            throw new \DomainException("Product is already discontinued");
        }
        
        $this->status = ProductStatus::DISCONTINUED;
        $this->recordEvent(new ProductDiscontinuedEvent($this->id, $reason));
    }
    
    public function getAverageRating(): ?float
    {
        if (empty($this->reviews)) return null;
        
        $total = array_sum(array_map(
            fn(ProductReview $r) => $r->getRating()->value(),
            $this->reviews
        ));
        
        return $total / count($this->reviews);
    }
    
    private function ensureActive(): void
    {
        if ($this->status !== ProductStatus::ACTIVE) {
            throw new \DomainException("Product is not active");
        }
    }
    
    private function recordEvent(\DomainEvent $event): void
    {
        $this->domainEvents[] = $event;
    }
    
    public function pullDomainEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }
}

// Product Variant - Part of Product Aggregate
class ProductVariant
{
    private Stock $stock;
    
    public function __construct(
        private ProductVariantId $id,
        private array $attributes, // ['color' => 'red', 'size' => 'L']
        private Money $price,
        int $initialStock
    ) {
        $this->stock = new Stock($initialStock);
    }
    
    public function adjustStock(int $delta, string $reason): void
    {
        if ($delta < 0 && $this->stock->quantity() < abs($delta)) {
            throw new \DomainException("Insufficient stock");
        }
        
        $this->stock = $this->stock->adjust($delta);
    }
    
    public function isAvailable(): bool
    {
        return $this->stock->isAvailable();
    }
    
    public function hasSameAttributes(ProductVariant $other): bool
    {
        return $this->attributes === $other->attributes;
    }
}
```

---

### 4. Domain Events

Domain Events แทน "สิ่งที่เกิดขึ้นใน Domain" ที่ส่วนอื่นๆ อาจสนใจ

```php
<?php

namespace App\Domain\Shared;

abstract class DomainEvent
{
    private readonly string $eventId;
    private readonly \DateTimeImmutable $occurredOn;
    
    public function __construct()
    {
        $this->eventId = uniqid('evt_', true);
        $this->occurredOn = new \DateTimeImmutable();
    }
    
    public function eventId(): string { return $this->eventId; }
    public function occurredOn(): \DateTimeImmutable { return $this->occurredOn; }
    abstract public function eventName(): string;
}

// Domain Events
class OrderConfirmedEvent extends DomainEvent
{
    public function __construct(
        public readonly OrderId $orderId,
        public readonly Money $total
    ) {
        parent::__construct();
    }
    
    public function eventName(): string { return 'order.confirmed'; }
}

class UserRegisteredEvent extends DomainEvent
{
    public function __construct(
        public readonly UserId $userId,
        public readonly Email $email,
        public readonly string $name
    ) {
        parent::__construct();
    }
    
    public function eventName(): string { return 'user.registered'; }
}

class PaymentProcessedEvent extends DomainEvent
{
    public function __construct(
        public readonly OrderId $orderId,
        public readonly Money $amount,
        public readonly string $transactionId
    ) {
        parent::__construct();
    }
    
    public function eventName(): string { return 'payment.processed'; }
}

// Domain Event Dispatcher
interface DomainEventDispatcher
{
    public function dispatch(DomainEvent $event): void;
}

class SynchronousDomainEventDispatcher implements DomainEventDispatcher
{
    private array $handlers = [];
    
    public function registerHandler(string $eventClass, callable $handler): void
    {
        $this->handlers[$eventClass][] = $handler;
    }
    
    public function dispatch(DomainEvent $event): void
    {
        $handlers = $this->handlers[get_class($event)] ?? [];
        
        foreach ($handlers as $handler) {
            $handler($event);
        }
    }
}

// Event Handlers
class SendOrderConfirmationEmailHandler
{
    public function __construct(private MailService $mail) {}
    
    public function __invoke(OrderConfirmedEvent $event): void
    {
        // Send confirmation email
        echo "Sending order confirmation for order: {$event->orderId}\n";
    }
}

class UpdateInventoryHandler
{
    public function __construct(private InventoryService $inventory) {}
    
    public function __invoke(OrderConfirmedEvent $event): void
    {
        // Update inventory
        echo "Updating inventory for order: {$event->orderId}\n";
    }
}
```

---

### 5. Repositories

Repository ทำหน้าที่เป็น Collection Interface สำหรับ Aggregates

```php
<?php

namespace App\Domain\Order;

interface OrderRepository
{
    public function findById(OrderId $id): ?Order;
    public function findByCustomerId(CustomerId $customerId): array;
    public function findPendingOrders(): array;
    public function save(Order $order): void;
    public function remove(Order $order): void;
    public function nextIdentity(): OrderId;
}

namespace App\Infrastructure\Persistence;

use App\Domain\Order\{Order, OrderId, OrderRepository};

class EloquentOrderRepository implements OrderRepository
{
    public function findById(OrderId $id): ?Order
    {
        $model = \App\Models\Order::find($id->value());
        
        if (!$model) return null;
        
        return $this->toDomainEntity($model);
    }
    
    public function findByCustomerId(CustomerId $customerId): array
    {
        return \App\Models\Order::where('customer_id', $customerId->value())
            ->get()
            ->map(fn($m) => $this->toDomainEntity($m))
            ->toArray();
    }
    
    public function findPendingOrders(): array
    {
        return \App\Models\Order::where('status', 'pending')
            ->orderBy('created_at')
            ->get()
            ->map(fn($m) => $this->toDomainEntity($m))
            ->toArray();
    }
    
    public function save(Order $order): void
    {
        $data = $this->toPersistence($order);
        
        \App\Models\Order::updateOrCreate(
            ['id' => $order->getId()->value()],
            $data
        );
        
        // Dispatch Domain Events
        foreach ($order->pullDomainEvents() as $event) {
            event($event);
        }
    }
    
    public function remove(Order $order): void
    {
        \App\Models\Order::where('id', $order->getId()->value())->delete();
    }
    
    public function nextIdentity(): OrderId
    {
        return OrderId::generate();
    }
    
    private function toDomainEntity(\App\Models\Order $model): Order
    {
        // Reconstitute domain object from persistence
        // ใช้ Reflection หรือ Named Constructor
        return Order::reconstitute([
            'id' => $model->id,
            'customer_id' => $model->customer_id,
            'status' => $model->status,
            'subtotal' => $model->subtotal,
            'currency' => $model->currency,
            'created_at' => $model->created_at,
        ]);
    }
    
    private function toPersistence(Order $order): array
    {
        return [
            'id' => $order->getId()->value(),
            'customer_id' => $order->getCustomerId()->value(),
            'status' => $order->getStatus()->value,
            'subtotal' => $order->getSubtotal()->getAmount(),
            'currency' => $order->getSubtotal()->getCurrency(),
        ];
    }
}
```

---

### 6. Application Services

Application Services จัดการ Use Cases และประสานงานระหว่าง Domain Objects

```php
<?php

namespace App\Application\Order;

// Command (Input DTO)
class PlaceOrderCommand
{
    public function __construct(
        public readonly string $customerId,
        public readonly array $items, // [['product_id' => '...', 'quantity' => 1], ...]
        public readonly string $currency = 'THB'
    ) {}
}

// Result (Output DTO)
class PlaceOrderResult
{
    public function __construct(
        public readonly string $orderId,
        public readonly float $total,
        public readonly string $status
    ) {}
}

// Application Service
class PlaceOrderService
{
    public function __construct(
        private OrderRepository $orderRepository,
        private ProductRepository $productRepository,
        private CustomerRepository $customerRepository,
        private DomainEventDispatcher $eventDispatcher
    ) {}
    
    public function execute(PlaceOrderCommand $command): PlaceOrderResult
    {
        // 1. Load Customer
        $customer = $this->customerRepository->findById(
            CustomerId::fromString($command->customerId)
        );
        
        if (!$customer) {
            throw new \DomainException("Customer not found: {$command->customerId}");
        }
        
        // 2. Create Order
        $order = Order::create($customer->getId());
        
        // 3. Add Items
        foreach ($command->items as $item) {
            $product = $this->productRepository->findById(
                ProductId::fromString($item['product_id'])
            );
            
            if (!$product) {
                throw new \DomainException("Product not found: {$item['product_id']}");
            }
            
            $variant = $product->getVariant($item['variant_id'] ?? null);
            
            if (!$variant->isAvailable()) {
                throw new \DomainException("Product variant is not available");
            }
            
            $order->addItem(
                productId: $product->getId(),
                quantity: new Quantity($item['quantity']),
                unitPrice: $variant->getPrice()
            );
        }
        
        // 4. Confirm Order
        $order->confirm();
        
        // 5. Save
        $this->orderRepository->save($order);
        
        // 6. Dispatch Events
        foreach ($order->pullDomainEvents() as $event) {
            $this->eventDispatcher->dispatch($event);
        }
        
        return new PlaceOrderResult(
            orderId: $order->getId()->value(),
            total: $order->getSubtotal()->getAmount(),
            status: $order->getStatus()->value
        );
    }
}

// Cancel Order Service
class CancelOrderService
{
    public function __construct(
        private OrderRepository $orderRepository,
        private DomainEventDispatcher $eventDispatcher
    ) {}
    
    public function execute(string $orderId, string $reason): void
    {
        $order = $this->orderRepository->findById(OrderId::fromString($orderId));
        
        if (!$order) {
            throw new \DomainException("Order not found: {$orderId}");
        }
        
        $order->cancel($reason);
        
        $this->orderRepository->save($order);
        
        foreach ($order->pullDomainEvents() as $event) {
            $this->eventDispatcher->dispatch($event);
        }
    }
}
```

---

## Workshop: E-commerce DDD Implementation

```php
<?php

// === Bounded Context: Product Catalog ===

namespace App\Domain\Catalog;

class Product
{
    private function __construct(
        private readonly ProductId $id,
        private ProductName $name,
        private ProductDescription $description,
        private Money $price,
        private CategoryId $categoryId,
        private array $images = [],
        private array $specifications = [],
        private ProductStatus $status = ProductStatus::DRAFT
    ) {}
    
    public static function draft(
        ProductName $name,
        ProductDescription $description,
        Money $price,
        CategoryId $categoryId
    ): self {
        return new self(
            id: ProductId::generate(),
            name: $name,
            description: $description,
            price: $price,
            categoryId: $categoryId
        );
    }
    
    public function addImage(ProductImage $image): void
    {
        if (count($this->images) >= 10) {
            throw new \DomainException("Maximum 10 images per product");
        }
        $this->images[] = $image;
    }
    
    public function updatePrice(Money $newPrice, string $reason): void
    {
        if ($newPrice->equals($this->price)) return;
        
        $oldPrice = $this->price;
        $this->price = $newPrice;
        
        $this->recordEvent(new ProductPriceChangedEvent(
            $this->id,
            $oldPrice,
            $newPrice,
            $reason
        ));
    }
}

// === Bounded Context: Inventory ===

namespace App\Domain\Inventory;

class StockItem
{
    private function __construct(
        private readonly StockItemId $id,
        private readonly ProductVariantId $variantId,
        private int $quantity,
        private int $reservedQuantity = 0
    ) {}
    
    public function reserve(int $quantity, OrderId $orderId): void
    {
        $available = $this->quantity - $this->reservedQuantity;
        
        if ($quantity > $available) {
            throw new InsufficientStockException(
                "Insufficient stock. Available: {$available}, Requested: {$quantity}"
            );
        }
        
        $this->reservedQuantity += $quantity;
        
        $this->recordEvent(new StockReservedEvent(
            $this->variantId,
            $quantity,
            $orderId
        ));
    }
    
    public function release(int $quantity, OrderId $orderId): void
    {
        $this->reservedQuantity = max(0, $this->reservedQuantity - $quantity);
        
        $this->recordEvent(new StockReleasedEvent(
            $this->variantId,
            $quantity,
            $orderId
        ));
    }
    
    public function fulfill(int $quantity): void
    {
        if ($quantity > $this->reservedQuantity) {
            throw new \DomainException("Cannot fulfill more than reserved quantity");
        }
        
        $this->quantity -= $quantity;
        $this->reservedQuantity -= $quantity;
    }
    
    public function replenish(int $quantity): void
    {
        if ($quantity <= 0) {
            throw new \InvalidArgumentException("Replenish quantity must be positive");
        }
        
        $this->quantity += $quantity;
        
        $this->recordEvent(new StockReplenishedEvent($this->variantId, $quantity));
    }
    
    public function availableQuantity(): int
    {
        return $this->quantity - $this->reservedQuantity;
    }
    
    public function isLowStock(int $threshold = 10): bool
    {
        return $this->availableQuantity() <= $threshold;
    }
}

// === Bounded Context: Customer ===

namespace App\Domain\Customer;

class Customer
{
    private array $addresses = [];
    private \DateTimeImmutable $registeredAt;
    private CustomerTier $tier;
    private array $domainEvents = [];
    
    private function __construct(
        private readonly CustomerId $id,
        private CustomerName $name,
        private Email $email,
        private ?PhoneNumber $phone = null
    ) {
        $this->tier = CustomerTier::STANDARD;
        $this->registeredAt = new \DateTimeImmutable();
    }
    
    public static function register(
        CustomerName $name,
        Email $email,
        string $password
    ): self {
        $customer = new self(
            id: CustomerId::generate(),
            name: $name,
            email: $email
        );
        
        $customer->recordEvent(new CustomerRegisteredEvent(
            $customer->id,
            $email,
            $name
        ));
        
        return $customer;
    }
    
    public function addAddress(Address $address, bool $setAsDefault = false): void
    {
        if (count($this->addresses) >= 5) {
            throw new \DomainException("Maximum 5 addresses per customer");
        }
        
        $this->addresses[] = new CustomerAddress(
            address: $address,
            isDefault: $setAsDefault || empty($this->addresses)
        );
    }
    
    public function upgradeTier(CustomerTier $newTier): void
    {
        if (!$newTier->isHigherThan($this->tier)) {
            throw new \DomainException("Can only upgrade to higher tier");
        }
        
        $oldTier = $this->tier;
        $this->tier = $newTier;
        
        $this->recordEvent(new CustomerTierUpgradedEvent(
            $this->id,
            $oldTier,
            $newTier
        ));
    }
    
    public function getDiscountRate(): float
    {
        return match($this->tier) {
            CustomerTier::STANDARD => 0.0,
            CustomerTier::SILVER => 0.05,
            CustomerTier::GOLD => 0.10,
            CustomerTier::PLATINUM => 0.15,
        };
    }
}

// === Application Layer: Checkout Use Case ===

namespace App\Application\Checkout;

class CheckoutService
{
    public function __construct(
        private readonly CustomerRepository $customers,
        private readonly ProductRepository $products,
        private readonly OrderRepository $orders,
        private readonly StockItemRepository $stock,
        private readonly PaymentService $payment,
        private readonly DomainEventBus $eventBus
    ) {}
    
    public function checkout(CheckoutCommand $command): CheckoutResult
    {
        // 1. Load Customer
        $customer = $this->customers->findById(
            CustomerId::fromString($command->customerId)
        );
        
        // 2. Validate and Load Products
        $cartItems = [];
        foreach ($command->cartItems as $cartItem) {
            $product = $this->products->findById(
                ProductId::fromString($cartItem['product_id'])
            );
            
            $stockItem = $this->stock->findByVariantId(
                ProductVariantId::fromString($cartItem['variant_id'])
            );
            
            // Check stock availability
            if (!$stockItem->availableQuantity() >= $cartItem['quantity']) {
                throw new InsufficientStockException(
                    "Not enough stock for product: {$product->getName()}"
                );
            }
            
            $cartItems[] = [
                'product' => $product,
                'stock_item' => $stockItem,
                'quantity' => $cartItem['quantity']
            ];
        }
        
        // 3. Create Order
        $order = Order::create($customer->getId());
        
        foreach ($cartItems as $item) {
            $order->addItem(
                $item['product']->getId(),
                new Quantity($item['quantity']),
                $item['product']->getPrice()
            );
        }
        
        // Apply customer discount
        $discount = $customer->getDiscountRate();
        if ($discount > 0) {
            $order->applyDiscount(
                new PercentageDiscount($discount, "Customer tier discount")
            );
        }
        
        $order->confirm();
        
        // 4. Reserve Stock
        foreach ($cartItems as $item) {
            $item['stock_item']->reserve($item['quantity'], $order->getId());
            $this->stock->save($item['stock_item']);
        }
        
        // 5. Process Payment
        $paymentResult = $this->payment->charge(
            $order->getTotal(),
            $command->paymentMethod
        );
        
        if (!$paymentResult->isSuccessful()) {
            // Release stock if payment fails
            foreach ($cartItems as $item) {
                $item['stock_item']->release($item['quantity'], $order->getId());
                $this->stock->save($item['stock_item']);
            }
            
            throw new PaymentFailedException($paymentResult->getErrorMessage());
        }
        
        $order->markAsPaid($paymentResult->getTransactionId());
        
        // 6. Save Order
        $this->orders->save($order);
        
        // 7. Dispatch Events
        foreach ($order->pullDomainEvents() as $event) {
            $this->eventBus->dispatch($event);
        }
        
        return new CheckoutResult(
            orderId: $order->getId()->value(),
            total: $order->getTotal()->getAmount(),
            currency: $order->getTotal()->getCurrency(),
            transactionId: $paymentResult->getTransactionId()
        );
    }
}
```

---

## Ubiquitous Language

ตัวอย่าง Ubiquitous Language สำหรับ E-commerce:

```
Domain Terms:
- Order: คำสั่งซื้อที่ Customer ทำ
- Cart: ตะกร้าสินค้าก่อนยืนยัน
- Product: สินค้าที่จำหน่าย
- Variant: รูปแบบของ Product (สี, ขนาด)
- SKU: Stock Keeping Unit - รหัสจัดการสต็อก
- Fulfillment: กระบวนการจัดเตรียมและส่งสินค้า
- Backorder: สินค้าที่รับออร์เดอร์แต่ยังไม่มีสต็อก
- Return: การคืนสินค้า
- Refund: การคืนเงิน
- Chargeback: การขอเงินคืนผ่าน Credit Card
```

---

## สรุป DDD Tactical Patterns

| Pattern | บทบาท | ตัวอย่าง |
|---------|--------|---------|
| Entity | Object ที่มี Identity | Order, Customer, Product |
| Value Object | Immutable, defined by value | Money, Email, Address |
| Aggregate | Cluster of related objects | Order + OrderLineItems |
| Aggregate Root | Entry point to Aggregate | Order (not OrderLineItem) |
| Domain Event | Something that happened | OrderConfirmed, PaymentProcessed |
| Repository | Collection interface | OrderRepository |
| Domain Service | Business logic ที่ไม่เป็น Entity | PriceCalculationService |
| Application Service | Orchestrate use cases | CheckoutService |
| Factory | Create complex objects | Order::create() |

---

*DDD ช่วยให้ Code สื่อสาร Business Logic ได้ชัดเจน และง่ายต่อการพัฒนาร่วมกันระหว่าง Developer และ Business Expert*
