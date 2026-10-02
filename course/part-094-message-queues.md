# Part 94: Message Queues ใน PHP/Laravel

## บทนำ

Message Queue คือระบบที่ช่วยให้ Components สามารถสื่อสารกันแบบ Asynchronous โดยส่ง Messages ผ่าน Queue แทนที่จะเรียกโดยตรง

**ประโยชน์**:
- Decoupling: ผู้ส่งและผู้รับไม่ต้องรู้จักกัน
- Scalability: สามารถเพิ่ม Workers ได้ตามโหลด
- Resilience: Messages ไม่หายแม้ Worker จะ Down
- Load Leveling: รับ Spike ของ Traffic ได้

---

## RabbitMQ

### Installation & Setup

```bash
# docker-compose.yml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: secret
      RABBITMQ_DEFAULT_VHOST: /
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq

volumes:
  rabbitmq_data:
```

```bash
# Install PHP AMQP Library
composer require php-amqplib/php-amqplib
```

### Basic Producer & Consumer

```php
<?php

use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;
use PhpAmqpLib\Exchange\AMQPExchangeType;

// Producer - ส่ง Messages
class MessageProducer
{
    private AMQPStreamConnection $connection;
    private \PhpAmqpLib\Channel\AMQPChannel $channel;
    
    public function __construct()
    {
        $this->connection = new AMQPStreamConnection(
            host: env('RABBITMQ_HOST', 'localhost'),
            port: env('RABBITMQ_PORT', 5672),
            user: env('RABBITMQ_USER', 'guest'),
            password: env('RABBITMQ_PASS', 'guest'),
            vhost: env('RABBITMQ_VHOST', '/')
        );
        $this->channel = $this->connection->channel();
    }
    
    public function publish(
        string $exchange,
        string $routingKey,
        array $payload,
        array $properties = []
    ): void {
        // Declare Exchange
        $this->channel->exchange_declare(
            exchange: $exchange,
            type: AMQPExchangeType::TOPIC,
            passive: false,
            durable: true,
            auto_delete: false
        );
        
        $message = new AMQPMessage(
            json_encode($payload),
            array_merge([
                'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT, // Survive restart
                'content_type' => 'application/json',
                'timestamp' => time(),
                'message_id' => uniqid('msg_', true),
                'app_id' => 'myapp',
            ], $properties)
        );
        
        $this->channel->basic_publish($message, $exchange, $routingKey);
    }
    
    public function __destruct()
    {
        $this->channel->close();
        $this->connection->close();
    }
}

// Consumer - รับและประมวลผล Messages
class MessageConsumer
{
    private AMQPStreamConnection $connection;
    private \PhpAmqpLib\Channel\AMQPChannel $channel;
    
    public function __construct()
    {
        $this->connection = new AMQPStreamConnection(
            env('RABBITMQ_HOST'), env('RABBITMQ_PORT'),
            env('RABBITMQ_USER'), env('RABBITMQ_PASS')
        );
        $this->channel = $this->connection->channel();
    }
    
    public function consume(
        string $exchange,
        string $queue,
        string $bindingKey,
        callable $handler
    ): void {
        // Declare Exchange
        $this->channel->exchange_declare($exchange, AMQPExchangeType::TOPIC, false, true, false);
        
        // Declare Queue
        $this->channel->queue_declare($queue, false, true, false, false);
        
        // Bind Queue to Exchange
        $this->channel->queue_bind($queue, $exchange, $bindingKey);
        
        // Prefetch 1 - ดู 1 Message ต่อ Worker
        $this->channel->basic_qos(null, 1, null);
        
        // Start consuming
        $this->channel->basic_consume(
            $queue,
            '',
            false,
            false, // manual ack
            false,
            false,
            function (AMQPMessage $message) use ($handler) {
                try {
                    $data = json_decode($message->body, true);
                    $handler($data, $message->get('message_id'));
                    $message->ack(); // Acknowledge - ลบออกจาก Queue
                } catch (\Exception $e) {
                    $redelivered = $message->isRedelivered();
                    
                    if ($redelivered) {
                        // ส่งไป Dead Letter Queue
                        $message->nack(false); // Don't requeue
                        $this->sendToDeadLetter($message, $e);
                    } else {
                        $message->nack(true); // Requeue once
                    }
                    
                    report($e);
                }
            }
        );
        
        // Blocking consume loop
        while ($this->channel->is_consuming()) {
            $this->channel->wait();
        }
    }
    
    private function sendToDeadLetter(AMQPMessage $original, \Exception $error): void
    {
        $producer = new MessageProducer();
        $producer->publish('dead-letters', 'failed', [
            'original_body' => json_decode($original->body, true),
            'error' => $error->getMessage(),
            'failed_at' => date('c'),
        ]);
    }
}

// Event-specific Publishers
class OrderEventPublisher
{
    public function __construct(private MessageProducer $producer) {}
    
    public function publishOrderPlaced(array $order): void
    {
        $this->producer->publish('orders', 'order.placed', [
            'event_type' => 'order.placed',
            'order_id' => $order['id'],
            'customer_id' => $order['customer_id'],
            'total' => $order['total'],
            'items' => $order['items'],
            'occurred_at' => date('c'),
        ]);
    }
    
    public function publishOrderShipped(string $orderId, string $trackingNumber): void
    {
        $this->producer->publish('orders', 'order.shipped', [
            'event_type' => 'order.shipped',
            'order_id' => $orderId,
            'tracking_number' => $trackingNumber,
            'occurred_at' => date('c'),
        ]);
    }
    
    public function publishOrderCancelled(string $orderId, string $reason): void
    {
        $this->producer->publish('orders', 'order.cancelled', [
            'event_type' => 'order.cancelled',
            'order_id' => $orderId,
            'reason' => $reason,
            'occurred_at' => date('c'),
        ]);
    }
}

// Worker Scripts
// workers/notification-worker.php
$consumer = new MessageConsumer();
$consumer->consume(
    exchange: 'orders',
    queue: 'notifications',
    bindingKey: 'order.*',
    handler: function (array $event, string $messageId) {
        echo "Processing: {$event['event_type']} | MessageId: {$messageId}\n";
        
        match($event['event_type']) {
            'order.placed' => sendOrderConfirmationEmail($event),
            'order.shipped' => sendShippingNotificationEmail($event),
            'order.cancelled' => sendCancellationEmail($event),
            default => null,
        };
    }
);
```

