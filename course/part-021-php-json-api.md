# Part 021: PHP JSON และ REST API

## ระดับ: Intermediate to Advanced
## เวลาเรียน: 4-5 ชั่วโมง

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ ผู้เรียนจะสามารถ:
- ใช้งาน json_encode/json_decode อย่างละเอียด
- สร้าง REST API ด้วย PHP Vanilla
- ส่ง HTTP requests ด้วย cURL
- ใช้ Guzzle HTTP Client
- สร้าง API Client สำหรับ third-party API

---

## 1. JSON encode/decode

### json_encode() Options

```php
<?php
$data = [
    'name' => 'สมชาย ใจดี',
    'email' => 'somchai@example.com',
    'age' => 30,
    'active' => true,
    'score' => 9.5,
    'tags' => ['php', 'laravel'],
    'address' => null,
    'profile' => [
        'bio' => 'PHP Developer <script>',
        'url' => 'https://example.com/path?q=1&r=2',
    ],
];

// ค่าเริ่มต้น
echo json_encode($data) . "\n";

// JSON_PRETTY_PRINT - อ่านง่าย
echo json_encode($data, JSON_PRETTY_PRINT) . "\n";

// JSON_UNESCAPED_UNICODE - ไม่ encode Thai/Unicode
echo json_encode($data, JSON_UNESCAPED_UNICODE) . "\n";

// JSON_UNESCAPED_SLASHES - ไม่ escape slashes
echo json_encode($data, JSON_UNESCAPED_SLASHES) . "\n";

// JSON_HEX_TAG - escape < และ >
echo json_encode($data, JSON_HEX_TAG) . "\n";

// Combine flags ด้วย |
$json = json_encode(
    $data,
    JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES
);
echo $json . "\n";

// JSON_THROW_ON_ERROR - throw exception แทน return false
try {
    $invalid = fopen('/dev/null', 'r'); // resource ไม่สามารถ encode ได้
    $result = json_encode(['resource' => $invalid], JSON_THROW_ON_ERROR);
} catch (\JsonException $e) {
    echo "JSON Error: " . $e->getMessage() . "\n";
}

// ตรวจสอบ error (วิธีเก่า ก่อน PHP 7.3)
$result = json_encode(["\xB1\x31"]); // invalid UTF-8
if (json_last_error() !== JSON_ERROR_NONE) {
    echo "Error: " . json_last_error_msg() . "\n";
}
```

### json_decode() Options

```php
<?php
$json = '{"name":"สมชาย","tags":["php","laravel"],"profile":{"level":5}}';

// decode เป็น object (default)
$obj = json_decode($json);
echo $obj->name . "\n";           // สมชาย
echo $obj->profile->level . "\n"; // 5
echo $obj->tags[0] . "\n";        // php

// decode เป็น array (assoc: true)
$arr = json_decode($json, true);
echo $arr['name'] . "\n";              // สมชาย
echo $arr['profile']['level'] . "\n";  // 5

// depth (default: 512, ลดได้เพื่อป้องกัน nested JSON attack)
$nested = json_decode($json, true, depth: 5);

// JSON_THROW_ON_ERROR
try {
    $data = json_decode('invalid json', true, 512, JSON_THROW_ON_ERROR);
} catch (\JsonException $e) {
    echo "Parse Error: " . $e->getMessage() . "\n"; // Syntax error
}

// JSON_BIGINT_AS_STRING - จัดการ big integers
$bigJson = '{"id":9999999999999999999}';
$data = json_decode($bigJson, true);
echo $data['id'] . "\n"; // อาจเป็น float ที่เสีย precision

$data = json_decode($bigJson, true, 512, JSON_BIGINT_AS_STRING);
echo $data['id'] . "\n"; // "9999999999999999999" (string)
```

### Custom JSON Serialization

