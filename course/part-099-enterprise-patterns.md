# Part 99: Enterprise Patterns ใน PHP

## บทนำ

Enterprise Patterns เป็น Patterns ที่ใช้ใน Systems ขนาดใหญ่ที่ต้องการ Scalability, Reliability และ Auditability สูง

---

## Event Sourcing

แทนที่จะ Store State ปัจจุบัน เราเก็บ **ลำดับของ Events** ทั้งหมดที่เกิดขึ้น

### ข้อดีของ Event Sourcing
- **Complete Audit Log** - รู้ทุกอย่างที่เกิดขึ้น
- **Time Travel** - Rebuild State ณ จุดใดก็ได้
- **Event Replay** - สร้าง Projection ใหม่จาก Events เก่า
- **Debugging** - เห็น "ทำไม" ไม่ใช่แค่ "อะไร"

```php
<?php

namespace App\EventSourcing;

// Domain Event Base
abstract class DomainEvent
{
    public readonly string $eventId;
    public readonly string $aggregateId;
    public readonly \DateTimeImmutable $occurredAt;
    public readonly int $aggregateVersion;
    
    public function __construct(
        string $aggregateId,
        int $aggregateVersion,
        ?\DateTimeImmutable $occurredAt = null
    ) {
        $this->eventId = \Ramsey\Uuid\Uuid::uuid4()->toString();
        $this->aggregateId = $aggregateId;
        $this->aggregateVersion = $aggregateVersion;
        $this->occurredAt = $occurredAt ?? new \DateTimeImmutable();
    }
    
    abstract public function getEventType(): string;
    abstract public function getPayload(): array;
    
    public static function fromArray(array $data): static
    {
        return new static(
            aggregateId: $data['aggregate_id'],
            aggregateVersion: $data['aggregate_version'],
            occurredAt: new \DateTimeImmutable($data['occurred_at'])
        );
    }
}

// Bank Account Events
class AccountOpened extends DomainEvent
{
    public function __construct(
        string $aggregateId,
        int $aggregateVersion,
        public readonly string $ownerId,
        public readonly string $accountNumber,
        public readonly string $currency,
        public readonly float $initialDeposit,
        ?\DateTimeImmutable $occurredAt = null
    ) {
        parent::__construct($aggregateId, $aggregateVersion, $occurredAt);
    }
    
    public function getEventType(): string { return 'account.opened'; }
    
    public function getPayload(): array
    {
        return [
            'owner_id' => $this->ownerId,
            'account_number' => $this->accountNumber,
            'currency' => $this->currency,
            'initial_deposit' => $this->initialDeposit,
        ];
    }
}

class MoneyDeposited extends DomainEvent
{
    public function __construct(
        string $aggregateId,
        int $aggregateVersion,
        public readonly float $amount,
        public readonly string $description,
        ?\DateTimeImmutable $occurredAt = null
    ) {
        parent::__construct($aggregateId, $aggregateVersion, $occurredAt);
    }
    
    public function getEventType(): string { return 'account.money_deposited'; }
    
    public function getPayload(): array
    {
        return ['amount' => $this->amount, 'description' => $this->description];
    }
}

class MoneyWithdrawn extends DomainEvent
{
    public function __construct(
        string $aggregateId,
        int $aggregateVersion,
        public readonly float $amount,
        public readonly string $description,
        ?\DateTimeImmutable $occurredAt = null
    ) {
        parent::__construct($aggregateId, $aggregateVersion, $occurredAt);
    }
    
    public function getEventType(): string { return 'account.money_withdrawn'; }
    
    public function getPayload(): array
    {
        return ['amount' => $this->amount, 'description' => $this->description];
    }
}

class AccountClosed extends DomainEvent
{
    public function __construct(
        string $aggregateId,
        int $aggregateVersion,
        public readonly string $reason,
        ?\DateTimeImmutable $occurredAt = null
    ) {
        parent::__construct($aggregateId, $aggregateVersion, $occurredAt);
    }
    
    public function getEventType(): string { return 'account.closed'; }
    public function getPayload(): array { return ['reason' => $this->reason]; }
}

// Aggregate Root
abstract class AggregateRoot
{
    private array $uncommittedEvents = [];
    protected int $version = 0;
    protected string $id;
    
    public function getId(): string { return $this->id; }
    public function getVersion(): int { return $this->version; }
    
    protected function recordEvent(DomainEvent $event): void
    {
        $this->uncommittedEvents[] = $event;
        $this->apply($event);
        $this->version++;
    }
    
    abstract protected function apply(DomainEvent $event): void;
    
    public function getUncommittedEvents(): array
    {
        return $this->uncommittedEvents;
    }
    
    public function clearUncommittedEvents(): void
    {
        $this->uncommittedEvents = [];
    }
    
    public static function reconstituteFrom(array $events): static
    {
        $aggregate = new static();
        
        foreach ($events as $event) {
            $aggregate->apply($event);
            $aggregate->version++;
        }
        
        return $aggregate;
    }
}

// BankAccount Aggregate
class BankAccount extends AggregateRoot
{
    private string $ownerId;
    private string $accountNumber;
    private string $currency;
    private float $balance = 0.0;
    private bool $isClosed = false;
    
    public static function open(
        string $accountId,
        string $ownerId,
        string $accountNumber,
        string $currency,
        float $initialDeposit
    ): self {
        if ($initialDeposit < 0) {
            throw new \InvalidArgumentException('Initial deposit cannot be negative');
        }
        
        $account = new self();
        $account->id = $accountId;
        
        $account->recordEvent(new AccountOpened(
            aggregateId: $accountId,
            aggregateVersion: 0,
            ownerId: $ownerId,
            accountNumber: $accountNumber,
            currency: $currency,
            initialDeposit: $initialDeposit
        ));
        
        return $account;
    }
    
    public function deposit(float $amount, string $description): void
    {
        $this->assertNotClosed();
        
        if ($amount <= 0) {
            throw new \InvalidArgumentException('Deposit amount must be positive');
        }
        
        $this->recordEvent(new MoneyDeposited(
            aggregateId: $this->id,
            aggregateVersion: $this->version,
            amount: $amount,
            description: $description
        ));
    }
    
    public function withdraw(float $amount, string $description): void
    {
        $this->assertNotClosed();
        
        if ($amount <= 0) {
            throw new \InvalidArgumentException('Withdrawal amount must be positive');
        }
        
        if ($amount > $this->balance) {
            throw new \DomainException(
                "Insufficient funds. Balance: {$this->balance}, Requested: {$amount}"
            );
        }
        
        $this->recordEvent(new MoneyWithdrawn(
            aggregateId: $this->id,
            aggregateVersion: $this->version,
            amount: $amount,
            description: $description
        ));
    }
    
    public function close(string $reason): void
    {
        $this->assertNotClosed();
        
        if ($this->balance > 0) {
            throw new \DomainException('Cannot close account with positive balance');
        }
        
        $this->recordEvent(new AccountClosed(
            aggregateId: $this->id,
            aggregateVersion: $this->version,
            reason: $reason
        ));
    }
    
    protected function apply(DomainEvent $event): void
    {
        match (true) {
            $event instanceof AccountOpened => $this->applyAccountOpened($event),
            $event instanceof MoneyDeposited => $this->applyMoneyDeposited($event),
            $event instanceof MoneyWithdrawn => $this->applyMoneyWithdrawn($event),
            $event instanceof AccountClosed => $this->applyAccountClosed($event),
            default => throw new \RuntimeException("Unknown event: " . get_class($event)),
        };
    }
    
    private function applyAccountOpened(AccountOpened $event): void
    {
        $this->id = $event->aggregateId;
        $this->ownerId = $event->ownerId;
        $this->accountNumber = $event->accountNumber;
        $this->currency = $event->currency;
        $this->balance = $event->initialDeposit;
    }
    
    private function applyMoneyDeposited(MoneyDeposited $event): void
    {
        $this->balance += $event->amount;
    }
    
    private function applyMoneyWithdrawn(MoneyWithdrawn $event): void
    {
        $this->balance -= $event->amount;
    }
    
    private function applyAccountClosed(AccountClosed $event): void
    {
        $this->isClosed = true;
    }
    
    private function assertNotClosed(): void
    {
        if ($this->isClosed) {
            throw new \DomainException('Account is closed');
        }
    }
    
    public function getBalance(): float { return $this->balance; }
    public function isClosed(): bool { return $this->isClosed; }
}
```

