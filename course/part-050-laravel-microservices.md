# Part 050: Laravel Microservices — แยก Monolith เป็น Services

**ระดับ: ระดับโลก (World-class)**
**เวลาเรียน: 8-10 ชั่วโมง**

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจ Microservices Architecture และ Trade-offs
- สื่อสารระหว่าง Services ด้วย HTTP REST และ Message Queue
- สร้าง API Gateway ด้วย Laravel
- ใช้ Message Queue Integration ระหว่าง Services
- แยก Monolith Laravel ออกเป็น Microservices

---

## 1. Microservices vs Monolith

### 1.1 Monolith Architecture

```
┌─────────────────────────────────┐
│         Monolith Laravel        │
│                                 │
│  ┌──────────┐  ┌─────────────┐  │
│  │  Users   │  │   Orders    │  │
│  └──────────┘  └─────────────┘  │
│  ┌──────────┐  ┌─────────────┐  │
│  │Products  │  │  Payments   │  │
│  └──────────┘  └─────────────┘  │
│                                 │
│         Single Database         │
└─────────────────────────────────┘
```

**ข้อดีของ Monolith:**
- ง่ายต่อการพัฒนาและ Debug
- Deploy ครั้งเดียว
- ไม่มี Network Latency ระหว่าง Services
- ACID Transactions ง่าย

**ข้อเสียของ Monolith:**
- Scale ยากเมื่อระบบใหญ่
- ทีมหลายทีมแก้ Code เดียวกัน
- Tech Stack เดียวทั้งระบบ
- Deploy ทั้งระบบทุกครั้ง

### 1.2 Microservices Architecture

```
                    ┌─────────────┐
                    │ API Gateway │
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
   ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐
   │User Service │  │Order Service│  │Product Svc  │
   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
          │                │                │
   ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐
   │  Users DB   │  │  Orders DB  │  │Products DB  │
   └─────────────┘  └─────────────┘  └─────────────┘
          
                     Message Bus (RabbitMQ/Kafka)
```

---

## 2. Service Communication

### 2.1 Synchronous Communication (HTTP REST)

```php
// app/Services/External/UserServiceClient.php
namespace App\Services\External;

use Illuminate\Http\Client\Factory as HttpClient;
use Illuminate\Http\Client\Response;

class UserServiceClient
{
    private readonly string $baseUrl;
    private readonly string $apiKey;
    
    public function __construct(
        private readonly HttpClient $http
    ) {
        $this->baseUrl = config('services.user-service.url');
        $this->apiKey = config('services.user-service.key');
    }
    
    public function getUser(int $userId): ?array
    {
        try {
            $response = $this->http
                ->withToken($this->apiKey)
                ->timeout(5)
                ->retry(3, 100)
                ->get("{$this->baseUrl}/api/users/{$userId}");
            
            if ($response->successful()) {
                return $response->json();
            }
            
            if ($response->status() === 404) {
                return null;
            }
            
            throw new \RuntimeException("User service error: " . $response->status());
        } catch (\Exception $e) {
            \Log::error("Failed to fetch user {$userId} from User Service", [
                'error' => $e->getMessage(),
            ]);
            
            throw new ServiceUnavailableException("User Service is unavailable", 0, $e);
        }
    }
    
    public function getUsers(array $userIds): array
    {
        $response = $this->http
            ->withToken($this->apiKey)
            ->timeout(10)
            ->get("{$this->baseUrl}/api/users", [
                'ids' => implode(',', $userIds),
            ]);
        
        return $response->json('data', []);
    }
    
    public function updateUser(int $userId, array $data): array
    {
        $response = $this->http
            ->withToken($this->apiKey)
            ->put("{$this->baseUrl}/api/users/{$userId}", $data);
        
        if (!$response->successful()) {
            throw new \RuntimeException("Failed to update user: " . $response->body());
        }
        
        return $response->json();
    }
}
```

### 2.2 Circuit Breaker Pattern

ป้องกัน Cascading Failures เมื่อ Service ล่ม

