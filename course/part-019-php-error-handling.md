# Part 019: PHP Error Handling และ Logging

## ระดับ: Intermediate to Advanced
## เวลาเรียน: 3-4 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:
- เข้าใจ Error Types ต่างๆ ใน PHP
- ออกแบบ Exception Hierarchy ที่ดี
- สร้าง Custom Exceptions
- ใช้ try/catch/finally อย่างถูกต้อง
- ตั้งค่า Error Handlers
- ใช้ Monolog สำหรับ Logging
- สร้าง Error Handling System ที่สมบูรณ์

---

## 1. PHP Error Types

### ประเภทของ Error ใน PHP

```php
<?php
// Error Levels ใน PHP
$errorTypes = [
    E_ERROR         => 'Fatal Error',           // script หยุดทำงาน
    E_WARNING       => 'Warning',               // script ยังทำงานต่อ
    E_NOTICE        => 'Notice',                // แจ้งเตือนปัญหาเล็กน้อย
    E_PARSE         => 'Parse Error',           // syntax error
    E_CORE_ERROR    => 'Core Error',            // PHP core เกิดปัญหา
    E_DEPRECATED    => 'Deprecated',            // feature ที่จะถูกลบในอนาคต
    E_USER_ERROR    => 'User Error',            // trigger_error() สร้าง
    E_USER_WARNING  => 'User Warning',
    E_USER_NOTICE   => 'User Notice',
    E_STRICT        => 'Strict',                // แนะนำ best practices
];

// ตัวอย่าง Error แต่ละประเภท
// E_NOTICE
$undefinedVar = $notDefined; // Notice: Undefined variable

// E_WARNING
include 'not-exists.php'; // Warning: include(): Failed opening...

// E_DEPRECATED
$arr = [1, 2, 3];
$val = each($arr); // Deprecated: Function each() is deprecated

// E_ERROR (Fatal - ไม่สามารถ catch ด้วย try/catch ปกติ)
// $obj = new NonExistentClass(); // Fatal Error

// PHP 7+ แปลง Fatal Errors เป็น Throwable
try {
    $obj = new NonExistentClass();
} catch (\Error $e) {
    echo "Caught Error: " . $e->getMessage() . "\n";
}
```

### PHP 7+ Error Hierarchy

```
Throwable (interface)
├── Error
│   ├── ArithmeticError
│   │   └── DivisionByZeroError
│   ├── AssertionError
│   ├── ParseError
│   ├── TypeError
│   │   └── (ใน PHP 8+) Fibers
│   └── ValueError
└── Exception
    ├── BadFunctionCallException
    │   └── BadMethodCallException
    ├── DomainException
    ├── InvalidArgumentException
    ├── LengthException
    ├── LogicException
    ├── OutOfRangeException
    ├── OverflowException
    ├── RangeException
    ├── RuntimeException
    │   ├── OutOfBoundsException
    │   ├── OverflowException
    │   ├── UnderflowException
    │   └── UnexpectedValueException
    └── (Custom Exceptions...)
```

---

## 2. Exception Hierarchy ที่ดี

### ออกแบบ Exception สำหรับ Application