---

## Event Store

```php
<?php

// Event Store Interface
interface EventStore
{
    public function save(string $aggregateId, array $events, int $expectedVersion): void;
    public function load(string $aggregateId, int $fromVersion = 0): array;
    public function loadAll(int $fromPosition = 0, int $limit = 100): array;
}

// MySQL Event Store
class MysqlEventStore implements EventStore
{
    public function __construct(
        private \PDO $pdo,
        private EventSerializer $serializer
    ) {}
    
    public function save(string $aggregateId, array $events, int $expectedVersion): void
    {
        $this->pdo->beginTransaction();
        
        try {
            // Optimistic Concurrency Control
            $stmt = $this->pdo->prepare(
                'SELECT MAX(aggregate_version) FROM events WHERE aggregate_id = ?'
            );
            $stmt->execute([$aggregateId]);
            $currentVersion = (int)$stmt->fetchColumn();
            
            if ($currentVersion !== $expectedVersion) {
                throw new ConcurrencyException(
                    "Expected version {$expectedVersion}, got {$currentVersion}"
                );
            }
            
            $stmt = $this->pdo->prepare(
                'INSERT INTO events 
                 (event_id, aggregate_id, aggregate_version, event_type, payload, occurred_at)
                 VALUES (?, ?, ?, ?, ?, ?)'
            );
            
            foreach ($events as $event) {
                $stmt->execute([
                    $event->eventId,
                    $event->aggregateId,
                    $event->aggregateVersion,
                    $event->getEventType(),
                    json_encode($event->getPayload()),
                    $event->occurredAt->format('Y-m-d H:i:s.u'),
                ]);
            }
            
            $this->pdo->commit();
            
        } catch (\Exception $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
    
    public function load(string $aggregateId, int $fromVersion = 0): array
    {
        $stmt = $this->pdo->prepare(
            'SELECT * FROM events 
             WHERE aggregate_id = ? AND aggregate_version >= ?
             ORDER BY aggregate_version ASC'
        );
        $stmt->execute([$aggregateId, $fromVersion]);
        
        return array_map(
            fn($row) => $this->serializer->deserialize($row),
            $stmt->fetchAll(\PDO::FETCH_ASSOC)
        );
    }
    
    public function loadAll(int $fromPosition = 0, int $limit = 100): array
    {
        $stmt = $this->pdo->prepare(
            'SELECT * FROM events WHERE id > ? ORDER BY id ASC LIMIT ?'
        );
        $stmt->execute([$fromPosition, $limit]);
        
        return array_map(
            fn($row) => $this->serializer->deserialize($row),
            $stmt->fetchAll(\PDO::FETCH_ASSOC)
        );
    }
}
```