```php
// app/Services/CircuitBreaker.php
namespace App\Services;

use Illuminate\Support\Facades\Cache;

class CircuitBreaker
{
    private const STATE_CLOSED = 'closed';     // ปกติ - request ผ่าน
    private const STATE_OPEN = 'open';         // Circuit เปิด - reject requests
    private const STATE_HALF_OPEN = 'half_open'; // ทดสอบว่าหายดีหรือยัง
    
    public function __construct(
        private readonly string $serviceName,
        private readonly int $threshold = 5,      // จำนวน failures ก่อน open
        private readonly int $timeout = 60,       // วินาทีที่จะ half-open
        private readonly int $successThreshold = 2 // successes ก่อน close
    ) {}
    
    public function call(callable $request): mixed
    {
        $state = $this->getState();
        
        if ($state === self::STATE_OPEN) {
            throw new CircuitOpenException("{$this->serviceName} circuit is open");
        }
        
        try {
            $result = $request();
            $this->onSuccess();
            
            return $result;
        } catch (\Exception $e) {
            $this->onFailure();
            throw $e;
        }
    }
    
    private function getState(): string
    {
        $failures = Cache::get($this->failureKey(), 0);
        $lastFailureTime = Cache::get($this->lastFailureKey());
        
        if ($failures >= $this->threshold) {
            // ตรวจสอบว่าผ่าน timeout หรือยัง
            if ($lastFailureTime && (time() - $lastFailureTime) > $this->timeout) {
                return self::STATE_HALF_OPEN;
            }
            
            return self::STATE_OPEN;
        }
        
        return self::STATE_CLOSED;
    }
    
    private function onSuccess(): void
    {
        $state = $this->getState();
        
        if ($state === self::STATE_HALF_OPEN) {
            $successes = Cache::increment($this->successKey());
            
            if ($successes >= $this->successThreshold) {
                // Reset circuit
                Cache::forget($this->failureKey());
                Cache::forget($this->lastFailureKey());
                Cache::forget($this->successKey());
            }
        }
    }
    
    private function onFailure(): void
    {
        Cache::increment($this->failureKey());
        Cache::put($this->lastFailureKey(), time(), $this->timeout * 2);
        Cache::forget($this->successKey());
    }
    
    private function failureKey(): string
    {
        return "circuit:{$this->serviceName}:failures";
    }
    
    private function lastFailureKey(): string
    {
        return "circuit:{$this->serviceName}:last_failure";
    }
    
    private function successKey(): string
    {
        return "circuit:{$this->serviceName}:successes";
    }
}
```

**ใช้งาน Circuit Breaker:**
```php
// app/Services/External/UserServiceClient.php
class UserServiceClient
{
    private readonly CircuitBreaker $circuitBreaker;
    
    public function __construct()
    {
        $this->circuitBreaker = new CircuitBreaker('user-service');
    }
    
    public function getUser(int $userId): ?array
    {
        return $this->circuitBreaker->call(function () use ($userId) {
            $response = Http::timeout(5)
                ->get("{$this->baseUrl}/api/users/{$userId}");
            
            $response->throw(); // throws on 4xx/5xx
            
            return $response->json();
        });
    }
}
```

---

## 3. API Gateway ด้วย Laravel

API Gateway เป็น Entry Point เดียวสำหรับ Clients ทุกตัว

### 3.1 สร้าง API Gateway

```php
// gateway/app/Http/Controllers/GatewayController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Services\Gateway\RouteRegistry;
use App\Services\Gateway\RequestForwarder;
use App\Services\Gateway\RateLimiter;

class GatewayController extends Controller
{
    public function __construct(
        private readonly RouteRegistry $routeRegistry,
        private readonly RequestForwarder $forwarder,
        private readonly RateLimiter $rateLimiter
    ) {}
    
    public function handle(Request $request, string $service, string $path = '')
    {
        // 1. ตรวจสอบ Rate Limiting
        $this->rateLimiter->check($request, $service);
        
        // 2. หา Service URL จาก Registry
        $serviceUrl = $this->routeRegistry->resolve($service);
        
        if (!$serviceUrl) {
            return response()->json(['error' => 'Service not found'], 404);
        }
        
        // 3. Forward Request ไปยัง Service
        return $this->forwarder->forward($request, $serviceUrl, $path);
    }
}
```