```php
<?php
// Base Exception สำหรับ Application ทั้งหมด
namespace App\Exceptions;

class AppException extends \RuntimeException {
    protected string $userMessage = 'เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง';
    protected array $context = [];
    
    public function __construct(
        string $message = '',
        int $code = 0,
        ?\Throwable $previous = null,
        array $context = []
    ) {
        parent::__construct($message, $code, $previous);
        $this->context = $context;
    }
    
    public function getUserMessage(): string {
        return $this->userMessage;
    }
    
    public function getContext(): array {
        return $this->context;
    }
    
    public function withContext(array $context): static {
        $clone = clone $this;
        $clone->context = array_merge($this->context, $context);
        return $clone;
    }
}

// Domain-specific exceptions
class DatabaseException extends AppException {
    protected string $userMessage = 'เกิดข้อผิดพลาดในการเชื่อมต่อฐานข้อมูล';
}

class ValidationException extends AppException {
    protected string $userMessage = 'ข้อมูลที่กรอกไม่ถูกต้อง';
    
    public function __construct(
        private array $errors,
        string $message = '',
        int $code = 422
    ) {
        parent::__construct($message ?: 'Validation failed', $code);
    }
    
    public function getErrors(): array {
        return $this->errors;
    }
    
    public function getFirstError(): ?string {
        return reset($this->errors) ?: null;
    }
}

class AuthException extends AppException {
    protected string $userMessage = 'กรุณาเข้าสู่ระบบก่อน';
}

class AuthorizationException extends AppException {
    protected string $userMessage = 'คุณไม่มีสิทธิ์เข้าถึงข้อมูลนี้';
    
    public function __construct(string $action = '', string $resource = '') {
        parent::__construct(
            "Unauthorized: cannot {$action} on {$resource}",
            403
        );
    }
}

class NotFoundException extends AppException {
    protected string $userMessage = 'ไม่พบข้อมูลที่ต้องการ';
    
    public function __construct(string $resource, int|string $id) {
        parent::__construct(
            "{$resource} with id '{$id}' not found",
            404
        );
        $this->userMessage = "ไม่พบ{$resource} รหัส {$id}";
    }
}

class RateLimitException extends AppException {
    protected string $userMessage = 'คุณส่งคำขอมากเกินไป กรุณารอสักครู่';
    
    public function __construct(
        private int $retryAfter = 60
    ) {
        parent::__construct("Rate limit exceeded", 429);
    }
    
    public function getRetryAfter(): int {
        return $this->retryAfter;
    }
}

// เฉพาะ module
class PaymentException extends AppException {
    protected string $userMessage = 'การชำระเงินล้มเหลว';
}

class InsufficientFundsException extends PaymentException {
    protected string $userMessage = 'ยอดเงินไม่เพียงพอ';
    
    public function __construct(float $required, float $available) {
        parent::__construct(
            "Insufficient funds: required {$required}, available {$available}",
            400
        );
    }
}
```

---

## 3. Custom Exceptions ขั้นสูง

```php
<?php
namespace App\Exceptions;

use Throwable;

// Exception พร้อม HTTP response context
class HttpException extends AppException {
    public function __construct(
        private int $statusCode,
        string $message = '',
        private array $headers = [],
        int $code = 0,
        ?Throwable $previous = null
    ) {
        parent::__construct($message, $code, $previous);
    }
    
    public function getStatusCode(): int {
        return $this->statusCode;
    }
    
    public function getHeaders(): array {
        return $this->headers;
    }
    
    // Factory methods
    public static function notFound(string $message = 'Not Found'): self {
        return new self(404, $message);
    }
    
    public static function forbidden(string $message = 'Forbidden'): self {
        return new self(403, $message);
    }
    
    public static function serverError(string $message = 'Internal Server Error'): self {
        return new self(500, $message);
    }
    
    public static function badRequest(string $message = 'Bad Request'): self {
        return new self(400, $message);
    }
}

// Exception พร้อม retry logic
class ExternalServiceException extends AppException {
    private bool $retryable = true;
    private int $maxRetries = 3;
    
    public function isRetryable(): bool {
        return $this->retryable;
    }
    
    public function getMaxRetries(): int {
        return $this->maxRetries;
    }
    
    public function withRetryable(bool $retryable): static {
        $clone = clone $this;
        $clone->retryable = $retryable;
        return $clone;
    }
    
    public function withMaxRetries(int $max): static {
        $clone = clone $this;
        $clone->maxRetries = $max;
        return $clone;
    }
}

// Exception chain สำหรับ debug
class WrappedException extends AppException {
    public function __construct(
        Throwable $original,
        string $context = ''
    ) {
        parent::__construct(
            $context ? "{$context}: {$original->getMessage()}" : $original->getMessage(),
            $original->getCode(),
            $original
        );
    }
    
    public function getOriginal(): ?Throwable {
        return $this->getPrevious();
    }
    
    public function getExceptionChain(): array {
        $chain = [];
        $e = $this;
        while ($e !== null) {
            $chain[] = [
                'class' => get_class($e),
                'message' => $e->getMessage(),
                'file' => $e->getFile(),
                'line' => $e->getLine(),
            ];
            $e = $e->getPrevious();
        }
        return $chain;
    }
}
```

---

## 4. try/catch/finally

### Pattern ต่างๆ

