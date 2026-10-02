# Part 90: Microservices ด้วย PHP/Laravel

## บทนำ

Microservices Architecture คือการแบ่ง Application ขนาดใหญ่ออกเป็น Services ขนาดเล็กๆ ที่ทำงานอิสระ แต่ละ Service:
- มี Business Capability ของตัวเอง
- Deploy ได้อิสระ
- มี Database ของตัวเอง
- สื่อสารผ่าน APIs หรือ Message Queues

---

## โครงสร้าง Microservices

```
┌─────────────────────────────────────────────────────────┐
│                    API Gateway (Kong/Nginx)              │
└──────┬──────────────┬──────────────────┬────────────────┘
       │              │                  │
┌──────▼──────┐ ┌─────▼──────┐  ┌───────▼───────┐
│  User       │ │  Product   │  │  Order        │
│  Service   │ │  Service   │  │  Service      │
│  :8001     │ │  :8002     │  │  :8003        │
└──────┬──────┘ └─────┬──────┘  └───────┬───────┘
       │              │                  │
┌──────▼──────┐ ┌─────▼──────┐  ┌───────▼───────┐
│  MySQL      │ │  MySQL     │  │  MySQL        │
│  users_db  │ │  products_db│  │  orders_db    │
└─────────────┘ └────────────┘  └───────────────┘
                                        │
                              ┌─────────▼─────────┐
                              │   Message Queue   │
                              │   (RabbitMQ)      │
                              └───────────────────┘
```

---

## User Service

```php
<?php

// User Service - app/Http/Controllers/UserController.php
namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\{Request, JsonResponse};
use Illuminate\Support\Facades\Hash;

class UserController extends Controller
{
    public function register(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8|confirmed',
        ]);
        
        $user = User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => Hash::make($validated['password']),
        ]);
        
        // Publish event to Message Queue
        app(MessageBus::class)->publish('user.registered', [
            'user_id' => $user->id,
            'email' => $user->email,
            'name' => $user->name,
        ]);
        
        return response()->json([
            'id' => $user->id,
            'name' => $user->name,
            'email' => $user->email,
        ], 201);
    }
    
    public function show(int $id): JsonResponse
    {
        $user = User::findOrFail($id);
        
        return response()->json([
            'id' => $user->id,
            'name' => $user->name,
            'email' => $user->email,
            'created_at' => $user->created_at->toISOString(),
        ]);
    }
    
    public function authenticate(Request $request): JsonResponse
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);
        
        $user = User::where('email', $request->email)->first();
        
        if (!$user || !Hash::check($request->password, $user->password)) {
            return response()->json(['message' => 'Invalid credentials'], 401);
        }
        
        $token = $user->createToken('api-token')->plainTextToken;
        
        return response()->json([
            'token' => $token,
            'user' => [
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
            ]
        ]);
    }
}
```

---

## Service Communication

### REST Communication ระหว่าง Services

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Http;

class ProductServiceClient
{
    private string $baseUrl;
    private int $timeout;
    
    public function __construct()
    {
        $this->baseUrl = config('services.product.url', 'http://product-service:8002');
        $this->timeout = config('services.product.timeout', 5);
    }
    
    public function findProduct(string $productId): ?array
    {
        try {
            $response = Http::timeout($this->timeout)
                ->retry(3, 100)
                ->withToken($this->getServiceToken())
                ->get("{$this->baseUrl}/api/products/{$productId}");
            
            if ($response->notFound()) {
                return null;
            }
            
            $response->throw();
            
            return $response->json();
        } catch (\Exception $e) {
            report($e);
            throw new ServiceUnavailableException("Product service unavailable: {$e->getMessage()}");
        }
    }
    
    public function checkStock(string $variantId, int $quantity): bool
    {
        try {
            $response = Http::timeout($this->timeout)
                ->withToken($this->getServiceToken())
                ->post("{$this->baseUrl}/api/inventory/check", [
                    'variant_id' => $variantId,
                    'quantity' => $quantity,
                ]);
            
            return $response->json('available', false);
        } catch (\Exception $e) {
            // Circuit Breaker pattern
            return $this->fallbackStockCheck($variantId, $quantity);
        }
    }
    
