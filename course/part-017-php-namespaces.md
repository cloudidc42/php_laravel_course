# Part 017: PHP Namespaces และ Autoloading

## ระดับ: Intermediate
## เวลาเรียน: 3-4 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:
- เข้าใจและใช้งาน Namespaces ได้ถูกต้อง
- ใช้ `use` statements ได้อย่างคล่องแคล่ว
- ตั้งค่า PSR-4 Autoloading
- ใช้ Composer Autoload
- สร้าง Library ที่มี Namespace ถูกต้อง

---

## 1. ทำไมต้องใช้ Namespaces?

### ปัญหาก่อนมี Namespaces

```php
<?php
// ไฟล์ library/database.php
class Database {
    public function connect() { echo "Library DB\n"; }
}

// ไฟล์ myapp/database.php  
class Database {
    public function connect() { echo "My App DB\n"; }
}

// ไฟล์ main.php
require 'library/database.php';
require 'myapp/database.php'; // Fatal Error! Cannot redeclare class Database

$db = new Database(); // คลาสไหน?
```

### วิธีแก้: ใช้ Namespaces

```php
<?php
// ไฟล์ library/database.php
namespace Library;

class Database {
    public function connect() { echo "Library DB\n"; }
}

// ไฟล์ myapp/database.php
namespace MyApp;

class Database {
    public function connect() { echo "My App DB\n"; }
}

// ไฟล์ main.php
require 'library/database.php';
require 'myapp/database.php';

$libDb = new \Library\Database();
$appDb = new \MyApp\Database();

$libDb->connect(); // Library DB
$appDb->connect(); // My App DB
```

---

## 2. Namespace Syntax

### การประกาศ Namespace

```php
<?php
// Namespace ต้องอยู่บรรทัดแรก (หลังจาก <?php)
namespace MyApp\Services;

class UserService {
    // ...
}

class EmailService {
    // ...
}

// ไฟล์เดียวกันมีหลาย Namespace ได้ (แต่ไม่แนะนำ)
// ใช้ syntax แบบ block
namespace MyApp\Helpers {
    class StringHelper { }
    class ArrayHelper { }
}

namespace MyApp\Models {
    class User { }
}
```

### Namespace Hierarchy

```php
<?php
// โครงสร้าง Namespace แบบ hierarchy
namespace Vendor\Package\SubPackage;

// เทียบเท่ากับ directory structure:
// src/
//   Vendor/
//     Package/
//       SubPackage/
//         ClassName.php

class ClassName {
    public function hello(): string {
        // __NAMESPACE__ = 'Vendor\Package\SubPackage'
        return "In namespace: " . __NAMESPACE__;
    }
}

// เรียกใช้งาน
$obj = new \Vendor\Package\SubPackage\ClassName();
echo $obj->hello() . "\n";

// หรือใช้ use statement
use Vendor\Package\SubPackage\ClassName;
$obj2 = new ClassName();
```

### Global Namespace

```php
<?php
namespace MyApp;

// คลาส PHP built-in อยู่ใน global namespace
// ต้องใช้ backslash นำหน้า
$dt = new \DateTime();
$arr = new \ArrayObject([1, 2, 3]);

// Functions และ constants ในทุก namespace
// ถ้าไม่เจอใน current namespace PHP จะ fallback ไป global
$length = strlen("hello"); // หา strlen ใน MyApp ก่อน ถ้าไม่เจอไปหาใน global
$result = \strlen("hello"); // บังคับใช้ global namespace

// Constants
echo PHP_EOL;    // PHP constant ใน global namespace
echo \PHP_EOL;   // แบบชัดเจน

// ประกาศ constant ใน namespace
define('MyApp\APP_NAME', 'My Application');
const VERSION = '1.0.0';

echo MyApp\APP_NAME . "\n";
echo VERSION . "\n";
```

---

## 3. use Statements

### use สำหรับ Classes