```php
<?php
use App\Exceptions\{NotFoundException, ValidationException, DatabaseException};

// Pattern 1: Basic
function basicPattern(): void {
    try {
        $result = riskyOperation();
        echo "Success: {$result}\n";
    } catch (NotFoundException $e) {
        echo "Not found: {$e->getMessage()}\n";
    } catch (DatabaseException $e) {
        echo "DB Error: {$e->getMessage()}\n";
    } catch (\Exception $e) {
        echo "Unknown error: {$e->getMessage()}\n";
    } finally {
        echo "This always runs (cleanup here)\n";
    }
}

// Pattern 2: Multiple Exception Types (PHP 8+)
function multipleExceptionPattern(): void {
    try {
        $result = anotherRiskyOperation();
    } catch (NotFoundException | ValidationException $e) {
        // จัดการ 2 exception type ด้วย code เดียวกัน
        http_response_code($e->getCode());
        echo json_encode(['error' => $e->getUserMessage()]);
    } catch (\Throwable $e) {
        // catch ทั้ง Exception และ Error
        http_response_code(500);
        echo json_encode(['error' => 'Server error']);
        logError($e);
    }
}

// Pattern 3: Re-throwing
function rethrowPattern(): void {
    try {
        connectToDatabase();
    } catch (\PDOException $e) {
        // Wrap exception พร้อมเพิ่ม context
        throw new DatabaseException(
            "Failed to connect: " . $e->getMessage(),
            500,
            $e // เก็บ original exception ไว้
        );
    }
}

// Pattern 4: Exception chaining
function exceptionChaining(): array {
    try {
        return loadUserData(1);
    } catch (\Exception $e) {
        throw new WrappedException($e, "Failed to load user #1");
    }
}

// Pattern 5: Finally สำหรับ cleanup
function finallyCleanup(): void {
    $resource = null;
    
    try {
        $resource = openFile('/path/to/file.txt');
        processFile($resource);
    } catch (\Exception $e) {
        echo "Error: " . $e->getMessage() . "\n";
        throw $e; // re-throw หลัง cleanup
    } finally {
        // cleanup จะรันเสมอ แม้จะมี return หรือ throw
        if ($resource !== null) {
            closeFile($resource);
            echo "File closed\n";
        }
    }
}

// Pattern 6: Try ซ้อน
function nestedTry(): void {
    try {
        try {
            $user = findUser(999);
        } catch (NotFoundException $e) {
            // inner catch: กลับค่า default
            $user = createGuestUser();
        }
        
        processUser($user);
        
    } catch (\Exception $e) {
        // outer catch: จัดการ error จาก processUser
        echo "Failed to process user: " . $e->getMessage();
    }
}
```

### Exception ใน Constructor และ Destructor

```php
<?php
class DatabaseConnection {
    private \PDO $pdo;
    private bool $inTransaction = false;
    
    public function __construct(
        private string $dsn,
        private string $username,
        private string $password
    ) {
        try {
            $this->pdo = new \PDO($dsn, $username, $password, [
                \PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION,
                \PDO::ATTR_DEFAULT_FETCH_MODE => \PDO::FETCH_ASSOC,
            ]);
        } catch (\PDOException $e) {
            throw new DatabaseException(
                "Connection failed: " . $e->getMessage(),
                500,
                $e
            );
        }
    }
    
    public function transaction(callable $callback): mixed {
        $this->pdo->beginTransaction();
        $this->inTransaction = true;
        
        try {
            $result = $callback($this->pdo);
            $this->pdo->commit();
            $this->inTransaction = false;
            return $result;
        } catch (\Throwable $e) {
            $this->pdo->rollback();
            $this->inTransaction = false;
            throw $e; // re-throw
        }
    }
    
    public function __destruct() {
        if ($this->inTransaction) {
            // Destructor ไม่ควร throw exceptions
            try {
                $this->pdo->rollback();
            } catch (\Throwable) {
                // swallow silently
            }
        }
    }
}
```

---

## 5. Error Handlers

### Custom Error Handler