---

## CQRS (Command Query Responsibility Segregation)

```php
<?php

// Command Side (Write Model)
class TransferMoneyCommand
{
    public function __construct(
        public readonly string $fromAccountId,
        public readonly string $toAccountId,
        public readonly float $amount,
        public readonly string $description
    ) {}
}

class TransferMoneyHandler
{
    public function __construct(
        private BankAccountRepository $repository,
        private EventBus $eventBus
    ) {}
    
    public function handle(TransferMoneyCommand $command): void
    {
        $fromAccount = $this->repository->load($command->fromAccountId);
        $toAccount = $this->repository->load($command->toAccountId);
        
        $fromAccount->withdraw($command->amount, "Transfer: {$command->description}");
        $toAccount->deposit($command->amount, "Transfer: {$command->description}");
        
        $this->repository->save($fromAccount);
        $this->repository->save($toAccount);
        
        // Publish events for read model update
        foreach ($fromAccount->getUncommittedEvents() as $event) {
            $this->eventBus->publish($event);
        }
        foreach ($toAccount->getUncommittedEvents() as $event) {
            $this->eventBus->publish($event);
        }
        
        $fromAccount->clearUncommittedEvents();
        $toAccount->clearUncommittedEvents();
    }
}

// Query Side (Read Model / Projection)
class AccountSummaryProjection
{
    public function __construct(private \PDO $pdo) {}
    
    // Handle events to update read model
    public function onAccountOpened(AccountOpened $event): void
    {
        $this->pdo->prepare(
            'INSERT INTO account_summaries 
             (account_id, owner_id, account_number, currency, balance, opened_at)
             VALUES (?, ?, ?, ?, ?, ?)'
        )->execute([
            $event->aggregateId,
            $event->ownerId,
            $event->accountNumber,
            $event->currency,
            $event->initialDeposit,
            $event->occurredAt->format('Y-m-d H:i:s'),
        ]);
    }
    
    public function onMoneyDeposited(MoneyDeposited $event): void
    {
        $this->pdo->prepare(
            'UPDATE account_summaries 
             SET balance = balance + ?, transaction_count = transaction_count + 1
             WHERE account_id = ?'
        )->execute([$event->amount, $event->aggregateId]);
        
        $this->recordTransaction($event->aggregateId, 'deposit', $event->amount, $event->description);
    }
    
    public function onMoneyWithdrawn(MoneyWithdrawn $event): void
    {
        $this->pdo->prepare(
            'UPDATE account_summaries 
             SET balance = balance - ?, transaction_count = transaction_count + 1
             WHERE account_id = ?'
        )->execute([$event->amount, $event->aggregateId]);
        
        $this->recordTransaction($event->aggregateId, 'withdrawal', $event->amount, $event->description);
    }
    
    private function recordTransaction(
        string $accountId,
        string $type,
        float $amount,
        string $description
    ): void {
        $this->pdo->prepare(
            'INSERT INTO transaction_history (account_id, type, amount, description, created_at)
             VALUES (?, ?, ?, ?, NOW())'
        )->execute([$accountId, $type, $amount, $description]);
    }
}

// Query - ใช้ Read Model
class GetAccountSummaryQuery
{
    public function __construct(public readonly string $accountId) {}
}

class GetAccountSummaryHandler
{
    public function __construct(private \PDO $pdo) {}
    
    public function handle(GetAccountSummaryQuery $query): ?array
    {
        $stmt = $this->pdo->prepare(
            'SELECT * FROM account_summaries WHERE account_id = ?'
        );
        $stmt->execute([$query->accountId]);
        
        return $stmt->fetch(\PDO::FETCH_ASSOC) ?: null;
    }
}

class GetTransactionHistoryQuery
{
    public function __construct(
        public readonly string $accountId,
        public readonly int $limit = 20,
        public readonly int $offset = 0
    ) {}
}

class GetTransactionHistoryHandler
{
    public function __construct(private \PDO $pdo) {}
    
    public function handle(GetTransactionHistoryQuery $query): array
    {
        $stmt = $this->pdo->prepare(
            'SELECT * FROM transaction_history 
             WHERE account_id = ?
             ORDER BY created_at DESC
             LIMIT ? OFFSET ?'
        );
        $stmt->execute([$query->accountId, $query->limit, $query->offset]);
        
        return $stmt->fetchAll(\PDO::FETCH_ASSOC);
    }
}
```