```php
// gateway/app/Services/Gateway/RequestForwarder.php
namespace App\Services\Gateway;

use Illuminate\Http\Request;
use Illuminate\Http\Client\Factory as Http;
use Illuminate\Support\Facades\Auth;

class RequestForwarder
{
    public function __construct(
        private readonly Http $http
    ) {}
    
    public function forward(Request $request, string $serviceUrl, string $path)
    {
        $targetUrl = rtrim($serviceUrl, '/') . '/' . ltrim($path, '/');
        
        // เพิ่ม Query String
        if ($request->query()) {
            $targetUrl .= '?' . http_build_query($request->query());
        }
        
        $headers = $this->buildHeaders($request);
        
        $response = match(strtoupper($request->method())) {
            'GET' => $this->http->withHeaders($headers)->get($targetUrl),
            'POST' => $this->http->withHeaders($headers)->post($targetUrl, $request->all()),
            'PUT' => $this->http->withHeaders($headers)->put($targetUrl, $request->all()),
            'PATCH' => $this->http->withHeaders($headers)->patch($targetUrl, $request->all()),
            'DELETE' => $this->http->withHeaders($headers)->delete($targetUrl),
            default => throw new \InvalidArgumentException("Unsupported method: {$request->method()}"),
        };
        
        return response(
            $response->body(),
            $response->status(),
            $response->headers()
        );
    }
    
    private function buildHeaders(Request $request): array
    {
        $headers = [
            'Content-Type' => 'application/json',
            'Accept' => 'application/json',
            'X-Request-ID' => $request->header('X-Request-ID', (string) \Illuminate\Support\Str::uuid()),
            'X-Forwarded-For' => $request->ip(),
        ];
        
        // Forward JWT Token
        if ($token = $request->bearerToken()) {
            $headers['Authorization'] = "Bearer {$token}";
        }
        
        // เพิ่ม Internal Service Token
        $headers['X-Service-Token'] = config('gateway.service_token');
        
        // เพิ่ม User Info จาก JWT
        if (auth('api')->check()) {
            $user = auth('api')->user();
            $headers['X-User-ID'] = $user->id;
            $headers['X-User-Roles'] = implode(',', $user->getRoleNames()->toArray());
        }
        
        return $headers;
    }
}
```

### 3.2 Service Registry

```php
// gateway/app/Services/Gateway/RouteRegistry.php
namespace App\Services\Gateway;

class RouteRegistry
{
    private array $routes;
    
    public function __construct()
    {
        $this->routes = config('gateway.services', []);
    }
    
    public function resolve(string $service): ?string
    {
        return $this->routes[$service] ?? null;
    }
    
    public function all(): array
    {
        return $this->routes;
    }
}
```

```php
// gateway/config/gateway.php
return [
    'service_token' => env('GATEWAY_SERVICE_TOKEN'),
    
    'services' => [
        'users' => env('USER_SERVICE_URL', 'http://user-service:8000'),
        'orders' => env('ORDER_SERVICE_URL', 'http://order-service:8001'),
        'products' => env('PRODUCT_SERVICE_URL', 'http://product-service:8002'),
        'payments' => env('PAYMENT_SERVICE_URL', 'http://payment-service:8003'),
        'notifications' => env('NOTIFICATION_SERVICE_URL', 'http://notification-service:8004'),
    ],
    
    'rate_limits' => [
        'default' => 60,
        'users' => 120,
        'orders' => 30,
    ],
];
```

### 3.3 Gateway Routes