```php
<?php
// Implement JsonSerializable
class User implements \JsonSerializable {
    private string $passwordHash;
    
    public function __construct(
        private int $id,
        private string $name,
        private string $email,
        string $password
    ) {
        $this->passwordHash = password_hash($password, PASSWORD_ARGON2ID);
    }
    
    // กำหนดว่าจะ serialize อะไร
    public function jsonSerialize(): mixed {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            // ไม่รวม passwordHash!
            'created_at' => date('Y-m-d H:i:s'),
        ];
    }
    
    public static function fromJson(string $json): self {
        $data = json_decode($json, true, flags: JSON_THROW_ON_ERROR);
        return new self(
            $data['id'],
            $data['name'],
            $data['email'],
            'placeholder' // จะไม่มี password ใน JSON
        );
    }
}

$user = new User(1, "สมชาย", "somchai@example.com", "secret123");
$json = json_encode($user, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT);
echo $json . "\n"; // ไม่มี passwordHash
```

---

## 2. REST API ด้วย PHP Vanilla

### โครงสร้าง REST API

```php
<?php
// index.php หรือ api.php
// .htaccess: RewriteRule ^(.*)$ index.php [QSA,L]

declare(strict_types=1);

// Bootstrap
define('APP_ROOT', dirname(__FILE__));
require_once APP_ROOT . '/vendor/autoload.php';

// Set Headers
header('Content-Type: application/json; charset=utf-8');
header('Access-Control-Allow-Origin: *');
header('Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS');
header('Access-Control-Allow-Headers: Content-Type, Authorization, X-Requested-With');

// Handle preflight
if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
    http_response_code(204);
    exit;
}

// Router
$method = $_SERVER['REQUEST_METHOD'];
$path = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
$path = trim($path, '/');
$segments = explode('/', $path);

// Remove API prefix if needed (e.g., /api/v1/users -> users)
if ($segments[0] === 'api') array_shift($segments);
if (isset($segments[0]) && preg_match('/^v\d+$/', $segments[0])) array_shift($segments);

$resource = $segments[0] ?? '';
$id = $segments[1] ?? null;
$subResource = $segments[2] ?? null;

// Parse body
$body = file_get_contents('php://input');
$input = !empty($body) ? json_decode($body, true) : [];
$query = $_GET;

// Route
try {
    $response = route($method, $resource, $id, $subResource, $input, $query);
    http_response_code($response['status'] ?? 200);
    echo json_encode($response['data'] ?? [], JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT);
} catch (\InvalidArgumentException $e) {
    http_response_code(400);
    echo json_encode(['error' => $e->getMessage()]);
} catch (\RuntimeException $e) {
    http_response_code($e->getCode() ?: 500);
    echo json_encode(['error' => $e->getMessage()]);
} catch (\Throwable $e) {
    http_response_code(500);
    echo json_encode(['error' => 'Internal Server Error']);
    error_log($e->getMessage() . "\n" . $e->getTraceAsString());
}

function route(
    string $method,
    string $resource,
    ?string $id,
    ?string $subResource,
    array $input,
    array $query
): array {
    // User endpoints
    if ($resource === 'users') {
        $controller = new UserController();
        
        return match(true) {
            $method === 'GET' && $id === null        => $controller->index($query),
            $method === 'GET' && $id !== null        => $controller->show((int) $id),
            $method === 'POST' && $id === null       => $controller->store($input),
            $method === 'PUT' && $id !== null        => $controller->update((int) $id, $input),
            $method === 'PATCH' && $id !== null      => $controller->patch((int) $id, $input),
            $method === 'DELETE' && $id !== null     => $controller->destroy((int) $id),
            $method === 'GET' && $subResource === 'posts' => $controller->getPosts((int) $id),
            default => throw new \RuntimeException("Route not found", 404),
        };
    }
    
    throw new \RuntimeException("Resource '{$resource}' not found", 404);
}
```

### Controller และ Response Helpers