```php
<?php
// Error Handler แปลง PHP errors เป็น Exceptions
set_error_handler(function(
    int $errno,
    string $errstr,
    string $errfile,
    int $errline
): bool {
    // ถ้า error ถูก suppress ด้วย @ operator
    if (error_reporting() === 0) {
        return false;
    }
    
    // แปลงเป็น Exception
    throw new \ErrorException($errstr, 0, $errno, $errfile, $errline);
});

// Exception Handler สำหรับ uncaught exceptions
set_exception_handler(function(\Throwable $e): void {
    $isApi = str_starts_with($_SERVER['REQUEST_URI'] ?? '', '/api/');
    
    if ($isApi) {
        header('Content-Type: application/json');
        http_response_code(500);
        echo json_encode([
            'error' => 'Server Error',
            'message' => 'An unexpected error occurred',
        ]);
    } else {
        http_response_code(500);
        echo '<h1>500 - Internal Server Error</h1>';
        
        if (($_ENV['APP_DEBUG'] ?? 'false') === 'true') {
            echo "<pre>{$e}</pre>";
        }
    }
    
    // Log the error
    error_log($e->getMessage() . " in " . $e->getFile() . ":" . $e->getLine());
});

// Shutdown Handler สำหรับ fatal errors
register_shutdown_function(function(): void {
    $error = error_get_last();
    
    if ($error !== null && in_array($error['type'], [E_ERROR, E_CORE_ERROR, E_PARSE])) {
        $message = "Fatal Error: {$error['message']} in {$error['file']}:{$error['line']}";
        error_log($message);
        
        if (!headers_sent()) {
            http_response_code(500);
            echo json_encode(['error' => 'Fatal Error']);
        }
    }
});

// ทดสอบ
try {
    $x = 10 / 0; // DivisionByZeroError
} catch (\DivisionByZeroError $e) {
    echo "Division error: " . $e->getMessage() . "\n";
}

// trigger_error จะถูกแปลงเป็น ErrorException
try {
    trigger_error("Custom warning", E_USER_WARNING);
} catch (\ErrorException $e) {
    echo "Caught trigger_error: " . $e->getMessage() . "\n";
}
```

### Central Error Handler Class

```php
<?php
namespace App\Error;

use Throwable;
use App\Exceptions\{AppException, HttpException, ValidationException};

class ErrorHandler {
    private array $reporters = [];
    private array $ignoredExceptions = [];
    
    public function addReporter(callable $reporter): void {
        $this->reporters[] = $reporter;
    }
    
    public function ignore(string ...$exceptionClasses): void {
        $this->ignoredExceptions = array_merge($this->ignoredExceptions, $exceptionClasses);
    }
    
    public function handle(Throwable $e): void {
        if (!$this->shouldReport($e)) {
            return;
        }
        
        foreach ($this->reporters as $reporter) {
            try {
                $reporter($e);
            } catch (Throwable) {
                // Reporter failure ไม่ควรทำให้ app crash
            }
        }
    }
    
    private function shouldReport(Throwable $e): bool {
        foreach ($this->ignoredExceptions as $class) {
            if ($e instanceof $class) {
                return false;
            }
        }
        return true;
    }
    
    public function render(Throwable $e): array {
        if ($e instanceof ValidationException) {
            return [
                'status' => 422,
                'body' => [
                    'message' => $e->getUserMessage(),
                    'errors' => $e->getErrors(),
                ],
            ];
        }
        
        if ($e instanceof HttpException) {
            return [
                'status' => $e->getStatusCode(),
                'body' => [
                    'message' => $e->getMessage(),
                ],
            ];
        }
        
        if ($e instanceof AppException) {
            return [
                'status' => $e->getCode() ?: 500,
                'body' => [
                    'message' => $e->getUserMessage(),
                ],
            ];
        }
        
        // Unknown exception
        return [
            'status' => 500,
            'body' => [
                'message' => 'An unexpected error occurred',
            ],
        ];
    }
}
```

---

## 6. Logging ด้วย Monolog

### ติดตั้ง Monolog

```bash
composer require monolog/monolog
```

### Monolog Levels

```
DEBUG     (100) - ข้อมูล debug
INFO      (200) - ข้อมูลทั่วไป
NOTICE    (250) - เหตุการณ์สำคัญแต่ปกติ
WARNING   (300) - เตือนปัญหาที่อาจเกิดขึ้น
ERROR     (400) - error ที่ต้องแก้ไข
CRITICAL  (500) - ปัญหาวิกฤต
ALERT     (550) - ต้องดำเนินการทันที
EMERGENCY (600) - ระบบล่ม
```

### ตั้งค่า Monolog