```php
// gateway/routes/api.php
use App\Http\Controllers\GatewayController;
use App\Http\Middleware\{AuthenticateGateway, RateLimitMiddleware, LogRequest};

Route::middleware([LogRequest::class])->group(function () {
    // Public routes
    Route::post('/auth/login', [GatewayController::class, 'handle'])
        ->defaults('service', 'users')
        ->defaults('path', 'auth/login');
    
    Route::post('/auth/register', [GatewayController::class, 'handle'])
        ->defaults('service', 'users')
        ->defaults('path', 'auth/register');
    
    // Protected routes
    Route::middleware([AuthenticateGateway::class, RateLimitMiddleware::class])
        ->group(function () {
            Route::any('/users/{path?}', [GatewayController::class, 'handle'])
                ->defaults('service', 'users')
                ->where('path', '.*');
            
            Route::any('/orders/{path?}', [GatewayController::class, 'handle'])
                ->defaults('service', 'orders')
                ->where('path', '.*');
            
            Route::any('/products/{path?}', [GatewayController::class, 'handle'])
                ->defaults('service', 'products')
                ->where('path', '.*');
        });
});
```

---

## 4. Message Queue Integration

Asynchronous Communication ระหว่าง Services

### 4.1 RabbitMQ Setup

```bash
# ติดตั้ง RabbitMQ
docker run -d --name rabbitmq \
    -p 5672:5672 \
    -p 15672:15672 \
    rabbitmq:3-management

# ติดตั้ง PHP Library
composer require php-amqplib/php-amqplib
```

```php
// app/Services/MessageBus/RabbitMQPublisher.php
namespace App\Services\MessageBus;

use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

class RabbitMQPublisher
{
    private AMQPStreamConnection $connection;
    
    public function __construct()
    {
        $this->connection = new AMQPStreamConnection(
            config('rabbitmq.host'),
            config('rabbitmq.port'),
            config('rabbitmq.user'),
            config('rabbitmq.password'),
            config('rabbitmq.vhost')
        );
    }
    
    public function publish(string $exchange, string $routingKey, array $payload): void
    {
        $channel = $this->connection->channel();
        
        $channel->exchange_declare($exchange, 'topic', false, true, false);
        
        $message = new AMQPMessage(
            json_encode($payload),
            [
                'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT,
                'content_type' => 'application/json',
                'timestamp' => time(),
                'message_id' => (string) \Illuminate\Support\Str::uuid(),
            ]
        );
        
        $channel->basic_publish($message, $exchange, $routingKey);
        
        $channel->close();
    }
    
    public function __destruct()
    {
        $this->connection->close();
    }
}
```

### 4.2 Event Publishing

```php
// app/Events/OrderCreated.php (Order Service)
namespace App\Events;

use App\Models\Order;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderCreated
{
    use Dispatchable, SerializesModels;
    
    public function __construct(
        public readonly Order $order
    ) {}
    
    public function toPayload(): array
    {
        return [
            'event' => 'order.created',
            'version' => '1.0',
            'timestamp' => now()->toISOString(),
            'data' => [
                'order_id' => $this->order->id,
                'user_id' => $this->order->user_id,
                'total' => $this->order->total,
                'items' => $this->order->items->map(fn($item) => [
                    'product_id' => $item->product_id,
                    'quantity' => $item->quantity,
                    'price' => $item->price,
                ])->toArray(),
                'shipping_address' => $this->order->shipping_address,
                'created_at' => $this->order->created_at->toISOString(),
            ],
        ];
    }
}
```

```php
// app/Listeners/PublishOrderCreatedEvent.php
namespace App\Listeners;

use App\Events\OrderCreated;
use App\Services\MessageBus\RabbitMQPublisher;

class PublishOrderCreatedEvent
{
    public function __construct(
        private readonly RabbitMQPublisher $publisher
    ) {}
    
    public function handle(OrderCreated $event): void
    {
        $this->publisher->publish(
            'orders',
            'order.created',
            $event->toPayload()
        );
    }
}
```

### 4.3 Event Consumer (Notification Service)