```php
<?php
namespace MyApp;

use DateTime;                           // import จาก global namespace
use DateInterval;
use Exception;
use RuntimeException;

// use กับ fully qualified name
use Vendor\Package\SomeClass;
use AnotherVendor\AnotherPackage\Controller;

// Aliasing - ตั้งชื่อใหม่ให้คลาส
use Vendor\Package\SomeClass as VendorClass;
use MyApp\Models\User as UserModel;
use AnotherVendor\AnotherPackage\Controller as BaseController;

class HomeController extends BaseController {
    public function index(): void {
        $now = new DateTime(); // ใช้ DateTime ได้โดยตรง (ไม่ต้อง \DateTime)
        $user = new UserModel();
        $vendor = new VendorClass();
        
        echo $now->format('Y-m-d') . "\n";
    }
}
```

### use สำหรับ Functions และ Constants

```php
<?php
namespace MyApp;

// use function - import function
use function array_map;
use function json_encode;
use function MyApp\Helpers\formatDate;

// use const - import constant
use const PHP_EOL;
use const MyApp\Config\DATABASE_HOST;

// Grouping (PHP 7+)
use MyApp\Models\{User, Post, Comment};
use MyApp\Services\{UserService, EmailService};
use function MyApp\Helpers\{formatDate, formatMoney};
use const MyApp\Config\{DB_HOST, DB_PORT, DB_NAME};

class Application {
    public function run(): void {
        $users = [new User(), new Post()];
        $service = new UserService();
        
        // ใช้ function ที่ import มา
        $result = array_map(fn($u) => get_class($u), $users);
        echo json_encode($result) . PHP_EOL;
    }
}
```

### use ใน Closures และ Arrow Functions

```php
<?php
namespace MyApp\Utils;

use DateTime;
use MyApp\Models\User;

// use ใน closure
$formatter = function(string $format) use (&$counter) {
    $counter++;
    return (new DateTime())->format($format);
};

// Class aliasing ช่วยให้โค้ดสั้นลง
class DateHelper {
    // ใน method body ไม่ต้องใช้ use (use ทำงานระดับ file)
    public static function today(): string {
        return (new DateTime())->format('Y-m-d');
    }
    
    public static function formatUser(User $user): string {
        return "User: {$user->getName()}";
    }
}
```

---

## 4. PSR-4 Autoloading

PSR-4 (PHP Standard Recommendation 4) คือมาตรฐานสำหรับ Autoloading ที่ map namespace กับ directory structure

### PSR-4 Rules

```
Namespace Prefix: Acme\Blog\
Base Directory:   ./src/
Fully-Qualified Class: Acme\Blog\Post\ArticlePost

File Path: ./src/Post/ArticlePost.php
```

**กฎ:**
1. Fully-qualified class name มี namespace prefix + class name
2. Namespace prefix map กับ base directory
3. Sub-namespaces map กับ sub-directories
4. Class name = filename (case-sensitive)

### โครงสร้าง Project แบบ PSR-4

```
myproject/
├── composer.json
├── src/
│   ├── Models/
│   │   ├── User.php          → MyApp\Models\User
│   │   └── Post.php          → MyApp\Models\Post
│   ├── Services/
│   │   ├── UserService.php   → MyApp\Services\UserService
│   │   └── MailService.php   → MyApp\Services\MailService
│   ├── Controllers/
│   │   └── HomeController.php → MyApp\Controllers\HomeController
│   └── Helpers/
│       └── StringHelper.php  → MyApp\Helpers\StringHelper
├── tests/
│   └── Models/
│       └── UserTest.php      → Tests\Models\UserTest
└── public/
    └── index.php
```

### ไฟล์ตัวอย่าง