---

## Redis Pub/Sub

```php
<?php

// Redis Publisher
class RedisPublisher
{
    public function __construct(private \Redis $redis) {}
    
    public function publish(string $channel, array $data): int
    {
        $payload = json_encode([
            'data' => $data,
            'published_at' => microtime(true),
            'message_id' => uniqid('', true),
        ]);
        
        return $this->redis->publish($channel, $payload);
    }
}

// Redis Subscriber
class RedisSubscriber
{
    private array $handlers = [];
    
    public function __construct(private \Redis $redis) {}
    
    public function subscribe(string $channel, callable $handler): void
    {
        $this->handlers[$channel] = $handler;
        
        $this->redis->subscribe([$channel], function ($redis, $channel, $message) {
            $data = json_decode($message, true);
            
            if (isset($this->handlers[$channel])) {
                ($this->handlers[$channel])($data['data'], $channel);
            }
        });
    }
    
    public function psubscribe(string $pattern, callable $handler): void
    {
        $this->redis->psubscribe([$pattern], function ($redis, $pattern, $channel, $message) use ($handler) {
            $data = json_decode($message, true);
            $handler($data['data'], $channel, $pattern);
        });
    }
}

// Real-time Dashboard Updates
class DashboardBroadcaster
{
    public function __construct(private RedisPublisher $publisher) {}
    
    public function broadcastOrderUpdate(array $order): void
    {
        $this->publisher->publish('dashboard:orders', [
            'type' => 'order_update',
            'order' => $order,
        ]);
    }
    
    public function broadcastInventoryAlert(int $productId, int $stock): void
    {
        $this->publisher->publish('dashboard:alerts', [
            'type' => 'low_inventory',
            'product_id' => $productId,
            'current_stock' => $stock,
        ]);
    }
}
```

---

## Laravel Queues

### Job Classes