```php
<?php
// ApiResponse.php
class ApiResponse {
    public static function success(mixed $data, string $message = '', int $status = 200): array {
        return [
            'status' => $status,
            'data' => [
                'success' => true,
                'message' => $message,
                'data' => $data,
            ],
        ];
    }
    
    public static function created(mixed $data, string $message = 'Resource created'): array {
        return self::success($data, $message, 201);
    }
    
    public static function paginated(array $items, int $total, int $page, int $perPage): array {
        return [
            'status' => 200,
            'data' => [
                'success' => true,
                'data' => $items,
                'meta' => [
                    'total' => $total,
                    'page' => $page,
                    'per_page' => $perPage,
                    'last_page' => (int) ceil($total / $perPage),
                    'has_more' => ($page * $perPage) < $total,
                ],
            ],
        ];
    }
    
    public static function error(string $message, int $status = 400, array $errors = []): array {
        $data = [
            'success' => false,
            'message' => $message,
        ];
        
        if (!empty($errors)) {
            $data['errors'] = $errors;
        }
        
        return ['status' => $status, 'data' => $data];
    }
    
    public static function noContent(): array {
        return ['status' => 204, 'data' => null];
    }
}

// UserController.php
class UserController {
    private UserRepository $users;
    
    public function __construct() {
        $this->users = new UserRepository();
    }
    
    public function index(array $query): array {
        $page = max(1, (int) ($query['page'] ?? 1));
        $perPage = min(100, max(1, (int) ($query['per_page'] ?? 15)));
        $search = $query['search'] ?? '';
        
        [$items, $total] = $this->users->paginate($page, $perPage, $search);
        return ApiResponse::paginated($items, $total, $page, $perPage);
    }
    
    public function show(int $id): array {
        $user = $this->users->findOrFail($id);
        return ApiResponse::success($user);
    }
    
    public function store(array $input): array {
        $this->validate($input, [
            'name' => 'required|min:2',
            'email' => 'required|email',
        ]);
        
        $user = $this->users->create($input);
        return ApiResponse::created($user, 'User created successfully');
    }
    
    public function update(int $id, array $input): array {
        $this->users->findOrFail($id); // ตรวจสอบว่ามีอยู่
        
        $this->validate($input, [
            'name' => 'required|min:2',
            'email' => 'required|email',
        ]);
        
        $user = $this->users->update($id, $input);
        return ApiResponse::success($user, 'User updated');
    }
    
    public function patch(int $id, array $input): array {
        $this->users->findOrFail($id);
        $user = $this->users->update($id, $input);
        return ApiResponse::success($user, 'User patched');
    }
    
    public function destroy(int $id): array {
        $this->users->findOrFail($id);
        $this->users->delete($id);
        return ApiResponse::noContent();
    }
    
    public function getPosts(int $id): array {
        $this->users->findOrFail($id);
        $posts = (new PostRepository())->findByUserId($id);
        return ApiResponse::success($posts);
    }
    
    private function validate(array $data, array $rules): void {
        $errors = [];
        
        foreach ($rules as $field => $ruleString) {
            $fieldRules = explode('|', $ruleString);
            
            foreach ($fieldRules as $rule) {
                [$ruleName, $ruleValue] = array_pad(explode(':', $rule, 2), 2, null);
                
                match($ruleName) {
                    'required' => empty($data[$field]) && ($errors[$field][] = "{$field} is required"),
                    'email' => !empty($data[$field]) && !filter_var($data[$field], FILTER_VALIDATE_EMAIL) && ($errors[$field][] = "Invalid email"),
                    'min' => !empty($data[$field]) && strlen($data[$field]) < $ruleValue && ($errors[$field][] = "{$field} min length is {$ruleValue}"),
                    default => null,
                };
            }
        }
        
        if (!empty($errors)) {
            throw new \InvalidArgumentException(json_encode(['errors' => $errors]));
        }
    }
}
```

### Fake Repository สำหรับ Demo