    public function reserveStock(string $variantId, int $quantity, string $orderId): bool
    {
        $response = Http::timeout($this->timeout)
            ->withToken($this->getServiceToken())
            ->post("{$this->baseUrl}/api/inventory/reserve", [
                'variant_id' => $variantId,
                'quantity' => $quantity,
                'order_id' => $orderId,
            ]);
        
        return $response->successful();
    }
    
    private function getServiceToken(): string
    {
        return cache()->remember('service.product.token', 3600, function () {
            $response = Http::post(config('services.auth.url') . '/service-token', [
                'service' => 'order-service',
                'secret' => config('services.order.secret'),
            ]);
            
            return $response->json('token');
        });
    }
    
    private function fallbackStockCheck(string $variantId, int $quantity): bool
    {
        // Fallback: ถามจาก Cache
        return cache()->get("stock.{$variantId}", 0) >= $quantity;
    }
}

// Circuit Breaker Pattern
class CircuitBreaker
{
    private int $failureCount = 0;
    private int $successCount = 0;
    private string $state = 'closed'; // closed, open, half-open
    private ?float $lastFailureTime = null;
    
    public function __construct(
        private int $failureThreshold = 5,
        private int $successThreshold = 2,
        private int $timeout = 60 // seconds
    ) {}
    
    public function call(callable $service): mixed
    {
        if ($this->state === 'open') {
            if ($this->shouldAttemptReset()) {
                $this->state = 'half-open';
            } else {
                throw new CircuitOpenException("Circuit breaker is OPEN");
            }
        }
        
        try {
            $result = $service();
            $this->onSuccess();
            return $result;
        } catch (\Exception $e) {
            $this->onFailure();
            throw $e;
        }
    }
    
    private function onSuccess(): void
    {
        $this->failureCount = 0;
        
        if ($this->state === 'half-open') {
            $this->successCount++;
            if ($this->successCount >= $this->successThreshold) {
                $this->state = 'closed';
                $this->successCount = 0;
            }
        }
    }
    
    private function onFailure(): void
    {
        $this->failureCount++;
        $this->lastFailureTime = microtime(true);
        
        if ($this->failureCount >= $this->failureThreshold) {
            $this->state = 'open';
        }
    }
    
    private function shouldAttemptReset(): bool
    {
        return $this->lastFailureTime !== null
            && (microtime(true) - $this->lastFailureTime) >= $this->timeout;
    }
    
    public function getState(): string
    {
        return $this->state;
    }
}

// การใช้งาน Circuit Breaker
class ResilientProductClient
{
    private CircuitBreaker $circuitBreaker;
    
    public function __construct(private ProductServiceClient $client)
    {
        $this->circuitBreaker = new CircuitBreaker(
            failureThreshold: 5,
            timeout: 30
        );
    }
    
    public function findProduct(string $id): ?array
    {
        try {
            return $this->circuitBreaker->call(
                fn() => $this->client->findProduct($id)
            );
        } catch (CircuitOpenException $e) {
            // Return cached data when circuit is open
            return cache()->get("product.{$id}");
        }
    }
}
```

---

## Message Queue Communication

```php
<?php

namespace App\Services;

use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

class RabbitMQMessageBus
{
    private AMQPStreamConnection $connection;
    private \PhpAmqpLib\Channel\AMQPChannel $channel;
    
    public function __construct()
    {
        $this->connection = new AMQPStreamConnection(
            config('rabbitmq.host'),
            config('rabbitmq.port'),
            config('rabbitmq.user'),
            config('rabbitmq.password')
        );
        $this->channel = $this->connection->channel();
    }
    
    public function publish(string $exchange, string $routingKey, array $data): void
    {
        $this->channel->exchange_declare($exchange, 'topic', false, true, false);
        
        $message = new AMQPMessage(
            json_encode($data),
            [
                'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT,
                'content_type' => 'application/json',
                'timestamp' => time(),
                'message_id' => uniqid('msg_', true),
            ]
        );
        
        $this->channel->basic_publish($message, $exchange, $routingKey);
    }
    