```php
// notification-service/app/Console/Commands/ConsumeOrderEvents.php
namespace App\Console\Commands;

use App\Services\NotificationService;
use Illuminate\Console\Command;
use PhpAmqpLib\Connection\AMQPStreamConnection;

class ConsumeOrderEvents extends Command
{
    protected $signature = 'rabbitmq:consume-orders';
    protected $description = 'Consume order events from RabbitMQ';
    
    public function __construct(
        private readonly NotificationService $notificationService
    ) {
        parent::__construct();
    }
    
    public function handle(): void
    {
        $connection = new AMQPStreamConnection(
            config('rabbitmq.host'),
            config('rabbitmq.port'),
            config('rabbitmq.user'),
            config('rabbitmq.password')
        );
        
        $channel = $connection->channel();
        
        $channel->exchange_declare('orders', 'topic', false, true, false);
        [$queue] = $channel->queue_declare('notification-service.orders', false, true, false, false);
        $channel->queue_bind($queue, 'orders', 'order.*');
        
        $this->info('Waiting for order events...');
        
        $callback = function ($msg) {
            $payload = json_decode($msg->body, true);
            
            $this->processEvent($payload);
            
            $msg->ack();
        };
        
        $channel->basic_qos(null, 1, null);
        $channel->basic_consume($queue, '', false, false, false, false, $callback);
        
        while ($channel->is_consuming()) {
            $channel->wait();
        }
        
        $channel->close();
        $connection->close();
    }
    
    private function processEvent(array $payload): void
    {
        match($payload['event']) {
            'order.created' => $this->notificationService->sendOrderConfirmation($payload['data']),
            'order.shipped' => $this->notificationService->sendShippingNotification($payload['data']),
            'order.cancelled' => $this->notificationService->sendCancellationNotification($payload['data']),
            default => $this->warn("Unknown event: {$payload['event']}"),
        };
    }
}
```

---

## 5. Service Discovery

```php
// app/Services/Discovery/ConsulServiceDiscovery.php
namespace App\Services\Discovery;

use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Http;

class ConsulServiceDiscovery
{
    private const CACHE_TTL = 30; // seconds
    
    public function __construct(
        private readonly string $consulUrl
    ) {}
    
    public function resolve(string $serviceName): string
    {
        $instances = $this->getInstances($serviceName);
        
        if (empty($instances)) {
            throw new \RuntimeException("No instances found for service: {$serviceName}");
        }
        
        // Round-robin load balancing
        $index = Cache::increment("discovery:index:{$serviceName}") % count($instances);
        $instance = $instances[$index];
        
        return "http://{$instance['Address']}:{$instance['Port']}";
    }
    
    private function getInstances(string $serviceName): array
    {
        return Cache::remember("discovery:instances:{$serviceName}", self::CACHE_TTL, function () use ($serviceName) {
            $response = Http::get("{$this->consulUrl}/v1/health/service/{$serviceName}", [
                'passing' => true,
            ]);
            
            if (!$response->successful()) {
                return [];
            }
            
            return array_map(fn($entry) => [
                'Address' => $entry['Service']['Address'],
                'Port' => $entry['Service']['Port'],
            ], $response->json());
        });
    }
    
    public function register(string $serviceName, string $host, int $port): void
    {
        Http::put("{$this->consulUrl}/v1/agent/service/register", [
            'ID' => $serviceName . '-' . uniqid(),
            'Name' => $serviceName,
            'Address' => $host,
            'Port' => $port,
            'Check' => [
                'HTTP' => "http://{$host}:{$port}/health",
                'Interval' => '10s',
                'Timeout' => '3s',
            ],
        ]);
    }
}
```

---

## 6. Workshop: แยก Monolith Laravel เป็น Microservices

### Step 1: วิเคราะห์ Monolith

```
Monolith Laravel E-commerce:
├── Users Module (Authentication, Profiles)
├── Products Module (Catalog, Inventory)
├── Orders Module (Cart, Checkout, History)
├── Payments Module (Processing, Refunds)
└── Notifications Module (Email, SMS, Push)
```