```php
<?php
class UserRepository {
    private static array $users = [
        ['id' => 1, 'name' => 'สมชาย', 'email' => 'somchai@example.com'],
        ['id' => 2, 'name' => 'สมหญิง', 'email' => 'somying@example.com'],
        ['id' => 3, 'name' => 'วิชัย', 'email' => 'wichai@example.com'],
    ];
    
    private static int $nextId = 4;
    
    public function paginate(int $page, int $perPage, string $search = ''): array {
        $filtered = $search
            ? array_filter(self::$users, fn($u) => stripos($u['name'], $search) !== false)
            : self::$users;
        
        $total = count($filtered);
        $items = array_slice(array_values($filtered), ($page - 1) * $perPage, $perPage);
        
        return [$items, $total];
    }
    
    public function findOrFail(int $id): array {
        $user = array_filter(self::$users, fn($u) => $u['id'] === $id);
        if (empty($user)) {
            throw new \RuntimeException("User #{$id} not found", 404);
        }
        return reset($user);
    }
    
    public function create(array $data): array {
        $user = array_merge(['id' => self::$nextId++], $data);
        self::$users[] = $user;
        return $user;
    }
    
    public function update(int $id, array $data): array {
        foreach (self::$users as &$user) {
            if ($user['id'] === $id) {
                $user = array_merge($user, $data);
                return $user;
            }
        }
        throw new \RuntimeException("User #{$id} not found", 404);
    }
    
    public function delete(int $id): bool {
        foreach (self::$users as $key => $user) {
            if ($user['id'] === $id) {
                unset(self::$users[$key]);
                return true;
            }
        }
        return false;
    }
}
```

---

## 3. cURL Requests

### cURL พื้นฐาน

```php
<?php
class CurlClient {
    private array $defaultOptions = [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_FOLLOWLOCATION => true,
        CURLOPT_MAXREDIRS      => 5,
        CURLOPT_TIMEOUT        => 30,
        CURLOPT_CONNECTTIMEOUT => 10,
        CURLOPT_SSL_VERIFYPEER => true,
        CURLOPT_SSL_VERIFYHOST => 2,
        CURLOPT_USERAGENT      => 'PHP-CurlClient/1.0',
    ];
    
    private array $headers = [];
    
    public function withHeader(string $name, string $value): static {
        $this->headers[$name] = $value;
        return $this;
    }
    
    public function withBearerToken(string $token): static {
        return $this->withHeader('Authorization', "Bearer {$token}");
    }
    
    public function get(string $url, array $params = []): array {
        if (!empty($params)) {
            $url .= '?' . http_build_query($params);
        }
        
        return $this->request('GET', $url);
    }
    
    public function post(string $url, array|string $data = []): array {
        return $this->request('POST', $url, $data);
    }
    
    public function put(string $url, array|string $data = []): array {
        return $this->request('PUT', $url, $data);
    }
    
    public function delete(string $url): array {
        return $this->request('DELETE', $url);
    }
    
    public function postJson(string $url, array $data): array {
        $this->withHeader('Content-Type', 'application/json');
        return $this->request('POST', $url, json_encode($data));
    }
    
    private function request(string $method, string $url, mixed $data = null): array {
        $ch = curl_init($url);
        
        $options = $this->defaultOptions;
        $options[CURLOPT_CUSTOMREQUEST] = $method;
        
        // Headers
        if (!empty($this->headers)) {
            $headerLines = array_map(
                fn($k, $v) => "{$k}: {$v}",
                array_keys($this->headers),
                array_values($this->headers)
            );
            $options[CURLOPT_HTTPHEADER] = $headerLines;
        }
        
        // Body
        if ($data !== null) {
            if (is_array($data)) {
                $options[CURLOPT_POSTFIELDS] = http_build_query($data);
            } else {
                $options[CURLOPT_POSTFIELDS] = $data;
            }
        }
        
        // เก็บ response headers
        $responseHeaders = [];
        $options[CURLOPT_HEADERFUNCTION] = function($ch, $header) use (&$responseHeaders) {
            $len = strlen($header);
            $header = explode(':', $header, 2);
            if (count($header) < 2) return $len;
            
            $name = strtolower(trim($header[0]));
            $responseHeaders[$name] = trim($header[1]);
            return $len;
        };
        
        curl_setopt_array($ch, $options);
        
        $body = curl_exec($ch);
        $statusCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        $errno = curl_errno($ch);
        
        curl_close($ch);
        
        if ($errno !== 0) {
            throw new \RuntimeException("cURL Error: {$error}", $errno);
        }
        
        return [
            'status' => $statusCode,
            'headers' => $responseHeaders,
            'body' => $body,
            'json' => fn() => json_decode($body, true),
        ];
    }
    
    // Async requests ด้วย curl_multi
    public function multiRequest(array $requests): array {
        $mh = curl_multi_init();
        $handles = [];
        
        foreach ($requests as $key => $request) {
            [$method, $url, $data] = array_pad($request, 3, null);
            $ch = curl_init($url);
            
            $options = $this->defaultOptions;
            $options[CURLOPT_CUSTOMREQUEST] = $method;
            if ($data) {
                $options[CURLOPT_POSTFIELDS] = is_array($data) 
                    ? json_encode($data) 
                    : $data;
            }
            
            curl_setopt_array($ch, $options);
            curl_multi_add_handle($mh, $ch);
            $handles[$key] = $ch;
        }
        
        // Execute all
        $running = null;
        do {
            curl_multi_exec($mh, $running);
            curl_multi_select($mh);
        } while ($running > 0);
        
        // Collect results
        $results = [];
        foreach ($handles as $key => $ch) {
            $results[$key] = [
                'status' => curl_getinfo($ch, CURLINFO_HTTP_CODE),
                'body' => curl_multi_getcontent($ch),
            ];
            curl_multi_remove_handle($mh, $ch);
            curl_close($ch);
        }
        
        curl_multi_close($mh);
        return $results;
    }
}

// ทดสอบ
$client = new CurlClient();
$client->withBearerToken('my-token');

// GET request
$response = $client->get('https://jsonplaceholder.typicode.com/users', ['_limit' => 3]);
$users = $response['json']();
foreach ($users as $user) {
    echo "- {$user['name']} ({$user['email']})\n";
}

// POST request
$newPost = $client->postJson('https://jsonplaceholder.typicode.com/posts', [
    'title' => 'PHP is awesome',
    'body' => 'Learning PHP OOP',
    'userId' => 1,
]);
echo "Created post ID: " . $newPost['json']()['id'] . "\n";
```