    public function subscribe(string $exchange, string $queue, string $pattern, callable $handler): void
    {
        $this->channel->exchange_declare($exchange, 'topic', false, true, false);
        $this->channel->queue_declare($queue, false, true, false, false);
        $this->channel->queue_bind($queue, $exchange, $pattern);
        
        $this->channel->basic_qos(null, 1, null);
        $this->channel->basic_consume(
            $queue,
            '',
            false,
            false,
            false,
            false,
            function (AMQPMessage $message) use ($handler) {
                try {
                    $data = json_decode($message->body, true);
                    $handler($data);
                    $message->ack();
                } catch (\Exception $e) {
                    report($e);
                    // Nack and requeue หรือ send to dead letter queue
                    $message->nack(true);
                }
            }
        );
        
        while ($this->channel->is_consuming()) {
            $this->channel->wait();
        }
    }
    
    public function __destruct()
    {
        $this->channel->close();
        $this->connection->close();
    }
}

// Event Publishers
class OrderEventPublisher
{
    public function __construct(private RabbitMQMessageBus $bus) {}
    
    public function publishOrderPlaced(array $order): void
    {
        $this->bus->publish('orders', 'order.placed', [
            'event' => 'order.placed',
            'order_id' => $order['id'],
            'customer_id' => $order['customer_id'],
            'items' => $order['items'],
            'total' => $order['total'],
            'occurred_at' => now()->toISOString(),
        ]);
    }
    
    public function publishOrderCancelled(string $orderId, string $reason): void
    {
        $this->bus->publish('orders', 'order.cancelled', [
            'event' => 'order.cancelled',
            'order_id' => $orderId,
            'reason' => $reason,
            'occurred_at' => now()->toISOString(),
        ]);
    }
}

// Event Consumers in Notification Service
class NotificationConsumer
{
    public function __construct(
        private RabbitMQMessageBus $bus,
        private EmailService $email
    ) {}
    
    public function listen(): void
    {
        $this->bus->subscribe(
            exchange: 'orders',
            queue: 'notification-service',
            pattern: 'order.*',
            handler: function (array $event) {
                match($event['event']) {
                    'order.placed' => $this->handleOrderPlaced($event),
                    'order.cancelled' => $this->handleOrderCancelled($event),
                    'order.shipped' => $this->handleOrderShipped($event),
                    default => null,
                };
            }
        );
    }
    
    private function handleOrderPlaced(array $event): void
    {
        // ส่ง Email ยืนยันคำสั่งซื้อ
        $this->email->send(
            to: $this->getUserEmail($event['customer_id']),
            subject: "ยืนยันคำสั่งซื้อ #{$event['order_id']}",
            template: 'order-confirmation',
            data: $event
        );
    }
    
    private function handleOrderCancelled(array $event): void
    {
        $this->email->send(
            to: $this->getUserEmail($event['customer_id']),
            subject: "คำสั่งซื้อ #{$event['order_id']} ถูกยกเลิก",
            template: 'order-cancelled',
            data: $event
        );
    }
}
```

---

## Docker & Kubernetes

### docker-compose.yml สำหรับ Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  # API Gateway
  nginx-gateway:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/gateway.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - user-service
      - product-service
      - order-service

  # User Service
  user-service:
    build:
      context: ./user-service
      dockerfile: Dockerfile
    environment:
      - DB_HOST=user-db
      - DB_DATABASE=users_db
      - RABBITMQ_HOST=rabbitmq
    depends_on:
      - user-db
      - rabbitmq
    volumes:
      - ./user-service:/var/www/html

  user-db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: users_db
      MYSQL_ROOT_PASSWORD: secret
    volumes:
      - user-db-data:/var/lib/mysql

  # Product Service
  product-service:
    build:
      context: ./product-service
      dockerfile: Dockerfile
    environment:
      - DB_HOST=product-db
      - DB_DATABASE: products_db
      - RABBITMQ_HOST: rabbitmq
    depends_on:
      - product-db
      - rabbitmq

  product-db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: products_db
      MYSQL_ROOT_PASSWORD: secret

  # Order Service
  order-service:
    build:
      context: ./order-service
      dockerfile: Dockerfile
    environment:
      - DB_HOST: order-db
      - DB_DATABASE: orders_db
      - PRODUCT_SERVICE_URL: http://product-service
      - USER_SERVICE_URL: http://user-service

  order-db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: orders_db
      MYSQL_ROOT_PASSWORD: secret

  # Message Queue
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: secret

  # Redis (Shared Cache)
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  user-db-data:
  product-db-data:
  order-db-data:
```