---

## Saga Pattern (Distributed Transactions)

```php
<?php

// Saga สำหรับ Order Processing
// Steps: Reserve Stock → Process Payment → Confirm Order
// Compensations: Cancel Order ← Refund Payment ← Release Stock

enum SagaStatus: string
{
    case STARTED = 'started';
    case COMPENSATING = 'compensating';
    case COMPLETED = 'completed';
    case FAILED = 'failed';
}

abstract class Saga
{
    protected string $sagaId;
    protected SagaStatus $status = SagaStatus::STARTED;
    protected array $completedSteps = [];
    
    abstract public function start(array $data): void;
    abstract public function compensate(): void;
    
    protected function markStepCompleted(string $step, array $data = []): void
    {
        $this->completedSteps[] = ['step' => $step, 'data' => $data];
    }
}

class OrderProcessingSaga extends Saga
{
    private string $orderId;
    private array $items;
    private float $totalAmount;
    private string $customerId;
    
    public function __construct(
        private InventoryService $inventoryService,
        private PaymentService $paymentService,
        private OrderService $orderService,
        private NotificationService $notificationService
    ) {
        $this->sagaId = \Ramsey\Uuid\Uuid::uuid4()->toString();
    }
    
    public function start(array $data): void
    {
        $this->orderId = $data['order_id'];
        $this->items = $data['items'];
        $this->totalAmount = $data['total_amount'];
        $this->customerId = $data['customer_id'];
        
        try {
            // Step 1: Reserve Stock
            $reservation = $this->inventoryService->reserveStock($this->items);
            $this->markStepCompleted('stock_reserved', [
                'reservation_id' => $reservation->id
            ]);
            
            // Step 2: Process Payment
            $payment = $this->paymentService->charge([
                'customer_id' => $this->customerId,
                'amount' => $this->totalAmount,
                'order_id' => $this->orderId,
            ]);
            $this->markStepCompleted('payment_processed', [
                'payment_id' => $payment->id,
                'transaction_id' => $payment->transactionId,
            ]);
            
            // Step 3: Confirm Order
            $this->orderService->confirm($this->orderId);
            $this->markStepCompleted('order_confirmed');
            
            // Step 4: Send Notification
            $this->notificationService->sendOrderConfirmation($this->orderId);
            $this->markStepCompleted('notification_sent');
            
            $this->status = SagaStatus::COMPLETED;
            
        } catch (\Exception $e) {
            $this->status = SagaStatus::COMPENSATING;
            $this->compensate();
            
            $this->status = SagaStatus::FAILED;
            throw new SagaFailedException(
                "Order processing saga failed: {$e->getMessage()}",
                $this->sagaId,
                $this->completedSteps,
                $e
            );
        }
    }
    
    public function compensate(): void
    {
        // Compensate in reverse order
        foreach (array_reverse($this->completedSteps) as $step) {
            match ($step['step']) {
                'order_confirmed' => $this->orderService->cancel($this->orderId),
                'payment_processed' => $this->paymentService->refund($step['data']['payment_id']),
                'stock_reserved' => $this->inventoryService->releaseReservation($step['data']['reservation_id']),
                default => null,
            };
        }
    }
}
```