---

## 4. Guzzle HTTP Client

```bash
composer require guzzlehttp/guzzle
```

### Guzzle พื้นฐาน

```php
<?php
use GuzzleHttp\Client;
use GuzzleHttp\Exception\RequestException;
use GuzzleHttp\Exception\ConnectException;
use GuzzleHttp\Middleware;
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Psr7\Request;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\RequestInterface;

// สร้าง Client
$client = new Client([
    'base_uri' => 'https://api.example.com',
    'timeout' => 30,
    'connect_timeout' => 10,
    'headers' => [
        'Accept' => 'application/json',
        'Content-Type' => 'application/json',
    ],
]);

// GET request
$response = $client->get('/users', [
    'query' => ['page' => 1, 'limit' => 10],
    'headers' => ['Authorization' => 'Bearer token123'],
]);

$data = json_decode($response->getBody()->getContents(), true);
echo $response->getStatusCode() . "\n"; // 200

// POST request
$response = $client->post('/users', [
    'json' => [
        'name' => 'สมชาย',
        'email' => 'somchai@example.com',
    ],
]);

// ดูตัวอย่างทั้งหมดของ options
$response = $client->request('GET', '/endpoint', [
    'query'   => ['key' => 'value'],
    'json'    => ['data' => 'value'],
    'form_params' => ['field' => 'value'],
    'headers' => ['X-Custom' => 'value'],
    'auth'    => ['username', 'password'],
    'timeout' => 5,
    'verify'  => false, // disable SSL (ไม่แนะนำ)
    'proxy'   => 'http://proxy:8080',
]);
```

### Middleware Stack