### Dockerfile สำหรับ Laravel Service

```dockerfile
# Dockerfile
FROM php:8.3-fpm-alpine

# Install dependencies
RUN apk add --no-cache \
    nginx \
    supervisor \
    nodejs \
    npm \
    git \
    curl

# Install PHP extensions
RUN docker-php-ext-install \
    pdo_mysql \
    opcache \
    pcntl \
    bcmath

# Install Redis extension
RUN pecl install redis && docker-php-ext-enable redis

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

# Copy application
COPY . .

# Install dependencies
RUN composer install --no-dev --optimize-autoloader

# Set permissions
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/storage

# Copy nginx config
COPY docker/nginx.conf /etc/nginx/http.d/default.conf

# Copy supervisor config
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

EXPOSE 80

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

---

## API Gateway

```nginx
# nginx/gateway.conf
upstream user-service {
    server user-service:9000;
}

upstream product-service {
    server product-service:9000;
}

upstream order-service {
    server order-service:9000;
}

server {
    listen 80;
    
    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
    limit_req zone=api burst=20 nodelay;
    
    # User Service
    location /api/users {
        proxy_pass http://user-service;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    location /api/auth {
        proxy_pass http://user-service;
        proxy_set_header Host $host;
    }
    
    # Product Service
    location /api/products {
        proxy_pass http://product-service;
        proxy_set_header Host $host;
    }
    
    location /api/inventory {
        proxy_pass http://product-service;
        proxy_set_header Host $host;
    }
    
    # Order Service
    location /api/orders {
        # Require authentication
        auth_request /auth/verify;
        
        proxy_pass http://order-service;
        proxy_set_header Host $host;
    }
    
    # Auth verification endpoint
    location = /auth/verify {
        internal;
        proxy_pass http://user-service/api/auth/verify;
        proxy_set_header Authorization $http_authorization;
    }
}
```

---

## Service Discovery

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\{Cache, Http};

class ServiceRegistry
{
    private array $services = [];
    
    public function register(string $name, string $host, int $port): void
    {
        $this->services[$name] = [
            'host' => $host,
            'port' => $port,
            'url' => "http://{$host}:{$port}",
            'registered_at' => now()->toISOString(),
        ];
        
        Cache::put("service.{$name}", $this->services[$name], 300);
    }
    
    public function discover(string $name): ?string
    {
        $service = Cache::get("service.{$name}");
        
        if (!$service) {
            // Try Consul/Kubernetes DNS
            $host = $this->resolveFromDns($name);
            if ($host) {
                return "http://{$host}";
            }
            return null;
        }
        
        return $service['url'];
    }
    
    private function resolveFromDns(string $name): ?string
    {
        // ใน Kubernetes: service-name.namespace.svc.cluster.local
        $hosts = [
            "{$name}-service",
            "{$name}.default.svc.cluster.local",
        ];
        
        foreach ($hosts as $host) {
            if (@dns_get_record($host, DNS_A)) {
                return $host;
            }
        }
        
        return null;
    }
    
    public function healthCheck(string $name): bool
    {
        $url = $this->discover($name);
        if (!$url) return false;
        
        try {
            $response = Http::timeout(2)->get("{$url}/health");
            return $response->successful();
        } catch (\Exception $e) {
            return false;
        }
    }
}

// Health Check Endpoint
class HealthController extends Controller
{
    public function check(): JsonResponse
    {
        $checks = [
            'database' => $this->checkDatabase(),
            'cache' => $this->checkCache(),
            'queue' => $this->checkQueue(),
        ];
        
        $healthy = !in_array(false, $checks, true);
        
        return response()->json([
            'status' => $healthy ? 'healthy' : 'unhealthy',
            'service' => config('app.name'),
            'version' => config('app.version', '1.0.0'),
            'checks' => $checks,
            'timestamp' => now()->toISOString(),
        ], $healthy ? 200 : 503);
    }
    
    private function checkDatabase(): bool
    {
        try {
            \DB::select('SELECT 1');
            return true;
        } catch (\Exception $e) {
            return false;
        }
    }
    
    private function checkCache(): bool
    {
        try {
            cache()->set('health_check', true, 10);
            return cache()->get('health_check') === true;
        } catch (\Exception $e) {
            return false;
        }
    }
    
    private function checkQueue(): bool
    {
        try {
            // Check RabbitMQ connection
            return app(RabbitMQMessageBus::class)->isConnected();
        } catch (\Exception $e) {
            return false;
        }
    }
}
```