```php
<?php
use Monolog\Logger;
use Monolog\Handler\StreamHandler;
use Monolog\Handler\RotatingFileHandler;
use Monolog\Handler\SlackWebhookHandler;
use Monolog\Handler\FirePHPHandler;
use Monolog\Formatter\JsonFormatter;
use Monolog\Formatter\LineFormatter;
use Monolog\Processor\UidProcessor;
use Monolog\Processor\WebProcessor;
use Monolog\Processor\IntrospectionProcessor;
use Monolog\Level;

// สร้าง Logger
$logger = new Logger('app');

// Handler 1: เขียน log ไปไฟล์ (rotating - เก็บ 30 วัน)
$fileHandler = new RotatingFileHandler(
    filename: '/var/log/app/app.log',
    maxFiles: 30,
    level: Level::Debug
);

// Format log เป็น JSON
$fileHandler->setFormatter(new JsonFormatter());
$logger->pushHandler($fileHandler);

// Handler 2: เขียน error ขึ้น stderr
$stderrHandler = new StreamHandler('php://stderr', Level::Error);
$stderrHandler->setFormatter(new LineFormatter(
    format: "[%datetime%] %channel%.%level_name%: %message% %context% %extra%\n",
    dateFormat: 'Y-m-d H:i:s'
));
$logger->pushHandler($stderrHandler);

// Handler 3: ส่ง critical errors ไป Slack
if (isset($_ENV['SLACK_WEBHOOK_URL'])) {
    $slackHandler = new SlackWebhookHandler(
        webhookUrl: $_ENV['SLACK_WEBHOOK_URL'],
        channel: '#alerts',
        username: 'PHP Error Bot',
        level: Level::Critical
    );
    $logger->pushHandler($slackHandler);
}

// Processors เพิ่ม metadata
$logger->pushProcessor(new UidProcessor()); // unique ID ต่อ request
$logger->pushProcessor(new WebProcessor());  // IP, method, URL
$logger->pushProcessor(new IntrospectionProcessor()); // file, line, class

// Custom Processor
$logger->pushProcessor(function(array $record) {
    $record['extra']['user_id'] = $_SESSION['user_id'] ?? null;
    $record['extra']['request_id'] = $_SERVER['HTTP_X_REQUEST_ID'] ?? uniqid();
    return $record;
});

// ใช้งาน
$logger->info('User logged in', ['user_id' => 123, 'ip' => '127.0.0.1']);
$logger->warning('Slow query detected', ['query' => 'SELECT...', 'duration' => 2.5]);
$logger->error('Payment failed', ['order_id' => 456, 'amount' => 1000.00]);
$logger->critical('Database connection lost');
```

### Logger Factory Pattern

```php
<?php
namespace App\Log;

use Monolog\Logger;
use Monolog\Handler\RotatingFileHandler;
use Monolog\Handler\StreamHandler;
use Monolog\Formatter\JsonFormatter;
use Monolog\Level;

class LoggerFactory {
    private static array $loggers = [];
    
    public static function make(string $channel = 'app'): Logger {
        if (!isset(self::$loggers[$channel])) {
            self::$loggers[$channel] = self::create($channel);
        }
        return self::$loggers[$channel];
    }
    
    private static function create(string $channel): Logger {
        $logger = new Logger($channel);
        $logDir = dirname(__DIR__, 2) . '/storage/logs';
        
        // สร้าง directory ถ้ายังไม่มี
        if (!is_dir($logDir)) {
            mkdir($logDir, 0755, true);
        }
        
        // Rotating file handler
        $handler = new RotatingFileHandler(
            "{$logDir}/{$channel}.log",
            maxFiles: 30,
            level: Level::Debug
        );
        $handler->setFormatter(new JsonFormatter());
        $logger->pushHandler($handler);
        
        // Development: เขียนไป stdout ด้วย
        if (($_ENV['APP_ENV'] ?? 'production') === 'development') {
            $logger->pushHandler(new StreamHandler('php://stdout', Level::Debug));
        }
        
        return $logger;
    }
}

// Usage
$log = LoggerFactory::make('payments');
$log->info('Payment processed', ['amount' => 500, 'currency' => 'THB']);

$log = LoggerFactory::make('auth');
$log->warning('Failed login attempt', ['username' => 'hacker@example.com', 'ip' => '1.2.3.4']);
```

---

## 7. Workshop: สร้าง Error Handling System

### โจทย์: Error Handler สำหรับ REST API