```php
<?php
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Middleware;
use GuzzleHttp\Client;

// Logging Middleware
$logMiddleware = Middleware::log(
    new \Monolog\Logger('guzzle'),
    new \GuzzleHttp\MessageFormatter('{method} {uri} - {code}')
);

// Retry Middleware
$retryMiddleware = Middleware::retry(
    function(
        int $retries,
        RequestInterface $request,
        ?ResponseInterface $response = null,
        ?\Throwable $exception = null
    ) {
        // Retry สูงสุด 3 ครั้ง
        if ($retries >= 3) return false;
        
        // Retry เมื่อ server error หรือ network error
        if ($exception instanceof ConnectException) return true;
        if ($response && $response->getStatusCode() >= 500) return true;
        
        return false;
    },
    function(int $retries) {
        return 1000 * $retries; // delay เพิ่มขึ้น: 1s, 2s, 3s
    }
);

// Auth Middleware
$authMiddleware = function(callable $handler) {
    return function(RequestInterface $request, array $options) use ($handler) {
        $request = $request->withHeader('Authorization', 'Bearer ' . getToken());
        return $handler($request, $options);
    };
};

// Build stack
$stack = HandlerStack::create();
$stack->push($logMiddleware, 'logging');
$stack->push($retryMiddleware, 'retry');
$stack->push($authMiddleware, 'auth');

$client = new Client([
    'handler' => $stack,
    'base_uri' => 'https://api.example.com',
]);
```

### Async Requests

```php
<?php
use GuzzleHttp\Client;
use GuzzleHttp\Pool;
use GuzzleHttp\Psr7\Request;
use GuzzleHttp\Promise\Utils;

$client = new Client(['base_uri' => 'https://jsonplaceholder.typicode.com']);

// Async ด้วย Promises
$promises = [
    'users' => $client->getAsync('/users'),
    'posts' => $client->getAsync('/posts?_limit=5'),
    'todos' => $client->getAsync('/todos?_limit=5'),
];

$responses = Utils::settle($promises)->wait();

foreach ($responses as $key => $result) {
    if ($result['state'] === 'fulfilled') {
        $data = json_decode($result['value']->getBody(), true);
        echo "{$key}: " . count($data) . " items\n";
    } else {
        echo "{$key}: Failed - " . $result['reason']->getMessage() . "\n";
    }
}

// Pool สำหรับ concurrent requests จำนวนมาก
$requests = function() use ($client) {
    for ($i = 1; $i <= 20; $i++) {
        yield new Request('GET', "/posts/{$i}");
    }
};

$pool = new Pool($client, $requests(), [
    'concurrency' => 5, // ส่งพร้อมกัน 5 requests
    'fulfilled' => function(ResponseInterface $response, int $index) {
        $post = json_decode($response->getBody(), true);
        echo "Got post {$index}: {$post['title']}\n";
    },
    'rejected' => function(\Throwable $reason, int $index) {
        echo "Failed post {$index}: " . $reason->getMessage() . "\n";
    },
]);

$pool->promise()->wait();
```

---

## 5. Workshop: สร้าง API Client สำหรับ Third-party API

### โจทย์: GitHub API Client