```php
<?php
// src/Models/User.php
namespace MyApp\Models;

class User {
    private static int $count = 0;
    
    public function __construct(
        private string $name,
        private string $email
    ) {
        self::$count++;
    }
    
    public function getName(): string { return $this->name; }
    public function getEmail(): string { return $this->email; }
    
    public static function getCount(): int {
        return self::$count;
    }
    
    public function __toString(): string {
        return "User({$this->name}, {$this->email})";
    }
}
```

```php
<?php
// src/Services/UserService.php
namespace MyApp\Services;

use MyApp\Models\User;
use MyApp\Exceptions\UserNotFoundException;

class UserService {
    private array $users = [];
    
    public function create(string $name, string $email): User {
        $user = new User($name, $email);
        $this->users[$email] = $user;
        return $user;
    }
    
    public function findByEmail(string $email): User {
        if (!isset($this->users[$email])) {
            throw new UserNotFoundException("User not found: {$email}");
        }
        return $this->users[$email];
    }
    
    public function getAllUsers(): array {
        return $this->users;
    }
}
```

```php
<?php
// src/Exceptions/UserNotFoundException.php
namespace MyApp\Exceptions;

use RuntimeException;

class UserNotFoundException extends RuntimeException {
    public function __construct(string $message, int $code = 404) {
        parent::__construct($message, $code);
    }
}
```

### สร้าง Autoloader แบบ Manual (PSR-4)

```php
<?php
// autoload.php

spl_autoload_register(function (string $className): void {
    // แปลง namespace เป็น path
    $baseDir = __DIR__ . '/src/';
    $prefix = 'MyApp\\';
    
    // ตรวจสอบว่า class อยู่ใน namespace ที่เรารับผิดชอบ
    if (strncmp($prefix, $className, strlen($prefix)) !== 0) {
        return; // ไม่ใช่ namespace ของเรา
    }
    
    // แปลง sub-namespace เป็น directory path
    $relativeClass = substr($className, strlen($prefix));
    $file = $baseDir . str_replace('\\', '/', $relativeClass) . '.php';
    
    if (file_exists($file)) {
        require $file;
    }
});

// ทดสอบ
use MyApp\Models\User;
use MyApp\Services\UserService;

$service = new UserService();
$user = $service->create("สมชาย", "somchai@example.com");
echo $user . "\n";
```

---

## 5. Composer Autoload

วิธีที่ถูกต้องและเป็นมาตรฐานคือใช้ Composer จัดการ Autoloading

### composer.json

```json
{
    "name": "myvendor/myapp",
    "description": "My PHP Application",
    "type": "project",
    "require": {
        "php": "^8.1"
    },
    "require-dev": {
        "phpunit/phpunit": "^10.0"
    },
    "autoload": {
        "psr-4": {
            "MyApp\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    }
}
```

### ใช้งาน Composer Autoload

```php
<?php
// public/index.php หรือ bootstrap.php
require_once __DIR__ . '/../vendor/autoload.php';

use MyApp\Models\User;
use MyApp\Services\UserService;
use MyApp\Exceptions\UserNotFoundException;

// Composer จัดการ autoloading ให้อัตโนมัติ
$service = new UserService();

try {
    $user1 = $service->create("สมชาย", "somchai@example.com");
    $user2 = $service->create("สมหญิง", "somying@example.com");
    
    echo "Created: {$user1}\n";
    echo "Created: {$user2}\n";
    
    $found = $service->findByEmail("somchai@example.com");
    echo "Found: {$found}\n";
    
    $notFound = $service->findByEmail("nobody@example.com"); // จะ throw exception
    
} catch (UserNotFoundException $e) {
    echo "Error {$e->getCode()}: {$e->getMessage()}\n";
}
```

### Multiple Namespace Mappings

```json
{
    "autoload": {
        "psr-4": {
            "MyApp\\": "src/",
            "MyApp\\Models\\": "src/models/",
            "Shared\\": "lib/shared/"
        },
        "classmap": [
            "src/legacy/"
        ],
        "files": [
            "src/helpers.php",
            "src/functions.php"
        ]
    }
}
```