```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\{InteractsWithQueue, SerializesModels};
use Illuminate\Queue\Middleware\{WithoutOverlapping, RateLimited, ThrottlesExceptions};

class ProcessOrderJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    // จำนวนครั้งที่ Retry
    public int $tries = 3;
    
    // Timeout
    public int $timeout = 120;
    
    // Delay ระหว่าง Retry (seconds)
    public int $backoff = 60;
    
    public function __construct(
        private readonly int $orderId,
        private readonly array $paymentDetails
    ) {}
    
    public function handle(
        OrderService $orderService,
        PaymentService $paymentService
    ): void {
        $order = Order::findOrFail($this->orderId);
        
        // ถ้าถูก Processed ไปแล้ว ให้ Skip
        if ($order->status !== 'pending') {
            $this->delete();
            return;
        }
        
        $result = $paymentService->charge(
            $order->total,
            $this->paymentDetails
        );
        
        if ($result->success) {
            $orderService->confirm($order, $result->transactionId);
        } else {
            throw new \RuntimeException("Payment failed: {$result->message}");
        }
    }
    
    // Middleware ที่ใช้กับ Job นี้
    public function middleware(): array
    {
        return [
            // ป้องกัน Process Order เดิมซ้อนกัน
            (new WithoutOverlapping($this->orderId))->releaseAfter(60),
            
            // จำกัด Rate
            new RateLimited('order-processing'),
            
            // จัดการ Exception อย่างฉลาด
            (new ThrottlesExceptions(3, 60))->backoff(5),
        ];
    }
    
    // Called when job fails after all retries
    public function failed(\Throwable $exception): void
    {
        logger()->error('Order processing failed', [
            'order_id' => $this->orderId,
            'error' => $exception->getMessage(),
        ]);
        
        // Notify operations team
        Notification::route('slack', config('services.slack.ops_webhook'))
            ->notify(new OrderProcessingFailedNotification($this->orderId, $exception));
    }
}

// Batch Jobs
class SendNewsletterJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public function __construct(
        private readonly int $userId,
        private readonly string $content
    ) {}
    
    public function handle(MailService $mail): void
    {
        $user = User::find($this->userId);
        if (!$user || !$user->newsletter_subscribed) return;
        
        $mail->send(
            to: $user->email,
            subject: 'Newsletter',
            body: $this->content
        );
    }
}

// การส่ง Batch Jobs
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

class NewsletterController extends Controller
{
    public function send(Request $request): JsonResponse
    {
        $content = $request->input('content');
        $userIds = User::where('newsletter_subscribed', true)->pluck('id');
        
        $jobs = $userIds->map(
            fn($id) => new SendNewsletterJob($id, $content)
        );
        
        $batch = Bus::batch($jobs)
            ->name('Newsletter: ' . date('Y-m-d'))
            ->allowFailures()
            ->onQueue('newsletters')
            ->then(function (Batch $batch) {
                logger()->info("Newsletter sent: {$batch->processedJobs()}/{$batch->totalJobs}");
            })
            ->catch(function (Batch $batch, \Throwable $e) {
                logger()->error("Newsletter batch error: {$e->getMessage()}");
            })
            ->finally(function (Batch $batch) {
                event(new NewsletterSentEvent($batch->id));
            })
            ->dispatch();
        
        return response()->json([
            'batch_id' => $batch->id,
            'total_jobs' => $batch->totalJobs,
        ]);
    }
    
    public function status(string $batchId): JsonResponse
    {
        $batch = Bus::findBatch($batchId);
        
        return response()->json([
            'id' => $batch->id,
            'name' => $batch->name,
            'total_jobs' => $batch->totalJobs,
            'processed_jobs' => $batch->processedJobs(),
            'failed_jobs' => $batch->failedJobs,
            'progress' => $batch->progress(),
            'finished' => $batch->finished(),
            'finished_at' => $batch->finishedAt?->toISOString(),
        ]);
    }
}
```

---

## Laravel Horizon

Laravel Horizon ให้ Dashboard สำหรับ Monitor Redis Queues

```bash
composer require laravel/horizon
php artisan horizon:install
php artisan migrate
```