```php
<?php
namespace App\GitHub;

use GuzzleHttp\Client;
use GuzzleHttp\Exception\RequestException;

class GitHubClient {
    private Client $http;
    private string $baseUrl = 'https://api.github.com';
    
    public function __construct(
        private string $token,
        private string $apiVersion = '2022-11-28'
    ) {
        $this->http = new Client([
            'base_uri' => $this->baseUrl,
            'headers' => [
                'Authorization' => "Bearer {$token}",
                'Accept' => 'application/vnd.github+json',
                'X-GitHub-Api-Version' => $apiVersion,
                'User-Agent' => 'PHP-GitHub-Client/1.0',
            ],
            'timeout' => 30,
        ]);
    }
    
    // Users
    public function getUser(string $username): array {
        return $this->get("/users/{$username}");
    }
    
    public function getAuthenticatedUser(): array {
        return $this->get('/user');
    }
    
    // Repositories
    public function getRepo(string $owner, string $repo): array {
        return $this->get("/repos/{$owner}/{$repo}");
    }
    
    public function listUserRepos(string $username, array $params = []): array {
        return $this->get("/users/{$username}/repos", $params);
    }
    
    public function createRepo(array $data): array {
        return $this->post('/user/repos', $data);
    }
    
    // Issues
    public function listIssues(string $owner, string $repo, array $params = []): array {
        return $this->get("/repos/{$owner}/{$repo}/issues", $params);
    }
    
    public function createIssue(string $owner, string $repo, array $data): array {
        return $this->post("/repos/{$owner}/{$repo}/issues", $data);
    }
    
    public function closeIssue(string $owner, string $repo, int $issueNumber): array {
        return $this->patch("/repos/{$owner}/{$repo}/issues/{$issueNumber}", [
            'state' => 'closed',
        ]);
    }
    
    // Pull Requests
    public function listPullRequests(string $owner, string $repo, array $params = []): array {
        return $this->get("/repos/{$owner}/{$repo}/pulls", $params);
    }
    
    // Search
    public function searchRepositories(string $query, array $params = []): array {
        return $this->get('/search/repositories', array_merge(
            ['q' => $query],
            $params
        ));
    }
    
    public function searchCode(string $query, array $params = []): array {
        return $this->get('/search/code', array_merge(
            ['q' => $query],
            $params
        ));
    }
    
    // Rate Limit
    public function getRateLimit(): array {
        return $this->get('/rate_limit');
    }
    
    // Pagination helper
    public function paginate(string $endpoint, array $params = [], int $maxPages = 10): \Generator {
        $page = 1;
        
        while ($page <= $maxPages) {
            $response = $this->get($endpoint, array_merge($params, ['page' => $page, 'per_page' => 100]));
            
            if (empty($response)) break;
            
            foreach ($response as $item) {
                yield $item;
            }
            
            if (count($response) < 100) break; // last page
            $page++;
        }
    }
    
    // HTTP Methods
    private function get(string $endpoint, array $params = []): array {
        try {
            $options = [];
            if (!empty($params)) {
                $options['query'] = $params;
            }
            
            $response = $this->http->get($endpoint, $options);
            return json_decode($response->getBody()->getContents(), true);
        } catch (RequestException $e) {
            $this->handleException($e);
        }
    }
    
    private function post(string $endpoint, array $data): array {
        try {
            $response = $this->http->post($endpoint, ['json' => $data]);
            return json_decode($response->getBody()->getContents(), true);
        } catch (RequestException $e) {
            $this->handleException($e);
        }
    }
    
    private function patch(string $endpoint, array $data): array {
        try {
            $response = $this->http->patch($endpoint, ['json' => $data]);
            return json_decode($response->getBody()->getContents(), true);
        } catch (RequestException $e) {
            $this->handleException($e);
        }
    }
    
    private function handleException(RequestException $e): never {
        $statusCode = $e->getResponse()?->getStatusCode() ?? 0;
        $body = $e->getResponse()?->getBody()->getContents() ?? '';
        $errorData = json_decode($body, true);
        $message = $errorData['message'] ?? $e->getMessage();
        
        throw match($statusCode) {
            401 => new GitHubAuthException("Authentication failed: {$message}"),
            403 => new GitHubRateLimitException("Rate limit or permission error: {$message}"),
            404 => new GitHubNotFoundException("Not found: {$message}"),
            422 => new GitHubValidationException("Validation failed: {$message}", $errorData['errors'] ?? []),
            default => new GitHubException("GitHub API Error {$statusCode}: {$message}"),
        };
    }
}

// Exception classes
class GitHubException extends \RuntimeException {}
class GitHubAuthException extends GitHubException {}
class GitHubRateLimitException extends GitHubException {}
class GitHubNotFoundException extends GitHubException {}
class GitHubValidationException extends GitHubException {
    public function __construct(string $message, private array $errors = []) {
        parent::__construct($message);
    }
    public function getErrors(): array { return $this->errors; }
}

// ============================================================
// ตัวอย่างการใช้งาน
// ============================================================

function demonstrateGitHubClient(): void {
    $token = getenv('GITHUB_TOKEN') ?: 'your-github-token';
    $github = new GitHubClient($token);
    
    try {
        // ดูข้อมูล user
        echo "=== GitHub User ===\n";
        $user = $github->getUser('torvalds');
        echo "Name: {$user['name']}\n";
        echo "Followers: {$user['followers']}\n";
        echo "Repos: {$user['public_repos']}\n";
        
        // ค้นหา repositories
        echo "\n=== Search PHP Repos ===\n";
        $results = $github->searchRepositories('PHP framework stars:>1000', [
            'sort' => 'stars',
            'order' => 'desc',
        ]);
        
        foreach (array_slice($results['items'], 0, 5) as $repo) {
            echo "⭐ {$repo['stargazers_count']} - {$repo['full_name']}\n";
        }
        
        // Rate limit
        echo "\n=== Rate Limit ===\n";
        $limits = $github->getRateLimit();
        $core = $limits['resources']['core'];
        echo "Remaining: {$core['remaining']}/{$core['limit']}\n";
        echo "Reset: " . date('H:i:s', $core['reset']) . "\n";
        
    } catch (GitHubAuthException $e) {
        echo "Auth Error: " . $e->getMessage() . "\n";
    } catch (GitHubRateLimitException $e) {
        echo "Rate Limit: " . $e->getMessage() . "\n";
    } catch (GitHubException $e) {
        echo "GitHub Error: " . $e->getMessage() . "\n";
    }
}

demonstrateGitHubClient();
```