### Autoload Types ใน Composer

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        },
        "psr-0": {
            "LegacyApp_": "legacy/"
        },
        "classmap": [
            "database/seeds/",
            "database/factories/"
        ],
        "files": [
            "src/helpers/functions.php"
        ]
    }
}
```

**ความแตกต่าง:**
- `psr-4` - มาตรฐานใหม่ (แนะนำ)
- `psr-0` - มาตรฐานเก่า (deprecated)
- `classmap` - สแกนทุก .php ใน directory
- `files` - โหลดทุกครั้ง (ใช้สำหรับ helper functions)

---

## 6. Workshop: สร้าง Library ที่มี Namespace ถูกต้อง

### โจทย์: สร้าง HTTP Client Library

```
myhttp/
├── composer.json
├── src/
│   ├── HttpClient.php
│   ├── Request.php
│   ├── Response.php
│   ├── Contracts/
│   │   ├── HttpClientInterface.php
│   │   └── MiddlewareInterface.php
│   ├── Middleware/
│   │   ├── AuthMiddleware.php
│   │   ├── LoggingMiddleware.php
│   │   └── RetryMiddleware.php
│   └── Exceptions/
│       ├── HttpException.php
│       ├── TimeoutException.php
│       └── NetworkException.php
└── tests/
    └── HttpClientTest.php
```

#### composer.json

```json
{
    "name": "myvendor/http-client",
    "description": "A simple HTTP client library",
    "type": "library",
    "license": "MIT",
    "authors": [
        {
            "name": "สมชาย PHP",
            "email": "somchai@example.com"
        }
    ],
    "require": {
        "php": "^8.1"
    },
    "autoload": {
        "psr-4": {
            "MyVendor\\HttpClient\\": "src/"
        }
    }
}
```

#### src/Contracts/HttpClientInterface.php

```php
<?php
namespace MyVendor\HttpClient\Contracts;

use MyVendor\HttpClient\Request;
use MyVendor\HttpClient\Response;

interface HttpClientInterface {
    public function get(string $url, array $options = []): Response;
    public function post(string $url, array $data = [], array $options = []): Response;
    public function put(string $url, array $data = [], array $options = []): Response;
    public function delete(string $url, array $options = []): Response;
    public function send(Request $request): Response;
}
```

#### src/Contracts/MiddlewareInterface.php

```php
<?php
namespace MyVendor\HttpClient\Contracts;

use MyVendor\HttpClient\Request;
use MyVendor\HttpClient\Response;

interface MiddlewareInterface {
    public function process(Request $request, callable $next): Response;
}
```

#### src/Request.php

```php
<?php
namespace MyVendor\HttpClient;

class Request {
    private array $headers = [];
    private mixed $body = null;
    
    public function __construct(
        private string $method,
        private string $url,
        private array $options = []
    ) {}
    
    public function getMethod(): string {
        return strtoupper($this->method);
    }
    
    public function getUrl(): string {
        return $this->url;
    }
    
    public function getHeaders(): array {
        return $this->headers;
    }
    
    public function withHeader(string $name, string $value): static {
        $clone = clone $this;
        $clone->headers[$name] = $value;
        return $clone;
    }
    
    public function withBody(mixed $body): static {
        $clone = clone $this;
        $clone->body = $body;
        return $clone;
    }
    
    public function getBody(): mixed {
        return $this->body;
    }
    
    public function getOption(string $key, mixed $default = null): mixed {
        return $this->options[$key] ?? $default;
    }
}
```

#### src/Response.php

```php
<?php
namespace MyVendor\HttpClient;

class Response {
    public function __construct(
        private int $statusCode,
        private string $body,
        private array $headers = []
    ) {}
    
    public function getStatusCode(): int {
        return $this->statusCode;
    }
    
    public function getBody(): string {
        return $this->body;
    }
    