---

## Workshop: แยก Monolith เป็น Microservices

### Strangler Fig Pattern

```php
<?php

// Step 1: สร้าง Feature Toggle
class FeatureFlag
{
    public static function isEnabled(string $feature): bool
    {
        return config("features.{$feature}", false)
            || Cache::get("feature.{$feature}", false);
    }
}

// Step 2: Strangler Facade - ค่อยๆ แทนที่ Monolith
class ProductFacade
{
    public function __construct(
        private MonolithProductRepository $monolithRepo,
        private ProductServiceClient $microserviceClient
    ) {}
    
    public function findProduct(string $id): ?array
    {
        // ถ้า Feature Flag เปิด ใช้ Microservice
        if (FeatureFlag::isEnabled('product-microservice')) {
            return $this->microserviceClient->findProduct($id);
        }
        
        // ไม่งั้นใช้ Monolith
        return $this->monolithRepo->findById($id);
    }
    
    public function createProduct(array $data): array
    {
        if (FeatureFlag::isEnabled('product-microservice')) {
            return $this->microserviceClient->createProduct($data);
        }
        
        return $this->monolithRepo->create($data);
    }
}

// Step 3: Data Sync - เขียนข้อมูลไปทั้งสองที่ระหว่าง Migration
class DualWriteProductRepository
{
    public function __construct(
        private MonolithProductRepository $monolith,
        private ProductServiceClient $microservice
    ) {}
    
    public function create(array $data): array
    {
        // Write to monolith first (Source of truth ระหว่าง Migration)
        $product = $this->monolith->create($data);
        
        // Async write to microservice
        dispatch(new SyncProductToMicroservice($product));
        
        return $product;
    }
}

// Step 4: Migration Script
class MigrateProductsCommand extends Command
{
    protected $signature = 'migrate:products {--batch=100}';
    
    public function handle(): void
    {
        $batch = (int)$this->option('batch');
        $offset = 0;
        
        $this->info("Starting product migration...");
        
        do {
            $products = DB::table('products')
                ->offset($offset)
                ->limit($batch)
                ->get();
            
            foreach ($products as $product) {
                try {
                    Http::post(config('services.product.url') . '/api/products/import', [
                        'id' => $product->id,
                        'name' => $product->name,
                        'price' => $product->price,
                        // ... other fields
                    ]);
                    
                    $this->info("Migrated product: {$product->id}");
                } catch (\Exception $e) {
                    $this->error("Failed to migrate product {$product->id}: {$e->getMessage()}");
                }
            }
            
            $offset += $batch;
            
        } while ($products->count() === $batch);
        
        $this->info("Migration completed!");
    }
}
```

---

## Kubernetes Deployment

```yaml
# k8s/order-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:latest
        ports:
        - containerPort: 80
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: order-db-secret
              key: host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: order-db-secret
              key: password
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: ecommerce
spec:
  selector:
    app: order-service
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: ecommerce
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## สรุป

| หัวข้อ | ใช้เมื่อ | ข้อควรระวัง |
|--------|---------|------------|
| REST | Simple request/response | Tight coupling, Latency |
| gRPC | High performance, Internal | Complex setup |
| Message Queue | Async, Decoupled | Eventual consistency |
| API Gateway | Single entry point | SPOF ถ้าไม่ทำ HA |
| Circuit Breaker | Fault tolerance | State management |
| Service Registry | Dynamic discovery | Overhead |
| Strangler Fig | Migration from Monolith | Dual-write complexity |

---

*Microservices เพิ่ม Complexity - ใช้เมื่อ Team ใหญ่พอและ Monolith กลายเป็น Bottleneck จริงๆ*