**Bounded Contexts (แต่ละ Domain แยกกัน):**
```
1. Identity Service - Users, Auth, Roles
2. Catalog Service - Products, Categories, Inventory  
3. Order Service - Orders, Cart, Checkout
4. Payment Service - Payment Processing, Refunds
5. Notification Service - Emails, SMS, Push
6. API Gateway - Routing, Auth, Rate Limiting
```

### Step 2: Strangler Fig Pattern

แยก Services ทีละส่วนโดยไม่ทำลาย Monolith

```
Phase 1: Extract Notification Service
├── Monolith ยังทำ Notification อยู่
├── สร้าง Notification Service ใหม่
└── Monolith ส่ง events ไป Queue แทน

Phase 2: Extract Product Catalog  
├── สร้าง Product Catalog Service
├── API Gateway route /products/* ไป Service ใหม่
└── Monolith ใช้ HTTP Client เรียก Product Service

Phase 3: Extract Order Service
└── ทำซ้ำ pattern เดิม
```

### Step 3: สร้าง Identity Service

```php
// identity-service/app/Http/Controllers/AuthController.php
namespace App\Http\Controllers;

use App\DTOs\LoginData;
use App\DTOs\RegisterData;
use App\Services\AuthService;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class AuthController extends Controller
{
    public function __construct(
        private readonly AuthService $authService
    ) {}
    
    public function register(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'name' => 'required|string|min:2',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8|confirmed',
        ]);
        
        $result = $this->authService->register(
            RegisterData::fromArray($validated)
        );
        
        return response()->json($result, 201);
    }
    
    public function login(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);
        
        $result = $this->authService->login(
            LoginData::fromArray($validated)
        );
        
        return response()->json($result);
    }
    
    public function me(Request $request): JsonResponse
    {
        return response()->json([
            'user' => $request->user(),
            'permissions' => $request->user()->getAllPermissions()->pluck('name'),
            'roles' => $request->user()->getRoleNames(),
        ]);
    }
    
    public function verify(Request $request): JsonResponse
    {
        // ใช้โดย Internal Services เพื่อ verify token
        $token = $request->bearerToken();
        
        if (!$token) {
            return response()->json(['valid' => false], 401);
        }
        
        $user = $this->authService->verifyToken($token);
        
        if (!$user) {
            return response()->json(['valid' => false], 401);
        }
        
        return response()->json([
            'valid' => true,
            'user' => [
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
                'roles' => $user->getRoleNames(),
                'permissions' => $user->getAllPermissions()->pluck('name'),
            ],
        ]);
    }
}
```

### Step 4: สร้าง Docker Compose สำหรับ Microservices

```yaml
# docker-compose.yml
version: '3.8'

services:
  gateway:
    build: ./gateway
    ports:
      - "80:80"
    environment:
      USER_SERVICE_URL: http://identity-service:8000
      ORDER_SERVICE_URL: http://order-service:8001
      PRODUCT_SERVICE_URL: http://product-service:8002
    depends_on:
      - identity-service
      - order-service
      - product-service
    networks:
      - microservices
  
  identity-service:
    build: ./identity-service
    ports:
      - "8000:8000"
    environment:
      APP_PORT: 8000
      DB_HOST: users-db
      DB_DATABASE: users
      REDIS_HOST: redis
    depends_on:
      - users-db
      - redis
    networks:
      - microservices
  
  order-service:
    build: ./order-service
    ports:
      - "8001:8001"
    environment:
      APP_PORT: 8001
      DB_HOST: orders-db
      DB_DATABASE: orders
      USER_SERVICE_URL: http://identity-service:8000
      PRODUCT_SERVICE_URL: http://product-service:8002
      RABBITMQ_HOST: rabbitmq
    depends_on:
      - orders-db
      - rabbitmq
    networks:
      - microservices
  
  product-service:
    build: ./product-service
    ports:
      - "8002:8002"
    environment:
      APP_PORT: 8002
      DB_HOST: products-db
      DB_DATABASE: products
      ELASTICSEARCH_URL: http://elasticsearch:9200
    depends_on:
      - products-db
    networks:
      - microservices
  
  notification-service:
    build: ./notification-service
    command: php artisan rabbitmq:consume-orders
    environment:
      RABBITMQ_HOST: rabbitmq
      MAIL_HOST: mailhog
    depends_on:
      - rabbitmq
    networks:
      - microservices
  
  # Databases (แยกกันสำหรับแต่ละ service)
  users-db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: users
      MYSQL_ROOT_PASSWORD: password
    volumes:
      - users_db:/var/lib/mysql
    networks:
      - microservices
  
  orders-db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: orders
      MYSQL_ROOT_PASSWORD: password
    volumes:
      - orders_db:/var/lib/mysql
    networks:
      - microservices
  
  products-db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: products
      MYSQL_ROOT_PASSWORD: password
    volumes:
      - products_db:/var/lib/mysql
    networks:
      - microservices
  
  # Infrastructure
  redis:
    image: redis:7-alpine
    networks:
      - microservices
  
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"
    networks:
      - microservices
  
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      discovery.type: single-node
    networks:
      - microservices

networks:
  microservices:
    driver: bridge

volumes:
  users_db:
  orders_db:
  products_db:
```