```php
<?php

// config/horizon.php
return [
    'use' => 'default',
    'prefix' => env('HORIZON_PREFIX', 'horizon:'),
    'middleware' => ['web'],
    'waits' => [
        'redis:default' => 60,
    ],
    'trim' => [
        'recent' => 60,
        'pending' => 60,
        'completed' => 60,
        'recent_failed' => 10080,
        'failed' => 10080,
        'monitored' => 10080,
    ],
    
    'environments' => [
        'production' => [
            // Default Queue
            'supervisor-default' => [
                'maxProcesses' => 10,
                'balanceMaxShift' => 1,
                'balanceCooldown' => 3,
                'connection' => 'redis',
                'queue' => ['default'],
                'balance' => 'auto',
                'autoScalingStrategy' => 'time',
                'minProcesses' => 1,
                'maxProcesses' => 10,
                'memory' => 128,
                'tries' => 3,
                'timeout' => 60,
            ],
            
            // High Priority Queue
            'supervisor-high' => [
                'connection' => 'redis',
                'queue' => ['high-priority'],
                'balance' => 'simple',
                'processes' => 5,
                'tries' => 3,
            ],
            
            // Email Queue
            'supervisor-emails' => [
                'connection' => 'redis',
                'queue' => ['emails'],
                'balance' => 'auto',
                'minProcesses' => 2,
                'maxProcesses' => 15,
                'tries' => 5,
            ],
        ],
        
        'local' => [
            'supervisor-local' => [
                'maxProcesses' => 3,
                'tries' => 3,
                'connection' => 'redis',
                'queue' => ['default', 'emails'],
                'balance' => 'simple',
            ],
        ],
    ],
    
    // Metrics
    'metrics' => [
        'trim_snapshots' => [
            'job' => 24,
            'queue' => 24,
        ],
    ],
    
    // Fast/Slow Jobs
    'fast_termination' => false,
];

// Horizon Access Control
class HorizonServiceProvider extends AuthServiceProvider
{
    public function boot(): void
    {
        parent::boot();
        
        Horizon::auth(function ($request) {
            return $request->user()?->isAdmin() ?? false;
        });
        
        // Notifications
        Horizon::routeMailNotificationsTo('ops@myapp.com');
        Horizon::routeSlackNotificationsTo(
            config('services.slack.webhook'),
            '#ops-alerts'
        );
        
        // SMS Notifications for critical
        Horizon::routeSmsNotificationsTo('+66-99-999-9999');
    }
}
```

---

## Apache Kafka Basics

```php
<?php

// ใช้ rdkafka extension
// composer require arnaud-lb/php-rdkafka

class KafkaProducer
{
    private \RdKafka\Producer $producer;
    
    public function __construct()
    {
        $config = new \RdKafka\Conf();
        $config->set('metadata.broker.list', env('KAFKA_BROKERS', 'localhost:9092'));
        $config->set('socket.timeout.ms', '5000');
        
        // Error callbacks
        $config->setErrorCb(function ($kafka, $err, $reason) {
            logger()->error("Kafka error: " . rd_kafka_err2str($err) . " | {$reason}");
        });
        
        $this->producer = new \RdKafka\Producer($config);
    }
    
    public function produce(string $topic, array $payload, ?string $key = null): void
    {
        $topic = $this->producer->newTopic($topic);
        
        $topic->produce(
            partition: \RD_KAFKA_PARTITION_UA, // Auto partition
            msgflags: 0,
            payload: json_encode($payload),
            key: $key
        );
        
        // Wait for message to be delivered
        $this->producer->flush(10000); // 10 second timeout
    }
}

class KafkaConsumer
{
    private \RdKafka\KafkaConsumer $consumer;
    
    public function __construct(string $groupId)
    {
        $config = new \RdKafka\Conf();
        $config->set('group.id', $groupId);
        $config->set('metadata.broker.list', env('KAFKA_BROKERS'));
        $config->set('auto.offset.reset', 'earliest');
        $config->set('enable.auto.commit', 'false'); // Manual commit
        
        $this->consumer = new \RdKafka\KafkaConsumer($config);
    }
    
    public function subscribe(array $topics): void
    {
        $this->consumer->subscribe($topics);
    }
    
    public function consume(callable $handler, int $timeoutMs = 10000): void
    {
        while (true) {
            $message = $this->consumer->consume($timeoutMs);
            
            switch ($message->err) {
                case \RD_KAFKA_RESP_ERR_NO_ERROR:
                    $payload = json_decode($message->payload, true);
                    
                    try {
                        $handler($payload, $message->topic_name, $message->partition, $message->offset);
                        $this->consumer->commit($message);
                    } catch (\Exception $e) {
                        report($e);
                        // Don't commit - message will be reprocessed
                    }
                    break;
                    
                case \RD_KAFKA_RESP_ERR__PARTITION_EOF:
                    // End of partition - normal
                    break;
                    
                case \RD_KAFKA_RESP_ERR__TIMED_OUT:
                    // Timeout - normal, continue
                    break;
                    
                default:
                    throw new \Exception("Kafka error: " . $message->errstr());
            }
        }
    }
}

// Event Streaming
class OrderEventStream
{
    public function __construct(
        private KafkaProducer $producer,
        private KafkaConsumer $consumer
    ) {}
    
    public function publishOrderEvent(string $type, array $data): void
    {
        $this->producer->produce('order-events', [
            'type' => $type,
            'data' => $data,
            'schema_version' => '1.0',
            'produced_at' => microtime(true),
        ], $data['order_id'] ?? null); // Use order_id as key for ordering
    }
    
    public function consumeOrderEvents(): void
    {
        $this->consumer->subscribe(['order-events']);
        $this->consumer->consume(function (array $event, string $topic) {
            echo "Event: {$event['type']} | Topic: {$topic}\n";
            
            match($event['type']) {
                'order.placed' => $this->handleOrderPlaced($event['data']),
                'order.confirmed' => $this->handleOrderConfirmed($event['data']),
                'payment.processed' => $this->handlePaymentProcessed($event['data']),
                default => null,
            };
        });
    }
}
```