```php
<?php
namespace App;

use App\Exceptions\{AppException, ValidationException, HttpException, AuthException};
use App\Log\LoggerFactory;
use Throwable;

// ============================================================
// Result Type Pattern (แทน throw/catch ในบางกรณี)
// ============================================================

class Result {
    private function __construct(
        private readonly bool $success,
        private readonly mixed $value,
        private readonly ?string $error
    ) {}
    
    public static function ok(mixed $value): self {
        return new self(true, $value, null);
    }
    
    public static function fail(string $error): self {
        return new self(false, null, $error);
    }
    
    public function isOk(): bool { return $this->success; }
    public function isFail(): bool { return !$this->success; }
    public function getValue(): mixed { return $this->value; }
    public function getError(): ?string { return $this->error; }
    
    public function map(callable $fn): self {
        if ($this->success) {
            return self::ok($fn($this->value));
        }
        return $this;
    }
    
    public function flatMap(callable $fn): self {
        if ($this->success) {
            return $fn($this->value);
        }
        return $this;
    }
    
    public function getOrElse(mixed $default): mixed {
        return $this->success ? $this->value : $default;
    }
}

// ============================================================
// API Response Handler
// ============================================================

class ApiErrorHandler {
    private array $handlers = [];
    private $logger;
    private bool $debug;
    
    public function __construct(bool $debug = false) {
        $this->debug = $debug;
        $this->logger = LoggerFactory::make('api');
        $this->registerDefaultHandlers();
    }
    
    private function registerDefaultHandlers(): void {
        // ValidationException
        $this->register(ValidationException::class, function(ValidationException $e): array {
            return [
                'status' => 422,
                'body' => [
                    'success' => false,
                    'message' => $e->getUserMessage(),
                    'errors' => $e->getErrors(),
                ],
            ];
        });
        
        // HttpException
        $this->register(HttpException::class, function(HttpException $e): array {
            return [
                'status' => $e->getStatusCode(),
                'headers' => $e->getHeaders(),
                'body' => [
                    'success' => false,
                    'message' => $e->getMessage(),
                ],
            ];
        });
        
        // AuthException
        $this->register(AuthException::class, function(AuthException $e): array {
            return [
                'status' => 401,
                'headers' => ['WWW-Authenticate' => 'Bearer'],
                'body' => [
                    'success' => false,
                    'message' => $e->getUserMessage(),
                ],
            ];
        });
        
        // AppException (generic)
        $this->register(AppException::class, function(AppException $e): array {
            $this->logger->error($e->getMessage(), [
                'exception' => get_class($e),
                'context' => $e->getContext(),
                'trace' => $e->getTraceAsString(),
            ]);
            
            return [
                'status' => $e->getCode() ?: 500,
                'body' => [
                    'success' => false,
                    'message' => $e->getUserMessage(),
                ],
            ];
        });
    }
    
    public function register(string $exceptionClass, callable $handler): void {
        $this->handlers[$exceptionClass] = $handler;
    }
    
    public function handle(Throwable $e): array {
        // หา handler ที่ match กับ exception type
        foreach ($this->handlers as $class => $handler) {
            if ($e instanceof $class) {
                $response = $handler($e);
                
                if ($this->debug) {
                    $response['body']['debug'] = [
                        'exception' => get_class($e),
                        'message' => $e->getMessage(),
                        'file' => $e->getFile(),
                        'line' => $e->getLine(),
                        'trace' => explode("\n", $e->getTraceAsString()),
                    ];
                }
                
                return $response;
            }
        }
        
        // Unknown exception
        $this->logger->critical('Unhandled exception', [
            'exception' => get_class($e),
            'message' => $e->getMessage(),
            'trace' => $e->getTraceAsString(),
        ]);
        
        $response = [
            'status' => 500,
            'body' => [
                'success' => false,
                'message' => 'เกิดข้อผิดพลาดที่ไม่คาดคิด',
            ],
        ];
        
        if ($this->debug) {
            $response['body']['debug'] = [
                'exception' => get_class($e),
                'message' => $e->getMessage(),
                'trace' => explode("\n", $e->getTraceAsString()),
            ];
        }
        
        return $response;
    }
    
    public function sendResponse(array $response): void {
        http_response_code($response['status']);
        
        header('Content-Type: application/json');
        
        foreach ($response['headers'] ?? [] as $name => $value) {
            header("{$name}: {$value}");
        }
        
        echo json_encode($response['body'], JSON_UNESCAPED_UNICODE);
    }
}

// ============================================================
// Middleware-style Error Handling
// ============================================================

class ApiKernel {
    private ApiErrorHandler $errorHandler;
    
    public function __construct() {
        $debug = ($_ENV['APP_DEBUG'] ?? 'false') === 'true';
        $this->errorHandler = new ApiErrorHandler($debug);
        
        // ตั้งค่า global error handlers
        $this->setupErrorHandlers();
    }
    
    private function setupErrorHandlers(): void {
        set_error_handler(function(int $errno, string $errstr, string $errfile, int $errline): bool {
            if (error_reporting() === 0) return false;
            throw new \ErrorException($errstr, 0, $errno, $errfile, $errline);
        });
        
        set_exception_handler(function(Throwable $e): void {
            $response = $this->errorHandler->handle($e);
            $this->errorHandler->sendResponse($response);
        });
    }
    
    public function run(callable $action): void {
        try {
            $result = $action();
            
            header('Content-Type: application/json');
            echo json_encode([
                'success' => true,
                'data' => $result,
            ], JSON_UNESCAPED_UNICODE);
            
        } catch (Throwable $e) {
            $response = $this->errorHandler->handle($e);
            $this->errorHandler->sendResponse($response);
        }
    }
}

// ============================================================
// ทดสอบการใช้งาน
// ============================================================

function demonstrateErrorHandling(): void {
    $handler = new ApiErrorHandler(debug: true);
    
    // ทดสอบ 1: ValidationException
    $validationEx = new ValidationException([
        'email' => 'รูปแบบ email ไม่ถูกต้อง',
        'password' => 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร',
    ]);
    
    echo "=== ValidationException ===\n";
    $response = $handler->handle($validationEx);
    echo json_encode($response, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT) . "\n\n";
    
    // ทดสอบ 2: NotFoundException
    $notFoundEx = new NotFoundException('User', 999);
    
    echo "=== NotFoundException ===\n";
    $response = $handler->handle($notFoundEx);
    echo json_encode($response, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT) . "\n\n";
    
    // ทดสอบ 3: Result Pattern
    echo "=== Result Pattern ===\n";
    
    $result = Result::ok(42)
        ->map(fn($v) => $v * 2)
        ->map(fn($v) => "Value is {$v}");
    
    echo $result->getValue() . "\n"; // Value is 84
    
    $failed = Result::fail("Something went wrong")
        ->map(fn($v) => $v * 2); // ไม่ทำงาน เพราะ fail
    
    echo $failed->getOrElse("default") . "\n"; // default
}

demonstrateErrorHandling();
```