### Step 5: Distributed Tracing

```php
// app/Http/Middleware/TraceRequest.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Str;
use Illuminate\Support\Facades\Log;

class TraceRequest
{
    public function handle(Request $request, Closure $next)
    {
        // รับหรือสร้าง Trace ID
        $traceId = $request->header('X-Trace-ID', (string) Str::uuid());
        $spanId = (string) Str::uuid();
        
        // เพิ่มใน Log context
        Log::withContext([
            'trace_id' => $traceId,
            'span_id' => $spanId,
            'service' => config('app.name'),
        ]);
        
        $response = $next($request);
        
        // ส่ง Trace ID ไปใน response
        return $response
            ->header('X-Trace-ID', $traceId)
            ->header('X-Span-ID', $spanId);
    }
}

// ใน HTTP Client Calls - forward trace headers
Http::withHeaders([
    'X-Trace-ID' => request()->header('X-Trace-ID'),
])->get($url);
```

---

## 7. Health Checks

```php
// app/Http/Controllers/HealthController.php
namespace App\Http\Controllers;

use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\DB;

class HealthController extends Controller
{
    public function check(): JsonResponse
    {
        $checks = [
            'database' => $this->checkDatabase(),
            'cache' => $this->checkCache(),
            'queue' => $this->checkQueue(),
        ];
        
        $allHealthy = collect($checks)->every('ok');
        $status = $allHealthy ? 200 : 503;
        
        return response()->json([
            'status' => $allHealthy ? 'healthy' : 'unhealthy',
            'service' => config('app.name'),
            'version' => config('app.version', '1.0.0'),
            'timestamp' => now()->toISOString(),
            'checks' => $checks,
        ], $status);
    }
    
    private function checkDatabase(): array
    {
        try {
            DB::select('SELECT 1');
            return ['ok' => true, 'message' => 'Connected'];
        } catch (\Exception $e) {
            return ['ok' => false, 'message' => $e->getMessage()];
        }
    }
    
    private function checkCache(): array
    {
        try {
            Cache::put('health:check', 'ok', 5);
            $value = Cache::get('health:check');
            
            return [
                'ok' => $value === 'ok',
                'message' => $value === 'ok' ? 'Working' : 'Cache mismatch',
            ];
        } catch (\Exception $e) {
            return ['ok' => false, 'message' => $e->getMessage()];
        }
    }
    
    private function checkQueue(): array
    {
        try {
            $failedCount = DB::table('failed_jobs')->count();
            
            return [
                'ok' => $failedCount < 50,
                'message' => "{$failedCount} failed jobs",
            ];
        } catch (\Exception $e) {
            return ['ok' => false, 'message' => $e->getMessage()];
        }
    }
}
```

---

## Quiz