    public function getHeaders(): array {
        return $this->headers;
    }
    
    public function getHeader(string $name): ?string {
        return $this->headers[strtolower($name)] ?? null;
    }
    
    public function json(): mixed {
        return json_decode($this->body, true);
    }
    
    public function isSuccess(): bool {
        return $this->statusCode >= 200 && $this->statusCode < 300;
    }
    
    public function isClientError(): bool {
        return $this->statusCode >= 400 && $this->statusCode < 500;
    }
    
    public function isServerError(): bool {
        return $this->statusCode >= 500;
    }
}
```

#### src/Exceptions/HttpException.php

```php
<?php
namespace MyVendor\HttpClient\Exceptions;

use RuntimeException;
use MyVendor\HttpClient\Response;

class HttpException extends RuntimeException {
    public function __construct(
        string $message,
        private ?Response $response = null,
        int $code = 0,
        ?\Throwable $previous = null
    ) {
        parent::__construct($message, $code, $previous);
    }
    
    public function getResponse(): ?Response {
        return $this->response;
    }
}
```

#### src/Exceptions/TimeoutException.php

```php
<?php
namespace MyVendor\HttpClient\Exceptions;

class TimeoutException extends HttpException {
    public function __construct(string $url, int $timeout) {
        parent::__construct(
            "Request to '{$url}' timed out after {$timeout} seconds",
            null,
            408
        );
    }
}
```

#### src/Middleware/LoggingMiddleware.php

```php
<?php
namespace MyVendor\HttpClient\Middleware;

use MyVendor\HttpClient\Contracts\MiddlewareInterface;
use MyVendor\HttpClient\Request;
use MyVendor\HttpClient\Response;

class LoggingMiddleware implements MiddlewareInterface {
    private array $logs = [];
    
    public function __construct(
        private bool $verbose = false
    ) {}
    
    public function process(Request $request, callable $next): Response {
        $start = microtime(true);
        
        $this->log("→ {$request->getMethod()} {$request->getUrl()}");
        
        if ($this->verbose) {
            foreach ($request->getHeaders() as $name => $value) {
                $this->log("  Header: {$name}: {$value}");
            }
        }
        
        $response = $next($request);
        
        $duration = round((microtime(true) - $start) * 1000, 2);
        $this->log("← {$response->getStatusCode()} ({$duration}ms)");
        
        return $response;
    }
    
    private function log(string $message): void {
        $entry = date('H:i:s') . " " . $message;
        $this->logs[] = $entry;
        echo $entry . "\n";
    }
    
    public function getLogs(): array {
        return $this->logs;
    }
}
```

#### src/Middleware/AuthMiddleware.php

```php
<?php
namespace MyVendor\HttpClient\Middleware;

use MyVendor\HttpClient\Contracts\MiddlewareInterface;
use MyVendor\HttpClient\Request;
use MyVendor\HttpClient\Response;

class AuthMiddleware implements MiddlewareInterface {
    public function __construct(
        private string $token,
        private string $type = 'Bearer'
    ) {}
    
    public function process(Request $request, callable $next): Response {
        $authenticatedRequest = $request->withHeader(
            'Authorization',
            "{$this->type} {$this->token}"
        );
        
        return $next($authenticatedRequest);
    }
}
```

#### src/Middleware/RetryMiddleware.php

```php
<?php
namespace MyVendor\HttpClient\Middleware;

use MyVendor\HttpClient\Contracts\MiddlewareInterface;
use MyVendor\HttpClient\Request;
use MyVendor\HttpClient\Response;
use MyVendor\HttpClient\Exceptions\HttpException;

class RetryMiddleware implements MiddlewareInterface {
    public function __construct(
        private int $maxRetries = 3,
        private int $delay = 100 // milliseconds
    ) {}
    