---

## Quiz

### คำถาม 1
`finally` block จะทำงานเมื่อใด?
- A. เมื่อ try block สำเร็จ
- B. เมื่อ catch block ทำงาน
- C. เสมอ ไม่ว่าจะเกิดอะไร
- D. เมื่อไม่มี exception

**เฉลย: C** - `finally` รันเสมอ แม้จะมี return หรือ throw ใน try/catch

### คำถาม 2
ข้อใดคือความแตกต่างระหว่าง `Exception` และ `Error` ใน PHP 7+?
- A. ไม่มีความแตกต่าง
- B. `Error` เกิดจาก PHP internal errors, `Exception` เกิดจาก userland code
- C. `Error` ไม่สามารถ catch ได้
- D. `Exception` serious กว่า `Error`

**เฉลย: B** - ทั้งคู่ implement `Throwable` แต่มีต้นกำเนิดต่างกัน

### คำถาม 3
Monolog Level ใดที่ serious ที่สุด?
- A. CRITICAL
- B. FATAL  
- C. EMERGENCY
- D. ALERT

**เฉลย: C** - `EMERGENCY` (600) คือ level สูงสุด

### คำถาม 4
`set_error_handler` ใช้ทำอะไร?

**เฉลย:** ใช้กำหนด function ที่จะถูกเรียกเมื่อ PHP triggers error (E_WARNING, E_NOTICE, etc.) แทนที่ default behavior ซึ่งช่วยให้เราแปลง PHP errors เป็น Exceptions ได้

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- **Error Types** - E_ERROR, E_WARNING, E_NOTICE และ PHP 7+ Throwable hierarchy
- **Exception Hierarchy** - การออกแบบ exception classes ที่มี structure ดี
- **Custom Exceptions** - สร้าง exceptions พร้อม context, user messages, HTTP status
- **try/catch/finally** - patterns ต่างๆ การใช้งานที่ถูกต้อง
- **Error Handlers** - set_error_handler, set_exception_handler, register_shutdown_function
- **Monolog** - logging ระดับ production พร้อม handlers และ processors
- **API Error Handler** - ระบบจัดการ error สำหรับ REST API

---

## ➡️ Part ถัดไป

[Part 020: PHP Regular Expressions](./part-020-php-regex.md)