---

## Workshop: Event-Driven Architecture

```php
<?php

// === Event Bus ===

interface EventBus
{
    public function publish(string $channel, array $event): void;
    public function subscribe(string $channel, callable $handler): void;
}

class LaravelEventBus implements EventBus
{
    public function publish(string $channel, array $event): void
    {
        // ใช้ Redis สำหรับ Real-time
        \Redis::publish($channel, json_encode($event));
        
        // ใช้ Queue สำหรับ Reliability
        dispatch(new ProcessEventJob($channel, $event))
            ->onQueue('events');
    }
    
    public function subscribe(string $channel, callable $handler): void
    {
        // Register handler
        app('event-bus.handlers')->register($channel, $handler);
    }
}

// === Order Service ===

class OrderService
{
    public function __construct(
        private OrderRepository $orders,
        private EventBus $eventBus
    ) {}
    
    public function placeOrder(array $data): Order
    {
        $order = Order::create($data);
        
        $this->eventBus->publish('order-service', [
            'type' => 'OrderPlaced',
            'aggregate_id' => $order->id,
            'data' => $order->toArray(),
            'timestamp' => now()->toISOString(),
        ]);
        
        return $order;
    }
}

// === Event Handlers ===

class InventoryEventHandler
{
    public function handle(array $event): void
    {
        match($event['type']) {
            'OrderPlaced' => $this->reserveInventory($event['data']),
            'OrderCancelled' => $this->releaseInventory($event['data']),
            'OrderShipped' => $this->deductInventory($event['data']),
            default => null,
        };
    }
    
    private function reserveInventory(array $order): void
    {
        foreach ($order['items'] as $item) {
            Inventory::where('product_id', $item['product_id'])
                ->decrement('available_quantity', $item['quantity']);
        }
    }
}

class NotificationEventHandler
{
    public function handle(array $event): void
    {
        $userId = $event['data']['user_id'];
        $user = User::find($userId);
        
        match($event['type']) {
            'OrderPlaced' => $this->sendOrderConfirmation($user, $event['data']),
            'OrderShipped' => $this->sendShippingNotification($user, $event['data']),
            'PaymentFailed' => $this->sendPaymentFailureAlert($user, $event['data']),
            default => null,
        };
    }
}

// === Saga Pattern สำหรับ Distributed Transactions ===

class OrderPlacementSaga
{
    private array $steps = [];
    private array $compensations = [];
    
    public function execute(array $orderData): bool
    {
        try {
            // Step 1: Reserve Inventory
            $reservation = $this->reserveInventory($orderData['items']);
            $this->compensations[] = fn() => $this->releaseInventory($reservation['id']);
            
            // Step 2: Process Payment
            $payment = $this->processPayment($orderData['payment']);
            $this->compensations[] = fn() => $this->refundPayment($payment['transaction_id']);
            
            // Step 3: Create Order
            $order = $this->createOrder(array_merge($orderData, [
                'reservation_id' => $reservation['id'],
                'transaction_id' => $payment['transaction_id'],
            ]));
            
            // Step 4: Notify Customer
            $this->notifyCustomer($order);
            
            return true;
            
        } catch (\Exception $e) {
            // Execute compensating transactions in reverse order
            $this->compensate();
            throw $e;
        }
    }
    
    private function compensate(): void
    {
        foreach (array_reverse($this->compensations) as $compensation) {
            try {
                $compensation();
            } catch (\Exception $e) {
                // Log but continue compensating
                logger()->error('Compensation failed: ' . $e->getMessage());
            }
        }
    }
    
    private function reserveInventory(array $items): array
    {
        // Call Inventory Service
        return ['id' => 'RES-' . uniqid()];
    }
    
    private function processPayment(array $payment): array
    {
        // Call Payment Service
        return ['transaction_id' => 'TXN-' . uniqid()];
    }
    
    private function releaseInventory(string $reservationId): void
    {
        // Release inventory reservation
        echo "Releasing inventory reservation: {$reservationId}\n";
    }
    
    private function refundPayment(string $transactionId): void
    {
        // Refund payment
        echo "Refunding payment: {$transactionId}\n";
    }
    
    private function createOrder(array $data): array
    {
        return Order::create($data)->toArray();
    }
    
    private function notifyCustomer(array $order): void
    {
        dispatch(new SendOrderConfirmationJob($order));
    }
}
```