    public function process(Request $request, callable $next): Response {
        $attempts = 0;
        
        while ($attempts <= $this->maxRetries) {
            try {
                $response = $next($request);
                
                if (!$response->isServerError()) {
                    return $response;
                }
                
                $attempts++;
                if ($attempts <= $this->maxRetries) {
                    echo "Server error, retrying ({$attempts}/{$this->maxRetries})...\n";
                    usleep($this->delay * 1000);
                }
                
            } catch (HttpException $e) {
                $attempts++;
                if ($attempts > $this->maxRetries) {
                    throw $e;
                }
                echo "Request failed, retrying ({$attempts}/{$this->maxRetries})...\n";
                usleep($this->delay * 1000);
            }
        }
        
        throw new HttpException("Max retries ({$this->maxRetries}) exceeded for {$request->getUrl()}");
    }
}
```

#### src/HttpClient.php (Main class)

```php
<?php
namespace MyVendor\HttpClient;

use MyVendor\HttpClient\Contracts\HttpClientInterface;
use MyVendor\HttpClient\Contracts\MiddlewareInterface;
use MyVendor\HttpClient\Exceptions\HttpException;
use MyVendor\HttpClient\Exceptions\TimeoutException;

class HttpClient implements HttpClientInterface {
    private array $middlewares = [];
    private array $defaultHeaders = [
        'Content-Type' => 'application/json',
        'Accept' => 'application/json',
    ];
    private int $timeout = 30;
    
    public function addMiddleware(MiddlewareInterface $middleware): static {
        $this->middlewares[] = $middleware;
        return $this;
    }
    
    public function withDefaultHeader(string $name, string $value): static {
        $this->defaultHeaders[$name] = $value;
        return $this;
    }
    
    public function setTimeout(int $seconds): static {
        $this->timeout = $seconds;
        return $this;
    }
    
    public function get(string $url, array $options = []): Response {
        return $this->send(new Request('GET', $url, $options));
    }
    
    public function post(string $url, array $data = [], array $options = []): Response {
        $request = new Request('POST', $url, $options);
        if (!empty($data)) {
            $request = $request->withBody(json_encode($data));
        }
        return $this->send($request);
    }
    
    public function put(string $url, array $data = [], array $options = []): Response {
        $request = new Request('PUT', $url, $options);
        if (!empty($data)) {
            $request = $request->withBody(json_encode($data));
        }
        return $this->send($request);
    }
    
    public function delete(string $url, array $options = []): Response {
        return $this->send(new Request('DELETE', $url, $options));
    }
    
    public function send(Request $request): Response {
        // เพิ่ม default headers
        foreach ($this->defaultHeaders as $name => $value) {
            $request = $request->withHeader($name, $value);
        }
        
        // สร้าง middleware pipeline
        $pipeline = $this->buildPipeline(function(Request $req) {
            return $this->executeRequest($req);
        });
        
        return $pipeline($request);
    }
    
    private function buildPipeline(callable $core): callable {
        $pipeline = $core;
        
        foreach (array_reverse($this->middlewares) as $middleware) {
            $pipeline = function(Request $request) use ($middleware, $pipeline): Response {
                return $middleware->process($request, $pipeline);
            };
        }
        
        return $pipeline;
    }
    
    private function executeRequest(Request $request): Response {
        // จำลองการ HTTP request (ในการใช้งานจริงใช้ cURL หรือ stream context)
        echo "  [HTTP] Executing {$request->getMethod()} {$request->getUrl()}\n";
        
        // Mock response based on URL
        $url = $request->getUrl();
        
        if (str_contains($url, '/users')) {
            return new Response(200, json_encode([
                'users' => [
                    ['id' => 1, 'name' => 'สมชาย'],
                    ['id' => 2, 'name' => 'สมหญิง'],
                ]
            ]));
        }
        
        if (str_contains($url, '/error')) {
            return new Response(500, json_encode(['error' => 'Internal Server Error']));
        }
        
        return new Response(200, json_encode(['message' => 'OK']));
    }
}
```

#### ตัวอย่างการใช้งาน Library

```php
<?php
require_once 'vendor/autoload.php';