---

## Outbox Pattern

Outbox Pattern แก้ปัญหา "Dual Write" - ป้องกันการ Update Database แต่ไม่ได้ส่ง Event

```php
<?php

// Database Schema
// CREATE TABLE outbox (
//     id BIGINT AUTO_INCREMENT PRIMARY KEY,
//     event_id VARCHAR(36) NOT NULL UNIQUE,
//     event_type VARCHAR(100) NOT NULL,
//     payload JSON NOT NULL,
//     created_at DATETIME NOT NULL,
//     processed_at DATETIME,
//     error TEXT,
//     retries INT DEFAULT 0,
//     INDEX idx_processed (processed_at),
//     INDEX idx_created (created_at)
// );

class OutboxPublisher
{
    public function __construct(
        private \PDO $pdo,
        private MessageBroker $broker
    ) {}
    
    // เรียกจาก Scheduler ทุก N วินาที
    public function processOutbox(int $batchSize = 100): void
    {
        $this->pdo->beginTransaction();
        
        try {
            // Lock rows for processing
            $stmt = $this->pdo->prepare(
                'SELECT * FROM outbox 
                 WHERE processed_at IS NULL AND retries < 3
                 ORDER BY created_at ASC
                 LIMIT ?
                 FOR UPDATE SKIP LOCKED'
            );
            $stmt->execute([$batchSize]);
            $messages = $stmt->fetchAll(\PDO::FETCH_ASSOC);
            
            foreach ($messages as $message) {
                try {
                    $this->broker->publish(
                        topic: $message['event_type'],
                        payload: json_decode($message['payload'], true),
                        eventId: $message['event_id']
                    );
                    
                    $this->pdo->prepare(
                        'UPDATE outbox SET processed_at = NOW() WHERE id = ?'
                    )->execute([$message['id']]);
                    
                } catch (\Exception $e) {
                    $this->pdo->prepare(
                        'UPDATE outbox 
                         SET retries = retries + 1, error = ?
                         WHERE id = ?'
                    )->execute([$e->getMessage(), $message['id']]);
                }
            }
            
            $this->pdo->commit();
            
        } catch (\Exception $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
}

// Service ที่ใช้ Outbox
class OrderService
{
    public function __construct(private \PDO $pdo) {}
    
    public function placeOrder(array $orderData): Order
    {
        $this->pdo->beginTransaction();
        
        try {
            // 1. Save Order to DB
            $stmt = $this->pdo->prepare(
                'INSERT INTO orders (id, user_id, total, status, created_at)
                 VALUES (?, ?, ?, "pending", NOW())'
            );
            $orderId = \Ramsey\Uuid\Uuid::uuid4()->toString();
            $stmt->execute([$orderId, $orderData['user_id'], $orderData['total']]);
            
            // 2. Save Event to Outbox (Same Transaction!)
            $eventId = \Ramsey\Uuid\Uuid::uuid4()->toString();
            $this->pdo->prepare(
                'INSERT INTO outbox (event_id, event_type, payload, created_at)
                 VALUES (?, ?, ?, NOW())'
            )->execute([
                $eventId,
                'order.placed',
                json_encode([
                    'order_id' => $orderId,
                    'user_id' => $orderData['user_id'],
                    'total' => $orderData['total'],
                    'items' => $orderData['items'],
                ]),
            ]);
            
            $this->pdo->commit();
            
            return Order::fromId($orderId);
            
        } catch (\Exception $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
}
```