**ข้อ 1:** Circuit Breaker Pattern แก้ปัญหาอะไร?

a) Load Balancing ระหว่าง Services  
b) ป้องกัน Cascading Failures เมื่อ Service ใดหยุดทำงาน  
c) เพิ่ม Throughput  
d) จัดการ Authentication

**เฉลย:** b) Circuit Breaker ป้องกัน Cascading Failures โดย "เปิดวงจร" เมื่อ Service fail บ่อยเกินไป ทำให้ไม่ต้องรอ Timeout ทุกครั้ง

---

**ข้อ 2:** Strangler Fig Pattern คืออะไร?

a) Pattern สำหรับ Database Migration  
b) วิธีแยก Monolith เป็น Microservices ทีละส่วน โดยค่อยๆ แทนที่ด้วย Services ใหม่  
c) Security Pattern  
d) Caching Strategy

**เฉลย:** b) Strangler Fig Pattern ค่อยๆ แทนที่ส่วนต่างๆ ของ Monolith ด้วย Services ใหม่ทีละส่วน ไม่ต้อง Rewrite ทั้งหมดพร้อมกัน

---

**ข้อ 3:** เหตุใดแต่ละ Microservice ควรมี Database แยกกัน?

a) เพื่อประหยัด Cost  
b) เพื่อ Loose Coupling ทำให้แต่ละ Service เปลี่ยน Database ได้อิสระ  
c) เพื่อ Performance  
d) ข้อจำกัดของ MySQL

**เฉลย:** b) Database แยกทำให้แต่ละ Service เป็น Fully Independent สามารถเปลี่ยน Database technology ได้ และ Scale แยกกันได้ ลด Coupling ระหว่าง Services

---

**ข้อ 4:** ข้อเสียหลักของ Microservices คืออะไร?

a) ช้ากว่า Monolith เสมอ  
b) Complexity สูงขึ้น: Network calls, Distributed transactions, Service discovery, Monitoring  
c) ใช้ PHP ไม่ได้  
d) ต้องใช้ Kubernetes เท่านั้น

**เฉลย:** b) Microservices เพิ่ม Operational Complexity อย่างมาก เหมาะกับทีมใหญ่ที่ต้องการ Scale แต่ละส่วนอิสระ ไม่เหมาะกับ Startup ขนาดเล็ก

---

## สรุปหลักสูตร Parts 043-050

คุณได้เรียนรู้เนื้อหาขั้นสูงสำหรับ Laravel Developer ระดับมืออาชีพ:

| Part | หัวข้อ | ระดับ |
|------|--------|-------|
| 043 | Caching (Redis, Tags, Rate Limiting) | สูง |
| 044 | Broadcasting (WebSocket, Real-time Chat) | มืออาชีพ |
| 045 | Livewire (Reactive UI, Shopping Cart) | มืออาชีพ |
| 046 | Package Development (Thai Address) | ระดับโลก |
| 047 | Performance (Octane, Horizon, Telescope) | ระดับโลก |
| 048 | Deployment (Forge, Docker, CI/CD) | มืออาชีพ |
| 049 | Advanced Patterns (Repository, DTO, Actions) | ระดับโลก |
| 050 | Microservices (API Gateway, Message Queue) | ระดับโลก |

**ขั้นตอนถัดไป:**
- ฝึกสร้าง Project จริงที่ใช้ Patterns เหล่านี้
- ศึกษา Domain-Driven Design (DDD) เพิ่มเติม
- เรียนรู้ Kubernetes สำหรับ Container Orchestration
- ทดลองใช้ Event Sourcing และ CQRS
- ศึกษา GraphQL กับ Laravel

---

## ทรัพยากรเพิ่มเติม

- [Laravel Documentation](https://laravel.com/docs)
- [Laracasts](https://laracasts.com)
- [Laravel News](https://laravel-news.com)
- [Spatie Packages](https://spatie.be/open-source)
- [Martin Fowler - Microservices](https://martinfowler.com/articles/microservices.html)
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