---

## Quiz

### คำถาม 1
json_encode() option ใดที่ใช้เพื่อให้ภาษาไทยแสดงผลได้ถูกต้อง (ไม่เป็น \uXXXX)?
- A. JSON_PRETTY_PRINT
- B. JSON_UNESCAPED_UNICODE
- C. JSON_HEX_TAG
- D. JSON_NUMERIC_CHECK

**เฉลย: B** - `JSON_UNESCAPED_UNICODE` ป้องกันการแปลง Unicode characters เป็น escape sequences

### คำถาม 2
HTTP method ใดที่ REST API ใช้สำหรับ partial update?
- A. POST
- B. PUT
- C. PATCH
- D. UPDATE

**เฉลย: C** - `PATCH` ใช้สำหรับ partial update, `PUT` ใช้สำหรับ full replacement

### คำถาม 3
ต้องการส่ง JSON body ใน Guzzle POST request ต้องใช้ option อะไร?
- A. `'body' => json_encode($data)`
- B. `'json' => $data`
- C. `'form_params' => $data`
- D. `'data' => $data`

**เฉลย: B** - `'json'` option จะ encode array เป็น JSON และตั้ง Content-Type header ให้อัตโนมัติ

### คำถาม 4
ทำไมถึงต้องใช้ Pool ใน Guzzle แทน async promises ธรรมดา?

**เฉลย:** Pool ควบคุม concurrency (จำนวน request พร้อมกัน) ซึ่งสำคัญมากเมื่อต้องส่ง request จำนวนมาก เช่น 100 requests ถ้าส่งพร้อมกันทั้งหมดอาจทำให้ server ปลายทาง rate limit หรือ server ล่ม Pool ช่วยจำกัดเช่น concurrency 5 ส่งทีละ 5 requests

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- **JSON** - encode/decode พร้อม options ต่างๆ และ JsonSerializable
- **REST API** - สร้าง API ด้วย PHP vanilla พร้อม routing, controllers, responses
- **cURL** - ส่ง HTTP requests และ concurrent requests ด้วย curl_multi
- **Guzzle** - HTTP client ระดับ production พร้อม middleware, async, pool
- **GitHub API Client** - ออกแบบ API client ที่สมบูรณ์และ reusable

---

## ➡️ Part ถัดไป

[Part 022: PHP Security](./part-022-php-security.md)