---

## Workshop: Banking System with Event Sourcing

```php
<?php

// Complete Banking System Demo
class BankingSystemDemo
{
    private EventStore $eventStore;
    private BankAccountRepository $repository;
    private AccountSummaryProjection $projection;
    private CommandBus $commandBus;
    private QueryBus $queryBus;
    
    public function run(): void
    {
        echo "=== Banking System Demo ===\n\n";
        
        // 1. เปิด Account ใหม่
        $accountId = \Ramsey\Uuid\Uuid::uuid4()->toString();
        
        $account = BankAccount::open(
            accountId: $accountId,
            ownerId: 'user-123',
            accountNumber: 'TH-0001-2024',
            currency: 'THB',
            initialDeposit: 10000.0
        );
        
        $this->repository->save($account);
        echo "Account opened with balance: {$account->getBalance()} THB\n";
        
        // 2. ฝากเงิน
        $account = $this->repository->load($accountId);
        $account->deposit(5000.0, 'Monthly salary');
        $this->repository->save($account);
        echo "After deposit: {$account->getBalance()} THB\n";
        
        // 3. ถอนเงิน
        $account = $this->repository->load($accountId);
        $account->withdraw(3000.0, 'Rent payment');
        $this->repository->save($account);
        echo "After withdrawal: {$account->getBalance()} THB\n";
        
        // 4. ดู Event History
        echo "\n=== Event History ===\n";
        $events = $this->eventStore->load($accountId);
        foreach ($events as $event) {
            echo sprintf(
                "[v%d] %s at %s\n",
                $event->aggregateVersion,
                $event->getEventType(),
                $event->occurredAt->format('Y-m-d H:i:s')
            );
        }
        
        // 5. Time Travel - Rebuild State at specific version
        echo "\n=== Time Travel: State at version 1 ===\n";
        $events = $this->eventStore->load($accountId, 0);
        $eventsUpToV1 = array_filter(
            $events,
            fn($e) => $e->aggregateVersion <= 1
        );
        
        $historicalAccount = BankAccount::reconstituteFrom(array_values($eventsUpToV1));
        echo "Balance at version 1: {$historicalAccount->getBalance()} THB\n";
        
        // 6. Query Read Model
        echo "\n=== Account Summary (Read Model) ===\n";
        $summary = $this->queryBus->handle(new GetAccountSummaryQuery($accountId));
        print_r($summary);
    }
}
```

---

## สรุป Enterprise Patterns

| Pattern | ปัญหาที่แก้ | Trade-offs |
|---------|-----------|-----------|
| Event Sourcing | Audit Log, Time Travel | ความซับซ้อนสูง, Storage มาก |
| CQRS | Read/Write Scaling แยกกัน | Eventual Consistency |
| Saga | Distributed Transactions | ยาก Debug |
| Outbox | Dual Write Problem | Polling Overhead |

---

*Enterprise Patterns เหมาะกับระบบที่ต้องการ Audit, Scale แบบ Independent, และ Resilience สูง*