use MyVendor\HttpClient\HttpClient;
use MyVendor\HttpClient\Middleware\LoggingMiddleware;
use MyVendor\HttpClient\Middleware\AuthMiddleware;
use MyVendor\HttpClient\Middleware\RetryMiddleware;
use MyVendor\HttpClient\Exceptions\HttpException;

// สร้าง client พร้อม middlewares
$client = new HttpClient();
$client
    ->addMiddleware(new LoggingMiddleware(verbose: true))
    ->addMiddleware(new AuthMiddleware('my-api-token'))
    ->addMiddleware(new RetryMiddleware(maxRetries: 2))
    ->setTimeout(30);

echo "=== GET /users ===\n";
try {
    $response = $client->get('https://api.example.com/users');
    
    if ($response->isSuccess()) {
        $data = $response->json();
        echo "Users found: " . count($data['users']) . "\n";
        foreach ($data['users'] as $user) {
            echo "  - {$user['name']}\n";
        }
    }
} catch (HttpException $e) {
    echo "Error: " . $e->getMessage() . "\n";
}

echo "\n=== POST /users ===\n";
try {
    $response = $client->post('https://api.example.com/users', [
        'name' => 'วิชัย',
        'email' => 'wichai@example.com',
    ]);
    
    echo "Status: " . $response->getStatusCode() . "\n";
} catch (HttpException $e) {
    echo "Error: " . $e->getMessage() . "\n";
}
```

---

## Quiz

### คำถาม 1
namespace ต้องอยู่ที่ใดในไฟล์?
- A. บรรทัดสุดท้าย
- B. หลัง `<?php` และก่อน code อื่นๆ
- C. ก่อน `<?php`
- D. ที่ไหนก็ได้

**เฉลย: B** - namespace declaration ต้องอยู่ก่อน code ใดๆ (ยกเว้น declare statement)

### คำถาม 2
ข้อใดคือการ import ถูกต้องแบบ grouped?
- A. `use MyApp\{Models\User, Services\UserService};`
- B. `use {MyApp\Models\User, MyApp\Services\UserService};`
- C. `use MyApp\Models\User, MyApp\Services\UserService;`
- D. ถูกทั้ง A และ C

**เฉลย: D** - ทั้ง A และ C ถูกต้อง แต่ A ใช้ group syntax ที่กระทัดรัดกว่า

### คำถาม 3
PSR-4 map namespace `App\Controllers\HomeController` กับ base dir `src/` จะหาไฟล์ที่ไหน?
- A. `src/App/Controllers/HomeController.php`
- B. `src/Controllers/HomeController.php`
- C. `src/HomeController.php`
- D. `Controllers/HomeController.php`

**เฉลย: B** - เมื่อ namespace prefix `App\` map กับ `src/` ส่วนที่เหลือคือ `Controllers/HomeController`

### คำถาม 4
ข้อแตกต่างระหว่าง `self::` และ `static::` ใน namespace คืออะไร?

**เฉลย:** ไม่เกี่ยวกับ namespace โดยตรง แต่ `static::class` ใน LSB จะให้ชื่อคลาสพร้อม fully-qualified namespace ของคลาสที่ถูกเรียกจริง

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- **Namespaces** - การจัดระเบียบ code และป้องกัน name collision
- **use statements** - การ import classes, functions, constants
- **PSR-4** - มาตรฐาน autoloading ที่ map namespace กับ directory
- **Composer Autoload** - วิธีมาตรฐานในการจัดการ dependencies และ autoloading
- **HTTP Client Library** - การออกแบบ library ด้วย namespace ที่ถูกต้อง

---

## ➡️ Part ถัดไป

[Part 018: PHP Composer ขั้นสูง](./part-018-php-composer.md)