---

## Queue Configuration

```php
<?php

// config/queue.php
return [
    'default' => env('QUEUE_CONNECTION', 'redis'),
    
    'connections' => [
        'sync' => ['driver' => 'sync'],
        
        'database' => [
            'driver' => 'database',
            'table' => 'jobs',
            'queue' => 'default',
            'retry_after' => 90,
            'after_commit' => false,
        ],
        
        'redis' => [
            'driver' => 'redis',
            'connection' => 'queue',
            'queue' => env('REDIS_QUEUE', 'default'),
            'retry_after' => 90,
            'block_for' => null,
            'after_commit' => false,
        ],
        
        'rabbitmq' => [
            'driver' => 'rabbitmq',
            'queue' => 'default',
            'connection' => PhpAmqpLib\Connection\AMQPLazyConnection::class,
            'hosts' => [
                [
                    'host' => env('RABBITMQ_HOST', '127.0.0.1'),
                    'port' => env('RABBITMQ_PORT', 5672),
                    'user' => env('RABBITMQ_USER', 'guest'),
                    'password' => env('RABBITMQ_PASSWORD', 'guest'),
                    'vhost' => env('RABBITMQ_VHOST', '/'),
                ],
            ],
        ],
    ],
    
    'failed' => [
        'driver' => env('QUEUE_FAILED_DRIVER', 'database-uuids'),
        'database' => env('DB_CONNECTION', 'mysql'),
        'table' => 'failed_jobs',
    ],
];

// Queue Priorities
SendCriticalAlertJob::dispatch()->onQueue('high-priority');
SendNewsletterJob::dispatch()->onQueue('low-priority');

// Delayed Jobs
ProcessRefundJob::dispatch($orderId)->delay(now()->addMinutes(5));

// Chained Jobs
Bus::chain([
    new ProcessPaymentJob($orderId),
    new CreateInvoiceJob($orderId),
    new SendReceiptEmailJob($orderId),
])->onQueue('orders')->dispatch();
```

---

## สรุป

| Queue System | เหมาะสำหรับ | Guarantee | Throughput |
|-------------|------------|-----------|-----------|
| Redis Queue | Laravel Jobs, Simple | At-least-once | สูง |
| RabbitMQ | Complex routing, DLQ | At-least-once | สูง |
| Kafka | Event Streaming, Big Data | At-least-once/Exactly-once | สูงมาก |
| Database Queue | Simple, Low traffic | At-least-once | ต่ำ |
| SQS | AWS ecosystem | At-least-once | สูงมาก |

---

*Queue = "พนักงานรับออร์เดอร์" ที่รับงานแล้วส่งต่อให้ "ครัว" ทำ - ไม่ต้องรอเสร็จทันที*
